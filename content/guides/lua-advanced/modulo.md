---
title: "Modulo"
authors: [Sheepolution]
date: 2025-02-24
---

We have demonstrated the different type of arithmetic operators (`+`, `-`, `*`, `/` and `^`), but there is one operator we have yet to mention. The **modulo operator**, for which we use the character `%`.

```lua
local a = 17
local b = 5
print(a % c) -- Output: 2
```

Modulo calculates the <ins>remainder</ins>. In the example above, the output is `2`. That is because, if you keep extracting `5` from `17`, `2` is what <ins>remains</ins>.
* `17 - 5 = 12`
* `12 - 5 = 7`
* `7 - 5 = 2` ✔️
* `2 - 5 = -3` ❌

The modulo operator is extremely useful. For example, we can use it to check if a number is <ins>odd</ins> or <ins>even</ins>.

```lua
local function odd_or_even(n)
    if n % 2 == 0 then
        print("Even!")
    else
        print("Odd!")
    end
end

odd_or_even(5) -- Output: Odd!
odd_or_even(200) -- Output: Even!
odd_or_even(486) -- Output: Even!
odd_or_even(7295621) -- Output: Odd!
```

Or we can check which day of the week a certain date is.

```lua
local days = {
    "Monday", "Tuesday", "Wednesday",
    "Thursday", "Friday", "Saturday",
    "Sunday"
}

local function check_day(date)
    print(days[(date - 1) % 7 + 1])
end

check_day(5) -- Output: Friday
check_day(14) -- Output: Sunday
check_day(32) -- Output: Thursday
check_day(38285) -- Output: Tuesday
```

The line `days[(date - 1) % 7 + 1]` includes the `- 1` and `+ 1` because the first element in a Lua table is at index `1`. Without, `check_day(14)` would turn into `14 % 7`, which results into `0`, and there is nothing to be found at `days[0]`.

To prevent this, we use this trick:

1. Decrease date by 1 (`14 - 1`).
2. Now the modulo operator checks `13 % 7`, which results into `6`.
3. We increase the number by 1 (`6 + 1`).

> [!NOTE]
> Lua has what we call <ins>1-based indexing</ins>, meaning that tables start counting from `1`. In many other programming languages, lists (often called <ins>arrays</ins>) start counting from `0`. Because of this, they won't have to use the same trick. It is the reason that programmers often prefer <ins>0-based indexing</ins>. Lua gave preference to a user-friendly design philosophy.

