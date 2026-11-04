# M1 Combat System (Roblox R6)

A single-script, unified M1 combat system for Roblox R6. Everything runs in one LocalScript placed inside a Tool — no external folders or dependencies in ReplicatedStorage.

## How it works

- Detects a left mouse click while the tool is equipped.
- Plays the default R6 punching animation.
- Builds a temporary invisible front-facing hitbox.
- Finds enemies overlapping the hitbox.
- Deals **15** damage to the first victim.
- Applies a knockback impulse sending the victim away from the attacker.
- Handles a small cooldown between punches.

## Install

1. In Roblox Studio, insert a new **Tool** into `StarterPack`.
2. Add a **LocalScript** inside the Tool.
3. Paste the contents of the script (below) into that LocalScript.
4. Optionally set a handle/mesh on the tool and a description.
5. Test in Play Solo.

## Use

- Equip the tool from the backpack.
- Click the left mouse button to punch.
- Each punch deals 15 damage and knocks back the first enemy hit.
- Wait 0.5 seconds between punches (cooldown).

## Configuration

Edit the constants at the top of the script to adjust combat behaviour:

| Constant | Meaning | Default |
|----------|---------|---------|
| `PUNCH_DAMAGE` | Damage per punch | 15 |
| `KNOCKBACK_FORCE` | Horizontal knockback | 80 |
| `KNOCKBACK_UPWARD` | Vertical knockback | 40 |
| `RANGE` | Hitbox reach | 4.5 |
| `ARC_RADIUS` | Hitbox width | 4 |
| `COOLDOWN` | Seconds between punches | 0.5 |

## Notes

- Uses the built-in R6 punch animation (asset id `2710428218`).
- Damage and knockback are applied client-side. For production use with anti-cheat, move damage/validation to a server Script.

## Script (place inside Tool as LocalScript)

```lua
--!strict
-- M1 Combat System for Roblox R6 (unified LocalScript inside a Tool)
-- Place this LocalScript as a child of a Tool in StarterPack.
-- No external dependencies: everything runs in this one script.

local Players = game:GetService("Players")

local tool = script.Parent
local character = tool.Parent

-- Tunable combat values
local PUNCH_DAMAGE = 15
local KNOCKBACK_FORCE = 80
local KNOCKBACK_UPWARD = 40
local RANGE = 4.5
local ARC_RADIUS = 4
local COOLDOWN = 0.5

-- Animation constants for the default R6 punch animation
local PUNCH_ANIMATION_ID = "rbxassetid://2710428218"
local PUNCH_TRACK_PRIORITY = Enum.AnimationPriority.Action

local function getHumanoid()
	local model = tool.Parent
	if not model then return nil end
	local humanoid = model:FindFirstChildOfClass("Humanoid")
	return humanoid
end

local function getHumanoidRootPart()
	local model = tool.Parent
	if not model then return nil end
	return model:FindFirstChild("HumanoidRootPart")
end

local function playPunchAnimation(humanoid)
	if not humanoid then return end

	-- Load the default R6 punch / weapon swing animation
	local animator = humanoid:FindFirstChildOfClass("Animator")
	if not animator then return end

	local animation = Instance.new("Animation")
	animation.AnimationId = PUNCH_ANIMATION_ID

	local track
	local ok, err = pcall(function()
		track = animator:LoadAnimation(animation)
	end)

	if not ok or not track then return end

	track.Priority = PUNCH_TRACK_PRIORITY
	track:Play()
	animation:Destroy()

	return track
end

-- Build a front-facing damage area: a box in front of the character.
local function buildHitbox(rootPart)
	local hitbox = Instance.new("Part")
	hitbox.Name = "M1Hitbox"
	hitbox.Size = Vector3.new(ARC_RADIUS, 5, RANGE)
	hitbox.Transparency = 1
	hitbox.CanCollide = false
	hitbox.CanQuery = false
	hitbox.CanTouch = false
	hitbox.Anchored = true

	-- Position it in front of the character, facing forward
	hitbox.CFrame = rootPart.CFrame
		* CFrame.new(0, 0, -RANGE / 2)
		* CFrame.Angles(0, math.rad(90), 0)

	return hitbox
end

local function applyKnockback(targetRootPart, direction)
	if not targetRootPart then return end

	local knockback = Instance.new("VectorForce")
	knockback.Name = "M1Knockback"
	knockback.Force = (direction * KNOCKBACK_FORCE) + (Vector3.new(0, KNOCKBACK_UPWARD, 0))
	knockback.RelativeTo = Enum.ActuatorRelativeTo.World
	knockback.ApplyAtCenterOfMass = true

	local attachment = Instance.new("Attachment")
	attachment.Name = "M1KnockbackAttachment"
	attachment.Parent = targetRootPart

	knockback.Attachment0 = attachment
	knockback.Parent = targetRootPart

	-- Remove the force after a short moment
	task.delay(0.1, function()
		knockback:Destroy()
		attachment:Destroy()
	end)
end

local function resolveDamage(target)
	local humanoid = target:FindFirstChildOfClass("Humanoid")
	if not humanoid then return nil end
	if humanoid.Health <= 0 then return nil end
	return humanoid
end

local function onPunch()
	local humanoid = getHumanoid()
	local rootPart = getHumanoidRootPart()
	if not humanoid or not rootPart then return end

	-- Cooldown check
	if rootPart:FindFirstChild("M1Cooldown") then return end

	-- Mark cooldown
	local cooldownToken = Instance.new("BoolValue")
	cooldownToken.Name = "M1Cooldown"
	cooldownToken.Parent = rootPart
	task.delay(COOLDOWN, function()
		cooldownToken:Destroy()
	end)

	-- Play the punching animation
	local track = playPunchAnimation(humanoid)

	-- Build the front hitbox to detect enemies
	local hitbox = buildHitbox(rootPart)
	hitbox.Parent = workspace

	task.delay(0.05, function()
		-- Gather overlapping parts with the hitbox
		local params = OverlapParams.new()
		params.FilterType = Enum.RaycastFilterType.Blacklist
		params.FilterDescendantsInstances = { character, tool }
		params.RespectCanCollide = false

		local hits = workspace:GetPartBoundsInBox(hitbox.CFrame, hitbox.Size, params)

		local hitHumanoids = {}
		for _, part in ipairs(hits) do
			local targetModel = part.Parent
			if targetModel and targetModel ~= character then
				local targetHumanoid = resolveDamage(targetModel)
				if targetHumanoid and not hitHumanoids[targetModel] then
					hitHumanoids[targetModel] = true

					-- Apply damage
					targetHumanoid:TakeDamage(PUNCH_DAMAGE)

					-- Apply knockback away from the attacker
					local targetRoot = targetModel:FindFirstChild("HumanoidRootPart")
					if targetRoot then
						local direction = (targetRoot.Position - rootPart.Position).Unit
						applyKnockback(targetRoot, direction)
					end

					break -- Only damage the first (closest) victim per punch
				end
			end
		end

		hitbox:Destroy()
	end)
end

-- Connect to mouse clicks. The Tool must be equipped for clicks to fire.
local function setupInput(equipped)
	if not equipped then return end

	if tool:FindFirstChild("M1Connection") then return end

	local player = Players.LocalPlayer
	if not player then return end

	local mouse = player:GetMouse()
	if not mouse then return end

	local connection = mouse.Button1Down:Connect(function()
		onPunch()
	end)

	-- Store the connection so we can clean it up on unequip
	local connectionValue = Instance.new("ObjectValue")
	connectionValue.Name = "M1Connection"
	connectionValue.Value = connection
	connectionValue.Parent = tool
end

local function teardownInput()
	local connectionValue = tool:FindFirstChild("M1Connection")
	if not connectionValue then return end

	local connection = connectionValue.Value
	if connection and typeof(connection) == "RBXScriptConnection" then
		connection:Disconnect()
	end

	connectionValue:Destroy()
end

tool.Equipped:Connect(function()
	setupInput(true)
end)

tool.Unequipped:Connect(function()
	teardownInput()
end)
```

---

More Roblox script projects by EnderDevelopment: https://github.com/EnderDevelopment

Project page: https://dash.enderdevelopment.com/community/ae0f6424-7866-4548-a2c8-d063e52b1faa?utm_source=github&utm_medium=readme&utm_campaign=auto_publish&utm_content=ae0f6424-7866-4548-a2c8-d063e52b1faa
