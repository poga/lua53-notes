# Value & TValue

Everything in Lua is a `Value`.

```c
/*
** Union of all Lua values
*/
typedef union Value {
  GCObject *gc;    //* collectable objects *//
  void *p;         //* light userdata *//
  int b;           //* booleans *//
  lua_CFunction f; //* light C functions *//
  lua_Integer i;   //* integer numbers *//
  lua_Number n;    //* float numbers *//
} Value;
```

Values are categorized into two types: **collectible objects** and **others**. Collectible Objects are subject to garbage collection, others are not.

The definition of `GCObject` is:

```c
/*
** Common type for all collectable objects/
*/
typedef struct GCObject GCObject;

/*
** Common Header for all collectable objects (in macro form, to be
** included in other objects)
*/
#define CommonHeader	GCObject *next; lu_byte tt; lu_byte marked

/*
** Common type has only the common header
*/
struct GCObject {
  CommonHeader;
};
```

`GCObject` are chained together as a linked list. This makes the implementation of mark-and-sweep much simpler.

Instead of `Value`,  `TValue` is used most of the time. It's just a `Value` and a type tag `tt_`.

```c
#define TValuefields	Value value_; int tt_

typedef struct lua_TValue {
  TValuefields;
} TValue;

/*
** tags for Tagged Values have the following use of bits:
** bits 0-3: actual tag (a LUA_T* value)
** bits 4-5: variant bits
** bit 6: whether value is collectable
*/

#define BIT_ISCOLLECTABLE	(1 << 6)

#define iscollectable(o)	(rttype(o) & BIT_ISCOLLECTABLE)

#define rttype(o)	((o)->tt_)
```

The tag `tt_` also indicates whether a value is collectible.

