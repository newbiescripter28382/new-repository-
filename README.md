-- https://scriptblox.com/script/4TH-OF-JULY-Tower-Defense-Simulator-Auto-TDS-44244

--// Services
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Workspace = game:GetService("Workspace")
local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer

--// GUI Setup
local gui = Instance.new("ScreenGui", game.CoreGui)
gui.Name = "AutoGUI_TDS"

-- GUI Elements (not in a frame to preserve positioning)
local toggleSkip = Instance.new("TextButton")
toggleSkip.Size = UDim2.new(0, 160, 0, 40)
toggleSkip.Position = UDim2.new(0, 10, 0, 10)
toggleSkip.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
toggleSkip.TextColor3 = Color3.fromRGB(255, 255, 255)
toggleSkip.Font = Enum.Font.SourceSansBold
toggleSkip.TextSize = 18
toggleSkip.Text = "Auto Skip [OFF]"
toggleSkip.BorderSizePixel = 0
toggleSkip.Parent = gui

local toggleUpgrade = Instance.new("TextButton")
toggleUpgrade.Size = UDim2.new(0, 160, 0, 40)
toggleUpgrade.Position = UDim2.new(0, 20, 0.5, -120)
toggleUpgrade.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
toggleUpgrade.TextColor3 = Color3.fromRGB(255, 255, 255)
toggleUpgrade.Font = Enum.Font.SourceSansBold
toggleUpgrade.TextSize = 18
toggleUpgrade.Text = "Auto Upgrade [OFF]"
toggleUpgrade.BorderSizePixel = 0
toggleUpgrade.Parent = gui

local upgradeInput = Instance.new("TextBox")
upgradeInput.Size = UDim2.new(0, 160, 0, 30)
upgradeInput.Position = UDim2.new(0, 20, 0.5, -80)
upgradeInput.PlaceholderText = "Skins, e.g. Crypto, PNG"
upgradeInput.Text = ""
upgradeInput.Visible = false
upgradeInput.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
upgradeInput.TextColor3 = Color3.fromRGB(255, 255, 255)
upgradeInput.ClearTextOnFocus = false
upgradeInput.Parent = gui

-- X Button to toggle GUI visibility
local toggleGuiButton = Instance.new("TextButton")
toggleGuiButton.Size = UDim2.new(0, 30, 0, 30)
toggleGuiButton.Position = UDim2.new(1, -40, 0, 10)
toggleGuiButton.BackgroundColor3 = Color3.fromRGB(255, 0, 0)
toggleGuiButton.TextColor3 = Color3.new(1, 1, 1)
toggleGuiButton.Text = "X"
toggleGuiButton.TextSize = 18
toggleGuiButton.Font = Enum.Font.SourceSansBold
toggleGuiButton.Parent = gui

-- Group for visibility toggle
local controls = {toggleSkip, toggleUpgrade, upgradeInput,ctaInput}

-- Toggle visibility on click
local isVisible = true
toggleGuiButton.MouseButton1Click:Connect(function()
    isVisible = not isVisible
    for _, item in ipairs(controls) do
        item.Visible = isVisible
    end
end)

-- State
local autoSkip = false
local upgrading = false
local autoCTA = false
local upgradeSkins = {}
local ctaSkin = ""
local upgradeCounts = {}

-- Special farm skin rules
local specialSkins = {
    ["default"] = true, ["arcade"] = true, ["crypto"] = true,
    ["tycoon"] = true, ["png"] = true, ["cozy camp"] = true,
    ["ducky"] = true, ["pirate"] = true
}

-- Helper Functions
local function parseInput(text)
    local t = {}
    for skin in text:gmatch("[^,]+") do
        skin = skin:match("^%s*(.-)%s*$")
        if skin ~= "" then table.insert(t, skin) end
    end
    return t
end

local function getLevelFromGui()
    for _, obj in pairs(game.CoreGui:GetDescendants()) do
        if obj:IsA("TextLabel") or obj:IsA("TextBox") then
            if obj.Name == "levelText" then
                local level = tonumber(obj.Text:match("Level:%s*(%d+)"))
                if level then return level end
            end
        end
    end
    return nil
end

local function fireStreaming(towerName)
    local args = {"Streaming", "SelectTower", "Farm", towerName}
    pcall(function()
        ReplicatedStorage:WaitForChild("RemoteEvent"):FireServer(unpack(args))
    end)
end

local function upgradeTower(tower)
    fireStreaming(tower.Name)
    local args = {
        "Troops", "Upgrade", "Set", {
            ["Troop"] = tower,
            ["Path"] = 1
        }
    }
    pcall(function()
        ReplicatedStorage:WaitForChild("RemoteFunction"):InvokeServer(unpack(args))
    end)
    task.wait(0)
    local level = getLevelFromGui()
    if level then upgradeCounts[tower] = level end
end

-- Toggle Handlers
toggleSkip.MouseButton1Click:Connect(function()
    autoSkip = not autoSkip
    toggleSkip.Text = autoSkip and "Auto Skip [ON]" or "Auto Skip [OFF]"
    toggleSkip.BackgroundColor3 = autoSkip and Color3.fromRGB(255, 170, 0) or Color3.fromRGB(50, 50, 50)
end)

toggleUpgrade.MouseButton1Click:Connect(function()
    upgrading = not upgrading
    toggleUpgrade.Text = upgrading and "Auto Upgrade [ON]" or "Auto Upgrade [OFF]"
    toggleUpgrade.BackgroundColor3 = upgrading and Color3.fromRGB(0, 170, 0) or Color3.fromRGB(50, 50, 50)
    upgradeInput.Visible = upgrading
    if upgrading then upgradeSkins = parseInput(upgradeInput.Text) end
end)

upgradeInput.FocusLost:Connect(function()
    if upgrading then upgradeSkins = parseInput(upgradeInput.Text) end
end)

-- Loops
task.spawn(function()
    while true do
        if autoSkip then
            pcall(function()
                ReplicatedStorage:WaitForChild("RemoteFunction"):InvokeServer("Voting", "Skip")
            end)
        end
        task.wait(0)
    end
end)

task.spawn(function()
    while true do
        task.wait(0)
        if upgrading and #upgradeSkins > 0 then
            local towersFolder = Workspace:FindFirstChild("Towers")
            if towersFolder then
                for _, skin in ipairs(upgradeSkins) do
                    local matching = {}
                    for _, tower in ipairs(towersFolder:GetChildren()) do
                        if tower.Name == skin then
                            table.insert(matching, tower)
                        end
                    end

                    if specialSkins[skin:lower()] then
                        local upgraded3 = 0
                        for _, t in ipairs(matching) do
                            if (upgradeCounts[t] or 0) >= 3 then upgraded3 += 1 end
                        end
                        if upgraded3 < 8 then
                            for _, t in ipairs(matching) do
                                if (upgradeCounts[t] or 0) < 3 then upgradeTower(t) end
                            end
                        else
                            for _, t in ipairs(matching) do
                                if (upgradeCounts[t] or 0) < 5 then
                                    upgradeTower(t)
                                    break
                                end
                            end
                        end
                    else
                        for _, t in ipairs(matching) do
                            upgradeTower(t)
                        end
                    end
                end
            end
        end
    end
end)

task.spawn(function()
    while true do
        if autoCTA and ctaSkin ~= "" then
            local towers = Workspace:FindFirstChild("Towers")
            if towers then
                for _, t in ipairs(towers:GetChildren()) do
                    if t.Name == ctaSkin then
                        local args = {
                            "Troops", "Abilities", "Activate",
                            {["Troop"] = t, ["Name"] = "Call Of Arms", ["Data"] = {}}
                        }
                        pcall(function()
                            ReplicatedStorage:WaitForChild("RemoteFunction"):InvokeServer(unpack(args))
                        end)
                        task.wait(0)
                    end
                end
            end
        else
            task.wait(0)
        end
    end
end)
