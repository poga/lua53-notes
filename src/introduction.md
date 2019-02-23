# Introduction

*Notes on the Implementation of Lua 5.3* is a collection of my notes on the Lua 5.3 source code. It's a mix of both high-level ideas and interesting details in the source code.

There can be errors. Feel free to [contact me](mailto:poga.po@gmail.com) if you have any question or feedback.

## What is Lua?

From [lua.org](https://lua.org):

> Lua is a powerful, efficient, lightweight, embeddable scripting language. It supports procedural programming, object-oriented programming, functional programming, data-driven programming, and data description.
>
> Lua combines simple procedural syntax with powerful data description constructs based on associative arrays and extensible semantics. Lua is dynamically typed, runs by interpreting bytecode with a register-based virtual machine, and has automatic memory management with incremental garbage collection, making it ideal for configuration, scripting, and rapid prototyping.
>
> Lua has been used in [many industrial applications](https://en.wikipedia.org/wiki/Category:Lua-scriptable_software) (e.g., [Adobe's Photoshop Lightroom](http://since1968.com/article/190/mark-hamburg-interview-adobe-photoshop-lightroom-part-2-of-2) ), with an emphasis on embedded systems (e.g., the [Ginga](http://www.ginga.org.br/) middleware for digital TV in Brazil) and [games](https://en.wikipedia.org/wiki/Category:Lua-scripted_video_games) (e.g., [World of Warcraft](http://www.wowwiki.com/Lua) and Angry Birds).

## How to read this book

**Reading code is personal**. It's different from reading a book. It's trying to understand the design of a system. It's non-linear, tearing down, cross-referencing, experimenting, put it back together, and repeat.

This book is not a comprehensive, line-by-line explanation. This book is a tour guide. It's a companion, instead of a replacement, of your journey.

## License

*Notes on the Implementation of Lua 5.3* is licensed under [CC BY-NC-SA 2.0](https://creativecommons.org/licenses/by-nc-sa/2.0/).
