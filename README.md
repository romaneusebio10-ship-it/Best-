local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Workspace = game:GetService("Workspace")

local player = Players.LocalPlayer
local camera = Workspace.CurrentCamera

-- Persistent aimbot toggle stored on player
local aimbotStateValue = player:FindFirstChild("AimbotEnabled")
if not aimbotStateValue then
    aimbotStateValue = Instance.new("BoolValue")
    aimbotStateValue.Name = "AimbotEnabled"
    aimbotStateValue.Value = false
    aimbotStateValue.Parent = player
end

local aimbotEnabled = aimbotStateValue.Value
local teamCheckEnabled = false
local deathCheckEnabled = false
local wallCheckEnabled = false

local maxAllowedFOV = 500
local maxFOV = 500
local currentFOV = 100
local smoothness = 1
local aimPartName = "Head"

local screenGui, mainFrame, menuToggleBtn, teamToggle, deathToggle, wallToggle, smoothInput, aimbotToggle, fovLabel, highlight, sliderFill, knob, sliderBg

local function clampFOV(val)
    if val > maxAllowedFOV then
        return maxAllowedFOV
    elseif val < 1 then
        return 1
    else
        return val
    end
end

local function createUI()
    if screenGui then screenGui:Destroy() end
    screenGui = Instance.new("ScreenGui")
    screenGui.Name = "AimbotUI"
    screenGui.ResetOnSpawn = false
    screenGui.Parent = player:WaitForChild("PlayerGui")

    mainFrame = Instance.new("Frame", screenGui)
    mainFrame.AnchorPoint = Vector2.new(0, 0)
    mainFrame.Position = UDim2.new(0, 10, 0, 50)
    mainFrame.Size = UDim2.new(0, 320, 0, 380)
    mainFrame.BackgroundColor3 = Color3.fromRGB(45, 45, 45)
    mainFrame.BorderSizePixel = 0
    mainFrame.Visible = false
    Instance.new("UICorner", mainFrame).CornerRadius = UDim.new(0, 12)

    local titleLabel = Instance.new("TextLabel", mainFrame)
    titleLabel.Position = UDim2.new(0, 15, 0, 15)
    titleLabel.Size = UDim2.new(1, -30, 0, 30)
    titleLabel.BackgroundTransparency = 1
    titleLabel.Font = Enum.Font.GothamBold
    titleLabel.TextSize = 22
    titleLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
    titleLabel.Text = "Aimbot Settings"

    -- Drag bar for draggable area limited to top part of the aimbot UI
    local dragBar = Instance.new("Frame", mainFrame)
    dragBar.Size = UDim2.new(1, 0, 0, 30) -- 30 pixels tall at top
    dragBar.Position = UDim2.new(0, 0, 0, 0)
    dragBar.BackgroundTransparency = 1 -- set to 0.5 for visible drag area during testing
    dragBar.ZIndex = mainFrame.ZIndex + 1

    local dragging = false
    local dragInput
    local dragStart
    local startPos

    local function update(input)
        local delta = input.Position - dragStart
        local newPos = UDim2.new(
            startPos.X.Scale,
            startPos.X.Offset + delta.X,
            startPos.Y.Scale,
            startPos.Y.Offset + delta.Y
        )
        mainFrame.Position = newPos
    end

    dragBar.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            dragStart = input.Position
            startPos = mainFrame.Position

            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then
                    dragging = false
                end
            end)
        end
    end)

    dragBar.InputChanged:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
            dragInput = input
        end
    end)

    UserInputService.InputChanged:Connect(function(input)
        if input == dragInput and dragging then
            update(input)
        end
    end)

    local function createToggle(name, yPos, default)
        local container = Instance.new("Frame", mainFrame)
        container.Size = UDim2.new(0.9, 0, 0, 40)
        container.Position = UDim2.new(0.05, 0, 0, yPos)
        container.BackgroundColor3 = Color3.fromRGB(65, 65, 65)
        Instance.new("UICorner", container).CornerRadius = UDim.new(0, 10)

        local label = Instance.new("TextLabel", container)
        label.Size = UDim2.new(0.7, 0, 1, 0)
        label.Position = UDim2.new(0.05, 0, 0, 0)
        label.Text = name
        label.Font = Enum.Font.GothamBold
        label.TextSize = 18
        label.TextColor3 = Color3.fromRGB(230, 230, 230)
        label.BackgroundTransparency = 1
        label.TextXAlignment = Enum.TextXAlignment.Left

        local toggleBtn = Instance.new("TextButton", container)
        toggleBtn.Size = UDim2.new(0.2, 0, 0.75, 0)
        toggleBtn.Position = UDim2.new(0.75, 0, 0.125, 0)
        toggleBtn.Text = default and "On" or "Off"
        toggleBtn.Font = Enum.Font.GothamBold
        toggleBtn.TextSize = 18
        toggleBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
        toggleBtn.BackgroundColor3 = default and Color3.fromRGB(55, 170, 70) or Color3.fromRGB(170, 50, 50)
        toggleBtn.AutoButtonColor = true
        Instance.new("UICorner", toggleBtn).CornerRadius = UDim.new(0, 8)

        toggleBtn.MouseButton1Click:Connect(function()
            if name == "Team Check" then
                teamCheckEnabled = not teamCheckEnabled
                toggleBtn.Text = teamCheckEnabled and "On" or "Off"
                toggleBtn.BackgroundColor3 = teamCheckEnabled and Color3.fromRGB(55, 170, 70) or Color3.fromRGB(170, 50, 50)
            elseif name == "Death Check" then
                deathCheckEnabled = not deathCheckEnabled
                toggleBtn.Text = deathCheckEnabled and "On" or "Off"
                toggleBtn.BackgroundColor3 = deathCheckEnabled and Color3.fromRGB(55, 170, 70) or Color3.fromRGB(170, 50, 50)
            elseif name == "Wall Check" then
                wallCheckEnabled = not wallCheckEnabled
                toggleBtn.Text = wallCheckEnabled and "On" or "Off"
                toggleBtn.BackgroundColor3 = wallCheckEnabled and Color3.fromRGB(55, 170, 70) or Color3.fromRGB(170, 50, 50)
            end
        end)
        return toggleBtn
    end

    teamToggle = createToggle("Team Check", 65, teamCheckEnabled)
    deathToggle = createToggle("Death Check", 115, deathCheckEnabled)
    wallToggle = createToggle("Wall Check", 165, wallCheckEnabled)

    local smoothLabel = Instance.new("TextLabel", mainFrame)
    smoothLabel.Position = UDim2.new(0.05, 0, 0, 215)
    smoothLabel.Size = UDim2.new(0.45, 0, 0, 40)
    smoothLabel.BackgroundTransparency = 1
    smoothLabel.Font = Enum.Font.GothamBold
    smoothLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
    smoothLabel.TextSize = 18
    smoothLabel.Text = "Smoothness (max 2):"

    smoothInput = Instance.new("TextBox", mainFrame)
    smoothInput.Position = UDim2.new(0.52, 0, 0, 215)
    smoothInput.Size = UDim2.new(0.35, 0, 0, 40)
    smoothInput.BackgroundColor3 = Color3.fromRGB(35, 35, 35)
    smoothInput.TextColor3 = Color3.fromRGB(255, 255, 255)
    smoothInput.Font = Enum.Font.GothamBold
    smoothInput.TextSize = 22
    smoothInput.Text = tostring(smoothness)
    Instance.new("UICorner", smoothInput).CornerRadius = UDim.new(0, 6)

    smoothInput.FocusLost:Connect(function(enterPressed)
        if enterPressed then
            local val = tonumber(smoothInput.Text)
            if val then
                if val > 2 then val = 2 end
                if val < 0 then val = 0 end
                smoothness = val
                smoothInput.Text = tostring(smoothness)
            else
                smoothInput.Text = tostring(smoothness)
            end
        end
    end)

    aimbotToggle = Instance.new("TextButton", mainFrame)
    aimbotToggle.Position = UDim2.new(0.05, 0, 0, 265)
    aimbotToggle.Size = UDim2.new(0.9, 0, 0, 45)
    aimbotToggle.BackgroundColor3 = aimbotEnabled and Color3.fromRGB(55, 170, 70) or Color3.fromRGB(45, 80, 45)
    aimbotToggle.Font = Enum.Font.GothamBold
    aimbotToggle.TextSize = 20
    aimbotToggle.TextColor3 = Color3.fromRGB(255, 255, 255)
    aimbotToggle.Text = aimbotEnabled and "Aimbot : ON" or "Aimbot : OFF"
    Instance.new("UICorner", aimbotToggle).CornerRadius = UDim.new(0, 10)

    aimbotToggle.MouseButton1Click:Connect(function()
        aimbotEnabled = not aimbotEnabled
        aimbotStateValue.Value = aimbotEnabled -- save state persistently
        if aimbotEnabled then
            aimbotToggle.Text = "Aimbot : ON"
            aimbotToggle.BackgroundColor3 = Color3.fromRGB(55, 170, 70)
        else
            aimbotToggle.Text = "Aimbot : OFF"
            aimbotToggle.BackgroundColor3 = Color3.fromRGB(45, 80, 45)
        end
    end)

    fovLabel = Instance.new("TextLabel", mainFrame)
    fovLabel.Position = UDim2.new(0.05, 0, 0, 320)
    fovLabel.Size = UDim2.new(0.9, 0, 0, 20)
    fovLabel.BackgroundTransparency = 1
    fovLabel.Font = Enum.Font.GothamBold
    fovLabel.TextColor3 = Color3.fromRGB(225, 225, 225)
    fovLabel.TextSize = 16
    fovLabel.Text = "FOV Radius: " .. currentFOV

    sliderBg = Instance.new("Frame", mainFrame)
    sliderBg.Position = UDim2.new(0.05, 0, 0, 345)
    sliderBg.Size = UDim2.new(0.9, 0, 0, 16)
    sliderBg.BackgroundColor3 = Color3.fromRGB(80, 80, 80)
    Instance.new("UICorner", sliderBg).CornerRadius = UDim.new(0, 8)

    sliderFill = Instance.new("Frame", sliderBg)
    sliderFill.Size = UDim2.new(currentFOV / maxFOV, 0, 1, 0)
    sliderFill.BackgroundColor3 = Color3.fromRGB(70, 170, 70)
    Instance.new("UICorner", sliderFill).CornerRadius = UDim.new(0, 8)

    knob = Instance.new("ImageLabel", sliderBg)
    knob.Size = UDim2.new(0, 24, 0, 24)
    knob.AnchorPoint = Vector2.new(0.5, 0.5)
    knob.Position = UDim2.new(currentFOV / maxFOV, 0, 0.5, 0)
    knob.BackgroundTransparency = 1
    knob.Image = "rbxassetid://2842083050"

    local draggingSlider = false
    local function updateSlider(input)
        if not draggingSlider then return end
        local relativeX = math.clamp(input.Position.X - sliderBg.AbsolutePosition.X, 0, sliderBg.AbsoluteSize.X)
        local scale = relativeX / sliderBg.AbsoluteSize.X
        local fovVal = math.floor(scale * maxFOV)
        fovVal = clampFOV(fovVal)
        sliderFill:TweenSize(UDim2.new(fovVal / maxFOV, 0, 1, 0), Enum.EasingDirection.Out, Enum.EasingStyle.Sine, 0.1, true)
        knob:TweenPosition(UDim2.new(fovVal / maxFOV, 0, 0.5, 0), Enum.EasingDirection.Out, Enum.EasingStyle.Sine, 0.1, true)
        fovLabel.Text = "FOV Radius: " .. fovVal
        currentFOV = fovVal
    end

    sliderBg.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            draggingSlider = true
            updateSlider(input)
        end
    end)

    sliderBg.InputChanged:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
            updateSlider(input)
        end
    end)

    UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            draggingSlider = false
        end
    end)

    menuToggleBtn = Instance.new("TextButton", screenGui)
    menuToggleBtn.Position = UDim2.new(0, 10, 0, 10)
    menuToggleBtn.Size = UDim2.new(0, 110, 0, 40)
    menuToggleBtn.Text = "Aimbot Menu"
    menuToggleBtn.Font = Enum.Font.GothamBold
    menuToggleBtn.TextSize = 20
    menuToggleBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    menuToggleBtn.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
    Instance.new("UICorner", menuToggleBtn).CornerRadius = UDim.new(0, 12)

    menuToggleBtn.MouseButton1Click:Connect(function()
        mainFrame.Visible = not mainFrame.Visible
    end)

    currentFOV = clampFOV(currentFOV)
    sliderFill.Size = UDim2.new(currentFOV / maxFOV, 0, 1, 0)
    knob.Position = UDim2.new(currentFOV / maxFOV, 0, 0.5, 0)
    fovLabel.Text = "FOV Radius: " .. currentFOV

    if highlight then
        highlight:Destroy()
    end
    highlight = Instance.new("Highlight")
    highlight.FillColor = Color3.fromRGB(255, 0, 0)
    highlight.OutlineColor = Color3.fromRGB(255, 255, 255)
    highlight.FillTransparency = 0.7
    highlight.OutlineTransparency = 0.3
    highlight.Adornee = nil
    highlight.Parent = player:WaitForChild("PlayerGui")
end

-- Improved visibility check: raycast ignores local player, returns true if hit is part of target character
local function isVisible(origin, targetPart)
    local direction = targetPart.Position - origin
    local distance = direction.Magnitude
    local rayParams = RaycastParams.new()
    rayParams.FilterDescendantsInstances = {player.Character}
    rayParams.FilterType = Enum.RaycastFilterType.Blacklist

    local result = Workspace:Raycast(origin, direction.Unit * distance, rayParams)
    if result then
        if result.Instance:IsDescendantOf(targetPart.Parent) then
            return true
        else
            return false
        end
    else
        return true
    end
end

local function getClosestTarget(fovRadius)
    local closestTarget = nil
    local closestDistance = math.huge
    local screenCenter = Vector2.new(camera.ViewportSize.X / 2, camera.ViewportSize.Y / 2)

    for _, targetPlayer in pairs(Players:GetPlayers()) do
        if targetPlayer ~= player then
            if teamCheckEnabled and targetPlayer.Team == player.Team then
                continue
            end

            local char = targetPlayer.Character
            if not char then continue end

            local humanoid = char:FindFirstChildOfClass("Humanoid")
            if deathCheckEnabled then
                if not humanoid then continue end
                local state = humanoid:GetState()
                if state == Enum.HumanoidStateType.Dead or humanoid.Health <= 0 then
                    continue
                end
            end

            local part = char:FindFirstChild(aimPartName)
            if not part then continue end

            if wallCheckEnabled and not isVisible(camera.CFrame.Position, part) then
                continue
            end

            local screenPos, onScreen = camera:WorldToViewportPoint(part.Position)
            if not onScreen then continue end

            local dist = (Vector2.new(screenPos.X, screenPos.Y) - screenCenter).Magnitude
            if dist <= fovRadius and dist < closestDistance then
                closestDistance = dist
                closestTarget = targetPlayer
            end
        end
    end

    return closestTarget
end

local function highlightTargetBody(target)
    if target and target.Character then
        highlight.Adornee = target.Character
    else
        highlight.Adornee = nil
    end
end

local function aimAt(target)
    if not target or not target.Character then
        highlightTargetBody(nil)
        return
    end
    local part = target.Character:FindFirstChild(aimPartName)
    if not part then
        highlightTargetBody(nil)
        return
    end
    local camPos = camera.CFrame.Position
    local dir = (part.Position - camPos).Unit
    local desiredCFrame = CFrame.new(camPos, camPos + dir)
    camera.CFrame = camera.CFrame:Lerp(desiredCFrame, smoothness)
    highlightTargetBody(target)
end

RunService.RenderStepped:Connect(function()
    if aimbotEnabled then
        local target = getClosestTarget(currentFOV)
        if target then
            aimAt(target)
        else
            highlightTargetBody(nil)
        end
    else
        highlightTargetBody(nil)
    end
end)

local fovCircle
if Drawing and Drawing.new then
    fovCircle = Drawing.new("Circle")
    fovCircle.Transparency = 0.6
    fovCircle.Filled = false
    fovCircle.Thickness = 2
    fovCircle.NumSides = 64
    fovCircle.Color = Color3.fromRGB(255, 0, 0)
    fovCircle.Position = Vector2.new(camera.ViewportSize.X / 2, camera.ViewportSize.Y / 2)
    fovCircle.Radius = currentFOV
    fovCircle.Visible = false
end

RunService.RenderStepped:Connect(function()
    if aimbotEnabled and fovCircle then
        fovCircle.Position = Vector2.new(camera.ViewportSize.X / 2, camera.ViewportSize.Y / 2)
        fovCircle.Radius = currentFOV
        fovCircle.Visible = true
    elseif fovCircle then
        fovCircle.Visible = false
    end
end)

local function onCharacterAdded(character)
    camera = Workspace.CurrentCamera
    aimbotEnabled = aimbotStateValue.Value -- apply persistent toggle state
    createUI()

    if aimbotEnabled then
        aimbotToggle.Text = "Aimbot : ON"
        aimbotToggle.BackgroundColor3 = Color3.fromRGB(55, 170, 70)
    else
        aimbotToggle.Text = "Aimbot : OFF"
        aimbotToggle.BackgroundColor3 = Color3.fromRGB(45, 80, 45)
    end
end

if player.Character then
    onCharacterAdded(player.Character)
end

player.CharacterAdded:Connect(onCharacterAdded)

 
