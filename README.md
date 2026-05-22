# GOO-CLSS Available for download
import SwiftUI
import SpriteKit
import Combine
import QuartzCore

// =========================================================================
// MARK: - 1. 全域核心數據與高級列舉 (Enterprise System Enumerations)
// =========================================================================

enum GameMode: String, Codable, CaseIterable {
    case single = "單人無盡闖關"
    case aiBattle = "冷酷專家挑戰"
    case pvp = "本地同屏雙打"
}

enum GameStatus {
    case start      
    case playing    
    case bossFight  
    case gameOver   
    case paused     
}

enum ItemType: CaseIterable {
    case heal        
    case shield      
    case enlarge     
    case speedUp     
    case magnet      
    case coinPack    
    case DoubleDamage
    case TimeWarp
}

enum BossPhase {
    case phase1 // 常規左右移動 + 單發散射
    case phase2 // 狂暴：護盾開啟 + 雙管追蹤彈
    case phase3 // 終焉：中場降臨 + 全螢幕正弦波螺旋彈幕地獄
}

enum PaddleSkin: String, CaseIterable, Codable {
    case classic = "經典科技"
    case neonPulse = "霓虹脈衝"
    case magmaCore = "熔岩核心"
    case chronoShift = "時空裂隙"
    case cyberGhost = "賽博幽靈"
    
    var cost: Int {
        switch self {
        case .classic: return 0
        case .neonPulse: return 150
        case .magmaCore: return 350
        case .chronoShift: return 600
        case .cyberGhost: return 1000
        }
    }
    
    var color: UIColor {
        switch self {
        case .classic: return .systemCyan
        case .neonPulse: return .systemGreen
        case .magmaCore: return .systemOrange
        case .chronoShift: return .systemPurple
        case .cyberGhost: return .systemPink
        }
    }
}

enum TrailEffect: String, CaseIterable, Codable {
    case standard = "標準彗星"
    case fireLine = "烈焰焚天"
    case ghostShadow = "幽靈幻影"
    case matrixCode = "矩陣代碼"
    
    var cost: Int {
        switch self {
        case .standard: return 0
        case .fireLine: return 200
        case .ghostShadow: return 450
        case .matrixCode: return 800
        }
    }
    
    var color: UIColor {
        switch self {
        case .standard: return .white
        case .fireLine: return .red
        case .ghostShadow: return .purple
        case .matrixCode: return .green
        }
    }
}

// =========================================================================
// MARK: - 2. RPG 天賦永久升級樹資料結構 (RPG Talent Tree Structure)
// =========================================================================

struct TalentNode: Codable, Identifiable {
    var id: String
    var name: String
    var currentLevel: Int
    var maxLevel: Int
    var baseCost: Int
    var costMultiplier: Double
    var description: String
    
    var currentCost: Int {
        if currentLevel >= maxLevel { return 0 }
        return Int(Double(baseCost) * pow(costMultiplier, Double(currentLevel)))
    }
}

// =========================================================================
// MARK: - 3. 工業級數據永久化與防止篡改存檔中心 (Enterprise Storage Manager)
// =========================================================================

class GameStorageManager {
    static let shared = GameStorageManager()
    
    private let keyHighScore = "Paddle_Strike_v5_HighScore"
    private let keyCoins = "Paddle_Strike_v5_Coins"
    private let keySkins = "Paddle_Strike_v5_Skins"
    private let keyTrails = "Paddle_Strike_v5_Trails"
    private let keyCurrentSkin = "Paddle_Strike_v5_CurrentSkin"
    private let keyCurrentTrail = "Paddle_Strike_v5_CurrentTrail"
    private let keyAchievements = "Paddle_Strike_v5_Achievements"
    private let keyTalents = "Paddle_Strike_v5_Talents"
    
    func getHighScore() -> Int { UserDefaults.standard.integer(forKey: keyHighScore) }
    func saveHighScore(_ score: Int) {
        if score > getHighScore() { UserDefaults.standard.set(score, forKey: keyHighScore) }
    }
    
    func getCoins() -> Int { UserDefaults.standard.integer(forKey: keyCoins) }
    func addCoins(_ amount: Int) {
        let current = getCoins()
        UserDefaults.standard.set(current + amount, forKey: keyCoins)
    }
    func spendCoins(_ amount: Int) -> Bool {
        let current = getCoins()
        if current >= amount {
            UserDefaults.standard.set(current - amount, forKey: keyCoins)
            return true
        }
        return false
    }
    
    // ✨ 錯誤 3 已修正：移除錯誤嵌入的 private func try? 宣告
    func getUnlockedSkins() -> [PaddleSkin] {
        if let data = UserDefaults.standard.data(forKey: keySkins),
           let decoded = try? JSONDecoder().decode([PaddleSkin].self, from: data) {
            return decoded
        }
        return [.classic]
    }
    
    func unlockSkin(_ skin: PaddleSkin) {
        var current = getUnlockedSkins()
        if !current.contains(skin) {
            current.append(skin)
            if let encoded = try? JSONEncoder().encode(current) { UserDefaults.standard.set(encoded, forKey: keySkins) }
        }
    }
    func getCurrentSkin() -> PaddleSkin {
        if let raw = UserDefaults.standard.string(forKey: keyCurrentSkin), let skin = PaddleSkin(rawValue: raw) { return skin }
        return .classic
    }
    func setCurrentSkin(_ skin: PaddleSkin) { UserDefaults.standard.set(skin.rawValue, forKey: keyCurrentSkin) }
    
    func getUnlockedTrails() -> [TrailEffect] {
        if let data = UserDefaults.standard.data(forKey: keyTrails),
           let decoded = try? JSONDecoder().decode([TrailEffect].self, from: data) { return decoded }
        return [.standard]
    }
    func unlockTrail(_ trail: TrailEffect) {
        var current = getUnlockedTrails()
        if !current.contains(trail) {
            current.append(trail)
            if let encoded = try? JSONEncoder().encode(current) { UserDefaults.standard.set(encoded, forKey: keyTrails) }
        }
    }
    func getCurrentTrail() -> TrailEffect {
        if let raw = UserDefaults.standard.string(forKey: keyCurrentTrail), let trail = TrailEffect(rawValue: raw) { return trail }
        return .standard
    }
    func setCurrentTrail(_ trail: TrailEffect) { UserDefaults.standard.set(trail.rawValue, forKey: keyCurrentTrail) }
    
    func getUnlockedAchievements() -> [String] { UserDefaults.standard.stringArray(forKey: keyAchievements) ?? [] }
    func unlockAchievement(_ id: String) -> Bool {
        var current = getUnlockedAchievements()
        if !current.contains(id) {
            current.append(id)
            UserDefaults.standard.set(current, forKey: keyAchievements)
            return true
        }
        return false
    }
    
    func initializeDefaultTalents() -> [TalentNode] {
        return [
            TalentNode(id: "p_width", name: "合金拓寬外殼", currentLevel: 0, maxLevel: 5, baseCost: 100, costMultiplier: 1.5, description: "永久增加初始擋板寬度 +8%"),
            TalentNode(id: "p_hp", name: "泰坦核心結構", currentLevel: 0, maxLevel: 3, baseCost: 200, costMultiplier: 2.0, description: "戰局初始生命上限永久增加 +1"),
            TalentNode(id: "p_magnet", name: "電磁脈衝擴充", currentLevel: 0, maxLevel: 5, baseCost: 120, costMultiplier: 1.4, description: "磁鐵道具持續效應延長 +1.5 秒"),
            TalentNode(id: "p_crit", name: "量子得分暴擊", currentLevel: 0, maxLevel: 5, baseCost: 150, costMultiplier: 1.6, description: "擊球時有 5% 機率獲得雙倍高額得分")
        ]
    }
    
    func loadTalents() -> [TalentNode] {
        if let data = UserDefaults.standard.data(forKey: keyTalents),
           let decoded = try? JSONDecoder().decode([TalentNode].self, from: data) {
            return decoded
        }
        let defaults = initializeDefaultTalents()
        saveTalents(defaults)
        return defaults
    }
    
    func saveTalents(_ talents: [TalentNode]) {
        if let encoded = try? JSONEncoder().encode(talents) {
            UserDefaults.standard.set(encoded, forKey: keyTalents)
        }
    }
}

class GameAudioSettings: ObservableObject {
    static let shared = GameAudioSettings()
    
    @Published var bgmVolume: Float = 0.65
    @Published var sfxVolume: Float = 0.85
    @Published var isMuted: Bool = false
    @Published var goldDisplay: Int = GameStorageManager.shared.getCoins()
    @Published var newlyUnlockedAchievement: String? = nil
    @Published var globalTalents: [TalentNode] = GameStorageManager.shared.loadTalents()
    
    func refreshCoins() { goldDisplay = GameStorageManager.shared.getCoins() }
    func triggerAchievementPopup(_ name: String) {
        newlyUnlockedAchievement = name
        DispatchQueue.main.asyncAfter(deadline: .now() + 3.5) { self.newlyUnlockedAchievement = nil }
    }
    func updateTalentTree(nodeID: String) {
        var list = globalTalents
        if let index = list.firstIndex(where: { $0.id == nodeID }) {
            let cost = list[index].currentCost
            if list[index].currentLevel < list[index].maxLevel && GameStorageManager.shared.spendCoins(cost) {
                list[index].currentLevel += 1
                GameStorageManager.shared.saveTalents(list)
                self.globalTalents = list
                refreshCoins()
            }
        }
    }
}

// =========================================================================
// MARK: - 4. 戰局全面大數據核心 (Elite Deep Match Metrics)
// =========================================================================

struct MatchRecord {
    var totalBallsDeflected: Int = 0
    var highestComboValue: Int = 0
    var itemCollectedCount: Int = 0
    var goldEarnedThisRound: Int = 0
    var bossDamageDealt: Int = 0
    var perfectCatches: Int = 0
    var bossBulletsDodged: Int = 0
    var totalShotsFiredByBoss: Int = 0 // ✨ 錯誤 4 已修正：成功加入結構體變數
    var timeElapsed: TimeInterval = 0.0
}

// =========================================================================
// MARK: - 5. 賽博龐克流體微粒子渲染矩陣 (Elite Particle Matrix)
// =========================================================================

class ParticleEngine {
    static func createExplosion(at pos: CGPoint, color: UIColor, in parent: SKNode, sizeScale: CGFloat = 1.0) {
        let count = Int(18.0 * sizeScale)
        for _ in 1...count {
            let spark = SKShapeNode(circleOfRadius: CGFloat.random(in: 2.0...5.0) * sizeScale)
            spark.fillColor = color
            spark.strokeColor = .clear
            spark.position = pos
            spark.zPosition = 80
            
            let angle = CGFloat.random(in: 0...(CGFloat.pi * 2))
            let dynamicSpeed = CGFloat.random(in: 120...280) * sizeScale
            let dx = cos(angle) * dynamicSpeed
            let dy = sin(angle) * dynamicSpeed
            
            parent.addChild(spark)
            
            let move = SKAction.moveBy(x: dx * 0.5, y: dy * 0.5, duration: 0.45)
            let fade = SKAction.fadeOut(withDuration: 0.45)
            
            // ✨ 錯誤 2 已修正：將 scaleTo 調整為全新 Swift 語法 scale(to:duration:)
            let shrink = SKAction.scale(to: 0.01, duration: 0.45)
            
            let combinedGroup = SKAction.group([move, fade, shrink])
            spark.run(SKAction.sequence([combinedGroup, SKAction.removeFromParent()]))
        }
    }
    
    static func createTrail(at pos: CGPoint, in parent: SKNode) {
        let activeTrailType = GameStorageManager.shared.getCurrentTrail()
        let trailRadius: CGFloat = (activeTrailType == .matrixCode) ? 10.0 : 13.0
        
        let trailNode = SKShapeNode(circleOfRadius: trailRadius)
        trailNode.fillColor = activeTrailType.color.withAlphaComponent(0.38)
        trailNode.strokeColor = .clear
        trailNode.position = pos
        trailNode.zPosition = 15
        
        parent.addChild(trailNode)
        
        let fadeDuration: TimeInterval = (activeTrailType == .ghostShadow) ? 0.42 : 0.25
        let finalScaleFactor: CGFloat = (activeTrailType == .fireLine) ? 0.28 : 0.04
        
        trailNode.run(SKAction.sequence([
            SKAction.group([SKAction.fadeOut(withDuration: fadeDuration), SKAction.scale(to: finalScaleFactor, duration: fadeDuration)]),
            SKAction.removeFromParent()
        ]))
    }
    
    static func createShockwave(at pos: CGPoint, in parent: SKNode, radius: CGFloat = 90.0) {
        let wave = SKShapeNode(circleOfRadius: 5.0)
        wave.position = pos
        wave.fillColor = .clear
        wave.strokeColor = .cyan
        wave.lineWidth = 3.0
        wave.zPosition = 45
        parent.addChild(wave)
        
        let expand = SKAction.scale(to: radius / 5.0, duration: 0.5)
        let fade = SKAction.fadeOut(withDuration: 0.5)
        wave.run(SKAction.sequence([SKAction.group([expand, fade]), SKAction.removeFromParent()]))
    }
}

// =========================================================================
// MARK: - 6. 完全體全場景架構核心 (Heavyweight Game Scene Core)
// =========================================================================

class GameScene: SKScene, SKPhysicsContactDelegate {
    
    var currentMode: GameMode = .single
    var gameState: GameStatus = .start
    var bossPhase: BossPhase = .phase1
    
    var onStateChange: ((Bool) -> Void)?
    var onMatchComplete: ((MatchRecord) -> Void)?
    var bgmNode: SKAudioNode?
    
    var currentLevel: Int = 1
    var p1Score: Int = 0
    var p2Score: Int = 0
    var p1Health: Int = 5
    var p2Health: Int = 5
    var maxHealth: Int = 5
    
    var bossMaxHealth: Int = 100
    var bossCurrentHealth: Int = 100
    private var bossDirection: CGFloat = 1.0
    private var bossTimeTracker: TimeInterval = 0.0
    private var matchStartTime: TimeInterval = 0.0
    
    let aiMaxSpeed: CGFloat = 9.5
    var liveStats = MatchRecord()
    var currentCombo = 0
    
    var p1ShieldActive = false
    var p2ShieldActive = false
    var magnetActive = false
    var doubleDamageActive = false
    var timeWarpActive = false
    
    var p1Touch: UITouch?
    var p2Touch: UITouch?
    
    let p1Paddle = SKSpriteNode(color: .cyan, size: CGSize(width: 150, height: 22))
    let p2Paddle = SKSpriteNode(color: .magenta, size: CGSize(width: 150, height: 22))
    
    let p1ShieldVisual = SKShapeNode(rectOf: CGSize(width: 175, height: 8))
    let p2ShieldVisual = SKShapeNode(rectOf: CGSize(width: 175, height: 8))
    
    let p1StatLabel = SKLabelNode(fontNamed: "AvenirNext-Bold")
    let p2StatLabel = SKLabelNode(fontNamed: "AvenirNext-Bold")
    let infoLabel = SKLabelNode(fontNamed: "AvenirNext-Heavy")
    let highScoreLabel = SKLabelNode(fontNamed: "AvenirNext-Medium")
    let comboLabel = SKLabelNode(fontNamed: "AvenirNext-Bold")
    
    var bossNode: SKSpriteNode?
    var bossShieldNode: SKShapeNode?
    var bossHealthBarBackground: SKShapeNode?
    var bossHealthBarForeground: SKShapeNode?
    
    // 物理遮罩矩陣
    let ballCat: UInt32        = 0x1 << 0
    let paddleCat: UInt32      = 0x1 << 1
    let wallCat: UInt32        = 0x1 << 2
    let itemCat: UInt32        = 0x1 << 3
    let obstacleCat: UInt32    = 0x1 << 4
    let coinCat: UInt32        = 0x1 << 5
    let bossBulletCat: UInt32  = 0x1 << 6
    
    func playBGM() {
        bgmNode?.removeFromParent()
        if GameAudioSettings.shared.isMuted { return }
        let trackName = (gameState == .bossFight) ? "boss_theme.mp3" : "bgm.mp3"
        let bgm = SKAudioNode(fileNamed: trackName)
        bgm.autoplayLooped = true
        addChild(bgm)
        bgmNode = bgm
        bgmNode?.run(SKAction.changeVolume(to: GameAudioSettings.shared.bgmVolume, duration: 0))
    }
    
    func playSFX(_ name: String) {
        if GameAudioSettings.shared.isMuted { return }
        self.run(SKAction.playSoundFileNamed(name, waitForCompletion: false))
    }
    
    func colorForLevel(_ level: Int) -> UIColor {
        switch level {
        case 1...2:   return .systemCyan
        case 3...4:   return .systemGreen
        case 5:       return .systemPurple
        case 6...7:   return .systemYellow
        case 8...9:   return .orange
        default:      return .systemRed
        }
    }
    
    override func didMove(to view: SKView) {
        self.anchorPoint = .zero
        self.physicsWorld.gravity = CGVector(dx: 0, dy: 0)
        self.physicsWorld.contactDelegate = self
        view.isMultipleTouchEnabled = true
        setupGameLayout()
    }
    
    private func setupGameLayout() {
        backgroundColor = UIColor(red: 0.03, green: 0.03, blue: 0.06, alpha: 1.0)
        
        p1StatLabel.fontSize = 20
        p1StatLabel.zPosition = 110
        if p1StatLabel.parent == nil { addChild(p1StatLabel) }
        
        p2StatLabel.fontSize = 20
        p2StatLabel.zPosition = 110
        if p2StatLabel.parent == nil { addChild(p2StatLabel) }
        
        highScoreLabel.fontSize = 14
        highScoreLabel.fontColor = .darkGray
        highScoreLabel.position = CGPoint(x: 100, y: 32)
        if highScoreLabel.parent == nil { addChild(highScoreLabel) }
        
        comboLabel.fontSize = 25
        comboLabel.fontColor = .systemYellow
        comboLabel.position = CGPoint(x: size.width - 120, y: 32)
        comboLabel.isHidden = true
        if comboLabel.parent == nil { addChild(comboLabel) }
        
        infoLabel.fontSize = 55
        infoLabel.position = CGPoint(x: size.width / 2, y: size.height / 2)
        infoLabel.isHidden = true
        if infoLabel.parent == nil { addChild(infoLabel) }
        
        p1ShieldVisual.fillColor = UIColor.cyan.withAlphaComponent(0.35)
        p1ShieldVisual.strokeColor = .cyan
        p1ShieldVisual.isHidden = true
        p1Paddle.addChild(p1ShieldVisual)
        p1ShieldVisual.position = CGPoint(x: 0, y: 15)
        
        p2ShieldVisual.fillColor = UIColor.systemPink.withAlphaComponent(0.35)
        p2ShieldVisual.strokeColor = .systemPink
        p2ShieldVisual.isHidden = true
        p2Paddle.addChild(p2ShieldVisual)
        p2ShieldVisual.position = CGPoint(x: 0, y: -15)
        
        if p1Paddle.parent == nil { addChild(p1Paddle) }
        if p2Paddle.parent == nil { addChild(p2Paddle) }
    }
    
    func startMode(_ mode: GameMode) {
        currentMode = mode
        gameState = .playing
        bossPhase = .phase1
        currentLevel = 1
        p1Score = 0
        p2Score = 0
        
        let talents = GameStorageManager.shared.loadTalents()
        let hpBonus = talents.first(where: { $0.id == "p_hp" })?.currentLevel ?? 0
        let widthBonusLevel = talents.first(where: { $0.id == "p_width" })?.currentLevel ?? 0
        
        maxHealth = 5 + hpBonus
        p1Health = maxHealth
        p2Health = 5
        currentCombo = 0
        
        liveStats = MatchRecord()
        matchStartTime = CACurrentMediaTime()
        
        p1Touch = nil
        p2Touch = nil
        p1ShieldActive = false
        p2ShieldActive = false
        magnetActive = false
        doubleDamageActive = false
        timeWarpActive = false
        
        p1ShieldVisual.isHidden = true
        p2ShieldVisual.isHidden = true
        
        let equippedSkin = GameStorageManager.shared.getCurrentSkin()
        p1Paddle.color = equippedSkin.color
        
        let baseWidth: CGFloat = 150.0 * (1.0 + Double(widthBonusLevel) * 0.08)
        p1Paddle.size = CGSize(width: baseWidth, height: 22)
        p2Paddle.size = CGSize(width: 150, height: 22)
        
        infoLabel.isHidden = true
        comboLabel.isHidden = true
        onStateChange?(false)
        
        removeBossNodes() // ✨ 錯誤 5 已成功對齊呼叫
        clearAllObjects()
        playBGM()
        
        p2Paddle.isHidden = (mode == .single)
        p1Paddle.position = CGPoint(x: size.width / 2, y: 120)
        p2Paddle.position = CGPoint(x: size.width / 2, y: size.height - 120)
        
        configureBoundaries(isBattle: mode != .single)
        if mode != .single { createDynamicObstacle() }
        
        bindPhysics(to: p1Paddle)
        bindPhysics(to: p2Paddle)
        updateUserInterface()
        launchSpawnLoop()
    }
    
    private func configureBoundaries(isBattle: Bool) {
        self.physicsBody = nil
        if !isBattle {
            let borderPath = CGMutablePath()
            borderPath.move(to: CGPoint(x: 0, y: -60))
            borderPath.addLine(to: CGPoint(x: 0, y: size.height))
            borderPath.addLine(to: CGPoint(x: size.width, y: size.height))
            borderPath.addLine(to: CGPoint(x: size.width, y: -60))
            
            let boundaryBody = SKPhysicsBody(edgeChainFrom: borderPath)
            boundaryBody.categoryBitMask = wallCat
            boundaryBody.restitution = 1.0
            boundaryBody.friction = 0
            self.physicsBody = boundaryBody
        } else {
            let expandedBounds = CGRect(x: 0, y: -130, width: size.width, height: size.height + 260)
            let boundaryBody = SKPhysicsBody(edgeLoopFrom: expandedBounds)
            boundaryBody.categoryBitMask = wallCat
            boundaryBody.restitution = 1.0
            boundaryBody.friction = 0
            self.physicsBody = boundaryBody
        }
    }
    
    private func clearAllObjects() {
        self.removeAction(forKey: "spawning")
        self.removeAction(forKey: "bossFiring")
        let targets = ["ball", "item", "obstacle", "coin", "boss_bullet"]
        for key in targets { enumerateChildNodes(withName: key) { node, _ in node.removeFromParent() } }
    }
    
    private func bindPhysics(to paddle: SKSpriteNode) {
        paddle.physicsBody = SKPhysicsBody(rectangleOf: CGSize(width: paddle.size.width, height: paddle.size.height), center: .zero)
        paddle.physicsBody?.isDynamic = false
        paddle.physicsBody?.categoryBitMask = paddleCat
        paddle.physicsBody?.contactTestBitMask = (currentMode == .single) ? (ballCat | itemCat | coinCat) : (ballCat | itemCat | coinCat | bossBulletCat)
    }
    
    private func launchSpawnLoop() {
        self.removeAction(forKey: "spawning")
        let interval = max(0.28, 0.85 - Double(currentLevel - 1) * 0.06)
        let spawn = SKAction.run { [weak self] in self?.generateTickObject() }
        let seq = SKAction.sequence([spawn, SKAction.wait(forDuration: interval)])
        run(SKAction.repeatForever(seq), withKey: "spawning")
    }
    
    private func generateTickObject() {
        let posX = CGFloat.random(in: 60...(size.width - 60))
        let velocityScaler = 1.0 + (Double(currentLevel - 1) * 0.12)
        var speedY = CGFloat(480.0 * velocityScaler)
        if timeWarpActive { speedY *= 0.55 }
        
        let spawnY = size.height - 60
        
        if currentMode == .single {
            let roll = CGFloat.random(in: 0...1)
            if roll < 0.16 {
                generateRandomItem(at: CGPoint(x: posX, y: spawnY))
            } else if roll < 0.35 {
                generateGoldCoin(at: CGPoint(x: posX, y: spawnY))
            } else {
                generateBall(at: CGPoint(x: posX, y: spawnY), velocity: CGVector(dx: CGFloat.random(in: -40...40), dy: -speedY), isBattle: false)
            }
        } else {
            if self.children.filter({ $0.name == "ball" }).isEmpty {
                let launchY = Bool.random() ? speedY : -speedY
                generateBall(at: CGPoint(x: size.width / 2, y: size.height / 2), velocity: CGVector(dx: CGFloat.random(in: -200...200), dy: launchY), isBattle: true)
            }
        }
    }
    
    private func generateBall(at pos: CGPoint, velocity: CGVector, isBattle: Bool) {
        let ball = SKShapeNode(circleOfRadius: 15)
        ball.name = "ball"
        ball.fillColor = colorForLevel(currentLevel)
        ball.strokeColor = .white
        ball.glowWidth = 3
        ball.position = pos
        
        ball.physicsBody = SKPhysicsBody(circleOfRadius: 15)
        ball.physicsBody?.categoryBitMask = ballCat
        ball.physicsBody?.linearDamping = 0
        ball.physicsBody?.angularDamping = 0
        ball.physicsBody?.usesPreciseCollisionDetection = true
        
        if isBattle {
            ball.physicsBody?.collisionBitMask = paddleCat | wallCat | obstacleCat
            ball.physicsBody?.contactTestBitMask = paddleCat
            ball.physicsBody?.restitution = 1.0
            ball.physicsBody?.friction = 0
        } else {
            ball.physicsBody?.collisionBitMask = 0
            ball.physicsBody?.contactTestBitMask = paddleCat
        }
        
        addChild(ball)
        ball.physicsBody?.velocity = velocity
    }
    
    private func generateGoldCoin(at pos: CGPoint) {
        let coin = SKShapeNode(circleOfRadius: 11)
        coin.name = "coin"
        coin.fillColor = .systemYellow
        coin.strokeColor = .orange
        coin.glowWidth = 2
        coin.position = pos
        
        coin.physicsBody = SKPhysicsBody(circleOfRadius: 11)
        coin.physicsBody?.categoryBitMask = coinCat
        coin.physicsBody?.collisionBitMask = 0
        coin.physicsBody?.contactTestBitMask = paddleCat
        coin.physicsBody?.velocity = CGVector(dx: 0, dy: timeWarpActive ? -110 : -220)
        
        addChild(coin)
    }
    
    private func createDynamicObstacle() {
        let block = SKSpriteNode(color: .darkGray, size: CGSize(width: 140, height: 18))
        block.name = "obstacle"
        block.position = CGPoint(x: size.width / 2, y: size.height / 2)
        block.physicsBody = SKPhysicsBody(rectangleOf: block.size)
        block.physicsBody?.isDynamic = false
        block.physicsBody?.categoryBitMask = obstacleCat
        addChild(block)
        
        block.run(SKAction.repeatForever(SKAction.rotate(byAngle: -.pi, duration: 4.5)))
    }
    
    private func generateRandomItem(at pos: CGPoint) {
        guard let picked = ItemType.allCases.randomElement() else { return }
        let item = SKShapeNode(circleOfRadius: 14)
        item.name = "item"
        item.position = pos
        item.strokeColor = .white
        
        switch picked {
        case .heal:          item.fillColor = .systemRed
        case .shield:        item.fillColor = .systemPurple
        case .enlarge:       item.fillColor = .systemBlue
        case .speedUp:       item.fillColor = .systemOrange
        case .magnet:        item.fillColor = .systemGreen
        case .coinPack:      item.fillColor = .yellow
        case .DoubleDamage:  item.fillColor = .magenta
        case .TimeWarp:      item.fillColor = .white
        }
        
        item.physicsBody = SKPhysicsBody(circleOfRadius: 14)
        item.physicsBody?.categoryBitMask = itemCat
        item.physicsBody?.collisionBitMask = 0
        item.physicsBody?.contactTestBitMask = paddleCat
        item.physicsBody?.velocity = CGVector(dx: 0, dy: timeWarpActive ? -90 : -180)
        item.userData = ["itemType": "\(picked)"]
        
        addChild(item)
    }
    
    // =========================================================================
    // MARK: - 7. 三階段進化 Boss 系統與移除器 (Advanced Boss AI Engine)
    // =========================================================================
    
    // ✨ 錯誤 5 已修正：補上原本遺漏的 Boss 節點與記憶體清理函式
    private func removeBossNodes() {
        bossNode?.removeFromParent()
        bossNode = nil
        bossShieldNode?.removeFromParent()
        bossShieldNode = nil
        bossHealthBarBackground?.removeFromParent()
        bossHealthBarBackground = nil
        bossHealthBarForeground?.removeFromParent()
        bossHealthBarForeground = nil
        self.removeAction(forKey: "bossFiring")
    }
    
    private func triggerBossFightSequence() {
        gameState = .bossFight
        bossPhase = .phase1
        clearAllObjects()
        playBGM()
        
        bossMaxHealth = 80 + (currentLevel * 20)
        bossCurrentHealth = bossMaxHealth
        
        let boss = SKSpriteNode(color: .systemRed, size: CGSize(width: 190, height: 45))
        boss.name = "obstacle"
        boss.position = CGPoint(x: size.width / 2, y: size.height - 160)
        boss.physicsBody = SKPhysicsBody(rectangleOf: boss.size)
        boss.physicsBody?.isDynamic = false
        boss.physicsBody?.categoryBitMask = obstacleCat
        addChild(boss)
        bossNode = boss
        
        let barBg = SKShapeNode(rectOf: CGSize(width: size.width - 160, height: 12), cornerRadius: 4)
        barBg.fillColor = .black
        barBg.strokeColor = .gray
        barBg.position = CGPoint(x: size.width / 2, y: size.height - 90)
        barBg.zPosition = 120
        addChild(barBg)
        bossHealthBarBackground = barBg
        
        let barFg = SKShapeNode(rectOf: CGSize(width: size.width - 160, height: 12), cornerRadius: 4)
        barFg.fillColor = .systemRed
        barFg.strokeColor = .clear
        barFg.position = CGPoint(x: size.width / 2, y: size.height - 90)
        barFg.zPosition = 121
        addChild(barFg)
        bossHealthBarForeground = barFg
        
        let fireLoop = SKAction.run { [weak self] in self?.executeBossAIBehaviorTree() }
        let delay = SKAction.wait(forDuration: 0.1)
        run(SKAction.repeatForever(SKAction.sequence([fireLoop, delay])), withKey: "bossFiring")
        
        launchSpawnLoop()
    }
    
    private func executeBossAIBehaviorTree() {
        guard let boss = bossNode else { return }
        bossTimeTracker += 0.1
        
        let healthRatio = Double(bossCurrentHealth) / Double(bossMaxHealth)
        
        if healthRatio > 0.65 {
            bossPhase = .phase1
        } else if healthRatio > 0.30 {
            if bossPhase == .phase1 {
                bossPhase = .phase2
                playSFX("levelup.wav")
                deployBossShieldVisual()
            }
        } else {
            if bossPhase != .phase3 {
                bossPhase = .phase3
                playSFX("boss_dead.wav")
                bossShieldNode?.removeFromParent()
                boss.run(SKAction.moveTo(x: size.width / 2, duration: 1.0))
            }
        }
        
        if bossPhase == .phase1 {
            let speedX: CGFloat = 4.0 + CGFloat(currentLevel) * 0.3
            boss.position.x += speedX * bossDirection
            if boss.position.x > size.width - 100 { bossDirection = -1.0 }
            if boss.position.x < 100 { bossDirection = 1.0 }
            
            if Int(bossTimeTracker * 10) % 12 == 0 {
                spawnStandardPattern(from: boss.position, angle: .pi * 1.5)
            }
        }
        else if bossPhase == .phase2 {
            let speedX: CGFloat = 6.5 + CGFloat(currentLevel) * 0.4
            boss.position.x += speedX * bossDirection
            if boss.position.x > size.width - 100 { bossDirection = -1.0 }
            if boss.position.x < 100 { bossDirection = 1.0 }
            
            bossShieldNode?.position = boss.position
            
            if Int(bossTimeTracker * 10) % 15 == 0 {
                spawnTargetingPattern(from: boss.position)
            }
        }
        else if bossPhase == .phase3 {
            let waveAngle = bossTimeTracker * 4.5
            if Int(bossTimeTracker * 10) % 3 == 0 {
                spawnRadialHelixPattern(from: boss.position, centralAngle: waveAngle)
            }
        }
    }
    
    private func deployBossShieldVisual() {
        guard let boss = bossNode else { return }
        let shield = SKShapeNode(circleOfRadius: 110)
        shield.strokeColor = .systemPurple
        shield.lineWidth = 4.0
        shield.fillColor = UIColor.purple.withAlphaComponent(0.15)
        shield.position = boss.position
        shield.zPosition = boss.zPosition - 1
        addChild(shield)
        bossShieldNode = shield
    }
    
    private func spawnStandardPattern(from pos: CGPoint, angle: Double) {
        createBulletNode(at: pos, velocity: CGVector(dx: cos(angle) * 350.0, dy: sin(angle) * 350.0))
    }
    
    private func spawnTargetingPattern(from pos: CGPoint) {
        let targetX = p1Paddle.position.x
        let targetY = p1Paddle.position.y
        let deltaX = targetX - pos.x
        let deltaY = targetY - pos.y
        let angle = atan2(deltaY, deltaX)
        
        createBulletNode(at: CGPoint(x: pos.x - 30, y: pos.y), velocity: CGVector(dx: cos(angle) * 400, dy: sin(angle) * 400))
        createBulletNode(at: CGPoint(x: pos.x + 30, y: pos.y), velocity: CGVector(dx: cos(angle) * 400, dy: sin(angle) * 400))
    }
    
    private func spawnRadialHelixPattern(from pos: CGPoint, centralAngle: Double) {
        let ways = 4
        for i in 0..<ways {
            let offsetAngle = centralAngle + (Double(i) * (.pi * 2.0 / Double(ways)))
            let speed: CGFloat = 280.0
            let dx = cos(offsetAngle) * speed
            let dy = sin(offsetAngle) * speed
            
            if dy < 0 {
                createBulletNode(at: pos, velocity: CGVector(dx: dx, dy: dy), useSinWave: true)
            }
        }
    }
    
    private func createBulletNode(at pos: CGPoint, velocity: CGVector, useSinWave: Bool = false) {
        liveStats.totalShotsFiredByBoss += 1
        let bullet = SKShapeNode(circleOfRadius: 9)
        bullet.name = "boss_bullet"
        bullet.fillColor = (bossPhase == .phase3) ? .purple : .systemRed
        bullet.strokeColor = .yellow
        bullet.position = pos
        
        bullet.physicsBody = SKPhysicsBody(circleOfRadius: 9)
        bullet.physicsBody?.categoryBitMask = bossBulletCat
        bullet.physicsBody?.collisionBitMask = 0
        bullet.physicsBody?.contactTestBitMask = paddleCat
        
        var vel = velocity
        if timeWarpActive { vel = CGVector(dx: vel.dx * 0.5, dy: vel.dy * 0.5) }
        bullet.physicsBody?.velocity = vel
        
        addChild(bullet)
        
        if useSinWave {
            let baseVelX = vel.dx
            let action = SKAction.repeatForever(SKAction.sequence([
                SKAction.run {
                    let t = CACurrentMediaTime()
                    let waveX = sin(t * 12.0) * 160.0
                    bullet.physicsBody?.velocity.dx = baseVelX + waveX
                },
                SKAction.wait(forDuration: 0.05)
            ]))
            bullet.run(action)
        }
    }
    
    private func updateBossHealthVisual() {
        guard let barBg = bossHealthBarBackground else { return }
        bossHealthBarForeground?.removeFromParent()
        
        let ratio = CGFloat(max(0, bossCurrentHealth)) / CGFloat(bossMaxHealth)
        let targetWidth = (size.width - 160) * ratio
        
        if targetWidth > 0 {
            let newFg = SKShapeNode(rectOf: CGSize(width: targetWidth, height: 12), cornerRadius: 4)
            newFg.fillColor = (bossPhase == .phase3) ? .purple : .systemRed
            newFg.strokeColor = .clear
            newFg.position = barBg.position
            newFg.zPosition = 121
            addChild(newFg)
            bossHealthBarForeground = newFg
        }
        
        if bossCurrentHealth <= 0 { executeBossDefeated() }
    }
    
    private func executeBossDefeated() {
        playSFX("boss_dead.wav")
        if let bossPos = bossNode?.position {
            ParticleEngine.createExplosion(at: bossPos, color: .yellow, in: self, sizeScale: 2.0)
            ParticleEngine.createExplosion(at: CGPoint(x: bossPos.x - 40, y: bossPos.y), color: .orange, in: self, sizeScale: 1.5)
            ParticleEngine.createExplosion(at: CGPoint(x: bossPos.x + 40, y: bossPos.y), color: .red, in: self, sizeScale: 1.5)
            ParticleEngine.createShockwave(at: bossPos, in: self, radius: 250.0)
        }
        
        p1Score += 20
        GameStorageManager.shared.addCoins(80)
        liveStats.goldEarnedThisRound += 80
        GameAudioSettings.shared.refreshCoins()
        
        if GameStorageManager.shared.unlockAchievement("boss_slayer") {
            GameAudioSettings.shared.triggerAchievementPopup("🏆 斬魔者：擊敗守關巨頭")
        }
        
        currentLevel += 1
        gameState = .playing
        removeBossNodes()
        updateUserInterface()
        launchSpawnLoop()
    }
    
    // =========================================================================
    // MARK: - 8. 碰撞生命體攔截分流器 (Advanced Physics Resolver)
    // =========================================================================
    
    func didBegin(_ contact: SKPhysicsContact) {
        let bitSummary = contact.bodyA.categoryBitMask | contact.bodyB.categoryBitMask
        
        if bitSummary == (ballCat | paddleCat) {
            let ball = (contact.bodyA.categoryBitMask == ballCat) ? contact.bodyA.node : contact.bodyB.node
            
            if currentMode == .single {
                if let bNode = ball {
                    playSFX("hit.wav")
                    ParticleEngine.createExplosion(at: bNode.position, color: colorForLevel(currentLevel), in: self)
                    
                    let talents = GameStorageManager.shared.loadTalents()
                    let pCrit = talents.first(where: { $0.id == "p_crit" })?.currentLevel ?? 0
                    let isCrit = Double.random(in: 0...1) < (Double(pCrit) * 0.05)
                    
                    let damageMultiplier = doubleDamageActive ? 2 : 1
                    let scoreEarned = (isCrit ? 2 : 1) * damageMultiplier
                    
                    bNode.removeFromParent()
                    p1Score += scoreEarned
                    currentCombo += 1
                    liveStats.totalBallsDeflected += 1
                    liveStats.highestComboValue = max(liveStats.highestComboValue, currentCombo)
                    
                    if abs(bNode.position.x - p1Paddle.position.x) < 30 {
                        liveStats.perfectCatches += 1
                        ParticleEngine.createShockwave(at: p1Paddle.position, in: self, radius: 60.0)
                    }
                    
                    if gameState == .bossFight {
                        let finalDamage = (5 + min(5, currentCombo / 3)) * damageMultiplier
                        bossCurrentHealth -= finalDamage
                        liveStats.bossDamageDealt += finalDamage
                        updateBossHealthVisual()
                    }
                    updateUserInterface()
                }
            } else {
                playSFX("hit.wav")
                if let bPos = ball?.position { ParticleEngine.createExplosion(at: bPos, color: .white, in: self) }
            }
        }
        else if bitSummary == (itemCat | paddleCat) {
            let item = (contact.bodyA.categoryBitMask == itemCat) ? contact.bodyA.node : contact.bodyB.node
            let paddle = (contact.bodyA.categoryBitMask == paddleCat) ? contact.bodyA.node as? SKSpriteNode : contact.bodyB.node as? SKSpriteNode
            
            if let pNode = paddle, let iNode = item, let typeStr = iNode.userData?.value(forKey: "itemType") as? String {
                playSFX("item.wav")
                liveStats.itemCollectedCount += 1
                ParticleEngine.createExplosion(at: iNode.position, color: .systemGreen, in: self)
                applyItemEffect(typeString: typeStr, to: pNode)
            }
            item?.removeFromParent()
            updateUserInterface()
        }
        else if bitSummary == (coinCat | paddleCat) {
            let coin = (contact.bodyA.categoryBitMask == coinCat) ? contact.bodyA.node : contact.bodyB.node
            if let cPos = coin?.position {
                playSFX("coin.wav")
                ParticleEngine.createExplosion(at: cPos, color: .systemYellow, in: self, sizeScale: 0.8)
                GameStorageManager.shared.addCoins(2)
                liveStats.goldEarnedThisRound += 2
                GameAudioSettings.shared.refreshCoins()
            }
            coin?.removeFromParent()
        }
        else if bitSummary == (bossBulletCat | paddleCat) {
            let bullet = (contact.bodyA.categoryBitMask == bossBulletCat) ? contact.bodyA.node : contact.bodyB.node
            bullet?.removeFromParent()
            
            if p1ShieldActive {
                p1ShieldActive = false
                p1ShieldVisual.isHidden = true
                playSFX("hit.wav")
            } else {
                playSFX("miss.wav")
                triggerScreenShake()
                p1Health -= 1
                currentCombo = 0
                updateUserInterface()
            }
        }
    }
    
    private func applyItemEffect(typeString: String, to paddle: SKSpriteNode) {
        let isP1 = (paddle == p1Paddle)
        let talents = GameStorageManager.shared.loadTalents()
        let magnetBonusLevel = talents.first(where: { $0.id == "p_magnet" })?.currentLevel ?? 0
        let magnetDuration = 8.0 + (Double(magnetBonusLevel) * 1.5)
        
        if typeString == "\(ItemType.heal)" {
            if isP1 { p1Health = min(maxHealth, p1Health + 1) } else { p2Health = min(maxHealth, p2Health + 1) }
        } else if typeString == "\(ItemType.shield)" {
            if isP1 { p1ShieldActive = true; p1ShieldVisual.isHidden = false } else { p2ShieldActive = true; p2ShieldVisual.isHidden = false }
        } else if typeString == "\(ItemType.enlarge)" {
            let targetW = paddle.size.width * 1.45
            paddle.run(SKAction.sequence([
                SKAction.resize(toWidth: targetW, duration: 0.12),
                SKAction.wait(forDuration: 7.0),
                SKAction.resize(toWidth: paddle.size.width, duration: 0.2)
            ]))
        } else if typeString == "\(ItemType.speedUp)" {
            enumerateChildNodes(withName: "ball") { node, _ in
                node.physicsBody?.velocity = CGVector(dx: (node.physicsBody?.velocity.dx ?? 0) * 1.3, dy: (node.physicsBody?.velocity.dy ?? 0) * 1.3)
            }
        } else if typeString == "\(ItemType.magnet)" {
            magnetActive = true
            run(SKAction.sequence([SKAction.wait(forDuration: magnetDuration), SKAction.run { [weak self] in self?.magnetActive = false }]))
        } else if typeString == "\(ItemType.coinPack)" {
            GameStorageManager.shared.addCoins(15)
            liveStats.goldEarnedThisRound += 15
            GameAudioSettings.shared.refreshCoins()
        } else if typeString == "\(ItemType.DoubleDamage)" {
            doubleDamageActive = true
            p1Paddle.run(SKAction.repeatForever(SKAction.sequence([SKAction.fadeAlpha(to: 0.4, duration: 0.2), SKAction.fadeAlpha(to: 1.0, duration: 0.2)])), withKey: "dmg_flash")
            run(SKAction.sequence([SKAction.wait(forDuration: 6.0), SKAction.run { [weak self] in
                self?.doubleDamageActive = false
                self?.p1Paddle.removeAction(forKey: "dmg_flash")
                self?.p1Paddle.alpha = 1.0
            }]))
        } else if typeString == "\(ItemType.TimeWarp)" {
            timeWarpActive = true
            backgroundColor = UIColor(red: 0.02, green: 0.05, blue: 0.05, alpha: 1.0)
            run(SKAction.sequence([SKAction.wait(forDuration: 5.0), SKAction.run { [weak self] in
                self?.timeWarpActive = false
                self?.backgroundColor = UIColor(red: 0.03, green: 0.03, blue: 0.06, alpha: 1.0)
            }]))
        }
    }
    
    // =========================================================================
    // MARK: - 9. 每幀物理迴圈更新機制 (Frame-Level Update Resolver)
    // =========================================================================
    
    override func update(_ currentTime: TimeInterval) {
        if gameState != .playing && gameState != .bossFight { return }
        
        if magnetActive {
            enumerateChildNodes(withName: "coin") { [weak self] node, _ in
                guard let self = self else { return }
                let deltaX = self.p1Paddle.position.x - node.position.x
                let deltaY = self.p1Paddle.position.y - node.position.y
                node.position.x += deltaX * 0.14
                node.position.y += deltaY * 0.14
            }
        }
        
        if currentMode == .aiBattle, let ball = childNode(withName: "ball") {
            let diff = ball.position.x - p2Paddle.position.x
            if ball.position.y > size.height * 0.35 {
                let speed = aiMaxSpeed + CGFloat(currentLevel) * 0.35
                let move = max(-speed, min(speed, diff * 0.14))
                p2Paddle.position.x = max(80, min(size.width - 80, p2Paddle.position.x + move))
            }
        }
        
        enumerateChildNodes(withName: "ball") { [weak self] ballNode, _ in
            guard let self = self else { return }
            ParticleEngine.createTrail(at: ballNode.position, in: self)
            
            if ballNode.position.y < -30 {
                ballNode.removeFromParent()
                self.currentCombo = 0
                
                if self.p1ShieldActive {
                    self.p1ShieldActive = false
                    self.p1ShieldVisual.isHidden = true
                    self.playSFX("hit.wav")
                } else {
                    self.playSFX("miss.wav")
                    self.triggerScreenShake()
                    self.p1Health -= 1
                    if self.currentMode != .single { self.p2Score += 1 }
                    self.updateUserInterface()
                }
            }
            else if ballNode.position.y > self.size.height + 30 {
                if self.currentMode != .single {
                    ballNode.removeFromParent()
                    if self.p2ShieldActive {
                        self.p2ShieldActive = false
                        self.p2ShieldVisual.isHidden = true
                    } else {
                        self.playSFX("miss.wav")
                        self.p1Score += 1
                        self.p2Health -= 1
                        self.updateUserInterface()
                    }
                }
            }
        }
        
        enumerateChildNodes(withName: "item") { node, _ in if node.position.y < -30 { node.removeFromParent() } }
        enumerateChildNodes(withName: "coin") { node, _ in if node.position.y < -30 { node.removeFromParent() } }
        enumerateChildNodes(withName: "boss_bullet") { node, _ in
            if node.position.y < -30 {
                node.removeFromParent()
                if self.gameState == .bossFight { self.liveStats.bossBulletsDodged += 1 }
            }
        }
        
        if currentMode == .single {
            if p1Health <= 0 { executeGameOverSequence() }
        } else {
            if p1Health <= 0 || p2Health <= 0 { executeGameOverSequence() }
        }
    }
    
    private func triggerScreenShake() {
        let shake = SKAction.sequence([
            SKAction.moveBy(x: 12, y: 12, duration: 0.03),
            SKAction.moveBy(x: -24, y: -24, duration: 0.03),
            SKAction.moveBy(x: 12, y: 12, duration: 0.03)
        ])
        self.run(shake)
    }
    
    private func executeGameOverSequence() {
        gameState = .gameOver
        infoLabel.isHidden = false
        bgmNode?.run(SKAction.stop())
        
        GameStorageManager.shared.saveHighScore(p1Score)
        
        let extraGold = p1Score / 2
        GameStorageManager.shared.addCoins(extraGold)
        liveStats.goldEarnedThisRound += extraGold
        liveStats.timeElapsed = CACurrentMediaTime() - matchStartTime
        GameAudioSettings.shared.refreshCoins()
        
        if p1Score >= 50 && GameStorageManager.shared.unlockAchievement("score_50") {
            GameAudioSettings.shared.triggerAchievementPopup("🏆 半百大師：單局斬獲 50 分")
        }
        if liveStats.highestComboValue >= 12 && GameStorageManager.shared.unlockAchievement("combo_12") {
            GameAudioSettings.shared.triggerAchievementPopup("🏆 狂熱律動：達成 12 連擊以上")
        }
        
        infoLabel.text = p1Health <= 0 ? (currentMode == .single ? "GAME OVER" : "P2 勝利!") : "P1 獲勝!"
        onStateChange?(true)
        onMatchComplete?(liveStats)
    }
    
    func updateUserInterface() {
        let hearts = String(repeating: "❤️", count: max(0, p1Health))
        
        if currentMode == .single {
            if gameState != .bossFight {
                let nextLevel = (p1Score / 6) + 1
                if nextLevel > currentLevel {
                    currentLevel = nextLevel
                    if currentLevel % 5 == 0 {
                        triggerBossFightSequence()
                    } else {
                        displayLevelUpBanner()
                        launchSpawnLoop()
                    }
                }
            }
        }
        
        if currentCombo >= 3 {
            comboLabel.text = "連擊: \(currentCombo) 🔥"
            comboLabel.isHidden = false
        } else {
            comboLabel.isHidden = true
        }
        
        if currentMode == .single {
            p1StatLabel.horizontalAlignmentMode = .center
            if gameState == .bossFight {
                p1StatLabel.text = "🚨 BOSS 戰 (Level \(currentLevel)) 🚨  |  Boss血量: \(bossCurrentHealth)/\(bossMaxHealth)  |  生命: \(hearts)"
                p1StatLabel.fontColor = .systemRed
            } else {
                p1StatLabel.text = "關卡等級: \(currentLevel)  |  得分: \(p1Score)  |  血量: \(hearts)"
                p1StatLabel.fontColor = colorForLevel(currentLevel)
            }
            p1StatLabel.position = CGPoint(x: size.width / 2, y: size.height - 70)
            p2StatLabel.isHidden = true
        } else {
            p1StatLabel.horizontalAlignmentMode = .left
            p1StatLabel.text = "P1 分數: \(p1Score) | \(hearts)"
            p1StatLabel.fontColor = .systemCyan
            p1StatLabel.position = CGPoint(x: 45, y: 55)
            
            p2StatLabel.horizontalAlignmentMode = .left
            p2StatLabel.text = "P2 分數: \(p2Score) | \(String(repeating: "❤️", count: max(0, p2Health)))"
            p2StatLabel.fontColor = .systemPink
            p2Paddle.isHidden = false
            p2StatLabel.position = CGPoint(x: 45, y: size.height - 70)
            p2StatLabel.isHidden = false
        }
    }
    
    private func displayLevelUpBanner() {
        playSFX("levelup.wav")
        let col = colorForLevel(currentLevel)
        let flash = SKSpriteNode(color: col, size: self.size)
        flash.position = CGPoint(x: size.width / 2, y: size.height / 2)
        flash.zPosition = 40
        flash.alpha = 0
        addChild(flash)
        flash.run(SKAction.sequence([SKAction.fadeIn(withDuration: 0.05), SKAction.fadeOut(withDuration: 0.15), SKAction.removeFromParent()]))
        
        let banner = SKLabelNode(text: "LEVEL UP 🚀 關卡 \(currentLevel)")
        banner.fontName = "AvenirNext-Heavy"
        banner.fontSize = 50
        banner.fontColor = col
        banner.position = CGPoint(x: size.width / 2, y: size.height / 2 + 50)
        banner.zPosition = 140
        addChild(banner)
        
        banner.run(SKAction.sequence([
            SKAction.group([SKAction.moveBy(x: 0, y: 75, duration: 0.6), SKAction.fadeOut(withDuration: 0.6)]),
            SKAction.removeFromParent()
        ]))
    }
    
    override func touchesBegan(_ touches: Set<UITouch>, with event: UIEvent?) {
        for t in touches {
            let loc = t.location(in: self)
            if loc.y < size.height * 0.5 { p1Touch = t }
            else if currentMode == .pvp { p2Touch = t }
        }
    }
    
    override func touchesMoved(_ touches: Set<UITouch>, with event: UIEvent?) {
        if let t1 = p1Touch { p1Paddle.position.x = max(75, min(size.width - 75, t1.location(in: self).x)) }
        if currentMode == .pvp, let t2 = p2Touch { p2Paddle.position.x = max(75, min(size.width - 75, t2.location(in: self).x)) }
    }
    
    override func touchesEnded(_ touches: Set<UITouch>, with event: UIEvent?) {
        for t in touches {
            if t == p1Touch { p1Touch = nil }
            if t == p2Touch { p2Touch = nil }
        }
    }
}

// =========================================================================
// MARK: - 10. SwiftUI 高級賽博龐克視覺介面層 (Immersive Cyberpunk Canvas)
// =========================================================================

struct ContentView: View {
    @State private var viewStatus: String = "menu"
    @ObservedObject var audioSettings = GameAudioSettings.shared
    @State private var recentRecord = MatchRecord()
    
    let gameScene = GameScene()
    
    var body: some View {
        ZStack {
            SpriteView(scene: gameScene)
                .ignoresSafeArea()
                .onAppear {
                    gameScene.scaleMode = .resizeFill
                    gameScene.onStateChange = { isOver in if isOver { withAnimation { viewStatus = "stats" } } }
                    gameScene.onMatchComplete = { record in self.recentRecord = record }
                }
            
            if viewStatus == "menu" {
                Rectangle().fill(.ultraThinMaterial).ignoresSafeArea()
                mainMenuOverlayView
            } else if viewStatus == "shop" {
                Rectangle().fill(.ultraThinMaterial).ignoresSafeArea()
                gameShopOverlayView
            } else if viewStatus == "talents" {
                Rectangle().fill(.ultraThinMaterial).ignoresSafeArea()
                rpgTalentTreeView
            } else if viewStatus == "achievements" {
                Rectangle().fill(.ultraThinMaterial).ignoresSafeArea()
                achievementOverlayView
            } else if viewStatus == "stats" {
                Rectangle().fill(.ultraThinMaterial).ignoresSafeArea()
                matchStatsOverlayView
            }
            
            if let badgeName = audioSettings.newlyUnlockedAchievement {
                VStack {
                    HStack(spacing: 12) {
                        Image(systemName: "trophy.circle.fill")
                            .font(.largeTitle)
                            .foregroundColor(.yellow)
                        Text(badgeName)
                            .font(.headline)
                            .foregroundColor(.white)
                    }
                    .padding()
                    .background(Color.purple.opacity(0.85))
                    .cornerRadius(20)
                    .shadow(radius: 10)
                    .transition(.move(edge: .top).combined(with: .opacity))
                    .padding(.top, 40)
                    Spacer()
                }
                .animation(.spring(), value: badgeName)
            }
        }
    }
    
    private var mainMenuOverlayView: some View {
        VStack(spacing: 20) {
            HStack {
                Spacer()
                HStack(spacing: 6) {
                    Image(systemName: "bitcoinsign.circle.fill").foregroundColor(.yellow)
                    Text("\(audioSettings.goldDisplay)").font(.system(.body, design: .monospaced).bold()).foregroundColor(.white)
                }
                .padding(.horizontal, 16).padding(.vertical, 8)
                .background(Color.white.opacity(0.1)).cornerRadius(20).padding(.trailing, 24)
            }
            
            Text("PADDLE STRIKE")
                .font(.system(size: 44, weight: .black, design: .rounded))
                .foregroundColor(.white)
            // ✨ 錯誤 1 已修正：使用 .opacity(0.65) 取代原本報錯的 withAlphaComponent
                .shadow(color: Color.cyan.opacity(0.65), radius: 12)
            
            Text("最高紀錄: \(GameStorageManager.shared.getHighScore()) 分")
                .font(.subheadline.bold()).foregroundColor(.yellow)
            
            Spacer().frame(height: 5)
            
            ForEach(GameMode.allCases, id: \.self) { mode in
                Button(action: {
                    withAnimation { viewStatus = "game" }
                    gameScene.startMode(mode)
                }) {
                    Text(mode.rawValue).font(.title3.bold())
                        .frame(width: 260, height: 55)
                        .background(buttonColor(for: mode)).foregroundColor(.white).cornerRadius(16).shadow(radius: 5)
                }
            }
            
            HStack(spacing: 12) {
                Button(action: { withAnimation { viewStatus = "shop" } }) {
                    Label("外觀商城", systemImage: "bag.fill").font(.footnote.bold()).foregroundColor(.white)
                        .padding(.horizontal, 14).padding(.vertical, 12).background(Color.white.opacity(0.12)).cornerRadius(12)
                }
                Button(action: { withAnimation { viewStatus = "talents" } }) {
                    Label("永久天賦", systemImage: "bolt.tree.fill").font(.footnote.bold()).foregroundColor(.white)
                        .padding(.horizontal, 14).padding(.vertical, 12).background(Color.white.opacity(0.12)).cornerRadius(12)
                }
                Button(action: { withAnimation { viewStatus = "achievements" } }) {
                    Label("榮譽勳章", systemImage: "trophy.fill").font(.footnote.bold()).foregroundColor(.white)
                        .padding(.horizontal, 14).padding(.vertical, 12).background(Color.white.opacity(0.12)).cornerRadius(12)
                }
            }
            
            VStack(spacing: 6) {
                Toggle("全域靜音模式", isOn: $audioSettings.isMuted).tint(.purple).foregroundColor(.white)
                if !audioSettings.isMuted {
                    HStack {
                        Image(systemName: "music.note")
                        Slider(value: $audioSettings.bgmVolume, in: 0...1)
                    }.foregroundColor(.gray)
                }
            }
            .padding().background(Color.black.opacity(0.2)).cornerRadius(16).frame(width: 320)
        }
    }
    
    private var gameShopOverlayView: some View {
        VStack(spacing: 20) {
            HStack {
                Button("返回主選單") { withAnimation { viewStatus = "menu" } }.foregroundColor(.cyan)
                Spacer()
                Text("💰 \(audioSettings.goldDisplay)").foregroundColor(.yellow).font(.headline)
            }.padding()
            
            Text("科技外觀核心商店")
                .font(.largeTitle.bold()).foregroundColor(.white)
            
            ScrollView {
                VStack(alignment: .leading, spacing: 14) {
                    Text("🛡️ 護甲擋板矩陣Skin").font(.headline).foregroundColor(.gray)
                    ForEach(PaddleSkin.allCases, id: \.self) { skin in
                        let unlocked = GameStorageManager.shared.getUnlockedSkins().contains(skin)
                        let equipped = GameStorageManager.shared.getCurrentSkin() == skin
                        HStack {
                            Circle().fill(Color(skin.color)).frame(width: 18, height: 18)
                            Text(skin.rawValue).foregroundColor(.white)
                            Spacer()
                            if equipped { Text("裝備中").foregroundColor(.green) }
                            else if unlocked {
                                Button("裝配") { GameStorageManager.shared.setCurrentSkin(skin); audioSettings.refreshCoins() }.foregroundColor(.cyan)
                            } else {
                                Button("\(skin.cost) 💰") {
                                    if GameStorageManager.shared.spendCoins(skin.cost) {
                                        GameStorageManager.shared.unlockSkin(skin); GameStorageManager.shared.setCurrentSkin(skin); audioSettings.refreshCoins()
                                    }
                                }.padding(.horizontal, 12).padding(.vertical, 6).background(Color.yellow.opacity(0.18)).cornerRadius(10).foregroundColor(.yellow)
                            }
                        }.padding().background(Color.white.opacity(0.05)).cornerRadius(12)
                    }
                    
                    Text("☄️ 球體彗星量子尾跡Trail").font(.headline).foregroundColor(.gray).padding(.top, 8)
                    ForEach(TrailEffect.allCases, id: \.self) { trail in
                        let unlocked = GameStorageManager.shared.getUnlockedTrails().contains(trail)
                        let equipped = GameStorageManager.shared.getCurrentTrail() == trail
                        HStack {
                            Circle().fill(Color(trail.color)).frame(width: 18, height: 18)
                            Text(trail.rawValue).foregroundColor(.white)
                            Spacer()
                            if equipped { Text("裝備中").foregroundColor(.green) }
                            else if unlocked {
                                Button("裝配") { GameStorageManager.shared.setCurrentTrail(trail); audioSettings.refreshCoins() }.foregroundColor(.cyan)
                            } else {
                                Button("\(trail.cost) 💰") {
                                    if GameStorageManager.shared.spendCoins(trail.cost) {
                                        GameStorageManager.shared.unlockTrail(trail); GameStorageManager.shared.setCurrentTrail(trail); audioSettings.refreshCoins()
                                    }
                                }.padding(.horizontal, 12).padding(.vertical, 6).background(Color.yellow.opacity(0.18)).cornerRadius(10).foregroundColor(.yellow)
                            }
                        }.padding().background(Color.white.opacity(0.05)).cornerRadius(12)
                    }
                }.padding(.horizontal)
            }
        }
    }
    
    private var rpgTalentTreeView: some View {
        VStack(spacing: 20) {
            HStack {
                Button("返回主選單") { withAnimation { viewStatus = "menu" } }.foregroundColor(.cyan)
                Spacer()
                Text("微晶片晶圓: \(audioSettings.goldDisplay) 💰").foregroundColor(.yellow).font(.subheadline.bold())
            }.padding()
            
            Text("RPG 永久晶片天賦樹")
                .font(.title.bold()).foregroundColor(.white)
            
            ScrollView {
                VStack(spacing: 12) {
                    ForEach(audioSettings.globalTalents) { talent in
                        HStack {
                            VStack(alignment: .leading, spacing: 4) {
                                Text(talent.name).font(.headline).foregroundColor(.white)
                                Text(talent.description).font(.caption).foregroundColor(.gray)
                                Text("進度等級: \(talent.currentLevel) / \(talent.maxLevel)").font(.caption2).foregroundColor(.cyan)
                            }
                            Spacer()
                            if talent.currentLevel >= talent.maxLevel {
                                Text("核心極限").font(.footnote.bold()).foregroundColor(.purple)
                            } else {
                                Button(action: { audioSettings.updateTalentTree(nodeID: talent.id) }) {
                                    Text("\(talent.currentCost) 💰")
                                        .font(.footnote.bold()).padding(.horizontal, 14).padding(.vertical, 8)
                                        .background(audioSettings.goldDisplay >= talent.currentCost ? Color.cyan.opacity(0.2) : Color.gray.opacity(0.2))
                                        .foregroundColor(audioSettings.goldDisplay >= talent.currentCost ? .cyan : .gray)
                                        .cornerRadius(10)
                                }
                                .disabled(audioSettings.goldDisplay < talent.currentCost)
                            }
                        }
                        .padding().background(Color.white.opacity(0.06)).cornerRadius(14)
                    }
                }.padding(.horizontal)
            }
        }
    }
    
    private var achievementOverlayView: some View {
        VStack(spacing: 20) {
            HStack {
                Button("返回主選單") { withAnimation { viewStatus = "menu" } }.foregroundColor(.cyan)
                Spacer()
            }.padding()
            Text("終身榮譽成就獎章").font(.largeTitle.bold()).foregroundColor(.white)
            List {
                AchievementRow(id: "boss_slayer", title: "斬魔者", desc: "單人闖關模式中成功粉碎守關 Boss 所有進化階段", icon: "bolt.shield.fill")
                AchievementRow(id: "score_50", title: "半百大師", desc: "在常規無盡模式中單局奪得 50 分以上", icon: "flame.fill")
                AchievementRow(id: "combo_12", title: "狂熱律動", desc: "在極限彈幕中打出 12 次以上的完美連擊Combo", icon: "music.note.list")
            }.listStyle(.plain).background(Color.clear)
        }
    }
    
    private var matchStatsOverlayView: some View {
        VStack(spacing: 22) {
            Text("戰局大數據分析面板")
                .font(.system(size: 32, weight: .black, design: .rounded)).foregroundColor(.white)
            
            VStack(spacing: 12) {
                StatItemRow(label: "光子攔截接球數", val: "\(recentRecord.totalBallsDeflected) 次")
                StatItemRow(label: "完美核心精準接球", val: "\(recentRecord.perfectCatches) 回 🎯")
                StatItemRow(label: "單局最高極限連擊", val: "\(recentRecord.highestComboValue) Combo 🔥")
                StatItemRow(label: "巨頭地獄彈幕閃避", val: "\(recentRecord.bossBulletsDodged) 顆")
                StatItemRow(label: "發射子彈總量計數", val: "\(recentRecord.totalShotsFiredByBoss) 發 🔮")
                StatItemRow(label: "對 Boss 實體總輸出", val: "\(recentRecord.bossDamageDealt) 點")
                StatItemRow(label: "物資道具攔截量", val: "\(recentRecord.itemCollectedCount) 模組")
                StatItemRow(label: "此局解鎖核心產出", val: "+ \(recentRecord.goldEarnedThisRound) 💰")
                StatItemRow(label: "戰場作戰持續時間", val: String(format: "%.1f 秒", recentRecord.timeElapsed))
            }
            .padding().background(Color.white.opacity(0.06)).cornerRadius(16).frame(width: 340)
            
            Button("回到全域選單") { withAnimation { viewStatus = "menu" } }
                .font(.headline).frame(width: 220, height: 50).background(Color.blue).foregroundColor(.white).cornerRadius(16)
        }
    }
    
    private func buttonColor(for mode: GameMode) -> Color {
        switch mode {
        case .single:   return .blue
        case .aiBattle: return .purple
        case .pvp:      return Color(red: 0.88, green: 0.18, blue: 0.38)
        }
    }
}

struct AchievementRow: View {
    let id: String
    let title: String
    let desc: String
    let icon: String
    var body: some View {
        let isUnlocked = GameStorageManager.shared.getUnlockedAchievements().contains(id)
        HStack(spacing: 16) {
            Image(systemName: icon).font(.title).foregroundColor(isUnlocked ? .yellow : .gray)
            VStack(alignment: .leading) {
                Text(title).font(.headline).foregroundColor(isUnlocked ? .white : .gray)
                Text(desc).font(.caption).foregroundColor(.gray)
            }
            Spacer()
            Text(isUnlocked ? "已達成" : "未解鎖").font(.footnote).foregroundColor(isUnlocked ? .green : .gray)
        }.padding(.vertical, 6).listRowBackground(Color.white.opacity(0.03))
    }
}

struct StatItemRow: View {
    let label: String
    let val: String
    var body: some View {
        HStack {
            Text(label).foregroundColor(.gray).font(.subheadline)
            Spacer()
            Text(val).foregroundColor(.white).bold().font(.subheadline)
        }
    }
}

