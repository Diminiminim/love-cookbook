---
title: "Canvases"
authors: [tomato]
date: 2026-07-09
---

# Canvases

A canvas is a texture that can be drawn to. You can think of it as an imaginary screen. In this chapter, we will learn how powerful they are, and how it differs from drawing directly to the screen.

# Creation

> [!WARNING]
> Canvases aren't cheap to create and consume GPU memory (VRAM). Making huge canvases, or making them constantly (like, every frame) will cause optimization problems. It is recommended to use as few canvases as possible, and making them once.

> [!WARNING]
> If you create a canvas matching the window's size and the user resizes the window, the canvas won't resize automatically. If you need it to match the new window size, create a new canvas and redraw its contents.

We use {% api "love.graphics.newCanvas" %} to create a canvas.
```lua
function love.load()
    canvas = love.graphics.newCanvas() -- Creates a canvas matching the current window's size.
end
```

```lua
function love.load()
    canvas = love.graphics.newCanvas(200,300) -- Creates a canvas with a specific width and height.
end
```

# Using a canvas

Once we created a canvas, we can draw on it using the normal drawing methods. However, we must switch to the canvas first.

> [!Warning]
> Remember that graphics state such as the current color, shader, blend mode, and scissor rectangle is shared between drawing to a canvas and drawing to the screen. Restore any state you change before continuing to render.

> [!Note]
> Unlike the screen, a canvas preserves what was previously drawn to it. This means you only need to redraw parts that change.

```lua
function love.load()

  canvas = love.graphics.newCanvas()
  
  love.graphics.setCanvas(canvas) -- Begins to draw on the canvas, instead of the screen.
  love.graphics.circle("fill", 0, 0 , 25) -- Draws a circle on the top-left corner of the canvas.
  love.graphics.setCanvas() -- Goes back to draw to the screen.
    
end
```

If you run the code above, you won't see anything. And that's the main point, whatever you draw isn't sent directly to the screen. That's why we draw on the canvas at love.load(), instead of love.draw().
However, if you want to make the canvas visible, you can draw it too!

> [!Tip]
> If you're confused, just remember this: what you draw on the canvas, stays on the canvas.

> [!Tip]
> Call {% api "love.graphics.clear" %} while the canvas is active to erase its contents.

> [!Tip]
> Anything that can be drawn to the screen can also be drawn to a canvas, including images, text, meshes, and even other canvases.

> [!Note]
> It is important to call .setCanvas() before drawing on screen, because it will draw on the canvas otherwise!

```lua
function love.draw()

  love.graphics.draw(canvas)
    
end
```

{% love 200,200 %}
function love.draw()
    love.graphics.circle("fill", 0, 0 , 25) -- Draws a circle on the top-left corner of the canvas.
end
{% endlove %}

# To finish understanding

## Comparison

### Drawing on screen
* Needs to be done every frame (inside love.draw).
* Doesn't consume extra GPU memory (VRAM).
* Useful for things that will change constantly.
* Can slow down the game based on the complexity.
* All you draw is immediately visible.

### Drawing on canvases
* Contents persist until you clear or overwrite them.
* Consumes GPU memory proportional to the canvas size.
* Useful for content that can be reused across many frames (cached backgrounds, paint programs, lighting buffers, post-processing effects, etc.).
* May improve performance by avoiding expensive redraws every frame, but it is not automatically faster in all cases.
* Nothing appears on screen until you draw the canvas with love.graphics.draw(canvas).

## Use cases
* Render a complex image once, and draw it with no optimization problems.
* Export your render to a file.
* Process drawings as an image.
* Use user-drawn assets.
* Simulate pixelart by drawing on a small canvas.
* Take screenshots.
* Lighting and shadow systems.
* Post-processing effects (blur, bloom, CRT shaders, etc.).
* And the list can go on and on.

# Examples

## Making a working Paint

In this example, we create a simple Paint copy, where you can draw while pressing your mouse. Pressing "C" key will clear the canvas.
This works because a canvas keeps its contents until you overwrite or clear them. This means our previous stroke will stay there.

```lua
local paintCanvas
local brushSize = 8

function love.load()
  paintCanvas = love.graphics.newCanvas(800, 600)

  -- Clear the canvas with a white background.
  love.graphics.setCanvas(paintCanvas)
  love.graphics.clear(1, 1, 1, 1)
  love.graphics.setCanvas()
end

function love.update(dt)
  if love.mouse.isDown(1) then
    local mouseX, mouseY = love.mouse.getPosition()

    -- Draw directly onto the canvas so the strokes become permanent.
    love.graphics.setCanvas(paintCanvas)
    love.graphics.setColor(0, 0, 0)
    love.graphics.circle("fill", mouseX, mouseY, brushSize)
    love.graphics.setCanvas()
  end
end

function love.draw()
  love.graphics.setColor(1, 1, 1)
  love.graphics.draw(paintCanvas)

  love.graphics.setColor(1, 0, 0)
  love.graphics.circle("line", love.mouse.getX(), love.mouse.getY(), brushSize)
end

function love.keypressed(key)
  if key == "c" then
    -- Clear the drawing while keeping the canvas.
    love.graphics.setCanvas(paintCanvas)
    love.graphics.clear(1, 1, 1, 1)
    love.graphics.setCanvas()
  end
end
```

{% love 500,500 %}
local paintCanvas
local brushSize = 8

function love.load()
  paintCanvas = love.graphics.newCanvas(800, 600)

  -- Clear the canvas with a white background.
  love.graphics.setCanvas(paintCanvas)
  love.graphics.clear(1, 1, 1, 1)
  love.graphics.setCanvas()
end

function love.update(dt)
  if love.mouse.isDown(1) then
    local mouseX, mouseY = love.mouse.getPosition()

    -- Draw directly onto the canvas so the strokes become permanent.
    love.graphics.setCanvas(paintCanvas)
    love.graphics.setColor(0, 0, 0)
    love.graphics.circle("fill", mouseX, mouseY, brushSize)
    love.graphics.setCanvas()
  end
end

function love.draw()
  love.graphics.setColor(1, 1, 1)
  love.graphics.draw(paintCanvas)

  love.graphics.setColor(1, 0, 0)
  love.graphics.circle("line", love.mouse.getX(), love.mouse.getY(), brushSize)
end

function love.keypressed(key)
  if key == "c" then
    -- Clear the drawing while keeping the canvas.
    love.graphics.setCanvas(paintCanvas)
    love.graphics.clear(1, 1, 1, 1)
    love.graphics.setCanvas()
  end
end
{% endlove %}

Additionally, we can create a function to export our masterpiece!

> [!Warning]
> This has been written for LÖVE 11.0

```lua
function exportCanvas(fileName)
  local imageData = paintCanvas:newImageData()
  imageData:encode("png", fileName) -- Exports the image to the game's save path.
end
```

## Drawing complex background

We'll draw a scene once in love.load(). Then we'll draw the resulting canvas every frame. This avoids rebuilding the entire scene each frame. We can still draw objects on top of the canvas.

```lua
local backgroundCanvas

function love.load()
  backgroundCanvas = love.graphics.newCanvas(800, 600)

  -- Render the entire background only once.
  love.graphics.setCanvas(backgroundCanvas)
  love.graphics.clear(0.6, 0.8, 1.0)

  -- Draw the sky.
  love.graphics.setColor(0.6, 0.8, 1.0)
  love.graphics.rectangle("fill", 0, 0, 800, 600)

  -- Draw some mountains.
  love.graphics.setColor(0.45, 0.45, 0.55)
  love.graphics.polygon("fill",
    0, 400,
    180, 180,
    360, 400
  )

  love.graphics.polygon("fill",
    220, 400,
    450, 140,
    680, 400
  )

  love.graphics.polygon("fill",
    520, 400,
    760, 220,
    900, 400
  )

  -- Draw the ground.
  love.graphics.setColor(0.2, 0.65, 0.25)
  love.graphics.rectangle("fill", 0, 375, 800, 225)

  -- Draw a bunch of trees.
  for i = 1, 26 do
    local x = i * 30

    love.graphics.setColor(0.45, 0.25, 0.1)
    love.graphics.rectangle("fill", x - 3, 350, 6, 40)

    love.graphics.setColor(0.15, 0.5, 0.15)
    love.graphics.circle("fill", x, 340, 16)
    love.graphics.circle("fill", x - 10, 345, 12)
    love.graphics.circle("fill", x + 10, 345, 12)
  end

  -- Draw some clouds.
  love.graphics.setColor(1, 1, 1, 0.9)

  for i = 1, 8 do
    local x = i * 90
    local y = 60 + (i % 2) * 30

    love.graphics.circle("fill", x, y, 20)
    love.graphics.circle("fill", x + 18, y - 5, 18)
    love.graphics.circle("fill", x + 36, y, 20)
  end

  love.graphics.setCanvas()
end

function love.draw()
  -- Drawing the whole scene is now just a single draw call.
  love.graphics.setColor(1, 1, 1)
  love.graphics.draw(backgroundCanvas)

  -- Dynamic objects can still be drawn on top.
  local mouseX, mouseY = love.mouse.getPosition()

  love.graphics.setColor(1, 0, 0)
  love.graphics.circle("fill", mouseX, mouseY, 12)
end
```

{% love 800,600 %}
local backgroundCanvas

function love.load()
  backgroundCanvas = love.graphics.newCanvas(800, 600)

  -- Render the entire background only once.
  love.graphics.setCanvas(backgroundCanvas)
  love.graphics.clear(0.6, 0.8, 1.0)

  -- Draw the sky.
  love.graphics.setColor(0.6, 0.8, 1.0)
  love.graphics.rectangle("fill", 0, 0, 800, 600)

  -- Draw some mountains.
  love.graphics.setColor(0.45, 0.45, 0.55)
  love.graphics.polygon("fill",
    0, 400,
    180, 180,
    360, 400
  )

  love.graphics.polygon("fill",
    220, 400,
    450, 140,
    680, 400
  )

  love.graphics.polygon("fill",
    520, 400,
    760, 220,
    900, 400
  )

  -- Draw the ground.
  love.graphics.setColor(0.2, 0.65, 0.25)
  love.graphics.rectangle("fill", 0, 375, 800, 225)

  -- Draw a bunch of trees.
  for i = 1, 26 do
    local x = i * 30

    love.graphics.setColor(0.45, 0.25, 0.1)
    love.graphics.rectangle("fill", x - 3, 350, 6, 40)

    love.graphics.setColor(0.15, 0.5, 0.15)
    love.graphics.circle("fill", x, 340, 16)
    love.graphics.circle("fill", x - 10, 345, 12)
    love.graphics.circle("fill", x + 10, 345, 12)
  end

  -- Draw some clouds.
  love.graphics.setColor(1, 1, 1, 0.9)

  for i = 1, 8 do
    local x = i * 90
    local y = 60 + (i % 2) * 30

    love.graphics.circle("fill", x, y, 20)
    love.graphics.circle("fill", x + 18, y - 5, 18)
    love.graphics.circle("fill", x + 36, y, 20)
  end

  love.graphics.setCanvas()
end

function love.draw()
  -- Drawing the whole scene is now just a single draw call.
  love.graphics.setColor(1, 1, 1)
  love.graphics.draw(backgroundCanvas)

  -- Dynamic objects can still be drawn on top.
  local mouseX, mouseY = love.mouse.getPosition()

  love.graphics.setColor(1, 0, 0)
  love.graphics.circle("fill", mouseX, mouseY, 12)
end
{% endlove %}

# Challenge

To practice using canvas, we invite you to continue the Paint example. Add these features:
* Different sizes.
* Different colors.
* Different brush types.
* A way to control alpha.
* A key to export the drawing.
* Add support to having several layers.
* Use linear interpolation to make the strokes smooth (instead of drawing a circle, draw a line from old mouse position to new mouse position).
to new mouse position).
* Add an undo feature by storing previous canvas states.
