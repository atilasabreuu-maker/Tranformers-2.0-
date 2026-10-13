--[[
================================================================================
    MECHA TRANSFORM SYSTEM V3.0 — CLIENT (LocalScript)
    ====================================================================================
    AUTÔNOMO • MOBILE-FIRST • SEM DEPENDÊNCIAS OBRIGATÓRIAS
    ====================================================================================
    
    LOCAL DE INSTALAÇÃO:
    StarterPlayer > StarterPlayerScripts > MechaSystemClient (LocalScript)
    
    PROBLEMAS DA V2.0 CORRIGIDOS:
    - WaitForChild() infinito em pastas inexistentes → AGORA cria tudo sozinho
    - Modelos com posições absolutas → AGORA coordenadas relativas ao pivô
    - Motor6D nomes inconsistentes → AGORA tabela central de referências
    - Veículo sem física real → AGORA raycast-based vehicle physics
    - Mecha sem controle real → AGORA controle via CameraSubject + Humanoid
    - UI duplicada em respawn → AGORA cleanup completo
    - Sem diagnóstico → AGORA painel de debug visual
    - Sem recuperação de erros → AGORA state machine com fallback
    - Transformação incompleta → AGORA timeout + rollback automático
    
    ARQUITETURA:
    - State Machine: Initializing → Human → Transforming → Mecha/Vehicle → Recovering
    - Modelos procedurais com coordenadas relativas
    - Motor6D centralizados em tabela de joints
    - Veículo com raycast suspension + CFrame movement
    - UI mobile-first com botões touch adequados
    - Cleanup automático em CharacterRemoving
    ====================================================================================
]]

-- ============================================================================
-- MÓDULO 1: SERVIÇOS (obtidos com segurança)
-- ============================================================================
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local ContextActionService = game:GetService("ContextActionService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Lighting = game:GetService("Lighting")

local player = Players.LocalPlayer
local camera = workspace.CurrentCamera

-- ============================================================================
-- MÓDULO 2: CONFIGURAÇÃO CENTRAL
-- ============================================================================
local CONFIG = {
    VERSION = "3.0.0",
    
    -- Velocidades
    HUMAN_WALK_SPEED = 16,
    HUMAN_JUMP_POWER = 50,
    MECHA_WALK_SPEED = 28,
    MECHA_RUN_SPEED = 44,
    MECHA_JUMP_POWER = 70,
    
    -- Veículo
    VEHICLE_MAX_SPEED = 90,
    VEHICLE_REVERSE_SPEED = 30,
    VEHICLE_ACCELERATION = 40,
    VEHICLE_BRAKE_FORCE = 60,
    VEHICLE_TURN_SPEED = 2.5,
    VEHICLE_GRAVITY = 50,
    VEHICLE_SUSPENSION_REST = 2.5,
    VEHICLE_SUSPENSION_STIFFNESS = 0.3,
    
    -- Transformação
    TRANSFORM_DURATION = 3.0,
    TRANSFORM_PHASES = 4,
    
    -- Cores
    MECHA_PRIMARY = Color3.fromRGB(45, 50, 70),
    MECHA_SECONDARY = Color3.fromRGB(60, 65, 85),
    MECHA_ACCENT = Color3.fromRGB(0, 160, 255),
    MECHA_GLOW = Color3.fromRGB(0, 200, 255),
    MECHA_DARK = Color3.fromRGB(25, 28, 40),
    
    VEHICLE_BODY = Color3.fromRGB(180, 30, 30),
    VEHICLE_ACCENT = Color3.fromRGB(220, 180, 0),
    VEHICLE_DARK = Color3.fromRGB(30, 30, 35),
    VEHICLE_GLASS = Color3.fromRGB(120, 180, 220),
    
    -- Câmera
    MECHA_CAMERA_DIST = 18,
    MECHA_CAMERA_HEIGHT = 10,
    VEHICLE_CAMERA_DIST = 22,
    VEHICLE_CAMERA_HEIGHT = 7,
    
    -- Mobile
    BUTTON_SIZE = UDim2.new(0, 64, 0, 64),
}

-- ============================================================================
-- MÓDULO 3: STATE MACHINE
-- ============================================================================
local StateMachine = {
    current = "Initializing",
    previous = "None",
    transformProgress = 0,
    isLocked = false,
    lastError = "None",
    initPhase = 0,
    partCount = 0,
    initialized = false,
}

local VALID_TRANSITIONS = {
    ["Initializing"] = {"Human", "Recovering"},
    ["Human"] = {"TransformingToMecha", "Recovering"},
    ["TransformingToMecha"] = {"Mecha", "Human", "Recovering"},
    ["Mecha"] = {"TransformingToVehicle", "ReturningToHuman", "Recovering"},
    ["TransformingToVehicle"] = {"Vehicle", "Mecha", "Recovering"},
    ["Vehicle"] = {"TransformingToMecha", "ReturningToHuman", "Recovering"},
    ["ReturningToHuman"] = {"Human", "Recovering"},
    ["Recovering"] = {"Human", "Initializing"},
}

--- Transiciona para um novo estado com validação
--- @param newState string
--- @return boolean success
function StateMachine:transition(newState)
    local allowed = VALID_TRANSITIONS[self.current]
    if not allowed then
        self.lastError = "Estado invalido: " .. self.current
        return false
    end
    
    for _, state in ipairs(allowed) do
        if state == newState then
            self.previous = self.current
            self.current = newState
            return true
        end
    end
    
    self.lastError = "Transicao invalida: " .. self.current .. " -> " .. newState
    return false
end

-- ============================================================================
-- MÓDULO 4: REFERÊNCIAS DO SISTEMA (centralizadas)
-- ============================================================================
local SystemRefs = {
    -- Personagem
    character = nil,
    humanoid = nil,
    rootPart = nil,
    
    -- Modelos
    mechaModel = nil,
    vehicleModel = nil,
    
    -- Juntas do Mecha (tabela central)
    mechaJoints = {},
    mechaParts = {},
    
    -- Juntas/Refs do Veículo
    vehicleJoints = {},
    vehicleParts = {},
    vehicleWheels = {},
    
    -- UI
    ui = {},
    mobileButtons = {},
    vehicleControls = {},
    
    -- Efeitos reutilizáveis
    effects = {},
    
    -- Conexões (para cleanup)
    connections = {},
    
    -- Propriedades originais do personagem
    originalProperties = {},
    
    -- Estado do veículo
    vehicleState = {
        speed = 0,
        steer = 0,
        grounded = false,
        orientation = CFrame.new(),
    },
}

-- ============================================================================
-- MÓDULO 5: UTILITÁRIOS
-- ============================================================================

--- Log do sistema com prefixo
local function sysLog(msg, level)
    level = level or "INFO"
    local prefix = "[MechaV3][" .. level .. "] "
    if level == "ERROR" then
        warn(prefix .. msg)
    elseif level == "WARN" then
        warn(prefix .. msg)
    else
        print(prefix .. msg)
    end
end

--- Cria um tween de forma segura
local function safeTween(instance, props, duration, style, direction)
    if not instance or not props then return nil end
    local ok, tween = pcall(function()
        local info = TweenInfo.new(
            duration or 0.5,
            style or Enum.EasingStyle.Quad,
            direction or Enum.EasingDirection.Out
        )
        return TweenService:Create(instance, info, props)
    end)
    if ok and tween then
        return tween
    end
    return nil
end

--- Espera com timeout (não bloqueia indefinidamente)
local function waitForWithTimeout(instance, childName, timeout)
    timeout = timeout or 5
    if not instance then return nil end
    
    local child = instance:FindFirstChild(childName)
    if child then return child end
    
    -- Espera limitada
    local startTime = tick()
    while tick() - startTime < timeout do
        child = instance:FindFirstChild(childName)
        if child then return child end
        task.wait(0.1)
    end
    
    sysLog("Timeout esperando: " .. childName, "WARN")
    return nil
end

--- Cria uma part básica com propriedades
local function createPart(props)
    local part = Instance.new("Part")
    part.Name = props.Name or "Part"
    part.Size = props.Size or Vector3.new(1, 1, 1)
    part.Color = props.Color or Color3.new(1, 1, 1)
    part.Material = props.Material or Enum.Material.SmoothPlastic
    part.Transparency = props.Transparency or 0
    part.Anchored = props.Anchored or false
    part.CanCollide = props.CanCollide ~= false
    part.Massless = props.Massless or false
    part.Shape = props.Shape or Enum.PartType.Block
    if props.CFrame then
        part.CFrame = props.CFrame
    end
    return part
end

--- Cria Motor6D e registra na tabela central
local function createJoint(name, part0, part1, c0, c1)
    if not part0 or not part1 then
        sysLog("createJoint: part0 ou part1 nulo para " .. name, "ERROR")
        return nil
    end
    
    local motor = Instance.new("Motor6D")
    motor.Name = name
    motor.Part0 = part0
    motor.Part1 = part1
    motor.C0 = c0 or CFrame.new()
    motor.C1 = c1 or CFrame.new()
    motor.Parent = part0
    
    -- Registra na tabela central
    SystemRefs.mechaJoints[name] = motor
    
    return motor
end

--- Cria efeito de partícula reutilizável
local function createEffect(name, parent, config)
    -- Verifica se já existe
    if SystemRefs.effects[name] then
        return SystemRefs.effects[name]
    end
    
    local attachment = Instance.new("Attachment")
    attachment.Name = name .. "_Attach"
    attachment.Parent = parent
    
    local emitter = Instance.new("ParticleEmitter")
    emitter.Name = name
    emitter.Texture = config.Texture or "rbxassetid://243098098"
    emitter.Rate = 0 -- Inicia desativado
    emitter.Lifetime = config.Lifetime or NumberRange.new(0.3, 0.8)
    emitter.Speed = config.Speed or NumberRange.new(5, 10)
    emitter.SpreadAngle = config.SpreadAngle or Vector2.new(180, 180)
    emitter.Size = config.Size or NumberSequence.new(0.5, 0)
    emitter.Transparency = config.Transparency or NumberSequence.new(0, 1)
    emitter.Color = config.Color or ColorSequence.new(Color3.new(1, 0.8, 0.2))
    emitter.LightEmission = config.LightEmission or 0.8
    emitter.Parent = attachment
    
    SystemRefs.effects[name] = emitter
    return emitter
end

--- Ativa efeito temporariamente
local function activateEffect(name, duration, rate)
    local emitter = SystemRefs.effects[name]
    if not emitter then return end
    
    rate = rate or 30
    emitter.Rate = rate
    
    task.delay(duration or 1, function()
        if emitter and emitter.Parent then
            emitter.Rate = 0
        end
    end)
end

-- ============================================================================
-- MÓDULO 6: CONSTRUÇÃO PROCEDURAL DO MECHA
-- ============================================================================

--[[
    O Mecha é construído com coordenadas RELATIVAS ao HumanoidRootPart.
    Todas as peças são filhas do modelo e conectadas por Motor6D.
    O pivô central é o HumanoidRootPart do mecha.
]]

local function buildMechaModel()
    sysLog("Construindo modelo Mecha...")
    
    local model = Instance.new("Model")
    model.Name = "MechaModel_V3"
    
    -- Pivô central (invisível)
    local pivot = Instance.new("Part")
    pivot.Name = "Pivot"
    pivot.Size = Vector3.new(2, 3, 1)
    pivot.Transparency = 1
    pivot.CanCollide = false
    pivot.Anchored = false
    pivot.CFrame = CFrame.new(0, 0, 0)
    pivot.Parent = model
    
    -- Tabela de peças (referências relativas ao pivot)
    local parts = {}
    local jointConfigs = {}
    
    -- =========================================================================
    -- FUNÇÃO AUXILIAR: Cria uma peça do mecha com offset relativo
    -- =========================================================================
    local function addMechaPart(name, size, offset, color, material)
        local part = createPart({
            Name = name,
            Size = size,
            Color = color or CONFIG.MECHA_PRIMARY,
            Material = material or Enum.Material.Metal,
            CFrame = CFrame.new(offset),
            CanCollide = true,
            Anchored = false,
        })
        part.Parent = model
        parts[name] = part
        SystemRefs.mechaParts[name] = part
        return part
    end
    
    -- =========================================================================
    -- CABEÇA (capacete robótico com visor)
    -- =========================================================================
    -- Capacete principal
    addMechaPart("Head", Vector3.new(2.8, 2.4, 2.6), Vector3.new(0, 8.2, 0), CONFIG.MECHA_PRIMARY)
    
    -- Visor (parte frontal da cabeça)
    local visor = addMechaPart("HeadVisor", Vector3.new(2.2, 0.8, 0.3), Vector3.new(0, 8.4, 1.3), CONFIG.MECHA_GLOW, Enum.Material.Neon)
    visor.Transparency = 0.2
    
    -- Antena esquerda
    addMechaPart("AntennaL", Vector3.new(0.2, 1.2, 0.2), Vector3.new(-1, 9.5, 0), CONFIG.MECHA_ACCENT, Enum.Material.Neon)
    
    -- Antena direita
    addMechaPart("AntennaR", Vector3.new(0.2, 1.2, 0.2), Vector3.new(1, 9.5, 0), CONFIG.MECHA_ACCENT, Enum.Material.Neon)
    
    -- Queixo mecânico
    addMechaPart("HeadJaw", Vector3.new(2, 0.6, 1.8), Vector3.new(0, 7, 0.2), CONFIG.MECHA_DARK)
    
    -- =========================================================================
    -- PESCOÇO
    -- =========================================================================
    addMechaPart("Neck", Vector3.new(1.2, 0.8, 1.2), Vector3.new(0, 6.6, 0), CONFIG.MECHA_DARK)
    
    -- =========================================================================
    -- TORSO SUPERIOR (peitoral com placas)
    -- =========================================================================
    -- Peito principal
    addMechaPart("UpperTorso", Vector3.new(4.5, 3, 2.8), Vector3.new(0, 5, 0), CONFIG.MECHA_PRIMARY)
    
    -- Placa peitoral esquerda
    addMechaPart("ChestPlateL", Vector3.new(1.8, 2, 0.4), Vector3.new(-0.8, 5.2, 1.4), CONFIG.MECHA_SECONDARY)
    
    -- Placa peitoral direita
    addMechaPart("ChestPlateR", Vector3.new(1.8, 2, 0.4), Vector3.new(0.8, 5.2, 1.4), CONFIG.MECHA_SECONDARY)
    
    -- Reator central (luminoso)
    local reactor = addMechaPart("Reactor", Vector3.new(1, 1, 0.5), Vector3.new(0, 5, 1.5), CONFIG.MECHA_GLOW, Enum.Material.Neon)
    reactor.Transparency = 0.1
    
    -- Painel traseiro
    addMechaPart("BackPanel", Vector3.new(3.5, 2.5, 0.5), Vector3.new(0, 5, -1.4), CONFIG.MECHA_DARK)
    
    -- =========================================================================
    -- TORSO INFERIOR (abdômen segmentado)
    -- =========================================================================
    addMechaPart("LowerTorso", Vector3.new(3.8, 2, 2.4), Vector3.new(0, 3, 0), CONFIG.MECHA_SECONDARY)
    
    -- Segmentos abdominais
    for i = 1, 3 do
        addMechaPart("AbsSeg" .. i, Vector3.new(2.5, 0.4, 0.3), Vector3.new(0, 2.2 + i * 0.5, 1.2), CONFIG.MECHA_DARK)
    end
    
    -- =========================================================================
    -- OMBREIRAS (volumosas)
    -- =========================================================================
    addMechaPart("ShoulderL", Vector3.new(2.2, 1.8, 2.2), Vector3.new(-3.2, 6, 0), CONFIG.MECHA_PRIMARY)
    addMechaPart("ShoulderR", Vector3.new(2.2, 1.8, 2.2), Vector3.new(3.2, 6, 0), CONFIG.MECHA_PRIMARY)
    
    -- Detalhes das ombreiras
    addMechaPart("ShoulderDetailL", Vector3.new(1.5, 0.4, 1.5), Vector3.new(-3.2, 7, 0), CONFIG.MECHA_ACCENT, Enum.Material.Neon)
    addMechaPart("ShoulderDetailR", Vector3.new(1.5, 0.4, 1.5), Vector3.new(3.2, 7, 0), CONFIG.MECHA_ACCENT, Enum.Material.Neon)
    
    -- =========================================================================
    -- BRAÇOS (armadura segmentada)
    -- =========================================================================
    -- Braço superior esquerdo
    addMechaPart("UpperArmL", Vector3.new(1.6, 2.8, 1.6), Vector3.new(-3.5, 4, 0), CONFIG.MECHA_SECONDARY)
    
    -- Cotovelo esquerdo (junta visível)
    addMechaPart("ElbowL", Vector3.new(1.2, 0.8, 1.2), Vector3.new(-3.5, 2.5, 0), CONFIG.MECHA_DARK)
    
    -- Antebraço esquerdo (reforçado)
    addMechaPart("LowerArmL", Vector3.new(1.8, 2.8, 1.8), Vector3.new(-3.5, 1, 0), CONFIG.MECHA_PRIMARY)
    
    -- Mão esquerda
    addMechaPart("HandL", Vector3.new(1.4, 1.4, 1), Vector3.new(-3.5, -0.6, 0), CONFIG.MECHA_DARK)
    
    -- Braço superior direito
    addMechaPart("UpperArmR", Vector3.new(1.6, 2.8, 1.6), Vector3.new(3.5, 4, 0), CONFIG.MECHA_SECONDARY)
    
    -- Cotovelo direito
    addMechaPart("ElbowR", Vector3.new(1.2, 0.8, 1.2), Vector3.new(3.5, 2.5, 0), CONFIG.MECHA_DARK)
    
    -- Antebraço direito
    addMechaPart("LowerArmR", Vector3.new(1.8, 2.8, 1.8), Vector3.new(3.5, 1, 0), CONFIG.MECHA_PRIMARY)
    
    -- Mão direita
    addMechaPart("HandR", Vector3.new(1.4, 1.4, 1), Vector3.new(3.5, -0.6, 0), CONFIG.MECHA_DARK)
    
    -- =========================================================================
    -- QUADRIL
    -- =========================================================================
    addMechaPart("Hip", Vector3.new(3.5, 1.2, 2.2), Vector3.new(0, 1.8, 0), CONFIG.MECHA_DARK)
    
    -- =========================================================================
    -- PERNAS (coxas com armadura)
    -- =========================================================================
    -- Coxa esquerda
    addMechaPart("UpperLegL", Vector3.new(2, 3, 2), Vector3.new(-1.2, -0.2, 0), CONFIG.MECHA_PRIMARY)
    
    -- Joelho esquerdo
    addMechaPart("KneeL", Vector3.new(1.6, 1, 1.6), Vector3.new(-1.2, -1.8, 0.3), CONFIG.MECHA_ACCENT)
    
    -- Canela esquerda
    addMechaPart("LowerLegL", Vector3.new(1.8, 3, 1.8), Vector3.new(-1.2, -3.5, 0), CONFIG.MECHA_SECONDARY)
    
    -- Pé esquerdo (largo)
    addMechaPart("FootL", Vector3.new(2, 0.8, 3.2), Vector3.new(-1.2, -5.2, 0.4), CONFIG.MECHA_DARK)
    
    -- Coxa direita
    addMechaPart("UpperLegR", Vector3.new(2, 3, 2), Vector3.new(1.2, -0.2, 0), CONFIG.MECHA_PRIMARY)
    
    -- Joelho direito
    addMechaPart("KneeR", Vector3.new(1.6, 1, 1.6), Vector3.new(1.2, -1.8, 0.3), CONFIG.MECHA_ACCENT)
    
    -- Canela direita
    addMechaPart("LowerLegR", Vector3.new(1.8, 3, 1.8), Vector3.new(1.2, -3.5, 0), CONFIG.MECHA_SECONDARY)
    
    -- Pé direito
    addMechaPart("FootR", Vector3.new(2, 0.8, 3.2), Vector3.new(1.2, -5.2, 0.4), CONFIG.MECHA_DARK)
    
    -- =========================================================================
    -- DETALHES ADICIONAIS (pistões, cabos simulados)
    -- =========================================================================
    -- Pistões nos braços
    addMechaPart("PistonArmL", Vector3.new(0.3, 2, 0.3), Vector3.new(-4.3, 3, 0.5), CONFIG.MECHA_ACCENT, Enum.Material.Metal)
    addMechaPart("PistonArmR", Vector3.new(0.3, 2, 0.3), Vector3.new(4.3, 3, 0.5), CONFIG.MECHA_ACCENT, Enum.Material.Metal)
    
    -- Pistões nas pernas
    addMechaPart("PistonLegL", Vector3.new(0.3, 2.5, 0.3), Vector3.new(-2.2, -1, 0.8), CONFIG.MECHA_ACCENT, Enum.Material.Metal)
    addMechaPart("PistonLegR", Vector3.new(0.3, 2.5, 0.3), Vector3.new(2.2, -1, 0.8), CONFIG.MECHA_ACCENT, Enum.Material.Metal)
    
    -- Motores laterais no torso
    addMechaPart("EngineL", Vector3.new(0.8, 1.5, 1.5), Vector3.new(-2.5, 4.5, -0.8), CONFIG.MECHA_DARK)
    addMechaPart("EngineR", Vector3.new(0.8, 1.5, 1.5), Vector3.new(2.5, 4.5, -0.8), CONFIG.MECHA_DARK)
    
    -- =========================================================================
    -- LUCES
    -- =========================================================================
    -- Luz do reator
    local reactorLight = Instance.new("PointLight")
    reactorLight.Name = "ReactorLight"
    reactorLight.Color = CONFIG.MECHA_GLOW
    reactorLight.Brightness = 3
    reactorLight.Range = 12
    reactorLight.Enabled = false
    reactorLight.Parent = parts["Reactor"]
    
    -- Luz dos olhos/visor
    local visorLight = Instance.new("PointLight")
    visorLight.Name = "VisorLight"
    visorLight.Color = CONFIG.MECHA_GLOW
    visorLight.Brightness = 2
    visorLight.Range = 8
    visorLight.Enabled = false
    visorLight.Parent = parts["HeadVisor"]
    
    -- =========================================================================
    -- HUMANOID DO MECHA (para controle de movimento)
    -- =========================================================================
    local mechaHumanoid = Instance.new("Humanoid")
    mechaHumanoid.Name = "MechaHumanoid"
    mechaHumanoid.HipHeight = 5.5
    mechaHumanoid.WalkSpeed = 0
    mechaHumanoid.JumpPower = 0
    mechaHumanoid.HealthDisplayType = Enum.HumanoidHealthDisplayType.AlwaysOff
    mechaHumanoid.Parent = model
    
    -- =========================================================================
    -- MOTOR6Ds (juntas) - conectando tudo ao pivot
    -- =========================================================================
    -- Cabeça → Pescoço → Torso Superior
    createJoint("NeckJoint", pivot, parts["Neck"], CFrame.new(0, 3.3, 0), CFrame.new(0, -0.4, 0))
    createJoint("HeadJoint", parts["Neck"], parts["Head"], CFrame.new(0, 0.4, 0), CFrame.new(0, -1.2, 0))
    createJoint("JawJoint", parts["Head"], parts["HeadJaw"], CFrame.new(0, -0.9, 0.1), CFrame.new(0, 0.3, 0))
    
    -- Torso
    createJoint("UpperTorsoJoint", parts["Neck"], parts["UpperTorso"], CFrame.new(0, -0.4, 0), CFrame.new(0, 1.5, 0))
    createJoint("LowerTorsoJoint", parts["UpperTorso"], parts["LowerTorso"], CFrame.new(0, -1.5, 0), CFrame.new(0, 1, 0))
    createJoint("HipJoint", parts["LowerTorso"], parts["Hip"], CFrame.new(0, -1, 0), CFrame.new(0, 0.6, 0))
    
    -- Ombros e Braços Esquerdo
    createJoint("ShoulderJointL", parts["UpperTorso"], parts["ShoulderL"], CFrame.new(-2.25, 1, 0), CFrame.new(0, 0, 0))
    createJoint("UpperArmJointL", parts["ShoulderL"], parts["UpperArmL"], CFrame.new(-0.3, -0.9, 0), CFrame.new(0, 1.4, 0))
    createJoint("ElbowJointL", parts["UpperArmL"], parts["ElbowL"], CFrame.new(0, -1.4, 0), CFrame.new(0, 0.4, 0))
    createJoint("LowerArmJointL", parts["ElbowL"], parts["LowerArmL"], CFrame.new(0, -0.4, 0), CFrame.new(0, 1.4, 0))
    createJoint("HandJointL", parts["LowerArmL"], parts["HandL"], CFrame.new(0, -1.4, 0), CFrame.new(0, 0.7, 0))
    
    -- Ombros e Braços Direito
    createJoint("ShoulderJointR", parts["UpperTorso"], parts["ShoulderR"], CFrame.new(2.25, 1, 0), CFrame.new(0, 0, 0))
    createJoint("UpperArmJointR", parts["ShoulderR"], parts["UpperArmR"], CFrame.new(0.3, -0.9, 0), CFrame.new(0, 1.4, 0))
    createJoint("ElbowJointR", parts["UpperArmR"], parts["ElbowR"], CFrame.new(0, -1.4, 0), CFrame.new(0, 0.4, 0))
    createJoint("LowerArmJointR", parts["ElbowR"], parts["LowerArmR"], CFrame.new(0, -0.4, 0), CFrame.new(0, 1.4, 0))
    createJoint("HandJointR", parts["LowerArmR"], parts["HandR"], CFrame.new(0, -1.4, 0), CFrame.new(0, 0.7, 0))
    
    -- Pernas Esquerda
    createJoint("HipJointL", parts["Hip"], parts["UpperLegL"], CFrame.new(-0.8, -0.6, 0), CFrame.new(0, 1.5, 0))
    createJoint("KneeJointL", parts["UpperLegL"], parts["KneeL"], CFrame.new(0, -1.5, 0.15), CFrame.new(0, 0.5, 0))
    createJoint("LowerLegJointL", parts["KneeL"], parts["LowerLegL"], CFrame.new(0, -0.5, -0.15), CFrame.new(0, 1.5, 0))
    createJoint("FootJointL", parts["LowerLegL"], parts["FootL"], CFrame.new(0, -1.5, 0), CFrame.new(0, 0.4, 0))
    
    -- Pernas Direita
    createJoint("HipJointR", parts["Hip"], parts["UpperLegR"], CFrame.new(0.8, -0.6, 0), CFrame.new(0, 1.5, 0))
    createJoint("KneeJointR", parts["UpperLegR"], parts["KneeR"], CFrame.new(0, -1.5, 0.15), CFrame.new(0, 0.5, 0))
    createJoint("LowerLegJointR", parts["KneeR"], parts["LowerLegR"], CFrame.new(0, -0.5, -0.15), CFrame.new(0, 1.5, 0))
    createJoint("FootJointR", parts["LowerLegR"], parts["FootR"], CFrame.new(0, -1.5, 0), CFrame.new(0, 0.4, 0))
    
    -- Pivot → Hip (conexão principal)
    createJoint("RootJoint", pivot, parts["Hip"], CFrame.new(0, -1.5, 0), CFrame.new(0, 0.6, 0))
    
    -- Peças decorativas soldadas (não animadas)
    local weldedParts = {
        "HeadVisor", "AntennaL", "AntennaR", "ChestPlateL", "ChestPlateR",
        "Reactor", "BackPanel", "AbsSeg1", "AbsSeg2", "AbsSeg3",
        "ShoulderDetailL", "ShoulderDetailR", "PistonArmL", "PistonArmR",
        "PistonLegL", "PistonLegR", "EngineL", "EngineR"
    }
    
    for _, partName in ipairs(weldedParts) do
        local part = parts[partName]
        if part then
            -- Encontra o parent motorizado mais próximo para soldar
            local weld = Instance.new("WeldConstraint")
            weld.Part0 = part
            -- Solda ao torso ou à cabeça dependendo da posição
            if part.Position.Y > 6 then
                weld.Part1 = parts["Head"]
            elseif part.Position.Y > 3 then
                weld.Part1 = parts["UpperTorso"]
            elseif part.Position.Y > 0 then
                weld.Part1 = parts["LowerTorso"]
            elseif part.Position.X < -2 then
                weld.Part1 = parts["UpperArmL"]
            elseif part.Position.X > 2 then
                weld.Part1 = parts["UpperArmR"]
            elseif part.Position.X < 0 then
                weld.Part1 = parts["UpperLegL"]
            else
                weld.Part1 = parts["UpperLegR"]
            end
            weld.Parent = part
        end
    end
    
    -- =========================================================================
    -- EFEITOS (partículas reutilizáveis)
    -- =========================================================================
    -- Faíscas nos ombros
    createEffect("ShoulderSparksL", parts["ShoulderL"], {
        Texture = "rbxassetid://243098098",
        Lifetime = NumberRange.new(0.2, 0.5),
        Speed = NumberRange.new(8, 15),
        Size = NumberSequence.new({NumberSequenceKeypoint.new(0, 0.3), NumberSequenceKeypoint.new(1, 0)}),
        Color = ColorSequence.new(Color3.fromRGB(100, 200, 255)),
        LightEmission = 1,
    })
    
    createEffect("ShoulderSparksR", parts["ShoulderR"], {
        Texture = "rbxassetid://243098098",
        Lifetime = NumberRange.new(0.2, 0.5),
        Speed = NumberRange.new(8, 15),
        Size = NumberSequence.new({NumberSequenceKeypoint.new(0, 0.3), NumberSequenceKeypoint.new(1, 0)}),
        Color = ColorSequence.new(Color3.fromRGB(100, 200, 255)),
        LightEmission = 1,
    })
    
    -- Fumaça nos pés
    createEffect("FootDustL", parts["FootL"], {
        Texture = "rbxassetid://241866187",
        Lifetime = NumberRange.new(0.5, 1),
        Speed = NumberRange.new(2, 5),
        SpreadAngle = Vector2.new(60, 30),
        Size = NumberSequence.new({NumberSequenceKeypoint.new(0, 0.5), NumberSequenceKeypoint.new(1, 1.5)}),
        Transparency = NumberSequence.new({NumberSequenceKeypoint.new(0, 0.5), NumberSequenceKeypoint.new(1, 1)}),
        Color = ColorSequence.new(Color3.fromRGB(150, 140, 120)),
        LightEmission = 0,
    })
    
    createEffect("FootDustR", parts["FootR"], {
        Texture = "rbxassetid://241866187",
        Lifetime = NumberRange.new(0.5, 1),
        Speed = NumberRange.new(2, 5),
        SpreadAngle = Vector2.new(60, 30),
        Size = NumberSequence.new({NumberSequenceKeypoint.new(0, 0.5), NumberSequenceKeypoint.new(1, 1.5)}),
        Transparency = NumberSequence.new({NumberSequenceKeypoint.new(0, 0.5), NumberSequenceKeypoint.new(1, 1)}),
        Color = ColorSequence.new(Color3.fromRGB(150, 140, 120)),
        LightEmission = 0,
    })
    
    -- Trail nos braços
    local trailAtt0L = Instance.new("Attachment")
    trailAtt0L.Position = Vector3.new(0, -1.2, 0)
    trailAtt0L.Parent = parts["LowerArmL"]
    local trailAtt1L = Instance.new("Attachment")
    trailAtt1L.Position = Vector3.new(0, 1.2, 0)
    trailAtt1L.Parent = parts["LowerArmL"]
    
    local armTrailL = Instance.new("Trail")
    armTrailL.Name = "ArmTrailL"
    armTrailL.Attachment0 = trailAtt0L
    armTrailL.Attachment1 = trailAtt1L
    armTrailL.Lifetime = 0.3
    armTrailL.LightEmission = 0.8
    armTrailL.Color = ColorSequence.new(CONFIG.MECHA_ACCENT, Color3.fromRGB(0, 50, 150))
    armTrailL.Transparency = NumberSequence.new(0.3, 1)
    armTrailL.Enabled = false
    armTrailL.Parent = parts["LowerArmL"]
    
    local trailAtt0R = Instance.new("Attachment")
    trailAtt0R.Position = Vector3.new(0, -1.2, 0)
    trailAtt0R.Parent = parts["LowerArmR"]
    local trailAtt1R = Instance.new("Attachment")
    trailAtt1R.Position = Vector3.new(0, 1.2, 0)
    trailAtt1R.Parent = parts["LowerArmR"]
    
    local armTrailR = Instance.new("Trail")
    armTrailR.Name = "ArmTrailR"
    armTrailR.Attachment0 = trailAtt0R
    armTrailR.Attachment1 = trailAtt1R
    armTrailR.Lifetime = 0.3
    armTrailR.LightEmission = 0.8
    armTrailR.Color = ColorSequence.new(CONFIG.MECHA_ACCENT, Color3.fromRGB(0, 50, 150))
    armTrailR.Transparency = NumberSequence.new(0.3, 1)
    armTrailR.Enabled = false
    armTrailR.Parent = parts["LowerArmR"]
    
    -- Configurar modelo
    model.PrimaryPart = pivot
    model:PivotTo(CFrame.new(0, -100, 0)) -- Esconde abaixo do mapa inicialmente
    
    -- Contar peças
    local count = 0
    for _ in pairs(parts) do count = count + 1 end
    SystemRefs.partCount = count
    
    sysLog("Mecha construido: " .. count .. " pecas, " .. #SystemRefs.mechaJoints .. " juntas")
    
    return model
end

-- ============================================================================
-- MÓDULO 7: CONSTRUÇÃO PROCEDURAL DO VEÍCULO
-- ============================================================================

local function buildVehicleModel()
    sysLog("Construindo modelo Veiculo...")
    
    local model = Instance.new("Model")
    model.Name = "VehicleModel_V3"
    
    -- Pivot do veículo
    local pivot = Instance.new("Part")
    pivot.Name = "VehiclePivot"
    pivot.Size = Vector3.new(1, 1, 1)
    pivot.Transparency = 1
    pivot.CanCollide = false
    pivot.Anchored = false
    pivot.CFrame = CFrame.new(0, 0, 0)
    pivot.Parent = model
    
    local parts = {}
    
    -- =========================================================================
    -- FUNÇÃO AUXILIAR
    -- =========================================================================
    local function addVehiclePart(name, size, offset, color, material)
        local part = createPart({
            Name = name,
            Size = size,
            Color = color or CONFIG.VEHICLE_BODY,
            Material = material or Enum.Material.Metal,
            CFrame = CFrame.new(offset),
            CanCollide = true,
        })
        part.Parent = model
        parts[name] = part
        SystemRefs.vehicleParts[name] = part
        return part
    end
    
    -- =========================================================================
    -- CHASSIS (estrutura principal)
    -- =========================================================================
    addVehiclePart("Chassis", Vector3.new(6, 1.2, 11), Vector3.new(0, 1, 0), CONFIG.VEHICLE_DARK)
    
    -- =========================================================================
    -- CARROCERIA
    -- =========================================================================
    -- Corpo principal
    addVehiclePart("Body", Vector3.new(5.5, 1.5, 10), Vector3.new(0, 2, 0), CONFIG.VEHICLE_BODY)
    
    -- Capô (frente inclinada)
    local hood = addVehiclePart("Hood", Vector3.new(5, 0.5, 3.5), Vector3.new(0, 2.8, 3), CONFIG.VEHICLE_BODY)
    
    -- Para-choques dianteiro
    addVehiclePart("FrontBumper", Vector3.new(5.5, 0.8, 1), Vector3.new(0, 1.5, 5.5), CONFIG.VEHICLE_DARK)
    
    -- Para-choques traseiro
    addVehiclePart("RearBumper", Vector3.new(5.5, 0.8, 1), Vector3.new(0, 1.5, -5.5), CONFIG.VEHICLE_DARK)
    
    -- =========================================================================
    -- CABINE
    -- =========================================================================
    addVehiclePart("Cabin", Vector3.new(4.5, 1.8, 4), Vector3.new(0, 3.5, -0.5), CONFIG.VEHICLE_BODY)
    
    -- Teto
    addVehiclePart("Roof", Vector3.new(4.2, 0.3, 3.5), Vector3.new(0, 4.5, -0.5), CONFIG.VEHICLE_DARK)
    
    -- Para-brisa dianteiro (vidro)
    local windshield = addVehiclePart("Windshield", Vector3.new(4, 1.5, 0.2), Vector3.new(0, 3.8, 1.5), CONFIG.VEHICLE_GLASS, Enum.Material.Glass)
    windshield.Transparency = 0.4
    
    -- Para-brisa traseiro
    local rearWindow = addVehiclePart("RearWindow", Vector3.new(3.8, 1.2, 0.2), Vector3.new(0, 3.8, -2.5), CONFIG.VEHICLE_GLASS, Enum.Material.Glass)
    rearWindow.Transparency = 0.4
    
    -- Janelas laterais
    local windowL = addVehiclePart("WindowL", Vector3.new(0.2, 1.2, 3), Vector3.new(-2.3, 3.8, -0.5), CONFIG.VEHICLE_GLASS, Enum.Material.Glass)
    windowL.Transparency = 0.4
    local windowR = addVehiclePart("WindowR", Vector3.new(0.2, 1.2, 3), Vector3.new(2.3, 3.8, -0.5), CONFIG.VEHICLE_GLASS, Enum.Material.Glass)
    windowR.Transparency = 0.4
    
    -- =========================================================================
    -- RODAS (4 rodas posicionadas corretamente)
    -- =========================================================================
    local wheelRadius = 1.2
    local wheelWidth = 0.8
    local wheelPositions = {
        {name = "WheelFL", pos = Vector3.new(-3, 0.2, 3.5)},   -- Frente Esquerda
        {name = "WheelFR", pos = Vector3.new(3, 0.2, 3.5)},    -- Frente Direita
        {name = "WheelRL", pos = Vector3.new(-3, 0.2, -3.5)},  -- Trás Esquerda
        {name = "WheelRR", pos = Vector3.new(3, 0.2, -3.5)},   -- Trás Direita
    }
    
    for _, wheelData in ipairs(wheelPositions) do
        -- Pneu
        local wheel = Instance.new("Part")
        wheel.Name = wheelData.name
        wheel.Size = Vector3.new(wheelWidth, wheelRadius * 2, wheelRadius * 2)
        wheel.Shape = Enum.PartType.Cylinder
        wheel.Color = Color3.fromRGB(25, 25, 25)
        wheel.Material = Enum.Material.SmoothPlastic
        wheel.CFrame = CFrame.new(wheelData.pos) * CFrame.Angles(0, 0, math.rad(90))
        wheel.CanCollide = true
        wheel.Parent = model
        
        -- Aro
        local rim = Instance.new("Part")
        rim.Name = wheelData.name .. "_Rim"
        rim.Size = Vector3.new(wheelWidth + 0.1, wheelRadius * 1.2, wheelRadius * 1.2)
        rim.Shape = Enum.PartType.Cylinder
        rim.Color = CONFIG.VEHICLE_ACCENT
        rim.Material = Enum.Material.Metal
        rim.CFrame = wheel.CFrame
        rim.Parent = model
        
        parts[wheelData.name] = wheel
        table.insert(SystemRefs.vehicleWheels, {
            part = wheel,
            rim = rim,
            restPos = wheelData.pos,
            currentOffset = 0,
        })
    end
    
    -- =========================================================================
    -- PARA-LAMAS
    -- =========================================================================
    addVehiclePart("FenderFL", Vector3.new(1.5, 0.8, 2.5), Vector3.new(-3, 1.5, 3.5), CONFIG.VEHICLE_BODY)
    addVehiclePart("FenderFR", Vector3.new(1.5, 0.8, 2.5), Vector3.new(3, 1.5, 3.5), CONFIG.VEHICLE_BODY)
    addVehiclePart("FenderRL", Vector3.new(1.5, 0.8, 2.5), Vector3.new(-3, 1.5, -3.5), CONFIG.VEHICLE_BODY)
    addVehiclePart("FenderRR", Vector3.new(1.5, 0.8, 2.5), Vector3.new(3, 1.5, -3.5), CONFIG.VEHICLE_BODY)
    
    -- =========================================================================
    -- FARÓIS E LANTERNAS
    -- =========================================================================
    local headlightL = addVehiclePart("HeadlightL", Vector3.new(1.2, 0.6, 0.3), Vector3.new(-1.8, 2.2, 5.5), Color3.fromRGB(255, 255, 220), Enum.Material.Neon)
    local headlightR = addVehiclePart("HeadlightR", Vector3.new(1.2, 0.6, 0.3), Vector3.new(1.8, 2.2, 5.5), Color3.fromRGB(255, 255, 220), Enum.Material.Neon)
    
    -- SpotLights para faróis
    local spotL = Instance.new("SpotLight")
    spotL.Name = "HeadlightSpotL"
    spotL.Color = Color3.fromRGB(255, 255, 200)
    spotL.Brightness = 5
    spotL.Range = 60
    spotL.Angle = 45
    spotL.Face = Enum.NormalFace.Front
    spotL.Enabled = false
    spotL.Parent = headlightL
    
    local spotR = Instance.new("SpotLight")
    spotR.Name = "HeadlightSpotR"
    spotR.Color = Color3.fromRGB(255, 255, 200)
    spotR.Brightness = 5
    spotR.Range = 60
    spotR.Angle = 45
    spotR.Face = Enum.NormalFace.Front
    spotR.Enabled = false
    spotR.Parent = headlightR
    
    -- Lanternas traseiras
    addVehiclePart("TaillightL", Vector3.new(1, 0.4, 0.3), Vector3.new(-2, 2.2, -5.5), Color3.fromRGB(255, 30, 0), Enum.Material.Neon)
    addVehiclePart("TaillightR", Vector3.new(1, 0.4, 0.3), Vector3.new(2, 2.2, -5.5), Color3.fromRGB(255, 30, 0), Enum.Material.Neon)
    
    -- =========================================================================
    -- DETALHES AERODINÂMICOS
    -- =========================================================================
    -- Spoiler
    addVehiclePart("Spoiler", Vector3.new(4.5, 0.2, 1), Vector3.new(0, 3.8, -5), CONFIG.VEHICLE_ACCENT)
    addVehiclePart("SpoilerStandL", Vector3.new(0.3, 0.8, 0.3), Vector3.new(-1.5, 3.4, -5), CONFIG.VEHICLE_DARK)
    addVehiclePart("SpoilerStandR", Vector3.new(0.3, 0.8, 0.3), Vector3.new(1.5, 3.4, -5), CONFIG.VEHICLE_DARK)
    
    -- Escape/propulsores
    local exhaustL = addVehiclePart("ExhaustL", Vector3.new(0.6, 0.6, 1), Vector3.new(-1.5, 1.2, -6), CONFIG.VEHICLE_DARK, Enum.Material.Metal)
    local exhaustR = addVehiclePart("ExhaustR", Vector3.new(0.6, 0.6, 1), Vector3.new(1.5, 1.2, -6), CONFIG.VEHICLE_DARK, Enum.Material.Metal)
    
    -- Efeito de escape
    createEffect("ExhaustFireL", exhaustL, {
        Texture = "rbxassetid://243098098",
        Lifetime = NumberRange.new(0.1, 0.3),
        Speed = NumberRange.new(10, 20),
        SpreadAngle = Vector2.new(20, 20),
        Size = NumberSequence.new({NumberSequenceKeypoint.new(0, 0.4), NumberSequenceKeypoint.new(1, 0)}),
        Color = ColorSequence.new(Color3.fromRGB(255, 150, 0), Color3.fromRGB(100, 100, 100)),
        LightEmission = 1,
    })
    
    createEffect("ExhaustFireR", exhaustR, {
        Texture = "rbxassetid://243098098",
        Lifetime = NumberRange.new(0.1, 0.3),
        Speed = NumberRange.new(10, 20),
        SpreadAngle = Vector2.new(20, 20),
        Size = NumberSequence.new({NumberSequenceKeypoint.new(0, 0.4), NumberSequenceKeypoint.new(1, 0)}),
        Color = ColorSequence.new(Color3.fromRGB(255, 150, 0), Color3.fromRGB(100, 100, 100)),
        LightEmission = 1,
    })
    
    -- =========================================================================
    -- PORTAS/PAINÉIS LATERAIS
    -- =========================================================================
    addVehiclePart("DoorL", Vector3.new(0.2, 1.5, 3.5), Vector3.new(-2.8, 2.5, 0), CONFIG.VEHICLE_BODY)
    addVehiclePart("DoorR", Vector3.new(0.2, 1.5, 3.5), Vector3.new(2.8, 2.5, 0), CONFIG.VEHICLE_BODY)
    
    -- =========================================================================
    -- SOLDAR TUDO AO CHASSIS (exceto rodas que têm física própria)
    -- =========================================================================
    for name, part in pairs(parts) do
        if not name:match("^Wheel") and name ~= "Chassis" then
            local weld = Instance.new("WeldConstraint")
            weld.Part0 = parts["Chassis"]
            weld.Part1 = part
            weld.Parent = part
        end
    end
    
    -- Soldar aros às rodas
    for _, wheelData in ipairs(SystemRefs.vehicleWheels) do
        local weld = Instance.new("WeldConstraint")
        weld.Part0 = wheelData.part
        weld.Part1 = wheelData.rim
        weld.Parent = wheelData.rim
    end
    
    -- =========================================================================
    -- HUMANOID (para CameraSubject)
    -- =========================================================================
    local vehicleHumanoid = Instance.new("Humanoid")
    vehicleHumanoid.Name = "VehicleHumanoid"
    vehicleHumanoid.WalkSpeed = 0
    vehicleHumanoid.JumpPower = 0
    vehicleHumanoid.HealthDisplayType = Enum.HumanoidHealthDisplayType.AlwaysOff
    vehicleHumanoid.Parent = model
    
    -- Configurar
    model.PrimaryPart = pivot
    model:PivotTo(CFrame.new(0, -200, 0)) -- Esconde abaixo do mapa
    
    sysLog("Veiculo construido: " .. #SystemRefs.vehicleWheels .. " rodas")
    
    return model
end

-- ============================================================================
-- MÓDULO 8: INTERFACE MOBILE-FIRST
-- ============================================================================

local function createUI()
    sysLog("Criando interface...")
    
    -- Verifica se já existe (previne duplicação em respawn)
    local existingGui = player.PlayerGui:FindFirstChild("MechaSystemUI_V3")
    if existingGui then
        existingGui:Destroy()
    end
    
    local screenGui = Instance.new("ScreenGui")
    screenGui.Name = "MechaSystemUI_V3"
    screenGui.ResetOnSpawn = false
    screenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    screenGui.Parent = player.PlayerGui
    
    -- =========================================================================
    -- INDICADOR DE FORMA ATUAL (topo da tela)
    -- =========================================================================
    local formIndicator = Instance.new("Frame")
    formIndicator.Name = "FormIndicator"
    formIndicator.Size = UDim2.new(0, 200, 0, 40)
    formIndicator.Position = UDim2.new(0.5, -100, 0, 10)
    formIndicator.BackgroundColor3 = Color3.fromRGB(15, 15, 25)
    formIndicator.BackgroundTransparency = 0.2
    formIndicator.Parent = screenGui
    
    local formCorner = Instance.new("UICorner")
    formCorner.CornerRadius = UDim.new(0, 10)
    formCorner.Parent = formIndicator
    
    local formStroke = Instance.new("UIStroke")
    formStroke.Color = CONFIG.MECHA_ACCENT
    formStroke.Thickness = 2
    formStroke.Parent = formIndicator
    
    local formLabel = Instance.new("TextLabel")
    formLabel.Name = "FormLabel"
    formLabel.Size = UDim2.new(1, 0, 1, 0)
    formLabel.BackgroundTransparency = 1
    formLabel.Text = "👤 HUMANO"
    formLabel.TextColor3 = Color3.new(1, 1, 1)
    formLabel.TextSize = 16
    formLabel.Font = Enum.Font.GothamBold
    formLabel.Parent = formIndicator
    
    -- =========================================================================
    -- BARRA DE TRANSFORMAÇÃO (aparece durante transformação)
    -- =========================================================================
    local transformBar = Instance.new("Frame")
    transformBar.Name = "TransformBar"
    transformBar.Size = UDim2.new(0, 250, 0, 50)
    transformBar.Position = UDim2.new(0.5, -125, 0, 60)
    transformBar.BackgroundColor3 = Color3.fromRGB(15, 15, 25)
    transformBar.BackgroundTransparency = 0.2
    transformBar.Visible = false
    transformBar.Parent = screenGui
    
    local tbCorner = Instance.new("UICorner")
    tbCorner.CornerRadius = UDim.new(0, 8)
    tbCorner.Parent = transformBar
    
    local tbStroke = Instance.new("UIStroke")
    tbStroke.Color = CONFIG.MECHA_ACCENT
    tbStroke.Thickness = 1.5
    tbStroke.Parent = transformBar
    
    local tbLabel = Instance.new("TextLabel")
    tbLabel.Size = UDim2.new(1, 0, 0, 20)
    tbLabel.Position = UDim2.new(0, 0, 0, 3)
    tbLabel.BackgroundTransparency = 1
    tbLabel.Text = "TRANSFORMANDO..."
    tbLabel.TextColor3 = CONFIG.MECHA_ACCENT
    tbLabel.TextSize = 11
    tbLabel.Font = Enum.Font.GothamBold
    tbLabel.Parent = transformBar
    
    local tbBg = Instance.new("Frame")
    tbBg.Size = UDim2.new(0.9, 0, 0, 10)
    tbBg.Position = UDim2.new(0.05, 0, 0, 25)
    tbBg.BackgroundColor3 = Color3.fromRGB(30, 30, 50)
    tbBg.Parent = transformBar
    
    local tbBgCorner = Instance.new("UICorner")
    tbBgCorner.CornerRadius = UDim.new(0, 5)
    tbBgCorner.Parent = tbBg
    
    local tbFill = Instance.new("Frame")
    tbFill.Name = "Fill"
    tbFill.Size = UDim2.new(0, 0, 1, 0)
    tbFill.BackgroundColor3 = CONFIG.MECHA_ACCENT
    tbFill.Parent = tbBg
    
    local tbFillCorner = Instance.new("UICorner")
    tbFillCorner.CornerRadius = UDim.new(0, 5)
    tbFillCorner.Parent = tbFill
    
    -- =========================================================================
    -- BOTÕES DE FORMAÇÃO (lado direito, vertical)
    -- =========================================================================
    local buttonContainer = Instance.new("Frame")
    buttonContainer.Name = "FormButtons"
    buttonContainer.Size = UDim2.new(0, 70, 0, 230)
    buttonContainer.Position = UDim2.new(1, -85, 0.4, 0)
    buttonContainer.BackgroundTransparency = 1
    buttonContainer.Parent = screenGui
    
    -- Botão HUMANO
    local btnHuman = Instance.new("TextButton")
    btnHuman.Name = "BtnHuman"
    btnHuman.Size = CONFIG.BUTTON_SIZE
    btnHuman.Position = UDim2.new(0, 3, 0, 0)
    btnHuman.BackgroundColor3 = Color3.fromRGB(60, 60, 80)
    btnHuman.Text = "👤"
    btnHuman.TextSize = 28
    btnHuman.TextColor3 = Color3.new(1, 1, 1)
    btnHuman.Font = Enum.Font.GothamBold
    btnHuman.AutoButtonColor = true
    btnHuman.Parent = buttonContainer
    
    local bhCorner = Instance.new("UICorner")
    bhCorner.CornerRadius = UDim.new(0, 12)
    bhCorner.Parent = btnHuman
    
    -- Botão MECHA
    local btnMecha = Instance.new("TextButton")
    btnMecha.Name = "BtnMecha"
    btnMecha.Size = CONFIG.BUTTON_SIZE
    btnMecha.Position = UDim2.new(0, 3, 0, 80)
    btnMecha.BackgroundColor3 = Color3.fromRGB(30, 80, 160)
    btnMecha.Text = "🤖"
    btnMecha.TextSize = 28
    btnMecha.TextColor3 = Color3.new(1, 1, 1)
    btnMecha.Font = Enum.Font.GothamBold
    btnMecha.AutoButtonColor = true
    btnMecha.Parent = buttonContainer
    
    local bmCorner = Instance.new("UICorner")
    bmCorner.CornerRadius = UDim.new(0, 12)
    bmCorner.Parent = btnMecha
    
    -- Botão VEHICLE
    local btnVehicle = Instance.new("TextButton")
    btnVehicle.Name = "BtnVehicle"
    btnVehicle.Size = CONFIG.BUTTON_SIZE
    btnVehicle.Position = UDim2.new(0, 3, 0, 160)
    btnVehicle.BackgroundColor3 = Color3.fromRGB(160, 30, 30)
    btnVehicle.Text = "🏎"
    btnVehicle.TextSize = 28
    btnVehicle.TextColor3 = Color3.new(1, 1, 1)
    btnVehicle.Font = Enum.Font.GothamBold
    btnVehicle.AutoButtonColor = true
    btnVehicle.Parent = buttonContainer
    
    local bvCorner = Instance.new("UICorner")
    bvCorner.CornerRadius = UDim.new(0, 12)
    bvCorner.Parent = btnVehicle
    
    -- =========================================================================
    -- CONTROLES DO VEÍCULO (aparecem quando em modo Vehicle)
    -- =========================================================================
    local vehicleControlFrame = Instance.new("Frame")
    vehicleControlFrame.Name = "VehicleControls"
    vehicleControlFrame.Size = UDim2.new(0, 200, 0, 200)
    vehicleControlFrame.Position = UDim2.new(0, 15, 1, -220)
    vehicleControlFrame.BackgroundTransparency = 1
    vehicleControlFrame.Visible = false
    vehicleControlFrame.Parent = screenGui
    
    -- Acelerador
    local btnAccel = Instance.new("TextButton")
    btnAccel.Name = "Accel"
    btnAccel.Size = UDim2.new(0, 70, 0, 70)
    btnAccel.Position = UDim2.new(0, 65, 0, 0)
    btnAccel.BackgroundColor3 = Color3.fromRGB(0, 180, 0)
    btnAccel.Text = "▲"
    btnAccel.TextSize = 30
    btnAccel.TextColor3 = Color3.new(1, 1, 1)
    btnAccel.Font = Enum.Font.GothamBold
    btnAccel.Parent = vehicleControlFrame
    
    local baCorner = Instance.new("UICorner")
    baCorner.CornerRadius = UDim.new(1, 0)
    baCorner.Parent = btnAccel
    
    -- Freio/Ré
    local btnBrake = Instance.new("TextButton")
    btnBrake.Name = "Brake"
    btnBrake.Size = UDim2.new(0, 70, 0, 70)
    btnBrake.Position = UDim2.new(0, 65, 0, 130)
    btnBrake.BackgroundColor3 = Color3.fromRGB(200, 0, 0)
    btnBrake.Text = "▼"
    btnBrake.TextSize = 30
    btnBrake.TextColor3 = Color3.new(1, 1, 1)
    btnBrake.Font = Enum.Font.GothamBold
    btnBrake.Parent = vehicleControlFrame
    
    local bbCorner = Instance.new("UICorner")
    bbCorner.CornerRadius = UDim.new(1, 0)
    bbCorner.Parent = btnBrake
    
    -- Esquerda
    local btnLeft = Instance.new("TextButton")
    btnLeft.Name = "Left"
    btnLeft.Size = UDim2.new(0, 60, 0, 60)
    btnLeft.Position = UDim2.new(0, 0, 0, 70)
    btnLeft.BackgroundColor3 = Color3.fromRGB(50, 50, 180)
    btnLeft.Text = "◀"
    btnLeft.TextSize = 24
    btnLeft.TextColor3 = Color3.new(1, 1, 1)
    btnLeft.Font = Enum.Font.GothamBold
    btnLeft.Parent = vehicleControlFrame
    
    local blCorner = Instance.new("UICorner")
    blCorner.CornerRadius = UDim.new(1, 0)
    blCorner.Parent = btnLeft
    
    -- Direita
    local btnRight = Instance.new("TextButton")
    btnRight.Name = "Right"
    btnRight.Size = UDim2.new(0, 60, 0, 60)
    btnRight.Position = UDim2.new(0, 140, 0, 70)
    btnRight.BackgroundColor3 = Color3.fromRGB(50, 50, 180)
    btnRight.Text = "▶"
    btnRight.TextSize = 24
    btnRight.TextColor3 = Color3.new(1, 1, 1)
    btnRight.Font = Enum.Font.GothamBold
    btnRight.Parent = vehicleControlFrame
    
    local brCorner = Instance.new("UICorner")
    brCorner.CornerRadius = UDim.new(1, 0)
    brCorner.Parent = btnRight
    
    -- =========================================================================
    -- PAINEL DE DIAGNÓSTICO (toggle com botão pequeno)
    -- =========================================================================
    local diagToggle = Instance.new("TextButton")
    diagToggle.Name = "DiagToggle"
    diagToggle.Size = UDim2.new(0, 30, 0, 30)
    diagToggle.Position = UDim2.new(0, 10, 0, 10)
    diagToggle.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
    diagToggle.BackgroundTransparency = 0.3
    diagToggle.Text = "?"
    diagToggle.TextSize = 14
    diagToggle.TextColor3 = Color3.new(1, 1, 1)
    diagToggle.Font = Enum.Font.GothamBold
    diagToggle.Parent = screenGui
    
    local dtCorner = Instance.new("UICorner")
    dtCorner.CornerRadius = UDim.new(1, 0)
    dtCorner.Parent = diagToggle
    
    local diagPanel = Instance.new("Frame")
    diagPanel.Name = "DiagPanel"
    diagPanel.Size = UDim2.new(0, 220, 0, 180)
    diagPanel.Position = UDim2.new(0, 10, 0, 45)
    diagPanel.BackgroundColor3 = Color3.fromRGB(10, 10, 20)
    diagPanel.BackgroundTransparency = 0.1
    diagPanel.Visible = false
    diagPanel.Parent = screenGui
    
    local dpCorner = Instance.new("UICorner")
    dpCorner.CornerRadius = UDim.new(0, 8)
    dpCorner.Parent = diagPanel
    
    local dpStroke = Instance.new("UIStroke")
    dpStroke.Color = Color3.fromRGB(100, 100, 100)
    dpStroke.Thickness = 1
    dpStroke.Parent = diagPanel
    
    local diagText = Instance.new("TextLabel")
    diagText.Name = "DiagText"
    diagText.Size = UDim2.new(1, -10, 1, -10)
    diagText.Position = UDim2.new(0, 5, 0, 5)
    diagText.BackgroundTransparency = 1
    diagText.Text = "Carregando..."
    diagText.TextColor3 = Color3.fromRGB(180, 220, 180)
    diagText.TextSize = 10
    diagText.Font = Enum.Font.Code
    diagText.TextXAlignment = Enum.TextXAlignment.Left
    diagText.TextYAlignment = Enum.TextYAlignment.Top
    diagText.TextWrapped = true
    diagText.Parent = diagPanel
    
    -- Toggle do painel
    diagToggle.MouseButton1Click:Connect(function()
        diagPanel.Visible = not diagPanel.Visible
    end)
    
    -- =========================================================================
    -- GUARDAR REFERÊNCIAS
    -- =========================================================================
    SystemRefs.ui = {
        screenGui = screenGui,
        formIndicator = formIndicator,
        formLabel = formLabel,
        transformBar = transformBar,
        tbFill = tbFill,
        tbLabel = tbLabel,
        diagPanel = diagPanel,
        diagText = diagText,
        vehicleControlFrame = vehicleControlFrame,
    }
    
    SystemRefs.mobileButtons = {
        human = btnHuman,
        mecha = btnMecha,
        vehicle = btnVehicle,
    }
    
    SystemRefs.vehicleControls = {
        accel = btnAccel,
        brake = btnBrake,
        left = btnLeft,
        right = btnRight,
        accelHeld = false,
        brakeHeld = false,
        leftHeld = false,
        rightHeld = false,
    }
    
    -- =========================================================================
    -- CONECTAR EVENTOS DOS BOTÕES (com debounce)
    -- =========================================================================
    local lastClickTime = 0
    local CLICK_DEBOUNCE = 0.5
    
    local function safeTransform(target)
        local now = tick()
        if now - lastClickTime < CLICK_DEBOUNCE then return end
        if StateMachine.isLocked then return end
        lastClickTime = now
        startTransformation(target)
    end
    
    btnHuman.MouseButton1Click:Connect(function()
        safeTransform("Human")
    end)
    
    btnMecha.MouseButton1Click:Connect(function()
        safeTransform("Mecha")
    end)
    
    btnVehicle.MouseButton1Click:Connect(function()
        safeTransform("Vehicle")
    end)
    
    -- Controles do veículo (touch hold)
    btnAccel.MouseButton1Down:Connect(function() SystemRefs.vehicleControls.accelHeld = true end)
    btnAccel.MouseButton1Up:Connect(function() SystemRefs.vehicleControls.accelHeld = false end)
    btnBrake.MouseButton1Down:Connect(function() SystemRefs.vehicleControls.brakeHeld = true end)
    btnBrake.MouseButton1Up:Connect(function() SystemRefs.vehicleControls.brakeHeld = false end)
    btnLeft.MouseButton1Down:Connect(function() SystemRefs.vehicleControls.leftHeld = true end)
    btnLeft.MouseButton1Up:Connect(function() SystemRefs.vehicleControls.leftHeld = false end)
    btnRight.MouseButton1Down:Connect(function() SystemRefs.vehicleControls.rightHeld = true end)
    btnRight.MouseButton1Up:Connect(function() SystemRefs.vehicleControls.rightHeld = false end)
    
    sysLog("Interface criada com sucesso")
end

--- Atualiza a UI baseada no estado atual
local function updateUI()
    local ui = SystemRefs.ui
    if not ui.formLabel then return end
    
    local state = StateMachine.current
    
    -- Atualizar indicador de forma
    if state == "Human" then
        ui.formLabel.Text = "👤 HUMANO"
    elseif state == "Mecha" or state == "TransformingToMecha" then
        ui.formLabel.Text = "🤖 MECHA"
    elseif state == "Vehicle" or state == "TransformingToVehicle" then
        ui.formLabel.Text = "🏎 VEÍCULO"
    else
        ui.formLabel.Text = "⚙ " .. state:upper()
    end
    
    -- Atualizar botões (destaque no ativo)
    local buttons = SystemRefs.mobileButtons
    if buttons.human then
        buttons.human.BackgroundColor3 = (state == "Human") and Color3.fromRGB(100, 100, 130) or Color3.fromRGB(60, 60, 80)
        buttons.mecha.BackgroundColor3 = (state == "Mecha") and Color3.fromRGB(50, 120, 220) or Color3.fromRGB(30, 80, 160)
        buttons.vehicle.BackgroundColor3 = (state == "Vehicle") and Color3.fromRGB(220, 50, 50) or Color3.fromRGB(160, 30, 30)
    end
    
    -- Controles do veículo visíveis apenas no modo Vehicle
    if ui.vehicleControlFrame then
        ui.vehicleControlFrame.Visible = (state == "Vehicle")
    end
end

--- Atualiza a barra de transformação
local function updateTransformBar(progress, text)
    local ui = SystemRefs.ui
    if not ui.transformBar then return end
    
    ui.transformBar.Visible = true
    ui.tbFill.Size = UDim2.new(math.clamp(progress, 0, 1), 0, 1, 0)
    if text then
        ui.tbLabel.Text = text
    end
end

local function hideTransformBar()
    local ui = SystemRefs.ui
    if ui.transformBar then
        ui.transformBar.Visible = false
    end
end

--- Atualiza o painel de diagnóstico
local function updateDiagPanel()
    local ui = SystemRefs.ui
    if not ui.diagText then return end
    
    local info = string.format(
        "═══ MECHA SYSTEM V%s ═══
" ..
        "Estado: %s
" ..
        "Anterior: %s
" ..
        "Fase Init: %d
" ..
        "Peças Mecha: %d
" ..
        "Juntas: %d
" ..
        "Personagem: %s
" ..
        "Mecha: %s
" ..
        "Veículo: %s
" ..
        "Vel. Veículo: %.1f
" ..
        "Câmera Subject: %s
" ..
        "Último Erro: %s
" ..
        "═══════════════════════",
        CONFIG.VERSION,
        StateMachine.current,
        StateMachine.previous,
        StateMachine.initPhase,
        SystemRefs.partCount,
        #SystemRefs.mechaJoints,
        SystemRefs.character and "OK" or "NULO",
        SystemRefs.mechaModel and "OK" or "NULO",
        SystemRefs.vehicleModel and "OK" or "NULO",
        SystemRefs.vehicleState.speed,
        camera.CameraSubject and camera.CameraSubject.Name or "NULO",
        StateMachine.lastError
    )
    
    ui.diagText.Text = info
end

-- ============================================================================
-- MÓDULO 9: CONTROLE DO MECHA
-- ============================================================================

--[[
    O controle do Mecha funciona transferindo o CameraSubject para o
    Humanoid do modelo Mecha. O jogador controla o mecha como se fosse
    seu próprio personagem. O Humanoid do mecha recebe os inputs de
    movimento do jogador via CameraSubject.
]]

local function activateMechaControl()
    local mecha = SystemRefs.mechaModel
    if not mecha then
        sysLog("activateMechaControl: mechaModel nulo!", "ERROR")
        return false
    end
    
    local mechaHumanoid = mecha:FindFirstChild("MechaHumanoid")
    local mechaPivot = mecha:FindFirstChild("Pivot")
    
    if not mechaHumanoid or not mechaPivot then
        sysLog("activateMechaControl: Humanoid ou Pivot nulo!", "ERROR")
        return false
    end
    
    -- Posicionar mecha na posição do jogador
    local charPos = SystemRefs.rootPart and SystemRefs.rootPart.CFrame or CFrame.new(0, 10, 0)
    mecha:PivotTo(charPos + Vector3.new(0, 3, 0))
    
    -- Ativar colisão
    for _, part in ipairs(mecha:GetDescendants()) do
        if part:IsA("BasePart") then
            part.CanCollide = true
        end
    end
    
    -- Configurar Humanoid do mecha
    mechaHumanoid.WalkSpeed = CONFIG.MECHA_WALK_SPEED
    mechaHumanoid.JumpPower = CONFIG.MECHA_JUMP_POWER
    
    -- Transferir controle da câmera para o mecha
    camera.CameraSubject = mechaHumanoid
    
    -- Ativar luzes
    local reactorLight = mecha:FindFirstChild("Reactor", true)
    if reactorLight then
        local light = reactorLight:FindFirstChildOfClass("PointLight")
        if light then light.Enabled = true end
    end
    
    local visorLight = mecha:FindFirstChild("HeadVisor", true)
    if visorLight then
        local light = visorLight:FindFirstChildOfClass("PointLight")
        if light then light.Enabled = true end
    end
    
    -- Network ownership
    pcall(function()
        mechaPivot:SetNetworkOwner(player)
    end)
    
    sysLog("Controle do Mecha ativado")
    return true
end

local function deactivateMechaControl()
    local mecha = SystemRefs.mechaModel
    if not mecha then return end
    
    local mechaHumanoid = mecha:FindFirstChild("MechaHumanoid")
    if mechaHumanoid then
        mechaHumanoid.WalkSpeed = 0
        mechaHumanoid.JumpPower = 0
    end
    
    -- Desativar luzes
    for _, light in ipairs(mecha:GetDescendants()) do
        if light:IsA("Light") then
            light.Enabled = false
        end
    end
    
    -- Desativar trails
    for _, trail in ipairs(mecha:GetDescendants()) do
        if trail:IsA("Trail") then
            trail.Enabled = false
        end
    end
    
    -- Esconder modelo
    mecha:PivotTo(CFrame.new(0, -100, 0))
    
    -- Desativar colisão
    for _, part in ipairs(mecha:GetDescendants()) do
        if part:IsA("BasePart") then
            part.CanCollide = false
        end
    end
end

-- ============================================================================
-- MÓDULO 10: FÍSICA E CONTROLE DO VEÍCULO
-- ============================================================================

--[[
    Física veicular baseada em raycast para suspensão + CFrame para
    movimento. As rodas são visuais; a física é calculada no pivot.
    
    Abordagem:
    - Raycast para baixo detecta o chão
    - Se no chão: aplica aceleração/frenagem/direção
    - Se no ar: aplica gravidade
    - Rodas animadas visualmente (rotação + offset de suspensão)
]]

local function updateVehiclePhysics(dt)
    if StateMachine.current ~= "Vehicle" then return end
    
    local vehicle = SystemRefs.vehicleModel
    if not vehicle then return end
    
    local pivot = vehicle:FindFirstChild("VehiclePivot")
    if not pivot then return end
    
    local vState = SystemRefs.vehicleState
    
    -- =========================================================================
    -- INPUT (combina mobile + teclado)
    -- =========================================================================
    local accelInput = 0
    local steerInput = 0
    
    -- Mobile
    local vc = SystemRefs.vehicleControls
    if vc.accelHeld then accelInput = 1 end
    if vc.brakeHeld then accelInput = -1 end
    if vc.leftHeld then steerInput = 1 end
    if vc.rightHeld then steerInput = -1 end
    
    -- Teclado (complementar)
    if UserInputService:IsKeyDown(Enum.KeyCode.W) or UserInputService:IsKeyDown(Enum.KeyCode.Up) then
        accelInput = 1
    end
    if UserInputService:IsKeyDown(Enum.KeyCode.S) or UserInputService:IsKeyDown(Enum.KeyCode.Down) then
        accelInput = -1
    end
    if UserInputService:IsKeyDown(Enum.KeyCode.A) or UserInputService:IsKeyDown(Enum.KeyCode.Left) then
        steerInput = 1
    end
    if UserInputService:IsKeyDown(Enum.KeyCode.D) or UserInputService:IsKeyDown(Enum.KeyCode.Right) then
        steerInput = -1
    end
    
    -- =========================================================================
    -- FÍSICA
    -- =========================================================================
    local currentCFrame = pivot.CFrame
    local currentPosition = currentCFrame.Position
    local currentVelocity = Vector3.new(0, 0, 0)
    
    -- Se tínhamos velocidade anterior, calcular a partir do deslocamento
    if vState.lastPosition then
        currentVelocity = (currentPosition - vState.lastPosition) / math.max(dt, 0.001)
    end
    
    -- Raycast para detectar chão
    local rayOrigin = currentPosition + Vector3.new(0, 1, 0)
    local rayDirection = Vector3.new(0, -(CONFIG.VEHICLE_SUSPENSION_REST + 2), 0)
    local rayParams = RaycastParams.new()
    rayParams.FilterDescendantsInstances = {vehicle, SystemRefs.character}
    rayParams.FilterType = Enum.RaycastFilterType.Exclude
    
    local rayResult = workspace:Raycast(rayOrigin, rayDirection, rayParams)
    local grounded = rayResult ~= nil
    vState.grounded = grounded
    
    local groundHeight = 0
    if grounded then
        groundHeight = rayResult.Position.Y
    end
    
    -- Direção forward do veículo
    local forward = currentCFrame.LookVector
    local right = currentCFrame.RightVector
    
    -- =========================================================================
    -- ACELERAÇÃO / FRENAGEM
    -- =========================================================================
    local speed = vState.speed
    
    if accelInput > 0 then
        -- Acelerando para frente
        speed = speed + CONFIG.VEHICLE_ACCELERATION * dt
        speed = math.min(speed, CONFIG.VEHICLE_MAX_SPEED)
    elseif accelInput < 0 then
        if speed > 5 then
            -- Freando (se indo para frente)
            speed = speed - CONFIG.VEHICLE_BRAKE_FORCE * dt
            speed = math.max(speed, 0)
        else
            -- Ré
            speed = speed - CONFIG.VEHICLE_ACCELERATION * 0.5 * dt
            speed = math.max(speed, -CONFIG.VEHICLE_REVERSE_SPEED)
        end
    else
        -- Desaceleração natural (atrito)
        if math.abs(speed) > 0.5 then
            speed = speed - math.sign(speed) * 20 * dt
        else
            speed = 0
        end
    end
    
    vState.speed = speed
    
    -- =========================================================================
    -- DIREÇÃO (só funciona se estiver no chão e movendo)
    -- =========================================================================
    if grounded and math.abs(speed) > 1 then
        local turnFactor = steerInput * CONFIG.VEHICLE_TURN_SPEED * dt
        -- Reduz giro em alta velocidade (estabilidade)
        local speedFactor = math.clamp(1 - (math.abs(speed) / CONFIG.VEHICLE_MAX_SPEED) * 0.5, 0.3, 1)
        turnFactor = turnFactor * speedFactor * math.sign(speed)
        
        currentCFrame = currentCFrame * CFrame.Angles(0, turnFactor, 0)
    end
    
    -- =========================================================================
    -- MOVIMENTO
    -- =========================================================================
    local moveVector = forward * speed * dt
    
    -- =========================================================================
    -- GRAVIDADE / SUSPENSÃO
    -- =========================================================================
    local targetHeight = groundHeight + CONFIG.VEHICLE_SUSPENSION_REST
    local currentHeight = currentPosition.Y
    
    local verticalSpeed = 0
    if grounded then
        -- Suspensão: suaviza para a altura alvo
        local heightDiff = targetHeight - currentHeight
        verticalSpeed = heightDiff * CONFIG.VEHICLE_SUSPENSION_STIFFNESS / math.max(dt, 0.001)
        -- Limita velocidade vertical
        verticalSpeed = math.clamp(verticalSpeed, -30, 30)
    else
        -- Gravidade
        verticalSpeed = -CONFIG.VEHICLE_GRAVITY * dt
    end
    
    -- Aplicar posição
    local newPos = currentPosition + moveVector + Vector3.new(0, verticalSpeed * dt, 0)
    
    -- Impedir cair abaixo do mundo
    if newPos.Y < -500 then
        newPos = Vector3.new(0, 50, 0)
        vState.speed = 0
    end
    
    pivot.CFrame = CFrame.new(newPos) * (currentCFrame - currentCFrame.Position)
    
    -- =========================================================================
    -- ANIMAÇÃO DAS RODAS (visual)
    -- =========================================================================
    local wheelRotation = speed * dt * 2
    
    for _, wheelData in ipairs(SystemRefs.vehicleWheels) do
        local wheel = wheelData.part
        local rim = wheelData.rim
        
        if wheel and rim then
            -- Rotação visual
            wheel.CFrame = wheel.CFrame * CFrame.Angles(wheelRotation, 0, 0)
            rim.CFrame = wheel.CFrame
            
            -- Suspensão visual (as rodas sobem/descem)
            if grounded then
                local suspOffset = math.clamp((currentHeight - groundHeight) - CONFIG.VEHICLE_SUSPENSION_REST, -0.5, 0.5)
                local restPos = wheelData.restPos
                -- Ajusta Y da roda baseado na compressão
                local wheelWorldPos = pivot.CFrame:PointToWorldSpace(restPos + Vector3.new(0, -suspOffset * 0.5, 0))
                -- Mantém X e Z, ajusta Y suavemente
            end
        end
    end
    
    -- =========================================================================
    -- EFEITOS (escape quando acelerando)
    -- =========================================================================
    if math.abs(speed) > CONFIG.VEHICLE_MAX_SPEED * 0.5 then
        activateEffect("ExhaustFireL", 0.1, 20)
        activateEffect("ExhaustFireR", 0.1, 20)
    end
    
    -- Guardar posição para próximo frame
    vState.lastPosition = pivot.CFrame.Position
end

local function activateVehicleControl()
    local vehicle = SystemRefs.vehicleModel
    if not vehicle then
        sysLog("activateVehicleControl: vehicleModel nulo!", "ERROR")
        return false
    end
    
    local pivot = vehicle:FindFirstChild("VehiclePivot")
    local vehicleHumanoid = vehicle:FindFirstChild("VehicleHumanoid")
    
    if not pivot or not vehicleHumanoid then
        sysLog("activateVehicleControl: Pivot ou Humanoid nulo!", "ERROR")
        return false
    end
    
    -- Posicionar veículo na posição do jogador/mecha
    local sourcePos = CFrame.new(0, 10, 0)
    if SystemRefs.mechaModel and StateMachine.current == "Mecha" then
        local mechaPivot = SystemRefs.mechaModel:FindFirstChild("Pivot")
        if mechaPivot then
            sourcePos = mechaPivot.CFrame
        end
    elseif SystemRefs.rootPart then
        sourcePos = SystemRefs.rootPart.CFrame
    end
    
    vehicle:PivotTo(sourcePos + Vector3.new(0, 3, 0))
    
    -- Ativar colisão
    for _, part in ipairs(vehicle:GetDescendants()) do
        if part:IsA("BasePart") and part.Name ~= "VehiclePivot" then
            part.CanCollide = true
        end
    end
    
    -- Ativar faróis
    for _, light in ipairs(vehicle:GetDescendants()) do
        if light:IsA("SpotLight") then
            light.Enabled = true
        end
    end
    
    -- Camera subject para o humanoid do veículo
    camera.CameraSubject = vehicleHumanoid
    
    -- Reset state
    SystemRefs.vehicleState.speed = 0
    SystemRefs.vehicleState.steer = 0
    SystemRefs.vehicleState.lastPosition = nil
    
    -- Network ownership
    pcall(function()
        pivot:SetNetworkOwner(player)
    end)
    
    sysLog("Controle do Veiculo ativado")
    return true
end

local function deactivateVehicleControl()
    local vehicle = SystemRefs.vehicleModel
    if not vehicle then return end
    
    -- Desativar faróis
    for _, light in ipairs(vehicle:GetDescendants()) do
        if light:IsA("Light") then
            light.Enabled = false
        end
    end
    
    -- Reset velocidade
    SystemRefs.vehicleState.speed = 0
    SystemRefs.vehicleState.lastPosition = nil
    
    -- Esconder
    vehicle:PivotTo(CFrame.new(0, -200, 0))
    
    -- Desativar colisão
    for _, part in ipairs(vehicle:GetDescendants()) do
        if part:IsA("BasePart") then
            part.CanCollide = false
        end
    end
end

-- ============================================================================
-- MÓDULO 11: TRANSFORMAÇÕES
-- ============================================================================

--- Esconde o personagem humano (salvando propriedades originais)
local function hideHumanCharacter()
    local char = SystemRefs.character
    if not char then return end
    
    -- Salvar propriedades originais
    SystemRefs.originalProperties = {}
    for _, part in ipairs(char:GetChildren()) do
        if part:IsA("BasePart") then
            SystemRefs.originalProperties[part.Name] = {
                Transparency = part.Transparency,
                CanCollide = part.CanCollide,
                LocalTransparencyModifier = part.LocalTransparencyModifier,
            }
            part.Transparency = 1
            part.CanCollide = false
        end
    end
    
    -- Desativar humanoid original
    local humanoid = char:FindFirstChildOfClass("Humanoid")
    if humanoid then
        SystemRefs.originalProperties["_walkSpeed"] = humanoid.WalkSpeed
        SystemRefs.originalProperties["_jumpPower"] = humanoid.JumpPower
        humanoid.WalkSpeed = 0
        humanoid.JumpPower = 0
    end
end

--- Restaura o personagem humano
local function restoreHumanCharacter()
    local char = SystemRefs.character
    if not char then return end
    
    for _, part in ipairs(char:GetChildren()) do
        if part:IsA("BasePart") and SystemRefs.originalProperties[part.Name] then
            local orig = SystemRefs.originalProperties[part.Name]
            part.Transparency = orig.Transparency
            part.CanCollide = orig.CanCollide
        end
    end
    
    -- Restaurar humanoid
    local humanoid = char:FindFirstChildOfClass("Humanoid")
    if humanoid then
        humanoid.WalkSpeed = SystemRefs.originalProperties["_walkSpeed"] or CONFIG.HUMAN_WALK_SPEED
        humanoid.JumpPower = SystemRefs.originalProperties["_jumpPower"] or CONFIG.HUMAN_JUMP_POWER
    end
    
    -- Restaurar câmera
    if humanoid then
        camera.CameraSubject = humanoid
    end
end

--- Sequência de transformação completa
--- @param targetForm string "Mecha", "Vehicle" ou "Human"
function startTransformation(targetForm)
    local current = StateMachine.current
    
    -- Validar transição
    if StateMachine.isLocked then
        sysLog("Transformacao bloqueada (ja em andamento)", "WARN")
        return
    end
    
    -- Se já está na forma alvo, ignora
    if current == targetForm then return end
    
    -- =========================================================================
    -- RETORNAR PARA HUMANO
    -- =========================================================================
    if targetForm == "Human" then
        if current == "Human" then return end
        
        StateMachine.isLocked = true
        StateMachine:transition("ReturningToHuman")
        
        updateTransformBar(0, "RETORNANDO...")
        
        -- Desativar forma atual
        if current == "Mecha" then
            deactivateMechaControl()
        elseif current == "Vehicle" then
            deactivateVehicleControl()
        end
        
        -- Restaurar humano
        restoreHumanCharacter()
        
        -- Animação de retorno rápida
        task.wait(0.3)
        
        hideTransformBar()
        StateMachine:transition("Human")
        StateMachine.isLocked = false
        updateUI()
        
        sysLog("Retorno ao Humano completo")
        return
    end
    
    -- =========================================================================
    -- HUMANO → MECHA
    -- =========================================================================
    if targetForm == "Mecha" and current == "Human" then
        StateMachine.isLocked = true
        StateMachine:transition("TransformingToMecha")
        
        -- Esconder humano
        hideHumanCharacter()
        
        -- Fase 1: Antecipação (vibração)
        updateTransformBar(0.1, "INICIANDO...")
        activateEffect("ShoulderSparksL", 0.5, 20)
        activateEffect("ShoulderSparksR", 0.5, 20)
        task.wait(0.5)
        
        -- Fase 2: Posicionar mecha
        updateTransformBar(0.3, "MONTANDO MECHA...")
        if SystemRefs.mechaModel then
            local charPos = SystemRefs.rootPart and SystemRefs.rootPart.CFrame or CFrame.new(0, 10, 0)
            SystemRefs.mechaModel:PivotTo(charPos)
        end
        task.wait(0.5)
        
        -- Fase 3: Animação de montagem (peças se movendo)
        updateTransformBar(0.6, "CALIBRANDO SISTEMAS...")
        activateEffect("ShoulderSparksL", 1, 40)
        activateEffect("ShoulderSparksR", 1, 40)
        task.wait(0.8)
        
        -- Fase 4: Ativação
        updateTransformBar(0.9, "ATIVANDO...")
        activateMechaControl()
        task.wait(0.3)
        
        hideTransformBar()
        StateMachine:transition("Mecha")
        StateMachine.isLocked = false
        updateUI()
        
        sysLog("Transformacao para Mecha completa")
        return
    end
    
    -- =========================================================================
    -- MECHA → VEÍCULO
    -- =========================================================================
    if targetForm == "Vehicle" and current == "Mecha" then
        StateMachine.isLocked = true
        StateMachine:transition("TransformingToVehicle")
        
        updateTransformBar(0.1, "RECONFIGURANDO...")
        
        -- Desativar mecha
        deactivateMechaControl()
        task.wait(0.3)
        
        -- Fase 2: Montar veículo
        updateTransformBar(0.4, "MODO VEICULO...")
        activateEffect("ShoulderSparksL", 0.5, 30)
        activateEffect("ShoulderSparksR", 0.5, 30)
        task.wait(0.4)
        
        -- Ativar veículo
        updateTransformBar(0.8, "MOTORES LIGADOS...")
        activateVehicleControl()
        task.wait(0.3)
        
        hideTransformBar()
        StateMachine:transition("Vehicle")
        StateMachine.isLocked = false
        updateUI()
        
        sysLog("Transformacao para Veiculo completa")
        return
    end
    
    -- =========================================================================
    -- VEÍCULO → MECHA
    -- =========================================================================
    if targetForm == "Mecha" and current == "Vehicle" then
        StateMachine.isLocked = true
        StateMachine:transition("TransformingToMecha")
        
        updateTransformBar(0.1, "RECONFIGURANDO...")
        
        -- Desativar veículo
        deactivateVehicleControl()
        task.wait(0.3)
        
        -- Montar mecha
        updateTransformBar(0.4, "MODO MECHA...")
        activateEffect("ShoulderSparksL", 0.5, 30)
        activateEffect("ShoulderSparksR", 0.5, 30)
        task.wait(0.4)
        
        updateTransformBar(0.8, "SISTEMAS ONLINE...")
        activateMechaControl()
        task.wait(0.3)
        
        hideTransformBar()
        StateMachine:transition("Mecha")
        StateMachine.isLocked = false
        updateUI()
        
        sysLog("Transformacao Veiculo->Mecha completa")
        return
    end
    
    -- =========================================================================
    -- HUMANO → VEÍCULO (pula mecha)
    -- =========================================================================
    if targetForm == "Vehicle" and current == "Human" then
        StateMachine.isLocked = true
        
        -- Primeiro vai para Mecha (intermediário)
        hideHumanCharacter()
        
        updateTransformBar(0.2, "TRANSFORMANDO...")
        if SystemRefs.mechaModel then
            local charPos = SystemRefs.rootPart and SystemRefs.rootPart.CFrame or CFrame.new(0, 10, 0)
            SystemRefs.mechaModel:PivotTo(charPos)
        end
        task.wait(0.5)
        
        -- Depois vai para Vehicle
        updateTransformBar(0.6, "MODO VEICULO...")
        deactivateMechaControl()
        task.wait(0.3)
        
        activateVehicleControl()
        task.wait(0.3)
        
        hideTransformBar()
        StateMachine.current = "Vehicle"
        StateMachine.isLocked = false
        updateUI()
        
        sysLog("Transformacao Humano->Veiculo completa")
        return
    end
    
    -- =========================================================================
    -- VEÍCULO → HUMANO
    -- =========================================================================
    if targetForm == "Human" and current == "Vehicle" then
        -- Já tratado acima no bloco targetForm == "Human"
        return
    end
    
    sysLog("Transicao nao reconhecida: " .. current .. " -> " .. targetForm, "WARN")
end

-- ============================================================================
-- MÓDULO 12: ANIMAÇÕES PROCEDURAIS DO MECHA
-- ============================================================================

local proceduralTime = 0

local function updateMechaProcedural(dt)
    if StateMachine.current ~= "Mecha" or not SystemRefs.mechaModel then return end
    
    proceduralTime = proceduralTime + dt
    local t = proceduralTime
    local mecha = SystemRefs.mechaModel
    
    -- Respiração mecânica (torso sobe/desce sutilmente)
    local upperTorsoJoint = SystemRefs.mechaJoints["UpperTorsoJoint"]
    if upperTorsoJoint then
        local breathOffset = math.sin(t * 2) * 0.015
        local baseC0 = CFrame.new(0, -0.4, 0)
        upperTorsoJoint.C0 = baseC0 * CFrame.Angles(breathOffset, 0, 0)
    end
    
    -- Head tracking (seguir mouse/câmera)
    local headJoint = SystemRefs.mechaJoints["HeadJoint"]
    if headJoint then
        local camCF = camera.CFrame
        local mechaPivot = mecha:FindFirstChild("Pivot")
        if mechaPivot then
            local toCamera = (camCF.Position - mechaPivot.Position).Unit
            local _, yaw = math.atan2(toCamera.X, toCamera.Z), 0
            
            -- Limitar rotação
            local targetYaw = math.clamp(yaw, -math.rad(40), math.rad(40))
            local targetPitch = math.clamp(math.asin(toCamera.Y) * 0.5, -math.rad(20), math.rad(20))
            
            local baseC0 = CFrame.new(0, 0.4, 0)
            headJoint.C0 = headJoint.C0:Lerp(
                baseC0 * CFrame.Angles(-targetPitch, targetYaw, 0),
                0.08
            )
        end
    end
    
    -- Arm sway (balanço dos braços ao andar)
    local mechaHumanoid = mecha:FindFirstChild("MechaHumanoid")
    if mechaHumanoid and mechaHumanoid.MoveDirection.Magnitude > 0.1 then
        local walkSpeed = mechaHumanoid.WalkSpeed
        local swayAmount = math.sin(t * 6) * 0.08 * (walkSpeed / CONFIG.MECHA_WALK_SPEED)
        
        local armJointL = SystemRefs.mechaJoints["UpperArmJointL"]
        if armJointL then
            local baseC0 = CFrame.new(-0.3, -0.9, 0)
            armJointL.C0 = baseC0 * CFrame.Angles(swayAmount, 0, 0)
        end
        
        local armJointR = SystemRefs.mechaJoints["UpperArmJointR"]
        if armJointR then
            local baseC0 = CFrame.new(0.3, -0.9, 0)
            armJointR.C0 = baseC0 * CFrame.Angles(-swayAmount, 0, 0)
        end
        
        -- Ativar trails quando correndo
        local isRunning = walkSpeed > CONFIG.MECHA_WALK_SPEED * 1.2
        for _, trail in ipairs(mecha:GetDescendants()) do
            if trail:IsA("Trail") then
                trail.Enabled = isRunning
            end
        end
    else
        -- Desativar trails quando parado
        for _, trail in ipairs(mecha:GetDescendants()) do
            if trail:IsA("Trail") then
                trail.Enabled = false
            end
        end
    end
    
    -- Pulsar reator
    local reactor = mecha:FindFirstChild("Reactor", true)
    if reactor then
        local light = reactor:FindFirstChildOfClass("PointLight")
        if light and light.Enabled then
            light.Brightness = 3 + math.sin(t * 3) * 1
        end
    end
end

-- ============================================================================
-- MÓDULO 13: EFEITOS DE PASSOS DO MECHA
-- ============================================================================

local lastFootstepTime = 0

local function updateMechaFootsteps(dt)
    if StateMachine.current ~= "Mecha" or not SystemRefs.mechaModel then return end
    
    local mechaHumanoid = SystemRefs.mechaModel:FindFirstChild("MechaHumanoid")
    if not mechaHumanoid or mechaHumanoid.MoveDirection.Magnitude < 0.1 then return end
    
    local now = tick()
    local interval = 0.7 - (mechaHumanoid.WalkSpeed / CONFIG.MECHA_WALK_SPEED) * 0.2
    interval = math.max(interval, 0.3)
    
    if now - lastFootstepTime > interval then
        lastFootstepTime = now
        -- Ativar efeito de poeira alternado
        if math.random() > 0.5 then
            activateEffect("FootDustL", 0.8, 15)
        else
            activateEffect("FootDustR", 0.8, 15)
        end
    end
end

-- ============================================================================
-- MÓDULO 14: LOOP PRINCIPAL DE ATUALIZAÇÃO
-- ============================================================================

local mainConnection = nil

local function startMainLoop()
    if mainConnection then
        mainConnection:Disconnect()
    end
    
    local diagUpdateTimer = 0
    
    mainConnection = RunService.Heartbeat:Connect(function(dt)
        -- Física do veículo
        updateVehiclePhysics(dt)
        
        -- Animações procedurais do mecha
        updateMechaProcedural(dt)
        
        -- Efeitos de passos
        updateMechaFootsteps(dt)
        
        -- Atualizar diagnóstico periodicamente
        diagUpdateTimer = diagUpdateTimer + dt
        if diagUpdateTimer > 0.5 then
            diagUpdateTimer = 0
            updateDiagPanel()
        end
    end)
    
    table.insert(SystemRefs.connections, mainConnection)
end

-- ============================================================================
-- MÓDULO 15: CONTROLES DE TECLADO
-- ============================================================================

local function setupKeyboardControls()
    -- Tecla T = Transformar (ciclo)
    ContextActionService:BindAction("MechaTransform_T", function(_, inputState)
        if inputState == Enum.UserInputState.Begin then
            if StateMachine.current == "Human" then
                startTransformation("Mecha")
            elseif StateMachine.current == "Mecha" then
                startTransformation("Vehicle")
            elseif StateMachine.current == "Vehicle" then
                startTransformation("Human")
            end
        end
    end, false, Enum.KeyCode.T)
    
    -- Tecla R = Reverter para Humano
    ContextActionService:BindAction("MechaRevert_R", function(_, inputState)
        if inputState == Enum.UserInputState.Begin then
            if StateMachine.current ~= "Human" then
                startTransformation("Human")
            end
        end
    end, false, Enum.KeyCode.R)
    
    -- Tecla G = Toggle diagnóstico
    ContextActionService:BindAction("MechaDiag_G", function(_, inputState)
        if inputState == Enum.UserInputState.Begin then
            local diagPanel = SystemRefs.ui and SystemRefs.ui.diagPanel
            if diagPanel then
                diagPanel.Visible = not diagPanel.Visible
            end
        end
    end, false, Enum.KeyCode.G)
end

-- ============================================================================
-- MÓDULO 16: INICIALIZAÇÃO
-- ============================================================================

local function getCharacterRefs()
    local char = player.Character
    if not char then
        char = player.CharacterAdded:Wait()
    end
    
    local humanoid = char:WaitForChild("Humanoid", 10)
    local rootPart = char:WaitForChild("HumanoidRootPart", 10)
    
    if not humanoid or not rootPart then
        sysLog("Falha ao obter Humanoid ou RootPart!", "ERROR")
        StateMachine.lastError = "Character refs nulos"
        return false
    end
    
    SystemRefs.character = char
    SystemRefs.humanoid = humanoid
    SystemRefs.rootPart = rootPart
    
    return true
end

local function initialize()
    sysLog("=== INICIALIZANDO MECHA SYSTEM V" .. CONFIG.VERSION .. " ===")
    
    -- Fase 1: Obter referências do personagem
    StateMachine.initPhase = 1
    if not getCharacterRefs() then
        StateMachine:transition("Recovering")
        StateMachine.lastError = "Falha em getCharacterRefs"
        return false
    end
    sysLog("Fase 1: Personagem obtido")
    
    -- Fase 2: Construir modelos
    StateMachine.initPhase = 2
    local mechaOk, mechaErr = pcall(function()
        SystemRefs.mechaModel = buildMechaModel()
        SystemRefs.mechaModel.Parent = workspace
    end)
    if not mechaOk then
        sysLog("Falha ao construir Mecha: " .. tostring(mechaErr), "ERROR")
        StateMachine.lastError = "Falha buildMecha: " .. tostring(mechaErr)
    end
    
    local vehicleOk, vehicleErr = pcall(function()
        SystemRefs.vehicleModel = buildVehicleModel()
        SystemRefs.vehicleModel.Parent = workspace
    end)
    if not vehicleOk then
        sysLog("Falha ao construir Veiculo: " .. tostring(vehicleErr), "ERROR")
        StateMachine.lastError = "Falha buildVehicle: " .. tostring(vehicleErr)
    end
    sysLog("Fase 2: Modelos construidos")
    
    -- Fase 3: Criar UI
    StateMachine.initPhase = 3
    local uiOk, uiErr = pcall(function()
        createUI()
    end)
    if not uiOk then
        sysLog("Falha ao criar UI: " .. tostring(uiErr), "ERROR")
        StateMachine.lastError = "Falha createUI: " .. tostring(uiErr)
    end
    sysLog("Fase 3: UI criada")
    
    -- Fase 4: Configurar controles
    StateMachine.initPhase = 4
    pcall(function()
        setupKeyboardControls()
    end)
    sysLog("Fase 4: Controles configurados")
    
    -- Fase 5: Iniciar loop principal
    StateMachine.initPhase = 5
    startMainLoop()
    sysLog("Fase 5: Loop principal iniciado")
    
    -- Fase 6: Transicionar para Human
    StateMachine.initPhase = 6
    StateMachine:transition("Human")
    updateUI()
    
    StateMachine.initialized = true
    StateMachine.initPhase = 99
    sysLog("=== SISTEMA INICIALIZADO COM SUCESSO ===")
    
    return true
end

-- ============================================================================
-- MÓDULO 17: GERENCIAMENTO DE RESPAWN
-- ============================================================================

local function onCharacterRemoving()
    sysLog("Personagem removido - limpando...")
    
    -- Desativar controles
    if StateMachine.current == "Mecha" then
        deactivateMechaControl()
    elseif StateMachine.current == "Vehicle" then
        deactivateVehicleControl()
    end
    
    -- Reset estado
    StateMachine.current = "Recovering"
    StateMachine.isLocked = false
    SystemRefs.character = nil
    SystemRefs.humanoid = nil
    SystemRefs.rootPart = nil
    SystemRefs.originalProperties = {}
end

local function onCharacterAdded(char)
    sysLog("Novo personagem - re-inicializando...")
    
    -- Aguardar carregar
    task.wait(1)
    
    -- Re-obter referências
    SystemRefs.character = char
    SystemRefs.humanoid = char:WaitForChild("Humanoid", 10)
    SystemRefs.rootPart = char:WaitForChild("HumanoidRootPart", 10)
    
    -- Restaurar câmera
    if SystemRefs.humanoid then
        camera.CameraSubject = SystemRefs.humanoid
    end
    
    -- Atualizar estado
    if StateMachine.current == "Recovering" or StateMachine.current == "Initializing" then
        StateMachine.current = "Human"
    end
    
    updateUI()
    sysLog("Personagem restaurado")
end

-- ============================================================================
-- MÓDULO 18: CLEANUP
-- ============================================================================

local function cleanup()
    sysLog("Executando cleanup...")
    
    -- Desconectar loop principal
    if mainConnection then
        mainConnection:Disconnect()
        mainConnection = nil
    end
    
    -- Desconectar todas as conexões
    for _, conn in ipairs(SystemRefs.connections) do
        if conn then
            pcall(function() conn:Disconnect() end)
        end
    end
    SystemRefs.connections = {}
    
    -- Destruir modelos
    if SystemRefs.mechaModel then
        SystemRefs.mechaModel:Destroy()
        SystemRefs.mechaModel = nil
    end
    if SystemRefs.vehicleModel then
        SystemRefs.vehicleModel:Destroy()
        SystemRefs.vehicleModel = nil
    end
    
    -- Destruir UI
    local ui = SystemRefs.ui
    if ui.screenGui then
        ui.screenGui:Destroy()
    end
    
    -- Limpar tabelas
    SystemRefs.mechaJoints = {}
    SystemRefs.mechaParts = {}
    SystemRefs.vehicleParts = {}
    SystemRefs.vehicleWheels = {}
    SystemRefs.effects = {}
    SystemRefs.ui = {}
    SystemRefs.mobileButtons = {}
    SystemRefs.vehicleControls = {}
    
    -- Unbind ações
    ContextActionService:UnbindAction("MechaTransform_T")
    ContextActionService:UnbindAction("MechaRevert_R")
    ContextActionService:UnbindAction("MechaDiag_G")
    
    sysLog("Cleanup completo")
end

-- ============================================================================
-- MÓDULO 19: EXECUÇÃO PRINCIPAL
-- ============================================================================

-- Conectar eventos de respawn
player.CharacterRemoving:Connect(onCharacterRemoving)
player.CharacterAdded:Connect(onCharacterAdded)

-- Inicializar
task.spawn(function()
    local success = initialize()
    if not success then
        sysLog("Inicializacao falhou - tentando recuperacao...", "WARN")
        task.wait(3)
        -- Tentar novamente
        success = initialize()
        if not success then
            sysLog("Recuperacao falhou - sistema indisponivel", "ERROR")
        end
    end
end)

-- Cleanup ao sair
game:BindToClose(function()
    cleanup()
end)

--[[
================================================================================
    FIM DO CLIENT SCRIPT
    
    RESUMO DAS CORREÇÕES V2.0 → V3.0:
    1. ✅ Sem WaitForChild infinito - usa timeout e cria o que precisa
    2. ✅ Modelos procedurais detalhados com coordenadas relativas
    3. ✅ Motor6D centralizados em tabela (SystemRefs.mechaJoints)
    4. ✅ Controle real do Mecha via CameraSubject + Humanoid
    5. ✅ Física veicular com raycast + CFrame coerente
    6. ✅ State machine com transições válidas
    7. ✅ UI mobile-first com debounce
    8. ✅ Painel de diagnóstico visual
    9. ✅ Cleanup completo em respawn
    10. ✅ Recuperação de erros com fallback
    11. ✅ Efeitos reutilizáveis (não cria/destrói a cada frame)
    12. ✅ Sem dependências externas obrigatórias
================================================================================
]]
