<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue'

// 游戏状态
const hitCount = ref(0)
const gameOver = ref(false)
const gameStarted = ref(false)
const gameTime = ref(0)
let gameTimer: number

// 游戏控制
function startGame() {
  gameStarted.value = true
  gameOver.value = false
  hitCount.value = 0
  gameTime.value = 0
  gameTimer = window.setInterval(() => {
    gameTime.value++
  }, 1000)
}

function resetGame() {
  gameStarted.value = false
  gameOver.value = false
  hitCount.value = 0
  gameTime.value = 0
  clearInterval(gameTimer)
}

// 雪球发射器位置  
const launcherPosition = ref(50) // 百分比位置
// const isDragging = ref(false)

// 雪人位置
const snowmanPosition = ref(50) // 百分比位置
const snowmanDirection = ref(1) // 1 向右, -1 向左
const snowmanSpeed = ref(0.8) // 移动速度

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
    if (Math.abs(ball.x - snowmanPosition.value) < 10 && ball.y > 80) {
      hitCount.value++
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

  // 检查游戏状态
  if (hitCount.value >= 150) {
    gameOver.value = true
    clearInterval(gameTimer)
  }
}

onMounted(() => {
  window.addEventListener('keydown', handleKeydown)
  window.addEventListener('keyup', handleKeyup)
  gameLoop = window.setInterval(updateGame, 16)
})

onUnmounted(() => {
  window.removeEventListener('keydown', handleKeydown)
  window.removeEventListener('keyup', handleKeyup)
  clearInterval(gameLoop)
  clearInterval(gameTimer)
})
</script>

<template>
  <div class="game-container">
    <div class="header">
      <div class="score">击中次数: {{ hitCount }}</div>
      <div class="timer" v-if="gameStarted">游戏时间: {{ Math.floor(gameTime / 60) }}:{{ (gameTime % 60).toString().padStart(2, '0') }}</div>
    </div>
    
    <!-- 游戏控制按钮 -->
    <div class="game-controls" v-if="!gameStarted && !gameOver">
      <button class="control-btn" @click="startGame">开始游戏</button>
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
      ></div>
    </div>
    
    <!-- 游戏结束提示 -->
    <div v-if="gameOver" class="game-over">
      <h2>恭喜你获得胜利！</h2>
      <p>用时: {{ Math.floor(gameTime / 60) }}分{{ gameTime % 60 }}秒</p>
      <p>击中次数: {{ hitCount }}</p>
      <button class="control-btn" @click="resetGame">重新开始</button>
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
}

.score {
  font-size: 32px;
  margin: 20px 0;
  color: #1565c0;
  text-shadow: 2px 2px 4px rgba(0,0,0,0.1);
}

.game-area {
  width: 100vw;
  height: calc(100vh - 80px);
  position: relative;
  background: transparent;
  overflow: hidden;
}

.snowman {
  position: absolute;
  top: 20px;
  width: 100px;
  height: 120px;
  transform: translateX(-50%);
  background: url('@/assets/snowman.svg') no-repeat center;
  background-size: contain;
  transition: background-image 0.3s;
}

.snowman--damaged {
  background-image: url('@/assets/damaged-snowman.svg');
}

.snowman--broken {
  background-image: url('@/assets/broken-snowman.svg');
}

.snowball {
  position: absolute;
  width: 30px;
  height: 30px;
  background: white;
  border-radius: 50%;
  transform: translateX(-50%);
  box-shadow: 0 0 10px rgba(0,0,0,0.1);
}

.particle {
  position: absolute;
  width: 10px;
  height: 10px;
  background: white;
  border-radius: 50%;
  transform: translateX(-50%);
  opacity: 0.8;
  animation: fade-out 0.5s linear forwards;
}

@keyframes fade-out {
  to {
    opacity: 0;
    transform: translateX(-50%) scale(0.5);
  }
}

.launcher {
  position: absolute;
  bottom: 40px;
  width: 120px;
  height: 60px;
  background: url('@/assets/cannon.svg') no-repeat center;
  background-size: contain;
  transform: translateX(-50%);
  cursor: pointer;
  transition: transform 0.2s, left 0.1s linear;
}

.launcher:hover {
  transform: translateX(-50%) scale(1.05);
}

.game-over {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  padding: 40px;
  background: rgba(255, 255, 255, 0.95);
  border-radius: 16px;
  text-align: center;
  box-shadow: 0 4px 20px rgba(0,0,0,0.1);
}

.game-over h2 {
  color: #1565c0;
  margin-bottom: 20px;
}

.game-over p {
  font-size: 18px;
  margin: 10px 0;
  color: #333;
}
</style>
