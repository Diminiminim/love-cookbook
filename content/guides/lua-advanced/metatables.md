---
title: "Metatables"
authors: [Sheepolution]
date: 2025-02-24
---

With **metatables** we can enhance our tables, making them a very powerful tool. It works by creating a second table that we assign as a <ins>metatable</ins> to our main table. The metatable has **metamethods** which are special functions that influence how our table works.

> [!NOTE]
> "Meta" means referring to itself. It's called a <ins>meta</ins>table because it is influencing the table itself.
>
> A **method** is what we call a <ins>function</ins> assigned to an <ins>object</ins>.

```lua
local my_table = {}
local my_metatable = {}
my_metatable.__call = function ()
    print("Hello world!")
end

setmetatable(my_table, my_metatable)

my_table() -- Output: Hello world!
```

We create a table `my_table`, and a second table `my_metatable`. We give the table the property `__call`. This is a <ins>metamethod</ins>. Next we use the function `setmetatable(table, metatable)`. The first argument is our table, the second argument the metatable we want to apply to it.

On the last line, we can finally see the power of metatables. We can <ins>call</ins> our table, as if it's a function. When we do, it triggers the event `call`, and calls the <ins>metamethod</ins> we assigned to `__call`.

> [!IMPORTANT]
> We have often stated that the name of a variable does not matter. That is different in this case. The keys for the metamethods are predefined. [See the list of all the events.](https://www.lua.org/manual/5.1/manual.html#2.8)

We won't be looking at all the events, but there are two important ones that deserve our attention. `index` and `newindex`.

## newindex

Let's say we want to prevent extra properties to be assigned to our table. With a `__newindex` metamethod we can make this happen.

```lua
local my_table = {}
local my_metatable = {}

my_metatable.__newindex = function (t, key, value)
    error("No new assignments allowed!")
end

my_table.animal = "whale"
setmetatable(my_table, my_metatable)
my_table.fruit = "apple" -- Error!
```

We first assign the property `animal`. All is fine, because we haven't set the metatable yet. But after setting the metatable, we get an error when assigning the property `fruit`. The `newindex` event has been triggered, the metamethod called, and the error thrown.

The first parameter of `__newindex` is the table. Why would we need the table? Don't we already know what table we try to assign a property to? Well no, because the same metatable can be set to multiple tables, meaning we won't know which of those tables has triggered the event to call the metamethod.

If we actually want the assignment to happen, we can't do `t[key] = value`. This would trigger the same event again, causing an infinite loop. Instead, we need to use `rawset(t, key, value)`. This is a special function that sets the value without triggering the event.

But what's the point of metamethods if we do a normal assignment? Let's have some fun with it, and create a table that magically doubles all the numbers we assign to it.

```lua
local my_table = {}
local my_metatable = {}

-- It doesn't matter whether we set the metatable
-- before or after assigning the metamethods.
setmetatable(my_table, my_metatable)

my_metatable.__newindex = function (t, key, value)
    rawset(t, key, value * 2)
end

my_table[1] = 5
my_table[2] = 17
my_table[3] = 80

for i, v in ipairs(my_table) do
    print(v) -- Output: 10, 34, 160
end

my_table[1] = 3
print(my_table[1]) -- Output: 3
```

As you can see, our numbers are being doubled. But something weird is going on there at the end. We assign the number `3`, and it stays `3`. This is because `__newindex` only works for <ins>new</ins> indexes. Because `my_table[1]` already had a value, it doesn't trigger the event anymore.

## index

The event `index` will trigger when trying to access a property that the table doesn't have. With this, we can create special properties that would normally require us to use a function. Well, we do use a function, but we don't need to call that function ourselves. We only need to access the non-existing property.

In the example below we have a triangle. We set the length of `a` and `b`, and let the metamethod calculate the length of `c`.

```lua
local my_triangle = {
    a = 3,
    b = 4
}

local my_metatable = {
    -- As with normal properties, we can assign a metamethod like this.
    __index = function(t, key)
        if key ~= "c" then
            error("Unknown property " .. key)
        end

        local a, b = rawget(t, "a"), rawget(t, "b")

        return math.sqrt(a ^ 2 + b ^ 2)
    end
}

setmetatable(my_triangle, my_metatable)

print(my_triangle.c) -- Output: 5
my_triangle.a = 10
print(my_triangle.c) -- Output: 10.77
my_triangle.b = 20
print(my_triangle.c) -- Output: 22.36
```

Upon trying to access `c`, the metamethod returns us a number calculated with the [Pythagorean theorem](https://en.wikipedia.org/wiki/Pythagorean_theorem). By changing `a` and `b`, we automatically change the value of `c`.

In the metamethod, we use `rawget(t, key)`. Similar to `rawset`, it is to get a value without triggering the event. In this case we wouldn't need it, since `my_triangle` definitely has the properties `a` and `b`, but it's always good to be safe.

## Strict mode

In [JavaScript](https://developer.mozilla.org/en-US/docs/Web/JavaScript) we can enable [strict mode](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Strict_mode). This prevents the creation of global variables, among other things.

Using the power of metatables, we can create our own strict mode. We do this by setting a metatable to [`_G`](scope#_g), where the metamethods for `newindex` and `index` throw an error.

```lua
animal = "whale"

setmetatable(_G, {
  __newindex = function(t, k, v)
      error("Cannot set undefined variable: " .. k, 2)
  end,
  __index = function(t, k)
    error("Cannot get undefined variable: " .. k, 2)
  end
})

print(animal) -- Output: whale
print(onimal) -- Error: Cannot get undefined variable: onimal
fruit = "apple" -- Error: Cannot set undefined variable: fruit
```

As you can see, this also helps us with finding mistyped variable names.