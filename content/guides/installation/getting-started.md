---
title: "Getting started"
authors: [Nykenik24]
date: 2025-02-15
---

Hi there! LÖVE is an *awesome* framework you can use to make 2D games in Lua. It's free, open-source, and works on Windows, macOS, Linux, Android and iOS.

## Community
If you get stuck, many friendly people are ready to help you at the forums. Be warned, however, that it sometimes gets too friendly.

People also post their games and projects on [the forums](https://love2d.org/forums/), so it's a nice way of exploring what LÖVE can do. Or at least what people choose to use it for.

There is also a [Discord server](https://discord.gg/rhUets9) and a [subreddit](https://www.reddit.com/r/love2d).

Get in touch with us on twitter [@obey_love](https://twitter.com/obey_love).

## Open source
LÖVE is licensed under the liberal zlib/libpng license. This means that:

- It costs nothing.
- You can use it freely for commercial purposes with no limitations.

The source can be found on [GitHub](https://github.com/love2d/love).

## A little example

In this example we will see how to draw text to the screen!
```lua
-- in main.lua
function love.draw()
    love.graphics.print("Hello, World!", 400, 300)
end
```
This will show this:

{% love 600, 450 %}
function love.draw()
    love.graphics.print("Hello, World!", 300, 225)
end
{% endlove %}