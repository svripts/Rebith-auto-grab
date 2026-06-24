local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local Stats = game:GetService("Stats")
local TweenService = game:GetService("TweenService")
local lp = Players.LocalPlayer

local STEAL_RADIUS = 68
local STEAL_DURATION = 1.3
local isStealing = false
local StealData = {}
local progressBarBg = nil
local progressFill = nil
local percentLabel = nil
local bannerFrame = nil
local infoLabel = nil

local function updateTopBar()
    if not infoLabel then return end
    local fps = 60
    local ping = 0
    local framesCount = 0
    local last = tick()
    RunService.RenderStepped:Connect(function()
        framesCount = framesCount + 1
        if tick() - last >= 1 then
            fps = framesCount
            framesCount = 0
            last = tick()
        end
        local network = Stats:FindFirstChild("Network")
        if network and network:FindFirstChild("ServerStatsItem") then
            local dataPing = network.ServerStatsItem:FindFirstChild("Data Ping")
            if dataPing then ping = math.floor(dataPing:GetValue()) end
        end
        infoLabel.Text = "404 AUTO GRAB  |  Ping: " .. ping .. "ms  |  FPS: " .. fps
    end)
end

local function setupUI()
    local sg = lp.PlayerGui:FindFirstChild("FourOhFourGrab")
    if not sg then
        sg = Instance.new("ScreenGui")
        sg.Name = "FourOhFourGrab"
        sg.ResetOnSpawn = false
        sg.IgnoreGuiInset = true
        sg.Parent = lp.PlayerGui
    end

    if not progressBarBg then
        local container = Instance.new("Frame")
        container.Size = UDim2.new(0, 320, 0, 78)
        container.Position = UDim2.new(0.5, -160, 0, 28)
        container.BackgroundTransparency = 1
        container.Parent = sg

        -- Banner (deep navy with cyan glow border)
        bannerFrame = Instance.new("Frame")
        bannerFrame.Size = UDim2.new(1, 0, 0, 42)
        bannerFrame.Position = UDim2.new(0, 0, 0, 0)
        bannerFrame.BackgroundColor3 = Color3.fromRGB(10, 18, 38)
        bannerFrame.BackgroundTransparency = 0.05
        bannerFrame.BorderSizePixel = 0
        bannerFrame.Parent = container
        Instance.new("UICorner", bannerFrame).CornerRadius = UDim.new(0, 12)

        local bannerGrad = Instance.new("UIGradient", bannerFrame)
        bannerGrad.Rotation = 90
        bannerGrad.Color = ColorSequence.new({
            ColorSequenceKeypoint.new(0, Color3.fromRGB(14, 24, 50)),
            ColorSequenceKeypoint.new(1, Color3.fromRGB(6, 12, 28)),
        })

        local bannerStroke = Instance.new("UIStroke", bannerFrame)
        bannerStroke.Color = Color3.fromRGB(64, 180, 255)
        bannerStroke.Thickness = 2
        bannerStroke.Transparency = 0.05
        bannerStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border

        -- Outer glow
        local glow = Instance.new("ImageLabel")
        glow.BackgroundTransparency = 1
        glow.Image = "rbxassetid://5028857084"
        glow.ImageColor3 = Color3.fromRGB(64, 180, 255)
        glow.ImageTransparency = 0.5
        glow.ScaleType = Enum.ScaleType.Slice
        glow.SliceCenter = Rect.new(24, 24, 276, 276)
        glow.Size = UDim2.new(1, 24, 1, 24)
        glow.Position = UDim2.new(0, -12, 0, -12)
        glow.ZIndex = 0
        glow.Parent = bannerFrame

        infoLabel = Instance.new("TextLabel")
        infoLabel.Size = UDim2.new(1, -20, 1, 0)
        infoLabel.Position = UDim2.new(0, 10, 0, 0)
        infoLabel.BackgroundTransparency = 1
        infoLabel.Font = Enum.Font.GothamBold
        infoLabel.TextSize = 16
        infoLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
        infoLabel.Text = "REBITH AUTO GRAB  |  Ping: 0ms  |  FPS: 0"
        infoLabel.TextXAlignment = Enum.TextXAlignment.Center
        infoLabel.TextYAlignment = Enum.TextYAlignment.Center
        infoLabel.ZIndex = 2
        infoLabel.Parent = bannerFrame

        -- Progress bar
        progressBarBg = Instance.new("Frame")
        progressBarBg.Size = UDim2.new(1, 0, 0, 22)
        progressBarBg.Position = UDim2.new(0, 0, 0, 50)
        progressBarBg.BackgroundColor3 = Color3.fromRGB(255, 255, 22)
        progressBarBg.BackgroundTransparency = 0
        progressBarBg.BorderSizePixel = 0
        progressBarBg.Parent = container
        Instance.new("UICorner", progressBarBg).CornerRadius = UDim.new(0, 10)

        local pbStroke = Instance.new("UIStroke", progressBarBg)
        pbStroke.Color = Color3.fromRGB(40, 90, 160)
        pbStroke.Thickness = 1
        pbStroke.Transparency = 0.3

        progressFill = Instance.new("Frame")
        progressFill.Size = UDim2.new(0, 0, 1, 0)
        progressFill.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
        progressFill.BorderSizePixel = 0
        progressFill.Parent = progressBarBg
        Instance.new("UICorner", progressFill).CornerRadius = UDim.new(0, 10)

        local fillGrad = Instance.new("UIGradient", progressFill)
        fillGrad.Color = ColorSequence.new({
            ColorSequenceKeypoint.new(0, Color3.fromRGB(80, 200, 255)),
            ColorSequenceKeypoint.new(1, Color3.fromRGB(40, 130, 230)),
        })

        percentLabel = Instance.new("TextLabel")
        percentLabel.Size = UDim2.new(1, 0, 1, 0)
        percentLabel.BackgroundTransparency = 1
        percentLabel.Font = Enum.Font.GothamBold
        percentLabel.TextSize = 14
        percentLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
        percentLabel.Text = "0%"
        percentLabel.ZIndex = 3
        percentLabel.Parent = progressBarBg

        updateTopBar()
    end
end

local function getHRP()
    local c = lp.Character
    if c then return c:FindFirstChild("HumanoidRootPart") or c:FindFirstChild("Torso") or c:FindFirstChild("UpperTorso") end
    return nil
end

local function isMyPlotByName(pn)
    local plots = workspace:FindFirstChild("Plots")
    if not plots then return false end
    local plot = plots:FindFirstChild(pn)
    if not plot then return false end
    local sign = plot:FindFirstChild("PlotSign")
    if sign then
        local yb = sign:FindFirstChild("YourBase")
        if yb and yb:IsA("BillboardGui") then return yb.Enabled == true end
    end
    return false
end

local function findNearestPrompt()
    local hrp = getHRP()
    if not hrp then return nil end
    local plots = workspace:FindFirstChild("Plots")
    if not plots then return nil end
    local nearest, dist = nil, math.huge
    for _, plot in ipairs(plots:GetChildren()) do
        if isMyPlotByName(plot.Name) then continue end
        local pods = plot:FindFirstChild("AnimalPodiums")
        if not pods then continue end
        for _, pod in ipairs(pods:GetChildren()) do
            local base = pod:FindFirstChild("Base")
            if not base then continue end
            local spawn = base:FindFirstChild("Spawn")
            if not spawn then continue end
            local d = (spawn.Position - hrp.Position).Magnitude
            if d <= STEAL_RADIUS and d < dist then
                local att = spawn:FindFirstChild("PromptAttachment")
                if att then
                    for _, p in ipairs(att:GetChildren()) do
                        if p:IsA("ProximityPrompt") and p.ActionText and p.ActionText:find("Steal") then
                            nearest, dist = p, d
                        end
                    end
                end
            end
        end
    end
    return nearest
end

local function updateProgressBar(p)
    if progressFill then
        progressFill.Size = UDim2.new(p, 0, 1, 0)
    end
    if percentLabel then
        percentLabel.Text = math.floor(p * 100) .. "%"
    end
end

local function executeSteal(prompt)
    if isStealing then return end
    if not StealData[prompt] then
        StealData[prompt] = {hold = {}, trigger = {}, ready = true}
        if getconnections then
            for _, c in ipairs(getconnections(prompt.PromptButtonHoldBegan)) do
                if c.Function then table.insert(StealData[prompt].hold, c.Function) end
            end
            for _, c in ipairs(getconnections(prompt.Triggered)) do
                if c.Function then table.insert(StealData[prompt].trigger, c.Function) end
            end
        end
    end
    local data = StealData[prompt]
    if not data.ready then return end
    data.ready = false
    isStealing = true
    local startTime = tick()
    task.spawn(function()
        for _, f in ipairs(data.hold) do pcall(f) end
        while tick() - startTime < STEAL_DURATION do
            local elapsed = tick() - startTime
            local p = math.clamp(elapsed / STEAL_DURATION, 0, 1)
            updateProgressBar(p)
            task.wait()
        end
        updateProgressBar(1)
        for _, f in ipairs(data.trigger) do pcall(f) end
        task.wait(0.05)
        updateProgressBar(0)
        data.ready = true
        isStealing = false
    end)
end

local heartbeatConn
local function startAutoSteal()
    setupUI()
    if heartbeatConn then return end
    heartbeatConn = RunService.Heartbeat:Connect(function()
        if isStealing then return end
        local success, prompt = pcall(findNearestPrompt)
        if success and prompt then pcall(executeSteal, prompt) end
    end)
end

local function stopAutoSteal()
    if heartbeatConn then heartbeatConn:Disconnect() heartbeatConn = nil end
    isStealing = false
    updateProgressBar(0)
end

startAutoSteal()
