local OrionLib = loadstring(game:HttpGet("https://raw.githubusercontent.com/jadpy/suki/refs/heads/main/orion"))()
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local RunService = game:GetService("RunService")
local Workspace = game:GetService("Workspace")

local LocalPlayer = Players.LocalPlayer
local GrabEvents = ReplicatedStorage:WaitForChild("GrabEvents")

local isKickEnabled = false
local selectedPlayerText = ""
local defaultDistance = 15
local maxTeleportDistance = 25

local SpawnToyRemote = ReplicatedStorage:WaitForChild("MenuToys"):WaitForChild("SpawnToyRemoteFunction")
local SetNetworkOwnerRemote = GrabEvents:WaitForChild("SetNetworkOwner")

local kickState = {
    enabled = false
}
local targetState = {
    targetName = nil,
}
local connections = {}
local isSpawningPallet = false

-- ===== 랙 방식 (초당 100회 / 0.01초에 1회) =====
local running = false
local ragServer = false
local targetLag = false
local selectedPlayerName = nil

local createLineRemote = GrabEvents:FindFirstChild("CreateGrabLine")
-- ================================================

-- 플레이어 목록 가져오기 함수
local function getPlayerList()
    local playerNames = {}
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer then
            table.insert(playerNames, player.DisplayName .. " (@" .. player.Name .. ")")
        end
    end
    return playerNames
end

-- 드롭다운 텍스트로 플레이어 객체 찾기
local function getPlayerFromText(playerText)
    for _, player in ipairs(Players:GetPlayers()) do
        local formattedText = player.DisplayName .. " (@" .. player.Name .. ")"
        if formattedText == playerText then return player end
    end
    return nil
end

-- 타겟 근처로 순간이동 및 네트워크 오너십 요청
local function claimNetworkOwnership(targetPlayer, myRootPart)
    if not targetPlayer.Character then return end
    local targetRootPart = targetPlayer.Character:FindFirstChild("HumanoidRootPart")
    if not targetRootPart or not myRootPart then return end

    local originalCFrame = myRootPart.CFrame
    myRootPart.CFrame = targetRootPart.CFrame * CFrame.new(0, 0, 2)

    for i = 1, 15 do
        SetNetworkOwnerRemote:FireServer(targetRootPart, targetRootPart.CFrame)
        task.wait()
    end

    myRootPart.CFrame = originalCFrame
end

-- 자식 오브젝트 찾기 도우미 함수
local function findOrWaitForChild(parent, childName, timeOut)
    return parent:FindFirstChild(childName) or parent:WaitForChild(childName, timeOut or 5)
end

-- 네트워크 오너십 전송
local function syncPartOwnership(part)
    if part and part:IsA("BasePart") then
        SetNetworkOwnerRemote:FireServer(part, part.CFrame)
        task.wait()
    end
end

-- 자식 존재 여부 확인
local function hasChild(parent, childName)
    return parent:FindFirstChild(childName) ~= nil
end

-- 장난감(팔레트) 스폰 함수
local function spawnToy(toyName)
    local myCharacter = LocalPlayer.Character or LocalPlayer.CharacterAdded:Wait()
    local myRootPart = myCharacter:WaitForChild("HumanoidRootPart")

    local waitCount = 0
    while (LocalPlayer.InPlot.Value and not LocalPlayer.InOwnedPlot.Value) and waitCount < 50 do
        task.wait(0.1)
        waitCount = waitCount + 1
    end
    waitCount = 0
    while not LocalPlayer.CanSpawnToy.Value and waitCount < 50 do
        task.wait(0.1)
        waitCount = waitCount + 1
    end

    local spawnCFrame = myRootPart.CFrame * CFrame.new(0, 14, 20)
    
    local spawnFolder = workspace:FindFirstChild(LocalPlayer.Name .. "SpawnedInToys")
    if not spawnFolder then
        spawnFolder = workspace:FindFirstChild("PlotItems")
        if spawnFolder then
            spawnFolder = spawnFolder:FindFirstChild("Plot1")
        end
    end
    if not spawnFolder then
        spawnFolder = workspace
    end

    local spawnedToy = nil
    local childAddedConnection = spawnFolder.ChildAdded:Connect(function(child)
        if child.Name == toyName then
            spawnedToy = child
        end
    end)

    task.spawn(function()
        pcall(function()
            SpawnToyRemote:InvokeServer(toyName, spawnCFrame, Vector3.zero)
        end)
    end)

    local startTime = tick()
    repeat task.wait(0.05) until spawnedToy or (tick() - startTime) > 5
    childAddedConnection:Disconnect()
    return spawnedToy
end

-- 래그돌 유발용 팔레트 스폰 및 설정
local function createRagdollPallet()
    if isSpawningPallet then return nil end
    isSpawningPallet = true

    local palletModel = spawnToy("PalletLightBrown")
    if not palletModel then
        isSpawningPallet = false
        return nil
    end

    local soundPart = findOrWaitForChild(palletModel, "SoundPart", 3)
    if not soundPart then
        palletModel:Destroy()
        isSpawningPallet = false
        return nil
    end

    local retryCount = 0
    while retryCount < 10 do
        if not kickState.enabled then
            palletModel:Destroy()
            isSpawningPallet = false
            return nil
        end
        syncPartOwnership(soundPart)
        task.wait()
        if hasChild(soundPart, "PartOwner") then
            break
        end
        retryCount = retryCount + 1
    end

    if not hasChild(soundPart, "PartOwner") then
        palletModel:Destroy()
        isSpawningPallet = false
        return nil
    end

    for _, descendant in pairs(palletModel:GetDescendants()) do
        if descendant:IsA("BasePart") then
            descendant.CanCollide = false
            descendant.Transparency = 0.8
        end
    end
    palletModel.Name = "RagdollPalete"

    local bodyVelocity = Instance.new("BodyVelocity")
    bodyVelocity.MaxForce = Vector3.new(0, math.huge, 0)
    bodyVelocity.Velocity = Vector3.new(0, 900, 0)
    bodyVelocity.Parent = soundPart

    isSpawningPallet = false
    return palletModel
end

-- ===== 랙 방식 (즉시 연타 / 0초 대기) =====
local function startLagSpam()
    if running then return end
    if not createLineRemote then return end
    running = true

    task.spawn(function()
        while running do
            local spawnLocation = Workspace:FindFirstChild("SpawnLocation")
                or Workspace:FindFirstChild("Spawn")
                or (LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart"))

            if spawnLocation then
                createLineRemote:FireServer(spawnLocation, CFrame.new(math.random(-2010000000, 2000000001), 0, math.random(-2008100000, 2000200000)))
            end
            task.wait()  -- 대기 시간 0초 (최속)
        end
    end)
end

local function stopLagSpam()
    if not running then return end
    running = false
end
-- ================================================

-- GUI 생성
local Window = OrionLib:MakeWindow({Name = "Grabkick", HidePremium = true, SaveConfig = false})

-- 탭 1
local Tab1 = Window:MakeTab({Name = "1", Icon = "rbxassetid://4483345998"})

local TargetDropdown = Tab1:AddDropdown({
    Name = "Target Player",
    Default = "",
    Options = getPlayerList(),
    Callback = function(selectedText)
        selectedPlayerText = selectedText
        local targetPlayer = getPlayerFromText(selectedText)
        if targetPlayer then
            targetState.targetName = targetPlayer.Name
        else
            targetState.targetName = nil
        end
    end
})

Tab1:AddToggle({
    Name = "Grab kick (BETA)",
    Default = false,
    Callback = function(enabled)
        isKickEnabled = enabled
        kickState.enabled = enabled

        if enabled then
            startLagSpam()
        else
            stopLagSpam()
        end

        if isKickEnabled then
            task.spawn(function()
                while isKickEnabled do
                    local targetPlayer = getPlayerFromText(selectedPlayerText)
                    local myCharacter = LocalPlayer.Character
                    local myRootPart = myCharacter and myCharacter:FindFirstChild("HumanoidRootPart")

                    if targetPlayer and myRootPart then
                        local targetCharacter = targetPlayer.Character
                        local targetRootPart = targetCharacter and targetCharacter:FindFirstChild("HumanoidRootPart")

                        if targetRootPart then
                            local distance = (myRootPart.Position - targetRootPart.Position).Magnitude
                            if distance > maxTeleportDistance then
                                claimNetworkOwnership(targetPlayer, myRootPart)
                            end

                            SetNetworkOwnerRemote:FireServer(targetRootPart, targetRootPart.CFrame)
                            if GrabEvents:FindFirstChild("DestroyGrabLine") then
                                GrabEvents.DestroyGrabLine:FireServer(targetRootPart)
                            end

                            targetRootPart.AssemblyLinearVelocity = Vector3.zero
                            targetRootPart.AssemblyAngularVelocity = Vector3.zero

                            local bodyPosition = targetRootPart:FindFirstChild("ControlBP")
                            if not bodyPosition then
                                bodyPosition = Instance.new("BodyPosition")
                                bodyPosition.Name = "ControlBP"
                                bodyPosition.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
                                bodyPosition.P = 800000
                                bodyPosition.Parent = targetRootPart
                            end
                            bodyPosition.Position = myRootPart.Position + Vector3.new(5, 10, 5)
                        end
                    end
                    task.wait()  -- 대기 시간 0초 (최속)
                end

                local targetPlayer = getPlayerFromText(selectedPlayerText)
                if targetPlayer and targetPlayer.Character then
                    local targetRootPart = targetPlayer.Character:FindFirstChild("HumanoidRootPart")
                    if targetRootPart and targetRootPart:FindFirstChild("ControlBP") then
                        targetRootPart.ControlBP:Destroy()
                    end
                end
            end)
        end

        if enabled then
            local myToysFolder = workspace:FindFirstChild(LocalPlayer.Name .. "SpawnedInToys")
            local activePallet = nil

            connections["renderLoop"] = RunService.RenderStepped:Connect(function()
                if not kickState.enabled then return end
                if not targetState.targetName then return end

                local targetPlayer = Players:FindFirstChild(targetState.targetName)
                if not targetPlayer or not targetPlayer.Character then return end

                local targetRootPart = targetPlayer.Character:FindFirstChild("HumanoidRootPart")
                local targetHumanoid = targetPlayer.Character:FindFirstChild("Humanoid")
                if not targetRootPart or not targetHumanoid then return end

                if activePallet and activePallet:IsDescendantOf(workspace) then
                    local soundPart = activePallet:FindFirstChild("SoundPart")
                    if soundPart then
                        if not hasChild(soundPart, "PartOwner") then
                            activePallet:Destroy()
                            activePallet = nil
                        end
                    else
                        activePallet:Destroy()
                        activePallet = nil
                    end
                end

                if not isSpawningPallet and (not activePallet or not activePallet:IsDescendantOf(workspace)) then
                    activePallet = myToysFolder and myToysFolder:FindFirstChild("RagdollPalete") or createRagdollPallet()
                end

                if activePallet and activePallet:FindFirstChild("SoundPart") then
                    local ragdollValue = targetHumanoid:FindFirstChild("Ragdolled")
                    if ragdollValue and not ragdollValue.Value then
                        activePallet.SoundPart.Position = targetRootPart.Position
                    end
                end
            end)
        else
            if connections["renderLoop"] then
                connections["renderLoop"]:Disconnect()
                connections["renderLoop"] = nil
            end
        end
    end
})

-- 플레이어 접속/퇴장 시 목록 갱신
local function updatePlayerDropdown(removedPlayer)
    if removedPlayer and targetState.targetName == removedPlayer.Name then
        targetState.targetName = nil
    end
    TargetDropdown:Refresh(getPlayerList(), true)
end

Players.PlayerAdded:Connect(function()
    task.wait(0.5)
    updatePlayerDropdown()
end)
Players.PlayerRemoving:Connect(updatePlayerDropdown)

OrionLib:Init()
