pcall(function()
    setthreadidentity(2)
    local DetectFunc, KillFunc
    for _, v in getgc(true) do
        if typeof(v) == "table" then
            local a, b = rawget(v, "Detected"), rawget(v, "Kill")
            if typeof(a) == "function" and not DetectFunc then
                DetectFunc = a
                hookfunction(DetectFunc, function() return true end)
            end
            if typeof(b) == "function" and not KillFunc then
                KillFunc = b
                hookfunction(KillFunc, function() end)
            end
        end
    end
    local targetFunc = getrenv().debug.info or debug.info
    if targetFunc then
        local oldDebugInfo; oldDebugInfo = hookfunction(targetFunc, newcclosure(function(...)
            if DetectFunc and (...) == DetectFunc then return coroutine.yield(coroutine.running()) end
            return oldDebugInfo(...)
        end))
    end
    setthreadidentity(7)
end)

local ATUR = {
	JEMPUT_JARAK = 500,
	ANTAR_JARAK = 200,
	JEMPUT_JEDA = 0.25,
	ANTAR_JEDA = 0.68,
	TERBANG_KECEPATAN = 150,
	TERBANG_CELAH_MAKS = 4500,
	TERBANG_MAKS_PER_JALAN = 1,
	TERBANG_JAUH_MIN = 600,
	TERBANG_KETINGGIAN = 350,
	BATAS_DIAM = 50,
}

do
	local function num(v, def)
		local n = tonumber(v)
		return n or def
	end
	ATUR.JEMPUT_JARAK = math.clamp(num(ATUR.JEMPUT_JARAK, 150), 10, 250)
	ATUR.ANTAR_JARAK = math.clamp(num(ATUR.ANTAR_JARAK, 90), 10, 150)
	ATUR.JEMPUT_JEDA = math.clamp(num(ATUR.JEMPUT_JEDA, 0.25), 0.05, 5)
	ATUR.ANTAR_JEDA = math.clamp(num(ATUR.ANTAR_JEDA, 0.68), 0.05, 5)
	ATUR.TERBANG_KECEPATAN = math.clamp(num(ATUR.TERBANG_KECEPATAN, 150), 30, 400)
	ATUR.TERBANG_CELAH_MAKS = math.clamp(num(ATUR.TERBANG_CELAH_MAKS, 4500), 60, 5000)
	ATUR.TERBANG_MAKS_PER_JALAN = math.floor(math.clamp(num(ATUR.TERBANG_MAKS_PER_JALAN, 1), 1, 5))
	ATUR.TERBANG_JAUH_MIN = math.clamp(num(ATUR.TERBANG_JAUH_MIN, 600), 100, 3000)
	ATUR.TERBANG_KETINGGIAN = math.clamp(num(ATUR.TERBANG_KETINGGIAN, 350), 50, 800)
	ATUR.BATAS_DIAM = math.clamp(num(ATUR.BATAS_DIAM, 50), 15, 300)
end

pcall(function()
	if _G.RIDEGO_STOP then _G.RIDEGO_STOP() end
end)
task.wait(0.2)

local BUILD = "v15f-clean 2026-11-23"

local RIDEGO_PLACE_ID = 131378148336503
local AUTO_BACK = true
if game.PlaceId ~= RIDEGO_PLACE_ID then
	_G.RIDEGO_INFO = function()
		return {
			build = BUILD,
			running = false,
			phase = "place salah",
			note = string.format("place %d, bukan RideGO", game.PlaceId),
			orders = 0,
			hops = 0,
			menu = false,
			log = {},
		}
	end

	if AUTO_BACK then
		task.spawn(function()
			task.wait(8)
			pcall(function() game:GetService("TeleportService"):Teleport(RIDEGO_PLACE_ID) end)
		end)
	end
	return { ok = false, reason = "wrong-place", place = game.PlaceId }
end

local CFG = {
	AUTO_JOB = true,
	AUTO_SPAWN = true,
	AUTO_ACK = true,
	AUTO_REJOIN = true,
	MIN_FARE = 0,
	ORDERS = 0,
	HOP_FAIL_MAX = 8,
	STALL_STEPS = 14,
	FLY_SPEED = ATUR.TERBANG_KECEPATAN,
	FLY_ARRIVE = 4,
	FLY_MAX_TIME = 40,
	FLY_MAX_GAP = ATUR.TERBANG_CELAH_MAKS,
	FLY_MAX_CROSS = ATUR.TERBANG_MAKS_PER_JALAN,
	FLY_LONG_MIN = ATUR.TERBANG_JAUH_MIN,
	FLY_RISE = ATUR.TERBANG_KETINGGIAN,
	ARRIVE_PICKUP = 16,
	ARRIVE_DROP = 16,
	GROUND_OFFSET = 1.8,
	WALL_CLEAR = 3.6,
	EDGE_MARGIN = 30,
	OFFER_TIMEOUT = 45,
	ACCEPT_TIMEOUT = 15,
	BOARD_WAIT = 8,
	BOARD_ROUNDS = 3,
	BOARD_BACKOFF = 22,
	DROP_WAIT = 30,
	DROP_ROUNDS = 3,
	DROP_HOLD = 40,
	TP_FAR = 350,
	TRAIL_FILE = "RideGO/trail_v1.json",
}

local Players = game:GetService("Players")
local RS = game:GetService("ReplicatedStorage")
local UIS = game:GetService("UserInputService")
local Debris = game:GetService("Debris")
local ws = workspace
local plr = Players.LocalPlayer
local TweenService = game:GetService("TweenService")
local RunService = game:GetService("RunService")
local AssetService = game:GetService("AssetService")
local playerGui = plr:WaitForChild("PlayerGui")

local S = {
	running = false,
	phase = "standby",
	note = "menunggu toggle",
	orders = 0,
	lastTp = 0,
	hops = 0,
	hover = false,
}

local nowT
do
	local ok, v = pcall(function() return workspace:GetServerTimeNow() end)
	if ok and type(v) == "number" then
		nowT = function() return workspace:GetServerTimeNow() end
	else
		nowT = function() return os.clock() end
	end
end

local ATUR_LIMIT = {
	JEMPUT_JARAK = { 10, 500 },
	ANTAR_JARAK = { 10, 250 },
	JEMPUT_JEDA = { 0.05, 5 },
	ANTAR_JEDA = { 0.05, 5 },
	BATAS_DIAM = { 15, 300 },
}

local function setAtur(key, value)
	local lim = ATUR_LIMIT[key]
	local n = tonumber(value)
	if not (lim and n) then return ATUR[key] end
	ATUR[key] = math.clamp(n, lim[1], lim[2])
	return ATUR[key]
end

local function setAuto(on)
	S.running = on and true or false
	if S.running then
		S.phase = "mulai"
		S.note = "auto dinyalakan"
	else
		S.phase = "standby"
		S.note = "auto dimatikan"
		if _G.RIDEGO_UNLOCK then pcall(_G.RIDEGO_UNLOCK) end
	end
end

_G.RIDEGO_INFO = function()
	return {
		build = BUILD,
		running = S.running,
		phase = S.phase,
		note = S.note,
		orders = S.orders,
		hops = S.hops,
		tripStuds = S.tripStuds,
		tripMoney = S.tripMoney,
		menu = (_G.RIDEGO_MENU and _G.RIDEGO_MENU() == true) or false,
		ping = S.ping,
		log = {},
	}
end

_G.RIDEGO_STOP = function()
	setAuto(false)
	S.note = "dihentikan manual (F8)"
end

_G.RIDEGO_START = function() setAuto(true) end

UIS.InputBegan:Connect(function(input, processed)
	if processed then return end
	if input.KeyCode == Enum.KeyCode.F8 then _G.RIDEGO_STOP() end
end)

local rayParams = RaycastParams.new()
rayParams.FilterType = Enum.RaycastFilterType.Exclude
rayParams.IgnoreWater = true

local waterParams = RaycastParams.new()
waterParams.FilterType = Enum.RaycastFilterType.Exclude
waterParams.IgnoreWater = false

local function rayHit(origin, dir, len, ignore)
	rayParams.FilterDescendantsInstances = ignore or {}
	return ws:Raycast(origin, dir * len, rayParams)
end

local BAD_GROUND = {
	[Enum.Material.Water] = true,
}

local function rayGroundBelow(p, depth, bike)
	local ignore = { plr.Character }
	if bike then table.insert(ignore, bike) end
	return rayHit(Vector3.new(p.X, p.Y + 10, p.Z), Vector3.new(0, -1, 0), depth or 40, ignore)
end

local overlapParams = OverlapParams.new()
overlapParams.FilterType = Enum.RaycastFilterType.Exclude

local function probeSize(bike)
	local _, size = bike:GetBoundingBox()
	return Vector3.new(math.max(size.X * 0.9, 3), math.max(size.Y * 0.75, 3), math.max(size.Z * 0.9, 3))
end

local function smallClearAt(pos, rot, bike)
	local size = probeSize(bike)
	local cf = CFrame.new(pos + Vector3.new(0, size.Y * 0.5 + 0.3, 0)) * rot
	overlapParams.FilterDescendantsInstances = { bike, plr.Character }
	local parts = ws:GetPartBoundsInBox(cf, size, overlapParams)
	for _, p in ipairs(parts) do
		if p:IsA("BasePart") and p.CanCollide and p.Size.Y >= 3 then
			local m = math.max(p.Size.X, p.Size.Y, p.Size.Z)
			if m < 40 then return false, p end
		end
	end
	return true
end

local function wallClearAt(pos, bike)
	local ignore = { bike, plr.Character }
	for _, h in ipairs({ 1.0, 2.2, 3.4 }) do
		local o = pos + Vector3.new(0, h, 0)
		for a = 0, 340, 20 do
			local rad = math.rad(a)
			local hit = rayHit(o, Vector3.new(math.cos(rad), 0, math.sin(rad)), CFG.WALL_CLEAR, ignore)
			if hit then return false, hit.Instance end
		end
	end
	return true
end

local function areaClearAt(pos, rot, bike)
	local ok, blk = smallClearAt(pos, rot, bike)
	if not ok then return false, blk end
	return wallClearAt(pos, bike)
end

local function uprightRot(pivot)
	local l = pivot.LookVector
	local flat = Vector3.new(l.X, 0, l.Z)
	if flat.Magnitude < 0.01 then flat = Vector3.new(0, 0, -1) end
	return CFrame.lookAt(Vector3.zero, flat)
end

local function bikeSnapshot(bike)
	local snap = {}
	for _, d in ipairs(bike:GetDescendants()) do
		if d:IsA("BasePart") then snap[#snap + 1] = { p = d, a = d.Anchored, c = d.CanCollide } end
	end
	return snap
end

local function bikeRestore(snap)
	for _, s in ipairs(snap) do
		if s.p and s.p.Parent then pcall(function()
			s.p.CanCollide = s.c
			s.p.Anchored = s.a
		end) end
	end
end

local function findSafeNear(p, maxR, bike, loose)
	maxR = maxR or 90
	local rot = uprightRot(bike:GetPivot())
	local ignore = { bike, plr.Character }
	for r = 6, maxR, 6 do
		for a = 0, 330, 30 do
			local rad = math.rad(a)
			local q = Vector3.new(p.X + math.cos(rad) * r, p.Y, p.Z + math.sin(rad) * r)
			local hit = rayGroundBelow(q, 40, bike)
			if not hit then
				local sky = rayHit(Vector3.new(q.X, p.Y + 500, q.Z), Vector3.new(0, -1, 0), 1000, ignore)
				if sky and math.abs(sky.Position.Y - p.Y) <= 15 then hit = sky end
			end
			if hit and not BAD_GROUND[hit.Material] and math.abs(hit.Position.Y - p.Y) <= (loose and 30 or 15) then
				local cand = Vector3.new(q.X, hit.Position.Y + 1.5, q.Z)
				local okUp = true
				if not loose then
					okUp = not rayHit(hit.Position + Vector3.new(0, 2, 0), Vector3.new(0, 1, 0), 10, ignore)
				end
				if okUp and areaClearAt(cand, rot, bike) then return cand end
			end
		end
	end
	return nil
end

local function bikeSettle(bike, rot)
	local p0 = bike:GetPivot().Position
	if areaClearAt(p0, rot, bike) then return true end
	for r = 3, 18, 3 do
		for i = 0, 7 do
			local a = math.rad(i * 45)
			local q = p0 + Vector3.new(math.cos(a) * r, 0, math.sin(a) * r)
			if areaClearAt(q, rot, bike) then
				pcall(function() bike:PivotTo(CFrame.new(q) * rot) end)
				return true
			end
		end
	end
	for _, lift in ipairs({ 4, 8, 12, 18, 26 }) do
		local q = p0 + Vector3.new(0, lift, 0)
		if areaClearAt(q, rot, bike) then
			pcall(function() bike:PivotTo(CFrame.new(q) * rot) end)
			return true
		end
	end
	return false
end

local ensurePassenger

local safePos

local function bikeDriveSeat(bike) return bike and bike:FindFirstChild("DriveSeat", true) end

local function charFollowBike(bike)
	local char = plr.Character
	local hum = char and char:FindFirstChildOfClass("Humanoid")
	local seat = bikeDriveSeat(bike)
	if not (char and hum and seat) then return false end
	if seat.Occupant ~= hum then return false end
	return pcall(function() char:PivotTo(seat.CFrame) end)
end

local function bikeReseat(bike, tries)
	local char = plr.Character
	local hum = char and char:FindFirstChildOfClass("Humanoid")
	local seat = bikeDriveSeat(bike)
	local hrp = char and char:FindFirstChild("HumanoidRootPart")
	if not (char and hum and seat and hrp) then return false end

	local near = (hrp.Position - seat.Position).Magnitude < 10
	if seat.Occupant == hum and hum.SeatPart == seat and near then return true end

	if hum.SeatPart then
		pcall(function() hum.Sit = false end)
		task.wait(0.05)
	end
	pcall(function() char:PivotTo(seat.CFrame) end)
	task.wait(0.1)

	pcall(function() hum:SetStateEnabled(Enum.HumanoidStateType.GettingUp, true) end)
	if hum:GetState() == Enum.HumanoidStateType.Ragdoll then
		pcall(function() hum:ChangeState(Enum.HumanoidStateType.GettingUp) end)
	end
	pcall(function() seat:Sit(hum) end)
	for _ = 1, (tries or 12) do
		task.wait(0.12)
		local r2 = char:FindFirstChild("HumanoidRootPart")
		if seat.Occupant == hum and r2 and (r2.Position - seat.Position).Magnitude < 10 then return true end
	end
	return false
end

local function bikeSnapGround(bike, rot)
	local p = bike:GetPivot().Position
	for _, h in ipairs({ 3, 40, 150, 400 }) do
		local hit = rayHit(p + Vector3.new(0, h, 0), Vector3.new(0, -1, 0), h + 250, { bike, plr.Character })
		if hit then
			local s = hit.Instance.Size
			if math.max(s.X, s.Y, s.Z) < 20000 and not BAD_GROUND[hit.Material] then
				local y = hit.Position.Y + CFG.GROUND_OFFSET
				pcall(function() bike:PivotTo(CFrame.new(Vector3.new(p.X, y, p.Z)) * rot) end)
				return true
			end
		end
	end
	return false
end

local noFallBike
local function fallScriptRemove()
	local pg = plr:FindFirstChildOfClass("PlayerGui")
	local iface = pg and pg:FindFirstChild("Interface")
	local fs = iface and iface:FindFirstChild("FallingScript")
	if fs and fs:IsA("LocalScript") and fs.Enabled then pcall(function() fs.Enabled = false end) end
end

local function fallGuardOn(bike)
	fallScriptRemove()
	if not (bike and bike.Parent) then return end
	if noFallBike == bike then return end
	noFallBike = bike
	local fs = bike:FindFirstChild("FallingScript", true)
	if fs and fs:IsA("LocalScript") then pcall(function() fs.Enabled = false end) end

	pcall(function()
		bike.DescendantAdded:Connect(function(d)
			if d.Name == "FallingScript" and d:IsA("LocalScript") then
				task.wait(0.1)
				pcall(function() d.Enabled = false end)
			end
		end)
	end)
end

local function bikeTeleport(bike, dest, label, blind, quick)
	if not (bike and bike.Parent and dest) then return false end
	local pivot = bike:GetPivot()
	if (dest - pivot.Position).Magnitude > CFG.TP_FAR then
		pcall(function() ws:RequestStreamAroundAsync(dest, 8) end)
	end
	local rot = uprightRot(pivot)
	local snap = bikeSnapshot(bike)
	getgenv().RIDEGO_PENDING = { bike = bike, snap = snap, at = nowT() }

	for _, s in ipairs(snap) do
		s.p.CanCollide = false
	end

	local function pivotApply(cf)
		pcall(function() bike:PivotTo(cf) end)
		charFollowBike(bike)
	end
	pivotApply(CFrame.new(dest + Vector3.new(0, 4, 0)) * rot)
	task.wait(0.05)
	if blind then
		local landed = false
		local ignore = { bike, plr.Character }
		for _ = 1, (quick and 12 or 60) do
			local hit = rayHit(Vector3.new(dest.X, dest.Y + 4, dest.Z), Vector3.new(0, -1, 0), 700, ignore)
			if not hit then
				hit = rayHit(Vector3.new(dest.X, dest.Y + 90, dest.Z), Vector3.new(0, -1, 0), 400, ignore)
			end
			if hit and not BAD_GROUND[hit.Material] and hit.Position.Y <= dest.Y + 60 then
				local y = hit.Position.Y + CFG.GROUND_OFFSET
				pcall(function() bike:PivotTo(CFrame.new(Vector3.new(dest.X, y, dest.Z)) * rot) end)
				charFollowBike(bike)
				landed = true
				break
			end
			pcall(function() ws:RequestStreamAroundAsync(dest, 10) end)
			task.wait(0.25)
		end
		if not landed then
			pcall(function() bike:PivotTo(pivot) end)
			charFollowBike(bike)
			bikeRestore(snap)
			getgenv().RIDEGO_PENDING = nil
			bikeReseat(bike)
			return false
		end
	end
	bikeSettle(bike, rot)
	if not bikeSnapGround(bike, rot) then
		pcall(function() bike:PivotTo(pivot) end)
		charFollowBike(bike)
		bikeRestore(snap)
		getgenv().RIDEGO_PENDING = nil
		bikeReseat(bike)
		return false
	end
	charFollowBike(bike)

	task.wait(0.1)
	fallGuardOn(bike)
	bikeRestore(snap)
	getgenv().RIDEGO_PENDING = nil

	bikeReseat(bike)
	safePos = bike:GetPivot().Position
	S.hops = S.hops + 1
	S.lastTp = nowT()
	return true
end

local function padFlat(pos, bike, rad)
	rad = rad or 6
	local ignore = { bike, plr.Character }
	local ok = 0
	local pts = { { 0, 0 }, { rad, 0 }, { -rad, 0 }, { 0, rad }, { 0, -rad } }
	for _, o in ipairs(pts) do
		local h = rayHit(Vector3.new(pos.X + o[1], pos.Y + 25, pos.Z + o[2]), Vector3.new(0, -1, 0), 60, ignore)
		if h and not BAD_GROUND[h.Material] and h.Normal.Y > 0.6 then ok = ok + 1 end
	end
	return ok >= 4
end

local function padExists(pos, bike) return padFlat(pos, bike, 6) or padFlat(pos, bike, 3) end

local function padFind(center, bike, maxR)
	maxR = maxR or 64
	local ignore = { bike, plr.Character }
	for r = 0, maxR, 8 do
		local n = (r == 0) and 1 or 8
		for i = 0, n - 1 do
			local a = (i / n) * math.pi * 2
			local q = center + Vector3.new(math.cos(a) * r, 0, math.sin(a) * r)
			local h = rayHit(Vector3.new(q.X, center.Y + 60, q.Z), Vector3.new(0, -1, 0), 260, ignore)
			if
				h
				and not BAD_GROUND[h.Material]
				and h.Normal.Y > 0.6
				and h.Position.Y > -80
				and padExists(Vector3.new(q.X, h.Position.Y, q.Z), bike)
			then
				return Vector3.new(q.X, h.Position.Y, q.Z)
			end
		end
	end
	return nil
end

local function hoverHold(cf)
	if S.holdPos and S.holdPos.Parent then S.holdPos.Position = cf.Position end
	if S.holdRot and S.holdRot.Parent then S.holdRot.CFrame = cf end
end

local function hoverLock(bike)
	if S.hoverAnchored then return end
	local root = bike.PrimaryPart or bike:FindFirstChildWhichIsA("BasePart", true)
	if not root then return end
	local cf = bike:GetPivot()
	local att = Instance.new("Attachment")
	att.Name = "RideGO_Hold"
	att.Parent = root
	local ap = Instance.new("AlignPosition")
	ap.Name = "RideGO_HoldPos"
	ap.Mode = Enum.PositionAlignmentMode.OneAttachment
	ap.Attachment0 = att
	ap.Position = cf.Position
	ap.RigidityEnabled = true
	ap.MaxForce = 1e7
	ap.Responsiveness = 200
	ap.Parent = root
	local ao = Instance.new("AlignOrientation")
	ao.Name = "RideGO_HoldRot"
	ao.Mode = Enum.OrientationAlignmentMode.OneAttachment
	ao.Attachment0 = att
	ao.CFrame = cf
	ao.RigidityEnabled = true
	ao.MaxTorque = 1e7
	ao.Responsiveness = 200
	ao.Parent = root
	S.hoverBike = bike
	S.hoverAtt = att
	S.holdPos = ap
	S.holdRot = ao
	S.hoverAnchored = true
	pcall(function()
		root.AssemblyLinearVelocity = Vector3.zero
		root.AssemblyAngularVelocity = Vector3.zero
	end)
end

local function hoverUnlock(bike)
	if not S.hoverAnchored then return end
	S.hoverAnchored = false
	for _, k in ipairs({ "holdPos", "holdRot" }) do
		local c = S[k]
		if c then
			pcall(function() c:Destroy() end)
			S[k] = nil
		end
	end
	if S.hoverAtt then
		pcall(function() S.hoverAtt:Destroy() end)
		S.hoverAtt = nil
	end
	S.hoverBike = nil
end

_G.RIDEGO_UNLOCK = function()
	local b = S.hoverBike
	if b then hoverUnlock(b) end
end

local function bikeFlyTo(bike, destPos, speed, arriveDist, maxTime, gap)
	if not (bike and bike.Parent and destPos) then return false end
	local root = bike.PrimaryPart or bikeDriveSeat(bike) or bike:FindFirstChildWhichIsA("BasePart", true)
	if not root then return false end

	speed = speed or CFG.FLY_SPEED
	arriveDist = arriveDist or CFG.FLY_ARRIVE
	maxTime = maxTime or CFG.FLY_MAX_TIME

	pcall(function() ws:RequestStreamAroundAsync(destPos, 20) end)

	local startPos = bike:GetPivot().Position
	local destY = destPos.Y or startPos.Y
	local long = (gap or 0) >= CFG.FLY_LONG_MIN

	local cruiseY = math.max(startPos.Y, destY + 2)
	if long then cruiseY = cruiseY + CFG.FLY_RISE end
	local target = Vector3.new(destPos.X, cruiseY, destPos.Z)

	local char = plr.Character
	local snap = bikeSnapshot(bike)
	local charSnap = {}
	if char and char.Parent then
		for _, d in ipairs(char:GetDescendants()) do
			if d:IsA("BasePart") then charSnap[#charSnap + 1] = { p = d, c = d.CanCollide } end
		end
	end

	local bg = Instance.new("BodyGyro")
	bg.P = 9e4
	bg.D = 50
	bg.MaxTorque = Vector3.new(9e9, 9e9, 9e9)
	bg.Parent = root
	local bv = Instance.new("BodyVelocity")
	bv.MaxForce = Vector3.new(9e9, 9e9, 9e9)
	bv.Velocity = Vector3.zero
	bv.Parent = root

	getgenv().RIDEGO_FLY = true
	local function noclipOn()
		for _, s in ipairs(snap) do
			if s.p and s.p.Parent then s.p.CanCollide = false end
		end
		for _, s in ipairs(charSnap) do
			if s.p and s.p.Parent then s.p.CanCollide = false end
		end
	end

	local ok = pcall(function()
		local t0 = nowT()
		local function faceDir(px, pz)
			bg.CFrame = CFrame.lookAt(root.Position, Vector3.new(px, root.Position.Y, pz))
		end

		if long then
			local tRise = nowT()
			while S.running do
				if not (bike.Parent and root.Parent) then error("motor hilang saat naik") end
				noclipOn()
				local cur = root.Position
				local need = cruiseY - cur.Y
				if need <= 6 or nowT() - tRise > 15 then break end
				bv.Velocity = Vector3.new(0, math.clamp(need * 2, 25, 140), 0)
				faceDir(target.X, target.Z)
				task.wait()
			end
		end

		while S.running do
			if not (bike.Parent and root.Parent) then error("motor hilang saat terbang") end
			noclipOn()
			local cur = root.Position
			local flat = Vector3.new(target.X - cur.X, 0, target.Z - cur.Z)
			local distance = flat.Magnitude
			if distance <= arriveDist then break end
			if nowT() - t0 > maxTime then error(string.format("terlalu lama (%.0f stud lagi)", distance)) end
			local dir = flat.Unit
			local vy, spd = 0, speed
			if long and distance < 260 then
				vy = math.clamp((destY + 2 - cur.Y) * 1.5, -150, 150)
				spd = math.min(speed, math.max(40, distance * 1.2))
			else
				vy = math.clamp((cruiseY - cur.Y) * 2, -60, 140)
			end
			bv.Velocity = Vector3.new(dir.X * spd, vy, dir.Z * spd)
			faceDir(target.X, target.Z)
			task.wait()
		end

		if long then
			local tDown = nowT()
			while S.running do
				if not (bike.Parent and root.Parent) then error("motor hilang saat turun") end
				noclipOn()
				local cur = root.Position
				local need = cur.Y - (destY + 2)
				if need <= 6 or nowT() - tDown > 20 then break end
				bv.Velocity = Vector3.new(0, math.clamp(-need * 2, -150, -20), 0)
				task.wait()
			end
		end
	end)

	pcall(function() bv.Velocity = Vector3.zero end)
	pcall(function() bg:Destroy() end)
	pcall(function() bv:Destroy() end)
	for _, s in ipairs(snap) do
		if s.p and s.p.Parent then
			pcall(function()
				s.p.AssemblyLinearVelocity = Vector3.zero
				s.p.AssemblyAngularVelocity = Vector3.zero
			end)
		end
	end
	bikeRestore(snap)
	for _, s in ipairs(charSnap) do
		if s.p and s.p.Parent then pcall(function() s.p.CanCollide = s.c end) end
	end
	getgenv().RIDEGO_FLY = nil

	if ok and not S.running then ok = false end

	local rot = uprightRot(bike:GetPivot())
	if ok then
		pcall(function() bike:PivotTo(CFrame.new(Vector3.new(destPos.X, destY, destPos.Z)) * rot) end)
		charFollowBike(bike)
	end
	if not ok then
		pcall(function() bike:PivotTo(CFrame.new(startPos) * rot) end)
		charFollowBike(bike)
		bikeSnapGround(bike, rot)
		bikeReseat(bike)
		return false
	end
	if not bikeSnapGround(bike, rot) then
		pcall(function() bike:PivotTo(CFrame.new(startPos) * rot) end)
		charFollowBike(bike)
		bikeSnapGround(bike, rot)
		bikeReseat(bike)
		return false
	end
	charFollowBike(bike)
	bikeReseat(bike)
	safePos = bike:GetPivot().Position
	S.hops = S.hops + 1
	S.lastTp = nowT()
	return true
end

local function bikeStep(bike, dest, label, allowDrop)
	if not (bike and bike.Parent and dest) then return false end
	local pivot = bike:GetPivot()
	local rot = uprightRot(pivot)
	local ignore = { bike, plr.Character }
	local curY = pivot.Position.Y

	if S.hover and S.hoverY then
		local hy = S.hoverY

		local g = rayHit(Vector3.new(dest.X, hy + 45, dest.Z), Vector3.new(0, -1, 0), 185, ignore)
		local land = nil
		if g and not BAD_GROUND[g.Material] and g.Position.Y > -80 then
			local dy = g.Position.Y - hy

			if (dy >= -18 and dy <= 25) or allowDrop then land = g.Position.Y end
		end

		if land and not padExists(Vector3.new(dest.X, land, dest.Z), bike) then
			local ps = padFind(Vector3.new(dest.X, land, dest.Z), bike, 24)
			if ps then
				land = ps.Y
				dest = Vector3.new(ps.X, land, ps.Z)
			elseif nowT() - (S.hoverAt or 0) < 10 then
				land = nil
			end
		end
		if not land then
			if nowT() - (S.hoverAt or 0) > 10 then
				local ps = padFind(Vector3.new(dest.X, hy, dest.Z), bike, 64)
				if ps then
					land = ps.Y
					dest = Vector3.new(ps.X, hy, ps.Z)
				end
			end
		end
		if not land then
			hoverLock(bike)
			local cf = CFrame.new(Vector3.new(dest.X, hy, dest.Z)) * rot
			hoverHold(cf)
			local okH = pcall(function() bike:PivotTo(cf) end)
			if not okH then return false end
			charFollowBike(bike)
			hoverHold(cf)
			S.hops = S.hops + 1
			S.lastTp = nowT()
			return true
		end

		S.hover, S.hoverY = false, nil
		hoverUnlock(bike)
		dest = Vector3.new(dest.X, land, dest.Z)
	else
		local top = math.max(dest.Y, curY) + 200
		local hit = rayHit(Vector3.new(dest.X, top, dest.Z), Vector3.new(0, -1, 0), 1200, ignore)
		if hit and BAD_GROUND[hit.Material] then hit = nil end

		if hit and hit.Normal.Y < 0.6 then hit = nil end
		if not hit then
			local deep = rayHit(Vector3.new(dest.X, curY + 4, dest.Z), Vector3.new(0, -1, 0), 2500, ignore)
			if deep and not BAD_GROUND[deep.Material] and deep.Position.Y > -80 and deep.Normal.Y > 0.6 then
				hit = deep
			end
		end

		local landPos = hit and Vector3.new(dest.X, hit.Position.Y, dest.Z) or nil
		if landPos and not padExists(landPos, bike) then landPos = padFind(landPos, bike, 32) end
		if not landPos then
			S.hover = true
			S.hoverY = curY
			S.hoverAt = nowT()
			hoverLock(bike)
			local cf = CFrame.new(Vector3.new(dest.X, curY, dest.Z)) * rot
			hoverHold(cf)
			local okH = pcall(function() bike:PivotTo(cf) end)
			if not okH then
				S.hover, S.hoverY = false, nil
				hoverUnlock(bike)
				return false
			end
			charFollowBike(bike)
			hoverHold(cf)
			S.hops = S.hops + 1
			S.lastTp = nowT()
			return true
		end
		dest = landPos
	end

	local ok = pcall(
		function() bike:PivotTo(CFrame.new(dest + Vector3.new(0, CFG.GROUND_OFFSET, 0)) * rot) end
	)
	if not ok then return false end
	local at = bike:GetPivot().Position
	local under = rayHit(at + Vector3.new(0, 1, 0), Vector3.new(0, -1, 0), 14, { bike, plr.Character })
	local solid = under
		and not BAD_GROUND[under.Material]
		and under.Normal.Y > 0.6
		and padExists(Vector3.new(at.X, under.Position.Y, at.Z), bike)
	if not solid then
		pcall(function() bike:PivotTo(pivot) end)
		charFollowBike(bike)
		local back =
			rayHit(pivot.Position + Vector3.new(0, 1, 0), Vector3.new(0, -1, 0), 14, { bike, plr.Character })
		if back then
			S.note = "tepi tanah, batal melangkah"
		else
			if not S.hover then
				S.hover, S.hoverY = true, pivot.Position.Y
				S.hoverAt = nowT()
				hoverLock(bike)
			end
			S.note = "tidak ada pijakan -> lanjut melayang"
		end
		return false
	end
	charFollowBike(bike)
	S.hops = S.hops + 1
	S.lastTp = nowT()

	safePos = bike:GetPivot().Position
	return true
end

local function pendingHeal()
	local pend = getgenv().RIDEGO_PENDING
	if type(pend) ~= "table" then return false end
	getgenv().RIDEGO_PENDING = nil
	if pend.bike and pend.bike.Parent and type(pend.snap) == "table" then
		bikeRestore(pend.snap)
		return true
	end
	return false
end

local NODES, BANNED = {}, {}
do
	if readfile and isfile and isfile(CFG.TRAIL_FILE) then
		local ok, data = pcall(
			function() return game:GetService("HttpService"):JSONDecode(readfile(CFG.TRAIL_FILE)) end
		)
		if ok and type(data) == "table" then
			for _, n in ipairs(data.nodes or data.n or {}) do
				local v
				if type(n) == "table" then
					if n[1] then
						v = Vector3.new(n[1], n[2], n[3])
					elseif n.x then
						v = Vector3.new(n.x, n.y, n.z)
					end
				end
				if v then NODES[#NODES + 1] = v end
			end
		end
	end
end

local function distXZ(a, b) return (Vector3.new(a.X, 0, a.Z) - Vector3.new(b.X, 0, b.Z)).Magnitude end

local NODE_SET = {}

local function mapAddNode(v)
	local key = string.format("%d_%d", math.floor(v.X / 22), math.floor(v.Z / 22))
	if NODE_SET[key] then return false end
	NODE_SET[key] = true
	NODES[#NODES + 1] = v
	return true
end

local lastScan = 0

local function mapScanRoads(force)
	if not force and (nowT() - lastScan) < 6 then return 0 end
	lastScan = nowT()
	local added = 0
	local seen = 0

	for _, d in ipairs(ws:GetDescendants()) do
		seen = seen + 1
		if seen % 300 == 0 then task.wait() end
		if
			d:IsA("BasePart")
			and d.CanCollide
			and d.Material == Enum.Material.Asphalt
			and d.Size.Y <= 10
			and math.max(d.Size.X, d.Size.Z) >= 12
		then
			local cf, size = d.CFrame, d.Size
			local nx = math.clamp(math.floor(size.X / 40), 1, 24)
			local nz = math.clamp(math.floor(size.Z / 40), 1, 24)
			for ix = 0, nx do
				for iz = 0, nz do
					local px = ((nx > 0) and (ix / nx - 0.5) or 0) * size.X
					local pz = ((nz > 0) and (iz / nz - 0.5) or 0) * size.Z
					local p = cf * Vector3.new(px, size.Y * 0.5, pz)
					if mapAddNode(p) then added = added + 1 end
					if added > 20000 then return added end
				end
			end
		end
	end
	return added
end

local function guideTarget()
	local am = ws:FindFirstChild("ActiveMissions")
	local g = am and am:FindFirstChild("RideGO_GuideTarget")
	if g and g:IsA("BasePart") then return g.Position end
	return nil
end

local function guideIsDrop()
	local am = ws:FindFirstChild("ActiveMissions")
	local g = am and am:FindFirstChild("RideGO_GuideTarget")
	if not g or not g:IsA("BasePart") then return false end
	local c
	local b = g:FindFirstChild("RideGO_Beam")
	if b and b:IsA("Beam") then
		local k = b.Color.Keypoints
		if k and k[1] then c = k[1].Value end
	end
	c = c or g.Color
	return c.G > 0.55 and c.R < 0.45
end

local function eventPos(d, key)
	if type(d) ~= "table" then return nil end
	local v = d[key]
	if typeof(v) == "Vector3" then return v end
	if type(v) == "table" and type(v.X) == "number" and type(v.Z) == "number" then
		return Vector3.new(v.X, v.Y or 0, v.Z)
	end
	return nil
end

local function eventDrop()
	local e = _G.RIDEGO_EV
	if type(e) ~= "table" then return nil end
	return eventPos(e.trip, "DropPos") or eventPos(e.accepted, "DropPos")
end

local function mapNearest(p)
	local best, bd = nil, math.huge
	for i, n in ipairs(NODES) do
		local d = distXZ(n, p)
		if d < bd then
			best, bd = n, d
		end
	end
	return best, bd
end

local GCELL = 130
local GRID, GRID_N = {}, -1

local function mapEnsureGrid()
	if GRID_N == #NODES then return end
	GRID = {}
	for i = 1, #NODES do
		local n = NODES[i]
		local k = math.floor(n.X / GCELL) .. "_" .. math.floor(n.Z / GCELL)
		local b = GRID[k]
		if not b then
			b = {}
			GRID[k] = b
		end
		b[#b + 1] = i
	end
	GRID_N = #NODES
end

local function mapNeighbors(i, out)
	mapEnsureGrid()
	local n = NODES[i]
	local cx = math.floor(n.X / GCELL)
	local cz = math.floor(n.Z / GCELL)
	for dx = -1, 1 do
		for dz = -1, 1 do
			local b = GRID[(cx + dx) .. "_" .. (cz + dz)]
			if b then
				for _, j in ipairs(b) do
					if j ~= i then
						local m = NODES[j]
						local ax, az = m.X - n.X, m.Z - n.Z
						if ax * ax + az * az <= GCELL * GCELL and math.abs(m.Y - n.Y) <= 45 then out[#out + 1] = j end
					end
				end
			end
		end
	end
	return out
end

local function mapNearIndex(p, maxR)
	mapEnsureGrid()
	maxR = maxR or 400
	local best, bd = nil, maxR
	local r = math.ceil(maxR / GCELL)
	local cx = math.floor(p.X / GCELL)
	local cz = math.floor(p.Z / GCELL)
	for dx = -r, r do
		for dz = -r, r do
			local b = GRID[(cx + dx) .. "_" .. (cz + dz)]
			if b then
				for _, i in ipairs(b) do
					local n = NODES[i]
					local ax, az = n.X - p.X, n.Z - p.Z
					local d = math.sqrt(ax * ax + az * az)
					if d < bd then
						best, bd = i, d
					end
				end
			end
		end
	end
	return best, bd
end

local function mapPlan(from, to)
	if #NODES < 4 then return nil end
	local s = mapNearIndex(from, 600)
	if not s then return nil end
	local g = mapNearIndex(to, 1500)
	if not g then return nil end
	if s == g then return nil end

	local open = { s }
	local inOpen = { [s] = true }
	local gScore = { [s] = 0 }
	local gn = NODES[g]
	local sx, sz = NODES[s].X - from.X, NODES[s].Z - from.Z
	local h0x, h0z = gn.X - from.X, gn.Z - from.Z
	local fScore = { [s] = math.sqrt(sx * sx + sz * sz) + math.sqrt(h0x * h0x + h0z * h0z) }
	local came = {}
	local closed = {}
	local budget = 6000
	local iter = 0

	while #open > 0 and budget > 0 do
		budget = budget - 1
		iter = iter + 1

		if iter % 400 == 0 then task.wait() end
		local bi, bf = 1, math.huge
		for k = 1, #open do
			local f = fScore[open[k]] or math.huge
			if f < bf then
				bi, bf = k, f
			end
		end
		local cur = table.remove(open, bi)
		inOpen[cur] = nil
		if cur == g then break end
		closed[cur] = true
		local n = NODES[cur]
		local ns = {}
		mapNeighbors(cur, ns)
		for _, j in ipairs(ns) do
			local ban = BANNED[j]
			if not closed[j] and not (ban and ban > nowT()) then
				local m = NODES[j]
				local dx, dz = m.X - n.X, m.Z - n.Z
				local ng = (gScore[cur] or 0) + math.sqrt(dx * dx + dz * dz)
				if ng < (gScore[j] or math.huge) then
					came[j] = cur
					gScore[j] = ng
					local hx, hz = gn.X - m.X, gn.Z - m.Z
					fScore[j] = ng + math.sqrt(hx * hx + hz * hz)
					if not inOpen[j] then
						open[#open + 1] = j
						inOpen[j] = true
					end
				end
			end
		end
	end

	if not gScore[g] then return nil end
	local path = {}
	local c = g
	local guard = 0
	while c and c ~= s and guard < 5000 do
		guard = guard + 1
		table.insert(path, 1, NODES[c] + Vector3.new(0, 1.5, 0))
		c = came[c]
	end
	if #path == 0 then return nil end
	return path
end

local function bikeFind()
	local char = plr.Character
	if not char then return nil end
	local hum = char:FindFirstChildOfClass("Humanoid")
	local seat = hum and hum.SeatPart

	if seat then
		local m = seat.Parent
		while m and m.Parent ~= ws do
			m = m.Parent
		end
		if m and m:FindFirstChild("DriveSeat", true) then return m, seat end
	end

	for _, m in ipairs(ws:GetChildren()) do
		if m:IsA("Model") and m.Name:find(plr.Name, 1, true) then
			local ds = m:FindFirstChild("DriveSeat", true)
			if ds then return m, ds end
		end
	end
	return nil
end

local function bikeSit(bike, seat)
	local char = plr.Character
	local hum = char and char:FindFirstChildOfClass("Humanoid")
	if not (hum and seat) then return false end
	if seat.Occupant == hum then return true end
	pcall(function() char:PivotTo(seat.CFrame * CFrame.new(0, 2.5, 0)) end)
	task.wait(0.25)
	pcall(function() seat:Sit(hum) end)
	for _ = 1, 20 do
		task.wait(0.2)
		if seat.Occupant == hum then return true end
	end
	return seat.Occupant == hum
end

local function jobEnsure()
	local JobEvents = RS:FindFirstChild("JobEvents")
	local req = JobEvents and JobEvents:FindFirstChild("TeamChangeRequest")
	if not req then return false end
	if plr.Team and plr.Team.Name == "RideGO Driver" then return true end
	S.phase = "join job"
	local ok = pcall(function() req:FireServer("RideGO Driver", 11378976, 1, 0, "MainMenu") end)
	if not ok then return false end
	for _ = 1, 40 do
		task.wait(0.25)
		if plr.Team and plr.Team.Name == "RideGO Driver" then return true end
	end
	return plr.Team ~= nil and plr.Team.Name == "RideGO Driver"
end

local function bikeEnsure()
	pendingHeal()
	local bike, seat = bikeFind()
	if bike then
		fallGuardOn(bike)
		bikeSit(bike, seat)
		return bike, seat
	end
	local sp = RS:FindFirstChild("SpawnCarEvents")
	local sc = sp and sp:FindFirstChild("SpawnCar")
	if not sc then return nil end
	S.phase = "spawn motor"
	pcall(function() sc:FireServer("Yamahax-MioSporty") end)
	for _ = 1, 50 do
		task.wait(0.3)
		bike, seat = bikeFind()
		if bike then
			bikeSit(bike, seat)
			return bike, seat
		end
	end
	return nil
end

local function guardChar(bike)
	bike = bike or bikeFind()
	local char = plr.Character
	if not (bike and bike.Parent and char and char.Parent) then return end
	local hum = char:FindFirstChildOfClass("Humanoid")
	local root = char:FindFirstChild("HumanoidRootPart")
	local seat = bikeDriveSeat(bike)
	if not (hum and root and seat) then return end
	if hum.Health <= 0 then return end
	local bp = bike:GetPivot().Position
	local off = (root.Position - bp).Magnitude
	local falling = root.Position.Y < bp.Y - 20
	if off <= 20 and not falling and seat.Occupant == hum then return end
	pcall(function()
		root.AssemblyLinearVelocity = Vector3.zero
		root.AssemblyAngularVelocity = Vector3.zero
	end)
	pcall(function() char:PivotTo(seat.CFrame) end)
	if nowT() - (S.charLogAt or 0) > 2 then S.charLogAt = nowT() end
	bikeReseat(bike, 4)
end

function guardVoid()
	local bike = bikeFind()
	if not bike or not bike.Parent then return end

	pcall(guardChar, bike)
	local p = bike:GetPivot().Position

	if not S.hover and not S.hoverAnchored then
		local any = false
		for _, d in ipairs(bike:GetDescendants()) do
			if d:IsA("BasePart") and d.Anchored then
				any = true
				break
			end
		end
		if any then
			S.lockSeenAt = S.lockSeenAt or nowT()
			if nowT() - S.lockSeenAt > 1.5 then
				for _, d in ipairs(bike:GetDescendants()) do
					if d:IsA("BasePart") then pcall(function() d.Anchored = false end) end
				end
				S.lockSeenAt = nil
			end
		else
			S.lockSeenAt = nil
		end
	end

	if S.hover then
		if p.Y < -80 and safePos then
			S.hover = false
			S.hoverY = nil
			hoverUnlock(bike)
			bikeTeleport(bike, safePos + Vector3.new(0, 2, 0), "tarik dari melayang")
		end
		return
	end
	if safePos then
		local underNow =
			rayHit(Vector3.new(p.X, p.Y + 1, p.Z), Vector3.new(0, -1, 0), 10, { bike, plr.Character })
		if not underNow then
			local back = rayHit(
				Vector3.new(safePos.X, safePos.Y + 30, safePos.Z),
				Vector3.new(0, -1, 0),
				60,
				{ bike, plr.Character }
			)
			if back then
				bikeTeleport(bike, safePos + Vector3.new(0, 2, 0), "tarik dari tepi")
			else
				safePos = nil
				bikeEnsure()
			end
			return
		end
	end

	if safePos and p.Y < safePos.Y - 15 and not S.hover then
		local g = rayHit(p + Vector3.new(0, 1, 0), Vector3.new(0, -1, 0), 25, { bike, plr.Character })
		if not g then
			bikeTeleport(bike, safePos + Vector3.new(0, 2, 0), "tarik dari jatuh")
			return
		end
	end
	local n = mapNearest(p)

	local floorY = n and (n.Y - 80) or -40
	if p.Y < math.min(floorY, -40) then
		if safePos and safePos.Y > -100 then
			bikeTeleport(bike, safePos + Vector3.new(0, 2, 0), "tarik dari void")
		else
			bikeEnsure()
		end
	elseif p.Y > -60 then
		local under = rayHit(Vector3.new(p.X, p.Y + 4, p.Z), Vector3.new(0, -1, 0), 40, { bike, plr.Character })
		if not under then
			local far = rayHit(Vector3.new(p.X, p.Y, p.Z), Vector3.new(0, -1, 0), 2000, { bike, plr.Character })
			if far then bikeTeleport(bike, far.Position + Vector3.new(0, 2, 0), "turun ke tanah") end
		else
			safePos = p
		end
	end
end

local TaxiAssets = RS:WaitForChild("TaxiAssets", 30)
if not TaxiAssets then return { ok = false, reason = "no-taxi-assets" } end
local TaxiEvents = TaxiAssets:WaitForChild("Events", 15)
local TaxiEvent = TaxiEvents and TaxiEvents:WaitForChild("TaxiEvent", 15)
if not TaxiEvent then return { ok = false, reason = "no-taxi-event" } end

local EV = {
	online = false,
	offer = nil,
	accepted = nil,
	boarded = false,
	trip = nil,
	done = nil,
	acked = false,
}
local ackEarly

TaxiEvent.OnClientEvent:Connect(function(action, data)
	pcall(function()
		EV.last = action
		if action == "DutyStarted" then
			EV.online = true
		elseif action == "OrderOffer" then
			EV.offer = data or {}
		elseif action == "OrderAccepted" then
			EV.accepted = data or {}
			EV.acked = false
			EV.online = true
		elseif action == "OrderDeclined" or action == "OrderExpired" then
			EV.offer = nil
		elseif action == "PassengerBoarding" then
			EV.boarded = true
		elseif action == "TripStarted" then
			EV.boarded = true
			EV.trip = data
		elseif action == "OrderCompleted" then
			EV.done = data or true
			EV.online = true
			if ackEarly then ackEarly() end
		end
	end)
end)

_G.RIDEGO_EV = EV

local hookedRemote = {}
local lastHook = 0
local function taxiHook()
	if nowT() - lastHook < 5 then return end
	lastHook = nowT()
	local seen = 0
	for _, d in ipairs(RS:GetDescendants()) do
		seen = seen + 1
		if seen % 200 == 0 then task.wait() end
		if d:IsA("RemoteEvent") and d ~= TaxiEvent and not hookedRemote[d] then
			hookedRemote[d] = true
			local nm = d.Name
			pcall(function()
				d.OnClientEvent:Connect(function(...)
					if nm ~= "ChangeTitles" then return end
					local a = { ... }
					for _, v in ipairs(a) do
						if type(v) == "table" and v.name == "KM" then
							local idx = table.find(a, v)
							local studs = a[idx + 1]
							local money = a[idx + 2]
							if type(studs) == "number" then
								S.tripStuds = math.floor(studs)
								S.tripMoney = type(money) == "number" and math.floor(money) or S.tripMoney
							end
						end
					end
				end)
			end)
		end
	end
end

local function taxiFire(action, ...)
	local args = table.pack(...)
	return pcall(function() TaxiEvent:FireServer(action, table.unpack(args, 1, args.n)) end)
end

local function tripAck()
	if EV.acked then return end
	EV.acked = true
	if CFG.AUTO_ACK then taxiFire("AckTripComplete") end
end
ackEarly = tripAck

local function waitUntil(fn, timeout, step)
	local t0 = nowT()
	step = step or 0.25
	while S.running do
		local v = fn()
		if v then return v end
		if nowT() - t0 > timeout then return nil end
		task.wait(step)
	end
	return nil
end

local function stepFindGround(from, to, maxStep, ignore)
	local dir = Vector3.new(to.X - from.X, 0, to.Z - from.Z)
	if dir.Magnitude < 0.5 then return nil end
	local dirUnit = dir.Unit
	local fromD = dir.Magnitude
	local topY = math.max(from.Y, to.Y) + 140

	local angles = { 0, 15, -15, 28, -28, 42, -42, 55, -55, 65, -65 }
	local fracs = { 1.0, 0.85, 0.7 }
	for _, a in ipairs(angles) do
		local rad = math.rad(a)
		local c, s = math.cos(rad), math.sin(rad)
		local d = Vector3.new(dirUnit.X * c - dirUnit.Z * s, 0, dirUnit.X * s + dirUnit.Z * c)
		for _, f in ipairs(fracs) do
			local st = maxStep * f
			local px, pz = from.X + d.X * st, from.Z + d.Z * st

			if fromD - st * 0.6 > 0 then
				local hit = rayHit(Vector3.new(px, topY, pz), Vector3.new(0, -1, 0), 1400, ignore)
				if hit and not BAD_GROUND[hit.Material] then return Vector3.new(px, hit.Position.Y, pz), a, st end
			end
		end
	end
	return nil
end

local function stepFindClear(pos, target, step, bike, ignore, rot)
	local base = Vector3.new(target.X - pos.X, 0, target.Z - pos.Z)
	if base.Magnitude < 0.5 then return nil end
	base = base.Unit
	local dirs = { 0, 25, -25, 50, -50, 75, -75, 100, -100 }
	for _, a in ipairs(dirs) do
		local rad = math.rad(a)
		local c, s = math.cos(rad), math.sin(rad)
		local d = Vector3.new(base.X * c - base.Z * s, 0, base.X * s + base.Z * c)
		for _, f in ipairs({ 1.0, 0.65 }) do
			local st = step * f
			local q = Vector3.new(pos.X + d.X * st, pos.Y, pos.Z + d.Z * st)
			local hit = rayHit(Vector3.new(q.X, q.Y + 200, q.Z), Vector3.new(0, -1, 0), 1600, ignore)
			if hit and not BAD_GROUND[hit.Material] then
				local cand = Vector3.new(q.X, hit.Position.Y, q.Z)
				if areaClearAt(cand + Vector3.new(0, CFG.GROUND_OFFSET, 0), rot, bike) then return cand, a end
			end
		end
	end
	return nil
end

local function groundAtPoint(p, ignore, lift)
	local hit = rayHit(Vector3.new(p.X, p.Y + 400, p.Z), Vector3.new(0, -1, 0), 1400, ignore)
	if hit and not BAD_GROUND[hit.Material] then return Vector3.new(p.X, hit.Position.Y + (lift or 0), p.Z) end
	return nil
end

local function groundAtXZ(x, z, baseY, ignore)
	local hit = rayHit(Vector3.new(x, (baseY or 0) + 150, z), Vector3.new(0, -1, 0), 450, ignore)
	if hit and not BAD_GROUND[hit.Material] then return hit.Position.Y end
	return nil
end

local function gapAhead(pos, target, ignore, maxGap)
	local dx, dz = target.X - pos.X, target.Z - pos.Z
	local d0 = math.sqrt(dx * dx + dz * dz)
	if d0 < 1 then return nil end
	local dir = Vector3.new(dx / d0, 0, dz / d0)
	maxGap = maxGap or CFG.FLY_MAX_GAP
	local baseY = pos.Y

	local function wet(x, z, groundY, refY)
		waterParams.FilterDescendantsInstances = ignore or {}
		local wh = ws:Raycast(Vector3.new(x, (refY or baseY) + 150, z), Vector3.new(0, -1, 0) * 460, waterParams)
		return wh ~= nil
			and wh.Material == Enum.Material.Water
			and (groundY == nil or wh.Position.Y > groundY - 1)
	end
	local gapStart = nil

	local step = math.clamp(maxGap / 80, 15, 60)
	local d = 10
	while d <= maxGap do
		local x = pos.X + dir.X * d
		local z = pos.Z + dir.Z * d
		local y = groundAtXZ(x, z, baseY, ignore)
		if y and wet(x, z, y, baseY) then y = nil end
		if y then
			if gapStart then
				local dLand, landY = nil, nil
				local scan = math.max(gapStart + 5, d - step)
				while scan <= d do
					local sx = pos.X + dir.X * scan
					local sz = pos.Z + dir.Z * scan
					local sy = groundAtXZ(sx, sz, baseY, ignore)
					if sy and wet(sx, sz, sy, baseY) then sy = nil end
					if sy then
						dLand, landY = scan, sy
						break
					end
					scan = scan + 5
				end
				if dLand then
					local ax = pos.X + dir.X * (dLand + CFG.EDGE_MARGIN)
					local az = pos.Z + dir.Z * (dLand + CFG.EDGE_MARGIN)
					local ay = groundAtXZ(ax, az, landY, ignore)
					if ay and not wet(ax, az, ay, landY) and (dLand - gapStart) >= 12 then
						return dLand - gapStart, Vector3.new(pos.X + dir.X * dLand, landY, pos.Z + dir.Z * dLand)
					end
				end
			end
			baseY = y
		elseif not gapStart then
			gapStart = d
		end
		d = d + step
	end
	if gapStart then return nil, nil, maxGap - gapStart end
	return nil
end

local function stepIsSafe(pos, dirUnit, step, ignore, margin)
	if step < 1 then return nil end
	local midX = pos.X + dirUnit.X * step * 0.5
	local midZ = pos.Z + dirUnit.Z * step * 0.5
	local midY = groundAtXZ(midX, midZ, pos.Y, ignore)
	if not midY then return nil end
	local endX = pos.X + dirUnit.X * step
	local endZ = pos.Z + dirUnit.Z * step
	local endY = groundAtXZ(endX, endZ, midY, ignore)
	if not endY then return nil end
	local aheadX = endX + dirUnit.X * (margin or CFG.EDGE_MARGIN)
	local aheadZ = endZ + dirUnit.Z * (margin or CFG.EDGE_MARGIN)
	if not groundAtXZ(aheadX, aheadZ, endY, ignore) then return nil end
	return Vector3.new(endX, endY, endZ)
end

local function stepAroundVoid(pos, target, maxStep, ignore, margin)
	local dx, dz = target.X - pos.X, target.Z - pos.Z
	local d0 = math.sqrt(dx * dx + dz * dz)
	if d0 < 1 then return nil end
	local want = Vector3.new(dx / d0, 0, dz / d0)

	local cap = math.min(maxStep, math.max(4, d0))
	local angles = { 0, 20, -20, 40, -40, 60, -60, 80, -80, 100, -100, 120, -120, 140, -140, 158, -158 }
	local best, bestAng, bestStep, bestScore
	for _, a in ipairs(angles) do
		local rad = math.rad(a)
		local c, s = math.cos(rad), math.sin(rad)
		local d = Vector3.new(want.X * c - want.Z * s, 0, want.X * s + want.Z * c)
		for _, f in ipairs({ 1.0, 0.8, 0.6 }) do
			local st = cap * f
			local p = stepIsSafe(pos, d, st, ignore, margin)
			if p then
				local gain = d0 - distXZ(p, target)
				local score = gain - math.abs(a) * 0.02
				if not bestScore or score > bestScore then
					best, bestAng, bestStep, bestScore = p, a, st, score
				end

				if a == 0 and gain > 0 then return p, a, st end
			end
		end
	end
	return best, bestAng, bestStep
end

local UNREACH = {}
local function unreachKey(p) return string.format("%d_%d", math.floor(p.X / 200), math.floor(p.Z / 200)) end
local function unreachMark(p, reason)
	UNREACH[unreachKey(p)] = {
		expire = nowT() + 300,
		at = nowT() - 31,
		reason = reason,
	}
end
local function unreachHit(p)
	local e = UNREACH[unreachKey(p)]
	if not (e and e.expire > nowT()) then return false end
	if nowT() - e.at > 30 then e.at = nowT() end
	return true
end

local function travelTo(target, arriveR, tag)
	if unreachHit(target) then return false, "tidak terjangkau" end
	local fails = 0
	local tightTries = 0

	local bestDist = math.huge
	local stall = 0
	local stage = 0
	local stallPos = nil

	local gainMark = nil
	local progressAt = nowT()
	S.spd, S.spdT, S.spdD = 0, nil, nil

	local voidTry = 0

	local fast = (tag == "jemput")
	local path, planAt = nil, 0

	local flyUsed = 0

	local flyFails = 0

	local gapBlocked = 0
	local function tryFly(pos, dist)
		if flyFails >= 2 then return false end
		if flyUsed >= CFG.FLY_MAX_CROSS then return false end
		local bike = bikeFind()
		if not (bike and bike.Parent) then return false end

		if nowT() - (S.streamAt or 0) > 5 then
			S.streamAt = nowT()
			pcall(function() ws:RequestStreamAroundAsync(target, 20) end)

			gainMark = nil
			progressAt = nowT()
		end
		local gap, landPos, remain = gapAhead(pos, target, { bike, plr.Character }, CFG.FLY_MAX_GAP)
		if not (gap and landPos) then
			if remain then
				gapBlocked = gapBlocked + 1

				local dx, dz = target.X - pos.X, target.Z - pos.Z
				local dl = math.sqrt(dx * dx + dz * dz)
				if dl > 1 then
					local mid = pos + Vector3.new(dx / dl, 0, dz / dl) * 150
					pcall(function() ws:RequestStreamAroundAsync(mid, 40) end)
				end
				if gapBlocked >= 3 then return false, true end
			end
			return false
		end

		if dist and distXZ(landPos, target) >= dist - 10 then return false end
		flyUsed = flyUsed + 1
		if S.hover or S.hoverAnchored then
			S.hover, S.hoverY = false, nil
			hoverUnlock(bike)
		end
		local longHop = gap >= CFG.FLY_LONG_MIN
		local flyTime = math.max(CFG.FLY_MAX_TIME, (gap / math.max(30, CFG.FLY_SPEED)) * 2 + 30)
		local label = longHop and "terbang JAUH" or "terbang menyeberang"
		S.phase = label
		S.note = string.format("terbang %.0f stud", gap)
		local okFly = bikeFlyTo(bike, landPos, CFG.FLY_SPEED, CFG.FLY_ARRIVE, flyTime, gap)

		gainMark = nil
		progressAt = nowT()
		if not okFly then
			flyFails = flyFails + 1
			flyUsed = math.max(0, flyUsed - 1)
			return false
		end
		voidTry = 0
		fails = 0
		stall = 0
		stallPos = nil
		stage = 0
		path = nil
		bestDist = distXZ(bike:GetPivot().Position, target)
		return true
	end
	while S.running do
		guardVoid()

		if tag ~= "jemput" then
			local g = guideTarget()
			if g and distXZ(g, target) > 60 then
				target = g

				gainMark = nil
				progressAt = nowT()
			end
		end
		local bike = bikeFind()
		if not bike then
			bike = bikeEnsure()
			if not bike then return false, "motor hilang" end
		end
		fallGuardOn(bike)

		do
			local char = plr.Character
			local hum = char and char:FindFirstChildOfClass("Humanoid")
			local seat = bikeDriveSeat(bike)
			if seat and hum and seat.Occupant ~= hum then bikeReseat(bike) end
		end
		local pos = bike:GetPivot().Position
		local dist = distXZ(pos, target)
		if dist <= arriveR then
			bikeReseat(bike)
			S.note = string.format("sampai %s (%.0f stud)", tag, dist)
			return true
		end

		if fails >= 2 and dist <= math.max(arriveR * 4, 60) then
			local closer
			if tightTries < 3 and dist > arriveR * 1.6 then closer = findSafeNear(target, 45, bike, true) end
			if closer and distXZ(closer, target) < dist - 4 then
				tightTries = tightTries + 1
				bikeTeleport(bike, closer, "merapat ke " .. tag)
				fails = 0
			else
				bikeReseat(bike)
				S.note = string.format("cukup dekat %s (%.0f stud)", tag, dist)
				return true
			end
		end

		local now = nowT()
		if not S.spdT then
			S.spdT, S.spdD = now, dist
		elseif now - S.spdT > 2 then
			S.spd = (S.spdD - dist) / (now - S.spdT)
			S.spdT, S.spdD = now, dist
		end

		if not gainMark then
			gainMark = dist
			progressAt = now
		elseif dist <= gainMark - 100 then
			gainMark = dist
			progressAt = now
		end
		if now - progressAt > ATUR.BATAS_DIAM then
			S.note = "tidak ada kemajuan"
			unreachMark(target, "tidak ada kemajuan")
			return false, "tidak ada kemajuan"
		end

		S.phase = "teleport " .. tag
		local spd = math.max(0, S.spd or 0)
		if spd > 1 then
			S.note = string.format(
				"%.0f stud lagi  (%.0f stud/s, ~%ds)",
				dist,
				spd,
				math.ceil(dist / spd)
			)
		else
			S.note = string.format("%.0f stud lagi", dist)
		end

		local gap = fast and math.max(0.05, ATUR.JEMPUT_JEDA) or math.max(0.05, ATUR.ANTAR_JEDA)
		if nowT() - S.lastTp < gap then task.wait(gap - (nowT() - S.lastTp)) end

		local dest, how

		local margin = math.min(CFG.EDGE_MARGIN, math.max(10, dist * 0.5))
		if fast then
			local st = ATUR.JEMPUT_JARAK
			if fails >= 6 then
				local gs = groundAtPoint(target, { bike, plr.Character }, 0)
				pcall(function() ws:RequestStreamAroundAsync(target, 20) end)
				if gs then
					dest, how = gs, "titik jemput"
				end
			else
				local sg, sang = stepAroundVoid(pos, target, st, { bike, plr.Character }, margin)
				if not sg then
					local g, ang = stepFindGround(pos, target, st, { bike, plr.Character })
					if g then
						sg, sang = g, ang
					end
				end
				if sg then
					voidTry = 0
					dest, how = sg, "jemput cepat"
					if sang and sang ~= 0 then how = string.format("jemput putar pinggir %d", sang) end
				else
					local flew, unflyable = tryFly(pos, dist)
					if flew then
						voidTry = 0
						how = "terbang menyeberang"
					elseif unflyable then
						S.note = "celah terlalu lebar"
						return false, "celah terlalu lebar"
					else
						voidTry = voidTry + 1
						pcall(function() ws:RequestStreamAroundAsync(target, 20) end)
						S.note = string.format("menunggu peta (putar pinggir %d)", voidTry)
					end
				end
			end
		else
			local step = ATUR.ANTAR_JARAK

			if not path or #path == 0 or nowT() - planAt > 10 then
				path = mapPlan(pos, target)
				if (not path or #path == 0) and dist > 300 and nowT() - (S.mapScanAt or 0) > 20 then
					S.mapScanAt = nowT()
					pcall(function() ws:RequestStreamAroundAsync(target, 20) end)
					mapScanRoads()

					gainMark = nil
					progressAt = nowT()
					path = mapPlan(pos, target)
				end
				planAt = nowT()
			end
			if path and #path > 0 then
				while #path > 0 do
					local n0 = path[1]
					local dNode = distXZ(pos, n0)
					if dNode < 20 or (dNode <= step * 2 and distXZ(n0, target) > dist - 5) then
						table.remove(path, 1)
					else
						break
					end
				end
			end
			local ignore = { bike, plr.Character }
			if path and #path > 0 then
				local nxt = path[1]
				local first = distXZ(pos, nxt)
				if first > step then
					local dir = Vector3.new(nxt.X - pos.X, 0, nxt.Z - pos.Z)
					if dir.Magnitude > 0.01 then
						dir = dir.Unit
						nxt = Vector3.new(pos.X + dir.X * step, nxt.Y, pos.Z + dir.Z * step)
					end
				else
					local walked = first
					local from = nxt
					local prev = Vector3.new(nxt.X - pos.X, 0, nxt.Z - pos.Z)
					if prev.Magnitude > 0.01 then prev = prev.Unit else prev = nil end
					local i = 2
					while prev and i <= #path and walked < step do
						local nn = path[i]
						local seg = Vector3.new(nn.X - from.X, 0, nn.Z - from.Z)
						if seg.Magnitude > 0.01 then
							local dir = seg.Unit
							if prev:Dot(dir) < 0.75 then break end
							if walked + seg.Magnitude > step then
								nxt = Vector3.new(
									from.X + dir.X * (step - walked),
									nn.Y,
									from.Z + dir.Z * (step - walked)
								)
								walked = step
								break
							end
							walked = walked + seg.Magnitude
							nxt = nn
							from = nn
							prev = dir
						end
						i = i + 1
					end
				end

				local stp = math.min(step, math.max(1, distXZ(pos, nxt)))
				local g = stepAroundVoid(pos, nxt, stp, ignore, margin)
					or stepFindGround(pos, nxt, stp, ignore)
					or groundAtPoint(nxt, ignore)
				if g then
					voidTry = 0
					dest, how = g, "jalan raya"
				end
			end

			if not dest then
				local sg, sang = stepAroundVoid(pos, target, step, ignore, margin)
				if sg then
					voidTry = 0
					local hd = math.floor(distXZ(pos, sg))
					dest, how = sg, string.format("langkah %d stud", hd)
					if sang and sang ~= 0 then how = string.format("putar pinggir %d (%d stud)", sang, hd) end
				elseif dist <= 120 then
					local g, ang = stepFindGround(pos, target, step, ignore)
					if g then
						voidTry = 0
						local hd = math.floor(distXZ(pos, g))
						dest, how = g, string.format("langkah %d stud", hd)
						if ang ~= 0 then how = string.format("langkah %d stud (belok %d)", hd, ang) end
					end
				end
				if not dest then
					local flew, unflyable = tryFly(pos, dist)
					if flew then
						voidTry = 0
						how = "terbang menyeberang"
					elseif unflyable then
						S.note = "celah terlalu lebar"
						return false, "celah terlalu lebar"
					else
						voidTry = voidTry + 1
						pcall(function() ws:RequestStreamAroundAsync(target, 20) end)
						S.note = string.format("menunggu peta (putar pinggir %d)", voidTry)
					end
				end
			end

			if
				dest
				and how ~= "jembatan"
				and not string.find(how or "", "melayang")
				and not S.hover
				and (fails > 0 or S.hops % 4 == 0)
			then
				local rot2 = uprightRot(bike:GetPivot())
				if not areaClearAt(dest + Vector3.new(0, CFG.GROUND_OFFSET, 0), rot2, bike) then
					local cs, ang = stepFindClear(pos, target, step, bike, ignore, rot2)
					if cs then
						dest, how = cs, string.format("belok %d", ang or 0)
					else
						dest = nil
						S.note = "mentok tembok, cari jalan"
						task.wait(0.25)
					end
				end
			end
		end
		if dest then
			local hadPax = EV.boarded or EV.trip

			bikeStep(bike, dest, how, dist <= math.max(CFG.ARRIVE_DROP * 4, 60))

			if hadPax and ensurePassenger then
				local paxGap = (S.paxMiss or 0) >= 2 and 12 or 4
				if nowT() - (S.paxAt or 0) > paxGap then
					S.paxAt = nowT()
					if ensurePassenger(bike) then
						S.paxMiss = 0
					else
						S.paxMiss = (S.paxMiss or 0) + 1
					end
				end
			end
			if how == "lompat buta ke tujuan" then mapScanRoads() end
			local after = distXZ(bike:GetPivot().Position, target)
			if after > dist - 3 then
				if path and #path > 0 then table.remove(path, 1) end
				fails = fails + 1
			else
				fails = 0
			end

			if after < bestDist - 5 then
				local prevBest = bestDist
				bestDist = after
				stall = 0
				stallPos = nil

				if stage > 0 and after <= prevBest - 40 then stage = 0 end
			else
				stall = stall + 1
				stallPos = stallPos or pos

				if distXZ(pos, stallPos) > 120 then
					stall = 0
					stallPos = pos
				end
			end
			if stall >= CFG.STALL_STEPS and dist > math.max(arriveR * 2, 40) then
				stall = 0
				stallPos = nil

				local flew, unflyable = tryFly(pos, dist)
				if flew then
					path = nil
					stage = 0
				elseif unflyable then
					S.note = "celah terlalu lebar"
					unreachMark(target, "celah terlalu lebar")
					return false, "celah terlalu lebar"
				elseif stage == 0 then
					stage = 1
					path = nil
					local bi = mapNearIndex(pos, 250)
					if bi then BANNED[bi] = nowT() + 120 end
				elseif stage == 1 then
					stage = 2
					local moved = false
					local bp = safePos
					if bp and distXZ(bp, pos) > 40 and distXZ(bp, target) <= dist + 60 then
						moved = bikeTeleport(bike, bp + Vector3.new(0, 2, 0), "tarik dari macet")
					end
					if not moved then
						local alt = findSafeNear(pos, 120, bike, true)
						if alt and distXZ(alt, pos) > 25 then moved = bikeTeleport(bike, alt, "geser dari macet") end
					end
					path = nil
					bestDist = distXZ(bike:GetPivot().Position, target)
				else
					S.note = "macet, ulangi rute"
					return false, "macet"
				end
			end
		else
			if how == "terbang menyeberang" then
				task.wait(0.2)
			elseif voidTry > 0 then
				task.wait(0.5)
			else
				fails = fails + 1
				task.wait(0.8)
			end
		end

		if fails >= CFG.HOP_FAIL_MAX then
			S.note = "tidak bisa mendekat"
			return false, how or "mentok"
		end
	end
	return false, "dihentikan"
end

local function paxOnboard(bike)
	local drive = bikeDriveSeat(bike)
	for _, d in ipairs(bike:GetDescendants()) do
		if d:IsA("VehicleSeat") and d ~= drive and d.Occupant then
			if d.Occupant.Parent ~= plr.Character then return true end
		end
	end
	return false
end

local function bikePaxSeat(bike)
	local drive = bikeDriveSeat(bike)
	for _, d in ipairs(bike:GetDescendants()) do
		if d:IsA("VehicleSeat") and d ~= drive then return d end
	end
	return nil
end

ensurePassenger = function(bike)
	if not bike then return false end
	if paxOnboard(bike) then return true end
	local seat = bikePaxSeat(bike)
	if not seat then return false end
	local am = ws:FindFirstChild("ActiveMissions")
	local pax = am and am:FindFirstChild("RideGO_Passenger")
	local ph = pax and pax:FindFirstChildOfClass("Humanoid")
	if not (pax and ph) then return false end
	pcall(function() pax:PivotTo(seat.CFrame * CFrame.new(0, 3, 0)) end)
	task.wait(0.12)
	pcall(function() seat:Sit(ph) end)
	for _ = 1, 8 do
		task.wait(0.12)
		if seat.Occupant == ph then return true end
	end
	return false
end

local function paxWaitBoarding(pickup)
	for round = 1, CFG.BOARD_ROUNDS do
		if not S.running then return false end
		S.phase = "tunggu penumpang"
		S.note = string.format("ronde %d/%d", round, CFG.BOARD_ROUNDS)
		local b = bikeFind()
		if b and paxOnboard(b) then
			EV.boarded = true
			return true
		end
		local boarded = waitUntil(function()
			if EV.boarded or EV.trip then return true end
			local bb = bikeFind()
			return bb and paxOnboard(bb) or nil
		end, CFG.BOARD_WAIT)
		if boarded then
			EV.boarded = true
			return true
		end

		local bike = bikeFind()
		if not bike then return false end

		if ensurePassenger and ensurePassenger(bike) then
			EV.boarded = true
			return true
		end
		local cur = bike:GetPivot().Position
		local dir = Vector3.new(cur.X - pickup.X, 0, cur.Z - pickup.Z)
		if dir.Magnitude < 1 then dir = Vector3.new(1, 0, 0) end
		dir = dir.Unit

		local back = Vector3.new(cur.X, pickup.Y, cur.Z) + dir * CFG.BOARD_BACKOFF
		local spot = findSafeNear(back, 80, bike)
		if spot then
			bikeTeleport(bike, spot, "mundur dari penumpang")
			task.wait(1.2)
			local again = waitUntil(function() return EV.boarded and true or nil end, 3)
			if again then return true end
		end

		local ahead = findSafeNear(pickup, 110, bike) or findSafeNear(pickup, 60, bike, true)
		if ahead then
			bikeTeleport(bike, ahead, "maju ke penumpang")
			task.wait(0.8)
		end
	end
	return false
end

local function findExactSpot(p, bike, ignoreWalls)
	local ignore = { bike, plr.Character }
	local hit = rayGroundBelow(Vector3.new(p.X, p.Y, p.Z), 60, bike)
		or rayHit(Vector3.new(p.X, p.Y + 500, p.Z), Vector3.new(0, -1, 0), 1000, ignore)
	if not hit then return nil end
	local cand = Vector3.new(p.X, hit.Position.Y + CFG.GROUND_OFFSET, p.Z)
	if ignoreWalls then return cand end
	local rot = uprightRot(bike:GetPivot())
	if areaClearAt(cand, rot, bike) then return cand end
	return nil
end

local function waitDrop(drop)
	local holdStart = nowT()
	while S.running and (nowT() - holdStart) < CFG.DROP_HOLD do
		if EV.done then return true end
		guardVoid()
		local bike = bikeFind()

		local g = guideTarget()
		if bike and g then
			local dg = distXZ(bike:GetPivot().Position, g)
			if dg > 80 and distXZ(g, drop) > 80 then
				drop = g
				if not travelTo(drop, CFG.ARRIVE_DROP, "antar") then return false end
				holdStart = nowT()
				bike = bikeFind()
			end
		end
		if bike then
			local cur = bike:GetPivot().Position
			if distXZ(cur, drop) > 8 then
				local tight = findExactSpot(drop, bike, true) or findSafeNear(drop, 40, bike, true)
				if tight then bikeTeleport(bike, tight, "tahan di titik antar") end
			end
		end
		S.phase = "tunggu selesai"
		S.note = string.format(
			"verifikasi %ds/%ds | %s",
			math.floor(nowT() - holdStart),
			CFG.DROP_HOLD,
			tostring(S.tripMoney or "-")
		)
		task.wait(1.0)
	end
	if EV.done then return true end

	for round = 1, CFG.DROP_ROUNDS do
		if not S.running then return false end
		S.phase = "tunggu selesai"
		S.note = string.format("diam di titik antar %d/%d", round, CFG.DROP_ROUNDS)

		local t0 = nowT()
		while S.running and (nowT() - t0) < CFG.DROP_WAIT do
			if EV.done then return true end
			guardVoid()
			local bike = bikeFind()
			local g = guideTarget()

			if bike and g then
				local dg = distXZ(bike:GetPivot().Position, g)
				if dg > 80 and distXZ(g, drop) > 80 then
					drop = g
					if not travelTo(drop, CFG.ARRIVE_DROP, "antar") then return false end
					break
				end
			end

			if bike and distXZ(bike:GetPivot().Position, drop) > 5 then
				local tight = findExactSpot(drop, bike) or findSafeNear(drop, 40, bike, true)
				if tight then bikeTeleport(bike, tight, "rapat ke titik antar") end
			end
			S.note =
				string.format("diam di titik antar %d/%d | %s", round, CFG.DROP_ROUNDS, tostring(S.tripMoney or "-"))
			task.wait(1.0)
		end
		if EV.done then return true end
	end

	tripAck()
	local am = ws:FindFirstChild("ActiveMissions")
	local g = am and am:FindFirstChild("RideGO_GuideTarget")
	if g then pcall(function() g:Destroy() end) end
	return false
end

local function offerRequest()
	for attempt = 1, 4 do
		if not S.running then return nil end
		S.phase = "cari order"
		S.note = string.format("percobaan %d/4", attempt)

		if attempt == 1 then
			if not EV.online then
				taxiFire("GoOnline")
				EV.online = true
			end
		else
			taxiFire("GoOffline")
			EV.online = false
			task.wait(1.6)
			taxiFire("GoOnline")
			EV.online = true
		end
		local offer = waitUntil(function() return EV.offer end, CFG.OFFER_TIMEOUT)
		if offer then return offer end
	end
	return nil
end

local function runOrder()
	EV.offer, EV.accepted, EV.boarded, EV.trip, EV.done = nil, nil, false, nil, nil
	EV.acked = false

	local bike = bikeEnsure()
	if not bike then
		S.note = "tidak dapat motor"
		return false
	end
	bikeReseat(bike)

	local offer = offerRequest()
	if not offer then return false end
	if (tonumber(offer.Fare) or 0) < CFG.MIN_FARE then
		if offer.Token then taxiFire("DeclineOrder", offer.Token) end
		EV.offer = nil
		return false
	end
	if offer.Token == nil then return false end

	S.phase = "accept order"
	taxiFire("AcceptOrder", offer.Token)

	local acc = waitUntil(function() return EV.accepted end, CFG.ACCEPT_TIMEOUT)
	if not acc then return false end
	local pickup, drop = acc.PickupPos, acc.DropPos
	if not pickup then pickup = guideTarget() end

	local okTravel = travelTo(pickup, CFG.ARRIVE_PICKUP, "jemput")
	if not okTravel then return false end

	local boarded = paxWaitBoarding(pickup)
	if not boarded then return false end

	local drop
	do
		drop = waitUntil(function()
			if guideIsDrop() then return guideTarget() end
			if EV.trip then
				local d = eventDrop()
				if d then return d end
				return guideTarget()
			end
			return nil
		end, 25)
		if not drop then drop = eventDrop() or guideTarget() end
	end
	if not drop then return false end

	local okDrop = travelTo(drop, CFG.ARRIVE_DROP, "antar")
	if not okDrop then return false end

	local done = waitDrop(drop)
	if not done then return false end
	tripAck()
	EV.accepted, EV.trip, EV.boarded, EV.done = nil, nil, false, nil
	S.orders = S.orders + 1
	return true
end

local function tripResumeGuide()
	local g = guideTarget()
	if not g then return false end
	local isDrop = guideIsDrop()
	S.phase = "lanjut trip"
	if not travelTo(g, CFG.ARRIVE_DROP, isDrop and "antar" or "tujuan") then return false end

	if EV.done then
		tripAck()
		EV.done, EV.boarded, EV.trip, EV.accepted = nil, false, nil, nil
		S.orders = S.orders + 1
		return true
	end

	if isDrop or EV.trip or EV.boarded then
		EV.boarded = true
		if waitDrop(g) then
			tripAck()
			EV.done, EV.boarded, EV.trip, EV.accepted = nil, false, nil, nil
			S.orders = S.orders + 1
			return true
		end
		return false
	end

	if paxWaitBoarding(g) then
		local dest = guideTarget() or g
		if travelTo(dest, CFG.ARRIVE_DROP, "antar") then
			if waitDrop(dest) then
				tripAck()
				EV.done, EV.boarded, EV.trip, EV.accepted = nil, false, nil, nil
				S.orders = S.orders + 1
				return true
			end
		end
	end
	return false
end

local function tripResume()
	if not EV.trip then return false end
	local drop = (type(EV.trip) == "table") and EV.trip.DropPos or nil
	if not drop then return false end
	local ok = travelTo(drop, CFG.ARRIVE_DROP, "antar")
	if not ok then return false end
	local done = waitDrop(drop)
	if done then
		tripAck()
		EV.trip, EV.done = nil, nil
		EV.boarded, EV.accepted = false, nil
		S.orders = S.orders + 1
		return true
	end
	return false
end

local MENU_BTN = {
	"applySelect",
	"deploySelect",
	"playButton",
	"PlayButton",
	"Play",
	"play",
	"startButton",
	"StartButton",
	"deployButton",
}

local function getPlayerGui() return plr:FindFirstChildOfClass("PlayerGui") end

local function getMenuRoot()
	local pg = getPlayerGui()
	return pg, pg and pg:FindFirstChild("mainMenuSystem")
end

local function menuHitAtCenter()
	local pg = getPlayerGui()
	local cam = workspace.CurrentCamera
	if not (pg and cam) then return nil end
	local vp = cam.ViewportSize
	if vp.X <= 0 or vp.Y <= 0 then return nil end
	local ok, list = pcall(function() return pg:GetGuiObjectsAtPosition(vp.X * 0.5, vp.Y * 0.5) end)
	if not ok or type(list) ~= "table" then return nil end
	return list[1]
end

local function menuTopScreen(o)
	local p = o
	while p do
		if p:IsA("ScreenGui") then return p end
		p = p.Parent
	end
	return nil
end

local function guiUnderName(o, wanted)
	local p = o
	while p do
		if p.Name == wanted then return true end
		p = p.Parent
	end
	return false
end

local function menuOpen()
	local hit = menuHitAtCenter()
	if not hit then return false end
	return guiUnderName(hit, "mainMenuSystem")
end

local function loadingOpen()
	local hit = menuHitAtCenter()
	if not hit then return false end
	local sg = menuTopScreen(hit)
	if not sg then return false end
	local n = string.lower(sg.Name)
	return (string.find(n, "loading") ~= nil)
end

local function guiClick(button)
	if not (button and button.Parent) then return false end
	local t0 = nowT()

	pcall(function()
		local last = button.AbsolutePosition
		while nowT() - t0 < 2 do
			task.wait(0.05)
			if button.AbsolutePosition == last then break end
			last = button.AbsolutePosition
		end
	end)
	task.wait(0.1)
	local done = false
	pcall(function()
		local gs = game:GetService("GuiService")
		local vim = game:GetService("VirtualInputManager")
		local old = gs.SelectedObject
		gs.SelectedObject = button
		task.wait(0.08)
		vim:SendKeyEvent(true, Enum.KeyCode.Return, false, game)
		task.wait(0.05)
		vim:SendKeyEvent(false, Enum.KeyCode.Return, false, game)
		task.wait(0.08)
		gs.SelectedObject = old
		done = true
	end)
	if not done then
		pcall(function()
			if firesignal then
				firesignal(button.MouseButton1Down)
				task.wait(0.05)
				firesignal(button.MouseButton1Up)
				firesignal(button.MouseButton1Click)
				firesignal(button.Activated)
			end
		end)
	else
		task.wait(0.5)
		if (menuOpen() or loadingOpen()) and firesignal then
			pcall(function()
				firesignal(button.MouseButton1Down)
				task.wait(0.05)
				firesignal(button.MouseButton1Up)
				firesignal(button.MouseButton1Click)
				firesignal(button.Activated)
			end)
		end
	end
	return true
end

local waitCharacter

_G.RIDEGO_MENU = menuOpen
_G.RIDEGO_LOADING = loadingOpen

local menuIgnore = false
local function menuClearScreens(tag)
	tag = tag or "-"
	if menuIgnore then return true end
	if not menuOpen() and not loadingOpen() then return true end
	local t0 = nowT()
	local presses = 0
	local said = false
	S.phase = "main menu"
	S.note = "menembus menu"
	while S.running and (nowT() - t0) < 150 do
		if not menuOpen() and not loadingOpen() then
			waitCharacter(tag)
			return true
		end
		if not said then
			said = true
			local _, mm = getMenuRoot()
			local names = {}
			if mm then
				for _, d in ipairs(mm:GetDescendants()) do
					if d:IsA("TextButton") or d:IsA("ImageButton") then
						local ok, sz = pcall(function() return d.AbsoluteSize end)
						if ok and sz and sz.X > 0 then names[#names + 1] = d.Name end
					end
				end
			end
		end
		local _, mm = getMenuRoot()
		local cand = {}
		for _, nm in ipairs(MENU_BTN) do
			local b = mm and mm:FindFirstChild(nm, true)
			if b and (b:IsA("TextButton") or b:IsA("ImageButton")) then cand[#cand + 1] = b end
		end

		local target = nil
		if #cand > 0 then target = cand[(presses % #cand) + 1] end
		if target then
			pcall(guiClick, target)
			presses = presses + 1
			S.note = string.format("tekan %s (%d)", target.Name, presses)
			task.wait(0.5)
		else
			S.note = "menunggu tombol Play"
			task.wait(0.5)
		end
	end

	local ch0 = plr.Character
	if ch0 and ch0:FindFirstChild("HumanoidRootPart") then
		menuIgnore = true
		return true
	end
	return false
end

waitCharacter = function(tag)
	tag = tag or "-"
	local t0 = nowT()
	while S.running and (nowT() - t0) < 45 do
		local ch = plr.Character
		if ch and ch:FindFirstChild("HumanoidRootPart") then
			local hum = ch:FindFirstChildOfClass("Humanoid")
			if hum and hum.Health > 0 then return true end
		end
		task.wait(0.5)
	end
	return false
end

task.spawn(function()
	pendingHeal()

	local prepped = false
	while true do
		if not S.running then
			if S.phase ~= "standby" then
				S.phase = "standby"
				S.note = "menunggu toggle"
			end
			task.wait(0.35)
		elseif CFG.ORDERS > 0 and S.orders >= CFG.ORDERS then
			break
		else
			if not prepped then
				menuClearScreens("awal")
				if CFG.AUTO_JOB then jobEnsure() end
				if CFG.AUTO_SPAWN then bikeEnsure() end
				mapScanRoads(true)
				taxiHook()
				prepped = S.running
			end
			if S.running then
				if not menuIgnore and (menuOpen() or loadingOpen()) then
					menuClearScreens("ulang")
					taxiHook()
				end
				taxiHook()

				local bikeNow = bikeFind()
				local busy = EV.trip or EV.accepted or guideTarget() or (bikeNow and paxOnboard(bikeNow))

				local ok, runErr = pcall(function()
					if busy then return tripResume() or tripResumeGuide() end
					return runOrder()
				end)
				if not ok then
					EV.done, EV.trip, EV.accepted, EV.boarded = nil, nil, nil, false
					S.phase = "error, ulangi"
					task.wait(2)
				elseif not runErr then
					S.phase = busy and "lanjut trip" or "istirahat"
					S.note = busy and "belum sampai tujuan, ulangi" or "cari order lagi"
					task.wait(2)
				end
			end
		end
	end
	S.phase = "selesai"
end)

task.spawn(function()
	local ps = nil
	pcall(function() ps = game:GetService("Stats").Network.ServerStatsItem["Data Ping"] end)
	while ps do
		task.wait(1)
		local ok, v = pcall(function() return ps:GetValue() end)
		if ok then S.ping = v end
	end
end)

if CFG.AUTO_REJOIN then
	task.spawn(function()
		local TS = game:GetService("TeleportService")
		local PS = game:GetService("Players")
		local lastTry = -99
		while true do
			task.wait(1.0)
			local lp = PS.LocalPlayer
			local gone = false
			if not lp then
				gone = true
			else
				local ok, parent = pcall(function() return lp.Parent end)
				gone = (not ok) or parent == nil
			end
			if gone and (nowT() - lastTry) > 6 then
				lastTry = nowT()
				local ok = pcall(function() TS:Teleport(game.PlaceId) end)
				if not ok then pcall(function() TS:TeleportToPlaceInstance(game.PlaceId, game.JobId, lp) end) end
			end
		end
	end)
end

local function mountDashboard(bridge)
	local FONT = Enum.Font.Gotham
	local FONT_MED = Enum.Font.GothamMedium
	local FONT_BOLD = Enum.Font.GothamBold

	local EASE = Enum.EasingStyle.Quint
	local EASE_SOFT = Enum.EasingStyle.Quad

	local MOTION = {
		fast = 0.14,
		base = 0.22,
		slow = 0.34,
	}

	local C = {
		bg = Color3.fromHex("#0B0B0D"),
		rail = Color3.fromHex("#0E0E11"),
		panel = Color3.fromHex("#131317"),
		panel2 = Color3.fromHex("#17171C"),
		panel3 = Color3.fromHex("#1C1C22"),
		panel4 = Color3.fromHex("#26262D"),
		text = Color3.fromHex("#F2F2F5"),
		muted = Color3.fromHex("#8E8E99"),
		faint = Color3.fromHex("#5E5E68"),
		accent = Color3.fromHex("#6D6DF6"),
		green = Color3.fromHex("#4ADE80"),
		red = Color3.fromHex("#F87171"),
		amber = Color3.fromHex("#FBBF24"),
		white = Color3.fromRGB(255, 255, 255),
	}

	local ACCENTS = {
		{ "Indigo", Color3.fromHex("#6D6DF6") },
		{ "Violet", Color3.fromHex("#A78BFA") },
		{ "Cyan", Color3.fromHex("#22D3EE") },
		{ "Emerald", Color3.fromHex("#34D399") },
		{ "Amber", Color3.fromHex("#FBBF24") },
		{ "Rose", Color3.fromHex("#FB7185") },
	}

	local function new(class, props, parent)
		local inst = Instance.new(class)
		for k, v in pairs(props or {}) do
			if k ~= "Parent" then inst[k] = v end
		end
		inst.Parent = parent
		return inst
	end

	local function corner(parent, radius)
		return new("UICorner", { CornerRadius = UDim.new(0, radius or 10) }, parent)
	end

	local function stroke(parent, color, transparency, thickness)
		return new("UIStroke", {
			Color = color or C.white,
			Transparency = transparency or 0.92,
			Thickness = thickness or 1,
			ApplyStrokeMode = Enum.ApplyStrokeMode.Border,
		}, parent)
	end

	local function pad(parent, t, r, b, l)
		t = t or 0
		return new("UIPadding", {
			PaddingTop = UDim.new(0, t),
			PaddingRight = UDim.new(0, r or t),
			PaddingBottom = UDim.new(0, b or t),
			PaddingLeft = UDim.new(0, l or t),
		}, parent)
	end

	local function label(parent, props)
		local d = {
			BackgroundTransparency = 1,
			Font = FONT,
			TextSize = 13,
			TextColor3 = C.text,
			TextXAlignment = Enum.TextXAlignment.Left,
		}
		for k, v in pairs(props) do
			d[k] = v
		end
		return new("TextLabel", d, parent)
	end

	local function tween(inst, time, goal, style, dir, delay)
		return TweenService:Create(
			inst,
			TweenInfo.new(time, style or EASE, dir or Enum.EasingDirection.Out, 0, false, delay or 0),
			goal
		)
	end

	local function play(inst, time, goal, style, dir, delay)
		local t = tween(inst, time, goal, style, dir, delay)
		t:Play()
		return t
	end

	local ACCENT = C.accent
	local accentListeners = {}

	local function onAccent(fn)
		accentListeners[#accentListeners + 1] = fn
		fn(ACCENT)
	end

	local ICON_DATA = {
		lucide = {
			["activity"] = "M22 12h-2.48a2 2 0 0 0-1.93 1.46l-2.35 8.36a.25.25 0 0 1-.48 0L9.24 2.18a.25.25 0 0 0-.48 0l-2.35 8.36A2 2 0 0 1 4.49 12H2",
			["arrow-right"] = "M5 12h14m-7-7l7 7l-7 7",
			["bell"] = "M10.268 21a2 2 0 0 0 3.464 0m-10.47-5.674A1 1 0 0 0 4 17h16a1 1 0 0 0 .74-1.673C19.41 13.956 18 12.499 18 8A6 6 0 0 0 6 8c0 4.499-1.411 5.956-2.738 7.326",
			["calendar"] = "M8 2v3m8-3v3 M5 3H19A2 2 0 0 1 21 5V19A2 2 0 0 1 19 21H5A2 2 0 0 1 3 19V5A2 2 0 0 1 5 3Z M3 9h18",
			["chart-column"] = "M3 3v16a2 2 0 0 0 2 2h16m-3-4V9m-5 8V5M8 17v-3",
			["check"] = "M20 6L9 17l-5-5",
			["copy"] = "M10 8H20A2 2 0 0 1 22 10V20A2 2 0 0 1 20 22H10A2 2 0 0 1 8 20V10A2 2 0 0 1 10 8Z M4 16C2.9 16 2 15.1 2 14V4C2 2.9 2.9 2 4 2H14C15.1 2 16 2.9 16 4",
			["chevron-down"] = "m6 9l6 6l6-6",
			["chevron-right"] = "m9 18l6-6l-6-6",
			["clock"] = "M2 12A10 10 0 1 0 22 12A10 10 0 1 0 2 12Z M12 6v6l4 2",
			["database"] = "M3 5A9 3 0 1 0 21 5A9 3 0 1 0 3 5Z M3 5v14a9 3 0 0 0 18 0V5 M3 12a9 3 0 0 0 18 0",
			["download"] = "M12 15V3m9 12v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4 m7 10l5 5l5-5",
			["external-link"] = "M15 3h6v6m-11 5L21 3m-3 10v6a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2V8a2 2 0 0 1 2-2h6",
			["file"] = "M6 22a2 2 0 0 1-2-2V4a2 2 0 0 1 2-2h8a2.4 2.4 0 0 1 1.704.706l3.588 3.588A2.4 2.4 0 0 1 20 8v12a2 2 0 0 1-2 2z M14 2v5a1 1 0 0 0 1 1h5",
			["folder"] = "M20 20a2 2 0 0 0 2-2V8a2 2 0 0 0-2-2h-7.9a2 2 0 0 1-1.69-.9L9.6 3.9A2 2 0 0 0 7.93 3H4a2 2 0 0 0-2 2v13a2 2 0 0 0 2 2Z",
			["house"] = "M15 21v-8a1 1 0 0 0-1-1h-4a1 1 0 0 0-1 1v8 M3 10a2 2 0 0 1 .709-1.528l7-6a2 2 0 0 1 2.582 0l7 6A2 2 0 0 1 21 10v9a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2z",
			["layout-grid"] = "M4 3H9A1 1 0 0 1 10 4V9A1 1 0 0 1 9 10H4A1 1 0 0 1 3 9V4A1 1 0 0 1 4 3Z M15 3H20A1 1 0 0 1 21 4V9A1 1 0 0 1 20 10H15A1 1 0 0 1 14 9V4A1 1 0 0 1 15 3Z M15 14H20A1 1 0 0 1 21 15V20A1 1 0 0 1 20 21H15A1 1 0 0 1 14 20V15A1 1 0 0 1 15 14Z M4 14H9A1 1 0 0 1 10 15V20A1 1 0 0 1 9 21H4A1 1 0 0 1 3 20V15A1 1 0 0 1 4 14Z",
			["lock"] = "M5 11H19A2 2 0 0 1 21 13V20A2 2 0 0 1 19 22H5A2 2 0 0 1 3 20V13A2 2 0 0 1 5 11Z M7 11V7a5 5 0 0 1 10 0v4",
			["mail"] = "m22 7l-8.991 5.727a2 2 0 0 1-2.009 0L2 7 M4 4H20A2 2 0 0 1 22 6V18A2 2 0 0 1 20 20H4A2 2 0 0 1 2 18V6A2 2 0 0 1 4 4Z",
			["maximize"] = "M8 3H5a2 2 0 0 0-2 2v3m18 0V5a2 2 0 0 0-2-2h-3M3 16v3a2 2 0 0 0 2 2h3m8 0h3a2 2 0 0 0 2-2v-3",
			["minus"] = "M5 12h14",
			["moon"] = "M20.985 12.486a9 9 0 1 1-9.473-9.472c.405-.022.617.46.402.803a6 6 0 0 0 8.268 8.268c.344-.215.825-.004.803.401",
			["package"] = "M11 21.73a2 2 0 0 0 2 0l7-4A2 2 0 0 0 21 16V8a2 2 0 0 0-1-1.73l-7-4a2 2 0 0 0-2 0l-7 4A2 2 0 0 0 3 8v8a2 2 0 0 0 1 1.73zm1 .27V12 M3.29 7L12 12l8.71-5M7.5 4.27l9 5.15",
			["palette"] = "M12 22a1 1 0 0 1 0-20a10 9 0 0 1 10 9a5 5 0 0 1-5 5h-2.25a1.75 1.75 0 0 0-1.4 2.8l.3.4a1.75 1.75 0 0 1-1.4 2.8z M13 6.5A0.5 0.5 0 1 0 14 6.5A0.5 0.5 0 1 0 13 6.5Z M17 10.5A0.5 0.5 0 1 0 18 10.5A0.5 0.5 0 1 0 17 10.5Z M6 12.5A0.5 0.5 0 1 0 7 12.5A0.5 0.5 0 1 0 6 12.5Z M8 7.5A0.5 0.5 0 1 0 9 7.5A0.5 0.5 0 1 0 8 7.5Z",
			["plus"] = "M5 12h14m-7-7v14",
			["search"] = "m21 21l-4.34-4.34 M3 11A8 8 0 1 0 19 11A8 8 0 1 0 3 11Z",
			["settings"] = "M9.671 4.136a2.34 2.34 0 0 1 4.659 0a2.34 2.34 0 0 0 3.319 1.915a2.34 2.34 0 0 1 2.33 4.033a2.34 2.34 0 0 0 0 3.831a2.34 2.34 0 0 1-2.33 4.033a2.34 2.34 0 0 0-3.319 1.915a2.34 2.34 0 0 1-4.659 0a2.34 2.34 0 0 0-3.32-1.915a2.34 2.34 0 0 1-2.33-4.033a2.34 2.34 0 0 0 0-3.831A2.34 2.34 0 0 1 6.35 6.051a2.34 2.34 0 0 0 3.319-1.915 M9 12A3 3 0 1 0 15 12A3 3 0 1 0 9 12Z",
			["shield"] = "M20 13c0 5-3.5 7.5-7.66 8.95a1 1 0 0 1-.67-.01C7.5 20.5 4 18 4 13V6a1 1 0 0 1 1-1c2 0 4.5-1.2 6.24-2.72a1.17 1.17 0 0 1 1.52 0C14.51 3.81 17 5 19 5a1 1 0 0 1 1 1z",
			["sliders-horizontal"] = "M10 5H3m9 14H3M14 3v4m2 10v4m5-9h-9m9 7h-5m5-14h-7m-6 5v4m0-2H3",
			["square"] = "M5 3H19A2 2 0 0 1 21 5V19A2 2 0 0 1 19 21H5A2 2 0 0 1 3 19V5A2 2 0 0 1 5 3Z",
			["star"] = "M11.525 2.295a.53.53 0 0 1 .95 0l2.31 4.679a2.12 2.12 0 0 0 1.595 1.16l5.166.756a.53.53 0 0 1 .294.904l-3.736 3.638a2.12 2.12 0 0 0-.611 1.878l.882 5.14a.53.53 0 0 1-.771.56l-4.618-2.428a2.12 2.12 0 0 0-1.973 0L6.396 21.01a.53.53 0 0 1-.77-.56l.881-5.139a2.12 2.12 0 0 0-.611-1.879L2.16 9.795a.53.53 0 0 1 .294-.906l5.165-.755a2.12 2.12 0 0 0 1.597-1.16z",
			["sun"] = "M8 12A4 4 0 1 0 16 12A4 4 0 1 0 8 12Z M12 2v2m0 16v2M4.93 4.93l1.41 1.41m11.32 11.32l1.41 1.41M2 12h2m16 0h2M6.34 17.66l-1.41 1.41M19.07 4.93l-1.41 1.41",
			["terminal"] = "M12 19h8M4 17l6-6l-6-6",
			["trash"] = "M10 11v6m4-6v6m5-11v14a2 2 0 0 1-2 2H7a2 2 0 0 1-2-2V6M3 6h18M8 6V4a2 2 0 0 1 2-2h4a2 2 0 0 1 2 2v2",
			["user"] = "M19 21v-2a4 4 0 0 0-4-4H9a4 4 0 0 0-4 4v2 M8 7A4 4 0 1 0 16 7A4 4 0 1 0 8 7Z",
			["users"] = "M16 21v-2a4 4 0 0 0-4-4H6a4 4 0 0 0-4 4v2M16 3.128a4 4 0 0 1 0 7.744M22 21v-2a4 4 0 0 0-3-3.87 M5 7A4 4 0 1 0 13 7A4 4 0 1 0 5 7Z",
			["x"] = "M18 6L6 18M6 6l12 12",
			["zap"] = "M15.914 4a1.5 1.5 0 0 0-2.474-1.561l-9 9A1.5 1.5 0 0 0 5.5 14h4.002a.5.5 0 0 1 .471.666L8.086 20a1.5 1.5 0 0 0 2.475 1.56l9-9A1.5 1.5 0 0 0 18.5 10h-3.997a.5.5 0 0 1-.472-.667z",
		},
		solar = {
			["bell"] = "M18.7491 9.70957V9.00496C18.7491 5.13623 15.7274 2 12 2C8.27256 2 5.25087 5.13623 5.25087 9.00496V9.70957C5.25087 10.5552 5.00972 11.3818 4.5578 12.0854L3.45036 13.8095C2.43882 15.3843 3.21105 17.5249 4.97036 18.0229C9.57274 19.3257 14.4273 19.3257 19.0296 18.0229C20.789 17.5249 21.5612 15.3843 20.5496 13.8095L19.4422 12.0854C18.9903 11.3818 18.7491 10.5552 18.7491 9.70957Z M7.5 19C8.15503 20.7478 9.92246 22 12 22C14.0775 22 15.845 20.7478 16.5 19",
			["bolt"] = "M8.73167 5.77133L5.66953 9.91436C4.3848 11.6526 3.74244 12.5217 4.09639 13.205C4.10225 13.2164 4.10829 13.2276 4.1145 13.2387C4.48945 13.9117 5.59888 13.9117 7.81775 13.9117C9.05079 13.9117 9.6673 13.9117 10.054 14.2754L10.074 14.2946C10.4526 14.6662 10.4526 15.2585 10.4526 16.4433L10.4525 16.7528C10.4525 20.0375 10.4525 21.6798 11.3758 21.9628C12.2992 22.2458 13.2889 20.9067 15.2683 18.2287L18.3304 14.0857C19.6152 12.3475 20.2576 11.4784 19.9036 10.795C19.8977 10.7837 19.8917 10.7724 19.8855 10.7613C19.5106 10.0884 18.4011 10.0884 16.1822 10.0884C14.9492 10.0884 14.3327 10.0884 13.946 9.72466L13.926 9.70541C13.5474 9.33386 13.5474 8.74151 13.5474 7.55682V7.24712C13.5474 3.96249 13.5474 2.32018 12.6241 2.03721C11.7007 1.75425 10.711 3.09327 8.73167 5.77133Z",
			["calendar"] = "M10 21.75H14C17.7712 21.75 19.6569 21.75 20.8284 20.5784C22 19.4069 22 17.5212 22 13.75V11.75C22 7.97876 22 6.09315 20.8284 4.92157C19.6569 3.75 17.7712 3.75 14 3.75H10C6.22876 3.75 4.34315 3.75 3.17157 4.92157C2 6.09315 2 7.97876 2 11.75V13.75C2 17.5212 2 19.4069 3.17157 20.5784C4.34315 21.75 6.22876 21.75 10 21.75Z M2.03516 8.75H21.965 M17 12.75H17.0001 M12 12.75H12.0001 M7 12.75H7.0001 M17 16.7734H17.0001 M12 16.7734H12.0001 M7 16.7734H7.0001 M7 3.78496V2.25 M17 3.78496V2.25",
			["chart-2"] = "M3 22H21 M5 19C4.05719 19 3.58579 19 3.29289 18.7071C3 18.4142 3 17.9428 3 17V11C3 10.0572 3 9.58579 3.29289 9.29289C3.58579 9 4.05719 9 5 9C5.94281 9 6.41421 9 6.70711 9.29289C7 9.58579 7 10.0572 7 11V17C7 17.9428 7 18.4142 6.70711 18.7071C6.41421 19 5.94281 19 5 19Z M12 19C11.0572 19 10.5858 19 10.2929 18.7071C10 18.4142 10 17.9428 10 17V7C10 6.05719 10 5.58579 10.2929 5.29289C10.5858 5 11.0572 5 12 5C12.9428 5 13.4142 5 13.7071 5.29289C14 5.58579 14 6.05719 14 7V17C14 17.9428 14 18.4142 13.7071 18.7071C13.4142 19 12.9428 19 12 19Z M19 19C18.0572 19 17.5858 19 17.2929 18.7071C17 18.4142 17 17.9428 17 17V4C17 3.05719 17 2.58579 17.2929 2.29289C17.5858 2 18.0572 2 19 2C19.9428 2 20.4142 2 20.7071 2.29289C21 2.58579 21 3.05719 21 4V17C21 17.9428 21 18.4142 20.7071 18.7071C20.4142 19 19.9428 19 19 19Z",
			["check-circle"] = "M2 12A10 10 0 1 0 22 12A10 10 0 1 0 2 12Z M8.5 12.5L10.5 14.5L15.5 9.5",
			["clock-circle"] = "M2 12A10 10 0 1 0 22 12A10 10 0 1 0 2 12Z M12 8V12L14.5 14.5",
			["danger-triangle"] = "M11.6363 21H12.3637C17.7864 21 20.4978 21 21.5937 19.428C22.6897 17.856 21.4771 15.7061 19.0519 11.4063L18.6883 10.7615C15.7699 5.58716 14.3107 3 12 3C9.68925 3 8.23007 5.58716 5.31171 10.7615L4.94805 11.4063C2.52291 15.7061 1.31034 17.856 2.40626 19.428C3.50217 21 6.21356 21 11.6363 21Z M12 8V13 M12 16H12.0001",
			["download"] = "M17 9.00195C19.175 9.01406 20.3529 9.11051 21.1213 9.8789C22 10.7576 22 12.1718 22 15.0002V16.0002C22 18.8286 22 20.2429 21.1213 21.1215C20.2426 22.0002 18.8284 22.0002 16 22.0002H16H8C5.17157 22.0002 3.75736 22.0002 2.87868 21.1215C2 20.2429 2 18.8286 2 16.0002V15.0002C2 12.1718 2 10.7576 2.87868 9.87889C3.64706 9.11051 4.82497 9.01406 7 9.00195 M12 2L12 15M15 11.5L12 15L9 11.5",
			["folder"] = "M18 10L13 10 M11 4L10.4497 3.44975C10.1763 3.17633 10.0396 3.03961 9.89594 2.92051C9.27652 2.40704 8.51665 2.09229 7.71557 2.01738C7.52976 2 7.33642 2 6.94975 2C6.06722 2 5.62595 2 5.25839 2.06935C3.64031 2.37464 2.37464 3.64031 2.06935 5.25839C2 5.62595 2 6.06722 2 6.94975V14C2 17.7712 2 19.6569 3.17157 20.8284C4.34315 22 6.22876 22 10 22H14C17.7712 22 19.6569 22 20.8284 20.8284C22 19.6569 22 17.7712 22 14V11.7979C22 9.16554 22 7.84935 21.2305 6.99383C21.1598 6.91514 21.0849 6.84024 21.0062 6.76946C20.1506 6 18.8345 6 16.2021 6H15.8284C14.6747 6 14.0979 6 13.5604 5.84678C13.2651 5.7626 12.9804 5.64471 12.7121 5.49543C12.2237 5.22367 11.8158 4.81578 11 4Z",
			["home-2"] = "M7.88403 3.86687L5.88403 5.10813C3.98695 6.28551 3.0384 6.87421 2.5192 7.82274C2 8.77128 2 9.91549 2 12.2039V13.725C2 17.6258 2 19.5763 3.17157 20.7881C4.34315 22 6.22876 22 10 22H14C17.7712 22 19.6569 22 20.8284 20.7881C22 19.5763 22 17.6258 22 13.725V12.2039C22 9.91549 22 8.77128 21.4808 7.82274C20.9616 6.87421 20.0131 6.28551 18.116 5.10812L16.116 3.86687C14.1106 2.62229 13.1079 2 12 2C10.8921 2 9.88939 2.62229 7.88403 3.86687Z M12 15L12 18",
			["letter"] = "M10 20H14C17.7712 20 19.6569 20 20.8284 18.8284C22 17.6569 22 15.7712 22 12C22 8.22876 22 6.34315 20.8284 5.17157C19.6569 4 17.7712 4 14 4H10C6.22876 4 4.34315 4 3.17157 5.17157C2 6.34315 2 8.22876 2 12C2 15.7712 2 17.6569 3.17157 18.8284C4.34315 20 6.22876 20 10 20Z M6 8L8.1589 9.79908C9.99553 11.3296 10.9139 12.0949 12 12.0949C13.0861 12.0949 14.0045 11.3296 15.8411 9.79908L18 8",
			["logout-2"] = "M9.00195 7C9.01406 4.82497 9.11051 3.64706 9.87889 2.87868C10.7576 2 12.1718 2 15.0002 2L16.0002 2C18.8286 2 20.2429 2 21.1215 2.87868C22.0002 3.75736 22.0002 5.17157 22.0002 8L22.0002 16C22.0002 18.8284 22.0002 20.2426 21.1215 21.1213C20.2429 22 18.8286 22 16.0002 22H15.0002C12.1718 22 10.7576 22 9.87889 21.1213C9.11051 20.3529 9.01406 19.175 9.00195 17 M15 12L2 12M5.5 15L2 12L5.5 9",
			["magnifier"] = "M2 11.5A9.5 9.5 0 1 0 21 11.5A9.5 9.5 0 1 0 2 11.5Z M18.2168 18.2178L21.9994 22.0004",
			["moon"] = "M12 22C17.5228 22 22 17.5228 22 12C22 11.5373 21.3065 11.4608 21.0672 11.8568C19.9289 13.7406 17.8615 15 15.5 15C11.9101 15 9 12.0899 9 8.5C9 6.13845 10.2594 4.07105 12.1432 2.93276C12.5392 2.69347 12.4627 2 12 2C6.47715 2 2 6.47715 2 12C2 17.5228 6.47715 22 12 22Z",
			["settings"] = "M9 12A3 3 0 1 0 15 12A3 3 0 1 0 9 12Z M12.0005 2C12.9324 2 13.3983 2 13.7658 2.15224C14.2559 2.35523 14.6452 2.74458 14.8482 3.23463C14.9409 3.45833 14.9772 3.71848 14.9914 4.09794C15.0122 4.65566 15.2982 5.17191 15.7816 5.45096C16.2649 5.73002 16.855 5.71959 17.3484 5.4588C17.6841 5.28136 17.9276 5.18269 18.1676 5.15109C18.6935 5.08185 19.2254 5.22436 19.6462 5.54727C19.9618 5.78945 20.1948 6.19297 20.6607 7C21.1267 7.80703 21.3596 8.21055 21.4116 8.60497C21.4808 9.13087 21.3383 9.66273 21.0154 10.0835C20.868 10.2757 20.6608 10.4371 20.3392 10.6392C19.8667 10.9361 19.5626 11.4418 19.5626 11.9999C19.5626 12.558 19.8666 13.0639 20.3392 13.3608C20.6607 13.5628 20.8679 13.7243 21.0153 13.9164C21.3382 14.3372 21.4807 14.8691 21.4115 15.395C21.3595 15.7894 21.1266 16.1929 20.6606 16.9999C20.1947 17.807 19.9617 18.2105 19.6461 18.4527C19.2253 18.7756 18.6934 18.9181 18.1675 18.8488C17.9275 18.8172 17.6841 18.7186 17.3484 18.5412C16.8549 18.2804 16.2648 18.2699 15.7815 18.549C15.2982 18.8281 15.0122 19.3443 14.9914 19.902C14.9772 20.2815 14.9409 20.5417 14.8482 20.7654C14.6452 21.2554 14.2559 21.6448 13.7658 21.8478C13.3983 22 12.9324 22 12.0005 22C11.0686 22 10.6027 22 10.2351 21.8478C9.74505 21.6448 9.35571 21.2554 9.15272 20.7654C9.06006 20.5417 9.0238 20.2815 9.0096 19.9021C8.98874 19.3443 8.70273 18.8281 8.21939 18.549C7.73605 18.27 7.14596 18.2804 6.65253 18.5412C6.31681 18.7186 6.07338 18.8173 5.83333 18.8489C5.30743 18.9181 4.77557 18.7756 4.35475 18.4527C4.03913 18.2106 3.80616 17.807 3.34022 17C2.87428 16.193 2.64131 15.7894 2.58938 15.395C2.52015 14.8691 2.66266 14.3373 2.98557 13.9165C3.13297 13.7244 3.34013 13.5629 3.66167 13.3608C4.13424 13.0639 4.43832 12.5581 4.43835 12C4.43838 11.4419 4.13429 10.936 3.66167 10.639C3.34019 10.437 3.13305 10.2756 2.98566 10.0835C2.66276 9.66266 2.52025 9.1308 2.58948 8.60491C2.64141 8.21048 2.87438 7.80697 3.34032 6.99993C3.80626 6.1929 4.03923 5.78938 4.35485 5.5472C4.77567 5.22429 5.30753 5.08178 5.83342 5.15102C6.07349 5.18262 6.31693 5.2813 6.65268 5.45876C7.14608 5.71954 7.73612 5.72996 8.21943 5.45093C8.70274 5.17189 8.98875 4.65568 9.0096 4.09799C9.02379 3.7185 9.06005 3.45834 9.15272 3.23463C9.35571 2.74458 9.74505 2.35523 10.2351 2.15224C10.6027 2 11.0686 2 12.0005 2Z",
			["star"] = "M6.67589 7.62805L6.03954 7.77203C3.57986 8.32856 2.35002 8.60682 2.05742 9.54773C1.76482 10.4886 2.60325 11.4691 4.2801 13.4299L4.71392 13.9372C5.19043 14.4944 5.42868 14.773 5.53586 15.1177C5.64305 15.4624 5.60703 15.8341 5.53498 16.5776L5.4694 17.2544C5.21588 19.8706 5.08912 21.1787 5.85515 21.7602C6.62118 22.3417 7.77268 21.8115 10.0757 20.7512L10.6715 20.4768C11.3259 20.1755 11.6531 20.0248 12 20.0248C12.3469 20.0248 12.6741 20.1755 13.3285 20.4768L13.9243 20.7512C16.2273 21.8115 17.3788 22.3417 18.1449 21.7602C18.9109 21.1787 18.7841 19.8706 18.5306 17.2544L18.465 16.5776C18.393 15.8341 18.357 15.4624 18.4641 15.1177C18.5713 14.773 18.8096 14.4944 19.2861 13.9372L19.7199 13.4299C21.3968 11.4691 22.2352 10.4886 21.9426 9.54773C21.65 8.60682 20.4201 8.32856 17.9605 7.77203L17.3241 7.62805C16.6251 7.4699 16.2757 7.39083 15.9951 7.17781C15.7144 6.96479 15.5345 6.64193 15.1745 5.99623L14.8468 5.40837C13.5802 3.13612 12.9469 2 12 2C11.0531 2 10.4198 3.13613 9.15316 5.40838L8.82547 5.99623C8.46553 6.64194 8.28555 6.96479 8.00494 7.17781C7.72433 7.39083 7.37485 7.4699 6.67589 7.62805Z",
			["sun"] = "M6 12A6 6 0 1 0 18 12A6 6 0 1 0 6 12Z M12 2V3 M12 21V22 M22 12L21 12 M3 12L2 12 M19.0703 4.9292L18.6775 5.32204 M5.32227 18.6782L4.92943 19.0711 M19.0703 19.0708L18.6775 18.678 M5.32227 5.32178L4.92943 4.92894",
			["user-rounded"] = "M8 6A4 4 0 1 0 16 6A4 4 0 1 0 8 6Z M5 17A7 4 0 1 0 19 17A7 4 0 1 0 5 17Z",
			["users-group-rounded"] = "M5 6A4 4 0 1 0 13 6A4 4 0 1 0 5 6Z M15 9C16.6569 9 18 7.65685 18 6C18 4.34315 16.6569 3 15 3 M2 17A7 4 0 1 0 16 17A7 4 0 1 0 2 17Z M18 14C19.7542 14.3847 21 15.3589 21 16.5C21 17.5293 19.9863 18.4229 18.5 18.8704",
			["widget-4"] = "M6.5 21.5C4.61438 21.5 3.67157 21.5 3.08579 20.9142C2.5 20.3284 2.5 19.3856 2.5 17.5V6.5C2.5 4.61438 2.5 3.67157 3.08579 3.08579C3.67157 2.5 4.61438 2.5 6.5 2.5C8.38562 2.5 9.32843 2.5 9.91421 3.08579C10.5 3.67157 10.5 4.61438 10.5 6.5V17.5C10.5 19.3856 10.5 20.3284 9.91421 20.9142C9.32843 21.5 8.38562 21.5 6.5 21.5Z M17.5 21.5C15.6144 21.5 14.6716 21.5 14.0858 20.9142C13.5 20.3284 13.5 19.3856 13.5 17.5V15.5C13.5 13.6144 13.5 12.6716 14.0858 12.0858C14.6716 11.5 15.6144 11.5 17.5 11.5C19.3856 11.5 20.3284 11.5 20.9142 12.0858C21.5 12.6716 21.5 13.6144 21.5 15.5V17.5C21.5 19.3856 21.5 20.3284 20.9142 20.9142C20.3284 21.5 19.3856 21.5 17.5 21.5Z M16.5 8.5H18.5C19.4319 8.5 19.8978 8.5 20.2654 8.34776C20.7554 8.14477 21.1448 7.75542 21.3478 7.26537C21.5 6.89782 21.5 6.43188 21.5 5.5C21.5 4.56812 21.5 4.10218 21.3478 3.73463C21.1448 3.24458 20.7554 2.85523 20.2654 2.65224C19.8978 2.5 19.4319 2.5 18.5 2.5H16.5C15.5681 2.5 15.1022 2.5 14.7346 2.65224C14.2446 2.85523 13.8552 3.24458 13.6522 3.73463C13.5 4.10218 13.5 4.56812 13.5 5.5C13.5 6.43188 13.5 6.89782 13.6522 7.26537C13.8552 7.75542 14.2446 8.14477 14.7346 8.34776C15.1022 8.5 15.5681 8.5 16.5 8.5Z",
		},
	}

	local ICON_GRID = 24
	local ICON_CHORD = 2.4
	local ICON_MIN_SEG = 0.28
	local ICON_COLLINEAR = 0.045

	local ICON_FILL = 1.14

	local ICON_STROKE = { lucide = 2, solar = 1.5 }

	local ICON_WEIGHT = { lucide = 1.05, solar = 1.3 }

	local ICON_ALIAS = {
		grid = "lucide:layout-grid",
		chart = "lucide:chart-column",
		users = "lucide:users",
		settings = "lucide:settings",
		search = "lucide:search",
		close = "lucide:x",
		minus = "lucide:minus",
		expand = "lucide:maximize",
	}

	local PATH_CMD = {}
	for ch in string.gmatch("MmLlHhVvCcSsQqTtAaZz", ".") do
		PATH_CMD[ch] = true
	end

	local function pathReader(d)
		local i = 1
		local n = #d

		local function skipSep()
			while i <= n do
				local ch = string.sub(d, i, i)
				if ch == " " or ch == "," or ch == "\n" or ch == "\t" or ch == "\r" then
					i = i + 1
				else
					return
				end
			end
		end

		local function peek()
			skipSep()
			local ch = string.sub(d, i, i)
			if PATH_CMD[ch] then return ch end
			return nil
		end

		local function take()
			local ch = peek()
			if ch then i = i + 1 end
			return ch
		end

		local function num()
			skipSep()
			local start = i
			local ch = string.sub(d, i, i)
			if ch == "+" or ch == "-" then i = i + 1 end
			local digit = false
			while i <= n do
				ch = string.sub(d, i, i)
				if ch >= "0" and ch <= "9" then
					digit = true
					i = i + 1
				else
					break
				end
			end
			if string.sub(d, i, i) == "." then
				i = i + 1
				while i <= n do
					ch = string.sub(d, i, i)
					if ch >= "0" and ch <= "9" then
						digit = true
						i = i + 1
					else
						break
					end
				end
			end
			if not digit then
				i = start
				return nil
			end
			ch = string.sub(d, i, i)
			if ch == "e" or ch == "E" then
				local save = i
				i = i + 1
				ch = string.sub(d, i, i)
				if ch == "+" or ch == "-" then i = i + 1 end
				local expDigit = false
				while i <= n do
					ch = string.sub(d, i, i)
					if ch >= "0" and ch <= "9" then
						expDigit = true
						i = i + 1
					else
						break
					end
				end
				if not expDigit then i = save end
			end
			return tonumber(string.sub(d, start, i - 1))
		end

		return { num = num, take = take, peek = peek }
	end

	local function flattenPath(d)
		local r = pathReader(d)
		local polys = {}
		local cur = nil
		local x, y = 0, 0
		local sx, sy = 0, 0
		local pcx, pcy = 0, 0
		local lastCubic, lastQuad = false, false
		local cmd = nil

		local function push(px, py)
			if cur then cur[#cur + 1] = { px, py } end
		end

		local function begin(px, py)
			cur = { { px, py } }
			polys[#polys + 1] = cur
		end

		local function stepsFor(len)
			local s = math.ceil(len / ICON_CHORD)
			if s < 1 then s = 1 end
			if s > 24 then s = 24 end
			return s
		end

		local function cubic(x1, y1, x2, y2, x3, y3)
			local len = math.sqrt((x2 - x) ^ 2 + (y2 - y) ^ 2)
				+ math.sqrt((x3 - x2) ^ 2 + (y3 - y2) ^ 2)
				+ math.sqrt((x1 - x) ^ 2 + (y1 - y) ^ 2)
			local n = stepsFor(len)
			for k = 1, n do
				local t = k / n
				local u = 1 - t
				push(
					u * u * u * x + 3 * u * u * t * x1 + 3 * u * t * t * x2 + t * t * t * x3,
					u * u * u * y + 3 * u * u * t * y1 + 3 * u * t * t * y2 + t * t * t * y3
				)
			end
		end

		local function quad(x1, y1, x2, y2)
			local len = math.sqrt((x1 - x) ^ 2 + (y1 - y) ^ 2) + math.sqrt((x2 - x1) ^ 2 + (y2 - y1) ^ 2)
			local n = stepsFor(len)
			for k = 1, n do
				local t = k / n
				local u = 1 - t
				push(u * u * x + 2 * u * t * x1 + t * t * x2, u * u * y + 2 * u * t * y1 + t * t * y2)
			end
		end

		local function arc(rx, ry, rot, large, sweep, x2, y2)
			local arx, ary = math.abs(rx), math.abs(ry)
			if arx == 0 or ary == 0 then
				push(x2, y2)
				return
			end
			local phi = math.rad(rot)
			local cosP, sinP = math.cos(phi), math.sin(phi)
			local dx, dy = (x - x2) / 2, (y - y2) / 2
			local x1p = cosP * dx + sinP * dy
			local y1p = -sinP * dx + cosP * dy

			local lambda = (x1p * x1p) / (arx * arx) + (y1p * y1p) / (ary * ary)
			if lambda > 1 then
				local k = math.sqrt(lambda)
				arx = arx * k
				ary = ary * k
			end

			local sign = 1
			if large == sweep then sign = -1 end
			local denom = arx * arx * y1p * y1p + ary * ary * x1p * x1p
			local numer = arx * arx * ary * ary - arx * arx * y1p * y1p - ary * ary * x1p * x1p
			local coef = 0
			if denom > 0 then coef = sign * math.sqrt(math.max(0, numer / denom)) end

			local cxp = (coef * arx * y1p) / ary
			local cyp = (-coef * ary * x1p) / arx
			local cx = cosP * cxp - sinP * cyp + (x + x2) / 2
			local cy = sinP * cxp + cosP * cyp + (y + y2) / 2

			local function angle(ux, uy, vx, vy)
				local dot = ux * vx + uy * vy
				local len = math.sqrt(ux * ux + uy * uy) * math.sqrt(vx * vx + vy * vy)
				local a = 0
				if len > 0 then a = math.acos(math.clamp(dot / len, -1, 1)) end
				if ux * vy - uy * vx < 0 then a = -a end
				return a
			end

			local ux, uy = (x1p - cxp) / arx, (y1p - cyp) / ary
			local vx, vy = (-x1p - cxp) / arx, (-y1p - cyp) / ary
			local theta1 = angle(1, 0, ux, uy)
			local dTheta = angle(ux, uy, vx, vy)
			if not sweep and dTheta > 0 then dTheta = dTheta - 2 * math.pi end
			if sweep and dTheta < 0 then dTheta = dTheta + 2 * math.pi end

			local steps = math.ceil(math.abs(dTheta) * math.max(arx, ary) / ICON_CHORD)
			if steps < 2 then steps = 2 end
			if steps > 32 then steps = 32 end
			for k = 1, steps do
				local t = theta1 + dTheta * k / steps
				local ct, st = math.cos(t), math.sin(t)
				push(cx + arx * ct * cosP - ary * st * sinP, cy + arx * ct * sinP + ary * st * cosP)
			end
		end

		local guard = 0
		while guard < 20000 do
			guard = guard + 1
			local nextCmd = r.peek()
			if nextCmd then
				cmd = r.take()
			elseif not cmd then
				break
			end

			local C = string.upper(cmd)
			local rel = cmd ~= C

			if C == "M" then
				local a, b = r.num(), r.num()
				if a == nil or b == nil then break end
				if rel then
					x, y = x + a, y + b
				else
					x, y = a, b
				end
				sx, sy = x, y
				begin(x, y)

				cmd = rel and "l" or "L"
				lastCubic, lastQuad = false, false
			elseif C == "Z" then
				if cur then push(sx, sy) end
				cur = nil
				x, y = sx, sy
				cmd = nil
				lastCubic, lastQuad = false, false
			elseif C == "L" then
				local a, b = r.num(), r.num()
				if a == nil or b == nil then break end
				local nx = rel and (x + a) or a
				local ny = rel and (y + b) or b
				if not cur then begin(x, y) end
				push(nx, ny)
				x, y = nx, ny
				lastCubic, lastQuad = false, false
			elseif C == "H" then
				local a = r.num()
				if a == nil then break end
				local nx = rel and (x + a) or a
				if not cur then begin(x, y) end
				push(nx, y)
				x = nx
				lastCubic, lastQuad = false, false
			elseif C == "V" then
				local a = r.num()
				if a == nil then break end
				local ny = rel and (y + a) or a
				if not cur then begin(x, y) end
				push(x, ny)
				y = ny
				lastCubic, lastQuad = false, false
			elseif C == "C" then
				local a, b, c, e, f, g = r.num(), r.num(), r.num(), r.num(), r.num(), r.num()
				if g == nil then break end
				local x1 = rel and (x + a) or a
				local y1 = rel and (y + b) or b
				local x2 = rel and (x + c) or c
				local y2 = rel and (y + e) or e
				local x3 = rel and (x + f) or f
				local y3 = rel and (y + g) or g
				if not cur then begin(x, y) end
				cubic(x1, y1, x2, y2, x3, y3)
				pcx, pcy = x2, y2
				x, y = x3, y3
				lastCubic, lastQuad = true, false
			elseif C == "S" then
				local a, b, c, e = r.num(), r.num(), r.num(), r.num()
				if e == nil then break end
				local x1 = lastCubic and (2 * x - pcx) or x
				local y1 = lastCubic and (2 * y - pcy) or y
				local x2 = rel and (x + a) or a
				local y2 = rel and (y + b) or b
				local x3 = rel and (x + c) or c
				local y3 = rel and (y + e) or e
				if not cur then begin(x, y) end
				cubic(x1, y1, x2, y2, x3, y3)
				pcx, pcy = x2, y2
				x, y = x3, y3
				lastCubic, lastQuad = true, false
			elseif C == "Q" then
				local a, b, c, e = r.num(), r.num(), r.num(), r.num()
				if e == nil then break end
				local x1 = rel and (x + a) or a
				local y1 = rel and (y + b) or b
				local x2 = rel and (x + c) or c
				local y2 = rel and (y + e) or e
				if not cur then begin(x, y) end
				quad(x1, y1, x2, y2)
				pcx, pcy = x1, y1
				x, y = x2, y2
				lastCubic, lastQuad = false, true
			elseif C == "T" then
				local a, b = r.num(), r.num()
				if b == nil then break end
				local x1 = lastQuad and (2 * x - pcx) or x
				local y1 = lastQuad and (2 * y - pcy) or y
				local x2 = rel and (x + a) or a
				local y2 = rel and (y + b) or b
				if not cur then begin(x, y) end
				quad(x1, y1, x2, y2)
				pcx, pcy = x1, y1
				x, y = x2, y2
				lastCubic, lastQuad = false, true
			elseif C == "A" then
				local rx, ry, rot = r.num(), r.num(), r.num()
				local large, sweep = r.num(), r.num()
				local ex, ey = r.num(), r.num()
				if ey == nil then break end
				local nx = rel and (x + ex) or ex
				local ny = rel and (y + ey) or ey
				if not cur then begin(x, y) end
				arc(rx, ry, rot, large ~= 0, sweep ~= 0, nx, ny)
				x, y = nx, ny
				lastCubic, lastQuad = false, false
			else
				break
			end
		end

		local out = {}
		local minSq = ICON_MIN_SEG * ICON_MIN_SEG
		for _, poly in ipairs(polys) do
			local pts = {}
			for _, p in ipairs(poly) do
				local last = pts[#pts]
				if not last then
					pts[1] = p
				else
					local dx, dy = p[1] - last[1], p[2] - last[2]
					if dx * dx + dy * dy >= minSq then pts[#pts + 1] = p end
				end
			end

			local keep = {}
			local m = #pts
			for i = 1, m do
				if i == 1 or i == m then
					keep[#keep + 1] = pts[i]
				else
					local a = keep[#keep]
					local b = pts[i]
					local c = pts[i + 1]
					local vx1, vy1 = b[1] - a[1], b[2] - a[2]
					local vx2, vy2 = c[1] - b[1], c[2] - b[2]
					local l1 = math.sqrt(vx1 * vx1 + vy1 * vy1)
					local l2 = math.sqrt(vx2 * vx2 + vy2 * vy2)
					local cross = 1
					if l1 * l2 > 0 then cross = math.abs(vx1 * vy2 - vy1 * vx2) / (l1 * l2) end
					if cross >= ICON_COLLINEAR then keep[#keep + 1] = b end
				end
			end

			if #keep >= 2 then out[#out + 1] = keep end
		end

		return out
	end

	local iconCache = {}

	local function iconKeyParts(key)
		local sep = string.find(key, ":", 1, true)
		if not sep then return "lucide", key end
		return string.sub(key, 1, sep - 1), string.sub(key, sep + 1)
	end

	local function iconPolys(key)
		local cached = iconCache[key]
		if cached then return cached end
		local set, name = iconKeyParts(key)
		local pack = ICON_DATA[set]
		local d = pack and pack[name]
		if not d or d == "" then return nil end
		local polys = flattenPath(d)
		iconCache[key] = polys
		return polys
	end

	local iconMeta = setmetatable({}, { __mode = "k" })
	local iconTex = {}

	local ICON_TEX_SCALE = 4
	local ICON_TEX_MIN = 32
	local ICON_TEX_MAX = 192
	local ICON_MIN_BAR = 3.5

	local rasterSupport
	local function rasterAvailable()
		if rasterSupport == nil then
			rasterSupport = false
			local ok, probe = pcall(
				function() return AssetService:CreateEditableImage({ Size = Vector2.new(8, 8) }) end
			)
			if ok and probe then
				probe:Destroy()
				rasterSupport = true
			end
		end
		return rasterSupport
	end

	local function rasterIcon(key, texPx)
		local set = iconKeyParts(key)
		local polys = iconPolys(key)
		if not polys then return nil end

		local s = (texPx / ICON_GRID) * ICON_FILL
		local origin = texPx / 2 - (ICON_GRID / 2) * s
		local strokePx = (ICON_STROKE[set] or 2) * s * (ICON_WEIGHT[set] or 1)
		if strokePx < 2 then strokePx = 2 end
		local half = strokePx * 0.5

		local cover = buffer.create(texPx * texPx)

		for _, poly in ipairs(polys) do
			for i = 1, #poly - 1 do
				local a, b = poly[i], poly[i + 1]
				local ax, ay = origin + a[1] * s, origin + a[2] * s
				local bx, by = origin + b[1] * s, origin + b[2] * s
				local ex, ey = bx - ax, by - ay
				local lenSq = ex * ex + ey * ey
				if lenSq > 1e-6 then
					local minX = math.max(0, math.floor(math.min(ax, bx) - half - 1))
					local maxX = math.min(texPx - 1, math.ceil(math.max(ax, bx) + half + 1))
					local minY = math.max(0, math.floor(math.min(ay, by) - half - 1))
					local maxY = math.min(texPx - 1, math.ceil(math.max(ay, by) + half + 1))
					for y = minY, maxY do
						local py = y + 0.5
						local rowBase = y * texPx
						for x = minX, maxX do
							local px = x + 0.5
							local t = ((px - ax) * ex + (py - ay) * ey) / lenSq
							if t < 0 then
								t = 0
							elseif t > 1 then
								t = 1
							end
							local dx = px - (ax + ex * t)
							local dy = py - (ay + ey * t)
							local d = math.sqrt(dx * dx + dy * dy)

							local c = half + 0.5 - d
							if c > 0 then
								if c > 1 then c = 1 end
								local idx = rowBase + x
								local v = math.floor(c * 255 + 0.5)
								if v > buffer.readu8(cover, idx) then buffer.writeu8(cover, idx, v) end
							end
						end
					end
				end
			end
		end

		local px = texPx * texPx
		local rgba = buffer.create(px * 4)
		for i = 0, px - 1 do
			local o = i * 4
			buffer.writeu8(rgba, o, 255)
			buffer.writeu8(rgba, o + 1, 255)
			buffer.writeu8(rgba, o + 2, 255)
			buffer.writeu8(rgba, o + 3, buffer.readu8(cover, i))
		end

		local image = AssetService:CreateEditableImage({ Size = Vector2.new(texPx, texPx) })
		if not image then return nil end
		image:WritePixelsBuffer(Vector2.zero, Vector2.new(texPx, texPx), rgba)
		return image
	end

	local function iconTexture(key, texPx)
		local bySize = iconTex[texPx]
		if not bySize then
			bySize = {}
			iconTex[texPx] = bySize
		end
		local hit = bySize[key]
		if hit ~= nil then
			if hit == false then return nil end
			return hit
		end
		local ok, image = pcall(rasterIcon, key, texPx)
		if not ok or not image then
			bySize[key] = false
			return nil
		end
		bySize[key] = image
		return image
	end

	local function drawFrames(root, key, size, color)
		local set = iconKeyParts(key)
		local polys = iconPolys(key)
		local s = (size / ICON_GRID) * ICON_FILL
		local origin = size / 2 - (ICON_GRID / 2) * s
		local thickness = (ICON_STROKE[set] or 2) * s * (ICON_WEIGHT[set] or 1)
		if thickness < 1 then thickness = 1 end

		for _, poly in ipairs(polys) do
			local i = 1
			while i < #poly do
				local a = poly[i]
				local j = i + 1

				while j < #poly do
					local nx, ny = poly[j + 1][1] - a[1], poly[j + 1][2] - a[2]
					if math.sqrt(nx * nx + ny * ny) * s >= ICON_MIN_BAR then break end
					j = j + 1
				end
				local b = poly[j]
				local dx, dy = b[1] - a[1], b[2] - a[2]
				local len = math.sqrt(dx * dx + dy * dy) * s
				if len > 0.02 then
					local seg = new("Frame", {
						Size = UDim2.fromOffset(len, thickness),
						Position = UDim2.fromOffset(origin + (a[1] + b[1]) * 0.5 * s, origin + (a[2] + b[2]) * 0.5 * s),
						AnchorPoint = Vector2.new(0.5, 0.5),
						BackgroundColor3 = color,
						BorderSizePixel = 0,
						Rotation = math.deg(math.atan2(dy, dx)),
					}, root)
					corner(seg, thickness / 2)
				end
				i = j
			end
		end
	end

	local function drawIcon(root, key, size, color)
		for _, child in ipairs(root:GetChildren()) do
			child:Destroy()
		end
		iconMeta[root] = { key = key, size = size, color = color }

		if not iconPolys(key) then
			local missing = new("Frame", {
				Size = UDim2.fromOffset(size * 0.35, size * 0.35),
				Position = UDim2.fromScale(0.5, 0.5),
				AnchorPoint = Vector2.new(0.5, 0.5),
				BackgroundColor3 = color,
				BackgroundTransparency = 0.4,
				BorderSizePixel = 0,
			}, root)
			corner(missing, 99)
			return root
		end

		if rasterAvailable() then
			local texPx = math.clamp(math.ceil(size * ICON_TEX_SCALE), ICON_TEX_MIN, ICON_TEX_MAX)
			local image = iconTexture(key, texPx)
			if image then
				local ok = pcall(function()
					local glyph = new("ImageLabel", {
						Name = "Glyph",
						Size = UDim2.fromScale(1, 1),
						BackgroundTransparency = 1,
						BorderSizePixel = 0,
						ScaleType = Enum.ScaleType.Fit,
						ImageColor3 = color,
					}, root)
					glyph.ImageContent = Content.fromObject(image)
				end)
				if ok then return root end
				for _, child in ipairs(root:GetChildren()) do
					child:Destroy()
				end
			end
		end

		drawFrames(root, key, size, color)
		return root
	end

	local iconRegistry = {}
	local iconEntry = setmetatable({}, { __mode = "k" })

	local function makeIcon(kind, parent, size, color)
		local key = ICON_ALIAS[kind] or kind

		if #iconRegistry > 96 then
			local kept = {}
			for _, e in ipairs(iconRegistry) do
				if e.root.Parent then kept[#kept + 1] = e end
			end
			iconRegistry = kept
		end
		local root = new("Frame", {
			Size = UDim2.fromOffset(size, size),
			BackgroundTransparency = 1,
		}, parent)
		local entry = { root = root, key = key }
		iconRegistry[#iconRegistry + 1] = entry
		iconEntry[root] = entry
		return drawIcon(root, key, size, color)
	end

	local function tintIcon(iconRoot, color)
		local meta = iconMeta[iconRoot]
		if meta then meta.color = color end
		local glyph = iconRoot:FindFirstChild("Glyph")
		if glyph and glyph:IsA("ImageLabel") then
			glyph.ImageColor3 = color
			return
		end
		for _, d in ipairs(iconRoot:GetDescendants()) do
			if d:IsA("Frame") then
				d.BackgroundColor3 = color
			elseif d:IsA("UIStroke") then
				d.Color = color
			end
		end
	end

	local ICON_TWIN = {
		house = "home-2",
		["layout-grid"] = "widget-4",
		["chart-column"] = "chart-2",
		users = "users-group-rounded",
		search = "magnifier",
		mail = "letter",
		user = "user-rounded",
		zap = "bolt",
		check = "check-circle",
	}

	local ICON_TWIN_REVERSE = {}
	for lucideName, solarName in pairs(ICON_TWIN) do
		ICON_TWIN_REVERSE[solarName] = lucideName
	end

	local ICON_PACK = "lucide"

	for _, old in ipairs(playerGui:GetChildren()) do
		if old.Name == "NexusDashboard" or old.Name == "RideGO_OptionC" then old:Destroy() end
	end

	local screenGui = new("ScreenGui", {
		Name = "NexusDashboard",
		ResetOnSpawn = false,
		IgnoreGuiInset = true,
		ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
		DisplayOrder = 50,
	}, playerGui)

	local shadowGroup = new("Frame", {
		Name = "ShadowGroup",
		Size = UDim2.fromOffset(0, 0),
		Position = UDim2.fromScale(0.5, 0.5),
		AnchorPoint = Vector2.new(0.5, 0.5),
		BackgroundTransparency = 1,
		Visible = false,
	}, screenGui)

	local shadowLayer = function(expand, transparency)
		return new("Frame", {
			Size = UDim2.fromOffset(1180 + expand, 720 + expand),
			Position = UDim2.fromScale(0.5, 0.5),
			AnchorPoint = Vector2.new(0.5, 0.5),
			BackgroundColor3 = Color3.new(0, 0, 0),
			BackgroundTransparency = transparency,
			BorderSizePixel = 0,
		}, shadowGroup)
	end

	local SHADOW_STEPS = {
		{ 96, 0.90, 48 },
		{ 68, 0.84, 40 },
		{ 46, 0.74, 32 },
		{ 28, 0.62, 26 },
		{ 14, 0.44, 22 },
	}

	local shadowLayerList = {}
	for i, step in ipairs(SHADOW_STEPS) do
		local layer = shadowLayer(step[1], step[2])
		corner(layer, step[3])
		shadowLayerList[i] = layer
	end

	local app = new("Frame", {
		Name = "Window",
		Size = UDim2.fromOffset(1180, 720),
		Position = UDim2.fromScale(0.5, 0.5),
		AnchorPoint = Vector2.new(0.5, 0.5),
		BackgroundColor3 = C.bg,
		BorderSizePixel = 0,
		ClipsDescendants = true,
		Visible = false,
	}, screenGui)
	corner(app, 20)
	stroke(app, C.white, 0.94, 1)

	local uiScale = new("UIScale", { Scale = 1 }, app)

	local TITLEBAR_H = 60
	local RAIL_W = 236
	local RAIL_W_COMPACT = 76

	local RESP = {
		navItems = {},
		groupLabels = {},
		groups = {},
		statCards = {},
		settings = {},
		ready = false,
		mode = "wide",
	}
	local STAT_H = 104
	local STAT_GAP = 16

	local winSize = Vector2.new(1180, 720)
	local sizeInitialised = false

	local MIN_W = 560
	local MIN_H = 400

	local windowState = "normal"

	local windowOffset = Vector2.new(0, 0)

	local KEEP_VISIBLE_X = 180
	local KEEP_VISIBLE_Y = 80

	local function applyWindowOffset()
		local pos = UDim2.new(0.5, windowOffset.X, 0.5, windowOffset.Y)
		app.Position = pos
		shadowGroup.Position = pos
	end

	local function dragLimits()
		local cam = workspace.CurrentCamera
		local vp = cam and cam.ViewportSize or Vector2.new(1280, 720)
		local scale = uiScale.Scale
		local w = app.Size.X.Offset * scale
		local h = app.Size.Y.Offset * scale
		return math.max(0, (vp.X + w) / 2 - KEEP_VISIBLE_X), math.max(0, (vp.Y + h) / 2 - KEEP_VISIBLE_Y)
	end

	local function clampWindowOffset()
		local maxX, maxY = dragLimits()
		windowOffset =
			Vector2.new(math.clamp(windowOffset.X, -maxX, maxX), math.clamp(windowOffset.Y, -maxY, maxY))
		applyWindowOffset()
	end

	local dragging = false

	local resizing = false
	local dragConn = nil
	local lastMouse = Vector2.new(0, 0)

	local function stepDrag(dx, dy)
		local maxX, maxY = dragLimits()
		local scale = uiScale.Scale
		windowOffset = Vector2.new(
			math.clamp(windowOffset.X + dx / scale, -maxX, maxX),
			math.clamp(windowOffset.Y + dy / scale, -maxY, maxY)
		)
		applyWindowOffset()
	end

	local function beginDrag()
		if windowState == "full" then return end
		if dragging or resizing then return end
		dragging = true
		lastMouse = UIS:GetMouseLocation()
		play(uiScale, MOTION.fast, { Scale = 0.992 })

		dragConn = RunService.RenderStepped:Connect(function()
			if not dragging then return end
			local now = UIS:GetMouseLocation()
			local d = now - lastMouse
			lastMouse = now
			if d.X ~= 0 or d.Y ~= 0 then stepDrag(d.X, d.Y) end
		end)
	end

	local function endDrag()
		if not dragging then return end
		dragging = false
		if dragConn then
			dragConn:Disconnect()
			dragConn = nil
		end
		play(uiScale, MOTION.fast, { Scale = 1 })
		clampWindowOffset()
	end

	local function bindDrag(guiObject)
		guiObject.InputBegan:Connect(function(input)
			local t = input.UserInputType
			if t == Enum.UserInputType.MouseButton1 or t == Enum.UserInputType.Touch then beginDrag() end
		end)
	end

	UIS.InputChanged:Connect(function(input)
		if not dragging then return end
		if input.UserInputType ~= Enum.UserInputType.Touch then return end
		stepDrag(input.Delta.X, input.Delta.Y)
	end)

	local function syncShadow()
		local w = app.Size.X.Offset
		local h = app.Size.Y.Offset
		for i, step in ipairs(SHADOW_STEPS) do
			shadowLayerList[i].Size = UDim2.fromOffset(w + step[1], h + step[1])
		end
	end

	local function maxWindowSize()
		local cam = workspace.CurrentCamera
		local vp = cam and cam.ViewportSize or Vector2.new(1280, 720)
		return math.max(MIN_W, vp.X - 40), math.max(MIN_H, vp.Y - 40)
	end

	local function applyWindowSize()
		local maxW, maxH = maxWindowSize()
		winSize = Vector2.new(math.clamp(winSize.X, MIN_W, maxW), math.clamp(winSize.Y, MIN_H, maxH))
		app.Size = UDim2.fromOffset(winSize.X, winSize.Y)
		syncShadow()
		if RESP.applyLayout then RESP.applyLayout() end
	end

	local function fitToViewport()
		local cam = workspace.CurrentCamera
		if not cam then return end
		local vp = cam.ViewportSize

		if not sizeInitialised then
			winSize =
				Vector2.new(math.min(1180, math.max(MIN_W, vp.X - 80)), math.min(720, math.max(MIN_H, vp.Y - 80)))
			sizeInitialised = true
		else
			winSize = Vector2.new(
				math.min(winSize.X, math.max(MIN_W, vp.X - 80)),
				math.min(winSize.Y, math.max(MIN_H, vp.Y - 80))
			)
		end

		if windowState == "normal" then applyWindowSize() end
		clampWindowOffset()
	end
	fitToViewport()
	do
		local cam = workspace.CurrentCamera
		if cam then cam:GetPropertyChangedSignal("ViewportSize"):Connect(fitToViewport) end
	end
	app:GetPropertyChangedSignal("Size"):Connect(syncShadow)

	local resizeConn = nil
	local resizeGrabCursor = Vector2.new(0, 0)
	local resizeGrabSize = Vector2.new(0, 0)
	local resizeTouchTotal = Vector2.new(0, 0)

	local function resizeTo(pointer)
		local maxW, maxH = maxWindowSize()
		local target = Vector2.new(
			math.clamp(resizeGrabSize.X + pointer.X, MIN_W, maxW),
			math.clamp(resizeGrabSize.Y + pointer.Y, MIN_H, maxH)
		)

		local dW = target.X - winSize.X
		local dH = target.Y - winSize.Y
		if dW == 0 and dH == 0 then return end

		winSize = target
		applyWindowSize()

		windowOffset = Vector2.new(windowOffset.X + dW / 2, windowOffset.Y + dH / 2)
		applyWindowOffset()
	end

	local function beginResize()
		if windowState ~= "normal" then return end
		if resizing or dragging then return end
		resizing = true
		resizeGrabCursor = UIS:GetMouseLocation()
		resizeGrabSize = winSize
		resizeTouchTotal = Vector2.new(0, 0)

		resizeConn = RunService.RenderStepped:Connect(function()
			if not resizing then return end
			local now = UIS:GetMouseLocation()
			resizeTo((now - resizeGrabCursor) / uiScale.Scale)
		end)
	end

	local function endResize()
		if not resizing then return end
		resizing = false
		if resizeConn then
			resizeConn:Disconnect()
			resizeConn = nil
		end
		clampWindowOffset()
	end

	local function bindResize(guiObject)
		guiObject.InputBegan:Connect(function(input)
			local t = input.UserInputType
			if t == Enum.UserInputType.MouseButton1 or t == Enum.UserInputType.Touch then beginResize() end
		end)
	end

	UIS.InputChanged:Connect(function(input)
		if not resizing then return end
		if input.UserInputType ~= Enum.UserInputType.Touch then return end
		resizeTouchTotal = resizeTouchTotal + input.Delta / uiScale.Scale
		resizeTo(resizeTouchTotal)
	end)

	UIS.InputEnded:Connect(function(input)
		local t = input.UserInputType
		if t == Enum.UserInputType.MouseButton1 or t == Enum.UserInputType.Touch then
			endDrag()
			endResize()
		end
	end)

	local function setShadowsVisible(state) shadowGroup.Visible = state end

	local function fadeShadows(opening)
		for i, layer in ipairs(shadowLayerList) do
			if opening then
				layer.BackgroundTransparency = 1
				play(layer, MOTION.base, { BackgroundTransparency = SHADOW_STEPS[i][2] })
			else
				play(layer, MOTION.fast, { BackgroundTransparency = 1 })
			end
		end
	end

	local titlebar = new("Frame", {
		Name = "Titlebar",
		Size = UDim2.new(1, 0, 0, TITLEBAR_H),
		BackgroundTransparency = 1,
		Active = true,
	}, app)

	local dragHandle = new("TextButton", {
		Name = "DragHandle",
		Size = UDim2.new(1, 0, 1, 0),
		BackgroundTransparency = 1,
		Text = "",
		AutoButtonColor = false,
		Selectable = false,
	}, titlebar)

	local mark = new("Frame", {
		Size = UDim2.fromOffset(28, 28),
		Position = UDim2.fromOffset(22, 16),
		BackgroundColor3 = C.panel2,
		BorderSizePixel = 0,
	}, titlebar)
	corner(mark, 8)
	stroke(mark, C.white, 0.9)
	local markInner = new("Frame", {
		Size = UDim2.fromOffset(10, 10),
		Position = UDim2.fromScale(0.5, 0.5),
		AnchorPoint = Vector2.new(0.5, 0.5),
		Rotation = 45,
		BorderSizePixel = 0,
	}, mark)
	corner(markInner, 2)
	onAccent(function(c) markInner.BackgroundColor3 = c end)

	local titleRow = new("Frame", {
		Size = UDim2.new(0, 0, 0, 18),
		AutomaticSize = Enum.AutomaticSize.X,
		Position = UDim2.fromOffset(60, 13),
		BackgroundTransparency = 1,
	}, titlebar)
	new("UIListLayout", {
		FillDirection = Enum.FillDirection.Horizontal,
		Padding = UDim.new(0, 8),
		VerticalAlignment = Enum.VerticalAlignment.Center,
		SortOrder = Enum.SortOrder.LayoutOrder,
	}, titleRow)

	label(titleRow, {
		Text = "Testing",
		Font = FONT_BOLD,
		TextSize = 14,
		Size = UDim2.new(0, 0, 0, 16),
		AutomaticSize = Enum.AutomaticSize.X,
		LayoutOrder = 1,
	})
	label(titlebar, {
		Text = "...",
		Font = FONT,
		TextSize = 11,
		TextColor3 = C.faint,
		Size = UDim2.new(0, 200, 0, 14),
		Position = UDim2.fromOffset(60, 30),
	})

	local versionChip = new("TextLabel", {
		Size = UDim2.new(0, 0, 0, 18),
		AutomaticSize = Enum.AutomaticSize.X,
		BackgroundColor3 = C.panel2,
		Text = "BETA",
		TextColor3 = C.muted,
		Font = FONT_MED,
		TextSize = 10,
		BorderSizePixel = 0,
		TextXAlignment = Enum.TextXAlignment.Center,
		LayoutOrder = 2,
	}, titleRow)
	corner(versionChip, 5)
	stroke(versionChip, C.white, 0.9)
	pad(versionChip, 0, 7, 0, 7)

	local controls = new("Frame", {
		Size = UDim2.fromOffset(108, 30),
		Position = UDim2.new(1, -22, 0, 15),
		AnchorPoint = Vector2.new(1, 0),
		BackgroundTransparency = 1,
	}, titlebar)
	new("UIListLayout", {
		FillDirection = Enum.FillDirection.Horizontal,
		HorizontalAlignment = Enum.HorizontalAlignment.Right,
		Padding = UDim.new(0, 4),
		SortOrder = Enum.SortOrder.LayoutOrder,
	}, controls)

	local windowButtons = {}

	local function windowButton(iconKind, order, danger)
		local btn = new("TextButton", {
			Size = UDim2.fromOffset(30, 30),
			BackgroundTransparency = 1,
			Text = "",
			AutoButtonColor = false,
			LayoutOrder = order,
		}, controls)
		corner(btn, 8)

		local ic = makeIcon(iconKind, btn, 14, C.muted)
		ic.Position = UDim2.fromScale(0.5, 0.5)
		ic.AnchorPoint = Vector2.new(0.5, 0.5)

		btn.MouseEnter:Connect(function()
			btn.BackgroundTransparency = 0
			btn.BackgroundColor3 = danger and C.red or C.panel3
			tintIcon(ic, danger and C.bg or C.text)
		end)
		btn.MouseLeave:Connect(function()
			btn.BackgroundTransparency = 1
			tintIcon(ic, C.muted)
		end)

		windowButtons[order] = btn
		return btn
	end

	windowButton("lucide:minus", 1, false)

	windowButton("lucide:square", 2, false)
	windowButton("lucide:x", 3, true)

	bindDrag(dragHandle)

	local EDGE = 6
	do
		local strips = {
			{ "DragLeft", UDim2.new(0, EDGE, 1, 0), UDim2.fromOffset(0, 0) },
			{ "DragRight", UDim2.new(0, EDGE, 1, 0), UDim2.new(1, -EDGE, 0, 0) },
			{ "DragBottom", UDim2.new(1, 0, 0, EDGE), UDim2.new(0, 0, 1, -EDGE) },
		}
		for _, s in ipairs(strips) do
			local strip = new("TextButton", {
				Name = s[1],
				Size = s[2],
				Position = s[3],
				BackgroundTransparency = 1,
				Text = "",
				AutoButtonColor = false,
				Selectable = false,
			}, app)
			bindDrag(strip)
		end
	end

	local body = new("Frame", {
		Name = "Body",
		Size = UDim2.new(1, 0, 1, -TITLEBAR_H),
		Position = UDim2.fromOffset(0, TITLEBAR_H),
		BackgroundTransparency = 1,
	}, app)

	local rail = new("Frame", {
		Size = UDim2.new(0, RAIL_W, 1, 0),
		BackgroundColor3 = C.rail,
		BorderSizePixel = 0,
	}, body)
	new("Frame", {
		Size = UDim2.new(0, 1, 1, 0),
		Position = UDim2.new(1, -1, 0, 0),
		BackgroundColor3 = C.white,
		BackgroundTransparency = 0.94,
		BorderSizePixel = 0,
	}, rail)

	local navButtons = {}
	local navOrder = 0

	local function navGroup(text, topPad)
		local g = new("Frame", {
			Size = UDim2.new(1, -24, 0, 0),
			Position = UDim2.fromOffset(12, topPad),
			BackgroundTransparency = 1,
			AutomaticSize = Enum.AutomaticSize.Y,
		}, rail)

		local groupLabel = label(g, {
			Text = text,
			Font = FONT_BOLD,
			TextSize = 10,
			TextColor3 = C.faint,
			Size = UDim2.new(1, -14, 0, 14),
			Position = UDim2.fromOffset(10, 0),
		})
		RESP.groupLabels[#RESP.groupLabels + 1] = groupLabel

		local list = new("Frame", {
			Size = UDim2.new(1, 0, 0, 0),
			Position = UDim2.fromOffset(0, 22),
			BackgroundTransparency = 1,
			AutomaticSize = Enum.AutomaticSize.Y,
		}, g)
		new("UIListLayout", {
			Padding = UDim.new(0, 2),
			SortOrder = Enum.SortOrder.LayoutOrder,
		}, list)

		g.AutomaticSize = Enum.AutomaticSize.Y
		g.Size = UDim2.new(1, -24, 0, 0)

		RESP.groups[#RESP.groups + 1] = { container = g, list = list, label = groupLabel }

		return list, g
	end

	local function navItem(parent, iconKind, text, index)
		navOrder = navOrder + 1

		local item = new("Frame", {
			Size = UDim2.new(1, 0, 0, 40),
			BackgroundColor3 = C.panel3,
			BackgroundTransparency = 1,
			BorderSizePixel = 0,
			LayoutOrder = navOrder,
		}, parent)
		corner(item, 10)

		local badge = new("Frame", {
			Size = UDim2.fromOffset(24, 24),
			Position = UDim2.new(0, 8, 0.5, 0),
			AnchorPoint = Vector2.new(0, 0.5),
			BackgroundColor3 = C.panel4,
			BackgroundTransparency = 1,
			BorderSizePixel = 0,
		}, item)
		corner(badge, 7)

		local icon = makeIcon(iconKind, badge, 16, C.muted)
		icon.Position = UDim2.fromScale(0.5, 0.5)
		icon.AnchorPoint = Vector2.new(0.5, 0.5)

		local textLabel = label(item, {
			Text = text,
			Font = FONT_MED,
			TextSize = 13,
			TextColor3 = C.muted,
			Size = UDim2.new(1, -54, 1, 0),
			Position = UDim2.fromOffset(42, 0),
		})

		local hit = new("TextButton", {
			Size = UDim2.fromScale(1, 1),
			BackgroundTransparency = 1,
			Text = "",
		}, item)

		local entry = {
			index = index,
			frame = item,
			badge = badge,
			icon = icon,
			text = textLabel,
			hit = hit,
			active = false,
		}

		local railMark = new("Frame", {
			Size = UDim2.new(0, 2, 0, 18),
			Position = UDim2.new(0, 0, 0.5, 0),
			AnchorPoint = Vector2.new(0, 0.5),
			BackgroundColor3 = ACCENT,
			BorderSizePixel = 0,
			Visible = false,
		}, item)
		corner(railMark, 99)
		entry.rail = railMark

		local function render()
			if entry.active then
				entry.text.TextColor3 = C.text
				entry.text.Font = FONT_BOLD
				item.BackgroundTransparency = 0
				badge.BackgroundTransparency = 0
				railMark.BackgroundColor3 = ACCENT
				railMark.Visible = true
				tintIcon(entry.icon, ACCENT)
			else
				entry.text.TextColor3 = C.muted
				entry.text.Font = FONT_MED
				item.BackgroundTransparency = 1
				badge.BackgroundTransparency = 1
				railMark.Visible = false
				tintIcon(entry.icon, C.muted)
			end
		end

		entry.setActive = function(isActive)
			entry.active = isActive
			render()
		end

		hit.MouseEnter:Connect(function()
			if entry.active then return end
			play(item, MOTION.fast, { BackgroundTransparency = 0.95 })
			entry.text.TextColor3 = C.text
			tintIcon(entry.icon, C.text)
		end)
		hit.MouseLeave:Connect(function()
			if entry.active then return end
			play(item, MOTION.fast, { BackgroundTransparency = 1 })
			entry.text.TextColor3 = C.muted
			tintIcon(entry.icon, C.muted)
		end)

		onAccent(function()
			if entry.active then render() end
		end)

		navButtons[index] = entry
		RESP.navItems[#RESP.navItems + 1] = entry
		return entry
	end

	local nav, navFrame = navGroup("CONTROL", 18)
	RESP.mainGroup = navFrame

	navItem(nav, "lucide:zap", "RideGO", 1)
	navItem(nav, "lucide:activity", "Status", 2)

	local userCard = new("Frame", {
		Size = UDim2.new(1, -20, 0, 56),
		Position = UDim2.new(0, 10, 1, -68),
		BackgroundColor3 = C.panel2,
		BackgroundTransparency = 0,
		BorderSizePixel = 0,
	}, rail)
	corner(userCard, 12)
	stroke(userCard, C.white, 0.93)

	local userAvatar = new("Frame", {
		Size = UDim2.fromOffset(32, 32),
		Position = UDim2.new(0, 12, 0.5, 0),
		AnchorPoint = Vector2.new(0, 0.5),
		BackgroundColor3 = Color3.fromHSV(((plr.UserId % 9973) * 137) % 360 / 360, 0.42, 0.66),
		BorderSizePixel = 0,
	}, userCard)
	corner(userAvatar, 99)
	label(userAvatar, {
		Text = string.upper(string.sub(plr.DisplayName, 1, 1)),
		Font = FONT_BOLD,
		TextSize = 13,
		TextColor3 = C.bg,
		Size = UDim2.fromScale(1, 1),
		TextXAlignment = Enum.TextXAlignment.Center,
	})

	local userName = label(userCard, {
		Text = plr.DisplayName,
		Font = FONT_MED,
		TextSize = 12,
		Size = UDim2.new(1, -110, 0, 15),
		Position = UDim2.fromOffset(54, 12),
		TextTruncate = Enum.TextTruncate.AtEnd,
	})
	local statusDot = new("Frame", {
		Size = UDim2.fromOffset(5, 5),
		Position = UDim2.fromOffset(54, 33),
		BackgroundColor3 = C.green,
		BorderSizePixel = 0,
	}, userCard)
	corner(statusDot, 99)
	local userStatus = label(userCard, {
		Text = "Online",
		TextSize = 10,
		TextColor3 = C.faint,
		Size = UDim2.new(1, -110, 0, 12),
		Position = UDim2.fromOffset(64, 29),
	})

	local userChevron = makeIcon("lucide:chevron-right", userCard, 14, C.faint)
	userChevron.Position = UDim2.new(1, -14, 0.5, 0)
	userChevron.AnchorPoint = Vector2.new(1, 0.5)
	RESP.userChevron = userChevron

	RESP.rail = rail
	RESP.userCard = userCard
	RESP.userAvatar = userAvatar
	RESP.userName = userName
	RESP.userStatus = userStatus
	RESP.statusDot = statusDot

	local content = new("Frame", {
		Size = UDim2.new(1, -RAIL_W, 1, 0),
		Position = UDim2.fromOffset(RAIL_W, 0),
		BackgroundTransparency = 1,
	}, body)
	local contentPad = pad(content, 28)
	RESP.content = content
	RESP.contentPad = contentPad

	local pageHeader = new("Frame", {
		Size = UDim2.new(1, 0, 0, 52),
		BackgroundTransparency = 1,
	}, content)

	local pageTitle = label(pageHeader, {
		Text = "Overview",
		Font = FONT_BOLD,
		TextSize = 26,
		Size = UDim2.new(1, -360, 0, 32),
		TextTruncate = Enum.TextTruncate.AtEnd,
	})
	local pageSubtitle = label(pageHeader, {
		Text = "",
		TextSize = 12,
		TextColor3 = C.muted,
		Size = UDim2.new(1, -360, 0, 15),
		Position = UDim2.fromOffset(0, 34),
		TextTruncate = Enum.TextTruncate.AtEnd,
	})

	local primary = new("TextButton", {
		Size = UDim2.fromOffset(96, 36),
		Position = UDim2.new(1, 0, 0, 8),
		AnchorPoint = Vector2.new(1, 0),
		BackgroundColor3 = C.text,
		Text = "Start",
		TextColor3 = C.bg,
		Font = FONT_BOLD,
		TextSize = 12,
		BorderSizePixel = 0,
		AutoButtonColor = false,
	}, pageHeader)
	corner(primary, 10)
	primary.MouseEnter:Connect(function() play(primary, MOTION.fast, { BackgroundColor3 = C.white }) end)
	primary.MouseLeave:Connect(function() play(primary, MOTION.fast, { BackgroundColor3 = C.text }) end)

	RESP.pageHeader = pageHeader
	RESP.title = pageTitle
	RESP.subtitle = pageSubtitle
	RESP.primary = primary

	local pageHost = new("ScrollingFrame", {
		Size = UDim2.new(1, 0, 1, -76),
		Position = UDim2.fromOffset(0, 76),
		BackgroundTransparency = 1,
		BorderSizePixel = 0,
		ScrollBarThickness = 2,
		ScrollBarImageColor3 = C.panel4,
		ScrollingDirection = Enum.ScrollingDirection.Y,
		CanvasSize = UDim2.new(0, 0, 0, 0),
		AutomaticCanvasSize = Enum.AutomaticSize.None,
		ClipsDescendants = true,
	}, content)
	RESP.pageHost = pageHost

	local showToast

	local function card(parent, order)
		local c = new("Frame", {
			Size = UDim2.new(0, 0, 0, 0),
			BackgroundColor3 = C.panel,
			BorderSizePixel = 0,
			LayoutOrder = order or 1,
		}, parent)
		corner(c, 14)
		return c
	end

	local function settingRow(parent, order, title, desc, control)
		local row = new("Frame", {
			Size = UDim2.new(1, 0, 0, 56),
			BackgroundTransparency = 1,
			LayoutOrder = order,
		}, parent)

		label(row, {
			Text = title,
			Font = FONT_MED,
			TextSize = 12,
			Size = UDim2.new(1, -110, 0, 16),
			Position = UDim2.fromOffset(0, 11),
		})
		if desc then
			label(row, {
				Text = desc,
				TextSize = 11,
				TextColor3 = C.faint,
				Size = UDim2.new(1, -110, 0, 14),
				Position = UDim2.fromOffset(0, 28),
				TextTruncate = Enum.TextTruncate.AtEnd,
			})
		end

		new("Frame", {
			Size = UDim2.new(1, 0, 0, 1),
			Position = UDim2.new(0, 0, 1, -1),
			BackgroundColor3 = C.white,
			BackgroundTransparency = 0.945,
			BorderSizePixel = 0,
		}, row)

		return row
	end

	local function makeToggle(parent, defaultOn, onChange)
		local on = defaultOn

		local track = new("Frame", {
			Size = UDim2.fromOffset(38, 22),
			Position = UDim2.new(1, 0, 0.5, 0),
			AnchorPoint = Vector2.new(1, 0.5),
			BackgroundColor3 = on and ACCENT or C.panel3,
			BorderSizePixel = 0,
		}, parent)
		corner(track, 99)
		stroke(track, C.white, on and 1 or 0.92)

		local knob = new("Frame", {
			Size = UDim2.fromOffset(16, 16),
			Position = on and UDim2.fromOffset(19, 3) or UDim2.fromOffset(3, 3),
			BackgroundColor3 = on and C.bg or C.muted,
			BorderSizePixel = 0,
		}, track)
		corner(knob, 99)

		local hit = new("TextButton", {
			Size = UDim2.fromScale(1, 1),
			BackgroundTransparency = 1,
			Text = "",
		}, track)

		local function apply()
			play(track, MOTION.fast, { BackgroundColor3 = on and ACCENT or C.panel3 })
			play(knob, MOTION.fast, {
				Position = on and UDim2.fromOffset(19, 3) or UDim2.fromOffset(3, 3),
				BackgroundColor3 = on and C.bg or C.muted,
			})
		end

		onAccent(function(c)
			if on then track.BackgroundColor3 = c end
		end)

		hit.MouseButton1Click:Connect(function()
			on = not on
			apply()
			if onChange then onChange(on) end
		end)

		return {
			set = function(v)
				on = v
				apply()
			end,
			get = function() return on end,
		}
	end

	local function toggleRow(parent, order, title, desc, defaultOn, onChange)
		local row = settingRow(parent, order, title, desc)
		local t = makeToggle(row, defaultOn, onChange)
		t.hit = row
		return t
	end

	local function sectionCard(parent, cfg)
		local c = card(parent, cfg.order or 1)
		c.Size = cfg.size
		if cfg.position then c.Position = cfg.position end
		pad(c, 18, 18, 8, 18)

		label(c, {
			Text = cfg.title,
			Font = FONT_BOLD,
			TextSize = 13,
			Size = UDim2.new(1, 0, 0, 18),
		})
		if cfg.desc then
			label(c, {
				Text = cfg.desc,
				TextSize = 11,
				TextColor3 = C.faint,
				Size = UDim2.new(1, 0, 0, 14),
				Position = UDim2.fromOffset(0, 18),
			})
		end

		local list = new("Frame", {
			Size = UDim2.new(1, 0, 1, -(cfg.desc and 48 or 30)),
			Position = UDim2.fromOffset(0, cfg.desc and 48 or 30),
			BackgroundTransparency = 1,
		}, c)
		new("UIListLayout", {
			Padding = UDim.new(0, 0),
			SortOrder = Enum.SortOrder.LayoutOrder,
		}, list)

		return c, list
	end

	local pages = {}
	local pageMeta = {}

	local function newPage()
		local p = new("Frame", {
			Size = UDim2.fromScale(1, 1),
			BackgroundTransparency = 1,
			Visible = false,
		}, pageHost)
		pages[#pages + 1] = p
		return p
	end

	local function buildRidegoPage()
		local page = newPage()
		pageMeta[#pageMeta + 1] = {
			title = "RideGO",
			subtitle = "Kontrol auto taxi",
		}

		local col = new("Frame", {
			Size = UDim2.new(1, 0, 0, 0),
			AutomaticSize = Enum.AutomaticSize.Y,
			BackgroundTransparency = 1,
		}, page)
		new("UIListLayout", {
			Padding = UDim.new(0, 16),
			SortOrder = Enum.SortOrder.LayoutOrder,
		}, col)

		local function numberField(parent, order, title, desc, key, step, decimals)
			local row = settingRow(parent, order, title, desc)
			local box = new("Frame", {
				Size = UDim2.fromOffset(158, 28),
				Position = UDim2.new(1, 0, 0.5, 0),
				AnchorPoint = Vector2.new(1, 0.5),
				BackgroundColor3 = C.panel2,
				BorderSizePixel = 0,
			}, row)
			corner(box, 8)
			local boxStroke = stroke(box, C.white, 0.92)

			local input = new("TextBox", {
				Size = UDim2.fromOffset(92, 26),
				Position = UDim2.fromOffset(32, 1),
				BackgroundTransparency = 1,
				Text = "",
				Font = FONT_MED,
				TextSize = 12,
				TextColor3 = C.text,
				TextXAlignment = Enum.TextXAlignment.Center,
				ClearTextOnFocus = false,
			}, box)

			local value = tonumber(bridge.getAtur(key)) or 0
			local function render()
				if decimals and decimals > 0 then
					input.Text = string.format("%." .. decimals .. "f", value)
				else
					input.Text = string.format("%.0f", value)
				end
			end
			local function commit(v, quiet)
				local n = tonumber((tostring(v):gsub(",", ".")))
				if not n then
					render()
					return
				end
				n = math.floor(n * 1000 + 0.5) / 1000
				local applied = tonumber(bridge.setAtur(key, n)) or n
				value = applied
				render()
				if not quiet and math.abs(applied - n) > 0.0001 and showToast then
					showToast({
						title = "Value limited",
						content = string.format("%s: %s", title, input.Text),
						kind = "info",
					})
				end
			end

			local function arrow(sym, x, dir)
				local b = new("TextButton", {
					Size = UDim2.fromOffset(26, 26),
					Position = UDim2.fromOffset(x, 1),
					BackgroundTransparency = 1,
					Text = sym,
					Font = FONT_BOLD,
					TextSize = 14,
					TextColor3 = C.muted,
					BorderSizePixel = 0,
					AutoButtonColor = false,
				}, box)
				b.MouseEnter:Connect(function() b.TextColor3 = C.text end)
				b.MouseLeave:Connect(function() b.TextColor3 = C.muted end)
				b.MouseButton1Click:Connect(function() commit(value + dir * step, true) end)
				return b
			end

			arrow("-", 1, -1)
			arrow("+", 131, 1)

			input.Focused:Connect(function()
				boxStroke.Transparency = 0
				boxStroke.Color = ACCENT
				task.defer(function()
					if input:IsFocused() then
						input.SelectionStart = 1
						input.CursorPosition = #input.Text + 1
					end
				end)
			end)
			input.FocusLost:Connect(function()
				boxStroke.Transparency = 0.92
				boxStroke.Color = C.white
				commit(input.Text)
			end)

			render()
		end

		local autoCard = card(col, 1)
		autoCard.Size = UDim2.new(1, 0, 0, 72)
		pad(autoCard, 8, 18, 8, 18)
		local autoList = new("Frame", {
			Size = UDim2.new(1, 0, 0, 56),
			BackgroundTransparency = 1,
		}, autoCard)
		new("UIListLayout", {
			Padding = UDim.new(0, 0),
			SortOrder = Enum.SortOrder.LayoutOrder,
		}, autoList)
		local switch = toggleRow(
			autoList,
			1,
			"Auto RideGO",
			"Start or stop the auto loop",
			bridge.isRunning(),
			function(on)
				bridge.setRunning(on)
				primary.Text = on and "Stop" or "Start"
				showToast({
					title = on and "Auto RideGO started" or "Auto RideGO paused",
					content = on and "The loop is driving now" or "Press F8 or this switch to stop",
					kind = on and "success" or "info",
				})
			end
		)

		local tuneCard, tuneList = sectionCard(col, {
			order = 2,
			size = UDim2.new(1, 0, 0, 336),
			title = "Distance & delay",
			desc = "Live values used by the next teleport",
		})
		numberField(
			tuneList,
			1,
			"Pickup distance",
			"Studs per hop while picking up (10-500)",
			"JEMPUT_JARAK",
			10,
			0
		)
		numberField(
			tuneList,
			2,
			"Drop distance",
			"Studs per hop while dropping off (10-250)",
			"ANTAR_JARAK",
			10,
			0
		)
		numberField(tuneList, 3, "Pickup delay", "Seconds between pickup hops (0.05-5)", "JEMPUT_JEDA", 0.05, 2)
		numberField(tuneList, 4, "Drop delay", "Seconds between drop hops (0.05-5)", "ANTAR_JEDA", 0.05, 2)
		numberField(
			tuneList,
			5,
			"Stall limit",
			"Seconds without progress before abort (15-300)",
			"BATAS_DIAM",
			5,
			0
		)

		primary.Text = bridge.isRunning() and "Stop" or "Start"

		return {
			switch = switch,
			toggle = function()
				local nextOn = not bridge.isRunning()
				bridge.setRunning(nextOn)
				switch.set(nextOn)
				primary.Text = nextOn and "Stop" or "Start"
			end,
		}
	end

	local function buildStatusPage(ui)
		local page = newPage()
		pageMeta[#pageMeta + 1] = {
			title = "Status",
			subtitle = "Statistik auto RideGO",
		}

		local col = new("Frame", {
			Size = UDim2.new(1, 0, 0, 0),
			AutomaticSize = Enum.AutomaticSize.Y,
			BackgroundTransparency = 1,
		}, page)
		new("UIListLayout", {
			Padding = UDim.new(0, 16),
			SortOrder = Enum.SortOrder.LayoutOrder,
		}, col)

		local panel = card(col, 1)
		panel.Size = UDim2.new(1, 0, 0, 468)
		pad(panel, 18, 18, 18, 18)

		local head = new("Frame", {
			Size = UDim2.new(1, 0, 0, 36),
			BackgroundTransparency = 1,
		}, panel)
		label(head, {
			Text = "Auto RideGO statistics",
			Font = FONT_BOLD,
			TextSize = 14,
			Size = UDim2.new(1, -110, 1, 0),
		})

		local pill = new("Frame", {
			Size = UDim2.fromOffset(92, 22),
			Position = UDim2.new(1, 0, 0, 2),
			AnchorPoint = Vector2.new(1, 0),
			BackgroundColor3 = C.panel3,
			BorderSizePixel = 0,
		}, head)
		corner(pill, 11)
		local pillDot = new("Frame", {
			Size = UDim2.fromOffset(6, 6),
			Position = UDim2.fromOffset(11, 8),
			BackgroundColor3 = C.muted,
			BorderSizePixel = 0,
		}, pill)
		corner(pillDot, 99)
		local pillValue = label(pill, {
			Text = "Paused",
			Font = FONT_MED,
			TextSize = 11,
			TextColor3 = C.muted,
			Size = UDim2.new(1, -26, 1, 0),
			Position = UDim2.fromOffset(23, 0),
			TextYAlignment = Enum.TextYAlignment.Center,
		})

		local rows = new("Frame", {
			Size = UDim2.new(1, 0, 1, -48),
			Position = UDim2.fromOffset(0, 48),
			BackgroundTransparency = 1,
		}, panel)
		new("UIListLayout", {
			Padding = UDim.new(0, 8),
			SortOrder = Enum.SortOrder.LayoutOrder,
		}, rows)

		local function statRow(order, title, iconKind, tint)
			local r = new("Frame", {
				Size = UDim2.new(1, 0, 0, 48),
				BackgroundColor3 = C.panel2,
				BorderSizePixel = 0,
				LayoutOrder = order,
			}, rows)
			corner(r, 10)
			stroke(r, C.white, 0.95)

			local box = new("Frame", {
				Size = UDim2.fromOffset(30, 30),
				Position = UDim2.fromOffset(9, 9),
				BackgroundColor3 = C.panel3,
				BorderSizePixel = 0,
			}, r)
			corner(box, 8)
			local icon = makeIcon(iconKind, box, 15, tint or C.accent)
			icon.AnchorPoint = Vector2.new(0.5, 0.5)
			icon.Position = UDim2.fromScale(0.5, 0.5)

			label(r, {
				Text = title,
				Font = FONT_MED,
				TextSize = 12,
				Size = UDim2.new(1, -58, 0, 14),
				Position = UDim2.fromOffset(50, 8),
			})

			local value = label(r, {
				Text = "-",
				TextSize = 11,
				TextColor3 = C.muted,
				Size = UDim2.new(1, -58, 0, 16),
				Position = UDim2.fromOffset(50, 23),
				TextTruncate = Enum.TextTruncate.AtEnd,
			})
			return value, icon
		end

		local stateVal, stateIcon = statRow(1, "Status", "lucide:activity", C.muted)
		local phaseVal = statRow(2, "Phase", "lucide:sliders-horizontal")
		local noteVal = statRow(3, "Note", "lucide:file")
		local ordersVal = statRow(4, "Orders", "lucide:package")
		local hopsVal = statRow(5, "Hops", "lucide:arrow-right")
		local pingVal = statRow(6, "Ping", "lucide:clock")
		local tripVal = statRow(7, "Trip", "lucide:chart-column")

		task.spawn(function()
			while page.Parent do
				local st = bridge.status()
				local running = st.running == true
				stateVal.Text = running and "Running" or "Paused"
				stateVal.TextColor3 = running and C.green or C.muted
				tintIcon(stateIcon, running and C.green or C.muted)
				pillValue.Text = running and "Running" or "Paused"
				pillValue.TextColor3 = running and C.green or C.muted
				pillDot.BackgroundColor3 = running and C.green or C.muted
				phaseVal.Text = st.phase or "-"
				noteVal.Text = (st.note and st.note ~= "") and st.note or "-"
				ordersVal.Text = tostring(st.orders or 0)
				hopsVal.Text = tostring(st.hops or 0)
				pingVal.Text = st.ping and string.format("%.0f ms", st.ping) or "-"
				tripVal.Text = string.format("%.0f stud | Rp %.0f", st.tripStuds or 0, st.tripMoney or 0)
				if ui.switch.get() ~= running then ui.switch.set(running) end
				local btn = running and "Stop" or "Start"
				if primary.Text ~= btn then primary.Text = btn end
				task.wait(0.5)
			end
		end)
	end

	local ridegoUI = buildRidegoPage()
	buildStatusPage(ridegoUI)

	local toastHost = new("Frame", {
		Name = "Toasts",
		Size = UDim2.fromOffset(320, 400),
		Position = UDim2.new(1, -24, 0, 24),
		AnchorPoint = Vector2.new(1, 0),
		BackgroundTransparency = 1,
	}, screenGui)
	new("UIListLayout", {
		Padding = UDim.new(0, 10),
		HorizontalAlignment = Enum.HorizontalAlignment.Right,
		SortOrder = Enum.SortOrder.LayoutOrder,
	}, toastHost)

	local toastOrder = 0

	showToast = function(cfg)
		toastOrder = toastOrder + 1
		local kindColors = {
			info = C.accent,
			success = C.green,
			error = C.red,
		}
		local accentColor = kindColors[cfg.kind or "info"] or C.accent
		local duration = cfg.duration or 3

		local t = new("CanvasGroup", {
			Size = UDim2.fromOffset(300, 66),
			BackgroundColor3 = C.panel2,
			BorderSizePixel = 0,
			GroupTransparency = 1,
			LayoutOrder = toastOrder,
		}, toastHost)
		corner(t, 12)
		stroke(t, C.white, 0.9, 1)

		local pop = new("UIScale", { Scale = 0.98 }, t)

		local dot = new("Frame", {
			Size = UDim2.fromOffset(6, 6),
			Position = UDim2.fromOffset(16, 20),
			BackgroundColor3 = accentColor,
			BorderSizePixel = 0,
		}, t)
		corner(dot, 99)

		label(t, {
			Text = cfg.title or "Notification",
			Font = FONT_BOLD,
			TextSize = 12,
			Size = UDim2.new(1, -60, 0, 16),
			Position = UDim2.fromOffset(30, 14),
		})
		label(t, {
			Text = cfg.content or "",
			TextSize = 11,
			TextColor3 = C.muted,
			Size = UDim2.new(1, -44, 0, 16),
			Position = UDim2.fromOffset(30, 32),
			TextTruncate = Enum.TextTruncate.AtEnd,
		})

		local bar = new("Frame", {
			Size = UDim2.new(1, -24, 0, 2),
			Position = UDim2.fromOffset(12, 56),
			BackgroundColor3 = accentColor,
			BackgroundTransparency = 0.4,
			BorderSizePixel = 0,
		}, t)
		corner(bar, 99)

		play(t, MOTION.base, { GroupTransparency = 0 })
		play(pop, MOTION.base, { Scale = 1 })

		play(bar, duration, { Size = UDim2.new(0, 0, 1, 1) }, Enum.EasingStyle.Linear).Completed:Connect(
			function()
				local fade = play(t, MOTION.fast, { GroupTransparency = 1 })
				fade.Completed:Connect(function() t:Destroy() end)
			end
		)
	end

	local activePage = 0

	local function setPage(index)
		if index == activePage then return end
		activePage = index

		for i, p in ipairs(pages) do
			p.Visible = (i == index)
		end

		for _, entry in ipairs(navButtons) do
			entry.setActive(entry.index == index)
		end

		pageTitle.Text = pageMeta[index].title
		pageSubtitle.Text = pageMeta[index].subtitle

		local p = pages[index]
		p.Position = UDim2.fromOffset(0, 10)
		play(p, MOTION.base, { Position = UDim2.fromOffset(0, 0) })

		if RESP.applyLayout then RESP.applyLayout() end
	end

	for _, entry in ipairs(navButtons) do
		entry.hit.MouseButton1Click:Connect(function() setPage(entry.index) end)
	end

	setPage(1)

	local RESIZE_GRAB = 18
	local resizeCorner = new("TextButton", {
		Name = "ResizeCorner",
		Size = UDim2.fromOffset(RESIZE_GRAB, RESIZE_GRAB),
		Position = UDim2.new(1, -RESIZE_GRAB, 1, -RESIZE_GRAB),
		BackgroundTransparency = 1,
		Text = "",
		AutoButtonColor = false,
		Selectable = false,
		ZIndex = 5,
	}, app)
	bindResize(resizeCorner)

	local resizeGrip = new("Frame", {
		Size = UDim2.fromOffset(10, 10),
		Position = UDim2.new(1, -12, 1, -12),
		AnchorPoint = Vector2.new(1, 1),
		BackgroundTransparency = 1,
		ZIndex = 5,
	}, resizeCorner)

	local gripDots = {}
	for i = 1, 3 do
		gripDots[i] = new("Frame", {
			Size = UDim2.fromOffset(2, 2),
			Position = UDim2.fromOffset((i - 1) * 3, 6 - (i - 1) * 3),
			BackgroundColor3 = C.faint,
			BorderSizePixel = 0,
			ZIndex = 6,
		}, resizeGrip)
		corner(gripDots[i], 99)
	end

	resizeCorner.MouseEnter:Connect(function()
		for _, d in ipairs(gripDots) do
			d.BackgroundColor3 = C.text
		end
	end)
	resizeCorner.MouseLeave:Connect(function()
		for _, d in ipairs(gripDots) do
			d.BackgroundColor3 = C.faint
		end
	end)

	local BP_WIDE = 1080
	local BP_MID = 840

	local function requiredPageHeight() return 500 end

	local resizeCanvas

	local function applyLayout()
		if not RESP.pageHost or not RESP.rail then return end
		local w = winSize.X
		local mode = w >= BP_WIDE and "wide" or (w >= BP_MID and "mid" or "narrow")
		local compact = mode ~= "wide"
		local narrow = mode == "narrow"
		RESP.mode = mode

		local railW = compact and RAIL_W_COMPACT or RAIL_W
		RESP.rail.Size = UDim2.new(0, railW, 1, 0)
		RESP.content.Size = UDim2.new(1, -railW, 1, 0)
		RESP.content.Position = UDim2.fromOffset(railW, 0)

		local cp = compact and 16 or 28
		RESP.contentPad.PaddingTop = UDim.new(0, cp)
		RESP.contentPad.PaddingBottom = UDim.new(0, cp)
		RESP.contentPad.PaddingLeft = UDim.new(0, cp)
		RESP.contentPad.PaddingRight = UDim.new(0, cp)

		local rowH = compact and 44 or 40
		for _, item in ipairs(RESP.navItems) do
			item.frame.Size = UDim2.new(1, 0, 0, rowH)
			item.text.Visible = not compact
			item.badge.Position = compact and UDim2.new(0.5, -12, 0.5, 0) or UDim2.new(0, 8, 0.5, 0)
		end
		for _, g in ipairs(RESP.groups) do
			g.label.Visible = not compact
			g.list.Position = UDim2.fromOffset(0, compact and 0 or 22)
			g.container.Size = UDim2.new(1, compact and -12 or -24, 0, 0)
		end
		RESP.mainGroup.Position = UDim2.fromOffset(compact and 6 or 12, 18)

		RESP.userCard.Size = UDim2.new(1, compact and -12 or -20, 0, 56)
		RESP.userCard.Position = UDim2.new(0, compact and 6 or 10, 1, -68)
		RESP.userAvatar.Position = compact and UDim2.new(0.5, -16, 0.5, 0) or UDim2.new(0, 12, 0.5, 0)
		RESP.userName.Visible = not compact
		RESP.userStatus.Visible = not compact
		RESP.statusDot.Visible = not compact
		if RESP.userChevron then RESP.userChevron.Visible = not compact end

		local titleInset = narrow and 104 or 128
		RESP.title.Size = UDim2.new(1, -titleInset, 0, 32)
		RESP.subtitle.Size = UDim2.new(1, -titleInset, 0, 15)
		RESP.primary.Size = UDim2.fromOffset(narrow and 84 or 96, 36)

		RESP.resizeCanvas()
	end

	function resizeCanvas()
		if not RESP.pageHost then return end
		local mode = RESP.mode or "wide"
		local cp = mode ~= "wide" and 16 or 28
		local pageH = winSize.Y - TITLEBAR_H - cp * 2 - 76
		local canvasH = math.max(requiredPageHeight(), pageH)
		for _, p in ipairs(pages) do
			p.Size = UDim2.new(1, 0, 0, canvasH)
		end
		RESP.pageHost.CanvasSize = UDim2.new(0, 0, 0, canvasH)
		RESP.pageHost.ScrollingEnabled = canvasH > pageH + 1
	end

	RESP.resizeCanvas = resizeCanvas

	RESP.applyLayout = applyLayout
	RESP.ready = true
	applyLayout()

	local open = false
	local offsetBeforeFull = nil
	local miniBadge

	local function setWindowState(nextState)
		if nextState == windowState then return end
		local previous = windowState
		windowState = nextState

		if previous == "min" then
			body.Visible = true
			titlebar.Visible = true
			resizeCorner.Visible = true
			if miniBadge then miniBadge.Visible = false end
			app.BackgroundColor3 = C.bg
			if open then setShadowsVisible(true) end
		end

		if nextState == "min" then
			play(app, MOTION.base, {
				Size = UDim2.fromOffset(56, 56),
			}).Completed:Connect(function()
				if windowState == "min" then
					body.Visible = false
					titlebar.Visible = false
					resizeCorner.Visible = false
					if miniBadge then miniBadge.Visible = true end
					setShadowsVisible(false)
				end
			end)
			return
		end

		local target
		if nextState == "full" then
			local cam = workspace.CurrentCamera
			local vp = cam and cam.ViewportSize or Vector2.new(1280, 720)
			target = UDim2.fromOffset(math.max(MIN_W, vp.X - 40), math.max(MIN_H, vp.Y - 40))

			if offsetBeforeFull == nil then
				offsetBeforeFull = windowOffset
				windowOffset = Vector2.new(0, 0)
				applyWindowOffset()
			end
		else
			target = UDim2.fromOffset(winSize.X, winSize.Y)
			if offsetBeforeFull then
				windowOffset = offsetBeforeFull
				offsetBeforeFull = nil
				applyWindowOffset()
			end
		end

		play(app, MOTION.slow, { Size = target }).Completed:Connect(clampWindowOffset)

		if nextState == "full" then
			showToast({ title = "Fullscreen", content = "Press the control again to exit", kind = "info" })
		end
	end

	miniBadge = new("TextButton", {
		Name = "MiniIcon",
		Size = UDim2.fromScale(1, 1),
		BackgroundTransparency = 1,
		Text = "",
		AutoButtonColor = false,
		Visible = false,
		ZIndex = 10,
	}, app)
	local miniIcon = makeIcon("lucide:zap", miniBadge, 24, C.text)
	miniIcon.AnchorPoint = Vector2.new(0.5, 0.5)
	miniIcon.Position = UDim2.fromScale(0.5, 0.5)
	miniBadge.MouseEnter:Connect(function() play(app, MOTION.fast, { BackgroundColor3 = C.panel2 }) end)
	miniBadge.MouseLeave:Connect(function() play(app, MOTION.fast, { BackgroundColor3 = C.bg }) end)

	local miniDragging = false
	local miniDragMoved = 0
	local miniDragConn = nil

	local function finishMiniDrag()
		if not miniDragging then return end
		miniDragging = false
		if miniDragConn then
			miniDragConn:Disconnect()
			miniDragConn = nil
		end
		if miniDragMoved < 6 then setWindowState("normal") end
	end

	miniBadge.InputBegan:Connect(function(input)
		if windowState ~= "min" or miniDragging then return end
		local t = input.UserInputType
		if t ~= Enum.UserInputType.MouseButton1 and t ~= Enum.UserInputType.Touch then return end
		miniDragging = true
		miniDragMoved = 0
		local last = UIS:GetMouseLocation()
		miniDragConn = RunService.RenderStepped:Connect(function()
			if not miniDragging then return end
			local now = UIS:GetMouseLocation()
			local d = now - last
			last = now
			local travel = math.abs(d.X) + math.abs(d.Y)
			if travel > 0 then
				miniDragMoved = miniDragMoved + travel
				stepDrag(d.X, d.Y)
			end
		end)
	end)

	UIS.InputEnded:Connect(function(input)
		local t = input.UserInputType
		if t == Enum.UserInputType.MouseButton1 or t == Enum.UserInputType.Touch then finishMiniDrag() end
	end)

	local launcher = new("TextButton", {
		Name = "Launcher",
		Size = UDim2.fromOffset(52, 52),
		Position = UDim2.new(1, -24, 1, -24),
		AnchorPoint = Vector2.new(1, 1),
		BackgroundColor3 = C.panel2,
		Text = "",
		BorderSizePixel = 0,
		AutoButtonColor = false,
	}, screenGui)
	corner(launcher, 14)
	stroke(launcher, C.white, 0.9)

	local launcherIcon = makeIcon("lucide:zap", launcher, 20, C.text)
	launcherIcon.Position = UDim2.fromScale(0.5, 0.5)
	launcherIcon.AnchorPoint = Vector2.new(0.5, 0.5)

	launcher.MouseEnter:Connect(function() play(launcher, MOTION.fast, { BackgroundColor3 = C.panel3 }) end)
	launcher.MouseLeave:Connect(function() play(launcher, MOTION.fast, { BackgroundColor3 = C.panel2 }) end)

	local function setOpen(state)
		open = state
		launcher.Visible = not state

		if state then
			if windowState == "min" then
				windowState = "normal"
				body.Visible = true
				titlebar.Visible = true
				if miniBadge then miniBadge.Visible = false end
				app.BackgroundColor3 = C.bg
				applyWindowSize()
				applyWindowOffset()
			end

			app.Visible = true
			setShadowsVisible(true)
			fadeShadows(true)

			app.Position = UDim2.new(0.5, windowOffset.X, 0.5, windowOffset.Y + 14)
			shadowGroup.Position = app.Position
			uiScale.Scale = 0.97

			local settled = UDim2.new(0.5, windowOffset.X, 0.5, windowOffset.Y)
			play(app, MOTION.base, { Position = settled })
			play(shadowGroup, MOTION.base, { Position = settled })
			play(uiScale, MOTION.base, { Scale = 1 })
		else
			play(uiScale, MOTION.fast, { Scale = 0.97 }, EASE_SOFT, Enum.EasingDirection.In)
			fadeShadows(false)

			local lifted = UDim2.new(0.5, windowOffset.X, 0.5, windowOffset.Y + 10)
			play(shadowGroup, MOTION.fast, { Position = lifted }, EASE_SOFT, Enum.EasingDirection.In)
			local closing = play(app, MOTION.fast, {
				Position = lifted,
			}, EASE_SOFT, Enum.EasingDirection.In)
			closing.Completed:Connect(function()
				if not open then
					app.Visible = false
					setShadowsVisible(false)
				end
			end)
		end
	end

	windowButtons[1].MouseButton1Click:Connect(
		function() setWindowState(windowState == "min" and "normal" or "min") end
	)
	windowButtons[2].MouseButton1Click:Connect(
		function() setWindowState(windowState == "full" and "normal" or "full") end
	)
	windowButtons[3].MouseButton1Click:Connect(function() setOpen(false) end)

	launcher.MouseButton1Click:Connect(function() setOpen(true) end)

	primary.MouseButton1Click:Connect(function() ridegoUI.toggle() end)

	UIS.InputBegan:Connect(function(input, processed)
		if processed then return end

		if input.KeyCode == Enum.KeyCode.K then
			if open and windowState == "min" then
				setWindowState("normal")
			else
				setOpen(not open)
			end
			return
		end

		if not open then return end

		if input.KeyCode == Enum.KeyCode.Escape then
			setOpen(false)
			return
		end
	end)

	setOpen(true)

	return {
		open = function() setOpen(true) end,
	}
end

mountDashboard({
	isRunning = function() return S.running == true end,
	setRunning = function(on) setAuto(on) end,
	getAtur = function(key) return ATUR[key] end,
	setAtur = function(key, value) return setAtur(key, value) end,
	status = function()
		return {
			running = S.running == true,
			phase = S.phase,
			note = S.note,
			orders = S.orders,
			hops = S.hops,
			ping = S.ping,
			tripStuds = S.tripStuds,
			tripMoney = S.tripMoney,
		}
	end,
})

return { ok = true, mode = "opsa", trailNodes = #NODES }
