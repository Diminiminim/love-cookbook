---
title: "Lua"
authors: [ellraiser]
date: 2025-07-30
---


Lua is the programming language that LÖVE uses! Lua (pronounded 'LOO-ah') means "Moon" in Portuguese, and was created back in 1993 by a team at [PUC-Rio](https://www.puc-rio.br/english/) as a language that could extend various software applications. 

The idea was to create a language that gave all the basic functionality needed but not get too complicated or domain-specific, instead allowing developers to extend and customise as needed.

> [!IMPORTANT]
> Lua is a name, <ins>not</ins> an abbreviation. Writing it as *LUA* is therefore <ins>wrong</ins>.

A small piece of example code in Lua:
```lua
-- a comment!

local data = {
  name = "Toast",
  counter = 0
}

function add()
  data.counter = data.counter + 1
end

for i=1,100 do
  add()
end

print('Done!', data.counter)
```


## Learning Lua
To get started with LÖVE, it's recommended to have some basic understanding of Lua first. Here are some resources that can help get you started:

**LÖVE Cookbook**   
You're already reading it! This Lua Basics section has a bunch of sections that will take you through the syntax, variables, functions, datatypes and general gotchas.   
[https://diminim.github.io/love-cookbook/guides/lua-basics/variables/](https://diminim.github.io/love-cookbook/guides/lua-basics/variables/)

**Sheepolution's "How To LÖVE"**   
This is a general guide to using LÖVE, but starts by introducing the concepts of Lua first.   
[https://sheepolution.com/learn/book/contents](https://sheepolution.com/learn/book/contents)

**Programming In Lua**   
This is the official programming guide for Lua, by the creators themselves. It can be a bit heavy in some places, but does showcase everything Lua has to offer (which can be overkill for what you need).  
[https://www.lua.org/pil/1.html](https://www.lua.org/pil/1.html)

**Learn X in Y minutes**   
A 'cheatsheet' type guide showcasing the main syntax and features of Lua and explaining how it works, useful once you've already started using Lua as a quick reference to things.
[https://learnxinyminutes.com/lua/](https://learnxinyminutes.com/lua/)

**Lua Reference Manual**   
This is the manual for the programming language, and so covers all the syntax and functions available. It's a long read, and while it would probably benefit you to read through it all one day, for now it is better to use as a lookup when you want to learn more about something specific, like functions or metatables.   
[https://www.lua.org/manual/5.0/](https://www.lua.org/manual/5.0/)


## LuaJIT
LÖVE uses a version of Lua called [LuaJIT](https://luajit.org/), a 'Just-In-Time' compiler that converts the your written Lua code into machine code at runtime. This allows the code to be executed extremely fast (and Lua is very fast to begin with!).

LuaJIT uses Lua version 5.1, but does implement a couple of 5.2 features, as such keep in mind the Lua version when looking up documentation as there are newer Lua versions and features not available in LuaJIT.
