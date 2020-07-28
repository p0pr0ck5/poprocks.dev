---
title: "Building an RPC System With Openresty, Part 1"
date: 2019-03-02T12:29:08-08:00
---

Most people don’t like flying, I think. No one likes standing in long lines or sitting around, but I don’t mind the extra free hours. It gives me a chance to hack around on fun things I normally wouldn’t have the time for. I’m on my way back home from San Francisco, so I took advantage of the time to start hacking around with building a simple RPC protocol in OpenResty. It’s been a good chance to work with binary data and exercise the Nginx stream module. Tossing some notes and snippets in here as an outlet; I’m hoping to have a more formal follow-up in a few weekends as life starts to settle back to normal.

<!--more-->

First up, we need to compile OpenResty with the stream-lua-module. The stream module version bundled with the current release of OpenResty doesn’t provided the peek method of reqsocket, so the socket calls themselves will rely on append a newline to the data packets, but we can still design the interface to declare the packet length in the body itself (ohp, I lied – while writing this, the official RC1 was kicked out).

We need interfaces for serialization. The data packets themselves are simply MessagePack over TCP (protobufs would be another fun option, if there’s a concrete specification for the application data):

```lua
-- serializes a msg into a format acceptable to ship over the wire
local function serialize(msg)
        local m = mp.pack(msg)
        local len = msg_len(m)
        return len .. m
end
_M.serialize = serialize

-- deserializes and returns a packet into a Lua table, as well as the packet length
local function deserialize(pkt)
        local len = pkt_len(pkt)
        local data = mp.unpack(pkt:sub(3))
        return data, len
end
_M.deserialize = deserialize
```

We calculate the length of the packet message and prepend it to the message itself when serializing the message. The message length header is comprised of two bytes, using a byte array to hold the data and coercing the bytes into a Lua string:

```lua
-- returns a two byte array as a lua string
-- indicating the length of the given proto message
local function msg_len(msg)
        local n = #msg
        assert(n < 57311)
        lbuf[0] = bit.rshift(n, 8) + 32
        lbuf[1] = bit.band(0xFF, n) + 32
        return ffi.string(lbuf, 2)
end

-- determines the length of the given packet
local function pkt_len(pkt)
        lbuf[0] = bit.lshift(tonumber(string.byte(pkt:sub(1, 1)) - 32), 8)
        lbuf[1] = tonumber(string.byte(pkt:sub(2, 2))) - 32
        return lbuf[0] + lbuf[1]
end
_M.pkt_len = pkt_len
```

I offset the length byte values by a little bit to avoid clashing with control characters – trying to call string.char on a byte containing 0x0a wasn’t actually returning back the value ’10’. I’d like to try to find a better solution for this, but the now-relatively arbitrary length limitation isn’t restrictive for something like this, so it’s not the end of the world.

From here, we can write a dispatch mechanism to take action based on the command sent in the packet:

```lua
local function build_res(msg)
        return serialize(msg)
end

local cmds = {
        "PING",
        "DELAY",
}
local handlers = {}

handlers.PING = function(msg)
        return build_res({ res = "PONG" })
end

handlers.DELAY = function(msg)
        if type(msg.sleep) ~= "number" or msg.sleep < 0 then
                return build_res({ err = "INVALID_SLEEP" })
        end
        local start = ngx.now()
        ngx.sleep(msg.sleep)
        return build_res({ res = "DONE", slept = ngx.now() - start })
end

local function handle_msg(msg)
        local cmd = msg.cmd
        if not cmd or not cmds[cmd] then
                return build_res({ err = "CMD_NOT_FOUND" })
        end
        return handlers[cmds[cmd]](msg)
end
_M.handle_msg = handle_msg
```

Standard stuff here, thrown together in a few minutes. The play between cmds and handlers lets the client send the command as an integer constant, rather than a string, to save space (which, yes, is minimal, but this is just for funsies :D). We can use a Lua content handler to read a packet and response appropriately:

```lua
server {
 listen 1235;

 content_by_lua_block {
    local comm = require "comm"
    local sock = assert(ngx.req.socket(true))

    local packet = sock:receive()
    local data, len = comm.deserialize(packet)
    ngx.log(ngx.NOTICE, "pkt len ", len)

    local res = comm.handle_msg(data)
    ngx.print(res .. "\n")
  }
}
```

And a simple little client:

```lua
local sock = ngx.socket.tcp()
local comm = require "comm"

local c, err = sock:connect("127.0.0.1", 1235)
if not c then error(err) end

local cmds = {
        [1] = "PING",
        PING = 1,
        [2] = "DELAY",
        DELAY = 2,
}

local req = {
        cmd = cmds[arg[1]]
}

if cmds[req.cmd] == "DELAY" then
        req.sleep = tonumber(arg[2])
end

local pkt = comm.serialize(req)
local b = sock:send(pkt .. "\n")

local r, err = sock:receive("*l")
local data, len = comm.deserialize(r)
print("you said " .. require("cjson").encode(data))
```

Again, remember that because we can’t peak into the stream to grab the header length bytes, we just need to read the whole packet directly. Giving it a bit of exercise, along with a packet dump showing the communications:

```bash
vagrant@ubuntu-bionic:~/stream$ resty ./client.lua PING
you said {"res":"PONG"}
vagrant@ubuntu-bionic:~/stream$ resty ./client.lua DELAY 1
you said {"res":"DONE","slept":1.0039999485016}
vagrant@ubuntu-bionic:~/stream$ resty ./client.lua DELAY -1
you said {"err":"INVALID_SLEEP"}
vagrant@ubuntu-bionic:~/stream$ resty ./client.lua FOO
you said {"err":"CMD_NOT_FOUND"}
```

```bash
vagrant@ubuntu-bionic:~$ sudo tcpdump -nni any port 1235 -X -s0
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on any, link-type LINUX_SLL (Linux cooked), capture size 262144 bytes
22:41:49.317755 IP 127.0.0.1.56288 > 127.0.0.1.1235: Flags [S], seq 2674771302, win 43690, options [mss 65495,sackOK,TS val 457297024 ecr 0,nop,wscale 7], length 0
	0x0000:  4500 003c 477e 4000 4006 f53b 7f00 0001  E..<G~@.@..;....
	0x0010:  7f00 0001 dbe0 04d3 9f6d c566 0000 0000  .........m.f....
	0x0020:  a002 aaaa fe30 0000 0204 ffd7 0402 080a  .....0..........
	0x0030:  1b41 cc80 0000 0000 0103 0307            .A..........
22:41:49.317787 IP 127.0.0.1.1235 > 127.0.0.1.56288: Flags [S.], seq 2342311135, ack 2674771303, win 43690, options [mss 65495,sackOK,TS val 457297024 ecr 457297024,nop,wscale 7], length 0
	0x0000:  4500 003c 0000 4000 4006 3cba 7f00 0001  E..<..@.@.<.....
	0x0010:  7f00 0001 04d3 dbe0 8b9c d4df 9f6d c567  .............m.g
	0x0020:  a012 aaaa fe30 0000 0204 ffd7 0402 080a  .....0..........
	0x0030:  1b41 cc80 1b41 cc80 0103 0307            .A...A......
22:41:49.317817 IP 127.0.0.1.56288 > 127.0.0.1.1235: Flags [.], ack 1, win 342, options [nop,nop,TS val 457297024 ecr 457297024], length 0
	0x0000:  4500 0034 477f 4000 4006 f542 7f00 0001  E..4G.@.@..B....
	0x0010:  7f00 0001 dbe0 04d3 9f6d c567 8b9c d4e0  .........m.g....
	0x0020:  8010 0156 fe28 0000 0101 080a 1b41 cc80  ...V.(.......A..
	0x0030:  1b41 cc80                                .A..
22:41:49.318536 IP 127.0.0.1.56288 > 127.0.0.1.1235: Flags [P.], seq 1:17, ack 1, win 342, options [nop,nop,TS val 457297025 ecr 457297024], length 16
	0x0000:  4500 0044 4780 4000 4006 f531 7f00 0001  E..DG.@.@..1....
	0x0010:  7f00 0001 dbe0 04d3 9f6d c567 8b9c d4e0  .........m.g....
	0x0020:  8018 0156 fe38 0000 0101 080a 1b41 cc81  ...V.8.......A..
	0x0030:  1b41 cc80 202d 82a5 736c 6565 7003 a363  .A...-..sleep..c
	0x0040:  6d64 020a                                md..
22:41:52.321647 IP 127.0.0.1.1235 > 127.0.0.1.56288: Flags [P.], seq 1:29, ack 17, win 342, options [nop,nop,TS val 457300029 ecr 457297025], length 28
	0x0000:  4500 0050 56f9 4000 4006 e5ac 7f00 0001  E..PV.@.@.......
	0x0010:  7f00 0001 04d3 dbe0 8b9c d4e0 9f6d c577  .............m.w
	0x0020:  8018 0156 fe44 0000 0101 080a 1b41 d83d  ...V.D.......A.=
	0x0030:  1b41 cc81 2039 82a3 7265 73a4 444f 4e45  .A...9..res.DONE
	0x0040:  a573 6c65 7074 cb40 0806 24e0 0000 000a  .slept.@..$.....
22:41:52.321680 IP 127.0.0.1.56288 > 127.0.0.1.1235: Flags [.], ack 29, win 342, options [nop,nop,TS val 457300029 ecr 457300029], length 0
	0x0000:  4500 0034 4781 4000 4006 f540 7f00 0001  E..4G.@.@..@....
	0x0010:  7f00 0001 dbe0 04d3 9f6d c577 8b9c d4fc  .........m.w....
	0x0020:  8010 0156 fe28 0000 0101 080a 1b41 d83d  ...V.(.......A.=
	0x0030:  1b41 d83d                                .A.=
22:41:52.321741 IP 127.0.0.1.1235 > 127.0.0.1.56288: Flags [F.], seq 29, ack 17, win 342, options [nop,nop,TS val 457300029 ecr 457300029], length 0
	0x0000:  4500 0034 56fa 4000 4006 e5c7 7f00 0001  E..4V.@.@.......
	0x0010:  7f00 0001 04d3 dbe0 8b9c d4fc 9f6d c577  .............m.w
	0x0020:  8011 0156 fe28 0000 0101 080a 1b41 d83d  ...V.(.......A.=
	0x0030:  1b41 d83d                                .A.=
22:41:52.325315 IP 127.0.0.1.56288 > 127.0.0.1.1235: Flags [F.], seq 17, ack 30, win 342, options [nop,nop,TS val 457300032 ecr 457300029], length 0
	0x0000:  4500 0034 4782 4000 4006 f53f 7f00 0001  E..4G.@.@..?....
	0x0010:  7f00 0001 dbe0 04d3 9f6d c577 8b9c d4fd  .........m.w....
	0x0020:  8011 0156 fe28 0000 0101 080a 1b41 d840  ...V.(.......A.@
	0x0030:  1b41 d83d                                .A.=
22:41:52.325341 IP 127.0.0.1.1235 > 127.0.0.1.56288: Flags [.], ack 18, win 342, options [nop,nop,TS val 457300032 ecr 457300032], length 0
	0x0000:  4500 0034 56fb 4000 4006 e5c6 7f00 0001  E..4V.@.@.......
	0x0010:  7f00 0001 04d3 dbe0 8b9c d4fd 9f6d c578  .............m.x
	0x0020:  8010 0156 fe28 0000 0101 080a 1b41 d840  ...V.(.......A.@
	0x0030:  1b41 d840                                .A.@
```

There’s a lot of fun potential with this, and I’m really enjoying playing around with some binary math and the streaming subsystem. I’m looking forward to fleshing this out in the upcoming weeks. The growth of microservice and service mesh patterns will mean that RESTful HTTP APIs will lose the massive foothold they have in modern architectures – gRPC, GraphQL, Thrift, and other protocols are continuing to grow and in adoption and will further shape the landscape of service communication in the coming years. OpenResty is positioning itself well to be a high-performance, flexible, and stable solution for binary and custom network protocols in the near future. It’s very exciting to see the developments being made and the possibilities opening up with existing and future releases.
