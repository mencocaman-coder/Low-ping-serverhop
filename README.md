-- FPS Booster Ultimate v5.0
-- Créditos: “Sua mãe”, @PepsiMannumero1 & @Lr-Scripts

local Players           = game:GetService("Players")
local RunService        = game:GetService("RunService")
local Workspace         = game:GetService("Workspace")
local Lighting          = game:GetService("Lighting")
local CollectionService = game:GetService("CollectionService")

local player    = Players.LocalPlayer
local PlayerGui = player:WaitForChild("PlayerGui")

-- == CONFIGURAÇÃO ==
local BATCH_SIZE       = 20     -- quantos objetos otimizar por frame
local SAMPLE_SIZE      = 30     -- tamanho do buffer de FPS
local FPS_THRESHOLD    = 75     -- título para ativar booster
local LOBBY_DELAY      = 10     -- segundos antes de otimizar após spawn
local STREAM_WAIT      = 1      -- segundos para carregar todo o lobby
local CREDIT_TIME      = 1.4    -- segundos por crédito na tela

-- == 1) CRÉDITOS ==
local function showCredits()
    local texts = {
        "FPS Booster ativado!",
        "@PepsiMannumero1 & @Lr-Scripts",
        "Sua mãe"
    }
    for _, txt in ipairs(texts) do
        local gui = Instance.new("ScreenGui")
        gui.Name         = "FB_Credits"
        gui.ResetOnSpawn = false
        gui.Parent       = PlayerGui

        local frame = Instance.new("Frame", gui)
        frame.Size              = UDim2.new(0.4, 0, 0.1, 0)
        frame.Position          = UDim2.new(0.5, 0, 0.8, 0)
        frame.AnchorPoint       = Vector2.new(0.5, 0.5)
        frame.BackgroundColor3  = Color3.fromRGB(30, 30, 30)
        frame.BorderSizePixel   = 0
        Instance.new("UICorner", frame).CornerRadius = UDim.new(0.2, 0)

        local label = Instance.new("TextLabel", frame)
        label.Size                   = UDim2.new(1, 0, 1, 0)
        label.BackgroundTransparency = 1
        label.Font                   = Enum.Font.GothamBold
        label.TextScaled             = true
        label.TextWrapped            = true
        label.TextColor3             = Color3.fromRGB(0, 174, 255)
        label.Text                   = txt

        task.delay(CREDIT_TIME, function()
            gui:Destroy()
        end)
        task.wait(CREDIT_TIME)
    end
end
task.spawn(showCredits)

-- == 2) QUALIDADE GRÁFICA MÍNIMA ==
pcall(function()
    settings().Rendering.QualityLevel            = 1
    settings().Rendering.MeshPartDetailLevel     = Enum.MeshPartDetailLevel.Low
    settings().Rendering.EagerBulkExecution       = false
    settings().Rendering.InterpolationThrottling = true
end)

-- == 3) ILUMINAÇÃO BALANCEADA ==
-- ambiente suave para visibilidade
Lighting.Ambient           = Color3.fromRGB(80, 80, 80)
Lighting.OutdoorAmbient    = Color3.fromRGB(80, 80, 80)
Lighting.ColorShift_Bottom = Color3.fromRGB(0,   0,  0)
Lighting.ColorShift_Top    = Color3.fromRGB(0,   0,  0)
Lighting.FogStart          = 0
Lighting.FogEnd            = 1e5
Lighting.GlobalShadows     = false

-- desativa efeitos caros
for _, eff in ipairs(Lighting:GetDescendants()) do
    if eff:IsA("PostEffect") then
        eff.Enabled = false
        eff:GetPropertyChangedSignal("Enabled"):Connect(function()
            eff.Enabled = false
        end)
    end
end
Lighting.DescendantAdded:Connect(function(inst)
    if inst:IsA("PostEffect") then
        inst.Enabled = false
    end
end)

-- remove sombras e reduz brilho das luzes
for _, inst in ipairs(Workspace:GetDescendants()) do
    if inst:IsA("BasePart") then
        inst.CastShadow = false
    elseif inst:IsA("Light") then
        pcall(function()
            inst.Brightness = math.clamp((inst.Brightness or 1) * 0.6, 0.3, 1)
            inst.Range      = math.clamp((inst.Range      or 8) * 0.6, 4, 16)
        end)
    end
end
Workspace.DescendantAdded:Connect(function(inst)
    if inst:IsA("BasePart") then
        inst.CastShadow = false
    elseif inst:IsA("Light") then
        pcall(function()
            inst.Brightness = math.clamp((inst.Brightness or 1) * 0.6, 0.3, 1)
            inst.Range      = math.clamp((inst.Range      or 8) * 0.6, 4, 16)
        end)
    end
end)

-- luz de preenchimento discreta na câmera
local cam = workspace.CurrentCamera
if cam then
    local fill = Instance.new("PointLight", cam)
    fill.Name       = "FB_FillLight"
    fill.Range      = 60
    fill.Brightness = 0.5
    fill.Shadows    = false
end

-- == 4) FUNÇÃO DE OTIMIZAÇÃO ==
local function optimize(obj)
    if CollectionService:HasTag(obj, "FB_Optimized") then
        return
    end
    CollectionService:AddTag(obj, "FB_Optimized")

    pcall(function()
        if obj:IsA("ParticleEmitter")
        or obj:IsA("Trail")
        or obj:IsA("Smoke")
        or obj:IsA("Sparkles")
        or obj:IsA("Fire")
        or obj:IsA("Explosion") then
            obj.Enabled  = false
            obj.Rate     = obj.Rate     and 0 or nil
            obj.Lifetime = NumberRange.new(0)
        elseif obj:IsA("BasePart") then
            obj.Material    = Enum.Material.SmoothPlastic
            obj.CastShadow  = false
            obj.Reflectance = 0
        end
    end)
end

-- == 5) OTIMIZAÇÃO GLOBAL DO LOBBY ==
workspace.StreamingEnabled = false
task.wait(STREAM_WAIT)
for _, inst in ipairs(Workspace:GetDescendants()) do
    optimize(inst)
end
workspace.StreamingEnabled = true

-- == 6) QUEUE E RESET ==
local optimizeQueue = {}
local fpsBuffer     = {}
local boosterActive = false

local function enqueue(obj)
    optimizeQueue[#optimizeQueue + 1] = obj
end

local function resetState()
    optimizeQueue = {}
    fpsBuffer     = {}
    boosterActive = false
    task.delay(LOBBY_DELAY, function()
        boosterActive = false
    end)
end

player.CharacterAdded:Connect(resetState)
resetState()

-- == 7) HEARTBEAT & BOOSTER ==
RunService.Heartbeat:Connect(function(dt)
    if boosterActive then
        -- já ativo: continua processando fila
    elseif #fpsBuffer < SAMPLE_SIZE then
        -- enche buffer até SAMPLE_SIZE antes de checar
    else
        -- calcula média
        local sum = 0
        for _, v in ipairs(fpsBuffer) do sum += v end
        local avg = sum / #fpsBuffer
        if avg < FPS_THRESHOLD then
            boosterActive = true
            for _, o in ipairs(Workspace:GetDescendants()) do
                enqueue(o)
            end
        end
    end

    -- atualiza buffer de FPS
    fpsBuffer[#fpsBuffer + 1] = 1/dt
    if #fpsBuffer > SAMPLE_SIZE then
        table.remove(fpsBuffer, 1)
    end

    -- processa até BATCH_SIZE objetos
    for i = 1, BATCH_SIZE do
        local obj = table.remove(optimizeQueue, 1)
        if not obj then break end
        optimize(obj)
    end
end)

-- == 8) OTIMIZAÇÃO DINÂMICA ==
Workspace.DescendantAdded:Connect(function(obj)
    if boosterActive then
        enqueue(obj)
    end
end)
