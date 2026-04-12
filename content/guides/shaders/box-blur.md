---
title: "Box Blur"
authors: [Jasper]
date: 2025-03-05
---

> [!CAUTION]
> This guide is made for LÖVE 12.0!


For an introduction to shaders, see: [shaders](shaders)

Blurring is a common effect in games, with lots of different implementations and uses

Blurring a picture works by sampling the values of all other nearby pixels within a certain radius and averaging those results.      
However, because of how this effect scales, larger blur radiuses become really computationally expensive.       
Due to the way this effect works, we can do it in two passes, one on the x-axis and one on the y-axis, effectively changing the amount of pixels to so sample per pixel from Count^2 to Count * 2

`boxBlur.fs`:
```glsl
uniform vec2 Offset;
uniform int SampleCount;
uniform Image SourceTexture;

vec4 effect(vec4 color, Image tex, vec2 texture_coords, vec2 screen_coords)
{
    // Keep a sum of all samples
    vec3 sum = vec3(0.0);

    // Type conversions are not done automatically in GLSL ES 
    // Knowing when they happen can help understanding our code better and improve compatability
    vec2 texel_size = 1.0 / vec2(love_ScreenSize.xy);

    for (int sample = 0; sample < SampleCount; sample++)
    {
        sum += Texel(SourceTexture, texture_coords + (Offset * float(sample)) * texel_size).rgb;
    }

    sum /= float(SampleCount);

    return vec4(sum, 1.0);
}
```

`main.lua`:
```lua
local image = love.graphics.newImage("YourImage.png")
local shader = love.graphics.newShader("boxBlur.fs")
local mainCanvas = love.graphics.newCanvas()

-- Since we are applying the shader twice, we need to store the image in between passes
local blurCanvas = love.graphics.newCanvas()
-- Size of the entire box will be (n * 2 + 1) x (n * 2 + 1) (n to each side + center)
shader:send("SampleCount", 16)


-- Quick helper function to draw a 1x1 image full-screen
-- Which avoids having to create a mesh with uv's, since images have that by default
local img = love.graphics.newImage(love.image.newImageData(1, 1))
function drawFullscreen()
    love.graphics.draw(img, 0, 0, 0, love.graphics.getDimensions())
end

function love.draw()
    love.graphics.setCanvas(mainCanvas)
    love.graphics.clear()
    love.graphics.draw(image)

    love.graphics.setCanvas(blurCanvas)
    love.graphics.clear()
    love.graphics.setShader(shader)

    -- Blur on the X-axis
    shader:send("Offset", { 1.0, 0.0 })
    shader:send("SourceTexture", mainCanvas)

    drawFullscreen()
    love.graphics.setCanvas(mainCanvas)

    -- Blur on the Y-axis
    shader:send("Offset", { 0.0, 1.0 })
    shader:send("SourceTexture", blurCanvas)
    drawFullscreen()

    love.graphics.setCanvas()
    love.graphics.setShader()
    love.graphics.draw(mainCanvas)
end
```