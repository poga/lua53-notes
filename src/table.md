# Table

In lua, a table is composed by 3 parts: `TKey`, `Node`, and `Table`.

```c
/*
** Tables
*/

typedef union TKey {
  struct {
    TValuefields;
    int next;  /* for chaining (offset for next node) */
  } nk;
  TValue tvk;
} TKey;

typedef struct Node {
  TValue i_val;
  TKey i_key;
} Node;

typedef struct Table {
  CommonHeader;
  lu_byte flags;  /* 1<<p means tagmethod(p) is not present */
  lu_byte lsizenode;  /* log2 of size of 'node' array */
  unsigned int sizearray;  /* size of 'array' array */
  TValue *array;  /* array part */
  Node *node;
  Node *lastfree;  /* any free position is before this position */
  struct Table *metatable;
  GCObject *gclist;
} Table;
```

A `Table` is built by multiple `Node`. Every `Node` is a pair of `TKey` and `TValue`.

In previous chapter, we know that the definition of `TValue` is:

```c
typedef struct lua_TValue {
  TValuefields;
} TValue;
```

However, this makes `TKey` looks unnecessarily complex. Why are we making a union on two `TValuefields`?

In fact, it's for better memory alignment. [The size of a `TValue` can be larger than the size of its elements](http://lua-users.org/lists/lua-l/2014-02/msg00158.html). The same technique is also used for Strings.

## Array part and Hash part

Lua table is notorious for its duality between arrays and hashtables. The duality is also reflected on the implmentation of Table. A quote from the comment in the source code:

> Implementation of tables (aka arrays, objects, or hash tables).
>
>  Tables keep its elements in two parts: an array part and a hash part. Non-negative integer keys are all candidates to be kept in the array part. The actual size of the array is the largest 'n' such that more than half the slots between 1 and n are in use. Hash uses a mix of chained scatter table with Brent's variation. A main invariant of these tables is that, if an element is not in its main position (i.e. the 'original' position that its hash gives to it), then the colliding element is in its own main position. Hence even when the load factor reaches 100%, performance remains good.

The **array part** is for optimizing integer keys in the table. The rest of the keys is stored in the **hash part**. For example, if we want to look up for a integer key in a table:

```c
/*
** search function for integers
*/
const TValue *luaH_getint (Table *t, lua_Integer key) {
  /* (1 <= key && key <= t->sizearray) */
  if (l_castS2U(key) - 1 < t->sizearray)
    return &t->array[key - 1];
  else {
    Node *n = hashint(t, key);
    for (;;) {  /* check whether 'key' is somewhere in the chain */
      if (ttisinteger(gkey(n)) && ivalue(gkey(n)) == key)
        return gval(n);  /* that's it */
      else {
        int nx = gnext(n);
        if (nx == 0) break;
        n += nx;
      }
    }
    return luaO_nilobject;
  }
}
```

We can see that if the size of array part is larger than the key, we can return the value stored in the array part directly.

The design of lua table makes it works like a mix between normal array, sparse array, and hashtable, which makes lua table well-equiped for all kinds of usage pattern.


## Inserting a new key

A key can be stored in the array part **or** the hash part. To maintain good performance, it's important to strike a balance between the usage of array part and hash part.

Here's the pseudo code of inserting a new key into a table:

```
1. If the new key is a float
1.1 if the key can be represented as a integer, convert it to integer
1.2 if the key is NaN, return error

2. find the main position of the key
2.1 if the main position if occupied, check if the occupier is on its main position.
2.1.1 if the occupier is also on its main position, insert the new key to the free position
2.1.2 if the occupier is not ot its main position, move the occupier to the free position and free the position we want.
2.1.3 if there's no free position available, invoke `rehash` to resize both the array part and the hash part.
```

And the code is:

```c
/*
** inserts a new key into a hash table; first, check whether key's main
** position is free. If not, check whether colliding node is in its main
** position or not: if it is not, move colliding node to an empty place and
** put new key in its main position; otherwise (colliding node is in its main
** position), new key goes to an empty position.
*/
TValue *luaH_newkey (lua_State *L, Table *t, const TValue *key) {
  Node *mp;
  TValue aux;
  if (ttisnil(key)) luaG_runerror(L, "table index is nil");
  else if (ttisfloat(key)) {
    lua_Integer k;
    if (luaV_tointeger(key, &k, 0)) {  /* does index fit in an integer? */
      setivalue(&aux, k);
      key = &aux;  /* insert it as an integer */
    }
    else if (luai_numisnan(fltvalue(key)))
      luaG_runerror(L, "table index is NaN");
  }
  mp = mainposition(t, key);
  if (!ttisnil(gval(mp)) || isdummy(t)) {  /* main position is taken? */
    Node *othern;
    Node *f = getfreepos(t);  /* get a free place */
    if (f == NULL) {  /* cannot find a free place? */
      rehash(L, t, key);  /* grow table */
      /* whatever called 'newkey' takes care of TM cache */
      return luaH_set(L, t, key);  /* insert key into grown table */
    }
    lua_assert(!isdummy(t));
    othern = mainposition(t, gkey(mp));
    if (othern != mp) {  /* is colliding node out of its main position? */
      /* yes; move colliding node into free position */
      while (othern + gnext(othern) != mp)  /* find previous */
        othern += gnext(othern);
      gnext(othern) = cast_int(f - othern);  /* rechain to point to 'f' */
      *f = *mp;  /* copy colliding node into free pos. (mp->next also goes) */
      if (gnext(mp) != 0) {
        gnext(f) += cast_int(mp - f);  /* correct 'next' */
        gnext(mp) = 0;  /* now 'mp' is free */
      }
      setnilvalue(gval(mp));
    }
    else {  /* colliding node is in its own main position */
      /* new node will go into free position */
      if (gnext(mp) != 0)
        gnext(f) = cast_int((mp + gnext(mp)) - f);  /* chain new position */
      else lua_assert(gnext(f) == 0);
      gnext(mp) = cast_int(f - mp);
      mp = f;
    }
  }
  setnodekey(L, &mp->i_key, key);
  luaC_barrierback(L, t, key);
  lua_assert(ttisnil(gval(mp)));
  return gval(mp);
}
```

There are many different hashtable implementation in the world. Lua choose [internal chained scatter table with Brent's variation](https://maths-people.anu.edu.au/~brent/pub/pub013.html) for performance and efficiency.

To know more about the main position/free position and internal chained scatter table. I recommend you to check the references for more detail:

* [The art of hashing](http://blog.reverberate.org/2009/02/art-of-hashing.html)
* [R. P. Brent, Reducing the retrieval time of scatter storage techniques, Communications of the ACM 16 (1973), 105-109.](https://maths-people.anu.edu.au/~brent/pub/pub013.html)


## Balancing between the array part and the hash part

Finally, let's see the implement of `rehash`. `rehash` is responsible for keeping the balance between the array part and the hash part. The design is pretty simple and efficient.

First, a `nums` array is used to keep track of the number of integer keys in each interval `i`.

```c
// nums[i] = number of keys 'k' where 2^(i - 1) < k <= 2^i
```

Then, `rehash` try to make sure each part contains about the half of the all integer keys. Here's the pseudo code:

```
1. count keys in the array part, save to nums
2. count integer keys in the hash part, save to nums
3. count the number of non-interger keys
4. Find out the `nums` interval which contains half of the total keys, resize array part to fit the interval, the rest goes to the hash part.
```

and the actual implementation:

```c
/*
** nums[i] = number of keys 'k' where 2^(i - 1) < k <= 2^i
*/
static void rehash (lua_State *L, Table *t, const TValue *ek) {
  unsigned int asize;  /* optimal size for array part */
  unsigned int na;  /* number of keys in the array part */
  unsigned int nums[MAXABITS + 1];
  int i;
  int totaluse;
  for (i = 0; i <= MAXABITS; i++) nums[i] = 0;  /* reset counts */
  na = numusearray(t, nums);  /* count keys in array part */
  totaluse = na;  /* all those keys are integer keys */
  totaluse += numusehash(t, nums, &na);  /* count keys in hash part */
  /* count extra key */
  na += countint(ek, nums);
  totaluse++;
  /* compute new size for array part */
  asize = computesizes(nums, &na);
  /* resize the table to new computed sizes */
  luaH_resize(L, t, asize, totaluse - na);
}
```
