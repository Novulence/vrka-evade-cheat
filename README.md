# vrka-evade-cheat
local Players = game:GetService("Players")
local UIS = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local player = Players.LocalPlayer

local humanoid
local hrp

-- Toggles
local autoJumping = false
local slopeAccelEnabled = false
local invertedControls = false
local edgeJumpEnabled = false

-- Settings
local slopeBoost = 70
local boostDuration = 3.5 - used to be 4
local ledgeBoostPower = 60
local ledgeBoostCooldown = 1.5

-- State
local dragging = false
local dragOffset
local boostActive = false
local boostEndTime = 0
local grounded = false
local lastLedgeBoost = 0

-- GUI setup
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "MovementFrameGUI"
screenGui.ResetOnSpawn = false
screenGui.Parent = player:WaitForChild("PlayerGui")

local frame = Instance.new("Frame")
frame.Name = "MainFrame"
frame.Size = UDim2.new(0, 240, 0, 220)
frame.Position = UDim2.new(1, -260, 0, 20)
frame.BackgroundColor3 = Color3.fromRGB(45, 55, 60)
frame.BorderSizePixel = 0
frame.Parent = screenGui

local uiCorner = Instance.new("UICorner")
uiCorner.CornerRadius = UDim.new(0, 8)
uiCorner.Parent = frame

-- Title bar
local titleBar = Instance.new("Frame")
titleBar.Size = UDim2.new(1, 0, 0, 30)
titleBar.BackgroundColor3 = Color3.fromRGB(35, 45, 50)
titleBar.BorderSizePixel = 0
titleBar.Parent = frame

local title = Instance.new("TextLabel")
title.Size = UDim2.new(1, -30, 1, 0)
title.Position = UDim2.new(0, 8, 0, 0)
title.BackgroundTransparency = 1
title.Font = Enum.Font.GothamBold
title.TextSize = 16
title.TextColor3 = Color3.fromRGB(255, 255, 255)
title.TextXAlignment = Enum.TextXAlignment.Left
title.Text = "vrka v1.2"
title.Parent = titleBar

local closeBtn = Instance.new("TextButton")
closeBtn.Size = UDim2.new(0, 30, 1, 0)
closeBtn.Position = UDim2.new(1, -30, 0, 0)
closeBtn.BackgroundColor3 = Color3.fromRGB(200, 80, 80)
closeBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
closeBtn.Font = Enum.Font.GothamBold
closeBtn.TextSize = 16
closeBtn.Text = "X"
closeBtn.Parent = titleBar

local closeCorner = Instance.new("UICorner")
closeCorner.CornerRadius = UDim.new(0, 6)
closeCorner.Parent = closeBtn

-- Button creator
local function createButton(text, posY)
	local btn = Instance.new("TextButton")
	btn.Size = UDim2.new(0, 200, 0, 35)
	btn.Position = UDim2.new(0.5, -100, 0, posY)
	btn.BackgroundColor3 = Color3.fromRGB(70, 85, 90)
	btn.TextColor3 = Color3.fromRGB(255, 255, 255)
	btn.Font = Enum.Font.Gotham
	btn.TextSize = 15
	btn.Text = text
	btn.Parent = frame

	local corner = Instance.new("UICorner")
	corner.CornerRadius = UDim.new(0, 6)
	corner.Parent = btn

	btn.MouseEnter:Connect(function()
		btn.BackgroundColor3 = Color3.fromRGB(90, 105, 110)
	end)
	btn.MouseLeave:Connect(function()
		btn.BackgroundColor3 = Color3.fromRGB(70, 85, 90)
	end)

	return btn
end

-- Buttons
local autoJumpBtn = createButton("Auto Jump: OFF", 45)
local slopeAccelBtn = createButton("Slope Accel: OFF", 90)
local invertBtn = createButton("Inverted Controls: OFF", 135)
local edgeJumpBtn = createButton("Edge Jump: OFF", 180)

-- Character refs
local function updateCharacterRefs()
	if player.Character then
		humanoid = player.Character:FindFirstChildOfClass("Humanoid")
		hrp = player.Character:FindFirstChild("HumanoidRootPart")

		if humanoid then
			humanoid:GetPropertyChangedSignal("FloorMaterial"):Connect(function()
				grounded = humanoid.FloorMaterial ~= Enum.Material.Air
			end)
		end

		-- Detect edge contacts for ledge boost
		for _, legName in pairs({"Left Leg", "Right Leg"}) do
			local leg = player.Character:FindFirstChild(legName)
			if leg and leg:IsA("BasePart") then
				leg.Touched:Connect(function(hit)
					if edgeJumpEnabled and tick() - lastLedgeBoost > ledgeBoostCooldown then
						if not grounded and hit:IsA("BasePart") and hit.CanCollide then
							lastLedgeBoost = tick()
							local boostDir = hrp.CFrame.LookVector + Vector3.new(0, 0.4, 0)
							hrp.Velocity = hrp.Velocity + boostDir.Unit * ledgeBoostPower
						end
					end
				end)
			end
		end
	end
end

player.CharacterAdded:Connect(function(char)
	char:WaitForChild("Humanoid")
	char:WaitForChild("HumanoidRootPart")
	updateCharacterRefs()
end)
updateCharacterRefs()

-- Button toggles
autoJumpBtn.MouseButton1Click:Connect(function()
	autoJumping = not autoJumping
	autoJumpBtn.Text = autoJumping and "Auto Jump: ON" or "Auto Jump: OFF"
end)

slopeAccelBtn.MouseButton1Click:Connect(function()
	slopeAccelEnabled = not slopeAccelEnabled
	slopeAccelBtn.Text = slopeAccelEnabled and "Slope Accel: ON" or "Slope Accel: OFF"
end)

invertBtn.MouseButton1Click:Connect(function()
	invertedControls = not invertedControls
	invertBtn.Text = invertedControls and "Inverted Controls: ON" or "Inverted Controls: OFF"
end)

edgeJumpBtn.MouseButton1Click:Connect(function()
	edgeJumpEnabled = not edgeJumpEnabled
	edgeJumpBtn.Text = edgeJumpEnabled and "Edge Jump: ON" or "Edge Jump: OFF"
end)

closeBtn.MouseButton1Click:Connect(function()
	frame.Visible = false
end)

-- Boost logic
local function activateBoost()
	boostActive = true
	boostEndTime = tick() + boostDuration
end

-- Main loop
RunService.Heartbeat:Connect(function()
	if humanoid and hrp then
		-- Auto Jump
		if autoJumping and grounded then
			humanoid.Jump = true
		end

		-- Slope acceleration
		if slopeAccelEnabled then
			local ray = Ray.new(hrp.Position, Vector3.new(0, -6, 0))
			local part, pos, norm = workspace:FindPartOnRay(ray, player.Character)
			if part and norm then
				local slopeAngle = math.deg(math.acos(norm.Y))
				local moveDir = humanoid.MoveDirection
				if slopeAngle > 10 and norm.Y < 0.99 then
					local slopeDir = Vector3.new(norm.X, 0, norm.Z).Unit
					local dot = moveDir:Dot(-slopeDir)
					if dot > 0.5 then
						activateBoost()
					end
				end
			end
		end

		-- Apply slope boost
		if boostActive then
			if tick() < boostEndTime then
				local moveDir = humanoid.MoveDirection
				if invertedControls then moveDir = -moveDir end
				if moveDir.Magnitude > 0 then
					hrp.Velocity = hrp.Velocity + moveDir * slopeBoost
				end
			else
				boostActive = false
			end
		end
	end
end)

-- Dragging
titleBar.InputBegan:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseButton1 then
		dragging = true
		dragOffset = input.Position - frame.AbsolutePosition
	end
end)

titleBar.InputEnded:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseButton1 then
		dragging = false
	end
end)

UIS.InputChanged:Connect(function(input)
	if dragging and input.UserInputType == Enum.UserInputType.MouseMovement then
		frame.Position = UDim2.fromOffset(input.Position.X - dragOffset.X, input.Position.Y - dragOffset.Y)
	end
end)
