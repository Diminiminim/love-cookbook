---
title: "String patterns"
authors: [Sheepolution]
date: 2025-02-25
---

With **string patterns** ([manual](https://www.lua.org/manual/5.1/manual.html#5.4.1)) we can search for a string in a more complex way. For example, let's say we're not searching for a specific character, but any uppercase letter. We can do this with <ins>string patterns</ins>

To demonstrate string patterns, we will use `string.match`. This returns the substring that is matching our pattern.

```lua
local fruit = "apPle"
print(fruit:match"[A-Z]") -- Output: P
```

We can use square brackets `[]` to denote a <ins>set</ins>. With `A-Z` we cover all uppercase letters of the alphabet. So the pattern says: Find me a character that is in this set of all uppercase letters.

> [!NOTE]
> String patterns is similar to [regular expression](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Regular_expressions), but the former is more limited. If they would have added regular expression, then Lua's code would have doubled in size.

But we can write it even shorter:

```lua
local fruit = "apPle"
print(fruit:match("%u")) -- Output: P
```

`%u` is what is called a **character class**. As you can see [here](https://www.lua.org/manual/5.1/manual.html#5.4.1) there are a bunch of them. `%u` represents all uppercase letters.

If, for whatever reason, we want to search for the actual string `%u` instead of the character class it represents, we can escape it by placing another `%` in front.

```lua
local fruit = "ap%uple"
print(fruit:match("%%u")) -- Output: %u
```

If we want to search for more than one character, we can use `+`. We can use a dot `.` to represent any character.

In the code below we search for a string that starts with an uppercase letter, is then followed by any characters, and ends with an uppercase letter.

```lua
local message = "Hello WORLD how ARE You?"
print(message:match("%u.+%u")) -- Output: Hello WORLD how ARE Y
```

The `+` is one of four characters with which we can control how many characters we are looking for.

* `+` — One or more characters (greedy)
* `*` — Zero or more characters (greedy)
* `-` — Zero or more characters (non-greedy)
* `?` — Zero or one character

<ins>Greedy</ins> and <ins>non-greedy</ins> refer whether how fast the pattern is satisfied. Notice how our output went all the way to the last `"Y"` of `"You"`. The substring `"Hello W"` satisfies our conditions of starting and ending with an uppercase letter, but the `+` continues the search up till the very last uppercase letter it could find. We get the shorter string if we use a `-` instead.

```lua
local message = "Hello WORLD how ARE You?"
print(message:match("%u.-%u")) -- Output: Hello W
```

## Capturing

But what if we don't care about the uppercase letters? Yes, the text we are looking for is surrounded by uppercase letters, but we only care about the characters in-between. For this, we can use **capturing**.

We can create a capture with parentheses `()`. Let's capture the characters in-between the uppercase letters.

```lua
local message = "Hello WORLD how ARE You?"
print(message:match("%u(.-)%u")) -- Output: ello
```

## gsub

By using the power of <ins>string patterns</ins>, we can use the full potential of `string.gsub`. For example, instead of providing a replacement string, we can provide a <ins>table</ins>.

```lua
local about_me = {
    -- This is how you assign a key with special
    -- characters in the creation of the table.
    -- You surround the string with square brackets.
    ["{animal}"] = "whales",
    ["{fruit}"] = "apples"
}
local message = "I like {animal} and {fruit}."

print(message:gsub("{.-}", about_me)) -- Output: I like whales and apples.
```

By using a table, `gsub` searches for keys that match the substrings it found, and uses the values as the replacement for the string.

> [!NOTE]
> This is another good example of where the difference between `+` and `-` is important. Had we used `+`, it would have matched with `"{animal} and {fruit}"`, as that is the longest string that opens and closes with `{}`.

But what if we don't simply want to replace the string that we find? For example, what if we want to reverse each word individually? We can use `%a+` to search for each word (`%a` represents all letters). Then instead of a replacement string, we're going to use a function, a [callback](../lua-basics/functions#callbacks). `gsub` will feed the function all the matches it could find, and the value we return is what that match will be replaced with.

```lua
local message = "Hello world how are you?"
print(message:gsub("%a+", function(str)
    return str:reverse()
end))
```

`gsub` can also be useful if we want to extract information from a string. Let's say we want to read all the numbers in a string and put them in a table.

```lua
local numbers = {}
local message = "A s2tr7ing w39ith hi6dden nu8mbe1rs!"
message:gsub("%d+", function(n)
    table.insert(numbers, tonumber(n))
end)

print(unpack(numbers)) -- Output: 2, 7, 39, 6, 8, 1
```

We don't return any value, because we don't care about modifying the string (besides, we're not printing the return value, or assigning it to a variable). Though if we wanted to remove the numbers, we could have returned an empty string `""`.