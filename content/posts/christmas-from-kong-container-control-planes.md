---
title: "Christmas from Kong? Container Control Planes!"
date: 2019-12-21T11:52:53-08:00
author: poprocks
---

Christmas came a few days early from the Kong family. The first release candidate of the Kong 2.0 release was pushed out yesterday. The biggest highlight of this release is the shiny new “hybrid mode” deployment, which provides a more formal separation of control plane (e.g., management of the Kong cluster) behavior from data plane (e.g., proxying user-facing traffic). I had some time to play with this and wanted to share a few extra thoughts beyond what’s in the release notes.

<!--more-->

_Disclaimer: I am a Kong contributor and employee. This blog post and my opinions are my own._

Getting up and running quickly with a Kong control plane and data plane is straightforward via Docker, though a bit of extra legwork is needed to get mTLS communication configured. First, we need a backing data store for the control plane to hold the state of the world:

```bash
$ docker run -d --name kong-database \
  --network kong \
  -p 5432:5432 \
  -e "POSTGRES_USER=kong" \
  -e "POSTGRES_DB=kong" \
  postgres:9.6

$ docker run -d --name kong-database \
  --network kong \
  -p 5432:5432 \
  -e "POSTGRES_USER=kong" \
  -e "POSTGRES_DB=kong" \
  postgres:9.6
```

And as with all Kong installations, the database needs to be bootstrapped:

```bash
$ docker run --rm \
  --network kong \
  -e "KONG_DATABASE=postgres" \
  -e "KONG_PG_HOST=kong-database" \
  kong:2.0.0rc1 kong migrations bootstrap

$ docker run --rm \
  --network kong \
  -e "KONG_DATABASE=postgres" \
  -e "KONG_PG_HOST=kong-database" \
  kong:2.0.0rc1 kong migrations bootstrap
```

Next, we need to setup the control plane. Before we can start the control plane process, we need to generate the keypair used for mTLS between control plane and data plane nodes. I say ‘keypair’ (singular), because the current RC release of Kong does not provide a formal PKI mechanism. The certificate and key are shared between control plane and data plane nodes. This provides for a fully functional mutual TLS handshake, but every control plane and data plane node need identical keypair material to communicate. There is a Kong CLI subcommand to generate an EC cert and key, but for setting up a Docker playground I found it was easier to just create my own:

```bash
$ cd /scratch

$ openssl req -x509 -newkey rsa:4096 -keyout cluster.key -out cluster.crt -days 365 -nodes -subj '/CN=kong_clustering/'

$ chmod 644 cluster.key
$ cd /scratch

$ openssl req -x509 -newkey rsa:4096 -keyout cluster.key -out cluster.crt -days 365 -nodes -subj '/CN=kong_clustering/'

$ chmod 644 cluster.key
```

The OpenSSL CLI command to generate the self-signed certificate will write the key file with restrictive permissions; this needs to be made readable for the user running Kong inside the Docker container, otherwise starting the control plane will fail with:

```bash
$ docker logs kong-cp
nginx: [emerg] SSL_CTX_use_PrivateKey_file("/data/cluster.key") failed (SSL: error:0200100D:system library:fopen:Permission denied:fopen('/data/cluster.key','r') error:20074002:BIO routines:file_ctrl:system lib error:140B0002:SSL routines:SSL_CTX_use_PrivateKey_file:system lib)

$ docker logs kong-cp
nginx: [emerg] SSL_CTX_use_PrivateKey_file("/data/cluster.key") failed (SSL: error:0200100D:system library:fopen:Permission denied:fopen('/data/cluster.key','r') error:20074002:BIO routines:file_ctrl:system lib error:140B0002:SSL routines:SSL_CTX_use_PrivateKey_file:system lib)
```

Once this is done, we can mount the key material into the container and start the control plane process:

```bash
$ docker run -d --name kong-cp \
  --network kong \
  -e "KONG_DATABASE=postgres" \
  -e "KONG_PG_HOST=kong-database" \
  -e "KONG_PROXY_ACCESS_LOG=/dev/stdout" \
  -e "KONG_ADMIN_ACCESS_LOG=/dev/stdout" \
  -e "KONG_PROXY_ERROR_LOG=/dev/stderr" \
  -e "KONG_ADMIN_ERROR_LOG=/dev/stderr" \
  -e "KONG_ADMIN_LISTEN=0.0.0.0:8001" \
  -e "KONG_ROLE=control_plane" \
  -e "KONG_CLUSTER_CERT=/data/cluster.crt" \
  -e "KONG_CLUSTER_CERT_KEY=/data/cluster.key" \
  -v "/scratch:/data" \
  -p 8001:8001 \
  -p 8005:8005 \
  kong:2.0.0rc1

$ docker run -d --name kong-cp \
  --network kong \
  -e "KONG_DATABASE=postgres" \
  -e "KONG_PG_HOST=kong-database" \
  -e "KONG_PROXY_ACCESS_LOG=/dev/stdout" \
  -e "KONG_ADMIN_ACCESS_LOG=/dev/stdout" \
  -e "KONG_PROXY_ERROR_LOG=/dev/stderr" \
  -e "KONG_ADMIN_ERROR_LOG=/dev/stderr" \
  -e "KONG_ADMIN_LISTEN=0.0.0.0:8001" \
  -e "KONG_ROLE=control_plane" \
  -e "KONG_CLUSTER_CERT=/data/cluster.crt" \
  -e "KONG_CLUSTER_CERT_KEY=/data/cluster.key" \
  -v "/scratch:/data" \
  -p 8001:8001 \
  -p 8005:8005 \
  kong:2.0.0rc1
```

Note that we expose port 8001 for Admin API traffic, and 8005 for control plane traffic. We can now start a data plane node and configure it to communicate with the control plane:

```bash
$ docker run -d --name kong-dp \
  --network kong \
  -e "KONG_DATABASE=off" \
  -e "KONG_PROXY_ACCESS_LOG=/dev/stdout" \
  -e "KONG_ADMIN_ACCESS_LOG=/dev/stdout" \
  -e "KONG_PROXY_ERROR_LOG=/dev/stderr" \
  -e "KONG_ADMIN_ERROR_LOG=/dev/stderr" \
  -e "KONG_ROLE=data_plane" \
  -e "KONG_CLUSTER_CONTROL_PLANE=kong-cp:8005" \
  -e "KONG_LUA_SSL_TRUSTED_CERTIFICATE=/data/cluster.crt" \
  -e "KONG_CLUSTER_CERT=/data/cluster.crt" \
  -e "KONG_CLUSTER_CERT_KEY=/data/cluster.key" \
  -v "/scratch:/data" \
  -p 8000:8000 \
  kong:2.0.0rc1

$ docker run -d --name kong-dp \
  --network kong \
  -e "KONG_DATABASE=off" \
  -e "KONG_PROXY_ACCESS_LOG=/dev/stdout" \
  -e "KONG_ADMIN_ACCESS_LOG=/dev/stdout" \
  -e "KONG_PROXY_ERROR_LOG=/dev/stderr" \
  -e "KONG_ADMIN_ERROR_LOG=/dev/stderr" \
  -e "KONG_ROLE=data_plane" \
  -e "KONG_CLUSTER_CONTROL_PLANE=kong-cp:8005" \
  -e "KONG_LUA_SSL_TRUSTED_CERTIFICATE=/data/cluster.crt" \
  -e "KONG_CLUSTER_CERT=/data/cluster.crt" \
  -e "KONG_CLUSTER_CERT_KEY=/data/cluster.key" \
  -v "/scratch:/data" \
  -p 8000:8000 \
  kong:2.0.0rc1
```

We rely on Docker’s network namespace DNS to forward control plane traffic from the data plane node, to the appropriate container. We’ve also disabled the database, as the data plane node will communicate only with the control plane for runtime configuration. When the data plane node comes up, it contacts the control plane and fetches the cluster configuration. This is noted in the logs:

```bash
$ docker logs kong-dp
<...snip...>
2019/12/24 21:38:58 [notice] 31#0: *13 [lua] init.lua:155: parse_table(): {"_format_version":"1.1"}, context: ngx.timer
2019/12/24 21:38:58 [notice] 31#0: *13 [lua] cache.lua:332: purge(): [DB cache] purging (local) cache, context: ngx.timer

$ docker logs kong-dp
<...snip...>
2019/12/24 21:38:58 [notice] 31#0: *13 [lua] init.lua:155: parse_table(): {"_format_version":"1.1"}, context: ngx.timer
2019/12/24 21:38:58 [notice] 31#0: *13 [lua] cache.lua:332: purge(): [DB cache] purging (local) cache, context: ngx.timer
```

We now can add a playground Service and Route to the cluster, and see the configuration results impacted on the data plane in real time:

```bash
$ http :8001/services name=test host=httpbin.org
HTTP/1.1 201 Created
Access-Control-Allow-Origin: *
Connection: keep-alive
Content-Length: 285
Content-Type: application/json; charset=utf-8
Date: Tue, 24 Dec 2019 21:39:27 GMT
Server: kong/2.0.0rc1
X-Kong-Admin-Latency: 114

{
    "client_certificate": null,
    "connect_timeout": 60000,
    "created_at": 1577223567,
    "host": "httpbin.org",
    "id": "b94fbfdc-0eac-4f7f-bf79-88af15c169c1",
    "name": "test",
    "path": null,
    "port": 80,
    "protocol": "http",
    "read_timeout": 60000,
    "retries": 5,
    "tags": null,
    "updated_at": 1577223567,
    "write_timeout": 60000
}

$ http :8001/services/test/routes paths:='["/"]'
HTTP/1.1 201 Created
Access-Control-Allow-Origin: *
Connection: keep-alive
Content-Length: 420
Content-Type: application/json; charset=utf-8
Date: Tue, 24 Dec 2019 21:39:41 GMT
Server: kong/2.0.0rc1
X-Kong-Admin-Latency: 14

{
    "created_at": 1577223581,
    "destinations": null,
    "headers": null,
    "hosts": null,
    "https_redirect_status_code": 426,
    "id": "9619dd61-4faa-41f7-a0fe-13f21bbb446b",
    "methods": null,
    "name": null,
    "path_handling": "v0",
    "paths": [
        "/"
    ],
    "preserve_host": false,
    "protocols": [
        "http",
        "https"
    ],
    "regex_priority": 0,
    "service": {
        "id": "b94fbfdc-0eac-4f7f-bf79-88af15c169c1"
    },
    "snis": null,
    "sources": null,
    "strip_path": true,
    "tags": null,
    "updated_at": 1577223581
}

$ http :8000/get
HTTP/1.1 200 OK
Access-Control-Allow-Credentials: true
Access-Control-Allow-Origin: *
Connection: keep-alive
Content-Encoding: gzip
Content-Length: 205
Content-Type: application/json
Date: Tue, 24 Dec 2019 21:39:52 GMT
Referrer-Policy: no-referrer-when-downgrade
Server: nginx
Via: kong/2.0.0rc1
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-Kong-Proxy-Latency: 107
X-Kong-Upstream-Latency: 138
X-XSS-Protection: 1; mode=block

{
    "args": {},
    "headers": {
        "Accept": "*/*",
        "Accept-Encoding": "gzip, deflate",
        "Host": "httpbin.org",
        "User-Agent": "HTTPie/0.9.8",
        "X-Forwarded-Host": "localhost"
    },
    "origin": "172.23.0.1, 76.250.198.113, 172.23.0.1",
    "url": "https://localhost/get"
}

$ http :8001/services name=test host=httpbin.org
HTTP/1.1 201 Created
Access-Control-Allow-Origin: *
Connection: keep-alive
Content-Length: 285
Content-Type: application/json; charset=utf-8
Date: Tue, 24 Dec 2019 21:39:27 GMT
Server: kong/2.0.0rc1
X-Kong-Admin-Latency: 114

{
    "client_certificate": null,
    "connect_timeout": 60000,
    "created_at": 1577223567,
    "host": "httpbin.org",
    "id": "b94fbfdc-0eac-4f7f-bf79-88af15c169c1",
    "name": "test",
    "path": null,
    "port": 80,
    "protocol": "http",
    "read_timeout": 60000,
    "retries": 5,
    "tags": null,
    "updated_at": 1577223567,
    "write_timeout": 60000
}

$ http :8001/services/test/routes paths:='["/"]'
HTTP/1.1 201 Created
Access-Control-Allow-Origin: *
Connection: keep-alive
Content-Length: 420
Content-Type: application/json; charset=utf-8
Date: Tue, 24 Dec 2019 21:39:41 GMT
Server: kong/2.0.0rc1
X-Kong-Admin-Latency: 14

{
    "created_at": 1577223581,
    "destinations": null,
    "headers": null,
    "hosts": null,
    "https_redirect_status_code": 426,
    "id": "9619dd61-4faa-41f7-a0fe-13f21bbb446b",
    "methods": null,
    "name": null,
    "path_handling": "v0",
    "paths": [
        "/"
    ],
    "preserve_host": false,
    "protocols": [
        "http",
        "https"
    ],
    "regex_priority": 0,
    "service": {
        "id": "b94fbfdc-0eac-4f7f-bf79-88af15c169c1"
    },
    "snis": null,
    "sources": null,
    "strip_path": true,
    "tags": null,
    "updated_at": 1577223581
}

$ http :8000/get
HTTP/1.1 200 OK
Access-Control-Allow-Credentials: true
Access-Control-Allow-Origin: *
Connection: keep-alive
Content-Encoding: gzip
Content-Length: 205
Content-Type: application/json
Date: Tue, 24 Dec 2019 21:39:52 GMT
Referrer-Policy: no-referrer-when-downgrade
Server: nginx
Via: kong/2.0.0rc1
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-Kong-Proxy-Latency: 107
X-Kong-Upstream-Latency: 138
X-XSS-Protection: 1; mode=block

{
    "args": {},
    "headers": {
        "Accept": "*/*",
        "Accept-Encoding": "gzip, deflate",
        "Host": "httpbin.org",
        "User-Agent": "HTTPie/0.9.8",
        "X-Forwarded-Host": "localhost"
    },
    "origin": "172.23.0.1, 76.250.198.113, 172.23.0.1",
    "url": "https://localhost/get"
}
```

Each update to the cluster runtime config is noted by the data plane:

```bash
$ docker logs kong-dp -f
<...snip...>
2019/12/24 21:39:27 [notice] 31#0: *13 [lua] init.lua:155: parse_table(): {"services":[{"host":"httpbin.org","created_at":1577223567,"connect_timeout":60000,"id":"b94fbfdc-0eac-4f7f-bf79-88af15c169c1","protocol":"http","name":"test","read_timeout":60000,"port":80,"updated_at":1577223567,"write_timeout":60000,"retries":5}],"_format_version":"1.1"}, context: ngx.timer
2019/12/24 21:39:27 [notice] 31#0: *13 [lua] cache.lua:332: purge(): [DB cache] purging (local) cache, context: ngx.timer
2019/12/24 21:39:41 [notice] 31#0: *13 [lua] init.lua:155: parse_table(): {"services":[{"host":"httpbin.org","created_at":1577223567,"connect_timeout":60000,"id":"b94fbfdc-0eac-4f7f-bf79-88af15c169c1","protocol":"http","name":"test","read_timeout":60000,"port":80,"updated_at":1577223567,"write_timeout":60000,"retries":5}],"_format_version":"1.1","routes":[{"created_at":1577223581,"strip_path":true,"path_handling":"v0","preserve_host":false,"regex_priority":0,"id":"9619dd61-4faa-41f7-a0fe-13f21bbb446b","paths":["\/"],"updated_at":1577223581,"https_redirect_status_code":426,"protocols":["http","https"],"service":"b94fbfdc-0eac-4f7f-bf79-88af15c169c1"}]}, context: ngx.timer
2019/12/24 21:39:41 [notice] 31#0: *13 [lua] cache.lua:332: purge(): [DB cache] purging (local) cache, context: ngx.timer

$ docker logs kong-dp -f
<...snip...>
2019/12/24 21:39:27 [notice] 31#0: *13 [lua] init.lua:155: parse_table(): {"services":[{"host":"httpbin.org","created_at":1577223567,"connect_timeout":60000,"id":"b94fbfdc-0eac-4f7f-bf79-88af15c169c1","protocol":"http","name":"test","read_timeout":60000,"port":80,"updated_at":1577223567,"write_timeout":60000,"retries":5}],"_format_version":"1.1"}, context: ngx.timer
2019/12/24 21:39:27 [notice] 31#0: *13 [lua] cache.lua:332: purge(): [DB cache] purging (local) cache, context: ngx.timer
2019/12/24 21:39:41 [notice] 31#0: *13 [lua] init.lua:155: parse_table(): {"services":[{"host":"httpbin.org","created_at":1577223567,"connect_timeout":60000,"id":"b94fbfdc-0eac-4f7f-bf79-88af15c169c1","protocol":"http","name":"test","read_timeout":60000,"port":80,"updated_at":1577223567,"write_timeout":60000,"retries":5}],"_format_version":"1.1","routes":[{"created_at":1577223581,"strip_path":true,"path_handling":"v0","preserve_host":false,"regex_priority":0,"id":"9619dd61-4faa-41f7-a0fe-13f21bbb446b","paths":["\/"],"updated_at":1577223581,"https_redirect_status_code":426,"protocols":["http","https"],"service":"b94fbfdc-0eac-4f7f-bf79-88af15c169c1"}]}, context: ngx.timer
2019/12/24 21:39:41 [notice] 31#0: *13 [lua] cache.lua:332: purge(): [DB cache] purging (local) cache, context: ngx.timer
```

Control plane nodes proactively push down content to the data plane. This provides for more responsive information dissemination than is currently provided by Kong’s polling-based communications mechanism in a traditional deployment.

Control plane nodes keep track of each connected data plane. A heartbeat is established between control plane and data plane nodes, allowing control planes to send smart diffs of the cluster configuration as necessary. We can see the update times tracked in the /clustering/status endpoint of the control plane Admin API:

```bash
$ http :8001/clustering/status
HTTP/1.1 200 OK
Access-Control-Allow-Origin: *
Connection: keep-alive
Content-Length: 158
Content-Type: application/json; charset=utf-8
Date: Tue, 24 Dec 2019 22:01:23 GMT
Server: kong/2.0.0rc1
X-Kong-Admin-Latency: 0

{
    "4a29bea4-9fba-4a11-9db2-494824d492e6": {
        "config_hash": "84c3bf402ab09c615978732d0796cb4e",
        "hostname": "59f2f1fd30d9",
        "ip": "172.23.0.4",
        "last_seen": 1577224858
    }
}

$ http :8001/clustering/status
HTTP/1.1 200 OK
Access-Control-Allow-Origin: *
Connection: keep-alive
Content-Length: 158
Content-Type: application/json; charset=utf-8
Date: Tue, 24 Dec 2019 22:01:23 GMT
Server: kong/2.0.0rc1
X-Kong-Admin-Latency: 0

{
    "4a29bea4-9fba-4a11-9db2-494824d492e6": {
        "config_hash": "84c3bf402ab09c615978732d0796cb4e",
        "hostname": "59f2f1fd30d9",
        "ip": "172.23.0.4",
        "last_seen": 1577224858
    }
}
$ echo $(($(date +%s) - $(http :8001/clustering/status | jq -r .[].last_seen))) ;  sleep 3; !!
29
2

$ echo $(($(date +%s) - $(http :8001/clustering/status | jq -r .[].last_seen))) ;  sleep 3; !!
29
2
```

So it looks like there’s a 30 second heartbeat ticker (heh).

Data planes were designed to be resilient against failure of the control plane. Once a data plane has received its cluster config, it can continue to proxy traffic even if the connection to the control plane is broken:

```bash
$ docker rm -f kong-cp
kong-cp

$ http :8000/get
HTTP/1.1 200 OK
Access-Control-Allow-Credentials: true
Access-Control-Allow-Origin: *
Connection: keep-alive
Content-Encoding: gzip
Content-Length: 202
Content-Type: application/json
Date: Tue, 24 Dec 2019 22:03:24 GMT
Referrer-Policy: no-referrer-when-downgrade
Server: nginx
Via: kong/2.0.0rc1
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-Kong-Proxy-Latency: 78
X-Kong-Upstream-Latency: 141
X-XSS-Protection: 1; mode=block

{
    "args": {},
    "headers": {
        "Accept": "*/*",
        "Accept-Encoding": "gzip, deflate",
        "Host": "httpbin.org",
        "User-Agent": "HTTPie/0.9.8",
        "X-Forwarded-Host": "localhost"
    },
    "origin": "172.23.0.1, 98.173.7.34, 172.23.0.1",
    "url": "https://localhost/get"
}

$ docker rm -f kong-cp
kong-cp

$ http :8000/get
HTTP/1.1 200 OK
Access-Control-Allow-Credentials: true
Access-Control-Allow-Origin: *
Connection: keep-alive
Content-Encoding: gzip
Content-Length: 202
Content-Type: application/json
Date: Tue, 24 Dec 2019 22:03:24 GMT
Referrer-Policy: no-referrer-when-downgrade
Server: nginx
Via: kong/2.0.0rc1
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-Kong-Proxy-Latency: 78
X-Kong-Upstream-Latency: 141
X-XSS-Protection: 1; mode=block

{
    "args": {},
    "headers": {
        "Accept": "*/*",
        "Accept-Encoding": "gzip, deflate",
        "Host": "httpbin.org",
        "User-Agent": "HTTPie/0.9.8",
        "X-Forwarded-Host": "localhost"
    },
    "origin": "172.23.0.1, 98.173.7.34, 172.23.0.1",
    "url": "https://localhost/get"
}
```

And of course, the data plane will whine at you until the connection is re-established:

```bash
$ docker logs kong-dp
<...snip...>
2019/12/24 22:05:13 [error] 31#0: *50572 [lua] clustering.lua:128: connection to control plane broken: failed to connect: [cosocket] DNS resolution failed: dns server error: 3 name error. Tried: ["(short)kong-cp:(na) - cache-miss","kong-cp:1 - cache-miss/scheduled/querying/dns server error: 3 name error","kong-cp.medusa.mezzonet.net:1 - cache-miss/scheduled/querying/dns server error: 3 name error","kong-cp:33 - cache-miss/scheduled/querying/dns server error: 3 name error","kong-cp.medusa.mezzonet.net:33 - cache-miss/scheduled/querying/dns server error: 3 name error","kong-cp:5 - cache-miss/scheduled/querying/dns server error: 3 name error","kong-cp.medusa.mezzonet.net:5 - cache-miss/scheduled/querying/dns server error: 3 name error"] retrying after 7 seconds, context: ngx.timer
$ docker logs kong-dp
<...snip...>
2019/12/24 22:05:13 [error] 31#0: *50572 [lua] clustering.lua:128: connection to control plane broken: failed to connect: [cosocket] DNS resolution failed: dns server error: 3 name error. Tried: ["(short)kong-cp:(na) - cache-miss","kong-cp:1 - cache-miss/scheduled/querying/dns server error: 3 name error","kong-cp.medusa.mezzonet.net:1 - cache-miss/scheduled/querying/dns server error: 3 name error","kong-cp:33 - cache-miss/scheduled/querying/dns server error: 3 name error","kong-cp.medusa.mezzonet.net:33 - cache-miss/scheduled/querying/dns server error: 3 name error","kong-cp:5 - cache-miss/scheduled/querying/dns server error: 3 name error","kong-cp.medusa.mezzonet.net:5 - cache-miss/scheduled/querying/dns server error: 3 name error"] retrying after 7 seconds, context: ngx.timer
```

Data plane nodes will store their copy of the cluster config on the file system, in Kong’s runtime prefix (by default /usr/local/kong). This means we could theoretically mount the prefix location to the host, and provide a path for data nodes to be robust against failure alongside a simultaneous failure of the control plane (e.g., new data plane containers can be launched, and use a previously-pushed copy of the cluster config, even if no control plane is available during the initialization phase). This is a huge improvement over the traditional deployment model for Kong, where the backing data store (Postgres or Cassandra) must be available when a Kong process starts.

There’s a lot to look forward to in the GA release of this series. I’m particularly excited about a support for a formal PKI mechanism in hybrid mode, as well as a lot of the other Christmas goodies announced by the development team. Check out both the release announcement, and the preview docs for hybrid mode.
