---
title: "Ternary operators"
authors: [Sheepolution]
date: 2025-02-24
---

Up until this point, if we wanted to assign a variable based on a condition, we would have to use an if-statement.

```lua
local a = 20
local animal = "whale"
local fruit = "apple"

local thing

if a > 10 then
    thing = animal
else
    thing = fruit
end

print(thing) -- Output: whale
```

A **ternary operator** is a feature in many other programming languages, [like JavaScript](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Conditional_operator). It is a short way to get a value based on a condition, and saves us from using an if-statement.

Officially, <ins>Lua does not have a ternary operator</ins>. But, by using `and` & `or` (see [If-statements#and & or](../lua-basics/if-statements#and-%26-or)) we can <ins>simulate</ins> it.

```lua 5
local a = 20
local animal = "whale"
local fruit = "apple"

local thing = a > 10 and animal or fruit
print(thing) -- Output: whale
```

By doing this, we have achieved the same result as by using the if-statement.

Let's create some simple expressions to really understand what `and` and `or` do.

```lua
print("whale" and "apple" and "blue") -- Output: blue
print("whale" or "apple" or "blue") -- Output: whale
print( "whale" and "apple" or "blue") -- Output: apple
```

In the first expression, we start with the [truthy](../lua-basics/if-statements#truthy-and-falsy) value `"whale"`. Because it is truthy, the `and` will continue to the next value. `"apple"` is also truthy, so again it will continue onto the next `and`. There we have the value `"blue"`. There is not another `and`, so this is the value that this expression ends up with.

In the second expression, we again start with `"whale"`. Because it is truthy, we are done. The `or` is for when the value on the left is falsy, which it's not. So `"whale"` is the value of this expression.

In the third example we again start with the value `"whale"`. Truthy, so we continue with the `and`. Truthy, so we <ins>don't</ins> continue with the `or`. The value of this expression is `"apple"`.

> [!TIP]
> Don't worry if you find this confusing. Play around with these expressions. Move around the `and`s and `or`s, and see what it prints to get a good understanding of they work.

If you can keep up, let's use some parentheses to combine these expressions. Can you figure out the output of the code below?

```lua
local a = 20
local animal = "whale"
local fruit = "apple"

local thing = (a > 10 and 5 or 15) > 10 and animal or fruit
print(thing) -- Output: ???
```

> [!WARNING]
> We say we <ins>simulate</ins> ternary operators because there is a limitation.
> ```lua
> local a = 20
> local animal = nil
> local fruit = "apple"
> local thing = a > 10 and animal or fruit
> print(thing) -- Output: apple
> ```
> We want the value to be the animal. If there is no animal, it should print `nil`. But because `nil` is falsy, the `and` is not satisfied. The expression continues with the `or`. This leaves us with `"apple"`. This is a common pitfall to be aware of, and a reason why an if-statement might sometimes be the better option after all.
