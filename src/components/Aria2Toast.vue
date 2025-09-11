<template>
  <div class="aria2-tip" v-if="show">
    <slot></slot>
  </div>
</template>

<script setup>
import { ref } from 'vue'

const show = ref(false)
let timer = null // 添加一个变量来跟踪定时器

const open = () => {
  // 清除之前的定时器
  if (timer) {
    clearTimeout(timer)
  }
  
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
  background: rgba(0, 0, 0, 0.85);
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
}
</style>
