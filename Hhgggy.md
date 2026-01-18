-- Serviços essenciais
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local LocalPlayer = Players.LocalPlayer
local PlayerGui = LocalPlayer.PlayerGui

-- Configurações do ESP OVERDRIVE (APENAS VOCÊ VÊ)
local CONFIG = {
    Assassino = {
        CorPrincipal = Color3.new(1, 0, 0),
        CorBrilho = Color3.new(1, 0.2, 0.2),
        Espessura = 5,
        Transparencia = 0.25,
        TamanhoExtra = Vector3.new(0.9, 0.9, 0.9),
        Texto = "[⚠️ ASSASSINO ⚠️] "
    },
    Xerife = {
        CorPrincipal = Color3.new(0, 0.5, 1),
        CorBrilho = Color3.new(0.2, 0.5, 1),
        Espessura = 4,
        Transparencia = 0.35,
        TamanhoExtra = Vector3.new(0.8, 0.8, 0.8),
        Texto = "[🛡️ XERIFE 🛡️] "
    },
    SemPapel = {RemoverESP = true}
}

local ESPsAtivos = {}

-- Remove ESP quando necessário
local function removerESP(jogadorAlvo)
    if ESPsAtivos[jogadorAlvo.UserId] then
        for _, elem in ipairs(ESPsAtivos[jogadorAlvo.UserId]) do
            pcall(function()
                if typeof(elem) == "RBXScriptConnection" then elem:Disconnect() else elem:Destroy() end
            end)
        end
        ESPsAtivos[jogadorAlvo.UserId] = nil
    end
end

-- Cria cubo que APENAS VOCÊ VÊ
local function criarCubo(personagemAlvo, configPapel)
    local root = personagemAlvo.HumanoidRootPart
    local tamanho = personagemAlvo:GetExtentsSize() + configPapel.TamanhoExtra

    -- Cubo só no seu PlayerGui (outros não enxergam)
    local cuboPrincipal = Instance.new("BoxHandleAdornment")
    cuboPrincipal.Name = "LocalESP_Cubo"
    cuboPrincipal.Adornee = root
    cuboPrincipal.Size = tamanho
    cuboPrincipal.Color3 = configPapel.CorPrincipal
    cuboPrincipal.Thickness = configPapel.Espessura
    cuboPrincipal.Transparency = configPapel.Transparencia
    cuboPrincipal.AlwaysOnTop = true
    cuboPrincipal.ZIndex = 25
    cuboPrincipal.Parent = PlayerGui -- Apenas no seu GUI!

    local cuboBrilho = Instance.new("BoxHandleAdornment")
    cuboBrilho.Name = "LocalESP_Brilho"
    cuboBrilho.Adornee = root
    cuboBrilho.Size = tamanho + Vector3.new(0.3, 0.3, 0.3)
    cuboBrilho.Color3 = configPapel.CorBrilho
    cuboBrilho.Thickness = 2
    cuboBrilho.Transparency = 0.6
    cuboBrilho.AlwaysOnTop = true
    cuboBrilho.ZIndex = 24
    cuboBrilho.Parent = PlayerGui -- Apenas no seu GUI!

    -- Pulsar só pra você ver
    local pulsar = RunService.RenderStepped:Connect(function()
        cuboBrilho.Transparency = math.sin(tick() * 4) * 0.15 + 0.45
        cuboBrilho.Size = tamanho + Vector3.new(0.3 + math.cos(tick() * 3) * 0.12, 0.3 + math.cos(tick() * 3) * 0.12, 0.3 + math.cos(tick() * 3) * 0.12)
    end)

    return {cuboPrincipal, cuboBrilho, pulsar}
end

-- Cria texto que APENAS VOCÊ VÊ
local function criarTexto(personagemAlvo, configPapel, nome)
    -- Texto só no seu PlayerGui
    local billboard = Instance.new("BillboardGui")
    billboard.Name = "LocalESP_Texto"
    billboard.Adornee = personagemAlvo.HumanoidRootPart
    billboard.Size = UDim2.new(0, 450, 0, 90)
    billboard.StudsOffset = Vector3.new(0, 5.5, 0)
    billboard.AlwaysOnTop = true
    billboard.Parent = PlayerGui -- Apenas no seu GUI!

    local texto = Instance.new("TextLabel")
    texto.Size = UDim2.new(1, 0, 1, 0)
    texto.BackgroundTransparency = 1
    texto.Text = configPapel.Texto .. nome
    texto.TextColor3 = configPapel.CorPrincipal
    texto.Font = Enum.Font.Bold
    texto.TextSize = 36
    texto.TextScaled = true
    texto.Parent = billboard

    local textoSombra = Instance.new("TextLabel")
    textoSombra.Size = UDim2.new(1, 0, 1, 0)
    textoSombra.BackgroundTransparency = 1
    textoSombra.Text = texto.Text
    textoSombra.TextColor3 = Color3.new(0, 0, 0)
    textoSombra.Position = UDim2.new(0, 2, 0, 2)
    textoSombra.Parent = billboard

    return {billboard}
end

-- Detecta papel correto
local function getPapelCorreto(jogadorAlvo)
    if not jogadorAlvo or jogadorAlvo == LocalPlayer then return "SemPapel" end
    local papel = jogadorAlvo:GetAttribute("Role") or (jogadorAlvo.Character and jogadorAlvo.Character.Role?.Value) or ""
    papel = string.upper(papel)
    return papel == "ASSASSIN" and "Assassino" or papel == "SHERIFF" and "Xerife" or "SemPapel"
end

-- Cria ESP só pra você ver
local function criarESP(jogadorAlvo, papelAtual)
    if jogadorAlvo == LocalPlayer or not CONFIG[papelAtual] then
        removerESP(jogadorAlvo)
        return
    end

    local personagemAlvo = jogadorAlvo.Character
    if not personagemAlvo or not personagemAlvo.HumanoidRootPart or personagemAlvo.Humanoid.Health <= 0 then return end

    removerESP(jogadorAlvo)
    local configPapel = CONFIG[papelAtual]

    local cubos = criarCubo(personagemAlvo, configPapel)
    local texto = criarTexto(personagemAlvo, configPapel, jogadorAlvo.Name)

    ESPsAtivos[jogadorAlvo.UserId] = {cubos[1], cubos[2], cubos[3], texto[1]}
end

-- Inicia sistema
local function iniciar()
    for _, jogador in ipairs(Players:GetPlayers()) do criarESP(jogador, getPapelCorreto(jogador)) end

    Players.PlayerAdded:Connect(function(jogador)
        jogador.CharacterAdded:Connect(function() wait(2) criarESP(jogador, getPapelCorreto(jogador)) end)
    end)

    RunService.Heartbeat:Connect(function()
        for _, jogador in ipairs(Players:GetPlayers()) do criarESP(jogador, getPapelCorreto(jogador)) end
    end)
end

wait(3.5)
iniciar()
