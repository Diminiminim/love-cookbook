---
title: "Pixelation"
authors: [Jasper]
date: 2025-03-05
---

> [!CAUTION]
> This guide is made for LÖVE 12.0!


For an introduction to shaders, see [Shaders](shaders)

Pixelation is a cool effect that can be used to create a retro look.

To achieve a pixelation effect, we basically want to increase the amount of pixels a pixel in our game covers,

For this effect we'll run a shader to average the results of every 2x2 block, effectively halving the resolution of our game.      
We can use a trick to improve the memory efficiency of our effect by storing the results of that average in a half resolution image, since all pixels in that 2x2 block will have the same average.     
To increase the amount of pixelation our effect allows for we can keep downscaling like that until we achieve our desired resolution.

`pixelationShader.fs`
```glsl
vec4 effect(vec4 color, Image tex, vec2 texture_coords, vec2 screen_coords) {
    // This shader will run on a half-resolution canvas
    // meaning the texture_coords are in the center of a 2x2 pixel block
    /*
    |---|---|
    | 0 | 1 |
    |---X---|
    | 2 | 3 |
    |---|---|
    */

    ivec2 size = textureSize(tex, 0); // Full-resolution size
    vec2 texelSize = 1.0 / size; // Size of a texel

    // Size we're offset 0.5 texels in each direction
    vec2 offset = vec2(0.5) * texelSize;

    // Calculate the positions on the Full-resolution canvas
    vec2 samplePosition0 = texture_coords + offset * vec2(-1.0, -1.0);
    vec2 samplePosition1 = texture_coords + offset * vec2( 1.0, -1.0);
    vec2 samplePosition2 = texture_coords + offset * vec2(-1.0,  1.0);
    vec2 samplePosition3 = texture_coords + offset * vec2( 1.0,  1.0);

    // Half-resolution size, converted to vec2, so we skip a few automatic conversions
    vec2 outputSize = vec2(love_ScreenSize.xy);

    vec4 average = vec4(0.0);

    // The gpu has something called a FMAD instruction, which is a Fused Multiply-Add
    // Meaning we can multiply and add at the same time for no extra cost!

    average = 0.25 * texelFetch(tex, ivec2(samplePosition0 * size), 0) + average;
    average = 0.25 * texelFetch(tex, ivec2(samplePosition1 * size), 0) + average;
    average = 0.25 * texelFetch(tex, ivec2(samplePosition2 * size), 0) + average;
    average = 0.25 * texelFetch(tex, ivec2(samplePosition3 * size), 0) + average;

    return average;
}
```

`main.lua`
```lua
local pixelationShader = love.graphics.newShader("pixelationShader.fs")

-- pixelScale must be an integer and a power of two
-- pixelationCanvases can just be a table we define outside of our main loop
-- the function will use this table to store the half-resultion canvases, it's a parameter
-- because we need to release them when resizing "gameCanvas" or when changing the format
function pixelate(gameCanvas, pixelScale, pixelationCanvases)
    -- Will try to avoid doing anything if the scale is 1
    -- Because then previousCanvas will be `gameCanvas` and a canvas can't be drawn to itself.
    if pixelScale == 1 then
        return
    end

    local width, height = gameCanvas:getDimensions()

    -- Make sure we don't try to create a 0x0 canvas later
    assert(math.floor(width / pixelScale) > 0, "Pixel scale is too high for the canvas width")
    assert(math.floor(height / pixelScale) > 0, "Pixel scale is too high for the canvas height")

    -- Though this is probably not used, it's good to have it here
    -- So this code doesn't need to be edited if we change the format of our game canvas
    local format = gameCanvas:getFormat()
    local settings = { format = format }

    -- Keep track of the full-size canvas relative to the one we're working on.
    local previousCanvas = gameCanvas

    local mode, alphaMode = love.graphics.getBlendMode()
    -- Disable any blending for the pixelation
    love.graphics.setBlendMode("none")
    love.graphics.setShader(pixelationShader)

    local scale = 2

    -- Create a canvas for each power of two scale if they don't exist yet
    -- and draw the previous canvas to it.
    while scale <= pixelScale do
        if not pixelationCanvases[scale] then
            pixelationCanvases[scale] = love.graphics.newCanvas(width / scale, height / scale, settings)
            pixelationCanvases[scale]:setFilter("nearest", "nearest")
        end

        local canvas = pixelationCanvases[scale]

        love.graphics.setCanvas(canvas)
        love.graphics.draw(previousCanvas, 0, 0, 0, 0.5, 0.5)

        previousCanvas = canvas

        scale = scale * 2
    end

    -- Draw the final canvas to the game canvas
    love.graphics.setCanvas(gameCanvas)
    love.graphics.setShader()
    love.graphics.draw(previousCanvas, 0, 0, 0, scale / 2, scale / 2)

    -- Clean up
    love.graphics.setCanvas()
    love.graphics.setBlendMode(mode, alphaMode)
end
```