-- [[ OLISE HUB - CUSTOM GUI ]] --
-- PARTE 1: SERVIÇOS, CONFIGURAÇÕES E FUNÇÕES AUXILIARES

-- Serviços
local Players = game:GetService("Players")
local Player = Players.LocalPlayer
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Workspace = game:GetService("Workspace")
local Camera = Workspace.CurrentCamera

-- Configurações e Estado
local cfg = {
    reach = 10,
    sphere = true,
    esp = true,
    touch = true,
    cd = 1.5,
    kick = 5,
    kickOn = false,
    spin = false,
    skillMode = false,
    spinSpeed = 3,
    skillSpeed = 5,
    autoFollow = false,
    magneticEnabled = false,
    magneticStrength = 50,
    fovEnabled = false,
    fovValue = 70,
    controlBallEnabled = false,
    controlBallTarget = nil,
    rgbPlayerEnabled = false,
    tpBallEnabled = false,
    autoCatch = false,
    catchIntensity = 5,
    trainingReach = 10,
    cooldown = 1.5,
    tpsAlert = false,
    espTrainingOnly = false,
    trainingMode = false
}

local balls = {}
local esps = {}
local lr = 0
local spinAngle = 0
local dt = 0
local sp = nil
local spectateEnabled = false
local spectateTarget = nil
local originalCameraCFrame = nil
local targetBall = nil
local char = nil
local humanoid = nil
local hrp = nil
local defaultFOV = 70
local rgbHue = 0
local rgbConnection = nil
local magneticWelds = {}
local bn = {TPS = true, ESA = true, MRS = true, PRS = true, MPS = true, SSS = true, AIFA = true, RBZ = true}
local tpsAlertActive = false
local tpsAlertGui = nil
local tpsDistance = 0
local floatBtn = nil
local autoFollowToggleObj = nil
local keybindKey = Enum.KeyCode.E
local keybindLabel = nil

-- Atualização de Personagem
local function updateChar()
    char = Player.Character
    if char then 
        humanoid = char:FindFirstChildOfClass("Humanoid")
        hrp = char:FindFirstChild("HumanoidRootPart")
    end
end

Player.CharacterAdded:Connect(function()
    task.wait(0.5)
    updateChar()
    if cfg.rgbPlayerEnabled and startRGBPlayer then
        startRGBPlayer()
    end
end)
updateChar()

-- Funções Auxiliares de Jogo
local function findClosestBall()
    if not hrp or not hrp.Parent then return nil end
    local closest = nil
    local closeDist = math.huge
    for _, v in ipairs(Workspace:GetDescendants()) do
        if v:IsA("BasePart") and (v.Name == "TPS" or (cfg.trainingMode and v.Name == "TrainingBall")) then
            local d = (v.Position - hrp.Position).Magnitude
            if d < closeDist then 
                closeDist = d
                closest = v 
            end
        end
    end
    return closest
end

local function findClosestTPS()
    if not hrp or not hrp.Parent then return nil end
    local closest = nil
    local closeDist = math.huge
    for _, v in ipairs(Workspace:GetDescendants()) do
        if v:IsA("BasePart") and v.Name == "TPS" then
            local d = (v.Position - hrp.Position).Magnitude
            if d < closeDist then 
                closeDist = d
                closest = v 
            end
        end
    end
    return closest
end

local function rb()
    if tick() - lr < cfg.cd then return end
    lr = tick()
    balls = {}
    for _, v in ipairs(Workspace:GetDescendants()) do
        if v:IsA("BasePart") and bn[v.Name] then 
            table.insert(balls, v) 
        end
    end
end

local function gp(c)
    local t = {}
    for _, v in ipairs(c:GetChildren()) do
        if v:IsA("BasePart") and v.Name ~= "HumanoidRootPart" then 
            table.insert(t, v) 
        end
    end
    return t
end

local function us()
    if not cfg.sphere then 
        if sp and sp.Parent then sp:Destroy() end
        sp = nil
        return 
    end
    if not sp or not sp.Parent then
        sp = Instance.new("Part")
        sp.Name = "CS"
        sp.Shape = Enum.PartType.Ball
        sp.Anchored = true
        sp.CanCollide = false
        sp.Transparency = 0.7
        sp.Material = Enum.Material.ForceField
        sp.Color = Color3.fromRGB(255, 50, 50)
        sp.Parent = Workspace
    end
    sp.Size = Vector3.new(cfg.reach * 2, cfg.reach * 2, cfg.reach * 2)
end

local function tp()
    if not Player.Character then return end
    local h = Player.Character:FindFirstChild("HumanoidRootPart")
    if not h then return end
    local cl = findClosestBall()
    if cl then 
        h.CFrame = cl.CFrame * CFrame.new(0, 3, 0)
    end
end

-- [[ OLISE HUB - CUSTOM GUI ]] --
-- PARTE 2: FUNÇÕES DE LÓGICA DAS FEATURES

local updateFloatBtnVisual

local function setAutoFollowState(state)
    cfg.autoFollow = state
    if cfg.autoFollow then 
        targetBall = findClosestBall() 
    else 
        targetBall = nil 
    end
    if updateFloatBtnVisual then updateFloatBtnVisual() end
end

local function doAutoFollow()
    if not cfg.autoFollow then return end
    if not humanoid or not hrp then updateChar() end
    if not humanoid or not hrp then return end
    if not targetBall or not targetBall.Parent then 
        targetBall = findClosestBall() 
    end
    if targetBall and targetBall.Parent then 
        humanoid:MoveTo(targetBall.Position) 
    end
end

function startRGBPlayer()
    if rgbConnection then rgbConnection:Disconnect() rgbConnection = nil end
    if not cfg.rgbPlayerEnabled then return end

    rgbConnection = RunService.RenderStepped:Connect(function()
        if not cfg.rgbPlayerEnabled then
            if rgbConnection then rgbConnection:Disconnect() rgbConnection = nil end
            return
        end
        rgbHue = (rgbHue + 0.5) % 360
        local color = Color3.fromHSV(rgbHue / 360, 1, 1)
        local ch = Player.Character
        if ch then
            for _, part in ipairs(ch:GetDescendants()) do
                if part:IsA("BasePart") and part.Name ~= "HumanoidRootPart" then
                    part.Color = color
                end
            end
        end
    end)
end

local function stopRGBPlayer()
    if rgbConnection then rgbConnection:Disconnect() rgbConnection = nil end
end

local function doTPBall()
    if not cfg.tpBallEnabled then return end
    if not hrp or not hrp.Parent then updateChar() end
    if not hrp or not hrp.Parent then return end

    local closest = findClosestBall()
    if closest then
        hrp.CFrame = closest.CFrame * CFrame.new(0, 3, 0)
    end
end

local function doMagneticBall()
    if not cfg.magneticEnabled then return end
    if not hrp or not hrp.Parent then updateChar() end
    if not hrp or not hrp.Parent then return end

    for _, ball in ipairs(balls) do
        if ball and ball.Parent then
            local dist = (ball.Position - hrp.Position).Magnitude
            if dist <= cfg.reach + 15 then
                if magneticWelds[ball] then
                    local offset = magneticWelds[ball]
                    local targetPos = hrp.CFrame * CFrame.new(offset) * CFrame.new(0, 0, -3)
                    ball.CFrame = targetPos
                    ball.Velocity = Vector3.zero
                    ball.RotVelocity = Vector3.zero
                else
                    local direction = (hrp.Position - ball.Position).Unit
                    local strength = math.min(cfg.magneticStrength / math.max(dist, 3) * 2, 30)
                    pcall(function()
                        local bv = ball:FindFirstChild("CLB_Mag")
                        if not bv then
                            bv = Instance.new("BodyVelocity")
                            bv.Name = "CLB_Mag"
                            bv.MaxForce = Vector3.new(50000, 50000, 50000)
                            bv.Parent = ball
                        end
                        bv.Velocity = direction * strength
                    end)

                    if dist < 5 then
                        pcall(function()
                            local bv = ball:FindFirstChild("CLB_Mag")
                            if bv then bv:Destroy() end
                        end)
                        local offset = (ball.Position - hrp.Position)
                        magneticWelds[ball] = offset
                        ball.CanCollide = false
                    end
                end
            else
                if magneticWelds[ball] then
                    magneticWelds[ball] = nil
                    ball.CanCollide = true
                end
            end
        end
    end
end

local function updateFOV()
    if cfg.fovEnabled then
        Camera.FieldOfView = cfg.fovValue
    else
        Camera.FieldOfView = defaultFOV
    end
end

local function doControlBall()
    if not cfg.controlBallEnabled then return end
    if not cfg.controlBallTarget or not cfg.controlBallTarget.Parent then
        cfg.controlBallTarget = findClosestBall()
        if not cfg.controlBallTarget then return end
    end
    if not hrp or not hrp.Parent then updateChar() end
    if not hrp then return end

    local ball = cfg.controlBallTarget
    local moveDirection = Vector3.zero

    if UserInputService:IsKeyDown(Enum.KeyCode.W) then moveDirection = moveDirection + Camera.CFrame.LookVector end
    if UserInputService:IsKeyDown(Enum.KeyCode.S) then moveDirection = moveDirection - Camera.CFrame.LookVector end
    if UserInputService:IsKeyDown(Enum.KeyCode.A) then moveDirection = moveDirection - Camera.CFrame.RightVector end
    if UserInputService:IsKeyDown(Enum.KeyCode.D) then moveDirection = moveDirection + Camera.CFrame.RightVector end
    if UserInputService:IsKeyDown(Enum.KeyCode.E) then moveDirection = moveDirection + Vector3.new(0, 1, 0) end
    if UserInputService:IsKeyDown(Enum.KeyCode.Q) then moveDirection = moveDirection - Vector3.new(0, 1, 0) end

    if moveDirection.Magnitude > 0 then
        moveDirection = moveDirection.Unit
        pcall(function()
            local bv = ball:FindFirstChild("CLB_Ctrl")
            if not bv then
                bv = Instance.new("BodyVelocity")
                bv.Name = "CLB_Ctrl"
                bv.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
                bv.Parent = ball
            end
            bv.Velocity = moveDirection * 50
            ball.RotVelocity = Vector3.zero
        end)
    end
end

local function spectatePlayer(targetPlayer)
    if not targetPlayer or targetPlayer == Player then return end
    if not targetPlayer.Character or not targetPlayer.Character:FindFirstChild("Head") then return end
    spectateEnabled = true
    spectateTarget = targetPlayer
    originalCameraCFrame = Camera.CFrame
    Camera.CameraSubject = targetPlayer.Character:FindFirstChildOfClass("Humanoid")
    Camera.CameraType = Enum.CameraType.Custom
end

local function stopSpectate()
    spectateEnabled = false
    spectateTarget = nil
    Camera.CameraSubject = Player.Character and Player.Character:FindFirstChildOfClass("Humanoid")
    Camera.CameraType = Enum.CameraType.Custom
    if Player.Character and Player.Character:FindFirstChild("HumanoidRootPart") then
        Camera.CFrame = originalCameraCFrame or Player.Character.HumanoidRootPart.CFrame * CFrame.new(0, 3, 10)
    end
end

local lastKickTime = 0
local function tryPowerKick(ball)
    if not cfg.kickOn or cfg.kick <= 0 then return end
    if tick() - lastKickTime < 0.5 then return end
    if not Player.Character then return end
    local hrp2 = Player.Character:FindFirstChild("HumanoidRootPart")
    if not hrp2 then return end
    local hum = Player.Character:FindFirstChildOfClass("Humanoid")
    if not hum or hum.MoveDirection.Magnitude < 0.1 then return end
    local dot = ((ball.Position - hrp2.Position).Unit):Dot(hum.MoveDirection.Unit)
    if dot > 0.5 then
        lastKickTime = tick()
        pcall(function()
            local old = ball:FindFirstChild("CLB_PK")
            if old then old:Destroy() end
            local bv = Instance.new("BodyVelocity")
            bv.Name = "CLB_PK"
            bv.Velocity = hrp2.CFrame.LookVector * cfg.kick * 45 + Vector3.new(0, cfg.kick * 9, 0)
            bv.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
            bv.Parent = ball
            task.delay(0.15, function() 
                pcall(function() if bv and bv.Parent then bv:Destroy() end end) 
            end)
        end)
    end
end

local function doSpin()
    if not cfg.spin then return end
    if not Player.Character then return end
    local hrp2 = Player.Character:FindFirstChild("HumanoidRootPart")
    if not hrp2 then return end
    spinAngle = spinAngle + (dt or 0.016) * cfg.spinSpeed * 2
    hrp2.CFrame = CFrame.new(hrp2.Position) * CFrame.Angles(0, spinAngle, 0)
end

local skillPhase = 0
local skillTimer = 0
local function doSkillMode()
    if not cfg.skillMode then return end
    if not Player.Character then return end
    local hrp2 = Player.Character:FindFirstChild("HumanoidRootPart")
    if not hrp2 then return end
    local closest = findClosestBall()
    if closest and (closest.Position - hrp2.Position).Magnitude < cfg.reach + 2 then
        skillTimer = skillTimer + (dt or 0.016) * cfg.skillSpeed
        if skillTimer >= 1 then 
            skillTimer = 0
            skillPhase = (skillPhase + 1) % 4 
        end
        local t = skillTimer
        local offset
        if skillPhase == 0 then offset = hrp2.CFrame.LookVector * math.sin(t * math.pi) * 2
        elseif skillPhase == 1 then offset = hrp2.CFrame.RightVector * math.sin(t * math.pi) * 2
        elseif skillPhase == 2 then offset = -hrp2.CFrame.RightVector * math.sin(t * math.pi) * 2
        else offset = -hrp2.CFrame.LookVector * math.sin(t * math.pi) * 2 end

        local targetPos = hrp2.Position + offset + Vector3.new(0, -1.5, 0)
        closest.CFrame = CFrame.new(closest.Position:Lerp(targetPos, 0.3), closest.Position)
        closest.Velocity = Vector3.zero
        closest.RotVelocity = Vector3.zero
    end
end

-- [[ OLISE HUB - CUSTOM GUI ]] --
-- PARTE 3: ALERTA TPS, SIMULAÇÃO E CRIAÇÃO DA GUI

local function createTPSAlert()
    if tpsAlertGui then
        tpsAlertGui:Destroy()
        tpsAlertGui = nil
    end

    tpsAlertGui = Instance.new("Frame")
    tpsAlertGui.Name = "TPSAlert"
    tpsAlertGui.Size = UDim2.new(0, 300, 0, 80)
    tpsAlertGui.Position = UDim2.new(0.5, -150, 0.1, 0)
    tpsAlertGui.BackgroundColor3 = Color3.fromRGB(255, 50, 50)
    tpsAlertGui.BackgroundTransparency = 0.15
    tpsAlertGui.BorderSizePixel = 0
    tpsAlertGui.Visible = false
    tpsAlertGui.Parent = Player:WaitForChild("PlayerGui")

    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 10)
    corner.Parent = tpsAlertGui

    local stroke = Instance.new("UIStroke")
    stroke.Color = Color3.fromRGB(255, 0, 0)
    stroke.Thickness = 3
    stroke.Parent = tpsAlertGui

    local title = Instance.new("TextLabel")
    title.Size = UDim2.new(1, -20, 0.5, 0)
    title.Position = UDim2.new(0, 10, 0, 5)
    title.BackgroundTransparency = 1
    title.Text = "⚠️ TPS PRÓXIMA!"
    title.TextColor3 = Color3.fromRGB(255, 255, 255)
    title.TextSize = 24
    title.Font = Enum.Font.SourceSansBold
    title.Parent = tpsAlertGui

    local distanceLabel = Instance.new("TextLabel")
    distanceLabel.Name = "DistanceLabel"
    distanceLabel.Size = UDim2.new(1, -20, 0.5, 0)
    distanceLabel.Position = UDim2.new(0, 10, 0.5, 0)
    distanceLabel.BackgroundTransparency = 1
    distanceLabel.Text = "Distância: 0.0 studs"
    distanceLabel.TextColor3 = Color3.fromRGB(255, 255, 200)
    distanceLabel.TextSize = 18
    distanceLabel.Font = Enum.Font.SourceSans
    distanceLabel.Parent = tpsAlertGui

    return tpsAlertGui
end

local function updateTPSAlert()
    if not cfg.tpsAlert then
        if tpsAlertGui then tpsAlertGui.Visible = false end
        tpsAlertActive = false
        return
    end

    if not hrp or not hrp.Parent then updateChar() end
    if not hrp then return end

    local closestTPS = findClosestTPS()

    if closestTPS then
        local dist = (closestTPS.Position - hrp.Position).Magnitude
        tpsDistance = dist

        if dist <= 3 then
            if not tpsAlertActive then
                if not tpsAlertGui then createTPSAlert() end
                tpsAlertGui.Visible = true
                tpsAlertActive = true
            end

            if tpsAlertGui then
                local distanceLabel = tpsAlertGui:FindFirstChild("DistanceLabel")
                if distanceLabel then
                    distanceLabel.Text = string.format("Distância: %.1f studs", dist)
                end

                tpsAlertGui.BackgroundTransparency = 0.1 + math.sin(tick() * 8) * 0.1
            end
        else
            if tpsAlertActive then
                tpsAlertGui.Visible = false
                tpsAlertActive = false
            end
        end
    else
        if tpsAlertActive then
            tpsAlertGui.Visible = false
            tpsAlertActive = false
        end
    end
end

local function simulateCatch(ball)
    if not cfg.autoCatch then return end
    if not ball or not ball.Parent then return end
    if not hrp or not hrp.Parent then return end

    local dist = (ball.Position - hrp.Position).Magnitude
    if dist <= cfg.trainingReach then
        pcall(function()
            local bv = ball:FindFirstChild("CatchSim")
            if not bv then
                bv = Instance.new("BodyVelocity")
                bv.Name = "CatchSim"
                bv.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
                bv.Parent = ball
            end
            local direction = (hrp.Position - ball.Position).Unit
            bv.Velocity = direction * cfg.catchIntensity * 2
            ball.RotVelocity = Vector3.zero

            task.delay(0.3, function()
                pcall(function()
                    if bv and bv.Parent then bv:Destroy() end
                end)
            end)
        end)
    end
end

-- ============================================
-- CRIAÇÃO DA GUI PERSONALIZADA (CUSTOM GUI)
-- ============================================
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "Olise_Hub_CustomGui"
ScreenGui.ResetOnSpawn = false
pcall(function() ScreenGui.Parent = Player:WaitForChild("PlayerGui") end)

local MainFrame = Instance.new("Frame")
MainFrame.Name = "MainFrame"
MainFrame.Size = UDim2.new(0, 500, 0, 400)
MainFrame.Position = UDim2.new(0.5, -250, 0.5, -200)
MainFrame.BackgroundColor3 = Color3.fromRGB(20, 20, 25)
MainFrame.BorderSizePixel = 0
MainFrame.Active = true
MainFrame.Draggable = true
MainFrame.Parent = ScreenGui

local MainCorner = Instance.new("UICorner")
MainCorner.CornerRadius = UDim.new(0, 10)
MainCorner.Parent = MainFrame

local MainStroke = Instance.new("UIStroke")
MainStroke.Color = Color3.fromRGB(255, 50, 50)
MainStroke.Thickness = 2
MainStroke.Parent = MainFrame

local TopBar = Instance.new("Frame")
TopBar.Size = UDim2.new(1, 0, 0, 40)
TopBar.BackgroundColor3 = Color3.fromRGB(15, 15, 20)
TopBar.BorderSizePixel = 0
TopBar.Parent = MainFrame

local TopBarCorner = Instance.new("UICorner")
TopBarCorner.CornerRadius = UDim.new(0, 10)
TopBarCorner.Parent = TopBar

local Title = Instance.new("TextLabel")
Title.Size = UDim2.new(1, -90, 1, 0)
Title.Position = UDim2.new(0, 15, 0, 0)
Title.BackgroundTransparency = 1
Title.Text = "OLISE HUB - V2.1"
Title.TextColor3 = Color3.fromRGB(255, 255, 255)
Title.TextSize = 18
Title.Font = Enum.Font.SourceSansBold
Title.TextXAlignment = Enum.TextXAlignment.Left
Title.Parent = TopBar

local MinimizeBtn = Instance.new("TextButton")
MinimizeBtn.Size = UDim2.new(0, 30, 0, 30)
MinimizeBtn.Position = UDim2.new(1, -70, 0, 5)
MinimizeBtn.BackgroundColor3 = Color3.fromRGB(60, 60, 75)
MinimizeBtn.Text = "-"
MinimizeBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
MinimizeBtn.Font = Enum.Font.SourceSansBold
MinimizeBtn.TextSize = 18
MinimizeBtn.Parent = TopBar

local MinimizeCorner = Instance.new("UICorner")
MinimizeCorner.CornerRadius = UDim.new(0, 6)
MinimizeCorner.Parent = MinimizeBtn

MinimizeBtn.MouseButton1Click:Connect(function()
    MainFrame.Visible = false
end)

local CloseBtn = Instance.new("T
