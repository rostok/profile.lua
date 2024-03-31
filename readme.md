# Semi Flamegraph and extension to profile.lua
Let's start with what you get:
![a report](screenshot.png)

This repo expands original profile.lua by adding two functions to export an HTML and JSON file with 
profile data. Generic usage could be done like that:
```lua
local filename = "profile.txt"
local depth = 30
local json = profile.tracingJSON()
local report = profile.report(depth or 30)

local js   = profile.flameJS()
local html = profile.flameHTML(filename:gsub(".txt",".js"),report)

profile.reset()
api.writeFile(filename, report)
api.writeFile(filename:gsub(".txt",".json"), json)
api.writeFile(filename:gsub(".txt",".js"), js)
api.writeFile(filename:gsub(".txt",".html"), html)

-- set tab separated clipboard, so it can be pasted into Excel
log("clipboard set")
love.system.setClipboardText( (report.."\n"):gsub('[^\n]*%+%-[^\n]*\n', ''):gsub('|', '\t') )
```

Once profiling is done we grab report, js and put them into HTML file. Additional json file can be imported 
directly into [[chrome://tracing]]. 

Please note, that order of function calls on each stack is alphabetical as I wanted to have a statstic breakdown in order to visualize bottlenecks. The rationale to export HTML and JS data file was to refresh quickly the report and compare it visually with previous one.
By default bars are stretched horizontally but input in lower left corner can adjust scale so two reports are normalized.

# LOVE2D profiling example
a main.lua stub with upate and draw profiling 
```lua
profile = require('profile')
__profiling = {
	S = -1, -- special profiling, set above 0 will profile this number of frames, at 0 writes report, at negative does nothing 
	R = -1, -- special render/draw profiling for _draw()
	U = -1, -- update profiling 
	D = -1, -- draw profiling
	frames = 2
}
function writeFile(name, contents) local file = love.filesystem.newFile(name, "w") file:write(contents) file:close() end
-- checks if counter reached 0, if so writes report
-- decreases counter and returns its value
function profileReport(counter, filename, depth)
	if counter>=0 then print("PROFILING ",counter,"  \r") end
	if counter == 0 then
		local json = profile.tracingJSON()
		local report = profile.report(depth or 30)
		local html = profile.flameHTML(nil,report)
		print(filename)
		print(report)
		profile.reset()
		writeFile(filename, report)
		writeFile(filename:gsub(".txt",".json"), json)
		writeFile(filename:gsub(".txt",".html"), html)
		-- set tab separated clipboard
		love.system.setClipboardText( (report.."\n"):gsub('[^\n]*%+%-[^\n]*\n', ''):gsub('|', '\t') )
	end
	return counter - 1
end
local function isCtrlOrGuiDown() return love.keyboard.isDown("lctrl") or love.keyboard.isDown("lgui") or love.keyboard.isDown("rctrl") or love.keyboard.isDown("rgui") end
local function isAltDown() return love.keyboard.isDown("lalt") or love.keyboard.isDown("ralt") end
local function isShiftDown() return love.keyboard.isDown("lshift") or love.keyboard.isDown("rshift")end
-- "F9+Shift          - profile update
-- "F9+Shift+Ctrl     - profile draw
-- "F9+Shift+Alt      - profile special
-- "F9+Shift+Ctrl+Alt - profile render
function love.keypressed(key)
	if key == "f9" and isShiftDown() and not isCtrlOrGuiDown() and not isAltDown() then
		if __profiling.U<0 then 
			print('PROFILING UPDATE ('..__profiling.frames..') ...')
			__profiling.U = __profiling.frames 
		end
	elseif key == "f9" and isShiftDown() and     isCtrlOrGuiDown() and not isAltDown() then
		if __profiling.D<0 then 
			print('PROFILING DRAW ('..__profiling.frames..') ...')
			__profiling.D = __profiling.frames
		end
	elseif key == "f9" and isShiftDown() and not isCtrlOrGuiDown() and     isAltDown() then
		if __profiling.S<0 then 
			print('PROFILING SPECIAL ('..__profiling.frames..') ...')
			__profiling.S = __profiling.frames
		end
	elseif key == "f9" and isShiftDown() and     isCtrlOrGuiDown() and     isAltDown() then
		if __profiling.D<0 then 
			print('PROFILING RENDER ('..__profiling.frames..') ...')
			__profiling.R = __profiling.frames
		end
	end
end
function love.update(dt)
	if __profiling.U>0 then profile.start() end
  -- UPDATE EVERYTHING
	if __profiling.U>0 then profile.stop() end
	
	__profiling.U = profileReport(__profiling.U, "profileU.txt", 30)
	__profiling.S = profileReport(__profiling.S, "profileS.txt", 30)
end

function love.draw()
	if __profiling.D>0 then profile.start() end
  -- DRAW EVERYTHING
	if __profiling.D>0 then profile.stop() end
	
	__profiling.D = profileReport(__profiling.D, "profileD.txt", 30)
	__profiling.R = profileReport(__profiling.R, "profileR.txt", 30)
end
```

# Profile.lua
profile.lua is a small, non-intrusive module for finding bottlenecks in your Lua code.
The profiler is used by making minimal changes to your existing code.
Basically, you require the profile.lua file and specify when to start or stop collecting data.
Once you are done profiling, a report is generated, describing:
* which functions were called most frequently and
* how much time was spent executing each function

# Documentation
The full documentation is available at: https://2dengine.com/?p=profile

# Compatibility
The profiler has been tested with both LuaJIT 2.0.5 and Lua 5.3 although there are no guarantees regarding its accuracy.
Use at your own discretion!
LuaJIT optimizations must be off when using the profiler and co-routines are unsupported.

# API
## profile.report(rows)
Generates a report and returns it as a string.
"rows" limits the number of rows in the report.

## profile.start()
Starts collecting data.

## profile.stop()
Stops collecting data.
For optimal accuracy, this function should be called from code that is NOT being profiled.

## profile.reset()
Resets all collected data.

## profile.setclock(func)
Defines a custom clock function that must return a number.

# Examples
## Basic
~~~~
local profile = require("profile")
profile.start()
-- execute code that will be profiled
profile.stop()
-- report for the top 10 functions, sorted by execution time
print(profile.report(10))
~~~~

## Love2D
~~~~
-- setup
function love.load()
  love.profiler = require('profile') 
  love.profiler.start()
end

-- generates a report every 100 frames
love.frame = 0
function love.update(dt)
  love.frame = love.frame + 1
  if love.frame%100 == 0 then
    love.report = love.profiler.report(20)
    love.profiler.reset()
  end
end

-- prints the report
function love.draw()
  love.graphics.print(love.report or "Please wait...")
end
~~~~

# Reports
The default report is in plain text:
~~~~
 +-----+----------------------------------+----------+--------------------------+----------------------------------+
 | #   | Function                         | Calls    | Time                     | Code                             |
 +-----+----------------------------------+----------+--------------------------+----------------------------------+
 | 1   | update                           | 1        | 9.0023296745494          | main.lua:23                      |
 | 2   | f                                | 1        | 9.0022503120126          | main.lua:12                      |
 | 3   | g                                | 8        | 8.0016986143455          | main.lua:5                       |
 | 4   | [string "boot.lua"]:185          | 3        | 2.4960798327811e-005     | [string "boot.lua"]:185          |
 | 5   | [string "boot.lua"]:134          | 2        | 1.7920567188412e-005     | [string "boot.lua"]:134          |
 | 6   | [string "boot.lua"]:188          | 1        | 1.6000514733605e-005     | [string "boot.lua"]:188          |
 | 7   | [string "boot.lua"]:182          | 1        | 1.2160395272076e-005     | [string "boot.lua"]:182          |
 | 8   | [string "boot.lua"]:131          | 1        | 1.0240328265354e-005     | [string "boot.lua"]:131          |
 | 9   | load                             | 0        | 0                        | main.lua:17                      |
 +-----+----------------------------------+----------+--------------------------+----------------------------------+
~~~~

It's easy to generate any type of report that you want, for example CSV:

~~~~
print('Position,Function name,Number of calls,Time,Source,')
for t in ipairs(profiler.query(10)) do
  print(table.concat(t, ",")..",")
end
~~~~

# Credits
0x25a0
grump
Roland Yonaba