if getgenv().RCAHubUnload then
    pcall(getgenv().RCAHubUnload)
end

--// Services
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local StarterGui = game:GetService("StarterGui")
local Workspace = game:GetService("Workspace")
local StarterPlayer = game:GetService("StarterPlayer")
local VirtualInputManager = game:GetService("VirtualInputManager")

local LocalPlayer = Players.LocalPlayer
local Camera = Workspace.CurrentCamera

--// Fluent
local Request = request
assert(type(Request) == "function", "request() required")

local function LoadURL(URL)
    local Response = Request({
        Url = URL,
        Method = "GET"
    })

    local Body = Response and (Response.Body or Response.body)
    assert(type(Body) == "string", "Failed to load " .. URL)

    return assert(loadstring(Body))()
end

local Fluent = LoadURL("https://github.com/dawid-scripts/Fluent/releases/latest/download/main.lua")
local SaveManager = LoadURL("https://raw.githubusercontent.com/dawid-scripts/Fluent/master/Addons/SaveManager.lua")
local InterfaceManager = LoadURL("https://raw.githubusercontent.com/dawid-scripts/Fluent/master/Addons/InterfaceManager.lua")

--// Config
local CraftConfig
local LeaderboardConfig
local FishingConfig

pcall(function()
    CraftConfig = require(ReplicatedStorage:WaitForChild("CraftConfig"))
end)

pcall(function()
    LeaderboardConfig = require(ReplicatedStorage:WaitForChild("LeaderboardConfig"))
end)

pcall(function()
    FishingConfig = require(ReplicatedStorage:WaitForChild("FishingConfig"))
end)

--// Remotes
local CraftRemotes = ReplicatedStorage:FindFirstChild("CraftRemotes")
local RequestCraft = CraftRemotes and CraftRemotes:FindFirstChild("RequestCraft")
local CraftResult = CraftRemotes and CraftRemotes:FindFirstChild("CraftResult")

local MarketRemotes = ReplicatedStorage:FindFirstChild("MarketRemotes")
local SellItem = MarketRemotes and MarketRemotes:FindFirstChild("SellItem")

--// Game Auto Farm Remotes
local AutoFarmRemotes = ReplicatedStorage:FindFirstChild("AutoFarmRemotes")
local AutoFarmAction = AutoFarmRemotes and AutoFarmRemotes:FindFirstChild("AutoFarmAction")
local AutoFarmState = AutoFarmRemotes and AutoFarmRemotes:FindFirstChild("AutoFarmState")

--// Fishing Remotes
local FishingRemotes = ReplicatedStorage:FindFirstChild("FishingRemotes")
local FishingAction = FishingRemotes and FishingRemotes:FindFirstChild("FishingAction")
local FishingState = FishingRemotes and FishingRemotes:FindFirstChild("FishingState")
local FishingHoldInput = FishingRemotes and FishingRemotes:FindFirstChild("HoldInput")
local FishingShopAction = FishingRemotes and FishingRemotes:FindFirstChild("ShopAction")

--// Shop
local BuyRemote = ReplicatedStorage:FindFirstChild("Buy")
local ShopItems

pcall(function()
    ShopItems = require(ReplicatedStorage:WaitForChild("Items"))
end)

local ShootEvent
pcall(function()
    ShootEvent = ReplicatedStorage:WaitForChild("CE_Resource"):WaitForChild("Events"):WaitForChild("ShootEvent")
end)

--// Farm Data
local Farms = {
    Banana = {
        Tools = {"Axe", "Axe(P)"},
        BuyTool = "Axe",
        Material = "BananaLeaf",
        MaxStack = 150,
        Zone = CFrame.new(-2754.88, -30, 3986.60),
        Distance = 5.5
    },
    Tank = {
        Tools = {"Crowbar", "Crowbar(P)"},
        BuyTool = "Crowbar",
        Material = "ScrapMetal",
        MaxStack = 200,
        Zone = CFrame.new(-2238.379, -49, 4504.090),
        Distance = 11
    }
}

if CraftConfig and type(CraftConfig.Materials) == "table" then
    for _, Material in pairs(CraftConfig.Materials) do
        for _, Farm in pairs(Farms) do
            if Material.Id == Farm.Material and Material.MaxStack then
                Farm.MaxStack = Material.MaxStack
            end
        end
    end
end

local CraftPosition = Vector3.new(-2103, -65, 3269)
local SellPosition = CFrame.new(-2087.526, -61.485, 3234.564)

--// Fishing
local FishingShopPosition = CFrame.new(-1627.63, -61.7, 3274)
local FishingFallbackPosition = CFrame.new(-1504.5, -59, 3128.5)

--// Auto Farm Safe
local SafeFarmCenter = Vector3.new(-2753, -31, 3991)
local SafeFarmRadius = 150
local SafeZone = CFrame.new(-1901, -49, 2456)

--// State
local State = {
    Alive = true,
    Selling = false,
    CurrentFarm = nil,
    EnteredFarm = false,
    Target = nil,

    GameFarmRunning = false,
    LastFarmStart = 0,
    LastExtraM1 = 0,

    BuyingFarmTool = false,
    LastFarmToolBuy = 0,
    LastFarmToolNotify = 0,

    SafeActive = false,
    SafeThreatCount = 0,
    LastSafeTeleport = 0,

    UndergroundActive = false,
    UndergroundForce = nil,
    UndergroundParts = {},

    FishingMode = "idle",
    FishingAutoRunning = false,
    FishingFull = false,
    FishingFullAmount = 0,
    FishingBuying = false,
    FishingSelling = false,
    FishingHolding = false,
    FishingAutoToken = 0,
    FishingPrevI = nil,
    FishingPrevG = nil,
    FishingPrevTick = nil,
    FishingIVelocity = 0,
    FishingGVelocity = 0,
    FishingVisualBind = false,
    FishingIndicatorConn = nil,
    FishingGreenConn = nil,
    FishingVisibleConn = nil,
    LastFishingStart = 0,
    LastFishingAutoOn = 0,
    LastBaitBuy = 0,

    CraftBusy = false,
    LastCraft = 0,
    LastSell = 0,

    WalkSpeed = 16,

    ESP = {},
    CharParts = {},
    NoclipParts = {},

    FlyVelocity = nil,
    FlyGyro = nil,

    TPTool = nil,

    RemoteActionBusy = false,
    LastRemoteAction = 0
}

local Connections = {}

local function Connect(Signal, Callback)
    local Connection = Signal:Connect(Callback)
    table.insert(Connections, Connection)
    return Connection
end

--// Basic Functions
local function Character()
    return LocalPlayer.Character
end

local function Humanoid()
    local Char = Character()
    return Char and Char:FindFirstChildOfClass("Humanoid")
end

local function Root()
    local Char = Character()
    return Char and Char:FindFirstChild("HumanoidRootPart")
end

local function Notify(Title, Text)
    pcall(function()
        StarterGui:SetCore("SendNotification", {
            Title = Title,
            Text = Text,
            Duration = 3
        })
    end)
end

local function Teleport(CF)
    local HRP = Root()
    if not HRP then return false end
    HRP.AssemblyLinearVelocity = Vector3.zero
    HRP.AssemblyAngularVelocity = Vector3.zero
    HRP.CFrame = CF
    return true
end

local function StopMoving()
    local HRP = Root()
    local Hum = Humanoid()
    if HRP and Hum then
        Hum:MoveTo(HRP.Position)
    end
end

local function Distance(A, B)
    return (Vector3.new(A.X, 0, A.Z) - Vector3.new(B.X, 0, B.Z)).Magnitude
end

local function Face(Position)
    local HRP = Root()
    if not HRP then return end
    local Target = Vector3.new(Position.X, HRP.Position.Y, Position.Z)
    if (Target - HRP.Position).Magnitude < 0.05 then return end
    HRP.CFrame = CFrame.lookAt(HRP.Position, Target)
end

local function GetInstanceCFrame(inst)
    if not inst then return nil end
    if inst:IsA("BasePart") then
        return inst.CFrame
    elseif inst:IsA("Model") then
        return inst:GetPivot() or (inst.PrimaryPart and inst.PrimaryPart.CFrame)
    end
    local part = inst:FindFirstChildWhichIsA("BasePart", true)
    return part and part.CFrame
end

local function UpdateCharParts()
    table.clear(State.CharParts)
    local Char = Character()
    if not Char then return end
    for _, Part in ipairs(Char:GetDescendants()) do
        if Part:IsA("BasePart") then
            table.insert(State.CharParts, Part)
        end
    end
end

--// Material Management
local function GetMaterialObject(Id)
    local Folder = LocalPlayer:FindFirstChild("Materials")
    return Folder and Folder:FindFirstChild(Id)
end

local function GetAmount(Id)
    local Object = GetMaterialObject(Id)
    if not Object then return 0 end
    if Object:IsA("IntValue") or Object:IsA("NumberValue") then
        return tonumber(Object.Value) or 0
    end
    return tonumber(Object:GetAttribute("Amount")) or 0
end

local function GetMaxStack(Farm)
    local Object = GetMaterialObject(Farm.Material)
    if Object then
        local Max = Object:GetAttribute("MaxStack")
        if Max then return tonumber(Max) or Farm.MaxStack end
    end
    return Farm.MaxStack
end

local function IsFull(Farm)
    return GetAmount(Farm.Material) >= GetMaxStack(Farm)
end

--// Tool Handler
local function GetFarmTool(Farm)
    local Char = Character()
    local Backpack = LocalPlayer:FindFirstChildOfClass("Backpack")
    if not Char then return end

    for _, Name in ipairs(Farm.Tools) do
        local Tool = Char:FindFirstChild(Name)
        if Tool and Tool:IsA("Tool") then return Tool end
    end

    if Backpack then
        for _, Name in ipairs(Farm.Tools) do
            local Tool = Backpack:FindFirstChild(Name)
            if Tool and Tool:IsA("Tool") then
                local Hum = Humanoid()
                if Hum then
                    Hum:EquipTool(Tool)
                    task.wait(0.1)
                end
                return Tool
            end
        end
    end
end

local function EnsureFarmTool(Farm)
    local Tool = GetFarmTool(Farm)
    if Tool then
        return Tool
    end

    if
        State.BuyingFarmTool or
        not BuyRemote or
        not BuyRemote:IsA("RemoteFunction") then

        return nil
    end

    local BuyName = Farm.BuyTool or Farm.Tools[1]
    if not BuyName then
        return nil
    end

    -- กัน Remote ซื้อรัวเมื่อเงินไม่พอ/ร้านปฏิเสธ
    if os.clock() - State.LastFarmToolBuy < 2.5 then
        return nil
    end

    State.BuyingFarmTool = true
    State.LastFarmToolBuy = os.clock()

    local Success, Result = pcall(function()
        return BuyRemote:InvokeServer(BuyName)
    end)

    if not Success or not Result then
        State.BuyingFarmTool = false

        if os.clock() - State.LastFarmToolNotify >= 5 then
            State.LastFarmToolNotify = os.clock()
            Notify(
                "Auto Farm",
                "ไม่มี " .. BuyName .. " และซื้ออัตโนมัติไม่สำเร็จ"
            )
        end

        return nil
    end

    -- รอ replication ของ Tool เข้า Backpack/Character แล้ว equip ต่อ
    local Deadline = os.clock() + 2

    repeat
        task.wait(0.1)
        Tool = GetFarmTool(Farm)
    until Tool or os.clock() >= Deadline

    State.BuyingFarmTool = false

    if Tool then
        Notify(
            "Auto Farm",
            "ซื้อ " .. BuyName .. " สำเร็จ → เริ่มฟาร์มต่อ"
        )
    elseif os.clock() - State.LastFarmToolNotify >= 5 then
        State.LastFarmToolNotify = os.clock()
        Notify(
            "Auto Farm",
            "ซื้อ " .. BuyName .. " แล้ว แต่ยังไม่พบ Tool"
        )
    end

    return Tool
end

local function NormalM1(Tool)
    if not Tool or not Tool.Parent then return false end

    local Char = Character()
    local Hum = Humanoid()

    if Tool.Parent ~= Char and Hum then
        Hum:EquipTool(Tool)
        task.wait(0.05)
    end

    return pcall(function()
        Tool:Activate()
    end)
end

--// Target Finding
local function GetTargetPart(Model)
    if not Model then return nil end
    return Model.PrimaryPart or Model:FindFirstChildWhichIsA("BasePart")
end

local function GetTargets(FarmId)
    local Targets = {}

    if FarmId == "Banana" then
        local Spawners = Workspace:FindFirstChild("Spawners")
        if not Spawners then return Targets end

        for _, Object in ipairs(Spawners:GetChildren()) do
            if Object:IsA("Model") and string.find(string.lower(Object.Name), "banana") then
                local Part = GetTargetPart(Object)
                if Part then
                    table.insert(Targets, { Model = Object, Part = Part })
                end
            elseif Object:IsA("Folder") then
                for _, SubObject in ipairs(Object:GetChildren()) do
                    if SubObject:IsA("Model") and string.find(string.lower(SubObject.Name), "banana") then
                        local Part = GetTargetPart(SubObject)
                        if Part then
                            table.insert(Targets, { Model = SubObject, Part = Part })
                        end
                    end
                end
            end
        end

    elseif FarmId == "Tank" then
        local System = Workspace:FindFirstChild("FARMCROWBARSYSTEM")
        local Folder = System and System:FindFirstChild("SpawnedTanks")
        if not Folder then return Targets end

        for _, Model in ipairs(Folder:GetChildren()) do
            if Model:IsA("Model") then
                local Part = GetTargetPart(Model)
                if Part then
                    table.insert(Targets, { Model = Model, Part = Part })
                end
            end
        end
    end

    return Targets
end

local function TargetValid(Target)
    return Target and Target.Model and Target.Model.Parent and Target.Part and Target.Part.Parent
end

local function GetNearestTarget(FarmId)
    local HRP = Root()
    if not HRP then return end

    local Best = nil
    local BestDistance = math.huge

    for _, Target in ipairs(GetTargets(FarmId)) do
        if TargetValid(Target) then
            local CurrentDistance = Distance(HRP.Position, Target.Part.Position)
            if CurrentDistance < BestDistance then
                Best = Target
                BestDistance = CurrentDistance
            end
        end
    end

    return Best
end

--// UI Init
local Window = Fluent:CreateWindow({
    Title = "จำลองชีวิตทหารไทย [RCA]",
    SubTitle = "made by BatmanScript",
    TabWidth = 150,
    Size = UDim2.fromOffset(590, 500),
    Acrylic = false,
    Theme = "Dark",
    MinimizeKey = Enum.KeyCode.RightControl
})

local Tabs = {
    Main = Window:AddTab({ Title = "Main", Icon = "home" }),
    ESP = Window:AddTab({ Title = "ESP", Icon = "eye" }),
    Player = Window:AddTab({ Title = "Player", Icon = "user" }),
    Teleport = Window:AddTab({ Title = "Teleport", Icon = "map-pin" }),
    Misc = Window:AddTab({ Title = "Misc", Icon = "wrench" }),
    Settings = Window:AddTab({ Title = "Settings", Icon = "settings" })
}

local Options = Fluent.Options

--// Gun Camera
local NO_GUN_ZOOM_FOV = 70
local DEFAULT_MAX_ZOOM = StarterPlayer.CameraMaxZoomDistance
local NO_ZOOM_BIND = "RCA_NoGunZoom"

local function GetEquippedGun()
    local Char = Character()
    if not Char then return nil end

    for _, Object in ipairs(Char:GetChildren()) do
        if Object:IsA("Tool") and Object:FindFirstChild("ConfigMods") then
            return Object
        end
    end

    return nil
end

local function ApplyNoGunZoom()
    if not Options.NoGunZoom or not Options.NoGunZoom.Value then return end
    if not GetEquippedGun() then return end

    if LocalPlayer.CameraMaxZoomDistance ~= DEFAULT_MAX_ZOOM then
        LocalPlayer.CameraMaxZoomDistance = DEFAULT_MAX_ZOOM
    end

    if math.abs(Camera.FieldOfView - NO_GUN_ZOOM_FOV) > 0.01 then
        Camera.FieldOfView = NO_GUN_ZOOM_FOV
    end
end


--// Auto Farm Safe Helpers
local function GetFarmThreatCount()
    local Count = 0

    for _, Player in ipairs(Players:GetPlayers()) do
        if Player ~= LocalPlayer then
            local Char = Player.Character
            local HRP = Char and Char:FindFirstChild("HumanoidRootPart")
            local Hum = Char and Char:FindFirstChildOfClass("Humanoid")

            if HRP and Hum and Hum.Health > 0 then
                local Offset = Vector2.new(
                    HRP.Position.X - SafeFarmCenter.X,
                    HRP.Position.Z - SafeFarmCenter.Z
                )

                if Offset.Magnitude <= SafeFarmRadius then
                    Count += 1
                end
            end
        end
    end

    return Count
end

local function HasFarmThreat()
    local Count = GetFarmThreatCount()
    State.SafeThreatCount = Count
    return Count > 0
end

--// MAIN TAB
Tabs.Main:AddSection("Auto Farm")

local FarmDropdown = Tabs.Main:AddDropdown("FarmType", {
    Title = "Farm",
    Values = {"Banana", "Tank"},
    Multi = false,
    Default = 1
})

local AutoFarmToggle = Tabs.Main:AddToggle("AutoFarm", {
    Title = "Auto Farm",
    Description = "ใช้ระบบ Auto Farm เดิมของเกม + M1 เสริม",
    Default = false
})

local AutoFarmSafeToggle = Tabs.Main:AddToggle("AutoFarmSafe", {
    Title = "Auto Farm Safe",
    Description = "มีผู้เล่นในรัศมี 150 รอบฟาร์มกล้วย → ไป Safe Zone",
    Default = false
})

local UndergroundFarmToggle = Tabs.Main:AddToggle("UndergroundFarm", {
    Title = "Underground Farm",
    Description = "ให้ Auto Farm เดินใต้ระดับฟาร์ม โดยยังใช้ระบบเดินของเกม",
    Default = false
})

Tabs.Main:AddSlider("UndergroundYOffset", {
    Title = "Underground Y Offset",
    Description = "ระดับ Y จากพื้นฟาร์ม เช่น -5 = ต่ำกว่าพื้น 5 studs",
    Min = -20,
    Max = 0,
    Default = -5,
    Rounding = 1
})

local ExtraM1Toggle = Tabs.Main:AddToggle("ExtraFarmM1", {
    Title = "Extra M1",
    Description = "ตอนเกมสั่ง swing จะ Tool:Activate เพิ่ม โดยไม่คลิก UI",
    Default = true
})

local AutoSellToggle = Tabs.Main:AddToggle("AutoSellFull", {
    Title = "Auto Sell When Full",
    Description = "ของเต็ม → หยุดฟาร์ม → ขาย → กลับมาฟาร์ม",
    Default = false
})

local SellFarmMaterial

local function RestoreUndergroundCollision()
    for Part, Original in pairs(State.UndergroundParts) do
        if Part and Part.Parent then
            if Options.Noclip and Options.Noclip.Value then
                Part.CanCollide = false
            else
                Part.CanCollide = Original
            end
        end
    end

    table.clear(State.UndergroundParts)
end

local function StopUndergroundFarm()
    State.UndergroundActive = false

    if State.UndergroundForce then
        pcall(function()
            State.UndergroundForce:Destroy()
        end)

        State.UndergroundForce = nil
    end

    RestoreUndergroundCollision()
end

local function StartUndergroundFarm()
    if
        not Options.UndergroundFarm.Value or
        not Options.AutoFarm.Value or
        State.Selling or
        State.SafeActive then

        StopUndergroundFarm()
        return
    end

    local Farm = Farms[
        Options.FarmType.Value or "Banana"
    ]

    local HRP = Root()
    local Char = Character()

    if not Farm or not HRP or not Char then
        return
    end

    local TargetY =
        Farm.Zone.Position.Y +
        Options.UndergroundYOffset.Value

    -- ปิด Collision เฉพาะช่วงฟาร์มใต้ดิน
    for _, Part in ipairs(Char:GetDescendants()) do
        if Part:IsA("BasePart") then
            if State.UndergroundParts[Part] == nil then
                local Original = Part.CanCollide

                if State.NoclipParts[Part] ~= nil then
                    Original = State.NoclipParts[Part]
                end

                State.UndergroundParts[Part] = Original
            end

            Part.CanCollide = false
        end
    end

    local Force = State.UndergroundForce

    if not Force or not Force.Parent then
        Force = Instance.new("BodyPosition")
        Force.Name = "RCA_UndergroundY"
        Force.MaxForce = Vector3.new(0, 1e8, 0)
        Force.P = 18000
        Force.D = 1250
        Force.Parent = HRP

        State.UndergroundForce = Force
    end

    Force.Position = Vector3.new(
        HRP.Position.X,
        TargetY,
        HRP.Position.Z
    )

    State.UndergroundActive = true

    -- กันหลุดตก Void หรือถูกแรงอื่นดึง Y ออกไปไกลเกิน
    if HRP.Position.Y < TargetY - 10 then
        local Rotation = HRP.CFrame - HRP.Position

        HRP.AssemblyLinearVelocity = Vector3.new(
            HRP.AssemblyLinearVelocity.X,
            0,
            HRP.AssemblyLinearVelocity.Z
        )

        HRP.CFrame =
            CFrame.new(
                HRP.Position.X,
                TargetY,
                HRP.Position.Z
            ) * Rotation
    end
end

local function StopGameFarm()
    State.GameFarmRunning = false

    if AutoFarmAction then
        pcall(function()
            AutoFarmAction:FireServer("stop")
        end)
    end

    StopMoving()
end

local function EnterSafeZone()
    if State.Selling then return end

    if not State.SafeActive then
        State.SafeActive = true
        StopUndergroundFarm()
        StopGameFarm()

        if os.clock() - State.LastSafeTeleport > 0.5 then
            State.LastSafeTeleport = os.clock()
            Teleport(SafeZone)
        end

        Notify(
            "Auto Farm Safe",
            "พบผู้เล่นในระยะ " .. tostring(SafeFarmRadius) .. " → Safe Zone"
        )
    end
end

local function LeaveSafeZone()
    if not State.SafeActive then return end

    State.SafeActive = false
    State.GameFarmRunning = false
    State.CurrentFarm = nil
    State.EnteredFarm = false

    Notify(
        "Auto Farm Safe",
        "พื้นที่ปลอดภัยแล้ว → กลับไปฟาร์ม"
    )
end

local function StartGameFarm()
    if not State.Alive or not Options.AutoFarm.Value or State.Selling then
        return
    end

    if not AutoFarmAction or not AutoFarmState then
        Notify("Auto Farm", "ไม่พบ AutoFarmRemotes ของเกม")
        return
    end

    if os.clock() - State.LastFarmStart < 0.75 then
        return
    end

    local FarmId = Options.FarmType.Value or "Banana"
    local Farm = Farms[FarmId]
    if not Farm then return end

    if
        FarmId == "Banana" and
        Options.AutoFarmSafe.Value and
        HasFarmThreat() then

        EnterSafeZone()
        return
    end

    if State.SafeActive then
        LeaveSafeZone()
    end

    if Options.AutoSellFull.Value and IsFull(Farm) then
        if SellFarmMaterial then
            task.spawn(SellFarmMaterial, Farm)
        end
        return
    end

    -- หา Tool ก่อน ถ้าไม่มีให้ซื้อผ่าน Remote ร้านของเกมอัตโนมัติ
    local Tool = EnsureFarmTool(Farm)
    if not Tool then
        return
    end

    State.LastFarmStart = os.clock()

    -- เข้าโซนแค่ตอนเริ่ม/เปลี่ยนฟาร์ม
    if State.CurrentFarm ~= FarmId or not State.EnteredFarm then
        StopGameFarm()
        Teleport(Farm.Zone)

        State.CurrentFarm = FarmId
        State.EnteredFarm = true
        State.Target = nil

        task.wait(0.3)
    end

    -- เช็ค/equip ซ้ำหลัง TP เผื่อ Character/Tool state เปลี่ยน
    Tool = GetFarmTool(Farm) or Tool

    if Tool.Parent ~= Character() then
        local Hum = Humanoid()
        if Hum then
            pcall(function()
                Hum:EquipTool(Tool)
            end)
            task.wait(0.1)
        end
    end

    task.wait(0.12)

    pcall(function()
        AutoFarmAction:FireServer("start", FarmId)
    end)
end

--// Auto Sell
SellFarmMaterial = function(Farm)
    if State.Selling or not SellItem then return end
    if os.clock() - State.LastSell < 1.5 then return end

    State.Selling = true
    State.LastSell = os.clock()

    StopUndergroundFarm()
    StopGameFarm()

    local Stall = Workspace:FindFirstChild("GlobalMarketStall", true)
    if Stall then
        local Part = Stall:IsA("BasePart")
            and Stall
            or Stall:FindFirstChildWhichIsA("BasePart", true)

        Teleport(
            Part and
            (Part.CFrame * CFrame.new(0, 3, 3))
            or SellPosition
        )
    else
        Teleport(SellPosition)
    end

    task.wait(0.5)

    local Before = GetAmount(Farm.Material)

    for _ = 1, 3 do
        local Current = GetAmount(Farm.Material)
        if Current <= 0 then break end

        pcall(function()
            SellItem:FireServer(Farm.Material, Current)
        end)

        local Timeout = os.clock() + 1.2

        repeat
            task.wait(0.1)
        until
            GetAmount(Farm.Material) < Current or
            os.clock() >= Timeout

        if GetAmount(Farm.Material) < Current then
            break
        end
    end

    local After = GetAmount(Farm.Material)

    Notify(
        "Auto Sell",
        After < Before
            and (Farm.Material .. " " .. tostring(Before) .. " → " .. tostring(After))
            or "Sell failed"
    )

    State.Selling = false
    State.GameFarmRunning = false
    State.CurrentFarm = nil
    State.EnteredFarm = false

    if State.Alive and Options.AutoFarm.Value then
        task.wait(0.25)

        local FarmId = Options.FarmType.Value or "Banana"

        if
            FarmId == "Banana" and
            Options.AutoFarmSafe.Value and
            HasFarmThreat() then

            EnterSafeZone()
        else
            State.SafeActive = false
            StartGameFarm()
        end
    end
end

--// ฟัง State จาก Auto Farm เดิมของเกม
if AutoFarmState then
    Connect(AutoFarmState.OnClientEvent, function(Action, Data)
        Data = type(Data) == "table" and Data or {}

        if Action == "start" then
            State.GameFarmRunning = true

        elseif Action == "swing" then
            if
                Options.AutoFarm.Value and
                Options.ExtraFarmM1.Value and
                not State.Selling and
                os.clock() - State.LastExtraM1 >= 0.2 then

                State.LastExtraM1 = os.clock()

                local Farm = Farms[
                    Options.FarmType.Value or "Banana"
                ]

                if Farm then
                    local Tool = GetFarmTool(Farm)

                    if Tool then
                        -- เสริม Tool Activate โดยตรง ไม่จำลอง Mouse Click
                        task.spawn(function()
                            NormalM1(Tool)
                        end)
                    end
                end
            end

        elseif Action == "reward" then
            local Farm = Farms[
                Options.FarmType.Value or "Banana"
            ]

            if
                Farm and
                Options.AutoSellFull.Value and
                IsFull(Farm) then

                task.spawn(SellFarmMaterial, Farm)
            end

        elseif Action == "stop" then
            State.GameFarmRunning = false

            local Farm = Farms[
                Options.FarmType.Value or "Banana"
            ]

            if
                Farm and
                Options.AutoSellFull.Value and
                (
                    Data.reason == "full" or
                    IsFull(Farm)
                ) then

                task.spawn(SellFarmMaterial, Farm)
            end

        elseif Action == "denied" then
            State.GameFarmRunning = false
        end
    end)
end

--// Underground Y Controller
Connect(RunService.Heartbeat, function()
    if
        Options.UndergroundFarm.Value and
        Options.AutoFarm.Value and
        not State.Selling and
        not State.SafeActive then

        StartUndergroundFarm()
    elseif State.UndergroundActive or State.UndergroundForce then
        StopUndergroundFarm()
    end
end)

UndergroundFarmToggle:OnChanged(function()
    if Options.UndergroundFarm.Value then
        if Options.AutoFarm.Value and not State.Selling and not State.SafeActive then
            StartUndergroundFarm()
        end
    else
        StopUndergroundFarm()
    end
end)

--// Recovery / Full-stack watcher
task.spawn(function()
    while State.Alive do
        if Options.AutoFarm.Value and not State.Selling then
            local FarmId = Options.FarmType.Value or "Banana"
            local Farm = Farms[FarmId]

            if Farm then
                -- Auto Sell มีสิทธิ์ก่อน Safe เพื่อไม่ให้ของเต็มค้าง
                if Options.AutoSellFull.Value and IsFull(Farm) then
                    SellFarmMaterial(Farm)

                elseif
                    FarmId == "Banana" and
                    Options.AutoFarmSafe.Value and
                    HasFarmThreat() then

                    EnterSafeZone()

                elseif State.SafeActive then
                    LeaveSafeZone()
                    task.wait(0.2)
                    StartGameFarm()

                elseif not State.GameFarmRunning then
                    StartGameFarm()
                end
            end
        end

        task.wait(0.25)
    end
end)

AutoFarmToggle:OnChanged(function()
    if Options.AutoFarm.Value then
        if Options.AutoFishing and Options.AutoFishing.Value then
            pcall(function()
                Options.AutoFishing:SetValue(false)
            end)
        end

        State.GameFarmRunning = false
        State.CurrentFarm = nil
        State.EnteredFarm = false
        State.Target = nil
        State.SafeActive = false

        task.spawn(StartGameFarm)
    else
        StopUndergroundFarm()
        StopGameFarm()

        State.CurrentFarm = nil
        State.EnteredFarm = false
        State.Target = nil
        State.SafeActive = false
    end
end)

AutoFarmSafeToggle:OnChanged(function()
    if not Options.AutoFarmSafe.Value then
        if State.SafeActive then
            State.SafeActive = false

            if Options.AutoFarm.Value and not State.Selling then
                State.CurrentFarm = nil
                State.EnteredFarm = false
                task.spawn(StartGameFarm)
            end
        end

        return
    end

    if
        Options.AutoFarm.Value and
        (Options.FarmType.Value or "Banana") == "Banana" and
        HasFarmThreat() then

        EnterSafeZone()
    end
end)

FarmDropdown:OnChanged(function()
    StopUndergroundFarm()
    StopGameFarm()

    State.CurrentFarm = nil
    State.EnteredFarm = false
    State.Target = nil
    State.SafeActive = false

    if Options.AutoFarm.Value then
        task.delay(0.25, StartGameFarm)
    end
end)

AutoSellToggle:OnChanged(function()
    if not Options.AutoSellFull.Value or not Options.AutoFarm.Value then
        return
    end

    local Farm = Farms[
        Options.FarmType.Value or "Banana"
    ]

    if Farm and IsFull(Farm) then
        task.spawn(SellFarmMaterial, Farm)
    end
end)

--// AUTO FISHING
Tabs.Main:AddSection("Auto Fishing")

local FishingRodValues = {}
local FishingRodMap = {}
local FishingBaitValues = {}
local FishingBaitMap = {}

local function FishingComma(Number)
    local Text = tostring(math.floor(tonumber(Number) or 0))

    while true do
        local NewText, Count = Text:gsub("^(-?%d+)(%d%d%d)", "%1,%2")
        Text = NewText
        if Count == 0 then break end
    end

    return Text
end

local FishingRods = FishingConfig and FishingConfig.Rods or {
    {
        Id = "Basic",
        ToolName = "Fishing Rod",
        DisplayName = "คันเบ็ดมาตรฐาน",
        Price = 450
    },
    {
        Id = "VIP",
        ToolName = "VIP Fishing Rod",
        DisplayName = "คันเบ็ด VIP",
        Price = 350
    }
}

for _, Rod in ipairs(FishingRods) do
    local Display =
        tostring(Rod.DisplayName or Rod.ToolName or Rod.Id) ..
        " | ฿" ..
        FishingComma(Rod.Price or 0)

    FishingRodMap[Display] = Rod
    table.insert(FishingRodValues, Display)
end

local BaitPrice = FishingConfig and FishingConfig.BaitPrice or 15
local BaitPacks = FishingConfig and FishingConfig.BaitPackSizes or {1, 10, 50}

for _, Amount in ipairs(BaitPacks) do
    local Display =
        "เหยื่อ x" ..
        tostring(Amount) ..
        " | ฿" ..
        FishingComma(Amount * BaitPrice)

    FishingBaitMap[Display] = Amount
    table.insert(FishingBaitValues, Display)
end

Tabs.Main:AddDropdown("FishingRod", {
    Title = "Fishing Rod",
    Description = "เลือกคันเบ็ดที่ใช้ Auto Fishing",
    Values = FishingRodValues,
    Multi = false,
    Default = 1
})

Tabs.Main:AddDropdown("FishingBaitPack", {
    Title = "Bait Pack",
    Description = "เกมนี้มีเหยื่อชนิดเดียว เลือกจำนวนแพ็กที่จะซื้อ",
    Values = FishingBaitValues,
    Multi = false,
    Default = #FishingBaitValues
})

local AutoFishingToggle = Tabs.Main:AddToggle("AutoFishing", {
    Title = "Auto Fishing",
    Description = "เริ่มตกปลาอัตโนมัติ โดยไม่เปิด Auto Farm ของเกมเอง",
    Default = false
})

Tabs.Main:AddToggle("FishingGameAutoAfterE", {
    Title = "Game Auto After E",
    Description = "เปิด Auto ของเกมหลัง cast เฉพาะเมื่อ Auto Fishing เปิดคู่กัน",
    Default = false
})

Tabs.Main:AddToggle("FishingMinigameAssist", {
    Title = "Minigame Assist",
    Description = "ล็อกขีด GUI ไว้กลาง Green Zone + คุม HoldInput อัตโนมัติ",
    Default = true
})

Tabs.Main:AddToggle("FishingAutoTP", {
    Title = "Auto TP Fishing Zone",
    Description = "วาปเข้า Fishing Zone ก่อนเริ่มอัตโนมัติ",
    Default = true
})

Tabs.Main:AddToggle("FishingAutoBuyBait", {
    Title = "Auto Buy Bait",
    Description = "ซื้อแพ็กเหยื่อที่เลือกเมื่อเหยื่อเหลือน้อย",
    Default = false
})

Tabs.Main:AddSlider("FishingBaitThreshold", {
    Title = "Buy Bait When ≤",
    Min = 0,
    Max = 50,
    Default = 5,
    Rounding = 0
})

local function GetFishingBait()
    local Folder = LocalPlayer:FindFirstChild("Fishing")
    local Bait = Folder and Folder:FindFirstChild("Bait")

    return Bait and tonumber(Bait.Value) or 0
end

local function GetFishingMaterialAmount()
    local Id =
        FishingConfig and
        FishingConfig.AutoFarm and
        FishingConfig.AutoFarm.MaterialId or
        "FishPart"

    return GetAmount(Id)
end

local function SelectedFishingRod()
    return FishingRodMap[Options.FishingRod.Value]
end

local function GetFishingRodTool()
    local Rod = SelectedFishingRod()
    if not Rod then return nil end

    local Char = Character()
    local Backpack = LocalPlayer:FindFirstChildOfClass("Backpack")
    local ToolName = Rod.ToolName

    local Tool = Char and Char:FindFirstChild(ToolName)

    if not Tool and Backpack then
        Tool = Backpack:FindFirstChild(ToolName)
    end

    return Tool
end

local function EquipFishingRod()
    local Tool = GetFishingRodTool()

    if not Tool then
        return false
    end

    local Char = Character()
    local Hum = Humanoid()

    if Char and Tool.Parent ~= Char and Hum then
        Hum:EquipTool(Tool)
        task.wait(0.15)
    end

    return Tool.Parent == Char
end

local function IsInsideFishingZone(Position)
    local Folder = Workspace:FindFirstChild("FishingZones")
    if not Folder then return false end

    local Radius = FishingConfig and FishingConfig.ZoneCheckRadius or 6

    for _, Zone in ipairs(Folder:GetChildren()) do
        if Zone:IsA("BasePart") then
            local LocalPosition = Zone.CFrame:PointToObjectSpace(Position)
            local Half = Zone.Size * 0.5

            if
                math.abs(LocalPosition.X) <= Half.X + Radius and
                math.abs(LocalPosition.Y) <= Half.Y + Radius and
                math.abs(LocalPosition.Z) <= Half.Z + Radius then

                return true
            end
        end
    end

    return false
end

local function GetFishingZoneCFrame()
    local Folder = Workspace:FindFirstChild("FishingZones")
    local HRP = Root()

    if not Folder then
        return FishingFallbackPosition
    end

    local Best
    local BestDistance = math.huge

    for _, Zone in ipairs(Folder:GetChildren()) do
        if Zone:IsA("BasePart") then
            local Distance =
                HRP and
                (HRP.Position - Zone.Position).Magnitude or
                0

            if Distance < BestDistance then
                Best = Zone
                BestDistance = Distance
            end
        end
    end

    if not Best then
        return FishingFallbackPosition
    end

    -- อยู่ใน volume ของ Fishing Zone แต่ใกล้ขอบบน ไม่ลงลึกเกินไป
    return Best.CFrame * CFrame.new(
        0,
        math.max(0, Best.Size.Y * 0.5 - 2),
        0
    )
end

local function BuyFishingBait(Silent)
    if
        State.FishingBuying or
        not FishingShopAction or
        not FishingShopAction:IsA("RemoteFunction") then

        return false
    end

    if os.clock() - State.LastBaitBuy < 2 then
        return false
    end

    local Amount = FishingBaitMap[Options.FishingBaitPack.Value]
    if not Amount then return false end

    State.FishingBuying = true
    State.LastBaitBuy = os.clock()

    local Success, Result = pcall(function()
        return FishingShopAction:InvokeServer("buyBait", Amount)
    end)

    State.FishingBuying = false

    local OK =
        Success and
        type(Result) == "table" and
        Result.ok == true

    if not Silent then
        Notify(
            "Fishing Bait",
            OK
                and (Result.message or ("ซื้อเหยื่อ x" .. Amount .. " สำเร็จ"))
                or (
                    type(Result) == "table" and
                    Result.message or
                    "ซื้อเหยื่อไม่สำเร็จ"
                )
        )
    end

    return OK
end

local function BuyFishingRod()
    local Rod = SelectedFishingRod()

    if
        not Rod or
        not FishingShopAction or
        not FishingShopAction:IsA("RemoteFunction") then

        Notify("Fishing", "ไม่พบ ShopAction")
        return
    end

    local Success, Result = pcall(function()
        return FishingShopAction:InvokeServer("buyRod", Rod.Id)
    end)

    Notify(
        "Fishing Rod",
        Success and
        type(Result) == "table" and
        (Result.message or (Result.ok and "ซื้อสำเร็จ" or "ซื้อไม่สำเร็จ")) or
        "ซื้อคันเบ็ดไม่สำเร็จ"
    )
end

local function SellAllFish()
    if
        State.FishingSelling or
        not FishingShopAction or
        not FishingShopAction:IsA("RemoteFunction") then

        return false
    end

    State.FishingSelling = true

    local Success, Result = pcall(function()
        return FishingShopAction:InvokeServer("sellFish", "all")
    end)

    State.FishingSelling = false

    local OK =
        Success and
        type(Result) == "table" and
        Result.data ~= nil

    Notify(
        "Fishing",
        OK and "ขายปลาทั้งหมดแล้ว" or "ไม่มีปลาให้ขาย / ขายไม่สำเร็จ"
    )

    return OK
end

local StartAutoFishing

local function SetFishingHold(Value)
    Value = Value == true

    if State.FishingHolding == Value then
        return
    end

    State.FishingHolding = Value

    if FishingHoldInput then
        pcall(function()
            FishingHoldInput:FireServer(Value)
        end)
    end
end

local function TryGameAutoAfterCast()
    if not FishingAction then return end

    State.FishingAutoToken += 1
    local Token = State.FishingAutoToken

    task.spawn(function()
        -- ให้ server เปลี่ยนจาก start -> cast ให้เสร็จก่อน
        task.wait(0.35)

        for Attempt = 1, 7 do
            if
                not State.Alive or
                Token ~= State.FishingAutoToken or
                State.FishingMode == "auto" or
                State.FishingMode == "idle" or
                State.FishingMode == "fighting" then

                return
            end

            State.LastFishingAutoOn = os.clock()

            pcall(function()
                FishingAction:FireServer("autoOn")
            end)

            task.wait(0.45)
        end
    end)
end

local FISHING_VISUAL_BIND = "RCA_FishingInstantGreen"

local function GetFishingMinigameObjects()
    local PlayerGui = LocalPlayer:FindFirstChild("PlayerGui")
    local Gui = PlayerGui and PlayerGui:FindFirstChild("FishingGui")
    local RootGui = Gui and Gui:FindFirstChild("Root")
    local Minigame = RootGui and RootGui:FindFirstChild("Minigame")
    local Track = Minigame and Minigame:FindFirstChild("Track")
    local Indicator = Track and Track:FindFirstChild("Indicator")
    local GreenZone = Track and Track:FindFirstChild("GreenZone")

    return Indicator, GreenZone, Minigame
end

local function ForceFishingIndicatorToGreen()
    if
        not Options.FishingMinigameAssist.Value then

        return
    end

    local Indicator, GreenZone, Minigame =
        GetFishingMinigameObjects()

    if
        not Indicator or
        not GreenZone or
        not Minigame or
        not Minigame.Visible then

        return
    end

    -- Indicator/GreenZone เป็น Frame ใต้ Track เดียวกัน
    -- และ AnchorPoint เดียวกัน 0.5,0.5
    -- จึงย้าย Indicator ไปทับจุดกึ่งกลาง GreenZone โดยตรง.
    local Target = GreenZone.Position

    if Indicator.Position ~= Target then
        Indicator.Position = Target
    end
end

local function DisconnectFishingVisualSignals()
    if State.FishingIndicatorConn then
        State.FishingIndicatorConn:Disconnect()
        State.FishingIndicatorConn = nil
    end

    if State.FishingGreenConn then
        State.FishingGreenConn:Disconnect()
        State.FishingGreenConn = nil
    end

    if State.FishingVisibleConn then
        State.FishingVisibleConn:Disconnect()
        State.FishingVisibleConn = nil
    end
end

local function BindFishingVisualSignals()
    DisconnectFishingVisualSignals()

    local Indicator, GreenZone, Minigame =
        GetFishingMinigameObjects()

    if not Indicator or not GreenZone or not Minigame then
        return false
    end

    -- จุดสำคัญ:
    -- FishingUI ของเกมเขียน Indicator.Position ใหม่ทุก Render frame.
    -- เราดัก PropertyChanged ของ Position โดยตรง ทำให้ทันทีที่เกมเขียนค่า i
    -- ขีดถูกเขียนกลับไป GreenZone.Position ใน event เดียวกัน.
    State.FishingIndicatorConn =
        Indicator:GetPropertyChangedSignal("Position"):Connect(function()
            if
                State.FishingVisualBind and
                Options.FishingMinigameAssist.Value and
                Minigame.Visible and
                Indicator.Position ~= GreenZone.Position then

                Indicator.Position = GreenZone.Position
            end
        end)

    State.FishingGreenConn =
        GreenZone:GetPropertyChangedSignal("Position"):Connect(function()
            if
                State.FishingVisualBind and
                Options.FishingMinigameAssist.Value and
                Minigame.Visible and
                Indicator.Position ~= GreenZone.Position then

                Indicator.Position = GreenZone.Position
            end
        end)

    State.FishingVisibleConn =
        Minigame:GetPropertyChangedSignal("Visible"):Connect(function()
            if
                State.FishingVisualBind and
                Options.FishingMinigameAssist.Value and
                Minigame.Visible then

                Indicator.Position = GreenZone.Position
            end
        end)

    return true
end

local function StartFishingVisualLock()
    if State.FishingVisualBind then
        ForceFishingIndicatorToGreen()
        return
    end

    State.FishingVisualBind = true
    BindFishingVisualSignals()

    pcall(function()
        RunService:UnbindFromRenderStep(FISHING_VISUAL_BIND)
    end)

    -- สำรองอีกชั้นหลัง Render loop ของเกม
    RunService:BindToRenderStep(
        FISHING_VISUAL_BIND,
        Enum.RenderPriority.Last.Value + 100,
        function()
            if
                not State.Alive or
                not Options.FishingMinigameAssist.Value then

                return
            end

            ForceFishingIndicatorToGreen()
        end
    )

    ForceFishingIndicatorToGreen()
end

local function StopFishingVisualLock()
    State.FishingVisualBind = false
    DisconnectFishingVisualSignals()

    pcall(function()
        RunService:UnbindFromRenderStep(FISHING_VISUAL_BIND)
    end)
end

Options.FishingMinigameAssist:OnChanged(function()
    if Options.FishingMinigameAssist.Value then
        if State.FishingMode == "fighting" then
            StartFishingVisualLock()
        end
    else
        StopFishingVisualLock()
        SetFishingHold(false)
    end
end)

local function ResetFishingAssistTracking()
    State.FishingPrevI = nil
    State.FishingPrevG = nil
    State.FishingPrevTick = nil
    State.FishingIVelocity = 0
    State.FishingGVelocity = 0
end

local function RunFishingMinigameTick(Data)
    if
        not Options.FishingMinigameAssist.Value or
        State.FishingMode ~= "fighting" or
        not FishingHoldInput then

        return
    end

    local Indicator = tonumber(Data.i)
    local Green = tonumber(Data.g)
    local Width = tonumber(Data.w) or 0.11

    if not Indicator or not Green then
        return
    end

    local Now = os.clock()
    local Dt =
        State.FishingPrevTick and
        math.clamp(Now - State.FishingPrevTick, 0.025, 0.12) or
        0.05

    if State.FishingPrevI ~= nil then
        local RawIVelocity =
            (Indicator - State.FishingPrevI) / Dt

        State.FishingIVelocity =
            State.FishingIVelocity * 0.45 +
            RawIVelocity * 0.55
    end

    if State.FishingPrevG ~= nil then
        local RawGVelocity =
            (Green - State.FishingPrevG) / Dt

        State.FishingGVelocity =
            State.FishingGVelocity * 0.35 +
            RawGVelocity * 0.65
    end

    State.FishingPrevI = Indicator
    State.FishingPrevG = Green
    State.FishingPrevTick = Now

    -- Server เป็นคนคำนวณ i จริง และ client ส่งได้เพียง Hold=true/false
    -- จึงเล็ง "กลางกรอบ" ล่วงหน้าจากความเร็วของทั้ง Indicator และ Green Zone.
    local InGreen = tonumber(Data.k) == 1
    local HalfGreen = math.max(0.025, Width)
    local RelativeVelocity =
        State.FishingIVelocity -
        State.FishingGVelocity

    -- Predict 1-2 server ticks ล่วงหน้าเพื่อลด overshoot.
    local Horizon = InGreen and 0.075 or 0.105

    local PredictedIndicator =
        Indicator +
        State.FishingIVelocity * Horizon

    local PredictedGreen =
        Green +
        State.FishingGVelocity * Horizon

    PredictedIndicator =
        math.clamp(PredictedIndicator, 0, 1)

    PredictedGreen =
        math.clamp(PredictedGreen, 0, 1)

    local Error = PredictedGreen - PredictedIndicator

    -- ถ้าอยู่ในกรอบแล้ว ให้เบรกก่อนถึงขอบเพื่อเกาะกลางมากที่สุด.
    local CenterBand =
        math.clamp(
            HalfGreen * (InGreen and 0.16 or 0.08),
            0.006,
            0.025
        )

    local Hold

    if Error > CenterBand then
        Hold = true

        -- กำลังวิ่งเข้าหากลางเร็วอยู่แล้ว ให้ปล่อยก่อนเพื่อลดการเลยกรอบ.
        if
            InGreen and
            RelativeVelocity > 0.25 and
            Error < HalfGreen * 0.42 then

            Hold = false
        end

    elseif Error < -CenterBand then
        Hold = false

        -- กำลังไหลกลับเร็วเกินไป ให้กดพยุงก่อนถึงกลาง.
        if
            InGreen and
            RelativeVelocity < -0.25 and
            -Error < HalfGreen * 0.42 then

            Hold = true
        end

    else
        -- ใกล้กลางมาก: ใช้ทิศความเร็วเป็นตัวเบรก.
        Hold = RelativeVelocity < 0
    end

    SetFishingHold(Hold)
end

local function StopAutoFishing()
    State.FishingMode = "idle"
    State.FishingAutoRunning = false
    State.FishingAutoToken += 1
    ResetFishingAssistTracking()
    StopFishingVisualLock()
    SetFishingHold(false)

    if FishingAction then
        pcall(function()
            FishingAction:FireServer("autoOff")
        end)

        task.wait(0.05)

        pcall(function()
            FishingAction:FireServer("cancel")
        end)
    end
end

StartAutoFishing = function()
    if
        not State.Alive or
        not Options.AutoFishing.Value or
        State.FishingSelling then

        return
    end

    if
        not FishingAction or
        not FishingState then

        Notify("Auto Fishing", "ไม่พบ FishingRemotes")
        return
    end

    if State.FishingFull then
        local Current = GetFishingMaterialAmount()

        if Current < State.FishingFullAmount then
            State.FishingFull = false
        else
            return
        end
    end

    if os.clock() - State.LastFishingStart < 1.5 then
        return
    end

    if Options.AutoFarm.Value then
        pcall(function()
            Options.AutoFarm:SetValue(false)
        end)
    end

    StopUndergroundFarm()
    StopGameFarm()

    if
        Options.FishingAutoBuyBait.Value and
        GetFishingBait() <= Options.FishingBaitThreshold.Value then

        BuyFishingBait(true)
        task.wait(0.2)
    end

    if GetFishingBait() <= 0 then
        Notify("Auto Fishing", "เหยื่อหมด")
        return
    end

    if not EquipFishingRod() then
        local Rod = SelectedFishingRod()

        Notify(
            "Auto Fishing",
            "ไม่พบ " .. tostring(Rod and Rod.ToolName or "Fishing Rod")
        )

        return
    end

    local HRP = Root()
    if not HRP then return end

    if not IsInsideFishingZone(HRP.Position) then
        if not Options.FishingAutoTP.Value then
            Notify("Auto Fishing", "ต้องอยู่ใน Fishing Zone")
            return
        end

        Teleport(GetFishingZoneCFrame())
        task.wait(0.45)
    end

    State.LastFishingStart = os.clock()
    State.FishingMode = "starting"
    State.FishingAutoRunning = false

    pcall(function()
        FishingAction:FireServer("start")
    end)
end

if FishingState then
    Connect(FishingState.OnClientEvent, function(Action, Data)
        Data = type(Data) == "table" and Data or {}

        if Action == "cast" then
            State.FishingMode = "waiting"
            State.FishingAutoRunning = false
            SetFishingHold(false)

            if
                Options.AutoFishing.Value and
                Options.FishingGameAutoAfterE.Value and
                FishingAction then

                TryGameAutoAfterCast()
            end

        elseif Action == "bite" then
            -- ถ้าไม่ได้เปิด Game Auto คู่กับ Auto Fishing จะเล่นมินิเกมจริง
            State.FishingMode = "fighting"
            State.FishingAutoRunning = false
            State.FishingAutoToken += 1
            ResetFishingAssistTracking()
            StartFishingVisualLock()
            SetFishingHold(false)

        elseif Action == "tick" then
            RunFishingMinigameTick(Data)

        elseif Action == "autoStart" then
            State.FishingMode = "auto"
            State.FishingAutoRunning = true
            State.FishingFull = false
            State.FishingAutoToken += 1
            ResetFishingAssistTracking()
            StopFishingVisualLock()
            SetFishingHold(false)

        elseif Action == "autoTick" or Action == "autoCatch" then
            State.FishingMode = "auto"
            State.FishingAutoRunning = true
            SetFishingHold(false)

        elseif Action == "autoStopNoBait" then
            State.FishingMode = "idle"
            State.FishingAutoRunning = false
            State.FishingAutoToken += 1
            SetFishingHold(false)

            if
                Options.AutoFishing.Value and
                Options.FishingAutoBuyBait.Value then

                task.delay(0.5, function()
                    if BuyFishingBait(false) then
                        task.wait(0.3)
                        StartAutoFishing()
                    end
                end)
            end

        elseif Action == "autoStopFull" then
            State.FishingMode = "idle"
            State.FishingAutoRunning = false
            State.FishingAutoToken += 1
            State.FishingFull = true
            State.FishingFullAmount = GetFishingMaterialAmount()
            StopFishingVisualLock()
            SetFishingHold(false)

            Notify(
                "Auto Fishing",
                "FishPart เต็ม — ลดจำนวนชิ้นส่วนปลาก่อน แล้วระบบจะเริ่มต่อเอง"
            )

        elseif Action == "caught" then
            State.FishingMode = "idle"
            State.FishingAutoRunning = false
            State.FishingAutoToken += 1
            ResetFishingAssistTracking()
            StopFishingVisualLock()
            SetFishingHold(false)

            -- Full Auto เริ่มรอบถัดไปเอง
            if Options.AutoFishing.Value then
                task.delay(0.8, StartAutoFishing)
            end

        elseif Action == "escaped" or Action == "cancelled" then
            State.FishingMode = "idle"
            State.FishingAutoRunning = false
            State.FishingAutoToken += 1
            ResetFishingAssistTracking()
            StopFishingVisualLock()
            SetFishingHold(false)

            if Options.AutoFishing.Value then
                task.delay(0.8, StartAutoFishing)
            end

        elseif Action == "denied" then
            State.FishingMode = "idle"
            State.FishingAutoRunning = false
            State.FishingAutoToken += 1
            SetFishingHold(false)
        end
    end)
end

task.spawn(function()
    while State.Alive do
        if Options.AutoFishing.Value then
            if
                Options.FishingAutoBuyBait.Value and
                GetFishingBait() <= Options.FishingBaitThreshold.Value and
                os.clock() - State.LastBaitBuy >= 3 then

                BuyFishingBait(true)
            end

            if State.FishingFull then
                if GetFishingMaterialAmount() < State.FishingFullAmount then
                    State.FishingFull = false
                    task.wait(0.2)
                    StartAutoFishing()
                end

            elseif
                State.FishingMode == "idle" and
                os.clock() - State.LastFishingStart >= 2 then

                StartAutoFishing()
            end
        end

        task.wait(0.75)
    end
end)

AutoFishingToggle:OnChanged(function()
    if Options.AutoFishing.Value then
        if Options.AutoFarm.Value then
            pcall(function()
                Options.AutoFarm:SetValue(false)
            end)
        end

        State.FishingMode = "idle"
        State.FishingAutoRunning = false
        State.FishingFull = false

        task.spawn(StartAutoFishing)
    else
        StopAutoFishing()
    end
end)

Tabs.Main:AddButton({
    Title = "Equip Fishing Rod",
    Callback = function()
        Notify(
            "Fishing Rod",
            EquipFishingRod() and "Equipped" or "ไม่พบคันเบ็ดที่เลือก"
        )
    end
})

Tabs.Main:AddButton({
    Title = "Buy Selected Rod",
    Callback = BuyFishingRod
})

Tabs.Main:AddButton({
    Title = "Buy Bait",
    Callback = function()
        BuyFishingBait(false)
    end
})

Tabs.Main:AddButton({
    Title = "Sell All Fish",
    Description = "ขายปลาในถังทั้งหมดผ่าน Fishing Shop",
    Callback = SellAllFish
})

Tabs.Main:AddButton({
    Title = "Fishing Status",
    Callback = function()
        local MaterialId =
            FishingConfig and
            FishingConfig.AutoFarm and
            FishingConfig.AutoFarm.MaterialId or
            "FishPart"

        Notify(
            "Fishing Status",
            "Bait: " ..
            tostring(GetFishingBait()) ..
            " | " ..
            tostring(MaterialId) ..
            ": " ..
            tostring(GetFishingMaterialAmount()) ..
            " | Mode: " ..
            tostring(State.FishingMode) ..
            " | Hold: " ..
            tostring(State.FishingHolding)
        )
    end
})

--// SHOP
Tabs.Main:AddSection("Shop")

local ShopValues = {}
local ShopMap = {}

local function FormatMoney(Number)
    local Text = tostring(math.floor(tonumber(Number) or 0))

    while true do
        local NewText, Count = Text:gsub("^(-?%d+)(%d%d%d)", "%1,%2")

        Text = NewText

        if Count == 0 then
            break
        end
    end

    return Text
end

if type(ShopItems) == "table" then
    local Names = {}

    for ItemName in pairs(ShopItems) do
        table.insert(Names, ItemName)
    end

    table.sort(Names, function(A, B)
        return string.lower(A) < string.lower(B)
    end)

    for _, ItemName in ipairs(Names) do
        local Data = ShopItems[ItemName]
        local Cost = type(Data) == "table" and tonumber(Data.Cost) or 0
        local Display = ItemName .. "  |  ฿" .. FormatMoney(Cost)

        ShopMap[Display] = ItemName
        table.insert(ShopValues, Display)
    end
end

if #ShopValues == 0 then
    local Fallback = {
        {"Axe", 155},
        {"Crowbar", 200},
        {"Barret-m114", 17500},
        {"Glock 19", 2200},
        {"HK416 Modded", 4850},
        {"M110", 11500},
        {"M16A4", 3500},
        {"MK18 EO", 4550},
        {"MK18 LPVO", 4550},
        {"MK18 TA31H", 4550},
        {"Noveske N4 Shorty", 4650},
        {"Noveske N4 Shorty2", 4650},
        {"WARSPORT LVOA, 14.7", 4650}
    }

    table.sort(Fallback, function(A, B)
        return string.lower(A[1]) < string.lower(B[1])
    end)

    for _, Data in ipairs(Fallback) do
        local Display = Data[1] .. "  |  ฿" .. FormatMoney(Data[2])

        ShopMap[Display] = Data[1]
        table.insert(ShopValues, Display)
    end
end

local ShopDropdown = Tabs.Main:AddDropdown("ShopItem", {
    Title = "Select Item",
    Description = "เลือกของที่ต้องการซื้อ",
    Values = ShopValues,
    Multi = false,
    Default = #ShopValues > 0 and 1 or nil,
    Searchable = true
})

Tabs.Main:AddButton({
    Title = "Buy Item",
    Description = "ซื้อไอเทมที่เลือกผ่านระบบร้านของเกม",

    Callback = function()
        if not BuyRemote or not BuyRemote:IsA("RemoteFunction") then
            Notify("Shop", "ReplicatedStorage.Buy not found")
            return
        end

        local Selected = Options.ShopItem.Value
        local ItemName = ShopMap[Selected]

        if not ItemName then
            Notify("Shop", "กรุณาเลือกไอเทม")
            return
        end

        local Success, Result = pcall(function()
            return BuyRemote:InvokeServer(ItemName)
        end)

        if not Success then
            Notify("Shop", "Remote error")
            return
        end

        if Result then
            Notify("Shop", "ซื้อ " .. ItemName .. " สำเร็จ")
        else
            Notify("Shop", "ซื้อ " .. ItemName .. " ไม่สำเร็จ")
        end
    end
})

--// AUTO CRAFT
Tabs.Main:AddSection("Auto Craft")

local RecipeValues = {}
local RecipeMap = {}

local function AddRecipe(Name, Id)
    if not Id then return end
    local Display = tostring(Name) .. " [" .. tostring(Id) .. "]"
    if RecipeMap[Display] then return end
    RecipeMap[Display] = Id
    table.insert(RecipeValues, Display)
end

if CraftConfig and type(CraftConfig.Recipes) == "table" then
    for _, Recipe in pairs(CraftConfig.Recipes) do
        if type(Recipe) == "table" and Recipe.Id then
            AddRecipe(Recipe.Name or Recipe.Id, Recipe.Id)
        end
    end
end

if #RecipeValues == 0 then
    AddRecipe("Axe", "Axe(P)")
    AddRecipe("Baton", "Baton(P)")
    AddRecipe("Fuel Can", "FuelCan")
    AddRecipe("Repair Kit", "RepairKit")
    AddRecipe("Steel", "Steel")
    AddRecipe("Gun Part", "MakeGunPart")
    AddRecipe("Powder", "MakePowder")
end

Tabs.Main:AddDropdown("CraftItem", {
    Title = "Craft Item",
    Values = RecipeValues,
    Multi = false,
    Default = 1
})

local AutoCraftToggle = Tabs.Main:AddToggle("AutoCraft", {
    Title = "Auto Craft",
    Default = false
})

if CraftResult then
    Connect(CraftResult.OnClientEvent, function(Data)
        if type(Data) ~= "table" then return end
        if Data.State == "start" then
            State.CraftBusy = true
        elseif Data.State == "done" or Data.State == "failed" then
            State.CraftBusy = false
            State.LastCraft = 0
        end
    end)
end

task.spawn(function()
    while State.Alive do
        if Options.AutoCraft.Value and not Options.AutoFarm.Value and not State.Selling and RequestCraft then
            local HRP = Root()
            if HRP then
                if (HRP.Position - CraftPosition).Magnitude > 10 then
                    Teleport(CFrame.new(CraftPosition))
                    task.wait(0.2)
                end

                if not State.CraftBusy and os.clock() - State.LastCraft >= 0.6 then
                    local RecipeId = RecipeMap[Options.CraftItem.Value]
                    if RecipeId then
                        State.LastCraft = os.clock()
                        RequestCraft:FireServer(RecipeId)
                    end
                end
            end
        end
        task.wait(0.15)
    end
end)

--// ESP TAB
Tabs.ESP:AddSection("Player ESP")

Tabs.ESP:AddToggle("ESPChams", { Title = "Chams", Default = false })
Tabs.ESP:AddToggle("ESPName", { Title = "Name", Default = false })
Tabs.ESP:AddToggle("ESPRank", { Title = "ยศ", Default = false })
Tabs.ESP:AddToggle("ESPHealth", { Title = "Health", Default = false })

local function GetRank(Player)
    local Leaderstats = Player:FindFirstChild("leaderstats")
    if Leaderstats then
        for _, Name in ipairs({"Rank", "ยศ", "Role"}) do
            local Value = Leaderstats:FindFirstChild(Name)
            if Value then return tostring(Value.Value) end
        end
    end

    local Tier = Player:GetAttribute("LB_Tier")
    local Rank = Player:GetAttribute("LB_Rank")
    if LeaderboardConfig and Tier and Rank then
        local Groups = LeaderboardConfig.GROUPS
        local Group = Groups and Groups[Tier]
        local Info = Group and Group.Ranks and Group.Ranks[Rank]
        if Info then
            return tostring(Info.Abbrev or Info.Name or Rank)
        end
    end
    return "N/A"
end

local function RemoveESP(Player)
    local Data = State.ESP[Player]
    if not Data then return end
    if Data.Highlight then Data.Highlight:Destroy() end
    if Data.Gui then Data.Gui:Destroy() end
    State.ESP[Player] = nil
end

local function CreateESP(Player)
    if Player == LocalPlayer then return end
    RemoveESP(Player)

    local Char = Player.Character
    local Hum = Char and Char:FindFirstChildOfClass("Humanoid")
    local Head = Char and Char:FindFirstChild("Head")
    if not Char or not Hum or not Head then return end

    local Highlight = Instance.new("Highlight")
    Highlight.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
    Highlight.FillTransparency = 0.65
    Highlight.OutlineTransparency = 0
    Highlight.Enabled = false
    Highlight.Parent = Char

    local Gui = Instance.new("BillboardGui")
    Gui.AlwaysOnTop = true
    Gui.Size = UDim2.fromOffset(250, 100)
    Gui.StudsOffset = Vector3.new(0, 3.5, 0)
    Gui.MaxDistance = 5000
    Gui.Parent = Head

    local Label = Instance.new("TextLabel")
    Label.Size = UDim2.fromScale(1, 1)
    Label.BackgroundTransparency = 1
    Label.Font = Enum.Font.GothamBold
    Label.TextSize = 13
    Label.TextColor3 = Color3.new(1, 1, 1)
    Label.TextStrokeTransparency = 0.2
    Label.TextWrapped = true
    Label.Parent = Gui

    State.ESP[Player] = {
        Character = Char,
        Humanoid = Hum,
        Highlight = Highlight,
        Gui = Gui,
        Label = Label,
        Rank = GetRank(Player)
    }
end

for _, Player in ipairs(Players:GetPlayers()) do
    if Player ~= LocalPlayer and Player.Character then
        task.defer(CreateESP, Player)
    end
end

Connect(Players.PlayerAdded, function(Player)
    Connect(Player.CharacterAdded, function()
        task.wait(0.3)
        CreateESP(Player)
    end)
end)

Connect(Players.PlayerRemoving, RemoveESP)

task.spawn(function()
    while State.Alive do
        local showChams = Options.ESPChams.Value
        local showName = Options.ESPName.Value
        local showRank = Options.ESPRank.Value
        local showHealth = Options.ESPHealth.Value

        for Player, Data in pairs(State.ESP) do
            if not Player.Parent or not Data.Character.Parent then
                RemoveESP(Player)
            else
                Data.Highlight.Enabled = showChams

                if showName or showRank or showHealth then
                    local Lines = {}
                    if showName then
                        table.insert(Lines, Player.DisplayName .. " (@" .. Player.Name .. ")")
                    end
                    if showRank then
                        table.insert(Lines, "ยศ: " .. Data.Rank)
                    end
                    if showHealth then
                        local HP = math.clamp(tonumber(Data.Humanoid.Health) or 0, 0, 999999999)
                        local MaxHP = math.clamp(tonumber(Data.Humanoid.MaxHealth) or 0, 0, 999999999)
                        table.insert(Lines, string.format("HP: %d/%d", math.floor(HP), math.floor(MaxHP)))
                    end

                    Data.Label.Text = table.concat(Lines, "\n")
                    Data.Gui.Enabled = true
                else
                    Data.Gui.Enabled = false
                end
            end
        end
        task.wait(0.3)
    end
end)

--// PLAYER TAB
Tabs.Player:AddSection("Movement")

local FlyToggle = Tabs.Player:AddToggle("Fly", { Title = "Fly", Default = false })
Tabs.Player:AddSlider("FlySpeed", { Title = "Fly Speed", Min = 10, Max = 300, Default = 60, Rounding = 0 })

local NoclipToggle = Tabs.Player:AddToggle("Noclip", { Title = "Noclip", Default = false })
Tabs.Player:AddToggle("TPWalk", { Title = "TP Walk", Default = false })
Tabs.Player:AddSlider("TPWalkSpeed", { Title = "TP Walk Speed", Min = 10, Max = 300, Default = 60, Rounding = 0 })

local WalkSpeedSlider = Tabs.Player:AddSlider("WalkSpeed", { Title = "WalkSpeed", Min = 0, Max = 250, Default = 16, Rounding = 0 })

local function StopFly()
    if State.FlyVelocity then State.FlyVelocity:Destroy() State.FlyVelocity = nil end
    if State.FlyGyro then State.FlyGyro:Destroy() State.FlyGyro = nil end
    local Hum = Humanoid()
    if Hum then
        Hum.PlatformStand = false
        Hum.AutoRotate = true
    end
end

local function StartFly()
    StopFly()
    local HRP = Root()
    local Hum = Humanoid()
    if not HRP or not Hum then return end

    local Velocity = Instance.new("BodyVelocity")
    Velocity.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
    Velocity.Velocity = Vector3.zero
    Velocity.Parent = HRP

    local Gyro = Instance.new("BodyGyro")
    Gyro.MaxTorque = Vector3.new(math.huge, math.huge, math.huge)
    Gyro.P = 30000
    Gyro.D = 500
    Gyro.Parent = HRP

    State.FlyVelocity = Velocity
    State.FlyGyro = Gyro
    Hum.PlatformStand = true
    Hum.AutoRotate = false
end

FlyToggle:OnChanged(function()
    if Options.Fly.Value then StartFly() else StopFly() end
end)

Connect(RunService.RenderStepped, function(Delta)
    if Options.Fly.Value then
        local HRP = Root()
        local Hum = Humanoid()
        if HRP and Hum then
            if not State.FlyVelocity or not State.FlyVelocity.Parent then StartFly() end

            if State.FlyVelocity then
                local Vertical = 0
                if UserInputService:IsKeyDown(Enum.KeyCode.Space) then Vertical += 1 end
                if UserInputService:IsKeyDown(Enum.KeyCode.LeftControl) then Vertical -= 1 end

                State.FlyVelocity.Velocity = Hum.MoveDirection * Options.FlySpeed.Value + Vector3.new(0, Vertical * Options.FlySpeed.Value, 0)
            end

            if State.FlyGyro then
                local Look = Camera.CFrame.LookVector
                local Flat = Vector3.new(Look.X, 0, Look.Z)
                if Flat.Magnitude > 0.01 then
                    State.FlyGyro.CFrame = CFrame.lookAt(HRP.Position, HRP.Position + Flat.Unit)
                end
            end
        end
    end

    if Options.TPWalk.Value then
        local HRP = Root()
        local Hum = Humanoid()
        if HRP and Hum then
            HRP.CFrame += Hum.MoveDirection * Options.TPWalkSpeed.Value * Delta
        end
    end
end)

local function RestoreNoclip()
    for Part, Value in pairs(State.NoclipParts) do
        if Part and Part.Parent then
            if State.UndergroundActive and State.UndergroundParts[Part] ~= nil then
                Part.CanCollide = false
            else
                Part.CanCollide = Value
            end
        end
    end

    table.clear(State.NoclipParts)
end

NoclipToggle:OnChanged(function()
    if not Options.Noclip.Value then RestoreNoclip() end
end)

Connect(RunService.Stepped, function()
    if not Options.Noclip.Value then return end
    for _, Part in ipairs(State.CharParts) do
        if Part and Part.Parent then
            if State.NoclipParts[Part] == nil then
                State.NoclipParts[Part] =
                    State.UndergroundParts[Part] ~= nil
                    and State.UndergroundParts[Part]
                    or Part.CanCollide
            end

            Part.CanCollide = false
        end
    end
end)

WalkSpeedSlider:OnChanged(function(Value)
    State.WalkSpeed = Value
    local Hum = Humanoid()
    if Hum then Hum.WalkSpeed = Value end
end)

--// TELEPORT TAB
Tabs.Teleport:AddSection("Teleport")

Tabs.Teleport:AddButton({
    Title = "Craft",
    Description = "-2127, -65, 3238",
    Callback = function()
        Teleport(CFrame.new(-2127, -65, 3238))
    end
})

Tabs.Teleport:AddButton({
    Title = "หน้าค่าย",
    Description = "-2111, -71, 2753",
    Callback = function()
        Teleport(CFrame.new(-2111, -71, 2753))
    end
})

Tabs.Teleport:AddButton({
    Title = "ค่ายทหาร",
    Description = "-2682, -43, 3344",
    Callback = function()
        Teleport(CFrame.new(-2682, -43, 3344))
    end
})

Tabs.Teleport:AddButton({
    Title = "Fishing Zone",
    Description = "ไปจุดตกปลา",
    Callback = function()
        Teleport(GetFishingZoneCFrame())
    end
})

Tabs.Teleport:AddButton({
    Title = "Fishing Shop",
    Description = "-1627, -62, 3274",
    Callback = function()
        Teleport(FishingShopPosition)
    end
})

--// MISC TAB - CAMERA
Tabs.Misc:AddSection("Camera")

local NoGunZoomToggle = Tabs.Misc:AddToggle("NoGunZoom", {
    Title = "No Gun Zoom",
    Description = "ถือปืนแล้วไม่บังคับซูม/ลด FOV",
    Default = false
})

NoGunZoomToggle:OnChanged(function()
    if Options.NoGunZoom.Value then
        ApplyNoGunZoom()
    else
        if GetEquippedGun() then
            LocalPlayer.CameraMaxZoomDistance = 0.5
        else
            LocalPlayer.CameraMaxZoomDistance = DEFAULT_MAX_ZOOM
        end
    end
end)

RunService:BindToRenderStep(
    NO_ZOOM_BIND,
    Enum.RenderPriority.Camera.Value + 10,
    ApplyNoGunZoom
)

--// MISC TAB - AUTO DRESS
Tabs.Misc:AddSection("Auto Dress")

local DRESS_TP_WAIT = 0.08
local DRESS_STEP_WAIT = 0.12

local function DressTP(Target, Offset)
    local CF = GetInstanceCFrame(Target)
    if not CF then return false end

    Teleport(CF * (Offset or CFrame.new(0, 1.5, 0)))
    task.wait(DRESS_TP_WAIT)
    return true
end

local function FastPrompt(Prompt)
    if not Prompt then return false end

    if fireproximityprompt then
        local OK = pcall(function()
            fireproximityprompt(Prompt)
        end)

        if OK then
            return true
        end
    end

    return false
end

local function RealEPrompt(Prompt)
    if not Prompt then return false end

    -- Prompt ตัว Red KG ต้องเข้าใกล้แล้วกด E จริง
    local Hold = math.max(
        0.28,
        tonumber(Prompt.HoldDuration) or 0
    )

    local OK = pcall(function()
        VirtualInputManager:SendKeyEvent(
            true,
            Enum.KeyCode.E,
            false,
            game
        )

        task.wait(Hold + 0.03)

        VirtualInputManager:SendKeyEvent(
            false,
            Enum.KeyCode.E,
            false,
            game
        )
    end)

    return OK
end

Tabs.Misc:AddButton({
    Title = "Auto Dress",
    Description = "ใส่ชุดทั้งหมดแบบเร็ว + Red KG ใช้ E จริง",
    Callback = function()
        task.spawn(function()
            local Maps = Workspace:FindFirstChild("Maps")
            local TargetMap =
                Maps and
                Maps:FindFirstChild(
                    "\224\184\149\224\184\182\224\184\129\224\184\129\224\184\173\224\184\135\224\184\151\224\184\177\224\184\158\224\184\154\224\184\129"
                )

            local Base =
                TargetMap and
                TargetMap:FindFirstChild("Model") and
                TargetMap.Model:FindFirstChild("Model")

            if not Base then
                Notify(
                    "Auto Dress",
                    "ไม่พบโฟลเดอร์แผนที่แต่งตัว"
                )
                return
            end

            -- 1. ArmBand01
            pcall(function()
                local C5 = Base:GetChildren()[5]
                local Arm =
                    C5 and
                    C5.ArmBand01[" Armband"].Arm1["Right Arm"]

                if DressTP(Arm) then
                    FastPrompt(
                        Arm:FindFirstChildOfClass("ProximityPrompt") or
                        Arm:FindFirstChild("ProximityPrompt")
                    )
                end
            end)

            task.wait(DRESS_STEP_WAIT)

            -- 2. Red KG
            -- fireproximityprompt() ตัวนี้ไม่ผ่าน แต่ E จริงผ่าน
            pcall(function()
                local C7 = Base:GetChildren()[7]
                local RedKG =
                    C7 and
                    C7:FindFirstChild("Model") and
                    C7.Model:FindFirstChild("Red KG")

                local Prompt =
                    RedKG and
                    (
                        RedKG:FindFirstChildOfClass("ProximityPrompt") or
                        RedKG:FindFirstChild("ProximityPrompt")
                    )

                if
                    RedKG and
                    DressTP(
                        RedKG,
                        CFrame.new(0, 0, -2)
                    ) then

                    RealEPrompt(Prompt)
                end
            end)

            task.wait(DRESS_STEP_WAIT)

            -- 3. Hat
            pcall(function()
                local C33 = Base:GetChildren()[33]
                local Hat =
                    C33 and
                    C33:GetChildren()[4] and
                    C33:GetChildren()[4]:FindFirstChild(
                        "Hat For Sergeant to First Sergeant"
                    )

                if DressTP(Hat) then
                    local Giver =
                        C33 and
                        C33:GetChildren()[3] and
                        C33:GetChildren()[3]:FindFirstChild("Giver")

                    FastPrompt(
                        Giver and
                        (
                            Giver:FindFirstChildOfClass("ProximityPrompt") or
                            Giver:FindFirstChild("ProximityPrompt")
                        )
                    )
                end
            end)

            task.wait(DRESS_STEP_WAIT)

            -- 4. Webbing2
            pcall(function()
                local Webbing = Base:FindFirstChild("Webbing2")

                if DressTP(Webbing) then
                    local CD =
                        Webbing and
                        (
                            Webbing:FindFirstChildOfClass("ClickDetector") or
                            Webbing:FindFirstChild("ClickDetector")
                        )

                    if CD and fireclickdetector then
                        fireclickdetector(CD)
                    end
                end
            end)

            task.wait(DRESS_STEP_WAIT)

            -- 5. Model 79
            pcall(function()
                local Item79 = Base:GetChildren()[79]

                if DressTP(Item79) then
                    FastPrompt(
                        Item79 and
                        (
                            Item79:FindFirstChildOfClass("ProximityPrompt") or
                            Item79:FindFirstChild("ProximityPrompt")
                        )
                    )
                end
            end)

            task.wait(DRESS_STEP_WAIT)

            -- 6. Headset
            pcall(function()
                local HeadsetModel =
                    Base:FindFirstChild(
                        "\224\184\156\224\184\161\224\184\156\224\184\185\224\185\137\224\184\138\224\184\178\224\184\162"
                    )

                local Headset =
                    HeadsetModel and
                    HeadsetModel:FindFirstChild("Headset")

                if DressTP(Headset) then
                    FastPrompt(
                        Headset and
                        (
                            Headset:FindFirstChildOfClass("ProximityPrompt") or
                            Headset:FindFirstChild("ProximityPrompt")
                        )
                    )
                end
            end)

            task.wait(0.1)

            -- กลับจุดเดิมของ Auto Dress
            Teleport(CFrame.new(-2716, -43, 3362))
            Notify("Auto Dress", "Done!")
        end)
    end
})

--// MISC TAB - PLAYER MANAGER
Tabs.Misc:AddSection("Player Manager")

local SelectedPlayer

local function GetPlayerNames()
    local Names = {}
    for _, Player in ipairs(Players:GetPlayers()) do
        if Player ~= LocalPlayer then
            table.insert(Names, Player.Name)
        end
    end
    table.sort(Names, function(A, B) return string.lower(A) < string.lower(B) end)
    return Names
end

local PlayerDropdown = Tabs.Misc:AddDropdown("SelectedPlayer", {
    Title = "Select Player",
    Values = GetPlayerNames(),
    Multi = false,
    Searchable = true
})

PlayerDropdown:OnChanged(function(Value)
    SelectedPlayer = Value
end)

local function RefreshPlayers()
    PlayerDropdown:SetValues(GetPlayerNames())
    if SelectedPlayer and not Players:FindFirstChild(SelectedPlayer) then
        SelectedPlayer = nil
    end
end

Connect(Players.PlayerAdded, function()
    task.wait(0.3)
    RefreshPlayers()
end)

Connect(Players.PlayerRemoving, function()
    task.wait(0.3)
    RefreshPlayers()
end)

Tabs.Misc:AddButton({
    Title = "Refresh Players",
    Callback = function()
        RefreshPlayers()
        Notify("Player List", "Updated")
    end
})

local function GetActionDamage(Hum, Mode)
    local Health = math.max(0, tonumber(Hum.Health) or 0)
    local MaxHealth = math.max(1, tonumber(Hum.MaxHealth) or 100)

    if Mode == "Kill" then
        return math.max(1000, Health + MaxHealth + 100)
    end

    local Missing = math.max(1, MaxHealth - math.min(Health, MaxHealth))
    return -Missing
end

local function FireTarget(Mode)
    if not ShootEvent then
        Notify("Script Status", "ShootEvent not found")
        return
    end

    if not SelectedPlayer then
        Notify("Target", "กรุณาเลือกผู้เล่นก่อน")
        return
    end

    local Target = Players:FindFirstChild(SelectedPlayer)
    local Hum = Target and Target.Character and Target.Character:FindFirstChildOfClass("Humanoid")

    if not Hum then
        Notify("Target", "ไม่พบตัวละครหรือ Humanoid")
        return
    end

    if Mode == "Heal" and Hum.Health >= Hum.MaxHealth then
        Notify("Target Action", Target.Name .. " เลือดเต็มแล้ว")
        return
    end

    local Success = pcall(function()
        ShootEvent:FireServer(
            Hum,
            GetActionDamage(Hum, Mode),
            "Left Arm",
            {"nil", "Auth", "nil", "nil"}
        )
    end)

    Notify(
        "Target Action",
        Success and (Mode .. " " .. Target.Name .. " สำเร็จ") or "เกิดข้อผิดพลาด"
    )
end

Tabs.Misc:AddButton({
    Title = "Kill Target",
    Description = "ทำ Damage ใส่ผู้เล่นที่เลือก",
    Callback = function()
        FireTarget("Kill")
    end
})

Tabs.Misc:AddButton({
    Title = "Heal Target",
    Description = "Heal ผู้เล่นที่เลือกจนเต็ม",
    Callback = function()
        FireTarget("Heal")
    end
})

Tabs.Misc:AddButton({
    Title = "Spectate",
    Callback = function()
        local Target = SelectedPlayer and Players:FindFirstChild(SelectedPlayer)
        local Hum = Target and Target.Character and Target.Character:FindFirstChildOfClass("Humanoid")
        if Hum then
            Camera.CameraSubject = Hum
            Notify("Spectate", Target.Name)
        end
    end
})

Tabs.Misc:AddButton({
    Title = "Stop Spectate",
    Callback = function()
        local Hum = Humanoid()
        if Hum then Camera.CameraSubject = Hum end
    end
})

--// MISC TAB - SERVER
Tabs.Misc:AddSection("Server")

local function FireAll(Mode)
    if not ShootEvent then
        Notify("Script Status", "ShootEvent not found")
        return
    end

    if State.RemoteActionBusy then
        Notify("Script Status", "Remote action กำลังทำงาน")
        return
    end

    if os.clock() - State.LastRemoteAction < 2 then
        Notify("Script Status", "รอสักครู่ก่อนใช้ซ้ำ")
        return
    end

    State.RemoteActionBusy = true
    State.LastRemoteAction = os.clock()

    task.spawn(function()
        local Targets = {}

        for _, Target in ipairs(Players:GetPlayers()) do
            if Target ~= LocalPlayer then
                local Hum = Target.Character and Target.Character:FindFirstChildOfClass("Humanoid")

                if Hum and Hum.Parent then
                    if Mode ~= "Heal" or Hum.Health < Hum.MaxHealth then
                        table.insert(Targets, Hum)
                    end
                end
            end
        end

        local Count = 0

        for Index, Hum in ipairs(Targets) do
            if not State.Alive then break end

            if Hum and Hum.Parent then
                local Success = pcall(function()
                    ShootEvent:FireServer(
                        Hum,
                        GetActionDamage(Hum, Mode),
                        "Left Arm",
                        {"nil", "Auth", "nil", "nil"}
                    )
                end)

                if Success then
                    Count += 1
                end
            end

            if Index % 3 == 0 then
                task.wait(0.12)
            else
                task.wait(0.04)
            end
        end

        table.clear(Targets)
        task.wait(0.15)

        State.RemoteActionBusy = false
        Notify("Script Status", "Done! (" .. tostring(Count) .. " Players)")
    end)
end

Tabs.Misc:AddButton({
    Title = "Heal All",
    Description = "Heal ทุกคนแบบแบ่ง batch ลด lag",
    Callback = function()
        FireAll("Heal")
    end
})

Tabs.Misc:AddButton({
    Title = "Kill All",
    Description = "Damage ทุกคนแบบแบ่ง batch ลด lag",
    Callback = function()
        FireAll("Kill")
    end
})

Tabs.Misc:AddButton({
    Title = "Immune Self",
    Description = "เพิ่มเลือดจำนวนมากให้ LocalPlayer",
    Callback = function()
        if not ShootEvent then
            Notify("Immune Self", "ShootEvent not found")
            return
        end

        local Hum = Humanoid()
        if not Hum then return end

        local Amount = -math.max(100000, (tonumber(Hum.MaxHealth) or 100) * 1000)

        local Success = pcall(function()
            ShootEvent:FireServer(
                Hum,
                Amount,
                "Left Arm",
                {"nil", "Auth", "nil", "nil"}
            )
        end)

        Notify("Immune Self", Success and "Done!" or "Failed")
    end
})

--// TP TOOL
Tabs.Misc:AddSection("Teleport Tool")

Tabs.Misc:AddButton({
    Title = "Get TP Tool",
    Callback = function()
        local Backpack = LocalPlayer:FindFirstChildOfClass("Backpack")
        if not Backpack then return end

        local Old = Backpack:FindFirstChild("TP Tool")
        if Old then Old:Destroy() end

        local Tool = Instance.new("Tool")
        Tool.Name = "TP Tool"
        Tool.RequiresHandle = false
        Tool.CanBeDropped = false
        Tool.Parent = Backpack

        State.TPTool = Tool
        local Mouse = LocalPlayer:GetMouse()

        Tool.Activated:Connect(function()
            if Mouse.Hit then
                Teleport(CFrame.new(Mouse.Hit.Position + Vector3.new(0, 3, 0)))
            end
        end)

        Notify("TP Tool", "Done!")
    end
})

--// Respawn & Character Cache
local function OnCharacterAdded(Char)
    State.CurrentFarm = nil
    State.EnteredFarm = false
    State.Target = nil
    State.Selling = false
    State.GameFarmRunning = false
    State.BuyingFarmTool = false
    State.SafeActive = false
    State.SafeThreatCount = 0

    State.FishingMode = "idle"
    State.FishingAutoRunning = false
    State.FishingAutoToken += 1
    SetFishingHold(false)

    StopUndergroundFarm()

    task.wait(0.5)
    UpdateCharParts()

    local Hum = Humanoid()
    if Hum then
        Hum.WalkSpeed = State.WalkSpeed
        Hum:GetPropertyChangedSignal("WalkSpeed"):Connect(function()
            if Hum.WalkSpeed ~= State.WalkSpeed then
                Hum.WalkSpeed = State.WalkSpeed
            end
        end)
    end

    if Options.Fly.Value then StartFly() end
    if Options.NoGunZoom.Value then task.defer(ApplyNoGunZoom) end

    if Options.AutoFishing and Options.AutoFishing.Value then
        task.delay(0.8, StartAutoFishing)
    end
end

if LocalPlayer.Character then
    UpdateCharParts()
end

Connect(LocalPlayer.CharacterAdded, OnCharacterAdded)

--// Cleanup
local function Cleanup()
    if not State.Alive then return end
    State.Alive = false

    if AutoFarmAction then
        pcall(function()
            AutoFarmAction:FireServer("stop")
        end)
    end

    State.FishingAutoToken += 1
    StopFishingVisualLock()
    SetFishingHold(false)

    if FishingAction then
        pcall(function()
            FishingAction:FireServer("autoOff")
            FishingAction:FireServer("cancel")
        end)
    end

    StopUndergroundFarm()
    StopMoving()
    StopFly()
    RestoreNoclip()

    pcall(function()
        RunService:UnbindFromRenderStep(NO_ZOOM_BIND)
    end)

    LocalPlayer.CameraMaxZoomDistance = DEFAULT_MAX_ZOOM

    for Player in pairs(State.ESP) do
        RemoveESP(Player)
    end

    if State.TPTool and State.TPTool.Parent then
        State.TPTool:Destroy()
    end

    for _, Connection in ipairs(Connections) do
        pcall(function() Connection:Disconnect() end)
    end
    table.clear(Connections)

    pcall(function() Fluent:Destroy() end)
end

getgenv().RCAHubUnload = Cleanup

--// Settings & Init
Tabs.Settings:AddButton({ Title = "Unload Script", Callback = Cleanup })

SaveManager:SetLibrary(Fluent)
InterfaceManager:SetLibrary(Fluent)
SaveManager:IgnoreThemeSettings()
SaveManager:SetIgnoreIndexes({})

InterfaceManager:SetFolder("RCA")
SaveManager:SetFolder("RCA/" .. tostring(game.PlaceId))

InterfaceManager:BuildInterfaceSection(Tabs.Settings)
SaveManager:BuildConfigSection(Tabs.Settings)

Window:SelectTab(1)
SaveManager:LoadAutoloadConfig()

Notify("จำลองชีวิตทหารไทย [RCA]", "made by BatmanScript")
