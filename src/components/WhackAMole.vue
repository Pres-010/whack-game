<template>
  <div id="wam-root" :class="{ 'day-mode': isDayMode }">

    <!-- Animated stars & fireflies -->
    <div class="stars-bg">
      <div
        v-for="s in stars"
        :key="'star-' + s.id"
        class="star"
        :style="s.style"
      ></div>
      <div
        v-for="f in fireflies"
        :key="'ff-' + f.id"
        class="firefly"
        :style="f.style"
      ></div>
    </div>

    <!-- Crescent moon -->
    <div class="moon"></div>

    <!-- Day/Night Toggle Button -->
    <div class="theme-toggle" @click="toggleTheme">
      <div class="toggle-icon">{{ isDayMode ? '🌙' : '☀️' }}</div>
    </div>

    <!-- Title -->
    <div class="title-row">
      <h1>🔨 Whack-a-Mole!</h1>
      <p class="subtitle">{{ isDayMode ? '☀️ Day Edition — tap the mole before it vanishes!' : '🌙 Night Edition — tap the mole before it vanishes!' }}</p>
    </div>

    <!-- Score panel with Coins -->
    <div class="score-panel">
      <div class="score-card">
        <div class="score-label">Score</div>
        <div class="score-value" :class="{ 'score-bounce': scoreBounce }">{{ score }}</div>
      </div>
      <div class="score-card">
        <div class="score-label">Best</div>
        <div class="score-value">{{ best }}</div>
      </div>
      <div class="score-card">
        <div class="score-label">Time</div>
        <div class="score-value" :class="{ 'timer-warn': timeLeft <= 5 }">{{ timeLeft }}</div>
      </div>
      <div class="score-card">
        <div class="score-label">Combo</div>
        <div class="score-value" :class="{ 'combo-active': combo >= 3 }">{{ combo }}x</div>
      </div>
      <div class="score-card coin-card" @click="showShop = !showShop">
        <div class="score-label">🪙 Coins</div>
        <div class="score-value coin-value">{{ coins }}</div>
      </div>
    </div>

    <!-- Shop Panel -->
    <transition name="shop-slide">
      <div class="shop-panel" v-if="showShop">
        <div class="shop-header">
          <h3>🛒 Power-Up Shop 🛒</h3>
          <button class="close-shop" @click="showShop = false">✖</button>
        </div>
        <div class="shop-items">
          <div v-for="item in shopItems" :key="item.id" class="shop-item">
            <div class="item-icon">{{ item.icon }}</div>
            <div class="item-info">
              <div class="item-name">{{ item.name }}</div>
              <div class="item-desc">{{ item.description }}</div>
            </div>
            <div class="item-cost">
              <span class="cost-value">{{ item.cost }}</span>
              <span class="cost-coin">🪙</span>
            </div>
            <button 
              class="buy-btn" 
              :disabled="coins < item.cost || (item.owned && item.purchasable === false)"
              @click="buyPowerup(item)"
            >
              {{ item.owned && item.type === 'passive' ? 'Use' : (item.owned ? 'Owned' : 'Buy') }}
            </button>
            <div v-if="item.active" class="active-badge">ACTIVE</div>
          </div>
        </div>
        
        <!-- Active Powerups Display -->
        <div class="active-powerups" v-if="activePowerups.length > 0">
          <div class="active-title">✨ Active Powerups:</div>
          <div class="active-list">
            <div v-for="powerup in activePowerups" :key="powerup.id" class="active-powerup">
              {{ powerup.icon }} {{ powerup.name }}: {{ powerup.timeLeft }}s
            </div>
          </div>
        </div>
      </div>
    </transition>

    <!-- Difficulty selector -->
    <div class="difficulty-row" v-if="!running && !showShop">
      <button 
        v-for="diff in difficulties" 
        :key="diff.name"
        class="difficulty-btn"
        :class="{ 'active-diff': currentDifficulty === diff.name }"
        @click="setDifficulty(diff.name)"
      >
        {{ diff.emoji }} {{ diff.name }}
      </button>
    </div>

    <!-- Buttons -->
    <div class="btn-row">
      <button
        class="btn btn-start"
        :disabled="running"
        :style="{ opacity: running ? '0.5' : '1' }"
        @click="startGame"
      >
        ▶ Start Game
      </button>
      <button class="btn btn-reset" @click="resetGame">↺ Reset</button>
    </div>

    <!-- Status message -->
    <div class="status-msg">{{ statusMsg }}</div>

    <!-- Statistics Panel -->
    <div class="stats-panel" v-if="statsVisible">
      <div class="stats-grid">
        <div class="stat-item">
          <div class="stat-label">🎯 Accuracy</div>
          <div class="stat-value">{{ stats.accuracy }}%</div>
        </div>
        <div class="stat-item">
          <div class="stat-label">⚡ Fastest</div>
          <div class="stat-value">{{ stats.fastestReaction }}ms</div>
        </div>
        <div class="stat-item">
          <div class="stat-label">🐭 Total Hits</div>
          <div class="stat-value">{{ stats.totalHits }}</div>
        </div>
        <div class="stat-item">
          <div class="stat-label">💫 Golden Hits</div>
          <div class="stat-value">{{ stats.goldenHits }}</div>
        </div>
        <div class="stat-item">
          <div class="stat-label">🪙 Coins Earned</div>
          <div class="stat-value">{{ stats.totalCoinsEarned }}</div>
        </div>
      </div>
    </div>
    <button class="toggle-stats" @click="statsVisible = !statsVisible">
      {{ statsVisible ? '▼ Hide Stats' : '▲ Show Stats' }}
    </button>

    <!-- Meadow + holes -->
    <div class="meadow">
      <div class="meadow-top">🌿 🍄 🌿 🍃 🌿 🍄 🌿🌿 🍄 🌿 🍃 🌿 🍄 🌿🌿 🍄 🌿 🍃 🌿 🍄 🌿</div>
      <div class="holes-grid">
        <div
          v-for="i in 9"
          :key="`hole-${i-1}`"
          class="hole-wrap"
        >
          <div
            class="hole"
            :class="{ 'has-mole': activeMole === i-1, 'hit-flash': flashHole === i-1 }"
            @click="whackMole(i-1)"
            @touchstart.prevent="whackMole(i-1)"
          >
            <span
              v-if="activeMole === i-1"
              class="mole"
              :class="{ 
                'mole-exit': exitingMole === i-1, 
                'golden-mole': isGoldenMole,
                'slow-mole': activePowerups.some(p => p.id === 'slow_mole')
              }"
            >
              {{ currentMoleEmoji }}
              <span v-if="activePowerups.some(p => p.id === 'double_points')" class="double-indicator">x2</span>
            </span>
          </div>
          <div class="grass-strip"></div>

          <!-- Floating score popup -->
          <transition name="pts-fade">
            <div
              v-if="popups[i-1]"
              :class="popups[i-1].hit ? 'pts-popup' : 'miss-popup'"
            >
              {{ popups[i-1].text }}
            </div>
          </transition>

          <!-- Particle canvas for each hole -->
          <canvas v-if="particles[i-1]" :ref="'particleCanvas' + (i-1)" class="particle-canvas"></canvas>
        </div>
      </div>
    </div>

    <!-- High score note -->
    <div class="high-score-note">{{ highScoreNote }}</div>

  </div>
</template>

<script>
const MOLE_EMOJIS = ['🐭', '🐹', '🦝']
const GOLDEN_MOJI = '🪙'
const BASE_GAME_DURATION = 30

// Difficulty settings
const DIFFICULTY_SETTINGS = {
  Easy: { interval: 1200, visible: 1200, points: 5, emoji: '🍃' },
  Normal: { interval: 900, visible: 900, points: 10, emoji: '🌙' },
  Hard: { interval: 600, visible: 600, points: 15, emoji: '⚡' }
}

import hitSoundFile from '../assets/sounds/hit.wav'
import missSoundFile from '../assets/sounds/miss.wav'
import startSoundFile from '../assets/sounds/start.wav'
import goldenSoundFile from '../assets/sounds/golden.mp3'
import coinSoundFile from '../assets/sounds/coin.mp3'

export default {
  name: 'WhackAMole',

  data() {
    return {
      // Game state
      score:          0,
      best:           0,
      timeLeft:       BASE_GAME_DURATION,
      activeMole:     null,
      exitingMole:    null,
      flashHole:      null,
      running:        false,
      currentMoleEmoji: '🐹',
      isGoldenMole:   false,
      
      // Coin system
      coins:          0,
      
      // Combo system
      combo:          0,
      comboMultiplier: 1,
      
      // Difficulty
      currentDifficulty: 'Normal',
      difficulties: [
        { name: 'Easy', emoji: '🍃' },
        { name: 'Normal', emoji: '🌙' },
        { name: 'Hard', emoji: '⚡' }
      ],
      
      // Shop
      showShop: false,
      shopItems: [
        {
          id: 'time_freeze',
          name: '⏰ Time Freeze',
          icon: '⏰',
          description: 'Stop timer for 10 seconds',
          cost: 150,
          type: 'active',
          duration: 10000,
          owned: false,
          purchasable: true,
          active: false
        },
        {
          id: 'slow_mole',
          name: '🐢 Slow Motion',
          icon: '🐢',
          description: 'Moles move 50% slower for 15s',
          cost: 100,
          type: 'active',
          duration: 15000,
          owned: false,
          purchasable: true,
          active: false
        },
        {
          id: 'double_points',
          name: '✨ Double Points',
          icon: '✨',
          description: '2x points for 20 seconds',
          cost: 120,
          type: 'active',
          duration: 20000,
          owned: false,
          purchasable: true,
          active: false
        },
        {
          id: 'extra_time',
          name: '➕ Extra Time',
          icon: '➕',
          description: '+10 seconds (instant)',
          cost: 80,
          type: 'instant',
          owned: false,
          purchasable: true,
          active: false,
          pendingUse: false
        },
        {
          id: 'mole_radar',
          name: '📡 Mole Radar',
          icon: '📡',
          description: 'See where mole appears (5 uses)',
          cost: 200,
          type: 'passive',
          uses: 5,
          owned: false,
          purchasable: true,
          active: false
        }
      ],
      activePowerups: [],
      radarActive: false,
      radarUses: 0,
      
      // Theme (Day/Night)
      isDayMode: false,
      
      // Store original interval for restoration
      originalMoleInterval: null,
      isTimeFrozen: false,
      
      // Statistics
      stats: {
        totalHits: 0,
        totalMisses: 0,
        accuracy: 0,
        fastestReaction: 999,
        goldenHits: 0,
        totalCoinsEarned: 0,
        hitTimes: []
      },
      statsVisible: false,
      
      // Reaction time tracking
      moleSpawnTime: null,
      
      // UI state
      statusMsg:      '🎮 Press Start to play!',
      highScoreNote:  '',
      scoreBounce:    false,
      popups:         Array(9).fill(null),
      particles:      Array(9).fill(false),

      // Timers
      gameInterval:   null,
      moleLoopTimer:  null,
      moleTimeout:    null,
      powerupTimers:  [],

      // Decorative particles
      stars:      [],
      fireflies:  [],
      
      // Sounds
      hitSound: new Audio(hitSoundFile),
      missSound: new Audio(missSoundFile),
      startSound: new Audio(startSoundFile),
      goldenSound: new Audio(goldenSoundFile),
      coinSound: new Audio(coinSoundFile),
    }
  },

  watch: {
    coins(newVal) {
      localStorage.setItem("playerCoins", newVal)
    }
  },
  
  mounted() {
    this.generateParticles()
    
    // Load best score
    const savedBest = localStorage.getItem("bestScore")
    if (savedBest) {
      this.best = parseInt(savedBest)
    }
    
    // Load saved coins
    const savedCoins = localStorage.getItem("playerCoins")
    if (savedCoins !== null) {
      this.coins = parseInt(savedCoins)
    } else {
      this.coins = 0
    }
    
    // Load saved theme
    this.applySavedTheme()
    
    // Load owned powerups
    const savedPowerups = localStorage.getItem("ownedPowerups")
    if (savedPowerups) {
      const owned = JSON.parse(savedPowerups)
      this.shopItems.forEach(item => {
        if (owned.includes(item.id)) {
          item.owned = true
          if (item.id === 'mole_radar' && item.uses === undefined) {
            item.uses = 5
          }
        }
      })
    }
    
    // Load saved stats
    const savedStats = localStorage.getItem("gameStats")
    if (savedStats) {
      const parsed = JSON.parse(savedStats)
      this.stats = { ...this.stats, ...parsed }
    }
    
    // Preload sounds
    [this.hitSound, this.missSound, this.startSound, this.goldenSound, this.coinSound].forEach(sound => {
      sound.load()
    })
  },

  beforeUnmount() {
    clearInterval(this.gameInterval)
    clearInterval(this.moleLoopTimer)
    clearTimeout(this.moleTimeout)
    this.powerupTimers.forEach(timer => clearTimeout(timer))
  },

  methods: {
    setDifficulty(diff) {
      this.currentDifficulty = diff
      this.statusMsg = `${DIFFICULTY_SETTINGS[diff].emoji} ${diff} mode selected!`
    },
    
    getDifficultySettings() {
      return DIFFICULTY_SETTINGS[this.currentDifficulty]
    },
    
    toggleTheme() {
      this.isDayMode = !this.isDayMode
      
      if (this.isDayMode) {
         document.documentElement.style.setProperty('--bg-gradient-start', '#87CEEB')  // Change this
    document.documentElement.style.setProperty('--bg-gradient-mid', '#98D8E8')    // Change this
    document.documentElement.style.setProperty('--bg-gradient-end', '#B0E0E6')    // Change this
    document.documentElement.style.setProperty('--hole-shadow', '0 8px 24px rgba(0,0,0,0.3)')
    document.documentElement.style.setProperty('--text-primary', '#2D3748')
    document.documentElement.style.setProperty('--text-secondary', '#4A5568')
        this.statusMsg = '☀️ Day Mode activated! ☀️'
      } else {
         document.documentElement.style.setProperty('--bg-gradient-start', '#0d0d2b')
    document.documentElement.style.setProperty('--bg-gradient-mid', '#1a0a2e')
    document.documentElement.style.setProperty('--bg-gradient-end', '#0f1a2e')
    document.documentElement.style.setProperty('--hole-shadow', '0 8px 24px rgba(0,0,0,0.7)')
    document.documentElement.style.setProperty('--text-primary', '#d8b4fe')
    document.documentElement.style.setProperty('--text-secondary', '#c4b5fd')
        this.statusMsg = '🌙 Night Mode activated! 🌙'
      }
      
      // Save preference
      localStorage.setItem('dayMode', this.isDayMode)
    },
    
    applySavedTheme() {
      const savedTheme = localStorage.getItem('dayMode')
      if (savedTheme !== null) {
        this.isDayMode = savedTheme === 'true'
      }
    },
    
    buyPowerup(item) {
      if (this.coins < item.cost) {
        this.statusMsg = `❌ Not enough coins! Need ${item.cost}🪙`
        return
      }
      
      if (item.type === 'active') {
        if (item.active) {
          this.statusMsg = `⚠️ ${item.name} is already active!`
          return
        }
        
        this.coins -= item.cost
        this.coinSound.play()
        this.stats.totalCoinsEarned -= item.cost
        this.saveCoins()
        
        // Activate powerup
        this.activatePowerup(item)
        this.statusMsg = `🎉 ${item.name} activated! ${item.description}`
      } 
      else if (item.type === 'passive') {
        if (!item.owned) {
          this.coins -= item.cost
          this.coinSound.play()
          item.owned = true
          item.uses = 5
          this.saveOwnedPowerups()
          this.saveCoins()
          this.statusMsg = `🎉 Purchased ${item.name}! You have ${item.uses} uses. Click again to use!`
        } else if (item.uses > 0) {
          // Use the radar
          this.useRadar()
          item.uses--
          this.saveOwnedPowerups()
          this.statusMsg = `📡 Radar used! ${item.uses} uses remaining.`
        } else {
          // Offer to recharge
          if (confirm(`No uses left! Buy ${item.name} again for ${item.cost}🪙?`)) {
            if (this.coins >= item.cost) {
              this.coins -= item.cost
              item.uses = 5
              this.saveCoins()
              this.saveOwnedPowerups()
              this.statusMsg = `🔄 ${item.name} recharged! ${item.uses} uses remaining.`
            } else {
              this.statusMsg = `❌ Not enough coins to recharge! Need ${item.cost}🪙`
            }
          }
        }
      }
      else if (item.type === 'instant') {
        // Handle instant powerups (Extra Time)
        this.coins -= item.cost
        this.coinSound.play()
        this.stats.totalCoinsEarned -= item.cost
        this.saveCoins()
        
        // Apply instant effect immediately
        if (item.id === 'extra_time') {
          if (this.running) {
            // Game is running - add time NOW
            this.timeLeft += 10
            this.statusMsg = `⏰ +10 seconds added! Time left: ${this.timeLeft}s`
            this.showPopupMessage(`+10s ⏰`)
            
            // Show visual feedback
            this.triggerScoreBounce()
            
            // Create a popup effect
            const randomHole = Math.floor(Math.random() * 9)
            this.showPopup(randomHole, true, '+10s ⏰')
          } else {
            // Game not running - save as a one-time use item
            if (!item.pendingUse) {
              item.pendingUse = true
              this.statusMsg = `🎁 Extra Time purchased! It will activate when you start a game!`
            } else {
              // Use the pending extra time
              item.pendingUse = false
              this.timeLeft += 10
              this.statusMsg = `⏰ +10 seconds added! Time left: ${this.timeLeft}s`
            }
          }
        }
      }
    },
    
    activatePowerup(item) {
      const powerup = {
        id: item.id,
        name: item.name,
        icon: item.icon,
        duration: item.duration,
        timeLeft: item.duration / 1000,
        active: true
      }
      
      this.activePowerups.push(powerup)
      item.active = true
      
      // Handle different powerup effects
      if (item.id === 'time_freeze') {
        this.activateTimeFreeze(item)
      }
      else if (item.id === 'slow_mole') {
        this.activateSlowMole(item)
      }
      else if (item.id === 'double_points') {
        this.statusMsg = '✨ DOUBLE POINTS! Score x2 for 20 seconds! ✨'
      }
      
      // Set timer to deactivate
      const deactivateTimer = setTimeout(() => {
        this.deactivatePowerup(item.id)
      }, item.duration)
      this.powerupTimers.push(deactivateTimer)
      
      // Update timer display
      const interval = setInterval(() => {
        const p = this.activePowerups.find(p => p.id === item.id)
        if (p) {
          p.timeLeft = Math.max(0, p.timeLeft - 1)
          if (p.timeLeft <= 0) clearInterval(interval)
        } else {
          clearInterval(interval)
        }
      }, 1000)
    },
    
    activateTimeFreeze(item) {
      if (this.isTimeFrozen) return
      
      this.isTimeFrozen = true
      
      // Clear any existing mole timeout to prevent mole from disappearing during freeze
      if (this.moleTimeout) {
        clearTimeout(this.moleTimeout)
        this.moleTimeout = null
      }
      
      this.statusMsg = '⏰ TIME FREEZE! Timer stopped for 10 seconds!'
      
      // Set timer to resume after duration
      const resumeTimer = setTimeout(() => {
        this.resumeAfterTimeFreeze()
      }, item.duration)
      this.powerupTimers.push(resumeTimer)
    },
    
    resumeAfterTimeFreeze() {
      if (!this.isTimeFrozen) return
      
      this.isTimeFrozen = false
      
      if (this.running) {
        this.statusMsg = '⏰ Time freeze ended! Game resumes!'
        
        // Make sure the current mole doesn't stay forever
        if (this.activeMole !== null) {
          const settings = this.getDifficultySettings()
          let visibleDuration = settings.visible
          if (this.activePowerups.some(p => p.id === 'slow_mole')) {
            visibleDuration = visibleDuration * 1.5
          }
          
          // Clear existing timeout first
          if (this.moleTimeout) {
            clearTimeout(this.moleTimeout)
          }
          
          // Set new timeout to hide mole
          this.moleTimeout = setTimeout(() => {
            this.hideMole(true)
          }, visibleDuration)
        }
      }
    },
    
    activateSlowMole(item) {
      // Restart mole spawn loop with slower interval
      if (this.moleLoopTimer && this.running && !this.isTimeFrozen) {
        clearInterval(this.moleLoopTimer)
        const settings = this.getDifficultySettings()
        const slowerInterval = settings.interval * 1.5
        this.moleLoopTimer = setInterval(() => {
          if (this.running && !this.isTimeFrozen) this.showMole()
        }, slowerInterval)
      }
      this.statusMsg = '🐢 Slow Motion active! Moles move slower!'
    },
    
    deactivatePowerup(id) {
      const index = this.activePowerups.findIndex(p => p.id === id)
      if (index !== -1) {
        this.activePowerups.splice(index, 1)
        const shopItem = this.shopItems.find(item => item.id === id)
        if (shopItem) shopItem.active = false
        
        // Special handling for different powerups
        if (id === 'slow_mole' && this.running && !this.isTimeFrozen) {
          // Restart mole spawn with normal speed
          if (this.moleLoopTimer) {
            clearInterval(this.moleLoopTimer)
            const settings = this.getDifficultySettings()
            this.moleLoopTimer = setInterval(() => {
              if (this.running && !this.isTimeFrozen) this.showMole()
            }, settings.interval)
          }
          this.statusMsg = `🐢 Slow Motion ended! Moles back to normal speed.`
        }
        else if (id === 'double_points') {
          this.statusMsg = `✨ Double Points ended!`
        }
        else if (id === 'time_freeze') {
          // Already handled by resumeAfterTimeFreeze
          this.statusMsg = `⏰ Time Freeze ended!`
        }
        else {
          this.statusMsg = `⏹️ ${shopItem?.name || 'Powerup'} has ended!`
        }
      }
    },
    
    useRadar() {
      if (this.activeMole !== null && this.running) {
        const holeIndex = this.activeMole
        this.triggerFlash(holeIndex)
        this.showPopup(holeIndex, true, '📍 HERE!')
        this.statusMsg = `📍 Mole detected at hole ${holeIndex + 1}!`
      } else {
        this.statusMsg = `🔍 No mole detected at the moment...`
      }
    },
    
    addCoins(amount) {
      this.coins += amount
      this.stats.totalCoinsEarned += amount
      this.saveCoins()
      this.triggerCoinAnimation()
    },
    
    saveCoins() {
      localStorage.setItem("playerCoins", this.coins)
      localStorage.setItem("gameStats", JSON.stringify({
        totalHits: this.stats.totalHits,
        totalMisses: this.stats.totalMisses,
        accuracy: this.stats.accuracy,
        fastestReaction: this.stats.fastestReaction,
        goldenHits: this.stats.goldenHits,
        totalCoinsEarned: this.stats.totalCoinsEarned
      }))
    },
    
    saveOwnedPowerups() {
      const owned = this.shopItems.filter(item => item.owned).map(item => item.id)
      localStorage.setItem("ownedPowerups", JSON.stringify(owned))
    },
    
    triggerCoinAnimation() {
      const coinMsg = { hit: true, text: '+🪙' }
      const randomHole = Math.floor(Math.random() * 9)
      const updated = [...this.popups]
      updated[randomHole] = coinMsg
      this.popups = updated
      setTimeout(() => {
        const cleared = [...this.popups]
        cleared[randomHole] = null
        this.popups = cleared
      }, 900)
    },
    
    showPopupMessage(text) {
      this.statusMsg = text
    },
    
    showMole() {
      // Don't spawn new moles during time freeze
      if (this.isTimeFrozen) return
      
      this.hideMole()
      
      // Random hole index between 0 and 8
      this.activeMole = Math.floor(Math.random() * 9)
      
      // Adjust golden mole chance based on theme
      let goldenChance = 0.15
      if (!this.isDayMode) {
        goldenChance = 0.2 // Slightly higher chance at night
      }
      this.isGoldenMole = Math.random() < goldenChance
      
      if (this.isGoldenMole) {
        this.currentMoleEmoji = GOLDEN_MOJI
      } else {
        this.currentMoleEmoji = MOLE_EMOJIS[Math.floor(Math.random() * MOLE_EMOJIS.length)]
      }
      
      // Track spawn time for reaction speed
      this.moleSpawnTime = Date.now()
      
      // Adjust visibility time if slow motion active
      let visibleDuration = this.getDifficultySettings().visible
      if (this.activePowerups.some(p => p.id === 'slow_mole')) {
        visibleDuration = visibleDuration * 1.5
      }
      
      // Hide the mole after visibility duration
      this.moleTimeout = setTimeout(() => {
        this.hideMole(true)
      }, visibleDuration)
    },
    
    hideMole(isMiss = false) {
      if (this.activeMole !== null) {
        if (isMiss && this.running && !this.isTimeFrozen) {
          this.combo = 0
          this.comboMultiplier = 1
          this.stats.totalMisses++
          this.updateAccuracy()
        }
        
        this.exitingMole = this.activeMole
        setTimeout(() => {
          this.exitingMole = null
        }, 220)
        this.activeMole = null
        this.moleSpawnTime = null
      }
      if (this.moleTimeout) {
        clearTimeout(this.moleTimeout)
        this.moleTimeout = null
      }
    },

    whackMole(idx) {
      if (!this.running) return
      if (this.isTimeFrozen) {
        this.statusMsg = '⏰ Time is frozen! Cannot whack!'
        return
      }

      if (this.activeMole === idx) {
        // Calculate reaction time
        let reactionTime = null
        if (this.moleSpawnTime) {
          reactionTime = Date.now() - this.moleSpawnTime
          this.stats.hitTimes.push(reactionTime)
          if (reactionTime < this.stats.fastestReaction) {
            this.stats.fastestReaction = reactionTime
          }
        }
        
        // Calculate points with combo and powerups
        let settings = this.getDifficultySettings()
        let basePoints = settings.points
        
        // Apply double points powerup
        const hasDouble = this.activePowerups.some(p => p.id === 'double_points')
        if (hasDouble) {
          basePoints *= 2
        }
        
        if (this.isGoldenMole) {
          basePoints = 100
          this.goldenSound.currentTime = 0
          this.goldenSound.play()
          this.stats.goldenHits++
          this.statusMsg = '🪙 GOLDEN MOLE! +100 points! 🪙'
          this.addCoins(25)
        } else {
          this.hitSound.currentTime = 0
          this.hitSound.play()
          this.addCoins(5)
        }
        
        // Combo multiplier
        if (this.combo >= 10) this.comboMultiplier = 3
        else if (this.combo >= 5) this.comboMultiplier = 2
        else this.comboMultiplier = 1
        
        const pointsEarned = basePoints * this.comboMultiplier
        this.score += pointsEarned
        this.combo++
        
        // Update stats
        this.stats.totalHits++
        this.updateAccuracy()
        
        // Clear the mole timeout
        clearTimeout(this.moleTimeout)
        this.moleTimeout = null
        
        this.triggerScoreBounce()
        this.triggerFlash(idx)
        this.triggerParticles(idx)
        
        let popupText = this.isGoldenMole ? `+${pointsEarned} GOLD! +25🪙` : `+${pointsEarned} +5🪙`
        if (this.comboMultiplier > 1) popupText += ` x${this.comboMultiplier}`
        if (this.activePowerups.some(p => p.id === 'double_points')) popupText += ' ✨2x✨'
        this.showPopup(idx, true, popupText)
        
        const hitMessages = [
          '🎯 Bullseye!', '💥 Got it!', '⚡ Nailed it!', 
          '🎉 Nice!', '🔥 Hot shot!', '✨ Dazzling!'
        ]
        if (this.combo >= 10) {
          this.statusMsg = '🌟🌟🌟 GODLIKE COMBO! 🌟🌟🌟'
        } else if (this.combo >= 5) {
          this.statusMsg = `⚡ ${this.combo}x COMBO! ⚡`
        } else {
          this.statusMsg = hitMessages[Math.floor(Math.random() * hitMessages.length)]
        }
        
        // Save stats
        this.saveCoins()
        
        // Remove the mole immediately after hit
        this.activeMole = null
      } else {
        // Miss
        this.missSound.currentTime = 0
        this.missSound.play()
        
        this.combo = 0
        this.comboMultiplier = 1
        this.stats.totalMisses++
        this.updateAccuracy()
        
        this.showPopup(idx, false, 'miss 💨')
        this.statusMsg = '😢 Missed it!'
      }
    },

    updateAccuracy() {
      const total = this.stats.totalHits + this.stats.totalMisses
      this.stats.accuracy = total > 0 ? Math.round((this.stats.totalHits / total) * 100) : 0
    },

    startGame() {
      if (this.running) return
      
      // Check for pending extra time powerup
      const extraTimeItem = this.shopItems.find(item => item.id === 'extra_time')
      if (extraTimeItem && extraTimeItem.pendingUse) {
        extraTimeItem.pendingUse = false
        setTimeout(() => {
          if (this.running) {
            this.timeLeft += 10
            this.statusMsg = `⏰ +10 seconds bonus activated! Time left: ${this.timeLeft}s`
            this.showPopup(Math.floor(Math.random() * 9), true, '+10s BONUS!')
          }
        }, 100)
      }
      
      // Clear all active powerups before starting new game
      this.activePowerups = []
      this.isTimeFrozen = false
      this.shopItems.forEach(item => {
        if (item.type === 'active') {
          item.active = false
        }
      })
      
      // Make sure any existing game is fully cleaned up
      if (this.gameInterval || this.moleLoopTimer) {
        this.resetGame()
      }
      
      this.startSound.play()
      this.running = true
      this.score = 0
      this.combo = 0
      this.comboMultiplier = 1
      this.timeLeft = BASE_GAME_DURATION
      this.statusMsg = `🌙 ${this.currentDifficulty} mode activated! Good luck!`
      this.highScoreNote = ''
      this.showShop = false
      
      // Reset reaction tracking
      this.stats.hitTimes = []
      this.stats.fastestReaction = 999

      // Countdown timer - checks isTimeFrozen
      this.gameInterval = setInterval(() => {
        // Only count down if game is running AND time is NOT frozen
        if (this.running && !this.isTimeFrozen && this.timeLeft > 0) {
          this.timeLeft--
          if (this.timeLeft <= 0) {
            this.endGame()
          }
        }
      }, 1000)

      const settings = this.getDifficultySettings()
      let interval = settings.interval
      
      // Adjust spawn rate if slow motion active
      if (this.activePowerups.some(p => p.id === 'slow_mole')) {
        interval = interval * 1.5
      }
      
      // Mole spawn loop - checks isTimeFrozen
      this.moleLoopTimer = setInterval(() => {
        // Only spawn moles if game is running AND time is NOT frozen
        if (this.running && !this.isTimeFrozen) {
          this.showMole()
        }
      }, interval)

      this.showMole()
    },

    endGame() {
      if (!this.running) return
      
      this.running = false
      this.isTimeFrozen = false
      
      // Clear intervals
      if (this.gameInterval) {
        clearInterval(this.gameInterval)
        this.gameInterval = null
      }
      
      if (this.moleLoopTimer) {
        clearInterval(this.moleLoopTimer)
        this.moleLoopTimer = null
      }
      
      this.hideMole()

      // Calculate average reaction time
      const avgReaction = this.stats.hitTimes.length > 0 
        ? Math.round(this.stats.hitTimes.reduce((a,b) => a + b, 0) / this.stats.hitTimes.length)
        : 0
      
      // Bonus coins for high score
      let bonusCoins = 0
      if (this.score > this.best) {
        this.best = this.score
        bonusCoins = 50
        this.addCoins(bonusCoins)
        localStorage.setItem("bestScore", this.best)
        this.highScoreNote = `🏆 NEW HIGH SCORE! ${this.score} pts | +${bonusCoins}🪙 BONUS! 🏆`
      } else {
        this.highScoreNote = `🎮 Final: ${this.score} pts | Best: ${this.best} pts | Best Combo: ${this.combo}x | Avg RT: ${avgReaction}ms`
      }
      
      this.saveCoins()
    },

    resetGame() {
      // Stop the game if it's running
      if (this.running) {
        this.running = false
      }
      
      this.isTimeFrozen = false
      
      // Clear ALL intervals and timeouts
      if (this.gameInterval) {
        clearInterval(this.gameInterval)
        this.gameInterval = null
      }
      
      if (this.moleLoopTimer) {
        clearInterval(this.moleLoopTimer)
        this.moleLoopTimer = null
      }
      
      if (this.moleTimeout) {
        clearTimeout(this.moleTimeout)
        this.moleTimeout = null
      }
      
      // Clear powerup timers
      this.powerupTimers.forEach(timer => clearTimeout(timer))
      this.powerupTimers = []
      
      // Deactivate all active powerups
      this.activePowerups = []
      this.shopItems.forEach(item => {
        if (item.type === 'active') {
          item.active = false
        }
      })
      
      // Hide any active mole
      this.hideMole()
      
      // Reset all game state
      this.score = 0
      this.combo = 0
      this.comboMultiplier = 1
      this.timeLeft = BASE_GAME_DURATION
      this.activeMole = null
      this.exitingMole = null
      this.flashHole = null
      
      // Reset UI
      this.statusMsg = '🎮 Press Start to play!'
      this.highScoreNote = ''
      
      // Clear any pending popups
      this.popups = Array(9).fill(null)
      this.particles = Array(9).fill(false)
      
      // Close shop if open
      this.showShop = false
    },
    
    triggerScoreBounce() {
      this.scoreBounce = false
      this.$nextTick(() => { this.scoreBounce = true })
      setTimeout(() => { this.scoreBounce = false }, 400)
    },

    triggerFlash(idx) {
      this.flashHole = idx
      setTimeout(() => { this.flashHole = null }, 450)
    },
    
    triggerParticles(idx) {
      this.particles[idx] = true
      this.$nextTick(() => {
        const canvas = this.$refs[`particleCanvas${idx}`]
        if (canvas && canvas[0]) {
          this.drawParticles(canvas[0], idx)
        }
      })
      setTimeout(() => {
        this.particles[idx] = false
      }, 500)
    },
    
    drawParticles(canvas, idx) {
      const ctx = canvas.getContext('2d')
      canvas.width = 100
      canvas.height = 100
      
      const particles = []
      for (let i = 0; i < 20; i++) {
        particles.push({
          x: 50,
          y: 50,
          vx: (Math.random() - 0.5) * 8,
          vy: (Math.random() - 0.5) * 8 - 2,
          life: 1,
          color: this.isGoldenMole ? `hsl(50, 100%, ${50 + Math.random() * 30}%)` : `hsl(${Math.random() * 60 + 280}, 100%, 60%)`
        })
      }
      
      let animationId
      const animate = () => {
        ctx.clearRect(0, 0, canvas.width, canvas.height)
        let allDead = true
        
        for (let i = 0; i < particles.length; i++) {
          const p = particles[i]
          if (p.life > 0) {
            allDead = false
            p.x += p.vx
            p.y += p.vy
            p.vy += 0.3
            p.life -= 0.03
            ctx.globalAlpha = p.life
            ctx.fillStyle = p.color
            ctx.fillRect(p.x, p.y, 4, 4)
          }
        }
        
        if (!allDead) {
          animationId = requestAnimationFrame(animate)
        } else {
          cancelAnimationFrame(animationId)
        }
      }
      
      animate()
    },

    showPopup(idx, hit, customText = null) {
      const updated = [...this.popups]
      let text = customText
      if (!customText) {
        text = hit ? `+${this.getDifficultySettings().points}` : 'miss 💨'
      }
      updated[idx] = { hit, text }
      this.popups = updated
      setTimeout(() => {
        const cleared = [...this.popups]
        cleared[idx] = null
        this.popups = cleared
      }, 900)
    },

    generateParticles() {
      this.stars = Array.from({ length: 80 }, (_, i) => {
        const size = Math.random() < 0.15 ? 3 : Math.random() < 0.4 ? 2 : 1
        return {
          id: i,
          style: {
            width:  size + 'px',
            height: size + 'px',
            left:   Math.random() * 100 + '%',
            top:    Math.random() * 55  + '%',
            '--dur':   (2 + Math.random() * 3) + 's',
            '--delay': (Math.random() * 4) + 's',
          }
        }
      })

      this.fireflies = Array.from({ length: 10 }, (_, i) => ({
        id: i,
        style: {
          left:      (20 + Math.random() * 60) + '%',
          top:       (50 + Math.random() * 40) + '%',
          '--fdur':  (3  + Math.random() * 4)  + 's',
          '--fdelay':(Math.random() * 5)        + 's',
          '--fx':    (Math.random() * 120 - 60) + 'px',
          '--fy':    (Math.random() * 80  - 40) + 'px',
        }
      }))
    },
  }
}
</script>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Fredoka+One&family=Nunito:wght@400;700;800&display=swap');

/* ── Theme Variables ─────────────────────────────────────────────── */
:root {
  --bg-gradient-start: #0d0d2b;
  --bg-gradient-mid: #1a0a2e;
  --bg-gradient-end: #0f1a2e;
  --hole-shadow: 0 8px 24px rgba(0, 0, 0, 0.7);
  --text-primary: #d8b4fe;
  --text-secondary: #c4b5fd;
}

/* ── Root layout ─────────────────────────────────────────────────── */
#wam-root {
  font-family: 'Nunito', sans-serif;
  min-height: 100vh;
  background: linear-gradient(160deg, var(--bg-gradient-start) 0%, var(--bg-gradient-mid) 40%, var(--bg-gradient-end) 100%);
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 24px 16px 40px;
  position: relative;
  overflow: hidden;
  transition: background 0.5s ease;
}

#wam-root::before {
  content: '';
  position: absolute;
  inset: 0;
  background:
    radial-gradient(circle at 15% 25%, rgba(120,60,220,0.25)  0%, transparent 45%),
    radial-gradient(circle at 85% 15%, rgba(0,180,255,0.15)   0%, transparent 40%),
    radial-gradient(circle at 50% 80%, rgba(0,220,150,0.12)   0%, transparent 50%);
  pointer-events: none;
}

/* ── Stars & fireflies ───────────────────────────────────────────── */
.stars-bg {
  position: absolute;
  inset: 0;
  pointer-events: none;
  overflow: hidden;
}

.star {
  position: absolute;
  background: #fff;
  border-radius: 50%;
  animation: twinkle var(--dur) ease-in-out infinite var(--delay);
}

@keyframes twinkle {
  0%, 100% { opacity: 0.15; transform: scale(1);   }
  50%       { opacity: 1;    transform: scale(1.6); }
}

.firefly {
  position: absolute;
  width: 4px;
  height: 4px;
  background: #bbf7d0;
  border-radius: 50%;
  box-shadow: 0 0 6px 2px rgba(187, 247, 208, 0.8);
  pointer-events: none;
  animation: firefly var(--fdur) ease-in-out infinite var(--fdelay);
}

@keyframes firefly {
  0%        { opacity: 0;   transform: translate(0, 0);               }
  25%       { opacity: 1;                                              }
  50%       { opacity: 0.6; transform: translate(var(--fx), var(--fy)); }
  75%       { opacity: 1;                                              }
  100%      { opacity: 0;   transform: translate(0, 0);               }
}

/* ── Moon ────────────────────────────────────────────────────────── */
.moon {
  position: absolute;
  top: 18px;
  right: 48px;
  width: 52px;
  height: 52px;
  background: radial-gradient(circle at 38% 38%, #fffde7, #ffe082);
  border-radius: 50%;
  box-shadow:
    0 0 30px 10px rgba(255, 224, 130, 0.25),
    0 0 60px 20px rgba(255, 200,  80, 0.10);
  pointer-events: none;
}

.moon::after {
  content: '';
  position: absolute;
  top: 6px;
  left: 14px;
  width: 32px;
  height: 32px;
  background: #ffffff;
  border-radius: 50%;
}

/* ── Theme Toggle Button ─────────────────────────────────────────── */
.theme-toggle {
  position: absolute;
  top: 20px;
  right: 20px;
  z-index: 100;
  cursor: pointer;
  width: 50px;
  height: 50px;
  background: rgba(255, 255, 255, 0.2);
  backdrop-filter: blur(10px);
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.3s ease;
  border: 1px solid rgba(255, 255, 255, 0.3);
}

.theme-toggle:hover {
  transform: scale(1.1);
  background: rgba(255, 255, 255, 0.3);
}

.toggle-icon {
  font-size: 1.8rem;
  transition: transform 0.3s ease;
}

.theme-toggle:active .toggle-icon {
  transform: rotate(180deg);
}

/* ── Title ───────────────────────────────────────────────────────── */
.title-row {
  position: relative;
  z-index: 2;
  margin-top: 30px;
  text-align: center;
}

.title-row h1 {
  font-family: 'Fredoka One', cursive;
  font-size: 2.8rem;
  color: #e0aaff;
  text-shadow:
    0 0 20px rgba(180, 100, 255, 0.8),
    3px 3px 0 #7b2ff7,
    6px 6px 0 rgba(123, 47, 247, 0.3);
  letter-spacing: 1px;
  line-height: 1;
  transition: all 0.3s ease;
}

.subtitle {
  font-size: 0.95rem;
  color: var(--text-secondary);
  margin-top: 6px;
  font-weight: 700;
  letter-spacing: 0.5px;
  transition: color 0.3s ease;
}

/* ── Score panel ─────────────────────────────────────────────────── */
.score-panel {
  position: relative;
  z-index: 2;
  display: flex;
  gap: 16px;
  margin: 20px 0 0;
  flex-wrap: wrap;
  justify-content: center;
}

.score-card {
  background: rgba(255, 255, 255, 0.06);
  border: 1px solid rgba(160, 100, 255, 0.25);
  border-radius: 20px;
  padding: 10px 22px;
  text-align: center;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.4), inset 0 1px 0 rgba(255, 255, 255, 0.07);
  min-width: 110px;
  transition: transform 0.15s, background 0.3s ease;
  cursor: pointer;
}

.score-card:hover { transform: translateY(-2px); }

.coin-card {
  background: linear-gradient(135deg, rgba(255, 215, 0, 0.15), rgba(255, 165, 0, 0.1));
  border: 1px solid rgba(255, 215, 0, 0.4);
}

.coin-value {
  color: #ffd700;
  text-shadow: 0 0 10px rgba(255, 215, 0, 0.5);
}

.score-label {
  font-size: 0.72rem;
  font-weight: 800;
  letter-spacing: 1.5px;
  text-transform: uppercase;
  color: var(--text-secondary);
  transition: color 0.3s ease;
}

.score-value {
  font-family: 'Fredoka One', cursive;
  font-size: 2rem;
  line-height: 1.1;
  color: var(--text-primary);
  transition: color 0.3s ease;
}

.combo-active {
  color: #fbbf24;
  text-shadow: 0 0 10px rgba(251, 191, 36, 0.5);
  animation: comboPulse 0.5s ease-in-out;
}

@keyframes comboPulse {
  0%, 100% { transform: scale(1); }
  50% { transform: scale(1.2); color: #f59e0b; }
}

.score-bounce {
  animation: scoreBounce 0.35s cubic-bezier(0.36, 0.07, 0.19, 0.97);
}

@keyframes scoreBounce {
  0%, 100% { transform: scale(1); }
  40% { transform: scale(1.5) rotate(-6deg); color: #f9a8d4; }
  70% { transform: scale(0.9); }
}

.timer-warn {
  color: #f87171 !important;
  text-shadow: 0 0 10px rgba(248, 113, 113, 0.5);
}

/* ── Shop Panel ──────────────────────────────────────────────────── */
.shop-panel {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  z-index: 1000;
  background: linear-gradient(135deg, #1a1a2e, #16213e);
  border-radius: 20px;
  padding: 20px;
  width: 90%;
  max-width: 500px;
  max-height: 80vh;
  overflow-y: auto;
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.5), 0 0 0 2px rgba(124, 58, 237, 0.3);
  border: 1px solid rgba(167, 139, 250, 0.3);
  transition: all 0.3s ease;
}

.shop-slide-enter-active, .shop-slide-leave-active {
  transition: all 0.3s ease;
}
.shop-slide-enter, .shop-slide-leave-to {
  transform: translate(-50%, -50%) scale(0.9);
  opacity: 0;
}

.shop-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
  padding-bottom: 10px;
  border-bottom: 2px solid rgba(167, 139, 250, 0.3);
}

.shop-header h3 {
  font-family: 'Fredoka One', cursive;
  color: #ffd700;
  margin: 0;
}

.close-shop {
  background: none;
  border: none;
  color: #c4b5fd;
  font-size: 1.5rem;
  cursor: pointer;
  padding: 0 5px;
}

.shop-items {
  display: flex;
  flex-direction: column;
  gap: 15px;
  margin-bottom: 20px;
}

.shop-item {
  display: grid;
  grid-template-columns: auto 1fr auto auto;
  gap: 12px;
  align-items: center;
  background: rgba(255, 255, 255, 0.05);
  padding: 12px;
  border-radius: 12px;
  position: relative;
}

.item-icon {
  font-size: 2rem;
}

.item-info {
  text-align: left;
}

.item-name {
  font-weight: bold;
  color: #d8b4fe;
  margin-bottom: 4px;
}

.item-desc {
  font-size: 0.75rem;
  color: rgba(200, 180, 255, 0.7);
}

.item-cost {
  display: flex;
  align-items: center;
  gap: 4px;
  font-weight: bold;
}

.cost-value {
  color: #ffd700;
  font-size: 1.1rem;
}

.buy-btn {
  background: linear-gradient(135deg, #a78bfa, #7c3aed);
  border: none;
  padding: 6px 15px;
  border-radius: 20px;
  color: white;
  cursor: pointer;
  font-weight: bold;
  transition: transform 0.1s;
}

.buy-btn:active { transform: scale(0.95); }
.buy-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.active-badge {
  position: absolute;
  top: -5px;
  right: -5px;
  background: #10b981;
  color: white;
  font-size: 0.7rem;
  padding: 2px 8px;
  border-radius: 10px;
  animation: pulse 1s infinite;
}

.active-powerups {
  margin-top: 15px;
  padding-top: 15px;
  border-top: 1px solid rgba(167, 139, 250, 0.3);
}

.active-title {
  font-size: 0.8rem;
  color: rgba(200, 180, 255, 0.8);
  margin-bottom: 8px;
}

.active-list {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
}

.active-powerup {
  background: rgba(167, 139, 250, 0.2);
  padding: 4px 10px;
  border-radius: 15px;
  font-size: 0.8rem;
  color: #a78bfa;
}

@keyframes pulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.6; }
}

/* ── Difficulty Buttons ───────────────────────────────────────────── */
.difficulty-row {
  position: relative;
  z-index: 2;
  margin: 15px 0 0;
  display: flex;
  gap: 10px;
  justify-content: center;
}

.difficulty-btn {
  font-family: 'Fredoka One', cursive;
  font-size: 0.9rem;
  padding: 8px 20px;
  border: none;
  border-radius: 50px;
  cursor: pointer;
  background: rgba(255, 255, 255, 0.1);
  color: #c4b5fd;
  transition: all 0.2s;
}

.difficulty-btn:hover {
  transform: translateY(-2px);
  background: rgba(124, 58, 237, 0.4);
}

.active-diff {
  background: linear-gradient(135deg, #a78bfa 0%, #7c3aed 100%);
  color: white;
  box-shadow: 0 0 15px rgba(124, 58, 237, 0.5);
}

/* ── Statistics Panel ─────────────────────────────────────────────── */
.stats-panel {
  position: relative;
  z-index: 2;
  margin: 15px 0 0;
  background: rgba(0, 0, 0, 0.5);
  border-radius: 20px;
  padding: 15px 20px;
  backdrop-filter: blur(10px);
  width: 100%;
  max-width: 500px;
  transition: all 0.3s ease;
}

.stats-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 15px;
}

.stat-item {
  text-align: center;
}

.stat-label {
  font-size: 0.7rem;
  font-weight: 700;
  color: rgba(180, 140, 255, 0.8);
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.stat-value {
  font-family: 'Fredoka One', cursive;
  font-size: 1.2rem;
  color: #d8b4fe;
  margin-top: 4px;
}

.toggle-stats {
  position: relative;
  z-index: 2;
  background: none;
  border: none;
  color: rgba(180, 140, 255, 0.6);
  cursor: pointer;
  font-size: 0.75rem;
  margin-top: 5px;
  font-family: 'Nunito', sans-serif;
  font-weight: 700;
}

/* ── Buttons ─────────────────────────────────────────────────────── */
.btn-row {
  position: relative;
  z-index: 2;
  margin: 18px 0 0;
  display: flex;
  gap: 12px;
  flex-wrap: wrap;
  justify-content: center;
}

.btn {
  font-family: 'Fredoka One', cursive;
  font-size: 1.1rem;
  letter-spacing: 0.5px;
  border: none;
  cursor: pointer;
  padding: 12px 32px;
  border-radius: 50px;
  transition: transform 0.12s, box-shadow 0.12s, background 0.3s ease;
}

.btn:active { transform: translateY(3px); }

.btn-start {
  background: linear-gradient(135deg, #a78bfa 0%, #7c3aed 100%);
  color: #ede9fe;
  box-shadow: 0 5px 0 #4c1d95, 0 8px 30px rgba(124, 58, 237, 0.5);
}

.btn-start:not(:disabled):hover {
  transform: translateY(-2px);
  box-shadow: 0 8px 0 #4c1d95, 0 12px 35px rgba(124, 58, 237, 0.6);
}

.btn-reset {
  background: linear-gradient(135deg, #f472b6 0%, #db2777 100%);
  color: #fff;
  box-shadow: 0 5px 0 #831843, 0 8px 30px rgba(219, 39, 119, 0.4);
}

.btn-reset:hover {
  transform: translateY(-2px);
  box-shadow: 0 8px 0 #831843, 0 12px 35px rgba(219, 39, 119, 0.5);
}

/* ── Status & notes ──────────────────────────────────────────────── */
.status-msg {
  position: relative;
  z-index: 2;
  font-size: 0.9rem;
  font-weight: 700;
  color: var(--text-secondary);
  margin-top: 10px;
  min-height: 22px;
  letter-spacing: 0.3px;
  transition: color 0.3s ease;
}

.high-score-note {
  position: relative;
  z-index: 2;
  margin-top: 14px;
  font-size: 0.82rem;
  color: var(--text-secondary);
  font-weight: 700;
  min-height: 20px;
  transition: color 0.3s ease;
}

/* ── Meadow ──────────────────────────────────────────────────────── */
.meadow {
  position: relative;
  z-index: 2;
  margin-top: 22px;
  background: linear-gradient(180deg, #4b6104 0%, #0f1f0f 50%, #433302 100%);
  border-radius: 32px;
  padding: 24px 24px 32px;
  box-shadow:
    0 0 0 1px rgba(100, 255, 150, 0.12),
    0 12px 50px rgba(0, 0, 0, 0.6),
    inset 0 1px 0 rgba(100, 255, 150, 0.10);
  width: 100%;
  max-width: 560px;
  transition: background 0.3s ease;
}

.meadow-top {
  text-align: center;
  font-size: 1.1rem;
  margin-bottom: 18px;
  letter-spacing: 3px;
  opacity: 0.7;
}

/* ── Holes grid ──────────────────────────────────────────────────── */
.holes-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 18px;
}

.hole-wrap {
  display: flex;
  flex-direction: column;
  align-items: center;
  position: relative;
}

.hole {
  width: 100%;
  max-width: 130px;
  aspect-ratio: 1.2 / 1;
  background: radial-gradient(ellipse at 50% 65%, #6f3c04 0%, #431e02 50%, #111a11 100%);
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  position: relative;
  box-shadow:
    0 8px 24px rgba(0, 0, 0, 0.7),
    inset 0 -3px 12px rgba(0, 0, 0, 0.8),
    inset 0 2px 6px rgba(100, 255, 120, 0.04),
    0 0 0 2px rgba(50, 120, 60, 0.2);
  overflow: hidden;
  transition: transform 0.1s, background 0.3s ease, box-shadow 0.3s ease;
}

.hole:hover {
  transform: scale(1.05);
  box-shadow:
    0 8px 24px rgba(0, 0, 0, 0.7),
    inset 0 -3px 12px rgba(0, 0, 0, 0.8),
    0 0 0 2px rgba(150, 255, 170, 0.3);
}

.hole:active { transform: scale(0.95); }

.hit-flash {
  animation: hitFlash 0.45s ease-out;
}

@keyframes hitFlash {
  0%   {
    background: radial-gradient(ellipse at 50% 65%, #7c3aed 0%, #4c1d95 60%, #2e1065 100%);
    transform: scale(1.18);
    box-shadow: 0 0 0 4px rgba(167, 139, 250, 0.6), 0 0 30px rgba(167, 139, 250, 0.4);
  }
  50%  { background: radial-gradient(ellipse at 50% 65%, #f472b6 0%, #db2777 60%, #831843 100%); }
  100% {
    background: radial-gradient(ellipse at 50% 65%, #050a05 0%, #0a120a 50%, #111a11 100%);
    transform: scale(1);
  }
}

/* ── Mole ────────────────────────────────────────────────────────── */
.mole {
  font-size: 3rem;
  animation: molePopUp 0.25s cubic-bezier(0.175, 0.885, 0.32, 1.275);
  transform-origin: bottom center;
  user-select: none;
  filter: drop-shadow(0 -6px 10px rgba(180, 100, 255, 0.5));
  position: relative;
  z-index: 1;
}

.golden-mole {
  filter: drop-shadow(0 0 15px rgba(255, 215, 0, 0.8));
  animation: goldenGlow 0.5s ease-in-out infinite alternate;
}

.slow-mole {
  animation: molePopUp 0.5s cubic-bezier(0.175, 0.885, 0.32, 1.275);
}

.double-mole {
  filter: drop-shadow(0 0 15px rgba(147, 51, 234, 0.8));
}

.double-indicator {
  position: absolute;
  top: -20px;
  right: -20px;
  font-size: 1rem;
  background: rgba(124, 58, 237, 0.8);
  border-radius: 50%;
  padding: 4px;
  width: 25px;
  height: 25px;
  display: flex;
  align-items: center;
  justify-content: center;
}

@keyframes goldenGlow {
  from { filter: drop-shadow(0 0 5px rgba(255, 215, 0, 0.5)); }
  to { filter: drop-shadow(0 0 20px rgba(255, 215, 0, 1)); }
}

@keyframes molePopUp {
  0%   { transform: translateY(70%) scale(0.3); opacity: 0; }
  60%  { transform: translateY(-12%) scale(1.2); opacity: 1; }
  100% { transform: translateY(0) scale(1);     opacity: 1; }
}

.mole-exit {
  animation: molePopDown 0.22s ease-in forwards;
}

@keyframes molePopDown {
  0%   { transform: translateY(0) scale(1);    opacity: 1; }
  100% { transform: translateY(80%) scale(0.2); opacity: 0; }
}

/* ── Grass strip ─────────────────────────────────────────────────── */
.grass-strip {
  width: 100%;
  max-width: 130px;
  height: 14px;
  background: linear-gradient(180deg, #1a3d1a 0%, #0f2a0f 100%);
  border-radius: 0 0 10px 10px;
  margin-top: -2px;
  box-shadow: 0 4px 10px rgba(0, 0, 0, 0.5);
}

/* ── Particle Canvas ──────────────────────────────────────────────── */
.particle-canvas {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  width: 100px;
  height: 100px;
  pointer-events: none;
  z-index: 20;
}

/* ── Score / miss popups ─────────────────────────────────────────── */
.pts-popup,
.miss-popup {
  position: absolute;
  top: -5px;
  left: 50%;
  transform: translateX(-50%);
  font-family: 'Fredoka One', cursive;
  pointer-events: none;
  z-index: 10;
  animation: ptsFloat 0.9s ease-out forwards;
}

.pts-popup {
  font-size: 1.5rem;
  color: #fde68a;
  text-shadow: 2px 2px 0 #92400e, 0 0 10px rgba(253, 230, 138, 0.6);
}

.miss-popup {
  font-size: 1rem;
  color: #f87171;
  text-shadow: 1px 1px 0 #7f1d1d;
}

@keyframes ptsFloat {
  0%   { opacity: 1; transform: translateX(-50%) translateY(0)     scale(1);   }
  50%  { opacity: 1; transform: translateX(-50%) translateY(-40px) scale(1.3); }
  100% { opacity: 0; transform: translateX(-50%) translateY(-72px) scale(0.7); }
}

/* ── DAY MODE SPECIFIC STYLES ────────────────────────────────────── */

/* Day Mode - Hide night elements */
.day-mode .moon {
  display: none !important;
}

.day-mode .stars-bg {
  opacity: 0.3;
}

.day-mode .firefly {
  opacity: 0.4;
}

/* Day Mode - Meadow */
.day-mode .meadow {
  background: linear-gradient(180deg, #7cb342 0%, #558b2f 50%, #33691e 100%);
}

/* Day Mode - Holes */
.day-mode .hole {
  background: radial-gradient(ellipse at 50% 65%, #8d6e63 0%, #5d4037 50%, #3e2723 100%);
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.3), inset 0 -3px 12px rgba(0, 0, 0, 0.5);
}

/* Day Mode - Title */
.day-mode .title-row h1 {
  color: #4a148c;
  text-shadow: 0 0 20px rgba(255, 215, 0, 0.5), 3px 3px 0 #ce93d8;
}

/* Day Mode - Score Cards */
.day-mode .score-card {
  background: rgba(255, 255, 255, 0.5);
  border: 1px solid rgba(0, 0, 0, 0.1);
}

.day-mode .score-label,
.day-mode .status-msg,
.day-mode .high-score-note {
  color: #2D3748;
}

.day-mode .score-value {
  color: #2D3748;
}

/* Day Mode - Buttons */
.day-mode .btn-start {
  background: linear-gradient(135deg, #66bb6a 0%, #43a047 100%);
  box-shadow: 0 5px 0 #2e7d32;
}

.day-mode .btn-reset {
  background: linear-gradient(135deg, #ff7043 0%, #f4511e 100%);
  box-shadow: 0 5px 0 #bf360c;
}

/* Day Mode - Difficulty Buttons */
.day-mode .difficulty-btn {
  background: rgba(0, 0, 0, 0.1);
  color: #2D3748;
}

.day-mode .active-diff {
  background: linear-gradient(135deg, #66bb6a 0%, #43a047 100%);
  color: white;
}

/* Day Mode - Shop */
.day-mode .shop-panel {
  background: linear-gradient(135deg, #ffffff, #f5f5f5);
  border: 1px solid rgba(0, 0, 0, 0.2);
}

.day-mode .shop-header h3 {
  color: #ff9800;
}

.day-mode .item-name {
  color: #2D3748;
}

.day-mode .shop-item {
  background: rgba(0, 0, 0, 0.05);
}

.day-mode .buy-btn {
  background: linear-gradient(135deg, #66bb6a, #43a047);
}

/* Day Mode - Statistics */
.day-mode .stats-panel {
  background: rgba(255, 255, 255, 0.8);
  backdrop-filter: blur(10px);
}

.day-mode .stat-label {
  color: #4a5568;
}

.day-mode .stat-value {
  color: #2d3748;
}

/* Day Mode - Theme Toggle */
.day-mode .theme-toggle {
  background: rgba(0, 0, 0, 0.2);
  border: 1px solid rgba(0, 0, 0, 0.3);
}

/* ── Responsive ──────────────────────────────────────────────────── */
@media (max-width: 480px) {
  .title-row h1  { font-size: 2rem; }
  .meadow        { padding: 18px 12px 24px; }
  .holes-grid    { gap: 12px; }
  .mole          { font-size: 2.2rem; }
  .stats-grid    { gap: 8px; }
  .stat-value    { font-size: 1rem; }
  .score-card    { padding: 8px 15px; min-width: 80px; }
  .score-value   { font-size: 1.5rem; }
  .shop-item     { grid-template-columns: auto 1fr auto; }
  .item-desc     { display: none; }
  .theme-toggle  { width: 40px; height: 40px; top: 15px; right: 15px; }
  .toggle-icon   { font-size: 1.4rem; }
}
</style>
