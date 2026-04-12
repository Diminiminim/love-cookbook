---
title: "OOP"
authors: [ellraiser]
date: 2025-05-21
description: "An introduction to OOP programming with LÖVE"
---

OOP is not just something you say when your code breaks, it stands for "Object-Oriented Programming", and is a way of organising your code.  
It is one of a few different architectures you can use when designing games. Another well-known one is ECS - "Entity-Component-System", but this page is just for OOP!

In this chapter we'll cover some basics around OOP and some ways to implement this in Lua, as well as work towards a larger example game!
It's worth noting there are lots of different ways this can be achieved, and what is shown here is just one example.

## Approach
The general idea for OOP is to break down the game into different classes, each with their own self-contained properties and logic required for their function.  
You create instances of these classes, called objects, to populate your game.

In this example we'll be working on making a classic arcade game - Asteroids.  
If you break down the elements of Asteroids, there's (at least) 3 main classes that everything falls into:
* Player/Ship - The player ship has a position, health, it can move, rotate, and fire bullets.
* Asteroid - The asteroids has a position, it can move, be destroyed, and split into other asteroids.
* Bullet - Bullets are fired from the player, they have a position, angle, speed and can destroy asteroids.

An example object for each of those could look like:
```lua
local player = {
  x = 10,
  y = 16,
  hp = 400,
  move = function() end,
  fire = function() end
}

local asteroid = {
  x = 30,
  y = 42,
  size = 50,
  move = function() end,
  destroy = function() end
}

local bullet = {
  x = 10,
  y = 16,
  angle = 40,
  speed = 100,
  collide = function() end
}
```

Each class is a template for every object of that type - every asteroid is going to have the same properties and methods, as they all get created from the same class.

Using an OOP approach might help you break down your game into these different "blueprint" classes, as well as help you keep logic in the right place - finding something related to player movement will be in the player class file, making it easy to keep track of.

You can also rely on certain things based on the type of class - you know all asteroid objects have a size property, you know all bullet objects have a collide method etc.

There's more to OOP than just the class/object system, but that's outside the scope of this tutorial. If you're interested in learning more you can have a read of the following resources:
* [https://www.freecodecamp.org/news/what-is-object-oriented-programming/](https://www.freecodecamp.org/news/what-is-object-oriented-programming/)
* [https://developer.mozilla.org/en-US/docs/Learn_web_development/Extensions/Advanced_JavaScript_objects/Object-oriented_programming](https://developer.mozilla.org/en-US/docs/Learn_web_development/Extensions/Advanced_JavaScript_objects/Object-oriented_programming)
* [https://en.wikipedia.org/wiki/Object-oriented_programming](https://en.wikipedia.org/wiki/Object-oriented_programming)

While some languages provide classes "out the box", Lua doesn't directly - but by using [metatables](../lua-intermediate/metatables) we can create some systems to allow us to implement a robust OOP system.

## Creating Classes
In Lua, both your classes and objects are just going to be your standard [tables](../lua-basics/tables), with their properties and their functions inside them.

Consider the following bad code, we'll explain why it's "bad" in a moment:
```lua
-- asteroid "class"
function newAsteroid(x, y, angle)
  -- new asteroid table
  local obj = {
    x = x,
    y = y,
    angle = math.rad(angle),
    speed = 10,
    size = 10,
    -- update script
    update = function(self)
      self.x = self.x + (math.cos(self.angle) * self.speed)
      self.y = self.y + (math.sin(self.angle) * self.speed)
    end,
    -- draw script
    draw = function(self)
      love.graphics.circle("fill", self.x, self.y, self.size)
    end
  }
  return obj
end
```

```lua
-- make some asteroids
local asteroids = {}
for a=1,10 do
  table.insert(asteroids, newAsteroid(love.math.random(0, 100), love.math.random(0, 100), love.math.random(0, 360)))
end
```

So we have a function moonlighting as a "class" that makes a new asteroid "object", defining some basic properties, and an update and a draw script.  
We then call this 10 times, creating an asteroid somewhere in a 100x100px area, with a random initial angle in any direction.

This seems like it would work fine (and technically it would), however it means that for each asteroid we're making new functions for both "update" and "draw" every single time.  
In our loop we've created 20 new functions while making our asteroids!

These functions are the same for every single asteroid, so really they should all use the "same" function.

## Basic Classes
To make things more efficient, we should have one class table, and all objects created from that class should use that same "blueprint" for their functions and logic.

This is where metatables come in, consider this bad boy:
```lua 13,14
-- asteroid class
local asteroid = {
  new = function(self, x, y, angle)
    -- new asteroid table
    local obj = {
      x = x,
      y = y,
      angle = math.rad(angle),
      speed = 10,
      size = 10,
    }
    -- the two magic lines
    setmetatable(obj, self)
    self.__index = self
    --
    return obj
  end,
  -- update script
  update = function(self)
    self.x = self.x + (math.cos(self.angle) * self.speed)
    self.y = self.y + (math.sin(self.angle) * self.speed)
  end,
  -- draw script
  draw = function(self)
    love.graphics.circle("fill", self.x, self.y, self.size)
  end,
  -- destroy script
  destroy = function(self)
    print("boom!")
  end
}
```

```lua
-- make some asteroids
local asteroids = {}
for a=1,10 do
  table.insert(asteroids, asteroid:new(love.math.random(0, 100), love.math.random(0, 100), love.math.random(0, 360)))
end
```

Some new stuff to notice here - first we've changed our asteroid class definition from being just a function that returns a table to being an actual table with a `new` method.  
This allows us to create new objects with `asteroid:new`, the `:` syntax passing the `asteroid` table as the first parameter, `self`.  

We then write these two magic lines which give us the basis of our OOP in Lua:
```lua
setmetatable(obj, self)
self.__index = self
```

These lines are doing two things - first as you learnt in [metatables](../lua-intermediate/metatables), we're setting the metatable of our `obj` to be the `asteroid` table.  
This means that any metamethods defined for `asteroid`, such as `__newindex` or `__index`, will be inherited by the `obj` table.

The second line sets the `__index` metamethod to the `asteroid` table itself, which essentially means all the methods on the `asteroid` table will now be available on our `obj` table - acting as a template! 

```lua
local obj = asteroid:new(10, 10, 50)
-- the following line will print the same (e.g. function: 0x02eebff9b2c0) 
-- which shows they are referring to the same function.
print(asteroid.destroy, obj.destroy)
obj:destroy() -- will print "boom!"
```

> [!NOTE]
> If you"re interested on how the two magic lines actually work, consider an example flow:
> 1. We want to call `obj:destroy()`
> 2. First the system will check for an `obj.destroy` property to call - but it doesn't exist
> 3. Then the system checks for a `obj.__index` metamethod
> 4. Because we used `setmetatable`, our `obj.__index` is the same as `asteroid.__index`
> 5. `asteroid.__index` is pointing at itself, the `asteroid` table
> 6. The system will now check for an `asteroid.destroy` property to call - it finds one!
> 7. The system calls `asteroid.destroy` passing the `obj` as the `self` parameter

## Alternative Structures

You might realise that we don't actually have to set `self.__index = self` in the `new` function above, as technically after the first object gets created this line is obsolete because we've already set `asteroid.__index` to `asteroid` once.

Something like this would work just as well:
```lua
-- another asteroid class
local asteroid = {
  new = function(self, x, y, angle)
    -- new asteroid table
    local obj = {
      x = x,
      y = y,
      angle = math.rad(angle),
      speed = 10,
      size = 10,
    }
    -- set the metatable as normal
    setmetatable(obj, self)
    return obj
  end,
  -- destroy script
  destroy = function(self)
    print("boom!")
  end
}
-- set the __index here instead
asteroid.__index = asteroid
```

Having it dumped at the bottom of your class file can be a bit awkward and you might forget to add it though, which is why it normally gets tidied away into the `new` method.

You could also define your class like this, breaking each definition out:
```lua
-- yet another asteroid class
local asteroid = {}
-- set __index here
asteroid.__index = asteroid
-- create script
asteroid.new = function(self, x, y, angle)
  -- new asteroid table
  local obj = {
    x = x,
    y = y,
    angle = math.rad(angle),
    speed = 10,
    size = 10,
  }
  setmetatable(obj, self)
  return obj
end
-- destroy script
asteroid.destroy = function(self)
  print("boom!")
end
```

Both approaches work exactly the same, it's just different styles based on your own preference and how you like to write your code!

Now we have a basic class stucture, let's look at extended them!

## Extended Classes
Extending a class means taking a class as a 'base' and adding or modifying it in some way.

For example, you have an asteroid class for all the asteroids in the game, but what if we want a sort of 'boss' asteroid, some sort of giant planet that functions like an asteroid but has a few differences? We could code the planet logic into the asteroid class, but things could get messy quickly.

Instead we could just have a planet class, that is made by extended the asteroid class - meaning we can re-use all the asteroid code and just focus on how the planet differs in functionality.

There's a few different ways you could 'extend' a class to inherit the base properties and logic and then add to it or override it in an extended class.  
They all have to deal with the problem of `setmetatable` only being able to set one table, not multiple.

One option you could do is simply add to or override methods in the extended class in it's `new` call:
```lua
-- basic asteroid class
local asteroid = {
  new = function(self, x, y, size)
    local obj = {
      x = x,
      y = y,
      size = size
    }
    setmetatable(obj, self)
    self.__index = self
    return obj
  end,
  destroy = function(self)
    print('boom!')
  end
}
```

```lua
-- extended planet class
local planet = {
  new = function(self, x, y)
    local obj = asteroid:new(x, y, 1000)
    obj.destroy = self.destroy -- overwrite destroy script
    obj.reposition = self.reposition -- add a new script
    return obj
  end,
  destroy = function(self)
    print('bigger boom!')
  end,
  reposition = function(self, x, y)
    self.x = x
    self.y = y
  end
}
```

```lua
-- planet will have same properties as asteroid, plus script changes
local obj = planet:new(10, 10)
obj:destroy() -- prints 'bigger boom!'
obj:reposition(20, 20)
print(obj.x) -- 20
```

This is fine, but having to define new methods in `new` might lead you to forgetting to update new scripts you add later - if you add another script to the `planet` table but don't define it in the `new` method then you won't be able to call it.

You could instead do something like:
```lua
local planet = {
  new = function(self, x, y)
    local obj = asteroid:new(x, y, 1000)
    for key, value in pairs(self) do
      obj[key] = value
    end
    return obj
  end,
  destroy = function(self)
    print('bigger boom!')
  end,
}
```
This would make sure any keys defined on the `planet` table would get added to all planet objects, and they'd still be a reference not a copy.

If you want to melt your brain thinking about `__index` references and prefer the more definition led structure shown earlier, you could also do:
```lua
-- asteroid class
local asteroid = {}
asteroid.new = function(self, x, y, size)
  local obj = {
    x = x,
    y = y,
    size = size
  }
  setmetatable(obj, self)
  self.__index = self -- note we're purposely setting __index here
  return obj
end
asteroid.split = function(self)
  for a=1,3 do
    asteroid:new(self.x, self.y, self.size/2)
  end
  print('split!')
end
asteroid.destroy = function(self)
  print('boom!')
end
```

```lua
-- planet class
local planet = asteroid:new()
planet.new = function(self, x, y)
  self.x = x
  self.y = y
  self.size = 1000
  return self
end
planet.destroy = function(self)
  print('bigger boom!')
end
```

```lua
-- check it all works!
local obj = planet:new(10, 10)
obj:destroy() -- prints 'bigger boom!'
obj:split() -- prints 'split!', still works!
```

In this version, we're defining the `asteroid` class as normal, but for the `planet` class, we first make a new `asteroid` object to use AS our class.  
This means when we call `asteroid:new()` we're passing in the `planet` table as the reference for the metatable, but because we created it initially as an `asteroid` object we have all the existing methods too.

By extended classes you can make lots of variations of the same base class, and only need to modify the base class code for common changes - if you had a planet class and some new 'shipwreck' class that both extended the asteroid class, and decided you wanted to change how all asteroids move or split, you only need to edit the code for the base class and all other classes will be updated.

If you want to see some more advanced examples of class and extension structures, you should check out rxi's [classic](https://github.com/rxi/classic/blob/master/classic.lua).  

This is small bit of code that lets you create classes, extend them, and also use 'mixins', which are essentially some predefined set of functions that you can then add to a class. You should be able to make sense of the code after what you've learned here and in the [metatables](../lua-intermediate/metatables) page, or maybe try using it to build your own class system!  


# Futher Abstraction
You have your Player, Asteroid, and a Bullet class. All 3 have an 'x' and a 'y' and a 'direction' and a 'speed' - you could make some sort of generic 'Object' class and all 3 classes are extensions of that, which initially wouldn't be that useful as all you're doing is saving yourself writing out those four keys 3 times.

However, if all those classes had some common logic, say some generic methods like a `destroy` or some default drawing code, then it might make sense to do this sort of abstraction. 

You could even play with having some common 'event' callback functions that your base class implements, that you can then define in your extended classes:
```lua
-- create a generic object class
local object = {}
object.__index = object
object.new = function(self, x, y, dir, speed)
  local obj = {
    x = x,
    y = y,
    dir = dir,
    speed = speed,
    destroy_script = nil
  }
  setmetatable(obj, self)
  return obj
end
-- when object:destroy() is called check for a destroy_script
-- call that first if we have one
object.destroy = function(self)
  if self.destroy_script then self:destroy_script() end
  print('some default cleanup here!')
end
```

```lua
-- extend the object class to make an asteroid class
local asteroid = object:new(0, 0, 0, 10)
asteroid.size = 100
-- define the custom destroy script to run on destroy
asteroid.destroy_script = function(self)
  for a=1,3 do
    asteroid:new(self.x, self.y, self.size/2)
  end
  print('split!')
end

-- call destroy, which in turn now calls our destroy_script as well
asteroid:destroy()
```

## Event Callbacks
With what we just looked at, you can see how you could have a way of setting up your own event callbacks, but what about the standard love events?

Doing this is pretty easy, all you'd need to do is standardise the naming for the various event scripts across all your classes - if you make sure all asteroids and the player and bullets have an 'update' script and a 'draw' script, we can add all these objects into a big table and iterate over that inside the main love events.
```lua
-- a list of objects to easily iterate over all objects
local all_asteroids = {}

-- asteroid class
local asteroid = {
  new = function(self, x, y, angle)
    -- new asteroid table
    local obj = {
      x = x,
      y = y,
      angle = math.rad(angle),
      speed = 10,
      size = 10,
    }
    setmetatable(obj, self)
    self.__index = self
    table.insert(all_asteroids, obj)
    return obj
  end,
  -- update script
  update = function(self, dt)
    self.x = self.x + (math.cos(self.angle) * self.speed * dt)
    self.y = self.y + (math.sin(self.angle) * self.speed * dt)
  end
}

-- now in the love.update hook we can simply iterate through all objects
love.update = function(dt)
  for i=1,#all_asteroids do
    -- all asteroids will have an update script, now called every frame
    all_asteroids[i]:update(dt)
  end
end
```

In this way you could set up callbacks for all the main events, `update`, `mousepressed`, `keypressed`, `draw`, and then let each of the objects handle their own logic when any of these events are called. Combine this with your own custom events, like say a `destroy` or a `gameover`, and you'll be able to have some complex logic without losing track of where everything is.

You could also have different lists, maybe one for each type of class, and then you only iterate over the type of object you need rather than all of them.

## Overcomplicating
When making a game with OOP, it's worth remembering that not every single thing has to fall under an object, and you do not always need to keep making more abstracted and 'simpler' objects - this can end up making things messier. OOP is a general approach to structuring the architecture of your game and helping you plan things out, but it doesn't have to be completely strict.

In the example above we looked at making a generic class for all 3 classes to use. However say you had a 4th class, like a UI Button, that only had an 'x' and a 'y' - would you make an even more abstracted class with just an 'x' and a 'y' that the 'Object' itself extends? Probably not, where does it end! A class for just the 'x' property??? (don't do this)

In the other direction, there will be things that don't quite fall inside one specific type of object - main menus, saving the game, incrementing score, drawing GUI, playing music etc. You might be tempted to just make one big 'Controller' object (a "god object") that handles all of this larger logic, which is fine but it's very easy for this to snowball into a large and unwieldy file.

The idea of a controller object is good though, you just might want to split that out into a few different ones, a central one for game states, one for audio, one for cutscenes. It also might be better to just have a few 'module' files, for common utility scripts - it doesn't have to be an object for every single function, and sometimes a function might make more sense sitting outside of objects.

## Asteroids!
Here's one final example, using all the stuff just mentioned as well as everything else we've covered in this chapter to take a basic stab at creating Asteroids!

We have a helper 'module' to store some reused scripts across classes, an asteroid class, a bullet class, and a player class.  
We create a player, and a bunch of asteroids, and then use event callbacks to automatically update and draw our objects.

Using "W" we can move forward, and use "A" and "D" to turn. Clicking with the mouse will fire a bullet that 'destroys' an asteroid!  
You will flash red when you are 'hit' by an asteroid.

If you go further down the page you can run the example code in your browser too!

```lua
-- a list of objects to easily iterate over all objects
local all_objects = {}

-- helper/utility 'module'
-- we use these methods in both classes, so makes sense to sit outside the objects
local helper = {
  -- set x delta using angle
  calculateVX = function(value, angle, speed)
    return value + (math.cos(angle) * speed)
  end,
  -- set y delta using angle
  calculateVY = function(value, angle, speed)
    return value + (math.sin(angle) * speed)
  end,
  -- check if an obj is colliding with a specific 'type' of object in the list
  checkCollision = function(obj, type)
    for o=1,#all_objects do
      local other = all_objects[o]
      if other ~= obj and other.type == type and not other.destroyed
      and obj.x >= other.x and obj.x <= other.x+16
      and obj.y >= other.y and obj.y <= other.y+16 then
        return other
      end
    end
    return nil
  end
}
```

```lua
-- asteroid class
local asteroid = {
  new = function(self, x, y)
    -- new asteroid table
    local obj = {
      x = x,
      y = y,
      angle = math.rad(love.math.random(0, 360)),
      speed = love.math.random(1, 10)/10,
      destroyed = false,
      type = 'asteroid'
    }
    setmetatable(obj, self)
    self.__index = self
    table.insert(all_objects, obj)
    return obj
  end,
  -- update script
  update = function(self)
    if self.destroyed then return nil end
    self.x = helper.calculateVX(self.x, self.angle, self.speed)
    self.y = helper.calculateVY(self.y, self.angle, self.speed)
    -- bounce at edges of screen
    if self.x < 0 or self.y < 0 or self.x > 800 or self.y > 600 then
      self.angle = self.angle * -1
    end
  end,
  -- draw script
  draw = function(self)
    if self.destroyed then return nil end
    love.graphics.setColor(0, 1, 1)
    love.graphics.rectangle('line', self.x, self.y, 16, 16)
  end,
  -- destroy script
  destroy = function(self)
    self.destroyed = true -- a 'fake' destroy, it's still in the list!
  end
}
```

```lua
-- bullet class
local bullet = {
  new = function(self, x, y, angle)
    local obj = {
      x = x,
      y = y,
      angle = angle,
      speed = 5,
      collided = false,
      destroyed = false,
      type = 'bullet'
    }
    setmetatable(obj, self)
    self.__index = self
    table.insert(all_objects, obj)
    return obj
  end,
  -- update script
  update = function(self, dt)
    if self.destroyed then return nil end
    self.x = helper.calculateVX(self.x, self.angle, self.speed)
    self.y = helper.calculateVY(self.y, self.angle, self.speed)
    -- check if colliding with asteroid
    local hit = helper.checkCollision(self, 'asteroid')
    if hit then
      hit:destroy()
      self:destroy()
    end
  end,
  -- draw script
  draw = function(self)
    if self.destroyed then return nil end
    love.graphics.setColor(1, 1, 0, 1)
    love.graphics.circle('fill', self.x, self.y, 2)
    love.graphics.setColor(1, 1, 1, 1)
  end,
  -- destroy script
  destroy = function(self)
    self.destroyed = true -- a 'fake' destroy, it's still in the list!
  end
}
```

```lua
-- player class
local player = {
  new = function(self, x, y, angle)
    local obj = {
      x = x,
      y = y,
      angle = math.rad(angle),
      speed = 150,
      hp = 100,
      collided = false,
      spr = love.graphics.newCanvas(16, 16),
      type = 'player'
    }
    -- @TEMP 
    -- this is just to draw a 'fake' sprite instead of using a texture+quad
    love.graphics.setCanvas(obj.spr)
      love.graphics.polygon('fill', 0,0, 16,8, 0,16)
    love.graphics.setCanvas()
    -- @TEMP
    setmetatable(obj, self)
    self.__index = self
    table.insert(all_objects, obj)
    return obj
  end,
  -- update script
  update = function(self, dt)
    local speed = 0
    -- a + d to change angle, w + s to move forward + backward
    if love.keyboard.isDown('a') then self.angle = self.angle + (-3 * dt) end
    if love.keyboard.isDown('d') then self.angle = self.angle + (3 * dt) end
    if love.keyboard.isDown('w') then speed = self.speed * dt end
    -- reuse the helper scripts
    self.x = helper.calculateVX(self.x, self.angle, speed)
    self.y = helper.calculateVY(self.y, self.angle, speed)
    -- see if we hit something!
    self.collided = helper.checkCollision(self, 'asteroid') ~= nil
  end,
  -- draw script
  draw = function(self)
    if self.collided then
      love.graphics.setColor(1, 0, 0, 1)
    else
      love.graphics.setColor(1, 1, 1, 1)
    end
    love.graphics.draw(self.spr, self.x, self.y, self.angle, 1, 1, 8, 8)
    love.graphics.setColor(1, 1, 1, 1)
  end,
  -- mousepress
  mousepressed = function(self)
    bullet:new(self.x, self.y, self.angle)
  end
}
```

```lua
-- load game
love.load = function()
  -- make a player
  local p = player:new(10, 10, 20)
  -- make some asteroids
  for a=1,30 do
    asteroid:new(love.math.random(0, 800), love.math.random(0, 600))
  end
end

-- love events
-- for all objects, check if there's an event script defined, then call it if so
love.update = function(dt)
  for i=1,#all_objects do
    if all_objects[i].update then all_objects[i]:update(dt) end
  end
end
love.draw = function()
  for i=1,#all_objects do
    if all_objects[i].draw then all_objects[i]:draw() end
  end
end
love.mousepressed = function()
  for i=1,#all_objects do
    if all_objects[i].mousepressed then all_objects[i]:mousepressed() end
  end
end
```

Try it out below!  
Use "W" to move forward, "A" + "D" to turn left and right, and then click to fire. Pew pew!

{% love 800, 600 %}
-- a list of objects to easily iterate over all objects
local all_objects = {}

-- helper/utility 'module'
-- we use these methods in both classes, so makes sense to sit outside the objects
local helper = {
  -- set x delta using angle
  calculateVX = function(value, angle, speed)
    return value + (math.cos(angle) * speed)
  end,
  -- set y delta using angle
  calculateVY = function(value, angle, speed)
    return value + (math.sin(angle) * speed)
  end,
  -- check if an obj is colliding with a specific 'type' of object in the list
  checkCollision = function(obj, type)
    for o=1,#all_objects do
      local other = all_objects[o]
      if other ~= obj and other.type == type and not other.destroyed
      and obj.x >= other.x and obj.x <= other.x+16
      and obj.y >= other.y and obj.y <= other.y+16 then
        return other
      end
    end
    return nil
  end
}

-- asteroid class
local asteroid = {
  new = function(self, x, y)
    -- new asteroid table
    local obj = {
      x = x,
      y = y,
      angle = math.rad(love.math.random(0, 360)),
      speed = love.math.random(1, 10)/10,
      destroyed = false,
      type = 'asteroid'
    }
    setmetatable(obj, self)
    self.__index = self
    table.insert(all_objects, obj)
    return obj
  end,
  -- update script
  update = function(self)
    if self.destroyed then return nil end
    self.x = helper.calculateVX(self.x, self.angle, self.speed)
    self.y = helper.calculateVY(self.y, self.angle, self.speed)
    -- bounce at edges of screen
    if self.x < 0 or self.y < 0 or self.x > 800 or self.y > 600 then
      self.angle = self.angle * -1
    end
  end,
  -- draw script
  draw = function(self)
    if self.destroyed then return nil end
    love.graphics.setColor(0, 1, 1)
    love.graphics.rectangle('line', self.x, self.y, 16, 16)
  end,
  -- destroy script
  destroy = function(self)
    self.destroyed = true -- a 'fake' destroy, it's still in the list!
  end
}

-- bullet class
local bullet = {
  new = function(self, x, y, angle)
    local obj = {
      x = x,
      y = y,
      angle = angle,
      speed = 5,
      collided = false,
      destroyed = false,
      type = 'bullet'
    }
    setmetatable(obj, self)
    self.__index = self
    table.insert(all_objects, obj)
    return obj
  end,
  -- update script
  update = function(self, dt)
    if self.destroyed then return nil end
    self.x = helper.calculateVX(self.x, self.angle, self.speed)
    self.y = helper.calculateVY(self.y, self.angle, self.speed)
    -- check if colliding with asteroid
    local hit = helper.checkCollision(self, 'asteroid')
    if hit then
      hit:destroy()
      self:destroy()
    end
  end,
  -- draw script
  draw = function(self)
    if self.destroyed then return nil end
    love.graphics.setColor(1, 1, 0, 1)
    love.graphics.circle('fill', self.x, self.y, 2)
    love.graphics.setColor(1, 1, 1, 1)
  end,
  -- destroy script
  destroy = function(self)
    self.destroyed = true -- a 'fake' destroy, it's still in the list!
  end
}

-- player class
local player = {
  new = function(self, x, y, angle)
    local obj = {
      x = x,
      y = y,
      angle = math.rad(angle),
      speed = 150,
      hp = 100,
      collided = false,
      spr = love.graphics.newCanvas(16, 16),
      type = 'player'
    }
    -- @TEMP 
    -- this is just to draw a 'fake' sprite instead of using a texture+quad
    love.graphics.setCanvas(obj.spr)
      love.graphics.polygon('fill', 0,0, 16,8, 0,16)
    love.graphics.setCanvas()
    -- @TEMP
    setmetatable(obj, self)
    self.__index = self
    table.insert(all_objects, obj)
    return obj
  end,
  -- update script
  update = function(self, dt)
    local speed = 0
    -- a + d to change angle, w + s to move forward + backward
    if love.keyboard.isDown('a') then self.angle = self.angle + (-3 * dt) end
    if love.keyboard.isDown('d') then self.angle = self.angle + (3 * dt) end
    if love.keyboard.isDown('w') then speed = self.speed * dt end
    -- reuse the helper scripts
    self.x = helper.calculateVX(self.x, self.angle, speed)
    self.y = helper.calculateVY(self.y, self.angle, speed)
    -- see if we hit something!
    self.collided = helper.checkCollision(self, 'asteroid') ~= nil
  end,
  -- draw script
  draw = function(self)
    if self.collided then
      love.graphics.setColor(1, 0, 0, 1)
    else
      love.graphics.setColor(1, 1, 1, 1)
    end
    love.graphics.draw(self.spr, self.x, self.y, self.angle, 1, 1, 8, 8)
    love.graphics.setColor(1, 1, 1, 1)
  end,
  -- mousepress
  mousepressed = function(self)
    bullet:new(self.x, self.y, self.angle)
  end
}

-- load game
love.load = function()
  -- make a player
  local p = player:new(10, 10, 20)
  -- make some asteroids
  for a=1,30 do
    asteroid:new(love.math.random(0, 800), love.math.random(0, 600))
  end
end

-- love events
-- for all objects, check if there's an event script defined, then call it if so
love.update = function(dt)
  for i=1,#all_objects do
    if all_objects[i].update then all_objects[i]:update(dt) end
  end
end
love.draw = function()
  for i=1,#all_objects do
    if all_objects[i].draw then all_objects[i]:draw() end
  end
end
love.mousepressed = function()
  for i=1,#all_objects do
    if all_objects[i].mousepressed then all_objects[i]:mousepressed() end
  end
end
{% endlove %}


## Next Steps

Want some homework? Why not try taking this example and expand it further! 
Here's some ideas for you:
* Right now the player just flashes when they get hit, but what about taking damage? 
* If the player is taking damage, you should draw a healthbar, and have a gameover screen!
* Maybe add a controller object to handle your gameover screen and displaying a final score/leaderboard?
* You shoot asteroids, they blink out of existence, try making smaller ones appear instead and fly out from the parent
* Destroyed objects are still in our list, you should clean up the list now to let the unused one get removed from memory!
* What about adding a score for shooting asteroids?
* Having this in one big file is a bit messy! Try splitting each class out into it's own `.lua` file
* Instead of basic shapes, try adding your own texture and quads to give the game some nice sprites!
* Instead of bouncing asteroids off the edge of the boundary, you could teleport them to the opposite side!
* How about adding ammo, and some asteroids drop ammo pickups? Or ammo modifiers? Faster bullets, multi-spread...
* We stop abruptly when we stop holding 'W', how could you handle having the player gradually come to a halt?