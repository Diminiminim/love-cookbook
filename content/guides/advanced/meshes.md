---
title: "Meshes"
authors: [clemapfel]
date: 2026-07-14
---

# Meshes

This chapter will cover **Meshes**, which are a central concept in graphics programming. They are what is used to display any geometry on screen. While usually shunned by beginners deemed unnecessary due  the high level of abstraction LÖVE offers when drawing to the screen, meshes are actually used internally by LÖVE for literally all drawing. This includes for `love.graphics.rectangle`, `love.graphics.circle`, `love.graphics.polygon`, and `draw`ing `Images`, `SpriteBatche`s, `ParticleSystem`s, etc. -- all are actually powered my meshes!

By overcoming complexity and mastering meshes, we can extend LÖVEs graphical capability significantly. For example, it allows us to render shapes not possible with any other of LÖVEs existing API. Using *geometry instancing*, we can even achieve drawing performance equal to or even better than `SpriteBatch` or `love.graphics.circle`. This is despite LÖVE being known for it's extremely fast drawing.

#### In this chapter we will learn:

+ Why and when to use meshes
+ What *vertices*, *edges*, *meshes* are
+ What vertex attributes, texture coordinates, texture bindings are
+ What a mesh's `DrawMode` is
+ What graphics `BufferUsage` means
+ How to access and modify vertex and index buffer of a mesh
+ How to use geometry instancing
+ How to create a graphics buffer and upload to them using Lua tables or extremely fast C memory

---

## Table of Contents

# Table of Contents

+ [0. TL;DR: Quick Start](#0-tldr-quick-start)
    + [0.1 Creating a Textured Rectangle](#01-creating-a-textured-rectangle)
    + [0.2 Creating a Colored Circle](#02-creating-a-colored-circle)
    + [0.3 Creating a Polygon Mesh using Triangulation](#03-creating-a-polygon-mesh-using-triangulation)
+ [1.0 What Are Meshes? Motivation](#10-what-are-meshes-motivation)
    + [1.0.1 Drawing Custom Shapes](#101-drawing-custom-shapes)
    + [1.0.2 Texture Deformation](#102-texture-deformation)
    + [1.0.3 Rendering 10k Copies of the same Shape](#103-rendering-10k-copies-of-the-same-shape)
    + [1.0.4 Drawing 3D Objects](#104-drawing-3d-objects)
+ [1.1 Glossary: Vertices, Edges, Tris, Meshes](#11-glossary-vertices-edges-tris-meshes)
+ [1.2 Mesh Draw Modes](#12-mesh-draw-modes)
+ [1.3 Index Buffer / Vertex map](#13-index-buffer--vertex-map)
    + [1.3.1 `setVertexMap`](#131-setvertexmap)
+ [1.5 Vertex Attribute Interpolation](#15-vertex-attribute-interpolation)
    + [1.5.1 Creating a Gradient](#151-creating-a-gradient)
+ [1.6 Texture Coordinates](#16-texture-coordinates)
    + [1.5.1 `setTexture`](#151-settexture)
    + [1.5.2 Texture Wrap Mode and Meshes](#152-texture-wrap-mode-and-meshes)
+ [1.7 Vertex Attributes and Shaders](#17-vertex-attributes-and-shaders)
    + [1.7.1 `effect` and `position`](#171-effect-and-position)
    + [1.7.2 Fragments](#172-fragments)
+ [1.8 Replacing Vertex Data: `setVertices`](#18-replacing-vertex-data-setvertices)
    + [1.8.1 Graphics Buffer Usage](#181-graphics-buffer-usage)
    + [1.8.2 `setVertices` Example](#182-setvertices-example)
+ [2. Advanced Usage: Geometry Instancing](#2-advanced-usage-geometry-instancing)
    + [2.1 Geometry Instancing](#21-geometry-instancing)
    + [2.2 Vertex Buffers](#22-vertex-buffers)
    + [2.3 Storage Buffers](#23-storage-buffers)
    + [2.4 Texel Buffers](#24-texel-buffers)
    + [2.5 C-Data](#25-c-data)
    + [2.5 Mesh Advanced Usage Example](#25-mesh-advanced-usage-example)

---

# 0. TL;DR: Quick Start

#### [\[click here to skip to section 1 of this chapter\]](#10-what-are-meshes-motivation)

Given here are code snippets that illustrate basic usage of meshes. These are **intended to be referenced after having read this chapter**. Readers new to meshes are not expected to understand these snippets and should [skip to section 1.0](#10-what-are-meshes-motivation).

### 0.1 Creating a Textured Rectangle

```lua
--- initialization
local x, y, w, h = 50, 50, 200, 200 -- top left xy, width, height
local r, g, b, a = 1, 1, 1, 1 -- color: pure white
local rectangle = love.graphics.newMesh({
    --    x,     y,   u, v,   r, g, b, a
    { x + 0, y + 0,   0, 0,   1, 1, 1, 1 },
    { x + w, y + 0,   1, 0,   1, 1, 1, 1 },
    { x + w, y + h,   1, 1,   1, 1, 1, 1 },
    { x + 0, y + h,   0, 1,   1, 1, 1, 1 },
},
    "fan",    -- mesh draw mode
    "dynamic" -- graphics buffer usage
)

-- load a texture
local texture = love.graphics.newTexture("assets/toast.jpg")
rectangle:setTexture(texture) -- associate mesh with texture

-- drawing the mesh
love.draw = function()
    love.graphics.setColor(1, 1, 1, 1)
    love.graphics.draw(rectangle)
end
```

### 0.2 Creating a Colored Circle

```lua
-- initialization
local x, y, xr, yr = 200, 200, 100, 100 -- center xy, x-radius, y-radius
local vertexCount = 64 -- number of outer vertices
local r, g, b, a = 1, 0, 1, 1 -- color: magenta

local circle -- love.Mesh
do
    local vertices = {}
    local circle_data = {
        -- center vertex
        { x, y, 0, 0, r, g, b, a }
    }

    -- using trigonometry to generate the circle vertices
    for i = 0, vertexCount do -- start at 0
        local angle = i / vertexCount * 2 * math.pi
        table.insert(circle_data, {
            x + math.cos(angle) * xr, -- x
            y + math.sin(angle) * yr, -- y
            math.cos(angle), -- u
            math.sin(angle), -- v
            r, g, b, a
        })
    end

    circle = love.graphics.newMesh(circle_data,
        "fan", -- draw mode: triangle fan
        "dynamic"
    )
end

-- drawing
love.draw = function()
    love.graphics.setColor(1, 1, 1, 1)
    love.graphics.draw(circle)
end
```

### 0.3 Creating a Polygon Mesh using Triangulation

```lua
-- input vertex data. format: { x1, y1, x2, y2, ... }
local input = {
    --   x,                y
    167.8622990257,   0,
    141.37412207129,  153.70547063563,
    -41.827243725195,  198.73007856213,
    -187.28713855132,  102.30176491128,
    -159.32457952398, -89.182472409685,
    -39.483305587746, -162.49970086587,
    148.0247332512,  -152.10238729167,
}

-- triangulate using loves built-in triangulation, only works for convex polygons
local triangles = love.math.triangulate(input)

-- create the polygon mesh
local polygon -- love.Mesh
do
    -- get bounding box, needed for texture coordinate calculation
    local xMin, yMin = math.huge, math.huge
    local xMax, yMax = -math.huge, -math.huge
    for i = 1, #input, 2 do
        local x = input[i + 0]
        local y = input[i + 1]
        xMin = math.min(xMin, x)
        yMin = math.min(yMin, y)
        xMax = math.max(xMax, x)
        yMax = math.max(yMax, y)
    end

    -- convert vertex position to texture coordinate
    local xy2uv = function(x, y)
        return (x - xMin) / (xMax - xMin),
        (y - yMin) / (yMax - yMin)
    end

    local vertex_data = {}
    local r, g, b, a = 1, 1, 1, 1

    -- iterate each vertex of each triangle, add it as a mesh vertex
    for _, triangle in ipairs(triangles) do
        -- `triangles` format: {{ x1, y1, ...}, {x1, y1 ...}, ... }
        for i = 1, #triangle, 2 do
            -- `triangle` format: { x1, y1, x2, y2, x3, y3 }
            local x = triangle[i + 0]
            local y = triangle[i + 1]
            local u, v = xy2uv(x, y)
            table.insert(vertex_data, {
                x, y, u, v, r, g, b, a
            })
        end
    end

    -- create the mesh
    polygon = love.graphics.newMesh(vertex_data,
        "triangles", -- draw mode: separate triangles
        "dynamic"
    )
end

-- usage
love.draw = function()
    love.graphics.setColor(1, 1, 1, 1)
    love.graphics.push()
    love.graphics.translate(300, 300) -- mesh is at origin otherwise
    love.graphics.draw(polygon)
    love.graphics.pop()
end
```

---

# 1.0 What Are Meshes? Motivation

Scrolling past the code snippets in section 0, meshes can seem quite intimidating - and they are. Both from a conceptual and implementation perspective. Before diving into this topic, we may want to first answer the question of **why we would want to use meshes at all**? What are situations where we *have* to use meshes, situations where no other LÖVE API can achieve a certain result?

### 1.0.1 Drawing Custom Shapes

The `love.graphics` shape drawing API has certain limitation, for example, there is no way to draw an **annulus** (known as a "ring" or "donut" in common language).

![](/assets/img/meshes/annulus_rainbow.png)

This image uses only a single mesh to achieve both the shape and rainbow gradient. **No shader is used** here.

`love.graphics` furthermore does not allow shapes to have more than one color, or to draw a textured shape that is not a rectangle. These types of drawables are only achievable with `love.Mesh`.

### 1.0.2 Texture Deformation

The indie game ["Who's Lila"](<https://store.steampowered.com/app/1697700/Whos_Lila/>) has a mechanic where a picture of the players face is displayed on screen, and the player is tasked to drag certain parts of the face to manually create certain facial expressions. 

![](/assets/img/meshes/whos_lila.gif)

*(Source: [IndieDB](<https://www.indiedb.com/games/whos-lila>), Copyright: [Garage Heathen](<https://store.steampowered.com/developer/GarageHeathen>))*

Reproducing this mechanic in LÖVE is perfectly possible, and would require nothing but meshes. Achieving this with any other technique would be impractical.

### 1.0.3 Rendering 10k Copies of the same Shape

While 3D in this example, the technique used to render tens of thousands of blades of grass in games like "Breath of the Wild" is a graphical capability of most modern GPUs called **geometry instancing**. This technology is available in LÖVE, and LÖVE could match Nintendos custom engine capabilities - at least in this specific. It is well suited to draw the same mesh tens of thousands if not hundreds of thousands of times in a single frame. This works exactly the same both in 3D and 2D.

![](/assets/img/meshes/botw_grass.png)

*(Source: [Nintendo Eshop](<https://www.nintendo.com/us/store/products/the-legend-of-zelda-breath-of-the-wild-switch/>), Copyright: Nintendo)*

#### 1.0.4 Drawing 3D Objects

While not introduced in this chapter, LÖVE does have the capability to draw true 3D. Only meshes - no other object - can represent a true 3D shape such as a sphere or model of a cat. Meshes are integral to any LÖVE project wanting to implement true 3D.

## 1.1 Glossary: Vertices, Edges, Tris, Meshes

Having realized how important meshes, we are now motivated to put in the effort to actually understand LÖVE meshes. We first turn our attention to *terminology*. While this chapter will avoid math (such as differential geometry) as much as possible, certain terms are unavoidable when speaking about meshes in a technical context. These will be defined here.

A **vertex** (plural: *vertices*) is a point in space (2D or 3D). It has an x coordinate and y coordinate (and z in 3D), as well as any number of additional properties, such as a color (encoded as four numbers, red, green, blue, opacity). Any such property of a vertex is called a **vertex attribute**.

Given *exactly two vertices*, an **edge** is a conceptual straight line, drawn from the first vertex' position to the second. An edge is ordered, meaning the edge from vertex `A` to `B` is **not** same object as an edge from `B` to `A`. Whether to count `A` or `B` as the start vertex is called the **[winding order](https://love2d.org/wiki/love.graphics.setFrontFaceWinding)**. 

Given *exactly three vertices*, a **tri** is the triangle that is defined by these 3 points. This is sometimes called a *face* in math, but *tri* or *triangle* is more commonly used when talking about it in relation to graphics programming and gamedev. A tri is also uniquely defined by exactly 3 edges.

A **mesh**, then, is a **collection of one or more tris**. Note that these tris do not need to be connected, though they can be. Since vertices can be 2D or 3D, tris, too, can be 2D or 3D. There is no practical difference in terms of vertex attributes or how meshes work when comparing 2D meshes to 3D meshes. Because of this, we will assume all vertices, edges, and meshes are 2D henceforth. 

In LÖVE, unless otherwise specified, vertices have the following attributes (where we will explain what uv coordinates are in a later section):

+ **position**: two numbers for the x-coordinate and y-coordinate, their *unit is pixels*
+ **texture coordinates**: two numbers for the u-coordinate and v-coordinate, each normalized to the value range `[0, 1]`
+ **color**: four numbers, one for the `r`, `g`, `b`, and `a` (alpha / opacity) component, each normalized to the value range `[0, 1]`

These properties **are stored inline**, meaning a vertex with this default *vertex format* (position, texture coordinate, rgba) is a flat table of numbers like so:

```lua
local vertex = { 
    200.5, 300,  -- x, y
    0.3, 0.4,    -- u, v
    1, 0, 1, 0.5 -- r, g, b, a
}
```

We see all components of each attributes are stored in the same table one after another. A vertex is always a table of 1 or more numbers.

A tri, then, is a collection of three vertices:

```lua
local tri = {
    { -- vertex #1
        45, 170,  -- xy: position is (45, 170)
        0.0, 0.0, -- uv: texture coordinate is (0, 0)
        1.0, 0.0, 0.0, 1.0 -- rgba: color is red
    },
    
    { -- vertex #2
        210, 45,  -- xy: position is (210, 45)
        1.0, 0.0, -- uv: texture coordinate is (1, 0)
        0.0, 1.0, 0.0, 1.0 -- rgba: color is blue
    },
    
    { -- vertex #3
        200, 235, -- xy: (210, 45)
        0.5, 1.0, -- uv: (0.5, 1.0)
        0.0, 0.0, 1.0, 1.0 -- rgba: green
    }
}
```

A table that contains three or more vertices, defining one or more tris, will also be called the **vertex data** of a mesh. 

Now that we have a tri, we can finally draw our first mesh. Recall that a mesh is a collection of one or more tris, meaning exactly a single tri still counts as a mesh.

We use `love.graphics.newMesh` to create a mesh, and `love.graphics.draw` to draw it. It reacts to `setColor` and transforms like `love.graphics.translate` as does any other love drawable, for example `love.Image` or `love.graphics.circle`.

The third and fourth parameter of `newMesh` will be looked at in a later section, for now we will put them to `"strip"`, and `"dynamic"`. 

```lua
local tri = {
    { 45,  170, 0.0, 0.0, 1.0, 0.0, 0.0, 1.0 },  -- vertex #1: xy uv red
    { 210,  45, 1.0, 0.0, 0.0, 1.0, 0.0, 1.0 },  -- vertex #2: xy uv green
    { 200, 235, 0.5, 1.0, 0.0, 0.0, 1.0, 1.0 },  -- vertex #3: xy uv blue
}

-- instance a new mesh
local mesh = love.graphics.newMesh(
    tri, -- vertex data
    "strip", 
    "dynamic" -- explained in later section
)

-- draw routine
love.draw = function()
    -- set color, it will be multiplied with each vertex' rgba
    love.graphics.setColor(1, 1, 1, 1)
    
    -- draw the mesh like any other LÖVE object
    love.graphics.draw(mesh)
end
```

![](/assets/img/meshes/tri_hello_world.png)

Before we analyze this image, let's add another vertex first, just to see what happens:

```lua
local tri = {
    { 45, 170, 0.0, 0.0, 1.0, 0.0, 0.0, 1.0 },  -- #1: xy uv red
    { 210, 45, 1.0, 0.0, 0.0, 1.0, 0.0, 1.0 },  -- #2: xy uv green
    { 200, 235, 0.5, 1.0, 0.0, 0.0, 1.0, 1.0 }, -- #3: xy uv blue
    
    -- new fourth vertex
    { 120, 280, 0.5, 1.0, 1.0, 0.0, 1.0, 1.0 }, -- #4: xy uv magenta
}

-- instance and draw like before
local mesh = love.graphics.newMesh(tri, "strip", "dynamic")

love.draw = function()
    love.graphics.setColor(1, 1, 1, 1)
    love.graphics.draw(mesh)
end
```

We now see:

![](/assets/img/meshes/quad_hello_world.png)

Two question arise: 
+ (i) Why do *four* vertices result in a valid mesh, when meshes are collection of triangles, and each triangle is exactly three vertices?
+ (ii) Why is the mesh colored like that? Why do we see a smooth mix of colors instead of a single color?

Turning our attention to (i) first, we can use one of LÖVEs rarely used API functions to answer this question. `love.graphics.setWireframe` makes any mesh draw call display the *edges*,  instead of the meshes *faces*. 

>[!WARNING]
> Due to unreliability, `setWireframe` should never be used in a shipped game. If we want to draw a wireframe of a mesh properly, we should use `love.graphics.line`

```lua
love.draw = function()
    love.graphics.setColor(1, 1, 1, 1)
    love.graphics.setWireframe(true) -- draw only edges
    love.graphics.draw(mesh)
    love.graphics.setWireframe(false) -- draw only faces (default)
end
```

![](/assets/img/meshes/quad_hello_world_wireframe.png)

This confirms that the mesh *is* actually made up out of two valid triangles. Our definition of a mesh being a collection of tris is still correct. 
We notice that the two triangles share an edge. This is a key observation. Indeed, we can reproduce the above shape like so:

```lua
--[[
before:
local tri = {
    { 45, 170, 0.0, 0.0, 1.0, 0.0, 0.0, 1.0 },  -- #1: xy uv green
    { 210, 45, 1.0, 0.0, 0.0, 1.0, 0.0, 1.0 },  -- #2: xy uv blue
    { 200, 235, 0.5, 1.0, 0.0, 0.0, 1.0, 1.0 }, -- #3: xy uv red
    { 120, 280, 0.5, 1.0, 1.0, 0.0, 1.0, 1.0 }, -- #4: xy uv magenta
}

local mesh = love.graphics.newMesh(tri, 
    "fan",
    "dynamic"
)
]]

-- now:
local tri = {
    { 45, 170, 0.0, 0.0, 1.0, 0.0, 0.0, 1.0 },  -- #1: xy uv green
    { 210, 45, 1.0, 0.0, 0.0, 1.0, 0.0, 1.0 },  -- #2: xy uv blue
    { 200, 235, 0.5, 1.0, 0.0, 0.0, 1.0, 1.0 }, -- #3: xy uv red
    
    -- duplicate edge: 1 -> 3
    { 45, 170, 0.0, 0.0, 1.0, 0.0, 0.0, 1.0 },  -- #1: xy uv green
    { 200, 235, 0.5, 1.0, 0.0, 0.0, 1.0, 1.0 }, -- #3: xy uv red
    { 120, 280, 0.5, 1.0, 1.0, 0.0, 1.0, 1.0 }, -- #4: xy uv magenta
} 

-- allocate and draw as before
local mesh = love.graphics.newMesh(tri, "triangles", "dynamic")

love.draw = function()
    love.graphics.setColor(1, 1, 1, 1)
    love.graphics.setWireframe(true)
    love.graphics.draw(mesh)
    love.graphics.setWireframe(false)
end
```

![](/assets/img/meshes/quad_hello_world_wireframe.png)

Where we changed the second argument of `newMesh` from `strip` to  `triangles`. We see that duplicating the shared edge manually like this results in the exact same shape as the mesh whose vertex data contained four vertices. What are `strip` and `triangles` exactly?

## 1.2 Mesh Draw Modes

The second argument of `love.graphics.newMesh` is called the **draw mode**. It is a string that is a value of the enum [`love.MeshDrawMode`](https://love2d.org/wiki/MeshDrawMode). The draw mode determines how the list of vertices in the vertex data is converted into triangles for drawing by the GPU. 

> [!TIP]
> A computer has two types of memory, **memory on the CPU is called RAM**, memory on the graphics card is calld **VRAM**. We will als use the terms *CPU-side* to refer to anything in RAM, and *GPU-side*  to refer to anything in VRAM

It's best to visualize this process of how the tris are construct GPU-side visually. To do this, we meet our two new example meshes for this section:

A **heptagon**, a 7-sided regular polygon:

![](/assets/img/meshes/draw_mode_heptagon_base.png)

And a **ribbon**, a connected strip of tris:

![](/assets/img/meshes/draw_mode_ribbon_base.png)

We first take a moment to analyze the vertex layout of these, as they are quite complex shapes.

Firstly, the **heptagon** has 8 vertices, 1 center vertex and 7 for each of it's corners. The **center vertex is the first vertex**, the outer vertices are ordered clockwise.

Secondly, the **ribbon** also has 8 vertices. The First vertex is at the bottom left, the second is on the opposite site, then we move one segment up, with the 3rd vertex being on the left, 4th on the right, 5th on left, and so on.

Disregarding the math to generate the vertex positions in the vertex data for these shapes, we'll only look at the `newMesh` calls. The vertex data for the heptagon is called `heptagonData`, the ribbon's vertex data is  `ribbonData`:

```lua
-- send heptagon vertex data from CPU to GPU
local heptagonMesh = love.graphics.newMesh(
    heptagonData, -- vertex data
    "fan",        -- draw mode
    "dynamic"     -- (explained later)
)

-- send ribbon vertex data to the GPU
local ribbonMesh = love.graphics.newMesh(
    ribbonData,   -- vertex data
    "strip",      -- draw mode
    "dynamic"
)
```

We see that, unlike before, the second argument is `"fan"` for the heptagon, and `"strip"` for the ribbon. What do these mean, how are they different from `"triangles"`?

When the GPU constructs the faces from the vertex data, it first constructs a list of indices, starting at `1`. It then goes through these indices in order and **groups 3 consecutive indices as a tri**. For example, given indices `i1, i2, i3, ..., k`, where `k` is the number of vertices of the mesh, the following list is grouped like so:

```lua
-- vertex indices
{ i1, i2, i3, i4, i5, i6, i7, ..., k-2, k-1, k }

-- triangle list
{ i1, i2, i3 }, 
{ i4, i5, i6}, 
..., 
{ k-2, k-1, k }
```
Where each table `{ a, b, c }` points to 3 different entries in the vertex data at position `a`, `b`, `c`. For exmaple, f the vertex at index `2` has the following values for its attributes:

```lua
local vertexData = {
    {
        -- (...) vertex #1
    },
    
    {
        -- vertex #2
        210, 320, 0, 0, 1, 1, 1, 1 -- xy uv rgba
    },
    
    {
        -- (...) vertex #3
    },
    
    -- ...
    
    {
        -- (...) vertex #k
    }
}
```

Then any tri that has `2` as one of its vertex indices will read the properties from vertex #2, for example, the position of the first vertex of that tris edge will be `210, 320`.

This is the intuitive thing to do, take each triplet of indices, and call it a tri. This is exactly how the draw mode `"triangles"` works internally.

Other draw mode have a different scheme of how to group vertex indices for tri construction. Draw mode `fan`, the draw mode used for our heptagon, groups indices as follows:

```lua
-- vertex indices
{ i1, i2, i3, i4, i5, i6, i7, ..., k-2, k-1, k }

-- triangle list
{ i1, i2, i3 }, 
{ i1, i3, i4 }, 
{ i1, i5, i6 }, 
... 
{ i1, k-2, k-1}, 
{ i1, k-1, k }
```

We see that each triangle is constructed as follows: the first vertex index **is always the first vertex in the vertex data**. The other two vertices of the current tri are taken consecutively from the list `2, 3, ..., k`.

Looking at our heptagon again, we can actually see grouping scheme clearly on the right `setWireframe(true)` view:

![](/assets/img/meshes/draw_mode_heptagon_base.png)

Each triangle shares the first vertex (`i1`). The first tri is `1, 2, 3`, the second is `1, 3, 4`, the third is `1, 4, 5`, `1, 5, 6`, `1, 6, 7`, etc. The first vertex index is always `1`, the others are taken consecutively. It may be instructive to trace out these triangles in the image above, `"fan"` is quite intuitive once we understand how it works: all tris share the first vertex.

Turning our attention to our ribbon now, it was created using draw mode `"strip"`, which groups indices as follows:

```lua
-- vertex indices
{ i1, i2, i3, i4, i5, i6, i7, ..., k-2, k-1, k } 

-- triangle list
{ i1, i2, i3 }, 
{ i2, i3, i4 }, 
{ i3, i4, i5 }, 
... 
{ k-3, k-2, k-1}, 
{ k-2, k-1, k }
```

We see that now, instead of each tri sharing index `1`, a triangle shares the **last two vertices of the triangle before it**. Looking at our ribbon mesh again:

![](/assets/img/meshes/draw_mode_ribbon_base.png)

We see the tris are `1, 2, 3`, `2, 3, 4`, `3, 4, 5`, `4, 5, 6`, etc. The last two indices of each tri are the first two indices of the next tri.

Now that we know what list of tris the three draw modes, `"triangles"`, `"fan"`, and `"strip"` will result in, we realize that even for the same vertex data, the number and shape of faces will be different depending on the draw mode.

Draw mode `"strip"` also explains why the mesh above had two valid faces, despite the vertex data only containing 4 vertices:


```lua
-- from section 1.1:
local mesh = love.graphics.newMesh(tri, "strip", "dynamic")
```
![](/assets/img/meshes/quad_hello_world.png)


The draw mode was `"strip"`. Therefore, the tris are `1, 2, 3` and `2, 3, 4`, because there are only 4 vertices. This results resulting in exactly two tris.

Just for fun, what happens if use the wrong draw mode? We'll try all of them on our heptagon and ribbon data:

```lua
local heptagonMesh = love.graphics.newMesh(
    heptagonData, 
    "strip",  -- was: "fan"
    "dynamic"
)
```

![](/assets/img/meshes/draw_mode_heptagon_strip.png)

```lua
local heptagonMesh = love.graphics.newMesh(
    heptagonData, 
    "triangles", -- was: "fan"
    "dynamic"
)
```

![](/assets/img/meshes/draw_mode_heptagon_triangles.png)

```lua
local ribbonMesh = love.graphics.newMesh(
    ribbonData, -- vertex data
    "fan",      -- was: "strip"
    "dynamic"
)
```

![](/assets/img/meshes/draw_mode_ribbon_fan.png)

```lua
local ribbonMesh = love.graphics.newMesh(
    ribbonData,  -- vertex data
    "triangles", -- was: "strip"
    "dynamic"
)
```

![](/assets/img/meshes/draw_mode_ribbon_triangles.png)

Our meshes get all corrupted, but no errors are thrown.

Let's try to understad *how* the mesh was corrupted exactly. We consider this last image, drawing the ribbon with `"triangles"` instead of `"strip"`:

![](/assets/img/meshes/draw_mode_ribbon_triangles.png)

Instead of the full ribbon, we only see two triangles. Why is this? We recall that `"triangles"` just goes through the list of vertices and groups them by three to create a tri. Our ribbon vertex data has 8 vertices, so the first triangle will `1, 2, 3`, the second will be `4, 5, 6`, and the last one will be `7, 8, ... 9`? Our ribbon only has 8 vertices, so we cannot construct the last triangle using this draw mode, we only have two vertices left. The GPU handles this elegantly and without any errors, it simply does not draw the last triangle.

This corrupted mesh also shows a part of a meshes definition: **tris do not need to be connected**. As long as the draw mode is `"triangle"`, we can create a single mesh that is made up out of completely separate, far away triangles. Because draw mode `"fan"` and `"strip"` share vertices between two consecutive tris, all triangles will be connected in some way there.

## 1.3 Index Buffer / Vertex map

In our exploration of draw modes, we have used vertex indices `i1, i2, i3` instead of the corresponding vertex coordinates `x1, y1, x2, y2, x3, y3` directly. The main reason the GPU and thus LÖVE deals in *indices* is because of something called an **index buffer** (also called a **vertex map**). All meshes have this buffer internally, it is a table of the form

```lua
{ ia, ib, ic, id, ..., ik }
```

Where `ia`, `ib`, etc. are vertex indices, meaning integers between `1` and `k`, where `k` is the number of vertices in the vertex data. We note this difference: `k` is the size of the vertex data table, **not the number of vertices** our mesh seems to exhibit when drawn to the screen. 

We use `ia`, `ib` here instead of `i1`, `i2` because the index buffer can crucially be **unordered**. For example the following is a perfectly valid index buffer, as long as the vertex data has 4 or more vertices:

```lua
{ 1, 3, 2, 3, 1, 4, 4, 2, 3 }
```

Not only are the elements of he index buffer unordered, but **vertex indices are allowed to be duplicated or omitted**. When the GPU constructs the tris using the draw mode, it goes through each index in the index buffer and groups them as tris as descibed - no matter the order or number of indices. This allows us to use certain entries of the vertex data multiple times, or not use certain vertices at all. As long as each index in the index buffer is between `1` and `k`, the mesh will be drawn as expected. 

Duplicating vertex indices in the index buffer also has the advantage of having lower memory usage, both CPU- and GPU-side. [Before]() we duplicated an edge in the vertex data to create the 2-tri, 4-vertex polygon. 

```lua
local dataFour = {
    { 45, 170, 0.0, 0.0, 1.0, 0.0, 0.0, 1.0 },  -- #1: xy uv green
    { 210, 45, 1.0, 0.0, 0.0, 1.0, 0.0, 1.0 },  -- #2: xy uv blue
    { 200, 235, 0.5, 1.0, 0.0, 0.0, 1.0, 1.0 }, -- #3: xy uv red
    { 120, 280, 0.5, 1.0, 1.0, 0.0, 1.0, 1.0 }, -- #4: xy uv magenta
}

local dataSix = {
    { 45, 170, 0.0, 0.0, 1.0, 0.0, 0.0, 1.0 },  -- #1: xy uv green
    { 210, 45, 1.0, 0.0, 0.0, 1.0, 0.0, 1.0 },  -- #2: xy uv blue
    { 200, 235, 0.5, 1.0, 0.0, 0.0, 1.0, 1.0 }, -- #3: xy uv red
    
    -- duplicate edge: 1 -> 3
    { 45, 170, 0.0, 0.0, 1.0, 0.0, 0.0, 1.0 },  -- #1: xy uv green
    { 200, 235, 0.5, 1.0, 0.0, 0.0, 1.0, 1.0 }, -- #3: xy uv red
    { 120, 280, 0.5, 1.0, 1.0, 0.0, 1.0, 1.0 }, -- #4: xy uv magenta
}
```

Instead of duplicated the vertices in the vertex data, we should instead duplicate the vertex indices in the index buffer. This saves one third of the memory, which is irrelevant for a mesh this small but can be highly relevant once we have meshes with tens of thousands of faces.

### 1.3.1 `setVertexMap`

Modifying the index buffer interacts with draw mode as described. This allows us to create well-formed meshes from almost any vertex data. 

Considering our ribbon again, this was draw mode `"strip"`

![](/assets/img/meshes/draw_mode_ribbon_base.png)

And this was draw mode `"triangles"`

![](/assets/img/meshes/draw_mode_ribbon_triangles.png)

We can actually make `"triangles"` work perfectly here by modifying the index buffer, and leaving the vertex data unchanged. 

If not manually specified, a mesh with `k` vertices will have the index buffer `{ 1, 2, 3, ..., k }`. This is why we see the behavior of `"triangles"` corrupting the ribbon. It groups them into faces as discussed: the index buffer is `{ 1, 2, 3, 4, 5, 6, 7, 8 }`, the draw mode is `"triangles"`, therefore the tris become `1, 2, 3`, `4, 5, 6`, and `7, 8, ...`, with the latter being malformed and thus omitted.

We can manually construct an index buffer that creates a valid list of tris after the mesh is created. To do this, use `setVertexMap` which takes a table or vararg of integers:

```lua
ribbonMesh = love.graphics.newMesh(ribbonData, 
    "triangles", -- now "triangles" instead of "strip"
    "dynamic"
)

-- manually override the index buffer
ribbonMesh:setVertexMap(
    1, 2, 3, -- triangle #1,
    2, 3, 4, -- triangle #2,
    3, 4, 5, -- ...
    4, 5, 6,
    5, 6, 7,
    6, 7, 8  -- triangle #6
)
```

> [!CAUTION]
> **`setVertexMap` is only available if the draw mode is `"triangles"`**. If we try to `setVertexMap` on a mesh with another draw mode, an error will be raised

Going through the new list of indices again, we see that it has the form `1, 2, 3`, then `2, 3, 4`, then `3, 4, 5`, etc. In our index buffer, each tri shares the last two vertex indices of the tri before it. This is exactly how draw mode `"strip"` works!

Drawing this mesh now:

![](/assets/img/meshes/draw_mode_ribbon_base.png)

Everything looks exactly like we want it to. By manually choosing the correct vertex map, we can draw seemingly ill-fitting vertex data correctly. Note that this does not incur any performance overhead. The vertex data of `ribbonMesh` stays unaltered, both CPU- and GPU-side.

In summary, we should choose the draw mode to fit our mesh, but if can't make it fit - for example if we don't have control over the meshes vertex data because we are loading it from a file - we should make the draw mode `"triangles"`, then `setVertexMap` manually. An example of this can be seen in the [triangulation of a polygon using math.triangulate above in section 0.3](#03-creating-a-polygon-mesh-using-triangulation).

## 1.5 Vertex Attribute Interpolation

We recall that we wondered in section 1.1 as to why the mesh had a smooth range of colors. When drawing shapes using `love.graphics.circle`, or `love.graphics.polygon`, these shapes only ever have a single color controlled by `love.graphics.setColor`. 

We note that in our four-vertex mesh example:

```lua
local tri = {
    { 45, 170, 0.0, 0.0, 1.0, 0.0, 0.0, 1.0 },  -- #1: xy uv green
    { 210, 45, 1.0, 0.0, 0.0, 1.0, 0.0, 1.0 },  -- #2: xy uv blue
    { 200, 235, 0.5, 1.0, 0.0, 0.0, 1.0, 1.0 }, -- #3: xy uv red
    { 120, 280, 0.5, 1.0, 1.0, 0.0, 1.0, 1.0 }, -- #4: xy uv magenta
}
```

![](/assets/img/meshes/quad_hello_world.png)

The vertices have the colors of red, blue, green, and magenta respectively. We can see these in the four corners of the polygon.

When a mesh is drawn, the GPU automatically **interpolates** between all vertex properties. This means it takes the property of three neighboring vertices (where neighboring means they are part of the same tri, according to the draw mode), and **mixes** between them using [barycentric distances](https://en.wikipedia.org/wiki/Barycentric_coordinate_system). A barycentric distance is the distance from the current point to all three vertices of the triangle that point is in. This **interpolation within the same tri happens for all vertex attributes**, including position and color.

### 1.5.1 Creating a Gradient

For example:

```lua
local x, y, w, h = 50, 50, 200, 200
local light = function() return 0.1, 0.1, 0.1, 1 end
local dark  = function() return 0.5, 0.5, 0.5, 1  end
local vertices = {
    --    x,     y,   u, v,   rgba
    { x + 0, y + 0,   0, 0,   light() }, -- #1
    { x + w, y + 0,   1, 0,    dark() }, -- #2
    { x + w, y + h,   1, 1,    dark() }, -- #3
    { x + 0, y + h,   0, 1,   light() }, -- #4
}

local rectangle = love.graphics.newMesh(vertices, "fan", "dynamic")
```

Here, we created a rectangular mesh made up of two tris (four vertices and draw mode `"fan"`). The Vertex indices are shown here:

![](/assets/img/meshes/interpolation_gray.png)

We see that the left side of the rectangle is `dark()` and the right side is `light()`. Turning our attention to just the `1 -> 2` edge for now (the line going from vertex `1` to vertex `2`, at the top of the rectangle), we see that vertex `1` is dark, and vertex `2` is light. Therefore, the GPU will blend from the dark gray of `1` to the light gray `2` going along that edge. We confirm this is the behavior for that row of pixels. This is not only done for each edge, but for every pixel in the mesh. Which vertices are chosen to be interpolated determines which face of the mesh that pixel is in.

Giving each vertex it's own color:

```lua
local x, y, w, h = 50, 50, 200, 200
local green   = function() return 0, 1, 0, 1 end
local cyan    = function() return 0, 1, 1, 1 end
local magenta = function() return 1, 0, 1, 1 end
local yellow  = function() return 1, 1, 0, 1 end
local vertices = {
    --    x,     y,   u, v,   r, g, b, a
    { x + 0, y + 0,   0, 0,   green() },   -- #1
    { x + w, y + 0,   1, 0,   cyan() },    -- #2
    { x + w, y + h,   1, 1,   magenta() }, -- #3
    { x + 0, y + h,   0, 1,   yellow() },  -- #4
}
local rectangle = love.graphics.newMesh(vertices, "fan", "dynamic")
```

we now see:

![](/assets/img/meshes/interpolation_color.png)

Readers are encourage to carefully trace how each color is blended between two neighboring edges as an exercise.

We also note some artifacting along the `1 -> 3` edge. This is a result of the fact that vertex attributes are only blendend **within the same tri**. For each point in the triangle, the GPU takes the distance to only those three vertices, then blends each vertex property (color in this case) by taking a linear weighted average based on the distances. It does not access any vertices other than these three, it has not concept of an entire mesh - only tris. This can cause artifacting like seen above can appear.

We can fix this by manually inserting an additional vertex:

```lua
local x, y, w, h = 50, 50, 200, 200
local green   = function() return 0, 1, 0, 1 end
local cyan    = function() return 0, 1, 1, 1 end
local magenta = function() return 1, 0, 1, 1 end
local yellow  = function() return 1, 1, 0, 1 end
local center  = function() return 0.5, 0.75, 0.5, 1 end
local vertexData = {
    --    x,     y,     u,   v,   r, g, b, a
    { x + 0, y + 0,     0,   0,   green() },   -- #1
    { x + w, y + 0,     1,   0,   cyan() },    -- #2
    { x + w, y + h,     1,   1,   magenta() }, -- #3
    { x + 0, y + h,     0,   1,   yellow() },  -- #4
}

-- add new center vertex
table.insert(vertexData,
    { x + w / 2, y + h / 2, 0.5, 0.5, center() } -- #5
)

local rectangle = love.graphics.newMesh(vertexData, "triangles", "dynamic")
rectangle:setVertexMap(
    1, 2, 5, -- top triangle
    2, 3, 5, -- right triangle
    3, 4, 5, -- bottom triangle
    1, 4, 5  -- left triangle
)
```

![](/assets/img/meshes/interpolation_color_fixed.png)

Where the `setVertexMap` may be easier to follow if we label each vertex and only draw the edges using `setWireframe`:

![](/assets/img/meshes/interpolation_color_fixed_wireframe.png)

We see the artifacting is almost gone now, it is a smooth mix of four colors.

## 1.6 Texture Coordinates

While each tri interpolates between three colors, **all vertex attributes are interpolated**. This includes *position*. The interpolation of all three vertex positions is actually how the GPU determines where to place each pixel a face. 

Along with position and color, interpolation happens for third default vertex attribute - one we have so far negleted: **texture coordinates**, also called `uv`.

A texture coordinate is a two-component vector in `[0, 1]`. Being in `[0, 1]` means that `u` is between `0` and `1` (inclusive), and `v` is between `0` and `1` (inclusive). Choosing a value outside of this range will not result in an error, instead it will result in deterministc behavior as described below.

> [! Tip]
> A **determinstic program** is a program whose exact behavior we can predict before the program ever started, given only its code. An example of non-deterministic behavior is reading the value at a fixed RAM address. We currently have no control over what value could be at adress `0xDEADBEEF`. It could be RAM allocated by a completely separate, non-LÖVE process. Because we cannot predict the outcome, reading that RAM address results in non-determinstic behavior. In contrast, we can always predict the outcome of choosing specific texture coordinates for a mesh

To see how texture coordinates work and why they matter, we first need a **textured mesh**. Each mesh can have up to one texture, but meshes can also have no texture. We create such an untextured mesh first:

```lua
-- create the (untextured) mesh
local vertexData = {
    --    x,     y,     u,   v,   r, g, b, a
    { x + 0, y + 0,     0,   0,   green() },   -- (1)
    { x + w, y + 0,     1,   0,   cyan() },    -- (2)
    { x + w, y + h,     1,   1,   magenta() }, -- (3)
    { x + 0, y + h,     0,   1,   yellow() },  -- (4)
}
local rectangle = love.graphics.newMesh(vertexData, "fan", "dynamic")
```

![](/assets/img/meshes/texture_coords_untextured.png)

Where the triangulation and draw mode are exactly the same as the 5-vertex rectangle from section 1.5. It only has four vertices, the center vertex intended to reduce artifacting is no longer present.

We see that the top left vertex (1) has `uv` of `0, 0`, the top right vertex (2) has uv `1, 0`, the bottom right vertex (3) has uv `1, 1`, and the bottom left vertex (4) has uv `0, 1`.

Texture coordinates work the same as pixel coordinates in terms of orientation. The `u` axis is negative going left, positive going right. And the `v` axis is **positive going down**, and negative going up. This is just like how the  `x` axis is left to right, and the `y` axis is top to bottom.

It may be complicated to write it out like this, but it's actually quite natural. It works exactly like pixel coordinates, except instead of values like `240px, -15px`, we have values like `0.2, 1`.

### 1.5.1 `setTexture`

Drawing the above mesh again - untextured for now - we see:

![](/assets/img/meshes/texture_coords_untextured.png)

We can now associate a texture with this mesh using `setTexture`:

```lua
-- load image from disk
local texture = love.graphics.newImage("toast.png")

-- set the meshes texture to this image
rectangle:setTexture(texture)
```

Drawing again:

```lua
love.draw = function()
    love.graphics.setColor(1, 1, 1, 1)
    love.graphics.draw(rectangle)
end
```

![](/assets/img/meshes/texture_coords_color.png)

Before we diagnose what happened here, let's first modify `vertexData`, such that each vertex is pure white `rgba = { 1, 1, 1, 1 }`. It was four different colors before.

```lua
local white = function() return 1, 1, 1, 1 end
local vertexData = {
    --    x,      y,    u,   v,   r, g, b, a
    { x + 0, y + 0,     0,   0,   white() },  -- was: green()
    { x + w, y + 0,     1,   0,   white() },  -- was: cyan()
    { x + w, y + h,     1,   1,   white() },  -- was: magenta()
    { x + 0, y + h,     0,   1,   white() },  -- was: yellow()
}
```

Drawing again:

```lua
love.draw = function()
    love.graphics.setColor(1, 1, 1, 1)
    love.graphics.draw(rectangle)
end
```

![](/assets/img/meshes/texture_coords_uncolored.png)

Now that we have a comparison, let's analyze. We first see that our texture is correctly displayed in terms of orientation. This is beceause our uv coordinates were set up correctly, with each corner being having a `u` and `v` of either `0` or `1`, depending on corner the vertex is. The four-vertex mesh above with those exact texture coordaintes is actually what `love.Image` uses internally when drawn, `love.Image` represents a GPU-side texture, but when we call `love.graphics.draw`, it creates a four-vertex like above, then draws that mesh with some additional optimization.

We notice that with the former, four-colored mesh, the color of the texture is all of. This is because we have no shader bound. In this state, each pixel in the mesh will take its interpolated vertex color, take its interpolate texture coordinate, look up the corresponding pixel in the texture, then multiply the vertices color with the textures pixels color. Again, for each pixel in each tri, the GPU interpolates *all* properties. The texture pixels color is multiplied with the face pixels color.

If we want the texture to be displayed as-is we should choose a mesh with all-white vertices and make sure to `setColor(1, 1, 1, 1)` before drawing. This means all pixels in the texture will be multiplied by `1`, which results in no change.

As an exercise for `uv` usage, let's flip the `u` coordinates like so:

```lua
local vertexData = {
    --    x,      y,    u,   v,   r, g, b, a
    { x + 0, y + 0,     1,   0,   white() },  -- u was: 0, now 1
    { x + w, y + 0,     0,   0,   white() },  -- u was: 1, now 0
    { x + w, y + h,     0,   1,   white() },  -- u was: 1, now 0, 
    { x + 0, y + h,     1,   1,   white() },  -- u was: 0, now 1
}

love.draw = function()
    love.graphics.setColor(1, 1, 1, 1)
    -- no love.graphics.scale
    love.graphics.draw(rectangle)
end
```

What do we expect to see now?

![](/assets/img/meshes/texture_coords_u_flipped.png)

The image is mirrored along the x axis. This makes sense, `u` is the horizontal texture coordinate. Flipping the horizontal coordinate flips the image horizontally. We see that the actual position of the mesh has stayed unchanged, it is still at position `x, y`, only the texture is flipped - because we only modified the texture coordinates.

Let's restore the uv such that the top left of the mesh gets the top left of the texture, meaning no flipping at all, then we divide all `uv` by `2`. What will happen?

```lua
local vertexData = {
    --    x,     y,         u,       v,   r, g, b, a
    { x + 0, y + 0,     0 / 2,   0 / 2,   white() },
    { x + w, y + 0,     1 / 2,   0 / 2,   white() },
    { x + w, y + h,     1 / 2,   1 / 2,   white() },
    { x + 0, y + h,     0 / 2,   1 / 2,   white() },
}

local rectangle = love.graphics.newMesh(vertexData, "fan", "dynamic")

local texture = rt.Texture("assets/sprites/why.png"):get_native()
texture:setWrap("repeat")
rectangle:setTexture(texture)

love.draw = function()
    love.graphics.setColor(1, 1, 1, 1)
    love.graphics.draw(rectangle)
    
    -- also draw only mesh edges using  `setWireframe` here
end
```

![](/assets/img/meshes/texture_coords_uv_half.png)

Where we are displaying the meshes edges on top of the regular textured mesh for instructive purposes. The white lines have no relation ot the divided `uv` and are not part of the mesh.

Since `u` and `v` are now in `[0, 0.5]`, instead of `[0, 1]`, only the top half of the texture is shown. `0.5` is half of `1`, it therefore corresponds to a pixel at the center of the texture. Crucially, the rectangle is still the same size, it still interpolates position just like before. The top left corner is still `x, y`, and it still has a size of `w, h`. The stretching only affects the texture, not the size of the shape. This is because we only modified the texture coordinates. An effect like this is possible to achieve without meshes using `love.graphics.scale` and stencils, but meshes require us only to change two numbers.

Our uv stayed in `[0, 1]` at all times so far. What happens if we go outside that range?

### 1.5.2 Texture Wrap Mode and Meshes

We restore our regular `uv` orientation, then multiply the `uv` by `4.5`:

```lua
local vertexData = {
    --    x,      y,          u,         v,   r, g, b, a
    { x + 0, y + 0,     0 * 4.5,   0 * 4.5,   white() }, -- #1
    { x + w, y + 0,     1 * 4.5,   0 * 4.5,   white() }, -- #2
    { x + w, y + h,     1 * 4.5,   1 * 4.5,   white() }, -- #3
    { x + 0, y + h,     0 * 4.5,   1 * 4.5,   white() }, -- #4
}

-- create mesh
local rectangle = love.graphics.newMesh(vertexData, "fan", "dynamic")

-- load texture
local texture = rt.Texture("toast.png"):get_native()

-- set texture wrap mode
texture:setWrap("repeat")

-- associate texture with mesh
rectangle:setTexture(texture)

-- love.draw as before, including wireframe
```

We use a relatively rarely used function of `love.Texture` here: `setWrap`. This sets the textures **[wrap mode](https://love2d.org/wiki/WrapMode)** to `"repeat"`. The texture warp mode is specifically used to determine how meshes handle texture coordinates outside of `0, 1`. What will `"repeat"` look like?

![](/assets/img/meshes/texture_coords_repeat.png)

We see a repeated tiling of the texture.

Let's try to explain why this happens. When the GPU sees a texture coordinate outside of `[0, 1]`, for example a `u` of `2.5`, it will map `u` back to into `[0, 1]`. How this mapping works is decided by the textures `WrapMode`. For wrap mode `"repeat"`, the GPU will take `uv` modulo `1`. For `u = 2.5`, it does `2.5 % 1 = 0.5`. That vertex will get a `u` of `0.5` only when drawing. The vertex datas entry will stay as `2.5`. The GPU still looks into the same texture, it still interpolates all vertex attributes as before. The rectangle is still the same size on screen, it still has the same number of pixels, all that change is which pixel of the texture is chosen when interpolating the texture coordinate before lookup. 

With `"repeat"`, we get a natural, non-integer tiling, which would be quite hard to achieve without using meshes in LÖVE.

Along with the wrap mode, texture coordinates also interact with the `setFilter` property of textures. This is often called a textures **[scale mode](https://love2d.org/wiki/FilterMode)**. To better exhibit how this affects our mesh, we change the actual `xy` position entry in the vertex data of our rectangle mesh such that it will fill the entire screen. We then "zoom in" the texture coordinates by 50%. To achieve this zoom in, we add or subtracting `50% / 2 -> 0.25` like so:

```lua
local x, y, w, h = -- ..
local vertexData = {
    --    x,     y,      u,    v,   r, g, b, a
    { x + 0, y + 0,   0.25, 0.25,   1, 1, 1, 1 }, -- uv was (0, 0)
    { x + w, y + 0,   0.75, 0.25,   1, 1, 1, 1 }, -- uv was (1, 0)
    { x + w, y + h,   0.75, 0.75,   1, 1, 1, 1 }, -- uv was (1, 1)
    { x + 0, y + h,   0.25, 0.75,   1, 1, 1, 1 }, -- uv was (0, 1)
}
```

We then create two separate textures from the same image on disk. One with `setFilter` set to `"nearest"`, and one with `setFilter` set to `"linear"`:

```lua
local left = rt.Texture("toast.png"):get_native()
left:setFilter("nearest")
leftRectangle:setTexture(left)

local right = rt.Texture("toast.png"):get_native()
right:setFilter("linear")
rightRectangle:setTexture(right)

love.draw = function()
    -- draw left: nearest-neighbor filter
    love.graphics.clear()
    love.graphics.setColor(1, 1, 1, 1)
    leftRectangle:setTexture(left)
    love.graphics.draw(rectangle)

    -- draw right: linear filter
    love.graphics.push()
    love.graphics.translate( --..
    rightRectangle:setTexture(right)
    love.graphics.draw(rectangle)
    love.graphics.pop()
end
```

![](/assets/img/meshes/texture_coords_filter.png)

We see a quite noticeable difference in how the image scales, this is what the filter mode governs. `"nearest"` results in pixelated scaling, while `"linear"` results in linearly blurred scaling.

Again, the mesh can be any kind of shape, not just a rectangle, it can be any size, any color, the texture coordinates can be any number, yet the mesh will automatically correctly scale and display the texture. This does not affect performance, **the texture lookup has no impact on drawing performance**, only the number of vertices and face pixels on screen does.

The "zoom in" effect above could be achieved using functions such as `love.graphics.scale` and `love.graphics.skew` with quite involved math if we were limit ourselves to only using `love.Quad`, which can also display a texture rectangle. With Meshes, all the complicated zoom and scaling is a single line of code.

## 1.7 Vertex Attributes and Shaders

> [!NOTE]
> This section requires readers to know the basics of shaders. People completely unfamiliar with shaders should [skip to section 1.8](#18-replacing-vertex-data)

Whenever we use the shader entry points LÖVE provide, we have to use these functions:

```glsl
#ifdef VERTEX // vertex shader

vec4 position(mat4 transformProjection, vec4 vertexPosition) {
    // vertex shader
    return transformProject * vec4( // ..
}

#endif

#ifdef FRAGMENT // fragment shader

vec4 effect(vec4 vertexColor, sampler2D tex, vec2 vertexTextureCoords, vec2 fragmentPosition) {
    // fragment shader
    return vertexColor * texture(tex, vertexTextureCoords);
}

#endif
```

The inconspicuous variable naming may haven given it away here, but yes, the actual arguments of both the vertex shaders `position` and the fragment shaders `effect` are **directly related to meshes** - and meshes alone. Remember that everything is a mesh internally, applying a shader to a `love.Quad` or `love.graphics.polygon` is still applying a shader to a mesh.

### 1.7.1 `effect` and `position`

Firstly, the **vertex shader** is named that because it is a **shader run once for every vertex of a mesh**. When using LÖVEs `position` function, only one attribute of the vertex is processed by us: the vertices `xy` position. Other attributes are also processed automatically by LÖVE itself, outside of `position`. The argument `vec4 vertexPosition` of `position` has the following value

```glsl
vec4(VertexPosition.x, VertexPosition.y, 0, 1)
```

The  `xy` of this value is exactly our **not yet interpolated** vertex position, padded with `0` and `1` for mathematical purposes. For example, if a vertex has position `240, -12`, then `vertexPosition` will be `vec4(240, -12, 0, 1)`. This is what the vertex shader actually does, it takes the uninterpolated vertex position, transforms it in some way (for example multiplying with the `transformProject` from `love.graphics`, then forwards it to the fragment shader. After the vertex shader, but before the fragment shader, interpolation of all attributes happens.

In the fragment shader, the arguments of `effect` are **the now interpolated tri properties**:
+ `vertexColor`: interpolates `rgba` from each vertex of the tri this pixels is in the face of
+ `tex`: the image `setTexture` was given, or a 1x1 white dummy texture if the mesh has no associated texture
+ `vertexTextureCoords`: the interpolated `uv` from each vertex of the current tri being drawn, in uv
+ `fragmentPosition`: the interpolated  `xy` from each vertex of the current tri being drawn, in px

We see that three of the arguments directly correspond to our `xy`, `uv`, and `rgba`. 

Indeed, their values are exactly as expected. 

`fragmentPosition` is the pixels `xy` position, for example `280, 11`, where 0, 0 is top left of the window. 

`vertexTextureCoords`s is the now interpolated uv according to the tris vertices, for example `0.25, 0.8239`. 

`vertexColor` is the interpolated `rgba`, for example `1, 1, 1, 0.76`.

Note that `vertexColor` is already multiplied with the color from `love.graphics.setColor`, so we need to set that to `1, 1, 1, 1` if we want the pixels raw color according to the mesh.

### 1.7.2 Fragments

So what is a fragment anyway? 

When a tri is about to be drawn, the GPU determines where to place each pixel by first interpolating each of the three vertices positions. This location is where a **fragment** will be drawn. A fragment is a "pixel" of a tri, drawn at the interpolated vertex position, interpolated uv, and interpolated color. Note how at this point, before the fragment shader is run, a fragment does not yet know anything about a texture - it only interpolates the values from the vertex data. 

The work of looking up the correct pixel in the texture and multiplying it with the fragments color is done in the fragment shader:

```glsl
return vertexColor * texture(tex, vertexTextureCoords);
```

This snipp up the pixel of the texture at the position specified by the texture coordinates using the `texture` function, then multiplies it with the interpolated vertex color. The `vec4` returned by `pixel` is the fragments final color.

Indeed, if we change that line in `effect` to 

```glsl
return vertexColor;
```

The drawn mesh will have a single flat color according to `love.graphics.setColor` and the vertex color. The texture will be ignored.

While this chapter is not about shaders specifically, hopefully having learned about how everything is an underlying mesh, and how the vertex and fragment shaders use that mesh, connected some wire. Increasing our understanding of shaders, meshes, and rendering in general can unlock powerful techniques not achievable otherwise.

## 1.8 Replacing Vertex Data: `setVertices`

So far, we have only created a mesh - then drawn it. We never modified a mesh vertex data after creation. 
While we could create a new mesh everytime something about the vertex data changes, this is very slow. It incurs significant overhead and **creating a new mesh every frame is almost never advisable**. 
Instead, we should create a mesh *once*, then **upload vertex data** to it directly. Uploading, here, refers to the process of transferring memory from RAM to VRAM. This upload happens once on `newMesh`, too. To change vertex data afterwards, we should use `setVertices`. 

This function is provided a table and replaces the entire vertex data of a mesh all at once. The table needs to be formatted just like it would need to be with `newMesh`.
Uploading like this is much faster than creating a new mesh. The main reason for this is that a mesh is more than just it's vertex data. We've seen it also having an index buffer, a draw mode, and a fourth property we have so far controlled but neglected to explain. The fourh argument of `newMesh` such as `"dynamic"`. This is called a **graphics buffer usage mode**.

## 1.8.1 Graphis Buffer Usage

The last argument of `newMesh` needs to be a string that is a value of the enum [`BufferDataUsage`](<https://love2d.org/wiki/BufferDataUsage>):

```lua
local mesh = love.graphics.newMesh(vertexData, 
    "triangles",
    "static" -- graphics buffer usage !
)
```

This value decides how memory is prepared and uploaded to the GPU. It has three possible values, which roughly correspond to the following use cases:

+ `"static"`: vertex data will never be replaced
+ `"dynamic"`: vertex data may be replaced sometimes
+ `"stream"`: vertex data will be replaced every frame

Setting `BufferDataUsage` does not change the meshes visuals or anything about its vertex data or index buffer. It is an optimization technique that increase performance when drawing, or when replacing vertex data using `setVertices`. This barely matter for small meshes, but once we start uploading 100k vertices every frame, the difference between `"stream"` and `"static"` is highly relevant. `"static"` is fastest to draw and slowest to upload, `"stream"` is fastest to upload but incurs som additional RAM overhead, `"dynamic"` is a happy medium between the two.

## 1.8.2 `setVertices` Example

To demonstrate the capabilities of replacing vertex data, we consider the following example:

```lua
love.window.setMode(300, 300)

-- create a texture, 5-vertex rectangle mesh
local x, y, w, h = 50, 50, 200, 200
local cx, cy = x + w / 2, y + h / 2 -- center of mesh
local vertexData = {
    --    x,     y,  u,   v,    r, g, b, a
    { x + 0, y + 0,  0,   0,    1, 1, 1, 1 }, -- #1
    { x + w, y + 0,  1,   0,    1, 1, 1, 1 }, -- #2
    { x + w, y + h,  1,   1,    1, 1, 1, 1 }, -- #3
    { x + 0, y + h,  0,   1,    1, 1, 1, 1 }, -- #4
    { cx,    cy,     0.5, 0.5,  1, 1, 1, 1 }, -- #5 (center)
}

local rectangle = love.graphics.newMesh(vertexData,
    "triangles",
    "stream" -- stream, will be replaced every frame
)
rectangle:setVertexMap( -- triangulation, cf. section 1.2
    1, 2, 5,
    2, 3, 5,
    3, 4, 5,
    4, 1, 5
)

-- set texture
rectangle:setTexture(love.graphics.newImage("toast.png"))

-- update routine: modify mesh data every frame
love.update = function(delta)
    local maxOffset = 40 -- maximum offset from center, in px

    -- vertex #5 (center), vertex property #1: x
    -- move cx by a random amount
    vertexData[5][1] = cx + maxOffset * ((love.math.perlinNoise(love.timer.getTime()) * 2) - 1)

    -- vertex #5 (center), vertex property #2: y
    -- move cy by a random amount
    vertexData[5][2] = cy + maxOffset * ((love.math.perlinNoise(-1 * love.timer.getTime()) * 2) - 1)

    -- upload vertices
    rectangle:setVertices(vertexData)
end

-- draw as before
love.draw = function()
    love.graphics.clear()
    love.graphics.setColor(1, 1, 1, 1)
    love.graphics.draw(rectangle)
end
```

Readers are encourage to located a texture such as `toast.png` and past this code as their `main.lua` to see the animation on screen.

`love.math.perlinNoise` (requires LÖVE 12.0 or newer) is used to randomly move the center vertex in a circlular area around the original center `cx, cy`. What will this look like? Nothing else about the mesh is changed, only the `x` of the center vertex is moved slightly, it's `uv` stays constant. 

Taking a screenshot of the animation, it looks like this:

![](/assets/img/meshes/mesh_upload_example.png)

While some consider this result to be already highly useful, distorting textures using meshes is a powerful technique that has many applications (cf. [Section 1.0.2](#102-texture-deformation)). 
We have free control over the position and texture coordinate of each vertex, we can change it smoothly by any amount (unlike with `love.Quad`, which only allows integers), and we can change each vertex individually - every frame. This allows for unprecendented freedom in terms of what can be drawn. Using this, with enough effort, we are able to draw literally any kind of animation - no exceptions. For example, we could swap the texture coordinates each frame such that they draw a new frame from a texture atlas, allowing us to rebuild `SpriteBatch` from scratch. 
Reasons like this is why meshes are not only the most powerful drawable in LÖVE, but the basis of all drawing in modern graphics. They allow us to draw literally anything that can be displayed on a computer screen.

`setVertices` will be instrumental in section 2.0 of this chapter, where we will use a mesh to store data for thousands of entities, and upload properties of those entities to the GPU every frame, thus changing the state of those entities

---

# 2. Advanced Usage: Geometry Instancing

> [!CAUTION]
> Unlike before, the rest of this chapter is intended for users already familiar with advanced concepts in graphics programming, including glsl shaders, vertex buffers, storage buffers, texel buffers, and C-memory. It will be assumed that readers are somewhat familiar with these concepts for the rest of this chapter.

The following fully functional code example shows various advanced techniques, which are listed here. We can navigate to the relevant code section using the section identifiers, e.g. for section `X.4`, a code comment `-- (X.4) ...` will be present in the code example below. Readers are encouraged to `ctrl+f` for a specific section at a time.

#### 2.1 Geometry Instancing

Geometry instancing is a technique that allows us to draw `n` copies of a mesh using a single draw command. This is as opposed to going through a for-loop and calling `love.graphics.draw` `n` times. The latter is much, much slower, as every individual `draw` call incurs some overhead.

Because we can only draw the same mesh `n` time, we need of these **instances** to have some kind of data associated with it. This **per-instance data** should modify some drawable property on a per-instance basis Without this, instanced drawing will just draw the same mesh, at the same position, `n` times.

To perform an instanced draw, we use `love.graphics.drawInstanced(mesh, n)`. Before this command, we need to **bind a custom vertex shader** which uses some kind of GPU-side data type to store the per-instance attributes. In the code example below, a circle mesh is used for drawing, this is what we will draw `instanceCount`-many copies of.
Each instance gets it's own position (`InstanceOffset`), scale (`InstanceScale`) and hue (`InstanceHue`). 
To properly scale and offset the mesh, it is constructed normalized and at origin. This means the center vertex is at `0, 0` and that it's radius is `1`. This way, we can simply offset the circle by `InstanceOffset`, then scale it by `InstanceScale`. This then draw a circle of radius `InstanceScale` at position `InstanceOffset`, giving each instance the appearance of a proper circle mesh.

To access the per-instance attributes, the code below shows how to use vertex buffers, storage buffers, and texel buffers to store this data on the GPU. It then queries these kind of GPU-side buffers in the vertex shader, where `BUFFER_MODE` decides which type of buffer to use. 
Each of these is represented by a Lua-side object: a `love.Mesh` for vertex buffers, or a `love.GraphicsBuffer` for storage and texel buffers. Each type of buffer can be filled with data from either a regular Lua table or C-data in the form of `love.ByteData`. We use `DATA_MODE` to switch between using Lua-data, C-data with FFI, or C-data with `ByteData`.
Since modifying `ByteData` using it's method, such as `setFloat` and `setUInt32`, is quite slow, the below example also shows how to modify the C-data directly using LuaJITs FFI library. In terms of performance, this is optimal, if uploading or modifying the CPU-side data is a bottleneck, we should choose either a vertex or storage buffer and keep the data as C-memory. This saves a lot of internal conversion that happens during buffer upload, and can lead to a multiple-fold speedup.

Using instanced drawing with this architecture, we match or exceed performance of `love.graphics.circle` or `SpriteBatch`. Instanced drawing with the correct type of buffer using C-data is the most performant way to draw or send data to the GPU in LÖVE. This technique is used in the example below to draw *hundreds of thousands* of meshes at 60fps.

Section relevant to geometry instancing in general include:

+ `(A.1)` specifying a custom vertex attribute format for the drawn mesh
+ `(A.2)` creating the drawable mesh
+ `(A.3)` performing the  `drawInstanced` command
+ `(A.4)` binding a shader before instanced drawing
+ `(A.5)` applying the per-instance mesh transform in the vertex shader
+ `(A.6)` declaring the per-instance data buffer format
+ `(A.7)` initializing the per-instance data
+ `(A.8)` creating the custom shader for use with  `drawInstanced`

Sections relevant to specific types of buffers are listed in the following sections.

#### 2.2 Vertex Buffers

Vertex buffers are represented by a `love.Mesh` CPU-side, which takes as it's first argument the buffer format (cf. section (`A.0`) below. 
In LÖVE 12.0, we can use `getVertexBuffer` to access the raw `love.GraphicsBuffer` of the underlying mesh. 
Similarly, to access the index buffer of a mesh, we can use `getIndexBuffer`. Replacing the vertex buffer is still usually done using `setVertices`, while replacing the index buffer is done using `setVertexMap`.

The code example includes the following sections relevant to vertex buffers:

+ `(B.1)` declaring a custom vertex buffer format
+ `(B.2)` initializing a vertex buffer from a Lua table
+ `(B.3)` initializing a vertex buffer from `love.ByteData`
+ `(B.4)` creating the vertex buffer object
+ `(B.5)` initializing a vertex buffer from FFI memory
+ `(B.6)` attaching attributes from one mesh to another, per-vertex or per-instance
+ `(B.7)` accessing attributes of vertex buffer in the vertex shader
+ `(B.8)` uploading new data to the GPU-side vertex buffer
+ `(B.9)` instantiating the `love.GraphicsBuffer` object

#### 2.3 **Storage Buffers**

Storage buffers are initialized CPU-side using `newBuffer`, which takes a buffer format as it's first argument. This is similar to how `newMesh` is used to create a vertex buffer. The second argument of `setBuffer` is the actual data, which needs to be formatted in congruence with the buffer format. We call `love.graphics.readback` to download the data of a graphics buffer, and `Buffer:setArrayData` to replace the entire buffers data at once, cf section `(C.9)` below.

Sections relevant to storage buffers include:

+ `(C.1)` declaring a custom storage buffer format
+ `(C.2)` initializing the storage buffer from a Lua table
+ `(C.3)` initializing the storage buffer from `love.ByteData`
+ `(C.4)` initializing the storage buffer from FFI memory
+ `(C.5)` declaring a matching buffer format in glsl
+ `(C.6)` declaring the storage buffer uniform
+ `(C.7)` using `gl_IntanceID` to access per-instance attributes of the storage buffer in the vertex shader
+ `(C.8)` binding a storage buffer to a glsl `buffer` declaration
+ `(C.9)` uploading new data to the GPU-side storage buffer
+ `(C.10)` instantiating the `love.GraphicsBuffer` object

#### 2.4 **Texel Buffers**

Texel buffers, called `samplerBuffer` in a glsl shader, can hold up to 4 float attributes (or `int`s for `isamplerBuffer`, `uint`s for `usamplerBuffer`). They are also represented CPU-side by a `love.GraphicsBuffer`, just like storage buffers. When creating the buffer format for a texel buffer, we need to make sure that it has exactly 1 entry, which contains 1 or more components of the same type. For example

```lua
local texelBufferFormat = {
    -- WRONG
    { location = 0, name = "First", format = "float" },
    { location = 1, name = "First", format = "float" },
    { location = 2, name = "First", format = "float" },
    { location = 3, name = "First", format = "float" }
}
```
Is incorrect, it should be

```lua
local texelBufferFormat = {
    -- CORRECT
    { location = 0, name = "All", format = "floatvec3" }
}
```

See section `(D.1)` below.

Texel buffer usage in the code example below includes:

+ `(D.1)` declaring a custom texel buffer format
+ `(D.2)` initializing the texel buffer from a Lua table
+ `(D.3)` initializing the texel buffer from `love.ByteData`
+ `(D.4)` initializing the texel buffer from FFI memory
+ `(D.5)` binding the texel buffer to a shader uniform
+ `(D.6)` declaring the texel buffer uniform
+ `(D.7)` using `gl_InstanceID` to index access per-instance attributes of the texel buffer in the vertex shader.
+ `(D.8)` uploading new data to the GPU-side texel buffer
+ `(D.9)`  instantiating the `love.GraphicsBuffer` object

#### 2.5 C-Data

Sections relevant to the handling of CPU-side data include:

+ `(E.1)` modifying Lua-tables directly
+ `(E.2)` modifying the C-Data using the `ByteData` interface
+ `(E.3)` modifying the C-Data using LuaJIT FFI
+ `(E.4)` updating a vertex-, storage- or texel buffer from C-data

### 2.5 Mesh Advanced Usage Example

> [!NOTE]
> Modifying `BUFFER_MODE` changes which buffer type (vertex, storage, texel) is used; Modifying `DATA_MODE` changes which kind of CPU-side data `(table, ByteData, FFI memory)` is used to fill the buffer. All permutations of these two work

> [!TIP]
> This code example is quite long and complicated. This is due to the nature and breadths of of the advanced techniques used, including custom shaders. To compensate, it has a lot of comments and works out-of-the-box. We can copy-paste it into a `main.lua`, then run it using LÖVE 12. The demo will draw 100000 different circles at 60 fps

```lua
ffi = require "ffi"

-- ### CONFIG ###

-- usage mode, decides which LÖVE 12.0 method to use for representing the instance date
local BUFFER_MODE_USE_VERTEX_BUFFER = "vertexbuffer"
local BUFFER_MODE_USE_TEXEL_BUFFER = "texelbuffer"
local BUFFER_MODE_USE_STORAGE_BUFFER = "storagebuffer"
local BUFFER_MODE = BUFFER_MODE_USE_STORAGE_BUFFER

-- upload mode, decides whether to use lua tables, `ByteData` or ffi data to upload the instance data
local DATA_MODE_USE_TABLES = "table"
local DATA_MODE_USE_BYTE_DATA = "bytedata"
local DATA_MODE_USE_FFI_DATA = "ffi"
local DATA_MODE = DATA_MODE_USE_FFI_DATA

-- ### GLOBALS ###

local drawMeshFormat -- table, vertex attribute format of mesh that will be drawn
local drawMesh -- love.Mesh, actual mesh that will be the shape of the particles
local drawShader -- love.Shader, custom shader that retrieves the per-instance data

local instanceCount = 100000 -- number of draw instances
local perInstanceFormat -- table, vertex attribute format of data mesh
local perInstanceData -- table of `love.ByteData`, CPU-side copy of per-instance data
local perInstanceDataBuffer -- `love.GraphicsBuffer`, GPU-side copy of per-instance data

--- ### INITIALIZATION ###

local generateOffset, generateScale, generateHue -- see "internals" below
local modifyOffset, modifyScale, modifyHue

love.load = function()
    -- (A.1) declare vertex attribute format of draw mesh
    drawMeshFormat = {
        { -- instance mesh attribute #1: position
            location = 0,            -- attribute #1 (0-based)
            name = "VertexPosition", -- name
            format = "floatvec2"     -- glsl format: vec2 (2 components)
        },

        { -- instance mesh attribute #2: texture coordinates
            location = 1,            -- attribute #2 (0-based)
            name = "VertexTextureCoordinates",
            format = "floatvec2"     -- glsl format: vec2 (2 components)
        },

        { -- instance mesh attribute #2: texture coordinates
            location = 2,            -- attribute #3 (0-based)
            name = "VertexColor",
            format = "floatvec4"     -- glsl format: vec4 (2 components)
        }
    }

    -- (A.6) declare format of data buffer
    do
        if BUFFER_MODE == BUFFER_MODE_USE_VERTEX_BUFFER then
            -- (B.1) declaring a custom vertex buffer format
            -- if using mesh as a buffer, location needs to be appended to locations of drawMeshFormat
            perInstanceFormat = {
                {
                    location = 3,
                    name = "InstanceOffset",
                    format = "floatvec2"
                },

                {
                    location = 4,
                    name = "InstanceScale",
                    format = "float"
                },

                {
                    location = 5,
                    name = "InstanceHue",
                    format = "float"
                }
            }

        elseif BUFFER_MODE == BUFFER_MODE_USE_STORAGE_BUFFER then
            -- (C.1) declaring a custom storage buffer format
            -- if using storage buffer, locations start at 0
            perInstanceFormat = {
                {
                    location = 0,
                    name = "InstanceOffset",
                    format = "floatvec2"
                },

                {
                    location = 1,
                    name = "InstanceScale",
                    format = "float"
                },

                {
                    location = 2,
                    name = "InstanceHue",
                    format = "float"
                },
            }
        elseif BUFFER_MODE == BUFFER_MODE_USE_TEXEL_BUFFER then
            -- (D.1) declaring a custom texel buffer format
            -- if using texel buffer, all components need to have the same format
            local format = "float"
            perInstanceFormat = {
                {
                    location = 0,
                    name = "InstanceData",
                    format = "floatvec4" -- only 4-component are supported
                }
            }
        end
    end

    -- (A.2) initializing the draw mesh as circle
    do
        local nVertices = 16
        local drawMeshData = {}
        for i = 1, nVertices + 1 do
            local angle = (i - 1) / nVertices * (2 * math.pi)
            table.insert(drawMeshData, {
                math.cos(angle), -- Attribute #1: VertexPosition.x (x)
                math.sin(angle), -- Attribute #1: VertexPosition.y (y)
                math.cos(angle), -- Attribute #2: VertexTextureCoordinates.x (u)
                math.sin(angle), -- Attribute #2: VertexTextureCoordinates.y (v)
                1, -- Attribute #3: VertexColor.x (r)
                1, -- Attribute #3: VertexColor.y (g)
                1, -- Attribute #3: VertexColor.z (b)
                1, -- Attribute #3: VertexColor.w (a)
            })
        end
        table.insert(drawMeshData, 1,  { 0, 0, 0, 0, 1, 1, 1, 1 }) -- add center vertex

        drawMesh = love.graphics.newMesh(
            drawMeshFormat, -- vertex attribute format
            drawMeshData,   -- vertex data
            "fan",   -- draw mode
            "static" -- buffer usage
        )

        -- we use draw mode `fan` to correctly draw a filled circle
        -- we use buffer usage `static`, since `drawMesh` will never change, only the per-instance data will
    end

    -- (A.3) initializing the data buffer with per instance data

    local initialData = {}
    for instanceIndex = 1, instanceCount do
        local x, y = generateOffset(instanceIndex) -- 2 floats
        local scale = generateScale(instanceIndex) -- 1 float
        local hue = generateHue(instanceIndex)
        table.insert(initialData, {
            x, y,
            scale,
            hue
        })
    end
    
    -- (A.7) initializing the per-instance data CPU-side

    if DATA_MODE == DATA_MODE_USE_TABLES then
        -- (B.2)(C.2)(D.2) using lua tables as instance data
        -- keep `perInstanceData` as a table
        perInstanceData = initialData
    elseif DATA_MODE == DATA_MODE_USE_BYTE_DATA then
        -- (B.3)(C.3)(D.3) using `ByteData` as instance data
        -- instance size is `InstanceOffset` + `InstanceScale` + `InstanceHue`
        local stride = 2 * ffi.sizeof("float") + ffi.sizeof("float") + ffi.sizeof("float")

        -- data size is (number of instances) * (size per instance)
        local byteData = love.data.newByteData(instanceCount * stride)

        -- assign initial byte data
        local floatBytes = ffi.sizeof("float")
        for i, data in ipairs(initialData) do
            local byteOffset = (i - 1) * stride
            byteData:setFloat(byteOffset + 0 * floatBytes, data[1])
            byteData:setFloat(byteOffset + 1 * floatBytes, data[2])
            byteData:setFloat(byteOffset + 2 * floatBytes, data[3])
            byteData:setFloat(byteOffset + 3 * floatBytes, data[4])
        end

        -- use byte data as global data
        perInstanceData = byteData
    elseif DATA_MODE == DATA_MODE_USE_FFI_DATA then
        -- (B.4)(C.4)(D.4) using FFI data as instance data
        -- we need to allocate as `love.ByteData` anyway, even though we use ffi to modify it later
        local stride = 2 * ffi.sizeof("float") + ffi.sizeof("float") + ffi.sizeof("float")
        local byteData = love.data.newByteData(instanceCount * stride)

        -- assign initial byte data
        local floatBytes = ffi.sizeof("float")
        local byteDataPtr = ffi.cast("uint8_t*", byteData:getFFIPointer())

        for i, data in ipairs(initialData) do
            local byteOffset = (i - 1) * stride
            ffi.cast("float*", byteDataPtr + byteOffset + 0 * floatBytes)[0] = data[1]
            ffi.cast("float*", byteDataPtr + byteOffset + 1 * floatBytes)[0] = data[2]
            ffi.cast("float*", byteDataPtr + byteOffset + 2 * floatBytes)[0] = data[3]
            ffi.cast("float*", byteDataPtr + byteOffset + 3 * floatBytes)[0] = data[4]
        end

        perInstanceData = byteData
    end

    -- (A.4) initialize the graphics buffer
    if BUFFER_MODE == BUFFER_MODE_USE_VERTEX_BUFFER then
        -- (B.5) intantiating a vertex buffer
        -- usage mesh as storage buffer
        local perInstanceDataMesh = love.graphics.newMesh(
            perInstanceFormat,
            initialData,
            "points", -- draw mode unused
            "stream"
        )

        -- (B.6) attach the mesh to the draw mesh
        for _, entry in pairs(perInstanceFormat) do
            drawMesh:attachAttribute(entry.name, 
                perInstanceDataMesh, 
                "perinstance" -- 1 attribute per 1 instanced
            )
        end

        -- (A.10) convert mesh to a buffer for love.update
        perInstanceDataBuffer = perInstanceDataMesh:getVertexBuffer()
    elseif BUFFER_MODE == BUFFER_MODE_USE_STORAGE_BUFFER then
        -- (B.9) upload the storage buffer
        perInstanceDataBuffer = love.graphics.newBuffer(
            perInstanceFormat, -- buffer format
            initialData,    -- initial buffer data
            { shaderstorage = true } -- declare as storage buffer
        )

    elseif BUFFER_MODE == BUFFER_MODE_USE_TEXEL_BUFFER then
        -- (C.9) upload the texel buffer
        perInstanceDataBuffer = love.graphics.newBuffer(
            perInstanceFormat,
            initialData,
            { texel = true } -- declare as texel buffer
        )
    end

    -- (A.8) initialize the shader

    if BUFFER_MODE == BUFFER_MODE_USE_VERTEX_BUFFER then
        drawShader = love.graphics.newShader([[
#ifdef VERTEX // vertex shader

// draw mesh attributes
layout (location = 0) in vec2 VertexPosition;      // drawMesh attribute #1
layout (location = 1) in vec2 VertexTextureCoords; // drawMesh attribute #2
layout (location = 2) in vec4 VertexColor;         // drawMesh attribute #3

// (B.7) data mesh attributes
layout (location = 3) in vec2 InstanceOffset;    // perInstanceDataBuffer attribute #1
layout (location = 4) in float InstanceScale;    // perInstanceDataBuffer attribute #2
layout (location = 5) in float hue;              // perInstanceDataBuffer attribute #3

out vec2 FragmentTextureCoords; // final interpolated texture coordinates, for fragment shader
out vec4 FragmentColor;         // final interpolated color, for fragment shader

void vertexmain() { // custom vertex shader entry point
    // (A.5)
    // compute position from custom vertex attributes
    vec2 position = VertexPosition;
    position.xy *= InstanceScale;
    position.xy += InstanceOffset;

    // set texture coords to default value
    FragmentTextureCoords = VertexTextureCoords; // xy = uv

    // set color to default value
    FragmentColor = ConstantColor * vec4(VertexColor.rgb, VertexColor.a * hue);

    // set position
    love_Position = TransformProjectionMatrix * vec4(position.xy, 0.0, 1.0);
    // where `TransformProjectionMatrix` is a hardcoded global that holds the `love.graphics` Transform
    // and `love_Position` is a hardcoded global variable that holds the position of a vertex, in px
}

#endif

#ifdef PIXEL // fragment shader

in vec2 FragmentTextureCoords; // texture coords from vertex shader
in vec4 FragmentColor;         // color from vertex shader

out vec4 FinalColor; // final fragment color drawn to the screen at love_Position

uniform sampler2D InstanceTexture; // texture of the instance mesh

void pixelmain() { // custom fragment shader entry point
    // default behavior of `effect`, implemented manually
    FinalColor = FragmentColor * texture(InstanceTexture, FragmentTextureCoords);
}

#endif
]])
    elseif BUFFER_MODE == BUFFER_MODE_USE_STORAGE_BUFFER then
        drawShader = love.graphics.newShader([[
#pragma language glsl4

#ifdef VERTEX // vertex shader

// draw mesh attributes
layout (location = 0) in vec2 VertexPosition;
layout (location = 1) in vec2 VertexTextureCoords;
layout (location = 2) in vec4 VertexColor;

out vec2 FragmentTextureCoords;
out vec4 FragmentColor;

// (C.5) new; storage buffer
struct InstanceData {
    vec2 offset;
    float scale;
    float hue;
};

// (C.6) declare assignable storage buffer
readonly layout(std430, binding = 0) buffer perInstanceDataBuffer {
    InstanceData data[];
};

void vertexmain() {
    // (C.7) access per-instance data using instance ID
    InstanceData instanceData = data[gl_InstanceID];

    vec2 position = VertexPosition;
    position.xy *= instanceData.scale;
    position.xy += instanceData.offset;

    FragmentTextureCoords = VertexTextureCoords;
    FragmentColor = ConstantColor * vec4(VertexColor.rgb, VertexColor.a * instanceData.hue);

    love_Position = TransformProjectionMatrix * vec4(position.xy, 0.0, 1.0);
}

#endif

#ifdef PIXEL

in vec2 FragmentTextureCoords;
in vec4 FragmentColor;

out vec4 FinalColor;
uniform sampler2D InstanceTexture;

void pixelmain() {
    FinalColor = FragmentColor * texture(InstanceTexture, FragmentTextureCoords);
}

#endif
]])
    elseif BUFFER_MODE == BUFFER_MODE_USE_TEXEL_BUFFER then
        drawShader = love.graphics.newShader([[
#ifdef VERTEX // vertex shader

// draw mesh attributes
layout (location = 0) in vec2 VertexPosition;
layout (location = 1) in vec2 VertexTextureCoords;
layout (location = 2) in vec4 VertexColor;

// no per-instance attributes

out vec2 FragmentTextureCoords;
out vec4 FragmentColor;

// (D.6) new: texel buffer
uniform samplerBuffer perInstanceDataBuffer;

void vertexmain() {
    // (D.7) using instance id to access per-instance data
    // we use texelFetch to prevent interpolation
    vec4 instanceData = texelFetch(perInstanceDataBuffer, gl_InstanceID);
    vec2 instanceOffset = instanceData.xy;
    float instanceScale = instanceData.z;
    float instanceHue = instanceData.w;

    vec2 position = VertexPosition;
    position.xy *= instanceScale;
    position.xy += instanceOffset;

    FragmentTextureCoords = VertexTextureCoords;
    FragmentColor = ConstantColor * vec4(VertexColor.rgb, VertexColor.a * instanceHue);

    love_Position = TransformProjectionMatrix * vec4(position.xy, 0.0, 1.0);
}

#endif

#ifdef PIXEL

in vec2 FragmentTextureCoords;
in vec4 FragmentColor;

out vec4 FinalColor;
uniform sampler2D InstanceTexture;

void pixelmain() {
    FinalColor = FragmentColor * texture(InstanceTexture, FragmentTextureCoords);
}

#endif
]])
    end
end -- love.load

--  draw loop
love.draw = function()
    love.graphics.clear()

    -- store graphics state, including shader, color, etc.
    love.graphics.push("all")

    -- (A.4) bind shader
    love.graphics.setShader(drawShader)

    -- assign the draw mesh texture if present
    if drawMesh:getTexture() ~= nil then
        drawShader:send("drawTexture", drawMesh:getTexture())
    end

    -- assign the storage / texel buffer if present
    if drawShader:hasUniform("perInstanceDataBuffer") then
        -- (C.8)(D.5) make the buffer accessible from within the shader
        drawShader:send("perInstanceDataBuffer", perInstanceDataBuffer)
    end

    -- set color / transform
    love.graphics.setColor(1, 1, 1, 1)

    -- (A.3)
    -- draw `instanceCount` many copies of the mesh instanced
    -- each instance will read the correct data using the vertex shader
    love.graphics.drawInstanced(drawMesh, instanceCount)

    -- unbind shader, etc.
    love.graphics.pop() -- all

    love.graphics.print("# Particles: " .. instanceCount .. " | FPS: " .. love.timer.getFPS(), 5, 5)
end

-- update loop

love.update = function()
    -- modify the per-instance data

    if DATA_MODE == DATA_MODE_USE_TABLES then
        -- `perInstanceData` is a simple table, modify it using regular lua

        local offsetX = 1   -- per-instance #1 component #1: InstanceOffset.x
        local offsetY = 2   -- per-instance #1 component #2: InstanceOffset.y
        local scale = 3     -- per-instance attribute #2 component #1: InstanceScale.x
        local hue = 4       -- per-instance attribute #3 component #1: InstanceHue.x

        -- (E.1) modify the Lua table directly
        for instanceIndex = 1, instanceCount do
            -- extract per-instance data
            local instanceData = perInstanceData[instanceIndex]

            -- write new per-instance attributes to CPU-side copy of instance data
            instanceData[offsetX], instanceData[offsetY] = modifyOffset(instanceIndex,
                instanceData[offsetX],
                instanceData[offsetY]
            )

            instanceData[scale] = modifyScale(instanceIndex,
                instanceData[scale]
            )

            instanceData[hue] = modifyHue(instanceIndex,
                instanceData[hue]
            )
        end
    elseif DATA_MODE == DATA_MODE_USE_BYTE_DATA then
        -- `perInstanceData` is a `love.ByteDatA`, use `set*` / `get*`

        -- get the size of one instance data element, in bytes
        local stride = perInstanceDataBuffer:getElementStride()

        -- (E.2) modify the byte data using its internal methods
        local offset = 0
        for instanceIndex = 1, instanceCount do
            local startOffset = offset

            -- read per-instance properties from byte data
            local readOffset = startOffset

            local x = perInstanceData:getFloat(readOffset)
            readOffset = readOffset + ffi.sizeof("float")

            local y = perInstanceData:getFloat(readOffset)
            readOffset = readOffset + ffi.sizeof("float")

            local scale = perInstanceData:getFloat(readOffset)
            readOffset = readOffset + ffi.sizeof("float")

            local hue = perInstanceData:getFloat(readOffset)
            readOffset = readOffset + ffi.sizeof("float")

            -- mutate
            x, y = modifyOffset(instanceIndex, x, y)
            scale = modifyScale(instanceIndex, scale)
            hue = modifyHue(instanceIndex, hue)

            -- write per-instance properties to byte data
            local writeOffset = startOffset

            perInstanceData:setFloat(writeOffset, x)
            writeOffset = writeOffset + ffi.sizeof("float")

            perInstanceData:setFloat(writeOffset, y)
            writeOffset = writeOffset + ffi.sizeof("float")

            perInstanceData:setFloat(writeOffset, scale)
            writeOffset = writeOffset + ffi.sizeof("float")

            perInstanceData:setFloat(writeOffset, hue)
            writeOffset = writeOffset + ffi.sizeof("float")

            offset = offset + stride
        end

    elseif DATA_MODE == DATA_MODE_USE_FFI_DATA then
        -- cast to 8-bit array so we can use byte offsets as before
        local perInstanceDataPtr = ffi.cast("uint8_t*", perInstanceData:getFFIPointer())

        -- get the size of one instance data element, in bytes
        local stride = perInstanceDataBuffer:getElementStride()

        -- (E.3) modify the C data using FFI
        local offset = 0
        for instanceIndex = 1, instanceCount do
            local startOffset = offset

            -- read per-instance properties from byte data
            local readOffset = startOffset

            -- cast the byte pointer at this offset to the appropriate type pointer, then dereference using `[0]`
            local x = ffi.cast("float*", perInstanceDataPtr + readOffset)[0]
            readOffset = readOffset + ffi.sizeof("float")

            local y = ffi.cast("float*", perInstanceDataPtr + readOffset)[0]
            readOffset = readOffset + ffi.sizeof("float")

            local scale = ffi.cast("float*", perInstanceDataPtr + readOffset)[0]
            readOffset = readOffset + ffi.sizeof("float")

            local hue = ffi.cast("float*", perInstanceDataPtr + readOffset)[0]
            readOffset = readOffset + ffi.sizeof("float")

            -- mutate
            x, y = modifyOffset(instanceIndex, x, y)
            scale = modifyScale(instanceIndex, scale)
            hue = modifyHue(instanceIndex, hue)

            -- write per-instance properties to byte data
            local writeOffset = startOffset

            ffi.cast("float*", perInstanceDataPtr + writeOffset)[0] = x
            writeOffset = writeOffset + ffi.sizeof("float")

            ffi.cast("float*", perInstanceDataPtr + writeOffset)[0] = y
            writeOffset = writeOffset + ffi.sizeof("float")

            ffi.cast("float*", perInstanceDataPtr + writeOffset)[0] = scale
            writeOffset = writeOffset + ffi.sizeof("float")

            ffi.cast("float*", perInstanceDataPtr + writeOffset)[0] = hue
            writeOffset = writeOffset + ffi.sizeof("float")

            offset = offset + stride

            -- these casts have a performance impact, we should instead cast once outside the loop. Here they are
            -- used inside the loop her for the sake of matching the `DATA_MODE_USE_BYTE_DATA` loop 1:1 for
            -- pedagogic purposes only
        end
    end

    -- (B.8)(C.9)(D.8)(E.4) update the per-instance data buffer
    perInstanceDataBuffer:setArrayData(perInstanceData)
end

-- ### INTERNALS ###

generateOffset = function(instanceIndex)
    local xMargin, yMargin = 0.25 * love.graphics.getWidth(), 0.25 * love.graphics.getHeight()
    local w, h = love.graphics.getWidth(), love.graphics.getHeight()

    local xCenter, yCenter = w / 2, h / 2
    local stddev = math.min((w - 2 * xMargin) / 6, (h - 2 * yMargin) / 6)

    local x = love.math.randomNormal(stddev, xCenter)
    local y = love.math.randomNormal(stddev, yCenter)

    x = math.max(xMargin, math.min(w - xMargin, x))
    y = math.max(yMargin, math.min(h - yMargin, y))

    return x, y
end

generateScale = function(instanceIndex)
    return love.math.random(1, 8)
end

generateHue = function(instanceIndex)
    local lower, upper = 0.06, 0.3
    local t = love.math.random()
    return lower * t + upper * (1 - t)
end

modifyOffset = function(instanceIndex, x, y)
    local maxOffset = 100 * love.timer.getDelta()
    local t = love.timer.getTime()

    local seedX = instanceIndex * 17.3
    local seedY = -instanceIndex * 31.1777777
    local newX = x + maxOffset * (love.math.perlinNoise(seedX, t) * 2 - 1)
    local newY = y + maxOffset * (love.math.perlinNoise(seedY, t) * 2 - 1)

    local minX, maxX = 0, love.graphics.getWidth()
    local minY, maxY = 0, love.graphics.getHeight()

    if newX < minX then
        newX = minX + (minX - newX)
    elseif newX > maxX then
        newX = maxX - (newX - maxX)
    end

    if newY < minY then
        newY = minY + (minY - newY)
    elseif newY > maxY then
        newY = maxY - (newY - maxY)
    end

    return newX, newY
end

modifyScale = function(instanceIndex, scale)
    return 1 + 9 * (math.sin(love.timer.getTime() + (instanceIndex / instanceCount) * (2 * math.pi)) + 1) / 2
end

modifyHue = function(instanceIndex, hue)
    return hue
end
```



