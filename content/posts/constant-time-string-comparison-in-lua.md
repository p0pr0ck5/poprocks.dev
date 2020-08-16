---
title: "Constant Time String Comparison in Lua"
date: 2017-04-28T12:56:37-08:00
---

```lua
str1 == str2
```

No really, it's that easy.

<!--more-->

I’m not going into the particulars of why constant time string comparison is important. Let’s leave that for the smart people, yeah? Here, we can examine the underlying mechanics of the Lua interpreter (PUC Lua, not LuaJIT, largely because I still contend that Mike Pall is a pseudonym), and the exact facilities used to compare two strings.

Conventional wisdom would tell us that interpreted languages short-circuit string comparisons in the name of performance (Lua being no exception); therefore, we need to circumvent built-in equality operators by comparing each byte manually. After all, crypto is hard, so string comparison should be hard, right? Well, no. Lua strings are interned, leading to a variation from expected behavior in interpreted languages. It follows, then, that comparison of interned strings inherently relies on a mechanism other than a byte-by-byte comparison of each element; let’s examine this theory.

Assuming that we’re still styudying the `==` operator, LuaForge gives us a starting point:

_EQ A B C if ((RK(B) == RK(C)) ~= A) then PC++_

Yay, letters! Okay, so the result of execution of the EQ operator is that, if RK(B) and RK(C) don’t equal A, then we increment PC. Without any other context this is fairly meaningless. What exactly is the EQ operator doing? From here we can consult the source: a quick parse of the 5.1 source tree lands us here:

```c
case OP_EQ: {
  TValue *rb = RKB(i);
  TValue *rc = RKC(i);
  Protect(
    if (equalobj(L, rb, rc) == GETARG_A(i))
      dojump(L, pc, GETARG_sBx(*pc));
  )
  pc++;
  continue;
}
```

This corresponds with our previous finding: assuming that elements b and c are equals, we execute a jump. The interesting call here, `equalobj`, is just a type-checking wrapper around `luaV_equalval`:

```c
int luaV_equalval (lua_State *L, const TValue *t1, const TValue *t2) {
  const TValue *tm;
  lua_assert(ttype(t1) == ttype(t2));
  switch (ttype(t1)) {
    case LUA_TNIL: return 1;
    case LUA_TNUMBER: return luai_numeq(nvalue(t1), nvalue(t2));
    case LUA_TBOOLEAN: return bvalue(t1) == bvalue(t2);  /* true must be 1 !! */
    case LUA_TLIGHTUSERDATA: return pvalue(t1) == pvalue(t2);
    case LUA_TUSERDATA: {
      if (uvalue(t1) == uvalue(t2)) return 1;
      tm = get_compTM(L, uvalue(t1)->metatable, uvalue(t2)->metatable,
                         TM_EQ);
      break;  /* will try TM */
    }
    case LUA_TTABLE: {
      if (hvalue(t1) == hvalue(t2)) return 1;
      tm = get_compTM(L, hvalue(t1)->metatable, hvalue(t2)->metatable, TM_EQ);
      break;  /* will try TM */
    }
    default: return gcvalue(t1) == gcvalue(t2);
  }
  if (tm == NULL) return 0;  /* no TM? */
  callTMres(L, L->top, tm, t1, t2);  /* call TM */
  return !l_isfalse(L->top);n b
}
```

Interestingly, the case we care about (where the type of t1 is a string) is the default case, so we’re comparing the primitive results of gcvalue for each parameter. From here, we just need to keep walking through macro access values (spared here for brevity), showing that we’re just comparing the GC values for the two compared strings. Thus, a byte-by-byte comparison is unnecessary when comparing strings in constant time.

_"But, what about timing attacks against GC pauses???"_ I hear you cry. Great thought! tedu even brought this up a few years ago. It’s not a bad idea- before you realize any real-world application would probably take longer than the lifetime of the universe to yield useful results. Besides, we can throw off attackers with tarpit defenses, rendering this side channel completely useless (and come on, if you’re exposing a vector doing such a comparison vulnerable to this kind of search, you’re doing something wayyyyy wrong).

Let’s not neglect our humble equality operator friend. He’s here to help. Abusing his faculties in the name of security theater is wrong. We’re all `==`.
