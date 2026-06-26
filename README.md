local global = getfenv().getgenv and getfenv().getgenv() or _G
local kp, kr = getfenv().keypress, getfenv().keyrelease
local key = "FFAutoplayLib"

if global[key] then
	return global[key]
end

local settings = {
	Version = "1.28", -- autoplayer version

	AutoPlay = false,
	PerfectSick = 0, -- 0 to 1, 0 = off, values above 1 can cause issues
	CopyEnemyNotes = false, -- I find this stupid
	UseScrollSpeedBuffer = true, -- used for more precise scroll speed calculation
	ScrollSpeedBufferSize = 1.25,
	DetectSV = true, -- detects SV songs and applies own fixes so it never misses/hits anything but sick
	HitOffset = 0, -- seconds, can be negative
	Performance = 2, -- 0 - 7. More value = less lags

	MaxKPSPerKey = 0, -- 0 or less = inf
	MaxKPS = 0, -- same here; MaxKPS limits keys per second for ALL keys, while MaxKPSPerKey limits keys per second for each key

	HoldDuration = 0.1,
	HoldDurationRandom = 0, -- both positive and negative

	MoreStats = false, -- if true, stats will have autoplayer stats also, if string, it will act as its "true", but will contain that one string on top of the stats

	Side = "Left", -- read only
	Playing = false, -- read only
	FPS = 0, -- approximated fps value, read only
	NotesRendered = 0, -- how many notes there currently is, read only
	KPS = 0, -- read only
	NotesVisible = 0, -- how many notes are visible right now (this can be lower than rendered, because of performance option), read only
	ScrollSpeed = 0, -- is not game's scroll speed, is calculated scroll speed; usually that value 3-6 times bigger than game's scrollspeed value, read only
	DownScroll = false, -- on ModChart songs it sometimes can freak out, read only
	Lanes = 0, -- e.g. how much keys are in the song (4, 5, 6, 7, 8, 9), read only
	RenderDelta = 0, -- a literal (tick() - start) time between RenderStepped events, read only
	MyNotesRendered = 0, -- you know the drill, read only
	EnemyNotesRendered = 0, -- same here, read only
	MyNotesVisible = 0, -- read only
	EnemyNotesVisible = 0, -- read only
	IsModChartSong = false, -- is modchart song, read only
	IsSVSong = false, -- is SV song, read only
	Rate = 1, -- song's rate (speed), read only
	TimeLeft = 0, -- song's time left, read only
	SongName = "", -- read only
	TotalNotes = 0, -- how many notes been generated in our side and (if enabled in game's settings) enemy's side, read only
	MyTotalNotes = 0, -- same as previous, but for your side, read only
	EnemyTotalNotes = 0, -- read only
	StaticHitOffset = 0, -- seconds, not implemented yet, read only
	SongDuration = 0, -- how long the song is, read only
	SongRealDuration = 0, -- same as previous value, but also is multiplied by rate, read only
	SongNameColor = Color3.new(), -- useless thing, read only
	SongDifficulty = "", -- e.g. Easy, Normal, Hard, Lunatic, etc, read only

	Chances = {
		Sick = 100,
		Good = 0,
		Ok = 0,
		Bad = 0, -- wont always hit the "Bad", very often it will hit the `Ok`
		Miss = 0 -- sometimes when `Miss` note and any other note are coming close, they both will be hit
	}
}

--[[ Extra readonly values:
Keybinds = { string? }
Events = { [string] = Event } -- a table that stores next events:

NoteAdded = Event(instance, isMine: boolean, isRegularNote: boolean)
NoteRemoved = Event(instance, isMine: boolean, isRegularNote: boolean)

KeybindsChanged = Event({ string })

GameStarted = Event()
GameEnded = Event()

Message = Event(string)

SettingChanged = Event(string, any) -- settings from the settings table below, does not apply for readonly values
Changed = Event(string, any, isReadOnly: bool) -- same as previous one, but being fired ALSO for readonly values (e.g. normal values are being fired also)

Supported = { [string]: number } -- scroll down in that code to see what it stores exactly

----

print(settings.Keybinds[1]) -- will print the first keybind, if has one, else nil

----

Each Event has
	:Connect(func) -> Connection
	:Once(func) -> Connection
	:Wait() -> Any -- uses task.wait(), so its not instant

Each Connection has
	.Connected : bool
	.Enabled : bool
	.Callback : function
	:Disconnect() -> never
]]

local readonlyStats = (table and table.freeze or function(t) return t end)({ "Side", "MyTotalNotes", "EnemyTotalNotes", "StaticHitOffset", "TotalNotes", "Playing", "FPS", "NotesRendered", "KPS", "SongName", "NotesVisible", "ScrollSpeed", "MyNotesRendered", "EnemyNotesRendered", "EnemyNotesVisible", "MyNotesVisible", "DownScroll", "Lanes", "RenderDelta", "IsSVSong", "IsModChartSong", "Rate", "TimeLeft", "SongDuration", "SongRealDuration", "SongDifficulty" })

local fps, estFps, lastDelta = 0, 0, 0
local psick = false
local ap = false
local maxkpspk, maxkps = 0, 0
local hd, hdr = 0, 0
local scrollSpeed = 0
local cn = false
local perf = 0
local chances = settings.Chances
local rendered, total = 0, 0
local renderedOnLanes = { }
local renderedOnEnemyLanes = { }
local speedBuffer = { }
local ussb = true
local ssbs = 1.25
local speed = 0
local rate = 1
local prevSpeed = false
local SVMode = false
local isModChart = false
local sv = false
local SV = false
local SVDetectAttempts = 0
local ho = 0
local HO = 0

local tick = tick
local game, workspace = game, workspace
local _wait, spawn = task.wait, task.spawn
local max, min, clamp, abs, random, round = math.max, math.min, math.clamp, math.abs, math.random, math.round
local inf = 1 / 0
local insert, remove, find, clone, sort, concat = table.insert, table.remove, table.find, --[[table.clone]] function(t) local copy = { } for i, v in t do copy[i] = v end return copy end, table.sort, table.concat
local v2, c3 = vector and vector.create or Vector2.new, Color3.new
local ipairs = ipairs
local tonumber = tonumber
local pcall = pcall
local request = getfenv().request

local function toN(n)
	return tonumber(n) or n
end

local lastOffset = 0
local function wait(t)
	t = max(tonumber(t) or 0, 0)
	local took = _wait(t)

	if t == 0 then
		lastOffset = took - t
	end

	return took
end

local oldSettings = clone(settings)
local function gHTTPG(url)
	return game:HttpGet(url)
end

local function httpGet(url)
	if request then
		local result = request({ Url = url, Method = "GET", Headers = { } })
		local success = result.Success or tostring(result.StatusCode):sub(1, 1) == "2"
		return success and result.Body or "", success
	else
		local s, e = pcall(gHTTPG, url)
		return s and e or "", s
	end
end

local function urlLoad(url)
	local ret
	while wait() do
		ret = nil

		local r, s = httpGet(url)
		if s then
			ret = loadstring(r)

			if ret then
				ret = ret()

				if ret then
					break
				end
			end
		end
	end

	return ret
end

local newEvent = urlLoad("https://raw.githubusercontent.com/Null-Cherry/Utilities/refs/heads/main/Event/Main.lua")

if global[key] then
	return global[key]
end

local settingChanged = newEvent()
local changed = newEvent()

local note = newEvent()
local noteRemoved = newEvent()
local gameStarted = newEvent()
local gameEnded = newEvent()
local message = newEvent()
local keybindsChanged = newEvent()

local guiServ = game:GetService("GuiService")
local uis = game:GetService("UserInputService")
local escOpen, buzy = guiServ.MenuIsOpen, uis:GetFocusedTextBox() ~= nil

local buzyWarn, escWarn = pcall, pcall

guiServ.MenuClosed:Connect(function()
	escOpen = false
end)

guiServ.MenuOpened:Connect(function()
	escOpen = true
	escWarn()
end)

uis.TextBoxFocused:Connect(function(tb)
	buzy = true

	if buzyWarn() then
		tb:ReleaseFocus()
		buzy = false
	end
end)

uis.TextBoxFocusReleased:Connect(function()
	wait()
	buzy = false
end)

settingChanged:Connect(function(setting, value)
	if setting == "AutoPlay" then
		ap = value
	elseif setting == "MaxKPSPerKey" then
		maxkpspk = value
	elseif setting == "MaxKPS" then
		maxkps = value
	elseif setting == "Chances" then
		chances = value
	elseif setting == "CopyEnemyNotes" then
		cn = value
	elseif setting == "HoldDuration" then
		hd = abs(value)
	elseif setting == "HoldDurationRandom" then
		hdr = abs(value)
	elseif setting == "Performance" then
		perf = value
	elseif setting == "UseScrollSpeedBuffer" then
		ussb = value
		speedBuffer = { speed }
	elseif setting == "ScrollSpeedBufferSize" then
		ssbs = value
	elseif setting == "DetectSV" then
		sv = value
	elseif setting == "HitOffset" then
		ho = value
	end
end)

local events = { -- not actually only events
	Message = message,

	NoteAdded = note,
	NoteRemoved = noteRemoved,

	KeybindsChanged = keybindsChanged,

	SettingChanged = settingChanged, -- (setting, value)
	Changed = changed, -- same as previous one, but applies also for readonly values

	GameStarted = gameStarted,
	GameEnded = gameEnded,

	ReadOnlyStats = readonlyStats,

	Supported = (table and table.freeze or function(t) return t end)({ -- 3 = supported, 2 = kinda supported, 1 = poorly supported, 0 = not supported
		["SV"] = 2,
		["Mod Charts"] = 3,
		["Multi-Key"] = 3,
		["60 FPS+"] = 3,
		["Low FPS"] = 3,
		["Down & Up scroll"] = 3,
		["Start Auto Play mid-song"] = 0
	})
}

local function getUnrepeated(data)
	local n = #data
	local assignment = { }
	local used = { }

	local sortedIndices = { }
	for i = 1, n do sortedIndices[i] = i end

	sort(sortedIndices, function(a, b) return #data[a] < #data[b] end)

	local backtrack; backtrack = function(idx)
		if idx > n then return true end

		local setIdx = sortedIndices[idx]
		local candidates = { }

		for _, val in ipairs(data[setIdx]) do
			if not used[val] then
				insert(candidates, val)
			end
		end

		sort(candidates)

		for _, val in ipairs(candidates) do
			assignment[setIdx] = val
			used[val] = true

			if backtrack(idx + 1) then
				return true
			end

			used[val] = nil
			assignment[setIdx] = nil
		end

		return false
	end

	if backtrack(1) then
		local result = { }
		for i = 1, n do result[i] = assignment[i] end

		return result
	end

	return false
end

local function rollChance()
	local total = 0

	for _, weight in chances do
		total += weight
	end

	local r = random() * total
	local sum = 0

	for k, weight in chances do
		sum += weight
		if r <= sum then
			return k
		end
	end

	return "Sick"
end

local timeOut = 9e9
local timePassed = 0
local plr = game:GetService("Players").LocalPlayer
local pgui = plr:WaitForChild("PlayerGui", timeOut)
local rs = game:GetService("RunService")
local kk = Enum.KeyCode
local vim = game:GetService("VirtualInputManager")
local rse = rs.RenderStepped
local isMobile = uis.TouchEnabled and not uis.KeyboardEnabled

local topLabel
local function decodeTime(time)
	local split = time:split(":")
	if #split == 1 then return tonumber(time) end

	while #split < 4 do
		insert(split, 1, 0)
	end

	return (split[1] * 86400) + (split[2] * 3600) + (split[3] * 60) + split[4]
end

spawn(function()
	local unpack = unpack or table.unpack
	local c3r, c3h = Color3.fromRGB, Color3.fromHex
	
	topLabel = pgui:WaitForChild("GameGui", timeOut):WaitForChild("Screen", timeOut):WaitForChild("TopLabel", timeOut):WaitForChild("Label", timeOut)
	topLabel.Changed:Connect(function()
		local text = topLabel.Text
		local splitted = text:split("\n")

		if #splitted == 3 then
			local l1, l3 = splitted[1], splitted[3]
			if l1:sub(-2, -1) == "x)" then
				local endBracket = -3
				while true do
					if l1:sub(endBracket, endBracket) == "(" then
						rate = tonumber(l1:sub(endBracket + 1, -3))
						settings.Rate = rate
						break
					end

					endBracket -= 1
					if -endBracket > #l1 then
						settings.Rate = 1
						break
					end
				end
			else
				settings.Rate = 1
			end
			
			local difficultyStart = l1:find("</font> (", 0, true)
			if difficultyStart then
				local difficultyEnd = l1:find(")", difficultyStart, true)
				settings.SongDifficulty = l1:sub(difficultyStart + 9, difficultyEnd - 1)
			end
			
			local colorStart = l1:find("><font color='", 0, true)
			if colorStart then
				local colorEnd = l1:find("'>", colorStart, true) + 1
				local color = l1:sub(colorStart + 14, colorEnd - 2):gsub("\n\r\f\t\s\0 ", ""):lower()
				
				if color:sub(1, 3) == "rgb" then
					settings.SongNameColor = c3r(unpack(color:sub(5, -2):split(",")))
				elseif color:sub(1, 1) == "#" then
					settings.SongNameColor = c3h(color:sub(2, -1))
				end
			end

			settings.SongName = l1:sub((l1:find(")'>", 0, true)) + 3, (l1:find("</f", 0, true)) - 1)

			local s, r = pcall(decodeTime, l3:sub(1, -10))
			settings.TimeLeft = s and r or 0
			settings.SongDuration = max(settings.TimeLeft, settings.SongDuration, 0)
			settings.SongRealDuration = round(settings.SongDuration * settings.Rate)
			
			timePassed = round((settings.SongDuration - settings.TimeLeft) * settings.Rate)
		end
	end)
end)

local function r(times)
	local dt = 0

	for i = 1, max(round(tonumber(times) or 1), 1) do
		local s = tick()
		rse:Wait()

		dt += tick() - s
	end

	return dt - (tick() - tick())
end

local function getAverage(t)
	local avg = 0
	local l = #t

	for i = 1, l do
		avg += t[i]
	end

	avg /= l
	return avg ~= avg and 0 or avg
end

local function append(table, value, size)
	local size = round(max(tonumber(size) or 0, 1))

	while #table >= size do
		remove(table, #table)
	end

	insert(table, 1, value)
	return getAverage(table)
end

local fpsBuffer = { }
local readonlyLookup = { }
local oldChances = clone(settings.Chances)
local SVPS

for _, v in readonlyStats do
	readonlyLookup[v] = true
end

spawn(function()
	local SVPSDefault = 1.5
	local mul = 1.12
	local SVPSMax = SVPSDefault * mul

	SVPS = SVPSDefault

	local function calcSVPSDef(rate)
		local res = min(SVPSDefault / (rate ^ (rate ^ rate)), 1.4)
		if res ~= res then
			res = SVPSDefault
		end

		return res
	end

	local preCalc = calcSVPSDef(2.1)
	local function calcSVPS()
		local res = calcSVPSDef(rate)
		if rate <= 1.8 then
			return res
		end

		return max(res - preCalc, 0)
	end

	while true do
		lastDelta = r()
		fps = 1 / lastDelta
		psick = fps > 20 and settings.PerfectSick or 0
		SV = sv and SVMode
		SVPS = calcSVPS()
		HO = ho - ((SV or isModChart) and 0.0125 or 0) - 0.0075

		estFps = round(append(fpsBuffer, fps, fps) * 10) / 10

		settings.FPS = estFps
		settings.RenderDelta = lastDelta

		local Changed = false
		local chancesChanged = false

		for i, v in settings do
			if oldSettings[i] ~= v then
				local isReadOnly = readonlyLookup[i]
				if not isReadOnly then
					settingChanged:Fire(i, v)
				end

				changed:Fire(i, v, isReadOnly)
				Changed = true
				chancesChanged = chancesChanged or i == "Chances"
				oldSettings[i] = v
			end
		end

		if not chancesChanged then
			local chances = settings.Chances
			Changed = false

			for i, v in chances do
				if oldChances[i] ~= v then
					Changed = true
					oldChances[i] = v
				end
			end

			if Changed then
				settingChanged:Fire("Chances", chances)
				changed:Fire("Chances", chances, false)
			end
		end
	end
end)

local function getDistance(a, b)
	return (a - b).Magnitude
end

local function getClosest(toIterate)
	if not plr.Character then return end

	local c, d = nil, inf

	for _, v in toIterate:GetChildren() do
		local m = getDistance(v:GetPivot().Position, plr.Character:GetPivot().Position)
		if m < d then
			c, d = v, m
		end
	end

	return c, d
end

local function getMyStage()
	if workspace:FindFirstChild("Map") and workspace.Map:FindFirstChild("Stages") then
		return getClosest(workspace.Map.Stages)
	end
end

local function getMySide()
	local stage = getMyStage()
	if stage then
		local node = getClosest(stage.Nodes)
		if node then
			return node.Name:sub(1, node.Name:find("_") - 1)
		end
	end
end

local side = getMySide() or "Left"
local lanes = 0
local isDownScroll = false
local downscrollPropability = 0
local power = 3 -- idc

local hit = { }

local useX = true
local function UDimToVector2(ud)
	return v2(useX and ud.X.Scale or 0, ud.Y.Scale, 0)
end

local kbVals = { }
local kbs = { }
local lastKbs = concat(kbs, ",")

events.Keybinds = kbs

local function refreshKbs(t)
	wait(t or 0)
	kbs = getUnrepeated(kbVals)

	local newKbs = concat(kbs, ",")
	if newKbs ~= lastKbs then
		lastKbs = newKbs
		settingChanged:Fire("Keybinds", kbs)

		events.Keybinds = kbs
		events.KeybindsChanged:Fire(kbs)
	end
end

local function raceEvents(events, timeout)
	timeout = timeout or 1

	local last = tick()
	local cons = { }
	local winner

	for i, v in events do
		cons[#cons + 1] = v:Once(function()
			winner = i

			for idx, val in cons do
				val:Disconnect()
			end
		end)
	end

	repeat wait() until winner or tick() - last > timeout

	winner = winner or 0
	for idx, val in cons do
		val:Disconnect()
	end

	return winner
end

local labelAdded; labelAdded = function(laneNum, label, cons)
	local textL = label:WaitForChild("Text", timeOut)
	kbVals[laneNum] = kbVals[laneNum] or { }

	local myIdx = #kbVals[laneNum] + 1
	kbVals[laneNum][myIdx] = textL.Text

	cons[#cons + 1] = textL:GetPropertyChangedSignal("Text"):Connect(function()
		if textL.TextTransparency >= 0.95 then return end
		kbVals[laneNum][myIdx] = textL.Text
		refreshKbs(0)
	end)
end

local rolled = { }
local badNotes = { }
local badNoteAssets = { "rbxassetid://103483801062498", "rbxassetid://88530467220950", "rbxassetid://109130876544260", "rbxassetid://120222801097284", "rbxassetid://101951481332606" }
local actuallyVisible = 0

local gpcsCache = { }
local function gpcs(v, n)
	local cache = gpcsCache[toN(v.Name)] or { }
	gpcsCache[v] = cache

	local got = cache[n] or v:GetPropertyChangedSignal(n)
	cache[n] = got

	return got
end

local function canRenderNote(lane, renderedOnLanes)
	local p3 = perf - 3
	local p = p3 == 0 and 0.67 or p3 == -1 and 0.42
	p3 = p and 1 / (p * 2) or p3
	
	return renderedOnLanes[lane] <= (64 // p3) // scrollSpeed and (renderedOnLanes[lane] <= (16 // p3) // scrollSpeed or renderedOnLanes[lane] % ((8 * p3) * scrollSpeed // 1) == 0)
end

local dummy = { }
local ms, es = "My", "Enemy"
local no, to = "Notes", "TotalNotes"
local black = c3()

local function noteAdded(v, notest, mine, lane)
	local sprite = v:WaitForChild("LayeredSprite", timeOut):WaitForChild("1", timeOut)
	local isGood = sprite.ImageColor3 ~= black and not find(badNoteAssets, sprite.Image)
	local toInsert
	if isGood then
		toInsert = notest
		rolled[v] = SV and "Sick" or rollChance()
	elseif mine then
		toInsert = badNotes[lane] or { }
		badNotes[lane] = toInsert
	else
		toInsert = dummy
	end

	note:Fire(v, mine, lane, isGood)
	insert(toInsert, v)

	local renderedOnLanes = mine and renderedOnLanes or renderedOnEnemyLanes

	renderedOnLanes[lane] = renderedOnLanes[lane] or 0
	local visible = perf < 7 and (perf >= 2 and (mine or perf < 7) and canRenderNote(lane, renderedOnLanes) or perf < 3)
	local val = visible and 1 or 0
	local m = mine and ms or es
	local idx = m .. no

	renderedOnLanes[lane] += val
	total += 1
	settings.TotalNotes += 1
	settings[m .. to] += 1

	settings[idx .. "Rendered"] += 1
	settings.NotesRendered += 1
	rendered += 1
	actuallyVisible += val
	settings.NotesVisible += val
	settings[idx .. "Visible"] += val

	v.Visible = visible

	gpcs(v, "Parent"):Wait()

	noteRemoved:Fire(v, mine, lane, isGood)

	renderedOnLanes[lane] -= val
	settings[idx .. "Rendered"] -= 1
	settings.NotesRendered -= 1
	rendered -= 1	
	actuallyVisible -= val
	settings.NotesVisible -= val
	settings[idx .. "Visible"] -= val

	local found = find(toInsert, v)
	if found then
		remove(toInsert, found)
	end
end

local function sortF(a, b)
	local cond = a.Position.Y.Scale > b.Position.Y.Scale
	if isDownScroll then
		cond = not cond
	end

	return cond
end

local function laneAdded(lane, isMine, notest, cons, receptors)
	local laneNum = tonumber(lane.Name:sub(5))
	if not laneNum then return end

	notest[laneNum] = { }
	notest = notest[laneNum]

	local receptor = lane:WaitForChild("Receptor", timeOut)

	receptors[laneNum] = receptor

	if isMine then
		lanes = max(lanes, laneNum)
		settings.Lanes = lanes

		wait(); r()

		if lanes == laneNum then
			local allNotes = { }
			local side = isMine

			for i = 1, lanes do
				local lane = side:FindFirstChild("Lane" .. i)
				if lane then
					local notes = lane:FindFirstChild("Notes")
					if notes then
						for _, v in notes:GetChildren() do
							insert(allNotes, v)
						end
					end
				end
			end

			local n = #allNotes
			if n ~= 0 then
				local topOnes = 0

				for _, v in allNotes do
					if v.Position.Y.Scale < receptor.Position.Y.Scale + 0.5 then
						topOnes += 1
					end
				end

				isDownScroll = topOnes / n > 0.5
				downscrollPropability = isDownScroll and 1 or -1
				settings.DownScroll = isDownScroll
			end
		end
	end

	local notes = lane:WaitForChild("Notes", timeOut)
	local children = notes:GetChildren()

	if #children ~= 0 then
		local unsorted = { }
		for _, v in children do
			insert(unsorted, v)
		end

		sort(unsorted, sortF)
		for i = 1, #unsorted do
			spawn(noteAdded, unsorted[i], notest, not not isMine, laneNum)
		end
	end

	cons[#cons + 1] = notes.ChildAdded:Connect(function(v)local downscroll = v.Position.Y.Scale < receptor.Position.Y.Scale + 0.5
		downscrollPropability = clamp((downscroll and power or -power) + downscrollPropability, -1, 1)
		isDownScroll = downscrollPropability > 0
		settings.DownScroll = isDownScroll

		local found = find(notest, v)
		if found then
			remove(notest, found)
		end

		noteAdded(v, notest, not not isMine, laneNum)
	end)

	local labels = lane:WaitForChild("Labels", timeOut)
	for i, v in labels:GetChildren() do
		spawn(labelAdded, laneNum, v, cons)
	end

	cons[#cons + 1] = labels.ChildAdded:Connect(function(v)
		spawn(labelAdded, laneNum, v, cons)
	end)

	spawn(refreshKbs, 0)
end

local offsets = {
	Sick = 0.05,
	Good = 0.1,
	Ok = 0.15,
	Bad = 0.2,
	Miss = 0.3
}

events.Offsets = (function()
	local cloned = clone(offsets)
	cloned.Miss = nil
	;(table.freeze or function() end)(cloned)
	
	return cloned
end)()

local downKeys, keys = { }, { }
local ske = vim.SendKeyEvent

local KPS = 0
local kps = { }

local function press(key, isDown)
	ske(vim, isDown, key, false, game)
end

local function kpsP(key)
	kps[key] += 1
	KPS += 1
	settings.KPS += 1

	wait(1)

	kps[key] -= 1
	KPS -= 1
	settings.KPS -= 1
end

local function kpsCount(key, mine)
	if maxkps > 0 and KPS > maxkps then return true end

	kps[key] = kps[key] or 0
	if maxkpspk > 0 and kps[key] > maxkpspk then return true end

	spawn(kpsP, key, mine)
end

local myKPS = 0
local function pressKey(key, duration, mine)
	local kk = kk:FromName(key)
	if downKeys[key] then
		downKeys[key] = false
		press(kk, false)
	end

	local val = mine and 1 or 0
	if kpsCount(key, mine) or escOpen or buzy then return end

	local myId = (keys[key] or 0) + 1
	keys[key] = myId
	myKPS += val

	press(kk, true)

	if duration and duration > 0 then
		downKeys[key] = true
		wait(duration)
	end

	if keys[key] == myId then
		downKeys[key] = false
		press(kk, false)
	end

	myKPS -= val
end

local function isBehind(x, y)
	local is = x.Y > y.Y
	if isDownScroll then
		is = not is
	end

	return is
end

local sickOffset  = offsets.Sick
local sickOffset2 = offsets.Sick / 1.35
local missOffset  = offsets.Miss
local missOffset2 = offsets.Miss * 2
local badOffset   = offsets.Bad
local goodOffset  = offsets.Good

local lastJump = 0
local canHit; canHit = function(note, receptor, isBadNote, laneIndex)
	local x = UDimToVector2(receptor.Position) + v2(useX and 0.5 or 0, 0.5, 0)
	local y = UDimToVector2(note.Position)

	local dist = getDistance(x, y) / speed
	local behind = isBehind(x, y)

	if behind then
		dist += HO
	else
		dist -= HO
	end

	if dist < 0 then
		dist = -dist
		behind = not behind
	end

	if isBadNote then
		return dist, behind
	else
		local bad = badNotes[laneIndex]
		local forceSick = false

		if bad then
			for i, v in bad do
				local d, b = canHit(v, note, true, laneIndex)
				forceSick = forceSick or b and d <= missOffset
				if b and d < missOffset or not b and d < dist then
					return false, behind, dist, false, false
				end
			end
		end

		if behind then
			return dist <= badOffset, true, dist, dist <= sickOffset, false
		end

		local rolled = not SV and not forceSick and rolled[note] or "Sick"
		local sick = rolled == "Sick"

		return rolled ~= "Miss" and dist <= offsets[rolled], false, dist, sick, SV and SVPS or forceSick
	end
end

local one = UDim2.fromScale(1, 1)
spawn(function()
	while true do
		wait()
		if not settings.Playing then
			side = getMySide() or side
			settings.Side = side
		end
	end
end)

local function sortLane(lane)
	sort(lane, sortF)
end

local longNoteIndex
local function calc(v)
	return abs(v.Size.Y.Scale / speed) + ((0.075 * rate) + 0.025)
end

local function hitNote(note, key, dist, sick, lane, force, mine, isBehind)
	local s = SV and SVPS or force or psick
	if sick and s > 0 then
		local t = isBehind and (s <= 1 and 0 or (sickOffset - dist) * (s - 1) - (0.001 + (lastOffset * 2))) or (s <= 1 and (dist * s) - (lastOffset / 2) or (dist + (sickOffset * (s - 1)) - (0.001 + (lastOffset * 2))))
		if t > 0 then
			wait(t)
		end
	end

	if hit[note] then return end
	hit[note] = true

	local time = hd
	if hdr > 0 then
		time += (random() - 0.5) * 2 * abs(hdr)
	end

	local holdTime = 0
	local children = note:GetChildren()
	if not longNoteIndex then
		for i, v in children do
			if v and v.Size ~= one then
				holdTime = calc(v)
				longNoteIndex = i
				break
			end
		end
	else
		local v = children[longNoteIndex]
		if v and v.Size ~= one then
			holdTime = calc(v)
		end
	end

	time += holdTime
	if holdTime ~= 0 then
		pressKey(key, time > 0 and time < 1000 and time, mine)
	else
		spawn(pressKey, key, time > 0 and time, mine)
	end

	local winner = raceEvents({ gpcs(note, "Parent"), gpcs(note, "Position") }, 0)
	hit[note] = false

	if winner ~= 0 and note.Parent and not find(lane, note) then
		insert(lane, 1, note)
		spawn(sortLane, lane)
	end
end

local function hitLane(lane, laneIndex, receptor, mine)
	if not receptor then return end

	local key = kbs[laneIndex]
	local meetYouAgain
	local maxIterations = #lane * (rendered > (30 / scrollSpeed) and 10 or 3)
	local iterations = 0

	while #lane ~= 0 and key and iterations < maxIterations do
		iterations += 1

		local note = lane[1]
		if note == meetYouAgain then break end

		local can, far, dist, sick, force = canHit(note, receptor, false, laneIndex)
		if can then
			remove(lane, 1)
			spawn(hitNote, note, key, dist, sick, lane, force, mine, far)
		elseif not far then
			if meetYouAgain then
				local pos = find(lane, meetYouAgain)
				if pos then
					insert(lane, 1, remove(lane, pos))
				end
			end

			break
		elseif not meetYouAgain then
			meetYouAgain = note
			insert(lane, remove(lane, 1))
		end
	end
end

local function notif(text)
	while #message._Connections == 0 do wait() end
	message:Fire(text)
end

local function msg(text)
	spawn(notif, text)
end

local ssMul = 1 / 5 * 100
local function swap(val)
	val[1] = v2(-(val[1].X - 0.5) + 0.5, -(val[1].Y - 0.5) + 0.5, 0)
	val[3] = v2(-val[3].X, -val[3].Y, 0)
end

local function isDownS(notes, receptors)
	local topOnes = 0
	local allNotes = 0

	for laneIndex, lane in notes do
		local receptor = receptors[laneIndex]

		if receptor then
			allNotes += #lane

			for _, note in lane do
				if note.Position.Y.Scale < receptor.Position.Y.Scale + 0.5 then
					topOnes += 1
				end
			end
		end
	end

	return topOnes / allNotes > 0.5
end

local function SVC()
	SVDetectAttempts += 1
	SVMode = SVDetectAttempts >= 3
	settings.IsSVSong = settings.IsSVSong or SVMode
	wait(20)
	SVDetectAttempts = SVDetectAttempts > 0 and SVDetectAttempts - 1 or 0
	SVMode = SVDetectAttempts >= 3
end

local lastJump
local function calculateNotes(notes, receptors, mine)
	local startPositions = { }

	for laneIndex, lane in notes do
		local receptor = receptors[laneIndex]

		if receptor then
			for _, note in lane do
				insert(startPositions, { UDimToVector2(note.Position), note, UDimToVector2(receptor.Position), receptor, lane, laneIndex })
				break
			end
		end
	end

	local delta = r()
	if #startPositions == 0 then return end -- no notes

	local gotSpeed = 0
	local dlt = min(delta, 0.1)

	if not isModChart and ap then
		for _, v in startPositions do
			if getDistance(UDimToVector2(v[4].Position), v[3]) / dlt > 0.1 then
				isModChart = true
				break
			end
		end
	end

	if isModChart and ap then
		local isDown = isDownS(notes, receptors)

		if isDownScroll ~= isDown then
			for i, v in startPositions do
				swap(v)
			end
		end

		isDownScroll = isDown
		settings.DownScroll = isDownScroll
	end

	for _, v in startPositions do
		gotSpeed += getDistance(UDimToVector2(v[2].Position) - (UDimToVector2(v[4].Position) - v[3]), v[1]) / delta
	end

	local resultSpeed = gotSpeed / #startPositions
	speedBuffer[1] = #speedBuffer == 0 and resultSpeed or speedBuffer[1]

	speed = (SV or ussb) and append(speedBuffer, resultSpeed, fps * (SV and 3 or ssbs)) or resultSpeed
	scrollSpeed = round(speed * ssMul) / 100

	if not prevSpeed then
		prevSpeed = resultSpeed
	elseif timePassed <= 50 then
		local jump = abs(resultSpeed - prevSpeed) * delta
		prevSpeed = resultSpeed

		lastJump = jump
		if not isModChart and jump > ((timePassed <= 30 and 0.135 or 0.167) + ((lanes - 4) / 125)) then
			spawn(SVC)
		end
	end

	settings.ScrollSpeed = speed
	settings.IsModChartSong = isModChart

	for _, v in startPositions do
		spawn(hitLane, v[5], v[6], v[4], mine)
	end
end

local function mainLoop(fields, window, dontStartAutoplay)
	local mySide = fields[side]:WaitForChild("Inner", timeOut)
	local enemySide = fields[side == "Left" and "Right" or "Left"]:WaitForChild("Inner", timeOut)

	local myNotes = { }
	local enemyNotes = { }
	local myReceptors = { }
	local enemyReceptors = { }

	local cons = { }

	for i, v in mySide:GetChildren() do
		spawn(laneAdded, v, mySide, myNotes, cons, myReceptors)
	end

	cons[#cons + 1] = mySide.ChildAdded:Connect(function(v)
		laneAdded(v, mySide, myNotes, cons, myReceptors)
	end)

	for i, v in enemySide:GetChildren() do
		spawn(laneAdded, v, false, enemyNotes, cons, enemyReceptors)
	end

	cons[#cons + 1] = enemySide.ChildAdded:Connect(function(v)
		laneAdded(v, false, enemyNotes, cons, enemyReceptors)
	end)

	if not dontStartAutoplay then
		spawn(refreshKbs, 0)
	end

	while true do
		if not window.Parent then break end

		if ap and not dontStartAutoplay then
			if cn and myKPS == 0 and speed ~= 0 then
				local iHaveNotes = false
				for laneIndex, v in myNotes do
					local rec = myReceptors[laneIndex]
					if rec then
						for _, note in v do
							if canHit(note, rec, true) <= missOffset2 then
								iHaveNotes = true
								break
							end
						end

						if iHaveNotes then break end
					end
				end

				calculateNotes(iHaveNotes and myNotes or enemyNotes, iHaveNotes and myReceptors or enemyReceptors, iHaveNotes)
			else
				calculateNotes(myNotes, myReceptors, true)
			end
		else
			r()
		end
	end

	for i, v in cons do
		v:Disconnect()
	end
end

local function statsAdded(stats, cons)
	if stats:FindFirstChild("Title") then return end

	local row = stats:WaitForChild("Combo", timeOut):Clone()
	local rows = { }
	local totalRows = 0

	local function addRow(name, isFirst)
		local safeName = name:gsub(" ", "")
		local row = row:Clone()
		row.Name = safeName
		row.LayoutOrder = isFirst and -999 or 999 + totalRows
		row.Parent = stats

		totalRows += 1

		local label = row.Label
		label.Text = name .. ": null"

		local fn = function(_, value, prefix)
			row.Visible = true
			if typeof(value) == "string" then
				label.Text = value
			elseif tonumber(value) then
				label.Text = name .. ": " .. (prefix or "") .. value
			else
				row.Visible = false
			end
		end

		rows[safeName] = fn
		return fn
	end

	addRow("Title", true)
	addRow("Total Notes")
	addRow("Rendered")
	addRow("Autoplay KPS")
	addRow("FPS")

	cons[#cons + 1] = rse:Connect(function()
		local hs = settings.MoreStats
		if hs then
			rows:Title(typeof(hs) == "string" and hs or false)
			rows:Rendered((perf < 4 or actuallyVisible == rendered) and rendered or rendered and "Rendered: " .. actuallyVisible .. " (" .. rendered .. ")" or "Rendered: ?")
			rows:TotalNotes(total, "~")
			rows:AutoplayKPS(KPS)
			rows:FPS("FPS: <font color=\"#" .. (estFps < 60 and c3(1):Lerp(c3(0, 1), estFps / 60) or estFps < 120 and c3(0, 1):Lerp(c3(1, 0, 1), (estFps - 60) / 60) or estFps < 240 and c3(1, 0, 1):Lerp(c3(0.33, 0, 1), (estFps - 120) / 120) or c3(0.6, 0.2, 1)):ToHex() .. "\">" .. ("%.1f"):format(estFps) .. "</font>")
		else
			for i, v in rows do
				rows[i](nil, false)
			end
		end
	end)
end

local function set3d(enabled)
	rs:Set3dRenderingEnabled(enabled)
end

local gui = Instance.new("ScreenGui", pgui)
gui.Name = "Black"
gui.DisplayOrder = -999
gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
gui.ResetOnSpawn = false

local black = Instance.new("Frame", gui)
black.BackgroundColor3 = c3()
black.Size = UDim2.fromScale(2, 2)
black.Position = UDim2.fromScale(0.5, 0.5)
black.AnchorPoint = Vector2.new(0.5, 0.5)
black.ZIndex = -999
black.BackgroundTransparency = 1

local pressChecked = false
local function pressCheck()
	if pressChecked then return end
	pressChecked = true

	local key = kk.RightAlt
	local pressed = false

	local con = uis.InputBegan:Connect(function(kk)
		if kk.KeyCode == key then
			pressed = true
		end
	end)

	local function pressCheck()
		if pressed then return true end

		pcall(press, key, true)
		pcall(press, key, false)
		r(2)

		if pressed then
			con:Disconnect()
		end

		return pressed
	end

	local function WARN()
		msg("Autoplayer might don't work, because executor for some reason doesn't support key pressing, or rejects them")
	end

	if not pressCheck() then
		if kp and kr then
			local oldPress = press
			press = function(key, isDown)
				(isDown and kp or kr)(key.Name)
			end

			if not pressCheck() then
				press = function(key, isDown)
					(isDown and kp or kr)(key)
				end

				if not pressCheck() then
					press = oldPress
					WARN()
				end
			end
		else
			WARN()
		end
	end

	con:Disconnect()
end

local function onWindow(window, dontStartAutoplay)
	if window.Name ~= "Window" then return end

	gameStarted:Fire()

	lanes = 0
	total = 0
	settings.TotalNotes = 0
	settings.MyTotalNotes = 0
	settings.EnemyTotalNotes = 0
	settings.SongDuration = 0
	settings.SongRealDuration = 0
	settings.Rate = 1
	settings.SongName = ""
	settings.SongDifficulty = ""
	settings.SongNameColor = c3()
	kbVals = { }
	badNotes = { }
	speedBuffer = { }
	speed = 0
	scrollSpeed = 0
	prevSpeed = false
	SVMode = false
	SVDetectAttempts = -1 -- for a reason, when a song starts it will make jump from 0 to actual scroll speed
	settings.ScrollSpeed = 0
	isModChart = false
	settings.IsModChartSong = false
	settings.IsSVSong = false
	settings.Lanes = 0
	settings.Playing = true

	local cons = { }
	local gameField = window:WaitForChild("Game", timeOut)
	local fields = gameField:WaitForChild("Fields", timeOut)
	fields = {
		Left = fields:WaitForChild("Left", timeOut),
		Right = fields:WaitForChild("Right", timeOut)
	}

	if isMobile then
		spawn(pressCheck)
	end

	spawn(mainLoop, fields, window, dontStartAutoplay)

	local hud = gameField:WaitForChild("HUD", timeOut)
	if hud:FindFirstChild("Stats") then
		spawn(statsAdded, hud.Stats, cons)
	end

	cons[#cons + 1] = hud.ChildAdded:Connect(function(child)
		if child.Name == "Stats" then
			statsAdded(child, cons)
		end
	end)

	local mySide = fields[side]
	local enemySide = fields[side == "Left" and "Right" or "Left"]
	local accuracy = hud:WaitForChild("AccuracyGauge", timeOut):WaitForChild("Ticks", timeOut)

	local function perfc(setting, value)
		if setting ~= "Performance" then return end

		mySide.Visible = value < 6
		enemySide.Visible = value < 5
		accuracy.Visible = value <= 1
		black.BackgroundTransparency = value >= 3 and 0 or 1

		pcall(set3d, value <= 3)
	end

	cons[#cons + 1] = settingChanged:Connect(perfc)
	perfc("Performance", perf)

	repeat wait() until not window.Parent

	gameEnded:Fire()

	settings.Playing = false
	black.BackgroundTransparency = 1
	pcall(set3d, true)

	for i, v in cons do
		v:Disconnect()
	end
end

setmetatable(settings, { __index = function(self, key) return key == "Events" and events or events[key] end })
global[key] = settings

local hasWindow = pgui:FindFirstChild("Window")
if hasWindow then
	spawn(onWindow, hasWindow, true)
end

spawn(function()
	while fps == 0 do r() end
	r()
	wait()

	if hasWindow then
		msg("Unable to start the autoplay:\nScript must be ran before the game starts")
	end

	local buzyWarned = false
	local buzyWaiting = false
	buzyWarn = function()
		if buzyWaiting then return false end
		buzyWaiting = true

		while not settings.Playing do wait() end
		buzyWaiting = false

		if buzyWarned or not buzy then return false end

		local before = buzyWarned
		buzyWarned = true

		msg("Auto Play cannot be working while you're focused on any kind of TextBox!")
		return not before
	end

	local escWarned = false
	local escWaiting = false
	escWarn = function()
		if escWaiting then return end
		escWaiting = true

		while not settings.Playing do wait() end
		escWaiting = false

		if escWarned or not escOpen then return end
		escWarned = true

		wait()
		if plr:GetNetworkPing() < 0 then return end
		msg("Auto Play cannot be working while ESC menu is open!")
	end

	if buzy then
		buzyWarn()
		uis:GetFocusedTextBox():ReleaseFocus()
	end

	local escWarned = false
	if escOpen then
		escWarn()
	end

	if not isMobile then
		pressCheck()
	end
end)

pgui.ChildAdded:Connect(onWindow)

for i, v in settings do
	settingChanged:Fire(i, v)
end

while fps == 0 do r() end
return settings
