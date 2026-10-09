-- 放置位置：StarterPlayer > StarterPlayerScripts 下的 LocalScript

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local HttpService = game:GetService("HttpService")
local TweenService = game:GetService("TweenService")
local Workspace = game:GetService("Workspace")
local UserInputService = game:GetService("UserInputService")

local player = Players.LocalPlayer

-- ================= 配置 =================
local BELLY_NAME = "CustomBelly"
local BASE_SIZE_X, BASE_SIZE_Y, BASE_SIZE_Z = 2, 2, 2
local BASE_SIZE_SCALE = 1
local BASE_OFFSET = Vector3.new(0, 0, -0.6)
local BASE_ROT = Vector3.new(0, 0, 0)

local SHAKE_DURATION = 1
local SHAKE_INTERVAL_MIN = 0.8
local SHAKE_INTERVAL_MAX = 2.0
local SHAKE_STRENGTH = 0.15

local NAVEL_SIZE_RATIO = 0.12
local NAVEL_DEPTH = 0.98

local WALK_SPEED_REF = 16
local WALK_SWING_X = 0.18
local WALK_SWING_Y = 0.10
local WALK_LEAN = 0.10
local WALK_SMOOTH = 12

local PULSE_SMOOTH = 3

local STRUGGLE_BULGE_COUNT = 3
local STRUGGLE_SWITCH_MIN = 0.4
local STRUGGLE_SWITCH_MAX = 1.2
local STRUGGLE_BULGE_SCALE = 1.35
local STRUGGLE_WOBBLE = 0.06
local STRUGGLE_SMOOTH = 8

local VIOLENT_BULGE_COUNT = 6
local VIOLENT_SWITCH_MIN = 0.1
local VIOLENT_SWITCH_MAX = 0.35
local VIOLENT_BULGE_SCALE = 2.0
local VIOLENT_WOBBLE = 0.18
local VIOLENT_AMP_MULT = 2.0
local VIOLENT_FART_MIN = 1.5
local VIOLENT_FART_MAX = 4.5

local CHEST_DEFAULT_OFFSET_X = 0.42
local CHEST_DEFAULT_OFFSET_Y = 0.50
local CHEST_DEFAULT_OFFSET_Z = -0.55
local BUTT_DEFAULT_OFFSET_X = 0.38
local BUTT_DEFAULT_OFFSET_Y = -0.35
local BUTT_DEFAULT_OFFSET_Z = 0.45

local JIGGLE_SMOOTH = 9
local JIGGLE_SPEED_FACTOR = 0.018
local BREATHE_AMP = 0.015
local BREATHE_SPEED = 2.0
local VIOLENT_JIGGLE_MULT = 1.5

local EAT_TOUCH_RANGE = 6
local DIGEST_TIME = 8
local DIGEST_SHRINK_TIME = 3
local DIGEST_KEEP_SCALE = 0.15
local FART_PARTICLES = 24
local FART_COLOR = Color3.fromRGB(100, 120, 20)

local EAT_FLY_TIME = 0.6
local EAT_SWALLOW_TIME = 0.55

-- 吃东西模式参数
local FOOD_MAX              = 100
local FOOD_DECAY_PER_SEC    = 2
local FOOD_START_LEVEL      = 30
local FOOD_BURGER_RESTORE   = 25
local FOOD_FART_INTERVAL    = 2.0
local FOOD_FART_JITTER      = 2.0
local FOOD_BURGER_COOLDOWN  = 10    -- 【新】汉堡 CD 10 秒
local FOOD_BURGER_FART_DELAY = 3    -- 【新】吃后 3 秒放屁

local SETTINGS_FILE = "belly_settings.json"

-- ================= 状态 =================
local belly, bellyWeld, navel, navelWeld, torso, rootPart, renderConn
local isShakingEnabled, shakeThread, isShakingNow = false, nil, false
local isStrugglingEnabled, struggleThread = false, nil
local struggleBulges = {}
local struggleWobble = Vector3.new(0, 0, 0)
local struggleWobbleTarget = Vector3.new(0, 0, 0)
local pulseThread, pulseScale, pulseTarget = nil, 1, 1
local walkOffset = Vector3.new(0, 0, 0)
local shakeOffset = Vector3.new(0, 0, 0)

local isViolentMode = false
local violentFartThread = nil

local isChestEnabled = false
local chestL, chestR, chestWeldL, chestWeldR
local chestJiggle = Vector3.new(0, 0, 0)

local isButtEnabled = false
local buttL, buttR, buttWeldL, buttWeldR
local buttJiggle = Vector3.new(0, 0, 0)

local isBellyHidden = false
local isBellyCollideEnabled = false
local isBellyPhysicalCollide = false
local bellySqueeze = Vector3.new(1, 1, 1)
local bellySqueezeTarget = Vector3.new(1, 1, 1)

local digestVisualScale = 1
local eatAppearAlpha = 1
local eatGrowScale = 1

-- 吃人
local eatState = nil
local eatModeEnabled = false
local hasEaten = false
local eatAnimationEnabled = true
local eatenPlayer = nil
local eatenOriginalTransparency = {}
local eatenOriginalPivot = nil
local eatGui = nil

-- 同步
local syncedPlayers = {}
local syncAllEnabled = false
local syncSelectedPlayerName = nil

-- 吃东西模式
local isFoodModeEnabled = false
local foodLevel = 0
local foodConn = nil
local foodGui = nil
local foodBarFill = nil
local foodLevelText = nil
local burgerSlot = nil
local lastFoodFartTime = 0
local burgerCooldown = 0            -- 【新】汉堡剩余 CD 秒
local burgerCdLabel = nil           -- 【新】CD 文字
local burgerFartThread = nil        -- 【新】吃后延迟放屁线程

local settings = {
    sizeScale = BASE_SIZE_SCALE,
    sizeX = BASE_SIZE_X, sizeY = BASE_SIZE_Y, sizeZ = BASE_SIZE_Z,
    offsetX = BASE_OFFSET.X, offsetY = BASE_OFFSET.Y, offsetZ = BASE_OFFSET.Z,
    rotX = BASE_ROT.X, rotY = BASE_ROT.Y, rotZ = BASE_ROT.Z,
    colorR = 255, colorG = 150, colorB = 100,
    whiteMix = 0,
    shakeAmp = 1.0, walkAmp = 1.0, pulseAmp = 0.08,
    pulseIntervalMin = 1.5, pulseIntervalMax = 3.5,
    struggleAmp = 1.0, squeezeAmp = 1.0,
    chestSize = 0.7, chestJiggle = 1.0,
    chestOffsetX = CHEST_DEFAULT_OFFSET_X,
    chestOffsetY = CHEST_DEFAULT_OFFSET_Y,
    chestOffsetZ = CHEST_DEFAULT_OFFSET_Z,
    chestColorR = 255, chestColorG = 200, chestColorB = 180,
    chestUseSkin = true,
    buttSize = 0.75, buttJiggle = 1.0,
    buttOffsetX = BUTT_DEFAULT_OFFSET_X,
    buttOffsetY = BUTT_DEFAULT_OFFSET_Y,
    buttOffsetZ = BUTT_DEFAULT_OFFSET_Z,
    buttColorR = 255, buttColorG = 200, buttColorB = 180,
    buttUseSkin = true,
    syncAllEnabled = false,
    syncSelectedPlayerName = "",
    foodModeEnabled = false,
}

-- ================= 持久化 =================
local function saveSettings()
    if not writefile then return end
    pcall(function() writefile(SETTINGS_FILE, HttpService:JSONEncode(settings)) end)
end

local function loadSettings()
    if not readfile then return end
    local ok, data = pcall(function()
        if isfile and isfile(SETTINGS_FILE) then return readfile(SETTINGS_FILE) end
        return nil
    end)
    if not ok or not data or data == "" then return end
    local ok2, decoded = pcall(function() return HttpService:JSONDecode(data) end)
    if not ok2 or type(decoded) ~= "table" then return end
    for k, v in pairs(decoded) do
        if settings[k] ~= nil and type(v) == type(settings[k]) then settings[k] = v end
    end
    syncAllEnabled = settings.syncAllEnabled
    if settings.syncSelectedPlayerName and settings.syncSelectedPlayerName ~= "" then
        syncSelectedPlayerName = settings.syncSelectedPlayerName
    end
end

local saveDebounce = false
local function queueSave()
    if saveDebounce then return end
    saveDebounce = true
    task.delay(1, function() saveDebounce = false saveSettings() end)
end

-- ================= 工具 =================
local function getTorso(character)
    return character:FindFirstChild("Torso") or character:FindFirstChild("UpperTorso")
end

local function destroyBelly()
    if renderConn then renderConn:Disconnect() renderConn = nil end
    if pulseThread then task.cancel(pulseThread) pulseThread = nil end
    if navel then navel:Destroy() navel = nil end
    if navelWeld then navelWeld:Destroy() navelWeld = nil end
    if belly then belly:Destroy() belly = nil end
    if bellyWeld then bellyWeld:Destroy() bellyWeld = nil end
end

local function finalColor()
    local base = Color3.fromRGB(settings.colorR, settings.colorG, settings.colorB)
    local w = math.clamp(settings.whiteMix, 0, 1)
    if w <= 0 then return base end
    return base:Lerp(Color3.new(1, 1, 1), w)
end

local function finalChestColor()
    if settings.chestUseSkin and torso then return torso.Color end
    return Color3.fromRGB(settings.chestColorR, settings.chestColorG, settings.chestColorB)
end

local function finalButtColor()
    if settings.buttUseSkin and torso then return torso.Color end
    return Color3.fromRGB(settings.buttColorR, settings.buttColorG, settings.buttColorB)
end

local function bellyVisibleTarget()
    local base
    if isBellyHidden then
        base = 1
    elseif eatModeEnabled and not hasEaten then
        base = 1
    else
        base = 0.05
    end
    if eatAppearAlpha < 1 then
        return 1 - (1 - base) * eatAppearAlpha
    end
    return base
end

local function remoteVisibleTarget()
    if isBellyHidden then return 1 end
    return 0.05
end

local function foodScale()
    if not isFoodModeEnabled then return 1 end
    return math.clamp(foodLevel / FOOD_MAX, 0, 1)
end

local function computeStruggleSizeFor(baseX, baseY, baseZ, bulges, amp)
    if not isStrugglingEnabled or not bulges or #bulges == 0 then
        return Vector3.new(baseX, baseY, baseZ)
    end
    local dirs = {
        Vector3.new(1,0,0), Vector3.new(-1,0,0),
        Vector3.new(0,1,0), Vector3.new(0,-1,0),
        Vector3.new(0,0,1), Vector3.new(0,0,-1),
        Vector3.new(0.7,0.7,0), Vector3.new(-0.7,0.7,0),
        Vector3.new(0,0.7,0.7), Vector3.new(0,-0.7,0.7),
        Vector3.new(0.7,0,0.7), Vector3.new(-0.7,0,0.7),
    }
    local maxX, maxY, maxZ = baseX, baseY, baseZ
    for _, d in ipairs(dirs) do
        local extra = 0
        for _, b in ipairs(bulges) do
            local influence = math.max(0, d.Unit:Dot(b.pos)) ^ 4
            extra = extra + (b.cur - 1) * influence
        end
        local m = 1 + extra * amp
        maxX = math.max(maxX, baseX * (1 + math.abs(d.X) * (m - 1)))
        maxY = math.max(maxY, baseY * (1 + math.abs(d.Y) * (m - 1)))
        maxZ = math.max(maxZ, baseZ * (1 + math.abs(d.Z) * (m - 1)))
    end
    return Vector3.new(maxX, maxY, maxZ)
end

local function computeStruggleSize(baseX, baseY, baseZ)
    local amp = settings.struggleAmp * (isViolentMode and VIOLENT_AMP_MULT or 1.0)
    return computeStruggleSizeFor(baseX, baseY, baseZ, struggleBulges, amp)
end

local function computeC0()
    local baseOffset = Vector3.new(settings.offsetX, settings.offsetY, settings.offsetZ)
    return CFrame.new(baseOffset + walkOffset + shakeOffset + struggleWobble)
        * CFrame.Angles(math.rad(settings.rotX), math.rad(settings.rotY), math.rad(settings.rotZ))
end

local function instComputeC0(inst)
    local baseOffset = Vector3.new(settings.offsetX, settings.offsetY, settings.offsetZ)
    return CFrame.new(baseOffset + inst.walkOffset + inst.shakeOffset + inst.struggleWobble)
        * CFrame.Angles(math.rad(settings.rotX), math.rad(settings.rotY), math.rad(settings.rotZ))
end

local function applyWeld()
    if not bellyWeld then return end
    bellyWeld.C0 = computeC0()
end

function applySettings()
    if not belly then return end
    local c = finalColor()
    belly.Color = c
    if navel then navel.Color = c end
    belly.Transparency = bellyVisibleTarget()
    if navel then navel.Transparency = bellyVisibleTarget() end
    applyWeld()
end

-- ================= 碰撞挤压 =================
local function computeSqueezeForPart(part, ownerCharacter)
    if not part or not part.Parent or not isBellyCollideEnabled then
        return Vector3.new(1, 1, 1)
    end
    local params = RaycastParams.new()
    params.FilterType = Enum.RaycastFilterType.Exclude
    if ownerCharacter then
        params.FilterDescendantsInstances = {ownerCharacter}
    end
    params.IgnoreWater = true

    local cf = part.CFrame
    local half = part.Size * 0.5
    local rx = math.max(half.X, 0.3)
    local ry = math.max(half.Y, 0.3)
    local rz = math.max(half.Z, 0.3)
    local PROBE, GAIN = 1.35, 2.4
    local origin = part.Position

    local function probe(dir, radius)
        local length = radius * PROBE
        local result = Workspace:Raycast(origin, dir * length, params)
        if not result then return 0 end
        local d = (result.Position - origin).Magnitude
        return math.clamp((1 - d / length) * GAIN, 0, 1)
    end

    local cx = math.max(probe(cf.RightVector, rx), probe(-cf.RightVector, rx))
    local cy = math.max(probe(cf.UpVector, ry),   probe(-cf.UpVector, ry))
    local cz = math.max(probe(cf.LookVector, rz), probe(-cf.LookVector, rz))

    local amp = settings.squeezeAmp
    cx = math.clamp(cx * amp, 0, 1)
    cy = math.clamp(cy * amp, 0, 1)
    cz = math.clamp(cz * amp, 0, 1)

    local SHRINK, BULGE = 0.6, 0.28
    return Vector3.new(
        1 - cx * SHRINK + (cy + cz) * 0.5 * BULGE,
        1 - cy * SHRINK + (cx + cz) * 0.5 * BULGE,
        1 - cz * SHRINK + (cx + cy) * 0.5 * BULGE
    )
end

local function computeBellySqueeze()
    return computeSqueezeForPart(belly, player.Character)
end

-- ================= 胸部 =================
local function destroyChests()
    if chestL then chestL:Destroy() chestL = nil end
    if chestR then chestR:Destroy() chestR = nil end
    if chestWeldL then chestWeldL:Destroy() chestWeldL = nil end
    if chestWeldR then chestWeldR:Destroy() chestWeldR = nil end
    chestJiggle = Vector3.new(0, 0, 0)
end

local function chestC0(xSign)
    return CFrame.new(settings.chestOffsetX * xSign, settings.chestOffsetY, settings.chestOffsetZ)
end

local function createChests(character)
    destroyChests()
    if not isChestEnabled then return end
    local t = getTorso(character)
    if not t then return end
    torso = t
    local s = settings.chestSize
    local color = finalChestColor()

    local function makeOne(name, xSign)
        local p = Instance.new("Part")
        p.Name = name
        p.Shape = Enum.PartType.Ball
        p.Size = Vector3.new(s, s, s)
        p.Color = color
        p.Material = Enum.Material.SmoothPlastic
        p.CanCollide, p.CanQuery, p.CanTouch = false, false, false
        p.Massless = true
        p.Parent = character
        local w = Instance.new("Weld")
        w.Name = name .. "Weld"
        w.Part0 = t
        w.Part1 = p
        w.C0 = chestC0(xSign)
        w.Parent = p
        return p, w
    end
    chestL, chestWeldL = makeOne("ChestL", 1)
    chestR, chestWeldR = makeOne("ChestR", -1)
end

local function updateChests()
    if chestL and chestR then
        local s = settings.chestSize
        chestL.Size = Vector3.new(s, s, s)
        chestR.Size = Vector3.new(s, s, s)
        local color = finalChestColor()
        chestL.Color = color
        chestR.Color = color
    end
    if chestWeldL then chestWeldL.C0 = chestC0(1) end
    if chestWeldR then chestWeldR.C0 = chestC0(-1) end
end

local function updateChestJiggle(dt)
    if not chestL or not chestWeldL or not rootPart then return end
    local vel = rootPart.AssemblyLinearVelocity
    local localVel = rootPart.CFrame:VectorToObjectSpace(vel)
    local violentMult = isViolentMode and VIOLENT_JIGGLE_MULT or 1.0
    local jiggleAmp = settings.chestJiggle * JIGGLE_SPEED_FACTOR * violentMult
    local breathe = math.sin(tick() * BREATHE_SPEED) * BREATHE_AMP * settings.chestJiggle
    local target = Vector3.new(
        -localVel.X * jiggleAmp,
        -localVel.Y * jiggleAmp * 1.5 + breathe,
        -localVel.Z * jiggleAmp
    )
    chestJiggle = chestJiggle:Lerp(target, math.clamp(dt * JIGGLE_SMOOTH, 0, 1))
    local bx = settings.chestOffsetX + chestJiggle.X
    local by = settings.chestOffsetY + chestJiggle.Y
    local bz = settings.chestOffsetZ + chestJiggle.Z
    chestWeldL.C0 = CFrame.new(bx, by, bz)
    if chestWeldR then chestWeldR.C0 = CFrame.new(-bx, by, bz) end
end

-- ================= 屁股 =================
local function destroyButts()
    if buttL then buttL:Destroy() buttL = nil end
    if buttR then buttR:Destroy() buttR = nil end
    if buttWeldL then buttWeldL:Destroy() buttWeldL = nil end
    if buttWeldR then buttWeldR:Destroy() buttWeldR = nil end
    buttJiggle = Vector3.new(0, 0, 0)
end

local function buttC0(xSign)
    return CFrame.new(settings.buttOffsetX * xSign, settings.buttOffsetY, settings.buttOffsetZ)
end

local function createButts(character)
    destroyButts()
    if not isButtEnabled then return end
    local t = getTorso(character)
    if not t then return end
    torso = t
    local s = settings.buttSize
    local color = finalButtColor()

    local function makeOne(name, xSign)
        local p = Instance.new("Part")
        p.Name = name
        p.Shape = Enum.PartType.Ball
        p.Size = Vector3.new(s, s, s)
        p.Color = color
        p.Material = Enum.Material.SmoothPlastic
        p.CanCollide, p.CanQuery, p.CanTouch = false, false, false
        p.Massless = true
        p.Parent = character
        local w = Instance.new("Weld")
        w.Name = name .. "Weld"
        w.Part0 = t
        w.Part1 = p
        w.C0 = buttC0(xSign)
        w.Parent = p
        return p, w
    end
    buttL, buttWeldL = makeOne("ButtL", 1)
    buttR, buttWeldR = makeOne("ButtR", -1)
end

local function updateButts()
    if buttL and buttR then
        local s = settings.buttSize
        buttL.Size = Vector3.new(s, s, s)
        buttR.Size = Vector3.new(s, s, s)
        local color = finalButtColor()
        buttL.Color = color
        buttR.Color = color
    end
    if buttWeldL then buttWeldL.C0 = buttC0(1) end
    if buttWeldR then buttWeldR.C0 = buttC0(-1) end
end

local function updateButtJiggle(dt)
    if not buttL or not buttWeldL or not rootPart then return end
    local vel = rootPart.AssemblyLinearVelocity
    local localVel = rootPart.CFrame:VectorToObjectSpace(vel)
    local violentMult = isViolentMode and VIOLENT_JIGGLE_MULT or 1.0
    local jiggleAmp = settings.buttJiggle * JIGGLE_SPEED_FACTOR * violentMult
    local breathe = math.sin(tick() * BREATHE_SPEED) * BREATHE_AMP * settings.buttJiggle
    local target = Vector3.new(
        -localVel.X * jiggleAmp,
        -localVel.Y * jiggleAmp * 1.5 + breathe,
        -localVel.Z * jiggleAmp
    )
    buttJiggle = buttJiggle:Lerp(target, math.clamp(dt * JIGGLE_SMOOTH, 0, 1))
    local bx = settings.buttOffsetX + buttJiggle.X
    local by = settings.buttOffsetY + buttJiggle.Y
    local bz = settings.buttOffsetZ + buttJiggle.Z
    buttWeldL.C0 = CFrame.new(bx, by, bz)
    if buttWeldR then buttWeldR.C0 = CFrame.new(-bx, by, bz) end
end

-- ================= 找最近玩家 =================
local function getNearestPlayer()
    if not rootPart then return nil end
    local myPos = rootPart.Position
    local bellyRadius = 0
    if belly and belly.Parent and belly.Transparency < 1 then
        bellyRadius = math.max(belly.Size.X, belly.Size.Y, belly.Size.Z) * 0.5
    end
    local range = math.max(EAT_TOUCH_RANGE, bellyRadius + 1.5)
    local best, bestDist = nil, range
    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= player and p.Character then
            local hrp = p.Character:FindFirstChild("HumanoidRootPart")
            local hum = p.Character:FindFirstChildOfClass("Humanoid")
            if hrp and hum and hum.Health > 0 then
                local d = (hrp.Position - myPos).Magnitude
                if d < bestDist then best, bestDist = p, d end
            end
        end
    end
    return best
end

-- ================= 同步 =================
local function makeRemotePart(name, shape, size, color, character, transparency)
    local p = Instance.new("Part")
    p.Name = name
    p.Shape = shape
    p.Size = size
    p.Color = color
    p.Material = Enum.Material.SmoothPlastic
    p.CanCollide, p.CanQuery, p.CanTouch = false, false, false
    p.Massless = true
    p.Transparency = transparency or 0.05
    p.Parent = character
    return p
end

local function createRemoteBelly(plr, character)
    if syncedPlayers[plr] then return end
    local t = getTorso(character)
    if not t then return end

    local inst = {
        plr = plr, character = character, torso = t,
        rootPart = character:FindFirstChild("HumanoidRootPart"),
        walkOffset = Vector3.new(0, 0, 0),
        shakeOffset = Vector3.new(0, 0, 0),
        struggleWobble = Vector3.new(0, 0, 0),
        struggleWobbleTarget = Vector3.new(0, 0, 0),
        struggleBulges = {},
        bellySqueeze = Vector3.new(1, 1, 1),
        bellySqueezeTarget = Vector3.new(1, 1, 1),
        pulseScale = 1, pulseTarget = 1,
        eatAppearAlpha = 1, eatGrowScale = 1, digestVisualScale = 1,
        nextStruggleSwitch = tick(), nextPulse = tick(),
        isRemote = true,
    }

    local skinColor = t.Color
    local s = settings.sizeScale

    inst.belly = makeRemotePart(BELLY_NAME, Enum.PartType.Ball,
        Vector3.new(settings.sizeX * s, settings.sizeY * s, settings.sizeZ * s),
        skinColor, character, 0.05)

    local bw = Instance.new("Weld")
    bw.Name = "BellyWeld"
    bw.Part0 = t
    bw.Part1 = inst.belly
    bw.C0 = instComputeC0(inst)
    bw.Parent = inst.belly
    inst.bellyWeld = bw

    local minSize = math.min(settings.sizeX, settings.sizeY, settings.sizeZ) * s
    local ns = minSize * NAVEL_SIZE_RATIO
    local r = (settings.sizeZ * s) / 2

    inst.navel = makeRemotePart("BellyNavel", Enum.PartType.Ball,
        Vector3.new(ns, ns, ns), skinColor, character, 0.05)

    local nw = Instance.new("Weld")
    nw.Name = "NavelWeld"
    nw.Part0 = inst.belly
    nw.Part1 = inst.navel
    nw.C0 = CFrame.new(0, 0, -r * NAVEL_DEPTH)
    nw.Parent = inst.navel
    inst.navelWeld = nw

    if isChestEnabled then
        local cs = settings.chestSize
        local function makeChest(nm, xSign)
            local p = makeRemotePart(nm, Enum.PartType.Ball,
                Vector3.new(cs, cs, cs), skinColor, character, 0)
            local w = Instance.new("Weld")
            w.Name = nm .. "Weld"
            w.Part0 = t
            w.Part1 = p
            w.C0 = chestC0(xSign)
            w.Parent = p
            return p, w
        end
        inst.chestL, inst.chestWeldL = makeChest("ChestL", 1)
        inst.chestR, inst.chestWeldR = makeChest("ChestR", -1)
        inst.chestJiggle = Vector3.new(0, 0, 0)
    end

    if isButtEnabled then
        local bs = settings.buttSize
        local function makeButt(nm, xSign)
            local p = makeRemotePart(nm, Enum.PartType.Ball,
                Vector3.new(bs, bs, bs), skinColor, character, 0)
            local w = Instance.new("Weld")
            w.Name = nm .. "Weld"
            w.Part0 = t
            w.Part1 = p
            w.C0 = buttC0(xSign)
            w.Parent = p
            return p, w
        end
        inst.buttL, inst.buttWeldL = makeButt("ButtL", 1)
        inst.buttR, inst.buttWeldR = makeButt("ButtR", -1)
        inst.buttJiggle = Vector3.new(0, 0, 0)
    end

    syncedPlayers[plr] = inst
end

local function destroyRemoteBelly(plr)
    local inst = syncedPlayers[plr]
    if not inst then return end
    for _, key in ipairs({
        "belly", "bellyWeld", "navel", "navelWeld",
        "chestL", "chestR", "chestWeldL", "chestWeldR",
        "buttL", "buttR", "buttWeldL", "buttWeldR",
    }) do
        local obj = inst[key]
        if obj then pcall(function() obj:Destroy() end) end
    end
    syncedPlayers[plr] = nil
end

local function shouldSyncPlayer(plr)
    if plr == player then return false end
    if syncAllEnabled then return true end
    if syncSelectedPlayerName and syncSelectedPlayerName ~= "" then
        return plr.Name == syncSelectedPlayerName
    end
    return false
end

local function refreshSyncedPlayers()
    for plr, _ in pairs(syncedPlayers) do
        if not shouldSyncPlayer(plr) or not plr.Character or not plr.Parent then
            destroyRemoteBelly(plr)
        end
    end
    for _, plr in ipairs(Players:GetPlayers()) do
        if shouldSyncPlayer(plr) and plr.Character then
            local t = getTorso(plr.Character)
            if t and not syncedPlayers[plr] then
                createRemoteBelly(plr, plr.Character)
            end
        end
    end
end

local function updateRemoteBelly(inst, dt)
    if not inst.bellyWeld or not inst.belly or not inst.rootPart then return end
    if not inst.belly.Parent then return end
    local rp = inst.rootPart

    local vel = rp.AssemblyLinearVelocity
    local horizontal = Vector3.new(vel.X, 0, vel.Z)
    local speed = horizontal.Magnitude
    local target = Vector3.new(0, 0, 0)
    local walkAmp = settings.walkAmp
    if speed > 0.5 and walkAmp > 0 then
        local intensity = math.clamp(speed / WALK_SPEED_REF, 0, 1.6)
        local t = tick()
        local localVel = rp.CFrame:VectorToObjectSpace(horizontal)
        target = Vector3.new(
            math.sin(t * 10) * WALK_SWING_X * intensity * walkAmp,
            -math.abs(math.sin(t * 20)) * WALK_SWING_Y * intensity * walkAmp,
            (localVel.Z / WALK_SPEED_REF) * WALK_LEAN * walkAmp
        )
    end
    inst.walkOffset = inst.walkOffset:Lerp(target, math.clamp(dt * WALK_SMOOTH, 0, 1))

    inst.pulseScale = inst.pulseScale + (inst.pulseTarget - inst.pulseScale) * math.clamp(dt * PULSE_SMOOTH, 0, 1)
    if tick() >= (inst.nextPulse or 0) then
        local lo = math.min(settings.pulseIntervalMin, settings.pulseIntervalMax)
        local hi = math.max(settings.pulseIntervalMin, settings.pulseIntervalMax)
        inst.nextPulse = tick() + lo + math.random() * (hi - lo)
        inst.pulseTarget = 1 + (math.random() * 2 - 1) * settings.pulseAmp
    end

    if isStrugglingEnabled then
        if tick() >= (inst.nextStruggleSwitch or 0) then
            local count = isViolentMode and VIOLENT_BULGE_COUNT or STRUGGLE_BULGE_COUNT
            while #inst.struggleBulges < count do
                table.insert(inst.struggleBulges, { pos = randomUnitDir(), target = randomBulgeTarget(), cur = 1 })
            end
            while #inst.struggleBulges > count do
                table.remove(inst.struggleBulges)
            end
            local b = inst.struggleBulges[math.random(1, #inst.struggleBulges)]
            if b then
                b.pos = randomUnitDir()
                b.target = randomBulgeTarget()
            end
            local wob = isViolentMode and VIOLENT_WOBBLE or STRUGGLE_WOBBLE
            inst.struggleWobbleTarget = Vector3.new(
                (math.random() - 0.5) * wob * 2,
                (math.random() - 0.5) * wob * 2,
                (math.random() - 0.5) * wob * 2
            ) * settings.struggleAmp
            local lo = isViolentMode and VIOLENT_SWITCH_MIN or STRUGGLE_SWITCH_MIN
            local hi = isViolentMode and VIOLENT_SWITCH_MAX or STRUGGLE_SWITCH_MAX
            inst.nextStruggleSwitch = tick() + lo + math.random() * (hi - lo)
        end
        local sAlpha = math.clamp(dt * STRUGGLE_SMOOTH, 0, 1)
        for _, b in ipairs(inst.struggleBulges) do
            b.cur = b.cur + (b.target - b.cur) * sAlpha
        end
        inst.struggleWobble = inst.struggleWobble:Lerp(inst.struggleWobbleTarget, sAlpha)
    else
        for _, b in ipairs(inst.struggleBulges) do b.target = 1; b.cur = 1 end
        inst.struggleWobble = inst.struggleWobble:Lerp(Vector3.new(0,0,0), math.clamp(dt * 4, 0, 1))
    end

    local totalScale = settings.sizeScale * inst.pulseScale * inst.digestVisualScale * inst.eatGrowScale
    local finalSize = computeStruggleSizeFor(
        settings.sizeX * totalScale,
        settings.sizeY * totalScale,
        settings.sizeZ * totalScale,
        inst.struggleBulges,
        settings.struggleAmp * (isViolentMode and VIOLENT_AMP_MULT or 1.0)
    )

    if isBellyCollideEnabled then
        inst.bellySqueezeTarget = computeSqueezeForPart(inst.belly, inst.character)
    else
        inst.bellySqueezeTarget = Vector3.new(1, 1, 1)
    end
    inst.bellySqueeze = inst.bellySqueeze:Lerp(inst.bellySqueezeTarget, math.clamp(dt * 10, 0, 1))
    finalSize = Vector3.new(
        finalSize.X * inst.bellySqueeze.X,
        finalSize.Y * inst.bellySqueeze.Y,
        finalSize.Z * inst.bellySqueeze.Z
    )

    inst.belly.Size = finalSize
    inst.belly.CanCollide = isBellyPhysicalCollide

    if inst.navel then
        local minSize = math.min(finalSize.X, finalSize.Y, finalSize.Z)
        local ns = minSize * NAVEL_SIZE_RATIO
        local r = finalSize.Z / 2
        inst.navel.Size = Vector3.new(ns, ns, ns)
        if inst.navelWeld then
            inst.navelWeld.C0 = CFrame.new(0, 0, -r * NAVEL_DEPTH)
        end
    end

    inst.bellyWeld.C0 = instComputeC0(inst)

    local vt = remoteVisibleTarget()
    if inst.belly.Transparency ~= vt then inst.belly.Transparency = vt end
    if inst.navel and inst.navel.Transparency ~= vt then inst.navel.Transparency = vt end

    if inst.torso and inst.torso.Parent then
        local sc = inst.torso.Color
        if inst.belly.Color ~= sc then inst.belly.Color = sc end
        if inst.navel and inst.navel.Color ~= sc then inst.navel.Color = sc end
    end

    if inst.chestL and inst.chestWeldL then
        local cs = settings.chestSize
        if inst.chestL.Size.X ~= cs then
            inst.chestL.Size = Vector3.new(cs, cs, cs)
            inst.chestR.Size = Vector3.new(cs, cs, cs)
        end
        local lv = rp.CFrame:VectorToObjectSpace(rp.AssemblyLinearVelocity)
        local violentMult = isViolentMode and VIOLENT_JIGGLE_MULT or 1.0
        local jiggleAmp = settings.chestJiggle * JIGGLE_SPEED_FACTOR * violentMult
        local breathe = math.sin(tick() * BREATHE_SPEED) * BREATHE_AMP * settings.chestJiggle
        local tgt = Vector3.new(-lv.X * jiggleAmp, -lv.Y * jiggleAmp * 1.5 + breathe, -lv.Z * jiggleAmp)
        inst.chestJiggle = inst.chestJiggle:Lerp(tgt, math.clamp(dt * JIGGLE_SMOOTH, 0, 1))
        local bx = settings.chestOffsetX + inst.chestJiggle.X
        local by = settings.chestOffsetY + inst.chestJiggle.Y
        local bz = settings.chestOffsetZ + inst.chestJiggle.Z
        inst.chestWeldL.C0 = CFrame.new(bx, by, bz)
        if inst.chestWeldR then inst.chestWeldR.C0 = CFrame.new(-bx, by, bz) end
    end

    if inst.buttL and inst.buttWeldL then
        local bs = settings.buttSize
        if inst.buttL.Size.X ~= bs then
            inst.buttL.Size = Vector3.new(bs, bs, bs)
            inst.buttR.Size = Vector3.new(bs, bs, bs)
        end
        local lv3 = rp.CFrame:VectorToObjectSpace(rp.AssemblyLinearVelocity)
        local violentMult = isViolentMode and VIOLENT_JIGGLE_MULT or 1.0
        local jiggleAmp = settings.buttJiggle * JIGGLE_SPEED_FACTOR * violentMult
        local breathe = math.sin(tick() * BREATHE_SPEED) * BREATHE_AMP * settings.buttJiggle
        local tgt = Vector3.new(-lv3.X * jiggleAmp, -lv3.Y * jiggleAmp * 1.5 + breathe, -lv3.Z * jiggleAmp)
        inst.buttJiggle = inst.buttJiggle:Lerp(tgt, math.clamp(dt * JIGGLE_SMOOTH, 0, 1))
        local bx = settings.buttOffsetX + inst.buttJiggle.X
        local by = settings.buttOffsetY + inst.buttJiggle.Y
        local bz = settings.buttOffsetZ + inst.buttJiggle.Z
        inst.buttWeldL.C0 = CFrame.new(bx, by, bz)
        if inst.buttWeldR then inst.buttWeldR.C0 = CFrame.new(-bx, by, bz) end
    end
end

-- ================= 渲染循环 =================
local function startRenderLoop()
    if renderConn then renderConn:Disconnect() end
    renderConn = RunService.RenderStepped:Connect(function(dt)
        if not bellyWeld or not belly or not rootPart then
            for _, inst in pairs(syncedPlayers) do
                updateRemoteBelly(inst, dt)
            end
            return
        end

        local vel = rootPart.AssemblyLinearVelocity
        local horizontal = Vector3.new(vel.X, 0, vel.Z)
        local speed = horizontal.Magnitude
        local target = Vector3.new(0, 0, 0)
        local walkAmp = settings.walkAmp

        if speed > 0.5 and walkAmp > 0 then
            local intensity = math.clamp(speed / WALK_SPEED_REF, 0, 1.6)
            local t = tick()
            local localVel = rootPart.CFrame:VectorToObjectSpace(horizontal)
            target = Vector3.new(
                math.sin(t * 10) * WALK_SWING_X * intensity * walkAmp,
                -math.abs(math.sin(t * 20)) * WALK_SWING_Y * intensity * walkAmp,
                (localVel.Z / WALK_SPEED_REF) * WALK_LEAN * walkAmp
            )
        end
        walkOffset = walkOffset:Lerp(target, math.clamp(dt * WALK_SMOOTH, 0, 1))

        pulseScale = pulseScale + (pulseTarget - pulseScale) * math.clamp(dt * PULSE_SMOOTH, 0, 1)

        if isStrugglingEnabled then
            local sAlpha = math.clamp(dt * STRUGGLE_SMOOTH, 0, 1)
            for _, b in ipairs(struggleBulges) do
                b.cur = b.cur + (b.target - b.cur) * sAlpha
            end
            struggleWobble = struggleWobble:Lerp(struggleWobbleTarget, sAlpha)
        end

        local totalScale = settings.sizeScale * pulseScale * digestVisualScale * eatGrowScale * foodScale()
        local finalSize = computeStruggleSize(
            settings.sizeX * totalScale,
            settings.sizeY * totalScale,
            settings.sizeZ * totalScale
        )

        if isBellyCollideEnabled then
            bellySqueezeTarget = computeBellySqueeze()
        else
            bellySqueezeTarget = Vector3.new(1, 1, 1)
        end
        bellySqueeze = bellySqueeze:Lerp(bellySqueezeTarget, math.clamp(dt * 10, 0, 1))
        finalSize = Vector3.new(
            finalSize.X * bellySqueeze.X,
            finalSize.Y * bellySqueeze.Y,
            finalSize.Z * bellySqueeze.Z
        )

        belly.Size = finalSize
        belly.CanCollide = isBellyPhysicalCollide

        if navel then
            local minSize = math.min(finalSize.X, finalSize.Y, finalSize.Z)
            local ns = minSize * NAVEL_SIZE_RATIO
            local r = finalSize.Z / 2
            navel.Size = Vector3.new(ns, ns, ns)
            if navelWeld then navelWeld.C0 = CFrame.new(0, 0, -r * NAVEL_DEPTH) end
        end

        bellyWeld.C0 = computeC0()

        local vt = bellyVisibleTarget()
        if belly.Transparency ~= vt then belly.Transparency = vt end
        if navel and navel.Transparency ~= vt then navel.Transparency = vt end

        if isChestEnabled then updateChestJiggle(dt) end
        if isButtEnabled then updateButtJiggle(dt) end

        if eatState == "hunting" and not isBellyHidden then
            local tgt = getNearestPlayer()
            if tgt then
                eatState = "struggling"
                task.spawn(function()
                    local ok, err = pcall(doEat, tgt)
                    if not ok then
                        warn("[Belly] 吃人流程出错: " .. tostring(err))
                        if eatState == "struggling" then
                            stopStruggleLoop()
                            if eatenPlayer and eatenPlayer.Character then
                                setCharacterTransparency(eatenPlayer.Character, nil)
                            end
                            eatenPlayer = nil
                            if eatModeEnabled then
                                eatState = "hunting"
                            else
                                eatState = nil
                            end
                            eatAppearAlpha = 1
                            eatGrowScale = 1
                            if belly then belly.Transparency = bellyVisibleTarget() end
                            if navel then navel.Transparency = bellyVisibleTarget() end
                        end
                    end
                end)
            end
        end

        for _, inst in pairs(syncedPlayers) do
            updateRemoteBelly(inst, dt)
        end
    end)
end

-- ================= 脉动 =================
local function startPulseLoop()
    if pulseThread then task.cancel(pulseThread) end
    pulseThread = task.spawn(function()
        while belly do
            local lo = math.min(settings.pulseIntervalMin, settings.pulseIntervalMax)
            local hi = math.max(settings.pulseIntervalMin, settings.pulseIntervalMax)
            task.wait(lo + math.random() * (hi - lo))
            if not belly then break end
            pulseTarget = 1 + (math.random() * 2 - 1) * settings.pulseAmp
        end
        pulseThread = nil
    end)
end

-- ================= 挣扎 =================
local function randomUnitDir()
    local theta = math.random() * math.pi * 2
    local z = math.random() * 2 - 1
    local r = math.sqrt(math.max(0, 1 - z * z))
    return Vector3.new(r * math.cos(theta), r * math.sin(theta), z)
end

local function randomBulgeTarget()
    local maxS = isViolentMode and VIOLENT_BULGE_SCALE or STRUGGLE_BULGE_SCALE
    return 1 + math.random() * (maxS - 1)
end

local function ensureBulges()
    local target = isViolentMode and VIOLENT_BULGE_COUNT or STRUGGLE_BULGE_COUNT
    while #struggleBulges < target do
        table.insert(struggleBulges, { pos = randomUnitDir(), target = randomBulgeTarget(), cur = 1 })
    end
end

local function startStruggleLoop()
    if struggleThread then return end
    struggleThread = task.spawn(function()
        while isStrugglingEnabled do
            ensureBulges()
            local lo = isViolentMode and VIOLENT_SWITCH_MIN or STRUGGLE_SWITCH_MIN
            local hi = isViolentMode and VIOLENT_SWITCH_MAX or STRUGGLE_SWITCH_MAX
            task.wait(lo + math.random() * (hi - lo))
            if not isStrugglingEnabled then break end
            local b = struggleBulges[math.random(1, #struggleBulges)]
            b.pos = randomUnitDir()
            b.target = randomBulgeTarget()
            local wob = isViolentMode and VIOLENT_WOBBLE or STRUGGLE_WOBBLE
            struggleWobbleTarget = Vector3.new(
                (math.random() - 0.5) * wob * 2,
                (math.random() - 0.5) * wob * 2,
                (math.random() - 0.5) * wob * 2
            ) * settings.struggleAmp
        end
        struggleThread = nil
    end)
end

local function stopStruggleLoop()
    isStrugglingEnabled = false
    if struggleThread then task.cancel(struggleThread) struggleThread = nil end
    for _, b in ipairs(struggleBulges) do b.target = 1 b.cur = 1 end
    struggleWobble = Vector3.new(0, 0, 0)
    struggleWobbleTarget = Vector3.new(0, 0, 0)
    struggleBulges = {}
end

-- ================= 创建肚子 =================
local function createBelly(character)
    destroyBelly()
    torso = getTorso(character)
    if not torso then return end
    rootPart = character:FindFirstChild("HumanoidRootPart")

    walkOffset = Vector3.new(0, 0, 0)
    shakeOffset = Vector3.new(0, 0, 0)
    pulseScale, pulseTarget = 1, 1
    struggleWobble = Vector3.new(0, 0, 0)
    struggleWobbleTarget = Vector3.new(0, 0, 0)
    struggleBulges = {}
    bellySqueeze = Vector3.new(1, 1, 1)
    bellySqueezeTarget = Vector3.new(1, 1, 1)
    digestVisualScale = 1
    eatAppearAlpha = 1
    eatGrowScale = 1

    local s = settings.sizeScale
    local c = finalColor()

    belly = Instance.new("Part")
    belly.Name = BELLY_NAME
    belly.Shape = Enum.PartType.Ball
    belly.Size = Vector3.new(settings.sizeX * s, settings.sizeY * s, settings.sizeZ * s)
    belly.Color = c
    belly.Material = Enum.Material.SmoothPlastic
    belly.CanCollide = isBellyPhysicalCollide
    belly.CanQuery, belly.CanTouch = false, false
    belly.Massless = true
    belly.Transparency = bellyVisibleTarget()
    belly.Parent = character

    bellyWeld = Instance.new("Weld")
    bellyWeld.Name = "BellyWeld"
    bellyWeld.Part0 = torso
    bellyWeld.Part1 = belly
    bellyWeld.C0 = computeC0()
    bellyWeld.Parent = belly

    local minSize = math.min(settings.sizeX, settings.sizeY, settings.sizeZ) * s
    local ns = minSize * NAVEL_SIZE_RATIO
    local r = (settings.sizeZ * s) / 2

    navel = Instance.new("Part")
    navel.Name = "BellyNavel"
    navel.Shape = Enum.PartType.Ball
    navel.Size = Vector3.new(ns, ns, ns)
    navel.Color = c
    navel.Material = Enum.Material.SmoothPlastic
    navel.CanCollide, navel.CanQuery, navel.CanTouch = false, false, false
    navel.Massless = true
    navel.Transparency = bellyVisibleTarget()
    navel.Parent = character

    navelWeld = Instance.new("Weld")
    navelWeld.Name = "NavelWeld"
    navelWeld.Part0 = belly
    navelWeld.Part1 = navel
    navelWeld.C0 = CFrame.new(0, 0, -r * NAVEL_DEPTH)
    navelWeld.Parent = navel

    applySettings()
    startRenderLoop()
    startPulseLoop()
    if isStrugglingEnabled then startStruggleLoop() end
end

-- ================= 抖动 =================
local function shakeOnce()
    if not belly or not bellyWeld or isShakingNow then return end
    isShakingNow = true
    local strength = SHAKE_STRENGTH * settings.shakeAmp
    local steps = 8
    local stepTime = SHAKE_DURATION / steps
    for i = 1, steps do
        shakeOffset = Vector3.new(
            (math.random() - 0.5) * strength,
            (math.random() - 0.5) * strength,
            (math.random() - 0.5) * strength
        )
        applyWeld()
        task.wait(stepTime)
    end
    shakeOffset = Vector3.new(0, 0, 0)
    applyWeld()
    isShakingNow = false
end

local function startShakeLoop()
    if shakeThread then return end
    shakeThread = task.spawn(function()
        while isShakingEnabled do
            task.wait(math.random(math.floor(SHAKE_INTERVAL_MIN*10), math.floor(SHAKE_INTERVAL_MAX*10)) / 10)
            if not isShakingEnabled then break end
            if belly and bellyWeld then shakeOnce() end
        end
        shakeThread = nil
    end)
end

local function stopShakeLoop()
    isShakingEnabled = false
    if shakeThread then task.cancel(shakeThread) shakeThread = nil end
    shakeOffset = Vector3.new(0, 0, 0)
    applyWeld()
    isShakingNow = false
end

-- ================= 吃人 =================
local function setCharacterTransparency(character, transparency)
    if not character then return end
    for _, part in ipairs(character:GetDescendants()) do
        if part:IsA("BasePart") then
            if transparency == nil then
                if eatenOriginalTransparency[part] ~= nil then
                    part.Transparency = eatenOriginalTransparency[part]
                    part.CanCollide = true
                end
            else
                if eatenOriginalTransparency[part] == nil then
                    eatenOriginalTransparency[part] = part.Transparency
                end
                part.Transparency = transparency
                part.CanCollide = false
            end
        elseif part:IsA("Decal") or part:IsA("Texture") then
            part.Transparency = transparency or 0
        end
    end
end

local function hideEatenPlayer(p)
    if not p or not p.Character then return end
    eatenOriginalTransparency = {}
    eatenOriginalPivot = p.Character:GetPivot()
    setCharacterTransparency(p.Character, 0)
    p.Character:PivotTo(CFrame.new(0, -500, 0))
end

local function playEatAnimation(p)
    if not p or not p.Character then return end
    local char = p.Character
    local hrp = char:FindFirstChild("HumanoidRootPart")
    local hum = char:FindFirstChildOfClass("Humanoid")
    if not hrp then
        hideEatenPlayer(p)
        return
    end

    eatenOriginalTransparency = {}
    eatenOriginalPivot = char:GetPivot()

    if hum then
        hum.WalkSpeed = 0
        hum.JumpPower = 0
        hum.PlatformStand = true
    end
    for _, part in ipairs(char:GetDescendants()) do
        if part:IsA("BasePart") then
            if eatenOriginalTransparency[part] == nil then
                eatenOriginalTransparency[part] = part.Transparency
            end
            part.Anchored = true
            part.CanCollide = false
        end
    end

    local bellyPos, mouthPos
    if belly and belly.Parent then
        bellyPos = belly.Position
        mouthPos = bellyPos + belly.CFrame.LookVector * (belly.Size.Z * 0.55)
    else
        bellyPos = hrp.Position
        mouthPos = hrp.Position
    end

    local startPos = hrp.Position
    local dist = (startPos - mouthPos).Magnitude
    local mid = (startPos + mouthPos) * 0.5
    local control = mid + Vector3.new(0, math.max(2.5, dist * 0.35), 0)

    local function bezier(t)
        local u = 1 - t
        return u * u * startPos + 2 * u * t * control + t * t * mouthPos
    end

    local flySteps = 30
    for i = 1, flySteps do
        local t = i / flySteps
        local ease = 1 - (1 - t) ^ 2
        local pos = bezier(ease)
        local spin = CFrame.Angles(math.rad(t * 540), math.rad(t * 360), math.rad(t * 200))
        local face = CFrame.lookAt(pos, mouthPos)
        char:PivotTo(face * spin)
        task.wait(EAT_FLY_TIME / flySteps)
    end

    local swallowSteps = 26
    for i = 1, swallowSteps do
        local t = i / swallowSteps
        local ease = t * t
        local pos = mouthPos:Lerp(bellyPos, ease)
        char:PivotTo(CFrame.new(pos))
        local fade = math.clamp(t * 1.25, 0, 1)
        for _, part in ipairs(char:GetDescendants()) do
            if part:IsA("BasePart") then
                part.Transparency = fade
            elseif part:IsA("Decal") or part:IsA("Texture") then
                part.Transparency = fade
            end
        end
        eatAppearAlpha = math.clamp(t * 1.6, 0, 1)
        eatGrowScale   = 0.45 + 0.7 * math.clamp(t * 1.4, 0, 1)
        if belly then belly.Transparency = bellyVisibleTarget() end
        if navel then navel.Transparency = bellyVisibleTarget() end
        task.wait(EAT_SWALLOW_TIME / swallowSteps)
    end

    for i = 1, 8 do
        local t = i / 8
        eatAppearAlpha = 1
        eatGrowScale = 1.15 - 0.15 * t
        task.wait(0.02)
    end

    eatAppearAlpha = 1
    eatGrowScale = 1
    char:PivotTo(CFrame.new(0, -500, 0))
end

local function restoreEatenPlayer()
    if eatenPlayer and eatenPlayer.Character then
        setCharacterTransparency(eatenPlayer.Character, nil)
        for _, part in ipairs(eatenPlayer.Character:GetDescendants()) do
            if part:IsA("BasePart") then part.Anchored = false end
        end
        local hum = eatenPlayer.Character:FindFirstChildOfClass("Humanoid")
        if hum then hum.PlatformStand = false end
        if eatenOriginalPivot then
            eatenPlayer.Character:PivotTo(eatenOriginalPivot)
        end
    end
    eatenOriginalTransparency = {}
    eatenOriginalPivot = nil
end

local function showDigestButton()
    if not eatGui then return end
    local old = eatGui:FindFirstChild("DigestBtn")
    if old then old:Destroy() end
    local btn = Instance.new("TextButton")
    btn.Name = "DigestBtn"
    btn.Size = UDim2.new(0, 200, 0, 60)
    btn.Position = UDim2.new(0.5, -100, 0.7, 0)
    btn.BackgroundColor3 = Color3.fromRGB(120, 200, 80)
    btn.BorderSizePixel = 0
    btn.Text = "消化"
    btn.TextColor3 = Color3.new(1, 1, 1)
    btn.Font = Enum.Font.GothamBold
    btn.TextSize = 20
    btn.Parent = eatGui
    local c = Instance.new("UICorner")
    c.CornerRadius = UDim.new(0, 8)
    c.Parent = btn
    btn.MouseButton1Click:Connect(function()
        btn:Destroy()
        startDigest()
    end)
end

function doEat(target)
    if not target then return end
    hasEaten = true
    digestVisualScale = 1
    eatenPlayer = target

    if eatAnimationEnabled then
        eatAppearAlpha = 0
        eatGrowScale = 0.45
        if belly then belly.Transparency = 1 end
        if navel then navel.Transparency = 1 end
        playEatAnimation(target)
    else
        hideEatenPlayer(target)
    end

    eatAppearAlpha = 1
    eatGrowScale = 1
    if belly then belly.Transparency = bellyVisibleTarget() end
    if navel then navel.Transparency = bellyVisibleTarget() end

    isStrugglingEnabled = true
    ensureBulges()
    startStruggleLoop()

    task.wait(DIGEST_TIME)

    if eatState == "struggling" then
        eatState = "digesting-ready"
        showDigestButton()
    end
end

local function doFart()
    if not rootPart then return end
    local origin = rootPart.Position + Vector3.new(0, -1.5, 0)
    for i = 1, FART_PARTICLES do
        local p = Instance.new("Part")
        p.Shape = Enum.PartType.Ball
        p.Size = Vector3.new(0.4, 0.4, 0.4)
        p.Color = FART_COLOR
        p.Material = Enum.Material.Neon
        p.CanCollide, p.CanQuery, p.CanTouch, p.Massless = false, false, false, true
        p.Transparency = 0.2
        p.Position = origin
        p.Parent = Workspace

        local dir = Vector3.new(
            (math.random() - 0.5) * 2,
            math.random() * 0.4 + 0.1,
            (math.random() - 0.5) * 2
        ).Unit * (5 + math.random() * 5)

        local bv = Instance.new("BodyVelocity")
        bv.Velocity = dir
        bv.MaxForce = Vector3.new(1e5, 1e5, 1e5)
        bv.Parent = p

        task.delay(0.6, function()
            if p and p.Parent then
                bv:Destroy()
                local tw = TweenService:Create(p, TweenInfo.new(0.4), {
                    Transparency = 1,
                    Size = Vector3.new(0.1, 0.1, 0.1),
                })
                tw:Play()
                tw.Completed:Connect(function() p:Destroy() end)
            end
        end)
    end
end

function startDigest()
    if eatState ~= "digesting-ready" then return end
    eatState = "digesting"
    doFart()
    stopStruggleLoop()

    local fromScale = digestVisualScale
    local toScale = DIGEST_KEEP_SCALE
    local steps = 24
    for i = 1, steps do
        local t = i / steps
        digestVisualScale = fromScale + (toScale - fromScale) * t
        task.wait(DIGEST_SHRINK_TIME / steps)
    end
    digestVisualScale = toScale

    if belly then belly.Transparency = 1 end
    if navel then navel.Transparency = 1 end

    restoreEatenPlayer()
    eatenPlayer = nil

    if eatModeEnabled then
        eatState = "hunting"
    else
        eatState = nil
    end
    if belly then belly.Transparency = bellyVisibleTarget() end
    if navel then navel.Transparency = bellyVisibleTarget() end
end

local function startEatMode()
    if eatState ~= nil then return end
    eatModeEnabled = true
    hasEaten = false
    digestVisualScale = 1
    eatAppearAlpha = 1
    eatGrowScale = 1
    eatState = "hunting"
    if belly then belly.Transparency = bellyVisibleTarget() end
    if navel then navel.Transparency = bellyVisibleTarget() end
end

local function stopEatMode()
    eatModeEnabled = false
    hasEaten = false
    digestVisualScale = 1
    eatAppearAlpha = 1
    eatGrowScale = 1
    if eatGui then
        local b = eatGui:FindFirstChild("DigestBtn")
        if b then b:Destroy() end
    end
    if eatState == "struggling" or eatState == "digesting-ready" or eatState == "digesting" then
        stopStruggleLoop()
        restoreEatenPlayer()
    end
    eatState = nil
    eatenPlayer = nil
    if belly then belly.Transparency = bellyVisibleTarget() end
    if navel then navel.Transparency = bellyVisibleTarget() end
end

-- ================= 剧烈模式 =================
local function startViolentFartLoop()
    if violentFartThread then return end
    violentFartThread = task.spawn(function()
        while isViolentMode do
            task.wait(VIOLENT_FART_MIN + math.random() * (VIOLENT_FART_MAX - VIOLENT_FART_MIN))
            if not isViolentMode then break end
            doFart()
        end
        violentFartThread = nil
    end)
end

local function stopViolentFartLoop()
    if violentFartThread then
        task.cancel(violentFartThread)
        violentFartThread = nil
    end
end

local function startViolentMode()
    if isViolentMode then return end
    isViolentMode = true
    isStrugglingEnabled = true
    ensureBulges()
    startStruggleLoop()
    startViolentFartLoop()
end

local function stopViolentMode()
    if not isViolentMode then return end
    isViolentMode = false
    stopViolentFartLoop()
    stopStruggleLoop()
end

-- ================= 吃东西模式 =================
local function updateFoodUI()
    if not foodGui then return end
    local ratio = math.clamp(foodLevel / FOOD_MAX, 0, 1)

    if foodBarFill then
        foodBarFill.Size = UDim2.new(1, -6, 0, ratio * 432)
        if ratio < 0.25 then
            foodBarFill.BackgroundColor3 = Color3.fromRGB(220, 60, 60)
        elseif ratio < 0.5 then
            foodBarFill.BackgroundColor3 = Color3.fromRGB(230, 180, 60)
        else
            foodBarFill.BackgroundColor3 = Color3.fromRGB(120, 200, 80)
        end
    end

    if foodLevelText then
        foodLevelText.Text = string.format("%d / %d", math.floor(foodLevel + 0.5), FOOD_MAX)
    end
end

-- 【新】更新汉堡 CD 显示
local function updateBurgerCD()
    if not burgerSlot then return end
    if burgerCooldown > 0 then
        burgerSlot.BackgroundColor3 = Color3.fromRGB(30, 30, 35)
        if burgerCdLabel then
            burgerCdLabel.Visible = true
            burgerCdLabel.Text = string.format("%.1f", burgerCooldown)
        end
        -- 汉堡图形变灰
        for _, ch in ipairs(burgerSlot:GetChildren()) do
            if ch:IsA("Frame") then
                ch.BackgroundColor3 = Color3.fromRGB(80, 80, 80)
            end
        end
    else
        burgerSlot.BackgroundColor3 = Color3.fromRGB(45, 45, 55)
        if burgerCdLabel then burgerCdLabel.Visible = false end
        -- 恢复汉堡颜色
        local parts = burgerSlot:GetChildren()
        for _, ch in ipairs(parts) do
            if ch.Name == "TopBun" then ch.BackgroundColor3 = Color3.fromRGB(210, 160, 90)
            elseif ch.Name == "Lettuce" then ch.BackgroundColor3 = Color3.fromRGB(100, 180, 70)
            elseif ch.Name == "Patty" then ch.BackgroundColor3 = Color3.fromRGB(90, 55, 35)
            elseif ch.Name == "BottomBun" then ch.BackgroundColor3 = Color3.fromRGB(210, 160, 90)
            end
        end
    end
end

local function startFoodLoop()
    if foodConn then foodConn:Disconnect() end
    foodConn = RunService.RenderStepped:Connect(function(dt)
        if not isFoodModeEnabled then return end

        foodLevel = math.max(0, foodLevel - FOOD_DECAY_PER_SEC * dt)
        updateFoodUI()

        -- 【新】汉堡 CD 计时
        if burgerCooldown > 0 then
            burgerCooldown = math.max(0, burgerCooldown - dt)
            updateBurgerCD()
        end

        -- 100% 时偶尔放屁
        if foodLevel >= FOOD_MAX - 0.01 then
            local now = tick()
            if now - lastFoodFartTime > FOOD_FART_INTERVAL + math.random() * FOOD_FART_JITTER then
                lastFoodFartTime = now
                doFart()
            end
        end

        -- 归零 → 死亡
        if foodLevel <= 0 then
            local hum = player.Character and player.Character:FindFirstChildOfClass("Humanoid")
            if hum and hum.Health > 0 then
                hum.Health = 0
            end
            stopFoodMode()
        end
    end)
end

local function startFoodMode()
    if isFoodModeEnabled then return end
    isFoodModeEnabled = true
    foodLevel = FOOD_START_LEVEL
    burgerCooldown = 0
    updateBurgerCD()
    if foodGui then
        local bg = foodGui:FindFirstChild("FoodBarBg")
        if bg then bg.Visible = true end
        if burgerSlot then burgerSlot.Visible = true end
    end
    updateFoodUI()
    startFoodLoop()
end

function stopFoodMode()
    if not isFoodModeEnabled then return end
    isFoodModeEnabled = false
    if foodConn then foodConn:Disconnect() foodConn = nil end
    if foodGui then
        local bg = foodGui:FindFirstChild("FoodBarBg")
        if bg then bg.Visible = false end
        if burgerSlot then burgerSlot.Visible = false end
    end
end

-- 【新】吃汉堡逻辑
local function eatBurger()
    if not isFoodModeEnabled then return end
    if burgerCooldown > 0 then return end

    foodLevel = math.min(FOOD_MAX, foodLevel + FOOD_BURGER_RESTORE)
    updateFoodUI()

    -- 开始 CD
    burgerCooldown = FOOD_BURGER_COOLDOWN
    updateBurgerCD()

    -- 弹跳反馈
    if burgerSlot then
        burgerSlot.Size = UDim2.new(0, 62, 0, 62)
        TweenService:Create(burgerSlot, TweenInfo.new(0.15), { Size = UDim2.new(0, 72, 0, 72) }):Play()
    end

    -- 【新】3 秒后放屁
    if burgerFartThread then task.cancel(burgerFartThread) end
    burgerFartThread = task.spawn(function()
        task.wait(FOOD_BURGER_FART_DELAY)
        if isFoodModeEnabled then
            doFart()
        end
        burgerFartThread = nil
    end)
end

-- ================= UI =================
local function buildUI()
    eatGui = Instance.new("ScreenGui")
    eatGui.Name = "BellyEatUI"
    eatGui.ResetOnSpawn = false
    eatGui.Parent = player:WaitForChild("PlayerGui")

    local screenGui = Instance.new("ScreenGui")
    screenGui.Name = "BellyControlUI"
    screenGui.ResetOnSpawn = false
    screenGui.Parent = player:WaitForChild("PlayerGui")

    -- ===== 主面板 =====
    local mainFrame = Instance.new("Frame")
    mainFrame.Name = "MainFrame"
    mainFrame.Size = UDim2.new(0, 280, 0, 440)
    mainFrame.Position = UDim2.new(0.02, 0, 0.1, 0)
    mainFrame.BackgroundColor3 = Color3.fromRGB(30, 30, 35)
    mainFrame.BackgroundTransparency = 0.15
    mainFrame.BorderSizePixel = 0
    mainFrame.Active = true
    mainFrame.Draggable = true
    mainFrame.Parent = screenGui

    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 8)
    corner.Parent = mainFrame

    local title = Instance.new("TextLabel")
    title.Size = UDim2.new(1, -40, 0, 36)
    title.BackgroundTransparency = 1
    title.Text = "肚子控制面板"
    title.TextColor3 = Color3.fromRGB(255, 255, 255)
    title.TextScaled = true
    title.Font = Enum.Font.GothamBold
    title.Parent = mainFrame

    local hideBtn = Instance.new("TextButton")
    hideBtn.Size = UDim2.new(0, 28, 0, 28)
    hideBtn.Position = UDim2.new(1, -34, 0, 4)
    hideBtn.BackgroundColor3 = Color3.fromRGB(80, 80, 90)
    hideBtn.BorderSizePixel = 0
    hideBtn.Text = "×"
    hideBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    hideBtn.Font = Enum.Font.GothamBold
    hideBtn.TextSize = 20
    hideBtn.Parent = mainFrame

    local hideCorner = Instance.new("UICorner")
    hideCorner.CornerRadius = UDim.new(0, 6)
    hideCorner.Parent = hideBtn

    local showBtn = Instance.new("TextButton")
    showBtn.Name = "ShowBtn"
    showBtn.Size = UDim2.new(0, 70, 0, 70)
    showBtn.Position = UDim2.new(0, 20, 0.4, 0)
    showBtn.BackgroundColor3 = Color3.fromRGB(80, 120, 200)
    showBtn.BackgroundTransparency = 0.1
    showBtn.BorderSizePixel = 0
    showBtn.Text = "肚子"
    showBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    showBtn.Font = Enum.Font.GothamBold
    showBtn.TextSize = 16
    showBtn.Visible = false
    showBtn.Active = true
    showBtn.Draggable = true
    showBtn.Parent = screenGui

    local showCorner = Instance.new("UICorner")
    showCorner.CornerRadius = UDim.new(1, 0)
    showCorner.Parent = showBtn

    hideBtn.MouseButton1Click:Connect(function()
        mainFrame.Visible = false
        showBtn.Visible = true
    end)
    showBtn.MouseButton1Click:Connect(function()
        mainFrame.Visible = true
        showBtn.Visible = false
    end)

    local scroll = Instance.new("ScrollingFrame")
    scroll.Size = UDim2.new(1, -20, 1, -50)
    scroll.Position = UDim2.new(0, 10, 0, 45)
    scroll.BackgroundTransparency = 1
    scroll.BorderSizePixel = 0
    scroll.ScrollBarThickness = 4
    scroll.CanvasSize = UDim2.new(0, 0, 0, 4200)
    scroll.Parent = mainFrame

    local layout = Instance.new("UIListLayout")
    layout.Padding = UDim.new(0, 6)
    layout.SortOrder = Enum.SortOrder.LayoutOrder
    layout.Parent = scroll

    local yIndex = 0
    local function nextY() yIndex = yIndex + 1 return yIndex end

    local function makeSlider(labelText, minVal, maxVal, initialVal, step, callback, fillColor, formatStr)
        local container = Instance.new("Frame")
        container.Size = UDim2.new(1, -10, 0, 46)
        container.BackgroundTransparency = 1
        container.LayoutOrder = nextY()
        container.Parent = scroll

        local fmt = formatStr or "%.2f"

        local label = Instance.new("TextLabel")
        label.Size = UDim2.new(1, 0, 0, 20)
        label.BackgroundTransparency = 1
        label.Text = labelText .. ": " .. string.format(fmt, initialVal)
        label.TextColor3 = Color3.fromRGB(200, 200, 200)
        label.TextXAlignment = Enum.TextXAlignment.Left
        label.Font = Enum.Font.Gotham
        label.TextSize = 14
        label.Parent = container

        local slider = Instance.new("TextButton")
        slider.Size = UDim2.new(1, 0, 0, 22)
        slider.Position = UDim2.new(0, 0, 0, 22)
        slider.BackgroundColor3 = Color3.fromRGB(60, 60, 70)
        slider.BorderSizePixel = 0
        slider.Text = ""
        slider.AutoButtonColor = false
        slider.Parent = container

        local c2 = Instance.new("UICorner")
        c2.CornerRadius = UDim.new(0, 4)
        c2.Parent = slider

        local fill = Instance.new("Frame")
        fill.Size = UDim2.new(0, 0, 1, 0)
        fill.BackgroundColor3 = fillColor or Color3.fromRGB(100, 160, 255)
        fill.BorderSizePixel = 0
        fill.Parent = slider

        local fc2 = Instance.new("UICorner")
        fc2.CornerRadius = UDim.new(0, 4)
        fc2.Parent = fill

        local currentVal = initialVal
        local function updateFill()
            fill.Size = UDim2.new((currentVal - minVal) / (maxVal - minVal), 0, 1, 0)
            label.Text = labelText .. ": " .. string.format(fmt, currentVal)
        end
        updateFill()

        local dragging = false
        local function onInput(input)
            local ratio = math.clamp((input.Position.X - slider.AbsolutePosition.X) / slider.AbsoluteSize.X, 0, 1)
            currentVal = minVal + ratio * (maxVal - minVal)
            currentVal = math.clamp(math.floor(currentVal / step + 0.5) * step, minVal, maxVal)
            updateFill()
            callback(currentVal)
            queueSave()
        end

        slider.InputBegan:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseButton1
                or input.UserInputType == Enum.UserInputType.Touch then
                dragging = true
                onInput(input)
            end
        end)
        slider.InputChanged:Connect(function(input)
            if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement
                or input.UserInputType == Enum.UserInputType.Touch) then
                onInput(input)
            end
        end)
        slider.InputEnded:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseButton1
                or input.UserInputType == Enum.UserInputType.Touch then
                dragging = false
            end
        end)

        return container
    end

    local function makeToggle(labelText, initialOn, onClick)
        local container = Instance.new("Frame")
        container.Size = UDim2.new(1, -10, 0, 40)
        container.BackgroundTransparency = 1
        container.LayoutOrder = nextY()
        container.Parent = scroll

        local lbl = Instance.new("TextLabel")
        lbl.Size = UDim2.new(0.6, 0, 1, 0)
        lbl.BackgroundTransparency = 1
        lbl.Text = labelText
        lbl.TextColor3 = Color3.fromRGB(200, 200, 200)
        lbl.TextXAlignment = Enum.TextXAlignment.Left
        lbl.Font = Enum.Font.Gotham
        lbl.TextSize = 14
        lbl.Parent = container

        local btn = Instance.new("TextButton")
        btn.Size = UDim2.new(0, 80, 0, 30)
        btn.Position = UDim2.new(1, -80, 0.5, -15)
        btn.BorderSizePixel = 0
        btn.TextColor3 = Color3.fromRGB(255, 255, 255)
        btn.Font = Enum.Font.GothamBold
        btn.TextSize = 14
        btn.Parent = container

        local c = Instance.new("UICorner")
        c.CornerRadius = UDim.new(0, 4)
        c.Parent = btn

        local state = initialOn
        local function refresh()
            btn.Text = state and "开" or "关"
            btn.BackgroundColor3 = state and Color3.fromRGB(80, 180, 100) or Color3.fromRGB(80, 80, 90)
        end
        refresh()

        btn.MouseButton1Click:Connect(function()
            state = not state
            refresh()
            onClick(state)
        end)

        return container
    end

    -- ===== 肚子基础 =====
    makeSlider("整体大小", 0.1, 5, settings.sizeScale, 0.05, function(v) settings.sizeScale = v end)
    makeSlider("宽 X", 0.5, 10, settings.sizeX, 0.1, function(v) settings.sizeX = v end)
    makeSlider("高 Y", 0.5, 10, settings.sizeY, 0.1, function(v) settings.sizeY = v end)
    makeSlider("深 Z", 0.5, 10, settings.sizeZ, 0.1, function(v) settings.sizeZ = v end)
    makeSlider("位置 X", -5, 5, settings.offsetX, 0.05, function(v) settings.offsetX = v applySettings() end)
    makeSlider("位置 Y", -5, 5, settings.offsetY, 0.05, function(v) settings.offsetY = v applySettings() end)
    makeSlider("位置 Z", -5, 5, settings.offsetZ, 0.05, function(v) settings.offsetZ = v applySettings() end)
    makeSlider("旋转 X", -180, 180, settings.rotX, 5, function(v) settings.rotX = v applySettings() end, Color3.fromRGB(180, 140, 255), "%.0f°")
    makeSlider("旋转 Y", -180, 180, settings.rotY, 5, function(v) settings.rotY = v applySettings() end, Color3.fromRGB(180, 140, 255), "%.0f°")
    makeSlider("旋转 Z", -180, 180, settings.rotZ, 5, function(v) settings.rotZ = v applySettings() end, Color3.fromRGB(180, 140, 255), "%.0f°")
    makeSlider("颜色 R", 0, 255, settings.colorR, 1, function(v) settings.colorR = math.floor(v) applySettings() end)
    makeSlider("颜色 G", 0, 255, settings.colorG, 1, function(v) settings.colorG = math.floor(v) applySettings() end)
    makeSlider("颜色 B", 0, 255, settings.colorB, 1, function(v) settings.colorB = math.floor(v) applySettings() end)
    makeSlider("白色条", 0, 1, settings.whiteMix, 0.05,
        function(v) settings.whiteMix = v applySettings() end,
        Color3.fromRGB(255, 255, 255), "%.2f")

    -- ===== 动态 =====
    makeSlider("抖动幅度", 0, 3, settings.shakeAmp, 0.1, function(v) settings.shakeAmp = v end)
    makeSlider("走路晃动幅度", 0, 3, settings.walkAmp, 0.1, function(v) settings.walkAmp = v end)
    makeSlider("脉动幅度", 0, 0.3, settings.pulseAmp, 0.01, function(v) settings.pulseAmp = v end)
    makeSlider("脉动最短间隔", 0.2, 10, settings.pulseIntervalMin, 0.1, function(v) settings.pulseIntervalMin = v end)
    makeSlider("脉动最长间隔", 0.2, 10, settings.pulseIntervalMax, 0.1, function(v) settings.pulseIntervalMax = v end)
    makeSlider("挣扎幅度", 0, 3, settings.struggleAmp, 0.1, function(v) settings.struggleAmp = v end, Color3.fromRGB(255, 180, 100))

    -- ===== 胸部 =====
    makeSlider("胸部大小", 0.3, 2.0, settings.chestSize, 0.05, function(v) settings.chestSize = v updateChests() end, Color3.fromRGB(255, 200, 220))
    makeSlider("胸部晃动", 0, 3, settings.chestJiggle, 0.1, function(v) settings.chestJiggle = v end, Color3.fromRGB(255, 200, 220))
    makeSlider("胸位置 X", -2, 2, settings.chestOffsetX, 0.02, function(v) settings.chestOffsetX = v updateChests() end, Color3.fromRGB(255, 170, 200))
    makeSlider("胸位置 Y", -2, 2, settings.chestOffsetY, 0.02, function(v) settings.chestOffsetY = v updateChests() end, Color3.fromRGB(255, 170, 200))
    makeSlider("胸位置 Z", -2, 1, settings.chestOffsetZ, 0.02, function(v) settings.chestOffsetZ = v updateChests() end, Color3.fromRGB(255, 170, 200))
    makeSlider("胸颜色 R", 0, 255, settings.chestColorR, 1, function(v) settings.chestColorR = math.floor(v) updateChests() end, Color3.fromRGB(255, 170, 200), "%.0f")
    makeSlider("胸颜色 G", 0, 255, settings.chestColorG, 1, function(v) settings.chestColorG = math.floor(v) updateChests() end, Color3.fromRGB(255, 170, 200), "%.0f")
    makeSlider("胸颜色 B", 0, 255, settings.chestColorB, 1, function(v) settings.chestColorB = math.floor(v) updateChests() end, Color3.fromRGB(255, 170, 200), "%.0f")

    -- ===== 屁股 =====
    makeSlider("屁股大小", 0.3, 2.5, settings.buttSize, 0.05, function(v) settings.buttSize = v updateButts() end, Color3.fromRGB(255, 210, 210))
    makeSlider("屁股晃动", 0, 3, settings.buttJiggle, 0.1, function(v) settings.buttJiggle = v end, Color3.fromRGB(255, 210, 210))
    makeSlider("屁股位置 X", -2, 2, settings.buttOffsetX, 0.02, function(v) settings.buttOffsetX = v updateButts() end, Color3.fromRGB(255, 180, 180))
    makeSlider("屁股位置 Y", -2, 2, settings.buttOffsetY, 0.02, function(v) settings.buttOffsetY = v updateButts() end, Color3.fromRGB(255, 180, 180))
    makeSlider("屁股位置 Z", -2, 2, settings.buttOffsetZ, 0.02, function(v) settings.buttOffsetZ = v updateButts() end, Color3.fromRGB(255, 180, 180))
    makeSlider("屁股颜色 R", 0, 255, settings.buttColorR, 1, function(v) settings.buttColorR = math.floor(v) updateButts() end, Color3.fromRGB(255, 180, 180), "%.0f")
    makeSlider("屁股颜色 G", 0, 255, settings.buttColorG, 1, function(v) settings.buttColorG = math.floor(v) updateButts() end, Color3.fromRGB(255, 180, 180), "%.0f")
    makeSlider("屁股颜色 B", 0, 255, settings.buttColorB, 1, function(v) settings.buttColorB = math.floor(v) updateButts() end, Color3.fromRGB(255, 180, 180), "%.0f")

    -- ===== 挤压 =====
    makeSlider("挤压幅度", 0, 2, settings.squeezeAmp, 0.1, function(v) settings.squeezeAmp = v end, Color3.fromRGB(200, 255, 160))

    -- ===== 开关 =====
    makeToggle("抖动效果", false, function(on)
        isShakingEnabled = on
        if on then startShakeLoop() else stopShakeLoop() end
    end)

    makeToggle("挣扎效果", false, function(on)
        if on then
            isStrugglingEnabled = true
            ensureBulges()
            startStruggleLoop()
        else
            stopStruggleLoop()
        end
    end)

    makeToggle("吃人玩法", false, function(on)
        if on then startEatMode() else stopEatMode() end
    end)

    makeToggle("吃人动画", true, function(on)
        eatAnimationEnabled = on
    end)

    makeToggle("真实胸部", false, function(on)
        isChestEnabled = on
        if on then
            if player.Character then createChests(player.Character) end
            for plr, inst in pairs(syncedPlayers) do
                if not inst.chestL and inst.torso then
                    local cs = settings.chestSize
                    local sc = inst.torso.Color
                    local function makeChest(nm, xSign)
                        local p = makeRemotePart(nm, Enum.PartType.Ball,
                            Vector3.new(cs, cs, cs), sc, inst.character, 0)
                        local w = Instance.new("Weld")
                        w.Name = nm .. "Weld"
                        w.Part0 = inst.torso
                        w.Part1 = p
                        w.C0 = chestC0(xSign)
                        w.Parent = p
                        return p, w
                    end
                    inst.chestL, inst.chestWeldL = makeChest("ChestL", 1)
                    inst.chestR, inst.chestWeldR = makeChest("ChestR", -1)
                    inst.chestJiggle = Vector3.new(0, 0, 0)
                end
            end
        else
            destroyChests()
            for plr, inst in pairs(syncedPlayers) do
                for _, key in ipairs({"chestL", "chestR", "chestWeldL", "chestWeldR"}) do
                    if inst[key] then inst[key]:Destroy() inst[key] = nil end
                end
            end
        end
    end)

    makeToggle("胸颜色随肤", true, function(on)
        settings.chestUseSkin = on
        updateChests()
        queueSave()
    end)

    makeToggle("真实屁股", false, function(on)
        isButtEnabled = on
        if on then
            if player.Character then createButts(player.Character) end
            for plr, inst in pairs(syncedPlayers) do
                if not inst.buttL and inst.torso then
                    local bs = settings.buttSize
                    local sc = inst.torso.Color
                    local function makeButt(nm, xSign)
                        local p = makeRemotePart(nm, Enum.PartType.Ball,
                            Vector3.new(bs, bs, bs), sc, inst.character, 0)
                        local w = Instance.new("Weld")
                        w.Name = nm .. "Weld"
                        w.Part0 = inst.torso
                        w.Part1 = p
                        w.C0 = buttC0(xSign)
                        w.Parent = p
                        return p, w
                    end
                    inst.buttL, inst.buttWeldL = makeButt("ButtL", 1)
                    inst.buttR, inst.buttWeldR = makeButt("ButtR", -1)
                    inst.buttJiggle = Vector3.new(0, 0, 0)
                end
            end
        else
            destroyButts()
            for plr, inst in pairs(syncedPlayers) do
                for _, key in ipairs({"buttL", "buttR", "buttWeldL", "buttWeldR"}) do
                    if inst[key] then inst[key]:Destroy() inst[key] = nil end
                end
            end
        end
    end)

    makeToggle("屁股颜色随肤", true, function(on)
        settings.buttUseSkin = on
        updateButts()
        queueSave()
    end)

    makeToggle("剧烈模式", false, function(on)
        if on then startViolentMode() else stopViolentMode() end
    end)

    makeToggle("肚子碰撞箱（挤压）", false, function(on)
        isBellyCollideEnabled = on
        if not on then bellySqueezeTarget = Vector3.new(1, 1, 1) end
    end)

    makeToggle("肚子实体碰撞", false, function(on)
        isBellyPhysicalCollide = on
        if belly then belly.CanCollide = on end
        for _, inst in pairs(syncedPlayers) do
            if inst.belly then inst.belly.CanCollide = on end
        end
    end)

    -- 吃东西模式
    makeToggle("吃东西模式", false, function(on)
        if on then startFoodMode() else stopFoodMode() end
    end)

    -- ===== 同步功能 =====
    makeToggle("同步所有玩家", syncAllEnabled, function(on)
        syncAllEnabled = on
        settings.syncAllEnabled = on
        queueSave()
        refreshSyncedPlayers()
    end)

    local selectPlayerBtn = Instance.new("TextButton")
    selectPlayerBtn.Name = "SelectPlayerBtn"
    selectPlayerBtn.Size = UDim2.new(1, -10, 0, 32)
    selectPlayerBtn.BackgroundColor3 = Color3.fromRGB(80, 120, 200)
    selectPlayerBtn.BorderSizePixel = 0
    selectPlayerBtn.Text = syncSelectedPlayerName and ("选定： " .. syncSelectedPlayerName) or "选择要同步的玩家"
    selectPlayerBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    selectPlayerBtn.Font = Enum.Font.GothamBold
    selectPlayerBtn.TextSize = 14
    selectPlayerBtn.LayoutOrder = nextY()
    selectPlayerBtn.Parent = scroll

    local spc = Instance.new("UICorner")
    spc.CornerRadius = UDim.new(0, 4)
    spc.Parent = selectPlayerBtn

    local playerListFrame = Instance.new("Frame")
    playerListFrame.Name = "PlayerListFrame"
    playerListFrame.Size = UDim2.new(0, 220, 0, 300)
    playerListFrame.Position = UDim2.new(0.5, -110, 0.5, -150)
    playerListFrame.BackgroundColor3 = Color3.fromRGB(35, 35, 40)
    playerListFrame.BorderSizePixel = 0
    playerListFrame.Visible = false
    playerListFrame.ZIndex = 10
    playerListFrame.Parent = screenGui

    local plc = Instance.new("UICorner")
    plc.CornerRadius = UDim.new(0, 8)
    plc.Parent = playerListFrame

    local plTitle = Instance.new("TextLabel")
    plTitle.Size = UDim2.new(1, -40, 0, 30)
    plTitle.BackgroundTransparency = 1
    plTitle.Text = "选择要同步的玩家"
    plTitle.TextColor3 = Color3.fromRGB(255, 255, 255)
    plTitle.Font = Enum.Font.GothamBold
    plTitle.TextSize = 15
    plTitle.ZIndex = 11
    plTitle.Parent = playerListFrame

    local plClose = Instance.new("TextButton")
    plClose.Size = UDim2.new(0, 24, 0, 24)
    plClose.Position = UDim2.new(1, -28, 0, 3)
    plClose.BackgroundColor3 = Color3.fromRGB(80, 80, 90)
    plClose.BorderSizePixel = 0
    plClose.Text = "×"
    plClose.TextColor3 = Color3.fromRGB(255, 255, 255)
    plClose.Font = Enum.Font.GothamBold
    plClose.TextSize = 16
    plClose.ZIndex = 11
    plClose.Parent = playerListFrame

    local plcc = Instance.new("UICorner")
    plcc.CornerRadius = UDim.new(0, 6)
    plcc.Parent = plClose

    local plScroll = Instance.new("ScrollingFrame")
    plScroll.Size = UDim2.new(1, -12, 1, -40)
    plScroll.Position = UDim2.new(0, 6, 0, 34)
    plScroll.BackgroundTransparency = 1
    plScroll.BorderSizePixel = 0
    plScroll.ScrollBarThickness = 4
    plScroll.CanvasSize = UDim2.new(0, 0, 0, 0)
    plScroll.ZIndex = 11
    plScroll.Parent = playerListFrame

    local plLayout = Instance.new("UIListLayout")
    plLayout.Padding = UDim.new(0, 4)
    plLayout.SortOrder = Enum.SortOrder.LayoutOrder
    plLayout.Parent = plScroll

    local function rebuildPlayerList()
        for _, ch in ipairs(plScroll:GetChildren()) do
            if ch:IsA("TextButton") then ch:Destroy() end
        end

        local noneBtn = Instance.new("TextButton")
        noneBtn.Size = UDim2.new(1, -6, 0, 30)
        noneBtn.BackgroundColor3 = Color3.fromRGB(70, 70, 80)
        noneBtn.BorderSizePixel = 0
        noneBtn.Text = "（不选定任何玩家）"
        noneBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
        noneBtn.Font = Enum.Font.Gotham
        noneBtn.TextSize = 13
        noneBtn.ZIndex = 12
        noneBtn.Parent = plScroll
        local nc = Instance.new("UICorner")
        nc.CornerRadius = UDim.new(0, 4)
        nc.Parent = noneBtn

        noneBtn.MouseButton1Click:Connect(function()
            syncSelectedPlayerName = nil
            settings.syncSelectedPlayerName = ""
            selectPlayerBtn.Text = "选择要同步的玩家"
            queueSave()
            refreshSyncedPlayers()
            playerListFrame.Visible = false
        end)

        for _, plr in ipairs(Players:GetPlayers()) do
            if plr ~= player then
                local btn = Instance.new("TextButton")
                btn.Size = UDim2.new(1, -6, 0, 30)
                local selected = (syncSelectedPlayerName == plr.Name)
                btn.BackgroundColor3 = selected and Color3.fromRGB(80, 160, 100) or Color3.fromRGB(55, 55, 65)
                btn.BorderSizePixel = 0
                btn.Text = plr.DisplayName .. " (@" .. plr.Name .. ")"
                btn.TextColor3 = Color3.fromRGB(255, 255, 255)
                btn.Font = Enum.Font.Gotham
                btn.TextSize = 13
                btn.ZIndex = 12
                btn.Parent = plScroll
                local bc = Instance.new("UICorner")
                bc.CornerRadius = UDim.new(0, 4)
                bc.Parent = btn

                btn.MouseButton1Click:Connect(function()
                    syncSelectedPlayerName = plr.Name
                    settings.syncSelectedPlayerName = plr.Name
                    selectPlayerBtn.Text = "选定： " .. plr.Name
                    queueSave()
                    refreshSyncedPlayers()
                    playerListFrame.Visible = false
                end)
            end
        end

        plScroll.CanvasSize = UDim2.new(0, 0, 0, plLayout.AbsoluteContentSize.Y + 8)
    end

    selectPlayerBtn.MouseButton1Click:Connect(function()
        rebuildPlayerList()
        playerListFrame.Visible = true
    end)

    plClose.MouseButton1Click:Connect(function()
        playerListFrame.Visible = false
    end)

    local refreshBtn = Instance.new("TextButton")
    refreshBtn.Size = UDim2.new(1, -10, 0, 28)
    refreshBtn.BackgroundColor3 = Color3.fromRGB(70, 90, 130)
    refreshBtn.BorderSizePixel = 0
    refreshBtn.Text = "刷新同步"
    refreshBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    refreshBtn.Font = Enum.Font.GothamBold
    refreshBtn.TextSize = 13
    refreshBtn.LayoutOrder = nextY()
    refreshBtn.Parent = scroll
    local rb = Instance.new("UICorner")
    rb.CornerRadius = UDim.new(0, 4)
    rb.Parent = refreshBtn
    refreshBtn.MouseButton1Click:Connect(function()
        refreshSyncedPlayers()
    end)

    -- ===== 关闭肚子 =====
    local closeBellyBtn = Instance.new("TextButton")
    closeBellyBtn.Name = "CloseBellyBtn"
    closeBellyBtn.Size = UDim2.new(1, -10, 0, 32)
    closeBellyBtn.BackgroundColor3 = Color3.fromRGB(90, 90, 100)
    closeBellyBtn.BorderSizePixel = 0
    closeBellyBtn.Text = "关闭肚子"
    closeBellyBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    closeBellyBtn.Font = Enum.Font.GothamBold
    closeBellyBtn.TextSize = 14
    closeBellyBtn.LayoutOrder = nextY()
    closeBellyBtn.Parent = scroll

    local cbc = Instance.new("UICorner")
    cbc.CornerRadius = UDim.new(0, 4)
    cbc.Parent = closeBellyBtn

    closeBellyBtn.MouseButton1Click:Connect(function()
        isBellyHidden = not isBellyHidden
        if isBellyHidden then
            closeBellyBtn.Text = "打开肚子"
            closeBellyBtn.BackgroundColor3 = Color3.fromRGB(80, 160, 100)
        else
            closeBellyBtn.Text = "关闭肚子"
            closeBellyBtn.BackgroundColor3 = Color3.fromRGB(90, 90, 100)
        end
        if belly then belly.Transparency = bellyVisibleTarget() end
        if navel then navel.Transparency = bellyVisibleTarget() end
        local vt = remoteVisibleTarget()
        for _, inst in pairs(syncedPlayers) do
            if inst.belly then inst.belly.Transparency = vt end
            if inst.navel then inst.navel.Transparency = vt end
        end
    end)

    -- ===== 重置 =====
    local resetBtn = Instance.new("TextButton")
    resetBtn.Size = UDim2.new(1, -10, 0, 32)
    resetBtn.BackgroundColor3 = Color3.fromRGB(180, 80, 80)
    resetBtn.BorderSizePixel = 0
    resetBtn.Text = "重置"
    resetBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    resetBtn.Font = Enum.Font.GothamBold
    resetBtn.TextSize = 14
    resetBtn.LayoutOrder = nextY()
    resetBtn.Parent = scroll

    local rc = Instance.new("UICorner")
    rc.CornerRadius = UDim.new(0, 4)
    rc.Parent = resetBtn

    resetBtn.MouseButton1Click:Connect(function()
        settings.sizeScale = BASE_SIZE_SCALE
        settings.sizeX, settings.sizeY, settings.sizeZ = BASE_SIZE_X, BASE_SIZE_Y, BASE_SIZE_Z
        settings.offsetX, settings.offsetY, settings.offsetZ = BASE_OFFSET.X, BASE_OFFSET.Y, BASE_OFFSET.Z
        settings.rotX, settings.rotY, settings.rotZ = BASE_ROT.X, BASE_ROT.Y, BASE_ROT.Z
        settings.colorR, settings.colorG, settings.colorB = 255, 150, 100
        settings.whiteMix = 0
        settings.shakeAmp, settings.walkAmp = 1.0, 1.0
        settings.pulseAmp = 0.08
        settings.pulseIntervalMin, settings.pulseIntervalMax = 1.5, 3.5
        settings.struggleAmp = 1.0
        settings.squeezeAmp = 1.0
        settings.chestSize = 0.7
        settings.chestJiggle = 1.0
        settings.chestOffsetX = CHEST_DEFAULT_OFFSET_X
        settings.chestOffsetY = CHEST_DEFAULT_OFFSET_Y
        settings.chestOffsetZ = CHEST_DEFAULT_OFFSET_Z
        settings.chestColorR, settings.chestColorG, settings.chestColorB = 255, 200, 180
        settings.chestUseSkin = true
        settings.buttSize = 0.75
        settings.buttJiggle = 1.0
        settings.buttOffsetX = BUTT_DEFAULT_OFFSET_X
        settings.buttOffsetY = BUTT_DEFAULT_OFFSET_Y
        settings.buttOffsetZ = BUTT_DEFAULT_OFFSET_Z
        settings.buttColorR, settings.buttColorG, settings.buttColorB = 255, 200, 180
        settings.buttUseSkin = true
        hasEaten = false
        digestVisualScale = 1
        eatAppearAlpha = 1
        eatGrowScale = 1
        pulseTarget = 1
        applySettings()
        updateChests()
        updateButts()
        saveSettings()
    end)

    -- ===== 吃东西模式 UI =====
    local foodGuiRef = Instance.new("ScreenGui")
    foodGuiRef.Name = "BellyFoodUI"
    foodGuiRef.ResetOnSpawn = false
    foodGuiRef.Parent = player:WaitForChild("PlayerGui")
    foodGui = foodGuiRef

    local barBg = Instance.new("Frame")
    barBg.Name = "FoodBarBg"
    barBg.Size = UDim2.new(0, 40, 0, 440)
    barBg.Position = UDim2.new(1, -70, 0.5, -220)
    barBg.BackgroundColor3 = Color3.fromRGB(20, 20, 25)
    barBg.BackgroundTransparency = 0.1
    barBg.BorderSizePixel = 0
    barBg.Visible = false
    barBg.Parent = foodGui

    local bgCorner = Instance.new("UICorner")
    bgCorner.CornerRadius = UDim.new(0, 6)
    bgCorner.Parent = barBg

    local bgStroke = Instance.new("UIStroke")
    bgStroke.Color = Color3.fromRGB(80, 80, 90)
    bgStroke.Thickness = 2
    bgStroke.Parent = barBg

    local fill = Instance.new("Frame")
    fill.Name = "Fill"
    fill.AnchorPoint = Vector2.new(0, 1)
    fill.Position = UDim2.new(0, 3, 1, -4)
    fill.Size = UDim2.new(1, -6, 0, 0)
    fill.BackgroundColor3 = Color3.fromRGB(120, 200, 80)
    fill.BorderSizePixel = 0
    fill.Parent = barBg
    foodBarFill = fill

    local fillCorner = Instance.new("UICorner")
    fillCorner.CornerRadius = UDim.new(0, 4)
    fillCorner.Parent = fill

    for k = 1, 19 do
        local line = Instance.new("Frame")
        line.Name = "Line" .. k
        line.Size = UDim2.new(1, -6, 0, 1)
        line.Position = UDim2.new(0, 3, 0, k * 21.6)
        line.BackgroundColor3 = Color3.fromRGB(60, 60, 70)
        line.BorderSizePixel = 0
        line.ZIndex = 2
        line.Parent = barBg
    end

    local levelText = Instance.new("TextLabel")
    levelText.Name = "LevelText"
    levelText.Size = UDim2.new(1, 0, 0, 22)
    levelText.Position = UDim2.new(0, 0, 0, -26)
    levelText.BackgroundTransparency = 1
    levelText.Text = "0 / 100"
    levelText.TextColor3 = Color3.fromRGB(255, 255, 255)
    levelText.Font = Enum.Font.GothamBold
    levelText.TextSize = 16
    levelText.Parent = barBg
    foodLevelText = levelText

    -- 汉堡物品栏槽
    local slot = Instance.new("TextButton")
    slot.Name = "BurgerSlot"
    slot.Size = UDim2.new(0, 72, 0, 72)
    slot.Position = UDim2.new(0.5, -36, 1, -100)
    slot.BackgroundColor3 = Color3.fromRGB(45, 45, 55)
    slot.BackgroundTransparency = 0.1
    slot.BorderSizePixel = 0
    slot.Text = ""
    slot.AutoButtonColor = false
    slot.Visible = false
    slot.Parent = foodGui
    burgerSlot = slot

    local slotCorner = Instance.new("UICorner")
    slotCorner.CornerRadius = UDim.new(0, 8)
    slotCorner.Parent = slot

    local slotStroke = Instance.new("UIStroke")
    slotStroke.Color = Color3.fromRGB(120, 120, 140)
    slotStroke.Thickness = 2
    slotStroke.Parent = slot

    -- 汉堡图形（命名便于 CD 变灰）
    local topBun = Instance.new("Frame")
    topBun.Name = "TopBun"
    topBun.Size = UDim2.new(0, 48, 0, 16)
    topBun.Position = UDim2.new(0.5, -24, 0, 12)
    topBun.BackgroundColor3 = Color3.fromRGB(210, 160, 90)
    topBun.BorderSizePixel = 0
    topBun.Parent = slot
    local c1 = Instance.new("UICorner")
    c1.CornerRadius = UDim.new(1, 0)
    c1.Parent = topBun

    local lettuce = Instance.new("Frame")
    lettuce.Name = "Lettuce"
    lettuce.Size = UDim2.new(0, 54, 0, 6)
    lettuce.Position = UDim2.new(0.5, -27, 0, 28)
    lettuce.BackgroundColor3 = Color3.fromRGB(100, 180, 70)
    lettuce.BorderSizePixel = 0
    lettuce.Parent = slot
    local c2 = Instance.new("UICorner")
    c2.CornerRadius = UDim.new(1, 0)
    c2.Parent = lettuce

    local patty = Instance.new("Frame")
    patty.Name = "Patty"
    patty.Size = UDim2.new(0, 52, 0, 12)
    patty.Position = UDim2.new(0.5, -26, 0, 34)
    patty.BackgroundColor3 = Color3.fromRGB(90, 55, 35)
    patty.BorderSizePixel = 0
    patty.Parent = slot
    local c3 = Instance.new("UICorner")
    c3.CornerRadius = UDim.new(1, 0)
    c3.Parent = patty

    local bottomBun = Instance.new("Frame")
    bottomBun.Name = "BottomBun"
    bottomBun.Size = UDim2.new(0, 48, 0, 14)
    bottomBun.Position = UDim2.new(0.5, -24, 0, 46)
    bottomBun.BackgroundColor3 = Color3.fromRGB(210, 160, 90)
    bottomBun.BorderSizePixel = 0
    bottomBun.Parent = slot
    local c4 = Instance.new("UICorner")
    c4.CornerRadius = UDim.new(0, 4)
    c4.Parent = bottomBun

    -- 【新】CD 文字
    local cdLabel = Instance.new("TextLabel")
    cdLabel.Name = "CdLabel"
    cdLabel.Size = UDim2.new(1, 0, 1, 0)
    cdLabel.BackgroundTransparency = 1
    cdLabel.Text = "10.0"
    cdLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
    cdLabel.Font = Enum.Font.GothamBold
    cdLabel.TextSize = 26
    cdLabel.TextStrokeTransparency = 0.3
    cdLabel.Visible = false
    cdLabel.ZIndex = 5
    cdLabel.Parent = slot
    burgerCdLabel = cdLabel

    slot.MouseButton1Click:Connect(eatBurger)
end

-- ================= 生命周期 =================
local function onCharacterAdded(character)
    local t = getTorso(character)
    local tries = 0
    while not t and tries < 50 do
        task.wait(0.1)
        t = getTorso(character)
        tries = tries + 1
    end
    if not t then return end
    task.wait(0.1)
    createBelly(character)
    if isChestEnabled then createChests(character) end
    if isButtEnabled then createButts(character) end

    if isFoodModeEnabled then
        foodLevel = FOOD_START_LEVEL
        updateFoodUI()
    end
end

loadSettings()

if player.Character then onCharacterAdded(player.Character) end
player.CharacterAdded:Connect(onCharacterAdded)

buildUI()

local function onPlayerAdded(plr)
    if plr == player then return end
    local function bind(char)
        task.wait(0.5)
        if shouldSyncPlayer(plr) then
            local t = getTorso(char)
            local tries = 0
            while not t and tries < 30 do
                task.wait(0.1)
                t = getTorso(char)
                tries = tries + 1
            end
            if t and shouldSyncPlayer(plr) then
                createRemoteBelly(plr, char)
            end
        end
    end
    if plr.Character then task.spawn(bind, plr.Character) end
    plr.CharacterAdded:Connect(function(char) task.spawn(bind, char) end)
    plr.CharacterRemoving:Connect(function() destroyRemoteBelly(plr) end)
    plr.CharacterAdded:Connect(function() task.delay(1.5, refreshSyncedPlayers) end)
end

for _, plr in ipairs(Players:GetPlayers()) do onPlayerAdded(plr) end
Players.PlayerAdded:Connect(onPlayerAdded)
Players.PlayerRemoving:Connect(function(plr) destroyRemoteBelly(plr) end)

task.spawn(function()
    while true do
        task.wait(3)
        refreshSyncedPlayers()
    end
end)

player.CharacterRemoving:Connect(function()
    stopShakeLoop()
    stopStruggleLoop()
    stopViolentMode()
    stopEatMode()
    destroyBelly()
    destroyChests()
    destroyButts()
end)
