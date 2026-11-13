--==============================================================
--  JEFIN MENU  —  menu com funções e binds configuráveis
--  Layout: sidebar de abas + header + cards roláveis
--  Toggles piscam visualmente, botões só dão feedback de clique.
--  Para plugar lógica depois, procure por:  -- [SEM FUNÇÃO]
--==============================================================

local Players          = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local TweenService     = game:GetService("TweenService")
local LocalPlayer      = Players.LocalPlayer

--==============================================================
--  TEMA
--==============================================================
local T = {
	Fundo        = Color3.fromRGB(16, 16, 18),
	Sidebar      = Color3.fromRGB(22, 22, 25),
	Conteudo     = Color3.fromRGB(19, 19, 22),
	Card         = Color3.fromRGB(30, 30, 34),
	CardHover    = Color3.fromRGB(40, 40, 46),
	CardAtivo    = Color3.fromRGB(45, 32, 34),
	Borda        = Color3.fromRGB(44, 44, 50),
	Accent       = Color3.fromRGB(229, 57, 53),
	AccentSuave  = Color3.fromRGB(255, 118, 112),
	Texto        = Color3.fromRGB(240, 240, 245),
	TextoFraco   = Color3.fromRGB(148, 148, 160),
	SwitchOff    = Color3.fromRGB(58, 58, 66),
	Verde        = Color3.fromRGB(76, 209, 122),
}

local FONTE      = Enum.Font.Gotham
local FONTE_BOLD = Enum.Font.GothamBold
local FONTE_MED  = Enum.Font.GothamMedium

--==============================================================
--  HELPERS
--==============================================================
local function new(class, props, children)
	local inst = Instance.new(class)
	local parent = nil
	if props then
		for k, v in pairs(props) do
			if k == "Parent" then parent = v else inst[k] = v end
		end
	end
	if children then
		for _, c in ipairs(children) do c.Parent = inst end
	end
	if parent then inst.Parent = parent end
	return inst
end

local function canto(inst, raio)
	return new("UICorner", { CornerRadius = UDim.new(0, raio), Parent = inst })
end

local function borda(inst, cor, espessura)
	return new("UIStroke", {
		Color = cor or T.Borda,
		Thickness = espessura or 1,
		ApplyStrokeMode = Enum.ApplyStrokeMode.Border,
		Parent = inst,
	})
end

local function tween(inst, tempo, props, estilo)
	local t = TweenService:Create(
		inst,
		TweenInfo.new(tempo, estilo or Enum.EasingStyle.Quad, Enum.EasingDirection.Out),
		props
	)
	t:Play()
	return t
end

local function getGuiParent()
	if typeof(gethui) == "function" then
		local ok, h = pcall(gethui)
		if ok and h then return h end
	end
	local ok, cg = pcall(function() return game:GetService("CoreGui") end)
	if ok and cg then
		local teste = Instance.new("Folder")
		local escreveu = pcall(function() teste.Parent = cg end)
		teste:Destroy()
		if escreveu then return cg end
	end
	return LocalPlayer:WaitForChild("PlayerGui")
end

local GUI_PARENT = getGuiParent()

local genv = (getgenv and getgenv()) or _G
pcall(function()
	if genv.GodzillaCleanup then genv.GodzillaCleanup(); genv.GodzillaCleanup = nil end
	local antigoG = GUI_PARENT:FindFirstChild("GodzillaUI")
	if antigoG then antigoG:Destroy() end
end)
if genv.JefinCleanup then pcall(genv.JefinCleanup) end

local Motor = { Toggles = {}, Botoes = {}, encerrado = false }
local UI    = {}
do
--==============================================================
--  MOTOR — lógica das funções
--==============================================================
local cr = cloneref or function(x) return x end
local Workspace       = cr(game:GetService("Workspace"))
local RunService      = cr(game:GetService("RunService"))
local Lighting        = cr(game:GetService("Lighting"))
local TeleportService = cr(game:GetService("TeleportService"))
local VirtualUser     = cr(game:GetService("VirtualUser"))
local HttpService     = cr(game:GetService("HttpService"))
local GuiService      = cr(game:GetService("GuiService"))
local Mouse           = LocalPlayer:GetMouse()

local Flags = {}
local conns = {}
local encerrado = false
local avisados = {}

-- ajustes rápidos (edite aqui)
local Cfg = {
	VelocidadeFly   = 50,   -- velocidade do voo (slider 10-190)
	VelocidadeSpeed = 50,   -- velocidade do speed hack (slider 10-190)
	DistMax      = 1500,   -- alcance do ESP
	CorESP       = Color3.fromRGB(229, 57, 53),
	FOV          = 160,    -- raio do círculo de mira (pixels)
	Suavizacao   = 0.28,   -- aimbot: 0.05 suave ... 1 trava seca
	ParteAlvo    = "Head", -- Head | HumanoidRootPart | UpperTorso
	TeamCheck    = false,  -- true ignora quem é do seu time
	WallCheck    = false,  -- só mira em quem está visível (toggle "Só Mirar Visível" começa desligado, igual o resto)
	Hitbox       = 12,
	TriggerDelay = 0.12,
}

local PADRAO_PEGAR   = { "pegar", "coletar", "pick", "take", "collect", "loot", "grab", "apanhar" }
local PADRAO_EMPREGO = { "emprego", "trabalh", "job", "work", "contrat", "profiss" }
local PADRAO_GARI    = { "lixo", "lixeira", "trash", "garbage", "litter", "sujeira" }
local PADRAO_MINERA  = { "minerar", "minério", "minerio", "pedra", "rocha", "ore", "rock", "mining", "cristal" }
local PADRAO_REVISTA = { "revistar", "search", "frisk", "saquear", "loot" }

--------------------------------------------------------------
--  utilitários
--------------------------------------------------------------
local function track(c)
	conns[#conns + 1] = c
	return c
end

local function toast(t)
	if UI.toast then UI.toast(t) end
end

local function hum(p)
	local c = p.Character
	return c and c:FindFirstChildOfClass("Humanoid")
end

local function raiz(p)
	local c = p.Character
	return c and c:FindFirstChild("HumanoidRootPart")
end

local function vivo(p)
	local h = hum(p)
	return h ~= nil and h.Health > 0
end

local function digitando()
	return UserInputService:GetFocusedTextBox() ~= nil
end

local function mesmoTime(p)
	return p.Team ~= nil and p.Team == LocalPlayer.Team
end

local function casa(txt, padroes)
	txt = string.lower(txt or "")
	for _, pt in ipairs(padroes) do
		if string.find(txt, pt, 1, true) then return true end
	end
	return false
end

local function jogadorDe(inst)
	local m = inst
	while m and m ~= Workspace do
		local p = Players:GetPlayerFromCharacter(m)
		if p then return p end
		m = m.Parent
	end
	return nil
end

local function irPara(cf)
	local r = raiz(LocalPlayer)
	if r then r.CFrame = cf end
end

local function posDe(obj)
	if obj:IsA("ProximityPrompt") then obj = obj.Parent end
	if obj and obj:IsA("Attachment") then obj = obj.Parent end
	if not obj then return nil end
	if obj:IsA("BasePart") then return obj.Position end
	if obj:IsA("Model") then return obj:GetPivot().Position end
	return nil
end

local function textoPrompt(d)
	local pai = d.Parent
	return d.ActionText .. " " .. d.ObjectText .. " " .. (pai and pai.Name or "")
		.. " " .. ((pai and pai.Parent) and pai.Parent.Name or "")
end

local tokens = {}
local function iniciarLoop(nome, intervalo, passo)
	tokens[nome] = (tokens[nome] or 0) + 1
	local id = tokens[nome]
	task.spawn(function()
		while not encerrado and Flags[nome] and tokens[nome] == id do
			local ok, err = pcall(passo)
			if not ok and not avisados[err] then
				avisados[err] = true
				warn("[Jefin] " .. nome .. ": " .. tostring(err))
			end
			task.wait(intervalo)
		end
	end)
end

local function T_(nome, fn)
	Motor.Toggles[nome] = function(on)
		Flags[nome] = on
		local res = nil
		if fn then res = fn(on) end
		if res == false then
			Flags[nome] = false
			return false
		end
	end
end

local function ehDoJogador(parte)
	return jogadorDe(parte) ~= nil
end

--------------------------------------------------------------
--  bala mágica (fura parede furando o próprio raycast do jogo)
--------------------------------------------------------------
local function furarParede(oldNamecall, metodo, args)
	if metodo ~= "Raycast" and metodo ~= "FindPartOnRayWithIgnoreList" then
		return oldNamecall(workspace, table.unpack(args, 1, args.n))
	end
	for _ = 1, 8 do
		local a, b, c, d = oldNamecall(workspace, table.unpack(args, 1, args.n))
		local parte = (metodo == "Raycast") and (a and a.Instance) or a
		if not parte or ehDoJogador(parte) then
			return a, b, c, d
		end
		if metodo == "Raycast" then
			local rp = args[3]
			if typeof(rp) ~= "RaycastParams" then rp = RaycastParams.new() end
			local novo = {}
			if rp.FilterDescendantsInstances then
				for _, v in ipairs(rp.FilterDescendantsInstances) do novo[#novo + 1] = v end
			end
			novo[#novo + 1] = parte
			rp.FilterDescendantsInstances = novo
			args[3] = rp
			if args.n < 3 then args.n = 3 end
		else
			local novo = {}
			if typeof(args[2]) == "table" then
				for _, v in ipairs(args[2]) do novo[#novo + 1] = v end
			end
			novo[#novo + 1] = parte
			args[2] = novo
		end
	end
	return oldNamecall(workspace, table.unpack(args, 1, args.n))
end

--------------------------------------------------------------
--  hooks do silent aim / bala mágica (instala uma vez)
--------------------------------------------------------------
local Hook = genv.JefinHook
if not Hook then
	Hook = { Ativo = false, Alvo = nil, Parede = false, Instalado = false }
	genv.JefinHook = Hook
end
Hook.Mouse = Mouse
Hook.Ativo = false
Hook.Alvo = nil
Hook.Parede = false

local function instalarHooks()
	if Hook.Instalado then return true end
	if not (hookmetamethod and getnamecallmethod and checkcaller) then return false end
	local nc = newcclosure or function(f) return f end
	local oldIndex, oldNamecall

	local metodosWS  = { Raycast = true, FindPartOnRay = true, FindPartOnRayWithIgnoreList = true, FindPartOnRayWithWhitelist = true }

	oldIndex = hookmetamethod(game, "__index", nc(function(self, chave)
		if (chave == "Hit" or chave == "Target" or chave == "UnitRay")
			and Hook.Ativo and Hook.Alvo and self == Hook.Mouse and not checkcaller() then
			local alvo = Hook.Alvo
			if chave == "Hit" then
				return CFrame.new(alvo.Position)
			elseif chave == "Target" then
				return alvo
			else
				local r = oldIndex(self, chave)
				return Ray.new(r.Origin, (alvo.Position - r.Origin).Unit)
			end
		end
		return oldIndex(self, chave)
	end))

	-- nota: o silent aim mira só via Mouse.Hit/Target/UnitRay (hook de __index acima).
	-- aqui NÃO mexe em raycast de workspace nem de câmera pro silent — isso é o que
	-- tava quebrando a câmera e o disparo (o jogo também usa esses raycasts pra
	-- outras coisas, tipo colisão de câmera, e redirecionar tudo pro alvo bagunçava).
	-- só a bala mágica toca em raycast, e só quando a parede atrapalha, sem mudar a direção.
	oldNamecall = hookmetamethod(game, "__namecall", nc(function(self, ...)
		local metodo = getnamecallmethod()
		if Hook.Parede and not checkcaller() and self == workspace and metodosWS[metodo] then
			local args = table.pack(...)
			return furarParede(oldNamecall, metodo, args)
		end
		return oldNamecall(self, ...)
	end))

	Hook.Instalado = true
	return true
end

--------------------------------------------------------------
--  pasta de efeitos e tela de ESP (tudo em GuiObject, sem Drawing)
--------------------------------------------------------------
local pasta = new("Folder", { Name = "JefinFX", Parent = GUI_PARENT })
local telaESP = new("ScreenGui", {
	Name = "JefinESP",
	ResetOnSpawn = false,
	IgnoreGuiInset = true,
	DisplayOrder = 10,
	ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
	Parent = GUI_PARENT,
})

--------------------------------------------------------------
--  alvo (aimbot / silent / trigger)
--------------------------------------------------------------
local function parteAlvo(p)
	local c = p.Character
	if not c then return nil end
	return c:FindFirstChild(Cfg.ParteAlvo) or c:FindFirstChild("HumanoidRootPart")
end

local function visivel(origem, parte, char)
	local params = RaycastParams.new()
	params.FilterType = Enum.RaycastFilterType.Exclude
	params.FilterDescendantsInstances = { LocalPlayer.Character, char }
	params.IgnoreWater = true
	return Workspace:Raycast(origem, parte.Position - origem, params) == nil
end

local function melhorAlvo(raio)
	local cam = Workspace.CurrentCamera
	if not cam then return nil end
	local mp = UserInputService:GetMouseLocation()
	local melhor, melhorD = nil, raio
	for _, p in ipairs(Players:GetPlayers()) do
		if p ~= LocalPlayer and vivo(p) and not (Cfg.TeamCheck and mesmoTime(p)) then
			local parte = parteAlvo(p)
			if parte then
				local sp, on = cam:WorldToViewportPoint(parte.Position)
				if on then
					local d = (Vector2.new(sp.X, sp.Y) - mp).Magnitude
					if d < melhorD and (not Cfg.WallCheck or visivel(cam.CFrame.Position, parte, p.Character)) then
						melhor, melhorD = parte, d
					end
				end
			end
		end
	end
	return melhor
end

local fovCirculo = new("Frame", {
	Name = "FOV",
	AnchorPoint = Vector2.new(0.5, 0.5),
	BackgroundTransparency = 1,
	Visible = false,
	ZIndex = 5,
	Parent = telaESP,
})
canto(fovCirculo, 9999)
borda(fovCirculo, T.Accent, 1)

--------------------------------------------------------------
--  ESP (caixa, nome, vida, tracer via GuiObject — mesma base que os chams)
--------------------------------------------------------------
local esp = {}

local function criarESP(p)
	if esp[p] or p == LocalPlayer then return end
	local o = {}
	o.caixa = new("Frame", {
		BackgroundTransparency = 1, BorderSizePixel = 0, Visible = false, ZIndex = 1, Parent = telaESP,
	})
	o.caixaStroke = borda(o.caixa, Cfg.CorESP, 1)
	o.nome = new("TextLabel", {
		BackgroundTransparency = 1, Font = FONTE_BOLD, TextSize = 13, TextStrokeTransparency = 0,
		TextColor3 = Color3.new(1, 1, 1), Visible = false, ZIndex = 2, Parent = telaESP,
	})
	o.vidaFundo = new("Frame", {
		BackgroundColor3 = Color3.new(0, 0, 0), BorderSizePixel = 0, Visible = false, ZIndex = 1, Parent = telaESP,
	})
	o.vidaBarra = new("Frame", { BorderSizePixel = 0, Visible = false, ZIndex = 2, Parent = telaESP })
	o.tracer = new("Frame", {
		AnchorPoint = Vector2.new(0, 0.5), BorderSizePixel = 0, Visible = false, ZIndex = 1, Parent = telaESP,
	})
	o.chams = new("Highlight", {
		FillTransparency = 0.5, OutlineTransparency = 0, Enabled = false,
		DepthMode = Enum.HighlightDepthMode.AlwaysOnTop, Parent = pasta,
	})
	esp[p] = o
end

local function removerESP(p)
	local o = esp[p]
	if not o then return end
	for _, v in pairs(o) do
		pcall(function() v:Destroy() end)
	end
	esp[p] = nil
end

local function atualizarESP(cam)
	local vp = cam.ViewportSize
	local baseTracer = Vector2.new(vp.X / 2, vp.Y)
	local camPos = cam.CFrame.Position
	local espOn, trOn, chOn = Flags["ESP Jogadores"], Flags["Tracers"], Flags["Chams"]

	for p, o in pairs(esp) do
		local r, h = raiz(p), hum(p)
		local vivoP = r ~= nil and h ~= nil and h.Health > 0
		local dist = vivoP and (r.Position - camPos).Magnitude or math.huge
		local okBase = vivoP and dist <= Cfg.DistMax and not (Cfg.TeamCheck and mesmoTime(p))

		o.chams.Enabled = (chOn and okBase) and true or false
		if chOn and okBase then
			o.chams.Adornee = p.Character
			o.chams.FillColor = Cfg.CorESP
			o.chams.OutlineColor = Color3.new(1, 1, 1)
		end

		local pos, naTela = nil, false
		if okBase then
			local sp, on = cam:WorldToViewportPoint(r.Position)
			pos, naTela = Vector2.new(sp.X, sp.Y), on
		end
		local ok = okBase and naTela

		local mostrarCaixa = (espOn and ok) and true or false
		o.caixa.Visible = mostrarCaixa
		o.nome.Visible = mostrarCaixa
		o.vidaFundo.Visible = mostrarCaixa
		o.vidaBarra.Visible = mostrarCaixa
		if mostrarCaixa then
			local topo = cam:WorldToViewportPoint(r.Position + Vector3.new(0, 3, 0))
			local pe = cam:WorldToViewportPoint(r.Position - Vector3.new(0, 3.5, 0))
			local alt = math.max(math.abs(pe.Y - topo.Y), 4)
			local larg = alt * 0.55
			local x, y = pos.X - larg / 2, topo.Y
			o.caixa.Position = UDim2.new(0, x, 0, y)
			o.caixa.Size = UDim2.new(0, larg, 0, alt)
			o.caixaStroke.Color = Cfg.CorESP
			o.nome.Position = UDim2.new(0, x - 10, 0, y - 16)
			o.nome.Size = UDim2.new(0, larg + 20, 0, 14)
			o.nome.Text = p.DisplayName .. " [" .. math.floor(dist) .. "m]"
			local frac = math.clamp(h.Health / math.max(h.MaxHealth, 1), 0, 1)
			o.vidaFundo.Position = UDim2.new(0, x - 6, 0, y)
			o.vidaFundo.Size = UDim2.new(0, 3, 0, alt)
			o.vidaBarra.Position = UDim2.new(0, x - 6, 0, y + alt * (1 - frac))
			o.vidaBarra.Size = UDim2.new(0, 3, 0, alt * frac)
			o.vidaBarra.BackgroundColor3 = Color3.fromRGB(255 * (1 - frac), 255 * frac, 60)
		end

		local mostrarTracer = (trOn and ok) and true or false
		o.tracer.Visible = mostrarTracer
		if mostrarTracer then
			local delta = pos - baseTracer
			local comprimento = delta.Magnitude
			o.tracer.Position = UDim2.new(0, baseTracer.X, 0, baseTracer.Y)
			o.tracer.Size = UDim2.new(0, comprimento, 0, 1)
			o.tracer.Rotation = math.deg(math.atan2(delta.Y, delta.X))
			o.tracer.BackgroundColor3 = Cfg.CorESP
		end
	end
end

for _, p in ipairs(Players:GetPlayers()) do criarESP(p) end
track(Players.PlayerAdded:Connect(criarESP))
track(Players.PlayerRemoving:Connect(removerESP))

--------------------------------------------------------------
--  varredura do mundo (prompts e tools), cache de 3s
--------------------------------------------------------------
local mundo = { t = -10, prompts = {}, tools = {} }
local function varrer()
	if os.clock() - mundo.t < 3 then return mundo end
	mundo.t = os.clock()
	local prompts, tools = {}, {}
	for _, d in ipairs(Workspace:GetDescendants()) do
		if d:IsA("ProximityPrompt") then
			prompts[#prompts + 1] = d
		elseif d:IsA("Tool") then
			local pai = d.Parent
			if pai and not Players:GetPlayerFromCharacter(pai)
				and not pai:FindFirstChildOfClass("Humanoid") and d:FindFirstChild("Handle") then
				tools[#tools + 1] = d
			end
		end
	end
	mundo.prompts, mundo.tools = prompts, tools
	return mundo
end

local function acionarPrompt(prompt, teleportar)
	if not fireproximityprompt then return false end
	if teleportar then
		local pos = posDe(prompt)
		if pos then
			irPara(CFrame.new(pos + Vector3.new(0, 3, 0)))
			task.wait(0.15)
		end
	end
	local hd = prompt.HoldDuration
	pcall(function() prompt.HoldDuration = 0 end)
	pcall(fireproximityprompt, prompt)
	pcall(function() prompt.HoldDuration = hd end)
	return true
end

local itensFX = {}
local function limparItens()
	for _, f in ipairs(itensFX) do pcall(function() f:Destroy() end) end
	itensFX = {}
end

local function desenharItens()
	limparItens()
	local cam = Workspace.CurrentCamera
	if not cam then return end
	local m = varrer()
	local n = 0
	local function marcar(parte, nome)
		if n >= 30 or not parte then return end
		if (parte.Position - cam.CFrame.Position).Magnitude > Cfg.DistMax then return end
		n += 1
		local hl = new("Highlight", {
			Adornee = parte, FillColor = Cfg.CorESP, OutlineColor = Color3.new(1, 1, 1),
			FillTransparency = 0.6, DepthMode = Enum.HighlightDepthMode.AlwaysOnTop, Parent = pasta,
		})
		local bb = new("BillboardGui", {
			Adornee = parte, Size = UDim2.new(0, 140, 0, 18), StudsOffset = Vector3.new(0, 2, 0),
			AlwaysOnTop = true, Parent = pasta,
		})
		new("TextLabel", {
			Size = UDim2.new(1, 0, 1, 0), BackgroundTransparency = 1, Text = nome, Font = FONTE_BOLD,
			TextSize = 12, TextColor3 = Cfg.CorESP, TextStrokeTransparency = 0, Parent = bb,
		})
		itensFX[#itensFX + 1] = hl
		itensFX[#itensFX + 1] = bb
	end
	for _, t in ipairs(m.tools) do
		local hnd = t:FindFirstChild("Handle")
		if hnd and hnd:IsA("BasePart") then marcar(hnd, t.Name) end
	end
	for _, p in ipairs(m.prompts) do
		if p.Enabled and casa(textoPrompt(p), PADRAO_PEGAR) then
			local base = p.Parent
			if base and base:IsA("Attachment") then base = base.Parent end
			if base and base:IsA("BasePart") then
				marcar(base, p.ObjectText ~= "" and p.ObjectText or base.Name)
			end
		end
	end
end

--------------------------------------------------------------
--  movimento
--------------------------------------------------------------
local function passoFly(r, h, cam)
	local ui = UserInputService
	local dir = Vector3.zero
	if not digitando() then
		local cf = cam.CFrame
		if ui:IsKeyDown(Enum.KeyCode.W) then dir += cf.LookVector end
		if ui:IsKeyDown(Enum.KeyCode.S) then dir -= cf.LookVector end
		if ui:IsKeyDown(Enum.KeyCode.D) then dir += cf.RightVector end
		if ui:IsKeyDown(Enum.KeyCode.A) then dir -= cf.RightVector end
		if ui:IsKeyDown(Enum.KeyCode.Space) then dir += Vector3.yAxis end
		if ui:IsKeyDown(Enum.KeyCode.LeftControl) or ui:IsKeyDown(Enum.KeyCode.Q) then dir -= Vector3.yAxis end
	end
	if dir.Magnitude > 0 then dir = dir.Unit end
	h.PlatformStand = true
	r.AssemblyLinearVelocity = dir * Cfg.VelocidadeFly
	r.AssemblyAngularVelocity = Vector3.zero
	local plano = Vector3.new(cam.CFrame.LookVector.X, 0, cam.CFrame.LookVector.Z)
	if plano.Magnitude > 0.01 then
		r.CFrame = CFrame.lookAt(r.Position, r.Position + plano)
	end
end

local ncOrig = setmetatable({}, { __mode = "k" })
local function passoNoclip()
	local c = LocalPlayer.Character
	if not c then return end
	for _, d in ipairs(c:GetDescendants()) do
		if d:IsA("BasePart") and d.CanCollide then
			ncOrig[d] = true
			d.CanCollide = false
		end
	end
end
local function restaurarNoclip()
	for parte in pairs(ncOrig) do
		if parte.Parent then parte.CanCollide = true end
		ncOrig[parte] = nil
	end
end

local hitOrig = setmetatable({}, { __mode = "k" })
local function passoHitbox()
	for _, p in ipairs(Players:GetPlayers()) do
		if p ~= LocalPlayer and not (Cfg.TeamCheck and mesmoTime(p)) then
			local r = raiz(p)
			if r and vivo(p) then
				if not hitOrig[r] then hitOrig[r] = { r.Size, r.Transparency, r.CanCollide } end
				r.Size = Vector3.new(Cfg.Hitbox, Cfg.Hitbox, Cfg.Hitbox)
				r.Transparency = 0.75
				r.CanCollide = false
			end
		end
	end
end
local function restaurarHitbox()
	for parte, d in pairs(hitOrig) do
		if parte.Parent then
			parte.Size, parte.Transparency, parte.CanCollide = d[1], d[2], d[3]
		end
		hitOrig[parte] = nil
	end
end

local fbOrig
local function fbAplicar()
	Lighting.Brightness = 2
	Lighting.ClockTime = 14
	Lighting.FogEnd = 1e6
	Lighting.GlobalShadows = false
	Lighting.Ambient = Color3.fromRGB(178, 178, 178)
	Lighting.OutdoorAmbient = Color3.fromRGB(178, 178, 178)
end

local ultimoTiro = 0
local function passoTrigger()
	local alvo = Mouse.Target
	if not alvo then return end
	local p = jogadorDe(alvo)
	if p and p ~= LocalPlayer and vivo(p) and not (Cfg.TeamCheck and mesmoTime(p))
		and os.clock() - ultimoTiro > Cfg.TriggerDelay then
		ultimoTiro = os.clock()
		mouse1click()
	end
end

--------------------------------------------------------------
--  no recoil / no spread (varredura de configs de arma)
--------------------------------------------------------------
local CH_RECUO  = { "Recoil", "recoil", "RecoilX", "RecoilY", "VerticalRecoil", "HorizontalRecoil", "RecoilMax", "RecoilMin", "RecoilAmount" }
local CH_SPREAD = { "Spread", "spread", "BaseSpread", "MinSpread", "MaxSpread", "SpreadX", "SpreadY", "Inaccuracy", "Dispersion" }
local armaRec = {}

local function conjunto(lista)
	local s = {}
	for _, k in ipairs(lista) do s[k] = true end
	return s
end

local function zerarEm(tipo, chaves)
	local set = conjunto(chaves)
	local fontes = {}
	if LocalPlayer.Character then fontes[#fontes + 1] = LocalPlayer.Character end
	local bp = LocalPlayer:FindFirstChildOfClass("Backpack")
	if bp then fontes[#fontes + 1] = bp end
	for _, f in ipairs(fontes) do
		for _, d in ipairs(f:GetDescendants()) do
			if (d:IsA("NumberValue") or d:IsA("IntValue")) and set[d.Name] and d.Value ~= 0 then
				armaRec[#armaRec + 1] = { tipo = tipo, inst = d, valor = d.Value }
				d.Value = 0
			end
			for k, v in pairs(d:GetAttributes()) do
				if set[k] and type(v) == "number" and v ~= 0 then
					armaRec[#armaRec + 1] = { tipo = tipo, dono = d, attr = k, valor = v }
					d:SetAttribute(k, 0)
				end
			end
		end
	end
	if getgc then
		for i, t in ipairs(getgc(true)) do
			if type(t) == "table" then
				for _, k in ipairs(chaves) do
					local v = rawget(t, k)
					if type(v) == "number" and v ~= 0 then
						armaRec[#armaRec + 1] = { tipo = tipo, tabela = t, chave = k, valor = v }
						pcall(rawset, t, k, 0)
					end
				end
			end
			if i % 3000 == 0 then task.wait() end
		end
	end
end

local function restaurarArma(tipo)
	for i = #armaRec, 1, -1 do
		local r = armaRec[i]
		if r.tipo == tipo then
			if r.tabela then
				pcall(rawset, r.tabela, r.chave, r.valor)
			elseif r.inst then
				pcall(function() r.inst.Value = r.valor end)
			elseif r.dono then
				pcall(function() r.dono:SetAttribute(r.attr, r.valor) end)
			end
			table.remove(armaRec, i)
		end
	end
end

local armaLoop = false
local function garantirLoopArma()
	if armaLoop then return end
	armaLoop = true
	task.spawn(function()
		while not encerrado and (Flags["No Recoil"] or Flags["No Spread"]) do
			if Flags["No Recoil"] then pcall(zerarEm, "recuo", CH_RECUO) end
			if Flags["No Spread"] then pcall(zerarEm, "spread", CH_SPREAD) end
			if not Flags["No Recoil"] then restaurarArma("recuo") end
			if not Flags["No Spread"] then restaurarArma("spread") end
			for _ = 1, 12 do
				if encerrado or not (Flags["No Recoil"] or Flags["No Spread"]) then break end
				task.wait(0.5)
			end
		end
		armaLoop = false
	end)
end

--------------------------------------------------------------
--  farm / coleta
--------------------------------------------------------------
local function passoFarm(padroes)
	local r = raiz(LocalPlayer)
	if not r then return end
	local m = varrer()
	local melhor, md = nil, math.huge
	for _, d in ipairs(m.prompts) do
		if d.Parent and d.Enabled and casa(textoPrompt(d), padroes) then
			local pos = posDe(d)
			if pos then
				local dist = (pos - r.Position).Magnitude
				if dist < md then melhor, md = d, dist end
			end
		end
	end
	if melhor then acionarPrompt(melhor, md > 10) end
end

local function passoColetar()
	local r = raiz(LocalPlayer)
	if not r then return end
	local m = varrer()
	for _, d in ipairs(m.prompts) do
		if d.Parent and d.Enabled and casa(textoPrompt(d), PADRAO_PEGAR) then
			local pos = posDe(d)
			if pos and (pos - r.Position).Magnitude <= 18 then acionarPrompt(d, false) end
		end
	end
	if firetouchinterest then
		for _, t in ipairs(m.tools) do
			local hnd = t:FindFirstChild("Handle")
			if hnd and hnd:IsA("BasePart") and (hnd.Position - r.Position).Magnitude <= 20 then
				pcall(function()
					firetouchinterest(r, hnd, 0)
					firetouchinterest(r, hnd, 1)
				end)
			end
		end
	end
end

--------------------------------------------------------------
--  loops principais
--------------------------------------------------------------
local function seguro(nome, f)
	return function(...)
		local ok, err = pcall(f, ...)
		if not ok and not avisados[err] then
			avisados[err] = true
			warn("[Jefin] " .. nome .. ": " .. tostring(err))
		end
	end
end

track(RunService.RenderStepped:Connect(seguro("render", function()
	local cam = Workspace.CurrentCamera
	if not cam then return end
	local usaFOV = (Flags["Aimbot"] or Flags["Silent Aim"]) and true or false
	fovCirculo.Visible = usaFOV
	if usaFOV then
		local mp = UserInputService:GetMouseLocation()
		fovCirculo.Position = UDim2.new(0, mp.X, 0, mp.Y)
		fovCirculo.Size = UDim2.new(0, Cfg.FOV * 2, 0, Cfg.FOV * 2)
	end
	Hook.Alvo = Flags["Silent Aim"] and melhorAlvo(Cfg.FOV) or nil
	if Flags["Aimbot"] and UserInputService:IsMouseButtonPressed(Enum.UserInputType.MouseButton2) then
		local alvo = Hook.Alvo or melhorAlvo(Cfg.FOV)
		if alvo then
			cam.CFrame = cam.CFrame:Lerp(CFrame.lookAt(cam.CFrame.Position, alvo.Position), Cfg.Suavizacao)
		end
	end
	atualizarESP(cam)
end)))

track(RunService.Heartbeat:Connect(seguro("heartbeat", function()
	local r, h = raiz(LocalPlayer), hum(LocalPlayer)
	local cam = Workspace.CurrentCamera
	if r and h then
		if Flags["Fly"] and cam then passoFly(r, h, cam) end
		if Flags["Speed Hack"] and not Flags["Fly"] and h.SeatPart == nil then
			local md = h.MoveDirection
			if md.Magnitude > 0 then
				local v = r.AssemblyLinearVelocity
				r.AssemblyLinearVelocity = Vector3.new(md.X * Cfg.VelocidadeSpeed, v.Y, md.Z * Cfg.VelocidadeSpeed)
			end
		end
		if Flags["Bypass de Queda"] then
			local v = r.AssemblyLinearVelocity
			if v.Y < -50 then r.AssemblyLinearVelocity = Vector3.new(v.X, -50, v.Z) end
		end
		if Flags["Bunny Hop"] and not digitando() and UserInputService:IsKeyDown(Enum.KeyCode.Space)
			and h.FloorMaterial ~= Enum.Material.Air then
			h:ChangeState(Enum.HumanoidStateType.Jumping)
		end
	end
	if Flags["Hitbox Expander"] then passoHitbox() end
	if Flags["Full Bright"] then fbAplicar() end
	if Flags["Triggerbot"] then passoTrigger() end
end)))

track(RunService.Stepped:Connect(seguro("noclip", function()
	if Flags["Noclip"] then passoNoclip() end
end)))

track(UserInputService.JumpRequest:Connect(seguro("pulo", function()
	if Flags["Infinite Jump"] then
		local h = hum(LocalPlayer)
		if h then h:ChangeState(Enum.HumanoidStateType.Jumping) end
	end
end)))

track(LocalPlayer.Idled:Connect(seguro("afk", function()
	if Flags["Anti AFK"] then
		VirtualUser:CaptureController()
		VirtualUser:ClickButton2(Vector2.new())
	end
end)))

track(GuiService.ErrorMessageChanged:Connect(seguro("rejoin", function(msg)
	if Flags["Auto Rejoin"] and msg and msg ~= "" then
		task.wait(2)
		TeleportService:Teleport(game.PlaceId, LocalPlayer)
	end
end)))

--------------------------------------------------------------
--  TOGGLES
--------------------------------------------------------------
T_("Aimbot")

T_("Silent Aim", function(on)
	if on then
		local ok, res = pcall(instalarHooks)
		if not ok or not res then
			toast("silent aim: executor sem hookmetamethod")
			return false
		end
	end
	Hook.Ativo = on
	if not on then Hook.Alvo = nil end
end)

T_("Bala Mágica", function(on)
	if on then
		local ok, res = pcall(instalarHooks)
		if not ok or not res then
			toast("bala mágica: executor sem hookmetamethod")
			return false
		end
	end
	Hook.Parede = on
end)

T_("Triggerbot", function(on)
	if on and not mouse1click then
		toast("executor sem mouse1click")
		return false
	end
end)

T_("No Recoil", function(on)
	if on then garantirLoopArma() else restaurarArma("recuo") end
end)

T_("No Spread", function(on)
	if on then garantirLoopArma() else restaurarArma("spread") end
end)

T_("Hitbox Expander", function(on)
	if not on then restaurarHitbox() end
end)

T_("ESP Jogadores")
T_("Tracers")
T_("Chams")

T_("ESP Itens", function(on)
	if on then
		iniciarLoop("ESP Itens", 3, desenharItens)
	else
		limparItens()
	end
end)

T_("Full Bright", function(on)
	if on then
		fbOrig = { Lighting.Brightness, Lighting.ClockTime, Lighting.FogEnd,
			Lighting.GlobalShadows, Lighting.Ambient, Lighting.OutdoorAmbient }
	elseif fbOrig then
		Lighting.Brightness, Lighting.ClockTime, Lighting.FogEnd,
			Lighting.GlobalShadows, Lighting.Ambient, Lighting.OutdoorAmbient = table.unpack(fbOrig)
		fbOrig = nil
	end
end)

T_("Ignorar Time", function(on)
	Cfg.TeamCheck = on
end)

T_("Só Mirar Visível", function(on)
	Cfg.WallCheck = on
end)

T_("Noclip", function(on)
	if not on then restaurarNoclip() end
end)

T_("Fly", function(on)
	if not on then
		local h = hum(LocalPlayer)
		if h then h.PlatformStand = false end
	end
end)

T_("Bypass de Queda")
T_("Speed Hack")
T_("Infinite Jump")
T_("Bunny Hop")

local function exigirPrompt(on)
	if on and not fireproximityprompt then
		toast("executor sem fireproximityprompt")
		return false
	end
end

T_("Farm Gari", function(on)
	if exigirPrompt(on) == false then return false end
	if on then iniciarLoop("Farm Gari", 0.6, function() passoFarm(PADRAO_GARI) end) end
end)

T_("Farm Minerador", function(on)
	if exigirPrompt(on) == false then return false end
	if on then iniciarLoop("Farm Minerador", 0.6, function() passoFarm(PADRAO_MINERA) end) end
end)

T_("Auto Coletar", function(on)
	if on then iniciarLoop("Auto Coletar", 0.5, passoColetar) end
end)

T_("Auto Rejoin")
T_("Anti AFK")

--------------------------------------------------------------
--  BOTÕES
--------------------------------------------------------------
local posSalva = nil
local rota, rotaAtiva = {}, false

Motor.Botoes["Auto Revistar"] = function()
	if not fireproximityprompt then toast("executor sem fireproximityprompt"); return end
	local r = raiz(LocalPlayer)
	if not r then return end
	task.spawn(function()
		local n = 0
		for _, d in ipairs(Workspace:GetDescendants()) do
			if d:IsA("ProximityPrompt") and d.Enabled and casa(textoPrompt(d), PADRAO_REVISTA) then
				local pos = posDe(d)
				if pos and (pos - r.Position).Magnitude <= 25 then
					acionarPrompt(d, false)
					n += 1
					task.wait(0.15)
				end
			end
		end
		toast(n > 0 and ("revistados: " .. n) or "nada para revistar por perto")
	end)
end

Motor.Botoes["Roubar Inventário"] = function()
	toast("depende dos remotes do jogo — falta o jogo alvo")
end

Motor.Botoes["Cor do ESP"] = function()
	UI.cores(function(cor) Cfg.CorESP = cor end)
end

Motor.Botoes["Distância Máxima"] = function()
	UI.slider("Distância Máxima", 100, 5000, Cfg.DistMax, function(v) Cfg.DistMax = v end)
end

Motor.Botoes["FOV do Aimbot"] = function()
	UI.slider("FOV do Aimbot", 30, 500, Cfg.FOV, function(v) Cfg.FOV = v end)
end

Motor.Botoes["Suavização do Aimbot"] = function()
	UI.slider("Suavização (5=suave, 100=seco)", 5, 100, math.floor(Cfg.Suavizacao * 100 + 0.5), function(v)
		Cfg.Suavizacao = v / 100
	end)
end

Motor.Botoes["Delay do Triggerbot"] = function()
	UI.slider("Delay do Triggerbot (ms)", 30, 500, math.floor(Cfg.TriggerDelay * 1000 + 0.5), function(v)
		Cfg.TriggerDelay = v / 1000
	end)
end

Motor.Botoes["Tamanho da Hitbox"] = function()
	UI.slider("Tamanho da Hitbox", 4, 30, Cfg.Hitbox, function(v) Cfg.Hitbox = v end)
end

Motor.Botoes["Parte do Corpo (Aimbot)"] = function()
	UI.menu("Mirar em", {
		{
			texto = "Cabeça (Head)",
			acao = function() Cfg.ParteAlvo = "Head"; toast("mirando na cabeça") end,
		},
		{
			texto = "Tronco (UpperTorso)",
			acao = function() Cfg.ParteAlvo = "UpperTorso"; toast("mirando no tronco") end,
		},
		{
			texto = "Centro (HumanoidRootPart)",
			acao = function() Cfg.ParteAlvo = "HumanoidRootPart"; toast("mirando no centro") end,
		},
	})
end

Motor.Botoes["Velocidade do Fly"] = function()
	UI.slider("Velocidade do Fly", 10, 190, Cfg.VelocidadeFly, function(v) Cfg.VelocidadeFly = v end)
end

Motor.Botoes["Velocidade do Speed"] = function()
	UI.slider("Velocidade do Speed", 10, 190, Cfg.VelocidadeSpeed, function(v) Cfg.VelocidadeSpeed = v end)
end

Motor.Botoes["Ir para Base"] = function()
	local sp = LocalPlayer.RespawnLocation or Workspace:FindFirstChildWhichIsA("SpawnLocation", true)
	if sp then
		irPara(sp.CFrame + Vector3.new(0, 5, 0))
		toast("teleportado para a base")
	else
		toast("nenhum spawn encontrado")
	end
end

local function opcoesJogadores(acao)
	local lista = {}
	for _, p in ipairs(Players:GetPlayers()) do
		if p ~= LocalPlayer then lista[#lista + 1] = p end
	end
	table.sort(lista, function(a, b) return a.Name:lower() < b.Name:lower() end)
	local ops = {}
	for _, p in ipairs(lista) do
		ops[#ops + 1] = {
			texto = p.DisplayName .. "  (@" .. p.Name .. ")",
			acao = function() acao(p) end,
		}
	end
	return ops
end

Motor.Botoes["Teleportar até Jogador"] = function()
	UI.menu("Teleportar até", opcoesJogadores(function(p)
		local r = raiz(p)
		if r then irPara(r.CFrame * CFrame.new(0, 0, 3)) else toast("alvo sem personagem") end
	end))
end

Motor.Botoes["Espectar Jogador"] = function()
	UI.menu("Espectar", opcoesJogadores(function(p)
		local h = hum(p)
		local cam = Workspace.CurrentCamera
		if h and cam then cam.CameraSubject = h else toast("alvo sem personagem") end
	end))
end

Motor.Botoes["Voltar Câmera"] = function()
	local h = hum(LocalPlayer)
	local cam = Workspace.CurrentCamera
	if h and cam then cam.CameraSubject = h end
end

Motor.Botoes["Salvar Posição"] = function()
	local r = raiz(LocalPlayer)
	if r then
		posSalva = r.CFrame
		toast("posição salva")
	end
end

Motor.Botoes["Ir para Salvo"] = function()
	if posSalva then irPara(posSalva) else toast("nenhuma posição salva") end
end

Motor.Botoes["Entrar em Emprego"] = function()
	local ops = {}
	for _, d in ipairs(Workspace:GetDescendants()) do
		if #ops >= 25 then break end
		if d:IsA("ProximityPrompt") and casa(textoPrompt(d), PADRAO_EMPREGO) then
			local nome = d.ObjectText ~= "" and d.ObjectText or (d.Parent and d.Parent.Name or "?")
			ops[#ops + 1] = {
				texto = nome .. "  —  " .. d.ActionText,
				acao = function()
					if not acionarPrompt(d, true) then toast("executor sem fireproximityprompt") end
				end,
			}
		end
	end
	if #ops == 0 then toast("nenhum emprego encontrado"); return end
	UI.menu("Empregos", ops)
end

local function iniciarRota()
	if #rota < 2 then toast("adicione pelo menos 2 pontos"); return false end
	rotaAtiva = true
	task.spawn(function()
		local i = 1
		while rotaAtiva and not encerrado do
			local h, r = hum(LocalPlayer), raiz(LocalPlayer)
			if h and r then
				h:MoveTo(rota[i])
				local t0 = os.clock()
				while rotaAtiva and not encerrado and os.clock() - t0 < 8 do
					r = raiz(LocalPlayer)
					if not r or (r.Position - rota[i]).Magnitude <= 4 then break end
					task.wait(0.1)
				end
				i = i % #rota + 1
			else
				task.wait(0.5)
			end
		end
	end)
	return true
end

Motor.Botoes["Rota Personalizada"] = function()
	UI.menu("Rota Personalizada", {
		{
			texto = "Adicionar ponto atual  (" .. #rota .. ")",
			manter = true,
			acao = function()
				local r = raiz(LocalPlayer)
				if r then rota[#rota + 1] = r.Position end
				return "Adicionar ponto atual  (" .. #rota .. ")"
			end,
		},
		{
			texto = "Limpar pontos",
			manter = true,
			acao = function()
				rota = {}
				rotaAtiva = false
				return "Limpar pontos  (feito)"
			end,
		},
		{
			texto = rotaAtiva and "Parar rota" or "Iniciar rota",
			manter = true,
			acao = function()
				if rotaAtiva then
					rotaAtiva = false
					return "Iniciar rota"
				end
				return iniciarRota() and "Parar rota" or "Iniciar rota"
			end,
		},
	})
end

Motor.Botoes["Lista de Staffs"] = function()
	task.spawn(function()
		local grupo = (game.CreatorType == Enum.CreatorType.Group) and game.CreatorId or nil
		local ops = {}
		for _, p in ipairs(Players:GetPlayers()) do
			local cargo, rank = nil, nil
			if grupo then
				pcall(function()
					rank = p:GetRankInGroup(grupo)
					cargo = p:GetRoleInGroup(grupo)
				end)
			end
			local equipe = p.Team and p.Team.Name or ""
			local porCargo = rank ~= nil and rank > 1
			local porEquipe = casa(equipe, { "staff", "admin", "mod", "suporte", "equipe" })
			if porCargo or porEquipe then
				ops[#ops + 1] = { texto = p.DisplayName .. "  —  " .. (porCargo and cargo or equipe) }
			end
		end
		if #ops == 0 then toast("nenhum staff online detectado"); return end
		UI.menu("Staffs online", ops)
	end)
end

Motor.Botoes["Entrar em Carro"] = function()
	local r = raiz(LocalPlayer)
	if not r then return end
	local melhor, ms = nil, 150
	for _, d in ipairs(Workspace:GetDescendants()) do
		if (d:IsA("VehicleSeat") or d:IsA("Seat")) and not d.Disabled and d.Occupant == nil then
			local score = (d.Position - r.Position).Magnitude - (d:IsA("VehicleSeat") and 20 or 0)
			if score < ms then melhor, ms = d, score end
		end
	end
	if melhor then
		irPara(melhor.CFrame * CFrame.new(0, 2, 0))
	else
		toast("nenhum carro livre por perto")
	end
end

Motor.Botoes["Copiar Job ID"] = function()
	if setclipboard then
		setclipboard(game.JobId)
		toast("Job ID copiado")
	else
		toast("executor sem setclipboard")
	end
end

Motor.Botoes["Servidor Hop"] = function()
	local req = request or http_request or (type(syn) == "table" and syn.request) or nil
	if not req then toast("executor sem request"); return end
	task.spawn(function()
		toast("procurando servidor...")
		local ok, res = pcall(req, {
			Url = "https://games.roblox.com/v1/games/" .. tostring(game.PlaceId)
				.. "/servers/Public?sortOrder=Asc&limit=100",
			Method = "GET",
		})
		if not ok or not res or not res.Body then toast("falha ao listar servidores"); return end
		local okJ, dados = pcall(function() return HttpService:JSONDecode(res.Body) end)
		if not okJ or type(dados) ~= "table" or type(dados.data) ~= "table" then
			toast("resposta inválida da API")
			return
		end
		local livres = {}
		for _, s in ipairs(dados.data) do
			if s.id ~= game.JobId and s.playing < s.maxPlayers then livres[#livres + 1] = s.id end
		end
		if #livres == 0 then toast("nenhum servidor livre"); return end
		TeleportService:TeleportToPlaceInstance(game.PlaceId, livres[math.random(#livres)], LocalPlayer)
	end)
end

--------------------------------------------------------------
--  limpeza (re-execução descarrega a versão anterior)
--------------------------------------------------------------
local function limpar()
	encerrado = true
	Motor.encerrado = true
	rotaAtiva = false
	for nome, fn in pairs(Motor.Toggles) do
		if Flags[nome] then pcall(fn, false) end
	end
	Hook.Ativo = false
	Hook.Alvo = nil
	Hook.Parede = false
	for _, c in ipairs(conns) do pcall(function() c:Disconnect() end) end
	for p in pairs(esp) do removerESP(p) end
	limparItens()
	pcall(function() telaESP:Destroy() end)
	pcall(function() pasta:Destroy() end)
	if UI.gui then pcall(function() UI.gui:Destroy() end) end
end
genv.JefinCleanup = limpar
end

--==============================================================
--  DADOS — abas e botões (só rótulos, nada funcional)
--  tipo: "botao" | "toggle"
--  chip: texto do atalho mostrado na direita (opcional)
--==============================================================
local ABAS = {
	{
		nome = "Combate",
		desc = "Mira, dano e vantagem em combate",
		itens = {
			{ tipo = "toggle", nome = "Aimbot",        desc = "Mira automática no alvo mais próximo" },
			{ tipo = "toggle", nome = "Silent Aim",    desc = "Acerta sem mover a câmera" },
			{ tipo = "toggle", nome = "Bala Mágica",   desc = "Disparos atravessam paredes (depende do jogo)" },
			{ tipo = "toggle", nome = "Triggerbot",    desc = "Atira ao cruzar a mira no inimigo" },
			{ tipo = "toggle", nome = "No Recoil",     desc = "Remove o recuo vertical das armas" },
			{ tipo = "toggle", nome = "No Spread",     desc = "Agrupa 100% dos disparos" },
			{ tipo = "toggle", nome = "Hitbox Expander", desc = "Aumenta a área de acerto" },
			{ tipo = "botao",  nome = "Auto Revistar",  desc = "Revista corpos próximos automaticamente" },
			{ tipo = "botao",  nome = "Roubar Inventário", desc = "Preenche os slots com os itens da lista" },
		},
	},
	{
		nome = "Visual",
		desc = "ESP, marcações e ajustes de cena",
		itens = {
			{ tipo = "toggle", nome = "ESP Jogadores",  desc = "Caixa + nome + vida através das paredes" },
			{ tipo = "toggle", nome = "ESP Itens",      desc = "Destaca armas e itens no mapa" },
			{ tipo = "toggle", nome = "Tracers",        desc = "Linhas ligando a mira ao alvo" },
			{ tipo = "toggle", nome = "Chams",          desc = "Coloriza o corpo dos jogadores" },
			{ tipo = "toggle", nome = "Full Bright",    desc = "Remove a escuridão do mapa" },
			{ tipo = "toggle", nome = "Noclip",         desc = "Atravessa paredes e objetos" },
		},
	},
	{
		nome = "Configs",
		desc = "Ajustes finos de mira, ESP e hitbox",
		itens = {
			{ tipo = "botao",  nome = "Cor do ESP",     desc = "Escolhe a cor usada nas marcações" },
			{ tipo = "botao",  nome = "Distância Máxima", desc = "Define o alcance do desenho do ESP" },
			{ tipo = "botao",  nome = "FOV do Aimbot",  desc = "Raio do círculo de mira (pixels)" },
			{ tipo = "botao",  nome = "Suavização do Aimbot", desc = "Quão seco ou suave o aimbot trava" },
			{ tipo = "botao",  nome = "Delay do Triggerbot", desc = "Tempo entre um disparo e outro (ms)" },
			{ tipo = "botao",  nome = "Tamanho da Hitbox", desc = "Define o tamanho do Hitbox Expander" },
			{ tipo = "botao",  nome = "Parte do Corpo (Aimbot)", desc = "Onde o aimbot e o silent miram" },
			{ tipo = "toggle", nome = "Ignorar Time",   desc = "Aimbot/ESP também marcam quem é do seu time" },
			{ tipo = "toggle", nome = "Só Mirar Visível", desc = "Aimbot ignora quem está atrás de parede" },
		},
	},
	{
		nome = "Movimento",
		desc = "Deslocamento, voo e física",
		itens = {
			{ tipo = "toggle", nome = "Fly",            desc = "Voo livre com velocidade ajustável" },
			{ tipo = "toggle", nome = "Bypass de Queda", desc = "Anula o dano de queda" },
			{ tipo = "toggle", nome = "Speed Hack",     desc = "Aumenta a velocidade de caminhada" },
			{ tipo = "toggle", nome = "Infinite Jump",  desc = "Pulo ilimitado" },
			{ tipo = "toggle", nome = "Bunny Hop",      desc = "Pulo automático ao pousar" },
			{ tipo = "botao",  nome = "Velocidade do Fly", desc = "Slider de 10 a 190" },
			{ tipo = "botao",  nome = "Velocidade do Speed", desc = "Slider de 10 a 190" },
		},
	},
	{
		nome = "Teleportes",
		desc = "Locomoção rápida pelo mapa",
		itens = {
			{ tipo = "botao",  nome = "Ir para Base",   desc = "Teleporta para o SpawnLocation" },
			{ tipo = "botao",  nome = "Teleportar até Jogador", desc = "Vai até o alvo digitado" },
			{ tipo = "botao",  nome = "Espectar Jogador", desc = "Assume a câmera do alvo" },
			{ tipo = "botao",  nome = "Voltar Câmera",  desc = "Retorna a câmera para você" },
			{ tipo = "botao",  nome = "Salvar Posição", desc = "Guarda o ponto atual" },
			{ tipo = "botao",  nome = "Ir para Salvo",  desc = "Volta ao ponto guardado" },
		},
	},
	{
		nome = "Farm",
		desc = "Rotinas automáticas e empregos",
		itens = {
			{ tipo = "botao",  nome = "Entrar em Emprego", desc = "Abre a lista de trabalhos" },
			{ tipo = "toggle", nome = "Farm Gari",      desc = "Recolhe os lixos da rota" },
			{ tipo = "toggle", nome = "Farm Minerador", desc = "Minera automaticamente" },
			{ tipo = "toggle", nome = "Auto Coletar",   desc = "Pega itens no chão por perto" },
			{ tipo = "botao",  nome = "Rota Personalizada", desc = "Define pontos do trajeto" },
		},
	},
	{
		nome = "Outros",
		desc = "Utilidades, servidor e informações",
		itens = {
			{ tipo = "botao",  nome = "Lista de Staffs", desc = "Mostra os staffs online" },
			{ tipo = "botao",  nome = "Entrar em Carro", desc = "Ocupa o veículo mais próximo" },
			{ tipo = "toggle", nome = "Auto Rejoin",    desc = "Reentra no servidor se cair" },
			{ tipo = "toggle", nome = "Anti AFK",       desc = "Impede a expulsão por inatividade" },
			{ tipo = "botao",  nome = "Copiar Job ID",  desc = "Copia o ID do servidor atual" },
			{ tipo = "botao",  nome = "Servidor Hop",   desc = "Troca para outro servidor" },
		},
	},
}

--==============================================================
--  SISTEMA DE BINDS (começa vazio — você define cada atalho)
--==============================================================
local IndiceItens = {}
for _, aba in ipairs(ABAS) do
	for _, item in ipairs(aba.itens) do
		IndiceItens[item.nome] = item
	end
end

local Binds = genv.JefinBinds or {}
genv.JefinBinds = Binds
local BindBotoes = {}
local escutando = nil

local ALIAS_TECLA = {
	LeftAlt = "ALT", RightAlt = "ALT",
	LeftControl = "CTRL", RightControl = "CTRL",
	LeftShift = "SHIFT", RightShift = "SHIFT",
	Space = "SPACE", Return = "ENTER", Tab = "TAB",
	CapsLock = "CAPS", Backspace = "BKSP",
}
local function nomeTecla(kc)
	if not kc then return "bind" end
	return ALIAS_TECLA[kc.Name] or kc.Name:upper()
end

local function pararEscuta()
	if escutando and BindBotoes[escutando] then
		local kc = Binds[escutando]
		BindBotoes[escutando].Text = nomeTecla(kc)
		BindBotoes[escutando].TextColor3 = kc and T.Accent or T.TextoFraco
	end
	escutando = nil
end

local function limparOutrosBinds(kc, exceto)
	for nome, k in pairs(Binds) do
		if k == kc and nome ~= exceto then
			Binds[nome] = nil
			if BindBotoes[nome] then
				BindBotoes[nome].Text = "bind"
				BindBotoes[nome].TextColor3 = T.TextoFraco
			end
		end
	end
end

local function iniciarEscutaBind(nome, botaoRef)
	if escutando == nome then pararEscuta(); return end
	pararEscuta()
	escutando = nome
	botaoRef.Text = "..."
	botaoRef.TextColor3 = T.Accent
end

local function limparBind(nome)
	Binds[nome] = nil
	if BindBotoes[nome] then
		BindBotoes[nome].Text = "bind"
		BindBotoes[nome].TextColor3 = T.TextoFraco
	end
end

UserInputService.InputBegan:Connect(function(input, processado)
	if Motor.encerrado then return end
	if input.UserInputType ~= Enum.UserInputType.Keyboard then return end
	if input.KeyCode == Enum.KeyCode.Unknown then return end

	if escutando then
		if input.KeyCode == Enum.KeyCode.Escape then
			pararEscuta()
			return
		end
		limparOutrosBinds(input.KeyCode, escutando)
		Binds[escutando] = input.KeyCode
		pararEscuta()
		return
	end

	if processado or UserInputService:GetFocusedTextBox() ~= nil then return end
	for nome, kc in pairs(Binds) do
		if kc == input.KeyCode then
			local item = IndiceItens[nome]
			if item then
				if item.tipo == "toggle" then
					UI.alternar(nome)
				else
					local fn = Motor.Botoes[nome]
					if fn then
						local ok, err = pcall(fn)
						if not ok then
							warn("[Jefin] " .. nome .. ": " .. tostring(err))
							UI.toast(nome .. ": erro")
						end
					end
				end
			end
			break
		end
	end
end)

--==============================================================
--  JANELA PRINCIPAL
--==============================================================
local NOME_UI = "JefinMenu"
local antigo = GUI_PARENT:FindFirstChild(NOME_UI)
if antigo then antigo:Destroy() end

local LARG, ALT = 580, 400
local SIDEBAR_W  = 152
local TOPBAR_H   = 46

local gui = new("ScreenGui", {
	Name = NOME_UI,
	ResetOnSpawn = false,
	ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
	Parent = GUI_PARENT,
})

-- sombra
local sombra = new("Frame", {
	Name = "Sombra",
	Size = UDim2.new(0, LARG + 10, 0, ALT + 10),
	Position = UDim2.new(0.5, -(LARG + 10) / 2, 0.5, -(ALT + 10) / 2 + 4),
	BackgroundColor3 = Color3.new(0, 0, 0),
	BackgroundTransparency = 0.55,
	BorderSizePixel = 0,
	ZIndex = 0,
	Parent = gui,
})
canto(sombra, 18)

local janela = new("Frame", {
	Name = "Janela",
	Size = UDim2.new(0, LARG, 0, ALT),
	Position = UDim2.new(0.5, -LARG / 2, 0.5, -ALT / 2),
	BackgroundColor3 = T.Fundo,
	BorderSizePixel = 0,
	ClipsDescendants = true,
	Active = true,
	ZIndex = 1,
	Parent = gui,
})
canto(janela, 14)
borda(janela, T.Borda, 1)

--==============================================================
--  DRAG
--==============================================================
local function tornarArrastavel(frame, alca)
	alca = alca or frame
	local arrastando, inicio, posBase = false, nil, nil

	alca.InputBegan:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1
		or input.UserInputType == Enum.UserInputType.Touch then
			arrastando = true
			inicio = input.Position
			posBase = frame.Position
		end
	end)
	UserInputService.InputChanged:Connect(function(input)
		if not arrastando then return end
		if input.UserInputType == Enum.UserInputType.MouseMovement
		or input.UserInputType == Enum.UserInputType.Touch then
			local d = input.Position - inicio
			frame.Position = UDim2.new(
				posBase.X.Scale, posBase.X.Offset + d.X,
				posBase.Y.Scale, posBase.Y.Offset + d.Y
			)
			sombra.Position = UDim2.new(
				posBase.X.Scale, posBase.X.Offset + d.X - 5,
				posBase.Y.Scale, posBase.Y.Offset + d.Y + 4
			)
		end
	end)
	UserInputService.InputEnded:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1
		or input.UserInputType == Enum.UserInputType.Touch then
			arrastando = false
		end
	end)
end

--==============================================================
--  TOPBAR
--==============================================================
local topbar = new("Frame", {
	Name = "Topbar",
	Size = UDim2.new(1, 0, 0, TOPBAR_H),
	BackgroundColor3 = Color3.fromRGB(21, 21, 24),
	BorderSizePixel = 0,
	ZIndex = 2,
	Parent = janela,
})
new("UIGradient", {
	Color = ColorSequence.new({
		ColorSequenceKeypoint.new(0, Color3.fromRGB(48, 20, 22)),
		ColorSequenceKeypoint.new(1, Color3.fromRGB(21, 21, 24)),
	}),
	Rotation = 0,
	Parent = topbar,
})
new("Frame", {
	Name = "LinhaAccent",
	Size = UDim2.new(1, 0, 0, 2),
	Position = UDim2.new(0, 0, 1, -2),
	BackgroundColor3 = T.Accent,
	BorderSizePixel = 0,
	ZIndex = 3,
	Parent = topbar,
})
new("UIGradient", {
	Transparency = NumberSequence.new({
		NumberSequenceKeypoint.new(0, 0),
		NumberSequenceKeypoint.new(0.55, 0.35),
		NumberSequenceKeypoint.new(1, 1),
	}),
	Parent = topbar.LinhaAccent,
})

local logo = new("Frame", {
	Name = "Logo",
	Size = UDim2.new(0, 28, 0, 28),
	Position = UDim2.new(0, 16, 0.5, -14),
	BackgroundColor3 = T.Accent,
	BorderSizePixel = 0,
	ZIndex = 3,
	Parent = topbar,
})
canto(logo, 8)
new("TextLabel", {
	Size = UDim2.new(1, 0, 1, 0),
	BackgroundTransparency = 1,
	Text = "J",
	Font = FONTE_BOLD,
	TextSize = 17,
	TextColor3 = Color3.new(1, 1, 1),
	ZIndex = 4,
	Parent = logo,
})

new("TextLabel", {
	Name = "Titulo",
	Size = UDim2.new(0, 260, 0, 18),
	Position = UDim2.new(0, 54, 0, 8),
	BackgroundTransparency = 1,
	Text = "JEFIN",
	Font = FONTE_BOLD,
	TextSize = 15,
	TextColor3 = T.Texto,
	TextXAlignment = Enum.TextXAlignment.Left,
	ZIndex = 3,
	Parent = topbar,
})

new("TextLabel", {
	Name = "Subtitulo",
	Size = UDim2.new(0, 260, 0, 14),
	Position = UDim2.new(0, 54, 0, 24),
	BackgroundTransparency = 1,
	Text = "painel de utilidades",
	Font = FONTE,
	TextSize = 11,
	TextColor3 = T.TextoFraco,
	TextXAlignment = Enum.TextXAlignment.Left,
	ZIndex = 3,
	Parent = topbar,
})

-- status
local statusBox = new("Frame", {
	Name = "Status",
	Size = UDim2.new(0, 96, 0, 24),
	Position = UDim2.new(1, -196, 0.5, -12),
	BackgroundColor3 = Color3.fromRGB(30, 30, 34),
	BorderSizePixel = 0,
	ZIndex = 3,
	Parent = topbar,
})
canto(statusBox, 7)

local statusDot = new("Frame", {
	Name = "Dot",
	Size = UDim2.new(0, 7, 0, 7),
	Position = UDim2.new(0, 9, 0.5, -3.5),
	BackgroundColor3 = T.Verde,
	BorderSizePixel = 0,
	ZIndex = 4,
	Parent = statusBox,
})
canto(statusDot, 4)
new("TextLabel", {
	Size = UDim2.new(1, -22, 1, 0),
	Position = UDim2.new(0, 22, 0, 0),
	BackgroundTransparency = 1,
	Text = "conectado",
	Font = FONTE_MED,
	TextSize = 11,
	TextColor3 = T.TextoFraco,
	TextXAlignment = Enum.TextXAlignment.Left,
	ZIndex = 4,
	Parent = statusBox,
})

-- botões da topbar
local function botaoTopbar(texto, xOffset, hoverCor)
	local btn = new("TextButton", {
		Size = UDim2.new(0, 26, 0, 26),
		Position = UDim2.new(1, xOffset, 0.5, -13),
		BackgroundColor3 = Color3.fromRGB(32, 32, 37),
		Text = texto,
		Font = FONTE_BOLD,
		TextSize = 14,
		TextColor3 = T.TextoFraco,
		BorderSizePixel = 0,
		AutoButtonColor = false,
		ZIndex = 3,
		Parent = topbar,
	})
	canto(btn, 7)
	btn.MouseEnter:Connect(function()
		tween(btn, 0.15, { BackgroundColor3 = hoverCor, TextColor3 = Color3.new(1, 1, 1) })
	end)
	btn.MouseLeave:Connect(function()
		tween(btn, 0.15, { BackgroundColor3 = Color3.fromRGB(32, 32, 37), TextColor3 = T.TextoFraco })
	end)
	return btn
end

local btnMinimizar = botaoTopbar("-", -74, Color3.fromRGB(52, 52, 60))
local btnFechar    = botaoTopbar("X", -42, T.Accent)

tornarArrastavel(janela, topbar)

--==============================================================
--  SIDEBAR
--==============================================================
local sidebar = new("Frame", {
	Name = "Sidebar",
	Size = UDim2.new(0, SIDEBAR_W, 1, -TOPBAR_H - 30),
	Position = UDim2.new(0, 0, 0, TOPBAR_H),
	BackgroundColor3 = T.Sidebar,
	BorderSizePixel = 0,
	ZIndex = 2,
	Parent = janela,
})

local listaAbas = new("ScrollingFrame", {
	Name = "ListaAbas",
	Size = UDim2.new(1, 0, 1, -10),
	Position = UDim2.new(0, 0, 0, 8),
	BackgroundTransparency = 1,
	BorderSizePixel = 0,
	ScrollBarThickness = 0,
	CanvasSize = UDim2.new(0, 0, 0, 0),
	AutomaticCanvasSize = Enum.AutomaticSize.Y,
	ScrollingDirection = Enum.ScrollingDirection.Y,
	Parent = sidebar,
})
new("UIListLayout", {
	Padding = UDim.new(0, 3),
	SortOrder = Enum.SortOrder.LayoutOrder,
	Parent = listaAbas,
})
new("UIPadding", {
	PaddingLeft = UDim.new(0, 10),
	PaddingRight = UDim.new(0, 10),
	Parent = listaAbas,
})

--==============================================================
--  CONTEÚDO
--==============================================================
local conteudo = new("Frame", {
	Name = "Conteudo",
	Size = UDim2.new(1, -SIDEBAR_W, 1, -TOPBAR_H - 30),
	Position = UDim2.new(0, SIDEBAR_W, 0, TOPBAR_H),
	BackgroundColor3 = T.Conteudo,
	BorderSizePixel = 0,
	ZIndex = 2,
	Parent = janela,
})

local header = new("Frame", {
	Name = "Header",
	Size = UDim2.new(1, 0, 0, 58),
	BackgroundTransparency = 1,
	ZIndex = 3,
	Parent = conteudo,
})

local headerTitulo = new("TextLabel", {
	Size = UDim2.new(1, -32, 0, 20),
	Position = UDim2.new(0, 18, 0, 12),
	BackgroundTransparency = 1,
	Text = "",
	Font = FONTE_BOLD,
	TextSize = 16,
	TextColor3 = T.Texto,
	TextXAlignment = Enum.TextXAlignment.Left,
	ZIndex = 4,
	Parent = header,
})

local headerDesc = new("TextLabel", {
	Size = UDim2.new(1, -32, 0, 16),
	Position = UDim2.new(0, 18, 0, 33),
	BackgroundTransparency = 1,
	Text = "",
	Font = FONTE,
	TextSize = 12,
	TextColor3 = T.TextoFraco,
	TextXAlignment = Enum.TextXAlignment.Left,
	ZIndex = 4,
	Parent = header,
})

new("Frame", {
	Name = "Divisor",
	Size = UDim2.new(1, -36, 0, 1),
	Position = UDim2.new(0, 18, 0, 58),
	BackgroundColor3 = T.Borda,
	BorderSizePixel = 0,
	ZIndex = 3,
	Parent = conteudo,
})

local scroll = new("ScrollingFrame", {
	Name = "Scroll",
	Size = UDim2.new(1, 0, 1, -59),
	Position = UDim2.new(0, 0, 0, 59),
	BackgroundTransparency = 1,
	BorderSizePixel = 0,
	ScrollBarThickness = 3,
	ScrollBarImageColor3 = T.Accent,
	ScrollBarImageTransparency = 0.45,
	CanvasSize = UDim2.new(0, 0, 0, 0),
	AutomaticCanvasSize = Enum.AutomaticSize.Y,
	ScrollingDirection = Enum.ScrollingDirection.Y,
	ZIndex = 3,
	Parent = conteudo,
})
new("UIListLayout", {
	Padding = UDim.new(0, 7),
	SortOrder = Enum.SortOrder.LayoutOrder,
	Parent = scroll,
})
new("UIPadding", {
	PaddingTop = UDim.new(0, 12),
	PaddingLeft = UDim.new(0, 18),
	PaddingRight = UDim.new(0, 18),
	PaddingBottom = UDim.new(0, 14),
	Parent = scroll,
})

--==============================================================
--  RODAPÉ
--==============================================================
local rodape = new("Frame", {
	Name = "Rodape",
	Size = UDim2.new(1, 0, 0, 30),
	Position = UDim2.new(0, 0, 1, -30),
	BackgroundColor3 = Color3.fromRGB(21, 21, 24),
	BorderSizePixel = 0,
	ZIndex = 2,
	Parent = janela,
})
new("Frame", {
	Size = UDim2.new(1, 0, 0, 1),
	BackgroundColor3 = T.Borda,
	BorderSizePixel = 0,
	ZIndex = 3,
	Parent = rodape,
})
new("TextLabel", {
	Size = UDim2.new(0, 300, 1, 0),
	Position = UDim2.new(0, 16, 0, 0),
	BackgroundTransparency = 1,
	Text = "Jefin Menu  •  v4.0",
	Font = FONTE,
	TextSize = 11,
	TextColor3 = T.TextoFraco,
	TextXAlignment = Enum.TextXAlignment.Left,
	ZIndex = 3,
	Parent = rodape,
})
new("TextLabel", {
	Size = UDim2.new(0, 200, 1, 0),
	Position = UDim2.new(1, -216, 0, 0),
	BackgroundTransparency = 1,
	Text = "clique em \"bind\" pra configurar atalhos",
	Font = FONTE,
	TextSize = 11,
	TextColor3 = Color3.fromRGB(96, 96, 106),
	TextXAlignment = Enum.TextXAlignment.Right,
	ZIndex = 3,
	Parent = rodape,
})

--==============================================================
--  TOAST / POPUPS
--==============================================================
UI.gui = gui

local toastId = 0
function UI.toast(texto)
	toastId += 1
	local id = toastId
	local antigoToast = gui:FindFirstChild("Toast")
	if antigoToast then antigoToast:Destroy() end
	local f = new("Frame", {
		Name = "Toast",
		Size = UDim2.new(0, 300, 0, 34),
		AnchorPoint = Vector2.new(0.5, 1),
		Position = UDim2.new(0.5, 0, 1, -30),
		BackgroundColor3 = T.Card,
		BorderSizePixel = 0,
		ZIndex = 50,
		Parent = gui,
	})
	canto(f, 9)
	borda(f, T.Accent, 1)
	new("TextLabel", {
		Size = UDim2.new(1, -16, 1, 0),
		Position = UDim2.new(0, 8, 0, 0),
		BackgroundTransparency = 1,
		Text = texto,
		Font = FONTE_MED,
		TextSize = 12,
		TextColor3 = T.Texto,
		TextTruncate = Enum.TextTruncate.AtEnd,
		ZIndex = 51,
		Parent = f,
	})
	task.delay(2.4, function()
		if id == toastId and f.Parent then f:Destroy() end
	end)
end

local popupAtual = nil

local function abrirPopup(titulo, altura)
	if popupAtual then popupAtual.fechar() end
	local conexoes = {}
	local H = altura + 52

	local fundoP = new("TextButton", {
		Size = UDim2.new(1, 0, 1, 0),
		BackgroundColor3 = Color3.new(0, 0, 0),
		BackgroundTransparency = 0.45,
		BorderSizePixel = 0,
		Text = "",
		AutoButtonColor = false,
		ZIndex = 20,
		Parent = janela,
	})
	local caixa = new("Frame", {
		Size = UDim2.new(0, 310, 0, H),
		Position = UDim2.new(0.5, -155, 0.5, -H / 2),
		BackgroundColor3 = T.Fundo,
		BorderSizePixel = 0,
		Active = true,
		ZIndex = 21,
		Parent = fundoP,
	})
	canto(caixa, 12)
	borda(caixa, T.Borda, 1)
	new("TextLabel", {
		Size = UDim2.new(1, -56, 0, 22),
		Position = UDim2.new(0, 14, 0, 10),
		BackgroundTransparency = 1,
		Text = titulo,
		Font = FONTE_BOLD,
		TextSize = 14,
		TextColor3 = T.Texto,
		TextXAlignment = Enum.TextXAlignment.Left,
		ZIndex = 22,
		Parent = caixa,
	})
	local x = new("TextButton", {
		Size = UDim2.new(0, 24, 0, 24),
		Position = UDim2.new(1, -34, 0, 9),
		BackgroundColor3 = Color3.fromRGB(32, 32, 37),
		Text = "X",
		Font = FONTE_BOLD,
		TextSize = 12,
		TextColor3 = T.TextoFraco,
		BorderSizePixel = 0,
		AutoButtonColor = false,
		ZIndex = 22,
		Parent = caixa,
	})
	canto(x, 6)
	local corpo = new("Frame", {
		Size = UDim2.new(1, -28, 1, -52),
		Position = UDim2.new(0, 14, 0, 42),
		BackgroundTransparency = 1,
		ZIndex = 22,
		Parent = caixa,
	})

	local pop = {}
	function pop.fechar()
		for i = #conexoes, 1, -1 do
			conexoes[i]:Disconnect()
			conexoes[i] = nil
		end
		if fundoP.Parent then fundoP:Destroy() end
		if popupAtual == pop then popupAtual = nil end
	end
	fundoP.MouseButton1Click:Connect(pop.fechar)
	x.MouseButton1Click:Connect(pop.fechar)
	popupAtual = pop
	return corpo, pop, conexoes
end

function UI.menu(titulo, opcoes)
	local altura = math.clamp(#opcoes * 36, 40, 230)
	local corpo, pop = abrirPopup(titulo, altura)
	local lista = new("ScrollingFrame", {
		Size = UDim2.new(1, 0, 1, 0),
		BackgroundTransparency = 1,
		BorderSizePixel = 0,
		ScrollBarThickness = 3,
		ScrollBarImageColor3 = T.Accent,
		CanvasSize = UDim2.new(0, 0, 0, 0),
		AutomaticCanvasSize = Enum.AutomaticSize.Y,
		ZIndex = 23,
		Parent = corpo,
	})
	new("UIListLayout", { Padding = UDim.new(0, 4), SortOrder = Enum.SortOrder.LayoutOrder, Parent = lista })

	if #opcoes == 0 then
		new("TextLabel", {
			Size = UDim2.new(1, 0, 0, 30),
			BackgroundTransparency = 1,
			Text = "nada para mostrar",
			Font = FONTE,
			TextSize = 12,
			TextColor3 = T.TextoFraco,
			ZIndex = 24,
			Parent = lista,
		})
	end

	for i, op in ipairs(opcoes) do
		local b = new("TextButton", {
			Size = UDim2.new(1, -6, 0, 32),
			BackgroundColor3 = T.Card,
			Text = op.texto,
			Font = FONTE_MED,
			TextSize = 12,
			TextColor3 = op.acao and T.Texto or T.TextoFraco,
			TextXAlignment = Enum.TextXAlignment.Left,
			TextTruncate = Enum.TextTruncate.AtEnd,
			AutoButtonColor = false,
			BorderSizePixel = 0,
			LayoutOrder = i,
			ZIndex = 24,
			Parent = lista,
		})
		canto(b, 8)
		new("UIPadding", { PaddingLeft = UDim.new(0, 10), PaddingRight = UDim.new(0, 10), Parent = b })
		if op.acao then
			b.MouseEnter:Connect(function() tween(b, 0.12, { BackgroundColor3 = T.CardHover }) end)
			b.MouseLeave:Connect(function() tween(b, 0.12, { BackgroundColor3 = T.Card }) end)
			b.MouseButton1Click:Connect(function()
				if not op.manter then pop.fechar() end
				local ok, novo = pcall(op.acao)
				if not ok then
					warn("[Jefin] " .. tostring(novo))
					UI.toast("erro ao executar")
				elseif op.manter and type(novo) == "string" then
					b.Text = novo
				end
			end)
		end
	end
end

function UI.slider(titulo, minimo, maximo, atual, aoMudar)
	local corpo, _pop, conexoes = abrirPopup(titulo, 64)
	local valor = new("TextLabel", {
		Size = UDim2.new(1, 0, 0, 22),
		BackgroundTransparency = 1,
		Text = tostring(atual),
		Font = FONTE_BOLD,
		TextSize = 16,
		TextColor3 = T.Accent,
		ZIndex = 23,
		Parent = corpo,
	})
	local trilho = new("Frame", {
		Size = UDim2.new(1, 0, 0, 6),
		Position = UDim2.new(0, 0, 0, 40),
		BackgroundColor3 = T.SwitchOff,
		BorderSizePixel = 0,
		ZIndex = 23,
		Parent = corpo,
	})
	canto(trilho, 3)
	local rel0 = math.clamp((atual - minimo) / (maximo - minimo), 0, 1)
	local preench = new("Frame", {
		Size = UDim2.new(rel0, 0, 1, 0),
		BackgroundColor3 = T.Accent,
		BorderSizePixel = 0,
		ZIndex = 24,
		Parent = trilho,
	})
	canto(preench, 3)
	local knob = new("Frame", {
		Size = UDim2.new(0, 14, 0, 14),
		Position = UDim2.new(rel0, -7, 0.5, -7),
		BackgroundColor3 = Color3.new(1, 1, 1),
		BorderSizePixel = 0,
		ZIndex = 25,
		Parent = trilho,
	})
	canto(knob, 7)
	local area = new("TextButton", {
		Size = UDim2.new(1, 0, 0, 28),
		Position = UDim2.new(0, 0, 0, 29),
		BackgroundTransparency = 1,
		Text = "",
		ZIndex = 26,
		Parent = corpo,
	})

	local arrastando = false
	local function aplicar(px)
		local rel = math.clamp((px - trilho.AbsolutePosition.X) / math.max(trilho.AbsoluteSize.X, 1), 0, 1)
		local v = math.floor(minimo + (maximo - minimo) * rel + 0.5)
		local relV = (v - minimo) / (maximo - minimo)
		preench.Size = UDim2.new(relV, 0, 1, 0)
		knob.Position = UDim2.new(relV, -7, 0.5, -7)
		valor.Text = tostring(v)
		aoMudar(v)
	end

	area.InputBegan:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1
		or input.UserInputType == Enum.UserInputType.Touch then
			arrastando = true
			aplicar(input.Position.X)
		end
	end)
	conexoes[#conexoes + 1] = UserInputService.InputChanged:Connect(function(input)
		if arrastando and (input.UserInputType == Enum.UserInputType.MouseMovement
		or input.UserInputType == Enum.UserInputType.Touch) then
			aplicar(input.Position.X)
		end
	end)
	conexoes[#conexoes + 1] = UserInputService.InputEnded:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1
		or input.UserInputType == Enum.UserInputType.Touch then
			arrastando = false
		end
	end)
end

local PALETA = {
	{ "vermelho", Color3.fromRGB(229, 57, 53) },
	{ "laranja",  Color3.fromRGB(255, 152, 0) },
	{ "amarelo",  Color3.fromRGB(255, 214, 10) },
	{ "verde",    Color3.fromRGB(76, 209, 122) },
	{ "ciano",    Color3.fromRGB(0, 200, 230) },
	{ "azul",     Color3.fromRGB(66, 133, 244) },
	{ "magenta",  Color3.fromRGB(224, 64, 251) },
	{ "branco",   Color3.fromRGB(240, 240, 245) },
}

function UI.cores(aoEscolher)
	local corpo, pop = abrirPopup("Cor do ESP", 78)
	local grade = new("Frame", {
		Size = UDim2.new(1, 0, 1, 0),
		BackgroundTransparency = 1,
		ZIndex = 23,
		Parent = corpo,
	})
	new("UIGridLayout", {
		CellSize = UDim2.new(0, 30, 0, 30),
		CellPadding = UDim2.new(0, 8, 0, 8),
		Parent = grade,
	})
	for _, par in ipairs(PALETA) do
		local b = new("TextButton", {
			BackgroundColor3 = par[2],
			Text = "",
			AutoButtonColor = false,
			BorderSizePixel = 0,
			ZIndex = 24,
			Parent = grade,
		})
		canto(b, 8)
		borda(b, T.Borda, 1)
		b.MouseButton1Click:Connect(function()
			aoEscolher(par[2])
			UI.toast("cor: " .. par[1])
			pop.fechar()
		end)
	end
end

--==============================================================
--  FÁBRICA DE CARD
--==============================================================
local function criarSwitch(parent, estadoInicial)
	local trilho = new("Frame", {
		Name = "Switch",
		Size = UDim2.new(0, 38, 0, 20),
		Position = UDim2.new(1, -54, 0.5, -10),
		BackgroundColor3 = estadoInicial and T.Accent or T.SwitchOff,
		BorderSizePixel = 0,
		ZIndex = 5,
		Parent = parent,
	})
	canto(trilho, 10)

	local knob = new("Frame", {
		Name = "Knob",
		Size = UDim2.new(0, 14, 0, 14),
		Position = estadoInicial and UDim2.new(1, -17, 0.5, -7) or UDim2.new(0, 3, 0.5, -7),
		BackgroundColor3 = Color3.new(1, 1, 1),
		BorderSizePixel = 0,
		ZIndex = 6,
		Parent = trilho,
	})
	canto(knob, 7)

	return trilho, knob
end

local Estado, Visuais = {}, {}

local function pintarToggle(nome, on)
	local v = Visuais[nome]
	if not v or not v.card.Parent then return end
	tween(v.trilho, 0.18, { BackgroundColor3 = on and T.Accent or T.SwitchOff })
	tween(v.knob, 0.18, {
		Position = on and UDim2.new(1, -17, 0.5, -7) or UDim2.new(0, 3, 0.5, -7),
	})
	tween(v.card, 0.18, { BackgroundColor3 = on and T.CardAtivo or T.Card })
end

local function definirToggle(nome, on)
	local fn = Motor.Toggles[nome]
	if fn then
		local ok, res = pcall(fn, on)
		if not ok then
			warn("[Jefin] " .. nome .. ": " .. tostring(res))
			UI.toast(nome .. ": erro")
			on = false
		elseif res == false then
			on = false
		end
	end
	Estado[nome] = on
	pintarToggle(nome, on)
end

function UI.alternar(nome)
	definirToggle(nome, not Estado[nome])
end

local function criarCard(item, ordem)
	local card = new("Frame", {
		Name = item.nome,
		Size = UDim2.new(1, 0, 0, 52),
		BackgroundColor3 = T.Card,
		BorderSizePixel = 0,
		LayoutOrder = ordem,
		ZIndex = 4,
		Parent = scroll,
	})
	canto(card, 10)
	local stroke = borda(card, T.Borda, 1)

	new("TextLabel", {
		Size = UDim2.new(1, -132, 0, 17),
		Position = UDim2.new(0, 14, 0, 8),
		BackgroundTransparency = 1,
		Text = item.nome,
		Font = FONTE_BOLD,
		TextSize = 13,
		TextColor3 = T.Texto,
		TextXAlignment = Enum.TextXAlignment.Left,
		ZIndex = 5,
		Parent = card,
	})

	new("TextLabel", {
		Size = UDim2.new(1, -132, 0, 15),
		Position = UDim2.new(0, 14, 0, 27),
		BackgroundTransparency = 1,
		Text = item.desc or "",
		Font = FONTE,
		TextSize = 11,
		TextColor3 = T.TextoFraco,
		TextXAlignment = Enum.TextXAlignment.Left,
		ZIndex = 5,
		Parent = card,
	})

	-- botão de bind (começa sem atalho — clique e aperte a tecla)
	local bindBtn = new("TextButton", {
		Size = UDim2.new(0, 46, 0, 18),
		Position = UDim2.new(1, -120, 0.5, -9),
		BackgroundColor3 = Color3.fromRGB(42, 42, 48),
		Text = nomeTecla(Binds[item.nome]),
		Font = FONTE_BOLD,
		TextSize = 10,
		TextColor3 = Binds[item.nome] and T.Accent or T.TextoFraco,
		AutoButtonColor = false,
		BorderSizePixel = 0,
		ZIndex = 8,
		Parent = card,
	})
	canto(bindBtn, 5)
	BindBotoes[item.nome] = bindBtn
	bindBtn.MouseButton1Click:Connect(function()
		iniciarEscutaBind(item.nome, bindBtn)
	end)
	bindBtn.MouseButton2Click:Connect(function()
		limparBind(item.nome)
	end)

	local botao = new("TextButton", {
		Size = UDim2.new(1, 0, 1, 0),
		BackgroundTransparency = 1,
		Text = "",
		AutoButtonColor = false,
		ZIndex = 7,
		Parent = card,
	})

	if item.tipo == "toggle" then
		local estado = Estado[item.nome] == true
		local trilho, knob = criarSwitch(card, estado)
		card.BackgroundColor3 = estado and T.CardAtivo or T.Card
		Visuais[item.nome] = { trilho = trilho, knob = knob, card = card }

		botao.MouseButton1Click:Connect(function()
			definirToggle(item.nome, not Estado[item.nome])
		end)
	else
		-- seta indicativa
		new("TextLabel", {
			Size = UDim2.new(0, 16, 0, 20),
			Position = UDim2.new(1, -34, 0.5, -10),
			BackgroundTransparency = 1,
			Text = ">",
			Font = FONTE_BOLD,
			TextSize = 14,
			TextColor3 = Color3.fromRGB(96, 96, 106),
			ZIndex = 5,
			Parent = card,
		})

		botao.MouseButton1Click:Connect(function()
			tween(card, 0.08, { BackgroundColor3 = T.CardHover })
			task.delay(0.12, function()
				tween(card, 0.2, { BackgroundColor3 = T.Card })
			end)
			local fn = Motor.Botoes[item.nome]
			if fn then
				local ok, err = pcall(fn)
				if not ok then
					warn("[Jefin] " .. item.nome .. ": " .. tostring(err))
					UI.toast(item.nome .. ": erro")
				end
			end
		end)
	end

	botao.MouseEnter:Connect(function()
		tween(card, 0.15, { BackgroundColor3 = T.CardHover })
		tween(stroke, 0.15, { Color = Color3.fromRGB(66, 66, 74) })
	end)
	botao.MouseLeave:Connect(function()
		local ativo = item.tipo == "toggle" and Estado[item.nome] == true
		tween(card, 0.15, { BackgroundColor3 = ativo and T.CardAtivo or T.Card })
		tween(stroke, 0.15, { Color = T.Borda })
	end)

	return card
end

--==============================================================
--  ABAS
--==============================================================
local botoesAba = {}
local abaAtual = nil

local function pintarAba(nome, ativo)
	local ref = botoesAba[nome]
	if not ref then return end
	tween(ref.fundo, 0.18, {
		BackgroundColor3 = ativo and Color3.fromRGB(38, 26, 28) or Color3.fromRGB(28, 28, 32),
	})
	tween(ref.barra, 0.18, {
		BackgroundTransparency = ativo and 0 or 1,
	})
	tween(ref.texto, 0.18, {
		TextColor3 = ativo and T.Texto or T.TextoFraco,
	})
end

local function mostrarAba(nome)
	if abaAtual == nome then return end
	abaAtual = nome

	for _, c in ipairs(scroll:GetChildren()) do
		if c:IsA("Frame") then c:Destroy() end
	end

	for _, aba in ipairs(ABAS) do
		pintarAba(aba.nome, aba.nome == nome)
		if aba.nome == nome then
			headerTitulo.Text = aba.nome
			headerDesc.Text = aba.desc
			for i, item in ipairs(aba.itens) do
				criarCard(item, i)
			end
		end
	end

	scroll.CanvasPosition = Vector2.new(0, 0)
end

for i, aba in ipairs(ABAS) do
	local item = new("TextButton", {
		Name = aba.nome,
		Size = UDim2.new(1, 0, 0, 36),
		BackgroundColor3 = Color3.fromRGB(28, 28, 32),
		Text = "",
		AutoButtonColor = false,
		BorderSizePixel = 0,
		LayoutOrder = i,
		ZIndex = 3,
		Parent = listaAbas,
	})
	canto(item, 8)

	local barra = new("Frame", {
		Name = "Barra",
		Size = UDim2.new(0, 3, 0, 18),
		Position = UDim2.new(0, 0, 0.5, -9),
		BackgroundColor3 = T.Accent,
		BackgroundTransparency = 1,
		BorderSizePixel = 0,
		ZIndex = 4,
		Parent = item,
	})
	canto(barra, 2)

	local texto = new("TextLabel", {
		Name = "Texto",
		Size = UDim2.new(1, -22, 1, 0),
		Position = UDim2.new(0, 14, 0, 0),
		BackgroundTransparency = 1,
		Text = aba.nome,
		Font = FONTE_MED,
		TextSize = 13,
		TextColor3 = T.TextoFraco,
		TextXAlignment = Enum.TextXAlignment.Left,
		ZIndex = 4,
		Parent = item,
	})

	botoesAba[aba.nome] = { fundo = item, barra = barra, texto = texto }

	item.MouseEnter:Connect(function()
		if abaAtual ~= aba.nome then
			tween(item, 0.15, { BackgroundColor3 = Color3.fromRGB(34, 34, 39) })
			tween(texto, 0.15, { TextColor3 = Color3.fromRGB(200, 200, 210) })
		end
	end)

	item.MouseLeave:Connect(function()
		if abaAtual ~= aba.nome then
			tween(item, 0.15, { BackgroundColor3 = Color3.fromRGB(28, 28, 32) })
			tween(texto, 0.15, { TextColor3 = T.TextoFraco })
		end
	end)

	item.MouseButton1Click:Connect(function()
		mostrarAba(aba.nome)
	end)
end

--==============================================================
--  MINIMIZAR / FECHAR  (só interface)
--==============================================================
local minimizado = false

btnMinimizar.MouseButton1Click:Connect(function()
	minimizado = not minimizado
	local altura = minimizado and TOPBAR_H or ALT

	tween(janela, 0.22, { Size = UDim2.new(0, LARG, 0, altura) }, Enum.EasingStyle.Quart)
	tween(sombra, 0.22, { Size = UDim2.new(0, LARG + 10, 0, altura + 10) }, Enum.EasingStyle.Quart)

	sidebar.Visible = not minimizado
	conteudo.Visible = not minimizado
	rodape.Visible = not minimizado
	statusBox.Visible = not minimizado

	btnMinimizar.Text = minimizado and "+" or "-"
end)

local reabrir = new("TextButton", {
	Name = "Reabrir",
	Size = UDim2.new(0, 46, 0, 46),
	Position = UDim2.new(0, 20, 0.5, -23),
	BackgroundColor3 = T.Accent,
	Text = "J",
	Font = FONTE_BOLD,
	TextSize = 20,
	TextColor3 = Color3.new(1, 1, 1),
	BorderSizePixel = 0,
	AutoButtonColor = false,
	Visible = false,
	ZIndex = 10,
	Parent = gui,
})
canto(reabrir, 14)
borda(reabrir, Color3.fromRGB(255, 90, 85), 1)
tornarArrastavel(reabrir)

reabrir.MouseEnter:Connect(function() tween(reabrir, 0.15, { BackgroundColor3 = T.AccentSuave }) end)
reabrir.MouseLeave:Connect(function() tween(reabrir, 0.15, { BackgroundColor3 = T.Accent }) end)
reabrir.MouseButton1Click:Connect(function()
	reabrir.Visible = false
	janela.Visible = true
	sombra.Visible = true
end)

btnFechar.MouseButton1Click:Connect(function()
	janela.Visible = false
	sombra.Visible = false
	reabrir.Visible = true
end)

--==============================================================
--  INÍCIO
--==============================================================
UI.gui = gui
mostrarAba(ABAS[1].nome)
print("[Jefin Menu] carregado — clique em bind pra configurar os atalhos.")
