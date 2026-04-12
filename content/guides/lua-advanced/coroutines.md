---
title: "Coroutines"
authors: [pippo]
date: 2026-03-03
---

## Coroutine Basics

A **coroutine** allows you to stop the execution of a function and then resume the execution where you paused. This can be beneficial in a wide variety of scenarios, such as turn based games or animations.

We can create a coroutine using the function **coroutine.create**, as shown:

```lua
local function myFunction()
  print("hello!")
end

local myCoroutine = coroutine.create(myFunction)
```

We can then start (or resume) the execution of a coroutine using the function **coroutine.resume**.

```lua
local function myFunction()
  print("hello!")
end

local myCoroutine = coroutine.create(myFunction)

coroutine.resume(myCoroutine) -- This will print "hello!".
```

A coroutine can be in one of three states: suspended, running, or dead. When we create a coroutine, it starts in the "suspended" state. We can view the state of a coroutine by using the function **coroutine.status**.

```lua
local function myFunction()
  print("hello!")
end

local myCoroutine = coroutine.create(myFunction)

print(coroutine.status(myCoroutine)) -- This will print "suspended".
```

After we resume our coroutine, it will be in the dead state:

```lua
local function myFunction()
  print("hello!")
end

local myCoroutine = coroutine.create(myFunction)

coroutine.resume(myCoroutine) -- This will print "hello!".

print(coroutine.status(myCoroutine)) -- This will print "dead".
```

## The Real Power

The real benefit of coroutines can be found in the function **coroutine.yield**. Calling the coroutine.yield function from inside a coroutine will cause it to suspend it's execution, which can be resumed later.

```lua
local function myFunction()
  print("i'm first!")
  coroutine.yield()
  print("i'm second!")
end

local myCoroutine = coroutine.create(myFunction)

coroutine.resume(myCoroutine) -- This will print "i'm first!".
coroutine.resume(myCoroutine) -- This will print "i'm second!".
```
If we try to resume a dead coroutine, the coroutine.resume function will return **false**, along with an error message.

```lua
local function myFunction()
  print("i'm first!")
  coroutine.yield()
  print("i'm second!")
end

local myCoroutine = coroutine.create(myFunction)

coroutine.resume(myCoroutine) -- This will print "i'm first!".
coroutine.resume(myCoroutine) -- This will print "i'm second!".

-- This will return false and the string
-- "cannot resume a dead coroutine"
coroutine.resume(myCoroutine)
```
We can pass parameters to the coroutine when it is resumed, as so:

```lua
local function myFunction(someText)
  print(someText)
end

local myCoroutine = coroutine.create(myFunction)

-- This will print "hello from the other side!".
coroutine.resume(myCoroutine, "hello from the other side!")
```
Additionally, when executed successfully, coroutine.resume will return the parameters passed to coroutine.yield.

```lua
local function myFunction()
  print("i'm first!")
  coroutine.yield("hi from the coroutine!")
  print("i'm second!")
end

local myCoroutine = coroutine.create(myFunction)
local yieldedString = coroutine.resume(myCoroutine)

-- This will print "hi from the coroutine!".
print(yieldedString)
```
