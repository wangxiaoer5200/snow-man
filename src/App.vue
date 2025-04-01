<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue'
import hitAudioSrc from '@/assets/hit.mp3'

const hitAudio = ref<HTMLAudioElement | null>(null) as any

// 游戏状态
const hitCount = ref(0)
const gameOver = ref(false)
const gameStarted = ref(false)
const gameTime = ref(0)
const bestTime = ref(localStorage.getItem('bestTime') ? parseInt(localStorage.getItem('bestTime')!) : 0)
const currentMessage = ref('')
const messageVisible = ref(false)
let gameTimer: number
let messageTimer: number

// 显示消息的函数
function showMessage(message: string) {
  currentMessage.value = message
  messageVisible.value = true
  
  // 清除之前的定时器
  if (messageTimer) {
    clearTimeout(messageTimer)
  }
  
  // 设置3秒后消失
  messageTimer = window.setTimeout(() => {
    messageVisible.value = false
  }, 3000)
}

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

// 分享功能
function getShareContent() {
  const score = hitCount.value
  const time = `${Math.floor(gameTime.value / 60)}:${(gameTime.value % 60).toString().padStart(2, '0')}`
  return {
    title: '雪球压力消除器',
    text: `我在雪球压力消除器中击中了${score}次雪人，用时${time}！来和我一起玩吧！`,
    url: window.location.href
  }
}

function shareToFacebook() {
  const content = getShareContent()
  // 使用Facebook分享API
  const shareUrl = `https://www.facebook.com/sharer/sharer.php?u=${encodeURIComponent(content.url)}&quote=${encodeURIComponent(content.text)}`
  
  try {
    // 打开新窗口进行分享
    const width = 550
    const height = 450
    const left = (window.screen.width - width) / 2
    const top = (window.screen.height - height) / 2
    const features = `width=${width},height=${height},left=${left},top=${top},toolbar=no,menubar=no,scrollbars=yes,resizable=yes,location=no,status=no`
    
    const shareWindow = window.open(shareUrl, 'facebook-share-dialog', features)
    
    // 检查窗口是否被阻止
    if (shareWindow === null || typeof shareWindow === 'undefined') {
      showMessage('请允许弹出窗口以分享到Facebook')
    }
  } catch (error) {
    console.error('Facebook分享失败:', error)
    showMessage('分享到Facebook时出现错误，请稍后重试')
  }
}

function shareToTwitter() {
  const content = getShareContent()
  const shareUrl = `https://x.com/intent/tweet?text=${encodeURIComponent(content.text)}&url=${encodeURIComponent(content.url)}`
  window.open(shareUrl, '_blank')
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
const directionChangeChance = ref(0.02) // 每帧改变方向的概率
const speedChangeChance = ref(0.03) // 每帧改变速度的概率

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
  lastShot: Date.now(),
  moveSpeed: 2 // 固定移动速度
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
  // 处理键盘输入和更新发射台位置
  if (keyState.value.ArrowLeft) {
    launcherPosition.value = Math.max(0, launcherPosition.value - 2)
  } 
  if (keyState.value.ArrowRight) {
    launcherPosition.value = Math.min(100, launcherPosition.value + 2)
  }

  // 处理空格键发射
  const now = Date.now()
  if (keyState.value.Space && now - keyState.value.lastShot > 100) {
    shootSnowball()
    keyState.value.lastShot = now
  }

  // 更新雪人位置
  // 随机改变方向
  if (Math.random() < directionChangeChance.value) {
    snowmanDirection.value = Math.random() < 0.5 ? -1 : 1
  }
  // 随机改变速度
  if (Math.random() < speedChangeChance.value) {
    snowmanSpeed.value = 0.2 + Math.random() * 0.4 // 速度在0.2到0.6之间随机
  }
  // 边界检测和位置更新
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
      for (let i = 0; i < 12; i++) {
        const angle = (Math.PI * 2 * i) / 12
        const speed = 2 + Math.random() * 2
        particles.value.push({
          x: ball.x,
          y: ball.y,
          id: nextSnowballId++,
          dx: Math.cos(angle) * speed,
          dy: Math.sin(angle) * speed
        })
      }
      return false
    }
    
    // 检查雪球是否在游戏区域内
    return ball.y > 0 && ball.y < 100 && ball.x >= 0 && ball.x <= 100
  })

  // 更新粒子
  particles.value = particles.value.filter(particle => {
    particle.x += particle.dx
    particle.y += particle.dy
    particle.dy += 0.1 // 重力效果
    return particle.y < 100 && particle.y > 0 && particle.x > 0 && particle.x < 100
  })

  // 检查游戏状态和更新提示信息
   if (hitCount.value >= 200) {
    gameOver.value = true
    // 检查是否为最佳时间
    if (bestTime.value === 0 || gameTime.value < bestTime.value) {
      bestTime.value = gameTime.value
      localStorage.setItem('bestTime', gameTime.value.toString())
      currentMessage.value = `新纪录！用时 ${Math.floor(gameTime.value / 60)}:${(gameTime.value % 60).toString().padStart(2, '0')}！`
    } else {
      currentMessage.value = 'Take a deep breath. Feel the calm. You\'ve got this.'
    }
    clearInterval(gameTimer)
  }
}

// 触摸状态
const touchStartX = ref(0)
const isTouching = ref(false)

// 触摸控制
function handleTouchStart(event: TouchEvent) {
  touchStartX.value = event.touches[0].clientX
  isTouching.value = true
}

function handleTouchMove(event: TouchEvent) {
  if (!isTouching.value) return
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
      <div class="score" v-if="gameStarted">Time: {{ Math.floor(gameTime / 60) }}:{{ (gameTime % 60).toString().padStart(2, '0') }}</div>
      <div class="best-time" v-if="bestTime > 0">Best Record: {{ Math.floor(bestTime / 60) }}:{{ (bestTime % 60).toString().padStart(2, '0') }}</div>
    </div>
    
    <!-- 游戏控制按钮 -->
    <div class="game-controls" v-if="!gameStarted && !gameOver">
      <div class="intro-snowman"></div>
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
          'snowman--broken': hitCount >= 120
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
      <div class="end-snowman"></div>
      <h2>{{ currentMessage }}</h2>
      <div class="feedback-section">
        <h3>Now, how do you feel?</h3>
        <div class="feedback-buttons">
          <button class="control-btn" @click="resetGame(true)">Yes, I'm good!</button>
          <button class="control-btn" @click="resetGame(false)">No, one more round</button>
        </div>
      </div>
      <div class="share-section">
        <h3>Share your achievement!</h3>
        <div class="share-buttons">
          <button class="share-btn facebook" @click="shareToFacebook">
            <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="currentColor"><path d="M24 12.1c0-6.6-5.4-12-12-12s-12 5.4-12 12c0 6 4.4 11 10.1 11.9v-8.4h-3v-3.5h3v-2.6c0-3 1.8-4.7 4.5-4.7 1.3 0 2.7.2 2.7.2v3h-1.5c-1.5 0-2 .9-2 1.8v2.2h3.4l-.5 3.5h-2.8v8.4c5.7-.9 10.1-5.9 10.1-11.9"/></svg>
            Facebook
          </button>
          <button class="share-btn x" @click="shareToTwitter">
            <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="currentColor"><path d="M18.244 2.25h3.308l-7.227 8.26 8.502 11.24H16.17l-5.214-6.817L4.99 21.75H1.68l7.73-8.835L1.254 2.25H8.08l4.713 6.231zm-1.161 17.52h1.833L7.084 4.126H5.117z"/></svg>
            X
          </button>
        </div>
      </div>
    </div>
    
    <!-- 游戏提示信息 -->
    <div v-if="currentMessage && !gameOver" class="game-message" :class="{ 'fade-out': !messageVisible }">
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

.share-section {
  margin-top: 20px;
  text-align: center;
}

.share-buttons {
  display: flex;
  justify-content: center;
  gap: 10px;
  margin-top: 10px;
}

.share-btn {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 8px 16px;
  border: none;
  border-radius: 20px;
  color: white;
  font-size: 14px;
  cursor: pointer;
  transition: transform 0.2s;
}

.share-btn:hover {
  transform: scale(1.05);
}

.share-btn svg {
  width: 20px;
  height: 20px;
}

.wechat {
  background-color: #07C160;
}

.facebook {
  background-color: #1877F2;
}

.x {
  background-color: #000000;
}

.game-over {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  background: rgba(255, 255, 255, 0.95);
  padding: 20px;
  border-radius: 20px;
  text-align: center;
  box-shadow: 0 4px 15px rgba(0,0,0,0.1);
  max-width: 90vw;
  width: 400px;
}

.share-section {
  margin-top: 20px;
  padding-top: 20px;
  border-top: 1px solid #e0e0e0;
}

.share-buttons {
  display: flex;
  justify-content: center;
  gap: 10px;
  margin-top: 10px;
}

.share-btn {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 8px 16px;
  border: none;
  border-radius: 20px;
  cursor: pointer;
  font-size: 14px;
  color: white;
  transition: transform 0.2s, opacity 0.2s;
}

.share-btn:hover {
  transform: translateY(-2px);
  opacity: 0.9;
}

.share-btn.wechat {
  background-color: #07C160;
}

.share-btn.facebook {
  background-color: #1877F2;
}

.share-btn.x {
  background-color: #000000;
}

.share-btn svg {
  width: 20px;
  height: 20px;
}

.header {
  display: flex;
  justify-content: space-around;
  align-items: center;
  width: 100%;
}

.score, .best-time {
  font-size: clamp(24px, 5vw, 32px);
  margin: 10px 0;
  color: #1565c0;
  text-shadow: 2px 2px 4px rgba(0,0,0,0.1);
  user-select: none;
  -webkit-user-select: none;
  touch-action: none;
}

.best-time {
  color: #f57c00;
}

.game-area {
  width: 100vw;
  height: calc(100vh - 60px);
  position: relative;
  background: transparent;
  overflow: hidden;
  touch-action: none;
  user-select: none;
  -webkit-user-select: none;
}

.snowman {
  position: absolute;
  top: 20px;
  width: clamp(120px, 35vw, 180px);
  height: clamp(150px, 43.75vw, 225px);
  transform: translateX(-50%);
  background: url('@/assets/1.png') no-repeat center;
  background-size: 160%;
  transition: background-image 0.3s;
  user-select: none;
  -webkit-user-select: none;
}

.snowman--damaged {
  background-image: url('@/assets/2.png');
}

.snowman--broken {
  background-image: url('@/assets/3.png');
}

.snowball {
  position: absolute;
  width: clamp(30px, 8vw, 45px);
  height: clamp(30px, 8vw, 45px);
  background: white;
  border-radius: 50%;
  transform: translateX(-50%);
  box-shadow: 0 0 20px rgba(255,255,255,0.3);
  animation: pulse 1s infinite;
  user-select: none;
  -webkit-user-select: none;
}

.launcher {
  position: absolute;
  bottom: clamp(40px, 10vh, 60px);
  width: clamp(120px, 30vw, 180px);
  height: clamp(60px, 15vw, 90px); 
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
  opacity: 1;
  transition: opacity 0.5s ease-out;
  user-select: none;
  -webkit-user-select: none;
  touch-action: none;
}

.game-message.fade-out {
  opacity: 0;
}

.intro-snowman {
  width: clamp(120px, 35vw, 180px);
  height: clamp(150px, 43.75vw, 225px);
  background: url('@/assets/1.png') no-repeat center;
  background-size: contain;
  margin: 20px auto;
  animation: bounce 2s ease-in-out infinite;
}

.end-snowman {
  width: clamp(120px, 35vw, 180px);
  height: clamp(150px, 43.75vw, 225px);
  background: url('@/assets/4.png') no-repeat center;
  background-size: contain;
  margin: 20px auto;
  animation: wave 2s ease-in-out infinite;
}

@keyframes bounce {
  0%, 100% { transform: translateY(0); }
  50% { transform: translateY(-20px); }
}

@keyframes wave {
  0%, 100% { transform: rotate(0deg); }
  25% { transform: rotate(-5deg); }
  75% { transform: rotate(5deg); }
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

.particle {
  position: absolute;
  width: 12px;
  height: 12px;
  background: white;
  border-radius: 50%;
  transform: translate(-50%, -50%);
  opacity: 0.8;
  animation: fadeOut 0.8s forwards;
}

@keyframes pulse {
  0% { transform: translateX(-50%) scale(1); }
  50% { transform: translateX(-50%) scale(1.1); }
  100% { transform: translateX(-50%) scale(1); }
}

@keyframes fadeOut {
  0% { opacity: 0.8; transform: translate(-50%, -50%) scale(1); }
  100% { opacity: 0; transform: translate(-50%, -50%) scale(0.5); }
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
