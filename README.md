local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")

local LocalPlayer = Players.LocalPlayer

local ESPEnabled = false
local SilentAimEnabled = false
local SpeedEnabled = false
local AntiStunEnabled = false

local PredictionEnabled = true
local PredictionAmount = 0.1
local MaxRange = 400
local SpeedValue = 700
local AntiStunPower = 1.2

local TargetPosition = nil
local ESPs = {}

local AllowedKeys = {
    [Enum.KeyCode.Z] = true,
    [Enum.KeyCode.X] = true,
    [Enum.KeyCode.C] = true,
    [Enum.KeyCode.V] = true,
    [Enum.KeyCode.F] = true
}

local function isSilentKeyDown()
    for key in pairs(AllowedKeys) do
        if UserInputService:IsKeyDown(key) then
            return true
        end
    end
    return false
end

local espFolder = game.CoreGui:FindFirstChild("PlayerESP")
if not espFolder then
    espFolder = Instance.new("Folder")
    espFolder.Name = "PlayerESP"
    espFolder.Parent = game.CoreGui
end

local function getHRP(char)
    return char and char:FindFirstChild("HumanoidRootPart")
end

local function isAllyWithMe(targetPlayer)
    local gui = LocalPlayer:FindFirstChild("PlayerGui")
    if not gui then return false end

    local scrolling =
        gui:FindFirstChild("Main")
        and gui.Main:FindFirstChild("Allies")
        and gui.Main.Allies:FindFirstChild("Container")
        and gui.Main.Allies.Container:FindFirstChild("Allies")
        and gui.Main.Allies.Container.Allies:FindFirstChild("ScrollingFrame")

    if scrolling then
        for _, frame in pairs(scrolling:GetDescendants()) do
            if frame:IsA("ImageButton") and frame.Name == targetPlayer.Name then
                return true
            end
        end
    end
    return false
end

local function isEnemy(plr)
    if not plr or plr == LocalPlayer then
        return false
    end

    local myTeam = LocalPlayer.Team
    local hisTeam = plr.Team

    if myTeam and hisTeam then
        if myTeam.Name == "Pirates" and hisTeam.Name == "Marines" then
            return true
        elseif myTeam.Name == "Marines" and hisTeam.Name == "Pirates" then
            return true
        end

        if myTeam.Name == "Pirates" and hisTeam.Name == "Pirates" then
            return not isAllyWithMe(plr)
        end

        if myTeam.Name == "Marines" and hisTeam.Name == "Marines" then
            return false
        end
    end

    return true
end

local function getMainColor(plr)
    if plr == LocalPlayer or isAllyWithMe(plr) then
        return Color3.fromRGB(0,255,0)
    elseif isEnemy(plr) then
        return Color3.fromRGB(255,255,0)
    end
    return Color3.fromRGB(0,255,0)
end

local function getPredicted(hrp)
    if not PredictionEnabled then
        return hrp.Position
    end
    return hrp.Position + (hrp.Velocity * PredictionAmount)
end

local function createESP(plr)
    if ESPs[plr] or not plr.Character then return end
    local head = plr.Character:FindFirstChild("Head")
    if not head then return end

    local gui = Instance.new("BillboardGui")
    gui.Name = plr.Name
    gui.Adornee = head
    gui.Size = UDim2.fromOffset(240,50)
    gui.StudsOffset = Vector3.new(0,3,0)
    gui.AlwaysOnTop = true
    gui.Parent = espFolder

    local lvl = Instance.new("TextLabel")
    lvl.Name = "Level"
    lvl.Size = UDim2.new(1,0,0.45,0)
    lvl.BackgroundTransparency = 1
    lvl.Font = Enum.Font.SourceSansBold
    lvl.TextSize = 13
    lvl.TextStrokeTransparency = 0.2
    lvl.TextColor3 = Color3.fromRGB(0,170,255)
    lvl.TextXAlignment = Enum.TextXAlignment.Center
    lvl.Parent = gui

    local main = Instance.new("TextLabel")
    main.Name = "Main"
    main.Size = UDim2.new(1,0,0.55,0)
    main.Position = UDim2.new(0,0,0.45,0)
    main.BackgroundTransparency = 1
    main.Font = Enum.Font.SourceSansBold
    main.TextSize = 14
    main.TextStrokeTransparency = 0.2
    main.TextXAlignment = Enum.TextXAlignment.Center
    main.Parent = gui

    ESPs[plr] = gui
end

local function getClosestHRP()
    local myHRP = getHRP(LocalPlayer.Character)
    if not myHRP then return nil end

    local closest, dist = nil, MaxRange
    for _, plr in ipairs(Players:GetPlayers()) do
        if plr ~= LocalPlayer and isEnemy(plr) and plr.Character then
            local hrp = getHRP(plr.Character)
            local hum = plr.Character:FindFirstChildOfClass("Humanoid")
            if hrp and hum and hum.Health > 0 then
                local d = (hrp.Position - myHRP.Position).Magnitude
                if d < dist then
                    dist = d
                    closest = hrp
                end
            end
        end
    end
    return closest
end

task.spawn(function()
    local mt = getrawmetatable(game)
    setreadonly(mt,false)
    local old
    old = hookmetamethod(game,"__namecall",function(self,...)
        local args = {...}
        if getnamecallmethod():lower() == "fireserver"
        and SilentAimEnabled
        and TargetPosition
        and typeof(args[1]) == "Vector3" then
            args[1] = TargetPosition
            return old(self, unpack(args))
        end
        return old(self,...)
    end)
    setreadonly(mt,true)
end)

RunService.RenderStepped:Connect(function()
    local char = LocalPlayer.Character
    local hrp = getHRP(char)
    local hum = char and char:FindFirstChildOfClass("Humanoid")
    if not hrp or not hum then return end

    if SpeedEnabled then
        hum.WalkSpeed = SpeedValue
    end

    if AntiStunEnabled then
        local dir = hum.MoveDirection
        if dir.Magnitude > 0 then
            hrp.CFrame = hrp.CFrame + dir.Unit * AntiStunPower
        end
    end

    if SilentAimEnabled and isSilentKeyDown() then
        local t = getClosestHRP()
        TargetPosition = t and getPredicted(t) or nil
    else
        TargetPosition = nil
    end

    for _, plr in ipairs(Players:GetPlayers()) do
        if plr ~= LocalPlayer then
            if not ESPs[plr] then
                createESP(plr)
            end

            local gui = ESPs[plr]
            local pChar = plr.Character
            local pHRP = getHRP(pChar)
            local pHum = pChar and pChar:FindFirstChildOfClass("Humanoid")

            if ESPEnabled and gui and pHRP and pHum then
                gui.Enabled = true
                gui.Adornee = pChar:FindFirstChild("Head")

                local dist = math.floor((hrp.Position - pHRP.Position).Magnitude)
                local lvl = "?"
                local data = plr:FindFirstChild("Data")
                if data and data:FindFirstChild("Level") then
                    lvl = data.Level.Value
                end

                gui.Level.Text = "Lv. "..lvl
                gui.Main.Text =
                    "["..math.floor(pHum.Health).."] "..plr.DisplayName.." ("..dist.."m)"
                gui.Main.TextColor3 = getMainColor(plr)
            elseif gui then
                gui.Enabled = false
            end
        end
    end
end)

Players.PlayerRemoving:Connect(function(plr)
    if ESPs[plr] then
        ESPs[plr]:Destroy()
        ESPs[plr] = nil
    end
end)

UserInputService.InputBegan:Connect(function(input,gp)
    if gp then return end

    if input.KeyCode == Enum.KeyCode.L then
        ESPEnabled = not ESPEnabled
    elseif input.KeyCode == Enum.KeyCode.B then
        SilentAimEnabled = not SilentAimEnabled
    elseif input.KeyCode == Enum.KeyCode.K then
        SpeedEnabled = not SpeedEnabled
        AntiStunEnabled = SpeedEnabled
    elseif input.KeyCode == Enum.KeyCode.P then
        AntiStunEnabled = not AntiStunEnabled
    end
end)
