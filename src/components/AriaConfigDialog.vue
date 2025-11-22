<template>
  <div v-if="show">
    <div class="dialog-overlay" @click="close"></div>
    <div style="width: 400px" class="dialog">
      <h2>请配置你的aria2</h2>
      <div class="close" @click="close">×</div>
      
      <!-- 连接状态显示 -->
      <div class="connection-status">
        <div class="status-indicator">
          <div class="status-dot" :class="connectionStatus.class"></div>
          <span class="status-text">{{ connectionStatus.text }}</span>
        </div>
        <button class="test-btn" @click="testConnection" :disabled="isTestingConnection">
          {{ isTestingConnection ? '测试中...' : '测试连接' }}
        </button>
      </div>
      <ul class="config-list">
        <li>
          <div class="label">RPC地址</div>
          <input v-model="form.host" type="text" placeholder="http://127.0.0.1:6800/jsonrpc">
          <p class="guidance">Aria2 RPC服务的地址，通常是 `http://127.0.0.1:6800/jsonrpc`。如果你在Docker中运行，可能需要使用宿主机的IP地址。</p>
        </li>
        <li>
          <div class="label">RPC密钥</div>
          <input v-model="form.token" type="text" placeholder="没有请留空">
          <p class="guidance">Aria2 RPC的密钥，如果你在Aria2配置中设置了 `rpc-secret`，请在此填写。如果没有设置，请留空。</p>
        </li>
        <li>
          <div class="label">下载路径</div>
          <input v-model="form.path" type="text" placeholder="C:/Users/admin/Downloads/">
          <p class="guidance">文件在服务器上的保存路径。例如：`/downloads/` (Linux) 或 `D:\Downloads` (Windows)\。请确保Aria2有写入该目录的权限。</p>
        </li>
        <li>
          <div class="label">其他参数</div>
          <input v-model="form.params" type="text" placeholder="user-agent=xxx;header=xxx">
          <p class="guidance">Aria2的额外参数，以分号 `;` 分隔，例如 `user-agent=Mozilla;split=10`。这些参数会直接传递给Aria2。</p>
        </li>
      </ul>
      <div class="footer">
        <div class="btn el-button el-button--primary" @click="save">保存</div>
      </div>
    </div>
  </div>
</template>

<script setup>
import {reactive, ref, computed, onMounted} from 'vue'
import { pushToAria } from '../api/index.js'

const props = defineProps({
  show: Boolean
})
const emits = defineEmits(['close', 'msg'])

const close = () => {
  emits('close')
}

const form = reactive({
  host: window.localStorage.getItem('ariaHost') || '',
  path: window.localStorage.getItem('ariaPath') || '',
  token: window.localStorage.getItem('ariaToken') || '',
  params: window.localStorage.getItem('ariaParams') || ''
})

// 连接状态管理
const connectionState = ref('unknown') // 'unknown', 'connected', 'disconnected', 'testing'
const isTestingConnection = ref(false)

// 连接状态计算属性
const connectionStatus = computed(() => {
  switch (connectionState.value) {
    case 'connected':
      return { class: 'connected', text: 'Aria2连接正常' }
    case 'disconnected':
      return { class: 'disconnected', text: 'Aria2连接失败' }
    case 'testing':
      return { class: 'testing', text: '正在测试连接...' }
    default:
      return { class: 'unknown', text: '连接状态未知' }
  }
})

// 测试连接函数
const testConnection = async () => {
  if (!form.host) {
    emits('msg', '请先填写RPC地址', 'warning')
    return
  }

  isTestingConnection.value = true
  connectionState.value = 'testing'

  try {
    const rpcUrl = form.host
    const rpcToken = form.token ? `token:${form.token}` : ''
    const payload = {
      jsonrpc: '2.0',
      method: 'aria2.getVersion',
      id: 1,
      params: rpcToken ? [rpcToken] : []
    }
    const response = await pushToAria(rpcUrl, payload)
    if (response && response.result) {
      connectionState.value = 'connected'
      // emits('msg', `Aria2连接成功！`, 'success')
    } else {
      connectionState.value = 'disconnected'
      emits('msg', 'Aria2连接失败，请检查配置', 'error')
    }
  } catch (error) {
    console.error('Aria2连接测试失败:', error)
    connectionState.value = 'disconnected'
    emits('msg', `连接失败: ${error.message || '请检查RPC地址和密钥'}`, 'error')
  } finally {
    isTestingConnection.value = false
  }
}

const save = async () => {
  // 路径验证
  if (form.path && !form.path.endsWith('/') && !form.path.endsWith('\\')) {
    form.path += '/'
  }

  window.localStorage.setItem('ariaHost',form.host)
  window.localStorage.setItem('ariaPath',form.path)
  window.localStorage.setItem('ariaToken',form.token)
  window.localStorage.setItem('ariaParams',form.params)
  
  emits('msg', '配置保存成功！', 'success')
  close()
}

// 组件挂载时检查连接状态
onMounted(() => {
  if (form.host) {
    testConnection()
  }
})
</script>

<style scoped>
/* 连接状态样式 */
.connection-status {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 16px 20px;
  margin-bottom: 20px;
  background: linear-gradient(135deg, rgba(99, 102, 241, 0.05) 0%, rgba(168, 85, 247, 0.05) 100%);
  border-radius: 12px;
  border: 1px solid rgba(99, 102, 241, 0.1);
}

.status-indicator {
  display: flex;
  align-items: center;
  gap: 12px;
}

.status-dot {
  width: 12px;
  height: 12px;
  border-radius: 50%;
  position: relative;
}

.status-dot.connected {
  background-color: #10b981;
  box-shadow: 0 0 0 2px rgba(16, 185, 129, 0.2);
}

.status-dot.connected::before {
  content: '';
  position: absolute;
  width: 12px;
  height: 12px;
  border-radius: 50%;
  background-color: #10b981;
  animation: pulse 2s infinite;
}

.status-dot.disconnected {
  background-color: #ef4444;
  box-shadow: 0 0 0 2px rgba(239, 68, 68, 0.2);
}

.status-dot.testing {
  background-color: #f59e0b;
  box-shadow: 0 0 0 2px rgba(245, 158, 11, 0.2);
  animation: pulse 1s infinite;
}

.status-dot.unknown {
  background-color: #6b7280;
  box-shadow: 0 0 0 2px rgba(107, 114, 128, 0.2);
}

@keyframes pulse {
  0% {
    transform: scale(1);
    opacity: 1;
  }
  50% {
    transform: scale(1.2);
    opacity: 0.7;
  }
  100% {
    transform: scale(1);
    opacity: 1;
  }
}

.status-text {
  font-size: 14px;
  font-weight: 500;
  color: #374151;
}

.test-btn {
  padding: 8px 16px;
  background: linear-gradient(135deg, #f8fafc 0%, #e2e8f0 100%);
  color: #475569;
  border: 2px solid #cbd5e1;
  border-radius: 8px;
  cursor: pointer;
  font-size: 14px;
  font-weight: 500;
  transition: all 0.2s ease;
}

.test-btn:hover:not(:disabled) {
  background: linear-gradient(135deg, #e2e8f0 0%, #cbd5e1 100%);
  border-color: #94a3b8;
  transform: translateY(-1px);
}

.test-btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.dialog-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.6);
  z-index: 3001;
}

.dialog {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  background: #fff;
  z-index: 10000;
  padding: 30px;
  box-shadow: 0 5px 15px rgba(0, 0, 0, 0.3);
  border-radius: 10px;
  width: 90%;
  max-width: 500px;
  box-sizing: border-box;
}

.dialog h2 {
  text-align: center;
  color: #333;
  margin-bottom: 20px;
}

.dialog .close {
  position: absolute;
  right: 15px;
  top: 15px;
  font-size: 30px;
  cursor: pointer;
  color: #999;
  transition: color 0.3s ease;
}

.dialog .close:hover {
  color: #666;
}

.config-list {
  margin-top: 20px;
}

.config-list li {
  margin-bottom: 15px;
}

.config-list li .label {
  font-weight: bold;
  margin-bottom: 5px;
  display: block;
  color: #555;
}

.config-list li input[type="text"] {
  width: 100%;
  padding: 10px;
  border: 1px solid #dcdfe6;
  border-radius: 4px;
  box-sizing: border-box;
  font-size: 14px;
  transition: border-color 0.3s ease;
}

.config-list li input[type="text"]:focus {
  border-color: #409eff;
  outline: none;
}

.footer {
  margin-top: 20px;
  display: flex;
  justify-content: right;
  align-items: center;
  gap: 12px;
}

.btn.el-button {
  padding: 12px 24px;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  font-size: 14px;
  font-weight: 500;
  transition: all 0.2s ease;
}

.btn.el-button--primary {
  background: linear-gradient(135deg, #6366f1 0%, #8b5cf6 100%);
  color: white;
}

.btn.el-button--primary:hover {
  background: linear-gradient(135deg, #5b21b6 0%, #7c3aed 100%);
  transform: translateY(-1px);
}

.btn.el-button--secondary {
  background: linear-gradient(135deg, #f8fafc 0%, #e2e8f0 100%);
  color: #475569;
  border: 2px solid #cbd5e1;
}

.btn.el-button--secondary:hover:not(:disabled) {
  background: linear-gradient(135deg, #e2e8f0 0%, #cbd5e1 100%);
  border-color: #94a3b8;
  transform: translateY(-1px);
}

.btn.el-button:disabled {
  opacity: 0.6;
  cursor: not-allowed;
  transform: none;
}

.guidance {
  font-size: 12px;
  color: #909399;
  margin-top: 5px;
  line-height: 1.5;
}
.form {
  margin-top: 20px;
}
.xz-input {
  border: #d9d9d9 1px solid;
  margin-bottom: 10px;
  padding: 5px;
  margin-top: 5px;
}
</style>
