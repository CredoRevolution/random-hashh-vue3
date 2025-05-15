<script setup>
import { ref, computed, watch } from "vue";
import CorrectHashList from "./components/CorrectHashList.vue";
import AllHashList from "./components/AllHashList.vue";

const hash = ref('');
const attemptsCount = ref(0);
const attemptsList = ref([]);
const isGuessing = ref(false);
const startTime = ref(null);
const elapsedTime = ref('00:00:00');
const timer = ref(null);
const useMemory = ref(false);
const knownCharacters = ref({});
const delayTime = ref(100);


watch(hash, () => {
  knownCharacters.value = {};
});

const startGuessing = async () => {
  if (!hash.value || isGuessing.value) return;

  isGuessing.value = true;
  attemptsCount.value = 0;
  attemptsList.value = [];
  startTime.value = Date.now();
  startTimer();

  const targetHash = hash.value;
  const length = targetHash.length;
  const characters = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz";

  let found = false;

  while (!found && isGuessing.value) {
    let randomHash = Array.from({length}, (_, index) => {
      // Используем известные символы если доступны
      if (useMemory.value && knownCharacters.value[index]) {
        return knownCharacters.value[index];
      }
      return characters[Math.floor(Math.random() * characters.length)];
    }).join('');

    // Обновляем известные символы
    if (useMemory.value) {
      const newKnown = {...knownCharacters.value};
      randomHash.split('').forEach((char, i) => {
        if (char === targetHash[i]) {
          newKnown[i] = char;
        }
      });
      knownCharacters.value = newKnown;
    }

    attemptsCount.value++;
    attemptsList.value.push(randomHash);

    if (attemptsList.value.length > 50) {
      attemptsList.value.shift();
    }

    if (randomHash === targetHash) {
      found = true;
    } else {
      await new Promise(resolve => setTimeout(resolve, delayTime.value));
    }
  }

  stopTimer();
  isGuessing.value = false;
};

const startTimer = () => {
  timer.value = setInterval(() => {
    const total = Date.now() - startTime.value;
    const hours = Math.floor(total / 3600000).toString().padStart(2, '0');
    const minutes = Math.floor((total % 3600000) / 60000).toString().padStart(2, '0');
    const seconds = Math.floor((total % 60000) / 1000).toString().padStart(2, '0');
    elapsedTime.value = `${hours}:${minutes}:${seconds}`;
  }, 1000);
};

const resetMemory = () => {
  knownCharacters.value = {};
};

const stopTimer = () => {
  clearInterval(timer.value);
  timer.value = null;
};

const stopGuessing = () => {
  isGuessing.value = false;
  stopTimer();
};

const chance = computed(() => {
  if (useMemory.value || !hash.value) return null;

  const charsetSize = 62; // 10 цифр + 26*2 букв
  const length = hash.value.length;
  const possibleCombinations = Math.pow(charsetSize, length);

  return {
    perAttempt: 1 / possibleCombinations,
    afterAttempts: 1 - Math.pow(1 - 1/possibleCombinations, attemptsCount.value)
  };
});

// Форматирование больших чисел
const formatProbability = (num) => {
  if (num === 0) return '0';
  if (num < 1e-6) return num.toExponential(2);
  if (num < 0.01) return num.toFixed(6);
  return num.toFixed(6);
};
</script>

<template>
  <div class="main-wrapper">
    <!-- Left Column -->
    <div class="control-panel">
      <div class="input-group">
        <input
            v-model="hash"
            placeholder="Enter target hash"
            :disabled="isGuessing"
        >
        <div class="button-group">
          <button
              @click="startGuessing"
              :disabled="isGuessing || !hash"
              class="start-btn"
          >
            {{ isGuessing ? 'Guessing...' : 'Start' }}
          </button>
          <button
              @click="stopGuessing"
              :disabled="!isGuessing"
              class="stop-btn"
          >
            Stop
          </button>
        </div>
        <div class="stats">
          <div>Attempts: {{ attemptsCount }}</div>
          <div>Time: {{ elapsedTime }}</div>
          <div v-if="!useMemory && hash">
            Success chance:
            <div class="chance-details">
              <span>Per attempt: {{ formatProbability(chance.perAttempt) }}</span>
              <span>After attempts: {{ formatProbability(chance.afterAttempts) }}</span>
              <span class="hint">(1 in {{ (1/chance.perAttempt).toLocaleString() }} combinations)</span>
            </div>
          </div>
        </div>
        <div class="speed-control">
          <label>Speed control: {{ delayTime }}ms</label>
          <div class="slider-container">
            <input
                type="range"
                v-model.number="delayTime"
                min="1"
                max="1000"
                step="1"
                class="slider"
                :disabled="isGuessing"
            >
            <div class="slider-labels">
              <span>Fast</span>
              <span>Slow</span>
            </div>
          </div>
        </div>
        <div class="memory-settings">
          <label>
            <input type="checkbox" v-model="useMemory">
            Use memory
          </label>
          <button
              @click="resetMemory"
              class="reset-btn"
              :disabled="!Object.keys(knownCharacters).length"
          >
            Clear Memory
          </button>
        </div>
        <div class="memory-preview" v-if="useMemory && hash">
          <div class="title">Known characters:</div>
          <div class="chars-preview">
            <span
                v-for="i in hash.length"
                :key="i"
                class="char-box"
            >
              {{ knownCharacters[i-1] || '_' }}
            </span>
          </div>
        </div>
      </div>
    </div>

    <!-- Middle Column -->
    <CorrectHashList
        :list="attemptsList"
        :correct-hash="hash"
        class="correct-list"
    />

    <!-- Right Column -->
    <AllHashList
        :list="attemptsList"
        class="all-hash-list"
    />
  </div>
</template>

<style lang="scss">
*{
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}
.main-wrapper {
  display: grid;
  grid-template-columns: 1fr 1fr 1fr;
  gap: 2rem;
  height: 100vh;
  padding: 2rem;
  background: #f0f2f5;
}

.control-panel {
  background: white;
  padding: 2rem;
  border-radius: 12px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);

  .input-group {
    display: flex;
    flex-direction: column;
    gap: 1rem;

    input {
      padding: 0.8rem;
      border: 1px solid #ddd;
      border-radius: 8px;
      font-size: 1rem;
      transition: border-color 0.3s;

      &:focus {
        outline: none;
        border-color: #646cff;
      }
    }
  }

  .button-group {
    display: flex;
    gap: 1rem;

    button {
      flex: 1;
      padding: 0.8rem;
      border: none;
      border-radius: 8px;
      cursor: pointer;
      transition: all 0.3s;

      &.start-btn {
        background: #646cff;
        color: white;

        &:hover:not(:disabled) {
          background: #535bf2;
        }
      }

      &.stop-btn {
        background: #ff4757;
        color: white;

        &:hover:not(:disabled) {
          background: #ff3f4f;
        }
      }

      &:disabled {
        opacity: 0.7;
        cursor: not-allowed;
      }
    }
  }

  .stats {
    margin-top: 1rem;
    padding: 1rem;
    background: #f8f9fa;
    border-radius: 8px;
    font-size: 0.9rem;
    color: #555;
    .chance-details {
      margin-top: 0.1rem;
      padding: 0.5rem;
      padding-top: 0;
      background: #f8f9fa;
      border-radius: 6px;
      font-size: 0.8rem;

      span {
        display: block;
        margin: 0.2rem 0;
      }

      .hint {
        color: #666;
        font-size: 0.7rem;
        margin-top: 0.3rem;
      }
    }
  }
.memory-settings {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1rem;
  padding: 0.5rem;
  background: #f8f9fa;
  border-radius: 8px;

  label {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    font-size: 0.9rem;
  }

  .reset-btn {
    padding: 0.3rem 0.6rem;
    background: #ff6b6b;
    color: white;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    transition: opacity 0.3s;

    &:disabled {
      opacity: 0.5;
      cursor: not-allowed;
    }
  }
}

  .memory-preview {
    margin: 1rem 0;
    padding: 1rem;
    background: #fff;
    border-radius: 8px;
    box-shadow: 0 1px 3px rgba(0,0,0,0.1);

    .title {
      font-size: 0.8rem;
      color: #666;
      margin-bottom: 0.5rem;
    }

    .chars-preview {
      display: flex;
      gap: 0.3rem;
      flex-wrap: wrap;

      .char-box {
        width: 28px;
        height: 28px;
        display: flex;
        align-items: center;
        justify-content: center;
        border: 2px solid #ddd;
        border-radius: 4px;
        font-family: monospace;
        font-weight: bold;
        background: #f8f9fa;

        &:not(:empty) {
          border-color: #2ecc71;
          background: #e8faf1;
        }
      }
    }
  }
  .speed-control {
    margin: 1.5rem 0;
    padding: 1rem;
    background: #fff;
    border-radius: 8px;
    box-shadow: 0 1px 3px rgba(0,0,0,0.1);

    label {
      display: block;
      font-size: 0.9rem;
      color: #666;
      margin-bottom: 0.5rem;
    }
  }

  .slider-container {
    position: relative;
    padding: 0.5rem 0;
  }

  .slider {
    -webkit-appearance: none;
    width: 100%;
    height: 6px;
    border-radius: 3px;
    background: #ddd;
    outline: none;
    opacity: 0.9;
    transition: opacity 0.2s;

    &:hover {
      opacity: 1;
    }

    &::-webkit-slider-thumb {
      -webkit-appearance: none;
      appearance: none;
      width: 18px;
      height: 18px;
      border-radius: 50%;
      background: #646cff;
      cursor: pointer;
      transition: transform 0.2s;

      &:hover {
        transform: scale(1.2);
      }
    }
  }

  .slider-labels {
    display: flex;
    justify-content: space-between;
    margin-top: 0.5rem;
    font-size: 0.8rem;
    color: #888;
  }
}
</style>