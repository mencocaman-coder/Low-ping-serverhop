-- 🔁 Teleporte para servidor menos cheio (versão segura e compatível)
-- Funciona em qualquer executor que permita TeleportService

local TeleportService = game:GetService("TeleportService")
local Players         = game:GetService("Players")

local player   = Players.LocalPlayer
local placeId  = game.PlaceId

-- Teleporta para outro servidor aleatório do mesmo jogo
TeleportService:Teleport(placeId, player)# Low-ping-serverhop
