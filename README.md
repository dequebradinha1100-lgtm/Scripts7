-- [[ SMILE 7 HUB - RAYFIELD VERSION ]] --

-- Verificação e Carregamento da Rayfield UI
local Rayfield = loadstring(game:HttpGet('https://sirius.menu/rayfield'))()

local Window = Rayfield:CreateWindow({
   Name = "Smile 7 Hub",
   LoadingTitle = "Carregando Hub...",
   LoadingSubtitle = "by anonymus7",
   ConfigurationSaving = {
      Enabled = false,
   },
   KeySystem = false,
})

-- Serviços e Variáveis Globais
local Players = game:GetService("Players")
local Player = Players.LocalPlayer
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Workspace = game:GetService("Workspace")
local Camera = Workspace.CurrentCamera

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
local char, humanoid, hrp = nil, nil, nil
local defaultFOV = Camera.FieldOfView
local rgbHue = 0
local rgbConnection = nil
local magneticWelds = {}
local bn = {TPS = true, ESA = true, MRS = true, PRS = true, MPS = true, SSS = true, AIFA = true, RBZ = true}
local tpsAlertActive = false
local tpsAlertGui = nil

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
end)
updateChar()

-- Funções Auxiliares de Jogo
local function findClosestBall()
    if not hrp or not hrp.Parent then return nil end
    local closest, closeDist = nil, math.huge
    for _, v in ipairs(Workspace:GetDescendants()) do
        if v:IsA("BasePart") and (v.Name == "TPS" or (cfg.trainingMode and v.Name == "TrainingBall")) then
            local d = (v.Position - hrp.Position).Magnitude
            if d < closeDist then closeDist = d; closest = v end
        end
    end
    return closest
end

local function findClosestTPS()
    if not hrp or not hrp.Parent then return nil end
    local closest, closeDist = nil, math.huge
    for _, v in ipairs(Workspace:GetDescendants()) do
        if v:IsA("BasePart") and v.Name == "TPS" then
            local d = (v.Position - hrp.Position).Magnitude
            if d < closeDist then closeDist = d; closest = v end
        end
    end
    return closest
end

local function rb()
    if tick() - lr < cfg.cd then return end
    lr = tick()
    balls = {}
    for _, v in ipairs(Workspace:GetDescendants()) do
        if v:IsA("BasePart") and bn[v.Name] then table.insert(balls, v) end
    end
end

local function gp(c)
    local t = {}
    for _, v in ipairs(c:GetChildren()) do
        if v:IsA("BasePart") and v.Name ~= "HumanoidRootPart" then table.insert(t, v) end
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

-- Lógicas de Recursos
local function doAutoFollow()
    if not cfg.autoFollow then return end
    if not humanoid or not hrp then updateChar() end
    if not humanoid or not hrp then return end
    if not targetBall or not targetBall.Parent then targetBall = findClosestBall() end
    if targetBall and targetBall.Parent then humanoid:MoveTo(targetBall.Position) end
end

local function doMagneticBall()
    if not cfg.magneticEnabled then return end
    if not hrp or not hrp.Parent then updateChar() end
    if not hrp then return end

    for _, ball in ipairs(balls) do
        if ball and ball.Parent then
            local dist = (ball.Position - hrp.Position).Magnitude
            if dist <= cfg.reach + 15 then
                if magneticWelds[ball] then
                    ball.CFrame = hrp.CFrame * CFrame.new(magneticWelds[ball]) * CFrame.new(0, 0, -3)
                    ball.Velocity = Vector3.zero
                    ball.RotVelocity = Vector3.zero
                else
                    local direction = (hrp.Position - ball.Position).Unit
                    local strength = math.min(cfg.magneticStrength / math.max(dist, 3) * 2, 30)
                    pcall(function()
                        local bv = ball:FindFirstChild("CLB_Mag") or Instance.new("BodyVelocity", ball)
                        bv.Name = "CLB_Mag"
                        bv.MaxForce = Vector3.new(50000, 50000, 50000)
                        bv.Velocity = direction * strength
                    end)
                    if dist < 5 then
                        pcall(function() if ball:FindFirstChild("CLB_Mag") then ball.CLB_Mag:Destroy() end end)
                        magneticWelds[ball] = (ball.Position - hrp.Position)
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

local function doControlBall()
    if not cfg.controlBallEnabled then return end
    if not cfg.controlBallTarget or not cfg.controlBallTarget.Parent then
        cfg.controlBallTarget = findClosestBall()
        if not cfg.controlBallTarget then return end
    end
    local ball = cfg.controlBallTarget
    local moveDirection = Vector3.zero
    if UserInputService:IsKeyDown(Enum.KeyCode.W) then moveDirection = moveDirection + Camera.CFrame.LookVector end
    if UserInputService:IsKeyDown(Enum.KeyCode.S) then moveDirection = moveDirection - Camera.CFrame.LookVector end
    if UserInputService:IsKeyDown(Enum.KeyCode.A) then moveDirection = moveDirection - Camera.CFrame.RightVector end
    if UserInputService:IsKeyDown(Enum.KeyCode.D) then moveDirection = moveDirection + Camera.CFrame.RightVector end

    if moveDirection.Magnitude > 0 then
        pcall(function()
            local bv = ball:FindFirstChild("CLB_Ctrl") or Instance.new("BodyVelocity", ball)
            bv.Name = "CLB_Ctrl"
            bv.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
            bv.Velocity = moveDirection.Unit * 50
            ball.RotVelocity = Vector3.zero
        end)
    end
end

local lastKickTime = 0
local function tryPowerKick(ball)
    if not cfg.kickOn or cfg.kick <= 0 or tick() - lastKickTime < 0.5 or not Player.Character then return end
    local hrp2 = Player.Character:FindFirstChild("HumanoidRootPart")
    local hum = Player.Character:FindFirstChildOfClass("Humanoid")
    if not hrp2 or not hum or hum.MoveDirection.Magnitude < 0.1 then return end
    
    if ((ball.Position - hrp2.Position).Unit):Dot(hum.MoveDirection.Unit) > 0.5 then
        lastKickTime = tick()
        pcall(function()
            if ball:FindFirstChild("CLB_PK") then ball.CLB_PK:Destroy() end
            local bv = Instance.new("BodyVelocity", ball)
            bv.Name = "CLB_PK"
            bv.Velocity = hrp2.CFrame.LookVector * cfg.kick * 45 + Vector3.new(0, cfg.kick * 9, 0)
            bv.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
            task.delay(0.15, function() if bv and bv.Parent then bv:Destroy() end end)
        end)
    end
end

-- ==================== CRIAÇÃO DAS ABAS (RAYFIELD) ====================
local MainTab = Window:CreateTab("Principal", "home")
local ControlTab = Window:CreateTab("Controles", "gamepad-2")
local TrainingTab = Window:CreateTab("Treino", "dumbbell")
local VisualTab = Window:CreateTab("Personagem", "user")

-- ABA PRINCIPAL
MainTab:CreateToggle({
   Name = "Auto Seguir Bola",
   CurrentValue = cfg.autoFollow,
   Flag = "AutoFollow",
   Callback = function(Value)
      cfg.autoFollow = Value
      targetBall = Value and findClosestBall() or nil
   end,
})

MainTab:CreateSlider({
   Name = "Reach (Alcance)",
   Range = {1, 50},
   Increment = 1,
   CurrentValue = cfg.reach,
   Flag = "ReachSlider",
   Callback = function(Value)
      cfg.reach = Value
      us()
   end,
})

MainTab:CreateToggle({
   Name = "Mostrar Esfera Reach",
   CurrentValue = cfg.sphere,
   Flag = "SphereToggle",
   Callback = function(Value)
      cfg.sphere = Value
      us()
   end,
})

MainTab:CreateToggle({
   Name = "ESP de Bolas",
   CurrentValue = cfg.esp,
   Flag = "ESPToggle",
   Callback = function(Value)
      cfg.esp = Value
   end,
})

MainTab:CreateToggle({
   Name = "ESP Apenas Treino",
   CurrentValue = cfg.espTrainingOnly,
   Flag = "ESPTrainingOnly",
   Callback = function(Value)
      cfg.espTrainingOnly = Value
   end,
})

MainTab:CreateButton({
   Name = "Teleportar para Bola",
   Callback = function()
      if hrp then
          local cl = findClosestBall()
          if cl then hrp.CFrame = cl.CFrame * CFrame.new(0, 3, 0) end
      end
   end,
})

-- ABA CONTROLES
ControlTab:CreateToggle({
   Name = "Magnetic Ball",
   CurrentValue = cfg.magneticEnabled,
   Flag = "MagneticToggle",
   Callback = function(Value)
      cfg.magneticEnabled = Value
      if not Value then
          for ball, _ in pairs(magneticWelds) do if ball and ball.Parent then ball.CanCollide = true end end
          magneticWelds = {}
      end
   end,
})

ControlTab:CreateSlider({
   Name = "Força Magnética",
   Range = {10, 200},
   Increment = 5,
   CurrentValue = cfg.magneticStrength,
   Flag = "MagStrength",
   Callback = function(Value)
      cfg.magneticStrength = Value
   end,
})

ControlTab:CreateToggle({
   Name = "Controlar Bola (WASD)",
   CurrentValue = cfg.controlBallEnabled,
   Flag = "ControlBall",
   Callback = function(Value)
      cfg.controlBallEnabled = Value
      cfg.controlBallTarget = Value and findClosestBall() or nil
   end,
})

ControlTab:CreateToggle({
   Name = "Chute Potente",
   CurrentValue = cfg.kickOn,
   Flag = "KickOn",
   Callback = function(Value)
      cfg.kickOn = Value
   end,
})

ControlTab:CreateSlider({
   Name = "Força do Chute",
   Range = {0, 10},
   Increment = 1,
   CurrentValue = cfg.kick,
   Flag = "KickPower",
   Callback = function(Value)
      cfg.kick = Value
   end,
})

-- ABA TREINO
TrainingTab:CreateToggle({
   Name = "Modo Treino",
   CurrentValue = cfg.trainingMode,
   Flag = "TrainingMode",
   Callback = function(Value)
      cfg.trainingMode = Value
   end,
})

TrainingTab:CreateButton({
   Name = "Criar Bola de Treino",
   Callback = function()
      if not Workspace:FindFirstChild("TrainingBall") then
          local tb = Instance.new("Part", Workspace)
          tb.Name = "TrainingBall"
          tb.Size = Vector3.new(2, 2, 2)
          tb.Shape = Enum.PartType.Ball
          tb.BrickColor = BrickColor.new("Bright yellow")
          tb.Position = (hrp and hrp.Position or Vector3.new(0, 10, 0)) + Vector3.new(0, 5, 10)
      end
   end,
})

TrainingTab:CreateButton({
   Name = "Destruir Bola de Treino",
   Callback = function()
      local tb = Workspace:FindFirstChild("TrainingBall")
      if tb then tb:Destroy() end
   end,
})

-- ABA PERSONAGEM
VisualTab:CreateToggle({
   Name = "Spin Mode",
   CurrentValue = cfg.spin,
   Flag = "SpinMode",
   Callback = function(Value)
      cfg.spin = Value
   end,
})

VisualTab:CreateToggle({
   Name = "RGB Player (Arco-Íris)",
   CurrentValue = cfg.rgbPlayerEnabled,
   Flag = "RGBPlayer",
   Callback = function(Value)
      cfg.rgbPlayerEnabled = Value
      if Value then
          rgbConnection = RunService.RenderStepped:Connect(function()
              rgbHue = (rgbHue + 0.5) % 360
              local color = Color3.fromHSV(rgbHue / 360, 1, 1)
              if Player.Character then
                  for _, part in ipairs(Player.Character:GetDescendants()) do
                      if part:IsA("BasePart") and part.Name ~= "HumanoidRootPart" then part.Color = color end
                  end
              end
          end)
      else
          if rgbConnection then rgbConnection:Disconnect(); rgbConnection = nil end
      end
   end,
})

-- ==================== LOOP PRINCIPAL DE EXECUÇÃO ====================
RunService.RenderStepped:Connect(function(delta)
    dt = delta or 0.016

    if cfg.autoFollow then doAutoFollow() end
    if cfg.magneticEnabled then doMagneticBall() end
    if cfg.controlBallEnabled then doControlBall() end

    -- Toque / Reach Principal
    if cfg.touch and hrp then
        for _, pt in ipairs(gp(Player.Character)) do
            for _, b in ipairs(balls) do
                if b and b.Parent and (b.Position - pt.Position).Magnitude <= cfg.reach then
                    pcall(function() firetouchinterest(b, pt, 0); firetouchinterest(b, pt, 1) end)
                    if cfg.kickOn then tryPowerKick(b) end
                end
            end
        end
    end

    -- Spin Mode
    if cfg.spin and hrp then
        spinAngle = spinAngle + dt * cfg.spinSpeed * 2
        hrp.CFrame = CFrame.new(hrp.Position) * CFrame.Angles(0, spinAngle, 0)
    end

    -- Atualizar Posição da Esfera Reach
    if sp and hrp then sp.Position = hrp.Position end

    -- ESP Otimizado
    if cfg.esp and hrp then
        for _, b in ipairs(balls) do
            local proceed = true
            if cfg.espTrainingOnly and b.Name ~= "TrainingBall" then proceed = false end

            if proceed then
                if b and b.Parent and not esps[b] then
                    pcall(function()
                        local bb = Instance.new("BillboardGui", Player:WaitForChild("PlayerGui"))
                        bb.Adornee = b
                        bb.Size = UDim2.new(0, 60, 0, 35)
                        bb.StudsOffset = Vector3.new(0, 3, 0)
                        bb.AlwaysOnTop = true

                        local n = Instance.new("TextLabel", bb)
                        n.Size = UDim2.new(1, 0, 0.5, 0)
                        n.BackgroundTransparency = 1
                        n.Text = " " .. b.Name
                        n.TextColor3 = Color3.fromRGB(50, 255, 50)
                        n.TextScaled = true

                        local dd = Instance.new("TextLabel", bb)
                        dd.Name = "D"
                        dd.Size = UDim2.new(1, 0, 0.5, 0)
                        dd.Position = UDim2.new(0, 0, 0.5, 0)
                        dd.BackgroundTransparency = 1
                        dd.TextColor3 = Color3.fromRGB(255, 255, 100)
                        dd.TextScaled = true

                        local hl = Instance.new("Highlight", Camera)
                        hl.Adornee = b
                        hl.FillColor = Color3.fromRGB(50, 255, 50)
                        hl.FillTransparency = 0.5

                        esps[b] = {bb = bb, d = dd, hl = hl}
                    end)
                end
                if esps[b] and esps[b].d then
                    pcall(function() esps[b].d.Text = math.floor((b.Position - hrp.Position).Magnitude) .. "m" end)
                end
            else
                if esps[b] then
                    pcall(function() esps[b].bb:Destroy(); esps[b].hl:Destroy() end)
                    esps[b] = nil
                end
            end
        end
    else
        for _, o in pairs(esps) do pcall(function() o.bb:Destroy(); o.hl:Destroy() end) end
        esps = {}
    end
end)

-- Tarefa de varredura periódica de objetos
task.spawn(function()
    while true do
        task.wait(0.5)
        rb()
    end
end)

us()
Rayfield:Notify({Title = "Sucesso!", Content = "Smile 7 Hub carregado com sucesso!", Duration = 5})
