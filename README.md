--[[
  Teleporte Smart v1.6
  • Busca servidores com até MAX_PLAYERS
  • Fallback para teleport aleatório se HTTP falhar
  • Mede ping e repete até MAX_ATTEMPTS
  Hugo
]]

local HttpService     = game:GetService("HttpService")
local TeleportService = game:GetService("TeleportService")
local Players         = game:GetService("Players")

local player  = Players.LocalPlayer
local placeId = game.PlaceId

-- CONFIGURAÇÃO
local MAX_PLAYERS      = 2     -- servidores considerados “vazios”
local MAX_ATTEMPTS     = 3     -- tentativas máximas
local PING_THRESHOLD   = 150   -- ping aceitável (ms)
local PING_CHECK_DELAY = 8     -- espera antes de checar ping
local PAGE_LIMIT       = 100   -- servidores por página de API

-- ESTADO
local attempts    = 0
local httpEnabled = true

-- Paginador seguro para buscar servidores públicos
local function fetchServers(cursor)
    if not httpEnabled then
        return {}, nil
    end

    local url = string.format(
        "https://games.roblox.com/v1/games/%d/servers/Public?sortOrder=Asc&limit=%d",
        placeId, PAGE_LIMIT
    )
    if cursor then
        url = url .. "&cursor=" .. cursor
    end

    local ok, res = pcall(function()
        return HttpService:GetAsync(url)
    end)
    if not ok then
        warn("fetchServers falhou:", res)
        httpEnabled = false
        return {}, nil
    end

    local success, data = pcall(function()
        return HttpService:JSONDecode(res)
    end)
    if not success or type(data) ~= "table" then
        warn("JSONDecode falhou.")
        httpEnabled = false
        return {}, nil
    end

    return data.data or {}, data.nextPageCursor
end

-- Retorna ID do servidor com menos jogadores (preferência por ≤ MAX_PLAYERS)
local function findBestServer()
    if not httpEnabled then
        return nil
    end

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

-- Tenta teleporte inteligente ou aleatório conforme disponibilidade de HTTP
local function teleportToBest()
    if attempts >= MAX_ATTEMPTS then
        warn(("Limite de tentativas atingido (%d)."):format(attempts))
        return
    end

    attempts = attempts + 1
    local targetId = findBestServer()

    if targetId and httpEnabled then
        warn(("Teleportando para servidor %s (tentativa %d/%d)").format(
            targetId, attempts, MAX_ATTEMPTS
        ))
        TeleportService:TeleportToPlaceInstance(placeId, targetId, player)
    else
        warn(("Teleport aleatório (HTTP desabilitado ou sem servidor). Tentativa %d/%d").format(
            attempts, MAX_ATTEMPTS
        ))
        TeleportService:Teleport(placeId, player)
    end
end

-- Inicia o primeiro teleporte
teleportToBest()

-- Após DELAY, checa o ping e, se necessário, repete
task.delay(PING_CHECK_DELAY, function()
    local pingMs = math.floor(player:GetNetworkPing() * 1000 + 0.5)
    warn(("Ping atual: %d ms").format(pingMs))

    if pingMs > PING_THRESHOLD and attempts < MAX_ATTEMPTS then
        warn("Ping alto — tentando outra instância...")
        teleportToBest()
    else
        warn("Processo concluído.")
    end
end)
