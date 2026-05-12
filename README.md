local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local LocalPlayer = Players.LocalPlayer

local toggle_states = {
    SilentAim = false
}

local config_values = {
    SilentAim = {
        Chance = 100,
        Draw = true,
        DrawSize = 150,
        DrawColor = Color3.fromRGB(255,255,255),
        CheckDowned = false,
        CheckTeam = false,
        CheckWhiteList = false,
        TargetParts = {"Head","HumanoidRootPart"}
    }
}

local whitelist_table = {}
local connection_table = {}
local drawing_table = {}

local ui_library = {}
ui_library.__index = ui_library

function ui_library:CreateUI()
    local self = setmetatable({}, ui_library)

    self.dragging = false
    self.drag_input = nil
    self.drag_start = nil
    self.start_pos = nil

    local gui = Instance.new("ScreenGui")
    gui.Name = "SilentAimUI"
    gui.ResetOnSpawn = false
    gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    pcall(function()
        gui.Parent = game:GetService("CoreGui")
    end)

    local main = Instance.new("Frame")
    main.Size = UDim2.new(0, 420, 0, 520)
    main.Position = UDim2.new(0.5, -210, 0.5, -260)
    main.BackgroundColor3 = Color3.fromRGB(25,25,30)
    main.BorderSizePixel = 0
    main.Parent = gui

    local mainCorner = Instance.new("UICorner")
    mainCorner.CornerRadius = UDim.new(0,10)
    mainCorner.Parent = main

    local mainStroke = Instance.new("UIStroke")
    mainStroke.Color = Color3.fromRGB(70,70,80)
    mainStroke.Thickness = 1
    mainStroke.Parent = main

    local topbar = Instance.new("Frame")
    topbar.Size = UDim2.new(1,0,0,40)
    topbar.BackgroundColor3 = Color3.fromRGB(35,35,40)
    topbar.BorderSizePixel = 0
    topbar.Parent = main

    local topCorner = Instance.new("UICorner")
    topCorner.CornerRadius = UDim.new(0,10)
    topCorner.Parent = topbar

    local fix = Instance.new("Frame")
    fix.Size = UDim2.new(1,0,0,10)
    fix.Position = UDim2.new(0,0,1,-10)
    fix.BackgroundColor3 = Color3.fromRGB(35,35,40)
    fix.BorderSizePixel = 0
    fix.Parent = topbar

    local title = Instance.new("TextLabel")
    title.Size = UDim2.new(1,-50,1,0)
    title.Position = UDim2.new(0,15,0,0)
    title.BackgroundTransparency = 1
    title.Text = "SilentAim Configuration"
    title.Font = Enum.Font.GothamBold
    title.TextColor3 = Color3.fromRGB(255,255,255)
    title.TextSize = 15
    title.TextXAlignment = Enum.TextXAlignment.Left
    title.Parent = topbar

    local close = Instance.new("TextButton")
    close.Size = UDim2.new(0,30,0,30)
    close.Position = UDim2.new(1,-35,0.5,-15)
    close.BackgroundTransparency = 1
    close.Text = "✕"
    close.Font = Enum.Font.GothamBold
    close.TextColor3 = Color3.fromRGB(255,100,100)
    close.TextSize = 18
    close.Parent = topbar

    close.MouseButton1Click:Connect(function()
        gui:Destroy()

        for _,v in pairs(connection_table) do
            pcall(function()
                v:Disconnect()
            end)
        end

        if drawing_table.SilentAim then
            for _,v in pairs(drawing_table.SilentAim) do
                pcall(function()
                    v:Remove()
                end)
            end
        end
    end)

    local container = Instance.new("ScrollingFrame")
    container.Size = UDim2.new(1,-20,1,-60)
    container.Position = UDim2.new(0,10,0,50)
    container.BackgroundTransparency = 1
    container.BorderSizePixel = 0
    container.ScrollBarThickness = 4
    container.CanvasSize = UDim2.new(0,0,0,0)
    container.AutomaticCanvasSize = Enum.AutomaticSize.Y
    container.Parent = main

    local layout = Instance.new("UIListLayout")
    layout.Padding = UDim.new(0,8)
    layout.Parent = container

    local padding = Instance.new("UIPadding")
    padding.PaddingBottom = UDim.new(0,10)
    padding.Parent = container

    local function update(input)
        local delta = input.Position - self.drag_start
        main.Position = UDim2.new(
            self.start_pos.X.Scale,
            self.start_pos.X.Offset + delta.X,
            self.start_pos.Y.Scale,
            self.start_pos.Y.Offset + delta.Y
        )
    end

    topbar.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            self.dragging = true
            self.drag_start = input.Position
            self.start_pos = main.Position

            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then
                    self.dragging = false
                end
            end)
        end
    end)

    topbar.InputChanged:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
            self.drag_input = input
        end
    end)

    UserInputService.InputChanged:Connect(function(input)
        if input == self.drag_input and self.dragging then
            update(input)
        end
    end)

    self.gui = gui
    self.main = main
    self.container = container

    return self
end

function ui_library:AddToggle(parent,text,default,callback)
    local enabled = default

    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(1,-5,0,50)
    frame.BackgroundColor3 = Color3.fromRGB(35,35,40)
    frame.BorderSizePixel = 0
    frame.Parent = parent

    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0,8)
    corner.Parent = frame

    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(1,-80,1,0)
    label.Position = UDim2.new(0,15,0,0)
    label.BackgroundTransparency = 1
    label.Text = text
    label.Font = Enum.Font.Gotham
    label.TextColor3 = Color3.fromRGB(255,255,255)
    label.TextSize = 14
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.Parent = frame

    local toggle = Instance.new("TextButton")
    toggle.Size = UDim2.new(0,50,0,24)
    toggle.Position = UDim2.new(1,-65,0.5,-12)
    toggle.Text = ""
    toggle.AutoButtonColor = false
    toggle.BackgroundColor3 = enabled and Color3.fromRGB(0,200,120) or Color3.fromRGB(70,70,80)
    toggle.Parent = frame

    local toggleCorner = Instance.new("UICorner")
    toggleCorner.CornerRadius = UDim.new(1,0)
    toggleCorner.Parent = toggle

    local circle = Instance.new("Frame")
    circle.Size = UDim2.new(0,20,0,20)
    circle.Position = enabled and UDim2.new(1,-22,0.5,-10) or UDim2.new(0,2,0.5,-10)
    circle.BackgroundColor3 = Color3.fromRGB(255,255,255)
    circle.BorderSizePixel = 0
    circle.Parent = toggle

    local circleCorner = Instance.new("UICorner")
    circleCorner.CornerRadius = UDim.new(1,0)
    circleCorner.Parent = circle

    local function set(state)
        enabled = state
        toggle.BackgroundColor3 = enabled and Color3.fromRGB(0,200,120) or Color3.fromRGB(70,70,80)
        circle.Position = enabled and UDim2.new(1,-22,0.5,-10) or UDim2.new(0,2,0.5,-10)

        if callback then
            callback(enabled)
        end
    end

    toggle.MouseButton1Click:Connect(function()
        set(not enabled)
    end)

    return {
        Set = set
    }
end

function ui_library:AddSlider(parent,text,min,max,default,callback)
    local value = default

    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(1,-5,0,65)
    frame.BackgroundColor3 = Color3.fromRGB(35,35,40)
    frame.BorderSizePixel = 0
    frame.Parent = parent

    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0,8)
    corner.Parent = frame

    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(1,-70,0,20)
    label.Position = UDim2.new(0,15,0,8)
    label.BackgroundTransparency = 1
    label.Text = text
    label.Font = Enum.Font.Gotham
    label.TextColor3 = Color3.fromRGB(255,255,255)
    label.TextSize = 14
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.Parent = frame

    local valueLabel = Instance.new("TextLabel")
    valueLabel.Size = UDim2.new(0,45,0,20)
    valueLabel.Position = UDim2.new(1,-55,0,8)
    valueLabel.BackgroundTransparency = 1
    valueLabel.Text = tostring(value)
    valueLabel.Font = Enum.Font.GothamBold
    valueLabel.TextColor3 = Color3.fromRGB(255,255,255)
    valueLabel.TextSize = 13
    valueLabel.Parent = frame

    local bar = Instance.new("Frame")
    bar.Size = UDim2.new(1,-30,0,5)
    bar.Position = UDim2.new(0,15,1,-20)
    bar.BackgroundColor3 = Color3.fromRGB(55,55,60)
    bar.BorderSizePixel = 0
    bar.Parent = frame

    local barCorner = Instance.new("UICorner")
    barCorner.CornerRadius = UDim.new(1,0)
    barCorner.Parent = bar

    local fill = Instance.new("Frame")
    fill.Size = UDim2.new((value-min)/(max-min),0,1,0)
    fill.BackgroundColor3 = Color3.fromRGB(0,150,255)
    fill.BorderSizePixel = 0
    fill.Parent = bar

    local fillCorner = Instance.new("UICorner")
    fillCorner.CornerRadius = UDim.new(1,0)
    fillCorner.Parent = fill

    local dragging = false

    local function update(input)
        local percent = math.clamp((input.Position.X - bar.AbsolutePosition.X) / bar.AbsoluteSize.X,0,1)
        local new = math.floor((min + ((max-min) * percent)) + 0.5)

        value = new
        valueLabel.Text = tostring(new)
        fill.Size = UDim2.new(percent,0,1,0)

        if callback then
            callback(new)
        end
    end

    bar.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            update(input)
        end
    end)

    UserInputService.InputChanged:Connect(function(input)
        if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
            update(input)
        end
    end)

    UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = false
        end
    end)
end

function ui_library:AddColorPicker(parent,text,default,callback)
    local colors = {
        Color3.fromRGB(255,255,255),
        Color3.fromRGB(255,0,0),
        Color3.fromRGB(0,255,0),
        Color3.fromRGB(0,170,255),
        Color3.fromRGB(255,255,0),
        Color3.fromRGB(255,0,255)
    }

    local index = 1

    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(1,-5,0,50)
    frame.BackgroundColor3 = Color3.fromRGB(35,35,40)
    frame.BorderSizePixel = 0
    frame.Parent = parent

    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0,8)
    corner.Parent = frame

    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(1,-80,1,0)
    label.Position = UDim2.new(0,15,0,0)
    label.BackgroundTransparency = 1
    label.Text = text
    label.Font = Enum.Font.Gotham
    label.TextColor3 = Color3.fromRGB(255,255,255)
    label.TextSize = 14
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.Parent = frame

    local button = Instance.new("TextButton")
    button.Size = UDim2.new(0,45,0,24)
    button.Position = UDim2.new(1,-60,0.5,-12)
    button.BackgroundColor3 = default
    button.Text = ""
    button.Parent = frame

    local buttonCorner = Instance.new("UICorner")
    buttonCorner.CornerRadius = UDim.new(0,6)
    buttonCorner.Parent = button

    button.MouseButton1Click:Connect(function()
        index += 1
        if index > #colors then
            index = 1
        end

        button.BackgroundColor3 = colors[index]

        if callback then
            callback(colors[index])
        end
    end)
end

local ui = ui_library:CreateUI()

ui:AddToggle(ui.container,"Enable SilentAim",false,function(state)
    toggle_states.SilentAim = state
end)

ui:AddSlider(ui.container,"Chance (%)",1,100,100,function(v)
    config_values.SilentAim.Chance = v
end)

ui:AddToggle(ui.container,"Draw FOV",true,function(state)
    config_values.SilentAim.Draw = state

    if not state then
        if drawing_table.SilentAim then
            for _,v in pairs(drawing_table.SilentAim) do
                v:Remove()
            end
        end
        return
    end

    drawing_table.SilentAim = {}

    for i = 1,60 do
        local line = Drawing.new("Line")
        line.Visible = true
        line.Thickness = 1.5
        line.Color = config_values.SilentAim.DrawColor
        drawing_table.SilentAim[i] = line
    end

    if connection_table.FOV then
        connection_table.FOV:Disconnect()
    end

    connection_table.FOV = RunService.RenderStepped:Connect(function()
    if not config_values.SilentAim.Draw then
        return
    end

    local camera = workspace.CurrentCamera
    local viewport = camera.ViewportSize

    local center = Vector2.new(
        viewport.X / 2,
        viewport.Y / 2
    )

    local radius = config_values.SilentAim.DrawSize
    local sides = #drawing_table.SilentAim

    for i,v in pairs(drawing_table.SilentAim) do
        local angle = math.rad((360 / sides) * i)
        local nextAngle = math.rad((360 / sides) * (i + 1))

        v.From = Vector2.new(
            center.X + math.cos(angle) * radius,
            center.Y + math.sin(angle) * radius
        )

        v.To = Vector2.new(
            center.X + math.cos(nextAngle) * radius,
            center.Y + math.sin(nextAngle) * radius
        )

        v.Color = config_values.SilentAim.DrawColor
    end
end)

ui:AddColorPicker(ui.container,"FOV Color",Color3.fromRGB(255,255,255),function(color)
    config_values.SilentAim.DrawColor = color
end)

ui:AddSlider(ui.container,"FOV Size",10,1000,150,function(v)
    config_values.SilentAim.DrawSize = v
end)

ui:AddToggle(ui.container,"Check Downed Players",false,function(state)
    config_values.SilentAim.CheckDowned = state
end)

ui:AddToggle(ui.container,"Check Team",false,function(state)
    config_values.SilentAim.CheckTeam = state
end)
