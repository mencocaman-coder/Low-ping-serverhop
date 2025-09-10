--[[
  Teleporte Inteligente v3.0
  • Calibração inicial de ping e FPS
  • HUD com ping/FPS em tempo real
  • Busca servers “vazios” (≤ MAX_PLAYERS)
  • TeleportToPlaceInstance ou fallback aleatório
  • Exponential backoff + retry até MAX_ATTEMPTS
  Basta colar em StarterPlayerScripts.
]]

-- Serviços
local HttpService     = game:GetService("HttpService")
local TeleportService = game:GetService("TeleportService")
local Players         = game:GetService("Players")
local RunService      = game:GetService("RunService")

-- Referências
local player  = Players.LocalPlayer
local placeId = game.PlaceId

-- CONFIGURAÇÃO
local CALIBRATE_TIME   = 5     -- segundos de calibração
local SAMPLE_INTERVAL  = 0.1   -- intervalo de amostragem (s)
local MAX_PLAYERS      = 2     -- instâncias com até X jogadores
local PAGE_LIMIT       = 100   -- servidores por página na API
local MAX_ATTEMPTS     = 3     -- tentativas de teleporte
local PING_SAFETY_X    = 1.2   -- multiplica avgPing para threshold
local FPS_SAFETY_X     = 0.9   -- multiplica avgFps para threshold
local BACKOFF_BASE     = 2     -- base do backoff exponencial
local HUD_INTERVAL     = 1     -- atualização do HUD (s)

-- Estado
local pingSamples = {}
local fpsSamples  = {}
local attempts    = 0
local PING_THRESHOLD, FPS_THRESHOLD

-- 1) Calibração de ping e FPS
local function calibrate()
    local startTime = tick()
    local conn
    conn = RunService.Heartbeat:Connect(function(dt)
        if tick() - startTime > CALIBRATE_TIME then
            conn:Disconnect()
            return
        end
        fpsSamples  [#fpsSamples+1] = 1/dt
        pingSamples[#pingSamples+1] = player:GetNetworkPing()*1000
    end)

    repeat task.wait(SAMPLE_INTERVAL) 
    until tick() - startTime >= CALIBRATE_TIME

    local sumPing, sumFps = 0, 0
    for _, p in ipairs(pingSamples) do sumPing = sumPing + p end
    for _, f in ipairs(fpsSamples)  do sumFps  = sumFps  + f end

    local avgPing = sumPing/#pingSamples
    local avgFps  = sumFps/#fpsSamples

    PING_THRESHOLD = avgPing * PING_SAFETY_X
    FPS_THRESHOLD  = avgFps  * FPS_SAFETY_X

    return avgPing, avgFps
end

-- 2) HUD para player
local hud = {}
local function createHUD()
    local gui = Instance.new("ScreenGui", player:WaitForChild("PlayerGui"))
    gui.Name = "TeleportHUD"

    local frame = Instance.new("Frame", gui)
    frame.Size = UDim2.new(0, 240, 0, 60)
    frame.Position = UDim2.new(0, 10, 0, 10)
    frame.BackgroundTransparency = 0.3
    frame.BackgroundColor3 = Color3.new(0, 0, 0)
    frame.BorderSizePixel = 0

    local function newLabel(y)
        local lbl = Instance.new("TextLabel", frame)
        lbl.Size = UDim2.new(1, -10, 0, 20)
        lbl.Position = UDim2.new(0, 5, 0, y)
        lbl.BackgroundTransparency = 1
        lbl.Font = Enum.Font.SourceSans
        lbl.TextSize = 18
        lbl.TextColor3 = Color3.new(1, 1, 1)
        return lbl
    end

    hud.ping   = newLabel(0)
    hud.fps    = newLabel(20)
    hud.status = newLabel(40)
end

local function startHUD()
    spawn(function()
        while attempts < MAX_ATTEMPTS do
            local curPing = math.floor(player:GetNetworkPing()*1000 + 0.5)
            local lastFps = math.floor((fpsSamples[#fpsSamples] or 0) + 0.5)
            hud.ping.Text   = ("Ping: %d ms / Thr: %d"):format(curPing, math.floor(PING_THRESHOLD))
            hud.fps.Text    = ("FPS: %d / Thr: %d"):format(lastFps,  math.floor(FPS_THRESHOLD))
            hud.status.Text = ("Tentativa: %d/%d"):format(attempts, MAX_ATTEMPTS)
            task.wait(HUD_INTERVAL)
        end
    end)
end

-- 3) Busca servidores via API
local function fetchServers(cursor)
    local url = ("https://games.roblox.com/v1/games/%d/servers/Public?sortOrder=Asc&limit=%d")
        :format(placeId, PAGE_LIMIT)
    if cursor then url = url .. "&cursor=" .. cursor end

    local ok, response = pcall(HttpService.GetAsync, HttpService, url)
    if not ok then return {}, nil end

    local success, data = pcall(HttpService.JSONDecode, HttpService, response)
    if not success or type(data) ~= "table" then return {}, nil end

    return data.data or {}, data.nextPageCursor
end

local function findBestServer()
    local cursor
    local bestId, lowestCount = nil, math.huge

    repeat
        local list, nextCursor = fetchServers(cursor)
        for _, srv in ipairs(list) do
            local cnt = srv.playing or math.huge
            if cnt <= MAX_PLAYERS then
                return srv.id
            end
            if cnt < lowestCount then
                lowestCount, bestId = cnt, srv.id
            end
        end
        cursor = nextCursor
    until not cursor

    return bestId
end

-- 4) Teleporte inteligente com backoff
local function smartTeleport(attempt)
    if attempt > MAX_ATTEMPTS then
        hud.status.Text = "Processo concluído"
        return
    end

    -- Se FPS abaixo do limiar, espera backoff
    local lastFps = fpsSamples[#fpsSamples] or 0
    if lastFps < FPS_THRESHOLD then
        local waitTime = BACKOFF_BASE^(attempt-1)
        hud.status.Text = ("Aguardando %ds por FPS"):format(waitTime)
        task.wait(waitTime)
    end

    attempts = attempt
    hud.status.Text = ("Buscando instância (%d/%d)"):format(attempt, MAX_ATTEMPTS)
    local targetId = findBestServer()

    if targetId then
        hud.status.Text = "Teleportando para servidor "..targetId
        TeleportService:TeleportToPlaceInstance(placeId, targetId, player)
    else
        hud.status.Text = "Fallback: teleport aleatório"
        TeleportService:Teleport(placeId, player)
    end

    -- Após backoff + 1s, verifica ping e talvez repete
    task.delay(BACKOFF_BASE^(attempt-1) + 1, function()
        local newPing = player:GetNetworkPing()*1000
        if newPing < PING_THRESHOLD then
            smartTeleport(attempt + 1)
        else
            hud.status.Text = "Ping aceitável (“..math.floor(newPing)..” ms) — fim"
        end
    end)
end

-- Execução principal
local avgPing, avgFps = calibrate()
createHUD()
startHUD()
smartTeleport(1)
