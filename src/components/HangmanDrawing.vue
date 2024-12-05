<template>
  <div class="hangman-container">
    <svg height="250" width="200" class="hangman-svg">
      <!-- Background elements for fun -->
      <circle cx="180" cy="30" r="15" class="cloud" />
      <circle cx="160" cy="30" r="20" class="cloud" />
      <circle cx="140" cy="30" r="15" class="cloud" />

      <!-- Platform with shadow -->
      <path
        d="M10 230 L110 230 Q115 230 115 225 L115 228 Q115 233 110 233 L10 233 Z"
        class="platform"
      />

      <!-- Pole with wood texture -->
      <path d="M57 20 L63 20 L63 230 L57 230 Z" class="pole">
        <pattern id="woodPattern" patternUnits="userSpaceOnUse" width="10" height="10">
          <path
            d="M0 0 L10 0 M-2 2 L12 2 M-4 4 L14 4"
            stroke="rgba(139, 69, 19, 0.3)"
            stroke-width="1"
            fill="none"
          />
        </pattern>
      </path>

      <!-- Top beam with rope -->
      <path d="M60 17 L140 17 Q143 17 143 20 L143 23 Q143 20 140 20 L60 20 Z" class="top" />

      <!-- Animated rope -->
      <path d="M140 20 Q142 35 140 50" class="rope" :class="{ swing: !isGameOver }" />

      <!-- Head with expressions -->
      <g v-if="mistakes >= 1" class="hangman-part head">
        <circle cx="140" cy="70" r="20" fill="none" />
        <g v-if="isGameOver">
          <!-- Dead expression -->
          <line x1="130" y1="65" x2="135" y2="70" />
          <line x1="135" y1="65" x2="130" y2="70" />
          <line x1="145" y1="65" x2="150" y2="70" />
          <line x1="150" y1="65" x2="145" y2="70" />
          <path d="M130 80 Q140 75 150 80" fill="none" />
        </g>
        <g v-else>
          <!-- Animated eyes -->
          <g class="eyes">
            <circle cx="132" cy="65" r="3" class="eye" />
            <circle cx="148" cy="65" r="3" class="eye" />
            <circle cx="133" cy="65" r="1" fill="white" class="eye-shine" />
            <circle cx="149" cy="65" r="1" fill="white" class="eye-shine" />
          </g>
          <!-- Dynamic worried expression -->
          <path :d="`M132 75 Q140 ${75 + mistakes * 2} 148 75`" fill="none" class="smile" />
        </g>
      </g>

      <!-- Body parts with better curves -->
      <path v-if="mistakes >= 2" d="M140 90 Q138 120 140 150" class="hangman-part body" />

      <path v-if="mistakes >= 3" d="M140 120 Q130 110 120 100" class="hangman-part arm-left" />

      <path v-if="mistakes >= 4" d="M140 120 Q150 110 160 100" class="hangman-part arm-right" />

      <path v-if="mistakes >= 5" d="M140 150 Q130 165 120 180" class="hangman-part leg-left" />

      <path v-if="mistakes >= 6" d="M140 150 Q150 165 160 180" class="hangman-part leg-right" />
    </svg>
  </div>
</template>

<script setup lang="ts">
defineProps<{
  mistakes: number
  isGameOver: boolean
}>()
</script>

<style scoped>
.hangman-container {
  display: flex;
  justify-content: center;
  margin: 20px 0;
  perspective: 1000px;
  position: relative;
  overflow: hidden;
}

.hangman-svg {
  background: var(--background-color);
  border-radius: 10px;
  padding: 20px;
  box-shadow: 0 2px 8px var(--shadow-color);
  transition: transform 0.3s ease;
  position: relative;
  z-index: 1;
}

/* Cloud decoration */
.cloud {
  fill: rgba(255, 255, 255, 0.8);
  animation: float 4s ease-in-out infinite;
}

@keyframes float {
  0%,
  100% {
    transform: translateY(0);
  }
  50% {
    transform: translateY(-5px);
  }
}

.platform {
  fill: #654321;
  stroke: none;
  filter: drop-shadow(0 3px 2px rgba(0, 0, 0, 0.2));
}

.pole {
  fill: #8b4513;
  fill: url(#woodPattern);
}

.top {
  fill: #8b4513;
}

.rope {
  stroke: #a0522d;
  stroke-width: 3;
  fill: none;
  stroke-dasharray: 4;
}

.rope.swing {
  animation: swingRope 2s ease-in-out infinite;
}

@keyframes swingRope {
  0%,
  100% {
    transform: rotate(-1deg);
  }
  50% {
    transform: rotate(1deg);
  }
}

.hangman-part {
  stroke: var(--danger-color);
  stroke-width: 4;
  fill: none;
  stroke-linecap: round;
  animation: appear 0.5s ease;
  filter: drop-shadow(1px 1px 2px rgba(0, 0, 0, 0.2));
}

.eye {
  fill: var(--danger-color);
  transition: transform 0.3s ease;
}

.eye-shine {
  animation: blink 4s infinite;
}

@keyframes blink {
  0%,
  48%,
  52%,
  100% {
    transform: scaleY(1);
  }
  50% {
    transform: scaleY(0.1);
  }
}

.head:hover .eye {
  transform: scale(1.2);
}

.smile {
  stroke: var(--danger-color);
  transition: d 0.3s ease;
}

@keyframes appear {
  from {
    opacity: 0;
    transform: scale(0.8);
  }
  to {
    opacity: 1;
    transform: scale(1);
  }
}
</style>
