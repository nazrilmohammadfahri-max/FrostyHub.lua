local Rayfield = loadstring(game:HttpGet('https://sirius.menu/rayfield'))()

local Window = Rayfield:CreateWindow({
   Name = "Frosty HUB", 
   Icon = 0, 
   LoadingTitle = "Universal script", 
   LoadingSubtitle = "Frosty is hot", 
   ShowText = "Rayfield", 
   Theme = "Default", 

   ToggleUIKeybind = "K", 
   DisableRayfieldPrompts = false,
   DisableBuildWarnings = false, 

   ConfigurationSaving = {
      Enabled = true,
      FolderName = nil, 
      FileName = "Big Hub"
   },

   Discord = {
      Enabled = false,
      Invite = "noinvitelink", 
      RememberJoins = true 
   },

   KeySystem = false, 
   KeySettings = {
      Title = "Untitled",
      Subtitle = "Key System",
      Note = "No method of obtaining the key is provided", 
      FileName = "Key", 
      SaveKey = true, 
      GrabKeyFromSite = false, 
      Key = {"Hello"} 
   }
})

-- Tab renamed to "QH"
local Tab = Window:CreateTab("QH", 4483362458)

-- Section: Speed
local SpeedSection = Tab:CreateSection("Speed")
local SpeedSlider = Tab:CreateSlider({
   Name = "WalkSpeed",
   Range = {16, 200},
   Increment = 1,
   Suffix = "Speed",
   CurrentValue = 16,
   Flag = "WalkSpeed", 
   Callback = function(Value)
      game.Players.LocalPlayer.Character.Humanoid.WalkSpeed = Value
   end,
})

-- Section: Jump Power
local JumpSection = Tab:CreateSection("Jump Power")
local JumpSlider = Tab:CreateSlider({
   Name = "JumpPower",
   Range = {50, 300},
   Increment = 5,
   Suffix = "Power",
   CurrentValue = 50,
   Flag = "JumpPower", 
   Callback = function(Value)
      game.Players.LocalPlayer.Character.Humanoid.JumpPower = Value
   end,
})

-- Section: Fire Rate
local FireRateSection = Tab:CreateSection("Fire Rate")

_G.FireRateMultiplier = 1 -- Default

local FireRateSlider = Tab:CreateSlider({
   Name = "Fire Rate Multiplier",
   Range = {1, 10}, -- 1x normal, up to 10x faster
   Increment = 1,
   Suffix = "x",
   CurrentValue = 1,
   Flag = "FireRate", 
   Callback = function(Value)
      _G.FireRateMultiplier = Value
   end,
})

-- Function to patch weapons
local function patchTool(Tool)
   if Tool:IsA("Tool") then
      -- Look for local scripts inside the tool
      for _, Obj in pairs(Tool:GetDescendants()) do
         if Obj:IsA("Script") or Obj:IsA("LocalScript") then
            -- Try replacing wait() calls with scaled time
            local old; old = hookfunction(wait, function(t)
               if _G.FireRateMultiplier > 1 then
                  return old(t / _G.FireRateMultiplier)
               end
               return old(t)
            end)
         end
      end

      -- Some tools use a "Cooldown" NumberValue or Attribute
      if Tool:FindFirstChild("Cooldown") then
         Tool.Cooldown.Changed:Connect(function()
            if _G.FireRateMultiplier > 1 then
               Tool.Cooldown.Value = Tool.Cooldown.Value / _G.FireRateMultiplier
            end
         end)
         Tool.Cooldown.Value = Tool.Cooldown.Value / _G.FireRateMultiplier
      end
   end
end

-- Patch existing tools
for _, Tool in pairs(game.Players.LocalPlayer.Backpack:GetChildren()) do
   patchTool(Tool)
end
for _, Tool in pairs(game.Players.LocalPlayer.Character:GetChildren()) do
   patchTool(Tool)
end

-- Patch new tools
game.Players.LocalPlayer.Backpack.ChildAdded:Connect(patchTool)
game.Players.LocalPlayer.Character.ChildAdded:Connect(patchTool)
-- Section: Damage Modifier
local DamageSection = Tab:CreateSection("Damage Modifier")

_G.DamageMultiplier = 1 -- Default

local DamageSlider = Tab:CreateSlider({
   Name = "Damage Multiplier",
   Range = {1, 1000}, -- 1x to 1000x damage
   Increment = 1,
   Suffix = "x",
   CurrentValue = 1,
   Flag = "DamageMultiplier",
   Callback = function(Value)
      _G.DamageMultiplier = Value
   end,
})

-- Section: No CD
local NoCDSection = Tab:CreateSection("No CD")

_G.NoCooldown = false -- Default state

local function patchNoCDTool(Tool)
   if Tool:IsA("Tool") and Tool:FindFirstChild("Handle") then
      -- Patch reload instantly if there's a reload value
      if Tool:FindFirstChild("ReloadTime") then
         Tool.ReloadTime.Changed:Connect(function()
            if _G.NoCooldown then
               Tool.ReloadTime.Value = 0 -- instant reload
            end
         end)
         Tool.ReloadTime.Value = 0
      end

      -- Patch cooldown instantly
      if Tool:FindFirstChild("Cooldown") then
         Tool.Cooldown.Changed:Connect(function()
            if _G.NoCooldown then
               Tool.Cooldown.Value = 0 -- instant cooldown
            end
         end)
         Tool.Cooldown.Value = 0
      end

      -- Hook into Activated event to reset reloads/cooldowns on use
      Tool.Activated:Connect(function()
         if _G.NoCooldown then
            if Tool:FindFirstChild("Cooldown") then
               Tool.Cooldown.Value = 0
            end
            if Tool:FindFirstChild("ReloadTime") then
               Tool.ReloadTime.Value = 0
            end
         end
      end)
   end
end

local NoCDBtn = Tab:CreateToggle({
   Name = "No Cooldown / Instant Reload",
   CurrentValue = false,
   Flag = "NoCooldown",
   Callback = function(Value)
      _G.NoCooldown = Value

      -- Apply to all current tools
      for _, Tool in pairs(game.Players.LocalPlayer.Backpack:GetChildren()) do
         patchNoCDTool(Tool)
      end
      for _, Tool in pairs(game.Players.LocalPlayer.Character:GetChildren()) do
         patchNoCDTool(Tool)
      end

      -- Auto apply to new tools
      game.Players.LocalPlayer.Backpack.ChildAdded:Connect(patchNoCDTool)
      game.Players.LocalPlayer.Character.ChildAdded:Connect(patchNoCDTool)
   end,
})

-- Section: Hitbox
local HitboxSection = Tab:CreateSection("Hitbox")

_G.HitboxSize = 2
_G.HitboxEnabled = false
_G.HitboxHighlight = true
_G.TeamCheck = false

-- Function to apply hitbox settings
local function applyHitbox(Player)
   if Player ~= game.Players.LocalPlayer then
      local function setupChar(Char)
         local hrp = Char:WaitForChild("HumanoidRootPart")

         if _G.HitboxEnabled then
            -- Team check
            if _G.TeamCheck and Player.Team == game.Players.LocalPlayer.Team then
               hrp.Size = Vector3.new(2, 2, 1) -- reset normal size
               if hrp:FindFirstChild("HitboxHighlight") then
                  hrp.HitboxHighlight:Destroy()
               end
               return
            end

            -- Resize hitbox
            hrp.Size = Vector3.new(_G.HitboxSize, _G.HitboxSize, _G.HitboxSize)
            hrp.CanCollide = false

            -- Highlight toggle
            if _G.HitboxHighlight then
               if not hrp:FindFirstChild("HitboxHighlight") then
                  local highlight = Instance.new("BoxHandleAdornment")
                  highlight.Name = "HitboxHighlight"
                  highlight.Adornee = hrp
                  highlight.AlwaysOnTop = true
                  highlight.ZIndex = 10
                  highlight.Size = hrp.Size
                  highlight.Color3 = Color3.fromRGB(255, 0, 0) -- Red
                  highlight.Transparency = 0.5
                  highlight.Parent = hrp
               else
                  hrp.HitboxHighlight.Size = hrp.Size
               end
            else
               if hrp:FindFirstChild("HitboxHighlight") then
                  hrp.HitboxHighlight:Destroy()
               end
            end
         else
            -- Reset if disabled
            hrp.Size = Vector3.new(2, 2, 1)
            if hrp:FindFirstChild("HitboxHighlight") then
               hrp.HitboxHighlight:Destroy()
            end
         end
      end

      -- Existing char
      if Player.Character then
         setupChar(Player.Character)
      end

      -- New char
      Player.CharacterAdded:Connect(setupChar)
   end
end

-- Toggle: Enable Hitbox
local HitboxToggle = Tab:CreateToggle({
   Name = "Enable Hitbox",
   CurrentValue = false,
   Flag = "HitboxToggle",
   Callback = function(Value)
      _G.HitboxEnabled = Value
      for _, Player in pairs(game.Players:GetPlayers()) do
         applyHitbox(Player)
      end
   end,
})

-- Slider: Hitbox Size
local HitboxSlider = Tab:CreateSlider({
   Name = "Hitbox Size",
   Range = {2, 50},
   Increment = 1,
   Suffix = "Size",
   CurrentValue = 2,
   Flag = "HitboxSize",
   Callback = function(Value)
      _G.HitboxSize = Value
      for _, Player in pairs(game.Players:GetPlayers()) do
         applyHitbox(Player)
      end
   end,
})

-- Toggle: Hitbox Highlight
local HighlightToggle = Tab:CreateToggle({
   Name = "Hitbox Highlight",
   CurrentValue = true,
   Flag = "HitboxHighlight",
   Callback = function(Value)
      _G.HitboxHighlight = Value
      for _, Player in pairs(game.Players:GetPlayers()) do
         applyHitbox(Player)
      end
   end,
})

-- Toggle: Team Check
local TeamCheckToggle = Tab:CreateToggle({
   Name = "Team Check",
   CurrentValue = false,
   Flag = "TeamCheck",
   Callback = function(Value)
      _G.TeamCheck = Value
      for _, Player in pairs(game.Players:GetPlayers()) do
         applyHitbox(Player)
      end
   end,
})

-- Section: Silent Aim
local SilentAimSection = Tab:CreateSection("Silent Aim")

_G.SilentAim = false
_G.SilentAimTargetPart = "HumanoidRootPart"
_G.SilentAimFOV = 150
_G.SilentAimHitChance = 100 -- percent
_G.ShowFOV = true -- toggle for circle

-- Dropdown: Choose Target Part
local TargetPartDropdown = Tab:CreateDropdown({
   Name = "Target Part",
   Options = {"HumanoidRootPart", "Head", "Torso"},
   CurrentOption = "HumanoidRootPart",
   Flag = "SilentAimPart",
   Callback = function(Value)
      _G.SilentAimTargetPart = Value
   end,
})

-- Toggle: Enable Silent Aim
local SilentAimToggle = Tab:CreateToggle({
   Name = "Enable Silent Aim",
   CurrentValue = false,
   Flag = "SilentAimToggle",
   Callback = function(Value)
      _G.SilentAim = Value
   end,
})

-- Slider: FOV Size (updated 1–360)
local SilentAimFOVSlider = Tab:CreateSlider({
   Name = "Silent Aim FOV",
   Range = {1, 360}, -- fixed
   Increment = 1,
   Suffix = "°",
   CurrentValue = 150,
   Flag = "SilentAimFOV",
   Callback = function(Value)
      _G.SilentAimFOV = Value
   end,
})

-- Slider: Hit Chance
local HitChanceSlider = Tab:CreateSlider({
   Name = "Hit Chance",
   Range = {1, 100},
   Increment = 1,
   Suffix = "%",
   CurrentValue = 100,
   Flag = "HitChance",
   Callback = function(Value)
      _G.SilentAimHitChance = Value
   end,
})

-- Toggle: Show FOV Circle
local ShowFOVToggle = Tab:CreateToggle({
   Name = "Show FOV Circle",
   CurrentValue = true,
   Flag = "ShowFOV",
   Callback = function(Value)
      _G.ShowFOV = Value
   end,
})

-- FOV Circle (Drawing API)
local circle = Drawing.new("Circle")
circle.Color = Color3.fromRGB(255, 0, 0)
circle.Thickness = 1
circle.NumSides = 64
circle.Radius = _G.SilentAimFOV
circle.Filled = false
circle.Transparency = 0.8
circle.Visible = false

game:GetService("RunService").RenderStepped:Connect(function()
   circle.Visible = _G.SilentAim and _G.ShowFOV
   circle.Radius = _G.SilentAimFOV
   circle.Position = game:GetService("UserInputService"):GetMouseLocation()
end)

-- Auto apply to new players
game.Players.PlayerAdded:Connect(function(Player)
   applyHitbox(Player)
end)

-- Section: Infinite Ammo
local AmmoSection = Tab:CreateSection("Infinite Ammo")

_G.InfiniteAmmo = false -- default state

local function patchInfiniteAmmo(Tool)
   if Tool:IsA("Tool") then
      -- Look for values commonly used to store ammo
      local function zeroOut(obj)
         if obj:IsA("IntValue") or obj:IsA("NumberValue") then
            if obj.Name:lower():find("ammo") or obj.Name:lower():find("clip") or obj.Name:lower():find("mag") then
               obj.Changed:Connect(function()
                  if _G.InfiniteAmmo then
                     obj.Value = math.huge -- never run out
                  end
               end)
               obj.Value = math.huge
            end
         end
      end

      for _, v in pairs(Tool:GetDescendants()) do
         zeroOut(v)
      end

      Tool.DescendantAdded:Connect(zeroOut)
   end
end

local InfiniteAmmoToggle = Tab:CreateToggle({
   Name = "Enable Infinite Ammo",
   CurrentValue = false,
   Flag = "InfiniteAmmo",
   Callback = function(Value)
      _G.InfiniteAmmo = Value

      -- Patch all current tools
      for _, Tool in pairs(game.Players.LocalPlayer.Backpack:GetChildren()) do
         patchInfiniteAmmo(Tool)
      end
      for _, Tool in pairs(game.Players.LocalPlayer.Character:GetChildren()) do
         patchInfiniteAmmo(Tool)
      end

      -- Patch new tools
      game.Players.LocalPlayer.Backpack.ChildAdded:Connect(patchInfiniteAmmo)
      game.Players.LocalPlayer.Character.ChildAdded:Connect(patchInfiniteAmmo)
   end,
})

-- Section: ESP
local ESPSection = Tab:CreateSection("ESP")

_G.ESPEnabled = false
_G.ESPTeamCheck = false
_G.ESPShowHealth = true
_G.ESPShowName = true

local function createESP(Player)
    if Player ~= game.Players.LocalPlayer then
        local function setupChar(Char)
            -- Highlight
            local highlight = Instance.new("Highlight")
            highlight.Name = "ESPHighlight"
            highlight.FillColor = Color3.fromRGB(255, 0, 0)
            highlight.FillTransparency = 0.5
            highlight.OutlineTransparency = 0
            highlight.Parent = Char

            -- Billboard for Name + Health
            local billboard = Instance.new("BillboardGui")
            billboard.Name = "ESPBillboard"
            billboard.Adornee = Char:WaitForChild("Head")
            billboard.Size = UDim2.new(0, 200, 0, 50)
            billboard.AlwaysOnTop = true
            billboard.Parent = Char

            local textLabel = Instance.new("TextLabel")
            textLabel.Size = UDim2.new(1, 0, 1, 0)
            textLabel.BackgroundTransparency = 1
            textLabel.TextColor3 = Color3.fromRGB(255, 0, 0)
            textLabel.Font = Enum.Font.SourceSansBold
            textLabel.TextSize = 16
            textLabel.TextStrokeTransparency = 0.5
            textLabel.Parent = billboard

            -- Update loop
            game:GetService("RunService").RenderStepped:Connect(function()
                if Char:FindFirstChild("Humanoid") then
                    local humanoid = Char:FindFirstChild("Humanoid")

                    -- Team check
                    if _G.ESPTeamCheck and Player.Team == game.Players.LocalPlayer.Team then
                        highlight.Enabled = false
                        billboard.Enabled = false
                        return
                    end

                    -- Visibility
                    highlight.Enabled = _G.ESPEnabled
                    billboard.Enabled = _G.ESPEnabled

                    -- Update text
                    local nameText = _G.ESPShowName and Player.Name or ""
                    local healthText = _G.ESPShowHealth and (" [" .. math.floor(humanoid.Health) .. "/" .. math.floor(humanoid.MaxHealth) .. "]") or ""
                    textLabel.Text = nameText .. healthText
                end
            end)
        end

        if Player.Character then
            setupChar(Player.Character)
        end

        Player.CharacterAdded:Connect(setupChar)
    end
end

for _, Player in pairs(game.Players:GetPlayers()) do
    createESP(Player)
end
game.Players.PlayerAdded:Connect(createESP)

-- Toggles
local ESPToggle = Tab:CreateToggle({
   Name = "Enable ESP",
   CurrentValue = false,
   Flag = "ESP",
   Callback = function(Value)
      _G.ESPEnabled = Value
   end,
})

local ESPTeamCheckToggle = Tab:CreateToggle({
   Name = "ESP Team Check",
   CurrentValue = false,
   Flag = "ESPTeamCheck",
   Callback
 
-- Crosshair Section (Upgraded)
local CrosshairSection = Tab:CreateSection("Crosshair")

_G.CrosshairEnabled = false
_G.CrosshairColor = Color3.fromRGB(255, 255, 255)
_G.CrosshairStyle = "Classic"
_G.CrosshairSize = 10
_G.CrosshairThickness = 2

-- Drawing crosshair
local CrosshairLines = {
    Left = Drawing.new("Line"),
    Right = Drawing.new("Line"),
    Up = Drawing.new("Line"),
    Down = Drawing.new("Line"),
    Dot = Drawing.new("Circle")
}

-- Setup defaults
for _, line in pairs(CrosshairLines) do
    line.Color = _G.CrosshairColor
    line.Visible = false
end
CrosshairLines.Dot.Radius = 2
CrosshairLines.Dot.Filled = true

-- Toggle
Tab:CreateToggle({
    Name = "Enable Crosshair",
    CurrentValue = false,
    Flag = "CrosshairToggle",
    Callback = function(Value)
        _G.CrosshairEnabled = Value
        for _, line in pairs(CrosshairLines) do
            line.Visible = Value
        end
    end,
})

-- Color Picker
Tab:CreateColorPicker({
    Name = "Crosshair Color",
    Color = Color3.fromRGB(255, 255, 255),
    Flag = "CrosshairColor",
    Callback = function(Value)
        _G.CrosshairColor = Value
        for _, line in pairs(CrosshairLines) do
            line.Color = Value
        end
    end,
})

-- Dropdown for style
Tab:CreateDropdown({
    Name = "Crosshair Style",
    Options = {"Classic", "Dot", "T-Style"},
    CurrentOption = "Classic",
    Flag = "CrosshairStyle",
    Callback = function(Value)
        _G.CrosshairStyle = Value
    end,
})

-- Slider for size
Tab:CreateSlider({
    Name = "Crosshair Size",
    Range = {2, 50},
    Increment = 1,
    Suffix = "px",
    CurrentValue = 10,
    Flag = "CrosshairSize",
    Callback = function(Value)
        _G.CrosshairSize = Value
    end,
})

-- Slider for thickness
Tab:CreateSlider({
    Name = "Crosshair Thickness",
    Range = {1, 10},
    Increment = 1,
    Suffix = "px",
    CurrentValue = 2,
    Flag = "CrosshairThickness",
    Callback = function(Value)
        _G.CrosshairThickness = Value
        for _, line in pairs(CrosshairLines) do
            if line.ClassName == "Line" then
                line.Thickness = Value
            elseif line.ClassName == "Circle" then
                line.Thickness = Value
            end
        end
    end,
})

-- Render loop
game:GetService("RunService").RenderStepped:Connect(function()
    if _G.CrosshairEnabled then
        local mousePos = game:GetService("UserInputService"):GetMouseLocation()
        local x, y = mousePos.X, mousePos.Y
        local size = _G.CrosshairSize
        local thickness = _G.CrosshairThickness

        -- Hide all first
        for _, line in pairs(CrosshairLines) do
            line.Visible = false
            line.Thickness = thickness
        end

        if _G.CrosshairStyle == "Classic" then
            CrosshairLines.Left.Visible = true
            CrosshairLines.Right.Visible = true
            CrosshairLines.Up.Visible = true
            CrosshairLines.Down.Visible = true

            CrosshairLines.Left.From = Vector2.new(x - size, y)
            CrosshairLines.Left.To = Vector2.new(x - 2, y)

            CrosshairLines.Right.From = Vector2.new(x + 2, y)
            CrosshairLines.Right.To = Vector2.new(x + size, y)

            CrosshairLines.Up.From = Vector2.new(x, y - size)
            CrosshairLines.Up.To = Vector2.new(x, y - 2)

            CrosshairLines.Down.From = Vector2.new(x, y + 2)
            CrosshairLines.Down.To = Vector2.new(x, y + size)

        elseif _G.CrosshairStyle == "Dot" then
            CrosshairLines.Dot.Visible = true
            CrosshairLines.Dot.Position = Vector2.new(x, y)
            CrosshairLines.Dot.Radius = math.floor(size / 4)

        elseif _G.CrosshairStyle == "T-Style" then
            CrosshairLines.Left.Visible = true
            CrosshairLines.Right.Visible = true
            CrosshairLines.Down.Visible = true

            CrosshairLines.Left.From = Vector2.new(x - size, y)
            CrosshairLines.Left.To = Vector2.new(x - 2, y)

            CrosshairLines.Right.From = Vector2.new(x + 2, y)
            CrosshairLines.Right.To = Vector2.new(x + size, y)

            CrosshairLines.Down.From = Vector2.new(x, y + 2)
            CrosshairLines.Down.To = Vector2.new(x, y + size)
        end
    end
end)

-- Silent Aim style (forces bullets to crosshair)
local Players = game:GetService("Players")
local Camera = game:GetService("Workspace").CurrentCamera
local LocalPlayer = Players.LocalPlayer

-- Hook for raycast weapons
local old; old = hookmetamethod(game, "__namecall", function(self, ...)
    local args = {...}
    local method = getnamecallmethod()

    if method == "Raycast" or method == "FindPartOnRay" then
        if _G.CrosshairEnabled then
            local mousePos = game:GetService("UserInputService"):GetMouseLocation()
            local targetPos = Camera:ViewportPointToRay(mousePos.X, mousePos.Y).Direction * 1000
            args[2] = targetPos -- force direction
            return old(self, unpack(args))
        end
    end
    
    return old(self, ...)
end)

-- New Tab: Hacks 2
local Hacks2Tab = Window:CreateTab("Hacks 2", 4483362458) -- you can change the icon ID

-- Section: Aimbot
local AimbotSection = Hacks2Tab:CreateSection("Aimbot")

_G.AimbotEnabled = false
_G.AimbotTeamCheck = false

local AimbotToggle = Hacks2Tab:CreateToggle({
    Name = "Enable Aimbot",
    CurrentValue = false,
    Flag = "AimbotToggle",
    Callback = function(Value)
        _G.AimbotEnabled = Value
    end,
})

local AimbotTeamCheckToggle = Hacks2Tab:CreateToggle({
    Name = "Aimbot Team Check",
    CurrentValue = false,
    Flag = "AimbotTeamCheck",
    Callback = function(Value)
        _G.AimbotTeamCheck = Value
    end,
})

-- Section: Fly
local FlySection = Hacks2Tab:CreateSection("Fly")

_G.FlyEnabled = false
_G.FlySpeed = 50

local FlyToggle = Hacks2Tab:CreateToggle({
    Name = "Enable Fly",
    CurrentValue = false,
    Flag = "FlyToggle",
    Callback = function(Value)
        _G.FlyEnabled = Value
        if Value then
            local Humanoid = game.Players.LocalPlayer.Character:FindFirstChildOfClass("Humanoid")
            local Root = game.Players.LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
            local UIS = game:GetService("UserInputService")
            local RunService = game:GetService("RunService")

            task.spawn(function()
                while _G.FlyEnabled and task.wait() do
                    local moveDirection = Vector3.zero
                    if UIS:IsKeyDown(Enum.KeyCode.W) then moveDirection = moveDirection + Root.CFrame.LookVector end
                    if UIS:IsKeyDown(Enum.KeyCode.S) then moveDirection = moveDirection - Root.CFrame.LookVector end
                    if UIS:IsKeyDown(Enum.KeyCode.A) then moveDirection = moveDirection - Root.CFrame.RightVector end
                    if UIS:IsKeyDown(Enum.KeyCode.D) then moveDirection = moveDirection + Root.CFrame.RightVector end
                    if UIS:IsKeyDown(Enum.KeyCode.Space) then moveDirection = moveDirection + Vector3.new(0,1,0) end
                    if UIS:IsKeyDown(Enum.KeyCode.LeftShift) then moveDirection = moveDirection - Vector3.new(0,1,0) end
                    Root.Velocity = moveDirection * _G.FlySpeed
                end
                if Root then Root.Velocity = Vector3.zero end
            end)
        end
    end,
})

local FlySpeedSlider = Hacks2Tab:CreateSlider({
    Name = "Fly Speed",
    Range = {1, 200},
    Increment = 1,
    Suffix = "Speed",
    CurrentValue = 50,
    Flag = "FlySpeed",
    Callback = function(Value)
        _G.FlySpeed = Value
    end,
})

-- Section: Spam Chat
local SpamSection = Hacks2Tab:CreateSection("Spam Chat")

_G.SpamChatEnabled = false
_G.SpamMessage = "Hello!"

local SpamToggle = Hacks2Tab:CreateToggle({
    Name = "Enable Spam Chat",
    CurrentValue = false,
    Flag = "SpamChatToggle",
    Callback = function(Value)
        _G.SpamChatEnabled = Value
        if Value then
            task.spawn(function()
                while _G.SpamChatEnabled do
                    game.ReplicatedStorage.DefaultChatSystemChatEvents.SayMessageRequest:FireServer(_G.SpamMessage, "All")
                    task.wait(1) -- interval
                end
            end)
        end
    end,
})

local SpamTextbox = Hacks2Tab:CreateInput({
    Name = "Spam Message",
    PlaceholderText = "Enter message...",
    RemoveTextAfterFocusLost = false,
    Callback = function(Text)
        _G.SpamMessage = Text
    end,
})

-- Section: Animation Speed
local AnimSection = Hacks2Tab:CreateSection("Animation Speed")

_G.AnimSpeed = 1

local AnimSlider = Hacks2Tab:CreateSlider({
    Name = "Animation Speed",
    Range = {1, 200},
    Increment = 1,
    Suffix = "x",
    CurrentValue = 1,
    Flag = "AnimSpeed",
    Callback = function(Value)
        _G.AnimSpeed = Value
        local Humanoid = game.Players.LocalPlayer.Character:FindFirstChildOfClass("Humanoid")
        if Humanoid then
            Humanoid.Animator:WaitForChild("Animator")
            for _, track in pairs(Humanoid.Animator:GetPlayingAnimationTracks()) do
                track:AdjustSpeed(Value)
            end
        end
    end,
})

-- Section: Orbit Player
local OrbitSection = Hacks2Tab:CreateSection("Orbit Player")

_G.OrbitEnabled = false
_G.OrbitTarget = nil

local OrbitDropdown = Hacks2Tab:CreateDropdown({
    Name = "Select Player to Orbit",
    Options = GetPlayerList(),
    CurrentOption = "",
    Flag = "OrbitTarget",
    Callback = function(Value)
        _G.OrbitTarget = Value
    end,
})

-- Auto refresh Orbit dropdown
game.Players.PlayerAdded:Connect(function()
    OrbitDropdown:SetOptions(GetPlayerList())
end)
game.Players.PlayerRemoving:Connect(function()
    OrbitDropdown:SetOptions(GetPlayerList())
end)

local OrbitToggle = Hacks2Tab:CreateToggle({
    Name = "Enable Orbit",
    CurrentValue = false,
    Flag = "OrbitToggle",
    Callback = function(Value)
        _G.OrbitEnabled = Value
        task.spawn(function()
            while _G.OrbitEnabled and _G.OrbitTarget and task.wait() do
                local Target = game.Players:FindFirstChild(_G.OrbitTarget)
                if Target and Target.Character and Target.Character:FindFirstChild("HumanoidRootPart") then
                    local Root = game.Players.LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
                    if Root then
                        local angle = tick() * 2
                        local radius = 10
                        local offset = Vector3.new(math.cos(angle) * radius, 0, math.sin(angle) * radius)
                        Root.CFrame = CFrame.new(Target.Character.HumanoidRootPart.Position + offset)
                    end
                end
            end
        end)
    end,
})

-- Section: TP Behind Player
local TPSection = Hacks2Tab:CreateSection("TP Behind Player")

_G.TPTarget = nil

local TPDropdown = Hacks2Tab:CreateDropdown({
    Name = "Select Player to TP Behind",
    Options = GetPlayerList(),
    CurrentOption = "",
    Flag = "TPTarget",
    Callback = function(Value)
        _G.TPTarget = Value
    end,
})

-- Auto refresh TP Behind dropdown
game.Players.PlayerAdded:Connect(function()
    TPDropdown:SetOptions(GetPlayerList())
end)
game.Players.PlayerRemoving:Connect(function()
    TPDropdown:SetOptions(GetPlayerList())
end)

local TPButton = Hacks2Tab:CreateButton({
    Name = "Teleport Behind Player",
    Callback = function()
        if _G.TPTarget then
            local Target = game.Players:FindFirstChild(_G.TPTarget)
            if Target and Target.Character and Target.Character:FindFirstChild("HumanoidRootPart") then
                local Root = game.Players.LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
                if Root then
                    local behind = Target.Character.HumanoidRootPart.CFrame.LookVector * -3
                    Root.CFrame = Target.Character.HumanoidRootPart.CFrame + behind
                end
            end
        end
    end,
})

-- Divider
local Divider = Tab:CreateDivider()
Divider:Set(false)

Rayfield:SetVisibility(false)
Rayfield:IsVisible()
