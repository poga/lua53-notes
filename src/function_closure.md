# Functions & Closures

In the source code, Lua Function is called `Closure`. A `Closure` is composed of `proto` and `UpVal`.

* A `proto` is basically a function not yet binded to `UpVal`.
* `proto` is generated at compile time. `closure` is generated at runtime.
* A `proto` can generate multiple `closure`.

The definition of `proto`.

```c
/*
** Function Prototypes
*/
typedef struct Proto {
  CommonHeader;
  lu_byte numparams;  /* number of fixed parameters */
  lu_byte is_vararg;
  lu_byte maxstacksize;  /* number of registers needed by this function */
  int sizeupvalues;  /* size of 'upvalues' */
  int sizek;  /* size of 'k' */
  int sizecode;
  int sizelineinfo;
  int sizep;  /* size of 'p' */
  int sizelocvars;
  int linedefined;  /* debug information  */
  int lastlinedefined;  /* debug information  */
  TValue *k;  /* constants used by the function */
  Instruction *code;  /* opcodes */
  struct Proto **p;  /* functions defined inside the function */
  int *lineinfo;  /* map from opcodes to source lines (debug information) */
  LocVar *locvars;  /* information about local variables (debug information) */
  Upvaldesc *upvalues;  /* upvalue information */
  struct LClosure *cache;  /* last-created closure with this prototype */
  TString  *source;  /* used for debug information */
  GCObject *gclist;
} Proto;
```

When parsing functions, Lua generate instructions with `luaK_codeABx`/`luaK_codeABC`, then collect them into a `FuncState`. A `FuncState` will be converted into a `Proto` after the parsing is finished.

If we compose a `Proto` and its corresponding `UpVal`s, we get a `Closure`. There are two kinds of `CLosure`:

* `LClosure` is implemented in Lua
* `CClosure` is implemented in C

```c
/*
** Closures
*/
typedef int (*lua_CFunction) (lua_State *L);

#define ClosureHeader \
    CommonHeader; lu_byte nupvalues; GCObject *gclist

typedef struct CClosure {
  ClosureHeader;
  lua_CFunction f;
  TValue upvalue[1];  /* list of upvalues */
} CClosure;


typedef struct LClosure {
  ClosureHeader;
  struct Proto *p;
  UpVal *upvals[1];  /* list of upvalues */
} LClosure;

typedef union Closure {
  CClosure c;
  LClosure l;
} Closure;
```

`CClosure` and `LClosure` both contains its upvalues and ClosureHeader. However, `CClosure` is just a function pointer and doesn't have a `Proto`.

With `ClosureHeader`, `Closure` are also chained together into a linked-list for garbage collection.
