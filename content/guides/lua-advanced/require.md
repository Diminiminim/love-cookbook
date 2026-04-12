---
title: "Require"
authors: [Sheepolution]
date: 2025-02-24
---

So far we have only worked with a single file, `main.lua`. We can use more files by using `require`.

```lua
-- main.lua
require("arithmetics")

print(add(7, 5)) -- Output: 12
print(subtract(7, 5)) -- Output: 2
```

```lua
-- arithmetics.lua
function add(a, b)
    return a + b
end

function subtract(a, b)
    return a - b
end
```

We call `require("arithmetics")`, which executes the file `arithmetics.lua`. Note that we don't need to add `.lua` in our require call.

Because the file has been executed, the <ins>global</ins> functions `add` and `subtract` have been created, which we can then use in our `main.lua`.

> [!TIP]
> When passing a single argument, and that single argument is a string or a table (not a variable but a [literal](https://en.wikipedia.org/wiki/Literal_(computer_programming))), then it is optional to add parentheses to the function call.
>
> This means we can write require like this:
> ```lua
> require "arithmetics"
> ```

We can put our Lua files in folders. File paths commonly use a slash (`/`), but when requiring a file, we use a dot.

```lua
-- main.lua
require "helpers.arithmetics"
```

If we want the functions to be local, we can assign the functions to a local object, and then return that object for the require.

```lua
-- main.lua
local arithmetics = require "arithmetics"

print(arithmetics.add(7, 5)) -- Output: 12
print(arithmetics.subtract(7, 5)) -- Output: 2
```

```lua
-- arithmetics.lua
local arithmetics = {}

function arithmetics.add(a, b)
    return a + b
end

function arithmetics.subtract(a, b)
    return a - b
end

return arithmetics
```

In `arithmetics.lua` we create a local table. At the end of the file, we return that table. So far we have only seen `return` in [functions](functions#return). By using it outside of a function, we can use it to decide what value we return upon the file being required.

In `main.lua` we catch that return value, the table, in a variable also named `arithmetics`. As always, the variable name is not important, but it's nice to be consistent. Having assigned the return value to a variable, we can use its functions.
