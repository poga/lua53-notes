# Garbage Collection

The most important aspect of GC is its design and trade-off. There's no "best GC" in the world.

In this chapter, we will focus on the high-level concept of Lua's GC. For more detail, check references

## History of Lua's GC


### mark-and-sweep

Before Lua 5.0, a simple mark-and-sweep is used by Lua. The idea is simple: Traverse all objects from the root, mark all visited object and release the un-visitable.

However, mark-and-sweep suffer from the stop-the-world, non-incremental algorithm. You have to check **all objects** at once, and **the object tree can't be modified** while the mark process is running. The performance is also proportional to the memory usage: More memory you use, Slower the GC will perform.

### incremental GC

Lua 5.1 employed a incremental GC. Incremental GC is based on the idea of the **mutator**: From the perspective of garbage collector, the program is just an outside who keep changing the memory. If we can keep track of which memory location is changed by the program, we don't have to check the whole memory every single time.

At the moment, Lua implemented tri-color marking. The algorithm can reduce the GC latency. However, the cost of allocation and space-cost is increased.

### (abandoned) generational GC

Lua 5.2 tried to switch to a generator GC. Unfortunately, it's reverted in Lua 5.3 due to some bug and naive implementation.

### Lua 5.4?

Lua 5.4-work1 re-implemented a generational GC. Since everything is still in flux, we won't go into the detail.


## The timing of Lua's GC

Lua GC is **not** based on wall-clock. It's based on your memory allocation: It checks the memory usage everything you allocate a new memory.

## The Pace of a Garbage Collector

A **pace** is the frequency of execution of a garbage collector.

* high pace: low memory overhead, high CPU overhead.
* low pace: high memory oerhead, low CPU overhead.



## References

* [Garbage Collection in Lua](https://www.lua.org/wshop18/Ierusalimschy.pdf)
* [LuaJIT - New Garbage Collector](http://wiki.luajit.org/New-Garbage-Collector)

