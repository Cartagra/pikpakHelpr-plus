<script setup>
// This starter template is using Vue 3 <script setup> SFCs
// Check out https://vuejs.org/api/sfc-script-setup.html#script-setup
import { ref } from "vue";
import AriaDownloadDialog from "./components/AriaDownloadDialog.vue";
import AriaConfigDialog from "./components/AriaConfigDialog.vue"
import Aria2Toast from './components/Aria2Toast.vue'
import FloatingWindow from './components/FloatingWindow.vue'

const downloadShow = ref(false) // 下载
const configShow = ref(false) // 配置
const tip = ref('') // 提示
const toastRef = ref(null)
const showPlugin = ref(false)

const showToast = (val, type = 'info') => {
  tip.value = val
  toastRef.value.open(type)
}

if (location.pathname !== '/') {
  showPlugin.value = true
}

</script>

<template>
  <FloatingWindow 
      v-if="showPlugin" 
      @download="downloadShow = true" 
    />
  <AriaDownloadDialog 
      :show="downloadShow" 
      @close="downloadShow = false"
      @openConfig="configShow = true"
      @msg="showToast"
    />
  <AriaConfigDialog 
      :show="configShow" 
      @close="configShow = false"
      @msg="showToast"
    />
  <Aria2Toast ref="toastRef">{{ tip }}</Aria2Toast>
</template>

<style scoped>
/* 悬浮窗组件的样式已在FloatingWindow.vue中定义 */
</style>
