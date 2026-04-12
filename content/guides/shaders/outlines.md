---
title: "Outlines"
authors: [Jasper]
date: 2025-03-05
---

> [!CAUTION]
> This guide is made for LÖVE 12.0!


For an introduction to shaders, see: [shaders](shaders)

Outlines can be used to make objects stand out from the background, or to create a more cartoony look.

There are almost an infinite amount of ways to create an outline shader but we'll cover two simple ones here.

The first implementation requires that objects with an outline are drawn to a black canvas, which a shader then calculates the outlines for and overlays over the game.

`outlineShader.fs`
```glsl
// radius in pixels of the outline
uniform float radius;
uniform vec4 OutlineColor;

// Increasing the bias will change how dark a pixel needs to be to be considered outside
// 0.0 is the darkest possible color, 1.0 is the lightest possible color
const float bias = 0.01;

bool outside(Image tex, vec2 texture_coords)
{
    vec3 color = Texel(tex, texture_coords).rgb;
    return color.r <= bias && color.g <= bias && color.b <= bias;
}

vec4 effect(vec4 color, Image tex, vec2 texture_coords, vec2 screen_coords) {
    // Make sure we only color outside of the shape
    if (!outside(tex, texture_coords))
        return vec4(0.0, 0.0, 0.0, 0.0);

    // Size of the outline in pixels
    int size = int(radius);

    vec2 texelSize = 1.0 / vec2(love_ScreenSize.xy);

    // Loop over all pixels within the radius of the outline
    // If any are inside the shape, color the pixel
    for (int x = -size; x <= size; x++)
    {
        for (int y = -size; y <= size; y++)
        {
            if (x * x + y * y <= size * size && !(x == 0 && y == 0))
            {
                vec2 offset = vec2(x, y) * texelSize;
                if (!outside(tex, texture_coords + offset))
                    return OutlineColor;
            }
        }
    }

    return vec4(0.0, 0.0, 0.0, 0.0);
}
```

`main.lua`
```lua
local outlineShader = love.graphics.newShader("outlineShader.fs")

function outline(gameCanvas)
    love.graphics.setShader(outlineShader)
    local mode, alphaMode = love.graphics.getBlendMode()
    love.graphics.setBlendMode("alpha", "premultiplied")

    -- Set the shader's uniform variables
    outlineShader:send("radius", 2)
    outlineShader:send("OutlineColor", { 1, 0, 0, 1 })

    -- Draw the game canvas
    love.graphics.draw(gameCanvas)

    -- Clean up
    love.graphics.setShader()
    love.graphics.setBlendMode(mode, alphaMode)
end
```

The next method does not need a contrast between our image and pure black to work.
This one works by calculating the average of the difference in color around the center pixel
and applying the outline color if it crosses a certain threshold.

`outlineShader.fs`
```glsl
// radius in pixels of the outline
uniform float radius;
uniform vec4 OutlineColor;
uniform float Threshold;

float difference(Image tex, vec2 texture_coords, vec4 color)
{
    vec3 sampleColor = Texel(tex, texture_coords).rgb;
    return distance(sampleColor, color.rgb);
}

vec4 effect(vec4 color, Image tex, vec2 texture_coords, vec2 screen_coords) {
    // Size of the outline in pixels
    int size = int(radius);

    vec2 texelSize = 1.0 / vec2(love_ScreenSize.xy);

    vec4 texColor = Texel(tex, texture_coords);

    float diffSum = 0.0;
    float count = 0.0;

    // Loop over all pixels within the radius of the outline
    // If any are inside the shape, color the pixel
    for (int x = -size; x <= size; x++)
    {
        for (int y = -size; y <= size; y++)
        {
            if (x * x + y * y <= size * size && !(x == 0 && y == 0))
            {
                vec2 offset = vec2(x, y) * texelSize;

                float diff = difference(tex, texture_coords + offset, texColor);

                diffSum += diff;
                count++;
            }
        }
    }

    if (diffSum / count >= Threshold)
    {
        return OutlineColor;
    }

    return vec4(0.0, 0.0, 0.0, 0.0);
}
```

```lua
local outlineShader = love.graphics.newShader("outlineShader.fs")
local outlineColor = { 1, 0, 0, 0.7 }

function outline(gameCanvas)
    love.graphics.setShader(outlineShader)
    local mode, alphaMode = love.graphics.getBlendMode()
    love.graphics.setBlendMode("alpha", "alphamultiply")

    -- Set the shader's uniform variables
    outlineShader:send("radius", 2.5)
    outlineShader:send("OutlineColor", outlineColor)
    outlineShader:send("Threshold", 0.35)

    -- Draw the game canvas
    love.graphics.draw(gameCanvas)

    -- Clean up
    love.graphics.setShader()
    love.graphics.setBlendMode(mode, alphaMode)
end
```