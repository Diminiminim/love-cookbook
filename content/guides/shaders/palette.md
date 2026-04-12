---
title: "Palette"
authors: [Jasper]
date: 2025-03-05
---

> [!CAUTION]
> This guide is made for LÖVE 12.0!


For an introduction to shaders, see: [shaders](shaders)

A palette is a limited set of colors that can be used to create an image. This can be used to create a more cartoony look.

To lock the colors we can have in our image and still have artistic control over those colors, we're going to create an image with all possible colors our output can be, then use a fragment shader to select one which fits the best using a simple approximation.

`paletteShader.fs`
```glsl
uniform sampler2D Palette;

vec4 effect(vec4 color, Image tex, vec2 texture_coords, vec2 screen_coords) {
    ivec2 size = textureSize(Palette, 0);

    // Store the color our image was originally going to be
    vec4 defaultColor = color * Texel(tex, texture_coords);
    // Store the "Distance" To the current closest color
    vec4 closestColor = vec4(1.0);

    // In LDR (Low Dynamic Range), the maximum distance is 1.0
    float closestDist = 1.0;

    // Loop over all colors in the palette
    for (int i = 0; i < size.x; i++) {
        vec4 paletteColor = texelFetch(Palette, ivec2(i, 0), 0);

        // FYI, this is a very simple color distance calculation
        // The visual difference between colors is not linear
        // and not all channels are equal

        float dist = distance(paletteColor.rgb, defaultColor.rgb);

        // If this color is closer than the previous closest, use it
        if (dist < closestDist) {
            closestDist = dist;
            closestColor = paletteColor;
        }
    }

    // Then just return the color
    return closestColor;
}
```

`main.lua`
```lua
local shader = love.graphics.newShader("paletteShader.fs")

-- Calculate a palette with some colors
local channels = 4
local paletteData = love.image.newImageData(channels * channels * channels, 1)

for red = 0, channels - 1 do
    for green = 0, channels - 1 do
        for blue = 0, channels - 1 do
            local x = red + green * channels + blue * channels * channels
            paletteData:setPixel(x, 0, red / (channels - 1), green / (channels - 1), blue / (channels - 1), 1)
        end
    end
end

local palette = love.graphics.newImage(paletteData)
shader:send('Palette', palette)
local image = love.graphics.newImage('YourImage.png')

function love.draw()
    love.graphics.setShader(shader)
    love.graphics.draw(image, 0, 0)
    love.graphics.setShader()
end
```