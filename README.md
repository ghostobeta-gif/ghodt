local Players = game:GetService("Players")
local UIS = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local player = Players.LocalPlayer
local purple = Color3.fromRGB(170, 65, 255)
local saved, ball, beam, link
local noclip, busy = false, false
local originals = {}
local generation = 0
local activeTween, locked, oldAnchor, human, oldRotate

local gui = Instance.new("ScreenGui")
gui.Name = "GhostTP"
gui.ResetOnSpawn = false
gui.Parent = player:WaitForChild("PlayerGui")

local function box(class, parent, text, x, y, w, h)
	local obj = Instance.new(class)
	obj.Position = UDim2.fromOffset(x, y)
	obj.Size = UDim2.fromOffset(w, h)
	obj.BackgroundColor3 = Color3.fromRGB(12, 12, 16)
	obj.Parent = parent
	if obj:IsA("TextButton") then
		obj.Text = text
		obj.TextColor3 = purple
		obj.Font = Enum.Font.GothamBold
		obj.TextSize = 13
	end
	local corner = Instance.new("UICorner", obj)
	corner.CornerRadius = UDim.new(0, 14)
	local stroke = Instance.new("UIStroke", obj)
	stroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
	stroke.Color = purple
	return obj
end

local panel = box("Frame", gui, "", 90, 100, 290, 290)
panel.Visible = false
local title = box("TextButton", panel, "Ghost TP", 10, 8, 225, 35)
local close = box("TextButton", panel, "X", 245, 8, 35, 35)
local save = box("TextButton", panel, "Salvar posição", 10, 55, 130, 40)
local tp = box("TextButton", panel, "Teletransportar", 150, 55, 130, 40)
local bug = box("TextButton", panel, "Bug TP (30x)", 10, 105, 130, 40)
local fly = box("TextButton", panel, "Ir pelo trajeto", 150, 105, 130, 40)
local clip = box("TextButton", panel, "Noclip: OFF", 10, 155, 270, 35)
local cancel = box("TextButton", panel, "Cancelar movimento", 10, 200, 270, 35)
local status = box("TextButton", panel, "Salve uma posição.", 10, 245, 270, 35)
status.TextWrapped = true
status.AutoButtonColor = false
local icon = box("TextButton", gui, "Ghost", 20, 180, 64, 64)
icon.UICorner.CornerRadius = UDim.new(1, 0)

-- Arraste por mouse/toque. Arrastar não conta como clique.
local function drag(handle, target, click)
	local input, start, origin, moved
	handle.InputBegan:Connect(function(i)
		if input then return end
		if i.UserInputType ~= Enum.UserInputType.Touch
			and i.UserInputType ~= Enum.UserInputType.MouseButton1 then return end
		input, start = i, Vector2.new(i.Position.X, i.Position.Y)
		origin = target.AbsolutePosition - gui.AbsolutePosition
		moved = false
	end)
	UIS.InputChanged:Connect(function(i)
		if not input then return end
		if input.UserInputType == Enum.UserInputType.Touch then
			if i ~= input then return end
		elseif i.UserInputType ~= Enum.UserInputType.MouseMovement then return end
		local delta = Vector2.new(i.Position.X, i.Position.Y) - start
		moved = moved or delta.Magnitude > 7
		if moved then
			local p, area = origin + delta, gui.AbsoluteSize - target.AbsoluteSize
			target.Position = UDim2.fromOffset(
				math.clamp(p.X, 0, math.max(0, area.X)),
				math.clamp(p.Y, 0, math.max(0, area.Y)))
		end
	end)
	UIS.InputEnded:Connect(function(i)
		if i ~= input then return end
		input = nil
		local delta = Vector2.new(i.Position.X, i.Position.Y) - start
		if not moved and delta.Magnitude <= 7 and click then click() end
	end)
end
drag(title, panel)
drag(icon, icon, function()
	icon.Visible, panel.Visible = false, true
end)
close.Activated:Connect(function()
	icon.Visible, panel.Visible = true, false
end)

local function character()
	local c = player.Character
	local r = c and c:FindFirstChild("HumanoidRootPart")
	local h = c and c:FindFirstChildOfClass("Humanoid")
	if r and h and h.Health > 0 then return r, h end
end
local function restore()
	for part, value in pairs(originals) do
		if part.Parent then part.CanCollide = value end
	end
	table.clear(originals)
end
local function stop()
	generation += 1
	if activeTween then activeTween:Cancel(); activeTween = nil end
	if locked and locked.Parent then
		locked.AssemblyLinearVelocity = Vector3.zero
		locked.AssemblyAngularVelocity = Vector3.zero
		locked.Anchored = oldAnchor
	end
	if human and human.Parent then human.AutoRotate = oldRotate end
	locked, human, busy = nil, nil, false
	if not noclip then restore() end
end
local function validate()
	if busy then status.Text = "Cancele o movimento atual."; return end
	if not saved then status.Text = "Salve uma posição primeiro."; return end
	local r, h = character()
	if not r then status.Text = "Aguarde seu personagem."; return end
	if h.SeatPart then status.Text = "Saia do assento."; return end
	return r, h
end

save.Activated:Connect(function()
	if busy then return end
	local r = character()
	if not r then return end
	saved = r.CFrame
	if beam then beam:Destroy(); beam = nil end
	if link then link:Destroy(); link = nil end
	if ball then ball:Destroy() end
	ball = Instance.new("Part")
	ball.Name = "GhostMarker"
	ball.Shape = Enum.PartType.Ball
	ball.Size = Vector3.new(1.3, 1.3, 1.3)
	ball.Material = Enum.Material.Neon
	ball.Color = purple
	ball.Anchored = true
	ball.CanCollide, ball.CanTouch, ball.CanQuery = false, false, false
	ball.Position = saved.Position
	Instance.new("Attachment", ball)
	ball.Parent = workspace
	status.Text = "Posição salva!"
end)

tp.Activated:Connect(function()
	local r = validate()
	if not r then return end
	r.AssemblyLinearVelocity = Vector3.zero
	r.AssemblyAngularVelocity = Vector3.zero
	r.CFrame = saved
	status.Text = "Teletransportado!"
end)

local function move(flight)
	local r, h = validate()
	if not r then return end
	busy = true
	generation += 1
	local id, destination = generation, saved
	locked, human = r, h
	oldAnchor, oldRotate = r.Anchored, h.AutoRotate
	r.Anchored, h.AutoRotate = true, false

	local function valid()
		return generation == id and character() == r
	end
	task.spawn(function()
		if flight then
			status.Text = "Voando até a posição salva..."
			local altitude = math.max(r.Position.Y, destination.Y) + 60
			local rotation = r.CFrame.Rotation
			local points = {
				CFrame.new(r.Position.X, altitude, r.Position.Z) * rotation,
				CFrame.new(destination.X, altitude, destination.Z) * rotation,
				destination
			}
			for _, point in ipairs(points) do
				if not valid() then break end
				local duration = math.max((r.Position - point.Position).Magnitude / 65, 0.1)
				activeTween = TweenService:Create(r,
					TweenInfo.new(duration, Enum.EasingStyle.Linear),
					{CFrame = point})
				activeTween:Play()
				activeTween.Completed:Wait()
			end
		else
			for count = 1, 30 do
				if not valid() then break end
				r.CFrame = destination
				status.Text = "Bug TP: " .. count .. "/30"
				task.wait(0.05)
			end
		end
		if generation == id then
			local completed = valid()
			stop()
			status.Text = completed and "Movimento concluído!" or "Interrompido."
		end
	end)
end
bug.Activated:Connect(function() move(false) end)
fly.Activated:Connect(function() move(true) end)
cancel.Activated:Connect(function()
	stop()
	status.Text = "Cancelado. Personagem liberado."
end)
clip.Activated:Connect(function()
	noclip = not noclip
	clip.Text = noclip and "Noclip: ON" or "Noclip: OFF"
	if not noclip and not busy then restore() end
	status.Text = noclip and "Cuidado: atravessa também o chão!" or "Noclip desligado."
end)

-- Colisão desativada durante o movimento e no noclip.
RunService.PreSimulation:Connect(function()
	if not noclip and not busy then return end
	local c = player.Character
	if not c then return end
	for _, part in ipairs(c:GetDescendants()) do
		if part:IsA("BasePart") then
			if originals[part] == nil then originals[part] = part.CanCollide end
			part.CanCollide = false
		end
	end
end)

-- A linha acompanha o personagem, inclusive após renascer.
RunService.Heartbeat:Connect(function()
	local r = character()
	if busy and r ~= locked then stop() end
	if not ball or not r then return end
	if link and link.Parent == r then return end
	if beam then beam:Destroy() end
	if link then link:Destroy() end
	link = Instance.new("Attachment", r)
	beam = Instance.new("Beam")
	beam.Attachment0, beam.Attachment1 = link, ball.Attachment
	beam.Color = ColorSequence.new(purple)
	beam.Width0, beam.Width1 = 0.12, 0.12
	beam.FaceCamera = true
	beam.LightEmission = 1
	beam.Parent = ball
end)
player.CharacterRemoving:Connect(function()
	noclip = false
	clip.Text = "Noclip: OFF"
	stop()
end)
