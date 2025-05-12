<template>
  <div class="trainer" :class="{ dark: isDark }">
    <div class="header">
      <h1 class="title">memorify</h1>
      <p class="subtitle">forget forgetting. memorize playfully</p>
      <button class="theme-toggle" @click="toggleTheme">{{ isDark ? '☀️' : '🌙' }}</button>
    </div>

    <div class="card">
      <div v-if="!training" class="input-area">
        <textarea
          v-model="originalText"
          placeholder="Drop a poem, a paragraph, a prophecy..."
          :class="{ dark: isDark }"
        />
        <button class="btn" @click="startTraining" :disabled="originalText.trim() === ''">
          <span class="btn-label">Start</span>
        </button>
      </div>

      <div v-else>
        <div v-if="isCompleted" class="complete-screen animated">
          <h2>🎉 You mastered this text!</h2>
          <button class="btn" @click="restart"><span class="btn-label">Restart</span></button>
        </div>

        <div v-else class="text-display">
          <div class="progress-container">
            <div class="progress-bar" :style="{ width: progressPercent + '%' }"></div>
          </div>

          <div
            class="text-output"
            :class="{ dark: isDark }"
            style="font-size: 1.25rem; white-space: pre-wrap"
          >
            <span v-for="(token, index) in tokens" :key="index">
              <template v-if="token === '\n'">
                <br />
              </template>
              <template v-else-if="missingIndices.has(index)">
                <div style="display: inline-block">
                  <span class="word input-word">
                    <input
                      v-model="userInput[index]"
                      :class="[
                        isDark ? 'dark' : '',
                        checked && isCorrect(index) ? 'correct' : '',
                        checked && isWrong(index) ? 'wrong' : '',
                      ]"
                      :ref="setInputRef(index)"
                      @input="handleAutoAdvance(index)"
                      @blur="handleAttempt(index)"
                      autocomplete="off"
                    />
                    <template v-if="checked && errorCount[index] >= 3 && !isCorrect(index)">
                      <div class="hint">
                        hint: starts with <strong>{{ tokens[index][0] }}</strong>
                      </div>
                    </template>
                  </span>
                </div>
              </template>

              <template v-else>
                <span>{{ token }}</span>
              </template>
            </span>
          </div>

          <button class="btn" @click="checkAnswers"><span class="btn-label">Check</span></button>
        </div>
      </div>
      <div class="footer">
        <p class="footer-text">made by <a href="github.com/byret">byret</a></p>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, nextTick, onMounted, watch } from 'vue'
import confetti from 'canvas-confetti'

const originalText = ref('')
const training = ref(false)
const round = ref(1)

const tokens = ref<string[]>([])
const userInput = ref<Record<number, string>>({})
const missingIndices = ref<Set<number>>(new Set())
const wordIndices = ref<number[]>([])
const inputRefs = ref<HTMLElement[]>([])
const isDark = ref(false)
const checked = ref(false)
const errorCount = ref<Record<number, number>>({})

const handleAttempt = (index: number) => {
  const value = normalize(userInput.value[index] || '')
  const expected = normalize(tokens.value[index])

  if (value === '') return
  if (value !== expected) {
    errorCount.value[index] = (errorCount.value[index] || 0) + 1
  } else {
    errorCount.value[index] = 0
  }
}

let successSound: HTMLAudioElement
onMounted(() => {
  successSound = new Audio('/success.mp3')
})

const maxRounds = computed(() => Math.ceil(wordIndices.value.length * 0.6))
const isCompleted = computed(() => round.value > maxRounds.value)
const progressPercent = computed(() => Math.min(100, (round.value / maxRounds.value) * 100))

watch(isCompleted, (newVal) => {
  if (newVal) triggerCelebration()
})

watch(isDark, (enabled) => {
  document.body.classList.toggle('dark', enabled)
})

const toggleTheme = () => (isDark.value = !isDark.value)

const restart = () => {
  training.value = false
  round.value = 1
  tokens.value = []
  userInput.value = {}
  missingIndices.value = new Set()
  wordIndices.value = []
}

const setInputRef = (index: number) => (el: HTMLElement | null) => {
  if (el) inputRefs.value[index] = el
}

const handleAutoAdvance = (index: number) => {
  if (!isCorrect(index)) return

  const missingList = Array.from(missingIndices.value).sort((a, b) => a - b)
  const currentPos = missingList.indexOf(index)
  const nextIndex = missingList[currentPos + 1]

  if (nextIndex !== undefined) {
    nextTick(() => inputRefs.value[nextIndex]?.focus())
  } else if (missingList.every((i) => isCorrect(i))) {
    checkAnswers()
  }
}

const startTraining = () => {
  training.value = true
  tokenizeText()
  generateMissingWords()
  nextTick(() => {
    const first = Array.from(missingIndices.value).sort((a, b) => a - b)[0]
    inputRefs.value[first]?.focus()
  })
}

const tokenizeText = () => {
  const rawLines = originalText.value.split(/(\n)/)
  const regex = /[\p{L}]+|\d+|[^\p{L}\d\s]|[\s]/gu // <== добавили \s как отдельный токен
  tokens.value = rawLines.flatMap((segment) =>
    segment === '\n' ? ['\n'] : Array.from(segment.matchAll(regex), (m) => m[0]),
  )
  wordIndices.value = tokens.value
    .map((token, index) => ({ token, index }))
    .filter(({ token }) => /^[\p{L}]+$/u.test(token))
    .map(({ index }) => index)
}

const generateMissingWords = () => {
  userInput.value = {}
  missingIndices.value.clear()
  inputRefs.value = []

  const count = Math.min(round.value, wordIndices.value.length)
  const shuffled = [...wordIndices.value].sort(() => Math.random() - 0.5)
  for (let i = 0; i < count; i++) {
    const index = shuffled[i]
    missingIndices.value.add(index)
    userInput.value[index] = ''
  }
}

const normalize = (s: string) =>
  s
    .trim()
    .toLowerCase()
    .replace(/^[^\p{L}\d]+|[^\p{L}\d]+$/gu, '')

const isCorrect = (index: number) =>
  normalize(userInput.value[index] || '') === normalize(tokens.value[index])

const isWrong = (index: number) => userInput.value[index] && !isCorrect(index)

const triggerConfetti = () => {
  confetti({ particleCount: 80, spread: 100, origin: { y: 0.6 }, scalar: 1.2 })
}

const triggerCelebration = () => {
  if (successSound) {
    successSound.currentTime = 0
    successSound.play().catch(() => {})
  }
  const interval = setInterval(triggerConfetti, 300)
  setTimeout(() => clearInterval(interval), 1000)
}

const checkAnswers = () => {
  checked.value = true
  const all = Array.from(missingIndices.value).every((i) => isCorrect(i))
  if (all) {
    triggerConfetti()
    successSound.play().catch(() => {})
    round.value += 1
    generateMissingWords()
    checked.value = false
    nextTick(() => {
      const first = Array.from(missingIndices.value).sort((a, b) => a - b)[0]
      inputRefs.value[first]?.focus()
    })
  }
}
</script>
