---
title: "Standard library"
authors: [Sheepolution]
date: 2025-02-25
---

The **standard library** is what we call the built-in functions that Lua provides. We have already used a few of these functions, like `print`, `require`, and `table.insert`.

The list of [all the functions](https://www.lua.org/manual/5.1/manual.html#2.8) of the standard library is quite long, so we definitely won't be covering all of the standard library. In this chapter we will focus on what is most useful.

## Basic functions

These are standalone functions that are not part of a library. As a counterexample, `table` is a library with functions like `table.insert` and `table.remove`.

### tostring & tonumber

We can use `tostring(value)` to change a value into a string, and we can use `tonumber(string)` to change a string into a number, if possible.

```lua
print(tostring(123) .. tostring(true)) -- Output: 123true
print(tonumber("4") + tonumber("2")) -- Output: 6
```

### error & assert

With `error` we can create our own error. This is called *throwing an error*.

```lua
function sum(a, b)
    if type(a) ~= "number" then
        error("'a' must be a number")
    end

    if type(b) ~= "number" then
        error("'b' must be a number")
    end

    return a + b
end

sum(4, "whale") -- Error: 'b' must be a number
```

With `assert` we can combine `error` with an <ins>if-statement</ins>. We *assert* that something is [truthy](../lua-basics/if-statements#truthy-and-falsy), and if not we throw an error.

```lua
function sum(a, b)
    assert(type(a) == "number", "'a' must be a number")
    assert(type(b) == "number", "'b' must be a number")

    return a + b
end

sum(4, "whale") -- Error: 'b' must be a number
```

### loadstring

With `loadstring` we can turn a string into a function.

```lua
local f = loadstring("animal = 'whale'")
```

The above is equivalent to:

```lua
local f = function () animal = 'whale' end
```

So for example:

```lua
local f = loadstring("animal = 'whale'")
f()
print(animal) -- Output: whale
-- animal is now a global variable.

f = loadstring("return 5")
print(f()) -- Output: 5
```

> [!WARNING]
> Never use `loadstring` with external information, like a save file. If someone shares their save file, modified with malicious code, it could do harmful things to the user's PC.

## table

We have already talked about the **table library** ([manual](https://www.lua.org/manual/5.1/manual.html#5.5)), showing the functions `table.insert` and `table.remove`, but there are two more functions worthy of mention.

### concat

Remember how combining a string using `..` is called concatenation? We can use `table.concat(t, sep)` to concatenate a table into a single string. We can use the parameter `sep` to decide what should *separate* the strings.

```lua
local fruits = { "apple", "pear", "tomato" }

print(table.concat(fruits)) -- Output: applepeartomato

print(table.concat(fruits, "---")) -- Output: apple---pear---tomato
```

### sort

We can use `table.sort` to sort our table.

```lua
local numbers = { 38, 72, 12, 45 }
table.sort(numbers)
print(unpack(numbers)) -- Output: 12, 38, 45, 72

local animals = { "whale", "bear", "tiger" }
table.sort(animals)
print(unpack(animals)) -- Output: bear, tiger, whale
```

But what if we want to sort it the other way? We can provide our own function.

```lua
local numbers = { 38, 72, 12, 45 }
table.sort(numbers, function (a, b)
    return a > b
end)
print(unpack(numbers)) -- Output: 72, 45, 38, 12

local animals = { "whale", "bear", "tiger" }
table.sort(animals, function (a, b)
    return a > b
end)
print(unpack(animals)) -- Output: whale, tiger, bear
```

We pass a [callback](../lua-basics/functions#callbacks) to `table.sort`. It uses this callback to compare two values, and does so repeatedly for a number of steps until your table is fully sorted. In our case, we return `true` if `a` is higher than `b`, to get it sorted from high to low.

## math

The **math library** ([manual](https://www.lua.org/manual/5.1/manual.html#5.6)) speaks for itself. You can do math with it. Some examples.

```lua
-- Get the absolute value of a number
print(math.abs(-25)) -- Output: 25

-- Round a number down
print(math.floor(3.8)) -- Output: 3

-- Round a number up
print(math.ceil(3.8)) -- Output: 4

-- Pi (not a function but a property)
print(math.pi) -- Output: 3.14159265359
```

### random

One very useful function is `math.random`, except that we don't recommend you use it. Instead, you should use {% api "love.math.random()" %}. It is an improved version of the standard random function.

```lua
-- A decimal number between 0 and 1
print(love.math.random()) -- Output: 0.3

-- An integer (whole number) between 2 and 8
print(love.math.random(2, 8)) -- Output: 7
```

These outputs might be different for you. They are random after all, so that would mean it does its job. However, if we want our random numbers to be consistent (for whatever reason) we can use {% api "love.math.setRandomSeed(seed)" %}. The `seed` manipulates what random values we get.

```lua
love.math.setRandomSeed(7)
print(love.math.random(1, 100)) -- Output: 19
```

Because we have set the same seed, you also should get the "random" value `19`.

## string

The **string library** ([manual](https://www.lua.org/manual/5.1/manual.html#5.4)) is for, you guessed it, strings. Let's check out some functions that this library has to offer.

```lua
local animal = "Whale"

print(string.lower(animal)) -- Output: whale

print(string.upper(animal)) -- Output: WHALE

print(string.rep(animal, 3)) -- Output: WhaleWhaleWhale

print(string.reverse(animal)) -- Output: elahW

print(string.len(animal)) -- Output: 5
-- Though we can also do
print(#animal) -- Output: 5
```

Note that the variable itself is not modified. These functions return a new value, but the value of `animal` always remains `"Whale"`.

> [!TIP]
> Instead of `string.lower(animal)` we can also do `animal:lower()`. This is because each string holds the functions of the <ins>string library</ins>, and the colon `:` passes the object that uses the method as the first argument.
> ```lua
> local t = { example = function (a, b) end  }
> -- Doing this:
> t:example("hello")
> -- Is the same as this:
> t.example(t, "hello")
> ```

### sub

With `string.sub` we can get part of a string, a **substring**. To make it easier to understand, we'll use a string of numbers.

```lua
local numbers = "123456789"

print(numbers:sub(2))      -- Output: 23456789
print(numbers:sub(2, 4))   -- Output: 234
print(numbers:sub(-2))     -- Output: 89
print(numbers:sub(-4, -2)) -- Output: 678
print(numbers:sub(3, 3))   -- Output: 3
```

1. `:sub(2)` — From the second character till the end of the string.
2. `:sub(2, 4)` — From the second character till the 4th character.
3. `:sub(-2)` — You can use negative numbers to start from the back of the string. `-1` is the last character. In this example we get the second to last character of the string, till the last character.
4. `:sub(-4, -2)` — This is the same as the second example, except starting from the back.
5. `:sub(3, 3)` — We start and end at the third character, giving us only that single character.

### find

With `string.find` we can check if a smaller string is part of a bigger string, and if so it tells us where in the string it is located.

```lua
local fruit = "apple"

print(fruit:find("ppl")) -- Output: 2, 4
print(fruit:find("p")) -- Output: 2, 2
print(fruit:find("t")) -- Output: nil
```

1. `:find("ppl")` — Returns `2, 4`. This is because the string `"ppl"` starts at the second character of `"apple"`, and ends at the 4th character.
2. `:find("p")` — Returns `2, 2`. It stops at the first `"p"` it can find, which starts (and ends) on position `2`.
3. `fruit:find("t")` — Returns `nil`. Because there is no character `"t"` in `"apple"`

### gsub

`string.gsub` allows us to replace characters in a string. We can do this in a very simple way.

```lua
local animal = "whale"
print(animal:gsub("wha", "app")) -- Output: apple, 1

local message = "Hello world how are you?"
print(message:gsub("o", "a")) -- Output: Hella warld haw are yau?, 4
```

> [!NOTE]
> The `1` and `4` tell us how many instances have been replaced.

But to unlock the full potential of `string.gsub, you will have to learn about [string patterns](string-patterns).