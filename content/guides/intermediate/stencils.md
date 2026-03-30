---
title: "Stencils"
authors: [clemapfel]
date: 2025-02-19
---

# Stencils

> [!CAUTION]
> This guide is made for LÖVE 12.0! These stencil functions are not available in prior versions.

This chapter will go over **stencils**, which open up a number of advanced drawing techniques which would be very hard to achieve otherwise.

# 0. TL;DR: Quick Start

For those without enough time to read this entire chapter, or people returning for reference purposes, below are example usages of the stencil-related functions in LÖVE 12.

**It is expected that first-time readers will not understand what is going on in these examples**, as the rest of this chapter will build the knowledge needed to fully understand them.

## 0.1 `setStencilMode` Example

```lua
-- choose a stencil value
local stencilValue = 123

-- start writing to the stencil buffer
love.graphics.setStencilMode("draw", stencilValue)

-- set every pixel touched by this command to `123`
love.graphics.circle("fill",
  0.5 * love.graphics.getWidth(),  -- x position of the circle
  0.5 * love.graphics.getHeight(), -- y position
  0.25 * love.graphics.getHeight() -- radius
)

-- enable stencil testing
love.graphics.setStencilMode("test", stencilValue)

-- draw a rectangle with stencil testing enabled
local margin = 10
love.graphics.rectangle("fill",
  margin, -- x position
  margin, -- y position
  love.graphics.getWidth() - 2 * margin, -- width
  love.graphics.getHeight() - 2 * margin -- height
)

-- disable stencil testing
love.graphics.setStencilMode("off")
```

## 0.2 `setStencilState` Example

```lua
-- choose a stencil value
local stencilValue = 123

-- make it so we are now drawing to the stencil buffer
love.graphics.setStencilState("replace", "always", stencilValue)
love.graphics.setColorMask(false, false, false, false)

-- set every pixel touched by this command to `123`
love.graphics.circle("fill",
  0.5 * love.graphics.getWidth(),  -- x position of the circle
  0.5 * love.graphics.getHeight(), -- y position
  0.25 * love.graphics.getHeight() -- radius
)

-- make it so we are now testing against the stencil buffer
love.graphics.setStencilState("keep", "notequal", stencilValue)
love.graphics.setColorMask(true, true, true, true)

-- draw a rectangle with stencil testing enabled
local margin = 10
love.graphics.rectangle("fill",
  margin, -- x position
  margin, -- y position
  love.graphics.getWidth() - 2 * margin, -- width
  love.graphics.getHeight() - 2 * margin -- height
)

-- disable stencil testing
love.graphics.setStencilState("keep", "always", nil)
love.graphics.setColorMask(true, true, true, true)
```

## 0.3 Clearing the Stencil Buffer

```lua
love.graphics.clear(
  false, -- do not clear color
  true,  -- clear stencil buffer
  false  -- do not clear depth buffer
)
```

## 0.4 Binding a Canvas with Stencils Active

```lua
-- create a canvas
local canvas = love.graphics.newTexture(100, 100, {
  canvas = true
})

-- bind the canvas with stenciling enabled
love.graphics.setCanvas({
  canvas,
  stencil = true
})

-- draw here, stencils will work as expected

-- unbind the canvas
love.graphics.setCanvas(nil)
```

For those still with us, let's begin our exploration of stencils.

## 1. Stencil Buffer

The main window (or currently bound canvas) will have what is called a **stencil buffer**. This is a special kind of texture that holds, on the graphics card, an unsigned integer value in `[0, 255)` for every pixel. Writing to the stencil buffer works the same as with the window or any canvas; when a draw command such as `love.graphics.draw` or `love.graphics.circle` is called, the object drawn will modify every pixel it touches. For normal textures and the main window, this modifies the RGBA values and thus the color of that pixel, for this reason it is also sometimes called a **color buffer**. **Stencil buffers** work the same way: we draw using a regular draw command, and any pixel touched will have a new value written to it. Instead of a color, it is the above-mentioned unsigned integer.

## 2. Stencil Compare Modes & Culling

The main use for stencils is controlling whether or not a draw command (such as `love.graphics.draw` or `love.graphics.circle`) will **actually modify the current window or canvas**. Internally, LÖVE keeps a global value which is called the **stencil compare mode**. When a pixel is drawn, LÖVE will compare the value of the stencil buffer at that position using this compare mode, and if this operation returns true, the pixel will be drawn to. If it returns false, **the pixel will not be modified** - the window or canvas will **not** have its color at that position changed. This is why stencils are very powerful: much like a real-life stencil, they allow us to cut out certain parts of a drawable object, preventing only that part from being drawn, without the use of any additional geometry.

In LÖVE 12, there are two kinds of stencil-related API functions. `setStencilMode` is the more user-friendly option; it is great for simple operations. For more advanced applications, the much more complex `setStencilState` is available.

## 3. `setStencilMode`

For now, we'll focus on `setStencilMode`. This function has the following signature:

```lua
--- @param mode love.graphics.StencilMode mode
--- @param value number stencil value
--- @return nil
love.graphics.setStencilMode(mode, v)
```

`mode` is of type `love.graphics.StencilMode`, which is an enum with the following values:

+ `"off"` - disable stenciling
+ `"draw"` - enable drawing to the stencil buffer
+ `"test"` - enable testing against the stencil buffer

If no mode is specified, the argument will default to `"off"`.

`v` is an unsigned integer in `[0, 255)`. If a float is given, it will be rounded down to the nearest integer. We need to make sure ourselves to **not specify values outside this range**, as setting the stencil value to, for example, `2045.23` will not cause an error - what exact value will be drawn to the stencil buffer in this case is deterministic but logically opaque and should thus be avoided.

### 3.1 Usage

Usage of `setStencilMode` will often follow this pattern:

```lua
-- (1) choose a stencil value in [0, 255]
local stencilValue = 123

-- (2) enable drawing to the stencil buffer
love.graphics.setStencilMode("draw", stencilValue)

-- any draw code here, e.g. love.graphics.draw, love.graphics.circle, etc.

-- (3) enable testing against the stencil buffer
love.graphics.setStencilMode("test", stencilValue)

-- more draw code here

-- (4) disable stencil testing
love.graphics.setStencilMode("off")
```

Let's go through each of these lines step by step. Since we have no way to visualize the stencil buffer, it is vital to understand exactly how drawing works in this context.

#### 3.1.1 Choosing a Stencil Value (1)

First we need to choose what value subsequent modifications to the stencil buffer will write. Here we arbitrarily chose `123`, which is an integer in the valid range `[0, 255]`. While we could simply not choose a stencil value and use the default `0`, once we start using stencils more frequently, **different usages will affect each other**, since they all modify the same buffer and the buffer state is maintained within the same frame unless manually cleared. Because of this, **each section should ideally have its own stencil value**, as this minimizes side effects.

#### 3.1.2 Enabling Stencil Draw (2)

With `setStencilMode("draw")`, we are telling LÖVE that from now on, all draws will only affect the stencil buffer. This also means they **will no longer modify the current canvas or window**. Any draw command will only touch the stencil buffer, meaning they will be invisible when viewing the window. For every subsequent draw, until we change the stencil mode, we are now **writing the value `stencilValue` to every pixel touched by our draw commands**. Note that "touching" a pixel means any kind of modification - for example, when drawing a texture, every pixel touched by the texture will be overwritten, regardless of the texture's alpha value or blend mode. For this reason, it's best to stick to drawing simple shapes using `love.graphics`, or using meshes.

#### 3.1.3 Enabling Stencil Testing (3)

Now that we have modified the stencil buffer, we can tell LÖVE to enter the **compare phase**, which we mentioned earlier. From this point onwards, LÖVE will perform the following calculation: for a pixel at `(x, y)`, get the current stencil value at `(x, y)` from the stencil buffer. **If this value equals `stencilValue`, perform the draw as usual; otherwise, discard it**. This is true for every pixel on the screen, not just the ones we wrote to earlier.

#### 3.1.4 Disabling Stencil Testing (4)

After drawing with stencil testing enabled is done, we need to manually disable stencil testing. To do this, we simply call `love.graphics.setStencilMode("off")`.

### 3.2 A Working Example

```lua
-- set background color
love.graphics.clear(0.5, 0.5, 0.5, 1)

local w, h = love.graphics.getDimensions()
local texture = love.graphics.newTexture("toast.png")
local textureWidth, textureHeight = texture:getDimensions()

-- choose a stencil value
local stencilValue = 123

-- make the entire stencil buffer 0
love.graphics.clear(
  false, -- do not clear color
  true   -- clear stencil
)

-- start writing to the stencil buffer
love.graphics.setStencilMode("draw", stencilValue)

-- set every pixel's stencil value touched by this command to `123`
love.graphics.circle("fill",
  0.5 * w,  -- x position of the circle
  0.5 * h,  -- y position
  0.5 * textureWidth,  -- radius
  5 -- number of outer vertices
)

-- enable stencil testing
love.graphics.setStencilMode("test", stencilValue)

-- draw the texture
love.graphics.draw(texture,
  0.5 * w - 0.5 * texture:getWidth(), -- x position of texture
  0.5 * h - 0.5 * texture:getHeight() -- y position
)

-- disable stencil testing
love.graphics.setStencilMode("off")
```

Here, we set the stencil buffer in a pentagonal region (a region shaped like a 5-sided regular polygon) to `stencilValue` (`123` in this case), then draw a texture at the same position with stencil testing enabled.

First let's draw just the texture. This is what it looks like with no stencils applied:

![](/assets/img/stencil_mode_texture_only.png)

This is what the `love.graphics.circle` call looks like if we render it to the window, again with no stencils applied:

![](/assets/img/stencils/stencil_mode_shape_only.png)

We now apply the stencil as shown above. What will the result look like?

![](/assets/img/stencils/stencil_mode.png)

We see that only the center part of the texture was drawn; parts outside that pentagon were discarded. We only see the pentagonal part because when testing using `setStencilMode`, only pixels for which the stencil buffer has a value **equal** to `stencilValue` will be updated. The stencil buffer was reset to `0`, we updated the value in the pentagonal area to `123`, then told LÖVE to only draw pixels where the stencil buffer is equal to `123`. Therefore, only where the pentagonal shape was drawn will parts of the texture be drawn as well.

This is a powerful result; drawing a texture clipped to a pentagon like this could otherwise only be achieved with a `Mesh` or a `Shader`. If we use geometry more complex than a simple pentagon, it becomes almost infeasible to only draw parts of the texture that overlap that geometry.

### 4. `setStencilState`

While `setStencilMode` is easy to use and somewhat easy to understand, it has the following limitations:

+ i) we can only overwrite the stencil value at a certain position, not perform any other kind of calculation on it
+ ii) when drawing with testing active, only regions for which the stencil value is **equal** to our value will be drawn; no other comparison is supported
+ iii) we can only draw to either the stencil buffer or the window, not both

`setStencilState` lifts all of these restrictions - at the cost of a much more complicated API. It is therefore a more advanced part of LÖVE, which we will try to make accessible in this chapter.

#### 4.1 Signature

`setStencilState` has the following signature:

```lua
--- @param action love.graphics.StencilAction how the stencil buffer will be modified
--- @param compareMode love.graphics.StencilCompareMode when testing, which equation will be used
--- @param value number stencil value
--- @return nil
love.graphics.setStencilState(
  action,
  compareMode,
  value
)
```

Let's also quickly look at a seemingly unrelated function, which will become important soon:

```lua
--- @param red boolean whether to render red component
--- @param green boolean whether to render green component
--- @param blue boolean whether to render blue component
--- @param alpha boolean whether to render alpha component
--- @return nil
love.graphics.setColorMask(
  red,
  green,
  blue,
  alpha
)
```

#### 4.1 Stencil Action

`action`, the first argument of `setStencilState`, takes a value of the enum `love.graphics.StencilAction`. This argument governs **how the stencil buffer will be modified when we perform a draw action**. Remember that **only pixels touched by the draw command will have their corresponding stencil buffer value modified**.

It can have one of the following values, where `current` is the value currently in the stencil buffer at the given position, `v` is the value given as the third argument to `setStencilState`, and `new` is the value the stencil buffer will have after modification.

| `StencilAction` | Behavior | Equation |
|-----------------|----------|----------|
| `"keep"` | do not modify the stencil buffer | `new = current` |
| `"zero"` | replace the stencil buffer value with `0` | `new = 0` |
| `"replace"` | replace the stencil buffer value with `v` | `new = v` |
| `"increment"` | add `v` to the stencil buffer value; if the new value would exceed `255`, clamp it to `255` | `new = min(current + v, 255)` |
| `"decrement"` | subtract `v` from the stencil buffer value; if the new value would be below `0`, clamp it to `0` | `new = max(current - v, 0)` |
| `"incrementwrap"` | add `v` to the stencil buffer value; if the new value would exceed `255`, wrap it back into `[0, 255)` | `new = mod(current + v, 255)`* |
| `"decrementwrap"` | subtract `v` from the stencil buffer value; if the new value would be below `0`, wrap it back into `[0, 255)` | `new = mod(current - v, 255)`* |
| `"invert"` | perform a bitwise `not` operation on the current value | `new = bnot(current)`**|

\* where `mod(a, b)` returns the modulo of `a` with `b`, equivalent to `a % b` in LuaJIT

\*\* where `bnot(a)` returns the bitwise negation of `a`, equivalent to `bit.bnot(a)` in LuaJIT

With these actions, we have a breadth of ways to change how exactly the stencil buffer will be modified. Attentive readers may have noticed that `setStencilMode("draw")` internally sets the stencil action to `"replace"`, since any pixel touched by the draw command will have its value overridden.

### 4.2 `StencilCompareMode`

The second argument of `setStencilState` governs what formula is used **to determine whether to draw or discard a pixel depending on the stencil buffer value** at that position. Recall that for `setStencilMode`, only pixels for which the stencil buffer value was **equal to** the second argument of `setStencilMode` will be drawn. `setStencilState` gives us more options than just equality, where, again, `current` is the value currently in the stencil buffer at the pixel's position and `v` is the third argument given to `setStencilState`:


| `StencilCompareMode` | Behavior | Equation |
|----------------------|----------|----------|
| `"always"` | always drawn | `true` |
| `"never"` | never drawn | `false` |
| `"equal"` | drawn if stencil buffer value equals `v` | `current == v` |
| `"notequal"` | drawn if stencil buffer value does not equal `v` | `current ~= v` |
| `"less"` | drawn if stencil buffer value is strictly less than `v` | `current < v` |
| `"lequal"` | drawn if stencil buffer value is less than or equal to `v` | `current <= v` |
| `"greater"` | drawn if stencil buffer value is strictly greater than `v` | `current > v` |
| `"gequal"` | drawn if stencil buffer value is greater than or equal to `v` | `current >= v` |

### 4.3 `setColorMask`

Lastly, we need a way to control whether we are currently drawing to the screen or the stencil buffer. For this, we use `setColorMask`. This function takes four booleans which control whether the red, green, blue, and alpha components of the pixel will be affected by subsequent draw calls. Since, when drawing to the stencil buffer, all we need is to not update the color buffer of the window at all, we simply call:

```lua
love.graphics.setColorMask(false, false, false, false)
```

Now, if `setStencilState` is set, draw calls will only affect the stencil buffer, not the window. To start drawing to the window again, we simply call:

```lua
love.graphics.setColorMask(true, true, true, true)
-- equivalent to `love.graphics.setColorMask()`
```

After which drawing works as normal.

### 4.4 A Working Example

Let's again use our texture and pentagon from before, except we now want the texture to be drawn everywhere **except where the pentagon is**. This is not possible to achieve with `setStencilMode`, since it is hardcoded to only use the `equal` compare mode. Since we do not want to draw where the stencil buffer value is equal, we choose `notequal` for the stencil compare mode.

```lua
-- set background color
love.graphics.clear(0.5, 0.5, 0.5, 1)

local w, h = love.graphics.getDimensions()
local texture = love.graphics.newTexture("toast.png")
local textureWidth, textureHeight = texture:getDimensions()

-- choose a stencil value
local stencilValue = 123

-- make the entire stencil buffer 0
love.graphics.clear(
  false, -- do not clear color
  true   -- clear stencil
)

-- start writing to the stencil buffer
love.graphics.setStencilState(
  "replace", -- stencil buffer value should be overwritten
  "always",  -- do not test against the buffer for now
  stencilValue
)

-- stop drawing to the main window's color buffer
love.graphics.setColorMask(false, false, false, false)

-- set every pixel's stencil value touched by this command to `123`
love.graphics.circle("fill",
  0.5 * w,  -- x position of the circle
  0.5 * h,  -- y position
  0.5 * textureWidth,  -- radius
  5 -- number of outer vertices
)

-- enable stencil testing
love.graphics.setStencilState(
  "keep",      -- do not modify the stencil buffer
  "notequal",  -- draw only where stencil buffer value is `not equal` to `123`
  stencilValue
)

-- start drawing to the color buffer again
love.graphics.setColorMask(true, true, true, true)

-- draw the texture
love.graphics.draw(texture,
  0.5 * w - 0.5 * texture:getWidth(), -- x position
  0.5 * h - 0.5 * texture:getHeight() -- y position
)

-- disable stencil testing
love.graphics.setStencilState("keep", "always", nil)
love.graphics.setColorMask() -- restore mask state (optional)
```

The texture and pentagon are in the same positions as before. What will this new `setStencilState` result look like?

![](assets/img/stencils/stencil_state.png)

We see that a pentagonal slice has been cut out of our texture. This is expected. We reset the stencil buffer to `0` everywhere using `love.graphics.clear`, then overwrote the pentagonal area of the stencil buffer to `123`. When drawing the texture, we told LÖVE to **only draw where the buffer is not equal to `123`**. Since the buffer is `0` everywhere else, only the pentagonal area of the texture was excluded from being drawn.

This is a powerful result, as producing the above image using any other technique would be exceedingly complicated.

While this shows the most basic usage of `setStencilState`, closing out this chapter of the LÖVE cookbook are some even more advanced techniques.

#### 4.4.1 Drawing to the Stencil Buffer and Window at the Same Time

```lua
love.graphics.push("all") -- "all" to store stencil-related settings

-- start drawing to the stencil buffer
love.graphics.setStencilState(
  "replace", -- overwrite values in the buffer
  "always",  -- disable stencil testing
  stencilValue
)

-- also enable color mask
love.graphics.setColorMask(true, true, true, true)

-- draw calls here will affect both the stencil buffer and the window at the same time

love.graphics.pop() -- restore the previous stencil state
```

#### 4.4.2 Testing Against the Stencil Buffer While Drawing to It

```lua
love.graphics.push("all")

do
  local stencilValue = 123

  -- update the stencil buffer with `123`
  love.graphics.setStencilState(
    "replace", -- overwrite values in the buffer
    "always",  -- disable stencil testing
    stencilValue
  )

  love.graphics.setColorMask(false, false, false, false)
end

-- draw calls here will write `stencilValue` to stencil buffer

do
  local stencilValue = 124

  -- continue updating the stencil buffer, now with `124`
  love.graphics.setStencilState(
    "replace", -- overwrite values in the buffer
    "equal",   -- enable stencil testing
    stencilValue
  )

  -- draws here will write `124` to the 
  -- stencil buffer while simultaneously testing if 
  -- the value is equal to `123`
end

-- restore the graphics state
love.graphics.pop()
```

Where the `do`-`end` blocks' scoping ensure `stencilValue` is different in both `setStencilState` calls.

### 5. Example Game

Below is a full `main.lua` for a game of "shoot the target", with stencils being used to draw the crosshair/spotlight - something that would be very hard to achieve without stencils. Take special note of `love.draw` at the top of the file, which uses `setStencilState` as discussed to stencil out the circle from the full black rectangle covering the screen.

![](/assets/img/stencils/shoot_the_target.png)

```lua
local score = 0
local field = nil
local reinitialize_field, check_success, draw_field_below, draw_field_above, update_field

love.draw = function()
    draw_field_below()
    local x, y = love.mouse.getPosition()

    -- clear the stencil buffer
    love.graphics.clear(false, true, false)

    -- choose a stencil value
    local stencilValue = 123

    -- make it so we are now drawing to the stencil buffer
    love.graphics.setStencilState("replace", "always", stencilValue)
    love.graphics.setColorMask(false, false, false, false)

    -- set circular area in buffer to `stencilValue`
    love.graphics.setColor(1, 1, 1, 1)
    love.graphics.circle("fill", x, y, field.crosshair.radius)

    -- make it so we are now testing against the stencil buffer
    love.graphics.setStencilState("keep", "notequal", stencilValue)
    love.graphics.setColorMask(true, true, true, true)

    -- black the entire screen; the stencil area will be excluded because of `notequal`
    love.graphics.setColor(0, 0, 0, 1)
    love.graphics.rectangle("fill", 0, 0, love.graphics.getDimensions())

    -- reset: neither write to the stencil buffer nor test against it
    love.graphics.setStencilState("keep", "always", nil)
    love.graphics.setColorMask(true, true, true, true)

    draw_field_above()
end

--- ### game logic ### ---

love.update = function(delta)
    update_field(delta)
end

love.mousepressed = function(_)
    check_success()
end

love.keypressed = function(_)
    check_success()
end

love.load = function()
    reinitialize_field()
    love.mouse.setVisible(false)
end

love.resize = function(w, h)
    reinitialize_field()
end

reinitialize_field = function()
    field = {
        circles = {},
        target = {},

        crosshair = {
            x = love.mouse.getX(),
            y = love.mouse.getY(),
            radius = 0,
            top = {},
            bottom = {},
            right = {},
            left = {}
        },

        reveal_elapsed = 0,
        reveal_duration = 1,

        cooldown_elapsed = 0,
        cooldown_duration = 2,

        found_message_x = 0,
        found_message_y = 0,

        title_x = 0,
        title_y = 0,

        score_x = 0,
        score_y = 0,

        font = love.graphics.getFont(),

        elapsed = 0
    }

    local screen_w, screen_h = love.graphics.getDimensions()

    local min_radius = 7
    local max_radius = 21

    local spotlight_radius = 128

    local target_hue = 0.14
    local min_hue = 0.45
    local max_hue = 0.85

    local min_home_radius = 40
    local max_home_radius = 120
    local min_velocity = 0
    local max_velocity = 20

    local cell_size = 64
    local min_per_cell = 1
    local max_per_cell = 7

    local random = function(lower, upper)
        local ratio = love.math.random()
        return lower * (1 - ratio) + upper * ratio
    end

    local function hsva_to_rgba(h, s, v, a)
        h = h * 360

        local c = v * s
        local h_2 = h / 60.0
        local x = c * (1 - math.abs(math.fmod(h_2, 2) - 1))

        local r, g, b

        if (0 <= h_2 and h_2 < 1) then
            r, g, b = c, x, 0
        elseif (1 <= h_2 and h_2 < 2) then
            r, g, b  = x, c, 0
        elseif (2 <= h_2 and h_2 < 3) then
            r, g, b  = 0, c, x
        elseif (3 <= h_2 and h_2 < 4) then
            r, g, b  = 0, x, c
        elseif (4 <= h_2 and h_2 < 5) then
            r, g, b  = x, 0, c
        else
            r, g, b  = c, 0, x
        end

        local m = v - c

        r = r + m
        g = g + m
        b = b + m

        return r, g, b, a
    end

    field.target_text_color = { hsva_to_rgba(target_hue, 1, 1, 1) }

    field.circles = {}

    local new_circle = function(x, y, radius, hue)
        return {
            x = x,
            y = y,
            home_x = x,
            home_y = y,
            radius = radius,
            noise_offset = random(-10e6, 10e6),
            velocity = random(min_velocity, max_velocity),
            home_radius = random(min_home_radius, max_home_radius),
            color = { hsva_to_rgba(hue, 1, 1, 1) }
        }
    end

    local n_rows = math.ceil(screen_w / cell_size)
    local n_columns = math.ceil(screen_h / cell_size)

    for row_i = 1, n_rows do
        for col_i = 1, n_columns do
            local n_per_cell = love.math.random(min_per_cell, max_per_cell)
            for i = 1, n_per_cell do
                local x = random(
                    (row_i - 1) * cell_size,
                    (row_i - 0) * cell_size
                )

                local y = random(
                    (col_i - 1) * cell_size,
                    (col_i - 0) * cell_size
                )

                local radius = random(min_radius, max_radius)
                local hue = random(min_hue, max_hue)

                table.insert(field.circles, new_circle(
                    x, y, radius, hue
                ))
            end
        end
    end

    local target_radius = random(min_radius, max_radius)

    local target_min_x = 0 + 2 * target_radius
    local target_max_x = screen_w - 2 * target_radius

    local target_min_y = 0 + 2 * target_radius
    local target_max_y = screen_h - 2 * target_radius

    local target_x = random(target_min_x, target_max_x)
    local target_y = random(target_min_y, target_max_y)

    field.target = new_circle(
        target_x,
        target_y,
        target_radius,
        target_hue
    )
    field.target.home_radius = max_home_radius

    field.crosshair.radius = spotlight_radius

    field.crosshair.left = {
        0 - field.target.radius, 0,
        0 - field.crosshair.radius, 0
    }

    field.crosshair.right = {
        0 + field.target.radius, 0,
        0 + field.crosshair.radius, 0
    }

    field.crosshair.top = {
        0, 0 - field.target.radius,
        0, 0 - field.crosshair.radius
    }

    field.crosshair.bottom = {
        0, 0 + field.target.radius,
        0, 0 + field.crosshair.radius
    }

    local font = love.graphics.newFont(40)
    local font_height = font:getHeight()

    field.shoot_the_text = "shoot the "
    field.target_text = "target"
    field.score_text = "score: "

    local margin = 20

    local title_width = font:getWidth(field.shoot_the_text) + font:getWidth(field.target_text)
    field.title_x = math.floor((screen_w - title_width) / 2)
    field.title_y = margin
    field.title_width = font:getWidth(field.shoot_the_text)

    field.score_text_x = margin
    field.score_text_y = math.floor(screen_h - font_height - margin)

    field.font = font
end

function draw_field_below()
    if field.cooldown_elapsed > 0 then
        love.graphics.clear(1, 0, 0, 1)
    else
        love.graphics.clear(0.5, 0.5, 0.5, 1)
    end

    love.graphics.setLineWidth(2)
    love.graphics.setLineStyle("smooth")

    local draw_circle = function(entry)
        love.graphics.setColor(0, 0, 0, 1)
        love.graphics.circle("line", entry.x, entry.y, entry.radius + 1)

        love.graphics.setColor(entry.color)
        love.graphics.circle("fill", entry.x, entry.y, entry.radius)
    end

    if field.cooldown_elapsed <= 0 then
        for _, entry in ipairs(field.circles) do
            draw_circle(entry)
        end
    end

    draw_circle(field.target)
end

function draw_field_above()
    local x, y = love.mouse.getPosition()

    local line_width = 5
    love.graphics.push()
    love.graphics.translate(x, y)
    love.graphics.setColor(0, 0, 0, 1)
    love.graphics.setLineWidth(line_width)
    love.graphics.line(field.crosshair.top)
    love.graphics.line(field.crosshair.right)
    love.graphics.line(field.crosshair.bottom)
    love.graphics.line(field.crosshair.left)
    love.graphics.pop()

    love.graphics.setColor(1, 0, 0, 1)
    love.graphics.circle("fill", x, y, line_width)

    love.graphics.setFont(field.font)

    local offsets = {
        { -1, -1 },
        {  0, -1 },
        {  1, -1 },
        { -1,  0 },
        {  1,  0 },
        { -1,  1 },
        {  0,  1 },
        {  1,  1 },
    }

    love.graphics.setColor(0, 0, 0, 1)
    for _, offset in ipairs(offsets) do
        love.graphics.push()
        love.graphics.translate(math.floor(2 * offset[1]), math.floor(2 * offset[2]))
        love.graphics.print(field.shoot_the_text, field.title_x, field.title_y)
        love.graphics.print(field.target_text, field.title_x + field.title_width, field.title_y)
        love.graphics.print(field.score_text .. score, field.score_text_x, field.score_text_y)
        love.graphics.pop()
    end

    love.graphics.setColor(1, 1, 1, 1)
    love.graphics.print(field.shoot_the_text, field.title_x, field.title_y)

    love.graphics.setColor(field.target_text_color)
    love.graphics.print(field.target_text, field.title_x + field.title_width, field.title_y)

    love.graphics.setColor(1, 1, 1, 1)
    love.graphics.print(field.score_text .. score, field.score_text_x, field.score_text_y)
end

function check_success()
    local x, y = love.mouse.getPosition()
    local distance_squared = (x - field.target.x)^2 + (y - field.target.y)^2
    if distance_squared < field.target.radius^2 then
        if field.cooldown_elapsed <= 0 then
            score = score + 1
            field.cooldown_elapsed = field.cooldown_duration
        end
    end
end

function update_field(delta)
    field.elapsed = field.elapsed + delta

    local before = field.cooldown_elapsed
    field.cooldown_elapsed = field.cooldown_elapsed - delta

    if before > 0 and field.cooldown_elapsed <= 0 then
        reinitialize_field()
    end

    if field.cooldown_elapsed >= 0 then return end

    local min_x, min_y = 0, 0
    local max_x, max_y = love.graphics.getWidth(), love.graphics.getHeight()

    local update_circle = function(circle, offset)
        local t = field.elapsed + circle.noise_offset

        local x_noise = (love.math.perlinNoise(t + offset * 0.3 - 100) * 2) - 1
        local y_noise = (love.math.perlinNoise(t + offset * 0.3 + 100) * 2) - 1

        local target_x = circle.home_x + x_noise * circle.home_radius
        local target_y = circle.home_y + y_noise * circle.home_radius

        circle.x = circle.x + (target_x - circle.x) * delta * circle.velocity
        circle.y = circle.y + (target_y - circle.y) * delta * circle.velocity

        circle.x = math.max(circle.x, min_x + circle.radius)
        circle.x = math.min(circle.x, max_x - circle.radius)
        circle.y = math.max(circle.y, min_x + circle.radius)
        circle.y = math.min(circle.y, max_x - circle.radius)
    end

    for i, circle in ipairs(field.circles) do
        update_circle(circle, i)
    end

    update_circle(field.target, 0)
end
```