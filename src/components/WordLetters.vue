<template>
  <div class="word-display">
    <div
      v-for="(letter, index) in word"
      :key="index"
      class="letter"
      :class="{ empty: guessedLetters[index] === '_' }"
    >
      {{ guessedLetters[index] === '_' ? '' : guessedLetters[index] }}
    </div>
  </div>
</template>

<script setup lang="ts">
import { defineProps } from 'vue'

defineProps<{
  word: string
  guessedLetters: string[]
}>()
</script>

<style scoped>
.word-display {
  display: flex;
  justify-content: center;
  gap: 12px;
  margin: 30px 0;
  flex-wrap: wrap;
  padding: 0 20px;
}

.letter {
  width: 45px;
  height: 45px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 24px;
  font-weight: bold;
  border-radius: 8px;
  background: var(--background-color);
  border: 2px solid var(--border-color);
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  text-transform: uppercase;
  color: var(--text-color);
}

.letter.empty {
  border: 2px dashed var(--border-color);
  background: rgba(255, 255, 255, 0.05);
}

@media (max-width: 600px) {
  .word-display {
    gap: 8px;
  }

  .letter {
    width: 35px;
    height: 35px;
    font-size: 18px;
  }
}

.letter:not(.empty) {
  animation: revealLetter 0.3s ease;
}

@keyframes revealLetter {
  from {
    transform: scale(0.8);
    opacity: 0;
  }
  to {
    transform: scale(1);
    opacity: 1;
  }
}
</style>
