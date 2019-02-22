# no-op macro

In the code, `lua_assert`, `lua_lock`, and `lua_unlock` are defined as `(void)0`, which is essentially a no-op. Why?

It's because:

* Vanilla Lua is pure ANSI C and runs on a single thread. So we don't need `lock` and `unlock`.
* Lua is designed to be ported to many different platforms. Porters can customize the behavior of Lua by overriding these macros. For example, you might need some form of `GIL` like Python if you want multi-threading.

You need to implement your own assert mechanism if you want to debug the internal of Lua

## References

* [https://stackoverflow.com/questions/3010974/purpose-of-lua-lock-and-lua-unlock](https://stackoverflow.com/questions/3010974/purpose-of-lua-lock-and-lua-unlock)
