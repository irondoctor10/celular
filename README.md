--[[
    NOVA OS 3.2
    Celular in-game para roleplay Roblox (iOS + Android).

    Local: StarterPlayer > StarterPlayerScripts
    Classe: LocalScript

    Integracao opcional:
      ReplicatedStorage > WorldState  (Attributes: ClockTime, Weather, DayPhase, Temperature)
      Player Attributes: Money (number), Job (string)
      ReplicatedStorage > NOVA_Notify (RemoteEvent) -> FireClient(player, titulo, mensagem, appId)
      PlayerGui > NOVA_OS > Notify (BindableEvent) -> :Fire(titulo, mensagem, appId)

    Icones proprios: preencha ICON_IMAGES com rbxassetid://... para trocar
    qualquer icone desenhado por uma imagem sua.
]]

--========================================================
-- SERVICES
--========================================================
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local UserInputService = game:GetService("UserInputService")
local ContextActionService = game:GetService("ContextActionService")
local TweenService = game:GetService("TweenService")
local Lighting = game:GetService("Lighting")

local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")
local WorldState = ReplicatedStorage:WaitForChild("WorldState", 10)

--========================================================
-- CONFIG
--========================================================
local PHONE_W, PHONE_H = 290, 590

local C = {
	Bg = Color3.fromRGB(8, 10, 16),
	Surface = Color3.fromRGB(22, 26, 38),
	SurfaceHi = Color3.fromRGB(34, 40, 56),
	Border = Color3.fromRGB(62, 70, 92),
	Text = Color3.fromRGB(246, 247, 252),
	Muted = Color3.fromRGB(150, 158, 178),
	Accent = Color3.fromRGB(10, 132, 255),
	Green = Color3.fromRGB(48, 209, 88),
	Red = Color3.fromRGB(255, 69, 58),
}

local ACCENTS = {
	Color3.fromRGB(10, 132, 255),
	Color3.fromRGB(48, 209, 88),
	Color3.fromRGB(191, 90, 242),
	Color3.fromRGB(255, 159, 10),
	Color3.fromRGB(255, 69, 58),
	Color3.fromRGB(100, 210, 255),
}

local settings = {
	dnd = false, wifi = true, bluetooth = true,
	flash = false, airplane = false, rotation = true,
	brightness = 1,
}

-- Exemplo: ICON_IMAGES.Bank = "rbxassetid://123456789"
local ICON_IMAGES = {}

local W = Color3.new(1, 1, 1)

--========================================================
-- HELPERS
--========================================================
local function create(className, props, parent)
	local o = Instance.new(className)
	if o:IsA("GuiObject") then
		o.BorderSizePixel = 0
	end
	for k, v in pairs(props or {}) do
		o[k] = v
	end
	o.Parent = parent
	return o
end

local function corner(parent, radius)
	return create("UICorner", {CornerRadius = UDim.new(0, radius)}, parent)
end

local function stroke(parent, color, transparency, thickness)
	return create("UIStroke", {
		Color = color or C.Border,
		Transparency = transparency or 0,
		Thickness = thickness or 1,
		ApplyStrokeMode = Enum.ApplyStrokeMode.Border,
	}, parent)
end

local function label(parent, str, size, props)
	local l = create("TextLabel", {
		BackgroundTransparency = 1,
		Text = str,
		Font = Enum.Font.Gotham,
		TextColor3 = C.Text,
		TextSize = size or 12,
		TextXAlignment = Enum.TextXAlignment.Left,
		TextYAlignment = Enum.TextYAlignment.Center,
	}, parent)
	for k, v in pairs(props or {}) do
		l[k] = v
	end
	return l
end

local function animate(object, props, duration, style)
	local tween = TweenService:Create(
		object,
		TweenInfo.new(duration or 0.22, style or Enum.EasingStyle.Quart, Enum.EasingDirection.Out),
		props
	)
	tween:Play()
	return tween
end

local function pressable(button, strength)
	local scale = create("UIScale", {Scale = 1}, button)
	button.InputBegan:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1
			or input.UserInputType == Enum.UserInputType.Touch then
			animate(scale, {Scale = strength or 0.92}, 0.09)
		end
	end)
	button.InputEnded:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1
			or input.UserInputType == Enum.UserInputType.Touch then
			animate(scale, {Scale = 1}, 0.2, Enum.EasingStyle.Back)
		end
	end)
end

local function counter()
	local n = 0
	return function()
		n += 1
		return n
	end
end

local function formatNumber(n)
	local s = tostring(math.floor(n))
	local r = s:reverse():gsub("(%d%d%d)", "%1."):reverse()
	return (r:gsub("^%.", ""))
end

--========================================================
-- WORLD STATE
--========================================================
local function worldAttribute(name, fallback)
	if not WorldState then return fallback end
	local v = WorldState:GetAttribute(name)
	if v == nil then return fallback end
	return v
end

local function getClock()
	local clock = worldAttribute("ClockTime", Lighting.ClockTime)
	if typeof(clock) ~= "number" then clock = Lighting.ClockTime end
	clock %= 24
	local h = math.floor(clock)
	local m = math.floor((clock - h) * 60)
	return string.format("%02d:%02d", h, m)
end

local function getWeather() return tostring(worldAttribute("Weather", "Sunny")) end
local function getPhase() return tostring(worldAttribute("DayPhase", "")) end

local WEATHER_NAMES = {
	Sunny = "Ensolarado", Cloudy = "Nublado", Rain = "Chuvoso", Storm = "Tempestade",
	Fog = "Neblina", Drizzle = "Garoa", Clear = "Ceu limpo",
}

local function weatherName()
	return WEATHER_NAMES[getWeather()] or getWeather()
end

local function weatherKind()
	local w = getWeather()
	if w == "Rain" or w == "Drizzle" then return "Rain"
	elseif w == "Storm" then return "Storm"
	elseif w == "Fog" then return "Fog"
	elseif w == "Cloudy" then return "Cloud" end
	return "Sun"
end

local MONTHS = {"Janeiro","Fevereiro","Marco","Abril","Maio","Junho","Julho","Agosto","Setembro","Outubro","Novembro","Dezembro"}
local WEEKDAYS = {"Domingo","Segunda-feira","Terca-feira","Quarta-feira","Quinta-feira","Sexta-feira","Sabado"}
local WEEKDAYS_SHORT = {"DOM","SEG","TER","QUA","QUI","SEX","SAB"}

-- Data do mundo: usa os Attributes Day, Month e Year do WorldState.
-- Sem eles, cai para a data real em horario de Brasilia (UTC-3).
local function getDate()
	local day = worldAttribute("Day", nil)
	local month = worldAttribute("Month", nil)

	if typeof(day) == "number" and typeof(month) == "number" then
		local year = worldAttribute("Year", nil)
		if typeof(year) ~= "number" then
			year = os.date("*t").year
		end
		local t = os.date("*t", os.time({year = year, month = month, day = day, hour = 12}))
		return {year = t.year, month = t.month, day = t.day, wday = t.wday}
	end

	local t = os.date("*t", os.time() - 3 * 3600)
	return {year = t.year, month = t.month, day = t.day, wday = t.wday}
end

local function dateString()
	local d = getDate()
	return string.format("%s, %d de %s", WEEKDAYS[d.wday], d.day, MONTHS[d.month])
end

--========================================================
-- APPS
--========================================================
local function app(id, name, c1, c2, desc)
	return {id = id, name = name, c1 = c1, c2 = c2, hole = c1:Lerp(c2, 0.5), desc = desc}
end
local rgb = Color3.fromRGB

local APPS = {
	app("Profile", "Perfil", rgb(100,176,255), rgb(28,98,230), ""),
	app("GPS", "GPS", rgb(70,226,196), rgb(0,148,160), ""),
	app("Garage", "Garagem", rgb(84,100,130), rgb(32,40,64), "Seus veiculos, chaves e a localizacao do carro vao aparecer aqui."),
	app("People", "Pessoas", rgb(200,120,255), rgb(112,62,216), ""),

	app("Chat", "Mensagens", rgb(96,226,116), rgb(30,170,72), "Conversas com outros jogadores e contatos da cidade."),
	app("Bank", "Banco", rgb(255,218,80), rgb(240,148,20), "Saldo, transferencias e faturas conectados a economia do servidor."),
	app("Properties", "Imoveis", rgb(255,120,120), rgb(220,45,70), "Casas, aluguel e chaves dos seus imoveis."),
	app("Work", "Trabalho", rgb(255,176,80), rgb(240,100,30), "Empregos, turnos e salario."),

	app("Inventory", "Inventario", rgb(206,156,100), rgb(138,92,48), ""),
	app("News", "Noticias", rgb(100,140,255), rgb(40,70,200), "Noticias e anuncios da cidade."),
	app("Weather", "Clima", rgb(90,196,255), rgb(30,118,232), ""),
	app("Calendar", "Calendario", rgb(255,255,255), rgb(228,231,238), ""),

	app("Settings", "Ajustes", rgb(158,166,184), rgb(88,96,116), ""),
	app("Help", "Ajuda", rgb(160,130,255), rgb(92,60,212), ""),
	app("Events", "Eventos", rgb(255,96,176), rgb(208,30,120), "Eventos, shows e encontros da cidade."),
}

local APP_BY_ID = {}
for _, a in ipairs(APPS) do APP_BY_ID[a.id] = a end

--========================================================
-- GLYPH ENGINE (icones vetoriais em grade 0-100)
--========================================================
local function makeGlyph(parent, size)
	local z = 2
	local g = {}

	function g.box(x, y, w, h, color, r, rot, tr)
		z += 1
		local f = create("Frame", {
			Position = UDim2.fromScale(x / 100, y / 100),
			Size = UDim2.fromScale(w / 100, h / 100),
			BackgroundColor3 = color,
			BackgroundTransparency = tr or 0,
			Rotation = rot or 0,
			ZIndex = z,
		}, parent)
		if r and r > 0 then
			create("UICorner", {CornerRadius = UDim.new(r, 0)}, f)
		end
		return f
	end

	function g.dot(cx, cy, d, color, tr)
		return g.box(cx - d / 2, cy - d / 2, d, d, color, 0.5, 0, tr)
	end

	function g.ring(cx, cy, d, t, color)
		local f = g.box(cx - d / 2, cy - d / 2, d, d, color, 0.5, 0, 1)
		create("UIStroke", {
			Color = color,
			Thickness = math.max(1, t * size / 100),
			ApplyStrokeMode = Enum.ApplyStrokeMode.Border,
		}, f)
		return f
	end

	function g.text(str, x, y, w, h, color, font, frac)
		z += 1
		return create("TextLabel", {
			Position = UDim2.fromScale(x / 100, y / 100),
			Size = UDim2.fromScale(w / 100, h / 100),
			BackgroundTransparency = 1,
			Text = str,
			Font = font or Enum.Font.GothamBold,
			TextColor3 = color,
			TextSize = math.max(8, math.floor(frac * size)),
			ZIndex = z,
		}, parent)
	end

	return g
end

local GLYPHS = {}

GLYPHS.Profile = function(g)
	g.box(22, 58, 56, 46, W, 0.45)
	g.dot(50, 34, 28, W)
end

GLYPHS.GPS = function(g, hole)
	g.box(34, 82, 32, 5, W, 0.5, 0, 0.55)
	g.box(35, 40, 30, 30, W, 0.12, 45)
	g.dot(50, 40, 44, W)
	g.dot(50, 40, 17, hole)
end

GLYPHS.Garage = function(g, hole)
	g.box(12, 46, 76, 26, W, 0.3)
	g.box(24, 30, 52, 26, W, 0.35)
	g.box(31, 37, 17, 13, hole, 0.2)
	g.box(52, 37, 17, 13, hole, 0.2)
	for _, x in ipairs({29, 71}) do
		g.dot(x, 72, 28, hole)
		g.dot(x, 72, 20, W)
		g.dot(x, 72, 8, hole)
	end
end

GLYPHS.People = function(g)
	g.box(50, 60, 34, 46, W, 0.45, 0, 0.5)
	g.dot(68, 40, 20, W, 0.5)
	g.box(14, 58, 44, 46, W, 0.45)
	g.dot(36, 36, 24, W)
end

GLYPHS.Chat = function(g, hole)
	g.box(22, 60, 18, 18, W, 0.12, 45)
	g.box(14, 20, 72, 52, W, 0.3)
	g.dot(34, 46, 9, hole)
	g.dot(50, 46, 9, hole)
	g.dot(66, 46, 9, hole)
end

GLYPHS.Bank = function(g)
	g.box(28, 14, 44, 14, W, 0.3)
	g.box(14, 30, 72, 9, W, 0.25)
	g.box(22, 43, 12, 31, W, 0.25)
	g.box(44, 43, 12, 31, W, 0.25)
	g.box(66, 43, 12, 31, W, 0.25)
	g.box(14, 77, 72, 9, W, 0.25)
end

GLYPHS.Properties = function(g, hole)
	g.box(31, 20, 38, 38, W, 0.08, 45)
	g.box(24, 48, 52, 36, W, 0.1)
	g.box(43, 62, 14, 22, hole, 0.2)
end

GLYPHS.Work = function(g, hole)
	g.box(36, 18, 28, 26, W, 0.25)
	g.box(43, 25, 14, 12, hole, 0.2)
	g.box(14, 34, 72, 50, W, 0.16)
	g.box(14, 54, 72, 5, hole, 0, 0, 0.5)
	g.box(44, 49, 12, 14, hole, 0.2)
end

GLYPHS.Inventory = function(g, hole)
	g.box(34, 16, 32, 26, W, 0.5)
	g.box(22, 30, 56, 56, W, 0.28)
	g.box(32, 56, 36, 22, hole, 0.25)
	g.box(22, 47, 56, 4, hole, 0, 0, 0.5)
end

GLYPHS.News = function(g, hole)
	g.box(18, 14, 64, 72, W, 0.1)
	g.box(26, 22, 48, 14, hole, 0.15)
	g.box(26, 44, 48, 5, hole, 0.5, 0, 0.35)
	g.box(26, 55, 48, 5, hole, 0.5, 0, 0.35)
	g.box(26, 66, 30, 5, hole, 0.5, 0, 0.35)
end

GLYPHS.Weather = function(g)
	g.dot(34, 34, 40, rgb(255, 214, 10))
	g.dot(36, 56, 28, W)
	g.dot(54, 48, 36, W)
	g.box(22, 58, 62, 24, W, 0.5)
end

GLYPHS.Calendar = function(g)
	local d = getDate()
	g.box(0, 0, 100, 32, rgb(255, 69, 58))
	g.text(WEEKDAYS_SHORT[d.wday], 0, 5, 100, 22, W, Enum.Font.GothamBold, 0.17)
	g.text(tostring(d.day), 0, 32, 100, 66, rgb(30, 33, 42), Enum.Font.GothamBold, 0.5)
end

GLYPHS.Settings = function(g, hole)
	for i = 0, 7 do
		local a = math.rad(i * 45)
		local cx, cy = 50 + math.cos(a) * 28, 50 + math.sin(a) * 28
		g.box(cx - 6, cy - 8, 12, 16, W, 0.2, i * 45 - 90)
	end
	g.dot(50, 50, 52, W)
	g.dot(50, 50, 22, hole)
end

GLYPHS.Help = function(g)
	g.ring(50, 50, 64, 7, W)
	g.text("?", 0, 0, 100, 100, W, Enum.Font.GothamBold, 0.46)
end

GLYPHS.Events = function(g, hole)
	g.box(10, 30, 80, 40, W, 0.2)
	g.dot(10, 50, 18, hole)
	g.dot(90, 50, 18, hole)
	for _, y in ipairs({36, 47, 58}) do
		g.box(62, y, 3.5, 7, hole, 0.3, 0, 0.4)
	end
	g.box(22, 42, 26, 5, hole, 0.5, 0, 0.35)
	g.box(22, 53, 18, 5, hole, 0.5, 0, 0.35)
end

-- Estiliza um objeto como icone de app (squircle + gradiente + brilho + glifo)
local function buildTile(container, a, size)
	container.ClipsDescendants = true
	local radius = math.floor(size * 0.235)
	corner(container, radius)

	local img = ICON_IMAGES[a.id]
	if img and img ~= "" then
		container.BackgroundTransparency = 1
		create("ImageLabel", {
			Size = UDim2.fromScale(1, 1),
			BackgroundTransparency = 1,
			Image = img,
			ScaleType = Enum.ScaleType.Fit,
		}, container)
		return
	end

	container.BackgroundColor3 = W
	create("UIGradient", {Rotation = 90, Color = ColorSequence.new(a.c1, a.c2)}, container)
	stroke(container, W, 0.86, 1)

	local gloss = create("Frame", {
		Size = UDim2.fromScale(1, 0.5),
		BackgroundColor3 = W,
		ZIndex = 1,
	}, container)
	corner(gloss, radius)
	create("UIGradient", {Rotation = 90, Transparency = NumberSequence.new(0.8, 1)}, gloss)

	local layer = create("Frame", {
		Size = UDim2.fromScale(1, 1),
		BackgroundTransparency = 1,
		ZIndex = 2,
	}, container)
	local fn = GLYPHS[a.id]
	if fn then
		fn(makeGlyph(layer, size), a.hole)
	end
end

-- Icone grande de clima
local function drawWeather(parent, size, kind)
	local g = makeGlyph(parent, size)
	local Y = rgb(255, 214, 10)
	local cloudColor = (kind == "Storm") and rgb(196, 202, 222) or W

	local function cloud(dy)
		g.dot(32, 56 + dy, 30, cloudColor)
		g.dot(52, 46 + dy, 40, cloudColor)
		g.box(18, 58 + dy, 66, 24, cloudColor, 0.5)
	end

	if kind == "Sun" then
		for i = 0, 7 do
			local a = math.rad(i * 45)
			local cx, cy = 50 + math.cos(a) * 38, 50 + math.sin(a) * 38
			g.box(cx - 3, cy - 8, 6, 16, Y, 0.5, i * 45 - 90)
		end
		g.dot(50, 50, 46, Y)
	elseif kind == "Cloud" then
		cloud(0)
	elseif kind == "Rain" then
		cloud(-10)
		for k = 0, 2 do
			g.box(28 + k * 18, 74, 5, 14, rgb(110, 190, 255), 0.5, 15)
		end
	elseif kind == "Storm" then
		cloud(-10)
		g.box(48, 62, 10, 16, Y, 0.15, 25)
		g.box(42, 74, 10, 16, Y, 0.15, 25)
	elseif kind == "Fog" then
		cloud(-16)
		g.box(20, 70, 60, 5, W, 0.5, 0, 0.2)
		g.box(30, 81, 50, 5, W, 0.5, 0, 0.35)
	end
end

local WEATHER_GRADIENTS = {
	Sun = {rgb(62,144,255), rgb(20,80,200)},
	Cloud = {rgb(104,118,142), rgb(56,66,90)},
	Rain = {rgb(72,92,128), rgb(30,40,70)},
	Storm = {rgb(64,58,100), rgb(24,22,48)},
	Fog = {rgb(128,138,152), rgb(80,88,102)},
}

--========================================================
-- GUI ROOT
--========================================================
local old = playerGui:FindFirstChild("NOVA_OS")
if old then old:Destroy() end

local gui = create("ScreenGui", {
	Name = "NOVA_OS",
	ResetOnSpawn = false,
	IgnoreGuiInset = true,
	ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
}, playerGui)

local phone = create("Frame", {
	Name = "Phone",
	Size = UDim2.fromOffset(PHONE_W, PHONE_H),
	AnchorPoint = Vector2.new(1, 0.5),
	Position = UDim2.new(1, PHONE_W + 40, 0.5, 0),
	BackgroundColor3 = rgb(4, 5, 9),
	Visible = false,
}, gui)
corner(phone, 38)
stroke(phone, rgb(92, 100, 122), 0.1, 1.5)

local phoneScale = create("UIScale", {Scale = 1}, phone)

local screen = create("Frame", {
	Size = UDim2.new(1, -12, 1, -12),
	Position = UDim2.fromOffset(6, 6),
	BackgroundColor3 = C.Bg,
	ClipsDescendants = true,
}, phone)
corner(screen, 32)
create("UIGradient", {
	Rotation = 135,
	Color = ColorSequence.new({
		ColorSequenceKeypoint.new(0, rgb(12, 16, 30)),
		ColorSequenceKeypoint.new(0.55, rgb(16, 20, 36)),
		ColorSequenceKeypoint.new(1, rgb(8, 10, 18)),
	}),
}, screen)

local function updateScale()
	local camera = workspace.CurrentCamera
	if not camera then return end
	local v = camera.ViewportSize
	phoneScale.Scale = math.clamp(math.min(v.X / 340, v.Y / 640, 1), 0.6, 1)
end
updateScale()
if workspace.CurrentCamera then
	workspace.CurrentCamera:GetPropertyChangedSignal("ViewportSize"):Connect(updateScale)
end

--========================================================
-- STATE
--========================================================
local currentPage = "Home"
local currentApp = nil
local phoneOpen = false
local pageToken = 0
local pageConns = {}
local permConns = {}
local live = {}

local renderHome, renderApp, notify

--========================================================
-- BRILHO (overlay)
--========================================================
local dimmer = create("Frame", {
	Name = "Dimmer",
	Size = UDim2.fromScale(1, 1),
	BackgroundColor3 = Color3.new(0, 0, 0),
	BackgroundTransparency = 1,
	ZIndex = 100,
	Active = false,
}, screen)

local function applyBrightness()
	dimmer.BackgroundTransparency = 0.4 + 0.6 * settings.brightness
end

--========================================================
-- STATUS BAR + DYNAMIC ISLAND
--========================================================
local status = create("Frame", {
	Size = UDim2.new(1, 0, 0, 34),
	BackgroundTransparency = 1,
	ZIndex = 10,
}, screen)

local statusClock = label(status, getClock(), 12, {
	Font = Enum.Font.GothamBold,
	Position = UDim2.fromOffset(26, 0),
	Size = UDim2.fromOffset(60, 34),
})

local statusRight = create("Frame", {
	AnchorPoint = Vector2.new(1, 0),
	Position = UDim2.new(1, -22, 0, 0),
	Size = UDim2.fromOffset(86, 34),
	BackgroundTransparency = 1,
}, status)
create("UIListLayout", {
	FillDirection = Enum.FillDirection.Horizontal,
	HorizontalAlignment = Enum.HorizontalAlignment.Right,
	VerticalAlignment = Enum.VerticalAlignment.Center,
	Padding = UDim.new(0, 6),
	SortOrder = Enum.SortOrder.LayoutOrder,
}, statusRight)

local signal = create("Frame", {Size = UDim2.fromOffset(15, 10), BackgroundTransparency = 1, LayoutOrder = 1}, statusRight)
for i = 1, 4 do
	create("Frame", {
		Size = UDim2.fromOffset(3, 2 + i * 2),
		Position = UDim2.fromOffset((i - 1) * 4, 10 - (2 + i * 2)),
		BackgroundColor3 = C.Text,
	}, signal)
end

label(statusRight, "5G", 9, {Font = Enum.Font.GothamBold, Size = UDim2.fromOffset(16, 12), LayoutOrder = 2})

local battery = create("Frame", {
	Size = UDim2.fromOffset(22, 10), BackgroundTransparency = 1, LayoutOrder = 3,
}, statusRight)
corner(battery, 3)
stroke(battery, C.Text, 0.55, 1)
local batteryFill = create("Frame", {
	Position = UDim2.fromOffset(2, 2), Size = UDim2.new(1, -4, 1, -4), BackgroundColor3 = C.Text,
}, battery)
corner(batteryFill, 2)
create("Frame", {
	Position = UDim2.fromOffset(23, 3), Size = UDim2.fromOffset(2, 4), BackgroundColor3 = C.Text, BackgroundTransparency = 0.5,
}, battery)

local island = create("Frame", {
	AnchorPoint = Vector2.new(0.5, 0),
	Position = UDim2.new(0.5, 0, 0, 8),
	Size = UDim2.fromOffset(82, 24),
	BackgroundColor3 = Color3.new(0, 0, 0),
	ZIndex = 70,
}, screen)
corner(island, 12)
local lens = create("Frame", {
	AnchorPoint = Vector2.new(1, 0.5), Position = UDim2.new(1, -9, 0.5, 0),
	Size = UDim2.fromOffset(8, 8), BackgroundColor3 = rgb(24, 28, 44), ZIndex = 71,
}, island)
corner(lens, 4)

--========================================================
-- CONTENT AREA
--========================================================
local content = create("Frame", {
	Position = UDim2.fromOffset(0, 36),
	Size = UDim2.new(1, 0, 1, -36),
	BackgroundTransparency = 1,
	ClipsDescendants = true,
	ZIndex = 1,
}, screen)

local function clearContent()
	for _, c in ipairs(pageConns) do c:Disconnect() end
	table.clear(pageConns)
	table.clear(live)
	for _, ch in ipairs(content:GetChildren()) do ch:Destroy() end
end

local function newPage(fade)
	clearContent()
	pageToken += 1
	local page = create("CanvasGroup", {
		Size = UDim2.fromScale(1, 1),
		BackgroundTransparency = 1,
		GroupTransparency = fade and 1 or 0,
		Position = fade and UDim2.fromOffset(0, 14) or UDim2.fromOffset(0, 0),
	}, content)
	if fade then
		animate(page, {GroupTransparency = 0, Position = UDim2.fromOffset(0, 0)}, 0.28)
	end
	return page, pageToken
end

local function loopWhile(token, interval, fn)
	task.spawn(function()
		while gui.Parent and pageToken == token do
			fn()
			task.wait(interval)
		end
	end)
end

--========================================================
-- DOCK (vidro) + HOME INDICATOR
--========================================================
local dock = create("Frame", {
	AnchorPoint = Vector2.new(0.5, 1),
	Position = UDim2.new(0.5, 0, 1, -24),
	Size = UDim2.new(1, -20, 0, 68),
	BackgroundColor3 = rgb(44, 50, 70),
	BackgroundTransparency = 0.3,
	ZIndex = 5,
}, screen)
corner(dock, 26)
stroke(dock, W, 0.88, 1)
create("UIListLayout", {
	FillDirection = Enum.FillDirection.Horizontal,
	HorizontalAlignment = Enum.HorizontalAlignment.Center,
	VerticalAlignment = Enum.VerticalAlignment.Center,
	Padding = UDim.new(0, 18),
}, dock)

for _, id in ipairs({"Profile", "Chat", "GPS", "People"}) do
	local a = APP_BY_ID[id]
	local b = create("TextButton", {
		Size = UDim2.fromOffset(48, 48), Text = "", AutoButtonColor = false,
	}, dock)
	buildTile(b, a, 48)
	pressable(b, 0.9)
	b.Activated:Connect(function() renderApp(id, true) end)
end

local indicatorHit = create("TextButton", {
	AnchorPoint = Vector2.new(0.5, 1),
	Position = UDim2.new(0.5, 0, 1, 0),
	Size = UDim2.fromOffset(140, 22),
	BackgroundTransparency = 1, Text = "", ZIndex = 12,
}, screen)
local indicator = create("Frame", {
	AnchorPoint = Vector2.new(0.5, 1),
	Position = UDim2.new(0.5, 0, 1, -7),
	Size = UDim2.fromOffset(96, 4),
	BackgroundColor3 = rgb(236, 238, 246),
}, indicatorHit)
corner(indicator, 2)
indicatorHit.Activated:Connect(function()
	if currentPage ~= "Home" then renderHome(true) end
end)

--========================================================
-- COMPONENTES DE UI
--========================================================
local function card(parent, height, order, isButton)
	local c = create(isButton and "TextButton" or "Frame", {
		Size = UDim2.new(1, 0, 0, height),
		BackgroundColor3 = C.Surface,
		LayoutOrder = order or 0,
	}, parent)
	if isButton then
		c.Text = ""
		c.AutoButtonColor = false
		pressable(c, 0.97)
	end
	corner(c, 16)
	stroke(c, C.Border, 0.72)
	return c
end

local function section(parent, str, order)
	return label(parent, string.upper(str), 10, {
		Font = Enum.Font.GothamBold,
		TextColor3 = C.Muted,
		Size = UDim2.new(1, 0, 0, 22),
		LayoutOrder = order,
	})
end

local function infoRow(parent, title, value, order)
	local c = card(parent, 44, order)
	label(c, title, 11, {Position = UDim2.fromOffset(14, 0), Size = UDim2.new(0.45, -14, 1, 0), TextColor3 = C.Muted})
	local v = label(c, tostring(value), 11, {
		Font = Enum.Font.GothamMedium,
		Position = UDim2.new(0.45, 0, 0, 0),
		Size = UDim2.new(0.55, -14, 1, 0),
		TextXAlignment = Enum.TextXAlignment.Right,
		TextTruncate = Enum.TextTruncate.AtEnd,
	})
	return c, v
end

local function textCard(parent, title, body, order)
	local c = create("Frame", {
		Size = UDim2.new(1, 0, 0, 0),
		AutomaticSize = Enum.AutomaticSize.Y,
		BackgroundColor3 = C.Surface,
		LayoutOrder = order,
	}, parent)
	corner(c, 16)
	stroke(c, C.Border, 0.72)
	create("UIPadding", {
		PaddingTop = UDim.new(0, 12), PaddingBottom = UDim.new(0, 12),
		PaddingLeft = UDim.new(0, 14), PaddingRight = UDim.new(0, 14),
	}, c)
	create("UIListLayout", {Padding = UDim.new(0, 4), SortOrder = Enum.SortOrder.LayoutOrder}, c)
	if title then
		label(c, title, 12, {Font = Enum.Font.GothamBold, Size = UDim2.new(1, 0, 0, 18), LayoutOrder = 1})
	end
	label(c, body, 11, {
		TextColor3 = C.Muted, TextWrapped = true, TextYAlignment = Enum.TextYAlignment.Top,
		Size = UDim2.new(1, 0, 0, 0), AutomaticSize = Enum.AutomaticSize.Y, LayoutOrder = 2,
	})
	return c
end

local function toggleSwitch(parent, initial, onChange)
	local track = create("Frame", {
		AnchorPoint = Vector2.new(1, 0.5), Position = UDim2.new(1, -14, 0.5, 0),
		Size = UDim2.fromOffset(42, 24),
		BackgroundColor3 = initial and C.Green or C.SurfaceHi,
	}, parent)
	corner(track, 12)
	local knob = create("Frame", {
		AnchorPoint = Vector2.new(0, 0.5),
		Position = initial and UDim2.new(1, -22, 0.5, 0) or UDim2.new(0, 2, 0.5, 0),
		Size = UDim2.fromOffset(20, 20), BackgroundColor3 = W,
	}, track)
	corner(knob, 10)
	local state = initial
	return function(v)
		state = v
		animate(track, {BackgroundColor3 = state and C.Green or C.SurfaceHi}, 0.18)
		animate(knob, {Position = state and UDim2.new(1, -22, 0.5, 0) or UDim2.new(0, 2, 0.5, 0)}, 0.18)
	end
end

local function toggleRow(parent, title, sub, initial, onChange, order)
	local row = card(parent, 54, order, true)
	label(row, title, 12, {Font = Enum.Font.GothamMedium, Position = UDim2.fromOffset(14, 8), Size = UDim2.new(1, -80, 0, 18)})
	label(row, sub, 9, {TextColor3 = C.Muted, Position = UDim2.fromOffset(14, 27), Size = UDim2.new(1, -80, 0, 16)})
	local setVisual = toggleSwitch(row, initial)
	local state = initial
	row.Activated:Connect(function()
		state = not state
		setVisual(state)
		onChange(state)
	end)
	return row
end

local function makeSlider(parent, initial, onChange, conns)
	local hit = create("TextButton", {
		Size = UDim2.new(1, 0, 0, 30), BackgroundTransparency = 1, Text = "", AutoButtonColor = false,
	}, parent)
	local track = create("Frame", {
		AnchorPoint = Vector2.new(0, 0.5), Position = UDim2.new(0, 0, 0.5, 0),
		Size = UDim2.new(1, 0, 0, 6), BackgroundColor3 = C.SurfaceHi,
	}, hit)
	corner(track, 3)
	local fill = create("Frame", {Size = UDim2.fromScale(initial, 1), BackgroundColor3 = C.Accent}, track)
	corner(fill, 3)
	local knob = create("Frame", {
		AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(initial, 0.5),
		Size = UDim2.fromOffset(18, 18), BackgroundColor3 = W,
	}, track)
	corner(knob, 9)

	local dragging = false
	local function set(v, fire)
		v = math.clamp(v, 0, 1)
		fill.Size = UDim2.fromScale(v, 1)
		knob.Position = UDim2.fromScale(v, 0.5)
		if fire and onChange then onChange(v) end
	end
	local function fromX(x)
		set((x - track.AbsolutePosition.X) / math.max(track.AbsoluteSize.X, 1), true)
	end

	hit.InputBegan:Connect(function(i)
		if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
			dragging = true
			fromX(i.Position.X)
		end
	end)
	table.insert(conns, UserInputService.InputChanged:Connect(function(i)
		if dragging and (i.UserInputType == Enum.UserInputType.MouseMovement or i.UserInputType == Enum.UserInputType.Touch) then
			fromX(i.Position.X)
		end
	end))
	table.insert(conns, UserInputService.InputEnded:Connect(function(i)
		if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
			dragging = false
		end
	end))

	return {set = function(v) set(v, false) end}
end

local function header(page, title, subtitle)
	local h = create("Frame", {Size = UDim2.new(1, 0, 0, 52), BackgroundTransparency = 1}, page)
	local back = create("TextButton", {
		Size = UDim2.fromOffset(36, 36), Position = UDim2.fromOffset(14, 8),
		BackgroundColor3 = C.SurfaceHi, Text = "", AutoButtonColor = false,
	}, h)
	corner(back, 18)
	pressable(back, 0.88)
	create("Frame", {Size = UDim2.fromOffset(11, 2.6), Position = UDim2.fromOffset(12.9, 12.8), Rotation = -45, BackgroundColor3 = C.Text}, back)
	create("Frame", {Size = UDim2.fromOffset(11, 2.6), Position = UDim2.fromOffset(12.9, 20.6), Rotation = 45, BackgroundColor3 = C.Text}, back)
	back.Activated:Connect(function() renderHome(true) end)

	label(h, title, 17, {Font = Enum.Font.GothamBold, Position = UDim2.fromOffset(60, 8), Size = UDim2.new(1, -74, 0, 22)})
	label(h, subtitle or "", 10, {TextColor3 = C.Muted, Position = UDim2.fromOffset(60, 28), Size = UDim2.new(1, -74, 0, 16)})
end

local function newScroll(page)
	local s = create("ScrollingFrame", {
		Position = UDim2.fromOffset(0, 56),
		Size = UDim2.new(1, 0, 1, -56),
		BackgroundTransparency = 1,
		ScrollBarThickness = 2,
		ScrollBarImageColor3 = C.Border,
		CanvasSize = UDim2.new(),
		AutomaticCanvasSize = Enum.AutomaticSize.Y,
		ScrollingDirection = Enum.ScrollingDirection.Y,
	}, page)
	create("UIListLayout", {Padding = UDim.new(0, 8), SortOrder = Enum.SortOrder.LayoutOrder}, s)
	create("UIPadding", {
		PaddingLeft = UDim.new(0, 14), PaddingRight = UDim.new(0, 14),
		PaddingTop = UDim.new(0, 2), PaddingBottom = UDim.new(0, 34),
	}, s)
	return s
end

local function fetchThumb(userId, target)
	task.spawn(function()
		local ok, img = pcall(function()
			return Players:GetUserThumbnailAsync(userId, Enum.ThumbnailType.HeadShot, Enum.ThumbnailSize.Size150x150)
		end)
		if ok and target.Parent then target.Image = img end
	end)
end

--========================================================
-- NOTIFICACOES (toast)
--========================================================
local toast = create("Frame", {
	Size = UDim2.new(1, -24, 0, 54),
	Position = UDim2.new(0, 12, 0, -80),
	BackgroundColor3 = rgb(36, 42, 62),
	ZIndex = 45,
}, screen)
corner(toast, 18)
stroke(toast, W, 0.86, 1)
local toastIcon = create("Frame", {Size = UDim2.fromOffset(34, 34), Position = UDim2.fromOffset(10, 10), ZIndex = 46}, toast)
local toastTitle = label(toast, "", 12, {Font = Enum.Font.GothamBold, Position = UDim2.fromOffset(54, 9), Size = UDim2.new(1, -64, 0, 18), ZIndex = 46})
local toastMsg = label(toast, "", 10, {TextColor3 = C.Muted, Position = UDim2.fromOffset(54, 27), Size = UDim2.new(1, -64, 0, 16), ZIndex = 46, TextTruncate = Enum.TextTruncate.AtEnd})
local toastToken = 0

notify = function(title, msg, appId)
	if settings.dnd then return end
	toastToken += 1
	local mine = toastToken
	for _, c in ipairs(toastIcon:GetChildren()) do c:Destroy() end
	toastIcon.BackgroundTransparency = 0
	buildTile(toastIcon, APP_BY_ID[appId or "Chat"] or APP_BY_ID.Chat, 34)
	toastTitle.Text = tostring(title or "NOVA")
	toastMsg.Text = tostring(msg or "")
	animate(toast, {Position = UDim2.new(0, 12, 0, 38)}, 0.38, Enum.EasingStyle.Back)
	task.delay(3.6, function()
		if toastToken == mine then
			animate(toast, {Position = UDim2.new(0, 12, 0, -80)}, 0.3)
		end
	end)
end

local notifyEvent = create("BindableEvent", {Name = "Notify"}, gui)
notifyEvent.Event:Connect(notify)
local remoteNotify = ReplicatedStorage:FindFirstChild("NOVA_Notify")
if remoteNotify and remoteNotify:IsA("RemoteEvent") then
	remoteNotify.OnClientEvent:Connect(notify)
end

--========================================================
-- CENTRAL DE CONTROLE (estilo Android - toque na barra de status)
--========================================================
local panel = create("Frame", {
	Name = "QuickPanel",
	Size = UDim2.new(1, 0, 0, 350),
	Position = UDim2.new(0, 0, 0, -370),
	BackgroundColor3 = rgb(16, 19, 30),
	BackgroundTransparency = 0.03,
	ZIndex = 40,
}, screen)
corner(panel, 28)
stroke(panel, W, 0.9, 1)
create("UIPadding", {
	PaddingTop = UDim.new(0, 52), PaddingLeft = UDim.new(0, 12), PaddingRight = UDim.new(0, 12),
}, panel)
create("UIListLayout", {Padding = UDim.new(0, 8), SortOrder = Enum.SortOrder.LayoutOrder}, panel)

local panelHead = create("Frame", {Size = UDim2.new(1, 0, 0, 44), BackgroundTransparency = 1, LayoutOrder = 1, ZIndex = 41}, panel)
local panelTime = label(panelHead, getClock(), 26, {Font = Enum.Font.GothamBold, Size = UDim2.new(0.5, 0, 1, 0), ZIndex = 41})
local panelDate = label(panelHead, dateString(), 10, {
	TextColor3 = C.Muted, TextXAlignment = Enum.TextXAlignment.Right,
	AnchorPoint = Vector2.new(1, 0), Position = UDim2.new(1, 0, 0, 0), Size = UDim2.new(0.55, 0, 1, 0), ZIndex = 41,
})

local tileGrid = create("Frame", {Size = UDim2.new(1, 0, 0, 120), BackgroundTransparency = 1, LayoutOrder = 2, ZIndex = 41}, panel)
create("UIGridLayout", {
	CellSize = UDim2.fromOffset(80, 56), CellPadding = UDim2.fromOffset(7, 8),
	SortOrder = Enum.SortOrder.LayoutOrder,
}, tileGrid)

local QUICK = {
	{"wifi", "Wi-Fi"}, {"bluetooth", "Bluetooth"}, {"dnd", "Nao perturbe"},
	{"flash", "Lanterna"}, {"airplane", "Modo aviao"}, {"rotation", "Rotacao"},
}
for i, q in ipairs(QUICK) do
	local key, name = q[1], q[2]
	local t = create("TextButton", {
		BackgroundColor3 = settings[key] and C.Accent or C.SurfaceHi,
		Text = "", AutoButtonColor = false, LayoutOrder = i, ZIndex = 42,
	}, tileGrid)
	corner(t, 16)
	pressable(t, 0.94)
	label(t, name, 11, {Font = Enum.Font.GothamBold, Position = UDim2.fromOffset(10, 10), Size = UDim2.new(1, -16, 0, 18), ZIndex = 43, TextTruncate = Enum.TextTruncate.AtEnd})
	local sub = label(t, settings[key] and "Ativado" or "Desativado", 9, {Position = UDim2.fromOffset(10, 29), Size = UDim2.new(1, -16, 0, 16), ZIndex = 43, TextColor3 = W, TextTransparency = 0.25})
	t.Activated:Connect(function()
		settings[key] = not settings[key]
		sub.Text = settings[key] and "Ativado" or "Desativado"
		animate(t, {BackgroundColor3 = settings[key] and C.Accent or C.SurfaceHi}, 0.18)
	end)
end

local brightCard = create("Frame", {Size = UDim2.new(1, 0, 0, 52), BackgroundColor3 = C.SurfaceHi, LayoutOrder = 3, ZIndex = 41}, panel)
corner(brightCard, 16)
label(brightCard, "Brilho", 10, {TextColor3 = C.Muted, Position = UDim2.fromOffset(14, 5), Size = UDim2.new(1, -28, 0, 16), ZIndex = 42})
local brightHolder = create("Frame", {Position = UDim2.fromOffset(14, 20), Size = UDim2.new(1, -28, 0, 30), BackgroundTransparency = 1, ZIndex = 42}, brightCard)
local panelSlider = makeSlider(brightHolder, settings.brightness, function(v)
	settings.brightness = v
	applyBrightness()
end, permConns)

local panelHandle = create("TextButton", {Size = UDim2.new(1, 0, 0, 22), BackgroundTransparency = 1, Text = "", LayoutOrder = 4, ZIndex = 42}, panel)
local handleBar = create("Frame", {
	AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.5),
	Size = UDim2.fromOffset(46, 4), BackgroundColor3 = C.Muted, ZIndex = 43,
}, panelHandle)
corner(handleBar, 2)

local panelOpen = false
local function setPanel(open)
	panelOpen = open
	if open then
		panelTime.Text = getClock()
		panelDate.Text = dateString()
		panelSlider.set(settings.brightness)
	end
	animate(panel, {Position = UDim2.new(0, 0, 0, open and -16 or -370)}, 0.32)
end
panelHandle.Activated:Connect(function() setPanel(false) end)

local statusHit = create("TextButton", {Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, Text = "", ZIndex = 11}, status)
statusHit.Activated:Connect(function() setPanel(not panelOpen) end)

--========================================================
-- LIVE UPDATES
--========================================================
local function refreshLive()
	local t = getClock()
	statusClock.Text = t
	if panelOpen then panelTime.Text = t end
	if live.time and live.time.Parent then live.time.Text = t end
	if live.phase and live.phase.Parent then live.phase.Text = getPhase() end
end

--========================================================
-- HOME
--========================================================
renderHome = function(fade)
	currentPage = "Home"
	currentApp = nil
	dock.Visible = true
	local page = newPage(fade ~= false)

	local scroll = create("ScrollingFrame", {
		Size = UDim2.fromScale(1, 1),
		BackgroundTransparency = 1,
		ScrollBarThickness = 2,
		ScrollBarImageColor3 = C.Border,
		CanvasSize = UDim2.new(),
		AutomaticCanvasSize = Enum.AutomaticSize.Y,
	}, page)
	create("UIListLayout", {Padding = UDim.new(0, 10), SortOrder = Enum.SortOrder.LayoutOrder}, scroll)
	create("UIPadding", {
		PaddingLeft = UDim.new(0, 12), PaddingRight = UDim.new(0, 12),
		PaddingTop = UDim.new(0, 4), PaddingBottom = UDim.new(0, 110),
	}, scroll)

	-- Widget "agora" (hora + clima)
	local glance = create("TextButton", {
		Size = UDim2.new(1, 0, 0, 112), Text = "", AutoButtonColor = false,
		BackgroundColor3 = W, LayoutOrder = 1,
	}, scroll)
	corner(glance, 24)
	pressable(glance, 0.97)
	create("UIGradient", {Rotation = 30, Color = ColorSequence.new(rgb(52, 70, 140), rgb(24, 30, 66))}, glance)
	stroke(glance, W, 0.88, 1)
	live.time = label(glance, getClock(), 40, {Font = Enum.Font.GothamBold, Position = UDim2.fromOffset(16, 8), Size = UDim2.fromOffset(150, 50)})
	label(glance, dateString(), 10, {TextColor3 = rgb(206, 214, 238), Position = UDim2.fromOffset(17, 60), Size = UDim2.new(1, -34, 0, 16)})
	live.phase = label(glance, getPhase(), 10, {Font = Enum.Font.GothamBold, TextColor3 = rgb(150, 190, 255), Position = UDim2.fromOffset(17, 82), Size = UDim2.new(0.5, 0, 0, 16)})
	local wHolder = create("Frame", {
		AnchorPoint = Vector2.new(1, 0), Position = UDim2.new(1, -14, 0, 10),
		Size = UDim2.fromOffset(52, 52), BackgroundTransparency = 1,
	}, glance)
	drawWeather(wHolder, 52, weatherKind())
	label(glance, weatherName(), 11, {
		Font = Enum.Font.GothamBold, TextXAlignment = Enum.TextXAlignment.Right,
		AnchorPoint = Vector2.new(1, 0), Position = UDim2.new(1, -14, 0, 70), Size = UDim2.fromOffset(110, 16),
	})
	glance.Activated:Connect(function() renderApp("Weather", true) end)

	-- Busca (pilula estilo Android)
	local searchBar = create("Frame", {Size = UDim2.new(1, 0, 0, 38), BackgroundColor3 = C.Surface, LayoutOrder = 2}, scroll)
	corner(searchBar, 19)
	stroke(searchBar, C.Border, 0.7)
	local lensRing = create("Frame", {Position = UDim2.fromOffset(15, 11), Size = UDim2.fromOffset(12, 12), BackgroundTransparency = 1}, searchBar)
	corner(lensRing, 6)
	stroke(lensRing, C.Muted, 0, 2)
	create("Frame", {Position = UDim2.fromOffset(25, 24), Size = UDim2.fromOffset(6, 2), Rotation = 45, BackgroundColor3 = C.Muted}, searchBar)
	local box = create("TextBox", {
		Position = UDim2.fromOffset(40, 0), Size = UDim2.new(1, -52, 1, 0),
		BackgroundTransparency = 1, Text = "", PlaceholderText = "Pesquisar aplicativos",
		PlaceholderColor3 = C.Muted, TextColor3 = C.Text, Font = Enum.Font.Gotham,
		TextSize = 12, TextXAlignment = Enum.TextXAlignment.Left, ClearTextOnFocus = false,
	}, searchBar)

	-- Grade de apps
	local grid = create("Frame", {
		Size = UDim2.new(1, 0, 0, 0), AutomaticSize = Enum.AutomaticSize.Y,
		BackgroundTransparency = 1, LayoutOrder = 3,
	}, scroll)
	create("UIGridLayout", {
		CellSize = UDim2.fromOffset(56, 76),
		CellPadding = UDim2.fromOffset(10, 8),
		FillDirectionMaxCells = 4,
		HorizontalAlignment = Enum.HorizontalAlignment.Center,
		SortOrder = Enum.SortOrder.LayoutOrder,
	}, grid)

	local cells = {}
	for i, a in ipairs(APPS) do
		local cell = create("Frame", {BackgroundTransparency = 1, LayoutOrder = i, Name = a.id}, grid)
		local tile = create("TextButton", {
			AnchorPoint = Vector2.new(0.5, 0), Position = UDim2.new(0.5, 0, 0, 0),
			Size = UDim2.fromOffset(54, 54), Text = "", AutoButtonColor = false,
		}, cell)
		buildTile(tile, a, 54)
		pressable(tile, 0.9)
		tile.Activated:Connect(function() renderApp(a.id, true) end)
		label(cell, a.name, 10, {
			Font = Enum.Font.GothamMedium, TextXAlignment = Enum.TextXAlignment.Center,
			Position = UDim2.fromOffset(-4, 58), Size = UDim2.new(1, 8, 0, 14),
			TextTruncate = Enum.TextTruncate.AtEnd,
		})
		cells[#cells + 1] = {cell = cell, name = string.lower(a.name)}
	end

	box:GetPropertyChangedSignal("Text"):Connect(function()
		local q = string.lower(box.Text)
		for _, c in ipairs(cells) do
			c.cell.Visible = q == "" or string.find(c.name, q, 1, true) ~= nil
		end
	end)
end

--========================================================
-- RENDERERS DOS APPS
--========================================================
local RENDERERS = {}

RENDERERS.Profile = function(scroll, nx, token)
	local hero = card(scroll, 158, nx())
	local avatar = create("ImageLabel", {
		Size = UDim2.fromOffset(76, 76), Position = UDim2.new(0.5, -38, 0, 16),
		BackgroundColor3 = C.SurfaceHi, Image = "",
	}, hero)
	corner(avatar, 38)
	stroke(avatar, C.Accent, 0, 2)
	fetchThumb(player.UserId, avatar)
	label(hero, player.DisplayName, 17, {Font = Enum.Font.GothamBold, TextXAlignment = Enum.TextXAlignment.Center, Position = UDim2.fromOffset(0, 98), Size = UDim2.new(1, 0, 0, 24)})
	label(hero, "@" .. player.Name, 11, {TextColor3 = C.Muted, TextXAlignment = Enum.TextXAlignment.Center, Position = UDim2.fromOffset(0, 122), Size = UDim2.new(1, 0, 0, 18)})

	section(scroll, "Informacoes", nx())
	infoRow(scroll, "ID", player.UserId, nx())
	infoRow(scroll, "Conta", player.AccountAge .. " dias", nx())
	local money = player:GetAttribute("Money")
	infoRow(scroll, "Saldo", typeof(money) == "number" and ("R$ " .. formatNumber(money)) or "—", nx())
	infoRow(scroll, "Profissao", player:GetAttribute("Job") or "—", nx())
end

RENDERERS.GPS = function(scroll, nx, token)
	local RANGE = 200 -- studs do centro ate a borda
	local mapCard = card(scroll, 250, nx())
	mapCard.ClipsDescendants = true
	mapCard.BackgroundColor3 = rgb(12, 20, 28)

	for k = 1, 3 do
		local d = k / 3
		local ring = create("Frame", {
			AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.5),
			Size = UDim2.fromScale(d, d), BackgroundTransparency = 1,
		}, mapCard)
		corner(ring, 200)
		stroke(ring, C.Accent, 0.8, 1)
	end
	create("Frame", {Size = UDim2.new(1, 0, 0, 1), Position = UDim2.fromScale(0, 0.5), BackgroundColor3 = C.Accent, BackgroundTransparency = 0.88}, mapCard)
	create("Frame", {Size = UDim2.new(0, 1, 1, 0), Position = UDim2.fromScale(0.5, 0), BackgroundColor3 = C.Accent, BackgroundTransparency = 0.88}, mapCard)
	label(mapCard, "N", 11, {Font = Enum.Font.GothamBold, TextColor3 = C.Red, TextXAlignment = Enum.TextXAlignment.Center, Position = UDim2.new(0.5, -10, 0, 6), Size = UDim2.fromOffset(20, 16)})
	label(mapCard, "raio " .. RANGE .. " studs", 9, {TextColor3 = C.Muted, Position = UDim2.new(0, 10, 1, -20), Size = UDim2.fromOffset(120, 14)})

	local pulse = create("Frame", {
		AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.5),
		Size = UDim2.fromOffset(14, 14), BackgroundColor3 = C.Accent, BackgroundTransparency = 0.5,
	}, mapCard)
	corner(pulse, 20)
	task.spawn(function()
		while gui.Parent and pageToken == token do
			pulse.Size = UDim2.fromOffset(14, 14)
			pulse.BackgroundTransparency = 0.4
			animate(pulse, {Size = UDim2.fromOffset(44, 44), BackgroundTransparency = 1}, 1.2, Enum.EasingStyle.Sine)
			task.wait(1.3)
		end
	end)
	local me = create("Frame", {
		AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.5),
		Size = UDim2.fromOffset(12, 12), BackgroundColor3 = C.Accent, ZIndex = 3,
	}, mapCard)
	corner(me, 6)
	stroke(me, W, 0, 2)

	section(scroll, "Coordenadas", nx())
	local _, posValue = infoRow(scroll, "Posicao (X, Y, Z)", "...", nx())

	local dots = {}
	local function refresh()
		local root = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
		if not root then
			posValue.Text = "Personagem indisponivel"
			return
		end
		local p = root.Position
		posValue.Text = string.format("%d, %d, %d", p.X, p.Y, p.Z)

		local seen = {}
		for _, other in ipairs(Players:GetPlayers()) do
			if other ~= player then
				local r = other.Character and other.Character:FindFirstChild("HumanoidRootPart")
				if r then
					seen[other] = true
					local dot = dots[other]
					if not dot then
						dot = create("Frame", {
							AnchorPoint = Vector2.new(0.5, 0.5), Size = UDim2.fromOffset(9, 9),
							BackgroundColor3 = C.Green, ZIndex = 2,
						}, mapCard)
						corner(dot, 5)
						dots[other] = dot
					end
					local dx = math.clamp((r.Position.X - p.X) / RANGE, -0.96, 0.96)
					local dz = math.clamp((r.Position.Z - p.Z) / RANGE, -0.96, 0.96)
					dot.Position = UDim2.fromScale(0.5 + dx * 0.5, 0.5 + dz * 0.5)
				end
			end
		end
		for other, dot in pairs(dots) do
			if not seen[other] then dot:Destroy(); dots[other] = nil end
		end
	end
	loopWhile(token, 0.3, refresh)
end

RENDERERS.People = function(scroll, nx, token)
	local searchCard = card(scroll, 40, nx())
	local box = create("TextBox", {
		Position = UDim2.fromOffset(14, 0), Size = UDim2.new(1, -28, 1, 0), BackgroundTransparency = 1,
		Text = "", PlaceholderText = "Buscar jogador", PlaceholderColor3 = C.Muted,
		TextColor3 = C.Text, Font = Enum.Font.Gotham, TextSize = 12,
		TextXAlignment = Enum.TextXAlignment.Left, ClearTextOnFocus = false,
	}, searchCard)
	section(scroll, "Jogadores na cidade", nx())

	local rows = {}
	local function refresh()
		local list = Players:GetPlayers()
		table.sort(list, function(a, b) return a.DisplayName:lower() < b.DisplayName:lower() end)
		local myRoot = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
		local q = box.Text:lower()
		local seen = {}

		for i, p in ipairs(list) do
			seen[p] = true
			local row = rows[p]
			if not row then
				local c = card(scroll, 54, 0)
				local av = create("ImageLabel", {Size = UDim2.fromOffset(36, 36), Position = UDim2.fromOffset(10, 9), BackgroundColor3 = C.SurfaceHi, Image = ""}, c)
				corner(av, 18)
				fetchThumb(p.UserId, av)
				local nm = label(c, p.DisplayName, 12, {Font = Enum.Font.GothamMedium, Position = UDim2.fromOffset(54, 9), Size = UDim2.new(1, -64, 0, 18), TextTruncate = Enum.TextTruncate.AtEnd})
				local sb = label(c, "", 10, {TextColor3 = C.Muted, Position = UDim2.fromOffset(54, 27), Size = UDim2.new(1, -64, 0, 16)})
				row = {frame = c, sub = sb, name = nm}
				rows[p] = row
			end
			row.frame.LayoutOrder = 10 + i
			local text = "@" .. p.Name
			if p == player then
				text ..= "  •  Voce"
			elseif myRoot then
				local r = p.Character and p.Character:FindFirstChild("HumanoidRootPart")
				if r then
					text ..= string.format("  •  a %d m", (r.Position - myRoot.Position).Magnitude * 0.28)
				end
			end
			row.sub.Text = text
			row.frame.Visible = q == "" or p.DisplayName:lower():find(q, 1, true) ~= nil or p.Name:lower():find(q, 1, true) ~= nil
		end
		for p, row in pairs(rows) do
			if not seen[p] then row.frame:Destroy(); rows[p] = nil end
		end
	end
	table.insert(pageConns, box:GetPropertyChangedSignal("Text"):Connect(refresh))
	loopWhile(token, 2, refresh)
end

RENDERERS.Inventory = function(scroll, nx, token)
	section(scroll, "Itens", nx())
	local holder = create("Frame", {Size = UDim2.new(1, 0, 0, 0), AutomaticSize = Enum.AutomaticSize.Y, BackgroundTransparency = 1, LayoutOrder = nx()}, scroll)
	create("UIListLayout", {Padding = UDim.new(0, 8), SortOrder = Enum.SortOrder.LayoutOrder}, holder)

	local lastSig = nil
	local function refresh()
		local tools = {}
		local char = player.Character
		if char then
			for _, t in ipairs(char:GetChildren()) do
				if t:IsA("Tool") then tools[#tools + 1] = {tool = t, equipped = true} end
			end
		end
		local backpack = player:FindFirstChildOfClass("Backpack")
		if backpack then
			for _, t in ipairs(backpack:GetChildren()) do
				if t:IsA("Tool") then tools[#tools + 1] = {tool = t, equipped = false} end
			end
		end

		local sig = {}
		for _, t in ipairs(tools) do sig[#sig + 1] = t.tool.Name .. (t.equipped and "*" or "") end
		local sigText = table.concat(sig, "|")
		if sigText == lastSig then return end
		lastSig = sigText

		for _, c in ipairs(holder:GetChildren()) do
			if not c:IsA("UIListLayout") then c:Destroy() end
		end

		if #tools == 0 then
			local c = card(holder, 70, 1)
			label(c, "Nenhum item no momento.", 11, {TextColor3 = C.Muted, TextXAlignment = Enum.TextXAlignment.Center, Size = UDim2.fromScale(1, 1)})
			return
		end

		for i, t in ipairs(tools) do
			local c = card(holder, 50, i, true)
			label(c, t.tool.Name, 12, {Font = Enum.Font.GothamMedium, Position = UDim2.fromOffset(14, 0), Size = UDim2.new(1, -110, 1, 0), TextTruncate = Enum.TextTruncate.AtEnd})
			local chip = create("Frame", {
				AnchorPoint = Vector2.new(1, 0.5), Position = UDim2.new(1, -12, 0.5, 0), Size = UDim2.fromOffset(78, 24),
				BackgroundColor3 = t.equipped and C.Accent or C.SurfaceHi,
			}, c)
			corner(chip, 12)
			label(chip, t.equipped and "Equipado" or "Mochila", 9, {Font = Enum.Font.GothamBold, TextXAlignment = Enum.TextXAlignment.Center, Size = UDim2.fromScale(1, 1)})
			c.Activated:Connect(function()
				local hum = player.Character and player.Character:FindFirstChildOfClass("Humanoid")
				if not hum then return end
				if t.equipped then hum:UnequipTools() else hum:EquipTool(t.tool) end
			end)
		end
	end
	loopWhile(token, 0.6, refresh)
end

RENDERERS.Weather = function(scroll, nx)
	local kind = weatherKind()
	local grad = WEATHER_GRADIENTS[kind]
	local hero = card(scroll, 170, nx())
	hero.BackgroundColor3 = W
	create("UIGradient", {Rotation = 60, Color = ColorSequence.new(grad[1], grad[2])}, hero)
	local gh = create("Frame", {Position = UDim2.new(1, -112, 0, 14), Size = UDim2.fromOffset(98, 98), BackgroundTransparency = 1}, hero)
	drawWeather(gh, 98, kind)
	local temp = worldAttribute("Temperature", nil)
	if typeof(temp) == "number" then
		label(hero, string.format("%d°", temp), 44, {Font = Enum.Font.GothamBold, Position = UDim2.fromOffset(16, 12), Size = UDim2.fromOffset(120, 54)})
	end
	label(hero, weatherName(), 20, {Font = Enum.Font.GothamBold, Position = UDim2.fromOffset(16, 100), Size = UDim2.new(1, -32, 0, 26)})
	label(hero, getPhase() .. (getPhase() ~= "" and "  •  " or "") .. getClock(), 11, {TextColor3 = rgb(226, 232, 250), Position = UDim2.fromOffset(16, 126), Size = UDim2.new(1, -32, 0, 18)})

	section(scroll, "Detalhes", nx())
	infoRow(scroll, "Condicao", weatherName(), nx())
	infoRow(scroll, "Codigo", getWeather(), nx())
	infoRow(scroll, "Periodo", getPhase() ~= "" and getPhase() or "—", nx())
	infoRow(scroll, "Horario", getClock(), nx())
end

RENDERERS.Calendar = function(scroll, nx)
	local now = getDate()
	local first = os.date("*t", os.time({year = now.year, month = now.month, day = 1, hour = 12}))
	local daysInMonth = os.date("*t", os.time({year = now.year, month = now.month + 1, day = 0, hour = 12})).day

	local c = card(scroll, 292, nx())
	label(c, MONTHS[now.month] .. " " .. now.year, 15, {Font = Enum.Font.GothamBold, Position = UDim2.fromOffset(14, 10), Size = UDim2.new(1, -28, 0, 26)})

	local wk = create("Frame", {Position = UDim2.fromOffset(10, 44), Size = UDim2.new(1, -20, 0, 20), BackgroundTransparency = 1}, c)
	create("UIGridLayout", {CellSize = UDim2.new(1 / 7, 0, 1, 0), CellPadding = UDim2.new(), SortOrder = Enum.SortOrder.LayoutOrder}, wk)
	for i, d in ipairs({"D", "S", "T", "Q", "Q", "S", "S"}) do
		label(wk, d, 10, {Font = Enum.Font.GothamBold, TextColor3 = (i == 1) and C.Red or C.Muted, TextXAlignment = Enum.TextXAlignment.Center, LayoutOrder = i})
	end

	local grid = create("Frame", {Position = UDim2.fromOffset(10, 68), Size = UDim2.new(1, -20, 0, 210), BackgroundTransparency = 1}, c)
	create("UIGridLayout", {CellSize = UDim2.new(1 / 7, 0, 0, 34), CellPadding = UDim2.new(), SortOrder = Enum.SortOrder.LayoutOrder}, grid)
	for i = 1, first.wday - 1 do
		create("Frame", {BackgroundTransparency = 1, LayoutOrder = i}, grid)
	end
	for day = 1, daysInMonth do
		local cell = create("Frame", {BackgroundTransparency = 1, LayoutOrder = first.wday - 1 + day}, grid)
		if day == now.day then
			local bubble = create("Frame", {
				AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.5),
				Size = UDim2.fromOffset(28, 28), BackgroundColor3 = C.Accent,
			}, cell)
			corner(bubble, 14)
		end
		label(cell, tostring(day), 11, {
			Font = day == now.day and Enum.Font.GothamBold or Enum.Font.Gotham,
			TextXAlignment = Enum.TextXAlignment.Center, Size = UDim2.fromScale(1, 1), ZIndex = 2,
		})
	end

	infoRow(scroll, "Hoje", dateString(), nx())
	infoRow(scroll, "Horario", getClock(), nx())
end

RENDERERS.Settings = function(scroll, nx, token)
	section(scroll, "Aparencia", nx())
	local accentCard = card(scroll, 60, nx())
	label(accentCard, "Cor de destaque", 10, {TextColor3 = C.Muted, Position = UDim2.fromOffset(14, 6), Size = UDim2.new(1, -28, 0, 16)})
	local chips = create("Frame", {Position = UDim2.fromOffset(14, 26), Size = UDim2.new(1, -28, 0, 28), BackgroundTransparency = 1}, accentCard)
	create("UIListLayout", {FillDirection = Enum.FillDirection.Horizontal, Padding = UDim.new(0, 10), VerticalAlignment = Enum.VerticalAlignment.Center}, chips)
	for _, col in ipairs(ACCENTS) do
		local chip = create("TextButton", {Size = UDim2.fromOffset(26, 26), BackgroundColor3 = col, Text = "", AutoButtonColor = false}, chips)
		corner(chip, 13)
		pressable(chip, 0.85)
		if col == C.Accent then stroke(chip, W, 0, 2) end
		chip.Activated:Connect(function()
			C.Accent = col
			renderApp("Settings", false)
		end)
	end

	section(scroll, "Tela", nx())
	local br = card(scroll, 58, nx())
	label(br, "Brilho", 10, {TextColor3 = C.Muted, Position = UDim2.fromOffset(14, 6), Size = UDim2.new(1, -28, 0, 16)})
	local holder = create("Frame", {Position = UDim2.fromOffset(14, 24), Size = UDim2.new(1, -28, 0, 30), BackgroundTransparency = 1}, br)
	makeSlider(holder, settings.brightness, function(v)
		settings.brightness = v
		applyBrightness()
	end, pageConns)

	section(scroll, "Notificacoes", nx())
	toggleRow(scroll, "Nao perturbe", "Silencia os avisos do celular", settings.dnd, function(v) settings.dnd = v end, nx())
	local test = card(scroll, 46, nx(), true)
	label(test, "Enviar notificacao de teste", 12, {Font = Enum.Font.GothamMedium, TextColor3 = C.Accent, TextXAlignment = Enum.TextXAlignment.Center, Size = UDim2.fromScale(1, 1)})
	test.Activated:Connect(function()
		local was = settings.dnd
		settings.dnd = false
		notify("NOVA OS", "Tudo funcionando por aqui!", "Chat")
		settings.dnd = was
	end)

	section(scroll, "Sistema", nx())
	infoRow(scroll, "Dispositivo", "NOVA", nx())
	infoRow(scroll, "Versao", "3.2.0", nx())
end

RENDERERS.Help = function(scroll, nx)
	textCard(scroll, "Bem-vindo a NOVA OS", "Pressione M (ou o botao N na lateral) para abrir e fechar o celular.", nx())
	textCard(scroll, "Central de controle", "Toque na barra de status (hora e sinal) para abrir os atalhos e o brilho. Toque na barrinha para fechar.", nx())
	textCard(scroll, "Navegacao", "Use a seta no topo ou a barra inferior para voltar a tela inicial. Pesquise apps no campo da tela inicial.", nx())
	textCard(scroll, "Servicos da cidade", "Banco, imoveis, mensagens e outros apps serao liberados conforme os sistemas do jogo forem conectados.", nx())
end

local function renderPlaceholder(scroll, nx, token, a)
	local hero = card(scroll, 210, nx())
	local tile = create("Frame", {AnchorPoint = Vector2.new(0.5, 0), Position = UDim2.new(0.5, 0, 0, 20), Size = UDim2.fromOffset(76, 76)}, hero)
	buildTile(tile, a, 76)
	label(hero, a.name, 16, {Font = Enum.Font.GothamBold, TextXAlignment = Enum.TextXAlignment.Center, Position = UDim2.fromOffset(0, 104), Size = UDim2.new(1, 0, 0, 24)})
	label(hero, a.desc, 10, {TextColor3 = C.Muted, TextWrapped = true, TextXAlignment = Enum.TextXAlignment.Center, TextYAlignment = Enum.TextYAlignment.Top, Position = UDim2.fromOffset(18, 130), Size = UDim2.new(1, -36, 0, 36)})
	local chip = create("Frame", {AnchorPoint = Vector2.new(0.5, 0), Position = UDim2.new(0.5, 0, 0, 172), Size = UDim2.fromOffset(150, 24), BackgroundColor3 = C.SurfaceHi}, hero)
	corner(chip, 12)
	label(chip, "Aguardando o servidor", 9, {Font = Enum.Font.GothamBold, TextColor3 = C.Accent, TextXAlignment = Enum.TextXAlignment.Center, Size = UDim2.fromScale(1, 1)})
end

renderApp = function(appId, fade)
	local a = APP_BY_ID[appId]
	if not a then return end
	currentPage = "App"
	currentApp = appId
	dock.Visible = false
	setPanel(false)

	local page, token = newPage(fade ~= false)
	header(page, a.name, "NOVA OS")
	local scroll = newScroll(page)
	local fn = RENDERERS[appId]
	if fn then
		fn(scroll, counter(), token, a)
	else
		renderPlaceholder(scroll, counter(), token, a)
	end
end

--========================================================
-- BOTAO FLUTUANTE + ABRIR/FECHAR
--========================================================
local floating = create("TextButton", {
	Name = "NOVA_FloatingButton",
	Size = UDim2.fromOffset(46, 46),
	Position = UDim2.new(1, -62, 0.3, 0),
	Text = "", AutoButtonColor = false, BackgroundColor3 = W,
}, gui)
corner(floating, 16)
create("UIGradient", {Rotation = 90, Color = ColorSequence.new(rgb(44, 54, 82), rgb(18, 22, 36))}, floating)
stroke(floating, C.Border, 0.2)
pressable(floating, 0.9)
label(floating, "N", 20, {Font = Enum.Font.GothamBlack, TextColor3 = C.Accent, TextXAlignment = Enum.TextXAlignment.Center, Size = UDim2.fromScale(1, 1)})

local function setPhoneVisible(state)
	phoneOpen = state
	if state then
		setPanel(false)
		renderHome(false)
		phone.Visible = true
		phone.Position = UDim2.new(1, PHONE_W + 40, 0.5, 0)
		animate(phone, {Position = UDim2.new(1, -16, 0.5, 0)}, 0.34, Enum.EasingStyle.Back)
		if not settings.welcomed then
			settings.welcomed = true
			task.delay(0.7, function()
				if phoneOpen then notify("NOVA OS", "Bem-vindo! Toque em Ajuda para dicas.", "Help") end
			end)
		end
	else
		animate(phone, {Position = UDim2.new(1, PHONE_W + 40, 0.5, 0)}, 0.24)
		task.delay(0.26, function()
			if not phoneOpen then phone.Visible = false end
		end)
	end
end

local function togglePhone()
	setPhoneVisible(not phoneOpen)
end

floating.Activated:Connect(togglePhone)

UserInputService.InputBegan:Connect(function(input, processed)
	if processed then return end
	if input.KeyCode == Enum.KeyCode.M then togglePhone() end
end)

ContextActionService:BindAction("NOVA_OS_Toggle", function(_, state)
	if state == Enum.UserInputState.Begin then togglePhone() end
	return Enum.ContextActionResult.Sink
end, false, Enum.KeyCode.ButtonY)

--========================================================
-- WORLD SIGNALS + LOOP
--========================================================
task.spawn(function()
	while gui.Parent do
		refreshLive()
		task.wait(1)
	end
end)

if WorldState then
	WorldState:GetAttributeChangedSignal("Weather"):Connect(function()
		if currentPage == "Home" then
			renderHome(false)
		elseif currentApp == "Weather" then
			renderApp("Weather", false)
		end
	end)
	WorldState:GetAttributeChangedSignal("DayPhase"):Connect(refreshLive)

	local function onDateChanged()
		if currentPage == "Home" then
			renderHome(false)
		elseif currentApp == "Calendar" then
			renderApp("Calendar", false)
		end
	end
	for _, name in ipairs({"Day", "Month", "Year"}) do
		WorldState:GetAttributeChangedSignal(name):Connect(onDateChanged)
	end
end

gui.Destroying:Connect(function()
	for _, c in ipairs(permConns) do c:Disconnect() end
	for _, c in ipairs(pageConns) do c:Disconnect() end
	ContextActionService:UnbindAction("NOVA_OS_Toggle")
end)

--========================================================
-- INIT
--========================================================
applyBrightness()
renderHome(false)
print("[NOVA OS 3.2] Inicializada com sucesso.")
