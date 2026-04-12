---
title: "Nested for-loops"
authors: [Sheepolution]
date: 2025-02-24
---

You have already learned that you can [nest statements](../lua-basics/if-statements#nested-statements). So of course, we can also nest for-loops.

```lua
for i = 1, 10 do
    for j = 1, 10 do
        print(i, j)
    end
end
-- Output:
-- 1, 1
-- 1, 2
-- ...
-- 1, 9
-- 1, 10
-- 2, 1
-- 2, 2
-- etc.
```

> [!NOTE]
> We use two different variables, `i` and `j`, so that they don't conflict with each other.

We have also learned that we can use `break` to end a for-loop early. Note that this only breaks the loop you use it in.

```lua
for i = 1, 10 do -- This loop continues.
    for j = 1, 10 do -- This loop breaks.
        print(i, j) -- Because we break, j does not go higher than 1.
        break
    end
end
-- Output:
-- 1, 1
-- 2, 1
-- 3, 1
-- etc.
```

If you want to break both loops at the same time, we could use a <ins>boolean</ins> and an <ins>if-statement</ins>.

```lua
local break_all

for i = 1, 10 do
    for j = 1, 10 do
        print(i, j)
        break_all = true
        break
    end

    if break_all then
        break
    end
end
-- Output: 1, 1
```

Or we could put the for-loop inside a function, and use a [return-statement](../lua-basics/functions#return).

```lua
local function counting()
    for i = 1, 10 do
        for j = 1, 10 do
            print(i, j)
            return -- Leave the function
        end
    end
end

counting()
--Output: 1, 1
```

## Goto-statements

A more efficient method is to use a **goto-statement**. This enables us to instruct Lua to jump to a specific line.

```lua
local a = 50

::retry::

if a > 100 then
    print("a is higher than 100!") -- Printed!
else
    print("Let's try that again!") -- Printed!
    a = 200
    goto retry
end
```

We create a **label** named `retry`, which we can do by surrounding a word with double colons. In the `else`, after changed the value of `a`, we tell Lua to go to that label. After doing so, it tries the if-statement again. This time the condition is `true`.

> [!WARNING]
> Be careful! Similar to [while-loops](../lua-basics/tables#while-loops), it is possible to create an infinite loop, which will cause LÖVE to hang.

We can use a <ins>goto</ins> statement to jump out of a nested for-loop.

```lua
for i = 1, 10 do
    for j = 1, 10 do
        print(i, j)
        goto break_all
    end
end

::break_all::
-- Output: 1, 1
```

A more common way to use goto-statements is by making them act as a <ins>continue</ins>. In many other programming languages, like [JavaScript](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/continue), a `continue` is a way to tell the program to end the current iteration of a for-loop. By placing a label at the bottom of our loop, we can do something similar.

```lua
for i = 1, 10 do
    if i % 2 then
        goto continue
    end

    print("Odd!") -- Printed 5 times

    if i % 3 then
        goto continue
    end

    print(i) -- Output: 1, 5, 7

    ::continue::
end
```

This is a very short and simple example, but with long for-loops this saves us from having lots of nested if-else statements.

> [!WARNING]
> Goto-statements are not available in [love.js](https://github.com/Davidobot/love.js), the unofficial LÖVE port for web.