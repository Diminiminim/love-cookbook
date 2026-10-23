---
title: "Threads"
authors: [clemapfel]
date: 2025-07-14
---

In this chapter, we'll learn how to use **Threads**, which allow for running separate Lua programs at the same time. This is not a guide on parallel programming in general, but it does introduce concepts relevant to LÖVE specifically for readers who may be unfamiliar with the topic.

Because Lua does not support threading in any way, LÖVE implements internally. This was necessary, as threading is vital to modern high-performance game engines. Because of their bespoke nature, LÖVE threads come with heavy restrictions when compared to languages such as [C++](<https://cppreference.com/>) or [Julia](<https://docs.julialang.org/en/v1/>). In return, using (and learning to use) LÖVE threads is much simpler than it would be in that language. This allows users with very little or no experience in parallel programming to still achieve relevant performance gains in their games. This chapter is aimed at that audience.

#### In this chapter we will learn:

+ What a thread is
+ Nomenclature around threads such as *concurrency*, *asynchronicity*, *main*, *worker*, *channel*, and *synchronization*
+ Why concurrency is non-deterministic
+ How to create a thread
+ How to transmit data between threads
+ How to implement a fully functioning thread pool
+ How to safely close a thread

---

## Table of Contents

- [1.0 What is a Thread?](#10-what-is-a-thread)
- [1.1 Glossary, Creating a Thread](#11-glossary-creating-a-thread)
- [1.2 Scheduling](#12-scheduling)
- [1.3 Sharing Memory](#13-sharing-memory)
- [2.0 Channels and Synchronizations](#20-channels-and-synchronizations)
    - [2.0.1 Creating an Unnamed Channel](#201-creating-an-unnamed-channel)
    - [2.0.2 Creating a Named Channel](#202-creating-a-named-channel)
- [2.1 Transmitting Data](#21-transmitting-data)
- [2.2 Allowed Data Types](#22-allowed-data-types)
- [2.3 Common Use Cases for Thread](#23-common-use-cases-for-thread)
- [3.0 A Practical Example](#30-a-practical-example)
- [4.0 Addendum](#40-addendum)
    - [4.1 Using a Channel as a Mutex](#41-using-a-channel-as-a-mutex)
    - [4.2 Sharing FFI Memory](#42-sharing-ffi-memory)

---

# 1.0 What is a Thread?

Every computer has a **CPU**, which is the physical chip on the motherboard that does all the computation. Modern CPUs have multiple **cores**. We can think of cores as their own separate tiny CPUs, even though they have access to the same memory, called **RAM**, and run right next to each other. Multi-core CPUs exhibit **[true parallelism](<https://en.wikipedia.org/wiki/Parallel_computing>)**, which means that two computations can happen on the same CPU, on different cores, at literally the same instant in time. Programs running on multiple cores at once are called **[non-concurrenct](<https://en.wikipedia.org/wiki/Concurrency_(computer_science)>)**. This is in opposition to **[concurrency](<https://en.wikipedia.org/wiki/Preemption_(computing)>)**, which is what normal Lua code (and Luas native `coroutines`) are. We need LÖVE if we want true parallelism, we cannot achieve it with just Lua.

# 1.1 Glossary, Creating a Thread

From this point onward, when we say "thread", we mean a truly *asynchronous* (non-concurrent) thread object, created via `love.Thread`. When we say "routine", we mean a concurrently running program, which Lua's `coroutine` falls into. For this chapter, not all "routines" are Lua coroutines, a regular Lua script will also be called a routine.

+ Threads are asynchronous, routines are synchronous
+ Threads run non-concurrently, routines run concurrently
+ Threads exhibit parallelism, routines do not

> [!NOTE]
> Confusingly, Lua itself calls it's coroutines "threads". If we do `print(type(coroutine.create())`, Lua will print the string "thread". Despite this, **coroutines are not true threads**.

With the definitions out of the way, let's create a thread along with a routine, and just see what happens:

```lua
-- create a thread, its code is a multi-line string
local thread = love.thread.newThread([[
    local args = ...
    print("Thread says: ", ...);
    print("\n")
]])

-- start the  thread
thread:start("hello")

-- create a routine
local routine = function(...)
    local args = ...
    print("Routine says: ", ...);
    print("\n")
end

-- start the routine
routine("begone")
```
```
Thread says: Routine says: 		begonehello
```

Before we get to this very weird output, let's first do a code walk. First, we created a `love.Thread` using its constructor `love.thread.newThread`, which either takes a filename (such as `common/player/player_thread.lua`), or a multi-line string as the first argument. The **thread is idle when created**, we have to start it manually using `start`. This member function can optionally take a number of arguments that are then forwarded to the thread, accessible via `...` inside the threads code, as shown. We provided the string `"hello"` as the argument for `start`. 

For comparison, we also create a Lua function - which is a routine by definition. Then, to "start" the routine, we simply call the function `routine("begone")`, providing `"begone"` as the argument. 

The output above is not a typo or malformatted. It is the the actual console output that the above program *can* exhibit. Why is that? And why "can", why does it not always print the same thing?

When we run a Lua program, it is ran in what we will call the "main" thread, or **main**. Any other thread we will call a **worker**. Note that both main and all workers are threads. All programs, including LÖVE, run in the main thread by default. In this situation, there is only a single thread, main, and therefore no concurrency is exhibited. Any program not using threads still runs in one thread: main.

 Once we have a thread in additiona to main, they will run concurrently with each other, meaning the CPU is capable of performing operations as instructed by main and worker at the same time. However, within the same theads, execution is synchronous.

With this in mind, let's try to understand why we see the above output. We started a worker, `thread`. From this point onwards, this worker will run through it's code. Right after starting worker from main, we continue executing code in main. the CPU is running through both `routine` in main and `thread` in the worker **concurrently**. While main is printing `Routine says: "begone"`, `thread` is printing `Thread says: hello`. Both objects push letters to the operating systems console, resulting in the The two `print`s output to become interleaved.

If we run the above program again, most of the time it will behave as expected, printing:

```
True Thread says: hello
Fake Thread says: begone
```

But sometimes also

```
True Thread says: hello
Fake Thread says: begone
```

Why can the order be different? Let's run another experiment:

## 1.2 Scheduling

```lua
local code = [[
    local n, name = ...
    for i = 1, n do
        io.write(name) -- write to stdout without a newline
    end
]]

local threadA = love.thread.newThread(code)
local threadB = love.thread.newThread(code)

-- start all threads
local n_calls = 20
threadA:start(n_calls, "A") -- print 20 As
threadB:start(n_calls, "B") -- print 20 Bs
```

Here we create two workes, one that prints `A` twenty times, and one that prints `B` twenty times. We start both as closely together in time as we can from main. After wards, main has nothing to do but LÖVE keeps running. Next to main, both workers start printing their letters.

Running the above program four times, it prints the following, where each run is a separate invocation of the above Lua script:

```
AAAAAAAAAAAAAAAAAAAABBBBBBBBBBBBBBBBBBBB
```
```
AAAAAAAAAAAAAAABABAABBBABABBBBBBBBBBBBBB
```
```
ABABABABBAAABBABBABBBABBABABABAAAAAAABBB
```
```
BBBBBBBBBBBBBBBBBBBBAAAAAAAAAAAAAAAAAAAA
```

Clearly both threads exhibit concurrency and the output is interleaved as before, but why is the order different every time we run the program?

While the CPU *can* run things at exactly the same instance in time, it does not always do so. Which core gets which operations is up to a lot of fancy technology, including the **CPU [scheduler](<https://en.wikipedia.org/wiki/Scheduling_(computing)>)**. This is a component of the chipset firmware which basically does the following task: when given a list of instructions, choose which instruction should run on which core, at what time, in what order. On multi-core CPUs, which core is chosen for which instruction is entirely **non-deterministic**. We, as programmers, cannot choose and have no way to influence which core which worker runs on. This is one of the main challenges with parallel computing, as it can make debugging a nightmare. If our program exhibits a bug 1% of the time we run it depending on the scheduler order, then there is no way to reproduce a certain [order of operations](<https://en.wikipedia.org/wiki/Out-of-order_execution>) to trigger that 1% bug every time. We need to rely on our skill, experience, and diligent testing to verify a program will run correctly 100% of the time - even in parallel.

## 1.3 Sharing Memory

So far, we let LÖVE compile a Lua program from a string. So far, the workers routines were completely self-contained. They may have been printing to the same console, but they weren't interacting with each other in any way. What happens when we *share* code between main and a worker? Let's create two new files `module.lua`, and `worker.lua`:

> [!TIP]
> From this point onwards, snippets are all **separate files**. The filename is printed in larger font above each snippet. If we want to run the following code ourselves for example, we need to create three separate new files `module.lua`, `worker.lua`, and `main.lua`, then paste the snippets' content into their respective Lua script.

##### `module.lua`
```lua
-- create a global module
if Module == nil then Module = {} end
Module.initialized = false

-- add an initialize function
function Module.initialize()
    -- print notice to the console
    print("main: initialized module\n")
  
    -- set the module-wide, global `initialized`
    Module.initialized = true
end
```
##### `worker.lua`
```lua
-- check the globla Module.initialize
if not Module.initialized then
    -- if not initialized, do it from `worker.lua`
    Module.initialize()
  
    -- print notice that worker did the initialization
    print("worker: initialized module\n")
else
    -- otherwise print that worker did not need to call `initialize`
    print("worker: module already initialized\n")
end
```
##### `main.lua`
```lua
require "module" -- load "module.lua"
assert(Module ~= nil) -- passes, module is now a global

-- create a thread, loads `worker.lua`
local thread = love.thread.newThread("worker.lua")
thread:start() -- start the thread
```

Here, we create a global table `Module` that is supposed to be shared among all files. We then create a single worker using a file instead of a string this time, then start it. The intended behavior is for the worker to initialize the module in main.

What will this print?

```
Error in Thread error (Thread: 0x0198dc22eef0)

worker.lua:1: attempt to index global 'Module' (a nil value)
stack traceback:
	worker.lua:1: in main chunk
```

We actually get an error. The code in the worker throws because `Module` is undefined. This is because of one of the major limitations of threads in LÖVE: threads **cannot share memory with each other**. More specifically, any object available in a worker's scope will not be available in any other thread, including main. 

We may think that the simple fix is to manually include the module for each thread:

##### `worker.lua`
```lua
require "module" -- include the module in worker scope
if not Module.initialized then
    Module.initialize()
    print("worker: initialized module\n")
else
    print("worker: module already initialized\n")
end
```
##### `main.lua`
```lua
-- create the worker
local thread = love.thread.newThread("worker.lua")

-- start the worker
thread:start()

-- wait for the worker to finish
while true do
    if not thread:isRunning() then
        break
    else
        love.timer.sleep(1 / 60) -- sleep main for 1/60s
    end 
end

assert(thread:isRunning() == false) -- passes, thread is done

require "module" -- include the module in main scope
if not Module.initialized then
    Module.initialize()
    print("main: initialized module\n")
else
    print("main: module already initialized\n")
end
```

Where we used `thread:isRunning` to check if the thread is currently mid-execution, and `love.timer.sleep` to wait one frame before checking again.

>[!CAUTION]
> `love.timer.sleep` causes main to do nothing for one frame, then check again if the thread is running every loop iteration. This technique is called a [*busy waiting*](<https://en.wikipedia.org/wiki/Busy_waiting>) and **is not recommended in any situation**. It pauses main completely, making it unable to do anything but wait. cause the game to freeze for that duration. If don't `sleep` at all, main will spike the CPU core usage to 100%, as it checks as often as possible, doing as many while loop iterations as it can. We will learn a much better solution later in this chapter, but for now this is a crude way to "wait" for a thread to be done.

Now that we have included the module manually in the worker, will the worker be able to access the module? It prints:

```
main: initialized module
worker: initialized module
```

We see that this time the strings are not interleaved, so our crude synchronization worked. However, we see that both the worker and main encountered an uninitialized module. This means the global variable `Module` *exists twice*, entirely separately in both main's and the worker's scope. **Threads do not share memory, they do not share globals**. They don't even share `require`d modules. Any two threads, be it two workers, or a worker and main, are completely separate environments.

*Nothing* is shared, not even LÖVE modules:

##### `worker.lua`
```lua
print("thread will sleep")
love.timer.sleep(love.math.random()) -- sleep between 0 and 1 seconds
print("thread woke back up")
```
##### `main.lua`
```lua
local thread = love.thread.newThread("worker.lua")
thread:start()

-- wait for the thread to finish
while true do
    if not thread:isRunning() then
        break
    else
        love.timer.sleep(1 / 60)
    end
end
```

Running this, it throws

```
thread will sleep
Error in Thread error (Thread: 0x0249f182f3e0)

worker.lua:2: attempt to index field 'timer' (a nil value)
stack traceback:
```

`love.timer` is not available in `worker.lua`. For any thread other than main, we need to manually include the LÖVE modules like this:

```lua
require "love.timer" -- manually include timer
require "love.math"  -- manually inlcude math

print("thread will sleep")
love.timer.sleep(love.math.random())
print("thread woke back up")
```
```
thread will sleep
thread woke back up
```

Any module other than `love.thread` itself must be included manually in worker scope.

> [!WARNING]
> Not all LÖVE modules are available in workers. Most of `love.graphics` and `love.window` is unable to be used at all. This **includes creating textures, drawing to canvases, or to the window**. Any kind of drawing, graphics-state, or window manipulation from within a worker is impossible in LÖVE 12.0 and all earlier versions.

---

## 2.0 Channels and Synchronizations

Since threads have completely separate Lua environments, if they truly have no way to interface with each other, wouuldn't they be close to useless.? Luckily, LÖVE *does* provide a mechanism to at least **send data back and forth between threads**: [`love.Channel`](<https://love2d.org/wiki/Channel>).

To access a channel from main or a worker, we can either share it using the varargs at the top of the thread's file (cf. [Section 1.1](#11-glossary-creating-a-thread))), or we can use `love.thread.getChannel`, which creates a new named channel if it does not yet exist. If does exit, `getChannel` retrieves it without reinitializing.

### 2.0.1 Creating an Unnamed Channel

###### `main.lua`
```lua
-- create two unnamed channels
local main2worker = love.thread.newChannel()
local worker2main = love.thread.newChannel()

-- create a thread
local worker = love.thread.newThread("worker.lua")

-- transmit the channels using `start` varargs
worker:start(main2worker, worker2main)
```
##### `worker.lua`
```lua
local main2worker, worker2main = ... -- get the two channels from `start`

-- use channels after this point
assert(main2worker:typeOf("Channel") and worker2main:typeOf("Channel"))
```

### 2.0.2 Creating a Named Channel

###### `main.lua`
```lua
-- create two named channels
local main2worker = love.thread.getChannel("main2worker")
local worker2main = love.thread.getChannel("worker2main")

-- create a thread
local worker = love.thread.newThread("worker.lua")
worker:start() -- start called without varargs this time
```
##### `worker.lua`
```lua
-- retrieve the channels by name, they already exist
local main2worker = love.thread.getChannel("main2worker")
local worker2main = love.thread.getChannel("worker2main")

-- use channels after this point
assert(main2worker:typeOf("Channel") and worker2main:typeOf("Channel"))
```

## 2.1 Transmitting Data

Now that we have our channels accessible in both main and the worker, we can send data between them using `Channel:push`. We will call these pieces of data a **"message"** for reasons that will become obvious later.

We can retrieve messages using `demand`, which **will sleep the current thread (main or a worker) until a message is found and retrieved**, or `pop`, which **does not sleep** the current thread. Pop returns `nil` if the channel currently has no messages.

In the following code example, we will add a label to each section which will make it easier to discuss:

###### `main.lua`
```lua
-- [MAIN_A] start main, initialize two channels and a worker
local main2worker = love.thread.getChannel("main2worker")
local worker2main = love.thread.getChannel("worker2main")

local worker = love.thread.newThread("worker.lua")
worker:start()

-- [MAIN_B] send a message from main to worker
main2worker:push("do you copy, over")

-- [MAIN_C] wait to receive a message from the worker
local message = worker2main:demand() -- main will halt execution here

-- [MAIN_D] message arrived, use it
assert(message == "roger")

-- [MAIN_E] exit main
```
##### `worker.lua`
```lua
-- [WORKER_A] start worker execution
local main2worker = love.thread.getChannel("main2worker")
local worker2main = love.thread.getChannel("worker2main")

-- [WORKER_B] wait to receive a message from main to worker
local message = main2worker:demand() -- worker will halt execution here

-- [WORKER_C] message arrived, use it
assert(type(message) == "string")

-- [WORKER_D] send a message from worker to main
worker2main:push("roger")

-- [WORKER_E] exit worker
```
We will trace through this carefully, as understanding the expected order of execution here is crucial. First, we notice that we have two **blocking calls**, using `demand`.

A blocking call will completely halt execution of that thread, just like `love.timer.sleep` would. Except instead of a hardcoded duration, the thread will automatically exit sleep the moment a message arrives. 

Because the sleeping part works just like `love.timer.sleep`, if we do `demand` in main for a duration longer than a frame, **our game will completely freeze until a message is retrieved**. If we `demand` inside a worker, that worker will also freeze until a message comes in. However this does not affect main in any way, and thus does not affect our game's framerate. Main and worker are completely separate, if worker sleeps, main does not need to aswell. If we accidentally `demand` in main with no message coming in, our game could completely lock up. The user's OS will event prompt them to kill the process, as it thinks our game has just crashed or frozen. For reasons like this, working with threads can be dangerous - even in LÖVE. `demand`ing in main should be done with the utmost care, almost always main should  `pop`, checking every frame if a message has arrived, as opposed to sleeping forever until one does. `pop` returns immediately, and if this message i `nil`, we simply continue with the rest of the game logic that frame, trying to `pop` again next frame.

Back to our code example, this is the general program flow:

```
MAIN_A: start main
MAIN_B: send main->worker ("do you copy")
MAIN_C: wait for worker->main ("roger")

WORKER_A: start worker
WORKER_B: wait for main->worker ("do you copy")
WORKER_C: receive main->worker ("do you copy")
WORKER_D: send worker->main ("roger")
WORKED_E: exit

MAIN_D: receive worker->main ("roger")
MAIN_E: exit
```

While this is how we conceptualize the above program, remember that **we cannot guarantee order between threads**. The above *could* run in the following order:

```
MAIN_A: start main
WORKER_A: start worker
WORKER_B: wait for main->worker message
MAIN_B: send main->worker
MAIN_C: wait for worker->main
WORKER_C: receives main->worker
WORKER_D: send worker->main
MAIN_D: receives worker->main
WORKED_E: exits
MAIN_E: exits
```

Or any other permutation, as long as `MAIN_A` comes before `MAIN_B`, and `MAIN_C`, and, separately, `WORKER_A` comes before `WORKER_B`, `WORKER_C` etc. Inside the same threads, execution is non-concurrent.

While this is just like our `AABABAB` example before, with this blocking setup utilizing `demand`, we can actually guarantee one thing in terms of order. The worker will wait for a message to come in before exiting, and main will wait for a message to come back before exiting. This pattern is well-suited and idiomatic **synchronization**: we made sure both main and worker at least read each others messages and used them (`assert` above). Both of these are guaranteed to happen before main exits. `Channel` is the only method for synchronization in LÖVE. It is powerful enough to create some semi-sophisticated systems, as we will see in section 3.

## 2.2 Allowed Data Types

So far we only send strings between threads. One huge limitation on `Channel`s is that the type of data we are allowed to send between threads is limited in the following way:

Data can only be transmitted if it is a `string`, `boolean`, `number`, a `love.Object`, `nil`, or a `table` that only contain `strings`, `boolean`s, `number`s or `love.Object`s. This means we **cannot send functions**, and we **cannot send tables that contain functions** between threads. This rules out most OOP-based Lua objects which may contain functions as members. Other examples of types of data we cannot send are `userdata`, `coroutine`s, and LuaJIT ffi-allocated objects, such as the pointer returned by `ffi.new`. 

>[!WARNING]
>Even though we can technically send graphics objects such as `love.Canvas` and `love.Canvas`, we **cannot use them in any way inside a worker**. Any `love.graphics` related object is off-limits, including but not limited to `love.Image`, `love.Graphics`, `love.SpriteBatch`, `love.Particlesystem`, `love.Mesh`, and `love.GraphicsBuffer`. If we try to use these objects, LÖVE may crash or error.

To get a better feeling of which objects are allowed under the above ruless, we will go through some examples:

```lua
-- create a channel for data transmission
local channel = love.thread.newChannel()

channel:push("str") -- `string`: ALLOWED
channel:push(1234) -- `number`: ALLOWED
channel:push({}) -- `table`: ALLOWED
channel:push(love.math.newRandomNumberGenerator()) -- `love.Object`: ALLOWED
channel:push(nil) -- `nil`: ALLOWED

channel:push(function() return 1234 end) -- DISALLOWED: `function`

channel:push({
    hash = 0x0231,
    native = love.math.newByteData(2048),
    slots = {
        { true },
        { false },
        { true }
    }
})-- table with only `number`, `love.Object`, `boolean`, `table`: ALLOWED

-- create an OOP-style object
local object = {}
object.name = "foo"
function object:initialize()  
    -- ...
end
channel:push(object) -- DISALLOWED: table contains `function`
```

Hopefully this elucidates these rules. In summary, message can be any of the plain data types, or a table that only contains plain data types, or tables that only contain plain data types, and so on, recursively.

Let's go through some edge cases that may not be immediately obvious:

```lua
-- create a `love.Thread`
local thread = love.thread.newThread("worker.lua")

-- create two `love.Channel`
local main2worker = love.thread.newChannel()
local worker2main = love,thread.newChannel()

-- push all inside a table:
main2worker:push({
    thread, 
    main2worker, 
    worker2main
}) -- ALLOWED
```

This is allowed, as the table only contains love objects, even if those objects are the thread or channel itself.

Another one:

##### `main.lua`
```lua
-- create an OOP-style type that has a function
local Type = {}
Type.name = "Type"
function Type:initialize()
    -- ...
end

-- create an empty table, it does not have any functions
local object = {}

-- set the objects metatable to the type
setmetatable({
    name = "Object"
}, {
    __index = Type
})

object:initialize() -- works now

-- print object's type from main
print("main says type is ", getmetatable(object).__index)

-- push it to a channel
love.thread.getChannel("main2worker"):push(object)
```

This is the typical paradigm used by Lua OOP-style object system. The `__index` metamethod is set to the type table. Is this allowed to be a message?

##### `worker.lua`
```lua
-- ALLOWED: object can be send and received
local object = love.thread.getChannel("main2worker"):demand()

-- print object's from worker
print("worker says type is ", getmetatable(object).__index.name)
```
```
Error in Thread error (Thread: 0x01c928a3be70)

worker.lua:2: attempt to index a nil value
```

It is allowed to be send, but we do get a worker-side error in the last line when trying to access the objects metatable we realize `push` stripped it, so `initialize` is `nil`, causing the above error. **Metatables are stripped upon sending**. 

> [!CAUTION]
> Any `love.Object` is sent **by-reference**, so both the main and worker have access to the same C++-side object (LÖVE uses C++ internally). Any other kind of data is sent **by-value**, meaning it will be **deep-copied** automatically. We can only reference the same object in memory in both main and any worker if it was specifically created using a LÖVE function, such as `love.data.newImageData`.

## 2.3 Common Use Cases for `Thread`

Not allowing functions, metatables, not sharing any scope with main, and entire modules like `love.graphics` and `love.window` being unavaiable to use imposes heavy limitations on what situations threads are actually useful for. What are they good for then? Valid and recommended usage cases of threads include the following:

+ **isolating a process inside a worker**, making it impossible to crash main. For example reading a user-supplied file, or requesting a resource using `http`
+ **running a process at a higher refresh rate than main**. For example at 240Hz (240 times per second), instead of usual vsync'd 60Hz (60 times per second, at 60fps). This is commonly required for things like music playback. Workers can run at any refresh rate they want.
+ **distributing a large number of small tasks to multpile workers [in parallel](<https://en.wikipedia.org/wiki/Embarrassingly_parallel>)**

This latter usecase can give huge performance gains if implemented well. For example, when loading a large number of image or sound files, we can distribute them among 8 threads. This distribution happens by sending a message to any worker currently not doing anything else. This message contains information on file to load. Any free worker can pick up that message, load that file, then send the loaded `SoundData`, `ByteData`, or `ImageData` back to main. We remember that  `love.graphics.newImage` and can **not** be called in a worker to load a texture. 
In main, after having send the message to request the data, we check every frame if any of the requested data has arrived yet. We do not block and sleep until data is received, we simply check every frame, going "is it there yet?" using `channel:pop`. If this call returns nil, no data is ready yet. 
 Since we distribute the tasks of loading a large number of separate data among 8 threads, the time until that data is done loading and main has received it is about 8 times less than if we loaded all that data in main. If the user has a big CPU with 16 cores, it could be up to 16 times faster! Usually very high thread counts `>8` come with deminishing returns, and are not recommend. We can check the actual number of cores a users CPU has using `love.system.getProcessorCount`.

Even if it is not exactly 8 times faster, it will stil be a many-fold increase in performance. This is the power of threads, doing a large number of small tasks very fast.

The "sending data to any available thread" architecture will be implemented as the conclusion to this chapter. The message passing system using `Channel` `pop` and `demand` is perfectly suited to implement what is called a [thread pool](<https://en.wikipedia.org/wiki/Thread_pool>). This is the object that achieves "send a small task to any non-busy thread" automatically, meaning all the programmer has to do is simply request tasks, then check if they are done periodically.

## 3.0 A Practical Example

Our goal is the following: we want to create `N` threads and start them. We then send a number of requests to these threads using `Channel` messages. Any thread that is currently not doing a task should pick up the message, start the task, load the data (though it can be any task unrelated to `love.graphics`), then send a response message back to main. This response contains the loaded data. Main will then check which data has arrived each frame, after which it can be treated just as one would using LÖVE fully synchronously.

Since messages are crucial to our architecture, we need to decide on a number of **message types**. This list of types needs to be known to both main and all workers. 

All messages will have the following form

```lua
local message = {
    type = MessageType.EXAMPLE_MESSAGE,
    -- additional data here
}
```

We declare the following message types, where the comments document what the "additional data" is for a message with that `type`. For example

```lua
-- worker -> main: error occurred during loading
ERROR = "ERROR" --[[
    type : MessageType,
    id : Number
    error : String
]]
```

Means this message will be sent from a worker to main, and it always has the following members:

```lua
channel:push({
    type = MesageType.ERROR, -- "ERROR"
    id = 32,
    error = "Example Error Message"
})
```

##### `message_type.lua`
```lua
--- @enum MessageTypes
local MessageType = {
    -- main -> worker: request loading of data
    REQUEST = "REQUEST" --[[
        type : MessageType
        path : String
    ]],
    
    -- worker -> main: data loading done, deliver data
    ANSWER = "ANSWER" --[[
        type : MessageType,
        id : Number -- unique thread id
        path : String
        data : love.SoundData
    ]],
    
    -- worker -> main: error occurred during loading
    ERROR = "ERROR" --[[
        type : MessageType,
        id : Number
        error : String
    ]],
    
    -- main -> worker: request threadpool shut down
    SHUTDOWN = "SHUTDOWN" --[[
        type : MessageType
    ]],
    
    -- worker -> main: shutdown finished
    SHUTDOWN_RESPONSE = "SHUTDOWN_RESPONSE" --[[
        type : MessageType
        id : Number
    ]]
}

return MessageType
```

With the message formats settled, following is a full implementation of a `ThreadPool`. This code is quite complex, readers to go through it multiple times over to study it. If that is an unattractive option, we can simply just copy-paste this thread pool into any project. We can customize it by adding custom message types and message handlers, allowing this architecture to be used in any project.

#### `thread_pool.lua`
```lua
local MessageType = require "message_type" -- import message type enum from `message_type.lua`
local ThreadPool = {}

--- @brief create a new thread pool
--- @param count number number of threads, optional
function ThreadPool.new(count)
    if count == nil then
        -- if not specified, use as many threads as the user has cores
        count = math.max(1, love.system.getProcessorCount() - 1 )
    end

    assert(type(count) == "number", "In ThreadPool.new: for argument #1: number expected")

    local self = {}

    -- list of threads
    self.threads = {} -- Table<love.Thread>

    -- shared channel: main -> worker
    self.main2worker = love.thread.newChannel()

    -- shared channel: worker -> main
    self.worker2main = love.thread.newChannel()

    -- storage for data, will be filled when worker responds
    self.data = {} -- Table<Path, ImageData>

    -- create the threads
    for i = 1, count do
        local thread = love.thread.newThread("thread_pool_worker.lua")
        table.insert(self.threads, thread)
    end

    -- start the threads, they will idle until a job is received
    for id, thread in ipairs(self.threads) do
        thread:start(
            self.main2worker,
            self.worker2main,
            id -- unique thread id
        ) -- forward channels to threads
    end

    return setmetatable(self, {
        __index = ThreadPool
    }) -- classic OOP idiom
end

--- @brief request data to be loaded
--- @param path string full path to data file
function ThreadPool:request(path)
    assert(type(path) == "string", "In ThreadPool.request: for argument #1: string expected")

    self.main2worker:push({
        type = MessageType.REQUEST,
        path = path,
        datatype = "SoundData"
    })
end

--- @brief
--- @param path string full path to data file
--- @param clear boolean whether the threadpool should release the reference to the data
--- @return love.Data? may be nil if data is not yet read
function ThreadPool:get(path, clear)
    assert(type(path) == "string", "In ThreadPool.get: for argument #1: string expected")
    local data = self.data[path] -- may be nil

    -- if data is ready and `clear` is set, release reference to the data
    if data ~= nil and clear == true then
        self.data[path] = nil
    end
  
    return data -- return data, or nil if not ready
end

--- @brief work through all received messages this frame
function ThreadPool:update()
    -- while at least one message in is channel, switch on message type
    while self.worker2main:getCount() > 0 do
        -- get the current message
        local message = self.worker2main:pop() -- `pop`, **not** `demand`, do not stall main
        if type(message) == "table" then
            if message.type == MessageType.ANSWER then
                -- if answer request, log data to be received with `get`
                self.data[message.path] = message.data
            elseif message.type == MessageType.ERROR then
                -- if worker errors, forward error to main and rethrow
                print("In ThreadPool.update: thread #" .. message.id .. " errored with " .. message.error .. "\n")
            elseif message.type == MessageType.SHUTDOWN_RESPONSE then
                -- noop
            else
                -- else, malformed message.type or wrong message type for worker -> main
                error("In ThreadPool.update: unhandled message type " .. tostring(message.type))
            end
        else
            -- else, message was not a table
            error("In ThreadPool.update: unhandled message: message is not a table")
        end
    end
end

--- @brief request irreversible shutdown of all workers
function ThreadPool:shutdown()
    -- push n shutdown messages, one for each thread
    for _ in ipairs(self.threads) do
        self.main2worker:push({
            type = MessageType.SHUTDOWN
        })
    end
end

return ThreadPool
```

##### `thread_pool_worker.lua`

```lua
-- include shared message types
local MessageType = require "message_type"

-- include necessary love modules
require "love.sound"

-- retrieve channels from thread:start
local main2worker, worker2main, id = ...

-- shutdown mode, once enabled, return if no messages left
local shutdownActive = false

-- infinitely wait for messages until shutdown is received
while not shutdownActive do
    -- get current message
    local message = main2worker:demand() -- `demand`, not `pop`. worker should stall
  
    -- switch on message type
    if type(message) == "table" then
        if message.type == MessageType.REQUEST then
            -- data request, load data and send back
            local success, error_or_data = pcall(love.sound.newSoundData, message.path)
            if success then
                -- load succeeded: send data
                worker2main:push({
                    type = MessageType.ANSWER,
                    id = id,
                    data = error_or_data,
                    path = message.path
                })
            else
                -- load failed: send error
                worker2main:push({
                    type = MessageType.ERROR,
                    id = id,
                    error = error_or_data
                })
            end
        elseif message.type == MessageType.SHUTDOWN then
             -- enter shutdown mode
             shutdownActive = true
        else
            -- unhandled message type, respond with error
            worker2main:push({
                type = MessageType.ERROR,
                id = id,
                error = "Unhandled message type: " .. tostring(message.type)
            })
        end
    else
        -- malformed message, respond with error
        worker2main:push({
            type = MessageType.ERROR,
            id = id,
            error = "Message is not a table"
        })
    end
end

-- thread is exiting, send shutdown message
worker2main:push({
    type = MessageType.SHUTDOWN_RESPONSE,
    id = id
})

return -- exit the worker
```

Where `pcall` is used to safely call `love.sound.newSoundData`, which could error. It is important to make sure no function in a worker can ever actually error and halt the workers execution.

Let's try our thread pool out:

##### `main.lua`
```lua
local ThreadPool = require "thread_pool" -- `thread_pool.lua`

-- instance thread pool
local pool = ThreadPool.new()

-- path to data if loaded, false otherwise
local paths = {
    ["path/to/resource.mp3"] = false,
    ["path/to/other.mp3"] = false,
    -- ...
}

-- request all data, thread pool will slowly work through them
for path, _ in pairs(paths) do
    pool:request(path)
end

love.update = function(dt)
    pool:update() -- handle all received messages this frame

    for path, ready in pairs(paths) do
      -- if not yet loaded
      if ready == false then
            -- check if loading done
            local data = pool:get(path)
            if data ~= nil then
                -- if data is ready, this is sound data
                paths[path] = data
            else
                -- else it is not yet ready
            end
        end
    end
end
```

We see that this a fully functional thread pool that can crunch through thousands of audio files in seconds. Even though it is only about 150 lines. It is easy to extend this thread pool by adding more message types, or by changing hat exact behavior is triggered by `"REQUEST"` in `thread_pool_worker.lua`.

Asynchronous programming is an incredibly deep topic, and we have only touched the surface. Because of it's complexity and proneness to errors, it is one of the most skillful fields in programming. Hopefully this chapter gave use enough knowledge to at least use threads in LÖVE specifically. We can properly learn the entire field by studying threads in another language at a later time.

---

### 4.0 Addendum

> [!CAUTION]
> The following is for users already familiar with asynchronous programming *outside of LÖVE*. It will use terms that will not be introduced, they are assumed to be known by any read after this point

### 4.1 Using a Channel as a Mutex

We can use a `love.Channel` as a [mutex](https://en.wikipedia.org/wiki/Lock_(computer_science)) (also called a "lock" in certain languages) using `performAtomic`. While `performAtomic` is active, no object can call `performAtomic` on the same channel across all threads. Instead, if `performAtomic` on that channel is active, all other threads will block and wait for it to be free, similar to  `demand`. This means, anything inside the function provided to `performAtomic` is truly synchronous:

```lua
-- decalre atomic routine
local sharedObject = love.image.newImageData( -- ...
local callback = function(shared) 
    -- do atomic synchronized operations here
    -- all other threads waiting on this channel will block
    shared:setPixel(
        4, 2, -- pixel xy 
        1, 0, 1, 1 -- rgba
    )
    -- we need to do this, as without it, race conditions will corrupt the image data
end

-- use channel like a RAII-style mutex:
local channel = love.thread.getChannel("imagelock")
channel:performAtomic(callback, sharedObject)
```

### 4.2 Sharing FFI Memory

Threads can share non-lua allocated memory, meaning memory allocated using `FFI` or (a C binding). **Doing this is highly unsafe**. Race conditions apply, any failure to synchronize by using `performAtomic` or a similar technique will result in RAM corruption, a hard crash, or the game fully seizing up. Despite this, this technique can be extremely powerful and performant. Only applications where a large amount of data that would be impractical to deep-copy through `Channel` is required should consider sharing FFI data.

We cannot send a `cdata` object directly. Instead, we **send over a pointer to the FFI data**. We do this like so:

#### `main.lua`
```lua
-- allocate shared memory
local shared = ffi.new("uint8_t[?]", 1024)

-- cast point to integer, then send it as a number
main2worker:send(tonumber(ffi.cast("uint32_t", shared)))

-- main can modify memory (race conditions apply, may crash)
main2worker:performAtomic(function()
    shared[12] = 0x4
end)
```
#### `worker.lua`
```lua
-- retrieve integer, cast back to shared memory pointer
local shared = ffi.cast("uint8_t[?]", main2worker:demand())

-- worker can modify memory (race conditions apply, may crash)
main2worker:performAtomic(function()
    shared[12] = 0x5
end)
```

Where we cast the array pointer to a plain number so we can send it using channels, then cast the plain number back to a pointer to reconstruct it it worker-side. Because the pointer points to the exact same address in RAM, both worker and main can modify it now.

The same can also be achieved with LÖVE 12.0's `ByteData` interface, where `ByteData` is a `love.Object` and can thus be passed by-reference between threads using `Channel`: 

```lua
-- allocate and send data using LuaJIT FFI:
local shared = ffi.new("uint8_t[?]", 1024) -- allocate 1024 bytes
shared[12] = 0x4 -- takes integer index, not byte offset
channel:push(shared) -- will error

-- allocate and send data using `love.ByteData`:
local shared = love.data.newBytedata(ffi.sizeof("uint8_t") * 1024)
shared:setUInt8(8 * 12, 0x4) -- `set*` takes byte offset, not index
channel:push(shared) -- will work
```

Where `ByteData`s new `set*` methods have no synchronization, race conditions apply and a mutex should be used when modifying them. We can also retreive a regular FFI pointer from the love `ByteData` instance using `getFFIPointer`.





