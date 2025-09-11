<template>
  <div class="aria2-tip" :class="`aria2-tip--${type}`" v-if="show">
    <div class="aria2-tip__icon">
      <span v-if="type === 'success'">✓</span>
      <span v-else-if="type === 'error'">✕</span>
      <span v-else-if="type === 'warning'">⚠</span>
      <span v-else-if="type === 'info'">ℹ</span>
    </div>
    <div class="aria2-tip__content">
      <slot></slot>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'

const show = ref(false)
const type = ref('info')
let timer = null

const open = (toastType = 'info') => {
  // 清除之前的定时器
  if (timer) {
    clearTimeout(timer)
  }
  
  type.value = toastType
  show.value = true
  timer = setTimeout(() => {
    show.value = false
  }, 3000)
}

defineExpose({ open })
</script>

<style scoped>
.aria2-tip {
  padding: 15px 20px;
  position: absolute;
  top: 30px;
  left: 50%;
  transform: translateX(-50%);
  color: #fff;
  border-radius: 8px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  font-size: 14px;
  z-index: 9999;
  min-width: 300px;
  text-align: center;
  display: flex;
  align-items: center;
  gap: 10px;
}

.aria2-tip--success {
  background: rgba(103, 194, 58, 0.9);
}

.aria2-tip--error {
  background: rgba(245, 108, 108, 0.9);
}

.aria2-tip--warning {
  background: rgba(230, 162, 60, 0.9);
}

.aria2-tip--info {
  background: rgba(64, 158, 255, 0.9);
}

.aria2-tip__icon {
  font-size: 18px;
  font-weight: bold;
  flex-shrink: 0;
}

.aria2-tip__content {
  flex: 1;
}
</style>
