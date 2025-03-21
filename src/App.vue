<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue'
import hitAudioSrc from '@/assets/hit.mp3'

const hitAudio = ref<HTMLAudioElement | null>(null) as any

// 游戏状态
const hitCount = ref(0)
const gameOver = ref(false)
const gameStarted = ref(false)
const gameTime = ref(0)
const currentMessage = ref('')
let gameTimer: number

// 游戏控制
function startGame() {
  gameStarted.value = true
  gameOver.value = false
  hitCount.value = 0
  gameTime.value = 0
  currentMessage.value = ''
  gameTimer = window.setInterval(() => {
    gameTime.value++
  }, 1000)
}

function resetGame(isFinished = false) {
  if (isFinished) {
    gameStarted.value = false
    gameOver.value = false
    hitCount.value = 0
    gameTime.value = 0
    clearInterval(gameTimer)
    currentMessage.value = ''
  } else {
    startGame()
  }
}

// 雪球发射器位置  
const launcherPosition = ref(50) // 百分比位置
// const isDragging = ref(false)

// 雪人位置
const snowmanPosition = ref(50) // 百分比位置
const snowmanDirection = ref(1) // 1 向右, -1 向左
const snowmanSpeed = ref(0.3) // 移动速度

// 雪球状态
const snowballs = ref<Array<{ x: number; y: number; id: number }>>([])

// 溅射效果粒子
const particles = ref<Array<{ x: number; y: number; id: number; dx: number; dy: number }>>([])
let nextSnowballId = 0

// 键盘状态
const keyState = ref({
  ArrowLeft: false,
  ArrowRight: false,
  Space: false,
  lastShot: Date.now()
}) as any

// 键盘控制
function handleKeydown(event: KeyboardEvent) {
  if (event.code === 'Space' || event.code === 'ArrowLeft' || event.code === 'ArrowRight') {
    event.preventDefault()
    keyState.value[event.code as keyof typeof keyState.value] = true
  }
}

function handleKeyup(event: KeyboardEvent) {
  if (event.code === 'Space' || event.code === 'ArrowLeft' || event.code === 'ArrowRight') {
    event.preventDefault()
    keyState.value[event.code as keyof typeof keyState.value] = false
  }
}


// 发射雪球
function shootSnowball() {
  snowballs.value.push({
    x: launcherPosition.value,
    y: 10, // 从底部发射
    id: nextSnowballId++
  })
}

// 游戏循环
let gameLoop: number

function updateGame() {
  // 处理键盘输入
  if (keyState.value.ArrowLeft) {
    launcherPosition.value = Math.max(0, launcherPosition.value - 1)
  }
  if (keyState.value.ArrowRight) {
    launcherPosition.value = Math.min(100, launcherPosition.value + 1)
  }
  // 处理空格键发射
  const now = Date.now()
  if (keyState.value.Space && now - keyState.value.lastShot > 100) { // 降低发射频率限制
    shootSnowball()
    keyState.value.lastShot = now
  }

  // 更新雪人位置
  if (snowmanPosition.value <= 0) snowmanDirection.value = 1
  if (snowmanPosition.value >= 100) snowmanDirection.value = -1
  snowmanPosition.value += snowmanSpeed.value * snowmanDirection.value

  // 更新雪球位置
  snowballs.value = snowballs.value.filter(ball => {
    ball.y += 2 // 向上移动
    
    // 检测碰撞
    if (gameStarted.value && Math.abs(ball.x - snowmanPosition.value) < 10 && ball.y > 80) {
      hitCount.value++
      // 播放击中音效
      if (hitAudio.value) {
        hitAudio.value.currentTime = 0
        hitAudio.value.play()
      }
      
      // 创建溅射效果
      for (let i = 0; i < 8; i++) {
        const angle = (Math.PI * 2 * i) / 8
        particles.value.push({
          x: ball.x,
          y: ball.y,
          id: nextSnowballId++,
          dx: Math.cos(angle) * 2,
          dy: Math.sin(angle) * 2
        })
      }
      return false
    }
    
    return ball.y > 0
  })

  // 更新粒子
  particles.value = particles.value.filter(particle => {
    particle.x += particle.dx
    particle.y += particle.dy
    particle.dy += 0.1 // 重力效果
    return particle.y < 100 && particle.y > 0 && particle.x > 0 && particle.x < 100
  })

  // 检查游戏状态和更新提示信息
  if (hitCount.value >= 50 && hitCount.value < 100 && currentMessage.value !== 'Nice shot! Your stress is melting away!') {
    currentMessage.value = 'Nice shot! Your stress is melting away!'
  } else if (hitCount.value >= 100 && hitCount.value < 150 && currentMessage.value !== 'Keep going, you\'re letting go!') {
    currentMessage.value = 'Keep going, you\'re letting go!'
  } else if (hitCount.value >= 150) {
    gameOver.value = true
    currentMessage.value = 'Take a deep breath. Feel the calm. You\'ve got this.'
    clearInterval(gameTimer)
  }
}

// 触摸状态
const touchStartX = ref(0)
const isTouching = ref(false)

// 触摸控制
function handleTouchStart(event: TouchEvent) {
  event.preventDefault()
  touchStartX.value = event.touches[0].clientX
  isTouching.value = true
}

function handleTouchMove(event: TouchEvent) {
  if (!isTouching.value) return
  event.preventDefault()
  const deltaX = event.touches[0].clientX - touchStartX.value
  touchStartX.value = event.touches[0].clientX
  launcherPosition.value = Math.max(0, Math.min(100, launcherPosition.value + (deltaX / window.innerWidth) * 100))
}

function handleTouchEnd() {
  isTouching.value = false
}

function handleLauncherClick() {
  if (!gameStarted.value || gameOver.value) return
  shootSnowball()
}

onMounted(() => {
  window.addEventListener('keydown', handleKeydown)
  window.addEventListener('keyup', handleKeyup)
  window.addEventListener('touchstart', handleTouchStart, { passive: true })
  window.addEventListener('touchmove', handleTouchMove, { passive: true })
  window.addEventListener('touchend', handleTouchEnd)
  gameLoop = window.setInterval(updateGame, 16)
})

onUnmounted(() => {
  window.removeEventListener('keydown', handleKeydown)
  window.removeEventListener('keyup', handleKeyup)
  window.removeEventListener('touchstart', handleTouchStart)
  window.removeEventListener('touchmove', handleTouchMove)
  window.removeEventListener('touchend', handleTouchEnd)
  clearInterval(gameLoop)
  clearInterval(gameTimer)
})
</script>

<template>
  <div class="game-container">
    <div class="header">
      <div class="score">Score: {{ hitCount }}</div>
      <div class="timer" v-if="gameStarted">Time: {{ Math.floor(gameTime / 60) }}:{{ (gameTime % 60).toString().padStart(2, '0') }}</div>
    </div>
    
    <!-- 游戏控制按钮 -->
    <div class="game-controls" v-if="!gameStarted && !gameOver">
      <p class="intro-message">Imagine this snowman is your worries, your stress, or anything bothering you<br>Take a deep breath... and throw snowballs at it! Watch your worries melt away.</p>
      <button class="control-btn" @click="startGame">Start the game</button>
    </div>

    <!-- 游戏区域 -->
    <div class="game-area" v-show="gameStarted && !gameOver">
      <!-- 雪人 -->
      <div 
        class="snowman" 
            :class="{
          'snowman--damaged': hitCount >= 50,
          'snowman--broken': hitCount >= 100
        }"
        :style="{
          left: `${snowmanPosition}%`
        }"
      ></div>
      
      <!-- 雪球 -->
      <div
        v-for="ball in snowballs"
        :key="ball.id"
        class="snowball"
        :style="{
          left: `${ball.x}%`,
          bottom: `${ball.y}%`
        }"
      ></div>
      
      <!-- 溅射效果 -->
      <div
        v-for="particle in particles"
        :key="particle.id"
        class="particle"
        :style="{
          left: `${particle.x}%`,
          bottom: `${particle.y}%`
        }"
      ></div>
      
      <!-- 发射器 -->
      <div
        class="launcher"
        :style="{ left: `${launcherPosition}%` }"
        @click="handleLauncherClick"
      ></div>
    </div>
    
    <!-- 音效 -->
    <audio ref="hitAudio" :src="hitAudioSrc" preload="auto"></audio>
    
    <!-- 游戏结束提示 -->
    <div v-if="gameOver" class="game-over">
      <h2>{{ currentMessage }}</h2>
      <div class="feedback-section">
        <h3>Now, how do you feel?</h3>
        <div class="feedback-buttons">
          <button class="control-btn" @click="resetGame(true)">Yes, I'm good!</button>
          <button class="control-btn" @click="resetGame(false)">No, one more round</button>
        </div>
      </div>
    </div>
    
    <!-- 游戏提示信息 -->
    <div v-if="currentMessage && !gameOver" class="game-message">
      {{ currentMessage }}
    </div>
  </div>
</template>

<style scoped>
.game-container {
  width: 100vw;
  height: 100vh;
  display: flex;
  flex-direction: column;
  align-items: center;
  background: linear-gradient(to bottom, #e3f2fd, #bbdefb);
  margin: 0;
  padding: 0;
  overflow: hidden;
  position: fixed;
  top: 0;
  left: 0;
  touch-action: none;
}

.score {
  font-size: clamp(24px, 5vw, 32px);
  margin: 10px 0;
  color: #1565c0;
  text-shadow: 2px 2px 4px rgba(0,0,0,0.1);
}

.game-area {
  width: 100vw;
  height: calc(100vh - 60px);
  position: relative;
  background: transparent;
  overflow: hidden;
  touch-action: none;
}

.snowman {
  position: absolute;
  top: 20px;
  width: clamp(100px, 30vw, 160px);
  height: clamp(125px, 37.5vw, 200px);
  transform: translateX(-50%);
  background: url('@/assets/1.png') no-repeat center;
  background-size: contain;
  transition: background-image 0.3s;
}
.snowman--damaged {
  background-image: url('@/assets/2.png');
}

.snowman--broken {
  background-image: url('@/assets/3.png');
}


.snowball {
  position: absolute;
  width: clamp(20px, 5vw, 30px);
  height: clamp(20px, 5vw, 30px);
  background: white;
  border-radius: 50%;
  transform: translateX(-50%);
  box-shadow: 0 0 10px rgba(0,0,0,0.1);
}

.launcher {
  position: absolute;
  bottom: clamp(20px, 5vh, 40px);
  width: clamp(80px, 20vw, 120px);
  height: clamp(40px, 10vw, 60px);
  background: url('@/assets/cannon.svg') no-repeat center;
  background-size: contain;
  transform: translateX(-50%) rotate(60deg);
  cursor: pointer;
  transition: transform 0.2s, left 0.1s linear;
  -webkit-tap-highlight-color: transparent;
  touch-action: none;
  user-select: none;
  -webkit-user-select: none;
}

.intro-message {
  font-size: clamp(18px, 4vw, 24px);
  color: #1565c0;
  text-align: center;
  margin: 20px 10px;
  line-height: 1.6;
  max-width: 600px;
}

.game-message {
  position: fixed;
  top: clamp(60px, 15vh, 100px);
  left: 50%;
  transform: translateX(-50%);
  background: rgba(255, 255, 255, 0.9);
  padding: 12px 20px;
  border-radius: 20px;
  font-size: clamp(16px, 3.5vw, 20px);
  color: #1565c0;
  text-align: center;
  animation: fadeIn 0.5s ease-out;
  z-index: 100;
  box-shadow: 0 4px 15px rgba(0,0,0,0.1);
  max-width: 90vw;
}

.control-btn {
  background: #1565c0;
  color: white;
  border: none;
  padding: clamp(8px, 2vw, 12px) clamp(16px, 4vw, 24px);
  border-radius: 25px;
  font-size: clamp(16px, 3.5vw, 18px);
  cursor: pointer;
  transition: transform 0.2s, background-color 0.2s;
  box-shadow: 0 4px 15px rgba(0,0,0,0.1);
  touch-action: manipulation;
}

.control-btn:hover {
  background: #1976d2;
  transform: translateY(-2px);
}

@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translate(-50%, -20px);
  }
  to {
    opacity: 1;
    transform: translate(-50%, 0);
  }
}
</style>
