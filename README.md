--========================================================
-- LUCKY HUB
-- AUTO STEAL + AUTO RETURN
-- GOD MODE
-- FPS BOOST
-- NOCLIP
-- SPEED SLIDER
-- MOBILE + PC
-- FIXED CLICKABLE UI
-- MINI BALL LOGO
--========================================================

local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Lighting = game:GetService("Lighting")

local Player = Players.LocalPlayer
local PlayerGui = Player:WaitForChild("PlayerGui")

--========================================================
-- CONFIG
--========================================================

local MIN_SPEED = 16
local MAX_SPEED = 850
local SPEED = 350

local RETURN_HEIGHT = 2

local TEMPLE_NAME = "Titan Temple"
local SAFEZONE_NAME = "SafeZone"

local TEMPLE_EGG_RADIUS = 180

local TEMPLE_DISTANCE = 8

local EGG_APPROACH_DISTANCE = 5
local EGG_PICKUP_DISTANCE = 2.5

local SAFEZONE_DISTANCE = 8

local MOVE_TIMEOUT = 25
local PICKUP_VERIFY_TIMEOUT = 2

--========================================================
-- SETTINGS
--========================================================

local AUTO_STEAL = false
local AUTO_RETURN = false
local GOD_MODE = false
local FPS_BOOST = false
local NOCLIP = false

--========================================================
-- STATES
--========================================================

local moving = false
local missionRunning = false
local currentEgg = nil

local Character = nil
local Humanoid = nil
local HRP = nil

local oldAutoRotate = true

local connectedPrompts = {}
local godConnections = {}

--========================================================
-- CHARACTER
--========================================================

local function setupCharacter(character)

	Character = character

	Humanoid =
		character:WaitForChild(
			"Humanoid",
			10
		)

	HRP =
		character:WaitForChild(
			"HumanoidRootPart",
			10
		)

	moving = false
	missionRunning = false
	currentEgg = nil

	if Humanoid then
		oldAutoRotate =
			Humanoid.AutoRotate
	end

	if GOD_MODE then

		task.wait(0.2)

		pcall(function()

			Humanoid.BreakJointsOnDeath =
				false

			Humanoid:SetStateEnabled(
				Enum.HumanoidStateType.Dead,
				false
			)

			Humanoid:SetStateEnabled(
				Enum.HumanoidStateType.FallingDown,
				false
			)

			Humanoid:SetStateEnabled(
				Enum.HumanoidStateType.Ragdoll,
				false
			)

			Humanoid.MaxHealth =
				math.max(
					Humanoid.MaxHealth,
					1000000
				)

			Humanoid.Health =
				Humanoid.MaxHealth

		end)
	end
end

if Player.Character then
	setupCharacter(
		Player.Character
	)
end

Player.CharacterAdded:Connect(
	function(character)

		-- Tidak menyimpan posisi sebelum mati.
		-- Respawn mengikuti sistem spawn normal game.

		task.wait(0.5)

		setupCharacter(
			character
		)
	end
)

--========================================================
-- GOD MODE
--========================================================

local function disconnectGodConnections()

	for _, connection in ipairs(
		godConnections
	) do

		pcall(function()
			connection:Disconnect()
		end)

	end

	table.clear(
		godConnections
	)
end

local function applyGodMode()

	disconnectGodConnections()

	if not GOD_MODE then
		return
	end

	if not Character
		or not Humanoid then

		return
	end

	pcall(function()

		Humanoid.BreakJointsOnDeath =
			false

		Humanoid:SetStateEnabled(
			Enum.HumanoidStateType.Dead,
			false
		)

		Humanoid:SetStateEnabled(
			Enum.HumanoidStateType.FallingDown,
			false
		)

		Humanoid:SetStateEnabled(
			Enum.HumanoidStateType.Ragdoll,
			false
		)

		Humanoid.MaxHealth =
			math.max(
				Humanoid.MaxHealth,
				1000000
			)

		Humanoid.Health =
			Humanoid.MaxHealth

	end)

	table.insert(
		godConnections,

		Humanoid.HealthChanged:Connect(
			function(health)

				if not GOD_MODE then
					return
				end

				if Humanoid
					and Humanoid.Parent
					and health
						< Humanoid.MaxHealth then

					Humanoid.Health =
						Humanoid.MaxHealth
				end
			end
		)
	)

	table.insert(
		godConnections,

		Humanoid.StateChanged:Connect(
			function(_, state)

				if not GOD_MODE then
					return
				end

				if state ==
					Enum.HumanoidStateType.Dead then

					pcall(function()

						Humanoid:SetStateEnabled(
							Enum.HumanoidStateType.Dead,
							false
						)

						Humanoid.Health =
							Humanoid.MaxHealth

					end)
				end
			end
		)
	)
end

local function disableGodMode()

	disconnectGodConnections()

	if Humanoid
		and Humanoid.Parent then

		pcall(function()

			Humanoid:SetStateEnabled(
				Enum.HumanoidStateType.Dead,
				true
			)

			Humanoid:SetStateEnabled(
				Enum.HumanoidStateType.FallingDown,
				true
			)

			Humanoid:SetStateEnabled(
				Enum.HumanoidStateType.Ragdoll,
				true
			)

			Humanoid.BreakJointsOnDeath =
				true

		end)
	end
end

local function setGodMode(enabled)

	GOD_MODE = enabled

	if GOD_MODE then
		applyGodMode()
	else
		disableGodMode()
	end
end

--========================================================
-- NOCLIP
--========================================================

local function updateNoclip()

	if not NOCLIP
		or not Character then

		return
	end

	for _, object in ipairs(
		Character:GetDescendants()
	) do

		if object:IsA("BasePart") then
			object.CanCollide = false
		end
	end
end

RunService.Stepped:Connect(
	function()

		if NOCLIP then
			updateNoclip()
		end
	end
)

--========================================================
-- FIND SAFEZONE
--========================================================

local function findSafeZone()

	local object =
		Workspace:FindFirstChild(
			SAFEZONE_NAME,
			true
		)

	if object then

		if object:IsA("BasePart") then
			return object
		end

		if object:IsA("Model") then

			if object.PrimaryPart then
				return object.PrimaryPart
			end

			local part =
				object:FindFirstChildWhichIsA(
					"BasePart",
					true
				)

			if part then
				return part
			end
		end
	end

	for _, object2 in ipairs(
		Workspace:GetDescendants()
	) do

		if object2.Name:lower()
			== SAFEZONE_NAME:lower() then

			if object2:IsA("BasePart") then
				return object2
			end

			if object2:IsA("Model") then

				if object2.PrimaryPart then
					return object2.PrimaryPart
				end

				local part =
					object2:FindFirstChildWhichIsA(
						"BasePart",
						true
					)

				if part then
					return part
				end
			end
		end
	end

	return nil
end

--========================================================
-- FIND TEMPLE
--========================================================

local function findTemple()

	local temple =
		Workspace:FindFirstChild(
			TEMPLE_NAME,
			true
		)

	if temple then
		return temple
	end

	for _, object in ipairs(
		Workspace:GetDescendants()
	) do

		if object.Name:lower()
			== TEMPLE_NAME:lower() then

			return object
		end
	end

	return nil
end

local function getObjectPosition(object)

	if not object then
		return nil
	end

	if object:IsA("BasePart") then
		return object.Position
	end

	if object:IsA("Model") then

		if object.PrimaryPart then
			return object.PrimaryPart.Position
		end

		local part =
			object:FindFirstChildWhichIsA(
				"BasePart",
				true
			)

		if part then
			return part.Position
		end
	end

	return nil
end

--========================================================
-- EGG CHECK
--========================================================

local function isEggPrompt(prompt)

	if not prompt
		or not prompt:IsA(
			"ProximityPrompt"
		) then

		return false
	end

	local text =
		string.lower(
			(prompt.Name or "")
			.. " "
			.. (prompt.ActionText or "")
			.. " "
			.. (prompt.ObjectText or "")
		)

	if prompt.Parent then

		text =
			text
			.. " "
			.. string.lower(
				prompt.Parent.Name
			)
	end

	return string.find(
		text,
		"egg"
	) ~= nil
end

--========================================================
-- FIND EGG
--========================================================

local function findEggInTemple()

	if not HRP then
		return nil
	end

	local temple =
		findTemple()

	if not temple then
		return nil
	end

	local templePosition =
		getObjectPosition(
			temple
		)

	if not templePosition then
		return nil
	end

	local closest = nil
	local closestDistance = math.huge

	for _, object in ipairs(
		Workspace:GetDescendants()
	) do

		if object:IsA(
			"ProximityPrompt"
		)
		and isEggPrompt(object) then

			local parent =
				object.Parent

			if parent
				and parent:IsA("BasePart") then

				local templeDistance =
					(
						parent.Position
						- templePosition
					).Magnitude

				if templeDistance
					<= TEMPLE_EGG_RADIUS then

					local playerDistance =
						(
							parent.Position
							- HRP.Position
						).Magnitude

					if playerDistance
						< closestDistance then

						closest =
							object

						closestDistance =
							playerDistance
					end
				end
			end
		end
	end

	return closest
end

--========================================================
-- STRAIGHT MOVEMENT
--========================================================

local function moveToPosition(
	targetPosition,
	stopDistance
)

	if not HRP
		or not HRP.Parent
		or not Humanoid
		or not Humanoid.Parent then

		return false
	end

	moving = true

	local startTime =
		os.clock()

	while moving
		and Character
		and Character.Parent
		and HRP
		and HRP.Parent
		and Humanoid
		and Humanoid.Parent do

		if os.clock()
			- startTime
			> MOVE_TIMEOUT then

			break
		end

		local offset =
			targetPosition
			- HRP.Position

		local distance =
			offset.Magnitude

		if distance <= stopDistance then
			break
		end

		local direction =
			offset.Unit

		Humanoid.WalkSpeed =
			SPEED

		Humanoid.AutoRotate =
			false

		HRP.AssemblyLinearVelocity =
			Vector3.new(
				direction.X * SPEED,
				direction.Y * SPEED,
				direction.Z * SPEED
			)

		HRP.CFrame =
			CFrame.lookAt(
				HRP.Position,
				HRP.Position + direction,
				Vector3.yAxis
			)

		RunService.Heartbeat:Wait()
	end

	if HRP
		and HRP.Parent then

		local velocity =
			HRP.AssemblyLinearVelocity

		HRP.AssemblyLinearVelocity =
			Vector3.new(
				0,
				velocity.Y,
				0
			)
	end

	moving = false

	return true
end

--========================================================
-- RETURN
--========================================================

local function returnToSafeZone()

	if not AUTO_RETURN then
		return
	end

	if not HRP
		or not HRP.Parent then

		return
	end

	local safeZone =
		findSafeZone()

	if not safeZone then
		return
	end

	local target =
		safeZone.Position
		+ Vector3.new(
			0,
			RETURN_HEIGHT,
			0
		)

	moveToPosition(
		target,
		SAFEZONE_DISTANCE
	)

	if Humanoid
		and Humanoid.Parent then

		Humanoid.AutoRotate =
			oldAutoRotate
	end
end

--========================================================
-- PROMPT ACTIVATION
--========================================================

local function activatePrompt(prompt)

	if not prompt
		or not prompt:IsA(
			"ProximityPrompt"
		) then

		return false
	end

	if not prompt.Enabled then
		return false
	end

	local success =
		pcall(function()

			prompt:InputHoldBegin()

			if prompt.HoldDuration > 0 then

				task.wait(
					prompt.HoldDuration
				)
			end

			prompt:InputHoldEnd()

		end)

	return success
end

--========================================================
-- AUTO STEAL
--========================================================

local function autoStealMission()

	if missionRunning then
		return
	end

	if not AUTO_STEAL then
		return
	end

	if not Character
		or not Character.Parent
		or not HRP
		or not HRP.Parent
		or not Humanoid then

		return
	end

	missionRunning = true

	--====================================================
	-- TEMPLE
	--====================================================

	local temple =
		findTemple()

	if not temple then

		missionRunning = false

		return
	end

	local templePosition =
		getObjectPosition(
			temple
		)

	if not templePosition then

		missionRunning = false

		return
	end

	--====================================================
	-- MOVE TO TEMPLE
	--====================================================

	if (
		HRP.Position
		- templePosition
	).Magnitude > TEMPLE_DISTANCE then

		moveToPosition(
			templePosition,
			TEMPLE_DISTANCE
		)
	end

	if not AUTO_STEAL then

		missionRunning = false

		return
	end

	--====================================================
	-- FIND EGG
	--====================================================

	local egg =
		findEggInTemple()

	if not egg then

		missionRunning = false

		return
	end

	currentEgg =
		egg

	local eggPart =
		egg.Parent

	if not eggPart
		or not eggPart:IsA(
			"BasePart"
		) then

		currentEgg = nil
		missionRunning = false

		return
	end

	--====================================================
	-- APPROACH EGG
	--====================================================

	local eggPosition =
		eggPart.Position

	local direction =
		HRP.Position
		- eggPosition

	if direction.Magnitude < 0.1 then

		direction =
			Vector3.new(
				0,
				0,
				1
			)
	end

	direction =
		direction.Unit

	local approachPosition =
		eggPosition
		+ direction
		* EGG_APPROACH_DISTANCE

	moveToPosition(
		approachPosition,
		EGG_PICKUP_DISTANCE
	)

	--====================================================
	-- PICKUP
	--====================================================

	if AUTO_STEAL
		and currentEgg
		and currentEgg.Parent then

		activatePrompt(
			currentEgg
		)

		task.wait(
			PICKUP_VERIFY_TIMEOUT
		)
	end

	currentEgg = nil

	--====================================================
	-- RETURN
	--====================================================

	if AUTO_STEAL
		and AUTO_RETURN then

		returnToSafeZone()
	end

	missionRunning = false
end

--========================================================
-- PROMPT CONNECTION
--========================================================

local function connectPrompt(prompt)

	if not prompt:IsA(
		"ProximityPrompt"
	) then

		return
	end

	if connectedPrompts[prompt] then
		return
	end

	if not isEggPrompt(prompt) then
		return
	end

	connectedPrompts[prompt] =
		true

	prompt.Triggered:Connect(
		function(triggeringPlayer)

			if triggeringPlayer
				and triggeringPlayer
					~= Player then

				return
			end

			if AUTO_RETURN
				and AUTO_STEAL then

				task.spawn(
					returnToSafeZone
				)
			end
		end
	)
end

for _, object in ipairs(
	Workspace:GetDescendants()
) do

	if object:IsA(
		"ProximityPrompt"
	) then

		connectPrompt(
			object
		)
	end
end

Workspace.DescendantAdded:Connect(
	function(object)

		if object:IsA(
			"ProximityPrompt"
		) then

			task.wait()

			pcall(function()
				connectPrompt(
					object
				)
			end)
		end
	end
)

--========================================================
-- REMOVE OLD UI
--========================================================

local oldUI =
	PlayerGui:FindFirstChild(
		"LuckyHubUI"
	)

if oldUI then
	oldUI:Destroy()
end

--========================================================
-- SCREEN GUI
--========================================================

local ScreenGui =
	Instance.new("ScreenGui")

ScreenGui.Name =
	"LuckyHubUI"

ScreenGui.ResetOnSpawn =
	false

ScreenGui.IgnoreGuiInset =
	true

ScreenGui.ZIndexBehavior =
	Enum.ZIndexBehavior.Sibling

ScreenGui.DisplayOrder =
	9999

ScreenGui.Parent =
	PlayerGui

--========================================================
-- MAIN
--========================================================

local Main =
	Instance.new("Frame")

Main.Name =
	"Main"

Main.Size =
	UDim2.fromOffset(
		270,
		300
	)

Main.AnchorPoint =
	Vector2.new(
		0.5,
		0.5
	)

Main.Position =
	UDim2.new(
		0.5,
		0,
		0.5,
		0
	)

Main.BackgroundColor3 =
	Color3.fromRGB(
		24,
		24,
		28
	)

Main.BackgroundTransparency =
	0

Main.BorderSizePixel =
	0

Main.Active =
	false

Main.Visible =
	true

Main.ZIndex =
	1

Main.Parent =
	ScreenGui

local MainCorner =
	Instance.new("UICorner")

MainCorner.CornerRadius =
	UDim.new(
		0,
		14
	)

MainCorner.Parent =
	Main

local MainStroke =
	Instance.new("UIStroke")

MainStroke.Thickness =
	2

MainStroke.Color =
	Color3.fromRGB(
		255,
		205,
		60
	)

MainStroke.Parent =
	Main

--========================================================
-- DRAG HANDLE
--========================================================

local DragHandle =
	Instance.new("TextButton")

DragHandle.Name =
	"DragHandle"

DragHandle.Size =
	UDim2.new(
		1,
		-65,
		0,
		45
	)

DragHandle.Position =
	UDim2.fromOffset(
		8,
		4
	)

DragHandle.BackgroundTransparency =
	1

DragHandle.BorderSizePixel =
	0

DragHandle.Text =
	""

DragHandle.AutoButtonColor =
	false

DragHandle.Active =
	true

DragHandle.Selectable =
	false

DragHandle.ZIndex =
	2

DragHandle.Parent =
	Main

--========================================================
-- TITLE
--========================================================

local Title =
	Instance.new("TextLabel")

Title.Size =
	UDim2.new(
		1,
		-65,
		0,
		40
	)

Title.Position =
	UDim2.fromOffset(
		12,
		5
	)

Title.BackgroundTransparency =
	1

Title.Text =
	"LUCKY HUB"

Title.TextColor3 =
	Color3.fromRGB(
		255,
		205,
		60
	)

Title.TextSize =
	20

Title.Font =
	Enum.Font.GothamBold

Title.TextXAlignment =
	Enum.TextXAlignment.Left

Title.Active =
	false

Title.ZIndex =
	3

Title.Parent =
	Main

--========================================================
-- CLOSE
--========================================================

local Close =
	Instance.new("TextButton")

Close.Name =
	"Close"

Close.Size =
	UDim2.fromOffset(
		34,
		34
	)

Close.Position =
	UDim2.new(
		1,
		-42,
		0,
		8
	)

Close.BackgroundColor3 =
	Color3.fromRGB(
		55,
		55,
		62
	)

Close.BorderSizePixel =
	0

Close.Text =
	"X"

Close.TextColor3 =
	Color3.fromRGB(
		255,
		255,
		255
	)

Close.TextSize =
	16

Close.Font =
	Enum.Font.GothamBold

Close.Active =
	true

Close.Selectable =
	true

Close.AutoButtonColor =
	true

Close.Visible =
	true

Close.ZIndex =
	30

Close.Parent =
	Main

local CloseCorner =
	Instance.new("UICorner")

CloseCorner.CornerRadius =
	UDim.new(
		0,
		8
	)

CloseCorner.Parent =
	Close

--========================================================
-- STATUS
--========================================================

local Status =
	Instance.new("TextLabel")

Status.Size =
	UDim2.new(
		1,
		-20,
		0,
		25
	)

Status.Position =
	UDim2.fromOffset(
		10,
		42
	)

Status.BackgroundTransparency =
	1

Status.Text =
	"LUCKY HUB READY"

Status.TextColor3 =
	Color3.fromRGB(
		190,
		190,
		190
	)

Status.TextSize =
	12

Status.Font =
	Enum.Font.Gotham

Status.Visible =
	true

Status.ZIndex =
	3

Status.Parent =
	Main

--========================================================
-- BUTTON CREATOR
--========================================================

local function createButton(
	name,
	text,
	y
)

	local button =
		Instance.new("TextButton")

	button.Name =
		name

	button.Size =
		UDim2.new(
			1,
			-30,
			0,
			35
		)

	button.Position =
		UDim2.fromOffset(
			15,
			y
		)

	button.BackgroundColor3 =
		Color3.fromRGB(
			42,
			42,
			48
		)

	button.BackgroundTransparency =
		0

	button.BorderSizePixel =
		0

	button.Text =
		text

	button.TextColor3 =
		Color3.fromRGB(
			255,
			255,
			255
		)

	button.TextTransparency =
		0

	button.TextSize =
		13

	button.Font =
		Enum.Font.GothamBold

	button.Visible =
		true

	button.Active =
		true

	button.Selectable =
		true

	button.AutoButtonColor =
		true

	button.ZIndex =
		20

	button.Parent =
		Main

	local corner =
		Instance.new("UICorner")

	corner.CornerRadius =
		UDim.new(
			0,
			8
		)

	corner.Parent =
		button

	return button
end

--========================================================
-- FEATURE BUTTONS
--========================================================

local AutoStealButton =
	createButton(
		"AutoSteal",
		"AUTO STEAL : OFF",
		68
	)

local AutoReturnButton =
	createButton(
		"AutoReturn",
		"AUTO RETURN : O --========================================================
-- CONTINUE FROM AUTO RETURN BUTTON
--========================================================

local AutoReturnButton =
	createButton(
		"AutoReturn",
		"AUTO RETURN : OFF",
		108
	)

local GodModeButton =
	createButton(
		"GodMode",
		"GOD MODE : OFF",
		148
	)

local FPSBoostButton =
	createButton(
		"FPSBoost",
		"FPS BOOST : OFF",
		188
	)

--========================================================
-- SPEED LABEL
--========================================================

local SpeedLabel =
	Instance.new("TextLabel")

SpeedLabel.Name =
	"SpeedLabel"

SpeedLabel.Size =
	UDim2.new(
		1,
		-30,
		0,
		22
	)

SpeedLabel.Position =
	UDim2.fromOffset(
		15,
		228
	)

SpeedLabel.BackgroundTransparency =
	1

SpeedLabel.Text =
	"SPEED : "
		.. tostring(SPEED)

SpeedLabel.TextColor3 =
	Color3.fromRGB(
		255,
		205,
		60
	)

SpeedLabel.TextSize =
	13

SpeedLabel.Font =
	Enum.Font.GothamBold

SpeedLabel.TextXAlignment =
	Enum.TextXAlignment.Left

SpeedLabel.ZIndex =
	20

SpeedLabel.Parent =
	Main

--========================================================
-- SPEED SLIDER
--========================================================

local Slider =
	Instance.new("Frame")

Slider.Name =
	"SpeedSlider"

Slider.Size =
	UDim2.new(
		1,
		-30,
		0,
		8
	)

Slider.Position =
	UDim2.fromOffset(
		15,
		258
	)

Slider.BackgroundColor3 =
	Color3.fromRGB(
		55,
		55,
		62
	)

Slider.BorderSizePixel =
	0

Slider.Active =
	true

Slider.ZIndex =
	20

Slider.Parent =
	Main

local SliderCorner =
	Instance.new("UICorner")

SliderCorner.CornerRadius =
	UDim.new(
		1,
		0
	)

SliderCorner.Parent =
	Slider

local SliderFill =
	Instance.new("Frame")

SliderFill.Name =
	"Fill"

SliderFill.Size =
	UDim2.new(
		(SPEED - MIN_SPEED)
			/ (MAX_SPEED - MIN_SPEED),
		0,
		1,
		0
	)

SliderFill.BackgroundColor3 =
	Color3.fromRGB(
		255,
		205,
		60
	)

SliderFill.BorderSizePixel =
	0

SliderFill.ZIndex =
	21

SliderFill.Parent =
	Slider

local FillCorner =
	Instance.new("UICorner")

FillCorner.CornerRadius =
	UDim.new(
		1,
		0
	)

FillCorner.Parent =
	SliderFill

local Knob =
	Instance.new("TextButton")

Knob.Name =
	"Knob"

Knob.Size =
	UDim2.fromOffset(
		18,
		18
	)

Knob.AnchorPoint =
	Vector2.new(
		0.5,
		0.5
	)

Knob.Position =
	UDim2.new(
		(SPEED - MIN_SPEED)
			/ (MAX_SPEED - MIN_SPEED),
		0,
		0.5,
		0
	)

Knob.BackgroundColor3 =
	Color3.fromRGB(
		255,
		220,
		100
	)

Knob.BorderSizePixel =
	0

Knob.Text =
	""

Knob.Active =
	true

Knob.Selectable =
	false

Knob.ZIndex =
	22

Knob.Parent =
	Slider

local KnobCorner =
	Instance.new("UICorner")

KnobCorner.CornerRadius =
	UDim.new(
		1,
		0
	)

KnobCorner.Parent =
	Knob

--========================================================
-- SPEED UPDATE
--========================================================

local function updateSpeedFromX(x)

	local sliderPosition =
		Slider.AbsolutePosition.X

	local sliderSize =
		Slider.AbsoluteSize.X

	local percent =
		(x - sliderPosition)
		/ sliderSize

	percent =
		math.clamp(
			percent,
			0,
			1
		)

	SPEED =
		math.floor(
			MIN_SPEED
			+ (
				MAX_SPEED
				- MIN_SPEED
			)
			* percent
		)

	SpeedLabel.Text =
		"SPEED : "
		.. tostring(SPEED)

	SliderFill.Size =
		UDim2.new(
			percent,
			0,
			1,
			0
		)

	Knob.Position =
		UDim2.new(
			percent,
			0,
			0.5,
			0
		)
end

local sliderDragging =
	false

Slider.InputBegan:Connect(
	function(input)

		if input.UserInputType
			== Enum.UserInputType.MouseButton1
			or input.UserInputType
			== Enum.UserInputType.Touch then

			sliderDragging =
				true

			updateSpeedFromX(
				input.Position.X
			)
		end
	end
)

Knob.InputBegan:Connect(
	function(input)

		if input.UserInputType
			== Enum.UserInputType.MouseButton1
			or input.UserInputType
			== Enum.UserInputType.Touch then

			sliderDragging =
				true
		end
	end
)

UserInputService.InputChanged:Connect(
	function(input)

		if not sliderDragging then
			return
		end

		if input.UserInputType
			== Enum.UserInputType.MouseMovement
			or input.UserInputType
			== Enum.UserInputType.Touch then

			updateSpeedFromX(
				input.Position.X
			)
		end
	end
)

UserInputService.InputEnded:Connect(
	function(input)

		if input.UserInputType
			== Enum.UserInputType.MouseButton1
			or input.UserInputType
			== Enum.UserInputType.Touch then

			sliderDragging =
				false
		end
	end
)

--========================================================
-- FPS BOOST
--========================================================

local function setFPSBoost(enabled)

	FPS_BOOST =
		enabled

	if not FPS_BOOST then
		return
	end

	pcall(function()

		Lighting.GlobalShadows =
			false

		Lighting.FogEnd =
			100000

		for _, object in ipairs(
			Workspace:GetDescendants()
		) do

			if object:IsA("ParticleEmitter")
				or object:IsA("Trail")
				or object:IsA("Smoke")
				or object:IsA("Fire")
				or object:IsA("Sparkles") then

				object.Enabled =
					false
			end
		end
	end)
end

--========================================================
-- AUTO STEAL BUTTON
--========================================================

AutoStealButton.Activated:Connect(
	function()

		AUTO_STEAL =
			not AUTO_STEAL

		if AUTO_STEAL then

			AutoStealButton.Text =
				"AUTO STEAL : ON"

			Status.Text =
				"AUTO STEAL ENABLED"

			task.spawn(
				function()

					while AUTO_STEAL do

						if not missionRunning then

							autoStealMission()
						end

						task.wait(
							0.25
						)
					end
				end
			)

		else

			AutoStealButton.Text =
				"AUTO STEAL : OFF"

			moving =
				false

			missionRunning =
				false

			Status.Text =
				"LUCKY HUB READY"
		end
	end
)

--========================================================
-- AUTO RETURN BUTTON
--========================================================

AutoReturnButton.Activated:Connect(
	function()

		AUTO_RETURN =
			not AUTO_RETURN

		if AUTO_RETURN then

			AutoReturnButton.Text =
				"AUTO RETURN : ON"

			Status.Text =
				"AUTO RETURN ENABLED"

		else

			AutoReturnButton.Text =
				"AUTO RETURN : OFF"

			Status.Text =
				"AUTO RETURN DISABLED"
		end
	end
)

--========================================================
-- GOD MODE BUTTON
--========================================================

GodModeButton.Activated:Connect(
	function()

		GOD_MODE =
			not GOD_MODE

		setGodMode(
			GOD_MODE
		)

		if GOD_MODE then

			GodModeButton.Text =
				"GOD MODE : ON"

			Status.Text =
				"GOD MODE ENABLED"

		else

			GodModeButton.Text =
				"GOD MODE : OFF"

			Status.Text =
				"GOD MODE DISABLED"
		end
	end
)

--========================================================
-- FPS BUTTON
--========================================================

FPSBoostButton.Activated:Connect(
	function()

		FPS_BOOST =
			not FPS_BOOST

		setFPSBoost(
			FPS_BOOST
		)

		if FPS_BOOST then

			FPSBoostButton.Text =
				"FPS BOOST : ON"

			Status.Text =
				"FPS BOOST ENABLED"

		else

			FPSBoostButton.Text =
				"FPS BOOST : OFF"

			Status.Text =
				"FPS BOOST DISABLED"
		end
	end
)

--========================================================
-- CLOSE / OPEN
--========================================================

local Mini =
	Instance.new("TextButton")

Mini.Name =
	"MiniBall"

Mini.Size =
	UDim2.fromOffset(
		52,
		52
	)

Mini.AnchorPoint =
	Vector2.new(
		0,
		0.5
	)

Mini.Position =
	UDim2.new(
		0,
		10,
		0.5,
		0
	)

Mini.BackgroundColor3 =
	Color3.fromRGB(
		24,
		24,
		28
	)

Mini.BorderSizePixel =
	0

Mini.Text =
	"LUCKY"

Mini.TextColor3 =
	Color3.fromRGB(
		255,
		205,
		60
	)

Mini.TextSize =
	10

Mini.Font =
	Enum.Font.GothamBold

Mini.Active =
	true

Mini.Selectable =
	true

Mini.AutoButtonColor =
	true

Mini.ZIndex =
	50

Mini.Visible =
	false

Mini.Parent =
	ScreenGui

local MiniCorner =
	Instance.new("UICorner")

MiniCorner.CornerRadius =
	UDim.new(
		1,
		0
	)

MiniCorner.Parent =
	Mini

local MiniStroke =
	Instance.new("UIStroke")

MiniStroke.Thickness =
	2

MiniStroke.Color =
	Color3.fromRGB(
		255,
		205,
		60
	)

MiniStroke.Parent =
	Mini

Close.Activated:Connect(
	function()

		Main.Visible =
			false

		Mini.Visible =
			true
	end
)

Mini.Activated:Connect(
	function()

		Main.Visible =
			true

		Mini.Visible =
			false
	end
)

--========================================================
-- DRAG MAIN
-- ONLY DRAG HANDLE
--========================================================

local dragging =
	false

local dragStart =
	nil

local startPosition =
	nil

local dragInput =
	nil

DragHandle.InputBegan:Connect(
	function(input)

		if input.UserInputType
			== Enum.UserInputType.MouseButton1
			or input.UserInputType
			== Enum.UserInputType.Touch then

			dragging =
				true

			dragStart =
				input.Position

			startPosition =
				Main.Position

			dragInput =
				input
		end
	end
)

DragHandle.InputChanged:Connect(
	function(input)

		if input.UserInputType
			== Enum.UserInputType.MouseMovement
			or input.UserInputType
			== Enum.UserInputType.Touch then

			dragInput =
				input
		end
	end
)

UserInputService.InputChanged:Connect(
	function(input)

		if not dragging then
			return
		end

		if input
			~= dragInput then

			return
		end

		local delta =
			input.Position
			- dragStart

		Main.Position =
			UDim2.new(
				startPosition.X.Scale,
				startPosition.X.Offset
					+ delta.X,

				startPosition.Y.Scale,
				startPosition.Y.Offset
					+ delta.Y
			)
	end
)

UserInputService.InputEnded:Connect(
	function(input)

		if input.UserInputType
			== Enum.UserInputType.MouseButton1
			or input.UserInputType
			== Enum.UserInputType.Touch then

			dragging =
				false

			dragInput =
				nil
		end
	end
)

--========================================================
-- STATUS LOOP
--========================================================

task.spawn(
	function()

		while ScreenGui
			and ScreenGui.Parent do

			if missionRunning then

				Status.Text =
					"AUTO STEAL RUNNING..."

			elseif moving then

				Status.Text =
					"MOVING..."

			elseif GOD_MODE then

				Status.Text =
					"GOD MODE ACTIVE"

			elseif AUTO_STEAL then

				Status.Text =
					"AUTO STEAL : READY"

			else

				Status.Text =
					"LUCKY HUB READY"
			end

			task.wait(
				0.2
			)
		end
	end
)

--========================================================
-- READY
--========================================================

Main.Visible =
	true

Mini.Visible =
	false

print(
	"[LuckyHub] Loaded successfully."
)
