--!strict

-- This script creates a smooth camera following effect, with an added aimlock feature.
-- When the aimlock key is pressed, the camera will attempt to lock onto the closest
-- other player's head.
-- It's designed to be placed in StarterPlayerScripts as a LocalScript.

-- Services
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Workspace = game:GetService("Workspace")

-- Configuration
local SMOOTHNESS_FACTOR = 0.15 -- How smoothly the camera interpolates (0-1, lower is smoother)
local CAMERA_OFFSET = Vector3.new(0, 5, 15) -- Offset from the target character (X, Y, Z) when NOT aimlocking
local AIMLOCK_CAMERA_OFFSET = Vector3.new(0, 2, 8) -- Offset from the target's head when aimlocking
local MAX_RAYCAST_DISTANCE = 20 -- Max distance for raycasting to check for obstacles
local MIN_CAMERA_DISTANCE = 2 -- Minimum distance the camera can be from the target
local CAMERA_FIELD_OF_VIEW = 70 -- Standard field of view for the camera

local AIMLOCK_KEY = Enum.KeyCode.E -- The key to toggle aimlock (e.g., E, Q, LeftControl)
local MAX_AIMLOCK_DISTANCE = 1000 -- Maximum distance to find a target for aimlock

-- Variables
local LocalPlayer = Players.LocalPlayer
local Camera = Workspace.CurrentCamera

local IsAimLockActive = false
local CurrentTargetPlayer: Player | nil = nil
local CurrentTargetCharacter: Model | nil = nil
local CurrentTargetHead: Part | nil = nil

-- Ensure the camera is set to scriptable mode
Camera.CameraType = Enum.CameraType.Scriptable
Camera.FieldOfView = CAMERA_FIELD_OF_VIEW

-- Function to find the closest player's head (excluding the local player)
local function findClosestPlayerHead(): (Player?, Model?, Part?)
    local closestPlayer: Player | nil = nil
    local closestCharacter: Model | nil = nil
    local closestHead: Part | nil = nil
    local minDistance = math.huge

    if not LocalPlayer or not LocalPlayer.Character or not LocalPlayer.Character:FindFirstChild("HumanoidRootPart") then
        return nil, nil, nil
    end

    local localPlayerRootPart = LocalPlayer.Character.HumanoidRootPart

    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and player.Character and player.Character:FindFirstChild("HumanoidRootPart") and player.Character:FindFirstChild("Head") then
            local targetRootPart = player.Character.HumanoidRootPart
            local targetHead = player.Character.Head
            local distance = (localPlayerRootPart.Position - targetRootPart.Position).Magnitude

            if distance < minDistance and distance <= MAX_AIMLOCK_DISTANCE then
                closestPlayer = player
                closestCharacter = player.Character
                closestHead = targetHead
                minDistance = distance
            end
        end
    end
    return closestPlayer, closestCharacter, closestHead
end

-- Handle keybind input for toggling aimlock
UserInputService.InputBegan:Connect(function(input, gameProcessedEvent)
    if not gameProcessedEvent and input.KeyCode == AIMLOCK_KEY then
        IsAimLockActive = not IsAimLockActive
        CurrentTargetPlayer = nil -- Clear target when toggling
        CurrentTargetCharacter = nil
        CurrentTargetHead = nil
        print("Aimlock Toggled: " .. tostring(IsAimLockActive))
    end
end)

-- Main update loop
RunService.Heartbeat:Connect(function(deltaTime)
    -- Ensure we have a local player and camera
    if not LocalPlayer or not LocalPlayer.Character or not LocalPlayer.Character:FindFirstChild("HumanoidRootPart") or not Camera then
        return
    end

    local localPlayerRootPart = LocalPlayer.Character.HumanoidRootPart
    local desiredCameraPosition: Vector3
    local targetLookAtPosition: Vector3
    local currentCameraOffset: Vector3

    if IsAimLockActive then
        -- If aimlock is active, try to find and lock onto a target's head
        if not CurrentTargetHead or not CurrentTargetHead.Parent or not CurrentTargetHead.Parent:FindFirstChildOfClass("Humanoid") or CurrentTargetHead.Parent:FindFirstChildOfClass("Humanoid").Health <= 0 then
            -- If current target is invalid or dead, find a new one
            CurrentTargetPlayer, CurrentTargetCharacter, CurrentTargetHead = findClosestPlayerHead()
        end

        if CurrentTargetHead then
            targetLookAtPosition = CurrentTargetHead.Position
            currentCameraOffset = AIMLOCK_CAMERA_OFFSET
            -- Position camera relative to the local player, but looking at the target's head
            desiredCameraPosition = localPlayerRootPart.Position + localPlayerRootPart.CFrame.RightVector * currentCameraOffset.X +
                                    localPlayerRootPart.CFrame.UpVector * currentCameraOffset.Y +
                                    localPlayerRootPart.CFrame.LookVector * -currentCameraOffset.Z
        else
            -- No target found, revert to following local player's root part
            targetLookAtPosition = localPlayerRootPart.Position
            currentCameraOffset = CAMERA_OFFSET
            desiredCameraPosition = localPlayerRootPart.Position + localPlayerRootPart.CFrame.RightVector * currentCameraOffset.X +
                                    localPlayerRootPart.CFrame.UpVector * currentCameraOffset.Y +
                                    localPlayerRootPart.CFrame.LookVector * -currentCameraOffset.Z
        end
    else
        -- Default camera following (follow local player's root part)
        targetLookAtPosition = localPlayerRootPart.Position
        currentCameraOffset = CAMERA_OFFSET
        desiredCameraPosition = localPlayerRootPart.Position + localPlayerRootPart.CFrame.RightVector * currentCameraOffset.X +
                                localPlayerRootPart.CFrame.UpVector * currentCameraOffset.Y +
                                localPlayerRootPart.CFrame.LookVector * -currentCameraOffset.Z
    end

    -- Calculate the CFrame for the desired camera position, looking at the target
    local desiredCameraCFrame = CFrame.lookAt(desiredCameraPosition, targetLookAtPosition)

    -- Raycast to check for obstacles between the desired camera position and the target
    -- This helps prevent the camera from going through walls.
    local raycastParams = RaycastParams.new()
    raycastParams.FilterType = Enum.RaycastFilterType.Exclude
    -- Exclude local player's character, current target character, and camera itself
    local filterInstances = {LocalPlayer.Character, Camera}
    if CurrentTargetCharacter then
        table.insert(filterInstances, CurrentTargetCharacter)
    end
    raycastParams.FilterDescendantsInstances = filterInstances

    local raycastResult = Workspace:Raycast(targetLookAtPosition, (desiredCameraPosition - targetLookAtPosition).Unit * MAX_RAYCAST_DISTANCE, raycastParams)

    if raycastResult then
        -- If an obstacle is hit, adjust the desired camera position to be closer to the target
        -- but still outside the obstacle.
        local hitPosition = raycastResult.Position
        local distanceToHit = (hitPosition - targetLookAtPosition).Magnitude

        -- Ensure the camera doesn't get too close
        if distanceToHit < (desiredCameraPosition - targetLookAtPosition).Magnitude - MIN_CAMERA_DISTANCE then
            local adjustedDirection = (hitPosition - targetLookAtPosition).Unit
            desiredCameraPosition = targetLookAtPosition + adjustedDirection * math.max(MIN_CAMERA_DISTANCE, distanceToHit - 1)
            desiredCameraCFrame = CFrame.lookAt(desiredCameraPosition, targetLookAtPosition)
        end
    end

    -- Smoothly interpolate the camera's CFrame to the desired CFrame
    -- We use deltaTime to make the smoothing frame-rate independent.
    local alpha = 1 - math.pow(1 - SMOOTHNESS_FACTOR, deltaTime * 60) -- Adjust for frame rate
    Camera.CFrame = Camera.CFrame:Lerp(desiredCameraCFrame, alpha)
end)

-- Optional: Reset camera type if the script is disabled or destroyed (for cleanup)
script.AncestryChanged:Connect(function()
    if not script.Parent then
        Camera.CameraType = Enum.CameraType.Custom
    end
end)

script.Disabled:Connect(function()
    if script.Disabled then
        Camera.CameraType = Enum.CameraType.Custom
    end
end)
