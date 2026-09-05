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
local ProximityPromptService = game:GetService("ProximityPromptService")
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
local FoodConfig

pcall(function()
    CraftConfig = require(ReplicatedStorage:WaitForChild("CraftConfig"))
end)

pcall(function()
    LeaderboardConfig = require(ReplicatedStorage:WaitForChild("LeaderboardConfig"))
end)

pcall(function()
    FishingConfig = require(ReplicatedStorage:WaitForChild("FishingConfig"))
end)

pcall(function()
    FoodConfig = require(ReplicatedStorage:WaitForChild("FoodConfig"))
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

--// Supermarket
local SuperMarketEvent = ReplicatedStorage:FindFirstChild("SuperMarketEvent")
local SuperMarketConfirm =
    SuperMarketEvent and
    SuperMarketEvent:FindFirstChild("ConfirmEvent")

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
local FishingFarmPosition = CFrame.new(-1514, -75, 2822)

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
    FishingSellingParts = false,
    FishingBuying = false,
    FishingBuyingRod = false,
    FishingBaitRequestQueued = false,
    FishingBaitRetryToken = 0,
    FishingBaitConnection = nil,
    FishingLastBaitAmount = nil,
    FishingShopPromptCache = nil,
    FishingSelling = false,
    FishingHolding = false,
    FishingAutoToken = 0,
    FishingPrevI = nil,
    FishingPrevG = nil,
    FishingPrevTick = nil,
    FishingIVelocity = 0,
    FishingGVelocity = 0,
    FishingArrow = 0,
    FishingArrowChanged = 0,
    FishingTickEMA = 0.05,
    FishingPendingHold = nil,
    FishingPendingTicks = 0,
    FishingLastHoldChange = 0,
    FishingVisualWriting = false,
    FishingVisualBind = false,
    FishingIndicatorConn = nil,
    FishingGreenConn = nil,
    FishingVisibleConn = nil,
    FishingAssistOverlay = nil,
    FishingStaticGreen = nil,
    FishingOriginalIndicatorVisible = nil,
    FishingOriginalGreenVisible = nil,
    FishingLowFXApplied = false,
    FishingFXTable = nil,
    FishingUITable = nil,
    FishingFXOriginal = nil,
    FishingUIOriginal = nil,
    FishingConfigOriginal = nil,
    ApplyFishingLowFX = nil,
    RestoreFishingLowFX = nil,
    FishingOriginalMinigamePosition = nil,
    FishingOriginalMinigameAnchorPoint = nil,
    LastFishingStart = 0,
    LastFishingAutoOn = 0,
    LastBaitBuy = 0,
    LastFishingRodBuy = 0,
    LastFishingRodNotify = 0,

    CraftBusy = false,
    LastCraft = 0,
    LastSell = 0,

    WalkSpeed = 16,

    ESP = {},
    CharParts = {},
    NoclipParts = {},
    ESPLoopToken = 0,
    ESPPlayerAddedConnection = nil,
    ESPPlayerRemovingConnection = nil,

    FlyVelocity = nil,
    FlyGyro = nil,
    MovementConnection = nil,
    NoclipConnection = nil,
    UndergroundConnection = nil,
    AutoFarmLoopToken = 0,
    FishingLoopToken = 0,
    CraftLoopToken = 0,

    AntiVoidFolder = nil,
    AntiVoidParts = {},
    AntiVoidConnection = nil,
    AntiVoidLastSafe = nil,
    AntiVoidLastSafeTick = 0,
    AntiVoidLastRescue = 0,

    DeleteMapBusy = false,
    DeleteMapToken = 0,
    DeleteMapCache = nil,
    DeletedMapItems = {},

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
    -- มี Tool อยู่แล้วก็ใช้ต่อทันที (รองรับทั้งตัวปกติและ (P))
    local Tool = GetFarmTool(Farm)
    if Tool then
        return Tool
    end

    if State.BuyingFarmTool then
        return nil
    end

    local BuyName = Farm.BuyTool or Farm.Tools[1]
    if not BuyName then
        return nil
    end

    -- หา Remote ใหม่ทุกครั้ง เผื่อยังไม่ replicate ตอนรัน script
    local Remote =
        ReplicatedStorage:FindFirstChild("Buy") or
        BuyRemote

    if not Remote or not Remote:IsA("RemoteFunction") then
        if os.clock() - State.LastFarmToolNotify >= 4 then
            State.LastFarmToolNotify = os.clock()
            Notify("Auto Farm", "ไม่พบ Remote ซื้อของ ReplicatedStorage.Buy")
        end
        return nil
    end

    BuyRemote = Remote

    -- กันยิงซื้อถี่เกินไป
    if os.clock() - State.LastFarmToolBuy < 1.25 then
        return nil
    end

    State.BuyingFarmTool = true
    State.LastFarmToolBuy = os.clock()

    -- สำคัญ: Remote ของเกมอาจซื้อสำเร็จแต่ return nil
    -- จึงตัดสินผลจาก Tool ที่ replicate เข้ามาจริง ไม่ใช่ Result
    local Success, Result = pcall(function()
        return Remote:InvokeServer(BuyName)
    end)

    if not Success then
        State.BuyingFarmTool = false

        if os.clock() - State.LastFarmToolNotify >= 4 then
            State.LastFarmToolNotify = os.clock()
            Notify(
                "Auto Farm",
                "เรียกซื้อ " .. tostring(BuyName) .. " ไม่สำเร็จ"
            )
        end

        return nil
    end

    -- ต่อให้ Result=nil ก็ยังรอ Tool เพราะ server อาจไม่ return ค่า
    local Deadline = os.clock() + 4

    repeat
        task.wait(0.08)
        Tool = GetFarmTool(Farm)
    until Tool or os.clock() >= Deadline

    State.BuyingFarmTool = false

    if Tool then
        Notify(
            "Auto Farm",
            "ซื้อ " .. tostring(BuyName) .. " สำเร็จ → เริ่มฟาร์ม"
        )
        return Tool
    end

    -- ถ้า server ตอบ table ที่มีข้อความ ให้เอามาแจ้งด้วย
    local Message = ""
    if type(Result) == "table" then
        Message = tostring(
            Result.message or
            Result.reason or
            Result.error or
            ""
        )
    end

    if os.clock() - State.LastFarmToolNotify >= 4 then
        State.LastFarmToolNotify = os.clock()
        Notify(
            "Auto Farm",
            "ยังไม่พบ " ..
            tostring(BuyName) ..
            " หลังสั่งซื้อ" ..
            (Message ~= "" and (" : " .. Message) or "")
        )
    end

    return nil
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

--// Auto Farm runtime - no loop at startup
local function StopUndergroundRuntime()
    if State.UndergroundConnection then
        pcall(function()
            State.UndergroundConnection:Disconnect()
        end)
        State.UndergroundConnection = nil
    end

    StopUndergroundFarm()
end

local function UpdateUndergroundRuntime()
    if
        Options.AutoFarm.Value and
        Options.UndergroundFarm.Value then

        if not State.UndergroundConnection then
            State.UndergroundConnection =
                RunService.Heartbeat:Connect(function()
                    if
                        not State.Alive or
                        not Options.AutoFarm.Value or
                        not Options.UndergroundFarm.Value then

                        return
                    end

                    if
                        not State.Selling and
                        not State.SafeActive then

                        StartUndergroundFarm()
                    elseif
                        State.UndergroundActive or
                        State.UndergroundForce then

                        StopUndergroundFarm()
                    end
                end)
        end
    else
        StopUndergroundRuntime()
    end
end

local function StartAutoFarmWatcher()
    State.AutoFarmLoopToken += 1
    local Token = State.AutoFarmLoopToken

    task.spawn(function()
        while
            State.Alive and
            Options.AutoFarm.Value and
            Token == State.AutoFarmLoopToken do

            if not State.Selling then
                local FarmId = Options.FarmType.Value or "Banana"
                local Farm = Farms[FarmId]

                if Farm then
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
                    elseif
                        not State.GameFarmRunning and
                        not State.BuyingFarmTool then

                        StartGameFarm()
                    end
                end
            end

            task.wait(0.3)
        end
    end)
end

UndergroundFarmToggle:OnChanged(function()
    UpdateUndergroundRuntime()
end)

AutoFarmToggle:OnChanged(function()
    State.AutoFarmLoopToken += 1
    UpdateUndergroundRuntime()

    if Options.AutoFarm.Value then
        StartAutoFarmWatcher()
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
    Description = "อ่านตำแหน่งจริงจาก server + คุม HoldInput อัตโนมัติ",
    Default = true
})

Tabs.Main:AddToggle("FishingLowFX", {
    Title = "Low FX Fishing",
    Description = "ปิดเอฟเฟกต์ตกปลาหนัก ๆ เพื่อลด FPS drop โดยเฉพาะ Legendary/Mythic",
    Default = true
})

Tabs.Main:AddToggle("FishingAutoTP", {
    Title = "Auto TP Fishing Zone",
    Description = "วาปไปจุดฟาร์ม -1514, -75, 2822 ก่อนเริ่มอัตโนมัติ",
    Default = true
})

Tabs.Main:AddToggle("FishingAutoBuyBait", {
    Title = "Auto Buy Bait",
    Description = "ซื้อแพ็กเหยื่อที่เลือกเมื่อเหยื่อเหลือน้อย",
    Default = false
})

Tabs.Main:AddToggle("FishingAutoSellFull", {
    Title = "Auto Sell Fish When Full",
    Description = "FishPart เต็ม 150/150 → ไปตลาดขาย → กลับมาตกต่อ",
    Default = true
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

local function GetFishingMaterialId()
    return
        FishingConfig and
        FishingConfig.AutoFarm and
        FishingConfig.AutoFarm.MaterialId or
        "FishPart"
end

local function GetFishingMaterialAmount()
    return GetAmount(GetFishingMaterialId())
end

local function GetFishingMaterialMax()
    local Id = GetFishingMaterialId()
    local Object = GetMaterialObject(Id)

    if Object then
        local Max = tonumber(Object:GetAttribute("MaxStack"))
        if Max and Max > 0 then
            return Max
        end
    end

    if CraftConfig and type(CraftConfig.Materials) == "table" then
        for _, Material in ipairs(CraftConfig.Materials) do
            if Material.Id == Id and tonumber(Material.MaxStack) then
                return tonumber(Material.MaxStack)
            end
        end
    end

    return 150
end

local function IsFishingMaterialFull()
    return GetFishingMaterialAmount() >= GetFishingMaterialMax()
end

local function SelectedFishingRod()
    return FishingRodMap[Options.FishingRod.Value]
end

local function GetSelectedRodToolName()
    local Rod = SelectedFishingRod()

    if Rod and Rod.ToolName then
        return tostring(Rod.ToolName)
    end

    return "Fishing Rod"
end

local function GetFishingRodTool()
    local ToolName = GetSelectedRodToolName()
    local Char = Character()
    local Backpack = LocalPlayer:FindFirstChild("Backpack")

    -- pathway ตรงตามของในเกม:
    -- LocalPlayer.Backpack["Fishing Rod"]
    -- LocalPlayer.Backpack["VIP Fishing Rod"]
    local Tool =
        Char and
        Char:FindFirstChild(ToolName)

    if Tool and Tool:IsA("Tool") then
        return Tool
    end

    Tool =
        Backpack and
        Backpack:FindFirstChild(ToolName)

    if Tool and Tool:IsA("Tool") then
        return Tool
    end

    return nil
end

local function IsFishingRodEquipped()
    local Char = Character()
    if not Char then return false end

    local ToolName = GetSelectedRodToolName()
    local Tool = Char:FindFirstChild(ToolName)

    return Tool and Tool:IsA("Tool") or false
end

local function EquipFishingRod()
    local Char = Character()
    local Hum = Humanoid()
    local Backpack = LocalPlayer:FindFirstChild("Backpack")

    if
        not Char or
        not Hum or
        Hum.Health <= 0 or
        not Backpack then

        return false
    end

    local ToolName = GetSelectedRodToolName()

    -- ถืออยู่แล้ว
    if IsFishingRodEquipped() then
        return true
    end

    -- รอ Tool ตรง pathway ใน Backpack หลังซื้อ
    local Tool =
        Backpack:FindFirstChild(ToolName)

    local Deadline = os.clock() + 3

    while
        not Tool and
        os.clock() < Deadline do

        task.wait(0.05)

        Backpack =
            LocalPlayer:FindFirstChild("Backpack")

        Tool =
            Backpack and
            Backpack:FindFirstChild(ToolName)

        if IsFishingRodEquipped() then
            return true
        end
    end

    if not Tool or not Tool:IsA("Tool") then
        return false
    end

    -- ปลดของที่ถืออยู่ก่อน
    pcall(function()
        Hum:UnequipTools()
    end)

    task.wait(0.08)

    -- รอบ 1: Roblox API ปกติ
    pcall(function()
        Hum:EquipTool(Tool)
    end)

    for _ = 1, 5 do
        task.wait(0.08)

        if IsFishingRodEquipped() then
            return true
        end

        Backpack =
            LocalPlayer:FindFirstChild("Backpack")

        Tool =
            Backpack and
            Backpack:FindFirstChild(ToolName)

        if Tool and Tool:IsA("Tool") then
            pcall(function()
                Hum:EquipTool(Tool)
            end)
        end
    end

    -- รอบ 2: pathway ตรง ๆ
    -- ถ้า EquipTool ไม่ย้าย Tool ให้ย้ายจาก Backpack -> Character
    Backpack =
        LocalPlayer:FindFirstChild("Backpack")

    Tool =
        Backpack and
        Backpack:FindFirstChild(ToolName)

    if Tool and Tool:IsA("Tool") then
        pcall(function()
            Tool.Parent = Char
        end)

        task.wait(0.12)

        if IsFishingRodEquipped() then
            return true
        end

        -- สั่ง Humanoid ซ้ำหลังย้ายเข้า Character
        pcall(function()
            Hum:EquipTool(Tool)
        end)

        task.wait(0.15)
    end

    return IsFishingRodEquipped()
end

local function GetFishingShopPrompt()
    local Cached = State.FishingShopPromptCache

    if Cached and Cached.Parent then
        return Cached
    end

    -- scan แค่ครั้งแรกที่ต้องซื้อคันเบ็ด แล้ว cache ไว้
    for _, Object in ipairs(Workspace:GetDescendants()) do
        if
            Object:IsA("ProximityPrompt") and
            Object.Name == "ShopPrompt" then

            State.FishingShopPromptCache = Object
            return Object
        end
    end

    return nil
end

local function GetPromptCFrame(Prompt)
    if not Prompt then return nil end

    local Parent = Prompt.Parent

    if Parent and Parent:IsA("BasePart") then
        return Parent.CFrame
    end

    if Parent and Parent:IsA("Attachment") then
        local Part = Parent.Parent
        if Part and Part:IsA("BasePart") then
            return Part.CFrame
        end
    end

    if Parent and Parent:IsA("Model") then
        return Parent:GetPivot()
    end

    local Part =
        Parent and
        Parent:FindFirstChildWhichIsA("BasePart", true)

    return Part and Part.CFrame or nil
end

local function EnsureFishingRod()
    local Tool = GetFishingRodTool()
    if Tool then
        return EquipFishingRod()
    end

    if State.FishingBuyingRod then
        return false
    end

    local Rod = SelectedFishingRod()
    if not Rod then
        return false
    end

    local Remote =
        FishingRemotes and
        FishingRemotes:FindFirstChild("ShopAction")

    if
        not Remote or
        not Remote:IsA("RemoteFunction") then

        if os.clock() - State.LastFishingRodNotify >= 4 then
            State.LastFishingRodNotify = os.clock()
            Notify("Auto Fishing", "ไม่พบ Fishing ShopAction")
        end

        return false
    end

    FishingShopAction = Remote

    -- กัน Auto Fishing ยิงซื้อซ้ำทุก watcher tick
    if os.clock() - State.LastFishingRodBuy < 2 then
        return false
    end

    State.FishingBuyingRod = true
    State.LastFishingRodBuy = os.clock()

    local HRP = Root()
    local ReturnCF = HRP and HRP.CFrame or nil

    -- ไปหน้าร้านจริงก่อน เผื่อ server เช็กระยะ
    local Prompt = GetFishingShopPrompt()
    local ShopCF = GetPromptCFrame(Prompt)

    if ShopCF then
        Teleport(ShopCF * CFrame.new(0, 0, -2.5))
    else
        Teleport(FishingShopPosition)
    end

    task.wait(0.15)

    -- Trigger prompt ให้ flow ใกล้กับการซื้อจาก UI จริง
    if Prompt and fireproximityprompt then
        pcall(function()
            fireproximityprompt(Prompt)
        end)
        task.wait(0.12)
    end

    local Success, Result = pcall(function()
        return Remote:InvokeServer("buyRod", Rod.Id)
    end)

    -- อย่าตัดสินจาก Result=nil อย่างเดียว
    -- รอ Tool จริงเข้า Backpack/Character
    local Deadline = os.clock() + 4

    repeat
        task.wait(0.08)
        Tool = GetFishingRodTool()
    until Tool or os.clock() >= Deadline

    State.FishingBuyingRod = false

    if Tool then
        -- รอให้คันเบ็ดอยู่ใน Backpack/Character จริงก่อน
        local ToolName = GetSelectedRodToolName()
        local WaitUntil = os.clock() + 2

        repeat
            local Backpack = LocalPlayer:FindFirstChild("Backpack")
            Tool =
                (Character() and Character():FindFirstChild(ToolName)) or
                (Backpack and Backpack:FindFirstChild(ToolName))

            if Tool then break end
            task.wait(0.05)
        until os.clock() >= WaitUntil

        -- กลับจุดฟาร์มก่อน แล้วบังคับถือจาก pathway ตรง
        Teleport(FishingFarmPosition)
        task.wait(0.2)

        if EquipFishingRod() then
            Notify(
                "Auto Fishing",
                "ซื้อ " ..
                tostring(Rod.ToolName or Rod.DisplayName or Rod.Id) ..
                " สำเร็จ → ถือคันเบ็ดแล้ว"
            )

            return true
        end
    end

    -- ถ้าซื้อไม่สำเร็จ กลับจุดเดิม/จุดฟาร์ม
    if ReturnCF then
        Teleport(ReturnCF)
    else
        Teleport(FishingFarmPosition)
    end

    if os.clock() - State.LastFishingRodNotify >= 4 then
        State.LastFishingRodNotify = os.clock()

        local Message = ""

        if type(Result) == "table" then
            Message = tostring(
                Result.message or
                Result.reason or
                Result.error or
                ""
            )
        end

        if not Success then
            Message =
                Message ~= "" and Message or
                "Remote error"
        end

        Notify(
            "Auto Fishing",
            "ซื้อ " ..
            tostring(Rod.ToolName or Rod.DisplayName or Rod.Id) ..
            " ไม่สำเร็จ" ..
            (Message ~= "" and (" : " .. Message) or "")
        )
    end

    return false
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
    -- จุดฟาร์มตกปลาที่กำหนดเอง
    return FishingFarmPosition
end

local function GetFishingBaitObject()
    local Folder = LocalPlayer:FindFirstChild("Fishing")
    local Bait = Folder and Folder:FindFirstChild("Bait")

    if
        Bait and
        (
            Bait:IsA("IntValue") or
            Bait:IsA("NumberValue")
        ) then

        return Bait
    end

    return nil
end

local function BuyFishingBait(Silent)
    if State.FishingBuying then
        return false
    end

    local Remote =
        FishingRemotes and
        FishingRemotes:FindFirstChild("ShopAction")

    if
        not Remote or
        not Remote:IsA("RemoteFunction") then

        return false
    end

    FishingShopAction = Remote

    -- Manual/forced buy ก็ยังมี debounce กัน double click
    if os.clock() - State.LastBaitBuy < 1.5 then
        return false
    end

    local Amount =
        FishingBaitMap[
            Options.FishingBaitPack.Value
        ]

    if not Amount then
        return false
    end

    State.FishingBuying = true
    State.LastBaitBuy = os.clock()

    local Before = GetFishingBait()

    local Success, Result =
        pcall(function()
            return Remote:InvokeServer(
                "buyBait",
                Amount
            )
        end)

    -- ไม่รอ Result อย่างเดียว
    -- รอ Bait.Value replication จริง แต่สูงสุดสั้น ๆ
    local Changed = false
    local Deadline = os.clock() + 1.5

    if Success then
        repeat
            task.wait(0.05)

            if GetFishingBait() > Before then
                Changed = true
                break
            end
        until os.clock() >= Deadline
    end

    State.FishingBuying = false

    local OK =
        Changed or
        (
            Success and
            type(Result) == "table" and
            Result.ok == true
        )

    if not Silent then
        local Message

        if OK then
            Message =
                type(Result) == "table" and
                Result.message or
                (
                    "ซื้อเหยื่อ x" ..
                    tostring(Amount) ..
                    " สำเร็จ"
                )
        else
            Message =
                type(Result) == "table" and
                (
                    Result.message or
                    Result.reason
                ) or
                "ซื้อเหยื่อไม่สำเร็จ"
        end

        Notify(
            "Fishing Bait",
            tostring(Message)
        )
    end

    return OK
end

local StartAutoFishing
local QueueAutoBuyBait

QueueAutoBuyBait = function(RestartFishing, Force)
    if
        not State.Alive or
        not Options.FishingAutoBuyBait.Value or
        State.FishingBaitRequestQueued or
        State.FishingBuying then

        return
    end

    local Current = GetFishingBait()
    local Threshold =
        tonumber(
            Options.FishingBaitThreshold.Value
        ) or 5

    if not Force and Current > Threshold then
        return
    end

    -- Auto buy มี cooldown ยาวกว่า manual
    -- ป้องกัน RemoteFunction spam / lag spike
    if
        not Force and
        os.clock() - State.LastBaitBuy < 8 then

        return
    end

    State.FishingBaitRequestQueued = true
    State.FishingBaitRetryToken += 1

    local Token = State.FishingBaitRetryToken

    task.spawn(function()
        -- ให้ fishing state/event จบรอบปัจจุบันก่อน
        task.wait(0.08)

        if
            not State.Alive or
            Token ~= State.FishingBaitRetryToken or
            not Options.FishingAutoBuyBait.Value then

            State.FishingBaitRequestQueued = false
            return
        end

        local Before = GetFishingBait()
        local OK = BuyFishingBait(true)
        local After = GetFishingBait()

        State.FishingBaitRequestQueued = false

        if OK and After > Before then
            -- ถ้าเคยหยุดเพราะเหยื่อหมด เริ่มตกต่อ
            if
                RestartFishing and
                Options.AutoFishing.Value and
                After > 0 then

                task.delay(0.18, function()
                    if
                        State.Alive and
                        Options.AutoFishing.Value and
                        not State.FishingBuying then

                        StartAutoFishing()
                    end
                end)
            end

            return
        end

        -- ซื้อไม่ผ่าน: ไม่ยิง Remote ซ้ำทุก watcher tick
        -- รอ 10 วิแล้วค่อยลองใหม่ ถ้ายังต่ำจริง
        task.delay(10, function()
            if
                not State.Alive or
                Token ~= State.FishingBaitRetryToken or
                not Options.FishingAutoBuyBait.Value then

                return
            end

            local AmountNow = GetFishingBait()

            if
                AmountNow <=
                (
                    tonumber(
                        Options.FishingBaitThreshold.Value
                    ) or 5
                ) then

                QueueAutoBuyBait(
                    AmountNow <= 0,
                    false
                )
            end
        end)
    end)
end

local function DisconnectAutoBaitConnection()
    if State.FishingBaitConnection then
        pcall(function()
            State.FishingBaitConnection:Disconnect()
        end)

        State.FishingBaitConnection = nil
    end

    State.FishingBaitRetryToken += 1
    State.FishingBaitRequestQueued = false
end

local function BindAutoBaitConnection()
    DisconnectAutoBaitConnection()

    if not Options.FishingAutoBuyBait.Value then
        return
    end

    local Bait = GetFishingBaitObject()

    if not Bait then
        return
    end

    State.FishingLastBaitAmount =
        tonumber(Bait.Value) or 0

    State.FishingBaitConnection =
        Bait:GetPropertyChangedSignal("Value"):
        Connect(function()

            local Current =
                tonumber(Bait.Value) or 0

            local Previous =
                State.FishingLastBaitAmount

            State.FishingLastBaitAmount =
                Current

            if
                not Options.FishingAutoBuyBait.Value or
                not Options.AutoFishing.Value then

                return
            end

            local Threshold =
                tonumber(
                    Options.FishingBaitThreshold.Value
                ) or 5

            -- Trigger เฉพาะตอนจำนวนลดลงจนถึง threshold
            -- ไม่ polling Remote ซ้ำ ๆ
            if
                Current <= Threshold and
                (
                    Previous == nil or
                    Previous > Threshold or
                    Current <= 0
                ) then

                QueueAutoBuyBait(
                    Current <= 0,
                    false
                )
            end
        end)

    -- เปิด toggle ตอนเหยื่อต่ำอยู่แล้ว ให้ซื้อ 1 รอบ
    if
        State.FishingLastBaitAmount <=
        (
            tonumber(
                Options.FishingBaitThreshold.Value
            ) or 5
        ) then

        QueueAutoBuyBait(
            State.FishingLastBaitAmount <= 0,
            false
        )
    end
end

Options.FishingAutoBuyBait:OnChanged(function()
    if Options.FishingAutoBuyBait.Value then
        BindAutoBaitConnection()
    else
        DisconnectAutoBaitConnection()
    end
end)

local function BuyFishingRod()
    local Rod = SelectedFishingRod()

    if not Rod then
        Notify("Fishing Rod", "ไม่ได้เลือกคันเบ็ด")
        return
    end

    if GetFishingRodTool() then
        Notify(
            "Fishing Rod",
            tostring(Rod.ToolName or Rod.DisplayName or Rod.Id) ..
            " มีอยู่แล้ว"
        )
        return
    end

    task.spawn(function()
        if not EnsureFishingRod() then
            -- EnsureFishingRod แจ้งเหตุผลเองแล้ว
            return
        end
    end)
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

local function SetFishingHold(Value, Force)
    Value = Value == true

    if State.FishingHolding == Value then
        return
    end

    local Now = os.clock()

    -- กันสลับ Remote true/false ถี่เกินจาก jitter
    -- Force ใช้เฉพาะตอนใกล้หลุดขอบจริง
    if
        not Force and
        Now - (State.FishingLastHoldChange or 0) < 0.045 then

        return
    end

    State.FishingHolding = Value
    State.FishingLastHoldChange = Now

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

local function GetFishingMinigameObjects()
    local PlayerGui = LocalPlayer:FindFirstChild("PlayerGui")
    local Gui = PlayerGui and PlayerGui:FindFirstChild("FishingGui")
    local RootGui = Gui and Gui:FindFirstChild("Root")
    local Minigame = RootGui and RootGui:FindFirstChild("Minigame")
    local Track = Minigame and Minigame:FindFirstChild("Track")
    local Indicator = Track and Track:FindFirstChild("Indicator")
    local GreenZone = Track and Track:FindFirstChild("GreenZone")

    return Indicator, GreenZone, Minigame, Track
end

local function ClearOldFishingVisualsOnce()
    local PlayerGui = LocalPlayer:FindFirstChild("PlayerGui")
    if not PlayerGui then return end

    -- FindFirstChild recursive แค่ชื่อที่รู้จัก
    -- ไม่ GetDescendants ทุกครั้งที่ bite
    for _, Name in ipairs({
        "RCA_StaticGreenZone",
        "RCA_StaticIndicator",
        "RCA_AssistIndicator"
    }) do
        while true do
            local Object =
                PlayerGui:FindFirstChild(
                    Name,
                    true
                )

            if not Object then
                break
            end

            pcall(function()
                Object:Destroy()
            end)
        end
    end

    local Indicator, GreenZone =
        GetFishingMinigameObjects()

    if Indicator then
        pcall(function()
            Indicator.Visible = true
        end)
    end

    if GreenZone then
        pcall(function()
            GreenZone.Visible = true
        end)
    end

    State.FishingAssistOverlay = nil
    State.FishingStaticGreen = nil

    pcall(function()
        RunService:UnbindFromRenderStep(
            "RCA_FishingInstantGreen"
        )
    end)
end

local function StartFishingVisualLock()
    -- ไม่แตะ GUI ระหว่างตกปลาเลย
    State.FishingVisualBind = true
end

local function StopFishingVisualLock()
    State.FishingVisualBind = false
end

task.defer(ClearOldFishingVisualsOnce)

do
    local function FindFishingClientModules()
        local PlayerScripts =
            LocalPlayer:FindFirstChild("PlayerScripts")

        if not PlayerScripts then
            return nil, nil
        end

        local Client =
            PlayerScripts:FindFirstChild(
                "FishingClient",
                true
            )

        if not Client then
            return nil, nil
        end

        local FXModule =
            Client:FindFirstChild("FishingFX")

        local UIModule =
            Client:FindFirstChild("FishingUI")

        local FX
        local UI

        if
            FXModule and
            FXModule:IsA("ModuleScript") then

            pcall(function()
                FX = require(FXModule)
            end)
        end

        if
            UIModule and
            UIModule:IsA("ModuleScript") then

            pcall(function()
                UI = require(UIModule)
            end)
        end

        return FX, UI
    end

    State.ApplyFishingLowFX = function()
        if State.FishingLowFXApplied then
            return true
        end

        if
            not FishingConfig or
            type(FishingConfig.Effects) ~= "table" then

            return false
        end

        local FX, UI =
            FindFishingClientModules()

        if
            type(FX) ~= "table" or
            type(UI) ~= "table" then

            return false
        end

        State.FishingFXTable = FX
        State.FishingUITable = UI

        if not State.FishingConfigOriginal then
            State.FishingConfigOriginal = {
                UseCinematic =
                    FishingConfig.UseCinematic,
                Effects = {}
            }

            for Key, Value in
                pairs(FishingConfig.Effects) do

                State.FishingConfigOriginal
                    .Effects[Key] = Value
            end
        end

        if not State.FishingFXOriginal then
            State.FishingFXOriginal = {
                BeginCameraEffects =
                    FX.BeginCameraEffects,
                BitePunch =
                    FX.BitePunch,
                PlayCinematic =
                    FX.PlayCinematic,
                SetReeling =
                    FX.SetReeling,
                ShowFish =
                    FX.ShowFish
            }
        end

        if not State.FishingUIOriginal then
            State.FishingUIOriginal = {
                UpdateMinigame =
                    UI.UpdateMinigame,
                FlashScreen =
                    UI.FlashScreen,
                ShowBiteAlert =
                    UI.ShowBiteAlert,
                ShowAnnounce =
                    UI.ShowAnnounce
            }
        end

        -- หยุด camera/reel effect ที่อาจกำลังค้างก่อน patch
        pcall(function()
            if FX.EndCameraEffects then
                FX.EndCameraEffects()
            end
        end)

        pcall(function()
            if FX.StopCinematic then
                FX.StopCinematic()
            end
        end)

        -- ปิด effect flags ที่ FishingUI/FishingFX อ่านทุก frame
        local Effects =
            FishingConfig.Effects

        Effects.BiteFlash = false
        Effects.BiteZoom = false
        Effects.EdgeGlow = false
        Effects.TensionShake = false
        Effects.TensionZoom = false
        Effects.Heartbeat = false
        Effects.ReelSound = false
        Effects.BarPunch = false
        Effects.GreenPulse = false

        FishingConfig.UseCinematic = false

        -- ตัด camera/VFX ที่ FishingClient เรียกตอน bite/catch
        FX.BeginCameraEffects =
            function() end

        FX.BitePunch =
            function() end

        FX.PlayCinematic =
            function() end

        FX.SetReeling =
            function() end

        -- Legendary/Mythic ตอนจับได้จะสร้าง ParticleEmitter +
        -- PointLight + RenderStepped หมุนปลา 3 วิ
        -- Auto Fishing ไม่จำเป็นต้อง render ฉากนี้
        FX.ShowFish =
            function() end

        -- ตัด flash/tween แจ้งเตือนที่ไม่จำเป็นกับ Auto Fishing
        UI.FlashScreen =
            function() end

        UI.ShowBiteAlert =
            function() end

        UI.ShowAnnounce =
            function() end

        local OriginalUpdate =
            State.FishingUIOriginal.UpdateMinigame

        if type(OriginalUpdate) == "function" then
            UI.UpdateMinigame =
                function(Data)

                    if type(Data) ~= "table" then
                        return OriginalUpdate(Data)
                    end

                    -- Controller ของเรายังได้ t/a จริงจาก FishingState
                    -- แต่ UI เกมไม่ต้อง render:
                    -- 1) bar shake ตอน t >= .75
                    -- 2) MoveArrow pulse ทุก frame
                    local RealT = Data.t
                    local RealA = Data.a

                    local T =
                        tonumber(RealT)

                    if T and T >= 0.75 then
                        Data.t = 0.70
                    end

                    Data.a = 0

                    local Result =
                        OriginalUpdate(Data)

                    Data.t = RealT
                    Data.a = RealA

                    return Result
                end
        end

        State.FishingLowFXApplied = true
        return true
    end

    State.RestoreFishingLowFX = function()
        if not State.FishingLowFXApplied then
            return
        end

        local FX =
            State.FishingFXTable

        local UI =
            State.FishingUITable

        local FXOriginal =
            State.FishingFXOriginal

        local UIOriginal =
            State.FishingUIOriginal

        if
            FX and
            FXOriginal then

            for Key, Value in
                pairs(FXOriginal) do

                FX[Key] = Value
            end
        end

        if
            UI and
            UIOriginal then

            for Key, Value in
                pairs(UIOriginal) do

                UI[Key] = Value
            end
        end

        if
            FishingConfig and
            State.FishingConfigOriginal then

            FishingConfig.UseCinematic =
                State.FishingConfigOriginal
                    .UseCinematic

            if
                type(FishingConfig.Effects) ==
                "table" then

                for Key, Value in
                    pairs(
                        State.FishingConfigOriginal
                            .Effects
                    ) do

                    FishingConfig.Effects[Key] =
                        Value
                end
            end
        end

        State.FishingLowFXApplied = false
    end

    local function TryApplyLowFX()
        if
            not State.Alive or
            not Options.FishingLowFX.Value then

            return
        end

        if State.ApplyFishingLowFX() then
            return
        end

        -- FishingClient บางเครื่องโหลดช้ากว่า Hub เล็กน้อย
        task.spawn(function()
            for _ = 1, 16 do
                if
                    not State.Alive or
                    not Options.FishingLowFX.Value then

                    return
                end

                if State.ApplyFishingLowFX() then
                    return
                end

                task.wait(0.25)
            end
        end)
    end

    Options.FishingLowFX:OnChanged(function()
        if Options.FishingLowFX.Value then
            TryApplyLowFX()
        else
            State.RestoreFishingLowFX()
        end
    end)

    task.defer(TryApplyLowFX)
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
    State.FishingArrow = 0
    State.FishingArrowChanged = 0
    State.FishingTickEMA = 0.05
    State.FishingPendingHold = nil
    State.FishingPendingTicks = 0
    State.FishingLastHoldChange = 0
end


local function RunFishingMinigameTick(Data)
    if
        not Options.FishingMinigameAssist.Value or
        State.FishingMode ~= "fighting" or
        not FishingHoldInput then

        return
    end

    -- ค่าจริงจาก server เท่านั้น
    local I = tonumber(Data.i)
    local G = tonumber(Data.g)

    if I == nil or G == nil then
        return
    end

    local W =
        math.max(
            0.012,
            tonumber(Data.w) or 0.11
        )

    local Arrow =
        math.sign(
            tonumber(Data.a) or 0
        )

    local InGreen =
        tonumber(Data.k) == 1

    local Tension =
        math.clamp(
            tonumber(Data.t) or 0,
            0,
            1
        )

    local Now = os.clock()
    local NominalDt = 0.05

    local Dt =
        State.FishingPrevTick and
        math.clamp(
            Now - State.FishingPrevTick,
            0.025,
            0.12
        ) or
        NominalDt

    -- EMA ของช่วง tick:
    -- ถ้า client มี spike จะเพิ่ม prediction horizon เอง
    State.FishingTickEMA =
        State.FishingTickEMA * 0.78 +
        Dt * 0.22

    local RawIV = 0
    local RawGV = 0

    if State.FishingPrevI ~= nil then
        RawIV =
            (I - State.FishingPrevI) / Dt
    end

    if State.FishingPrevG ~= nil then
        RawGV =
            (G - State.FishingPrevG) / Dt
    end

    -- smoothing พอประมาณ ไม่ตาม noise แต่ยังตอบสนองปลาเร็วทัน
    State.FishingIVelocity =
        State.FishingIVelocity * 0.28 +
        RawIV * 0.72

    State.FishingGVelocity =
        State.FishingGVelocity * 0.38 +
        RawGV * 0.62

    if Arrow ~= State.FishingArrow then
        State.FishingArrow = Arrow
        State.FishingArrowChanged = Now
    end

    State.FishingPrevI = I
    State.FishingPrevG = G
    State.FishingPrevTick = Now

    local Mini =
        FishingConfig and
        FishingConfig.Minigame or
        {}

    local HoldAccel =
        tonumber(Mini.HoldAccel) or 2.8

    local GravityAccel =
        tonumber(Mini.GravityAccel) or 2.1

    local TelegraphTime =
        tonumber(Mini.TelegraphTime) or 0.3

    -- กรอบยิ่งแคบ = ปลายิ่งยาก
    local Narrow =
        math.clamp(
            (0.12 - W) / 0.055,
            0,
            1
        )

    -- --------------------------------------------------------
    -- Adaptive look-ahead
    -- ปกติ ~1.2 tick
    -- ถ้า tick เลท/เครื่องกระตุก เพิ่ม horizon ชดเชย
    -- --------------------------------------------------------
    local Jitter =
        math.max(
            0,
            State.FishingTickEMA -
            NominalDt
        )

    local Horizon =
        math.clamp(
            0.058 +
            Jitter * 1.35 +
            Narrow * 0.012,
            0.052,
            0.120
        )

    local ArrowAge =
        Arrow ~= 0 and
        math.max(
            0,
            Now - State.FishingArrowChanged
        ) or
        0

    local ArrowProgress =
        Arrow ~= 0 and
        math.clamp(
            ArrowAge /
            math.max(TelegraphTime, 0.05),
            0,
            1
        ) or
        0

    -- --------------------------------------------------------
    -- Predict GreenZone
    -- --------------------------------------------------------
    local FutureG =
        G +
        State.FishingGVelocity *
        Horizon

    if Arrow ~= 0 then
        -- >/ < เป็น telegraph ก่อนเปลี่ยนทิศ
        -- เล็งด้านในของ GreenZone ล่วงหน้า ไม่ไปสุดขอบ
        local ArrowLead =
            W *
            (
                0.18 +
                ArrowProgress * 0.22 +
                Narrow * 0.06
            )

        -- tension สูงเล่น conservative ขึ้น
        ArrowLead *=
            1 -
            math.max(
                0,
                Tension - 0.60
            ) * 0.45

        FutureG +=
            Arrow *
            ArrowLead
    end

    FutureG =
        math.clamp(
            FutureG,
            0,
            1
        )

    -- ตอนหลุดกรอบ ให้ไล่ center ปัจจุบันมากกว่าอนาคต
    local Target

    if InGreen then
        Target = FutureG
    else
        Target =
            G +
            State.FishingGVelocity *
            math.min(
                Horizon,
                0.045
            )

        if Arrow ~= 0 then
            Target +=
                Arrow *
                W *
                0.10
        end

        Target =
            math.clamp(
                Target,
                0,
                1
            )
    end

    -- --------------------------------------------------------
    -- Predict Indicator จาก input ปัจจุบัน
    -- --------------------------------------------------------
    local CurrentAccel =
        State.FishingHolding and
        HoldAccel or
        -GravityAccel

    local PredictedI =
        I +
        State.FishingIVelocity *
        Horizon +
        0.5 *
        CurrentAccel *
        Horizon *
        Horizon

    local Error =
        Target - PredictedI

    local RelativeVelocity =
        State.FishingIVelocity -
        State.FishingGVelocity

    -- --------------------------------------------------------
    -- Braking-distance controller
    -- ช่วยไม่ให้พุ่งเลย GreenZone แล้วค่อยแก้ทีหลัง
    -- --------------------------------------------------------
    local BrakeDistance = 0

    if RelativeVelocity > 0 then
        BrakeDistance =
            RelativeVelocity *
            RelativeVelocity /
            math.max(
                2 * GravityAccel,
                0.1
            )
    elseif RelativeVelocity < 0 then
        BrakeDistance =
            RelativeVelocity *
            RelativeVelocity /
            math.max(
                2 * HoldAccel,
                0.1
            )
    end

    local DeadBand =
        math.clamp(
            W *
            (
                0.055 -
                Narrow * 0.012
            ),
            0.0025,
            0.007
        )

    local Candidate =
        State.FishingHolding

    if RelativeVelocity > 0 then
        -- กำลังวิ่งขวา: ปล่อยก่อนถึง target ตาม stopping distance
        if
            Error <=
            BrakeDistance +
            DeadBand then

            Candidate = false
        elseif Error > DeadBand then
            Candidate = true
        end

    elseif RelativeVelocity < 0 then
        -- กำลังวิ่งซ้าย: กดเบรกก่อนถึง target
        if
            -Error <=
            BrakeDistance +
            DeadBand then

            Candidate = true
        elseif Error < -DeadBand then
            Candidate = false
        end

    else
        if Error > DeadBand then
            Candidate = true
        elseif Error < -DeadBand then
            Candidate = false
        end
    end

    -- --------------------------------------------------------
    -- Emergency edge guard
    -- ใกล้หลุดกรอบจริง = เปลี่ยน input ทันที ไม่รอ confirmation
    -- --------------------------------------------------------
    local LeftEdge = G - W
    local RightEdge = G + W

    local EdgeMargin =
        math.clamp(
            W *
            (
                0.17 +
                Narrow * 0.035
            ),
            0.008,
            0.025
        )

    local Emergency = false

    if I <= LeftEdge + EdgeMargin then
        Candidate = true
        Emergency = true
    elseif I >= RightEdge - EdgeMargin then
        Candidate = false
        Emergency = true
    end

    -- --------------------------------------------------------
    -- Lag compensation
    -- ไม่ freeze input เมื่อ tick เลท เพราะปลา Legendary/Mythic
    -- เปลี่ยนทิศเร็ว; Horizon ด้านบนชดเชย Dt/Jitter แล้ว
    -- --------------------------------------------------------

    -- --------------------------------------------------------
    -- 2-tick confirmation
    -- ป้องกัน HoldInput true/false สลับทุก tick จาก noise
    -- แต่ arrow เริ่มใหม่/หลุดขอบให้ตอบสนองเร็ว
    -- --------------------------------------------------------
    if Candidate ~= State.FishingHolding then
        if Candidate == State.FishingPendingHold then
            State.FishingPendingTicks += 1
        else
            State.FishingPendingHold = Candidate
            State.FishingPendingTicks = 1
        end

        local ArrowFresh =
            Arrow ~= 0 and
            ArrowAge <= 0.075

        local Needed =
            (
                Emergency or
                ArrowFresh or
                not InGreen or
                Narrow >= 0.55 or
                Dt >= 0.075
            ) and
            1 or
            2

        if State.FishingPendingTicks >= Needed then
            SetFishingHold(
                Candidate,
                Emergency
            )

            State.FishingPendingHold = nil
            State.FishingPendingTicks = 0
        end
    else
        State.FishingPendingHold = nil
        State.FishingPendingTicks = 0
    end
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

local SellFishingPartsWhenFull

SellFishingPartsWhenFull = function()
    if
        State.FishingSellingParts or
        State.FishingSelling or
        not SellItem then

        return false
    end

    local MaterialId = GetFishingMaterialId()
    local Before = GetFishingMaterialAmount()

    if Before <= 0 then
        State.FishingFull = false
        return true
    end

    State.FishingSellingParts = true
    State.FishingFull = true
    State.FishingMode = "idle"
    State.FishingAutoRunning = false
    State.FishingAutoToken += 1

    StopFishingVisualLock()
    SetFishingHold(false)

    if FishingAction then
        pcall(function()
            FishingAction:FireServer("autoOff")
            FishingAction:FireServer("cancel")
        end)
    end

    -- FishPart เป็น Material ของ Market ไม่ใช่ bucket fish ของ FishingShop
    local Stall = Workspace:FindFirstChild("GlobalMarketStall", true)
    local StallPart =
        Stall and
        (
            Stall:IsA("BasePart") and Stall or
            Stall:FindFirstChildWhichIsA("BasePart", true)
        )

    Teleport(
        StallPart and
        (StallPart.CFrame * CFrame.new(0, 3, 3)) or
        SellPosition
    )

    task.wait(0.35)

    local Sold = false

    for _ = 1, 3 do
        local Current = GetFishingMaterialAmount()

        if Current <= 0 then
            Sold = true
            break
        end

        pcall(function()
            SellItem:FireServer(MaterialId, Current)
        end)

        local Deadline = os.clock() + 1.25

        repeat
            task.wait(0.08)

            if GetFishingMaterialAmount() < Current then
                Sold = true
                break
            end
        until os.clock() >= Deadline

        if Sold then
            break
        end
    end

    local After = GetFishingMaterialAmount()

    if After < Before then
        Sold = true
        State.FishingFull = false
        State.FishingFullAmount = After

        Notify(
            "Auto Fishing",
            "ขาย " ..
            tostring(MaterialId) ..
            " " ..
            tostring(Before) ..
            " → " ..
            tostring(After)
        )
    else
        Notify(
            "Auto Fishing",
            "FishPart เต็ม แต่ขายไม่สำเร็จ"
        )
    end

    -- กลับจุดตกปลาเสมอ
    Teleport(GetFishingZoneCFrame())
    task.wait(0.25)

    State.FishingSellingParts = false

    if
        Sold and
        State.Alive and
        Options.AutoFishing.Value then

        EquipFishingRod()
        task.wait(0.12)
        task.spawn(StartAutoFishing)
    end

    return Sold
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

    local CurrentBait = GetFishingBait()

    if
        Options.FishingAutoBuyBait.Value and
        CurrentBait <= Options.FishingBaitThreshold.Value then

        QueueAutoBuyBait(
            CurrentBait <= 0,
            false
        )
    end

    if CurrentBait <= 0 then
        -- รอ Bait.Value Changed เป็นตัวเริ่มใหม่หลังซื้อ
        return
    end

    if not EquipFishingRod() then
        if not EnsureFishingRod() then
            return
        end
    end

    local HRP = Root()
    if not HRP then return end

    if not IsInsideFishingZone(HRP.Position) then
        if not Options.FishingAutoTP.Value then
            Notify("Auto Fishing", "ต้องอยู่ใน Fishing Zone")
            return
        end

        Teleport(GetFishingZoneCFrame())
        task.wait(0.35)
    end

    -- วาป/ซื้อของอาจทำ Tool หลุดมือ จึงเช็กอีกครั้งก่อน start จริง
    if not EquipFishingRod() then
        if os.clock() - State.LastFishingRodNotify >= 3 then
            State.LastFishingRodNotify = os.clock()
            Notify(
                "Auto Fishing",
                "พบ Backpack[\"" ..
                GetSelectedRodToolName() ..
                "\"] แต่ยังถือไม่สำเร็จ — กำลังลองใหม่"
            )
        end
        State.FishingMode = "idle"
        return
    end

    local EquippedTool = GetFishingRodTool()
    local CurrentChar = Character()

    if not EquippedTool or EquippedTool.Parent ~= CurrentChar then
        State.FishingMode = "idle"
        return
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
            -- ไม่ scan/แก้ GUI ตอน bite เพราะเป็นจังหวะที่ต้องลด spike
            if
                Options.FishingLowFX.Value and
                State.ApplyFishingLowFX then

                State.ApplyFishingLowFX()
            end

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

                QueueAutoBuyBait(
                    true,
                    true
                )
            end

        elseif Action == "autoStopFull" then
            State.FishingMode = "idle"
            State.FishingAutoRunning = false
            State.FishingAutoToken += 1
            State.FishingFull = true
            State.FishingFullAmount = GetFishingMaterialAmount()
            StopFishingVisualLock()
            SetFishingHold(false)

            if
                Options.FishingAutoSellFull.Value and
                SellFishingPartsWhenFull then

                task.spawn(SellFishingPartsWhenFull)
            else
                Notify(
                    "Auto Fishing",
                    "FishPart เต็ม " ..
                    tostring(GetFishingMaterialAmount()) ..
                    "/" ..
                    tostring(GetFishingMaterialMax())
                )
            end

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

local function StartFishingWatcher()
    State.FishingLoopToken += 1
    local Token = State.FishingLoopToken

    task.spawn(function()
        while
            State.Alive and
            Options.AutoFishing.Value and
            Token == State.FishingLoopToken do

            if
                not State.FishingBuyingRod and
                not State.FishingSellingParts then

                if
                    Options.FishingAutoSellFull.Value and
                    IsFishingMaterialFull() then

                    State.FishingFull = true
                    State.FishingFullAmount = GetFishingMaterialAmount()
                    SellFishingPartsWhenFull()

                elseif State.FishingFull then
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

            task.wait(0.6)
        end
    end)
end

Options.FishingAutoSellFull:OnChanged(function()
    if
        Options.FishingAutoSellFull.Value and
        Options.AutoFishing.Value and
        IsFishingMaterialFull() and
        SellFishingPartsWhenFull then

        task.spawn(SellFishingPartsWhenFull)
    end
end)

Options.FishingBaitThreshold:OnChanged(function()
    if
        Options.FishingAutoBuyBait.Value and
        Options.AutoFishing.Value then

        local Current = GetFishingBait()

        if
            Current <=
            (
                tonumber(
                    Options.FishingBaitThreshold.Value
                ) or 5
            ) then

            QueueAutoBuyBait(
                Current <= 0,
                false
            )
        end
    end
end)

AutoFishingToggle:OnChanged(function()
    State.FishingLoopToken += 1

    if Options.AutoFishing.Value then
        StartFishingWatcher()

        if Options.FishingAutoBuyBait.Value then
            BindAutoBaitConnection()
        end
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

--// SUPERMARKET
Tabs.Main:AddSection("Supermarket")

local SuperMarketValues = {}

if
    FoodConfig and
    type(FoodConfig.Names) == "function" then

    local OK, Names =
        pcall(function()
            return FoodConfig.Names()
        end)

    if OK and type(Names) == "table" then
        for _, Name in ipairs(Names) do
            table.insert(
                SuperMarketValues,
                tostring(Name)
            )
        end
    end
end

if #SuperMarketValues == 0 then
    SuperMarketValues = {
        "เบอร์เกอร์",
        "ขนมชุบ",
        "ขนมถ้วย",
        "ขนมวุ้น",
        "ขนมเทียน",
        "ขนมชั้น",
        "ต้มยำกุ้ง",
        "นมโปรตีนสตรอเบอร์รี",
        "ชาเขียวเย็น",
        "ชาเขียวเย็นผสมกาแฟ",
        "โคลด์ไต้หวันครีม",
        "สตรอเบอร์รีโซดา"
    }
end

local SuperMarketDropdown =
    Tabs.Main:AddDropdown(
        "SuperMarketItem",
        {
            Title = "Select Supermarket Item",
            Description = "เลือกอาหาร/เครื่องดื่มจาก Supermarket",
            Values = SuperMarketValues,
            Multi = false,
            Default =
                #SuperMarketValues > 0 and
                1 or
                nil,
            Searchable = true
        }
    )

Tabs.Main:AddButton({
    Title = "Buy Supermarket Item",
    Description = "ซื้อของที่เลือกผ่าน SuperMarketEvent.ConfirmEvent",

    Callback = function()
        local Remote =
            ReplicatedStorage:
                FindFirstChild("SuperMarketEvent")

        Remote =
            Remote and
            Remote:FindFirstChild("ConfirmEvent")

        if
            not Remote or
            not Remote:IsA("RemoteEvent") then

            Notify(
                "Supermarket",
                "ไม่พบ SuperMarketEvent.ConfirmEvent"
            )

            return
        end

        SuperMarketConfirm = Remote

        local ItemName =
            Options.SuperMarketItem.Value

        if
            not ItemName or
            ItemName == "" then

            Notify(
                "Supermarket",
                "กรุณาเลือกสินค้า"
            )

            return
        end

        local OK =
            pcall(function()
                Remote:FireServer(ItemName)
            end)

        Notify(
            "Supermarket",
            OK and
            ("สั่งซื้อ " .. tostring(ItemName)) or
            "Remote error"
        )
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

local function StartCraftWatcher()
    State.CraftLoopToken += 1
    local Token = State.CraftLoopToken

    task.spawn(function()
        while
            State.Alive and
            Options.AutoCraft.Value and
            Token == State.CraftLoopToken do

            if not Options.AutoFarm.Value and not State.Selling and RequestCraft then
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
end

AutoCraftToggle:OnChanged(function()
    State.CraftLoopToken += 1
    if Options.AutoCraft.Value then
        StartCraftWatcher()
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

local function AnyESPEnabled()
    return
        Options.ESPChams.Value or
        Options.ESPName.Value or
        Options.ESPRank.Value or
        Options.ESPHealth.Value
end

local function StopESPRuntime()
    State.ESPLoopToken += 1

    if State.ESPPlayerAddedConnection then
        pcall(function()
            State.ESPPlayerAddedConnection:Disconnect()
        end)
        State.ESPPlayerAddedConnection = nil
    end

    if State.ESPPlayerRemovingConnection then
        pcall(function()
            State.ESPPlayerRemovingConnection:Disconnect()
        end)
        State.ESPPlayerRemovingConnection = nil
    end

    for Player in pairs(State.ESP) do
        RemoveESP(Player)
    end
end

local function StartESPRuntime()
    if not AnyESPEnabled() then
        StopESPRuntime()
        return
    end

    if not State.ESPPlayerAddedConnection then
        State.ESPPlayerAddedConnection =
            Players.PlayerAdded:Connect(function(Player)
                Player.CharacterAdded:Connect(function()
                    if not AnyESPEnabled() then return end
                    task.wait(0.2)
                    CreateESP(Player)
                end)
            end)
    end

    if not State.ESPPlayerRemovingConnection then
        State.ESPPlayerRemovingConnection =
            Players.PlayerRemoving:Connect(RemoveESP)
    end

    for _, Player in ipairs(Players:GetPlayers()) do
        if
            Player ~= LocalPlayer and
            Player.Character and
            not State.ESP[Player] then

            task.defer(CreateESP, Player)
        end
    end

    State.ESPLoopToken += 1
    local Token = State.ESPLoopToken

    task.spawn(function()
        while
            State.Alive and
            AnyESPEnabled() and
            Token == State.ESPLoopToken do

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

            task.wait(0.35)
        end
    end)
end

for _, Name in ipairs({"ESPChams", "ESPName", "ESPRank", "ESPHealth"}) do
    Options[Name]:OnChanged(function()
        if AnyESPEnabled() then
            StartESPRuntime()
        else
            StopESPRuntime()
        end
    end)
end

--// PLAYER TAB
Tabs.Player:AddSection("Movement")

local FlyToggle = Tabs.Player:AddToggle("Fly", { Title = "Fly", Default = false })
Tabs.Player:AddSlider("FlySpeed", { Title = "Fly Speed", Min = 10, Max = 300, Default = 60, Rounding = 0 })

local NoclipToggle = Tabs.Player:AddToggle("Noclip", { Title = "Noclip", Default = false })
Tabs.Player:AddToggle("TPWalk", { Title = "TP Walk", Default = false })
Tabs.Player:AddSlider("TPWalkSpeed", { Title = "TP Walk Speed", Min = 10, Max = 300, Default = 60, Rounding = 0 })

local WalkSpeedSlider = Tabs.Player:AddSlider("WalkSpeed", { Title = "WalkSpeed", Min = 0, Max = 250, Default = 16, Rounding = 0 })

Tabs.Player:AddToggle("AntiVoid", {
    Title = "Anti Void",
    Description = "สร้างพื้นกันตก 3 ชั้น + วาปกลับจุดปลอดภัยก่อนลง Void",
    Default = false
})

local ANTI_VOID_LEVELS = {-125, -145, -165}
local ANTI_VOID_TRIGGER_Y = -110
local ANTI_VOID_FALLBACK_Y = -90
local ANTI_VOID_SIZE = Vector3.new(700, 2, 700)

local function StopAntiVoid()
    if State.AntiVoidConnection then
        pcall(function()
            State.AntiVoidConnection:Disconnect()
        end)
        State.AntiVoidConnection = nil
    end

    if State.AntiVoidFolder then
        pcall(function()
            State.AntiVoidFolder:Destroy()
        end)
        State.AntiVoidFolder = nil
    end

    table.clear(State.AntiVoidParts)
    State.AntiVoidLastSafe = nil
end

local function StartAntiVoid()
    StopAntiVoid()

    local HRP = Root()
    if not HRP then return end

    local Folder = Instance.new("Folder")
    Folder.Name = "RCA_AntiVoid"
    Folder.Parent = Workspace
    State.AntiVoidFolder = Folder

    for Index, Y in ipairs(ANTI_VOID_LEVELS) do
        local Plate = Instance.new("Part")
        Plate.Name = "AntiVoid_" .. tostring(Index)
        Plate.Anchored = true
        Plate.CanCollide = true
        Plate.CanTouch = false
        Plate.CanQuery = false
        Plate.Size = ANTI_VOID_SIZE
        Plate.Transparency = 0.72
        Plate.Position = Vector3.new(
            HRP.Position.X,
            Y,
            HRP.Position.Z
        )
        Plate.Parent = Folder

        table.insert(State.AntiVoidParts, Plate)
    end

    State.AntiVoidLastSafe = HRP.CFrame
    State.AntiVoidLastSafeTick = os.clock()

    State.AntiVoidConnection =
        RunService.Heartbeat:Connect(function()
            if not State.Alive or not Options.AntiVoid.Value then
                return
            end

            local CurrentHRP = Root()
            local Hum = Humanoid()

            if not CurrentHRP or not Hum or Hum.Health <= 0 then
                return
            end

            -- ให้พื้น 3 ชั้นตาม X/Z ของผู้เล่น แต่ Y คงที่
            for Index, Plate in ipairs(State.AntiVoidParts) do
                local Y = ANTI_VOID_LEVELS[Index]

                if Plate and Plate.Parent and Y then
                    Plate.Position = Vector3.new(
                        CurrentHRP.Position.X,
                        Y,
                        CurrentHRP.Position.Z
                    )
                end
            end

            local Now = os.clock()
            local Y = CurrentHRP.Position.Y
            local FallingSpeed = CurrentHRP.AssemblyLinearVelocity.Y

            -- จำเฉพาะจุดที่ยืน/เคลื่อนที่แบบปลอดภัย
            -- ไม่อัปเดต last safe ระหว่างกำลังร่วงลง Void
            if
                Y > ANTI_VOID_TRIGGER_Y + 8 and
                (
                    Hum.FloorMaterial ~= Enum.Material.Air or
                    FallingSpeed > -4
                ) and
                Now - State.AntiVoidLastSafeTick >= 0.2 then

                State.AntiVoidLastSafe = CurrentHRP.CFrame
                State.AntiVoidLastSafeTick = Now
            end

            local Danger =
                Y <= ANTI_VOID_TRIGGER_Y or
                (
                    Y < ANTI_VOID_TRIGGER_Y + 12 and
                    FallingSpeed <= -90
                )

            if
                Danger and
                Now - State.AntiVoidLastRescue >= 0.8 then

                State.AntiVoidLastRescue = Now

                local Target = State.AntiVoidLastSafe

                if
                    not Target or
                    Target.Position.Y <= ANTI_VOID_TRIGGER_Y + 5 then

                    Target = CFrame.new(
                        CurrentHRP.Position.X,
                        ANTI_VOID_FALLBACK_Y,
                        CurrentHRP.Position.Z
                    )
                end

                -- ยกขึ้นจากจุดปลอดภัยเล็กน้อย กันติดพื้น/ขอบ part
                Teleport(Target * CFrame.new(0, 4, 0))

                pcall(function()
                    Hum:ChangeState(Enum.HumanoidStateType.Jumping)
                end)

                Notify(
                    "Anti Void",
                    "ช่วยขึ้นจาก Void แล้ว"
                )
            end
        end)
end

do
    local AntiVoidOption = Options.AntiVoid

    if
        AntiVoidOption and
        type(AntiVoidOption.OnChanged) == "function" then

        AntiVoidOption:OnChanged(function()
            if AntiVoidOption.Value then
                StartAntiVoid()
            else
                StopAntiVoid()
            end
        end)
    end
end


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

local function StopMovementRuntime()
    if State.MovementConnection then
        pcall(function() State.MovementConnection:Disconnect() end)
        State.MovementConnection = nil
    end
end

local function UpdateMovementRuntime()
    if Options.Fly.Value or Options.TPWalk.Value then
        if State.MovementConnection then return end
State.MovementConnection = RunService.RenderStepped:Connect(function(Delta)
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

    else
        StopMovementRuntime()
    end
end

-- Bind movement toggles only after UpdateMovementRuntime exists.
FlyToggle:OnChanged(function()
    if Options.Fly.Value then
        StartFly()
    else
        StopFly()
    end

    UpdateMovementRuntime()
end)

Options.TPWalk:OnChanged(function()
    UpdateMovementRuntime()
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

local function StopNoclipRuntime()
    if State.NoclipConnection then
        pcall(function()
            State.NoclipConnection:Disconnect()
        end)
        State.NoclipConnection = nil
    end

    RestoreNoclip()
end

do
    local NoclipOption = Options.Noclip

    if NoclipOption and type(NoclipOption.OnChanged) == "function" then
        NoclipOption:OnChanged(function()
            if NoclipOption.Value then
                UpdateCharParts()

                if not State.NoclipConnection then
                    State.NoclipConnection =
                        RunService.Stepped:Connect(function()
                            if not NoclipOption.Value then return end

                            for _, Part in ipairs(State.CharParts) do
                                if Part and Part.Parent then
                                    if State.NoclipParts[Part] == nil then
                                        State.NoclipParts[Part] =
                                            State.UndergroundParts[Part] ~= nil and
                                            State.UndergroundParts[Part] or
                                            Part.CanCollide
                                    end

                                    Part.CanCollide = false
                                end
                            end
                        end)
                end
            else
                StopNoclipRuntime()
            end
        end)
    end
end

do
    local WalkSpeedOption = Options.WalkSpeed

    if WalkSpeedOption and type(WalkSpeedOption.OnChanged) == "function" then
        WalkSpeedOption:OnChanged(function(Value)
            State.WalkSpeed = tonumber(Value) or State.WalkSpeed

            local Hum = Humanoid()
            if Hum then
                Hum.WalkSpeed = State.WalkSpeed
            end
        end)
    end
end

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
    Description = "-1514, -75, 2822",
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

--// MISC TAB - PERFORMANCE
local RestoreDeleteMap

do
Tabs.Misc:AddSection("Performance")

local DELETE_MAP_ARMY_NAME =
    "\224\184\149\224\184\182\224\184\129\224\184\129\224\184\173\224\184\135\224\184\151\224\184\177\224\184\158\224\184\154\224\184\129"

local DELETE_MAP_CRITICAL = {
    Vector3.new(-1514, -75, 2822),   -- Fishing farm
    Vector3.new(-1627, -62, 3274),   -- Fishing shop
    Vector3.new(-2088, -61, 3235),   -- Market
    Vector3.new(-2103, -65, 3269),   -- Craft
    Vector3.new(-2755, -30, 3987),   -- Banana
    Vector3.new(-2238, -49, 4504),   -- Tank
    Vector3.new(-2716, -43, 3362)    -- Dress
}

local DELETE_MAP_KEEP_WORDS = {
    "floor", "ground", "road", "baseplate",
    "spawn", "shop", "market", "stall",
    "zone", "giver", "armor", "armband",
    "red kg", "headset", "webbing", "peaked",
    "พื้น", "ถนน", "ขาย", "ร้าน", "ตลาด",
    "ตกปลา", "ฟาร์ม", "เกิด"
}

local function HasHumanoidAncestor(Object, StopAt)
    local Parent = Object.Parent

    while Parent and Parent ~= StopAt do
        if
            Parent:IsA("Model") and
            Parent:FindFirstChildOfClass("Humanoid") then

            return true
        end

        Parent = Parent.Parent
    end

    return false
end

local function NearDeleteMapCritical(Position, Radius)
    Radius = Radius or 80

    for _, Point in ipairs(DELETE_MAP_CRITICAL) do
        local Offset =
            Vector2.new(
                Position.X - Point.X,
                Position.Z - Point.Z
            )

        if Offset.Magnitude <= Radius then
            return true
        end
    end

    return false
end

local function KeepMapPart(Part, Maps, Army)
    if not Part or not Part.Parent then
        return true
    end

    -- Auto Dress ใช้ GetChildren()[index] ในตึกนี้
    -- ต้องเก็บทั้งตึกไว้ ไม่งั้น index เปลี่ยน
    if Army and Part:IsDescendantOf(Army) then
        return true
    end

    if Part:IsA("SpawnLocation") then
        return true
    end

    if HasHumanoidAncestor(Part, Maps) then
        return true
    end

    if
        Part:FindFirstChildOfClass("ProximityPrompt") or
        Part:FindFirstChildOfClass("ClickDetector") then

        return true
    end

    local Name = string.lower(tostring(Part.Name or ""))

    for _, Word in ipairs(DELETE_MAP_KEEP_WORDS) do
        if string.find(Name, Word, 1, true) then
            return true
        end
    end

    -- เก็บพื้นแผ่นใหญ่ไว้เดิน
    local Size = Part.Size

    if
        Size.Y <= 12 and
        Size.X >= 35 and
        Size.Z >= 35 then

        return true
    end

    -- เก็บพื้น/ชิ้นส่วนใกล้จุดใช้งานหลัก
    if
        Size.Y <= 18 and
        NearDeleteMapCritical(Part.Position, 90) then

        return true
    end

    return false
end

RestoreDeleteMap = function()
    State.DeleteMapToken += 1
    State.DeleteMapBusy = false

    for Index = #State.DeletedMapItems, 1, -1 do
        local Entry = State.DeletedMapItems[Index]

        if
            Entry and
            Entry.Object and
            Entry.Object.Parent and
            Entry.Parent and
            Entry.Parent.Parent then

            pcall(function()
                Entry.Object.Parent = Entry.Parent
            end)
        end

        State.DeletedMapItems[Index] = nil

        if Index % 150 == 0 then
            task.wait()
        end
    end

    if State.DeleteMapCache then
        pcall(function()
            State.DeleteMapCache:Destroy()
        end)

        State.DeleteMapCache = nil
    end
end

local function ApplyDeleteMap()
    if State.DeleteMapBusy then
        return
    end

    RestoreDeleteMap()

    local Maps = Workspace:FindFirstChild("Maps")
    if not Maps then
        Notify("Delete Map", "ไม่พบ Workspace.Maps")
        return
    end

    local Army = Maps:FindFirstChild(DELETE_MAP_ARMY_NAME)

    local Cache = Instance.new("Folder")
    Cache.Name = "RCA_DeletedMapCache"
    Cache.Parent = ReplicatedStorage

    State.DeleteMapCache = Cache
    State.DeleteMapBusy = true
    State.DeleteMapToken += 1

    local Token = State.DeleteMapToken
    local Candidates = {}

    for _, Object in ipairs(Maps:GetDescendants()) do
        if
            Object:IsA("BasePart") and
            not KeepMapPart(Object, Maps, Army) then

            -- ถ้ามี BasePart ancestor ที่จะถูกย้ายอยู่แล้ว
            -- ไม่ต้องย้าย child ซ้ำ
            local Parent = Object.Parent
            local Nested = false

            while Parent and Parent ~= Maps do
                if
                    Parent:IsA("BasePart") and
                    not KeepMapPart(Parent, Maps, Army) then

                    Nested = true
                    break
                end

                Parent = Parent.Parent
            end

            if not Nested then
                table.insert(Candidates, Object)
            end
        end
    end

    task.spawn(function()
        local Removed = 0

        for Index, Object in ipairs(Candidates) do
            if
                not State.Alive or
                Token ~= State.DeleteMapToken or
                not Options.DeleteMap.Value then

                break
            end

            if
                Object and
                Object.Parent and
                Object:IsDescendantOf(Maps) then

                local Parent = Object.Parent

                table.insert(
                    State.DeletedMapItems,
                    {
                        Object = Object,
                        Parent = Parent
                    }
                )

                pcall(function()
                    Object.Parent = Cache
                end)

                Removed += 1
            end

            if Index % 120 == 0 then
                task.wait()
            end
        end

        State.DeleteMapBusy = false

        if
            Options.DeleteMap.Value and
            Token == State.DeleteMapToken then

            Notify(
                "Delete Map",
                "ลดแมพแล้ว " .. tostring(Removed) .. " parts"
            )
        end
    end)
end

local DeleteMapToggle = Tabs.Misc:AddToggle("DeleteMap", {
    Title = "Delete Map",
    Description = "ซ่อน/ย้ายส่วนแมพที่ไม่จำเป็น แต่เก็บ NPC, พื้น, ร้าน, จุดฟาร์ม และตึก Auto Dress",
    Default = false
})

DeleteMapToggle:OnChanged(function()
    if Options.DeleteMap.Value then
        ApplyDeleteMap()
    else
        task.spawn(RestoreDeleteMap)
    end
end)
end

--// MISC TAB - CAMERA
Tabs.Misc:AddSection("Camera")

local NoGunZoomToggle = Tabs.Misc:AddToggle("NoGunZoom", {
    Title = "No Gun Zoom",
    Description = "ถือปืนแล้วไม่บังคับซูม/ลด FOV",
    Default = false
})

NoGunZoomToggle:OnChanged(function()
    pcall(function()
        RunService:UnbindFromRenderStep(NO_ZOOM_BIND)
    end)

    if Options.NoGunZoom.Value then
        ApplyNoGunZoom()
        RunService:BindToRenderStep(
            NO_ZOOM_BIND,
            Enum.RenderPriority.Camera.Value + 10,
            ApplyNoGunZoom
        )
    else
        if GetEquippedGun() then
            LocalPlayer.CameraMaxZoomDistance = 0.5
        else
            LocalPlayer.CameraMaxZoomDistance = DEFAULT_MAX_ZOOM
        end
    end
end)

--// MISC TAB - AUTO DRESS
do
Tabs.Misc:AddSection("Auto Dress")

local DRESS_PROMPT_TIMEOUT = 1.25
local DRESS_READY_STABLE = 0.14
local DRESS_APPLY_WAIT = 0.22
local DRESS_CLICK_WAIT = 0.20
local DRESS_RETURN_CF = CFrame.new(-2716, -43, 3362)

local function GetDressBase()
    local Maps = Workspace:FindFirstChild("Maps")
    local TargetMap =
        Maps and
        Maps:FindFirstChild(
            "\224\184\149\224\184\182\224\184\129\224\184\129\224\184\173\224\184\135\224\184\151\224\184\177\224\184\158\224\184\154\224\184\129"
        )

    return
        TargetMap and
        TargetMap:FindFirstChild("Model") and
        TargetMap.Model:FindFirstChild("Model")
end

local function GetPromptWorldCFrame(Prompt)
    if not Prompt then return nil end

    local Parent = Prompt.Parent
    if not Parent then return nil end

    if Parent:IsA("Attachment") then
        return Parent.WorldCFrame
    end

    if Parent:IsA("BasePart") then
        return Parent.CFrame
    end

    if Parent:IsA("Model") then
        return Parent:GetPivot()
    end

    local Part = Parent:FindFirstChildWhichIsA("BasePart", true)
    return Part and Part.CFrame or nil
end

local function WaitDressPromptShown(Prompt, Offset, Timeout)
    if not Prompt or not Prompt.Parent then
        return false
    end

    local Shown = false
    local Ready = false
    local InRangeSince = nil

    local Connection =
        ProximityPromptService.PromptShown:Connect(
            function(CurrentPrompt)
                if CurrentPrompt == Prompt then
                    Shown = true
                end
            end
        )

    local CF = GetPromptWorldCFrame(Prompt)

    if not CF then
        Connection:Disconnect()
        return false
    end

    Teleport(
        CF *
        (
            Offset or
            CFrame.new(0, 0, -1.8)
        )
    )

    local Started = os.clock()
    Timeout = Timeout or DRESS_PROMPT_TIMEOUT

    repeat
        if Shown then
            Ready = true
            break
        end

        local HRP = Root()
        local PromptCF = GetPromptWorldCFrame(Prompt)

        if
            HRP and
            PromptCF and
            Prompt.Enabled then

            local MaxDistance =
                math.max(
                    1,
                    tonumber(Prompt.MaxActivationDistance) or 10
                )

            local Distance =
                (HRP.Position - PromptCF.Position).Magnitude

            if Distance <= MaxDistance - 0.15 then
                if not InRangeSince then
                    InRangeSince = os.clock()
                elseif
                    os.clock() - InRangeSince >=
                    DRESS_READY_STABLE then

                    -- fallback เผื่อ Custom Prompt ไม่ส่ง PromptShown
                    Ready = true
                    break
                end
            else
                InRangeSince = nil
            end
        end

        task.wait(0.03)
    until os.clock() - Started >= Timeout

    Connection:Disconnect()

    -- ให้ GUI/Highlight ของ Prompt มีเวลา render อีก 1 frame
    task.wait(0.05)

    return Ready or Shown
end

local function FireDressPrompt(Prompt)
    if not Prompt or not fireproximityprompt then
        return false
    end

    local OK = pcall(function()
        fireproximityprompt(Prompt)
    end)

    if OK then
        task.wait(DRESS_APPLY_WAIT)
    end

    return OK
end

local function RealEDressPrompt(Prompt)
    if not Prompt then
        return false
    end

    local Hold =
        math.max(
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

        task.wait(Hold + 0.05)

        VirtualInputManager:SendKeyEvent(
            false,
            Enum.KeyCode.E,
            false,
            game
        )
    end)

    if OK then
        task.wait(DRESS_APPLY_WAIT)
    end

    return OK
end

local function DressPromptStep(Prompt, Offset, UseRealE)
    if not Prompt then
        return false
    end

    WaitDressPromptShown(
        Prompt,
        Offset,
        DRESS_PROMPT_TIMEOUT
    )

    if UseRealE then
        return RealEDressPrompt(Prompt)
    end

    return FireDressPrompt(Prompt)
end

local function DressClickStep(ClickDetector)
    if not ClickDetector or not ClickDetector.Parent then
        return false
    end

    local CF = GetInstanceCFrame(ClickDetector.Parent)
    if not CF then return false end

    Teleport(CF * CFrame.new(0, 1.3, 0))
    task.wait(DRESS_READY_STABLE)

    if not fireclickdetector then
        return false
    end

    local OK = pcall(function()
        fireclickdetector(ClickDetector)
    end)

    if OK then
        task.wait(DRESS_CLICK_WAIT)
    end

    return OK
end

Tabs.Misc:AddButton({
    Title = "Auto dress พลทหาร",
    Description = "วาปทีละชิ้น รอ Prompt/Highlight ขึ้นก่อนค่อยใส่อันต่อไป",
    Callback = function()
        task.spawn(function()
            local Base = GetDressBase()

            if not Base then
                Notify(
                    "Auto dress พลทหาร",
                    "ไม่พบตึกกองทัพบก"
                )
                return
            end

            -- 1) ArmBand01
            pcall(function()
                local C5 = Base:GetChildren()[5]
                local Prompt =
                    C5 and
                    C5:FindFirstChild("ArmBand01") and
                    C5.ArmBand01:FindFirstChild(" Armband") and
                    C5.ArmBand01[" Armband"]:FindFirstChild("Arm1") and
                    C5.ArmBand01[" Armband"].Arm1:FindFirstChild("Right Arm") and
                    C5.ArmBand01[" Armband"].Arm1["Right Arm"]:FindFirstChild("ProximityPrompt")

                DressPromptStep(Prompt)
            end)

            -- 2) Red KG พลทหาร
            pcall(function()
                local C7 = Base:GetChildren()[7]
                local FirstModel =
                    C7 and
                    C7:GetChildren()[1]

                local RedKG =
                    FirstModel and
                    FirstModel:FindFirstChild("Red KG")

                local Prompt =
                    RedKG and
                    RedKG:FindFirstChild("ProximityPrompt")

                DressPromptStep(
                    Prompt,
                    CFrame.new(0, 0, -2),
                    true
                )
            end)

            -- 3) Hat/Giver
            pcall(function()
                local C33 = Base:GetChildren()[33]
                local Child3 =
                    C33 and
                    C33:GetChildren()[3]

                local Giver =
                    Child3 and
                    Child3:FindFirstChild("Giver")

                local Prompt =
                    Giver and
                    Giver:FindFirstChild("ProximityPrompt")

                DressPromptStep(Prompt)
            end)

            -- 4) Webbing2
            pcall(function()
                local Webbing =
                    Base:FindFirstChild("Webbing2")

                local CD =
                    Webbing and
                    (
                        Webbing:FindFirstChildOfClass("ClickDetector") or
                        Webbing:FindFirstChild("ClickDetector")
                    )

                DressClickStep(CD)
            end)

            -- 5) Helmet / Model 79
            pcall(function()
                local Item79 =
                    Base:GetChildren()[79]

                local Prompt =
                    Item79 and
                    Item79:FindFirstChild("ProximityPrompt")

                DressPromptStep(Prompt)
            end)

            -- 6) Headset
            pcall(function()
                local HeadsetModel =
                    Base:FindFirstChild(
                        "\224\184\156\224\184\161\224\184\156\224\184\185\224\185\137\224\184\138\224\184\178\224\184\162"
                    )

                local Headset =
                    HeadsetModel and
                    HeadsetModel:FindFirstChild("Headset")

                local Prompt =
                    Headset and
                    Headset:FindFirstChild("ProximityPrompt")

                DressPromptStep(Prompt)
            end)

            Teleport(DRESS_RETURN_CF)
            Notify("Auto dress พลทหาร", "ใส่ชุดเสร็จแล้ว")
        end)
    end
})

Tabs.Misc:AddButton({
    Title = "Auto dress ร้อยเอก",
    Description = "ArmBand09 → Armor → Giver → Peaked cap → Red KG",
    Callback = function()
        task.spawn(function()
            local Base = GetDressBase()

            if not Base then
                Notify(
                    "Auto dress ร้อยเอก",
                    "ไม่พบตึกกองทัพบก"
                )
                return
            end

            -- 1) ArmBand09
            pcall(function()
                local C5 = Base:GetChildren()[5]
                local ArmBand09 =
                    C5 and
                    C5:FindFirstChild("ArmBand09")

                local Prompt =
                    ArmBand09 and
                    ArmBand09:FindFirstChild(" Armband") and
                    ArmBand09[" Armband"]:FindFirstChild("Arm1") and
                    ArmBand09[" Armband"].Arm1:FindFirstChild("Right Arm") and
                    ArmBand09[" Armband"].Arm1["Right Arm"]:FindFirstChild("ProximityPrompt")

                DressPromptStep(Prompt)
            end)

            -- 2) Armor.Torso
            pcall(function()
                local C4 = Base:GetChildren()[4]
                local Armor =
                    C4 and
                    C4:FindFirstChild("Armor")

                local Torso =
                    Armor and
                    Armor:FindFirstChild("Torso")

                local Prompt =
                    Torso and
                    Torso:FindFirstChild("ProximityPrompt")

                DressPromptStep(Prompt)
            end)

            -- 3) GetChildren()[33]:GetChildren()[3].Giver
            pcall(function()
                local C33 = Base:GetChildren()[33]
                local Child3 =
                    C33 and
                    C33:GetChildren()[3]

                local Giver =
                    Child3 and
                    Child3:FindFirstChild("Giver")

                local Prompt =
                    Giver and
                    Giver:FindFirstChild("ProximityPrompt")

                DressPromptStep(Prompt)
            end)

            -- 4) Peaked caps by nonny
            pcall(function()
                local C27 = Base:GetChildren()[27]
                local Part =
                    C27 and
                    C27:FindFirstChild("Part")

                local Model1 =
                    Part and
                    Part:FindFirstChild("Model")

                local Model2 =
                    Model1 and
                    Model1:FindFirstChild("Model")

                local Cap =
                    Model2 and
                    Model2:FindFirstChild("Peaked caps by nonny")

                local Face =
                    Cap and
                    Cap:FindFirstChild("Face")

                local Prompt =
                    Face and
                    Face:FindFirstChild("ProximityPrompt")

                DressPromptStep(Prompt)
            end)

            -- 5) Red KG ร้อยเอก
            -- จุดนี้ต้องกด E จริง
            pcall(function()
                local C7 = Base:GetChildren()[7]
                local SecondModel =
                    C7 and
                    C7:GetChildren()[2]

                local RedKG =
                    SecondModel and
                    SecondModel:FindFirstChild("Red KG")

                local Prompt =
                    RedKG and
                    RedKG:FindFirstChild("ProximityPrompt")

                DressPromptStep(
                    Prompt,
                    CFrame.new(0, 0, -2),
                    true
                )
            end)

            Teleport(DRESS_RETURN_CF)
            Notify("Auto dress ร้อยเอก", "ใส่ชุดเสร็จแล้ว")
        end)
    end
})
end

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
    Values = {},
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

Tabs.Misc:AddButton({
    Title = "Load / Refresh Players",
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

    local Hum = Humanoid()
    if Hum and State.WalkSpeed ~= 16 then
        Hum.WalkSpeed = State.WalkSpeed
    end

    if Options.Noclip.Value then
        UpdateCharParts()
    end

    if Options.Fly.Value then StartFly() end
    UpdateMovementRuntime()
    UpdateUndergroundRuntime()
    if Options.NoGunZoom.Value then task.defer(ApplyNoGunZoom) end
    if Options.AntiVoid and Options.AntiVoid.Value then
        task.defer(StartAntiVoid)
    end

    if Options.AutoFishing and Options.AutoFishing.Value then
        task.delay(0.8, StartAutoFishing)
    end
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
    DisconnectAutoBaitConnection()
    StopFishingVisualLock()
    SetFishingHold(false)

    if State.RestoreFishingLowFX then
        State.RestoreFishingLowFX()
    end

    if FishingAction then
        pcall(function()
            FishingAction:FireServer("autoOff")
            FishingAction:FireServer("cancel")
        end)
    end

    State.AutoFarmLoopToken += 1
    State.FishingLoopToken += 1
    State.CraftLoopToken += 1
    State.ESPLoopToken += 1

    StopUndergroundRuntime()
    StopMovementRuntime()
    StopNoclipRuntime()
    StopESPRuntime()
    StopMoving()
    StopFly()
    StopAntiVoid()
    RestoreDeleteMap()

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

-- Cold start: ไม่เปิด toggle จาก config เก่าอัตโนมัติ
-- Config ยังอยู่ใน Settings และกด Load เองได้ตามปกติ
Notify("จำลองชีวิตทหารไทย [RCA]", "Cold Start - ไม่มีระบบหนักทำงานจนกว่าจะเปิด")
