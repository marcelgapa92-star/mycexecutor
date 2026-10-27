local TS = game:GetService("TweenService")
local UIS = game:GetService("UserInputService")
local RS = game:GetService("RunService")
local Players = game:GetService("Players")
local Lighting = game:GetService("Lighting")
local LogService = game:GetService("LogService")
local HttpService = game:GetService("HttpService")
local plr = Players.LocalPlayer
local rnd = Random.new()

local RED, GREEN = Color3.fromRGB(255,59,59), Color3.fromRGB(70,255,130)

-- ================= SAVE FILE (persistent) =================
local FILE = "ZenithData.json"
local canFS = typeof(writefile) == "function" and typeof(readfile) == "function" and typeof(isfile) == "function"
local Data = {theme = "Mono", scripts = {}, autoAttach = false, autoOpen = false}
local function loadData()
	if canFS and isfile(FILE) then
		local ok, d = pcall(function() return HttpService:JSONDecode(readfile(FILE)) end)
		if ok and type(d) == "table" then
			Data.theme = d.theme or "Mono"
			Data.scripts = type(d.scripts) == "table" and d.scripts or {}
			Data.autoAttach = d.autoAttach == true
			Data.autoOpen = d.autoOpen == true
		end
	end
end
local function saveData()
	if not canFS then return false end
	return (pcall(function() writefile(FILE, HttpService:JSONEncode(Data)) end))
end
loadData()

-- ================= THEMES =================
local THEMES, THEME_BY = {}, {}
local function addTheme(name, t) t.name = name THEMES[#THEMES+1] = t THEME_BY[name] = t end
local c3 = Color3.fromRGB
local function mkTheme(name, ac, f)
	f = f or 1
	local l = ac.R * .299 + ac.G * .587 + ac.B * .114
	addTheme(name, {
		accent = ac, onAccent = l > .6 and c3(10,10,12) or c3(255,255,255),
		bg = c3(9,9,11):Lerp(ac, .07 * f), bg2 = c3(19,19,24):Lerp(ac, .12 * f),
		panel = c3(20,20,24):Lerp(ac, .10 * f), panel2 = c3(34,34,40):Lerp(ac, .15 * f),
		line = c3(58,58,66):Lerp(ac, .30 * f), dim = c3(125,125,134):Lerp(ac, .30 * f),
		mid = c3(200,200,208):Lerp(ac, .35 * f), text = c3(255,255,255),
	})
end
mkTheme("Mono", c3(255,255,255), 0)
mkTheme("Crimson", c3(235,64,82))
mkTheme("Ocean", c3(56,152,255))
mkTheme("Emerald", c3(40,200,140))
mkTheme("Violet", c3(160,120,255))
mkTheme("Sunset", c3(255,140,50))
mkTheme("Rose", c3(255,105,180))
mkTheme("Gold", c3(250,200,30))
mkTheme("Cyan", c3(30,210,230))
mkTheme("Lime", c3(150,230,50))
mkTheme("Ice", c3(170,215,255), .6)
addTheme("Light", {
	accent = c3(24,24,28), onAccent = c3(255,255,255), bg = c3(238,238,243), bg2 = c3(250,250,253),
	panel = c3(255,255,255), panel2 = c3(226,226,233), line = c3(200,200,210),
	dim = c3(105,105,118), mid = c3(70,70,82), text = c3(18,18,22),
})

local T = {}
local function setT(def) for k, v in pairs(def) do if k ~= "name" then T[k] = v end end end
if not THEME_BY[Data.theme] then Data.theme = "Mono" end
setT(THEME_BY[Data.theme])

-- ===== SCRIPT HUB: all sources are searched at the same time =====
local SOURCES = {
	{
		name = "ScriptBlox", tag = "BLOX",
		search = "https://scriptblox.com/api/script/search?max=20&q=",
		trending = "https://scriptblox.com/api/script/fetch?page=1",
		raw = "https://scriptblox.com/raw/",
		list = function(d) return d.result and d.result.scripts end,
	},
	-- RBXScript: unverified endpoint, edit it if needed
	{
		name = "RBXScript", tag = "RBXS",
		search = "https://rbxscript.com/api/scripts?search=",
		trending = "https://rbxscript.com/api/scripts",
		list = function(d) return d.scripts or d.data or d.result or (d[1] and d) end,
	},
	-- rscripts.net: endpoint from memory, not verified
	{
		name = "Rscripts", tag = "RSCR",
		search = "https://rscripts.net/api/v2/scripts?page=1&orderBy=date&sort=desc&q=",
		trending = "https://rscripts.net/api/v2/scripts?page=1&orderBy=views&sort=desc",
		raw = "https://rscripts.net/raw/",
		list = function(d) return d.scripts or d.data or d.result end,
	},
	-- scriptrb: endpoint guessed, not verified
	{
		name = "ScriptRB", tag = "SCRB",
		search = "https://scriptrb.com/api/scripts?search=",
		trending = "https://scriptrb.com/api/scripts",
		list = function(d) return d.scripts or d.data or d.result or (d[1] and d) end,
	},
	-- scripthub.com: endpoint guessed, not verified
	{
		name = "ScriptHub", tag = "SHUB",
		search = "https://scripthub.com/api/scripts?q=",
		trending = "https://scripthub.com/api/scripts",
		list = function(d) return d.scripts or d.data or d.result or (d[1] and d) end,
	},
}

-- ================= HELPERS (theme-aware) =================
local function url(id) return "http://www.roblox.com/asset/?id=" .. id end
local function tw(o, t, p, s, d)
	local x = TS:Create(o, TweenInfo.new(t, s or Enum.EasingStyle.Quint, d or Enum.EasingDirection.Out), p)
	x:Play() return x
end

local reg, btns = {}, {}
local function r(role) return {_r = role} end
local function new(c, par, props)
	local o = Instance.new(c)
	for k, v in pairs(props or {}) do
		if type(v) == "table" and v._r then
			reg[#reg+1] = {o, k, v._r}
			o[k] = T[v._r]
		else
			o[k] = v
		end
	end
	o.Parent = par
	return o
end
local function round(o, rad) return new("UICorner", o, {CornerRadius = UDim.new(0, rad)}) end
local function stroke(o, t, c, tr)
	return new("UIStroke", o, {
		Thickness = t or 1,
		Color = (type(c) == "string" and r(c)) or c or r("accent"),
		Transparency = tr or 0,
		ApplyStrokeMode = Enum.ApplyStrokeMode.Border,
	})
end
local function styled(b, bg, tx, hbg, htx)
	btns[#btns+1] = {b = b, bg = bg, tx = tx}
	b.BackgroundColor3 = T[bg]
	b.TextColor3 = T[tx]
	b.MouseEnter:Connect(function() tw(b, .2, {BackgroundColor3 = T[hbg], TextColor3 = T[htx]}) end)
	b.MouseLeave:Connect(function() tw(b, .2, {BackgroundColor3 = T[bg], TextColor3 = T[tx]}) end)
end

local postApply
local function applyTheme(name, instant)
	local def = THEME_BY[name]
	if not def then return end
	setT(def)
	Data.theme = name
	local dur = instant and 0 or .35
	for i = #reg, 1, -1 do
		local e = reg[i]
		if not e[1].Parent then table.remove(reg, i)
		else tw(e[1], dur, {[e[2]] = T[e[3]]}) end
	end
	for i = #btns, 1, -1 do
		local e = btns[i]
		if not e.b.Parent then table.remove(btns, i)
		else tw(e.b, dur, {BackgroundColor3 = T[e.bg], TextColor3 = T[e.tx]}) end
	end
	if postApply then postApply(dur) end
end

-- ================= EFFECT ENGINE =================
local function fx(fn)
	return function()
		local objs, conns, cleanups = {}, {}, {}
		local env = {alive = true}
		function env.add(o) objs[#objs+1] = o return o end
		function env.conn(c) conns[#conns+1] = c end
		function env.onClean(f) cleanups[#cleanups+1] = f end
		local ok, err = pcall(fn, env)
		if not ok then warn("[Zenith] " .. tostring(err)) end
		return function()
			env.alive = false
			for _, c in ipairs(conns) do pcall(function() c:Disconnect() end) end
			for _, f in ipairs(cleanups) do pcall(f) end
			for _, o in ipairs(objs) do pcall(function() o:Destroy() end) end
		end
	end
end

local FACES = {"SkyboxBk","SkyboxDn","SkyboxFt","SkyboxLf","SkyboxRt","SkyboxUp"}
local function setSky(env, id)
	local s = env.add(Instance.new("Sky"))
	for _, f in ipairs(FACES) do s[f] = url(id) end
	s.Parent = Lighting
	return s
end
local function setSound(env, id, vol, pitch)
	local s = env.add(Instance.new("Sound"))
	s.SoundId = "rbxassetid://" .. id
	s.Volume, s.Pitch, s.Looped = vol or 5, pitch or 1, true
	s.Parent = workspace
	s:Play()
	return s
end
local function decalSpam(env, id, limit)
	local n = 0
	for _, p in ipairs(workspace:GetDescendants()) do
		if not env.alive then return end
		if p:IsA("BasePart") and not p:IsA("Terrain") then
			for _, face in ipairs(Enum.NormalId:GetEnumItems()) do
				local d = Instance.new("Decal")
				d.Name, d.Face, d.Texture = "ZDecal", face, url(id)
				d.Parent = p
				env.add(d)
			end
			n += 1
			if n >= (limit or 600) then return end
			if n % 40 == 0 then task.wait() end
		end
	end
end
local function eachPlayerPart(env, names, fn)
	local function apply(char)
		for _, nm in ipairs(names) do
			local part = char:FindFirstChild(nm)
			if part then fn(part) return end
		end
	end
	local function hook(p)
		if p.Character then apply(p.Character) end
		env.conn(p.CharacterAdded:Connect(function(c) task.wait(.7) if env.alive then apply(c) end end))
	end
	for _, p in ipairs(Players:GetPlayers()) do hook(p) end
	env.conn(Players.PlayerAdded:Connect(hook))
end
local function particles(env, textures)
	eachPlayerPart(env, {"UpperTorso", "Torso"}, function(torso)
		for _, t in ipairs(textures) do
			local pe = Instance.new("ParticleEmitter")
			pe.Texture, pe.VelocitySpread, pe.Rate = url(t), 50, 12
			pe.Parent = torso
			env.add(pe)
		end
	end)
end
local function grayscale(env, sat, con)
	local cc = Instance.new("ColorCorrectionEffect")
	cc.Saturation, cc.Contrast = sat, con
	cc.Parent = Lighting
	return env.add(cc)
end

-- ================= LOCAL LIBRARY =================
local LIBRARY = {
	{"Coolkid Skybox", "sky + music", fx(function(e) setSky(e, 358313209) setSound(e, 1839246711, 8) end)},
	{"MLG Particles", "all players", fx(function(e) particles(e, {8979672, 176067516, 179012130}) end)},
	{"Spooky Skeleton", "animated sky", fx(function(e)
		local s = setSound(e, 104835515231080, 3)
		s.TimePosition = 8.6
		local ds = Instance.new("DistortionSoundEffect") ds.Level = 2.51 ds.Parent = s
		local sky = setSky(e, 169585459)
		local frames = {169585459,169585475,169585485,169585502,169585515,169585502,169585485,169585475}
		task.spawn(function()
			while e.alive do
				for _, id in ipairs(frames) do
					if not e.alive then return end
					for _, f in ipairs(FACES) do sky[f] = url(id) end
					task.wait(.16)
				end
			end
		end)
	end)},
	{"Anonymous Particles", "all players", fx(function(e) particles(e, {127476787}) end)},
	{"Decal Spam 1", "whole map", fx(function(e) decalSpam(e, 110291868) end)},
	{"Coolkid Decal Spam", "decals + sky", fx(function(e)
		setSky(e, 8408806737) setSound(e, 72089843969979, 8, .2) decalSpam(e, 8408806737)
	end)},
	{"1x1x1x1", "music + grayscale", fx(function(e)
		local h = Instance.new("Hint") h.Text = "1x1x1x1 has returned..." h.Parent = workspace e.add(h)
		setSound(e, 27697743, 6, 3)
		setSound(e, 35935204, 6, .9)
		grayscale(e, -1, .4)
	end)},
	{"Troll Face", "decals + sky", fx(function(e)
		setSky(e, 85004246493058) setSound(e, 7705506391, 8) decalSpam(e, 85004246493058)
	end)},
	{"Snoop Dogg Face All", "all players", fx(function(e)
		eachPlayerPart(e, {"Head"}, function(head)
			local old = head.Transparency
			head.Transparency = 1
			e.onClean(function() if head then head.Transparency = old end end)
			local bb = Instance.new("BillboardGui")
			bb.Adornee, bb.Size, bb.StudsOffset, bb.AlwaysOnTop = head, UDim2.new(2.5,0,2.5,0), Vector3.new(0,.2,0), true
			bb.Parent = head
			e.add(bb)
			local il = Instance.new("ImageLabel")
			il.Image, il.Size, il.BackgroundTransparency = url(125148536957515), UDim2.fromScale(1,1), 1
			il.Parent = bb
		end)
	end)},
	{"Mario.exe", "paste your IDs", fx(function(e)
		local SKY_ID, SOUND_ID, DECAL_ID = nil, nil, nil
		if SKY_ID then setSky(e, SKY_ID) end
		if SOUND_ID then setSound(e, SOUND_ID, 8) end
		if DECAL_ID then decalSpam(e, DECAL_ID) end
		grayscale(e, -1, .5)
	end)},
	{"iPhone Morph", "you become a phone", fx(function(e)
		local char = plr.Character
		if not char then return end
		local torso = char:FindFirstChild("Torso") or char:FindFirstChild("UpperTorso")
		if not torso then return end
		for _, v in ipairs(char:GetDescendants()) do
			if (v:IsA("BasePart") and v ~= torso and v.Name ~= "HumanoidRootPart") or v:IsA("Decal") then
				local old = v.Transparency
				v.Transparency = 1
				e.onClean(function() if v then v.Transparency = old end end)
			end
		end
		local m = Instance.new("SpecialMesh")
		m.MeshType, m.MeshId, m.TextureId, m.Scale = Enum.MeshType.FileMesh, url(430345282), url(430345284), Vector3.new(1,1,1)
		m.Parent = torso
		e.add(m)
	end)},
}
local FX = {}
for _, e in ipairs(LIBRARY) do FX[e[1]] = e[3] end

-- ================= STATE =================
local injected, token, logConn = false, 0, nil
local active, stoppers, paints = 0, {}, {}
local open = false
local Mono = {}
local function lineOf(name) return '_G.Mono.run("' .. name .. '")' end

-- ================= GUI ROOT =================
local gui = new("ScreenGui", plr:WaitForChild("PlayerGui"), {Name = "ZenithV5", ResetOnSpawn = false, IgnoreGuiInset = true})

-- ---------- ICON ----------
local ICON = 48
local icon = new("TextButton", gui, {Size = UDim2.fromOffset(ICON,ICON), Position = UDim2.new(0,16,0.5,-24), BackgroundColor3 = r("bg"), Text = "", AutoButtonColor = false})
round(icon, 13)
local iconStroke = stroke(icon, 1.8)
local ring = new("Frame", icon, {AnchorPoint = Vector2.new(.5,.5), Position = UDim2.fromScale(.5,.5), Size = UDim2.fromOffset(31,31), BackgroundTransparency = 1, Rotation = 45})
round(ring, 7) stroke(ring, 1.5, "accent", .5)
new("TextLabel", icon, {Size = UDim2.fromScale(1,1), BackgroundTransparency = 1, Text = "Z", TextColor3 = r("accent"), Font = Enum.Font.GothamBlack, TextSize = 20})
RS.RenderStepped:Connect(function(dt) ring.Rotation = (ring.Rotation + dt * 40) % 360 end)
icon.MouseEnter:Connect(function() tw(icon, .25, {Size = UDim2.fromOffset(ICON+6,ICON+6)}, Enum.EasingStyle.Back) end)
icon.MouseLeave:Connect(function() tw(icon, .25, {Size = UDim2.fromOffset(ICON,ICON)}) end)

-- ---------- WINDOW ----------
local WW, WH = 480, 280
local main = new("CanvasGroup", gui, {AnchorPoint = Vector2.new(.5,.5), Position = UDim2.fromScale(.5,.5), Size = UDim2.fromOffset(WW,WH), BackgroundColor3 = r("bg"), GroupTransparency = 1, Visible = false})
round(main, 14) stroke(main, 1.4, "accent", .35)
local mainGrad = new("UIGradient", main, {Rotation = 90, Color = ColorSequence.new(T.bg2, T.bg)})
local scale = new("UIScale", main, {Scale = .3})

local FIT = 1
local function computeFit()
	local cam = workspace.CurrentCamera
	local v = cam and cam.ViewportSize or Vector2.new(800,400)
	FIT = math.clamp(math.min((v.Y - 24) / WH, (v.X - 24) / WW), .55, 1.2)
	if open then scale.Scale = FIT end
end
computeFit()
if workspace.CurrentCamera then workspace.CurrentCamera:GetPropertyChangedSignal("ViewportSize"):Connect(computeFit) end

local dustLayer = new("Frame", main, {Size = UDim2.fromScale(1,1), BackgroundTransparency = 1})

-- top bar
local BAR = 36
local bar = new("Frame", main, {Size = UDim2.new(1,0,0,BAR), BackgroundTransparency = 1})
local logoMini = new("Frame", bar, {Position = UDim2.fromOffset(14,13), Size = UDim2.fromOffset(10,10), BackgroundColor3 = r("accent"), Rotation = 45}) round(logoMini, 2)
local titleLbl = new("TextLabel", bar, {Size = UDim2.fromOffset(110,BAR), Position = UDim2.fromOffset(32,0), BackgroundTransparency = 1, Text = "ZENITH", TextColor3 = r("text"), Font = Enum.Font.GothamBlack, TextSize = 13, TextXAlignment = Enum.TextXAlignment.Left})
local titleGrad = new("UIGradient", titleLbl, {Color = ColorSequence.new(T.text, T.dim)})

-- status pill (red = off, green = injected)
local pill = new("Frame", bar, {Size = UDim2.fromOffset(110,22), Position = UDim2.new(1,-148,0,7), BackgroundColor3 = r("panel2")}) round(pill, 11)
local pillStroke = stroke(pill, 1, T.line)
local glow = new("Frame", pill, {AnchorPoint = Vector2.new(.5,.5), Position = UDim2.fromOffset(15,11), Size = UDim2.fromOffset(10,10), BackgroundColor3 = RED, BackgroundTransparency = .55}) round(glow, 10)
local dot = new("Frame", pill, {AnchorPoint = Vector2.new(.5,.5), Position = UDim2.fromOffset(15,11), Size = UDim2.fromOffset(8,8), BackgroundColor3 = RED}) round(dot, 4)
local status = new("TextLabel", pill, {Size = UDim2.new(1,-30,1,0), Position = UDim2.fromOffset(28,0), BackgroundTransparency = 1, Text = "Not injected", TextColor3 = RED, Font = Enum.Font.GothamMedium, TextSize = 9, TextXAlignment = Enum.TextXAlignment.Left, TextTruncate = Enum.TextTruncate.AtEnd})

local close = new("TextButton", bar, {Size = UDim2.fromOffset(26,26), Position = UDim2.new(1,-34,0,5), Text = "✕", Font = Enum.Font.GothamBold, TextSize = 11, AutoButtonColor = false})
round(close, 13)
styled(close, "panel2", "dim", "accent", "onAccent")
close.MouseEnter:Connect(function() tw(close, .15, {Rotation = 90}) end)
close.MouseLeave:Connect(function() tw(close, .15, {Rotation = 0}) end)
local barLine = new("Frame", main, {Size = UDim2.new(0,0,0,1), Position = UDim2.fromOffset(0,BAR), BackgroundColor3 = r("accent"), BorderSizePixel = 0, BackgroundTransparency = .5})

-- toast
local toastLbl = new("TextLabel", main, {AnchorPoint = Vector2.new(.5,0), Position = UDim2.new(.5,0,0,BAR-6), Size = UDim2.fromOffset(0,22), AutomaticSize = Enum.AutomaticSize.X, BackgroundColor3 = r("panel2"), TextColor3 = r("text"), Font = Enum.Font.GothamMedium, TextSize = 10, Text = "", BackgroundTransparency = 1, TextTransparency = 1, ZIndex = 40})
round(toastLbl, 11)
new("UIPadding", toastLbl, {PaddingLeft = UDim.new(0,12), PaddingRight = UDim.new(0,12)})
local toastStroke = stroke(toastLbl, 1, "accent", 1)
local toastTok = 0
local function toast(text)
	toastTok += 1
	local my = toastTok
	toastLbl.Text = text
	toastLbl.Position = UDim2.new(.5,0,0,BAR-6)
	tw(toastLbl, .35, {Position = UDim2.new(.5,0,0,BAR+6), BackgroundTransparency = 0, TextTransparency = 0}, Enum.EasingStyle.Back)
	tw(toastStroke, .35, {Transparency = .4})
	task.delay(2.2, function()
		if my ~= toastTok then return end
		tw(toastLbl, .3, {Position = UDim2.new(.5,0,0,BAR-6), BackgroundTransparency = 1, TextTransparency = 1})
		tw(toastStroke, .3, {Transparency = 1})
	end)
end

-- layout numbers
local SIDE = 96
local BTN_H = 34
local BOT_Y = WH - BTN_H - 8
local PAGE_Y = BAR + 8
local PAGE_H = BOT_Y - PAGE_Y - 8
local CX = SIDE + 14
local PAGE_W = WW - CX - 8

-- ---------- SIDEBAR (7 tabs) ----------
local sidebar = new("Frame", main, {Size = UDim2.fromOffset(SIDE, BOT_Y - PAGE_Y - 8), Position = UDim2.fromOffset(8, PAGE_Y), BackgroundColor3 = r("panel"), BackgroundTransparency = .2})
round(sidebar, 11) stroke(sidebar, 1, "line")

local TAB_H, TAB_STEP = 24, 26
local indicator = new("Frame", sidebar, {Size = UDim2.new(1,-8,0,TAB_H), Position = UDim2.fromOffset(4,4), BackgroundColor3 = r("accent"), BackgroundTransparency = .88})
round(indicator, 8)
stroke(indicator, 1, "accent", .7)
new("Frame", indicator, {Size = UDim2.fromOffset(3,12), Position = UDim2.new(0,0,.5,-6), BackgroundColor3 = r("accent"), BorderSizePixel = 0}).Name = "Accent"

local TABS = {{"EDITOR","</>"},{"HUB","◎"},{"LIBRARY","▦"},{"SAVED","▤"},{"THEMES","◐"},{"SETTINGS","⚙"},{"CONSOLE",">_"}}
local tabBtns, tabGlyph, tabY = {}, {}, {}
for i, t in ipairs(TABS) do
	local y = 4 + (i-1) * TAB_STEP
	tabY[t[1]] = y
	local b = new("TextButton", sidebar, {Size = UDim2.new(1,-8,0,TAB_H), Position = UDim2.fromOffset(4,y), BackgroundTransparency = 1, Text = t[1], TextColor3 = T.dim, Font = Enum.Font.GothamBold, TextSize = 10, TextXAlignment = Enum.TextXAlignment.Left, AutoButtonColor = false})
	new("UIPadding", b, {PaddingLeft = UDim.new(0,32)})
	local g = new("TextLabel", b, {Size = UDim2.fromOffset(26,TAB_H), Position = UDim2.fromOffset(-28,0), BackgroundTransparency = 1, Text = t[2], TextColor3 = T.dim, Font = Enum.Font.GothamBold, TextSize = 10})
	tabBtns[t[1]], tabGlyph[t[1]] = b, g
end

-- ---------- PAGES ----------
local pages = {}
local function newPage(name)
	local p = new("CanvasGroup", main, {Size = UDim2.fromOffset(PAGE_W,PAGE_H), Position = UDim2.fromOffset(CX,PAGE_Y), BackgroundTransparency = 1, GroupTransparency = 1, Visible = false})
	pages[name] = p
	return p
end
local function panel(par, sz, pos)
	local f = new("Frame", par, {Size = sz, Position = pos or UDim2.new(), BackgroundColor3 = r("panel"), BackgroundTransparency = .2})
	round(f, 10) stroke(f, 1, "line")
	return f
end

-- ---- EDITOR ----
local pgEditor = newPage("EDITOR")
local edPanel = panel(pgEditor, UDim2.fromScale(1,1))
local editorScroll = new("ScrollingFrame", edPanel, {Size = UDim2.new(1,-8,1,-26), Position = UDim2.fromOffset(4,4), BackgroundTransparency = 1, BorderSizePixel = 0, ScrollBarThickness = 3, ScrollBarImageColor3 = r("accent"), CanvasSize = UDim2.new(), AutomaticCanvasSize = Enum.AutomaticSize.XY, ScrollingDirection = Enum.ScrollingDirection.XY})
local nums = new("TextLabel", editorScroll, {Size = UDim2.fromOffset(22,0), AutomaticSize = Enum.AutomaticSize.Y, BackgroundTransparency = 1, Text = "1", TextColor3 = r("dim"), Font = Enum.Font.Code, TextSize = 12, TextXAlignment = Enum.TextXAlignment.Right, TextYAlignment = Enum.TextYAlignment.Top})
local box = new("TextBox", editorScroll, {Position = UDim2.fromOffset(30,0), Size = UDim2.fromOffset(330,0), AutomaticSize = Enum.AutomaticSize.XY, BackgroundTransparency = 1, MultiLine = true, ClearTextOnFocus = false, TextWrapped = false, TextXAlignment = Enum.TextXAlignment.Left, TextYAlignment = Enum.TextYAlignment.Top, Font = Enum.Font.Code, TextSize = 12, TextColor3 = r("text"), PlaceholderText = "-- type a script or pick one from HUB / LIBRARY / SAVED", PlaceholderColor3 = r("dim"), Text = ""})
new("UISizeConstraint", box, {MinSize = Vector2.new(330,120)})
local lineInfo = new("TextLabel", edPanel, {Size = UDim2.fromOffset(200,18), Position = UDim2.new(1,-208,1,-20), BackgroundTransparency = 1, Text = "1 line · 0 chars", TextColor3 = r("dim"), Font = Enum.Font.Gotham, TextSize = 9, TextXAlignment = Enum.TextXAlignment.Right})

-- ---- HUB (all sources at once) ----
local pgHub = newPage("HUB")
local hubTop = panel(pgHub, UDim2.new(1,0,0,32))
local sf = new("Frame", hubTop, {Size = UDim2.new(1,-88,0,22), Position = UDim2.fromOffset(5,5), BackgroundColor3 = r("panel2")}) round(sf, 7)
local sfStroke = stroke(sf, 1, "line")
new("TextLabel", sf, {Size = UDim2.fromOffset(20,22), BackgroundTransparency = 1, Text = "🔍", TextSize = 10, TextColor3 = r("dim")})
local search = new("TextBox", sf, {Size = UDim2.new(1,-26,1,0), Position = UDim2.fromOffset(22,0), BackgroundTransparency = 1, Text = "", PlaceholderText = "Search all 5 sources...", PlaceholderColor3 = r("dim"), TextColor3 = r("text"), Font = Enum.Font.Gotham, TextSize = 10, TextXAlignment = Enum.TextXAlignment.Left, ClearTextOnFocus = false})
search.Focused:Connect(function() tw(sfStroke, .2, {Color = T.accent}) end)
search.FocusLost:Connect(function() tw(sfStroke, .2, {Color = T.line}) end)
local goBtn = new("TextButton", hubTop, {Size = UDim2.fromOffset(74,22), Position = UDim2.new(1,-79,0,5), Text = "SEARCH", Font = Enum.Font.GothamBold, TextSize = 9, AutoButtonColor = false})
round(goBtn, 7)
styled(goBtn, "accent", "onAccent", "mid", "onAccent")

local hubInfo = new("TextLabel", pgHub, {Size = UDim2.new(1,0,0,14), Position = UDim2.fromOffset(2,35), BackgroundTransparency = 1, Text = "Open HUB to load scripts", TextColor3 = r("dim"), Font = Enum.Font.Gotham, TextSize = 9, TextXAlignment = Enum.TextXAlignment.Left, TextTruncate = Enum.TextTruncate.AtEnd})
local hubList = new("ScrollingFrame", pgHub, {Size = UDim2.new(1,0,1,-52), Position = UDim2.fromOffset(0,52), BackgroundTransparency = 1, BorderSizePixel = 0, ScrollBarThickness = 3, ScrollBarImageColor3 = r("accent"), CanvasSize = UDim2.new(), AutomaticCanvasSize = Enum.AutomaticSize.Y})
new("UIListLayout", hubList, {Padding = UDim.new(0,4), SortOrder = Enum.SortOrder.LayoutOrder})
new("UIPadding", hubList, {PaddingRight = UDim.new(0,6), PaddingTop = UDim.new(0,1)})

-- ---- LIBRARY ----
local pgLib = newPage("LIBRARY")
local libTop = panel(pgLib, UDim2.new(1,0,0,32))
new("TextLabel", libTop, {Size = UDim2.fromOffset(150,32), Position = UDim2.fromOffset(12,0), BackgroundTransparency = 1, Text = "TAP TO QUEUE / STOP", TextColor3 = r("dim"), Font = Enum.Font.GothamBold, TextSize = 9, TextXAlignment = Enum.TextXAlignment.Left})
local stopAll = new("TextButton", libTop, {Size = UDim2.fromOffset(112,22), Position = UDim2.new(1,-117,0,5), Text = "STOP ALL · 0 live", Font = Enum.Font.GothamBold, TextSize = 9, AutoButtonColor = false})
round(stopAll, 7)
styled(stopAll, "panel2", "text", "accent", "onAccent")
local libList = new("ScrollingFrame", pgLib, {Size = UDim2.new(1,0,1,-38), Position = UDim2.fromOffset(0,38), BackgroundTransparency = 1, BorderSizePixel = 0, ScrollBarThickness = 3, ScrollBarImageColor3 = r("accent"), CanvasSize = UDim2.new(), AutomaticCanvasSize = Enum.AutomaticSize.Y})
new("UIGridLayout", libList, {CellSize = UDim2.new(.5,-5,0,40), CellPadding = UDim2.fromOffset(5,5), SortOrder = Enum.SortOrder.LayoutOrder})
new("UIPadding", libList, {PaddingRight = UDim.new(0,6), PaddingTop = UDim.new(0,1)})

-- ---- SAVED ----
local pgSaved = newPage("SAVED")
local savedTop = panel(pgSaved, UDim2.new(1,0,0,26))
new("TextLabel", savedTop, {Size = UDim2.fromOffset(110,26), Position = UDim2.fromOffset(12,0), BackgroundTransparency = 1, Text = "SAVED SCRIPTS", TextColor3 = r("dim"), Font = Enum.Font.GothamBold, TextSize = 9, TextXAlignment = Enum.TextXAlignment.Left})
new("TextLabel", savedTop, {Size = UDim2.fromOffset(150,26), Position = UDim2.fromOffset(120,0), BackgroundTransparency = 1,
	Text = canFS and "● auto-saved to file" or "⚠ memory only (no writefile)",
	TextColor3 = canFS and GREEN or RED, Font = Enum.Font.Gotham, TextSize = 9, TextXAlignment = Enum.TextXAlignment.Left})
local savedCount = new("TextLabel", savedTop, {Size = UDim2.fromOffset(80,26), Position = UDim2.new(1,-90,0,0), BackgroundTransparency = 1, Text = "0 saved", TextColor3 = r("text"), Font = Enum.Font.GothamBold, TextSize = 9, TextXAlignment = Enum.TextXAlignment.Right})

local nameFrame = new("Frame", pgSaved, {Size = UDim2.new(1,-66,0,24), Position = UDim2.fromOffset(0,30), BackgroundColor3 = r("panel2")}) round(nameFrame, 7)
local nameStroke = stroke(nameFrame, 1, "line")
local nameBox = new("TextBox", nameFrame, {Size = UDim2.new(1,-16,1,0), Position = UDim2.fromOffset(8,0), BackgroundTransparency = 1, Text = "", PlaceholderText = "Script name...", PlaceholderColor3 = r("dim"), TextColor3 = r("text"), Font = Enum.Font.Gotham, TextSize = 10, TextXAlignment = Enum.TextXAlignment.Left, ClearTextOnFocus = false})
nameBox.Focused:Connect(function() tw(nameStroke, .2, {Color = T.accent}) end)
nameBox.FocusLost:Connect(function() tw(nameStroke, .2, {Color = T.line}) end)
local saveBtn = new("TextButton", pgSaved, {Size = UDim2.fromOffset(60,24), Position = UDim2.new(1,-60,0,30), Text = "SAVE", Font = Enum.Font.GothamBold, TextSize = 10, AutoButtonColor = false})
round(saveBtn, 7)
styled(saveBtn, "accent", "onAccent", "mid", "onAccent")

local codeFrame = new("Frame", pgSaved, {Size = UDim2.new(1,0,0,48), Position = UDim2.fromOffset(0,58), BackgroundColor3 = r("panel2")}) round(codeFrame, 7)
local codeStroke = stroke(codeFrame, 1, "line")
local codeBox = new("TextBox", codeFrame, {Size = UDim2.new(1,-14,1,-8), Position = UDim2.fromOffset(7,4), BackgroundTransparency = 1, MultiLine = true, TextWrapped = true, ClearTextOnFocus = false, Text = "", PlaceholderText = "Paste your script here...", PlaceholderColor3 = r("dim"), TextColor3 = r("text"), Font = Enum.Font.Code, TextSize = 10, TextXAlignment = Enum.TextXAlignment.Left, TextYAlignment = Enum.TextYAlignment.Top})
codeBox.Focused:Connect(function() tw(codeStroke, .2, {Color = T.accent}) end)
codeBox.FocusLost:Connect(function() tw(codeStroke, .2, {Color = T.line}) end)

local savedList = new("ScrollingFrame", pgSaved, {Size = UDim2.new(1,0,1,-112), Position = UDim2.fromOffset(0,112), BackgroundTransparency = 1, BorderSizePixel = 0, ScrollBarThickness = 3, ScrollBarImageColor3 = r("accent"), CanvasSize = UDim2.new(), AutomaticCanvasSize = Enum.AutomaticSize.Y})
new("UIListLayout", savedList, {Padding = UDim.new(0,4), SortOrder = Enum.SortOrder.LayoutOrder})
new("UIPadding", savedList, {PaddingRight = UDim.new(0,6), PaddingTop = UDim.new(0,1)})

-- ---- THEMES ----
local pgThemes = newPage("THEMES")
local themesTop = panel(pgThemes, UDim2.new(1,0,0,32))
new("TextLabel", themesTop, {Size = UDim2.fromOffset(200,32), Position = UDim2.fromOffset(12,0), BackgroundTransparency = 1, Text = "TAP A COLOR TO APPLY", TextColor3 = r("dim"), Font = Enum.Font.GothamBold, TextSize = 9, TextXAlignment = Enum.TextXAlignment.Left})
local themeList = new("ScrollingFrame", pgThemes, {Size = UDim2.new(1,0,1,-38), Position = UDim2.fromOffset(0,38), BackgroundTransparency = 1, BorderSizePixel = 0, ScrollBarThickness = 3, ScrollBarImageColor3 = r("accent"), CanvasSize = UDim2.new(), AutomaticCanvasSize = Enum.AutomaticSize.Y})
new("UIGridLayout", themeList, {CellSize = UDim2.new(.5,-5,0,40), CellPadding = UDim2.fromOffset(5,5), SortOrder = Enum.SortOrder.LayoutOrder})
new("UIPadding", themeList, {PaddingRight = UDim.new(0,6), PaddingTop = UDim.new(0,1)})

-- ---- SETTINGS ----
local pgSet = newPage("SETTINGS")
local setTop = panel(pgSet, UDim2.new(1,0,0,32))
new("TextLabel", setTop, {Size = UDim2.fromOffset(200,32), Position = UDim2.fromOffset(12,0), BackgroundTransparency = 1, Text = "SETTINGS", TextColor3 = r("dim"), Font = Enum.Font.GothamBold, TextSize = 9, TextXAlignment = Enum.TextXAlignment.Left})
local setList = new("Frame", pgSet, {Size = UDim2.new(1,0,1,-38), Position = UDim2.fromOffset(0,38), BackgroundTransparency = 1})
new("UIListLayout", setList, {Padding = UDim.new(0,5), SortOrder = Enum.SortOrder.LayoutOrder})

local togglePaints = {}
local function addToggle(order, title, sub, key)
	local row = new("TextButton", setList, {Size = UDim2.new(1,-6,0,44), BackgroundColor3 = r("panel"), BackgroundTransparency = .2, Text = "", AutoButtonColor = false, LayoutOrder = order})
	round(row, 9)
	local st = stroke(row, 1, "line")
	new("TextLabel", row, {Size = UDim2.new(1,-64,0,18), Position = UDim2.fromOffset(10,6), BackgroundTransparency = 1, Text = title, TextColor3 = r("text"), Font = Enum.Font.GothamBold, TextSize = 11, TextXAlignment = Enum.TextXAlignment.Left})
	new("TextLabel", row, {Size = UDim2.new(1,-64,0,12), Position = UDim2.fromOffset(10,25), BackgroundTransparency = 1, Text = sub, TextColor3 = r("dim"), Font = Enum.Font.Gotham, TextSize = 8, TextXAlignment = Enum.TextXAlignment.Left, TextTruncate = Enum.TextTruncate.AtEnd})
	local pillF = Instance.new("Frame")
	pillF.Size, pillF.Position, pillF.BackgroundColor3 = UDim2.fromOffset(36,18), UDim2.new(1,-46,.5,-9), T.panel2
	pillF.Parent = row
	round(pillF, 9)
	local knob = Instance.new("Frame")
	knob.Size, knob.Position, knob.BackgroundColor3 = UDim2.fromOffset(12,12), UDim2.fromOffset(3,3), T.dim
	knob.Parent = pillF
	round(knob, 6)
	local function paint(dur)
		local on = Data[key]
		tw(pillF, dur or .25, {BackgroundColor3 = on and T.accent or T.panel2})
		tw(knob, dur or .25, {Position = on and UDim2.fromOffset(21,3) or UDim2.fromOffset(3,3), BackgroundColor3 = on and T.onAccent or T.dim}, Enum.EasingStyle.Back)
	end
	togglePaints[#togglePaints+1] = paint
	paint(0)
	row.MouseEnter:Connect(function() tw(st, .2, {Color = T.accent}) end)
	row.MouseLeave:Connect(function() tw(st, .2, {Color = T.line}) end)
	row.MouseButton1Click:Connect(function()
		Data[key] = not Data[key]
		paint(.25)
		saveData()
		toast(title .. (Data[key] and ": ON" or ": OFF"))
	end)
end
addToggle(1, "Auto Attach", "Injects automatically when the script starts", "autoAttach")
addToggle(2, "Auto Open", "Opens the window together with Auto Attach", "autoOpen")

-- ---- CONSOLE ----
local pgCon = newPage("CONSOLE")
local conPanel = panel(pgCon, UDim2.fromScale(1,1))
local conScroll = new("ScrollingFrame", conPanel, {Size = UDim2.new(1,-8,1,-8), Position = UDim2.fromOffset(4,4), BackgroundTransparency = 1, BorderSizePixel = 0, ScrollBarThickness = 3, ScrollBarImageColor3 = r("accent"), CanvasSize = UDim2.new(), AutomaticCanvasSize = Enum.AutomaticSize.Y})
new("UIListLayout", conScroll, {Padding = UDim.new(0,1), SortOrder = Enum.SortOrder.LayoutOrder})
new("UIPadding", conScroll, {PaddingLeft = UDim.new(0,5), PaddingTop = UDim.new(0,2), PaddingRight = UDim.new(0,5)})

local logCount = 0
local function log(msg, col)
	logCount += 1
	new("TextLabel", conScroll, {Size = UDim2.new(1,0,0,0), AutomaticSize = Enum.AutomaticSize.Y, BackgroundTransparency = 1, Text = msg, TextWrapped = true, TextColor3 = (type(col) == "string" and r(col)) or col or r("text"), Font = Enum.Font.Code, TextSize = 11, TextXAlignment = Enum.TextXAlignment.Left, LayoutOrder = logCount})
	local labels = {}
	for _, k in ipairs(conScroll:GetChildren()) do if k:IsA("TextLabel") then labels[#labels+1] = k end end
	if #labels > 120 then labels[1]:Destroy() end
	task.defer(function() conScroll.CanvasPosition = Vector2.new(0, 1e6) end)
end

-- ---------- BOTTOM BUTTONS: left / center / right ----------
local btnList, btnBase = {}, {}
local function makeBtn(text, xScale, anchor, filled)
	local pos = UDim2.new(xScale, anchor == 0 and 8 or (anchor == 1 and -8 or 0), 0, BOT_Y)
	local b = new("TextButton", main, {Size = UDim2.new(.31,0,0,BTN_H), AnchorPoint = Vector2.new(anchor,0), Position = pos, Text = text, Font = Enum.Font.GothamBold, TextSize = 11, AutoButtonColor = false})
	round(b, 10) stroke(b, 1, "line")
	if filled then styled(b, "accent", "onAccent", "mid", "onAccent")
	else styled(b, "panel2", "text", "accent", "onAccent") end
	local s = new("UIScale", b, {Scale = 1})
	b.MouseButton1Down:Connect(function() tw(s, .08, {Scale = .94}) end)
	b.MouseButton1Up:Connect(function() tw(s, .3, {Scale = 1}, Enum.EasingStyle.Back) end)
	btnList[#btnList+1] = b
	btnBase[b] = pos
	return b
end
local executeBtn = makeBtn("▶  EXECUTE", 0, 0, true)
local clearBtn   = makeBtn("CLEAR", .5, .5, false)
local injectBtn  = makeBtn("INJECT", 1, 1, false)

-- ---------- INTRO CURTAIN ----------
local curtain = new("Frame", main, {AnchorPoint = Vector2.new(1,0), Position = UDim2.fromScale(1,0), Size = UDim2.fromScale(1,1), BackgroundColor3 = r("bg"), BorderSizePixel = 0, ZIndex = 50, Visible = false})
new("Frame", curtain, {Size = UDim2.new(0,2,1,0), BackgroundColor3 = r("accent"), BorderSizePixel = 0, ZIndex = 51})

-- ================= HELPERS =================
local libScales = {}
local function popRows(list, from)
	for i, s in ipairs(list) do
		s.Scale = .82
		task.delay((from or 0) + i * .04, function() tw(s, .45, {Scale = 1}, Enum.EasingStyle.Back) end)
	end
end

local currentTab, tabToken = nil, 0
local function styleTabs(dur)
	for n, b in pairs(tabBtns) do
		local on = n == currentTab
		tw(b, dur or .25, {TextColor3 = on and T.text or T.dim})
		tw(tabGlyph[n], dur or .25, {TextColor3 = on and T.text or T.dim})
	end
end
local function setTab(name)
	if currentTab == name then return end
	tabToken += 1
	local my = tabToken
	local prev = currentTab
	currentTab = name
	styleTabs(.25)
	tw(indicator, .5, {Position = UDim2.fromOffset(4, tabY[name])}, Enum.EasingStyle.Back)
	if prev and pages[prev] then
		local p = pages[prev]
		tw(p, .2, {GroupTransparency = 1, Position = UDim2.fromOffset(CX - 10, PAGE_Y - 6)})
		task.delay(.22, function() if tabToken == my then p.Visible = false end end)
	end
	local np = pages[name]
	np.Position = UDim2.fromOffset(CX + 12, PAGE_Y + 14)
	np.GroupTransparency = 1
	np.Visible = true
	tw(np, .55, {GroupTransparency = 0, Position = UDim2.fromOffset(CX, PAGE_Y)}, Enum.EasingStyle.Quint)
	if name == "LIBRARY" then popRows(libScales, .1) end
end
for n, b in pairs(tabBtns) do b.MouseButton1Click:Connect(function() setTab(n) end) end

local function setStatus(t, col)
	status.Text = t
	status.TextColor3 = (type(col) == "string" and T[col]) or col or (injected and GREEN or RED)
end
local function refreshCount() stopAll.Text = "STOP ALL · " .. active .. " live" end
local function shake(o)
	local p = o.Position
	for _, dx in ipairs({-5, 5, -3, 3, 0}) do
		tw(o, .05, {Position = UDim2.new(p.X.Scale, p.X.Offset + dx, p.Y.Scale, p.Y.Offset)}, Enum.EasingStyle.Linear)
		task.wait(.05)
	end
end
local function needInject() setStatus("Inject first", RED) toast("Press INJECT first") task.spawn(shake, injectBtn) end

local function repaintAll()
	local text = box.Text
	for name, paint in pairs(paints) do
		if stoppers[name] then paint("live")
		elseif text:find(lineOf(name), 1, true) then paint("queued")
		else paint("idle") end
	end
end

function Mono.run(name)
	if not FX[name] then log("✖ unknown script: " .. tostring(name), RED) return end
	if stoppers[name] then return end
	stoppers[name] = FX[name]()
	active += 1
	refreshCount() repaintAll()
	log("▶ running: " .. name, GREEN)
end
function Mono.stop(name)
	local s = stoppers[name]
	if not s then return end
	stoppers[name] = nil
	active -= 1
	task.spawn(s)
	refreshCount() repaintAll()
	log("■ stopped: " .. name, "dim")
end
function Mono.stopAll() for n in pairs(stoppers) do Mono.stop(n) end end
_G.Mono = Mono

local function updateLines()
	local t = box.Text
	local n = 1
	for _ in t:gmatch("\n") do n += 1 end
	local parts = table.create(n)
	for i = 1, n do parts[i] = tostring(i) end
	nums.Text = table.concat(parts, "\n")
	lineInfo.Text = n .. (n == 1 and " line" or " lines") .. " · " .. #t .. " chars"
	repaintAll()
end
box:GetPropertyChangedSignal("Text"):Connect(updateLines)

local function addLine(name)
	if box.Text:find(lineOf(name), 1, true) then return end
	box.Text = (box.Text ~= "" and (box.Text .. "\n") or "") .. lineOf(name)
end
local function removeLine(name)
	local out = {}
	for l in (box.Text .. "\n"):gmatch("(.-)\n") do
		if l ~= lineOf(name) then out[#out+1] = l end
	end
	box.Text = table.concat(out, "\n")
end

-- ================= LIBRARY ROWS =================
for i, e in ipairs(LIBRARY) do
	local name, sub = e[1], e[2]
	local row = new("TextButton", libList, {BackgroundColor3 = T.panel, BackgroundTransparency = .2, Text = "", AutoButtonColor = false, LayoutOrder = i})
	round(row, 9)
	local st = stroke(row, 1, "line")
	libScales[#libScales+1] = new("UIScale", row, {Scale = 1})
	local lbl = new("TextLabel", row, {Size = UDim2.new(1,-50,0,16), Position = UDim2.fromOffset(9,5), BackgroundTransparency = 1, Text = name, TextColor3 = T.text, Font = Enum.Font.GothamBold, TextSize = 10, TextXAlignment = Enum.TextXAlignment.Left, TextTruncate = Enum.TextTruncate.AtEnd})
	local sl = new("TextLabel", row, {Size = UDim2.new(1,-50,0,12), Position = UDim2.fromOffset(9,22), BackgroundTransparency = 1, Text = sub, TextColor3 = T.dim, Font = Enum.Font.Gotham, TextSize = 8, TextXAlignment = Enum.TextXAlignment.Left, TextTruncate = Enum.TextTruncate.AtEnd})
	local badge = new("TextLabel", row, {Size = UDim2.fromOffset(40,14), Position = UDim2.new(1,-44,.5,-7), BackgroundTransparency = 1, Text = "", TextColor3 = T.dim, Font = Enum.Font.GothamBold, TextSize = 7})
	local bs = stroke(badge, 1, T.accent, 1) round(badge, 7)
	local state = "idle"
	paints[name] = function(s)
		state = s
		local live, queued = s == "live", s == "queued"
		tw(row, .25, {BackgroundColor3 = live and T.accent or T.panel, BackgroundTransparency = live and 0 or .2})
		tw(lbl, .25, {TextColor3 = live and T.onAccent or T.text})
		tw(sl, .25, {TextColor3 = live and T.line or T.dim})
		tw(badge, .25, {TextColor3 = live and T.onAccent or (queued and T.text or T.dim)})
		tw(bs, .25, {Transparency = (live or queued) and 0 or 1, Color = live and T.onAccent or T.accent})
		badge.Text = live and "LIVE" or (queued and "QUEUED" or "")
	end
	row.MouseEnter:Connect(function() tw(st, .2, {Color = T.accent}) end)
	row.MouseLeave:Connect(function() tw(st, .2, {Color = T.line}) end)
	row.MouseButton1Click:Connect(function()
		if not injected then needInject() return end
		if state == "live" then
			Mono.stop(name) setStatus(name .. " stopped", "dim")
		elseif state == "queued" then
			removeLine(name) setStatus("Removed: " .. name, "dim")
		else
			addLine(name) setStatus("Queued · press EXECUTE", GREEN)
		end
	end)
end
stopAll.MouseButton1Click:Connect(function() Mono.stopAll() setStatus("All stopped", "dim") end)

-- ================= SAVED SCRIPTS =================
local function rebuildSaved()
	for _, c in ipairs(savedList:GetChildren()) do if c:IsA("GuiObject") then c:Destroy() end end
	savedCount.Text = #Data.scripts .. " saved"
	if #Data.scripts == 0 then
		new("TextLabel", savedList, {Size = UDim2.new(1,0,0,50), BackgroundTransparency = 1, Text = "Nothing saved yet.\nType a name, paste a script and press SAVE.", TextColor3 = r("dim"), Font = Enum.Font.Gotham, TextSize = 10, TextWrapped = true})
		return
	end
	for i, s in ipairs(Data.scripts) do
		local row = new("TextButton", savedList, {Size = UDim2.new(1,0,0,40), BackgroundColor3 = r("panel"), BackgroundTransparency = .2, Text = "", AutoButtonColor = false, LayoutOrder = i})
		round(row, 9)
		local st = stroke(row, 1, "line")
		new("TextLabel", row, {Size = UDim2.new(1,-62,0,18), Position = UDim2.fromOffset(10,5), BackgroundTransparency = 1, Text = s.name, TextColor3 = r("text"), Font = Enum.Font.GothamBold, TextSize = 11, TextXAlignment = Enum.TextXAlignment.Left, TextTruncate = Enum.TextTruncate.AtEnd})
		local first = (tostring(s.code):match("[^\r\n]+")) or ""
		new("TextLabel", row, {Size = UDim2.new(1,-62,0,12), Position = UDim2.fromOffset(10,24), BackgroundTransparency = 1, Text = first, TextColor3 = r("dim"), Font = Enum.Font.Code, TextSize = 9, TextXAlignment = Enum.TextXAlignment.Left, TextTruncate = Enum.TextTruncate.AtEnd})
		local del = new("TextButton", row, {Size = UDim2.fromOffset(40,24), Position = UDim2.new(1,-46,.5,-12), Text = "✕", Font = Enum.Font.GothamBold, TextSize = 9, AutoButtonColor = false})
		round(del, 7)
		styled(del, "panel2", "dim", "accent", "onAccent")
		local armed = false
		del.MouseButton1Click:Connect(function()
			if not armed then
				armed = true
				del.Text = "DEL?"
				task.delay(2, function() if armed and del.Parent then armed = false del.Text = "✕" end end)
				return
			end
			for k, v in ipairs(Data.scripts) do if v == s then table.remove(Data.scripts, k) break end end
			saveData()
			rebuildSaved()
			toast("Deleted: " .. s.name)
		end)
		row.MouseEnter:Connect(function() tw(st, .2, {Color = T.accent}) end)
		row.MouseLeave:Connect(function() tw(st, .2, {Color = T.line}) end)
		row.MouseButton1Click:Connect(function()
			box.Text = s.code
			toast("Loaded: " .. s.name)
			setTab("EDITOR")
		end)
	end
end

local function saveScript()
	local code = codeBox.Text
	if (code:gsub("%s", "")) == "" then toast("Paste a script first") return end
	local name = (nameBox.Text:gsub("^%s+", ""):gsub("%s+$", ""))
	if name == "" then name = "Script " .. (#Data.scripts + 1) end
	local found
	for _, v in ipairs(Data.scripts) do if v.name == name then found = v break end end
	if found then found.code = code else Data.scripts[#Data.scripts+1] = {name = name, code = code} end
	local ok = saveData() and (not canFS or isfile(FILE))
	rebuildSaved()
	if canFS and ok then toast((found and "Updated: " or "Saved: ") .. name .. " ✓")
	elseif canFS then toast("Save failed (file error)")
	else toast("Saved in memory only (no writefile)") end
	log("● saved script: " .. name, GREEN)
	nameBox.Text, codeBox.Text = "", ""
end
saveBtn.MouseButton1Click:Connect(saveScript)
rebuildSaved()

-- ================= THEME CARDS =================
local themeCards = {}
local function updateThemeCards(dur)
	for name, c in pairs(themeCards) do
		local sel = name == Data.theme
		tw(c.stroke, dur or .25, {Color = sel and T.accent or T.line, Thickness = sel and 1.8 or 1})
		tw(c.check, dur or .25, {TextTransparency = sel and 0 or 1})
	end
end
for i, def in ipairs(THEMES) do
	local card = new("TextButton", themeList, {BackgroundColor3 = r("panel"), BackgroundTransparency = .2, Text = "", AutoButtonColor = false, LayoutOrder = i})
	round(card, 9)
	local st = stroke(card, 1, T.line)
	local sw = new("Frame", card, {Size = UDim2.fromOffset(22,22), Position = UDim2.fromOffset(10,9), BackgroundColor3 = def.accent})
	round(sw, 11) stroke(sw, 2, def.line)
	new("TextLabel", card, {Size = UDim2.new(1,-74,1,0), Position = UDim2.fromOffset(40,0), BackgroundTransparency = 1, Text = def.name, TextColor3 = r("text"), Font = Enum.Font.GothamBold, TextSize = 11, TextXAlignment = Enum.TextXAlignment.Left})
	local check = new("TextLabel", card, {Size = UDim2.fromOffset(24,40), Position = UDim2.new(1,-28,0,0), BackgroundTransparency = 1, Text = "✓", TextColor3 = r("accent"), Font = Enum.Font.GothamBold, TextSize = 14, TextTransparency = 1})
	themeCards[def.name] = {stroke = st, check = check}
	card.MouseEnter:Connect(function() tw(st, .2, {Color = T.accent}) end)
	card.MouseLeave:Connect(function() tw(st, .2, {Color = def.name == Data.theme and T.accent or T.line}) end)
	card.MouseButton1Click:Connect(function()
		applyTheme(def.name)
		saveData()
		toast("Theme: " .. def.name)
	end)
end

postApply = function(dur)
	for _, f in ipairs(togglePaints) do f(dur) end
	mainGrad.Color = ColorSequence.new(T.bg2, T.bg)
	titleGrad.Color = ColorSequence.new(T.text, T.dim)
	if not injected then tw(pillStroke, dur, {Color = T.line}) end
	styleTabs(dur)
	repaintAll()
	updateThemeCards(dur)
end
styleTabs(0)
updateThemeCards(0)

-- ================= SCRIPT HUB LOGIC (multi-source) =================
local function httpGet(u)
	local ok, res = pcall(function() return game:HttpGet(u) end)
	if ok and type(res) == "string" then return true, res end
	local ok2, res2 = pcall(function() return HttpService:GetAsync(u) end)
	if ok2 then return true, res2 end
	return false, tostring(res2 or res)
end

local function parseSource(src, data)
	local out = {}
	local ok, list = pcall(src.list, data)
	if not ok or type(list) ~= "table" then return out end
	for _, s in ipairs(list) do
		local code = s.script or s.code or s.source
		if (not code or code == "") then
			local rawUrl = s.rawScript or s.raw or s.rawUrl
			if type(rawUrl) == "string" and rawUrl ~= "" then
				code = 'loadstring(game:HttpGet("' .. rawUrl .. '"))()'
			elseif (s.slug or s._id) and src.raw then
				code = 'loadstring(game:HttpGet("' .. src.raw .. tostring(s.slug or s._id) .. '"))()'
			end
		end
		if code and code ~= "" then
			local g = s.game
			local gname = (type(g) == "table" and (g.name or g.title)) or g or s.gameName or "Universal"
			out[#out+1] = {
				title = tostring(s.title or s.name or "Untitled"),
				game = tostring(gname),
				views = tonumber(s.views) or 0,
				verified = s.verified == true,
				code = code, src = src.name, tag = src.tag,
			}
		end
	end
	return out
end

local hubToken = 0
local function clearHub()
	for _, c in ipairs(hubList:GetChildren()) do if c:IsA("GuiObject") then c:Destroy() end end
end

local function showResults(results)
	clearHub()
	for i, res in ipairs(results) do
		local card = new("TextButton", hubList, {Size = UDim2.new(1,0,0,42), BackgroundColor3 = r("panel"), BackgroundTransparency = .2, Text = "", AutoButtonColor = false, LayoutOrder = i})
		round(card, 9)
		local st = stroke(card, 1, "line")
		local cs = new("UIScale", card, {Scale = .85})
		new("TextLabel", card, {Size = UDim2.new(1,-84,0,18), Position = UDim2.fromOffset(10,5), BackgroundTransparency = 1, Text = (res.verified and "✔ " or "") .. res.title, TextColor3 = r("text"), Font = Enum.Font.GothamBold, TextSize = 11, TextXAlignment = Enum.TextXAlignment.Left, TextTruncate = Enum.TextTruncate.AtEnd})
		new("TextLabel", card, {Size = UDim2.new(1,-84,0,12), Position = UDim2.fromOffset(10,24), BackgroundTransparency = 1, Text = res.tag .. " · " .. res.game .. " · " .. res.views .. " views", TextColor3 = r("dim"), Font = Enum.Font.Gotham, TextSize = 9, TextXAlignment = Enum.TextXAlignment.Left, TextTruncate = Enum.TextTruncate.AtEnd})
		local ld = new("TextLabel", card, {Size = UDim2.fromOffset(52,22), Position = UDim2.new(1,-60,.5,-11), BackgroundColor3 = r("panel2"), Text = "LOAD", TextColor3 = r("text"), Font = Enum.Font.GothamBold, TextSize = 9})
		round(ld, 6)
		card.MouseEnter:Connect(function() tw(st, .2, {Color = T.accent}) tw(ld, .2, {BackgroundColor3 = T.accent, TextColor3 = T.onAccent}) end)
		card.MouseLeave:Connect(function() tw(st, .2, {Color = T.line}) tw(ld, .2, {BackgroundColor3 = T.panel2, TextColor3 = T.text}) end)
		card.MouseButton1Click:Connect(function()
			box.Text = "-- " .. res.title .. " (" .. res.src .. ")\n" .. res.code
			toast("Loaded: " .. res.title)
			log("● loaded from " .. res.src .. ": " .. res.title, GREEN)
			setTab("EDITOR")
		end)
		task.delay(math.min(i, 12) * .035, function() if card.Parent then tw(cs, .45, {Scale = 1}, Enum.EasingStyle.Back) end end)
	end
end

local function hubSearch(query)
	hubToken += 1
	local my = hubToken
	query = (query or ""):gsub("^%s+", ""):gsub("%s+$", "")
	hubInfo.Text = (query == "" and "Loading popular" or ("Searching \"" .. query .. "\"")) .. " in " .. #SOURCES .. " sources..."
	local all, done, okCount, fails = {}, 0, 0, {}
	for _, src in ipairs(SOURCES) do
		task.spawn(function()
			local u = query == "" and src.trending or (src.search .. HttpService:UrlEncode(query))
			local ok, body = httpGet(u)
			if ok then
				local okJ, data = pcall(function() return HttpService:JSONDecode(body) end)
				if okJ and type(data) == "table" then
					for _, x in ipairs(parseSource(src, data)) do all[#all+1] = x end
					okCount += 1
				else
					fails[#fails+1] = src.tag .. " (bad JSON)"
				end
			else
				fails[#fails+1] = src.tag
				log("✖ hub " .. src.name .. ": " .. tostring(body), RED)
			end
			done += 1
		end)
	end
	task.spawn(function()
		local t0 = os.clock()
		while done < #SOURCES and os.clock() - t0 < 12 do task.wait(.1) end
		if my ~= hubToken then return end
		table.sort(all, function(a, b) return a.views > b.views end)
		showResults(all)
		local msg = #all .. " results from " .. okCount .. "/" .. #SOURCES .. " sources"
		if #fails > 0 then msg ..= " · failed: " .. table.concat(fails, ", ") end
		hubInfo.Text = msg .. " · read scripts before EXECUTE"
	end)
end

goBtn.MouseButton1Click:Connect(function() hubSearch(search.Text) end)
search.FocusLost:Connect(function(enter) if enter then hubSearch(search.Text) end end)
local hubLoaded = false
tabBtns["HUB"].MouseButton1Click:Connect(function()
	if not hubLoaded then hubLoaded = true hubSearch("") end
end)

-- ================= INJECT / EXECUTE / CLEAR =================
local function setInjected(on)
	injected = on
	token += 1
	local my = token
	if on then
		tw(dot, .35, {BackgroundColor3 = GREEN})
		tw(glow, .35, {BackgroundColor3 = GREEN})
		tw(status, .35, {TextColor3 = GREEN})
		tw(pillStroke, .35, {Color = GREEN, Transparency = .5})
		status.Text = "Injected"
		injectBtn.Text = "DETACH"
		logConn = LogService.MessageOut:Connect(function(msg, t)
			if msg:sub(1,8) == "[Zenith]" then return end
			if t == Enum.MessageType.MessageError then log("✖ " .. msg, RED)
			elseif t == Enum.MessageType.MessageWarning then log("▲ " .. msg, "mid")
			elseif t == Enum.MessageType.MessageOutput then log("› " .. msg, "dim") end
		end)
		log("● injected", GREEN)
		task.spawn(function()
			while injected and token == my do
				tw(glow, .9, {Size = UDim2.fromOffset(22,22), BackgroundTransparency = .85}, Enum.EasingStyle.Sine)
				task.wait(.9)
				tw(glow, .9, {Size = UDim2.fromOffset(10,10), BackgroundTransparency = .45}, Enum.EasingStyle.Sine)
				task.wait(.9)
			end
		end)
	else
		tw(dot, .35, {BackgroundColor3 = RED})
		tw(glow, .35, {BackgroundColor3 = RED, Size = UDim2.fromOffset(10,10), BackgroundTransparency = .55})
		tw(status, .35, {TextColor3 = RED})
		tw(pillStroke, .35, {Color = T.line, Transparency = 0})
		status.Text = "Not injected"
		injectBtn.Text = "INJECT"
		if logConn then logConn:Disconnect() logConn = nil end
		log("○ detached (running scripts stay on)", "dim")
	end
end
injectBtn.MouseButton1Click:Connect(function() setInjected(not injected) end)

clearBtn.MouseButton1Click:Connect(function()
	if currentTab == "CONSOLE" then
		for _, k in ipairs(conScroll:GetChildren()) do if k:IsA("TextLabel") then k:Destroy() end end
		setStatus("Console cleared", "dim")
	else
		box.Text = ""
		setStatus(injected and "Injected" or "Not injected")
	end
end)

executeBtn.MouseButton1Click:Connect(function()
	if not injected then needInject() return end
	if (box.Text:gsub("%s", "")) == "" then setStatus("Editor is empty", "dim") return end
	if not loadstring then
		setStatus("loadstring disabled", RED)
		log("✖ loadstring is not available in this environment", RED)
		return
	end
	local fn, err = loadstring(box.Text)
	if not fn then
		setStatus("Syntax error", RED)
		log("✖ " .. tostring(err), RED)
		setTab("CONSOLE")
		return
	end
	local ok, runErr = pcall(fn)
	if ok then setStatus("Executed ✓", GREEN) toast("Executed ✓")
	else
		setStatus("Runtime error", RED)
		log("✖ " .. tostring(runErr), RED)
		setTab("CONSOLE")
	end
end)

-- ================= OPEN / CLOSE (ANIMATIONS) =================
local openToken = 0
local function setOpen(v)
	open = v
	openToken += 1
	local my = openToken
	computeFit()
	local c = icon.AbsolutePosition + icon.AbsoluteSize / 2
	if v then
		main.Position = UDim2.fromOffset(c.X, c.Y)
		main.Visible = true
		scale.Scale = FIT * .3
		for _, p in pairs(pages) do p.Visible = false p.GroupTransparency = 1 end
		barLine.Size = UDim2.new(0,0,0,1)
		titleLbl.TextTransparency = 1
		logoMini.BackgroundTransparency = 1
		pill.BackgroundTransparency = 1
		indicator.BackgroundTransparency = 1
		for _, t in ipairs(TABS) do
			local b, g = tabBtns[t[1]], tabGlyph[t[1]]
			b.Position = UDim2.new(0,-34,0,tabY[t[1]])
			b.TextTransparency = 1
			g.TextTransparency = 1
		end
		for _, b in ipairs(btnList) do
			local bp = btnBase[b]
			b.Position = UDim2.new(bp.X.Scale, bp.X.Offset, 0, BOT_Y + 28)
			b.BackgroundTransparency = 1
			b.TextTransparency = 1
		end
		curtain.Size = UDim2.fromScale(1,1)
		curtain.Visible = true

		tw(main, .55, {GroupTransparency = 0, Position = UDim2.fromScale(.5,.5)}, Enum.EasingStyle.Exponential)
		tw(scale, .7, {Scale = FIT}, Enum.EasingStyle.Back)
		tw(iconStroke, .4, {Thickness = 3.5})
		tw(ring, .5, {Size = UDim2.fromOffset(38,38)}, Enum.EasingStyle.Back)

		task.delay(.12, function()
			local t = tw(curtain, .7, {Size = UDim2.fromScale(0,1)}, Enum.EasingStyle.Quint, Enum.EasingDirection.InOut)
			t.Completed:Once(function() if openToken == my then curtain.Visible = false end end)
		end)

		task.delay(.35, function()
			tw(barLine, .8, {Size = UDim2.new(1,0,0,1)}, Enum.EasingStyle.Exponential)
			tw(titleLbl, .5, {TextTransparency = 0})
			tw(logoMini, .5, {BackgroundTransparency = 0})
			tw(pill, .5, {BackgroundTransparency = 0})
		end)

		for i, t in ipairs(TABS) do
			task.delay(.4 + i * .06, function()
				if openToken ~= my then return end
				tw(tabBtns[t[1]], .55, {Position = UDim2.new(0,4,0,tabY[t[1]]), TextTransparency = 0}, Enum.EasingStyle.Back)
				tw(tabGlyph[t[1]], .55, {TextTransparency = 0})
			end)
		end
		task.delay(.85, function() if openToken == my then tw(indicator, .5, {BackgroundTransparency = .88}) end end)

		for i, b in ipairs(btnList) do
			task.delay(.55 + i * .08, function()
				if openToken ~= my then return end
				tw(b, .6, {Position = btnBase[b], BackgroundTransparency = 0, TextTransparency = 0}, Enum.EasingStyle.Back)
			end)
		end

		local keep = currentTab or "EDITOR"
		currentTab = nil
		task.delay(.5, function() if openToken == my then setTab(keep) end end)
	else
		tw(main, .4, {GroupTransparency = 1, Position = UDim2.fromOffset(c.X, c.Y)}, Enum.EasingStyle.Quint, Enum.EasingDirection.In)
		local t = tw(scale, .4, {Scale = FIT * .3}, Enum.EasingStyle.Quint, Enum.EasingDirection.In)
		tw(iconStroke, .4, {Thickness = 1.8})
		tw(ring, .4, {Size = UDim2.fromOffset(31,31)})
		t.Completed:Once(function() if not open then main.Visible = false end end)
	end
end
close.MouseButton1Click:Connect(function() setOpen(false) end)

-- ================= DRAG =================
local function makeDrag(handle, target, onClick)
	local dragging, moved, ds, sp = false, false
	handle.InputBegan:Connect(function(i)
		if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
			dragging, moved, ds, sp = true, false, i.Position, target.Position
			i.Changed:Connect(function()
				if i.UserInputState == Enum.UserInputState.End then
					dragging = false
					if not moved and onClick then onClick() end
				end
			end)
		end
	end)
	UIS.InputChanged:Connect(function(i)
		if dragging and (i.UserInputType == Enum.UserInputType.MouseMovement or i.UserInputType == Enum.UserInputType.Touch) then
			local d = i.Position - ds
			if d.Magnitude > 6 then moved = true end
			if moved then target.Position = UDim2.new(sp.X.Scale, sp.X.Offset + d.X, sp.Y.Scale, sp.Y.Offset + d.Y) end
		end
	end)
end
makeDrag(icon, icon, function() setOpen(not open) end)
makeDrag(bar, main)

-- ================= LIGHT DUST =================
local dots = {}
for i = 1, 14 do
	local s = rnd:NextInteger(2, 3)
	local f = new("Frame", dustLayer, {Size = UDim2.fromOffset(s,s), BackgroundColor3 = r("accent"), BorderSizePixel = 0})
	round(f, 2)
	dots[i] = {f = f, x = rnd:NextNumber(), y = rnd:NextNumber(), vy = rnd:NextNumber(.01, .03), ph = rnd:NextNumber(0, 6)}
end
RS.RenderStepped:Connect(function(dt)
	if not main.Visible then return end
	local t = os.clock()
	for _, p in ipairs(dots) do
		p.y -= p.vy * dt
		if p.y < -.03 then p.y = 1.03 p.x = rnd:NextNumber() end
		p.f.Position = UDim2.fromScale(p.x + math.sin(t + p.ph) * .004, p.y)
		p.f.BackgroundTransparency = .75 + .2 * math.sin(t * 2 + p.ph)
	end
end)

updateLines()
log("Zenith ready. Theme: " .. Data.theme .. " · " .. #Data.scripts .. " saved scripts loaded.", "dim")
if not canFS then log("▲ writefile not available: saves, theme and settings will not persist after restart", RED) end

-- ================= AUTO ATTACH =================
if Data.autoAttach then
	task.defer(function()
		setInjected(true)
		toast("Auto Attach ✓")
		if Data.autoOpen then setOpen(true) end
	end)
end
