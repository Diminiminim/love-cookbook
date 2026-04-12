---
title: "Scope"
authors: [Sheepolution]
date: 2025-02-24
---

We have been using [local variables](../lua-basics/variables#local-variables), and explained that they are only available within their **scope**. Let's expand on what that exactly means.

```lua
local one = 1

if true then -- Code block starts
    local two = 2
    print(one, two) -- Output: 1, 2
end -- Code block ends

print(one, two) -- 1, nil
```

The local variable `one` is created outside of a codeblock, and therefore available in the entire file. The local variable `two` is created in an if-statement. In this if-statement, we can access both `one` and `two`. After the if-statement ends, `two` no longer exists. By <ins>indenting</ins> our code, we automatically visualize the scope of local variables.

As shown in the example below, you can create a local variable with <ins>the same name</ins> in a deeper scope level, without affecting the variable in the level above.

```lua
-- main.lua
a = 10 -- A global variable
require "example"
print(a) -- Output: 10
-- The global variable is unaffected by the local variable
```

```lua
-- example.lua
print(a) -- Output: 10
local a = 20 -- A local variable

function my_function(a) -- A parameter
    if true then
        local a = 40 -- Another local variable
        print(a) -- Output: 40
    end

    print(a) -- Output: 30
    -- The parameter is unaffected by the local variable
end

some_function(30)

print(a) -- Output: 20
-- The local variable is unaffected by the parameter
```

## _G

All global variables are actually part of the table `_G`.

```lua
animal = "whale"
print(_G.animal) -- Output: whale

_G.fruit = "apple"
print(fruit) -- Output: apple
```

This means that we can use `_G` to check out all the global variables.

```lua
animal = "whale"
fruit = "apple"

for k, v in pairs(_G) do
    print(k, v)
end
-- Output:
-- animal, whale
-- fruit, apple
```

In a [later chapter](metatables#strict-mode) we will learn how we can modify `_G` to prevent the creation of global variables.