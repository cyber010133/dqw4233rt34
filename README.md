local player = game.Players.LocalPlayer
local UIS = game:GetService("UserInputService")
local RS = game:GetService("RunService")
local Players = game:GetService("Players")
local Camera = workspace.CurrentCamera

local repo = 'https://raw.githubusercontent.com/violin-suzutsuki/LinoriaLib/main/'
local Library = loadstring(game:HttpGet(repo .. 'Library.lua'))()
local ThemeManager = loadstring(game:HttpGet(repo .. 'addons/ThemeManager.lua'))()
local SaveManager = loadstring(game:HttpGet(repo .. 'addons/SaveManager.lua'))()

local Window = Library:CreateWindow({Title = 'Infinity Menu - By Pixotinho & Polengo V0.2', Center = true, AutoShow = true, TabPadding = 8, MenuFadeTime = 0.2})

local Tabs = {
    Main = Window:AddTab('Main'),
    Combat = Window:AddTab('Combate'),
    PVP = Window:AddTab('PVP'),
    Movement = Window:AddTab('Movimento'),
    Player = Window:AddTab('Jogador'),
    Vehicle = Window:AddTab('Veiculos'),
    Troll = Window:AddTab('Troll'),
    OP = Window:AddTab('OP')
}

local connections, states, ESPObjects = {}, {}, {}
local ESPEnabled = {Box = false, Skeleton = false, Hologram = false, Name = false, Health = false}
local ESPColor, ESPDistance = Color3.fromRGB(255, 0, 0), 500
local AimbotSettings = {Enabled = false, Intensity = 1, ShowFOV = false, FOVSize = 100, LegitEnabled = false, LegitMode = "Fraca", TeamCheck = true, WallCheck = true}

local FOVCircle = Drawing.new("Circle")
FOVCircle.Thickness, FOVCircle.NumSides, FOVCircle.Radius, FOVCircle.Filled, FOVCircle.Visible = 2, 50, 100, false, false
FOVCircle.Color, FOVCircle.Transparency = Color3.fromRGB(255, 255, 255), 1

local VehicleSystem = {
    selectedPlayer = nil,
    pulledVehicles = {},
    usedVehicles = {},
    loopActive = false,
    maxDistance = 999999,
    cooldownTime = 60,
    boostSettings = {enabled = false, noclipEnabled = false, boostKey = Enum.KeyCode.E, brakeKey = Enum.KeyCode.Q, boostForce = 50}
}

local function disconnect(name) if connections[name] then connections[name]:Disconnect() connections[name] = nil end end
local function getCharacter() return player.Character or player.CharacterAdded:Wait() end
local function getHumanoid() local c = getCharacter() return c and c:FindFirstChildOfClass("Humanoid") end
local function getRootPart() local c = getCharacter() return c and c:FindFirstChild("HumanoidRootPart") end

local function ClearESP()
    for _, objects in pairs(ESPObjects) do
        for _, obj in pairs(objects) do
            if obj and obj.Remove then obj:Remove() elseif obj then pcall(function() obj:Destroy() end) end
        end
    end
    ESPObjects = {}
end

local function IsInDistance(p)
    if not player.Character or not player.Character:FindFirstChild("HumanoidRootPart") or not p.Character or not p.Character:FindFirstChild("HumanoidRootPart") then return false end
    return (player.Character.HumanoidRootPart.Position - p.Character.HumanoidRootPart.Position).Magnitude <= ESPDistance
end

local function CreateBoxESP(p)
    local char = p.Character
    if not char or not char:FindFirstChild("HumanoidRootPart") then return end
    local box = Drawing.new("Square")
    box.Visible, box.Color, box.Thickness, box.Transparency, box.Filled = false, ESPColor, 2, 1, false
    if not ESPObjects[p] then ESPObjects[p] = {} end
    table.insert(ESPObjects[p], box)
    local connection = RS.RenderStepped:Connect(function()
        if char and char.Parent and char:FindFirstChild("HumanoidRootPart") and char:FindFirstChild("Humanoid") and char.Humanoid.Health > 0 and IsInDistance(p) then
            local vector, onScreen = Camera:WorldToViewportPoint(char.HumanoidRootPart.Position)
            if onScreen then
                local headPos = Camera:WorldToViewportPoint(char.Head.Position + Vector3.new(0, 0.5, 0))
                local legPos = Camera:WorldToViewportPoint(char.HumanoidRootPart.Position - Vector3.new(0, 3, 0))
                local height, width = math.abs(headPos.Y - legPos.Y), math.abs(headPos.Y - legPos.Y) / 2
                box.Size, box.Position, box.Visible = Vector2.new(width, height), Vector2.new(vector.X - width / 2, vector.Y - height / 2), true
            else box.Visible = false end
        else box.Visible = false connection:Disconnect() end
    end)
    table.insert(ESPObjects[p], connection)
end

local function CreateNameESP(p)
    local char = p.Character
    if not char or not char:FindFirstChild("HumanoidRootPart") then return end
    local nameText = Drawing.new("Text")
    nameText.Visible, nameText.Color, nameText.Size, nameText.Center, nameText.Outline = false, ESPColor, 16, true, true
    nameText.Text = p.Name
    if not ESPObjects[p] then ESPObjects[p] = {} end
    table.insert(ESPObjects[p], nameText)
    local connection = RS.RenderStepped:Connect(function()
        if char and char.Parent and char:FindFirstChild("HumanoidRootPart") and char:FindFirstChild("Humanoid") and char.Humanoid.Health > 0 and IsInDistance(p) then
            local headPos, onScreen = Camera:WorldToViewportPoint(char.Head.Position + Vector3.new(0, 1, 0))
            if onScreen then
                nameText.Position = Vector2.new(headPos.X, headPos.Y)
                nameText.Visible = true
            else nameText.Visible = false end
        else nameText.Visible = false connection:Disconnect() end
    end)
    table.insert(ESPObjects[p], connection)
end

local function CreateHealthESP(p)
    local char = p.Character
    if not char or not char:FindFirstChild("HumanoidRootPart") or not char:FindFirstChild("Humanoid") then return end
    local healthText = Drawing.new("Text")
    healthText.Visible, healthText.Color, healthText.Size, healthText.Center, healthText.Outline = false, Color3.fromRGB(0, 255, 0), 14, true, true
    if not ESPObjects[p] then ESPObjects[p] = {} end
    table.insert(ESPObjects[p], healthText)
    local connection = RS.RenderStepped:Connect(function()
        if char and char.Parent and char:FindFirstChild("HumanoidRootPart") and char:FindFirstChild("Humanoid") and char.Humanoid.Health > 0 and IsInDistance(p) then
            local headPos, onScreen = Camera:WorldToViewportPoint(char.Head.Position + Vector3.new(0, 1.5, 0))
            if onScreen then
                local health = math.floor(char.Humanoid.Health)
                local maxHealth = math.floor(char.Humanoid.MaxHealth)
                healthText.Text = health .. "/" .. maxHealth
                local healthPercent = health / maxHealth
                healthText.Color = Color3.fromRGB(255 * (1 - healthPercent), 255 * healthPercent, 0)
                healthText.Position = Vector2.new(headPos.X, headPos.Y)
                healthText.Visible = true
            else healthText.Visible = false end
        else healthText.Visible = false connection:Disconnect() end
    end)
    table.insert(ESPObjects[p], connection)
end

local function CreateSkeletonESP(p)
    local char = p.Character
    if not char then return end
    local skeletonParts = {{"Head", "UpperTorso"}, {"UpperTorso", "LowerTorso"}, {"UpperTorso", "LeftUpperArm"}, {"LeftUpperArm", "LeftLowerArm"}, {"LeftLowerArm", "LeftHand"}, {"UpperTorso", "RightUpperArm"}, {"RightUpperArm", "RightLowerArm"}, {"RightLowerArm", "RightHand"}, {"LowerTorso", "LeftUpperLeg"}, {"LeftUpperLeg", "LeftLowerLeg"}, {"LeftLowerLeg", "LeftFoot"}, {"LowerTorso", "RightUpperLeg"}, {"RightUpperLeg", "RightLowerLeg"}, {"RightLowerLeg", "RightFoot"}}
    if not ESPObjects[p] then ESPObjects[p] = {} end
    local lines = {}
    for i, parts in pairs(skeletonParts) do
        local line = Drawing.new("Line")
        line.Visible, line.Color, line.Thickness, line.Transparency = false, ESPColor, 2, 1
        lines[i] = {line = line, part1 = parts[1], part2 = parts[2]}
        table.insert(ESPObjects[p], line)
    end
    local connection = RS.RenderStepped:Connect(function()
        if not char or not char.Parent or not char:FindFirstChild("Humanoid") or char.Humanoid.Health <= 0 then
            for _, data in pairs(lines) do data.line.Visible = false end
            connection:Disconnect() return
        end
        if not IsInDistance(p) then for _, data in pairs(lines) do data.line.Visible = false end return end
        for _, data in pairs(lines) do
            local part1, part2 = char:FindFirstChild(data.part1), char:FindFirstChild(data.part2)
            if part1 and part2 then
                local pos1, visible1 = Camera:WorldToViewportPoint(part1.Position)
                local pos2, visible2 = Camera:WorldToViewportPoint(part2.Position)
                if visible1 and visible2 then data.line.From, data.line.To, data.line.Visible = Vector2.new(pos1.X, pos1.Y), Vector2.new(pos2.X, pos2.Y), true
                else data.line.Visible = false end
            else data.line.Visible = false end
        end
    end)
    table.insert(ESPObjects[p], connection)
end

local function CreateHologramESP(p)
    local char = p.Character
    if not char or not char:FindFirstChild("HumanoidRootPart") then return end
    local highlight = Instance.new("Highlight")
    highlight.Name, highlight.Adornee, highlight.FillColor, highlight.OutlineColor = "HologramESP", char, ESPColor, ESPColor
    highlight.FillTransparency, highlight.OutlineTransparency, highlight.Parent = 0.5, 0, char
    if not ESPObjects[p] then ESPObjects[p] = {} end
    table.insert(ESPObjects[p], highlight)
    local connection = RS.Heartbeat:Connect(function()
        if not char or not char.Parent or not char:FindFirstChild("Humanoid") or char.Humanoid.Health <= 0 then highlight:Destroy() connection:Disconnect()
        elseif not IsInDistance(p) then highlight.Enabled = false else highlight.Enabled = true end
    end)
    table.insert(ESPObjects[p], connection)
end

local function UpdateESPForPlayer(p)
    if ESPObjects[p] then
        for _, obj in pairs(ESPObjects[p]) do
            if typeof(obj) == "RBXScriptConnection" then obj:Disconnect()
            elseif obj and obj.Remove then obj:Remove()
            elseif obj then pcall(function() obj:Destroy() end) end
        end
        ESPObjects[p] = {}
    end
    if p.Character then
        if ESPEnabled.Box then CreateBoxESP(p) end
        if ESPEnabled.Name then CreateNameESP(p) end
        if ESPEnabled.Health then CreateHealthESP(p) end
        if ESPEnabled.Skeleton then CreateSkeletonESP(p) end
        if ESPEnabled.Hologram then CreateHologramESP(p) end
    end
end

local function UpdateAllESP() for _, p in pairs(Players:GetPlayers()) do if p ~= player then UpdateESPForPlayer(p) end end end

local function CheckWall(targetPos)
    if not AimbotSettings.WallCheck then return true end
    local origin, direction = Camera.CFrame.Position, (targetPos - Camera.CFrame.Position).Unit * (targetPos - Camera.CFrame.Position).Magnitude
    local raycastParams = RaycastParams.new()
    raycastParams.FilterType, raycastParams.FilterDescendantsInstances = Enum.RaycastFilterType.Blacklist, {player.Character, Camera}
    local raycastResult = workspace:Raycast(origin, direction, raycastParams)
    if raycastResult then
        local hitCharacter = raycastResult.Instance:FindFirstAncestorOfClass("Model")
        if hitCharacter and hitCharacter:FindFirstChild("Humanoid") then return true end
        return false
    end
    return true
end

local function GetClosestPlayerToCursor()
    local closestPlayer, shortestDistance = nil, math.huge
    for _, p in pairs(Players:GetPlayers()) do
        if p ~= player and p.Character and p.Character:FindFirstChild("Head") and p.Character:FindFirstChild("Humanoid") and p.Character.Humanoid.Health > 0 then
            if AimbotSettings.TeamCheck and p.Team == player.Team then continue end
            local head = p.Character.Head
            local screenPos, onScreen = Camera:WorldToViewportPoint(head.Position)
            if onScreen then
                local mousePos = UIS:GetMouseLocation()
                local distance = (Vector2.new(screenPos.X, screenPos.Y) - mousePos).Magnitude
                if distance < AimbotSettings.FOVSize and distance < shortestDistance and CheckWall(head.Position) then
                    closestPlayer, shortestDistance = p, distance
                end
            end
        end
    end
    return closestPlayer
end

local function AimbotLoop()
    if AimbotSettings.Enabled and UIS:IsMouseButtonPressed(Enum.UserInputType.MouseButton2) then
        local target = GetClosestPlayerToCursor()
        if target and target.Character and target.Character:FindFirstChild("Head") then
            local targetPos, cameraCFrame = target.Character.Head.Position, Camera.CFrame
            Camera.CFrame = cameraCFrame:Lerp(CFrame.new(cameraCFrame.Position, targetPos), AimbotSettings.Intensity)
        end
    end
end

local function LegitAimbotLoop()
    if AimbotSettings.LegitEnabled and UIS:IsMouseButtonPressed(Enum.UserInputType.MouseButton2) then
        local target = GetClosestPlayerToCursor()
        if target and target.Character and target.Character:FindFirstChild("Head") then
            local targetPos, cameraCFrame = target.Character.Head.Position, Camera.CFrame
            local legitIntensity = 0.02
            if AimbotSettings.LegitMode == "Media" then legitIntensity = 0.06 elseif AimbotSettings.LegitMode == "Forte" then legitIntensity = 0.12 end
            Camera.CFrame = cameraCFrame:Lerp(CFrame.new(cameraCFrame.Position, targetPos), legitIntensity)
        end
    end
end

function VehicleSystem:TeleportVehicle(vehicle, targetCFrame)
    local vehicleParent = vehicle.Parent
    if not vehicleParent then return end
    for _, part in pairs(vehicleParent:GetDescendants()) do
        if part:IsA("BasePart") then
            part.Anchored = true
            part.AssemblyLinearVelocity = Vector3.new(0, 0, 0)
            part.AssemblyAngularVelocity = Vector3.new(0, 0, 0)
        end
    end
    local mainPart = vehicleParent:FindFirstChild("HumanoidRootPart") or vehicleParent.PrimaryPart or vehicleParent:FindFirstChildOfClass("Part") or vehicle
    if mainPart then
        for _, part in pairs(vehicleParent:GetDescendants()) do
            if part:IsA("BasePart") then
                local partOffset = mainPart.CFrame:ToObjectSpace(part.CFrame)
                part.CFrame = targetCFrame * partOffset
            end
        end
    end
    for _, part in pairs(vehicleParent:GetDescendants()) do
        if part:IsA("BasePart") then part.Anchored = false end
    end
end

function VehicleSystem:GetAllVehicles()
    local vehicles = {}
    for _, obj in ipairs(workspace:GetDescendants()) do
        if obj:IsA("VehicleSeat") and not obj.Occupant then table.insert(vehicles, obj) end
    end
    return vehicles
end

function VehicleSystem:FindClosestVehicle()
    local character = getCharacter()
    if not character or not character:FindFirstChild("HumanoidRootPart") then return nil end
    local myRoot = character.HumanoidRootPart
    local closestDistance = math.huge
    local closestVehicle = nil
    local currentTime = tick()
    for _, obj in ipairs(workspace:GetDescendants()) do
        if obj:IsA("VehicleSeat") and not obj.Occupant then
            local vehicleKey = obj:GetFullName()
            local canPull = not self.pulledVehicles[vehicleKey] or (currentTime - self.pulledVehicles[vehicleKey] >= self.cooldownTime)
            if canPull then
                local distance = (obj.Position - myRoot.Position).Magnitude
                if distance < closestDistance then
                    closestDistance = distance
                    closestVehicle = obj
                end
            end
        end
    end
    return closestVehicle
end

function VehicleSystem:PullVehicle(vehicle, targetCFrame)
    local character = getCharacter()
    if not character or not character:FindFirstChild("HumanoidRootPart") then return end
    local humanoid = getHumanoid()
    local root = character.HumanoidRootPart
    if humanoid and root and vehicle and not vehicle.Occupant then
        local vehicleKey = vehicle:GetFullName()
        self.pulledVehicles[vehicleKey] = tick()
        root.CFrame = vehicle.CFrame + Vector3.new(0, 3, 0)
        task.wait(0.2)
        vehicle:Sit(humanoid)
        self:TeleportVehicle(vehicle, targetCFrame)
        humanoid.Sit = false
    end
end

function VehicleSystem:CleanOldVehicles()
    task.spawn(function()
        while true do
            task.wait(10)
            local currentTime = tick()
            for key, time in pairs(self.pulledVehicles) do
                if currentTime - time >= self.cooldownTime then self.pulledVehicles[key] = nil end
            end
        end
    end)
end

VehicleSystem:CleanOldVehicles()

RS.RenderStepped:Connect(function()
    if AimbotSettings.ShowFOV then FOVCircle.Visible, FOVCircle.Radius, FOVCircle.Position = true, AimbotSettings.FOVSize, UIS:GetMouseLocation()
    else FOVCircle.Visible = false end
    AimbotLoop() LegitAimbotLoop()
end)

Players.PlayerAdded:Connect(function(p) p.CharacterAdded:Connect(function() wait(0.5) UpdateESPForPlayer(p) end) end)
for _, p in pairs(Players:GetPlayers()) do
    if p ~= player then
        if p.Character then UpdateESPForPlayer(p) end
        p.CharacterAdded:Connect(function() wait(0.5) UpdateESPForPlayer(p) end)
    end
end
Players.PlayerRemoving:Connect(function(p)
    if ESPObjects[p] then
        for _, obj in pairs(ESPObjects[p]) do
            if typeof(obj) == "RBXScriptConnection" then obj:Disconnect()
            elseif obj and obj.Remove then obj:Remove()
            elseif obj then pcall(function() obj:Destroy() end) end
        end
        ESPObjects[p] = nil
    end
end)

local MainGroupBox = Tabs.Main:AddLeftGroupbox('Informações')
MainGroupBox:AddLabel('MENU FEITO POR PIXOTINHO & POLENGO')
MainGroupBox:AddLabel('Infinity Menu - Universal Script V0.2')
MainGroupBox:AddLabel('Anti-Cheat: Sintonia | Ilha Bela | Mini City')

local MenuGroup = Tabs.Main:AddRightGroupbox('Controles do Menu')
MenuGroup:AddButton('Descarregar Menu', function() Library:Unload() end)
MenuGroup:AddLabel('Tecla do Menu'):AddKeyPicker('MenuKeybind', {Default = 'K', NoUI = true, Text = 'Menu keybind'})

local PVPAimbotBox = Tabs.PVP:AddLeftGroupbox('Aimbot')
PVPAimbotBox:AddToggle('Aimbot', {Text = 'Aimbot', Default = false, Callback = function(state) AimbotSettings.Enabled = state end})
PVPAimbotBox:AddSlider('AimbotIntensity', {Text = 'Intensidade', Default = 1, Min = 0, Max = 1, Rounding = 2, Compact = false, Callback = function(value) AimbotSettings.Intensity = value end})
PVPAimbotBox:AddToggle('ShowFOV', {Text = 'Mostrar FOV', Default = false, Callback = function(state) AimbotSettings.ShowFOV = state end})
PVPAimbotBox:AddSlider('FOVSize', {Text = 'Tamanho FOV', Default = 100, Min = 50, Max = 500, Rounding = 0, Compact = false, Callback = function(value) AimbotSettings.FOVSize = value end})
PVPAimbotBox:AddToggle('TeamCheck', {Text = 'Verificar Time', Default = true, Callback = function(state) AimbotSettings.TeamCheck = state end})
PVPAimbotBox:AddToggle('WallCheck', {Text = 'Verificar Parede', Default = true, Callback = function(state) AimbotSettings.WallCheck = state end})

local PVPLegitBox = Tabs.PVP:AddRightGroupbox('Aim Legit')
PVPLegitBox:AddToggle('LegitAimbot', {Text = 'Aim Legit', Default = false, Callback = function(state) AimbotSettings.LegitEnabled = state end})
PVPLegitBox:AddDropdown('LegitMode', {Values = {'Fraca', 'Media', 'Forte'}, Default = 1, Multi = false, Text = 'Modo Legit', Callback = function(value) AimbotSettings.LegitMode = value end})

local ESPBox = Tabs.PVP:AddLeftGroupbox('ESP')
ESPBox:AddToggle('ESPBox', {Text = 'ESP Box', Default = false, Callback = function(state) ESPEnabled.Box = state UpdateAllESP() end})
ESPBox:AddToggle('ESPName', {Text = 'ESP Nome', Default = false, Callback = function(state) ESPEnabled.Name = state UpdateAllESP() end})
ESPBox:AddToggle('ESPHealth', {Text = 'ESP Vida', Default = false, Callback = function(state) ESPEnabled.Health = state UpdateAllESP() end})
ESPBox:AddToggle('ESPSkeleton', {Text = 'ESP Skeleton', Default = false, Callback = function(state) ESPEnabled.Skeleton = state UpdateAllESP() end})
ESPBox:AddToggle('ESPHologram', {Text = 'ESP Holograma', Default = false, Callback = function(state) ESPEnabled.Hologram = state UpdateAllESP() end})
ESPBox:AddSlider('ESPDistance', {Text = 'Distância ESP', Default = 500, Min = 100, Max = 2000, Rounding = 0, Compact = false, Callback = function(value) ESPDistance = value end})

local CombatBox = Tabs.Combat:AddLeftGroupbox('Modificações de Armas')
CombatBox:AddToggle('InfiniteAmmo', {Text = 'Munição Infinita', Default = false, Callback = function(state)
    states.infAmmoEnabled = state
    if state then
        connections.infAmmo = RS.Heartbeat:Connect(function()
            if not states.infAmmoEnabled then return end
            for _, tool in ipairs(player.Backpack:GetChildren()) do
                if tool:IsA("Tool") and tool:FindFirstChild("Ammo") then tool.Ammo.Value = 999 end
            end
            local char = getCharacter()
            if char then for _, tool in ipairs(char:GetChildren()) do if tool:IsA("Tool") and tool:FindFirstChild("Ammo") then tool.Ammo.Value = 999 end end end
        end)
    else disconnect("infAmmo") end
end})

CombatBox:AddToggle('NoRecoil', {Text = 'Sem Recuo', Default = false, Callback = function(state)
    states.noRecoilEnabled = state
    if state then
        connections.noRecoil = RS.Heartbeat:Connect(function()
            if not states.noRecoilEnabled then return end
            for _, tool in ipairs(player.Backpack:GetChildren()) do
                if tool:IsA("Tool") then
                    if tool:FindFirstChild("Recoil") then tool.Recoil.Value = 0 end
                    if tool:FindFirstChild("RecoilControl") then tool.RecoilControl.Value = 0 end
                end
            end
        end)
    else disconnect("noRecoil") end
end})

CombatBox:AddToggle('NoSpread', {Text = 'Sem Dispersão', Default = false, Callback = function(state)
    states.noSpreadEnabled = state
    if state then
        connections.noSpread = RS.Heartbeat:Connect(function()
            if not states.noSpreadEnabled then return end
            for _, tool in ipairs(player.Backpack:GetChildren()) do
                if tool:IsA("Tool") and tool:FindFirstChild("Spread") then tool.Spread.Value = 0 end
            end
        end)
    else disconnect("noSpread") end
end})

local WeaponList = {}
local function UpdateWeaponList()
    WeaponList = {}
    for _, tool in pairs(game:GetDescendants()) do if tool:IsA("Tool") and tool.Parent.Parent ~= player then table.insert(WeaponList, tool.Name) end end
    return WeaponList
end

local WeaponBox = Tabs.Combat:AddRightGroupbox('Gerenciar Armas')
local WeaponDropdown = WeaponBox:AddDropdown('WeaponSelect', {Values = {}, Default = 1, Multi = false, Text = 'Selecionar Arma'})
WeaponBox:AddButton({Text = 'Atualizar Lista', Func = function()
    local weapons = UpdateWeaponList()
    Options.WeaponSelect:SetValues(weapons)
end})
WeaponBox:AddButton({Text = 'Pegar Arma Selecionada', Func = function()
    local selectedWeapon = Options.WeaponSelect.Value
    if selectedWeapon then
        for _, tool in pairs(game:GetDescendants()) do
            if tool:IsA("Tool") and tool.Name == selectedWeapon and tool.Parent.Parent ~= player then
                local cloned = tool:Clone()
                cloned.Parent = player.Backpack
                break
            end
        end
    end
end})
WeaponBox:AddButton({Text = 'Coletar Todas Armas', Func = function()
    for _, tool in pairs(game:GetDescendants()) do if tool:IsA("Tool") and tool.Parent.Parent ~= player then tool:Clone().Parent = player.Backpack end end
end})

local speedValue, jumpValue, flySpeed = 16, 50, 1

local MovementBox = Tabs.Movement:AddLeftGroupbox('Movimento')
MovementBox:AddSlider('SpeedSlider', {Text = 'Velocidade', Default = 16, Min = 16, Max = 200, Rounding = 0, Compact = false, Callback = function(value)
    speedValue = value
    if states.speedEnabled then local humanoid = getHumanoid() if humanoid then humanoid.WalkSpeed = value end end
end})

MovementBox:AddToggle('SpeedToggle', {Text = 'Velocidade Custom', Default = false, Callback = function(state)
    states.speedEnabled = state
    local humanoid = getHumanoid()
    if humanoid then
        if state then
            humanoid.WalkSpeed = speedValue
            connections.speedReset = humanoid:GetPropertyChangedSignal("WalkSpeed"):Connect(function()
                if states.speedEnabled and humanoid.WalkSpeed ~= speedValue then humanoid.WalkSpeed = speedValue end
            end)
        else humanoid.WalkSpeed = 16 disconnect("speedReset") end
    end
end})

MovementBox:AddSlider('JumpSlider', {Text = 'Poder de Pulo', Default = 50, Min = 50, Max = 500, Rounding = 0, Compact = false, Callback = function(value)
    jumpValue = value
    if states.jumpEnabled then local humanoid = getHumanoid() if humanoid then humanoid.JumpPower = value end end
end})

MovementBox:AddToggle('JumpToggle', {Text = 'Pulo Custom', Default = false, Callback = function(state)
    states.jumpEnabled = state
    local humanoid = getHumanoid()
    if humanoid then
        if state then
            humanoid.UseJumpPower, humanoid.JumpPower = true, jumpValue
            connections.jumpReset = humanoid:GetPropertyChangedSignal("JumpPower"):Connect(function()
                if states.jumpEnabled and humanoid.JumpPower ~= jumpValue then humanoid.JumpPower = jumpValue end
            end)
        else humanoid.JumpPower = 50 disconnect("jumpReset") end
    end
end})

local MovementBox2 = Tabs.Movement:AddRightGroupbox('Movimento Avançado')
MovementBox2:AddSlider('FlySlider', {Text = 'Velocidade de Voo', Default = 1, Min = 0.5, Max = 3, Rounding = 1, Compact = false, Callback = function(value) flySpeed = value end})

MovementBox2:AddToggle('FlyToggle', {Text = 'Fly (Undetected)', Default = false, Callback = function(state)
    states.flyEnabled = state
    local char, root = getCharacter(), getRootPart()
    if not char or not root then return end
    if state then
        local camera = workspace.CurrentCamera
        local bodyVel = Instance.new("BodyVelocity")
        bodyVel.Name = "FlyVel"
        bodyVel.MaxForce = Vector3.new(0, 0, 0)
        bodyVel.Velocity = Vector3.new(0, 0, 0)
        bodyVel.Parent = root
        connections.fly = RS.Heartbeat:Connect(function(delta)
            if not states.flyEnabled then return end
            local currentRoot = getRootPart()
            if not currentRoot or not currentRoot:FindFirstChild("FlyVel") then return end
            local moveDirection = Vector3.new()
            if UIS:IsKeyDown(Enum.KeyCode.W) then moveDirection = moveDirection + camera.CFrame.LookVector end
            if UIS:IsKeyDown(Enum.KeyCode.S) then moveDirection = moveDirection - camera.CFrame.LookVector end
            if UIS:IsKeyDown(Enum.KeyCode.A) then moveDirection = moveDirection - camera.CFrame.RightVector end
            if UIS:IsKeyDown(Enum.KeyCode.D) then moveDirection = moveDirection + camera.CFrame.RightVector end
            if UIS:IsKeyDown(Enum.KeyCode.Space) then moveDirection = moveDirection + Vector3.new(0, 1, 0) end
            if UIS:IsKeyDown(Enum.KeyCode.LeftShift) then moveDirection = moveDirection - Vector3.new(0, 1, 0) end
            if moveDirection.Magnitude > 0 then
                moveDirection = moveDirection.Unit
                bodyVel.MaxForce = Vector3.new(9e9, 9e9, 9e9)
                bodyVel.Velocity = moveDirection * (flySpeed * 50)
            else
                bodyVel.MaxForce = Vector3.new(9e9, 9e9, 9e9)
                bodyVel.Velocity = Vector3.new(0, 0, 0)
            end
        end)
    else
        disconnect("fly")
        local currentRoot = getRootPart()
        if currentRoot and currentRoot:FindFirstChild("FlyVel") then currentRoot.FlyVel:Destroy() end
    end
end})

MovementBox2:AddToggle('NoClipToggle', {Text = 'Atravessar Paredes', Default = false, Callback = function(state)
    states.noclipEnabled = state
    if state then
        connections.noclip = RS.Stepped:Connect(function()
            if not states.noclipEnabled then return end
            local char = getCharacter()
            if char then for _, part in pairs(char:GetDescendants()) do if part:IsA("BasePart") and part.CanCollide then part.CanCollide = false task.wait() part.CanCollide = false end end end
        end)
    else
        disconnect("noclip")
        local char = getCharacter()
        if char then for _, part in pairs(char:GetDescendants()) do if part:IsA("BasePart") and part.Name ~= "HumanoidRootPart" then part.CanCollide = true end end end
    end
end})

local jumpCooldown = false
MovementBox2:AddToggle('InfiniteJumpToggle', {Text = 'Pulo Infinito', Default = false, Callback = function(state)
    states.infJumpEnabled = state
    if state then
        connections.infJump = UIS.JumpRequest:Connect(function()
            if states.infJumpEnabled and not jumpCooldown then
                jumpCooldown = true
                local humanoid = getHumanoid()
                if humanoid then humanoid:ChangeState(Enum.HumanoidStateType.Jumping) end
                task.wait(0.3)
                jumpCooldown = false
            end
        end)
    else disconnect("infJump") end
end})

MovementBox2:AddToggle('AntiFallDamage', {Text = 'Anti Dano de Queda', Default = false, Callback = function(state)
    states.antiFallEnabled = state
    if state then
        connections.antiFall = RS.Heartbeat:Connect(function()
            if not states.antiFallEnabled then return end
            local humanoid = getHumanoid()
            if humanoid then
                local stateType = humanoid:GetState()
                if stateType == Enum.HumanoidStateType.Freefall or stateType == Enum.HumanoidStateType.Flying then
                    humanoid:ChangeState(Enum.HumanoidStateType.Landed)
                end
            end
        end)
    else disconnect("antiFall") end
end})

local PlayerBox = Tabs.Player:AddLeftGroupbox('Modificações')
PlayerBox:AddToggle('GodMode', {Text = 'God Mode', Default = false, Callback = function(state)
    states.godModeEnabled = state
    local humanoid = getHumanoid()
    if humanoid then
        if state then
            local oldHealth = humanoid.Health
            connections.godMode = humanoid.HealthChanged:Connect(function(health)
                if states.godModeEnabled and health < oldHealth then humanoid.Health = oldHealth end
                oldHealth = humanoid.Health
            end)
        else disconnect("godMode") end
    end
end})

PlayerBox:AddToggle('InfiniteStamina', {Text = 'Stamina Infinita', Default = false, Callback = function(state)
    states.infStaminaEnabled = state
    if state then
        connections.infStamina = RS.Heartbeat:Connect(function()
            if not states.infStaminaEnabled then return end
            local char = getCharacter()
            if char then for _, v in pairs(char:GetDescendants()) do if v.Name == "Stamina" or v.Name == "Energy" then v.Value = 100 end end end
        end)
    else disconnect("infStamina") end
end})

PlayerBox:AddToggle('AntiRagdoll', {Text = 'Anti Ragdoll', Default = false, Callback = function(state)
    states.antiRagdollEnabled = state
    local humanoid = getHumanoid()
    if humanoid then
        if state then
            connections.antiRagdoll = humanoid.StateChanged:Connect(function(old, new)
                if states.antiRagdollEnabled and new == Enum.HumanoidStateType.Ragdoll then humanoid:ChangeState(Enum.HumanoidStateType.Running) end
            end)
        else disconnect("antiRagdoll") end
    end
end})

PlayerBox:AddToggle('AntiSlow', {Text = 'Anti Lentidão', Default = false, Callback = function(state)
    states.antiSlowEnabled = state
    if state then
        connections.antiSlow = RS.Heartbeat:Connect(function()
            if not states.antiSlowEnabled then return end
            local humanoid = getHumanoid()
            if humanoid and states.speedEnabled and humanoid.WalkSpeed < speedValue then humanoid.WalkSpeed = speedValue end
        end)
    else disconnect("antiSlow") end
end})

PlayerBox:AddToggle('Invisivel', {Text = 'Ficar Invisível', Default = false, Callback = function(state)
    states.invisEnabled = state
    local char = getCharacter()
    if char then
        if state then
            for _, part in pairs(char:GetDescendants()) do
                if part:IsA("BasePart") then part.Transparency = 1 if part.Name == "Head" then for _, face in pairs(part:GetChildren()) do if face:IsA("Decal") then face.Transparency = 1 end end end
                elseif part:IsA("Decal") then part.Transparency = 1 end
            end
            for _, accessory in pairs(char:GetChildren()) do if accessory:IsA("Accessory") then for _, partDesc in pairs(accessory:GetDescendants()) do if partDesc:IsA("BasePart") then partDesc.Transparency = 1 end end end end
        else
            for _, part in pairs(char:GetDescendants()) do
                if part:IsA("BasePart") then if part.Name ~= "HumanoidRootPart" then part.Transparency = 0 end if part.Name == "Head" then for _, face in pairs(part:GetChildren()) do if face:IsA("Decal") then face.Transparency = 0 end end end
                elseif part:IsA("Decal") then part.Transparency = 0 end
            end
            for _, accessory in pairs(char:GetChildren()) do if accessory:IsA("Accessory") then for _, partDesc in pairs(accessory:GetDescendants()) do if partDesc:IsA("BasePart") then partDesc.Transparency = 0 end end end end
        end
    end
end})

local PlayerBox2 = Tabs.Player:AddRightGroupbox('Ações')
PlayerBox2:AddButton({Text = 'Resetar Personagem', Func = function() local humanoid = getHumanoid() if humanoid then humanoid.Health = 0 end end})
PlayerBox2:AddButton({Text = 'Remover Acessórios', Func = function() local char = getCharacter() if char then for _, accessory in pairs(char:GetChildren()) do if accessory:IsA("Accessory") then accessory:Destroy() end end end end})

local function getPlayerNames()
    local names = {}
    for _, p in pairs(Players:GetPlayers()) do if p ~= player then table.insert(names, p.Name) end end
    return names
end

local VehicleBox = Tabs.Vehicle:AddLeftGroupbox('Seleção')
local PlayerDropdownVehicle = VehicleBox:AddDropdown('PlayerSelectVehicle', {Values = getPlayerNames(), Default = 1, Multi = false, Text = 'Selecionar Jogador', Callback = function(value)
    VehicleSystem.selectedPlayer = Players:FindFirstChild(value)
    VehicleSystem.usedVehicles = {}
end})

VehicleBox:AddButton({Text = 'Atualizar Lista', Func = function() Options.PlayerSelectVehicle:SetValues(getPlayerNames()) end})

local VehicleBox2 = Tabs.Vehicle:AddRightGroupbox('Ações Básicas')

VehicleBox2:AddButton({Text = 'Entrar Veículo Mais Próximo', Func = function()
    local root = getRootPart()
    local humanoid = getHumanoid()
    if not root or not humanoid then return end
    
    local closestVehicle, closestDistance = nil, math.huge
    for _, obj in pairs(workspace:GetDescendants()) do
        if obj:IsA("VehicleSeat") and not obj.Occupant then
            local distance = (root.Position - obj.Position).Magnitude
            if distance < closestDistance then
                closestDistance = distance
                closestVehicle = obj
            end
        end
    end
    
    if closestVehicle then
        root.CFrame = closestVehicle.CFrame + Vector3.new(0, 3, 0)
        task.wait(0.1)
        closestVehicle:Sit(humanoid)
    end
end})

VehicleBox2:AddButton({Text = 'Puxar Veículo Mais Próximo', Func = function()
    task.spawn(function()
        local vehicle = VehicleSystem:FindClosestVehicle()
        if vehicle then
            local root = getRootPart()
            if root then 
                VehicleSystem:PullVehicle(vehicle, root.CFrame + Vector3.new(5, 0, 0))
                task.wait(0.5)
            end
        else
            VehicleSystem.pulledVehicles = {}
            local newVehicle = VehicleSystem:FindClosestVehicle()
            if newVehicle then
                local root = getRootPart()
                if root then 
                    VehicleSystem:PullVehicle(newVehicle, root.CFrame + Vector3.new(5, 0, 0))
                    task.wait(0.5)
                end
            end
        end
    end)
end})

VehicleBox2:AddButton({Text = 'Puxar Todos Para Mim', Func = function()
    task.spawn(function()
        local root = getRootPart()
        if not root then return end
        local originalCFrame = root.CFrame
        local allVehicles = VehicleSystem:GetAllVehicles()
        for i, vehicle in ipairs(allVehicles) do
            if not vehicle.Occupant then
                local offset = Vector3.new(5 + i * 2, 0, 0)
                VehicleSystem:PullVehicle(vehicle, root.CFrame + offset)
                task.wait(0.5)
            end
        end
        root.CFrame = originalCFrame
    end)
end})

VehicleBox2:AddButton({Text = 'Destrancar Todos Veículos', Func = function()
    for _, vehicle in ipairs(workspace:GetDescendants()) do
        if vehicle:IsA("VehicleSeat") then vehicle.Disabled = false end
        if vehicle:FindFirstChild("Locked") then vehicle.Locked.Value = false end
    end
end})

VehicleBox2:AddButton({Text = 'Resetar Veículos Puxados', Func = function()
    VehicleSystem.pulledVehicles = {}
    VehicleSystem.usedVehicles = {}
end})

local VehicleBox3 = Tabs.Vehicle:AddLeftGroupbox('Ações Avançadas')
VehicleBox3:AddButton({Text = 'Chuva de Veículos', Func = function()
    if not VehicleSystem.selectedPlayer then return end
    task.spawn(function()
        local allVehicles = VehicleSystem:GetAllVehicles()
        for _, vehicle in ipairs(allVehicles) do
            if not VehicleSystem.loopActive then break end
            if VehicleSystem.selectedPlayer and VehicleSystem.selectedPlayer.Character and VehicleSystem.selectedPlayer.Character:FindFirstChild("HumanoidRootPart") then
                VehicleSystem:PullVehicle(vehicle, VehicleSystem.selectedPlayer.Character.HumanoidRootPart.CFrame)
                task.wait(0.5)
            end
        end
    end)
end})

VehicleBox3:AddToggle('LoopRain', {Text = 'Loop Chuva de Veículos', Default = false, Callback = function(state)
    VehicleSystem.loopActive = state
    if state then
        task.spawn(function()
            while VehicleSystem.loopActive do
                if not VehicleSystem.selectedPlayer then break end
                local allVehicles = VehicleSystem:GetAllVehicles()
                for _, vehicle in ipairs(allVehicles) do
                    if not VehicleSystem.loopActive then break end
                    if VehicleSystem.selectedPlayer and VehicleSystem.selectedPlayer.Character and VehicleSystem.selectedPlayer.Character:FindFirstChild("HumanoidRootPart") then
                        VehicleSystem:PullVehicle(vehicle, VehicleSystem.selectedPlayer.Character.HumanoidRootPart.CFrame)
                        task.wait(0.5)
                    end
                end
                task.wait(1.5)
            end
        end)
    end
end})

VehicleBox3:AddButton({Text = 'Fling Veículo Próximo', Func = function()
    local root = getRootPart()
    if not root then return end
    local closestVehicle, closestDistance = nil, math.huge
    for _, obj in pairs(workspace:GetDescendants()) do
        if obj:IsA("VehicleSeat") then
            local distance = (root.Position - obj.Position).Magnitude
            if distance < closestDistance and distance < 50 then closestDistance, closestVehicle = distance, obj end
        end
    end
    if closestVehicle then
        if closestVehicle:FindFirstChild("SeatWeld") then closestVehicle.SeatWeld:Destroy() end
        root.CFrame = closestVehicle.CFrame
        task.wait(0.1)
        local bv = Instance.new("BodyVelocity")
        bv.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
        bv.Velocity = Vector3.new(math.random(-300, 300), 500, math.random(-300, 300))
        bv.Parent = closestVehicle
        local bav = Instance.new("BodyAngularVelocity")
        bav.MaxTorque = Vector3.new(math.huge, math.huge, math.huge)
        bav.AngularVelocity = Vector3.new(math.random(-50, 50), math.random(-50, 50), math.random(-50, 50))
        bav.Parent = closestVehicle
        game:GetService("Debris"):AddItem(bv, 0.5)
        game:GetService("Debris"):AddItem(bav, 0.5)
    end
end})

local VehicleBox4 = Tabs.Vehicle:AddRightGroupbox('Modificações')
VehicleBox4:AddToggle('VehicleBoost', {Text = 'Ativar Boost', Default = false, Callback = function(state) VehicleSystem.boostSettings.enabled = state end})
VehicleBox4:AddToggle('VehicleNoclip', {Text = 'Noclip Veículo', Default = false, Callback = function(state) VehicleSystem.boostSettings.noclipEnabled = state end})
VehicleBox4:AddLabel('Tecla Boost: E')
VehicleBox4:AddLabel('Tecla Freio: Q')
VehicleBox4:AddSlider('BoostForce', {Text = 'Força do Boost', Default = 50, Min = 1, Max = 500, Rounding = 0, Compact = false, Callback = function(value) VehicleSystem.boostSettings.boostForce = value end})

UIS.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed or not VehicleSystem.boostSettings.enabled then return end
    local character = getCharacter()
    if not character then return end
    local humanoid = getHumanoid()
    if not humanoid then return end
    local seat = humanoid.SeatPart
    if not seat then return end
    if input.KeyCode == VehicleSystem.boostSettings.boostKey then
        local boostVector = seat.CFrame.LookVector * VehicleSystem.boostSettings.boostForce
        seat.AssemblyLinearVelocity = seat.AssemblyLinearVelocity + boostVector
    elseif input.KeyCode == VehicleSystem.boostSettings.brakeKey then
        seat.AssemblyLinearVelocity = Vector3.new(0, 0, 0)
    end
end)

RS.Heartbeat:Connect(function()
    if not VehicleSystem.boostSettings.noclipEnabled then return end
    local character = getCharacter()
    if not character then return end
    local humanoid = getHumanoid()
    if not humanoid then return end
    local seat = humanoid.SeatPart
    if not seat then return end
    local vehicle = seat.Parent
    if not vehicle then return end
    for _, part in pairs(vehicle:GetDescendants()) do
        if part:IsA("BasePart") then part.CanCollide = false end
    end
end)

local selectedPlayer = nil
local function updatePlayerList()
    local playerNames = {}
    for _, p in ipairs(game.Players:GetPlayers()) do if p ~= player then table.insert(playerNames, p.Name) end end
    return playerNames
end

local TrollBox = Tabs.Troll:AddLeftGroupbox('Seleção')
local PlayerDropdown = TrollBox:AddDropdown('PlayerSelect', {Values = updatePlayerList(), Default = 1, Multi = false, Text = 'Selecionar Jogador', Callback = function(value) selectedPlayer = value end})
TrollBox:AddButton({Text = 'Atualizar Lista', Func = function() Options.PlayerSelect:SetValues(updatePlayerList()) end})
game.Players.PlayerAdded:Connect(function() Options.PlayerSelect:SetValues(updatePlayerList()) end)
game.Players.PlayerRemoving:Connect(function() Options.PlayerSelect:SetValues(updatePlayerList()) end)

local TrollBox2 = Tabs.Troll:AddRightGroupbox('Ações')
TrollBox2:AddButton({Text = 'Explodir Jogador', Func = function()
    if selectedPlayer then
        local targetPlayer = game.Players:FindFirstChild(selectedPlayer)
        if targetPlayer and targetPlayer.Character and targetPlayer.Character:FindFirstChild("HumanoidRootPart") then
            local explosion = Instance.new("Explosion")
            explosion.Position, explosion.BlastRadius, explosion.BlastPressure, explosion.Parent = targetPlayer.Character.HumanoidRootPart.Position, 15, 500000, workspace
        end
    end
end})

TrollBox2:AddButton({Text = 'Teleportar Para', Func = function()
    if selectedPlayer then
        local targetPlayer = game.Players:FindFirstChild(selectedPlayer)
        local root = getRootPart()
        if targetPlayer and targetPlayer.Character and targetPlayer.Character:FindFirstChild("HumanoidRootPart") and root then
            root.CFrame = targetPlayer.Character.HumanoidRootPart.CFrame * CFrame.new(0, 0, 3)
        end
    end
end})

TrollBox2:AddButton({Text = 'Trazer Jogador', Func = function()
    if selectedPlayer then
        local targetPlayer = game.Players:FindFirstChild(selectedPlayer)
        local root = getRootPart()
        if targetPlayer and targetPlayer.Character and targetPlayer.Character:FindFirstChild("HumanoidRootPart") and root then
            targetPlayer.Character.HumanoidRootPart.CFrame = root.CFrame * CFrame.new(0, 0, -3)
        end
    end
end})

TrollBox2:AddButton({Text = 'Lançar Para Cima', Func = function()
    if selectedPlayer then
        local targetPlayer = game.Players:FindFirstChild(selectedPlayer)
        local root = getRootPart()
        if targetPlayer and targetPlayer.Character and targetPlayer.Character:FindFirstChild("HumanoidRootPart") and root then
            local targetRoot = targetPlayer.Character.HumanoidRootPart
            local originalCFrame = root.CFrame
            root.CFrame = targetRoot.CFrame
            task.wait(0.2)
            root.Velocity = Vector3.new(0, 500, 0)
            task.wait(0.3)
            root.Velocity = Vector3.new(0, 0, 0)
            root.CFrame = originalCFrame
        end
    end
end})

TrollBox2:AddButton({Text = 'Orbit', Func = function()
    if selectedPlayer then
        local targetPlayer = game.Players:FindFirstChild(selectedPlayer)
        local root = getRootPart()
        if targetPlayer and targetPlayer.Character and targetPlayer.Character:FindFirstChild("HumanoidRootPart") and root then
            task.spawn(function()
                local targetRoot, angle = targetPlayer.Character.HumanoidRootPart, 0
                for i = 1, 50 do
                    angle = angle + 0.2
                    local offset = Vector3.new(math.cos(angle) * 10, 0, math.sin(angle) * 10)
                    root.CFrame = CFrame.new(targetRoot.Position + offset, targetRoot.Position)
                    task.wait(0.05)
                end
            end)
        end
    end
end})

TrollBox2:AddButton({Text = 'Spam Teleport', Func = function()
    if selectedPlayer then
        local targetPlayer = game.Players:FindFirstChild(selectedPlayer)
        local root = getRootPart()
        if targetPlayer and targetPlayer.Character and targetPlayer.Character:FindFirstChild("HumanoidRootPart") and root then
            task.spawn(function()
                local targetRoot = targetPlayer.Character.HumanoidRootPart
                for i = 1, 20 do 
                    root.CFrame = targetRoot.CFrame * CFrame.new(math.random(-5, 5), 0, math.random(-5, 5)) 
                    task.wait(0.1) 
                end
            end)
        end
    end
end})

TrollBox2:AddButton({Text = 'Fling Jogador', Func = function()
    if selectedPlayer then
        local targetPlayer = game.Players:FindFirstChild(selectedPlayer)
        local root = getRootPart()
        if targetPlayer and targetPlayer.Character and targetPlayer.Character:FindFirstChild("HumanoidRootPart") and root then
            task.spawn(function()
                local targetRoot = targetPlayer.Character.HumanoidRootPart
                local originalCFrame = root.CFrame
                local wasNoClip = states.noclipEnabled
                if not wasNoClip then 
                    states.noclipEnabled = true
                    task.wait(0.1)
                end
                
                root.CFrame = targetRoot.CFrame
                task.wait(0.2)
                
                local spinPart = Instance.new("Part")
                spinPart.Size = Vector3.new(8, 1, 8)
                spinPart.Transparency = 1
                spinPart.Anchored = false
                spinPart.CanCollide = true
                spinPart.Massless = false
                spinPart.Parent = workspace
                spinPart.CFrame = root.CFrame
                
                local bodyPos = Instance.new("BodyPosition")
                bodyPos.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
                bodyPos.Position = root.Position
                bodyPos.Parent = spinPart
                
                local weld = Instance.new("WeldConstraint")
                weld.Part0 = root
                weld.Part1 = spinPart
                weld.Parent = spinPart
                
                local angularVel = Instance.new("BodyAngularVelocity")
                angularVel.MaxTorque = Vector3.new(math.huge, math.huge, math.huge)
                angularVel.AngularVelocity = Vector3.new(0, 250, 0)
                angularVel.Parent = spinPart
                
                task.wait(2)
                
                if spinPart then spinPart:Destroy() end
                task.wait(0.1)
                root.CFrame = originalCFrame
                
                if not wasNoClip then 
                    states.noclipEnabled = false
                end
            end)
        end
    end
end})

local TrollBox3 = Tabs.Troll:AddLeftGroupbox('Troll Global')
TrollBox3:AddToggle('SpinPlayers', {Text = 'Girar Todos', Default = false, Callback = function(state)
    states.spinPlayersEnabled = state
    if state then
        connections.spinPlayers = RS.Heartbeat:Connect(function()
            if not states.spinPlayersEnabled then return end
            for _, p in ipairs(game.Players:GetPlayers()) do
                if p ~= player and p.Character and p.Character:FindFirstChild("HumanoidRootPart") then
                    p.Character.HumanoidRootPart.CFrame = p.Character.HumanoidRootPart.CFrame * CFrame.Angles(0, math.rad(20), 0)
                end
            end
        end)
    else disconnect("spinPlayers") end
end})

TrollBox3:AddToggle('FlingAura', {Text = 'Fling Aura', Default = false, Callback = function(state)
    states.flingAuraEnabled = state
    if state then
        task.spawn(function()
            while states.flingAuraEnabled do
                local root = getRootPart()
                if root then
                    for _, p in ipairs(game.Players:GetPlayers()) do
                        if p ~= player and p.Character and p.Character:FindFirstChild("HumanoidRootPart") then
                            local targetRoot = p.Character.HumanoidRootPart
                            local distance = (root.Position - targetRoot.Position).Magnitude
                            if distance < 15 then
                                local originalCFrame = root.CFrame
                                root.CFrame = targetRoot.CFrame
                                root.Velocity = Vector3.new(math.random(-200, 200), 100, math.random(-200, 200))
                                root.RotVelocity = Vector3.new(math.random(-100, 100), math.random(-100, 100), math.random(-100, 100))
                                task.wait(0.3)
                                root.CFrame = originalCFrame
                                root.Velocity = Vector3.new(0, 0, 0)
                                root.RotVelocity = Vector3.new(0, 0, 0)
                            end
                        end
                    end
                end
                task.wait(0.5)
            end
        end)
    else states.flingAuraEnabled = false end
end})

TrollBox3:AddToggle('FreezePlayers', {Text = 'Congelar Todos', Default = false, Callback = function(state)
    states.freezePlayersEnabled = state
    if state then
        connections.freezePlayers = RS.Heartbeat:Connect(function()
            if not states.freezePlayersEnabled then return end
            for _, p in ipairs(game.Players:GetPlayers()) do
                if p ~= player and p.Character and p.Character:FindFirstChild("HumanoidRootPart") then p.Character.HumanoidRootPart.Anchored = true end
            end
        end)
    else
        disconnect("freezePlayers")
        for _, p in ipairs(game.Players:GetPlayers()) do
            if p ~= player and p.Character and p.Character:FindFirstChild("HumanoidRootPart") then p.Character.HumanoidRootPart.Anchored = false end
        end
    end
end})

TrollBox3:AddToggle('LoopFling', {Text = 'Loop Fling', Default = false, Callback = function(state)
    states.loopFlingEnabled = state
    if state then
        task.spawn(function()
            while states.loopFlingEnabled do
                local root = getRootPart()
                if root then
                    for _, p in ipairs(game.Players:GetPlayers()) do
                        if not states.loopFlingEnabled then break end
                        if p ~= player and p.Character and p.Character:FindFirstChild("HumanoidRootPart") then
                            local targetRoot = p.Character.HumanoidRootPart
                            local distance = (root.Position - targetRoot.Position).Magnitude
                            if distance < 30 then
                                local originalCFrame = root.CFrame
                                root.CFrame = targetRoot.CFrame
                                root.Velocity = Vector3.new(math.random(-150, 150), 50, math.random(-150, 150))
                                task.wait(0.4)
                                root.CFrame = originalCFrame
                                root.Velocity = Vector3.new(0, 0, 0)
                            end
                        end
                    end
                end
                task.wait(0.2)
            end
        end)
    else states.loopFlingEnabled = false end
end})

TrollBox3:AddToggle('KillAura', {Text = 'Kill Aura', Default = false, Callback = function(state)
    states.killAuraEnabled = state
    if state then
        connections.killAura = RS.Heartbeat:Connect(function()
            if not states.killAuraEnabled then return end
            local root = getRootPart()
            if not root then return end
            for _, p in ipairs(game.Players:GetPlayers()) do
                if p ~= player and p.Character and p.Character:FindFirstChild("HumanoidRootPart") then
                    local targetRoot = p.Character.HumanoidRootPart
                    local distance = (root.Position - targetRoot.Position).Magnitude
                    if distance < 20 then
                        local humanoid = p.Character:FindFirstChildOfClass("Humanoid")
                        if humanoid then humanoid.Health = 0 end
                    end
                end
            end
        end)
    else disconnect("killAura") end
end})

local OPBox = Tabs.OP:AddLeftGroupbox('Funções OP')
OPBox:AddButton({Text = 'Fling Servidor Inteiro', Func = function()
    task.spawn(function()
        local root = getRootPart()
        if not root then return end
        
        for _, p in ipairs(game.Players:GetPlayers()) do
            if p ~= player and p.Character and p.Character:FindFirstChild("HumanoidRootPart") then
                local targetRoot = p.Character.HumanoidRootPart
                local originalCFrame = root.CFrame
                
                root.CFrame = targetRoot.CFrame
                task.wait(0.2)
                
                root.Velocity = Vector3.new(math.random(-300, 300), 200, math.random(-300, 300))
                root.RotVelocity = Vector3.new(math.random(-100, 100), math.random(-100, 100), math.random(-100, 100))
                
                task.wait(0.5)
                root.CFrame = originalCFrame
                root.Velocity = Vector3.new(0, 0, 0)
                root.RotVelocity = Vector3.new(0, 0, 0)
                task.wait(0.3)
            end
        end
    end)
end})

OPBox:AddButton({Text = 'Crashar Jogador', Func = function()
    if selectedPlayer then
        task.spawn(function()
            local targetPlayer = game.Players:FindFirstChild(selectedPlayer)
            if targetPlayer and targetPlayer.Character and targetPlayer.Character:FindFirstChild("HumanoidRootPart") then
                local root = getRootPart()
                if not root then return end
                local targetRoot = targetPlayer.Character.HumanoidRootPart
                
                for i = 1, 100 do
                    root.CFrame = targetRoot.CFrame
                    root.Velocity = Vector3.new(math.random(-500, 500), math.random(-500, 500), math.random(-500, 500))
                    root.RotVelocity = Vector3.new(math.random(-200, 200), math.random(-200, 200), math.random(-200, 200))
                    task.wait(0.05)
                end
            end
        end)
    end
end})

OPBox:AddButton({Text = 'Teleportar Todos Para Mim', Func = function()
    local root = getRootPart()
    if not root then return end
    
    for _, p in ipairs(game.Players:GetPlayers()) do
        if p ~= player and p.Character and p.Character:FindFirstChild("HumanoidRootPart") then
            p.Character.HumanoidRootPart.CFrame = root.CFrame * CFrame.new(math.random(-10, 10), 0, math.random(-10, 10))
        end
    end
end})

OPBox:AddButton({Text = 'Explodir Servidor', Func = function()
    for _, p in ipairs(game.Players:GetPlayers()) do
        if p ~= player and p.Character and p.Character:FindFirstChild("HumanoidRootPart") then
            local explosion = Instance.new("Explosion")
            explosion.Position = p.Character.HumanoidRootPart.Position
            explosion.BlastRadius = 20
            explosion.BlastPressure = 1000000
            explosion.Parent = workspace
        end
    end
end})

local OPBox2 = Tabs.OP:AddRightGroupbox('Loops OP')
OPBox2:AddToggle('MassTP', {Text = 'TP em Massa', Default = false, Callback = function(state)
    states.massTpEnabled = state
    if state then
        task.spawn(function()
            while states.massTpEnabled do
                local root = getRootPart()
                if root then
                    for _, p in ipairs(game.Players:GetPlayers()) do
                        if not states.massTpEnabled then break end
                        if p ~= player and p.Character and p.Character:FindFirstChild("HumanoidRootPart") then
                            p.Character.HumanoidRootPart.CFrame = root.CFrame * CFrame.new(math.random(-15, 15), 5, math.random(-15, 15))
                            task.wait(0.1)
                        end
                    end
                end
                task.wait(0.5)
            end
        end)
    else states.massTpEnabled = false end
end})

OPBox2:AddToggle('MassFling', {Text = 'Fling em Massa', Default = false, Callback = function(state)
    states.massFlingEnabled = state
    if state then
        task.spawn(function()
            while states.massFlingEnabled do
                local root = getRootPart()
                if root then
                    for _, p in ipairs(game.Players:GetPlayers()) do
                        if not states.massFlingEnabled then break end
                        if p ~= player and p.Character and p.Character:FindFirstChild("HumanoidRootPart") then
                            local targetRoot = p.Character.HumanoidRootPart
                            local originalCFrame = root.CFrame
                            root.CFrame = targetRoot.CFrame
                            root.Velocity = Vector3.new(math.random(-250, 250), 150, math.random(-250, 250))
                            task.wait(0.4)
                            root.CFrame = originalCFrame
                            root.Velocity = Vector3.new(0, 0, 0)
                        end
                    end
                end
                task.wait(0.3)
            end
        end)
    else states.massFlingEnabled = false end
end})

OPBox2:AddToggle('TpAllToMe', {Text = 'Trazer Todos', Default = false, Callback = function(state)
    states.tpAllEnabled = state
    if state then
        connections.tpAll = RS.Heartbeat:Connect(function()
            if not states.tpAllEnabled then return end
            local root = getRootPart()
            if not root then return end
            for _, p in ipairs(game.Players:GetPlayers()) do
                if p ~= player and p.Character and p.Character:FindFirstChild("HumanoidRootPart") then
                    p.Character.HumanoidRootPart.CFrame = root.CFrame * CFrame.new(0, 3, 0)
                end
            end
        end)
    else disconnect("tpAll") end
end})

OPBox2:AddToggle('ServerLag', {Text = 'Lagar Servidor', Default = false, Callback = function(state)
    states.serverLagEnabled = state
    if state then
        task.spawn(function()
            while states.serverLagEnabled do
                local root = getRootPart()
                if root then
                    for i = 1, 10 do
                        if not states.serverLagEnabled then break end
                        for _, p in ipairs(game.Players:GetPlayers()) do
                            if p ~= player and p.Character and p.Character:FindFirstChild("HumanoidRootPart") then
                                root.CFrame = p.Character.HumanoidRootPart.CFrame
                                root.Velocity = Vector3.new(math.random(-1000, 1000), math.random(-1000, 1000), math.random(-1000, 1000))
                            end
                        end
                        task.wait(0.05)
                    end
                end
                task.wait(0.1)
            end
        end)
    else states.serverLagEnabled = false end
end})

ThemeManager:SetLibrary(Library)
SaveManager:SetLibrary(Library)
SaveManager:IgnoreThemeSettings()
SaveManager:SetIgnoreIndexes({'MenuKeybind'})
ThemeManager:SetFolder('InfinityMenu')
SaveManager:SetFolder('InfinityMenu/configs')
SaveManager:BuildConfigSection(Tabs.Main)
ThemeManager:ApplyToTab(Tabs.Main)
Library.ToggleKeybind = Options.MenuKeybind
SaveManager:LoadAutoloadConfig()
