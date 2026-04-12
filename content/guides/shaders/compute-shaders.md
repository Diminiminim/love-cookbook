---
title: "Compute Shaders"
authors: [Jasper]
date: 2025-02-21
---

**Compute shaders** are a more general way to do calculations with GLSL, as opposed to the vertex and fragment shader, which are a lot more constrained.

> [!CAUTION]
> This guide is made for LÖVE 12.0!


To edit data outside of the shader, we can use **SSBO's** (Shader storage buffer object) and textures.
Textures need to be marked as computewritable with the `computewrite` tag, but we'll get to that.

**SSBO's** are a way to store a large amount of data.
A compute shader can perform `read` and `write` operations on these `buffers`.

One of the things we're going to encounter sometimes when writing compute shaders is issues with memory read and write operations.
This is because we don't have complete control over when threads are accessing data.

## Buffer types

Let's go over some of the different types of `buffers`, these can be combined if need be.
* `shaderstorage`, allows the buffer to be read and written to and from in a shader 
(we can read in fragment, vertex and compute shaders, but only write in compute shaders). One thing to keep in mind is memory alignment of 4 bytes.

* `vertex`, allows the buffer to be used as inputs to a vertex shader, this is what is used when creating a new mesh. Memory alignment isn't as strict allowing for more compact data storage,
If we combine this with `shaderstorage`, the memory alignment will be forced to 4 bytes again, which is something that needs to be accounted for

* `index`, allows the buffer to be used as an index buffer for a vertex shader, which is used to store indices we send with the `setVertexMap` method of meshes.
the format can only be `uint16` and `uint32`.

* `indirectarguments`, allows the buffer to be used parameters for a draw command, effectively allowing the gpu to generate it's own work without needing readbacks.

> [!NOTE] When defining `buffers` in GLSL, LÖVE automatically adds the std430 qualifier, which allows for better packing of data. So we don't have to add that.

## Thread groups

Compute shaders are executed in three dimensional thread groups, each group has N amount of [threads](shaders#thread), which we can define using `local_size_n = m` in the compute shader later.
Defining the local size to amount to 64 [threads](shaders#thread) per thread group is usually optimal, [threads](shaders#thread) within a thread group can communicate with eachother using `shared` variables.
They can also be synced with [barriers](#barriers) meaning every [thread](shaders#thread) has to be at the same point in execution to continue, though this should be done sparingly.

### Built-in variables

The local position in the thread group is stored in the `gl_LocalInvocationID` `uvec3`,
The position of the entire group is stored in the `gl_WorkGroupID` `uvec3`,
And finally the global position (group pos + local pos in group) is stored in the `gl_GlobalInvocationID` `uvec3` input variable  .     

## Particles

let's start with a ~~small~~ compute shader for moving particles around on the screen.
We have two shader files,

`updateParticles.glsl` Is the compute shader which edits the particle data stored in our SSBO, by moving them around.
`drawParticles.glsl` Is the **vertex** and **fragment shader** which draw the particles to the screen.

Finally, our `main.lua` file will tell GLSL how to update our particles and where they spawn initially.

`updateParticles.glsl`
```glsl
// A final local size amounting to 64 is optimal.
layout(local_size_x = 64, local_size_y = 1, local_size_z = 1) in;

// Let's define a struct for our particles
struct Particle {
    vec2 Position;
    vec2 Velocity;
    vec4 Color;
};

// The buffer will be called "Particles" when sending it from the CPU
restrict buffer Particles {
    // An array of the `Particle` struct with an unknown size.
    Particle particles[];
};

uniform mediump float DeltaTime;
uniform mediump uint ParticleCount;

// Min-X, min-Y, max-X, max-Y
uniform mediump vec4 WorldSize;

void computemain() {
    // get the ID of this thread, which we'll use as the index of the particle to simulate.
    uint index = gl_GlobalInvocationID.x;

    // Since this compute shader has a group size bigger than 1 (Which we should always use),
    // The Particle count might not be evenly divisible by the group size,
    // causing us to launch a few extra threads that won't be doing anything.
    if (index >= ParticleCount)
        return;

    // Move the particle
    particles[index].Position += particles[index].Velocity * DeltaTime;

    // Let's make the particles bounce around the screen.

    vec2 Position = particles[index].Position;

    if (Position.x < WorldSize[0]) particles[index].Velocity.x = abs(particles[index].Velocity.x);
    if (Position.x > WorldSize[2]) particles[index].Velocity.x = -abs(particles[index].Velocity.x);

    if (Position.y < WorldSize[1]) particles[index].Velocity.y = abs(particles[index].Velocity.y);
    if (Position.y > WorldSize[3]) particles[index].Velocity.y = -abs(particles[index].Velocity.y);
}
```

`drawParticles.glsl`
```glsl
#pragma language glsl4
// Define our particles again
struct Particle {
    vec2 Position;
    vec2 Velocity;
    vec4 Color;
};

// The restrict keyword allows the compiler to optimize the buffer access better.
// Readonly means we won't be writing to the buffer. (Which we want anyways since that's faster)
// But it will also cause an error if we don't use the buffer as readonly in the shader.
restrict readonly buffer Particles {
    Particle particles[];
};

#ifdef VERTEX
out vec4 vColor;
vec4 position(mat4 transform_projection, vec4 vertex_position) {
    gl_PointSize = 2.0;
    uint index = love_VertexID;
    vColor = particles[index].Color;

    // Ignore the input vertex position and use the particle position instead.
    return transform_projection * vec4(particles[index].Position, 0.0, 1.0);
}
#endif
#ifdef PIXEL
in vec4 vColor;
vec4 effect(vec4 color, Image tex, vec2 texture_coords, vec2 screen_coords) {
    return vColor;
}
#endif
```

```lua
local drawShader = love.graphics.newShader("drawParticles.glsl")

local particleShader = love.graphics.newComputeShader("updateParticles.glsl")

local particleFormat = {
    -- name doesn't do anything but it's nicer to read
    { name = "Position", format = "floatvec2" },
    { name = "Velocity", format = "floatvec2" },
    { name = "Color",    format = "floatvec4" },
}

local particleCount = 1000000
local buffer = love.graphics.newBuffer(particleFormat, particleCount, { shaderstorage = true })

local worldSize = { 0, 0, love.graphics.getWidth(), love.graphics.getHeight() }
particleShader:send("WorldSize", worldSize)
particleShader:send("ParticleCount", particleCount)
particleShader:send("Particles", buffer)
drawShader:send("Particles", buffer)

-- FYI, If we want to update particles from the cpu every frame, or make it start faster,
-- it's better to use ByteData.

local particles = {}
local width, height = love.graphics.getDimensions()

for i = 1, particleCount do
    table.insert(particles, {
        love.math.random(width), love.math.random(height),
        love.math.randomNormal(100), love.math.randomNormal(100),
        love.math.random(), love.math.random(), love.math.random(), love.math.random()
    })
end

buffer:setArrayData(particles)

-- Create a mesh to run the vertex shader
local format = { { name = 'VertexPosition', location = 0, format = 'float' } }
local mesh = love.graphics.newMesh(format, particleCount, 'points',
    'static')

local function updateParticles(dt)
    -- Update the delta time
    particleShader:send("DeltaTime", dt)

    -- Get the local thread group size and divide the amount of particles we have by that amount
    -- Since every thread group will edit that amount of particles.
    local sizeX, sizeY, sizeZ = particleShader:getLocalThreadgroupSize()
    sizeX = math.ceil(particleCount / sizeX)

    -- Use this function to dispatch the compute shader
    love.graphics.dispatchThreadgroups(particleShader, sizeX, sizeY, sizeZ)
end

function love.update(dt)
    updateParticles(dt)
end

function love.draw()
    love.graphics.setShader(drawShader)
    love.graphics.draw(mesh)
    love.graphics.setShader()
    love.graphics.print("Simulating " .. particleCount .. " Particles at " .. love.timer.getFPS() .. " FPS")
end
```

## Average of pixels

> [!IMPORTANT] `Image` and `image2D` are two different things, the first defining a 2D sampler (Readonly texture), the other a 2D image (Read / Write texture)

This compute shader will take any image with a size that is a multiple of 8, and calculate the average of those 64 pixels, then store it in another image.

### Barriers

To make sure all threads have read their data before we start calculating the average, we use a `barrier`, this will make sure all threads are at the same point in execution before continuing.

Barriers only act between threads in the same thread group, meaning we can't sync all of the threads in the compute shader.
GLSL has these barrier functions available:

- `barrier`       
Halts execution until all threads in the thread group have reached this point.
- `memoryBarrier`     
Ensures that all memory accesses before the barrier are visible to all threads in the thread group after the barrier.

The following are variations of the `memoryBarrier` function, each only looking at a specific type of memory access:
- groupMemoryBarrier        
- memoryBarrierAtomicCounter        
- memoryBarrierBuffer
- memoryBarrierImage
- memoryBarrierShared
    

```glsl
// 8*8*1 = 64 threads
layout(local_size_x = 8, local_size_y = 8, local_size_z = 1) in;

// Input Texture
uniform Image InputImage;

layout(rgba8) uniform mediump restrict writeonly image2D OutputImage;
// This line has way too many qualifiers :O
// Let's break it down!
// layout(rgba8), the type of an image needs to be defined beforehand, which we do like so. (rgba8 meaning the format of the image)
// uniform, meaning this can be set from the CPU, which is necessary for images
// mediump, meaning we want mediump precision (only really necessary for mobile)
// restrict, allows the compiler to optimise read and write operations better
// writeonly, tells the compiler we only want to write to this image

// Our first shared variable, every thread within the thread group can read and write to this!
shared vec4[8][8] Colors;
shared vec4 Average;

void computemain() {
    ivec2 position = ivec2(gl_GlobalInvocationID.xy);

    ivec2 size = textureSize(InputImage, 0);

    if (position.x > size.x || position.y > size.y)
        return;

    // Sample at the desired position and mip 0
    vec4 CurrentColor = texelFetch(InputImage, position, 0);

    Colors[gl_LocalInvocationID.x][gl_LocalInvocationID.y] = CurrentColor;

    // Now, if we were to try to calculate the average now, some threads might still be waiting on their texture fetch
    // and we'd be using random numbers (as variables aren't reset to a default when creating them)

    barrier();

    // Let's let the first thread compute the Average

    if (gl_LocalInvocationID.x == 0u && gl_LocalInvocationID.y == 0u)
    {
        vec4 sum = vec4(0.0);
        for (int x = 0; x < 8; x++)
            for (int y = 0; y < 8; y++)
                sum += Colors[x][y];

        Average = sum * (1.0 / 64.0);
    }

    // Wait for the first thread to compute the average
    barrier();

    imageStore(OutputImage, position, Average);
}
```

```lua
local shader = love.graphics.newComputeShader("AveragingShader.glsl")

local img = love.graphics.newImage("YourImage.png")
local blurred = love.graphics.newTexture(love.graphics.getWidth(), love.graphics.getHeight(), { computewrite = true })

shader:send("InputImage", img)
shader:send("OutputImage", blurred)

love.graphics.dispatchThreadgroups(shader, math.ceil(love.graphics.getWidth() / 8),
    math.ceil(love.graphics.getHeight() / 8), 1)

function love.draw()
    love.graphics.draw(blurred)
end
```