<!-- AllHashList.vue -->
<script setup>
import {computed, defineProps} from 'vue'

const props = defineProps({
  list: Array
})

const displayedList = computed(() =>
    props.list.slice(-50).reverse()
)
</script>

<template>
  <div class="all-hash-list">
    <h3>Recent Attempts (Last 50)</h3>
    <ul style="list-style: none">
      <li
          v-for="(item, index) in displayedList"
          :key="index"
          class="hash-item"
      >
        {{ item }}
      </li>
    </ul>
  </div>
</template>

<style scoped lang="scss">
.all-hash-list {
  background: white;
  padding: 1.5rem;
  border-radius: 12px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
  height: calc(100vh - 4rem);
  overflow-y: auto;
  list-style: none;
  h3 {
    margin-bottom: 1rem;
    color: #2c3e50;
    font-size: 1.1rem;
  }

  .hash-item {
    padding: 0.5rem 1rem;
    margin-bottom: 0.25rem;
    font-family: monospace;
    background: #f8f9fa;
    border-radius: 4px;
    font-size: 0.9rem;
    color: #666;
    transition: all 0.2s;

    &:hover {
      background: #e9ecef;
    }
  }
}
</style>