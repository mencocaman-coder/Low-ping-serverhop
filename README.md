--[[
  Teleporte Automático v1.3
  • Sem filtro de amigos
  • Prioriza servidores com baixa população
  • Teleporta automaticamente ao entrar
  Hugo
]]

local HttpService     = game:GetService("HttpService")
local TeleportService = game:GetService("TeleportService")
local Players         = game:GetService("Players")

local player  = Players.LocalPlayer
local placeId = game.PlaceId

-- CONFIGURAÇÃO
local MAX_PLAYERS = 2    -- servidores com até X jogadores
local PAGE_LIMIT  = 100  -- quantos servidores buscar por página

-- Função paginada para buscar servidores públicos
local function fetchServers(cursor)
    local url = string.format(
        "https://games.roblox.com/v1/games/%d/servers/Public?sortOrder=Asc&limit=%d",
        placeId, PAGE_LIMIT
    )
    if cursor then
        url = url .. "&cursor=" .. cursor
    end

    local ok, res = pcall(HttpService.GetAsync, HttpService, url)
    if not ok then
        warn("Erro ao buscar servidores:", res)
        return {}, nil
    end

    local data = HttpService:JSONDecode(res)
    return data.data or {}, data.nextPageCursor
end

-- Encontra o melhor servidor baseado apenas na população
local function findBestServer()
    local cursor
    local bestId, lowestCount = nil, math.huge

    repeat
        local list, nextCursor = fetchServers(cursor)
        for _, srv in ipairs(list) do
            local cnt = srv.playing
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

-- Fluxo principal: busca e teleporta
local function main()
    local targetId = findBestServer()
    if targetId then
        TeleportService:TeleportToPlaceInstance(placeId, targetId, player)
    else
        TeleportService:Teleport(placeId, player)
    end
end

-- Executa automaticamente ao entrar
main()
