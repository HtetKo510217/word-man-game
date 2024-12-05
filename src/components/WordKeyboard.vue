<template>
  <div class="keyboard-container">
    <div v-for="(key, index) in keys" :key="index" class="key-row">
      <button
        v-for="(letter, letterIndex) in key"
        :key="letter"
        @click="handleKeyPress(letter)"
        :disabled="usedLetters.includes(letter)"
        :style="{ backgroundColor: getKeyColor(index, letterIndex) }"
        class="key-button"
      >
        {{ letter }}
      </button>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue'
const keys = ref(['QWERTYUIOP', 'ASDFGHJKL', 'ZXCVBNM'])
const emit = defineEmits(['key-pressed'])
const { usedLetters } = defineProps<{ usedLetters: string[] }>()
const handleKeyPress = (key: string) => {
  emit('key-pressed', key)
}

function getKeyColor(rowIndex: number, columnIndex: number) {
  const colors = ['#3498db', '#f1c40f', '#2ecc71', '#e74c3c', '#9b59b6', '#e67e22']
  return colors[(rowIndex + columnIndex) % colors.length]
}
</script>

<style scoped>
.keyboard-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 8px;
  width: 100%;
  max-width: 600px;
  margin: 0 auto;
  padding: 10px;
  box-sizing: border-box;
}

.key-row {
  display: flex;
  justify-content: center;
  gap: 6px;
}

.key-button {
  width: 45px;
  height: 45px;
  font-size: 18px;
  font-weight: 600;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  transition: all 0.2s ease;
}

.key-button:hover:not(:disabled) {
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
}

.key-button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
  background-color: #e0e0e0;
}

@media (max-width: 480px) {
  .keyboard-container {
    padding: 0px;
  }

  .key-button {
    width: 32px;
    height: 32px;
    font-size: 14px;
    border-radius: 6px;
  }

  .key-row {
    gap: 4px;
  }
}
</style>
