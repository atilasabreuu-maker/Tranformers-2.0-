--[[
================================================================================
    SISTEMA DE TRANSFORMAÇÃO MECHA (TRANSFORMERS) PARA ROBLOX
    Versão: 2.0 - Mobile Optimized
    Autor: Sistema de IA
    Descrição: Sistema completo de transformação Humanoid <-> Mecha <-> Veículo
    com animações procedurais, efeitos visuais imersivos e suporte 100% mobile.
================================================================================
    
    ESTRUTURA DE PASTAS NECESSÁRIA (ReplicatedStorage):
    
    ReplicatedStorage/
    ├── MechaSystem/
    │   ├── LocalScript (este script)
    │   └── RemoteEvents/
    │       ├── TransformEvent (RemoteEvent)
    │       └── MechaStateEvent (RemoteEvent)
    ├── MechaModels/
    │   ├── MechaRig (Model)
    │   └── VehicleRig (Model)
    ├── Animations/
    │   ├── MechaIdle (Animation)
    │   ├── MechaWalk (Animation)
    │   ├── MechaRun (Animation)
    │   ├── MechaJump (Animation)
    │   ├── VehicleIdle (Animation)
    │   ├── TransformToMecha (Animation)
    │   └── TransformToVehicle (Animation)
    └── Sounds/
        ├── TransformStart (Sound)
        ├── TransformMid (Sound)
        ├── TransformEnd (Sound)
        ├── FootstepMecha (Sound)
        ├── EngineIdle (Sound)
        └── ServoMotor (Sound)

================================================================================
]]

-- ============================================================================
-- SEÇÃO 1: SERVIÇOS E DEPENDÊNCIAS
-- ============================================================================
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local ContextActionService = game:GetService("ContextActionService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Lighting = game:GetService("Lighting")
local SoundService = game:GetService("SoundService")

-- ============================================================================
-- SEÇÃO 2: REFERÊNCIAS DO JOGADOR LOCAL
-- ============================================================================
local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")
local character = player.Character or player.CharacterAdded:Wait()
local humanoid = character:WaitForChild("Humanoid")
local humanoidRootPart = character:WaitForChild("HumanoidRootPart")
local camera = workspace.CurrentCamera

-- Aguardar o personagem carregar completamente
player.CharacterAdded:Connect(function(char)
    character = char
    humanoid = char:WaitForChild("Humanoid")
    humanoidRootPart = char:WaitForChild("HumanoidRootPart")
end)

-- ============================================================================
-- SEÇÃO 3: CONFIGURAÇÕES E CONSTANTES
-- ============================================================================
local CONFIG = {
    -- Velocidades de movimento
    HUMANOID_WALK_SPEED = 16,
    HUMANOID_JUMP_POWER = 50,
    MECHA_WALK_SPEED = 32,
    MECHA_RUN_SPEED = 48,
    MECHA_JUMP_POWER = 80,
    VEHICLE_MAX_SPEED = 120,
    VEHICLE_TURN_SPEED = 3,
    
    -- Timing da transformação
    TRANSFORM_ANTICIPATION = 0.5,    -- Fase 1: Antecipação
    TRANSFORM_DISASSEMBLY = 1.0,     -- Fase 2: Desmontagem
    TRANSFORM_ASSEMBLY = 1.5,        -- Fase 3: Montagem
    TRANSFORM_FINALIZE = 0.5,        -- Fase 4: Finalização
    TOTAL_TRANSFORM_TIME = 3.5,      -- Tempo total
    
    -- Efeitos visuais
    SPARK_COUNT = 20,
    SMOKE_DENSITY = 0.5,
    FLASH_INTENSITY = 5,
    CAMERA_SHAKE_INTENSITY = 0.3,
    CAMERA_SHAKE_DECAY = 0.9,
    
    -- Câmera
    MECHA_CAMERA_OFFSET = CFrame.new(0, 8, 16),
    VEHICLE_CAMERA_OFFSET = CFrame.new(0, 5, 20),
    HEAD_TRACK_SPEED = 0.1,
    HEAD_TRACK_LIMIT = math.rad(45),
    
    -- Cores e efeitos
    MECHA_PRIMARY_COLOR = Color3.fromRGB(40, 40, 60),
    MECHA_ACCENT_COLOR = Color3.fromRGB(0, 150, 255),
    MECHA_EYE_COLOR = Color3.fromRGB(0, 200, 255),
    MECHA_REACTOR_COLOR = Color3.fromRGB(255, 100, 0),
    VEHICLE_PRIMARY_COLOR = Color3.fromRGB(200, 30, 30),
    VEHICLE_ACCENT_COLOR = Color3.fromRGB(255, 200, 0),
    
    -- Mobile
    MOBILE_BUTTON_SIZE = UDim2.new(0, 80, 0, 80),
    MOBILE_BUTTON_POSITION = UDim2.new(1, -100, 0.5, 0),
    
    -- Física
    MECHA_HIP_HEIGHT = 8,
    VEHICLE_GROUND_OFFSET = 2,
}

-- ============================================================================
-- SEÇÃO 4: ESTADOS DO SISTEMA
-- ============================================================================
local SystemState = {
    currentForm = "Humanoid",        -- "Humanoid", "Transforming", "Mecha", "Vehicle"
    isTransforming = false,
    transformProgress = 0,
    mechaModel = nil,
    vehicleModel = nil,
    activeAnimations = {},
    cameraShake = { intensity = 0, decay = CONFIG.CAMERA_SHAKE_DECAY },
    headTrackTarget = Vector3.zero,
    proceduralTime = 0,
    mobileButtons = {},
    uiElements = {},
    connections = {},
    originalProperties = {},
}

-- ============================================================================
-- SEÇÃO 5: REMOTE EVENTS
-- ============================================================================
local mechaFolder = ReplicatedStorage:WaitForChild("MechaSystem")
local transformEvent = mechaFolder:WaitForChild("RemoteEvents"):WaitForChild("TransformEvent")
local mechaStateEvent = mechaFolder:WaitForChild("RemoteEvents"):WaitForChild("MechaStateEvent")

-- ============================================================================
-- SEÇÃO 6: FUNÇÕES UTILITÁRIAS
-- ============================================================================

--- Cria um tween com configurações padrão
--- @param instance Instance - Objeto a ser animado
--- @param properties table - Propriedades alvo
--- @param duration number - Duração em segundos
--- @param easingStyle Enum.EasingStyle - Estilo de easing
--- @param easingDirection Enum.EasingDirection - Direção do easing
--- @return Tween
local function createTween(instance, properties, duration, easingStyle, easingDirection)
    local tweenInfo = TweenInfo.new(
        duration or 0.5,
        easingStyle or Enum.EasingStyle.Quad,
        easingDirection or Enum.EasingDirection.Out
    )
    return TweenService:Create(instance, tweenInfo, properties)
end

--- Toca um som com parâmetros opcionais
--- @param soundId string - ID do som (rbxassetid://...)
--- @param parent Instance - Parent do som
--- @param volume number - Volume (0-1)
--- @param playbackSpeed number - Velocidade de reprodução
--- @return Sound
local function playSound(soundId, parent, volume, playbackSpeed)
    local sound = Instance.new("Sound")
    sound.SoundId = soundId
    sound.Volume = volume or 1
    sound.PlaybackSpeed = playbackSpeed or 1
    sound.Parent = parent or humanoidRootPart
    sound:Play()
    
    -- Limpar após terminar
    sound.Ended:Connect(function()
        sound:Destroy()
    end)
    
    return sound
end

--- Cria um Motor6D entre duas partes
--- @param part0 BasePart - Parte pai
--- @param part1 BasePart - Parte filha
--- @param c0 CFrame - Offset da parte 0
--- @param c1 CFrame - Offset da parte 1
--- @return Motor6D
local function createMotor6D(part0, part1, c0, c1)
    local motor = Instance.new("Motor6D")
    motor.Name = part1.Name .. "_Motor"
    motor.Part0 = part0
    motor.Part1 = part1
    motor.C0 = c0 or CFrame.new()
    motor.C1 = c1 or CFrame.new()
    motor.Parent = part0
    return motor
end

--- Cria um ParticleEmitter com configurações
--- @param parent Instance - Parent do emitter
--- @param config table - Configurações do emitter
--- @return ParticleEmitter
local function createParticleEmitter(parent, config)
    local emitter = Instance.new("ParticleEmitter")
    emitter.Name = config.Name or "Effect"
    emitter.Texture = config.Texture or "rbxassetid://243098098"
    emitter.Rate = config.Rate or 10
    emitter.Lifetime = config.Lifetime or NumberRange.new(0.5, 1)
    emitter.Speed = config.Speed or NumberRange.new(5, 10)
    emitter.SpreadAngle = config.SpreadAngle or Vector2.new(360, 360)
    emitter.Rotation = config.Rotation or NumberRange.new(0, 360)
    emitter.RotSpeed = config.RotSpeed or NumberRange.new(-180, 180)
    emitter.Size = config.Size or NumberSequence.new(1, 0)
    emitter.Transparency = config.Transparency or NumberSequence.new(0, 1)
    emitter.Color = config.Color or ColorSequence.new(Color3.new(1, 1, 1))
    emitter.LightEmission = config.LightEmission or 0.5
    emitter.Parent = parent
    return emitter
end

--- Cria um Attachment em uma parte
--- @param parent BasePart - Parte pai
--- @param name string - Nome do attachment
--- @param position Vector3 - Posição local
--- @return Attachment
local function createAttachment(parent, name, position)
    local attachment = Instance.new("Attachment")
    attachment.Name = name
    attachment.Position = position or Vector3.zero
    attachment.Parent = parent
    return attachment
end

--- Aplica shake de câmera
--- @param intensity number - Intensidade do shake
local function applyCameraShake(intensity)
    SystemState.cameraShake.intensity = math.max(
        SystemState.cameraShake.intensity, 
        intensity or CONFIG.CAMERA_SHAKE_INTENSITY
    )
end

--- Lerp entre dois CFrames
--- @param cf1 CFrame - CFrame inicial
--- @param cf2 CFrame - CFrame final
--- @param alpha number - Interpolação (0-1)
--- @return CFrame
local function lerpCFrame(cf1, cf2, alpha)
    return cf1:Lerp(cf2, math.clamp(alpha, 0, 1))
end

--- Obtém todas as partes de um modelo recursivamente
--- @param model Model - Modelo a buscar
--- @return table - Array de BaseParts
local function getAllParts(model)
    local parts = {}
    for _, descendant in ipairs(model:GetDescendants()) do
        if descendant:IsA("BasePart") then
            table.insert(parts, descendant)
        end
    end
    return parts
end

--- Define a cor de todas as partes de um modelo
--- @param model Model - Modelo
--- @param color Color3 - Cor alvo
local function setModelColor(model, color)
    for _, part in ipairs(getAllParts(model)) do
        part.Color = color
    end
end

-- ============================================================================
-- SEÇÃO 7: GERAÇÃO PROCEDURAL DO MECHA
-- ============================================================================

--- Cria o modelo do Mecha proceduralmente
--- @return Model - Modelo completo do Mecha
local function generateMechaModel()
    local mecha = Instance.new("Model")
    mecha.Name = "MechaRig"
    
    -- Materiais base
    local armorMaterial = Instance.new("MaterialVariant")
    
    -- Configuração das partes do mecha
    local partConfigs = {
        -- Torso Superior (peito do mecha)
        {
            name = "UpperTorso",
            size = Vector3.new(4, 3, 2.5),
            color = CONFIG.MECHA_PRIMARY_COLOR,
            position = Vector3.new(0, 5, 0),
            meshType = "Block",
            transparency = 0,
            material = Enum.Material.Metal,
        },
        -- Torso Inferior (abdômen do mecha)
        {
            name = "LowerTorso",
            size = Vector3.new(3.5, 2, 2),
            color = CONFIG.MECHA_PRIMARY_COLOR,
            position = Vector3.new(0, 2.5, 0),
            meshType = "Block",
            material = Enum.Material.Metal,
        },
        -- Cabeça do Mecha
        {
            name = "Head",
            size = Vector3.new(2.5, 2.5, 2.5),
            color = CONFIG.MECHA_PRIMARY_COLOR,
            position = Vector3.new(0, 7.5, 0),
            meshType = "Block",
            material = Enum.Material.Metal,
        },
        -- Braço Superior Esquerdo
        {
            name = "LeftUpperArm",
            size = Vector3.new(1.8, 3, 1.8),
            color = CONFIG.MECHA_PRIMARY_COLOR,
            position = Vector3.new(-3.5, 5, 0),
            meshType = "Block",
            material = Enum.Material.Metal,
        },
        -- Braço Inferior Esquerdo
        {
            name = "LeftLowerArm",
            size = Vector3.new(1.5, 3, 1.5),
            color = CONFIG.MECHA_ACCENT_COLOR,
            position = Vector3.new(-3.5, 2, 0),
            meshType = "Block",
            material = Enum.Material.Metal,
        },
        -- Braço Superior Direito
        {
            name = "RightUpperArm",
            size = Vector3.new(1.8, 3, 1.8),
            color = CONFIG.MECHA_PRIMARY_COLOR,
            position = Vector3.new(3.5, 5, 0),
            meshType = "Block",
            material = Enum.Material.Metal,
        },
        -- Braço Inferior Direito
        {
            name = "RightLowerArm",
            size = Vector3.new(1.5, 3, 1.5),
            color = CONFIG.MECHA_ACCENT_COLOR,
            position = Vector3.new(3.5, 2, 0),
            meshType = "Block",
            material = Enum.Material.Metal,
        },
        -- Perna Superior Esquerda
        {
            name = "LeftUpperLeg",
            size = Vector3.new(2, 3.5, 2),
            color = CONFIG.MECHA_PRIMARY_COLOR,
            position = Vector3.new(-1.5, -0.5, 0),
            meshType = "Block",
            material = Enum.Material.Metal,
        },
        -- Perna Inferior Esquerda
        {
            name = "LeftLowerLeg",
            size = Vector3.new(1.8, 3.5, 1.8),
            color = CONFIG.MECHA_PRIMARY_COLOR,
            position = Vector3.new(-1.5, -4, 0),
            meshType = "Block",
            material = Enum.Material.Metal,
        },
        -- Pé Esquerdo
        {
            name = "LeftFoot",
            size = Vector3.new(2, 1, 3),
            color = CONFIG.MECHA_ACCENT_COLOR,
            position = Vector3.new(-1.5, -6, 0.5),
            meshType = "Block",
            material = Enum.Material.Metal,
        },
        -- Perna Superior Direita
        {
            name = "RightUpperLeg",
            size = Vector3.new(2, 3.5, 2),
            color = CONFIG.MECHA_PRIMARY_COLOR,
            position = Vector3.new(1.5, -0.5, 0),
            meshType = "Block",
            material = Enum.Material.Metal,
        },
        -- Perna Inferior Direita
        {
            name = "RightLowerLeg",
            size = Vector3.new(1.8, 3.5, 1.8),
            color = CONFIG.MECHA_PRIMARY_COLOR,
            position = Vector3.new(1.5, -4, 0),
            meshType = "Block",
            material = Enum.Material.Metal,
        },
        -- Pé Direito
        {
            name = "RightFoot",
            size = Vector3.new(2, 1, 3),
            color = CONFIG.MECHA_ACCENT_COLOR,
            position = Vector3.new(1.5, -6, 0.5),
            meshType = "Block",
            material = Enum.Material.Metal,
        },
    }
    
    -- HumanoidRootPart do Mecha (maior que o normal)
    local hrp = Instance.new("Part")
    hrp.Name = "HumanoidRootPart"
    hrp.Size = Vector3.new(2, 3, 1)
    hrp.Transparency = 1
    hrp.CanCollide = false
    hrp.Anchored = false
    hrp.CFrame = humanoidRootPart.CFrame
    hrp.Parent = mecha
    
    -- Criar cada parte do mecha
    local parts = {}
    for _, config in ipairs(partConfigs) do
        local part = Instance.new("Part")
        part.Name = config.name
        part.Size = config.size
        part.Color = config.color
        part.Material = config.material or Enum.Material.Metal
        part.Transparency = config.transparency or 0
        part.Anchored = false
        part.CanCollide = true
        part.Massless = false
        part.CFrame = CFrame.new(config.position)
        
        -- Adicionar detalhes visuais (bordas, painéis)
        local mesh = Instance.new("SpecialMesh")
        mesh.MeshType = Enum.MeshType.Brick
        mesh.Scale = Vector3.new(1, 1, 1)
        mesh.Parent = part
        
        -- Adicionar detalhes de superfície
        local surfaceGlow = Instance.new("SurfaceLight")
        surfaceGlow.Brightness = 0.5
        surfaceGlow.Color = CONFIG.MECHA_ACCENT_COLOR
        surfaceGlow.Range = 3
        surfaceGlow.Face = Enum.NormalFace.Front
        surfaceGlow.Enabled = false
        surfaceGlow.Parent = part
        
        part.Parent = mecha
        parts[config.name] = part
    end
    
    -- Criar Humanoid do Mecha
    local mechaHumanoid = Instance.new("Humanoid")
    mechaHumanoid.HipHeight = CONFIG.MECHA_HIP_HEIGHT
    mechaHumanoid.WalkSpeed = 0
    mechaHumanoid.JumpPower = 0
    mechaHumanoid.HealthDisplayType = Enum.HumanoidHealthDisplayType.AlwaysOff
    mechaHumanoid.Parent = mecha
    
    -- Criar Motor6Ds para as juntas
    local joints = {
        -- Torso
        {part0 = "HumanoidRootPart", part1 = "LowerTorso", c0 = CFrame.new(0, -1, 0), c1 = CFrame.new(0, 1, 0)},
        {part0 = "LowerTorso", part1 = "UpperTorso", c0 = CFrame.new(0, 1, 0), c1 = CFrame.new(0, -1.5, 0)},
        {part0 = "UpperTorso", part1 = "Head", c0 = CFrame.new(0, 1.5, 0), c1 = CFrame.new(0, -1.25, 0)},
        -- Braço Esquerdo
        {part0 = "UpperTorso", part1 = "LeftUpperArm", c0 = CFrame.new(-2, 1, 0), c1 = CFrame.new(0, 1.5, 0)},
        {part0 = "LeftUpperArm", part1 = "LeftLowerArm", c0 = CFrame.new(0, -1.5, 0), c1 = CFrame.new(0, 1.5, 0)},
        -- Braço Direito
        {part0 = "UpperTorso", part1 = "RightUpperArm", c0 = CFrame.new(2, 1, 0), c1 = CFrame.new(0, 1.5, 0)},
        {part0 = "RightUpperArm", part1 = "RightLowerArm", c0 = CFrame.new(0, -1.5, 0), c1 = CFrame.new(0, 1.5, 0)},
        -- Perna Esquerda
        {part0 = "LowerTorso", part1 = "LeftUpperLeg", c0 = CFrame.new(-1, -1, 0), c1 = CFrame.new(0, 1.75, 0)},
        {part0 = "LeftUpperLeg", part1 = "LeftLowerLeg", c0 = CFrame.new(0, -1.75, 0), c1 = CFrame.new(0, 1.75, 0)},
        {part0 = "LeftLowerLeg", part1 = "LeftFoot", c0 = CFrame.new(0, -1.75, 0), c1 = CFrame.new(0, 0.5, 0)},
        -- Perna Direita
        {part0 = "LowerTorso", part1 = "RightUpperLeg", c0 = CFrame.new(1, -1, 0), c1 = CFrame.new(0, 1.75, 0)},
        {part0 = "RightUpperLeg", part1 = "RightLowerLeg", c0 = CFrame.new(0, -1.75, 0), c1 = CFrame.new(0, 1.75, 0)},
        {part0 = "RightLowerLeg", part1 = "RightFoot", c0 = CFrame.new(0, -1.75, 0), c1 = CFrame.new(0, 0.5, 0)},
    }
    
    for _, joint in ipairs(joints) do
        if parts[joint.part0] and parts[joint.part1] then
            createMotor6D(parts[joint.part0], parts[joint.part1], joint.c0, joint.c1)
        elseif joint.part0 == "HumanoidRootPart" then
            createMotor6D(hrp, parts[joint.part1], joint.c0, joint.c1)
        end
    end
    
    -- Adicionar Attachments para efeitos
    local attachments = {
        {parent = "LeftUpperArm", name = "Attachment_Thruster", pos = Vector3.new(-0.9, 0, 0)},
        {parent = "RightUpperArm", name = "Attachment_Thruster", pos = Vector3.new(0.9, 0, 0)},
        {parent = "UpperTorso", name = "Attachment_Reactor", pos = Vector3.new(0, 0, 1.3)},
        {parent = "Head", name = "Attachment_EyeL", pos = Vector3.new(-0.5, 0.2, 1.3)},
        {parent = "Head", name = "Attachment_EyeR", pos = Vector3.new(0.5, 0.2, 1.3)},
        {parent = "LeftFoot", name = "Attachment_FootL", pos = Vector3.new(0, -0.5, 0)},
        {parent = "RightFoot", name = "Attachment_FootR", pos = Vector3.new(0, -0.5, 0)},
        {parent = "LeftLowerArm", name = "Attachment_Blade", pos = Vector3.new(0, -1.5, 0)},
        {parent = "RightLowerArm", name = "Attachment_Blade", pos = Vector3.new(0, -1.5, 0)},
    }
    
    for _, att in ipairs(attachments) do
        if parts[att.parent] then
            createAttachment(parts[att.parent], att.name, att.pos)
        end
    end
    
    -- Adicionar efeitos de partículas nos ombros
    local sparkEmitter = createParticleEmitter(parts["LeftUpperArm"]:FindFirstChild("Attachment_Thruster") or parts["LeftUpperArm"], {
        Name = "TransformSparks",
        Texture = "rbxassetid://243098098",
        Rate = 0,
        Lifetime = NumberRange.new(0.3, 0.8),
        Speed = NumberRange.new(10, 20),
        SpreadAngle = Vector2.new(180, 180),
        Size = NumberSequence.new({
            NumberSequenceKeypoint.new(0, 0.3),
            NumberSequenceKeypoint.new(1, 0),
        }),
        Transparency = NumberSequence.new({
            NumberSequenceKeypoint.new(0, 0),
            NumberSequenceKeypoint.new(0.5, 0.2),
            NumberSequenceKeypoint.new(1, 1),
        }),
        Color = ColorSequence.new({
            ColorSequenceKeypoint.new(0, Color3.fromRGB(255, 200, 50)),
            ColorSequenceKeypoint.new(0.5, Color3.fromRGB(255, 100, 0)),
            ColorSequenceKeypoint.new(1, Color3.fromRGB(100, 100, 100)),
        }),
        LightEmission = 1,
    })
    
    -- Duplicar para o ombro direito
    local sparkEmitterR = sparkEmitter:Clone()
    sparkEmitterR.Parent = parts["RightUpperArm"]:FindFirstChild("Attachment_Thruster") or parts["RightUpperArm"]
    
    -- Reator no peito (PointLight pulsante)
    local reactorLight = Instance.new("PointLight")
    reactorLight.Name = "ReactorLight"
    reactorLight.Color = CONFIG.MECHA_REACTOR_COLOR
    reactorLight.Brightness = 2
    reactorLight.Range = 15
    reactorLight.Enabled = false
    reactorLight.Parent = parts["UpperTorso"]
    
    -- Luzes dos olhos
    local eyeLightL = Instance.new("PointLight")
    eyeLightL.Name = "EyeLight"
    eyeLightL.Color = CONFIG.MECHA_EYE_COLOR
    eyeLightL.Brightness = 3
    eyeLightL.Range = 8
    eyeLightL.Enabled = false
    eyeLightL.Parent = parts["Head"]
    
    -- Definir PrimaryPart
    mecha.PrimaryPart = hrp
    
    -- Inicialmente invisível (aparece durante transformação)
    mecha.Parent = workspace
    
    -- Network ownership para o jogador local
    pcall(function()
        hrp:SetNetworkOwner(player)
    end)
    
    return mecha
end

-- ============================================================================
-- SEÇÃO 8: GERAÇÃO PROCEDURAL DO VEÍCULO
-- ============================================================================

--- Cria o modelo do Veículo proceduralmente
--- @return Model - Modelo completo do Veículo
local function generateVehicleModel()
    local vehicle = Instance.new("Model")
    vehicle.Name = "VehicleRig"
    
    -- Chassi principal
    local chassis = Instance.new("Part")
    chassis.Name = "Chassis"
    chassis.Size = Vector3.new(6, 2, 12)
    chassis.Color = CONFIG.VEHICLE_PRIMARY_COLOR
    chassis.Material = Enum.Material.Metal
    chassis.CFrame = humanoidRootPart.CFrame
    chassis.Parent = vehicle
    
    -- Cabine
    local cabin = Instance.new("Part")
    cabin.Name = "Cabin"
    cabin.Size = Vector3.new(5, 2, 5)
    cabin.Color = CONFIG.VEHICLE_PRIMARY_COLOR
    cabin.Material = Enum.Material.Metal
    cabin.CFrame = chassis.CFrame * CFrame.new(0, 1.5, -1)
    cabin.Parent = vehicle
    
    -- Vidro da cabine
    local windshield = Instance.new("Part")
    windshield.Name = "Windshield"
    windshield.Size = Vector3.new(4.5, 1.5, 0.2)
    windshield.Color = Color3.fromRGB(100, 150, 200)
    windshield.Material = Enum.Material.Glass
    windshield.Transparency = 0.3
    windshield.CFrame = cabin.CFrame * CFrame.new(0, 0.5, 2.5)
    windshield.Parent = vehicle
    
    -- Rodas
    local wheelPositions = {
        Vector3.new(-3, -1, 4),
        Vector3.new(3, -1, 4),
        Vector3.new(-3, -1, -4),
        Vector3.new(3, -1, -4),
    }
    
    for i, pos in ipairs(wheelPositions) do
        local wheel = Instance.new("Part")
        wheel.Name = "Wheel_" .. i
        wheel.Size = Vector3.new(0.5, 3, 3)
        wheel.Shape = Enum.PartType.Cylinder
        wheel.Color = Color3.fromRGB(30, 30, 30)
        wheel.Material = Enum.Material.SmoothPlastic
        wheel.CFrame = chassis.CFrame * CFrame.new(pos.X, pos.Y, pos.Z) * CFrame.Angles(0, 0, math.rad(90))
        wheel.Parent = vehicle
        
        -- Aro da roda
        local rim = Instance.new("Part")
        rim.Name = "Rim_" .. i
        rim.Size = Vector3.new(0.3, 2, 2)
        rim.Shape = Enum.PartType.Cylinder
        rim.Color = CONFIG.VEHICLE_ACCENT_COLOR
        rim.Material = Enum.Material.Metal
        rim.CFrame = wheel.CFrame
        rim.Parent = vehicle
    end
    
    -- Detalhes aerodinâmicos
    local spoiler = Instance.new("Part")
    spoiler.Name = "Spoiler"
    spoiler.Size = Vector3.new(5, 0.3, 1.5)
    spoiler.Color = CONFIG.VEHICLE_ACCENT_COLOR
    spoiler.Material = Enum.Material.Metal
    spoiler.CFrame = chassis.CFrame * CFrame.new(0, 2, -5.5)
    spoiler.Parent = vehicle
    
    -- Faróis
    local headlightL = Instance.new("Part")
    headlightL.Name = "HeadlightL"
    headlightL.Size = Vector3.new(1, 0.5, 0.3)
    headlightL.Color = Color3.fromRGB(255, 255, 200)
    headlightL.Material = Enum.Material.Neon
    headlightL.CFrame = chassis.CFrame * CFrame.new(-2, 0, 6)
    headlightL.Parent = vehicle
    
    local headlightR = headlightL:Clone()
    headlightR.Name = "HeadlightR"
    headlightR.CFrame = chassis.CFrame * CFrame.new(2, 0, 6)
    headlightR.Parent = vehicle
    
    -- SpotLights para faróis
    local spotL = Instance.new("SpotLight")
    spotL.Color = Color3.fromRGB(255, 255, 200)
    spotL.Brightness = 3
    spotL.Range = 50
    spotL.Angle = 45
    spotL.Face = Enum.NormalFace.Front
    spotL.Enabled = false
    spotL.Parent = headlightL
    
    local spotR = spotL:Clone()
    spotR.Parent = headlightR
    
    -- Humanoid para o veículo (usado para controle)
    local vehicleHumanoid = Instance.new("Humanoid")
    vehicleHumanoid.WalkSpeed = 0
    vehicleHumanoid.JumpPower = 0
    vehicleHumanoid.HealthDisplayType = Enum.HumanoidHealthDisplayType.AlwaysOff
    vehicleHumanoid.Parent = vehicle
    
    -- HumanoidRootPart para o veículo
    local vehicleHRP = Instance.new("Part")
    vehicleHRP.Name = "HumanoidRootPart"
    vehicleHRP.Size = Vector3.new(2, 2, 2)
    vehicleHRP.Transparency = 1
    vehicleHRP.CanCollide = false
    vehicleHRP.CFrame = chassis.CFrame
    vehicleHRP.Parent = vehicle
    
    -- Weld do chassis ao HRP
    local chassisWeld = Instance.new("Weld")
    chassisWeld.Part0 = vehicleHRP
    chassisWeld.Part1 = chassis
    chassisWeld.C0 = CFrame.new()
    chassisWeld.C1 = CFrame.new()
    chassisWeld.Parent = vehicleHRP
    
    -- Weld de todas as partes ao chassis
    for _, part in ipairs(vehicle:GetChildren()) do
        if part:IsA("BasePart") and part ~= vehicleHRP and part ~= chassis then
            local weld = Instance.new("Weld")
            weld.Part0 = chassis
            weld.Part1 = part
            weld.C0 = chassis.CFrame:Inverse() * part.CFrame
            weld.Parent = part
        end
    end
    
    vehicle.PrimaryPart = vehicleHRP
    vehicle.Parent = workspace
    
    -- Inicialmente invisível
    for _, part in ipairs(getAllParts(vehicle)) do
        part.Transparency = 1
        part.CanCollide = false
    end
    
    pcall(function()
        vehicleHRP:SetNetworkOwner(player)
    end)
    
    return vehicle
end

-- ============================================================================
-- SEÇÃO 9: INTERFACE DO USUÁRIO (UI)
-- ============================================================================

--- Cria a interface de transformação
local function createTransformUI()
    local screenGui = Instance.new("ScreenGui")
    screenGui.Name = "MechaTransformUI"
    screenGui.ResetOnSpawn = false
    screenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    screenGui.Parent = playerGui
    
    -- Container principal
    local mainFrame = Instance.new("Frame")
    mainFrame.Name = "MainFrame"
    mainFrame.Size = UDim2.new(0, 300, 0, 80)
    mainFrame.Position = UDim2.new(0.5, -150, 0, 20)
    mainFrame.BackgroundColor3 = Color3.fromRGB(15, 15, 25)
    mainFrame.BackgroundTransparency = 0.2
    mainFrame.Visible = false
    mainFrame.Parent = screenGui
    
    -- Borda neon
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 8)
    corner.Parent = mainFrame
    
    local stroke = Instance.new("UIStroke")
    stroke.Color = CONFIG.MECHA_ACCENT_COLOR
    stroke.Thickness = 2
    stroke.Transparency = 0.3
    stroke.Parent = mainFrame
    
    -- Texto de status
    local statusText = Instance.new("TextLabel")
    statusText.Name = "StatusText"
    statusText.Size = UDim2.new(1, 0, 0, 30)
    statusText.Position = UDim2.new(0, 0, 0, 5)
    statusText.BackgroundTransparency = 1
    statusText.Text = "SISTEMA DE TRANSFORMAÇÃO"
    statusText.TextColor3 = CONFIG.MECHA_ACCENT_COLOR
    statusText.TextSize = 14
    statusText.Font = Enum.Font.Code
    statusText.Parent = mainFrame
    
    -- Barra de progresso (fundo)
    local progressBg = Instance.new("Frame")
    progressBg.Name = "ProgressBg"
    progressBg.Size = UDim2.new(0.9, 0, 0, 12)
    progressBg.Position = UDim2.new(0.05, 0, 0, 40)
    progressBg.BackgroundColor3 = Color3.fromRGB(30, 30, 50)
    progressBg.Parent = mainFrame
    
    local progressCorner = Instance.new("UICorner")
    progressCorner.CornerRadius = UDim.new(0, 6)
    progressCorner.Parent = progressBg
    
    -- Barra de progresso (preenchimento)
    local progressBar = Instance.new("Frame")
    progressBar.Name = "ProgressBar"
    progressBar.Size = UDim2.new(0, 0, 1, 0)
    progressBar.BackgroundColor3 = CONFIG.MECHA_ACCENT_COLOR
    progressBar.Parent = progressBg
    
    local barCorner = Instance.new("UICorner")
    barCorner.CornerRadius = UDim.new(0, 6)
    barCorner.Parent = progressBar
    
    -- Texto de percentual
    local percentText = Instance.new("TextLabel")
    percentText.Name = "PercentText"
    percentText.Size = UDim2.new(1, 0, 0, 20)
    percentText.Position = UDim2.new(0, 0, 0, 55)
    percentText.BackgroundTransparency = 1
    percentText.Text = "0%"
    percentText.TextColor3 = Color3.fromRGB(200, 200, 200)
    percentText.TextSize = 12
    percentText.Font = Enum.Font.Code
    percentText.Parent = mainFrame
    
    -- Blur Effect para a tela
    local blurEffect = Instance.new("BlurEffect")
    blurEffect.Name = "TransformBlur"
    blurEffect.Size = 0
    blurEffect.Enabled = false
    blurEffect.Parent = Lighting
    
    -- Guardar referências
    SystemState.uiElements = {
        screenGui = screenGui,
        mainFrame = mainFrame,
        statusText = statusText,
        progressBar = progressBar,
        percentText = percentText,
        blurEffect = blurEffect,
    }
end

--- Atualiza a UI de transformação
--- @param progress number - Progresso (0-1)
--- @param statusText string - Texto de status
local function updateTransformUI(progress, statusText)
    local ui = SystemState.uiElements
    
    if ui.mainFrame then
        ui.mainFrame.Visible = true
        ui.progressBar.Size = UDim2.new(math.clamp(progress, 0, 1), 0, 1, 0)
        ui.percentText.Text = math.floor(progress * 100) .. "%"
        if statusText then
            ui.statusText.Text = statusText
        end
    end
    
    if ui.blurEffect then
        ui.blurEffect.Enabled = true
        ui.blurEffect.Size = progress * 10
    end
end

--- Esconde a UI de transformação
local function hideTransformUI()
    local ui = SystemState.uiElements
    
    if ui.mainFrame then
        ui.mainFrame.Visible = false
    end
    
    if ui.blurEffect then
        local fadeOut = createTween(ui.blurEffect, {Size = 0}, 0.5)
        fadeOut:Play()
        fadeOut.Completed:Connect(function()
            ui.blurEffect.Enabled = false
        end)
    end
end

-- ============================================================================
-- SEÇÃO 10: BOTÕES MOBILE
-- ============================================================================

--- Cria os botões de controle mobile
local function createMobileButtons()
    -- Verificar se é dispositivo mobile/touch
    if not UserInputService.TouchEnabled then return end
    
    local ui = SystemState.uiElements.screenGui
    
    -- Botão de Transformar para Mecha
    local transformBtn = Instance.new("TextButton")
    transformBtn.Name = "TransformBtn"
    transformBtn.Size = CONFIG.MOBILE_BUTTON_SIZE
    transformBtn.Position = UDim2.new(1, -100, 0.5, -50)
    transformBtn.BackgroundColor3 = CONFIG.MECHA_ACCENT_COLOR
    transformBtn.Text = "🤖"
    transformBtn.TextSize = 32
    transformBtn.TextColor3 = Color3.new(1, 1, 1)
    transformBtn.Font = Enum.Font.GothamBold
    transformBtn.Parent = ui
    
    local btnCorner = Instance.new("UICorner")
    btnCorner.CornerRadius = UDim.new(0, 15)
    btnCorner.Parent = transformBtn
    
    local btnStroke = Instance.new("UIStroke")
    btnStroke.Color = Color3.new(1, 1, 1)
    btnStroke.Thickness = 2
    btnStroke.Parent = transformBtn
    
    -- Botão de Transformar para Veículo
    local vehicleBtn = Instance.new("TextButton")
    vehicleBtn.Name = "VehicleBtn"
    vehicleBtn.Size = CONFIG.MOBILE_BUTTON_SIZE
    vehicleBtn.Position = UDim2.new(1, -100, 0.5, 50)
    vehicleBtn.BackgroundColor3 = CONFIG.VEHICLE_PRIMARY_COLOR
    vehicleBtn.Text = "🚗"
    vehicleBtn.TextSize = 32
    vehicleBtn.TextColor3 = Color3.new(1, 1, 1)
    vehicleBtn.Font = Enum.Font.GothamBold
    vehicleBtn.Visible = false
    vehicleBtn.Parent = ui
    
    local vBtnCorner = Instance.new("UICorner")
    vBtnCorner.CornerRadius = UDim.new(0, 15)
    vBtnCorner.Parent = vehicleBtn
    
    local vBtnStroke = Instance.new("UIStroke")
    vBtnStroke.Color = Color3.new(1, 1, 1)
    vBtnStroke.Thickness = 2
    vBtnStroke.Parent = vehicleBtn
    
    -- Botão de Voltar (quando em Mecha ou Veículo)
    local revertBtn = Instance.new("TextButton")
    revertBtn.Name = "RevertBtn"
    revertBtn.Size = CONFIG.MOBILE_BUTTON_SIZE
    revertBtn.Position = UDim2.new(1, -100, 0.5, 150)
    revertBtn.BackgroundColor3 = Color3.fromRGB(80, 80, 80)
    revertBtn.Text = "↩️"
    revertBtn.TextSize = 32
    revertBtn.TextColor3 = Color3.new(1, 1, 1)
    revertBtn.Font = Enum.Font.GothamBold
    revertBtn.Visible = false
    revertBtn.Parent = ui
    
    local rBtnCorner = Instance.new("UICorner")
    rBtnCorner.CornerRadius = UDim.new(0, 15)
    rBtnCorner.Parent = revertBtn
    
    -- Conectar eventos dos botões
    transformBtn.MouseButton1Click:Connect(function()
        if SystemState.currentForm == "Humanoid" then
            startTransformation("Mecha")
        end
    end)
    
    vehicleBtn.MouseButton1Click:Connect(function()
        if SystemState.currentForm == "Mecha" then
            startTransformation("Vehicle")
        end
    end)
    
    revertBtn.MouseButton1Click:Connect(function()
        if SystemState.currentForm == "Mecha" or SystemState.currentForm == "Vehicle" then
            startTransformation("Humanoid")
        end
    end)
    
    SystemState.mobileButtons = {
        transformBtn = transformBtn,
        vehicleBtn = vehicleBtn,
        revertBtn = revertBtn,
    }
end

--- Atualiza visibilidade dos botões baseado no estado
local function updateMobileButtons()
    local buttons = SystemState.mobileButtons
    if not buttons.transformBtn then return end
    
    local form = SystemState.currentForm
    
    buttons.transformBtn.Visible = (form == "Humanoid")
    buttons.vehicleBtn.Visible = (form == "Mecha")
    buttons.revertBtn.Visible = (form == "Mecha" or form == "Vehicle")
end

-- ============================================================================
-- SEÇÃO 11: FASES DA TRANSFORMAÇÃO
-- ============================================================================

--- FASE 1: Antecipação - Preparação para a transformação
--- @param targetForm string - Forma alvo ("Mecha" ou "Vehicle")
local function phase1Anticipation(targetForm)
    -- Parar o movimento
    humanoid.WalkSpeed = 0
    humanoid.JumpPower = 0
    
    -- Vibrar o personagem (efeito de tensão)
    local originalCFrame = humanoidRootPart.CFrame
    local shakeCount = 5
    local shakeDuration = CONFIG.TRANSFORM_ANTICIPATION / shakeCount
    
    for i = 1, shakeCount do
        local offset = CFrame.new(
            math.random(-3, 3) * 0.05,
            math.random(-3, 3) * 0.05,
            math.random(-3, 3) * 0.05
        )
        local tween = createTween(humanoidRootPart, {CFrame = originalCFrame * offset}, shakeDuration * 0.5, Enum.EasingStyle.Quad, Enum.EasingDirection.InOut)
        tween:Play()
        tween.Completed:Wait()
        
        local returnTween = createTween(humanoidRootPart, {CFrame = originalCFrame}, shakeDuration * 0.5, Enum.EasingStyle.Quad, Enum.EasingDirection.InOut)
        returnTween:Play()
        returnTween.Completed:Wait()
    end
    
    -- Tocar som de servo motor
    playSound("rbxassetid://9113869830", humanoidRootPart, 0.5, 1.2)
    
    -- Atualizar UI
    updateTransformUI(0.1, "INICIANDO SEQUÊNCIA...")
end

--- FASE 2: Desmontagem - O avatar se desmonta
--- @param targetForm string - Forma alvo
local function phase2Disassembly(targetForm)
    -- Tornar partes do personagem semi-transparentes e movê-las
    local bodyParts = {
        "Head", "UpperTorso", "LowerTorso",
        "LeftUpperArm", "LeftLowerArm", "LeftHand",
        "RightUpperArm", "RightLowerArm", "RightHand",
        "LeftUpperLeg", "LeftLowerLeg", "LeftFoot",
        "RightUpperLeg", "RightLowerLeg", "RightFoot",
    }
    
    -- Guardar propriedades originais para reverter
    for _, partName in ipairs(bodyParts) do
        local part = character:FindFirstChild(partName)
        if part and part:IsA("BasePart") then
            SystemState.originalProperties[partName] = {
                CFrame = part.CFrame,
                Transparency = part.Transparency,
                Color = part.Color,
            }
        end
    end
    
    -- Animar cada parte se movendo para posições de "montagem"
    local startTime = tick()
    local duration = CONFIG.TRANSFORM_DISASSEMBLY
    
    -- Tocar som de metal
    playSound("rbxassetid://9114263040", humanoidRootPart, 0.7, 0.8)
    
    -- TweenService para mover as partes
    for _, partName in ipairs(bodyParts) do
        local part = character:FindFirstChild(partName)
        if part and part:IsA("BasePart") then
            -- Calcular posição alvo (expandida para fora)
            local direction = (part.Position - humanoidRootPart.Position).Unit
            local targetPos = part.Position + direction * 2
            
            local tween = createTween(part, {
                CFrame = CFrame.new(targetPos) * CFrame.Angles(
                    math.random(-45, 45) * math.rad(1),
                    math.random(-45, 45) * math.rad(1),
                    math.random(-45, 45) * math.rad(1)
                ),
                Transparency = 0.5,
            }, duration, Enum.EasingStyle.Back, Enum.EasingDirection.Out)
            tween:Play()
        end
    end
    
    -- Atualizar UI durante a desmontagem
    local uiUpdateConnection = RunService.Heartbeat:Connect(function()
        local elapsed = tick() - startTime
        local progress = 0.1 + (elapsed / duration) * 0.3
        updateTransformUI(progress, "DES MONTANDO AVATAR...")
    end)
    
    task.wait(duration)
    uiUpdateConnection:Disconnect()
    
    -- Ativar partículas de faísca
    if SystemState.mechaModel then
        for _, part in ipairs(getAllParts(SystemState.mechaModel)) do
            local emitter = part:FindFirstChildOfClass("ParticleEmitter")
            if emitter and emitter.Name == "TransformSparks" then
                emitter.Rate = 50
                task.delay(1, function()
                    emitter.Rate = 0
                end)
            end
        end
    end
end

--- FASE 3: Montagem do Mecha/Veículo
--- @param targetForm string - Forma alvo
local function phase3Assembly(targetForm)
    local model = targetForm == "Mecha" and SystemState.mechaModel or SystemState.vehicleModel
    if not model then return end
    
    local startTime = tick()
    local duration = CONFIG.TRANSFORM_ASSEMBLY
    
    -- Tocar som de montagem
    playSound("rbxassetid://9114521098", humanoidRootPart, 0.8, 1.0)
    
    -- Posições iniciais "explodidas" para as partes do mecha
    local parts = getAllParts(model)
    local originalCFrames = {}
    
    -- Guardar CFrames originais e mover para posições aleatórias
    for _, part in ipairs(parts) do
        if part.Name ~= "HumanoidRootPart" then
            originalCFrames[part.Name] = part.CFrame
            
            -- Posição aleatória ao redor do personagem
            local randomOffset = Vector3.new(
                math.random(-15, 15),
                math.random(5, 20),
                math.random(-15, 15)
            )
            part.CFrame = CFrame.new(humanoidRootPart.Position + randomOffset)
            part.Transparency = 0.3
            part.CanCollide = false
        end
    end
    
    -- Animar as partes voltando para suas posições corretas
    for _, part in ipairs(parts) do
        if part.Name ~= "HumanoidRootPart" and originalCFrames[part.Name] then
            local targetCFrame = originalCFrames[part.Name]
            
            -- Delay aleatório para efeito cascata
            local delay = math.random() * 0.5
            
            task.delay(delay, function()
                local tween = createTween(part, {
                    CFrame = targetCFrame,
                    Transparency = 0,
                }, duration * 0.7, Enum.EasingStyle.Back, Enum.EasingDirection.Out)
                tween:Play()
            end)
        end
    end
    
    -- Ativar faíscas durante montagem
    if SystemState.mechaModel then
        for _, part in ipairs(getAllParts(SystemState.mechaModel)) do
            local emitter = part:FindFirstChildOfClass("ParticleEmitter")
            if emitter and emitter.Name == "TransformSparks" then
                emitter.Rate = 30
            end
        end
    end
    
    -- Atualizar UI durante montagem
    local uiUpdateConnection = RunService.Heartbeat:Connect(function()
        local elapsed = tick() - startTime
        local progress = 0.4 + (elapsed / duration) * 0.4
        updateTransformUI(progress, "MONTANDO " .. string.upper(targetForm) .. "...")
    end)
    
    task.wait(duration)
    uiUpdateConnection:Disconnect()
    
    -- Desativar faíscas
    if SystemState.mechaModel then
        for _, part in ipairs(getAllParts(SystemState.mechaModel)) do
            local emitter = part:FindFirstChildOfClass("ParticleEmitter")
            if emitter and emitter.Name == "TransformSparks" then
                emitter.Rate = 0
            end
        end
    end
end

--- FASE 4: Finalização - Completar a transformação
--- @param targetForm string - Forma alvo
local function phase4Finalize(targetForm)
    -- Flash de luz
    local flash = Instance.new("PointLight")
    flash.Color = Color3.new(1, 1, 1)
    flash.Brightness = CONFIG.FLASH_INTENSITY
    flash.Range = 30
    flash.Parent = humanoidRootPart
    
    -- Tocar som de impacto
    playSound("rbxassetid://9114825498", humanoidRootPart, 1.0, 0.9)
    
    -- Camera shake
    applyCameraShake(0.5)
    
    -- Fade out do flash
    local fadeTween = createTween(flash, {Brightness = 0}, 0.5, Enum.EasingStyle.Quad, Enum.EasingDirection.Out)
    fadeTween:Play()
    fadeTween.Completed:Connect(function()
        flash:Destroy()
    end)
    
    -- Esconder avatar humanoide
    for _, part in ipairs(character:GetChildren()) do
        if part:IsA("BasePart") and part.Name ~= "HumanoidRootPart" then
            part.Transparency = 1
            part.CanCollide = false
        end
    end
    
    -- Configurar o modelo ativo
    local model = targetForm == "Mecha" and SystemState.mechaModel or SystemState.vehicleModel
    if model then
        -- Ativar colisão
        for _, part in ipairs(getAllParts(model)) do
            part.CanCollide = true
            part.Transparency = 0
        end
        
        -- Ativar luzes
        local reactorLight = model:FindFirstChild("UpperTorso")
        if reactorLight then
            local light = reactorLight:FindFirstChild("ReactorLight")
            if light then
                light.Enabled = true
            end
        end
        
        local head = model:FindFirstChild("Head")
        if head then
            local eyeLight = head:FindFirstChild("EyeLight")
            if eyeLight then
                eyeLight.Enabled = true
            end
        end
        
        -- Ativar faróis se for veículo
        if targetForm == "Vehicle" then
            for _, part in ipairs(getAllParts(model)) do
                local spotLight = part:FindFirstChildOfClass("SpotLight")
                if spotLight then
                    spotLight.Enabled = true
                end
            end
        end
        
        -- Teleportar HRP do mecha para posição do jogador
        local modelHRP = model:FindFirstChild("HumanoidRootPart")
        if modelHRP then
            modelHRP.CFrame = humanoidRootPart.CFrame
        end
    end
    
    -- Configurar velocidades
    if targetForm == "Mecha" then
        local mechaHumanoid = SystemState.mechaModel and SystemState.mechaModel:FindFirstChildOfClass("Humanoid")
        if mechaHumanoid then
            mechaHumanoid.WalkSpeed = CONFIG.MECHA_WALK_SPEED
            mechaHumanoid.JumpPower = CONFIG.MECHA_JUMP_POWER
        end
    elseif targetForm == "Vehicle" then
        -- Vehicle usa controle customizado
    end
    
    -- Atualizar UI final
    updateTransformUI(1.0, string.upper(targetForm) .. " ATIVADO!")
    
    task.wait(0.5)
    hideTransformUI()
    
    -- Atualizar estado
    SystemState.currentForm = targetForm
    SystemState.isTransforming = false
    
    -- Atualizar botões mobile
    updateMobileButtons()
    
    -- Replicar estado para o servidor
    pcall(function()
        mechaStateEvent:FireServer(targetForm)
    end)
end

-- ============================================================================
-- SEÇÃO 12: FUNÇÃO PRINCIPAL DE TRANSFORMAÇÃO
-- ============================================================================

--- Inicia a sequência de transformação
--- @param targetForm string - Forma alvo ("Mecha", "Vehicle", ou "Humanoid")
function startTransformation(targetForm)
    -- Verificar se já está transformando
    if SystemState.isTransforming then return end
    
    -- Verificar se a transformação é válida
    local currentForm = SystemState.currentForm
    if targetForm == currentForm then return end
    
    -- Se voltando para humanoide
    if targetForm == "Humanoid" then
        SystemState.isTransforming = true
        
        -- Mostrar UI
        updateTransformUI(0, "RETORNANDO AO MODO HUMANO...")
        
        -- Esconder modelo atual
        local currentModel = currentForm == "Mecha" and SystemState.mechaModel or SystemState.vehicleModel
        if currentModel then
            for _, part in ipairs(getAllParts(currentModel)) do
                part.Transparency = 1
                part.CanCollide = false
            end
            
            -- Desativar luzes
            for _, light in ipairs(currentModel:GetDescendants()) do
                if light:IsA("Light") then
                    light.Enabled = false
                end
            end
        end
        
        -- Restaurar avatar humanoide
        for partName, props in pairs(SystemState.originalProperties) do
            local part = character:FindFirstChild(partName)
            if part and part:IsA("BasePart") then
                local tween = createTween(part, {
                    CFrame = props.CFrame,
                    Transparency = props.Transparency,
                }, 0.5, Enum.EasingStyle.Quad, Enum.EasingDirection.Out)
                tween:Play()
            end
        end
        
        task.wait(0.5)
        
        -- Restaurar velocidades
        humanoid.WalkSpeed = CONFIG.HUMANOID_WALK_SPEED
        humanoid.JumpPower = CONFIG.HUMANOID_JUMP_POWER
        
        -- Atualizar estado
        SystemState.currentForm = "Humanoid"
        SystemState.isTransforming = false
        
        hideTransformUI()
        updateMobileButtons()
        
        pcall(function()
            mechaStateEvent:FireServer("Humanoid")
        end)
        
        return
    end
    
    -- Iniciar transformação para Mecha ou Vehicle
    SystemState.isTransforming = true
    
    -- Gerar modelos se necessário
    if not SystemState.mechaModel then
        SystemState.mechaModel = generateMechaModel()
    end
    if not SystemState.vehicleModel then
        SystemState.vehicleModel = generateVehicleModel()
    end
    
    -- Executar fases da transformação
    phase1Anticipation(targetForm)
    phase2Disassembly(targetForm)
    phase3Assembly(targetForm)
    phase4Finalize(targetForm)
end

-- ============================================================================
-- SEÇÃO 13: ANIMAÇÕES PROCEDURAIS (LOOP PRINCIPAL)
-- ============================================================================

--- Loop de animação procedural para o Mecha
local function mechaProceduralAnimation(dt)
    if not SystemState.mechaModel or SystemState.currentForm ~= "Mecha" then return end
    
    SystemState.proceduralTime = SystemState.proceduralTime + dt
    
    local mecha = SystemState.mechaModel
    local t = SystemState.proceduralTime
    
    -- Respiração mecânica (movimento sutil do torso)
    local upperTorso = mecha:FindFirstChild("UpperTorso")
    if upperTorso then
        local breathMotor = upperTorso:FindFirstChild("UpperTorso_Motor")
        if breathMotor then
            local breathOffset = math.sin(t * 2) * 0.02
            breathMotor.C0 = breathMotor.C0:Lerp(
                CFrame.new(0, 1, 0) * CFrame.Angles(breathOffset, 0, 0),
                0.1
            )
        end
    end
    
    -- Head Tracking - cabeça segue o mouse
    local head = mecha:FindFirstChild("Head")
    if head then
        local headMotor = head:FindFirstChild("Head_Motor")
        if headMotor then
            local mousePos = UserInputService:GetMouseLocation()
            local screenCenter = Vector2.new(camera.ViewportSize.X / 2, camera.ViewportSize.Y / 2)
            local mouseOffset = (mousePos - screenCenter) / screenCenter
            
            local targetAngleX = math.clamp(mouseOffset.Y * CONFIG.HEAD_TRACK_LIMIT, -CONFIG.HEAD_TRACK_LIMIT, CONFIG.HEAD_TRACK_LIMIT)
            local targetAngleY = math.clamp(-mouseOffset.X * CONFIG.HEAD_TRACK_LIMIT, -CONFIG.HEAD_TRACK_LIMIT, CONFIG.HEAD_TRACK_LIMIT)
            
            headMotor.C0 = headMotor.C0:Lerp(
                CFrame.new(0, 1.5, 0) * CFrame.Angles(-targetAngleX, targetAngleY, 0),
                CONFIG.HEAD_TRACK_SPEED
            )
        end
    end
    
    -- Arm Sway - balanço dos braços baseado em velocidade
    local mechaHumanoid = mecha:FindFirstChildOfClass("Humanoid")
    if mechaHumanoid and mechaHumanoid.MoveDirection.Magnitude > 0.1 then
        local walkSpeed = mechaHumanoid.WalkSpeed
        local swayAmount = math.sin(t * 8) * 0.1 * (walkSpeed / CONFIG.MECHA_WALK_SPEED)
        
        local leftUpperArm = mecha:FindFirstChild("LeftUpperArm")
        if leftUpperArm then
            local motor = leftUpperArm:FindFirstChild("LeftUpperArm_Motor")
            if motor then
                motor.C0 = motor.C0:Lerp(
                    CFrame.new(-2, 1, 0) * CFrame.Angles(swayAmount, 0, 0),
                    0.15
                )
            end
        end
        
        local rightUpperArm = mecha:FindFirstChild("RightUpperArm")
        if rightUpperArm then
            local motor = rightUpperArm:FindFirstChild("RightUpperArm_Motor")
            if motor then
                motor.C0 = motor.C0:Lerp(
                    CFrame.new(2, 1, 0) * CFrame.Angles(-swayAmount, 0, 0),
                    0.15
                )
            end
        end
    end
    
    -- Pulsar luz do reator
    local reactorLight = upperTorso and upperTorso:FindFirstChild("ReactorLight")
    if reactorLight and reactorLight.Enabled then
        reactorLight.Brightness = 2 + math.sin(t * 3) * 0.5
    end
end

--- Loop de controle do veículo
local lastVehicleCFrame = nil
local vehicleSpeed = 0
local vehicleSteer = 0

local function vehicleControl(dt)
    if SystemState.currentForm ~= "Vehicle" or not SystemState.vehicleModel then return end
    
    local vehicle = SystemState.vehicleModel
    local hrp = vehicle:FindFirstChild("HumanoidRootPart")
    if not hrp then return end
    
    -- Controles via teclado ou mobile
    local moveDirection = Vector3.zero
    local steerInput = 0
    
    -- Input do teclado
    if UserInputService:IsKeyDown(Enum.KeyCode.W) then
        moveDirection = Vector3.new(0, 0, -1)
    elseif UserInputService:IsKeyDown(Enum.KeyCode.S) then
        moveDirection = Vector3.new(0, 0, 1)
    end
    
    if UserInputService:IsKeyDown(Enum.KeyCode.A) then
        steerInput = 1
    elseif UserInputService:IsKeyDown(Enum.KeyCode.D) then
        steerInput = -1
    end
    
    -- Aceleração/desaceleração suave
    local targetSpeed = moveDirection.Magnitude > 0 and CONFIG.VEHICLE_MAX_SPEED or 0
    vehicleSpeed = vehicleSpeed + (targetSpeed - vehicleSpeed) * dt * 2
    
    -- Aplicar movimento
    if vehicleSpeed > 0.1 then
        local forward = hrp.CFrame.LookVector
        hrp.CFrame = hrp.CFrame + forward * vehicleSpeed * dt
        
        -- Rotação (steering)
        if math.abs(steerInput) > 0.1 then
            hrp.CFrame = hrp.CFrame * CFrame.Angles(0, steerInput * CONFIG.VEHICLE_TURN_SPEED * dt, 0)
        end
    end
    
    -- Rotação das rodas
    for _, part in ipairs(vehicle:GetChildren()) do
        if part.Name:sub(1, 6) == "Wheel_" then
            part.CFrame = part.CFrame * CFrame.Angles(vehicleSpeed * dt * 0.5, 0, 0)
        end
    end
end

--- Loop de camera shake
local function updateCameraShake(dt)
    if SystemState.cameraShake.intensity > 0.01 then
        local shake = CFrame.new(
            (math.random() - 0.5) * SystemState.cameraShake.intensity,
            (math.random() - 0.5) * SystemState.cameraShake.intensity,
            (math.random() - 0.5) * SystemState.cameraShake.intensity
        )
        camera.CFrame = camera.CFrame * shake
        SystemState.cameraShake.intensity = SystemState.cameraShake.intensity * SystemState.cameraShake.decay
    end
end

--- Loop principal de atualização (RenderStepped)
RunService.RenderStepped:Connect(function(dt)
    -- Animações procedurais do Mecha
    mechaProceduralAnimation(dt)
    
    -- Controle do veículo
    vehicleControl(dt)
    
    -- Camera shake
    updateCameraShake(dt)
end)

-- ============================================================================
-- SEÇÃO 14: CONTROLE DE CÂMERA
-- ============================================================================

--- Ajusta a câmera baseado na forma atual
local function updateCameraForForm()
    local form = SystemState.currentForm
    
    if form == "Mecha" then
        -- Câmera mais distante para o mecha
        local offset = CONFIG.MECHA_CAMERA_OFFSET
        camera.CameraSubject = SystemState.mechaModel and SystemState.mechaModel:FindFirstChildOfClass("Humanoid")
    elseif form == "Vehicle" then
        -- Câmera atrás do veículo
        local offset = CONFIG.VEHICLE_CAMERA_OFFSET
        camera.CameraSubject = SystemState.vehicleModel and SystemState.vehicleModel:FindFirstChildOfClass("Humanoid")
    else
        -- Câmera padrão para humanoide
        camera.CameraSubject = humanoid
    end
end

-- ============================================================================
-- SEÇÃO 15: BIND DE INPUT (TECLADO + MOBILE)
-- ============================================================================

--- Handler de input para transformação
local function onTransformInput(actionName, inputState, inputObject)
    if inputState == Enum.UserInputState.Begin then
        if SystemState.currentForm == "Humanoid" then
            startTransformation("Mecha")
        elseif SystemState.currentForm == "Mecha" then
            startTransformation("Vehicle")
        elseif SystemState.currentForm == "Vehicle" then
            startTransformation("Humanoid")
        end
    end
end

--- Handler para reverter transformação
local function onRevertInput(actionName, inputState, inputObject)
    if inputState == Enum.UserInputState.Begin then
        if SystemState.currentForm ~= "Humanoid" then
            startTransformation("Humanoid")
        end
    end
end

-- Registrar binds
ContextActionService:BindAction("Transform", onTransformInput, true, Enum.KeyCode.T)
ContextActionService:SetPosition("Transform", UDim2.new(0, 10, 0.5, -50))
ContextActionService:SetTitle("Transform", "Transformar [T]")

ContextActionService:BindAction("RevertTransform", onRevertInput, true, Enum.KeyCode.R)
ContextActionService:SetPosition("RevertTransform", UDim2.new(0, 10, 0.5, 50))
ContextActionService:SetTitle("RevertTransform", "Reverter [R]")

-- ============================================================================
-- SEÇÃO 16: CONTROLE MOBILE (TOUCH)
-- ============================================================================

-- Detectar input de touch para movimento do veículo
if UserInputService.TouchEnabled then
    local touchMoveVector = Vector3.zero
    
    -- Usar o TouchMovement do próprio Roblox para direção
    -- Adicionar controles customizados para o veículo
    local function setupMobileVehicleControls()
        -- Botões de direção para o veículo (aparecem quando em modo Vehicle)
        local ui = SystemState.uiElements and SystemState.uiElements.screenGui
        if not ui then return end
        
        -- Botão de Acelerar
        local accelerateBtn = Instance.new("TextButton")
        accelerateBtn.Name = "AccelerateBtn"
        accelerateBtn.Size = UDim2.new(0, 70, 0, 70)
        accelerateBtn.Position = UDim2.new(0, 20, 1, -170)
        accelerateBtn.BackgroundColor3 = Color3.fromRGB(0, 200, 0)
        accelerateBtn.Text = "▲"
        accelerateBtn.TextSize = 28
        accelerateBtn.TextColor3 = Color3.new(1, 1, 1)
        accelerateBtn.Font = Enum.Font.GothamBold
        accelerateBtn.Visible = false
        accelerateBtn.Parent = ui
        
        local accCorner = Instance.new("UICorner")
        accCorner.CornerRadius = UDim.new(1, 0)
        accCorner.Parent = accelerateBtn
        
        -- Botão de Ré
        local reverseBtn = Instance.new("TextButton")
        reverseBtn.Name = "ReverseBtn"
        reverseBtn.Size = UDim2.new(0, 70, 0, 70)
        reverseBtn.Position = UDim2.new(0, 20, 1, -90)
        reverseBtn.BackgroundColor3 = Color3.fromRGB(200, 0, 0)
        reverseBtn.Text = "▼"
        reverseBtn.TextSize = 28
        reverseBtn.TextColor3 = Color3.new(1, 1, 1)
        reverseBtn.Font = Enum.Font.GothamBold
        reverseBtn.Visible = false
        reverseBtn.Parent = ui
        
        local revCorner = Instance.new("UICorner")
        revCorner.CornerRadius = UDim.new(1, 0)
        revCorner.Parent = reverseBtn
        
        -- Botão de Virar Esquerda
        local leftBtn = Instance.new("TextButton")
        leftBtn.Name = "LeftBtn"
        leftBtn.Size = UDim2.new(0, 70, 0, 70)
        leftBtn.Position = UDim2.new(0, 20, 1, -250)
        leftBtn.BackgroundColor3 = Color3.fromRGB(50, 50, 200)
        leftBtn.Text = "◄"
        leftBtn.TextSize = 28
        leftBtn.TextColor3 = Color3.new(1, 1, 1)
        leftBtn.Font = Enum.Font.GothamBold
        leftBtn.Visible = false
        leftBtn.Parent = ui
        
        local leftCorner = Instance.new("UICorner")
        leftCorner.CornerRadius = UDim.new(1, 0)
        leftCorner.Parent = leftBtn
        
        -- Botão de Virar Direita
        local rightBtn = Instance.new("TextButton")
        rightBtn.Name = "RightBtn"
        rightBtn.Size = UDim2.new(0, 70, 0, 70)
        rightBtn.Position = UDim2.new(0, 100, 1, -250)
        rightBtn.BackgroundColor3 = Color3.fromRGB(50, 50, 200)
        rightBtn.Text = "►"
        rightBtn.TextSize = 28
        rightBtn.TextColor3 = Color3.new(1, 1, 1)
        rightBtn.Font = Enum.Font.GothamBold
        rightBtn.Visible = false
        rightBtn.Parent = ui
        
        local rightCorner = Instance.new("UICorner")
        rightCorner.CornerRadius = UDim.new(1, 0)
        rightCorner.Parent = rightBtn
        
        -- Estado dos botões
        local vehicleMobileState = {
            accelerating = false,
            reversing = false,
            turningLeft = false,
            turningRight = false,
        }
        
        -- Conectar eventos
        accelerateBtn.MouseButton1Down:Connect(function()
            vehicleMobileState.accelerating = true
        end)
        accelerateBtn.MouseButton1Up:Connect(function()
            vehicleMobileState.accelerating = false
        end)
        
        reverseBtn.MouseButton1Down:Connect(function()
            vehicleMobileState.reversing = true
        end)
        reverseBtn.MouseButton1Up:Connect(function()
            vehicleMobileState.reversing = false
        end)
        
        leftBtn.MouseButton1Down:Connect(function()
            vehicleMobileState.turningLeft = true
        end)
        leftBtn.MouseButton1Up:Connect(function()
            vehicleMobileState.turningLeft = false
        end)
        
        rightBtn.MouseButton1Down:Connect(function()
            vehicleMobileState.turningRight = true
        end)
        rightBtn.MouseButton1Up:Connect(function()
            vehicleMobileState.turningRight = false
        end)
        
        -- Guardar referências
        SystemState.mobileButtons.vehicleControls = {
            accelerate = accelerateBtn,
            reverse = reverseBtn,
            left = leftBtn,
            right = rightBtn,
            state = vehicleMobileState,
        }
    end
    
    -- Atualizar visibilidade dos controles mobile do veículo
    local function updateVehicleMobileControls()
        local controls = SystemState.mobileButtons.vehicleControls
        if not controls then return end
        
        local visible = SystemState.currentForm == "Vehicle"
        controls.accelerate.Visible = visible
        controls.reverse.Visible = visible
        controls.left.Visible = visible
        controls.right.Visible = visible
    end
    
    -- Integrar com o loop de controle do veículo
    local originalVehicleControl = vehicleControl
    vehicleControl = function(dt)
        if SystemState.currentForm ~= "Vehicle" or not SystemState.vehicleModel then return end
        
        local vehicle = SystemState.vehicleModel
        local hrp = vehicle:FindFirstChild("HumanoidRootPart")
        if not hrp then return end
        
        local controls = SystemState.mobileButtons.vehicleControls
        local mobileState = controls and controls.state
        
        -- Combinar input de teclado e mobile
        local accelerating = UserInputService:IsKeyDown(Enum.KeyCode.W) or (mobileState and mobileState.accelerating)
        local reversing = UserInputService:IsKeyDown(Enum.KeyCode.S) or (mobileState and mobileState.reversing)
        local turningLeft = UserInputService:IsKeyDown(Enum.KeyCode.A) or (mobileState and mobileState.turningLeft)
        local turningRight = UserInputService:IsKeyDown(Enum.KeyCode.D) or (mobileState and mobileState.turningRight)
        
        -- Aceleração
        local targetSpeed = 0
        if accelerating then
            targetSpeed = CONFIG.VEHICLE_MAX_SPEED
        elseif reversing then
            targetSpeed = -CONFIG.VEHICLE_MAX_SPEED * 0.5
        end
        
        vehicleSpeed = vehicleSpeed + (targetSpeed - vehicleSpeed) * dt * 2
        
        -- Steering
        local steerInput = 0
        if turningLeft then steerInput = 1 end
        if turningRight then steerInput = -1 end
        
        -- Aplicar movimento
        if math.abs(vehicleSpeed) > 0.1 then
            local forward = hrp.CFrame.LookVector
            local direction = vehicleSpeed > 0 and 1 or -1
            hrp.CFrame = hrp.CFrame + forward * vehicleSpeed * dt * direction
            
            -- Rotação
            if math.abs(steerInput) > 0.1 then
                local turnRate = CONFIG.VEHICLE_TURN_SPEED * (math.abs(vehicleSpeed) / CONFIG.VEHICLE_MAX_SPEED)
                hrp.CFrame = hrp.CFrame * CFrame.Angles(0, steerInput * turnRate * dt, 0)
            end
        end
        
        -- Rotação das rodas
        for _, part in ipairs(vehicle:GetChildren()) do
            if part.Name:sub(1, 6) == "Wheel_" then
                part.CFrame = part.CFrame * CFrame.Angles(vehicleSpeed * dt * 0.5, 0, 0)
            end
        end
        
        -- Efeito de inclinação ao virar
        local chassis = vehicle:FindFirstChild("Chassis")
        if chassis and math.abs(steerInput) > 0.1 then
            local tiltAngle = steerInput * 0.05 * (math.abs(vehicleSpeed) / CONFIG.VEHICLE_MAX_SPEED)
            -- Aplicar tilt visual (não afeta a física)
        end
    end
    
    -- Atualizar controles mobile quando o estado muda
    local originalUpdateMobileButtons = updateMobileButtons
    updateMobileButtons = function()
        originalUpdateMobileButtons()
        updateVehicleMobileControls()
    end
    
    -- Setup inicial (com delay para garantir que a UI existe)
    task.delay(1, function()
        setupMobileVehicleControls()
        updateVehicleMobileControls()
    end)
end

-- ============================================================================
-- SEÇÃO 17: EFEITOS DE PASSOS (MECHA)
-- ============================================================================

--- Cria efeito de passo do Mecha
local function createFootstepEffect(position)
    -- Partícula de poeira/fumaça no pé
    local attachment = Instance.new("Attachment")
    attachment.Position = Vector3.new(0, -0.5, 0)
    attachment.Parent = humanoidRootPart
    
    local dustEmitter = createParticleEmitter(attachment, {
        Name = "FootstepDust",
        Texture = "rbxassetid://241866187",
        Rate = 30,
        Lifetime = NumberRange.new(0.5, 1.5),
        Speed = NumberRange.new(2, 5),
        SpreadAngle = Vector2.new(90, 0),
        Size = NumberSequence.new({
            NumberSequenceKeypoint.new(0, 0.5),
            NumberSequenceKeypoint.new(1, 2),
        }),
        Transparency = NumberSequence.new({
            NumberSequenceKeypoint.new(0, 0.5),
            NumberSequenceKeypoint.new(1, 1),
        }),
        Color = ColorSequence.new(Color3.fromRGB(150, 130, 100)),
        LightEmission = 0,
    })
    
    -- Som de passo metálico
    playSound("rbxassetid://9115132456", humanoidRootPart, 0.5, math.random(80, 120) / 100)
    
    -- Camera shake sutil
    applyCameraShake(0.1)
    
    -- Limpar após um tempo
    task.delay(1.5, function()
        dustEmitter.Rate = 0
        task.delay(2, function()
            attachment:Destroy()
        end)
    end)
end

-- Detectar passos do mecha
local lastFootstepTime = 0
RunService.Heartbeat:Connect(function(dt)
    if SystemState.currentForm ~= "Mecha" then return end
    
    local mecha = SystemState.mechaModel
    if not mecha then return end
    
    local mechaHumanoid = mecha:FindFirstChildOfClass("Humanoid")
    if not mechaHumanoid then return end
    
    -- Verificar se está andando
    if mechaHumanoid.MoveDirection.Magnitude > 0.1 then
        local now = tick()
        local stepInterval = 0.6 -- Intervalo entre passos
        
        if now - lastFootstepTime > stepInterval then
            lastFootstepTime = now
            local hrp = mecha:FindFirstChild("HumanoidRootPart")
            if hrp then
                createFootstepEffect(hrp.Position)
            end
        end
    end
end)

-- ============================================================================
-- SEÇÃO 18: TRAILS NOS BRAÇOS (MOVIMENTO RÁPIDO)
-- ============================================================================

--- Cria trails nos braços do mecha
local function createArmTrails()
    if not SystemState.mechaModel then return end
    
    local mecha = SystemState.mechaModel
    
    local function addTrailToPart(partName, attachName)
        local part = mecha:FindFirstChild(partName)
        if not part then return end
        
        local att0 = part:FindFirstChild(attachName .. "_0") or createAttachment(part, attachName .. "_0", Vector3.new(0, -1, 0))
        local att1 = part:FindFirstChild(attachName .. "_1") or createAttachment(part, attachName .. "_1", Vector3.new(0, 1, 0))
        
        local trail = Instance.new("Trail")
        trail.Name = "ArmTrail"
        trail.Attachment0 = att0
        trail.Attachment1 = att1
        trail.Lifetime = 0.3
        trail.MinLength = 0.1
        trail.LightEmission = 0.8
        trail.Color = ColorSequence.new({
            ColorSequenceKeypoint.new(0, CONFIG.MECHA_ACCENT_COLOR),
            ColorSequenceKeypoint.new(1, Color3.fromRGB(0, 50, 150)),
        })
        trail.Transparency = NumberSequence.new({
            NumberSequenceKeypoint.new(0, 0.3),
            NumberSequenceKeypoint.new(1, 1),
        })
        trail.Enabled = false
        trail.Parent = part
        
        return trail
    end
    
    local leftTrail = addTrailToPart("LeftLowerArm", "TrailAttach")
    local rightTrail = addTrailToPart("RightLowerArm", "TrailAttach")
    
    -- Ativar trails quando se move rápido
    RunService.Heartbeat:Connect(function()
        if SystemState.currentForm ~= "Mecha" then return end
        
        local mechaHumanoid = mecha:FindFirstChildOfClass("Humanoid")
        if not mechaHumanoid then return end
        
        local isRunning = mechaHumanoid.WalkSpeed > CONFIG.MECHA_WALK_SPEED * 1.2
        
        if leftTrail then leftTrail.Enabled = isRunning end
        if rightTrail then rightTrail.Enabled = isRunning end
    end)
end

-- ============================================================================
-- SEÇÃO 19: INICIALIZAÇÃO
-- ============================================================================

--- Função de inicialização principal
local function initialize()
    -- Aguardar o personagem estar pronto
    if not character or not humanoid then
        player.CharacterAdded:Wait()
        character = player.Character
        humanoid = character:WaitForChild("Humanoid")
        humanoidRootPart = character:WaitForChild("HumanoidRootPart")
    end
    
    -- Criar UI
    createTransformUI()
    
    -- Criar botões mobile
    createMobileButtons()
    
    -- Configurar velocidades iniciais
    humanoid.WalkSpeed = CONFIG.HUMANOID_WALK_SPEED
    humanoid.JumpPower = CONFIG.HUMANOID_JUMP_POWER
    
    -- Pré-gerar modelos (opcional - pode ser lazy-loaded)
    -- SystemState.mechaModel = generateMechaModel()
    -- SystemState.vehicleModel = generateVehicleModel()
    
    -- Criar trails (após o mecha existir)
    task.delay(2, function()
        if SystemState.mechaModel then
            createArmTrails()
        end
    end)
    
    print("[MechaSystem] Sistema de Transformação inicializado com sucesso!")
    print("[MechaSystem] Controles: T = Transformar | R = Reverter")
    print("[MechaSystem] Mobile: Use os botões na tela")
end

-- Iniciar o sistema
initialize()

-- ============================================================================
-- SEÇÃO 20: LIMPEZA E DESTRUIÇÃO
-- ============================================================================

--- Limpa todos os recursos quando o personagem morre ou é resetado
player.CharacterRemoving:Connect(function()
    -- Destruir modelos
    if SystemState.mechaModel then
        SystemState.mechaModel:Destroy()
        SystemState.mechaModel = nil
    end
    if SystemState.vehicleModel then
        SystemState.vehicleModel:Destroy()
        SystemState.vehicleModel = nil
    end
    
    -- Resetar estado
    SystemState.currentForm = "Humanoid"
    SystemState.isTransforming = false
    SystemState.originalProperties = {}
    
    -- Desconectar conexões
    for _, connection in ipairs(SystemState.connections) do
        if connection then
            connection:Disconnect()
        end
    end
    SystemState.connections = {}
end)

-- Reconectar quando novo personagem spawnar
player.CharacterAdded:Connect(function(char)
    character = char
    humanoid = char:WaitForChild("Humanoid")
    humanoidRootPart = char:WaitForChild("HumanoidRootPart")
    
    -- Re-inicializar
    task.wait(1)
    initialize()
end)

--[[
================================================================================
    FIM DO SCRIPT
    
    INSTRUÇÕES DE CONFIGURAÇÃO:
    
    1. Crie a estrutura de pastas no ReplicatedStorage conforme descrito no topo
    2. Coloque este LocalScript em StarterPlayer > StarterPlayerScripts
    3. Crie os RemoteEvents dentro de MechaSystem > RemoteEvents
    4. Substitua os IDs de som (rbxassetid://...) por sons reais da sua escolha
    5. Para animações pré-gravadas, adicione Animation objects com IDs válidos
    
    CONTROLES:
    - Teclado: T = Transformar (ciclo: Humano → Mecha → Veículo)
    - Teclado: R = Reverter para Humano
    - Teclado: WASD = Mover/Dirigir
    - Mobile: Botões na tela para transformar e controlar veículo
    
    CUSTOMIZAÇÃO:
    - Edite a tabela CONFIG no topo do script para ajustar velocidades, cores, etc.
    - Adicione Meshes personalizados às partes do mecha para melhor visual
    - Substitua as partes procedurais por modelos importados do Blender
    
    OTIMIZAÇÕES IMPLEMENTADAS:
    - NetworkOwnership definido para o jogador local
    - Instances criados uma vez e reutilizados
    - pcall em operações críticas
    - Cleanup automático ao resetar
    - Suporte completo a mobile/touch
================================================================================
]]
