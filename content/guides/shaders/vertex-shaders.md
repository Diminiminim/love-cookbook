---
title: "Vertex Shaders"
authors: [Jasper]
date: 2025-03-21
---

Vertex shaders are programs which calculate the final position of a [vertex](#vertex) on screen.

> [!CAUTION]
> This guide is made for LÖVE 12.0!

This guide assumes you've read the [Shaders](shaders) guide.

Let's start with the standard shader LÖVE uses and break it down.

```glsl
varying vec4 vpos;

vec4 position(mat4 transform_projection, vec4 vertex_position)
{
    vpos = vertex_position;
    return transform_projection * vertex_position;
}
```

`varying` is a keyword which defines a variable that will be shared between the [vertex](#vertex) and fragment shader stage.       
By default, these values are interpolated, meaning halfway between, for example, pure red and green, the final value will be [0.5, 0.5, 0.0]

{% love 200, 200, true %}
local PI13 = math.pi * 2 / 3
local mesh = love.graphics.newMesh({
    { 100 + math.cos(PI13 * 2 + math.pi / 6) * 100, 120 + math.sin(PI13 * 2 + math.pi / 6) * 100, 0, 0, 1, 0, 0 },
    { 100 + math.cos(PI13 * 3 + math.pi / 6) * 100, 120 + math.sin(PI13 * 3 + math.pi / 6) * 100, 0, 0, 0, 1, 0 },
    { 100 + math.cos(PI13 + math.pi / 6) * 100,     120 + math.sin(PI13 + math.pi / 6) * 100,     0, 1, 0, 0, 1 },
}, "triangles", "static")

function love.draw()
    love.graphics.draw(mesh)
end

{% endlove %}

`mat4 transform_projection` is the variable which is doing all of the hard work,        
it's the [matrix](#Matrices) which stores the current coordinate system transform, so translation, rotation and scale and the [projection matrix](#Projection-Matrix).         
By default LÖVE uses an [orthographic](#Orthographic) [projection matrix](#Projection-Matrix) which goes [-10, 10] on the z-axis, meaning if objects are outside of that range, they won't be visible anymore.        
This is because the z values of the pixels will lie outside of the [-1 to 1] range and GLSL will skip drawing those.

`vec4 vertex_position` is the input [vertex](#vertex)'s position, by default `w = 1`, any time a position is multiplied with a [projection matrix](#Projection-Matrix), it should be `1` otherwise it will cause issues.

Let's do some shader magic and create a [vertex](#vertex) shader which automatically covers the entire screen with a single triangle, instead of having to create a 1x1 image like we did before.

## Fullscreen triangle

One thing to note is that this shader is written with `void vertexmain` instead of `vec4 position`, because we don't need the parameters it provides us.

`fullscreenTriangle.vs`
```glsl
// Store UV-Coordinates to be used in the fragment shader
varying vec2 VarVertexCoord;
#ifdef VERTEX
void vertexmain() {
    // We want a triangle which covers the entire screen, so we need to generate a triangle with vertices at the positions:
    // [0, 0], [0, 2], [2, 0]
    
    // Bit shifting magic to create these coordinates
    VarVertexCoord = vec2((love_VertexID << 1) & 2, love_VertexID & 2);

    vec4 VarScreenPosition = vec4(VarVertexCoord.xy * vec2(2.0) + vec2(-1.0), 0.0, 1.0);

    gl_Position = VarScreenPosition;
}
#endif
```

## Technical terms

### Vertex
A vertex is a point in 2D or 3D space, it's the most basic building block of any model.
They're what's used to create triangles, which are then used to create more complex shapes.
For example, a rectangle has 4 vertices, one on each corner forming two triangles.

A vertex can contain more information than just a position. In fact, we can fully customize what a vertex contains.
This is done with a `vertex format` which is the first input when creating a mesh.

The default vertex format looks like this:
```lua
{       
    { location = 0, format = "floatvec2" },    
    { location = 1, format = "floatvec2" }, 
    { location = 2, format = "unorm8vec4" },  
}       
```

The first entry defines format of the x, y position of the vertex.
The location is used to tell the shader where to find this information, we can define this in our shader with `layout(location = 0) in vec2 position;`.
the format is `floatvec2` which is just a `vec2` in GLSL.

The second entry defines the [uv](shaders/#uv-coordinates) coordinates of the vertex.  

And lastly the third entry defines the color of the vertex.
This format is a bit different with `unorm8vec4`, a `norm` value means it's normalized to the range of [-1, 1]
and `unorm` means it's normalized to the range of [0, 1].
`8` means it's an 8-bit value, so it can be between 0 and 255. So, putting it all together, this is a `vec4` with values between 0 and 1.

### Matrices
Matrices are a way to store a 2D array of values, like a 4x4 array of floats.
Matrices are very useful for transforming coordinates, like rotating, scaling and translating.      
Usually starting from the identity matrix, which looks like:
```
1 0 0 0
0 1 0 0
0 0 1 0
0 0 0 1
```

### Projection-Matrix
The projection matrix is a [4x4 matrix](#Matrices) which brings the 3D world to a 2D screen, more specifically, transforms a 3D point in view space to a 2D point in [NDC space](#NDC-space).

### NDC-space
Normalized Device Coordinates, a vec3 ranging from [-1, 1] on the x, y and z axis.      
vec3(-1) is the top left corner of the screen, vec3(1) is the bottom right corner of the screen.

### Orthographic
An orthographic projection is a way to project 3D objects to a 2D screen, without any perspective, meaning things don't get smaller the further away they are, as opposed to a perspective projection.