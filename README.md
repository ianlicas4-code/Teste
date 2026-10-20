
-- ADM AIM | Roblox Studio
-- ESP e Aim Assist para NPCs de teste
-- LocalScript em StarterPlayerScripts

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")

local player = Players.LocalPlayer
local camera = workspace.CurrentCamera

local settings = {
    ESP = false,
    Vida = true,
    Linhas = false,
    AimAssist = false,
    FOV = true
}

local aimFOV = 120
local aimStrength = 0.12
local npcFolder = workspace:FindFirstChild("TestNPCs")

local gui = Instance.new("ScreenGui")
gui.Name = "ADMAIM"
gui.ResetOnSpawn = false
gui.Parent = player:WaitForChild("PlayerGui")

local panel = Instance.new("Frame")
panel.Size = UDim2.fromOffset(270, 350)
panel.Position = UDim2.new(0, 20, 0.3, 0)
panel.BackgroundColor3 = Color3.fromRGB(24, 28, 40)
panel.BorderSizePixel = 0
panel.Parent = gui

Instance.new("UICorner", panel).CornerRadius =
    UDim.new(0, 12)

local title = Instance.new("TextLabel")
title.Size = UDim2.new(1, 0, 0, 45)
title.BackgroundTransparency = 1
title.Text = "ADM AIM"
title.TextColor3 = Color3.new(1, 1, 1)
title.Font = Enum.Font.GothamBold
title.TextSize = 21
title.Parent = panel

local buttons = {}

local function makeButton(name, y, callback)
    local button = Instance.new("TextButton")
    button.Size = UDim2.new(1, -24, 0, 36)
    button.Position = UDim2.fromOffset(12, y)
    button.BorderSizePixel = 0
    button.TextColor3 = Color3.new(1, 1, 1)
    button.Font = Enum.Font.Gotham
    button.TextSize = 14
    button.Parent = panel

    Instance.new("UICorner", button).CornerRadius =
        UDim.new(0, 7)

    buttons[name] = button

    button.Activated:Connect(callback)
    return button
end

local function refresh(name)
    local button = buttons[name]
    if not button then return end

    button.Text = name .. ": " ..
        (settings[name] and "ON" or "OFF")

    button.BackgroundColor3 = settings[name]
        and Color3.fromRGB(25, 145, 105)
        or Color3.fromRGB(50, 58, 75)
end

local names = {"ESP", "Vida", "Linhas", "AimAssist", "FOV"}

for index, name in ipairs(names) do
    makeButton(name, 50 + (index - 1) * 42, function()
        settings[name] = not settings[name]
        refresh(name)
    end)
    refresh(name)
end

local fovButton = makeButton("Ajustar FOV", 270, function()
    aimFOV += 20
    if aimFOV > 300 then aimFOV = 40 end
    fovButton = buttons["Ajustar FOV"]
    fovButton.Text = "Aim FOV: " .. aimFOV .. " px"
end)
fovButton.Text = "Aim FOV: " .. aimFOV .. " px"
fovButton.BackgroundColor3 = Color3.fromRGB(70, 85, 115)

local hide = Instance.new("TextButton")
hide.Size = UDim2.fromOffset(120, 36)
hide.Position = UDim2.fromOffset(20, 10)
hide.Text = "Ocultar painel"
hide.TextColor3 = Color3.new(1, 1, 1)
hide.BackgroundColor3 = Color3.fromRGB(25, 145, 105)
hide.Parent = gui

hide.Activated:Connect(function()
    panel.Visible = not panel.Visible
    hide.Text = panel.Visible and
        "Ocultar painel" or "Mostrar painel"
end)

-- Círculo de FOV
local circle = Instance.new("Frame")
circle.Name = "FOVCircle"
circle.AnchorPoint = Vector2.new(0.5, 0.5)
circle.Size = UDim2.fromOffset(aimFOV * 2, aimFOV * 2)
circle.BackgroundTransparency = 1
circle.BorderSizePixel = 0
circle.Parent = gui

local stroke = Instance.new("UIStroke")
stroke.Color = Color3.fromRGB(30, 220, 150)
stroke.Thickness = 2
stroke.Parent = circle

Instance.new("UICorner", circle).CornerRadius =
    UDim.new(1, 0)

-- Elementos de ESP por NPC
local visuals = {}

local function removeVisual(model)
    local visual = visuals[model]
    if not visual then return end

    if visual.highlight then visual.highlight:Destroy() end
    if visual.billboard then visual.billboard:Destroy() end
    if visual.line then visual.line:Destroy() end

    visuals[model] = nil
end

local function createVisual(model)
    if visuals[model] or not model:IsA("Model") then
        return
    end

    local humanoid = model:FindFirstChildOfClass("Humanoid")
    local root = model:FindFirstChild("HumanoidRootPart")

    if not humanoid or not root then return end

    local highlight = Instance.new("Highlight")
    highlight.Adornee = model
    highlight.FillTransparency = 0.8
    highlight.OutlineColor = Color3.fromRGB(30, 230, 150)
    highlight.Enabled = false
    highlight.Parent = gui

    local billboard = Instance.new("BillboardGui")
    billboard.Adornee = root
    billboard.Size = UDim2.fromOffset(110, 18)
    billboard.StudsOffset = Vector3.new(0, 3.2, 0)
    billboard.AlwaysOnTop = true
    billboard.Enabled = false
    billboard.Parent = gui

    local health = Instance.new("TextLabel")
    health.Size = UDim2.fromScale(1, 1)
    health.BackgroundTransparency = 1
    health.TextColor3 = Color3.fromRGB(80, 255, 120)
    health.TextStrokeTransparency = 0.3
    health.TextSize = 13
    health.Font = Enum.Font.GothamBold
    health.Parent = billboard

    local line = Instance.new("Frame")
    line.AnchorPoint = Vector2.new(0, 0.5)
    line.BorderSizePixel = 0
    line.BackgroundColor3 = Color3.fromRGB(30, 230, 150)
    line.Visible = false
    line.Parent = gui

    visuals[model] = {
        highlight = highlight,
        billboard = billboard,
        health = health,
        line = line
    }
end

local function getNPCFolder()
    return workspace:FindFirstChild("TestNPCs")
end

local function findTarget()
    local folder = getNPCFolder()
    if not folder then return nil end

    local center = camera.ViewportSize / 2
    local bestTarget, bestDistance = nil, aimFOV

    for _, model in ipairs(folder:GetChildren()) do
        local humanoid = model:FindFirstChildOfClass("Humanoid")
        local root = model:FindFirstChild("HumanoidRootPart")

        if humanoid and humanoid.Health > 0 and root then
            local point, visible =
                camera:WorldToViewportPoint(root.Position)

            if visible and point.Z > 0 then
                local distance = (
                    Vector2.new(point.X, point.Y) - center
                ).Magnitude

                if distance < bestDistance then
                    bestDistance = distance
                    bestTarget = root
                end
            end
        end
    end

    return bestTarget
end

RunService.RenderStepped:Connect(function()
    camera = workspace.CurrentCamera
    if not camera then return end

    local viewport = camera.ViewportSize
    circle.Position = UDim2.fromOffset(
        viewport.X / 2, viewport.Y / 2
    )
    circle.Size = UDim2.fromOffset(aimFOV * 2, aimFOV * 2)
    circle.Visible = settings.FOV

    local folder = getNPCFolder()

    if folder then
        for _, model in ipairs(folder:GetChildren()) do
            createVisual(model)
        end
    end

    for model, visual in pairs(visuals) do
        local humanoid = model:FindFirstChildOfClass("Humanoid")
        local root = model:FindFirstChild("HumanoidRootPart")

        if not model.Parent or not humanoid or not root then
            removeVisual(model)
        else
            visual.highlight.Enabled = settings.ESP
                and humanoid.Health > 0

            visual.billboard.Enabled = settings.Vida
                and humanoid.Health > 0

            visual.health.Text = string.format(
                "HP: %d / %d",
                math.ceil(humanoid.Health),
                math.ceil(humanoid.MaxHealth)
            )

            local point, visible =
                camera:WorldToViewportPoint(root.Position)

            local center = camera.ViewportSize / 2
            local delta = Vector2.new(point.X, point.Y) - center

            visual.line.Visible = settings.Linhas
                and visible and point.Z > 0

            if visual.line.Visible then
                visual.line.Position = UDim2.fromOffset(
                    center.X, center.Y
                )
                visual.line.Size = UDim2.fromOffset(
                    delta.Magnitude, 2
                )
                visual.line.Rotation =
                    math.deg(math.atan2(delta.Y, delta.X))
            end
        end
    end

    -- Assistência de mira para NPCs da pasta TestNPCs.
    -- Mantém a mira próxima do alvo dentro do FOV.
    if settings.AimAssist then
        local target = findTarget()

        if target then
            local desired = CFrame.lookAt(
                camera.CFrame.Position, target.Position
            )

            camera.CFrame = camera.CFrame:Lerp(
                desired, aimStrength
            )
        end
    end
end)

print("ADM AIM carregado.")
print("Crie a pasta Workspace/TestNPCs e coloque NPCs nela.")
