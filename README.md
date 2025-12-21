--// SERVICES
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")

local player = Players.LocalPlayer
local char = player.Character or player.CharacterAdded:Wait()
local humanoid = char:WaitForChild("Humanoid")
local hrp = char:WaitForChild("HumanoidRootPart")
local workspaceItems = workspace:WaitForChild("Items")

--// VARS
local selectedItem = {}
local cooldown = 0.3

local desiredWalkSpeed = humanoid.WalkSpeed
local desiredJumpPower = humanoid.JumpPower
local lockSpeed = false
local lockJump = false
local infiniteJump = false

--// RESPAWN FIX
player.CharacterAdded:Connect(function(c)
    char = c
    humanoid = c:WaitForChild("Humanoid")
    hrp = c:WaitForChild("HumanoidRootPart")
    task.wait(0.1)
    humanoid.UseJumpPower = true
    humanoid.WalkSpeed = desiredWalkSpeed
    humanoid.JumpPower = desiredJumpPower
end)

--// LOCK SPEED + JUMP
RunService.Heartbeat:Connect(function()
    if humanoid and humanoid.Parent then
        if lockSpeed and humanoid.WalkSpeed ~= desiredWalkSpeed then
            humanoid.WalkSpeed = desiredWalkSpeed
        end
        if lockJump then
            humanoid.UseJumpPower = true
            if humanoid.JumpPower ~= desiredJumpPower then
                humanoid.JumpPower = desiredJumpPower
            end
        end
    end
end)

--// INFINITE JUMP
UserInputService.JumpRequest:Connect(function()
    if infiniteJump and humanoid then
        humanoid:ChangeState(Enum.HumanoidStateType.Jumping)
    end
end)

--// DRAG
local function startDrag(item)
    pcall(function()
        ReplicatedStorage.RemoteEvents.RequestStartDraggingItem:FireServer(item)
    end)
end

local function stopDrag(item)
    pcall(function()
        ReplicatedStorage.RemoteEvents.StopDraggingItem:FireServer(item)
    end)
end

--// BRING
local function bringItem(item, pos)
    if item.Name:find("Chest") then return end
    local part = item:IsA("Model") and item.PrimaryPart or item:FindFirstChildWhichIsA("BasePart", true)
    if not part then return end
    local p = pos or (hrp.Position + Vector3.new(0,2,0))
    if item:IsA("Model") and item.PrimaryPart then
        item:SetPrimaryPartCFrame(CFrame.new(p))
    else
        part.CFrame = CFrame.new(p)
    end
end

--// GUI
local Luna = loadstring(game:HttpGet(
    "https://raw.githubusercontent.com/Nebula-Softworks/Luna-Interface-Suite/refs/heads/master/source.lua"
))()

local Window = Luna:CreateWindow({
    Name = "Bring Hub",
    LogoID = "82795327169782",
    KeySystem = false
})

--// BRING TAB
local BringTab = Window:CreateTab({Name="Bring",Icon="inventory",ImageSource="Material"})

BringTab:CreateSlider({
    Name="Cooldown",
    Range={0.05,1},
    CurrentValue=cooldown,
    Callback=function(v) cooldown=v end
})

BringTab:CreateDropdown({
    Name="Item",
    Options=(function()
        local t={}
        for _,v in ipairs(workspaceItems:GetChildren()) do
            if not table.find(t,v.Name) then table.insert(t,v.Name) end
        end
        return t
    end)(),
    MultipleOptions=true,
    Callback=function(opt) selectedItem=opt end
})

BringTab:CreateButton({
    Name="BRING",
    Callback=function()
        for _,item in ipairs(workspaceItems:GetChildren()) do
            if table.find(selectedItem,item.Name) then
                bringItem(item)
                startDrag(item)
                task.wait(cooldown)
                stopDrag(item)
            end
        end
    end
})

BringTab:CreateButton({
    Name="BRING ALL",
    Callback=function()
        for _,item in ipairs(workspaceItems:GetChildren()) do
            bringItem(item)
            startDrag(item)
            task.wait(cooldown)
            stopDrag(item)
        end
    end
})

BringTab:CreateButton({
    Name="Infinite HP",
    Callback=function()
        loadstring(game:HttpGet(
            "https://rawscripts.net/raw/99-Nights-in-the-Forest-Inf-Health-54808"
        ))()
    end
})

--// PLAYER TAB
local PlayerTab = Window:CreateTab({Name="Player",Icon="person",ImageSource="Material"})

PlayerTab:CreateSlider({
    Name="WalkSpeed",
    Range={16,500},
    CurrentValue=desiredWalkSpeed,
    Callback=function(v)
        desiredWalkSpeed=v
        humanoid.WalkSpeed=v
    end
})

PlayerTab:CreateToggle({
    Name="Lock Speed",
    Default=false,
    Callback=function(v) lockSpeed=v end
})

PlayerTab:CreateSlider({
    Name="JumpPower",
    Range={50,500},
    CurrentValue=desiredJumpPower,
    Callback=function(v)
        desiredJumpPower=v
        humanoid.UseJumpPower=true
        humanoid.JumpPower=v
    end
})

PlayerTab:CreateToggle({
    Name="Lock Jump",
    Default=false,
    Callback=function(v) lockJump=v end
})

PlayerTab:CreateToggle({
    Name="Infinite Jump",
    Default=false,
    Callback=function(v) infiniteJump=v end
})

--// AUTO TREE FARM
local ToolDamageObject = ReplicatedStorage.RemoteEvents.ToolDamageObject
local autoTreeFarm=false
local autoTreeThread

local function getAxe()
    local inv = player:FindFirstChild("Inventory")
    return inv and (inv:FindFirstChild("Old Axe") or inv:FindFirstChildWhichIsA("Tool"))
end

local function getTrees()
    local map=workspace:FindFirstChild("Map")
    if not map then return {} end
    local f=map:FindFirstChild("Foliage") or map:FindFirstChild("Landmarks")
    if not f then return {} end
    local t={}
    for _,v in ipairs(f:GetChildren()) do
        if v.Name=="Small Tree" and v:IsA("Model") then
            local tr=v:FindFirstChild("Trunk") or v.PrimaryPart
            if tr then table.insert(t,{tree=v,trunk=tr}) end
        end
    end
    return t
end

local AuraTab = Window:CreateTab({Name="Aura",Icon="flash_on",ImageSource="Material"})

AuraTab:CreateToggle({
    Name="Auto Tree Farm",
    Default=false,
    Callback=function(s)
        autoTreeFarm=s
        if s then
            autoTreeThread=task.spawn(function()
                while autoTreeFarm do
                    for _,t in ipairs(getTrees()) do
                        if not autoTreeFarm then break end
                        local axe=getAxe()
                        if axe then
                            hrp.CFrame=t.trunk.CFrame*CFrame.new(3,0,0)
                            task.wait(0.2)
                            axe.Parent=char
                            ToolDamageObject:InvokeServer(t.tree,axe,"1_8264699301",t.trunk.CFrame)
                        end
                        task.wait(0.5)
                    end
                    task.wait(1)
                end
            end)
        else
            if autoTreeThread then task.cancel(autoTreeThread) autoTreeThread=nil end
        end
    end
})

--// SETTING TAB
local SettingTab = Window:CreateTab({Name="Setting",Icon="settings",ImageSource="Material"})
local setclipboard = setclipboard or toclipboard or function() end

SettingTab:CreateButton({
    Name="Copy Discord",
    Callback=function()
        setclipboard("https://discord.gg/BV7vp2APC2")
    end
})
