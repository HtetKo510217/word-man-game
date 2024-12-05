<template>
  <div class="word-container" :class="[currentTheme, `category-${category}`]">
    <div v-if="isLoading" class="loading-overlay">
      <div class="spinner"></div>
    </div>
    <div class="game-header">
      <button class="help-button" @click="showInstructions = true">
        <i class="fas fa-question-circle"></i>
      </button>
      <h1 class="title">
        <span class="gradient-text">Word Guessing Game</span>
      </h1>

      <!-- Theme Switcher -->
      <div class="theme-switcher">
        <button
          v-for="theme in themes"
          :key="theme"
          @click="changeTheme(theme)"
          :class="{ active: currentTheme === theme }"
          class="theme-button"
        >
          <i :class="getThemeIcon(theme)"></i>
          {{ theme }}
        </button>
      </div>

      <div class="category-display">
        <span class="category-label">Current Category:</span>
        <span class="category-value">{{ formatCategory(category) }}</span>
      </div>

      <div class="game-stats">
        <div class="stat-card timer" :class="{ 'low-time': timeLeft <= 30 }">
          <i class="fas fa-clock"></i>
          {{ formatTime(timeLeft) }}
        </div>

        <div class="stat-card attempts" :class="{ 'low-attempts': attempts <= 3 }">
          <div class="progress-bar">
            <div :style="{ width: `${(attempts / maxAttempts) * 100}%` }" class="progress"></div>
          </div>
          <span>Attempts: {{ attempts }}</span>
        </div>

        <div class="stat-card hints">
          <button @click="useHint" :disabled="hintsRemaining === 0" class="hint-button">
            <i class="fas fa-lightbulb"></i>
            Hints: {{ hintsRemaining }}
          </button>
        </div>
      </div>

      <button class="sound-toggle" @click="toggleSound">
        <i :class="soundEnabled ? 'fas fa-volume-up' : 'fas fa-volume-mute'"></i>
      </button>
    </div>

    <HangmanDrawing :mistakes="maxAttempts - attempts" :is-game-over="attempts === 0" />

    <div class="word-display">
      <WordLetters :guessed-letters="guessedLetters" :word="word" :shake="shakeWord" />
    </div>

    <WordKeyboard
      @key-pressed="guessWord"
      :used-letters="usedLetters"
      :correct-letters="correctLetters"
      :wrong-letters="wrongLetters"
    />

    <div class="controls">
      <select v-model="category" class="category-select">
        <option v-for="cat in Object.keys(json.categories)" :key="cat" :value="cat">
          {{ formatCategory(cat) }}
        </option>
      </select>
      <select v-model="difficulty" class="difficulty-select">
        <option value="easy">Easy</option>
        <option value="medium">Medium</option>
        <option value="hard">Hard</option>
      </select>
      <button type="button" class="reset-button" @click="reset">
        <i class="fas fa-redo"></i> New Game
      </button>
    </div>

    <!-- High Scores -->
    <div class="high-scores">
      <h3>Top Scores</h3>
      <div v-for="(score, index) in topScores" :key="index" class="score-item">
        <span class="rank">{{ index + 1 }}</span>
        <span class="score-details"> {{ score.name }} - {{ formatTime(score.time) }} </span>
      </div>
    </div>

    <!-- Game Over Modal -->
    <Transition name="modal">
      <div v-if="showModal" class="modal">
        <div class="modal-content" :class="{ win: isWin, lose: !isWin }">
          <h2>{{ isWin ? 'Congratulations! 🎉' : 'Game Over! 😢' }}</h2>
          <p>{{ isWin ? 'You guessed it!' : `The word was: ${word}` }}</p>
          <div class="stats">
            <p>Time taken: {{ formatTime(difficultySettings[difficulty].time - timeLeft) }}</p>
            <p>Attempts left: {{ attempts }}</p>
          </div>

          <div v-if="isWin" class="score-input">
            <input v-model="playerName" placeholder="Enter your name" maxlength="20" />
            <button @click="saveScore" class="save-score-btn">Save Score</button>
          </div>

          <button @click="closeModal" class="modal-button">Play Again</button>
        </div>
      </div>
    </Transition>

    <!-- Notification Toast -->
    <Transition name="toast">
      <div v-if="showToast" class="toast" :class="toastType">
        {{ toastMessage }}
      </div>
    </Transition>

    <!-- Add instructions modal -->
    <Transition name="modal">
      <div v-if="showInstructions" class="modal">
        <div class="modal-content instructions">
          <h2>How to Play</h2>
          <ul>
            <li>Guess the word by selecting letters</li>
            <li>You have limited attempts based on difficulty</li>
            <li>Use hints wisely - they reveal random letters</li>
            <li>Beat the timer to win!</li>
          </ul>
          <div class="difficulty-info">
            <h3>Difficulty Levels:</h3>
            <p>Easy: 8 attempts, 3 minutes</p>
            <p>Medium: 6 attempts, 2 minutes</p>
            <p>Hard: 5 attempts, 1.5 minutes</p>
          </div>
          <button @click="showInstructions = false" class="modal-button">Got it!</button>
        </div>
      </div>
    </Transition>

    <div class="player-stats">
      <div class="player-avatar" :class="{ active: isPlayerTurn }">
        <div class="avatar">
          <i class="fas fa-user"></i>
        </div>
        <div class="score">{{ playerScore }} pts</div>
      </div>
      <div class="player-avatar" :class="{ active: !isPlayerTurn }">
        <div class="avatar">
          <i class="fas fa-robot"></i>
        </div>
        <div class="score">{{ computerScore }} pts</div>
      </div>
    </div>

    <div class="puzzle-counter">PUZZLE {{ currentPuzzle }} OF {{ totalPuzzles }}</div>
  </div>
</template>

<script setup lang="ts">
import WordKeyboard from '../components/WordKeyboard.vue'
import WordLetters from '../components/WordLetters.vue'
import HangmanDrawing from '../components/HangmanDrawing.vue'
import json from '../../words.json'
import { ref, onMounted, watch } from 'vue'

type DifficultyLevel = 'easy' | 'medium' | 'hard'

// Game state
const word = ref('')
const guessedLetters = ref<string[]>([])
const attempts = ref(7)
const isLoading = ref(false)
const winStreak = ref(0)
const usedLetters = ref<string[]>([])
const difficulty = ref<DifficultyLevel>('medium')
const timeLeft = ref(120)
const timer = ref<ReturnType<typeof setInterval> | null>(null)
const showModal = ref(false)
const isWin = ref(false)
const category = ref('countries')
const maxAttempts = ref(7)
const currentPuzzle = ref(1)
const totalPuzzles = ref(5)
const playerScore = ref(0)
const computerScore = ref(0)
const isPlayerTurn = ref(true)

// Game settings based on difficulty
const difficultySettings = {
  easy: { attempts: 8, time: 240 },
  medium: { attempts: 6, time: 180 },
  hard: { attempts: 4, time: 120 },
} as const

// Sound effects
const soundEnabled = ref(localStorage.getItem('wordGameSound') !== 'false')
const sounds = {
  correct: new Audio('/sounds/correct.mp3'),
  wrong: new Audio('/sounds/wrong.mp3'),
  win: new Audio('/sounds/win.mp3'),
  lose: new Audio('/sounds/lose.mp3'),
  hint: new Audio('/sounds/hint.mp3'),
  click: new Audio('/sounds/click.mp3'),
}

// Update sound toggle function
const toggleSound = () => {
  soundEnabled.value = !soundEnabled.value
  localStorage.setItem('wordGameSound', String(soundEnabled.value))
  // Play click sound when toggling
  if (soundEnabled.value) {
    sounds.click.play().catch(() => {})
  }
}

// Update playSound function
const playSound = (type: keyof typeof sounds) => {
  if (!soundEnabled.value) return
  sounds[type].play().catch(() => {})
}

// New state variables
const currentTheme = ref('classic')
const themes = ['classic', 'dark', 'colorful']
const hintsRemaining = ref(3)
const shakeWord = ref(false)
const correctLetters = ref<string[]>([])
const wrongLetters = ref<string[]>([])
const playerName = ref('')
const topScores = ref<Array<{ name: string; time: number }>>([])
const showToast = ref(false)
const toastMessage = ref('')
const toastType = ref('')
const showInstructions = ref(false)

// Load scores from localStorage
const loadScores = () => {
  const saved = localStorage.getItem('wordGameScores')
  if (saved) {
    topScores.value = JSON.parse(saved)
  }
}

// Save score
const saveScore = () => {
  if (!playerName.value) {
    showToastMessage('Please enter your name', 'error')
    return
  }

  const newScore = {
    name: playerName.value,
    time: difficultySettings[difficulty.value].time - timeLeft.value,
  }

  topScores.value.push(newScore)
  topScores.value.sort((a, b) => a.time - b.time)
  topScores.value = topScores.value.slice(0, 5) // Keep top 5

  localStorage.setItem('wordGameScores', JSON.stringify(topScores.value))
  showToastMessage('Score saved!', 'success')
  closeModal()
}

// Use hint
const useHint = () => {
  if (attempts.value === 0 || !guessedLetters.value.includes('_')) {
    showToastMessage('Game is over!', 'error')
    return
  }

  if (hintsRemaining.value > 0) {
    const unrevealedIndices = word.value
      .split('')
      .map((_, index) => index)
      .filter((index) => guessedLetters.value[index] === '_')

    // Calculate remaining letters needed to win
    const remainingToWin = Math.ceil(word.value.length * 0.3) // At least 30% should be solved by player

    // Don't allow hints if it would reveal too many letters
    if (unrevealedIndices.length <= remainingToWin) {
      showToastMessage('Try to solve the remaining letters yourself!', 'error')
      return
    }

    if (unrevealedIndices.length > 0) {
      const randomIndex = unrevealedIndices[Math.floor(Math.random() * unrevealedIndices.length)]
      const letter = word.value[randomIndex]

      guessedLetters.value[randomIndex] = letter
      usedLetters.value.push(letter)
      correctLetters.value.push(letter)
      hintsRemaining.value--

      playSound('hint')
    }
  }
}

// Show toast message
const showToastMessage = (message: string, type: string) => {
  toastMessage.value = message
  toastType.value = type
  showToast.value = true
  setTimeout(() => {
    showToast.value = false
  }, 3000)
}

// Change theme
const changeTheme = (theme: string) => {
  currentTheme.value = theme
  localStorage.setItem('wordGameTheme', theme)
}

// Keep this function
function getRandomIndices(length: number, count: number) {
  const indices: number[] = []
  while (indices.length < count) {
    const index = Math.floor(Math.random() * length)
    if (!indices.includes(index)) {
      indices.push(index)
    }
  }
  return indices
}

// Update reset function to show initial letters
const reset = async () => {
  isLoading.value = true
  try {
    // Reset scores if it's a new game set
    if (currentPuzzle.value >= totalPuzzles.value) {
      playerScore.value = 0
      computerScore.value = 0
      currentPuzzle.value = 1
    }
    const selectedCategory = json.categories[category.value as keyof typeof json.categories]
    word.value = selectedCategory[Math.floor(Math.random() * selectedCategory.length)].toUpperCase()

    // Show some initial letters (1 letter for short words, 2 for longer words)
    const initialLettersCount = word.value.length <= 4 ? 1 : 2
    const randomIndices = getRandomIndices(word.value.length, initialLettersCount)

    guessedLetters.value = word.value
      .split('')
      .map((letter, index) => (randomIndices.includes(index) ? letter : '_'))

    attempts.value =
      difficultySettings[difficulty.value as keyof typeof difficultySettings].attempts
    maxAttempts.value =
      difficultySettings[difficulty.value as keyof typeof difficultySettings].attempts
    usedLetters.value = randomIndices.map((i) => word.value[i]) // Add revealed letters to used letters
    hintsRemaining.value = calculateHints(word.value.length)
    correctLetters.value = [...usedLetters.value] // Add revealed letters to correct letters
    wrongLetters.value = []
    shakeWord.value = false
    startTimer()
  } finally {
    isLoading.value = false
  }
}

// Update guessWord function
const guessWord = (letter: string) => {
  if (usedLetters.value.includes(letter)) {
    sounds.click.play().catch(() => {})
    return
  }
  // Prevent guessing if game is over
  if (attempts.value === 0 || !guessedLetters.value.includes('_')) return

  usedLetters.value.push(letter)
  if (word.value.includes(letter)) {
    correctLetters.value.push(letter)
    word.value.split('').forEach((char, index) => {
      if (char === letter) {
        guessedLetters.value[index] = letter
      }
    })
    playSound('correct')
  } else {
    wrongLetters.value.push(letter)
    attempts.value--
    playSound('wrong')
    shakeWord.value = true
    setTimeout(() => {
      shakeWord.value = false
    }, 500)
  }

  checkGameStatus()
}

// Update gameOver function
const gameOver = (won: boolean) => {
  if (showModal.value) return

  showModal.value = true
  isWin.value = won
  playSound(won ? 'win' : 'lose')
  if (timer.value) clearInterval(timer.value)

  // Update scores when game ends
  updateScore(won)

  if (won) {
    winStreak.value++
    if (winStreak.value >= 3 && difficulty.value !== 'hard') {
      showToastMessage("You're doing great! Try a harder difficulty?", 'success')
    }
  } else {
    winStreak.value = 0
  }
}

const formatTime = (seconds: number): string => {
  const mins = Math.floor(seconds / 60)
  const secs = seconds % 60
  return `${mins}:${secs.toString().padStart(2, '0')}`
}

const startTimer = () => {
  if (timer.value) clearInterval(timer.value)
  timeLeft.value = difficultySettings[difficulty.value as keyof typeof difficultySettings].time
  timer.value = setInterval(() => {
    timeLeft.value--
    if (timeLeft.value <= 0) {
      clearInterval(timer.value!)
      gameOver(false)
    }
  }, 1000)
}

const closeModal = () => {
  showModal.value = false
  reset()
}

const formatCategory = (cat: string): string => {
  return cat
    .split('_')
    .map((word) => word.charAt(0).toUpperCase() + word.slice(1))
    .join(' ')
}

// Watch for changes
watch([difficulty, category], () => {
  reset()
})

// Initialize game
onMounted(() => {
  loadScores()
  reset()
  window.addEventListener('keydown', handleKeyPress)
})

// Add this helper function
const getThemeIcon = (theme: string): string => {
  switch (theme) {
    case 'classic':
      return 'fas fa-sun'
    case 'dark':
      return 'fas fa-moon'
    case 'colorful':
      return 'fas fa-palette'
    default:
      return 'fas fa-circle'
  }
}

// Add function to calculate hints based on word length
const calculateHints = (wordLength: number): number => {
  if (wordLength <= 4) return 1
  if (wordLength <= 6) return 2
  return 3
}

// Update keyboard click handler
const handleKeyPress = (event: KeyboardEvent) => {
  if (event.key.match(/^[a-zA-Z]$/)) {
    const letter = event.key.toUpperCase()
    sounds.click.play().catch(() => {})
    guessWord(letter)
  }
}

// Check game status
const checkGameStatus = () => {
  if (!guessedLetters.value.includes('_')) {
    gameOver(true)
  } else if (attempts.value === 0) {
    gameOver(false)
  }
}

// Basic scoring system
const updateScore = (won: boolean) => {
  const baseScore = 100
  const attemptBonus = attempts.value * 10
  const score = baseScore + attemptBonus

  if (won) {
    playerScore.value += score
    isPlayerTurn.value = true
  } else {
    computerScore.value += Math.floor(score / 2)
    isPlayerTurn.value = false
  }
}
</script>

<style scoped>
/* Theme Variables */
.word-container {
  --primary-color: #2c3e50;
  --secondary-color: #3498db;
  --background-color: #ffffff;
  --text-color: #2c3e50;
  --border-color: #dee2e6;
  --success-color: #28a745;
  --danger-color: #dc3545;
  --shadow-color: rgba(0, 0, 0, 0.1);
}

/* Dark Theme */
.dark {
  --primary-color: #bb86fc;
  --secondary-color: #03dac6;
  --background-color: #1a1a1a;
  --text-color: #ffffff;
  --border-color: #333333;
  --success-color: #00c853;
  --danger-color: #ff1744;
  --shadow-color: rgba(0, 0, 0, 0.3);
}

/* Colorful Theme */
.colorful {
  --primary-color: #ff6b6b;
  --secondary-color: #4ecdc4;
  --background-color: #f8f9fa;
  --text-color: #2c3e50;
  --border-color: #ff6b6b;
  --success-color: #96ceb4;
  --danger-color: #ff6b6b;
  --shadow-color: rgba(255, 107, 107, 0.2);
}

.word-container {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
  font-family: 'Arial', sans-serif;
  background-color: var(--background-color);
  color: var(--text-color);
  min-height: 100vh;
  transition: all 0.3s ease;
  display: flex;
  flex-direction: column;
  border-radius: 15px;
  box-shadow: 0 0 20px rgba(0, 0, 0, 0.2);
}

/* Theme Switcher */
.theme-switcher {
  display: flex;
  justify-content: center;
  gap: 10px;
  margin-bottom: 20px;
}

.theme-button {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 10px 20px;
  border: 2px solid var(--border-color);
  border-radius: 25px;
  background: transparent;
  color: var(--text-color);
  cursor: pointer;
  transition: all 0.3s ease;
  font-weight: 500;
}

.theme-button i {
  font-size: 1.1em;
}

.theme-button.active {
  background-color: var(--primary-color);
  color: var(--background-color);
}

.theme-button:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 8px var(--shadow-color);
}

/* Game Header */
.game-header {
  text-align: center;
  margin-bottom: 30px;
}

.title {
  color: var(--primary-color);
  margin-bottom: 20px;
  font-size: 2.5em;
  font-weight: bold;
}

.category-display {
  background: rgba(255, 255, 255, 0.1);
  padding: 10px 20px;
  border-radius: 10px;
  backdrop-filter: blur(5px);
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.2);
  margin: 20px auto;
  max-width: 400px;
}

.category-label {
  color: var(--text-color);
  opacity: 0.8;
  margin-right: 8px;
}

.category-value {
  color: var(--primary-color);
  font-weight: bold;
  text-shadow: 0 1px 2px rgba(0, 0, 0, 0.2);
}

/* Game Stats */
.game-stats {
  display: flex;
  justify-content: center;
  gap: 20px;
  margin: 25px 0;
  flex-wrap: wrap;
}

.stat-card {
  background: var(--background-color);
  padding: 20px 25px;
  border-radius: 15px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.2);
  min-width: 120px;
  transition: all 0.3s ease;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 10px;
  border: 1px solid rgba(255, 255, 255, 0.1);
}

/* Timer styles */
.timer {
  font-size: 1.5em;
  font-weight: bold;
  color: var(--text-color);
  text-shadow: 0 1px 2px rgba(0, 0, 0, 0.2);
}

.timer.low-time {
  color: var(--danger-color);
  animation: pulse 1s infinite;
}

.timer i {
  margin-right: 8px;
  color: var(--primary-color);
}

/* Attempts styles */
.attempts {
  text-align: center;
}

.progress-bar {
  width: 100%;
  height: 8px;
  background-color: var(--border-color);
  border-radius: 4px;
  overflow: hidden;
  margin: 8px 0;
}

.progress {
  height: 100%;
  background-color: var(--success-color);
  transition: width 0.3s ease;
  border-radius: 4px;
}

.attempts.low-attempts .progress {
  background-color: var(--danger-color);
}

/* Hints button styles */
.hint-button {
  background: linear-gradient(45deg, var(--secondary-color), var(--primary-color));
  color: white;
  border: none;
  padding: 12px 24px;
  border-radius: 25px;
  cursor: pointer;
  transition: all 0.3s ease;
  font-weight: 500;
  display: flex;
  align-items: center;
  gap: 10px;
  font-size: 1.1em;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
}

.hint-button i {
  font-size: 1.2em;
}

.hint-button:not(:disabled):hover {
  transform: translateY(-2px);
  box-shadow: 0 6px 15px rgba(0, 0, 0, 0.2);
}

.hint-button:disabled {
  background: var(--border-color);
  opacity: 0.7;
  cursor: not-allowed;
}

/* Mobile styles */
@media (max-width: 600px) {
  .game-stats {
    gap: 15px;
    padding: 0 10px;
  }

  .stat-card {
    padding: 15px 20px;
    min-width: calc(50% - 20px);
    font-size: 0.9em;
  }

  .timer {
    font-size: 1.3em;
  }

  .hint-button {
    padding: 10px 20px;
    font-size: 1em;
  }
}

/* Controls */
.controls {
  display: flex;
  justify-content: center;
  gap: 15px;
  margin: 20px 0;
}

.category-select,
.difficulty-select {
  padding: 10px 15px;
  border: 2px solid var(--border-color);
  border-radius: 8px;
  font-size: 1em;
  cursor: pointer;
  background-color: var(--background-color);
  color: var(--text-color);
  transition: all 0.3s ease;
}

.reset-button {
  padding: 10px 20px;
  background-color: var(--secondary-color);
  color: var(--background-color);
  border: none;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.3s ease;
}

.reset-button:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 8px var(--shadow-color);
}

/* High Scores */
.high-scores {
  margin-top: 30px;
  padding: 20px;
  border-radius: 8px;
  background-color: var(--background-color);
  box-shadow: 0 2px 4px var(--shadow-color);
}

.score-item {
  display: flex;
  align-items: center;
  padding: 10px;
  border-bottom: 1px solid var(--border-color);
}

.rank {
  width: 30px;
  height: 30px;
  display: flex;
  align-items: center;
  justify-content: center;
  background-color: var(--primary-color);
  color: var(--background-color);
  border-radius: 50%;
  margin-right: 10px;
}

/* Modal */
.modal {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.85);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
}

.modal-content {
  background-color: var(--background-color);
  padding: 40px;
  border-radius: 20px;
  text-align: center;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.3);
  max-width: 450px;
  width: 90%;
  animation: modalPop 0.3s ease;
  backdrop-filter: blur(8px);
}

.modal-content h2 {
  font-size: 2.2em;
  margin-bottom: 20px;
  color: var(--primary-color);
}

.modal-content.win {
  border: 3px solid var(--success-color);
  background: linear-gradient(to bottom right, var(--background-color), rgba(40, 167, 69, 0.15));
}

.modal-content.lose {
  border: 3px solid var(--danger-color);
  background: linear-gradient(to bottom right, var(--background-color), rgba(220, 53, 69, 0.15));
}

.score-input {
  margin: 20px 0;
}

.score-input input {
  padding: 12px 20px;
  border: 2px solid var(--border-color);
  border-radius: 25px;
  font-size: 1em;
  width: 80%;
  margin-bottom: 10px;
  background: var(--background-color);
  color: var(--text-color);
}

.save-score-btn,
.modal-button {
  padding: 12px 30px;
  border: none;
  border-radius: 25px;
  font-size: 1.1em;
  cursor: pointer;
  transition: all 0.3s ease;
  background: var(--primary-color);
  color: var(--background-color);
  margin: 5px;
}

.save-score-btn:hover,
.modal-button:hover {
  transform: translateY(-2px);
  box-shadow: 0 5px 15px var(--shadow-color);
}

/* Toast */
.toast {
  position: fixed;
  bottom: 20px;
  right: 20px;
  padding: 15px 25px;
  border-radius: 8px;
  color: white;
  z-index: 1000;
  animation: slideIn 0.3s ease;
}

.toast.success {
  background-color: var(--success-color);
}

.toast.error {
  background-color: var(--danger-color);
}

/* Animations */
@keyframes pulse {
  0% {
    opacity: 1;
  }
  50% {
    opacity: 0.5;
  }
  100% {
    opacity: 1;
  }
}

@keyframes slideIn {
  from {
    transform: translateX(100%);
  }
  to {
    transform: translateX(0);
  }
}

/* Transitions */
.modal-enter-active,
.modal-leave-active {
  transition: opacity 0.3s ease;
}

.modal-enter-from,
.modal-leave-to {
  opacity: 0;
}

.toast-enter-active,
.toast-leave-active {
  transition: all 0.3s ease;
}

.toast-enter-from,
.toast-leave-to {
  transform: translateX(100%);
  opacity: 0;
}

/* Responsive Design */
@media (max-width: 600px) {
  .word-container {
    padding: 10px;
    min-height: 100vh;
    width: 100%;
    margin: 0;
  }

  .title {
    font-size: 1.8em;
    margin-bottom: 15px;
  }

  .theme-switcher {
    flex-wrap: wrap;
    gap: 8px;
  }

  .theme-button {
    padding: 8px 15px;
    font-size: 0.9em;
  }

  .category-display {
    padding: 10px 15px;
    font-size: 0.9em;
    margin: 15px auto;
  }

  .game-stats {
    flex-direction: row;
    flex-wrap: wrap;
    justify-content: center;
    gap: 10px;
    margin: 15px 0;
  }

  .stat-card {
    padding: 10px 15px;
    min-width: auto;
    width: calc(50% - 10px);
    font-size: 0.9em;
  }

  .keyboard-container {
    padding: 5px;
    max-width: 100%;
    margin: 15px auto;
    background-color: var(--background-color);
  }

  .keyboard-container button {
    width: 32px;
    height: 32px;
    margin: 2px;
    font-size: 14px;
    padding: 0;
    border-radius: 6px;
  }

  .controls {
    flex-direction: column;
    align-items: stretch;
    gap: 10px;
    margin: 15px 0;
    padding: 0 10px;
  }

  .category-select,
  .difficulty-select,
  .reset-button {
    width: 100%;
    padding: 12px;
    font-size: 1em;
    border-radius: 12px;
  }

  .high-scores {
    margin: 15px 10px;
    padding: 15px;
    border-radius: 12px;
  }

  .score-item {
    padding: 8px;
    font-size: 0.9em;
  }

  .modal-content {
    padding: 20px;
    width: 90%;
    margin: 0 10px;
  }

  .modal-content h2 {
    font-size: 1.5em;
  }

  .score-input input {
    width: 90%;
    padding: 10px 15px;
  }

  .save-score-btn,
  .modal-button {
    width: 90%;
    padding: 10px 20px;
    margin: 5px auto;
  }

  .progress-bar {
    width: 100%;
    height: 8px;
  }

  .hint-button {
    padding: 8px 15px;
    font-size: 0.9em;
  }

  .toast {
    left: 10px;
    right: 10px;
    bottom: 10px;
    text-align: center;
  }
}

/* Add modal pop animation */
@keyframes modalPop {
  0% {
    transform: scale(0.8);
    opacity: 0;
  }
  100% {
    transform: scale(1);
    opacity: 1;
  }
}

.gradient-text {
  background: linear-gradient(45deg, var(--primary-color), var(--secondary-color));
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

.stat-card {
  background: var(--background-color);
  padding: 15px 25px;
  border-radius: 15px;
  box-shadow: 0 4px 8px var(--shadow-color);
  min-width: 150px;
  transition: all 0.3s ease;
}

.stat-card:hover {
  transform: translateY(-2px);
  box-shadow: 0 6px 12px var(--shadow-color);
}

.hint-button {
  position: relative;
  background: var(--secondary-color);
  color: var(--background-color);
  border: none;
  padding: 10px 20px;
  border-radius: 25px;
  cursor: pointer;
  transition: all 0.3s ease;
  font-weight: 500;
  display: flex;
  align-items: center;
  gap: 8px;
}

.hint-button:disabled {
  opacity: 0.6;
  cursor: not-allowed;
  background: var(--border-color);
}

.hint-button:disabled::after {
  content: '(Used)';
  position: absolute;
  bottom: -20px;
  left: 50%;
  transform: translateX(-50%);
  font-size: 0.8em;
  color: var(--text-color);
  white-space: nowrap;
}

.hint-button:not(:disabled):hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 8px var(--shadow-color);
}

/* Update keyboard styles */
.keyboard-container {
  width: 100%;
  max-width: 600px;
  margin: 20px auto;
  padding: 15px;
  background-color: var(--background-color);
  display: flex;
  flex-direction: column;
  gap: 8px;
  align-items: center;
}

.keyboard-row {
  display: flex;
  gap: 6px;
  justify-content: center;
  flex-wrap: wrap;
  width: 100%;
  max-width: 500px;
}

.keyboard-container button {
  width: 36px;
  height: 36px;
  border-radius: 8px;
  border: none;
  font-size: 16px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s ease;
  background: linear-gradient(145deg, rgba(255, 255, 255, 0.1), rgba(255, 255, 255, 0.05));
  border: 1px solid rgba(255, 255, 255, 0.1);
  color: var(--text-color);
  font-weight: 600;
  text-shadow: 0 1px 2px rgba(0, 0, 0, 0.2);
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
  margin: 2px;
}

.keyboard-container button:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 8px var(--shadow-color);
}

.keyboard-container button.used {
  opacity: 0.5;
  cursor: not-allowed;
}

.keyboard-container button.correct {
  background: linear-gradient(145deg, #4caf50, #388e3c);
  border-color: #2e7d32;
  box-shadow: 0 2px 4px rgba(76, 175, 80, 0.3);
  color: white;
}

.keyboard-container button.wrong {
  background: linear-gradient(145deg, #f44336, #d32f2f);
  border-color: #c62828;
  box-shadow: 0 2px 4px rgba(244, 67, 54, 0.3);
  color: white;
}

/* Update mobile keyboard styles */
@media (max-width: 600px) {
  .keyboard-container {
    padding: 10px 5px;
    gap: 4px;
    margin: 10px auto;
  }

  .keyboard-row {
    gap: 3px;
    padding: 0 5px;
  }

  .keyboard-container button {
    width: 30px;
    height: 30px;
    font-size: 14px;
    border-radius: 6px;
    margin: 1px;
  }
}

/* Add specific styles for smaller screens */
@media (max-width: 350px) {
  .keyboard-row {
    gap: 2px;
  }

  .keyboard-container button {
    width: 28px;
    height: 28px;
    font-size: 12px;
  }
}

/* Add success/failure animations */
@keyframes success {
  0% {
    transform: scale(1);
  }
  50% {
    transform: scale(1.2);
  }
  100% {
    transform: scale(1);
  }
}

@keyframes shake {
  0%,
  100% {
    transform: translateX(0);
  }
  25% {
    transform: translateX(-5px);
  }
  75% {
    transform: translateX(5px);
  }
}

.letter-correct {
  animation: success 0.5s ease;
  color: var(--success-color);
}

.letter-wrong {
  animation: shake 0.5s ease;
  color: var(--danger-color);
}

/* Improve keyboard feedback */
.keyboard-container button:active {
  transform: scale(0.95);
}

.difficulty-select option {
  position: relative;
}

.difficulty-select option:hover::after {
  content: attr(data-info);
  position: absolute;
  bottom: 100%;
  left: 50%;
  transform: translateX(-50%);
  background: var(--background-color);
  padding: 8px;
  border-radius: 4px;
  box-shadow: 0 2px 4px var(--shadow-color);
  font-size: 0.9em;
  white-space: nowrap;
}

.loading-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
}

.spinner {
  width: 50px;
  height: 50px;
  border: 5px solid var(--background-color);
  border-top-color: var(--primary-color);
  border-radius: 50%;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}

.win-animation {
  animation: float 2s ease-in-out infinite;
}

@keyframes float {
  0%,
  100% {
    transform: translateY(0);
  }
  50% {
    transform: translateY(-20px);
  }
}

.balloon-animation {
  animation: sway 3s ease-in-out infinite;
}

@keyframes sway {
  0%,
  100% {
    transform: rotate(-5deg);
  }
  50% {
    transform: rotate(5deg);
  }
}

.player-stats {
  display: flex;
  justify-content: center;
  gap: 30px;
  margin: 20px 0;
}

.player-avatar {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 15px 25px;
  border-radius: 12px;
  background: white;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  opacity: 0.8;
  transition: all 0.3s ease;
}

.player-avatar.active {
  opacity: 1;
  transform: scale(1.05);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
}

.avatar {
  width: 40px;
  height: 40px;
  background: #f0f0f0;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  margin-bottom: 8px;
  font-size: 20px;
  color: #333;
}

.score {
  font-weight: bold;
  color: #333;
}

@media (max-width: 600px) {
  .player-avatar {
    padding: 10px 20px;
  }

  .avatar {
    width: 32px;
    height: 32px;
    font-size: 16px;
  }
}

.puzzle-counter {
  text-align: center;
  font-size: 1.2em;
  font-weight: bold;
  margin: 15px 0;
  color: var(--text-color);
  text-transform: uppercase;
  letter-spacing: 1px;
}

@media (max-width: 600px) {
  .player-avatar {
    padding: 10px;
  }

  .player-avatar img {
    width: 30px;
    height: 30px;
  }

  .puzzle-counter {
    font-size: 1em;
  }
}

/* Improve dark mode visibility */
.word-container {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
  font-family: 'Arial', sans-serif;
  background-color: var(--background-color);
  color: var(--text-color);
  min-height: 100vh;
  transition: all 0.3s ease;
  display: flex;
  flex-direction: column;
  border-radius: 15px;
  box-shadow: 0 0 20px rgba(0, 0, 0, 0.2);
}

/* Improve title visibility in dark mode */
.title .gradient-text {
  background: linear-gradient(45deg, #60a5fa, #8b5cf6);
  -webkit-background-clip: text;
  background-clip: text;
  color: transparent;
  text-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

/* Improve category display */
.category-display {
  background: rgba(255, 255, 255, 0.1);
  padding: 10px 20px;
  border-radius: 10px;
  backdrop-filter: blur(5px);
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.2);
  margin: 20px auto;
  max-width: 400px;
}

.category-value {
  color: var(--primary-color);
  font-weight: bold;
  text-shadow: 0 1px 2px rgba(0, 0, 0, 0.2);
}

/* Improve game stats visibility */
.stat-card {
  background: var(--background-color);
  padding: 20px 25px;
  border-radius: 15px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.2);
  min-width: 120px;
  transition: all 0.3s ease;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 10px;
  border: 1px solid rgba(255, 255, 255, 0.1);
}

/* Improve timer visibility */
.timer {
  color: var(--text-color);
  font-weight: bold;
  text-shadow: 0 1px 2px rgba(0, 0, 0, 0.2);
}

/* Improve keyboard visibility */
.keyboard-container button {
  background: linear-gradient(145deg, rgba(255, 255, 255, 0.1), rgba(255, 255, 255, 0.05));
  border: 1px solid rgba(255, 255, 255, 0.1);
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
  color: var(--text-color);
  font-weight: 600;
  text-shadow: 0 1px 2px rgba(0, 0, 0, 0.2);
}

.keyboard-container button.correct {
  background: linear-gradient(145deg, #4caf50, #388e3c);
  border-color: #2e7d32;
  box-shadow: 0 2px 4px rgba(76, 175, 80, 0.3);
}

.keyboard-container button.wrong {
  background: linear-gradient(145deg, #f44336, #d32f2f);
  border-color: #c62828;
  box-shadow: 0 2px 4px rgba(244, 67, 54, 0.3);
}

/* Improve controls visibility */
.controls select,
.controls button {
  background: rgba(255, 255, 255, 0.1);
  border: 1px solid rgba(255, 255, 255, 0.2);
  color: var(--text-color);
  backdrop-filter: blur(5px);
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.2);
}

/* Improve puzzle counter visibility */
.puzzle-counter {
  text-align: center;
  font-size: 1.2em;
  font-weight: bold;
  margin: 15px 0;
  color: var(--primary-color);
  text-transform: uppercase;
  letter-spacing: 1px;
  text-shadow: 0 1px 2px rgba(0, 0, 0, 0.2);
}

/* Improve theme buttons */
.theme-button {
  background: rgba(255, 255, 255, 0.1);
  border: 1px solid rgba(255, 255, 255, 0.2);
  backdrop-filter: blur(5px);
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.2);
  color: var(--text-color);
}

.theme-button.active {
  background: var(--primary-color);
  border-color: transparent;
  box-shadow: 0 2px 8px rgba(var(--primary-color-rgb), 0.3);
}
</style>
