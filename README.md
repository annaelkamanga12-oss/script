loadstring(game:HttpGet("https://sirius.menu/rayfield"))() -- preload check

-- Unnamed Enhancements | Rivals v2.0
-- Paste this entire block into your executor

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TeleportService = game:GetService("TeleportService")
local Lighting = game:GetService("Lighting")
local VirtualUser = game:GetService("VirtualUser")
local lp = Players.LocalPlayer

local ESP = loadstring(game:HttpGet("https://raw.githubusercontent.com/linemaster2/esp-library/main/library.lua"))()
ESP.Enabled = false
ESP.ShowBox = false
ESP.ShowName = false
ESP.ShowHealth = false
ESP.ShowTracer = false
ESP.ShowDistance = false
ESP.ShowSkeletons = false
ESP.TeamCheck = false

local Rayfield = loadstring(game:HttpGet("https://sirius.menu/rayfield"))()

local State = {
    aimbot = false, silentAim = false, prediction = true,
    aimPart = "Head", fov = 305, smoothness = 0.71,
    hitChance = 100, wallCheck = false, teamCheck = true,
    targetNPCs = false, triggerbot = false, triggerKey = "MB2",
    reactionTime = 25, reactionOffset = 5, forgetTime = 1.5,
    shootDelay = 0, maxDist = 250, ragebot = false,
    voidSpam = false, hiderTime = 1, attackTime = 0.04,
    shootAttempts = 2, noSpread = false, fullAuto = false,
    alwaysBackstab = false, firerate = 10, espEnabled = false,
    espBox = false, espFill = false, espSkeleton = false,
    espName = false, espWeapon = false, espDistance = false,
    espHealthbar = false, espTeamCheck = false, chams = false,
    crosshair = false, crosshairStyle = "Cross", crosshairSize = 10,
    hitEffects = false, removeHitSound = false, infiniteJump = false,
    noclip = false, fly = false, flySpeed = 60, walkSpeed = 16,
    jumpPower = 50, bigHeads = false, antiVoid = false,
    humanoidLock = false, autoParry = false, autoDash = false,
    antiAFK = false, fullbright = false, noFog = false,
    autoRejoin = false,
}

local aimbotConn, noclipConn, flyConn, antiVoidConn, humanoidLockConn
local triggerbotConn, chamConns, crosshairGui

local function getChar() return lp.Character or lp.CharacterAdded:Wait() end
local function getHum()
    local c = getChar()
    return c and c:FindFirstChildOfClass("Humanoid")
end
local function notify(t, c, d)
    Rayfield:Notify({ Title = t, Content = c, Duration = d or 3 })
end

-- ============================================================
-- CLOSEST TARGET
-- ============================================================
local function getClosestTarget()
    local cam = workspace.CurrentCamera
    local char = lp.Character
    if not char then return nil end
    local nearest, shortest = nil, math.huge
    local center = Vector2.new(cam.ViewportSize.X / 2, cam.ViewportSize.Y / 2)

    local function check(model)
        if not model or not model:IsA("Model") then return end
        local hum = model:FindFirstChildOfClass("Humanoid")
        local part = model:FindFirstChild(State.aimPart)
        if not hum or not part or hum.Health <= 0 then return end
        local screenPos, onScreen = cam:WorldToViewportPoint(part.Position)
        if not onScreen then return end
        local dist2D = (Vector2.new(screenPos.X, screenPos.Y) - center).Magnitude
        if dist2D > State.fov then return end
        if State.wallCheck then
            local params = RaycastParams.new()
            params.FilterDescendantsInstances = {char}
            params.FilterType = Enum.RaycastFilterType.Blacklist
            local result = workspace:Raycast(cam.CFrame.Position, (part.Position - cam.CFrame.Position).Unit * 1000, params)
            if not result or not result.Instance:IsDescendantOf(model) then return end
        end
        if dist2D < shortest then shortest = dist2D nearest = model end
    end

    for _, player in pairs(Players:GetPlayers()) do
        if player ~= lp then
            if State.teamCheck and player.Team == lp.Team then continue end
            check(player.Character)
        end
    end
    if State.targetNPCs then
        for _, obj in pairs(workspace:GetDescendants()) do
            if obj:IsA("Model") and not Players:GetPlayerFromCharacter(obj) then check(obj) end
        end
    end
    return nearest
end

-- ============================================================
-- AIMBOT
-- ============================================================
local function startAimbot()
    if aimbotConn then aimbotConn:Disconnect() end
    aimbotConn = RunService.RenderStepped:Connect(function()
        if not State.aimbot then aimbotConn:Disconnect() return end
        if math.random(1, 100) > State.hitChance then return end
        local target = getClosestTarget()
        if not target then return end
        local part = target:FindFirstChild(State.aimPart)
        if not part then return end
        local cam = workspace.CurrentCamera
        local targetPos = part.Position
        if State.prediction then
            local vel = part.AssemblyLinearVelocity or Vector3.zero
            targetPos = targetPos + vel * 0.07
        end
        cam.CFrame = cam.CFrame:Lerp(CFrame.new(cam.CFrame.Position, targetPos), State.smoothness)
    end)
end

-- ============================================================
-- TRIGGERBOT
-- ============================================================
local function startTriggerbot()
    if triggerbotConn then triggerbotConn:Disconnect() end
    triggerbotConn = RunService.Heartbeat:Connect(function()
        if not State.triggerbot then triggerbotConn:Disconnect() return end

        local keyHeld = false
        if State.triggerKey == "Always On" then
            keyHeld = true
        elseif State.triggerKey == "MB2" then
            keyHeld = UserInputService:IsMouseButtonPressed(Enum.UserInputType.MouseButton2)
        elseif State.triggerKey == "MB1" then
            keyHeld = UserInputService:IsMouseButtonPressed(Enum.UserInputType.MouseButton1)
        else
            local keyMap = {Q=Enum.KeyCode.Q, E=Enum.KeyCode.E, F=Enum.KeyCode.F, X=Enum.KeyCode.X}
            local kc = keyMap[State.triggerKey]
            if kc then keyHeld = UserInputService:IsKeyDown(kc) end
        end
        if not keyHeld then return end

        local cam = workspace.CurrentCamera
        local char = lp.Character
        if not char then return end
        local unitRay = cam:ScreenPointToRay(cam.ViewportSize.X / 2, cam.ViewportSize.Y / 2)
        local params = RaycastParams.new()
        params.FilterDescendantsInstances = {char}
        params.FilterType = Enum.RaycastFilterType.Blacklist
        local result = workspace:Raycast(unitRay.Origin, unitRay.Direction * State.maxDist, params)
        if not result then return end

        local hitModel = result.Instance and result.Instance.Parent
        if not hitModel then return end
        local hum = hitModel:FindFirstChildOfClass("Humanoid")
        if not hum or hum.Health <= 0 then return end
        local hitPlayer = Players:GetPlayerFromCharacter(hitModel)
        if not hitPlayer then return end
        if State.teamCheck and hitPlayer.Team == lp.Team then return end

        local delay = (State.reactionTime + math.random(0, State.reactionOffset)) / 1000
        task.wait(delay)
        if State.shootDelay > 0 then task.wait(State.shootDelay / 1000) end

        local mouse = lp:GetMouse()
        mouse:Button1Down()
        task.wait(0.05)
        mouse:Button1Up()
    end)
end

-- ============================================================
-- FLY
-- ============================================================
local function startFly()
    if flyConn then flyConn:Disconnect() end
    local char = getChar()
    local root = char and char:FindFirstChild("HumanoidRootPart")
    local hum = char and char:FindFirstChildOfClass("Humanoid")
    if not root or not hum then return end

    hum.PlatformStand = true

    local bv = Instance.new("BodyVelocity")
    bv.MaxForce = Vector3.new(1e9, 1e9, 1e9)
    bv.Velocity = Vector3.zero
    bv.Parent = root

    local bg = Instance.new("BodyGyro")
    bg.MaxTorque = Vector3.new(1e9, 1e9, 1e9)
    bg.CFrame = root.CFrame
    bg.Parent = root

    flyConn = RunService.RenderStepped:Connect(function()
        if not State.fly then
            flyConn:Disconnect()
            bv:Destroy()
            bg:Destroy()
            hum.PlatformStand = false
            return
        end

        local cam = workspace.CurrentCamera
        local speed = State.flySpeed or 60
        local moveVec = Vector3.zero

        if UserInputService:IsKeyDown(Enum.KeyCode.W) then
            moveVec = moveVec + cam.CFrame.LookVector
        end
        if UserInputService:IsKeyDown(Enum.KeyCode.S) then
            moveVec = moveVec - cam.CFrame.LookVector
        end
        if UserInputService:IsKeyDown(Enum.KeyCode.A) then
            moveVec = moveVec - cam.CFrame.RightVector
        end
        if UserInputService:IsKeyDown(Enum.KeyCode.D) then
            moveVec = moveVec + cam.CFrame.RightVector
        end
        if UserInputService:IsKeyDown(Enum.KeyCode.Space) then
            moveVec = moveVec + Vector3.new(0, 1, 0)
        end
        if UserInputService:IsKeyDown(Enum.KeyCode.LeftControl) then
            moveVec = moveVec - Vector3.new(0, 1, 0)
        end

        if moveVec.Magnitude > 0 then
            bv.Velocity = moveVec.Unit * speed
        else
            bv.Velocity = Vector3.zero
        end
        bg.CFrame = cam.CFrame
    end)
end

-- ============================================================
-- NOCLIP
-- ============================================================
local function startNoclip()
    if noclipConn then noclipConn:Disconnect() end
    noclipConn = RunService.Stepped:Connect(function()
        if not State.noclip then noclipConn:Disconnect() return end
        local char = lp.Character
        if not char then return end
        for _, p in pairs(char:GetDescendants()) do
            if p:IsA("BasePart") then p.CanCollide = false end
        end
    end)
end

-- ============================================================
-- ANTI VOID
-- ============================================================
local function startAntiVoid()
    if antiVoidConn then antiVoidConn:Disconnect() end
    antiVoidConn = RunService.Heartbeat:Connect(function()
        if not State.antiVoid then antiVoidConn:Disconnect() return end
        local char = lp.Character
        if not char then return end
        local root = char:FindFirstChild("HumanoidRootPart")
        if root and root.Position.Y < -400 then
            root.CFrame = CFrame.new(root.Position.X, 10, root.Position.Z)
        end
    end)
end

-- ============================================================
-- HUMANOID LOCK
-- ============================================================
local function startHumanoidLock()
    if humanoidLockConn then humanoidLockConn:Disconnect() end
    humanoidLockConn = RunService.Heartbeat:Connect(function()
        if not State.humanoidLock then humanoidLockConn:Disconnect() return end
        local hum = getHum()
        if hum then
            hum.WalkSpeed = State.walkSpeed
            hum.JumpPower = State.jumpPower
        end
    end)
end

-- ============================================================
-- CHAMS (Highlight through walls)
-- ============================================================
chamConns = {}
local function applyChams(v)
    for _, c in pairs(chamConns) do pcall(function() c:Disconnect() end) end
    chamConns = {}

    if not v then
        for _, player in pairs(Players:GetPlayers()) do
            if player ~= lp and player.Character then
                local hl = player.Character:FindFirstChildOfClass("Highlight")
                if hl then hl:Destroy() end
            end
        end
        return
    end

    local function addHighlight(char)
        if not char then return end
        local existing = char:FindFirstChildOfClass("Highlight")
        if existing then existing:Destroy() end
        local hl = Instance.new("Highlight")
        hl.FillColor = Color3.fromRGB(255, 80, 180)
        hl.OutlineColor = Color3.fromRGB(255, 255, 255)
        hl.FillTransparency = 0.5
        hl.OutlineTransparency = 0
        hl.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
        hl.Parent = char
    end

    for _, player in pairs(Players:GetPlayers()) do
        if player ~= lp then
            addHighlight(player.Character)
            local c = player.CharacterAdded:Connect(addHighlight)
            table.insert(chamConns, c)
        end
    end

    local c2 = Players.PlayerAdded:Connect(function(player)
        if player == lp then return end
        addHighlight(player.Character)
        local c = player.CharacterAdded:Connect(addHighlight)
        table.insert(chamConns, c)
    end)
    table.insert(chamConns, c2)
end

-- ============================================================
-- CROSSHAIR (Drawing)
-- ============================================================
local crosshairLines = {}
local function destroyCrosshair()
    for _, line in pairs(crosshairLines) do
        pcall(function() line:Remove() end)
    end
    crosshairLines = {}
end

local function buildCrosshair()
    destroyCrosshair()
    if not State.crosshair then return end

    local style = State.crosshairStyle or "Cross"
    local size  = State.crosshairSize or 10
    local cx    = workspace.CurrentCamera.ViewportSize.X / 2
    local cy    = workspace.CurrentCamera.ViewportSize.Y / 2
    local color = Color3.fromRGB(255, 80, 180)
    local thick = 1.5

    if style == "Dot" then
        local dot = Drawing.new("Circle")
        dot.Visible   = true
        dot.Radius     = 3
        dot.Position   = Vector2.new(cx, cy)
        dot.Color      = color
        dot.Filled     = true
        dot.Thickness  = 1
        table.insert(crosshairLines, dot)

    elseif style == "Circle" then
        local circle = Drawing.new("Circle")
        circle.Visible   = true
        circle.Radius     = size
        circle.Position   = Vector2.new(cx, cy)
        circle.Color      = color
        circle.Filled     = false
        circle.Thickness  = thick
        table.insert(crosshairLines, circle)

    elseif style == "Cross" or style == "T-Shape" or style == "Dynamic" then
        local lines = {
            { Vector2.new(cx - size, cy), Vector2.new(cx - 3, cy) },
            { Vector2.new(cx + 3,    cy), Vector2.new(cx + size, cy) },
            { Vector2.new(cx, cy - size), Vector2.new(cx, cy - 3) },
        }
        if style ~= "T-Shape" then
            table.insert(lines, { Vector2.new(cx, cy + 3), Vector2.new(cx, cy + size) })
        end
        for _, pts in ipairs(lines) do
            local l = Drawing.new("Line")
            l.Visible   = true
            l.From      = pts[1]
            l.To        = pts[2]
            l.Color     = color
            l.Thickness = thick
            table.insert(crosshairLines, l)
        end
    end
end

-- Update crosshair on viewport resize
RunService.RenderStepped:Connect(function()
    if not State.crosshair then return end
    for _, line in pairs(crosshairLines) do
        if not line.Visible then line.Visible = true end
    end
end)

-- ============================================================
-- FOV CIRCLE
-- ============================================================
local fovCircle = Drawing.new("Circle")
fovCircle.Visible   = false
fovCircle.Thickness = 1
fovCircle.Color     = Color3.fromRGB(255, 255, 255)
fovCircle.Filled    = false
fovCircle.Transparency = 1

RunService.RenderStepped:Connect(function()
    if State.aimbot or State.silentAim then
        local cam = workspace.CurrentCamera
        fovCircle.Visible  = true
        fovCircle.Radius   = State.fov
        fovCircle.Position = Vector2.new(cam.ViewportSize.X / 2, cam.ViewportSize.Y / 2)
    else
        fovCircle.Visible = false
    end
end)

-- ============================================================
-- INFINITE JUMP
-- ============================================================
UserInputService.JumpRequest:Connect(function()
    if State.infiniteJump then
        local hum = getHum()
        if hum then hum:ChangeState(Enum.HumanoidStateType.Jumping) end
    end
end)

-- ============================================================
-- ANTI AFK
-- ============================================================
lp.Idled:Connect(function()
    if State.antiAFK then
        VirtualUser:Button2Down(Vector2.new(0, 0), workspace.CurrentCamera.CFrame)
        task.wait(1)
        VirtualUser:Button2Up(Vector2.new(0, 0), workspace.CurrentCamera.CFrame)
    end
end)

-- ============================================================
-- WINDOW
-- ============================================================
local Window = Rayfield:CreateWindow({
    Name = "Unnamed Enhancements",
    Icon = 0,
    LoadingTitle = "Unnamed Enhancements",
    LoadingSubtitle = "discord.gg/enhancements",
    Theme = "Ocean",
    DisableRayfieldPrompts = false,
    DisableBuildWarnings = false,
    ConfigurationSaving = {
        Enabled = true,
        FolderName = "UnnamedEnhancements",
        FileName = "Rivals"
    },
    Discord = { Enabled = false },
    KeySystem = false
})

-- ============================================================
-- TAB: MAIN
-- ============================================================
local MainTab = Window:CreateTab("Main", 4483362458)

MainTab:CreateSection("Aimbot")

MainTab:CreateToggle({
    Name = "Aimbot",
    CurrentValue = false,
    Flag = "Aimbot",
    Callback = function(v)
        State.aimbot = v
        if v then startAimbot() end
    end
})

MainTab:CreateToggle({
    Name = "Prediction",
    CurrentValue = true,
    Flag = "Prediction",
    Callback = function(v) State.prediction = v end
})

MainTab:CreateToggle({
    Name = "Wall Check",
    CurrentValue = false,
    Flag = "AimWallCheck",
    Callback = function(v) State.wallCheck = v end
})

MainTab:CreateToggle({
    Name = "Team Check",
    CurrentValue = true,
    Flag = "AimTeamCheck",
    Callback = function(v) State.teamCheck = v end
})

MainTab:CreateToggle({
    Name = "Target NPCs",
    CurrentValue = false,
    Flag = "TargetNPCs",
    Callback = function(v) State.targetNPCs = v end
})

MainTab:CreateDropdown({
    Name = "Aim Part",
    Options = {"Head", "HumanoidRootPart", "Torso", "Neck"},
    CurrentOption = {"Head"},
    MultipleOptions = false,
    Flag = "AimPart",
    Callback = function(v)
        State.aimPart = v
        notify("Aim Part", "Now targeting: " .. v, 2)
    end
})

MainTab:CreateDropdown({
    Name = "Aim Key",
    Options = {"Always On", "RMB", "LMB", "Q", "E", "F"},
    CurrentOption = {"Always On"},
    MultipleOptions = false,
    Flag = "AimKey",
    Callback = function(v)
        State.aimKey = v
    end
})

MainTab:CreateSlider({
    Name = "FOV Radius",
    Range = {10, 500},
    Increment = 5,
    Suffix = "px",
    CurrentValue = 305,
    Flag = "FovRadius",
    Callback = function(v) State.fov = v end
})

MainTab:CreateSlider({
    Name = "Smoothness",
    Range = {1, 100},
    Increment = 1,
    Suffix = "%",
    CurrentValue = 71,
    Flag = "AimSmooth",
    Callback = function(v) State.smoothness = v / 100 end
})

MainTab:CreateSlider({
    Name = "Hit Chance",
    Range = {1, 100},
    Increment = 1,
    Suffix = "%",
    CurrentValue = 100,
    Flag = "HitChance",
    Callback = function(v) State.hitChance = v end
})

MainTab:CreateSection("Triggerbot")

MainTab:CreateToggle({
    Name = "Triggerbot",
    CurrentValue = false,
    Flag = "Triggerbot",
    Callback = function(v)
        State.triggerbot = v
        if v then startTriggerbot() end
    end
})

MainTab:CreateDropdown({
    Name = "Trigger Key",
    Options = {"Always On", "MB2", "MB1", "Q", "E", "F"},
    CurrentOption = {"MB2"},
    MultipleOptions = false,
    Flag = "TriggerKey",
    Callback = function(v) State.triggerKey = v end
})

MainTab:CreateSlider({
    Name = "Reaction Time",
    Range = {0, 500},
    Increment = 5,
    Suffix = "ms",
    CurrentValue = 25,
    Flag = "ReactionTime",
    Callback = function(v) State.reactionTime = v end
})

MainTab:CreateSlider({
    Name = "Reaction Offset",
    Range = {0, 100},
    Increment = 1,
    Suffix = "ms",
    CurrentValue = 5,
    Flag = "ReactionOffset",
    Callback = function(v) State.reactionOffset = v end
})

MainTab:CreateSlider({
    Name = "Shoot Delay",
    Range = {0, 500},
    Increment = 5,
    Suffix = "ms",
    CurrentValue = 0,
    Flag = "ShootDelay",
    Callback = function(v) State.shootDelay = v end
})

MainTab:CreateSlider({
    Name = "Max Distance",
    Range = {50, 2000},
    Increment = 25,
    Suffix = "st",
    CurrentValue = 250,
    Flag = "TriggerMaxDist",
    Callback = function(v) State.maxDist = v end
})

-- ============================================================
-- TAB: WORLD
-- ============================================================
local WorldTab = Window:CreateTab("World", 4483362458)

WorldTab:CreateSection("Environment")

WorldTab:CreateToggle({
    Name = "Fullbright",
    CurrentValue = false,
    Flag = "Fullbright",
    Callback = function(v)
        State.fullbright = v
        if v then
            Lighting.Brightness = 2
            Lighting.ClockTime = 14
            Lighting.GlobalShadows = false
            Lighting.Ambient = Color3.new(1, 1, 1)
            Lighting.OutdoorAmbient = Color3.new(1, 1, 1)
        else
            Lighting.Brightness = 1
            Lighting.GlobalShadows = true
            Lighting.Ambient = Color3.new(0.5, 0.5, 0.5)
            Lighting.OutdoorAmbient = Color3.new(0.5, 0.5, 0.5)
        end
    end
})

WorldTab:CreateToggle({
    Name = "No Fog",
    CurrentValue = false,
    Flag = "NoFog",
    Callback = function(v)
        State.noFog = v
        Lighting.FogEnd   = v and 100000 or 1000
        Lighting.FogStart = v and 99999  or 0
    end
})

WorldTab:CreateToggle({
    Name = "No Sky",
    CurrentValue = false,
    Flag = "NoSky",
    Callback = function(v)
        local sky = Lighting:FindFirstChildOfClass("Sky")
        if sky then sky.Parent = v and game:GetService("ServerStorage") or Lighting end
    end
})

WorldTab:CreateSlider({
    Name = "Brightness",
    Range = {0, 10},
    Increment = 1,
    Suffix = "",
    CurrentValue = 1,
    Flag = "Brightness",
    Callback = function(v) Lighting.Brightness = v end
})

WorldTab:CreateSlider({
    Name = "Clock Time",
    Range = {0, 24},
    Increment = 1,
    Suffix = "h",
    CurrentValue = 14,
    Flag = "ClockTime",
    Callback = function(v) Lighting.ClockTime = v end
})

-- ============================================================
-- TAB: ESP
-- ============================================================
local ESPTab = Window:CreateTab("ESP", 4483362458)

ESPTab:CreateSection("Player Options")

ESPTab:CreateToggle({
    Name = "Enable ESP",
    CurrentValue = false,
    Flag = "ESPEnabled",
    Callback = function(v)
        State.espEnabled = v
        ESP.Enabled = v
    end
})

ESPTab:CreateToggle({
    Name = "Box",
    CurrentValue = false,
    Flag = "ESPBox",
    Callback = function(v) State.espBox = v ESP.ShowBox = v end
})

ESPTab:CreateToggle({
    Name = "Skeleton",
    CurrentValue = false,
    Flag = "ESPSkeleton",
    Callback = function(v) State.espSkeleton = v ESP.ShowSkeletons = v end
})

ESPTab:CreateToggle({
    Name = "Name",
    CurrentValue = false,
    Flag = "ESPName",
    Callback = function(v) State.espName = v ESP.ShowName = v end
})

ESPTab:CreateToggle({
    Name = "Distance",
    CurrentValue = false,
    Flag = "ESPDistance",
    Callback = function(v) State.espDistance = v ESP.ShowDistance = v end
})

ESPTab:CreateToggle({
    Name = "Health Bar",
    CurrentValue = false,
    Flag = "ESPHealth",
    Callback = function(v) State.espHealthbar = v ESP.ShowHealth = v end
})

ESPTab:CreateToggle({
    Name = "Tracers",
    CurrentValue = false,
    Flag = "ESPTracer",
    Callback = function(v) ESP.ShowTracer = v end
})

ESPTab:CreateToggle({
    Name = "Team Check",
    CurrentValue = false,
    Flag = "ESPTeamCheck",
    Callback = function(v) State.espTeamCheck = v ESP.TeamCheck = v end
})

ESPTab:CreateSection("Style")

ESPTab:CreateDropdown({
    Name = "Box Type",
    Options = {"2D", "Corner Box"},
    CurrentOption = {"2D"},
    MultipleOptions = false,
    Flag = "ESPBoxType",
    Callback = function(v) ESP.BoxType = v end
})

ESPTab:CreateDropdown({
    Name = "Tracer Origin",
    Options = {"Bottom", "Top", "Middle"},
    CurrentOption = {"Bottom"},
    MultipleOptions = false,
    Flag = "ESPTracerPos",
    Callback = function(v) ESP.TracerPosition = v end
})

-- ============================================================
-- TAB: VISUALS
-- ============================================================
local VisualsTab = Window:CreateTab("Visuals", 4483362458)

VisualsTab:CreateSection("Chams")

VisualsTab:CreateToggle({
    Name = "Chams (Highlight through walls)",
    CurrentValue = false,
    Flag = "Chams",
    Callback = function(v)
        State.chams = v
        applyChams(v)
    end
})

VisualsTab:CreateSection("Crosshair")

VisualsTab:CreateToggle({
    Name = "Custom Crosshair",
    CurrentValue = false,
    Flag = "Crosshair",
    Callback = function(v)
        State.crosshair = v
        if v then buildCrosshair() else destroyCrosshair() end
    end
})

VisualsTab:CreateDropdown({
    Name = "Crosshair Style",
    Options = {"Cross", "Dot", "Circle", "T-Shape"},
    CurrentOption = {"Cross"},
    MultipleOptions = false,
    Flag = "CrosshairStyle",
    Callback = function(v)
        State.crosshairStyle = v
        if State.crosshair then buildCrosshair() end
    end
})

VisualsTab:CreateSlider({
    Name = "Crosshair Size",
    Range = {3, 40},
    Increment = 1,
    Suffix = "px",
    CurrentValue = 10,
    Flag = "CrosshairSize",
    Callback = function(v)
        State.crosshairSize = v
        if State.crosshair then buildCrosshair() end
    end
})

VisualsTab:CreateSection("FOV Circle")

VisualsTab:CreateToggle({
    Name = "Show FOV Circle",
    CurrentValue = false,
    Flag = "ShowFOV",
    Callback = function(v)
        fovCircle.Visible = v
    end
})

-- ============================================================
-- TAB: CHARACTER
-- ============================================================
local CharTab = Window:CreateTab("Character", 4483362458)

CharTab:CreateSection("Movement")

CharTab:CreateToggle({
    Name = "Infinite Jump",
    CurrentValue = false,
    Flag = "InfiniteJump",
    Callback = function(v) State.infiniteJump = v end
})

CharTab:CreateToggle({
    Name = "No Clip",
    CurrentValue = false,
    Flag = "NoClip",
    Callback = function(v)
        State.noclip = v
        if v then startNoclip() end
    end
})

CharTab:CreateToggle({
    Name = "Fly",
    CurrentValue = false,
    Flag = "Fly",
    Callback = function(v)
        State.fly = v
        if v then
            startFly()
        else
            if flyConn then flyConn:Disconnect() end
            local char = lp.Character
            if char then
                local root = char:FindFirstChild("HumanoidRootPart")
                if root then
                    local bv = root:FindFirstChildOfClass("BodyVelocity")
                    local bg = root:FindFirstChildOfClass("BodyGyro")
                    if bv then bv:Destroy() end
                    if bg then bg:Destroy() end
                end
                local hum = char:FindFirstChildOfClass("Humanoid")
                if hum then hum.PlatformStand = false end
            end
        end
    end
})

CharTab:CreateSlider({
    Name = "Fly Speed",
    Range = {10, 300},
    Increment = 5,
    Suffix = "",
    CurrentValue = 60,
    Flag = "FlySpeed",
    Callback = function(v) State.flySpeed = v end
})

CharTab:CreateSlider({
    Name = "Walk Speed",
    Range = {0, 150},
    Increment = 1,
    Suffix = "",
    CurrentValue = 16,
    Flag = "WalkSpeed",
    Callback = function(v)
        State.walkSpeed = v
        local hum = getHum()
        if hum then hum.WalkSpeed = v end
    end
})

CharTab:CreateSlider({
    Name = "Jump Power",
    Range = {0, 300},
    Increment = 5,
    Suffix = "",
    CurrentValue = 50,
    Flag = "JumpPower",
    Callback = function(v)
        State.jumpPower = v
        local hum = getHum()
        if hum then hum.JumpPower = v end
    end
})

CharTab:CreateSection("Player")

CharTab:CreateToggle({
    Name = "Big Heads",
    CurrentValue = false,
    Flag = "BigHeads",
    Callback = function(v)
        State.bigHeads = v
        if v then
            for _, player in pairs(Players:GetPlayers()) do
                if player ~= lp and player.Character then
                    local head = player.Character:FindFirstChild("Head")
                    if head then
                        head.Size = Vector3.new(5, 5, 5)
                        head.CanCollide = false
                    end
                end
            end
        end
    end
})

CharTab:CreateToggle({
    Name = "Anti Void",
    CurrentValue = false,
    Flag = "AntiVoid",
    Callback = function(v)
        State.antiVoid = v
        if v then startAntiVoid() end
    end
})

CharTab:CreateToggle({
    Name = "Humanoid Lock",
    CurrentValue = false,
    Flag = "HumanoidLock",
    Callback = function(v)
        State.humanoidLock = v
        if v then startHumanoidLock() end
    end
})

CharTab:CreateSection("Abilities")

CharTab:CreateToggle({
    Name = "Anti AFK",
    CurrentValue = false,
    Flag = "AntiAFK",
    Callback = function(v) State.antiAFK = v end
})

-- ============================================================
-- TAB: MISC
-- ============================================================
local MiscTab = Window:CreateTab("Misc", 4483362458)

MiscTab:CreateSection("Utility")

MiscTab:CreateToggle({
    Name = "Auto Rejoin",
    CurrentValue = false,
    Flag = "AutoRejoin",
    Callback = function(v)
        State.autoRejoin = v
        if v then
            lp.OnTeleport:Connect(function(s)
                if s == Enum.TeleportState.Failed then
                    TeleportService:Teleport(game.PlaceId)
                end
            end)
        end
    end
})

MiscTab:CreateButton({
    Name = "Rejoin Server",
    Callback = function()
        TeleportService:Teleport(game.PlaceId)
    end
})

MiscTab:CreateButton({
    Name = "Copy Game ID",
    Callback = function()
        setclipboard(tostring(game.PlaceId))
        notify("Copied", "Game ID: " .. game.PlaceId, 3)
    end
})

-- ============================================================
-- TAB: SETTINGS
-- ============================================================
local SettingsTab = Window:CreateTab("Settings", 4483362458)

SettingsTab:CreateSection("Info")
SettingsTab:CreateLabel("Unnamed Enhancements | Rivals v2.0")
SettingsTab:CreateLabel("discord.gg/enhancements")
SettingsTab:CreateLabel("Author: x2zu")
SettingsTab:CreateLabel("UI: Rayfield by Sirius")

-- ============================================================
-- INIT
-- ============================================================
notify("Unnamed Enhancements", "Rivals loaded. Stay safe.", 5)
