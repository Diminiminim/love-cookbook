---
title: "VSCode"
authors: [Keyslam]
date: 2026-04-05
---

When using VSCode for LÖVE we will use Sumneko's [`Lua Language Server`](https://github.com/LuaLS/lua-language-server/wiki) and Tom Blind's [`Local Lua Debugger`](https://github.com/tomblind/local-lua-debugger-vscode).

## Prerequisites 

This guide assumes that you have [`VSCode`](https://code.visualstudio.com/) installed, and that you have [`added LÖVE to your system's path`](/guides/installation/add-to-system-path).

> [!NOTE]  
> Instead of following these steps you can also use [`Keyslam's template`](https://github.com/Keyslam/LOVE-VSCode-Starter-Template).

## Setup

Open your game's folder with VSCode and go to the extensions tab (`Ctrl + Shift + X`).  
Search for 'Lua' and install the one from Sumneko.  
Then search for 'Local Lua Debugger' and install the one by Tom Blind.

Go back to your explorer (`Ctrl + Shift + E`) and create a folder called `.vscode` at the root.  
In here we will put some configurations to make the extensions work properly for LÖVE.

### Lua Language Server

The Lua Language Server will give VSCode superpowers for working with Lua and LÖVE.  
You'll get autocomplete and hints for function calls and your own variables, and it will even warn you when you're making silly mistakes, like trying to access `myVirable` instead of `myVariable`.  
To make this work properly we need to tell the extension what our enviornment is.

Create a file inside the `.vscode` folder called `settings.json`. Inside paste the following:
```json
{
	"Lua.workspace.library": [
		"${3rd}/love2d/library",
		"lib"
	],
	"Lua.runtime.version": "LuaJIT",
	"Lua.workspace.checkThirdParty": false,
}
```

### Local Lua Debugger

With Local Lua Debugger we can launch our game with a debugger attached.  
That way when our code throws an error, instead of getting the usual LOVE errorscreen, the game will pause and VSCode will show on what line of code what went wrong.  
We can even inspect all the variables' values to get a much better understanding of what's going on in our program.  
Additionally you can set [`breakpoints`](https://code.visualstudio.com/docs/debugtest/debugging#_breakpoints), which will cause the same pausing whenever that line of code runs.  
We can use that to step through our code line by line. These features are super helpful when you're stuck trying to fix a complex issue (or debugging, as they say).

Create a file inside the `.vscode` folder called `launch.json`. Inside paste the following:
```json
{
	"version": "0.2.0",
	"configurations": [
		{
			"type": "lua-local",
			"request": "launch",
			"name": "Debug",
			"program": {
				"command": "love"
			},
			"args": [
				".",
				"debug"
			],
		},
		{
			"type": "lua-local",
			"request": "launch",
			"name": "Release",
			"program": {
				"command": "love"
			},
			"args": [
				".",
			],
		},
	]
}
```

Now when you press F5 your game should start, but the debugger won't quite work yet.
Make a `conf.lua` at the root, if you don't have one already, and at the very top put the following:
```lua
local IS_DEBUG = os.getenv("LOCAL_LUA_DEBUGGER_VSCODE") == "1" and arg[2] == "debug"
if IS_DEBUG then
	require("lldebugger").start()

	function love.errorhandler(msg)
		error(msg, 2)
	end
end
```

Now when LÖVE starts it will check if the debugger if it was started with debugging enabled, and if so attach to the debugger.

> [!NOTE]
> Any `print`s will go to the 'DEBUG CONSOLE' tab, instead of your standard terminal.

> [!NOTE]
> Running with the debugger has a small performance impact.   
> If you're profiling be sure to launch with the 'Release' configuration.  
> You can change the active configuration in the 'Run and Debug' tab (Ctrl + Shift + D)