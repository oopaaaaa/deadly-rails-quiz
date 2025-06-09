local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local TeleportService = game:GetService("TeleportService")
local HttpService = game:GetService("HttpService")

local player = Players.LocalPlayer
local gui = Instance.new("ScreenGui")
gui.Name = "AutoTeleportGui"
gui.Parent = player:WaitForChild("PlayerGui")

-- Frame base do GUI
local frame = Instance.new("Frame")
frame.Size = UDim2.new(0, 150, 0, 50)
frame.Position = UDim2.new(0.5, -75, 0.1, 0)
frame.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
frame.Parent = gui

-- Botão ON/OFF
local toggleButton = Instance.new("TextButton")
toggleButton.Size = UDim2.new(1, -10, 1, -10)
toggleButton.Position = UDim2.new(0, 5, 0, 5)
toggleButton.BackgroundColor3 = Color3.fromRGB(0, 170, 255)
toggleButton.TextColor3 = Color3.new(1,1,1)
toggleButton.Font = Enum.Font.SourceSansBold
toggleButton.TextSize = 24
toggleButton.Text = "ON"  -- Já começa ligado
toggleButton.Parent = frame

-- Variáveis para controle
local running = true
local teleportConnection = nil

-- Pega todas as partes TeleportPart2
local function getTeleportParts()
    local parts = {}
    for _, part in ipairs(workspace:GetDescendants()) do
        if part:IsA("BasePart") and part.Name == "TeleportPart2" then
            table.insert(parts, part)
        end
    end
    return parts
end

-- Teleporte contínuo
local function startTeleport()
    if teleportConnection then return end -- evita múltiplas conexões

    teleportConnection = RunService.Heartbeat:Connect(function()
        if not running then return end

        local character = player.Character
        if not character then return end
        local hrp = character:FindFirstChild("HumanoidRootPart")
        if not hrp then return end

        local parts = getTeleportParts()
        if #parts == 0 then return end

        local part = parts[math.random(1, #parts)]
        hrp.CFrame = part.CFrame + Vector3.new(0, 3, 0)

        wait(0.25)
    end)
end

local function stopTeleport()
    if teleportConnection then
        teleportConnection:Disconnect()
        teleportConnection = nil
    end
end

-- Alterna o estado ON/OFF ao clicar
toggleButton.MouseButton1Click:Connect(function()
    running = not running
    if running then
        toggleButton.Text = "ON"
        startTeleport()
    else
        toggleButton.Text = "OFF"
        stopTeleport()
    end
end)

-- Mantém rodando após morte/reset
player.CharacterAdded:Connect(function()
    if running then
        wait(1)
        startTeleport()
    end
end)

-- Função para pegar servidores públicos do jogo (retorna lista de servidores)
local function getServers(cursor)
    local placeId = game.PlaceId
    local url = "https://games.roblox.com/v1/games/"..placeId.."/servers/Public?sortOrder=Asc&limit=100"
    if cursor then
        url = url .. "&cursor=" .. cursor
    end
    local success, response = pcall(function()
        return HttpService:GetAsync(url)
    end)
    if success then
        return HttpService:JSONDecode(response)
    else
        warn("Falha ao obter servidores:", response)
        return nil
    end
end

-- Função para trocar para o servidor menos cheio
local function teleportToLeastPopulatedServer()
    local placeId = game.PlaceId
    local serversData = getServers(nil)
    if not serversData then return end

    local leastPopulated = nil
    local minPlayers = math.huge

    for _, server in ipairs(serversData.data) do
        if server.playing < server.maxPlayers and server.id ~= game.JobId then
            if server.playing < minPlayers then
                minPlayers = server.playing
                leastPopulated = server.id
            end
        end
    end

    if leastPopulated then
        TeleportService:TeleportToPlaceInstance(placeId, leastPopulated, player)
    else
        print("Nenhum servidor disponível para teleportar")
    end
end

-- Começa o teleporte automático
startTeleport()

-- A cada 5 minutos (300 segundos), troca para o servidor menos cheio
while true do
    wait(300)
    teleportToLeastPopulatedServer()
end
