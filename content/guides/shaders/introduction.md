---
title: "Shaders"
authors: [Jasper]
date: 2025-03-05
---

With **shaders** we can create fun but advanced graphical effects for our games.

> [!CAUTION]
> This guide is made for LÖVE 12.0!

Ranging from simple color changes to complex post-processing effects.

There are multiple types of shaders available to be used in LÖVE
* The **Fragment**, also known as the **Pixel** shader
* The **Vertex** shader
* The **Compute** shader

A shader edits the color of what's drawn, like when drawing images or text.

Shaders can be created using `love.graphics.newShader`.

Compute shaders are covered in the [Compute Shaders](compute-shaders) guide, but it's more advanced and it's recommended to read this guide first.

## Syntax

LÖVE shaders are written in the shading language GLSL, which is quite different from lua.

Because of this, we can transfer shaders between different languages and engines, with some minor changes.

Let's start out with a grayscale shader and break it down from there.

```glsl
vec4 effect(vec4 color, Image tex, vec2 texture_coords, vec2 screen_coords)
{
    color *= Texel(tex, texture_coords);
    float gray = length(color.rgb);
    return vec4(gray, gray, gray, 1.0);
}
```

In lua this would look something like:

```lua

function length(vec3)
    return math.sqrt(vec3.x * vec3.x + vec3.y * vec3.y + vec3.z * vec3.z)
end

function effect(color, tex, texture_coords, screen_coords)
    color = color * Texel(tex, texture_coords)
    local gray = length(color)
    return { gray, gray, gray, 1.0 }
end

```

> [!NOTE] this code won't actually work but it illustrates how the syntax is different.

Let's compare the syntax of GLSL and lua.

### Variables

```lua
local x = 1
local y = 2
local z = x + y
```

We can write this in two ways, either using floats or ints. A float stands for floating point number, meaning the . can be anywhere in the number.

On the other hand, an int is an integer, which can't have a decimal point.

```glsl
float x = 1.0;
float y = 2.0;
float z = x + y;
```

```glsl
int x = 1;
int y = 2;
int z = x + y;
```

However, in some situations we might want to add a float and an int together. We can do this in two ways, converting the values to an int, or a float.

```lua
local x = 1.5
local y = 2
local z = x + y
```

We can do the calculation resulting in a float like this:
```glsl
float x = 1.5;
int y = 2;
float z = x + float(y); // z = 3.5
```

Or resulting in an int like this:
```glsl
float x = 1.5;
int y = 2;
int z = int(x) + y; // z = 3, because x = 1.5 -> int = 1
```


### Conditionals
```lua
if x > y then
    return x
else
    return y
end
```

```glsl
if (x > y)
{
    return x;
}
else
{
    return y;
}
```
Note that brackets for statements {}, aren't needed if you only have a single line of code inside, This does not apply to functions however.

It does mean we can write it like this as well:

```glsl
if (x > y)
    return x;
else
    return y;
```

### Loops
```lua
local sum = 0

for i = 0, 10 do
    sum = sum + i
end
```

```glsl
int sum = 0;

for (int i = 0; i < 11; i++)
{
    sum += i;
}
```

### Functions
```lua
function add(x, y)
    return x + y
end
```

```glsl
float add(float x, float y)
{
    return x + y;
}
```

### Arrays
```lua
local array = { 1, 2, 3 }
```

```glsl
int array[3] = { 1, 2, 3 };
```

### Structs
```lua
local myData = { 
    x = 1, 
    y = 2,
    isActive = true
}
```

```glsl
struct {
    float x;
    float y;
    bool isActive;
} MyDataStruct;

MyDataStruct myData = MyDataStruct(1.0, 2.0, true);
```

GLSL also supports overloading, meaning you can have multiple functions with the same name, as long as the output type is the same.

## The Fragment shader

This shader is executed for every pixel the effect covers.This is possible because shaders are really fast and so is the GPU.

Let's make a simple fragment shader from a string to show it's functionality.

During this guide, we will be creating new shader files, usually with the `.fs`: Fragment Shader, `.vs`: Vertex Shader or `.glsl` extension.

We start by defining the output, `vec4`.

You can think of a vec4 as a table with 4 numbers, like { x, y, z, w }.
`vec4 color` is the color we provide with `love.graphics.setColor`.

`Image tex` is the texture of our draw call, like the image from `love.graphics.draw(image)`.

`vec2 texture_coords` are the [uv-coordinates](#uv-coordinates) defined by the vertices of our mesh or automatically by LÖVE for images and similar [0-1].

`vec2 screen_coords` are the coordinates on screen, [0.5 to love_ScreenSize.xy-0.5].

### Grayscale color

`main.lua`
```lua
local shaderCode = [[
// This is one of the "entry points" supported by LÖVE.
vec4 effect(vec4 color, Image tex, vec2 texture_coords, vec2 screen_coords)
{
    float gray = length(color.rgb);
    // Colors are returned as (Red, Green, Blue, Alpha)
    return vec4(gray, gray, gray, 1.0); // Color the square gray
}
]]

local shader = love.graphics.newShader(shaderCode)

function love.draw()
    love.graphics.setShader(shader)
    -- We'll draw a couple of differently colored squares to see the effect.
    love.graphics.setColor(1, 0.2, 0)
    love.graphics.rectangle("fill", 100, 100, 200, 200)

    love.graphics.setColor(0, 1, 0.5)
    love.graphics.rectangle("fill", 400, 100, 200, 200)

    love.graphics.setColor(0.3, 0, 1)
    love.graphics.rectangle("fill", 700, 100, 200, 200)
end
```

---

### Drawing images

Let's step things up by drawing an inverted image this time and by using a different file instead of a string.     
We'll have to change our shader a bit to allow it to draw images.       
We do this by telling the shader to load the color values of our image when coloring a pixel.

`drawImage.fs`
```glsl
vec4 effect(vec4 color, Image tex, vec2 texture_coords, vec2 screen_coords)
{
    // Use Texel to sample the color of a texture at a coordinate.
    // We can use .rgb to convert a vec4 to a vec3, as we don't need the alpha value.
    vec3 imageColor = Texel(tex, texture_coords).rgb;

    // Make the image red by multiplying by our Red vec4.
    // Invert the color and create a vec4 with w set to 1.0;
    return vec4(1.0 - imageColor, 1.0);
}
```

`main.lua`
```lua
local image = love.graphics.newImage("YourImage.png")

local shader = love.graphics.newShader("drawImage.fs")

function love.draw()
    love.graphics.setShader(shader)
    love.graphics.draw(image)
end
```

---

### Uniform values

Uniform values are a way for LÖVE send data from Lua to GLSL **Not the other way around!**     
Let's edit our shader to change the color of our image over time, using a uniform value.
In this recipe we'll be using `camelCase` for local variables and `PascalCase` for global variables and uniform values.

> [!NOTE]
> Uniform values can get optimised out of shader code,      
> if shader:send is encountering an error even though the uniform is defined,
> make sure it's contributing to the final output of the shader.

`unusedUniform.fs`

```glsl
uniform float myUniformValue;

vec4 effect(vec4 color, Image tex, vec2 texture_coords, vec2 screen_coords)
{
    return vec4(1.0);
}
```

`main.lua`

```lua
local shader = love.graphics.newShader("unusedUniform.fs")

function love.draw()
    shader:send("myUniformValue", 1.0)
    love.graphics.setShader(shader)
    love.graphics.rectangle("fill", 100, 100, 200, 200)
end
```

This will throw an error, because the uniform value is not used in the shader code.

```
Error

main.lua:4: Shader uniform 'AnUnusedUniform' does not exist.
A common error is to define but not use the variable.
```


```lua
local image = love.graphics.newImage("YourImage.png")

local shaderCode = [[

// Add a uniform vec4, which we can edit in lua.
uniform vec4 ColorOffset;

vec4 effect(vec4 color, Image tex, vec2 texture_coords, vec2 screen_coords)
{
    // Use Texel to sample the color of a texture at a coordinate
    vec4 imageColor = Texel(tex, texture_coords);

    // Make the image red by multiplying by our Red vec4
    return imageColor * vec4(1.0, 0.0, 0.0, 1.0) + ColorOffset;
}
]]

local shader = love.graphics.newShader(shaderCode)

local time = 0

function love.update(dt)
    time = time + dt
end

function love.draw()
    -- Create a sin-wave [0-1], as the green color to add.
    shader:send("ColorOffset", { 
        0, 
        math.sin(time) * 0.5 + 0.5, -- Multiplying by 0.5 and adding 0.5 brings the sin wave from [-1, 1 to [0, 1]
        0, 
        0
    })

    love.graphics.setShader(shader)
    love.graphics.draw(image)
end
```

### Wave effect

Let's create a wave effect using a sine wave, something that's quite difficult to do without shaders.

`wave.fs`

```glsl
#define PI 3.14159265359

uniform float Time;
uniform float WaveCount;
uniform float Scale;

vec4 effect(vec4 color, Image tex, vec2 texture_coords, vec2 screen_coords)
{
    // Create a sine wave using the time value.
    float wave = sin(texture_coords.y * PI * 2.0 * WaveCount + Time);

    // Use the sine wave to offset the texture coordinates.
    vec2 offsetCoords = texture_coords + vec2(wave * Scale, 0.0);

    // Use Texel to sample the color of a texture at a coordinate.
    vec4 imageColor = Texel(tex, offsetCoords);

    return imageColor * color;
}
```

`main.lua`

```lua
local image = love.graphics.newImage("YourImage.png")

local shader = love.graphics.newShader("wave.fs")

local time = 0
local waveCount = 5
local scale = 0.1

function love.update(dt)
    time = time + dt
end

function love.draw()
    shader:send("Time", time)
    shader:send("WaveCount", waveCount)
    shader:send("Scale", scale)

    love.graphics.setShader(shader)
    love.graphics.draw(image)
end
```

There are some limitations to take in to consideration when writing shader code, the main one is that, due to the way GPUs work the entire [register](#Registers) usage needs to be known beforehand, meaning dynamic memory allocations like these:
```glsl
uniform int IntCount;
uniform int[IntCount] UniformIntegers
```
aren't allowed.     
Which leads to another limitation, recursive code is not allowed either, use a stack and a while loop instead.

### Types

Scalar (single value) types:
* `float`: 32-bit floating point number, for storing fractional values
* `int`: 32-bit integer value
* `uint`: unsigned version of int, meaning it can't be negative, allowing for twice the range of values
* `bool`: boolean value (`true` or `false`)

Vector types:
* `vecn`: float type vector
* `ivecn`: integer type vector
* `uvecn`: unsigned integer type vector
* `bvecn`: boolean type vector

[Matrix](#matrices) types:
* mat*n*x*m*: n columns, m rows (column major) 

>[!IMPORTANT]
> When doing math in shaders, make sure to use the correct literal number formatting        
> `1 / 2 = 0` Values without type suffixes will be interpreted as integers      
> `1.0 / 2.0 = 0.5` Floating point division

Type suffixes
* `float`: *x*.;*x*.0;*x*.f (1. ; 1.0 ; 1.f)
* `int`: *x* ( 1 )
* `uint` *x*u ( 1u )

[This](https://learnwebgl.brown37.net/12_shader_language/glsl_mathematical_operations.html) page explains operators.

## Object Management
You can think of `Textures` and `Buffers` like tables, like tables, they have a reference to the actual data, but they're not the data itself.

Meaning if you change the data in the table, the data will change as well.

For example, if we send a canvas to a shader and render to it with a `setCanvas` call, the memory in [VRAM](#Vram) will have changed.

So when using that canvas later, the contents will be different from when you sent the image, as it's just a reference.

Sending images and buffers to the gpu is expensive (Using newTexture, newBuffer or similar), but sending the reference (Using shader:send) is not.

## Shader optimisation

While shader optimisation is an incredibly advanced topic, these are still some useful tips to keep in mind while creating stunning effects!

* Try to minimise texture samples, and try to keep the position consistent and close to the source.
* The gpu is great at doing a lot of small tasks at the same time, so try to avoid really long for loops or huge shaders.
* Implicit conversions, while doing a multiplication two different types is fine, try to avoid it in a for loop if possible, converting an int to a float once is better than 100x!
* Precision qualifiers, these don't do anything on desktop gpus but mobile devices do calculations a lot faster at lower precisions.

These are also useful but might not be as relevant when creating normal effects.
* Data packing, instead of using 4 32-bit integers to store 4 8-bit integer values, if the size is known beforehand, they can be packed before writing and unpacked before reading.
* Register usage, using more registers (determined by the maximum amount of values used at any point in a shader) *can* cause the shader to run slower.

## Technical terms

### UV-coordinates
UV-coordinates are a way to map a 2D texture to an object of any shape, 2D and 3D alike.
The U and V stand for the horizontal and vertical axis, respectively.
When present, each vertex stores the coordinates on the texture where it should be drawn,
usually ranging from [0, 1] but can be outside of that range.

> [!WARNING] These are quite in-depth terms and you probably won't need to know much about them when creating shaders.

### Registers
Registers are a way to store data on the GPU, like variables in lua, each "register" can old a 32-bit value, like a float or an int. There are two types of registers on the gpu,       
Vector and Scalar registers, the name is very misleading, you'd expect that a vector register can hold multiple values and scalar only one, but it has nothing to do with the data type.        
Rather it's about how the data is used.     

A vector register can store a single variable which varies per thread, like the color of a pixel. These are usually allocated in batches of 8, using too many can slow down your shaders considerably.

A scalar register can store a single variable which is the same for all threads, like a uniform value. They're also usually allocated in batches of 256, so you have a lot of them to work with.

### VRAM
Vram is just the memory on your GPU, where all the textures, buffers and shaders are stored. It's a lot faster than normal RAM, but usually a lot smaller.

### Thread
A thread is a single unit of execution, like a single pixel on the screen, a vertex to transform or instance of a [compute shader](compute-shaders).
