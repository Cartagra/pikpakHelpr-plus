<template>
  <div style="width: 400px" v-if="show" class="dialog">
    <h2>请配置你的aria2</h2>
    <div class="close" @click="close">×</div>
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
</template>

<script setup>
import {reactive, watch} from 'vue'
import { pushToAria } from '../api/index.js'

const props = defineProps({
  show: Boolean
})
const emits = defineEmits(['update:show', 'msg'])

const close = () => {
  emits('update:show', false)
}

const form = reactive({
  host: window.localStorage.getItem('ariaHost') || '',
  path: window.localStorage.getItem('ariaPath') || '',
  token: window.localStorage.getItem('ariaToken') || '',
  params: window.localStorage.getItem('ariaParams') || ''
})

const save = async () => {
  // 路径验证
  if (form.path && !form.path.endsWith('/') && !form.path.endsWith('\\')) {
    form.path += '/'
  }

  window.localStorage.setItem('ariaHost',form.host)
  window.localStorage.setItem('ariaPath',form.path)
  window.localStorage.setItem('ariaToken',form.token)
  window.localStorage.setItem('ariaParams',form.params)
  close()

  // 连接测试
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
      emits('msg', '保存成功！Aria2连接成功！')
    } else {
      emits('msg', '保存成功！但Aria2连接失败，请检查配置。')
    }
  } catch (error) {
    console.error('Aria2连接测试失败:', error)
    emits('msg', '保存成功！但Aria2连接失败，请检查配置。')
  }
}
</script>

<style scoped>
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
  flex-direction: row-reverse;
}

.btn.el-button {
  padding: 10px 20px;
  background-color: #409eff;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 16px;
  transition: background-color 0.3s ease;
}

.btn.el-button:hover {
  background-color: #66b1ff;
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
