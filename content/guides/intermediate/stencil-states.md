---
title: "Stencil states"
authors: [clem]
date: 2025-02-19
---

# Stencils

> [!CAUTION]
> This guide is made for LÖVE 12.0!

This chapter will go over **stencils**, which open up a number of advanced drawing techniques which are very hard to achieve otherwise. 

## 1. Stencil Buffer

The main window (or currently bound canvas) will have what is called a **stencil buffer**. This is a special kind of texture that holds, on the graphics card, an unsigned integer value in `[0, 255)` for every pixel. Writing to the stencil buffer works the same as with the window or any canvas; when a draw command such as `love.graphics.draw` or `love.graphics.circle` is called, the object drawn will modify every pixel it touches. For normal textures, this modifies the rgba values and thus the color of that pixel when the window is drawn. Stencil buffers work the same way, we draw using a regular draw command, and any pixel that object touches will have a new value written to it. Instead of a color, it is the above mention unsigned integer.

We will see soon how to achieve this, first we need to go over what stencils are used for.

## 2. Stencil Compare Modes & Culling

The main use for stencils is controlling whether or not a draw command (such as `love.graphics.draw` or `love.graphics.circle`) will **actually modify current window or canvas**. Internally, love keeps a global value which is called the **stencil compare mode**. When a pixel is drawn, love will compare the value of the stencil buffer at that position using this compare mode, and if this operations returns true, the pixel will be drawn. If it returns false, **no pixel will be draw**, the window or canvas will **not** have it's color at that position changed. This is why stencils are very powerful, much like a real-life stencil, it allows use to cut out certain parts of a drawable object, preventing only that part to be drawn, without the use of any additional geometry.

In love 12, there are two kinds of stencil-related API functions, `setStencilMode` is the more user-friendly option, it is great for simple operations, but is limited in some way. In more advanced applications, we can use `setStencilState`.

## 3. `setStencilMode`

For now, we'll focus on `setStencilMode`. This function has the following signature

```lua
--- @param mode love.graphics.StencilMode mode
--- @param value number stencil value
love.graphics.setStencilMode(mode, value)
```

`mode` is of type `love.graphics.StencilMode`, which is an enum with the following values

+ `off` disable stenciling
+ `draw` enable drawing to the stencil buffer
+ `test` enable testing against the stencil buffer

If no mode is specified, the argument will default to `off`,

`value` is an integer in `[0, 255)`. If a float is given, it will round down to the nearest integer. Make sure to not specify values outside this range, while this does not cause an error, it will be clamped back into `[0, 255]`, which could cause unexpected behavior.

### 3.1 Usage

Usage of `setStencilMode` will often have the following pattern

```lua

-- [1] choose a stencil value in 0, 255
local stencil_value = 123

-- [2] enable drawing to the stencil buffer
love.graphics.setStencilMode("draw", stencil_value)

-- any draw code here, eg. love.graphics.draw(*), love.graphics.circle, etc.

-- [3] enable testing against the stencil buffer
love.graphics.setStencilMode("test", stencil_value)

-- more draw code here

-- [4] disable stencil testing
love.graphics.setStencilMode("off")
```

Let's go through each of these lines step-by-step. Since we have no way to visualize the stencil buffer, it is vital to understand exactly how drawing works in this context.

#### 3.1.1 Choosing a Stencil Value (1)

First we need to choose what value we want subsequent modifcations to the stencil buffer will write. Here we arbitrarily chose `123`,
which is an integer and in the valid range `[0, 255]`. While we could simply not choose a stencil value as use the default `0`, once we start using stencils more frequently, different usages will affect each other, since all modify the same buffer. Because of this, each code section ideally have their own stencil value to minimize side-effects.

##### 3.1.2 Enabling Stencil Draw (2)

With `setStencilMode("draw")`, we are telling love that from now on, all draws will only affect the stencil buffer. This also means they **will no longer modify the current canvas or window**. Any draw command will only touch the stencil buffer, meaning they will be invisible when viewing the window. For every subsequent draw, until we change the stencil mode, we are now **writing the value `stencil_value` to every pixel touched by our draw commands**. Note that "touching" a pixel means any kind of modification, for example drawing a texture, every pixel touched by the texture will be overwritten, regardless of the textures alpha value or blend mode. For this reason, it's best to stick to drawing simple shapes using `love.graphics`, or meshes.

#### 3.1.3 Enabling Stencil Test (3)

Now that we have modified the stencil buffer, we can tell love to go into the **compare phase**, which we mentioned earlier. From this point onwards, love will perform the following calculation: For a pixel at `(x, y)`, get the current stencil value at `(x, y)` from the stencil buffer. **If this value equals `stencil_value`, perform the draw as usual, otherwise `discard` it**. This is true for every pixel on the screen, not just the ones we wrote to earlier. This is why we should choose a specific stencil value, since the stencil buffer will remain modified between frames.

Now that stencil testing is enabled

#### 3.1.4 Disabling Stencil Tests (4)


### 3.2 A working example

```lua
local w, h = love.graphics.getDimensions()
local texture = love.graphics.newTexture("toast.png")

-- choose a stencil value
local stencil_value = 123

-- make the entire stencil buffer 0
love.graphics.clear(
    false, -- do not clear color
    true   -- clear stencil
)

-- start writing to the stencil buffer
love.graphics.setStencilMode("draw", stencil_value)

-- set every pixel touched by this command to `123`
love.graphics.circle("fill",
    0.5 * w,  -- x position of the circle
    0.5 * h,  -- y position
    0.25 * h,  -- radius,
    5 -- number of outer vertices
)

-- enable stencil testing
love.graphics.setStencilMode("test", stencil_value)

-- draw the texture
love.graphics.draw(texture,
    0.5 * w - 0.5 * texture:getWidth(), -- x position of texture
    0.5 * h - 0.5 * texture:getHeight() -- y position
)

-- disable stencil testing
love.graphics.setStencilMode("off")
```

Here, we set the stencil buffer at a pentagonal a region (a region that is shaped like a 5-sided regular polygon) to `stencil_value` (`123` in this case), then draw a texture at the same position with stencil testing enabled. What will the result look like?

TODO: toast

We see that only the center part of the texture was drawn, parts outside that pentagon were discarded. We only see the pentagonal part because when testing using `setStencilMode`, only pixels for whom the stencil buffer has a value **equal** to `stencil_value` will be updated. The stencil buffer was reset to `0`, we updated the pentagonal part to `123`, then told love to only draw pixels where the stencil buffer is `123`. Therefore, only the center part of the texture in that region will be drawn.

This is a powerful result, drawing a texture pentagon like this could otherwise only be achieved with a `love.Mesh` or `Shader`. If we use even complex geometry than just a `love.graphics.circle`, it becomes almost unfeasable to draw only a certain part of the texture. 

### 4. `setStencilState`

While `setStencilMode` is easy to use and somewhat easy to understand, it has the following limitations:

+ (a) we can only overwrite the stencil value at a certain position, not perform any other kind of calculation to it
+ (b) when drawing with testing active, only regions for whom the stencil value is **equal** to our value can be drawn, no other comparison is supported
+ (c) we can only draw to either the stencil buffer, or window, not both

`setStencilState` lifsts these restriction, at the cost of being a much more complicated API. It is therefore a more advanced part of love, which we will try to make accessible in this chapter. Let's go through each of the restrictions, (a), (b), and (c), and learn how `setStencilState` is used to overcome them.

#### 4.1 Signature

`setStencilState` has the following signature

```lua
--- @param action love.graphics.StencilAction how the stencil buffer will be modifier
--- @param compare_move love.graphics.StencilCompareMode when testing, which equation will be used
--- @param value number stencil value
love.graphics.setStencilState(
    action,
    compare_mode,
    value
)
```

Let's also quickly look at a seemingly unrelated function, which will become important soon

```lua
--- @param red boolean whether to render red component
--- @param green boolean whether to render green component
--- @param blue boolean whether to render blue component
--- @param alpha boolean whether to render alpha component
love.graphics.setColorMask(
    red, 
    green, 
    blue, 
    alpha
)
```

#### 4.1 Stencil Action

`action`, the first argument of `setStencilMode`, takes a value of the enum `love.graphics.StencilAction`. This arguments governs **how the stencil buffer will be modified when we perform a draw action**. Remember that only pixels touched by the draw command will have their corresponding stencil buffer value modified. It can have one of the following values, where `current` is the value currently in the stencil buffer at the given position, `value` is the value given as the third argument to `setStencilState`, and `new` is the value the stencil buffer will have after modifiction.


| `StencilAction` | Behavior                                                                                                                                               | Equation                          |
|-----------------|--------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------|
| `"keep"`          | do not modify the stencil buffer                                                                                                                       | `new = current`                   |
| `"zero"`          | replace the stencil buffer value with `0`                                                                                                              | `new = 0`                         |
| `"replace"`       | replace the stencil buffer value with `value`, the 3rd argument of `setStencilState`                                                                   | `new = value`                     |
| `"increment"`     | Add `value` to the stencil buffer value. If the new value would exceed `255`, make it `255`                                                            | `new = min(current + value, 255)` |
| `"decrement"`     | Subtract `value` from the stencil buffer value. If the new value would be below `0`, make it `0`                                                       | `new = max(current - value, 0)`   |
| `"incrementwrap"` | Add `value` to the stencil buffer value. If the new value would exceed `255`, perform a modulo operation on it to wrap it back into `[0, 255)`         | `new = mod(current + value, 255)`* |
| `"decrementwrap"` | Subtract `value `from the stencil buffer value. If the new value would be below `0`, performa a module operation on it to wrap it back into `[0, 255)` | `new = mod(current - value, 255)`* |
| `"invert"`        | Perform a bitwise not operation on the current value                                                                                                   | `new = bnot(current)`****         | 

** where `mod(a, b)` returns the modulo of `a` with `b`, equivalent to `a % b` in lua
*** where `bnot(a)` returns the bitwise negation of `a`, equivalent to `bit.bnot(a)` in lua

With these actions, we have a breadth of ways to change  how exactly the stencil buffer will be modified. Attentive viewers may have realized that `setStencilMode("draw")` internally sets the stencil action to `"replace"`, since any pixel touched by the draw command will have it's value overriden.


### 4.2 `StencilCompareMode`

The second argument of `setStencilState` 










