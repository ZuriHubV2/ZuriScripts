-- MHX HUB — Survive 99 Nights in the Forest
-- Rebuilt, robust, Rayfield UI, YT: YuriOnTop branding
-- Version: 1.2 (stable)

-- ====== Rayfield Load ======
local success, Rayfield = pcall(function()
    return loadstring(game:HttpGet("https://raw.githubusercontent.com/shlexware/Rayfield/main/source"))()
end)
if not success or not Rayfield then
    -- try backup url
    success, Rayfield = pcall(function()
        return loadstring(game:HttpGet("https://sirius.menu/rayfield"))()
    end)
end
if not success or not Rayfield then
    warn("Rayfield failed to load. Script will stop.")
    return
end

local Window = Rayfield:CreateWindow({
   Name = "MHX HUB Survive 99 Nights in the Forest | YT: YuriOnTop",
   LoadingTitle = "MHX HUB",
   LoadingSubtitle = "Survive 99 Nights in the Forest",
   Theme = "Default",
   ConfigurationSaving = {
      Enabled = true,
      FolderName = "MHXHub",
      FileName = "Config"
   },
   Discord = {
      Enabled = false
   },
   KeySystem = false
})

-- ====== SERVICES / CONSTANTS ======
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Workspace = game:GetService("Workspace")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local LocalPlayer = Players.LocalPlayer
local Camera = Workspace.CurrentCamera

local DEFAULT_HRP_SIZE = Vector3.new(2, 2, 1)
local LOOP_WAIT = 0.25

-- ====== SAFE HELPERS ======
local function safeFindRemote(name)
    if not ReplicatedStorage then return nil end
    local reFolder = ReplicatedStorage:FindFirstChild("RemoteEvents")
    if reFolder then
        return reFolder:FindFirstChild(name)
    end
    return nil
end

local function safeFire(remoteName, ...)
    local ok, err = pcall(function()
        local remote = safeFindRemote(remoteName)
        if remote and remote.FireServer then
            remote:FireServer(...)
        end
    end)
    return ok, err
end

local function safeGetChildren(obj)
    if obj and obj.GetChildren then
        local ok, children = pcall(function() return obj:GetChildren() end)
        if ok and children then return children end
    end
    return {}
end

-- DragItem: preferred remote; fallback to ClickDetector fire
local function DragItem(item)
    if not item then return end
    -- prefer RemoteEvents.RequestStartDraggingItem / StopDraggingItem
    local ok = pcall(function()
        local re = ReplicatedStorage:FindFirstChild("RemoteEvents")
        if re then
            local start = re:FindFirstChild("RequestStartDraggingItem")
            local stop = re:FindFirstChild("StopDraggingItem")
            if start and stop then
                start:FireServer(item)
                task.wait(0.01)
                stop:FireServer(item)
                return
            end
        end
    end)
    if not ok then
        -- fallback: try to fire click detector on a child
        for _, d in pairs(item:GetDescendants()) do
            if d:IsA("ClickDetector") then
                pcall(function() fireclickdetector(d) end)
                return
            end
        end
    end
end

-- Clean up ESP elements that we create
local function cleanupEspFor(model)
    if not model then return end
    -- Highlight named "MHX_ESP_Highlight"
    local h = model:FindFirstChild("MHX_ESP_Highlight")
    if h and h:IsA("Highlight") then pcall(function() h:Destroy() end) end
    -- Billboards named "MHX_ESP_Billboard" inside model (or its parts)
    for _, d in pairs(model:GetDescendants()) do
        if d:IsA("BillboardGui") and d.Name == "MHX_ESP_Billboard" then
            pcall(function() d:Destroy() end)
        end
    end
end

local function createEsp(model, color, text, adorneePart, yOffset)
    if not model or not adorneePart or not adorneePart:IsA("BasePart") then return end
    -- prevent duplicates
    if model:FindFirstChild("MHX_ESP_Highlight") or adorneePart:FindFirstChild("MHX_ESP_Billboard") then
        return
    end

    -- Highlight
    local highlight = Instance.new("Highlight")
    highlight.Name = "MHX_ESP_Highlight"
    highlight.Adornee = model
    highlight.FillColor = color
    highlight.OutlineColor = color
    highlight.FillTransparency = 1
    highlight.OutlineTransparency = 0
    highlight.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
    highlight.Parent = model

    -- BillboardGui
    local billboard = Instance.new("BillboardGui")
    billboard.Name = "MHX_ESP_Billboard"
    billboard.Adornee = adorneePart
    billboard.Size = UDim2.new(0, 120, 0, 28)
    billboard.StudsOffset = Vector3.new(0, yOffset or 2, 0)
    billboard.AlwaysOnTop = true
    billboard.Parent = adorneePart

    local label = Instance.new("TextLabel")
    label.Name = "MHX_ESP_Label"
    label.Size = UDim2.new(1, 0, 1, 0)
    label.BackgroundTransparency = 1
    label.Text = text or model.Name
    label.TextScaled = true
    label.Font = Enum.Font.SourceSansSemibold
    label.TextColor3 = color
    label.Parent = billboard

    -- distance updater
    task.spawn(function()
        while highlight and highlight.Parent and billboard and billboard.Parent do
            pcall(function()
                local cam = Workspace.CurrentCamera
                if cam and billboard.Adornee and billboard.Adornee:IsA("BasePart") then
                    local dist = (cam.CFrame.Position - billboard.Adornee.Position).Magnitude
                    label.Text = (text or model.Name) .. " (" .. tostring(math.floor(dist + 0.5)) .. " m)"
                end
            end)
            task.wait(0.5)
        end
    end)
end

-- ====== GLOBAL FLAGS & CONNECTIONS ======
local Flags = {
    Hitbox = false,
    Fly = false,
    WalkSpeed = false,
    GodMode = false,
    PlayerESP = false,
    ScrapESP = false,
    WoodESP = false,
    FoodESP = false,
    MedkitESP = false,
    ForestGemESP = false,
    CulisticGemESP = false,
    NpcESP = false,
    ChestESP = false,
    KillAura = false
}
local HitboxSize = 10
local DistanceForKillAura = 25

local flyConnections = {}
local flyObjects = {}

-- ====== FLY (safe UInput-based) ======
local function startFly()
    if Flags.Fly == false then return end
    local char = LocalPlayer.Character
    if not char then return end
    local hrp = char:FindFirstChild("HumanoidRootPart")
    local humanoid = char:FindFirstChildOfClass("Humanoid")
    if not hrp then return end

    -- cleanup existing
    for _, c in pairs(flyConnections) do
        pcall(function() c:Disconnect() end)
    end
    flyConnections = {}

    local control = {F=0,B=0,L=0,R=0,Q=0,E=0}
    local lastControl = {F=0,B=0,L=0,R=0,Q=0,E=0}
    local SPEED = 0
    local BG = Instance.new("BodyGyro")
    local BV = Instance.new("BodyVelocity")
    BG.P = 9e4
    BG.MaxTorque = Vector3.new(9e9,9e9,9e9)
    BG.CFrame = Camera and Camera.CFrame or hrp.CFrame
    BV.MaxForce = Vector3.new(9e9,9e9,9e9)
    BV.Velocity = Vector3.new(0,0,0)
    BG.Parent = hrp
    BV.Parent = hrp
    if humanoid then pcall(function() humanoid.PlatformStand = true end) end

    local function update()
        if (control.L + control.R) ~= 0 or (control.F + control.B) ~= 0 or (control.Q + control.E) ~= 0 then
            SPEED = 50
            local camCF = Workspace.CurrentCamera and Workspace.CurrentCamera.CFrame or hrp.CFrame
            BV.Velocity = ((camCF.LookVector * (control.F + control.B)) + ((camCF * CFrame.new(control.L + control.R, (control.F + control.B + control.Q + control.E) * 0.2, 0).p) - camCF.p)) * SPEED
            lastControl = {F = control.F, B = control.B, L = control.L, R = control.R}
        else
            SPEED = 0
            BV.Velocity = Vector3.new(0,0,0)
        end
        BG.CFrame = Workspace.CurrentCamera and Workspace.CurrentCamera.CFrame or hrp.CFrame
    end

    local began = UserInputService.InputBegan:Connect(function(input, gp)
        if gp then return end
        if input.UserInputType == Enum.UserInputType.Keyboard then
            local k = tostring(input.KeyCode):gsub("Enum.KeyCode.", ""):lower()
            if k == "w" then control.F = 1 end
            if k == "s" then control.B = -1 end
            if k == "a" then control.L = -1 end
            if k == "d" then control.R = 1 end
            if k == "e" then control.Q = 2 end
            if k == "q" then control.E = -2 end
        end
    end)
    local ended = UserInputService.InputEnded:Connect(function(input, gp)
        if gp then return end
        if input.UserInputType == Enum.UserInputType.Keyboard then
            local k = tostring(input.KeyCode):gsub("Enum.KeyCode.", ""):lower()
            if k == "w" then control.F = 0 end
            if k == "s" then control.B = 0 end
            if k == "a" then control.L = 0 end
            if k == "d" then control.R = 0 end
            if k == "e" then control.Q = 0 end
            if k == "q" then control.E = 0 end
        end
    end)

    table.insert(flyConnections, began)
    table.insert(flyConnections, ended)

    flyObjects.BG = BG
    flyObjects.BV = BV

    task.spawn(function()
        while Flags.Fly do
            pcall(update)
            task.wait(0.03)
        end
        -- cleanup
        if BG and BG.Parent then pcall(function() BG:Destroy() end) end
        if BV and BV.Parent then pcall(function() BV:Destroy() end) end
        if humanoid and humanoid.Parent then pcall(function() humanoid.PlatformStand = false end) end
        for _, c in pairs(flyConnections) do pcall(function() c:Disconnect() end) end
        flyConnections = {}
        flyObjects = {}
    end)
end

local function stopFly()
    Flags.Fly = false
    for _, c in pairs(flyConnections) do pcall(function() c:Disconnect() end) end
    flyConnections = {}
    if flyObjects.BG and flyObjects.BG.Parent then pcall(function() flyObjects.BG:Destroy() end) end
    if flyObjects.BV and flyObjects.BV.Parent then pcall(function() flyObjects.BV:Destroy() end) end
    flyObjects = {}
    local hum = LocalPlayer.Character and LocalPlayer.Character:FindFirstChildOfClass("Humanoid")
    if hum then pcall(function() hum.PlatformStand = false end) end
end

-- ====== HITBOX ======
local function startHitbox(size)
    Flags.Hitbox = true
    HitboxSize = size or HitboxSize
    task.spawn(function()
        while Flags.Hitbox do
            for _, plr in pairs(Players:GetPlayers()) do
                if plr ~= LocalPlayer then
                    local ch = plr.Character
                    if ch and ch:FindFirstChild("HumanoidRootPart") then
                        pcall(function()
                            local hrp = ch.HumanoidRootPart
                            hrp.Size = Vector3.new(HitboxSize, HitboxSize, HitboxSize)
                            hrp.CanCollide = false
                        end)
                    end
                end
            end
            task.wait(0.15)
        end
        -- reset sizes
        for _, plr in pairs(Players:GetPlayers()) do
            if plr ~= LocalPlayer then
                local ch = plr.Character
                if ch and ch:FindFirstChild("HumanoidRootPart") then
                    pcall(function()
                        local hrp = ch.HumanoidRootPart
                        hrp.Size = DEFAULT_HRP_SIZE
                        hrp.CanCollide = true
                    end)
                end
            end
        end
    end)
end

local function stopHitbox()
    Flags.Hitbox = false
end

-- ====== WALKSPEED ======
local function startWalkSpeed()
    Flags.WalkSpeed = true
    task.spawn(function()
        while Flags.WalkSpeed do
            local char = LocalPlayer.Character
            if char then
                local hum = char:FindFirstChildOfClass("Humanoid")
                if hum then pcall(function() hum.WalkSpeed = 150 end) end
            end
            task.wait(0.2)
        end
        -- reset
        local char = LocalPlayer.Character
        if char then
            local hum = char:FindFirstChildOfClass("Humanoid")
            if hum then pcall(function() hum.WalkSpeed = 16 end) end
        end
    end)
end

-- ====== GOD MODE ======
local function startGodMode()
    Flags.GodMode = true
    task.spawn(function()
        while Flags.GodMode do
            local char = LocalPlayer.Character
            if char then
                local hum = char:FindFirstChildOfClass("Humanoid")
                if hum then
                    pcall(function() hum.Health = hum.MaxHealth end)
                end
                -- attempt to cancel via remote
                pcall(function()
                    local reDamage = safeFindRemote("DamagePlayer")
                    if reDamage then reDamage:FireServer(0) end
                end)
            end
            task.wait(0.5)
        end
    end)
end

-- ====== KILL AURA ======
local function startKillAura()
    Flags.KillAura = true
    task.spawn(function()
        while Flags.KillAura do
            local char = LocalPlayer.Character
            if char and char:FindFirstChild("HumanoidRootPart") then
                local hrp = char.HumanoidRootPart
                for _, obj in pairs(safeGetChildren(Workspace:FindFirstChild("Characters") or Workspace)) do
                    if obj and obj:IsA("Model") and obj.PrimaryPart and obj.Name ~= LocalPlayer.Name then
                        local ok, distance = pcall(function() return (hrp.Position - obj.PrimaryPart.Position).Magnitude end)
                        if ok and distance and distance <= DistanceForKillAura then
                            pcall(function()
                                local re = safeFindRemote("DamageEnemy")
                                if re then re:FireServer(obj, 1000) end
                            end)
                        end
                    end
                end
            end
            task.wait(0.2)
        end
    end)
end

-- ====== ESP LOOPS ======
local function runEspLoop(toggleFlagName, scanRoot, predicateFn, color, label, offset)
    task.spawn(function()
        while Flags[toggleFlagName] do
            local root = scanRoot and (Workspace:FindFirstChild(scanRoot) or Workspace) or Workspace
            for _, obj in pairs(safeGetChildren(root)) do
                if predicateFn(obj) then
                    local primary = obj.PrimaryPart or (obj:IsA("BasePart") and obj) or obj:FindFirstChildWhichIsA("BasePart")
                    if primary then
                        createEsp(obj, color, label or obj.Name, primary, offset or 2)
                    end
                end
            end
            task.wait(0.6)
        end
        -- cleanup when disabling
        local root = scanRoot and (Workspace:FindFirstChild(scanRoot) or Workspace) or Workspace
        for _, obj in pairs(safeGetChildren(root)) do
            if predicateFn(obj) then cleanupEspFor(obj) end
        end
    end)
end

-- ====== UI: TABS & CONTROLS ======
local PlayerTab = Window:CreateTab("Player")

PlayerTab:CreateSlider({
    Name = "Hitbox Size",
    Range = {5, 40},
    Increment = 1,
    CurrentValue = HitboxSize,
    Flag = "HitboxSize",
    Callback = function(Value)
        HitboxSize = Value
        if Flags.Hitbox then
            stopHitbox()
            task.wait(0.05)
            startHitbox(HitboxSize)
        end
    end
})

PlayerTab:CreateToggle({
    Name = "Enable Hitbox",
    CurrentValue = Flags.Hitbox,
    Flag = "EnableHitbox",
    Callback = function(Value)
        if Value then
            startHitbox(HitboxSize)
        else
            stopHitbox()
        end
    end
})

PlayerTab:CreateToggle({
    Name = "Fly (WASD + Q/E)",
    CurrentValue = Flags.Fly,
    Flag = "FlyToggle",
    Callback = function(Value)
        Flags.Fly = Value
        if Value then startFly() else stopFly() end
    end
})

PlayerTab:CreateToggle({
    Name = "Walk Speed (150)",
    CurrentValue = Flags.WalkSpeed,
    Flag = "WalkSpeedToggle",
    Callback = function(Value)
        Flags.WalkSpeed = Value
        if Value then startWalkSpeed() end
    end
})

PlayerTab:CreateToggle({
    Name = "God Mode",
    CurrentValue = Flags.GodMode,
    Flag = "GodModeToggle",
    Callback = function(Value)
        Flags.GodMode = Value
        if Value then startGodMode() end
    end
})

-- ====== ESP TAB ======
local EspTab = Window:CreateTab("ESP")

EspTab:CreateToggle({
    Name = "Player ESP",
    CurrentValue = Flags.PlayerESP,
    Flag = "PlayerEsp",
    Callback = function(Value)
        Flags.PlayerESP = Value
        if Value then
            runEspLoop("PlayerESP", nil, function(obj)
                return obj:IsA("Player") == false and obj:IsA("Model") and obj:FindFirstChild("HumanoidRootPart")
            end, Color3.fromRGB(0,0,255), "Player", 3) -- note: this predicate isn't ideal; we will iterate players separately below
            -- separate loop to iterate actual players (keeps labels accurate)
            task.spawn(function()
                while Flags.PlayerESP do
                    for _, p in pairs(Players:GetPlayers()) do
                        if p ~= LocalPlayer and p.Character and p.Character:FindFirstChild("HumanoidRootPart") then
                            createEsp(p.Character, Color3.fromRGB(0,0,255), p.Name, p.Character.HumanoidRootPart, 3)
                        end
                    end
                    task.wait(0.6)
                end
                for _, p in pairs(Players:GetPlayers()) do
                    if p.Character then cleanupEspFor(p.Character) end
                end
            end)
        else
            -- cleanup done by loop
        end
    end
})

EspTab:CreateToggle({
    Name = "Scrap ESP",
    CurrentValue = Flags.ScrapESP,
    Flag = "ScrapEsp",
    Callback = function(Value)
        Flags.ScrapESP = Value
        if Value then
            runEspLoop("ScrapESP", "Items", function(obj)
                return obj:IsA("Model") and obj.Name:match("Scrap")
            end, Color3.fromRGB(0,255,0), "Scrap", 2)
        end
    end
})

EspTab:CreateToggle({
    Name = "Wood ESP",
    CurrentValue = Flags.WoodESP,
    Flag = "WoodEsp",
    Callback = function(Value)
        Flags.WoodESP = Value
        if Value then
            runEspLoop("WoodESP", "Items", function(obj) return obj:IsA("Model") and obj.Name == "Log" end, Color3.fromRGB(255,0,0), "Wood", 2)
        end
    end
})

EspTab:CreateToggle({
    Name = "Food ESP",
    CurrentValue = Flags.FoodESP,
    Flag = "FoodEsp",
    Callback = function(Value)
        Flags.FoodESP = Value
        if Value then
            runEspLoop("FoodESP", "Items", function(obj) return obj:IsA("Model") and obj.Name:match("Food") end, Color3.fromRGB(128,0,128), "Food", 2)
        end
    end
})

EspTab:CreateToggle({
    Name = "Medkit/Bandage ESP",
    CurrentValue = Flags.MedkitESP,
    Flag = "MedkitEsp",
    Callback = function(Value)
        Flags.MedkitESP = Value
        if Value then
            runEspLoop("MedkitESP", "Items", function(obj) return obj:IsA("Model") and (obj.Name == "Medkit" or obj.Name == "Bandage") end, Color3.fromRGB(255,105,180), "Medkit", 2)
        end
    end
})

EspTab:CreateToggle({
    Name = "Piece of Forest (Forest Gem) ESP",
    CurrentValue = Flags.ForestGemESP,
    Flag = "ForestGemEsp",
    Callback = function(Value)
        Flags.ForestGemESP = Value
        if Value then
            runEspLoop("ForestGemESP", "Items", function(obj) return obj:IsA("Model") and obj.Name == "Piece of Forest" end, Color3.fromRGB(0,255,0), "Forest Gem", 2)
        end
    end
})

EspTab:CreateToggle({
    Name = "Culistic Gem ESP",
    CurrentValue = Flags.CulisticGemESP,
    Flag = "CulisticGemEsp",
    Callback = function(Value)
        Flags.CulisticGemESP = Value
        if Value then
            runEspLoop("CulisticGemESP", "Items", function(obj) return obj:IsA("Model") and obj.Name == "Culistic Gem" end, Color3.fromRGB(255,255,255), "Culistic Gem", 2)
        end
    end
})

EspTab:CreateToggle({
    Name = "NPC ESP",
    CurrentValue = Flags.NpcESP,
    Flag = "NpcEsp",
    Callback = function(Value)
        Flags.NpcESP = Value
        if Value then
            runEspLoop("NpcESP", "Characters", function(obj) return obj:IsA("Model") and obj.PrimaryPart end, Color3.fromRGB(255,165,0), "NPC", 3)
        end
    end
})

EspTab:CreateToggle({
    Name = "Chest ESP",
    CurrentValue = Flags.ChestESP,
    Flag = "ChestEsp",
    Callback = function(Value)
        Flags.ChestESP = Value
        if Value then
            runEspLoop("ChestESP", nil, function(obj) return obj:IsA("Model") and obj.Name:match("Chest") end, Color3.fromRGB(255,255,0), "Chest", 2)
        end
    end
})

-- ====== WEAPONS TAB ======
local WeaponsTab = Window:CreateTab("Weapons")

WeaponsTab:CreateButton({
    Name = "Open Weapons/Armors",
    Callback = function()
        local WeaponsWindow = Rayfield:CreateWindow({
            Name = "Weapons and Armors",
            LoadingTitle = "Select Item",
            LoadingSubtitle = "MHX HUB",
            Theme = "Default"
        })
        local items = {
            "Strong Axe", "Good Axe", "Rifle", "Shotgun", "Spear", "Alien Weapon",
            "Strong Flashlight", "Old Flashlight", "Iron Body", "Leather Body", "Alien Body"
        }
        for _, item in pairs(items) do
            WeaponsWindow:CreateButton({
                Name = item,
                Callback = function()
                    pcall(function()
                        local re = safeFindRemote("GiveItem")
                        if re then re:FireServer(item) end
                    end)
                end
            })
        end
    end
})

-- ====== EXPLOSIONS & TOOLS TAB ======
local ExplosionsTab = Window:CreateTab("Explosions")

ExplosionsTab:CreateToggle({
    Name = "Kill Aura",
    CurrentValue = Flags.KillAura,
    Flag = "KillAura",
    Callback = function(Value)
        Flags.KillAura = Value
        if Value then startKillAura() end
    end
})

ExplosionsTab:CreateButton({
    Name = "Break the Map (teleport down)",
    Callback = function()
        local char = LocalPlayer.Character
        if char and char:FindFirstChild("HumanoidRootPart") then
            pcall(function() char.HumanoidRootPart.CFrame = CFrame.new(0, -50, 0) end)
        end
    end
})

ExplosionsTab:CreateButton({
    Name = "Kill All Enemies",
    Callback = function()
        for _, obj in pairs(safeGetChildren(Workspace:FindFirstChild("Characters") or Workspace)) do
            if obj:IsA("Model") and obj.PrimaryPart and obj.Name ~= LocalPlayer.Name then
                pcall(function()
                    local re = safeFindRemote("DamageEnemy")
                    if re then re:FireServer(obj, 1000) end
                end)
            end
        end
    end
})

ExplosionsTab:CreateButton({
    Name = "Spawn All Metals to Machine",
    Callback = function()
        local machine = Workspace:FindFirstChild("MetalMachine")
        if not (machine and machine.PrimaryPart) then return end
        for _, obj in pairs(safeGetChildren(Workspace:FindFirstChild("Items") or Workspace)) do
            if obj:IsA("Model") and obj.Name:match("Scrap") and obj.PrimaryPart then
                pcall(function()
                    obj.PrimaryPart.CFrame = machine.PrimaryPart.CFrame
                    DragItem(obj)
                end)
            end
        end
    end
})

ExplosionsTab:CreateButton({
    Name = "Spawn All Wood to Machine",
    Callback = function()
        local machine = Workspace:FindFirstChild("WoodMachine")
        if not (machine and machine.PrimaryPart) then return end
        for _, obj in pairs(safeGetChildren(Workspace:FindFirstChild("Items") or Workspace)) do
            if obj:IsA("Model") and obj.Name == "Log" and obj.PrimaryPart then
                pcall(function()
                    obj.PrimaryPart.CFrame = machine.PrimaryPart.CFrame
                    DragItem(obj)
                end)
            end
        end
    end
})

ExplosionsTab:CreateButton({
    Name = "All Fuels to Fire",
    Callback = function()
        local fire = Workspace:FindFirstChild("Fire")
        if not (fire and fire.PrimaryPart) then return end
        for _, obj in pairs(safeGetChildren(Workspace:FindFirstChild("Items") or Workspace)) do
            if obj:IsA("Model") and (obj.Name == "Petrol" or obj.Name == "Gas Can" or obj.Name == "Coal" or obj.Name == "Log" or obj.Name:match("Corpse")) and obj.PrimaryPart then
                pcall(function()
                    obj.PrimaryPart.CFrame = fire.PrimaryPart.CFrame
                    DragItem(obj)
                end)
            end
        end
    end
})

ExplosionsTab:CreateButton({
    Name = "AFK Mode (auto-eat + anti-AFK)",
    Callback = function()
        local char = LocalPlayer.Character
        if char and char:FindFirstChild("HumanoidRootPart") then
            pcall(function() char.HumanoidRootPart.CFrame = CFrame.new(0, 100, 0) end)
            task.spawn(function()
                while true do
                    pcall(function()
                        local reEat = safeFindRemote("EatFood")
                        if reEat then reEat:FireServer("Food") end
                        local reAnti = safeFindRemote("AntiAfk")
                        if reAnti then reAnti:FireServer() end
                    end)
                    task.wait(60)
                end
            end)
        end
    end
})

-- ====== ITEMS TAB ======
local ItemsTab = Window:CreateTab("Items")

ItemsTab:CreateButton({
    Name = "Spawn Woods (to you)",
    Callback = function()
        local char = LocalPlayer.Character
        if not (char and char:FindFirstChild("HumanoidRootPart")) then return end
        for _, obj in pairs(safeGetChildren(Workspace:FindFirstChild("Items") or Workspace)) do
            if obj:IsA("Model") and obj.Name == "Log" and obj.PrimaryPart then
                pcall(function()
                    obj.PrimaryPart.CFrame = char.HumanoidRootPart.CFrame
                    DragItem(obj)
                end)
            end
        end
    end
})

ItemsTab:CreateButton({
    Name = "Spawn Bandages/Medkits",
    Callback = function()
        local char = LocalPlayer.Character
        if not (char and char:FindFirstChild("HumanoidRootPart")) then return end
        for _, obj in pairs(safeGetChildren(Workspace:FindFirstChild("Items") or Workspace)) do
            if obj:IsA("Model") and (obj.Name == "Medkit" or obj.Name == "Bandage") and obj.PrimaryPart then
                pcall(function()
                    obj.PrimaryPart.CFrame = char.HumanoidRootPart.CFrame
                    DragItem(obj)
                end)
            end
        end
    end
})

ItemsTab:CreateButton({
    Name = "Spawn Fuels",
    Callback = function()
        local char = LocalPlayer.Character
        if not (char and char:FindFirstChild("HumanoidRootPart")) then return end
        for _, obj in pairs(safeGetChildren(Workspace:FindFirstChild("Items") or Workspace)) do
            if obj:IsA("Model") and (obj.Name == "Petrol" or obj.Name == "Gas Can" or obj.Name == "Coal" or obj.Name == "Log" or obj.Name:match("Corpse")) and obj.PrimaryPart then
                pcall(function()
                    obj.PrimaryPart.CFrame = char.HumanoidRootPart.CFrame
                    DragItem(obj)
                end)
            end
        end
    end
})

ItemsTab:CreateButton({
    Name = "Spawn Foods",
    Callback = function()
        local char = LocalPlayer.Character
        if not (char and char:FindFirstChild("HumanoidRootPart")) then return end
        for _, obj in pairs(safeGetChildren(Workspace:FindFirstChild("Items") or Workspace)) do
            if obj:IsA("Model") and obj.Name:match("Food") and obj.PrimaryPart then
                pcall(function()
                    obj.PrimaryPart.CFrame = char.HumanoidRootPart.CFrame
                    DragItem(obj)
                end)
            end
        end
    end
})

-- ====== OTHER TAB ======
local OtherTab = Window:CreateTab("Other")

OtherTab:CreateButton({
    Name = "Skip Players/Chests Cooldown",
    Callback = function()
        pcall(function()
            local re = ReplicatedStorage:FindFirstChild("RemoteEvents")
            if re and re:FindFirstChild("SkipCooldown") then
                re.SkipCooldown:FireServer("PlayerHeal")
                re.SkipCooldown:FireServer("ChestOpen")
            end
        end)
    end
})

OtherTab:CreateButton({
    Name = "Notify: Test",
    Callback = function()
        Rayfield:Notify({
            Title = "MHX HUB",
            Content = "YT: YuriOnTop — Script running",
            Duration = 4
        })
    end
})

-- ====== FINAL NOTIFY ======
Rayfield:Notify({
    Title = "MHX HUB Loaded",
    Content = "Version 1.2 — YT: YuriOnTop",
    Duration = 4,
    Image = "rewind"
})

-- ====== CLEANUP ON CHARACTER ADD ======
LocalPlayer.CharacterAdded:Connect(function(char)
    task.wait(0.5)
    -- restore or reapply walk speed/god as needed
    if Flags.WalkSpeed then
        task.wait(0.2)
        local hum = char:FindFirstChildOfClass("Humanoid")
        if hum then pcall(function() hum.WalkSpeed = 150 end) end
    end
    if Flags.GodMode then
        task.wait(0.2)
        local hum = char:FindFirstChildOfClass("Humanoid")
        if hum then pcall(function() hum.Health = hum.MaxHealth end) end
    end
end)
