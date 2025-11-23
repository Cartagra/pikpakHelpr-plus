<template>
  <div v-if="show">
    <div class="dialog-overlay" @click="close"></div>
    <div style="width: 60%" class="dialog">
      <h2>请勾选你要下载的</h2>
      <div class="close" @click="close">×</div>
      <div class="toolbar">
        <label for="checkbox">
          <input @change="onCheckAll" type="checkbox" id="checkbox" v-model="checkedAll">
          全选
        </label>
        <div class="sort-options">
          <label for="sort-by">排序方式:</label>
          <select id="sort-by" v-model="sortBy" @change="sortList">
            <option value="name">名称</option>
            <option value="created_time">创建时间</option>
            <option value="modified_time">修改时间</option>
            <option value="size">大小</option>
            <option value="file_category">文件类型</option>
          </select>
          <select id="sort-direction" v-model="sortDirection" @change="sortList">
            <option value="asc">升序</option>
            <option value="desc">降序</option>
          </select>
        </div>
      </div>
      <ul class="movies">
        <li v-for="(item, index) in list" :key="item.id">
          <input @change="onCheck" type="checkbox" :id="item.id" :value="index" v-model="selected">
          <span class="icon">{{ item.type === 'drive#folder' ? '📁' : '📄' }}</span>
          <span class="file-name">{{ item.name }}</span>
          <span class="file-info">{{ formatFileInfo(item) }}</span>
        </li>
      </ul>
      
      <!-- aria2连接状态显示 -->
      <div class="connection-status">
        <div class="status-indicator">
          <div class="status-dot" :class="connectionStatus.class"></div>
          <span class="status-text">{{ connectionStatus.text }}</span>
        </div>
        <button class="test-btn" @click="testConnection" :disabled="isTestingConnection">
          {{ isTestingConnection ? '测试中...' : '测试连接' }}
        </button>
      </div>
      
      <div class="footer">
        <div class="btn el-button el-button--secondary" @click="openConfig">配置aria2</div>
        <div class="btn el-button el-button--primary" @click="pushBefore">推送到aria2</div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, watch, computed, onMounted } from 'vue'
import { getDownload, pushToAria, getList } from '../api'

const props = defineProps({
  show: Boolean
})
const emits = defineEmits(['close', 'msg', 'openConfig'])

const list = ref([])
const selected = ref([])
const checkedAll = ref(false)
const selectedItems = ref([]) // 选中的项目
const isForbidden = ref(false) // 按钮是否禁用，防抖

const sortBy = ref('name') // Default sort by name
const sortDirection = ref('asc') // Default sort direction

// aria2连接状态管理
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
      return { class: 'unknown', text: 'Aria2连接状态未知' }
  }
})

watch(
  () => props.show,
  (val) => {
    if (val) {
      const tempList = []
      let parent_id = window.location.href.split('/').pop()
      if (parent_id == 'all') parent_id = ''
      emits('msg', '开始加载文件列表，请稍等', 'info')
      getList(parent_id).then(res => {
        res.files.forEach(item => {
          tempList.push({ id: item.id, name: item.name, type: item.kind, created_time: item.created_time, modified_time: item.modified_time, size: item.size, file_category: item.file_category })
        })
        list.value = tempList
        sortList(); // Apply default sort after loading
      }).finally(() => {
        emits('msg', '文件列表加载完成', 'success')
      })
      
      // 对话框打开时检测aria2连接状态
      setTimeout(() => {
        testConnection()
      }, 500)
    }
  }
)


const close = () => {
  selected.value = []
  checkedAll.value = false
  isForbidden.value = false
  emits('close')
}

const openConfig = () => {
  emits('openConfig')
}

// 测试aria2连接
const testConnection = async () => {
  if (isTestingConnection.value) return
  
  isTestingConnection.value = true
  connectionState.value = 'testing'
  
  try {
    const host = window.localStorage.getItem('ariaHost') || ''
    const token = window.localStorage.getItem('ariaToken') || ''
    
    if (!host) {
      throw new Error('请先配置Aria2 RPC地址')
    }
    
    // 使用aria2.getVersion方法测试连接
    const payload = {
      jsonrpc: '2.0',
      method: 'aria2.getVersion',
      id: 1,
      params: token ? [`token:${token}`] : []
    }
    
    const response = await pushToAria(host, payload)
    if (response && response.result) {
      connectionState.value = 'connected'
      // emits('msg', 'Aria2连接测试成功', 'success')
    } else {
      connectionState.value = 'disconnected'
      emits('msg', 'Aria2连接失败，请检查配置', 'error')
    }
  } catch (error) {
    connectionState.value = 'disconnected'
    emits('msg', `Aria2连接测试失败: ${error.message}`, 'error')
  } finally {
    isTestingConnection.value = false
  }
}

// 组件挂载时检查连接
onMounted(() => {
  // 延迟检查连接，避免与文件列表加载冲突
  setTimeout(() => {
    testConnection()
  }, 1000)
})

// 选择
const onCheckAll = () => {
  if (checkedAll.value) {
    selected.value = list.value.map((item, index) => index)
  } else {
    selected.value = []
  }
}
const onCheck = () => {
  checkedAll.value = selected.value.length === list.value.length
}

const sortList = () => {
  list.value.sort((a, b) => {
    // Folders first
    if (a.type === 'drive#folder' && b.type !== 'drive#folder') {
      return -1;
    }
    if (a.type !== 'drive#folder' && b.type === 'drive#folder') {
      return 1;
    }

    // Then by selected sort option
    let aValue = a[sortBy.value];
    let bValue = b[sortBy.value];

    if (sortBy.value === 'size') {
      aValue = parseInt(aValue);
      bValue = parseInt(bValue);
    } else if (sortBy.value === 'created_time' || sortBy.value === 'modified_time') {
      aValue = new Date(aValue).getTime();
      bValue = new Date(bValue).getTime();
    }

    let comparison = 0;
    if (aValue > bValue) {
      comparison = 1;
    } else if (aValue < bValue) {
      comparison = -1;
    }

    return sortDirection.value === 'asc' ? comparison : -comparison;
  });
  // After sorting, re-select items to maintain checked state
  updateSelectedIndices();
};

const formatBytes = (bytes, decimals = 2) => {
  if (bytes === 0) return '0 Bytes';

  const k = 1024;
  const dm = decimals < 0 ? 0 : decimals;
  const sizes = ['Bytes', 'KB', 'MB', 'GB', 'TB', 'PB', 'EB', 'ZB', 'YB'];

  const i = Math.floor(Math.log(bytes) / Math.log(k));

  return parseFloat((bytes / Math.pow(k, i)).toFixed(dm)) + ' ' + sizes[i];
}

const formatFileInfo = (item) => {
  switch (sortBy.value) {
    case 'size':
      return item.size ? formatBytes(parseInt(item.size)) : 'N/A';
    case 'created_time':
      return item.created_time ? new Date(item.created_time).toLocaleString() : 'N/A';
    case 'modified_time':
      return item.modified_time ? new Date(item.modified_time).toLocaleString() : 'N/A';
    case 'file_category':
      return item.file_category || 'N/A';
    default:
      return '';
  }
};

const updateSelectedIndices = () => {
  const currentlySelectedIds = new Set(selected.value.map(index => list.value[index].id));
  selected.value = [];
  list.value.forEach((item, index) => {
    if (currentlySelectedIds.has(item.id)) {
      selected.value.push(index);
    }
  });
};

const getAllList = async () => {
  emits('msg', '开始获取文件内容', 'info');
  const initialSelectedItems = [];
  for (const index of selected.value) {
    initialSelectedItems.push(list.value[index]);
  }

  const allFiles = [];
  const foldersToProcess = [];

  // Separate initial selection into files and folders
  initialSelectedItems.forEach(item => {
    if (item.type === 'drive#folder') {
      // Start with folder name as path.
      foldersToProcess.push({ id: item.id, name: item.name, path: item.name });
    } else {
      // Files in root have no extra path
      allFiles.push({ ...item, path: '' });
    }
  });

  let processedCount = 0;
  while (foldersToProcess.length > 0) {
    const currentFolder = foldersToProcess.shift(); // Get a folder to process
    processedCount++;
    emits('msg', `正在扫描第 ${processedCount} 个文件夹: ${currentFolder.name}`, 'info');
    try {
      const result = await getList(currentFolder.id);
      if (result.files) {
        for (const file of result.files) {
          if (file.kind === 'drive#folder') {
            foldersToProcess.push({ id: file.id, name: file.name, path: `${currentFolder.path}/${file.name}` });
          } else {
            allFiles.push({ ...file, path: currentFolder.path });
          }
        }
      }
    } catch (e) {
      emits('msg', `获取文件夹 ${currentFolder.name} 内容失败`, 'error');
      console.error(e);
    }
  }

  selectedItems.value = allFiles;
  emits('msg', `文件获取完毕，共${allFiles.length}个文件。`, 'success');
}

const pushBefore = async () => {
  if (!isForbidden.value) {
    isForbidden.value = true
    await getAllList()
    push()
  } else {
    emits('msg', '已经开始推送了', 'warning')
  }

}


const push = async () => {
  let total = selectedItems.value.length
  let success = 0
  let fail = 0
  let ariaHost = window.localStorage.getItem('ariaHost') || ''
  let ariaPath = window.localStorage.getItem('ariaPath') || ''
  let ariaToken = window.localStorage.getItem('ariaToken') || ''
  let ariaParams = window.localStorage.getItem('ariaParams') || ''
  let errorMSG = ''
  let retryList = [] // 重试列表
  if (!ariaHost) {
    emits('msg', '请先配置aria2', 'error')
    close()
    return
  }
  console.log(`共${selectedItems.value.length}个项目`);
  let testIndex = 0
  for (let item of selectedItems.value) {
    getDownload(item.id).then((res) => {

      if (res.error_description) {
        emits('msg', `失败原因: ${res.error_description} 请刷新！`, 'error')
        return
      }
      emits('msg', `第${testIndex + 1}个项目下载链接获取成功`, 'success')
      console.log(`第${testIndex + 1}个项目下载链接获取成功`);
      let ariaData = {
        id: new Date().getTime(),
        jsonrpc: '2.0',
        method: 'aria2.addUri',
        params: [
          [res.web_content_link],
          { out: res.name }
        ]
      }
      if (ariaPath) {
        // 拼接路径
        ariaData.params[1].dir = ariaPath + (item.path || '')
      }
      if (ariaParams) {
        const customParams = ariaParams.split(';')
        customParams.forEach(item => {
          const customParam = item.split('=')
          ariaData.params[1][customParam[0]] = customParam[1]
        })
      }
      ariaToken && (ariaData.params.unshift(`token:${ariaToken}`))
      pushToAria(ariaHost, ariaData).then((ariares) => {
        let resoj = ariares
        // 失败时ariares为字符串类型，将其转为对象
        if(typeof ariares === 'string'){
          resoj = JSON.parse(ariares)
        }
        if (resoj.result) {
          success++
        } else {
          console.log(resoj);
          console.log(ariaData);
          errorMSG = resoj.error.message === 'Unauthorized' ? '密钥不对' : '推送失败'
          fail++
        }
      }).catch((e) => {
        console.warn(e);
        console.log(ariares);
        console.log(ariaData);
        errorMSG = `${e.statusText} 请检测配置`
        emits('msg', `失败原因: ${e.statusText}`, 'error')
        fail++
      }).finally(() => {
        total--
        if (total === 0) {
          const resultType = fail === 0 ? 'success' : (success === 0 ? 'error' : 'warning')
          emits('msg', `成功：${success} 失败: ${fail} ${fail !== 0 ? '失败原因：' + errorMSG : ''}`, resultType)
          console.info(`成功：${success} 失败: ${fail} ${fail !== 0 ? '失败原因：' + errorMSG : ''}`);
          if (retryList.length > 0) {
            console.log(retryList);
            emits('msg', `即将重试${retryList.length}个项目`, 'warning')
            console.log(`即将重试${retryList.length}个项目`)
            selectedItems.value = retryList
            retryList = [] // 清空重试列表
            push()
          } else {
            close()
          }
        }
      })
    }).catch((e) => {
      console.warn(`第${testIndex + 1}个项目下载链接获取失败`);
      retryList.push(selectedItems.value[testIndex])
      fail++
      total--
    })
      .finally(() => {
        testIndex++
      })
  }
}
</script>

<style scoped>
.dialog-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.6);
  z-index: 3000;
}

/* 现代化对话框样式 */
.dialog {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  background: linear-gradient(135deg, #ffffff 0%, #f8fafc 100%);
  z-index: 10000;
  padding: 32px;
  box-shadow: 0 25px 50px -12px rgba(0, 0, 0, 0.25), 0 0 0 1px rgba(255, 255, 255, 0.05);
  border-radius: 20px;
  width: 90vw;
  max-width: 800px;
  min-width: 320px;
  max-height: 90vh;
  box-sizing: border-box;
  backdrop-filter: blur(10px);
  border: 1px solid rgba(255, 255, 255, 0.2);
  animation: dialogFadeIn 0.3s cubic-bezier(0.4, 0, 0.2, 1);

  /* 修改对话框样式 */
  display: flex;
  flex-direction: column;
  max-height: 90vh;
  overflow: hidden; /* 防止整个对话框滚动 */
}

@keyframes dialogFadeIn {
  from {
    opacity: 0;
    transform: translate(-50%, -50%) scale(0.95);
  }
  to {
    opacity: 1;
    transform: translate(-50%, -50%) scale(1);
  }
}

/* 响应式设计 */
@media (max-width: 768px) {
  .dialog {
    width: 95vw;
    padding: 24px;
    border-radius: 16px;
  }
}

@media (max-width: 480px) {
  .dialog {
    width: 100vw;
    height: 100vh;
    max-height: 100vh;
    border-radius: 0;
    padding: 20px;
  }
}

/* 标题样式 */
.dialog h2 {
  text-align: center;
  color: #1e293b;
  margin-bottom: 24px;
  font-size: 24px;
  font-weight: 700;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

@media (max-width: 480px) {
  .dialog h2 {
    font-size: 20px;
    margin-bottom: 20px;
  }
}

/* 关闭按钮 */
.dialog .close {
  position: absolute;
  right: 20px;
  top: 20px;
  width: 40px;
  height: 40px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 24px;
  cursor: pointer;
  color: #64748b;
  background: rgba(248, 250, 252, 0.8);
  border-radius: 50%;
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  border: 1px solid rgba(226, 232, 240, 0.5);
}

.dialog .close:hover {
  color: #ef4444;
  background: rgba(254, 226, 226, 0.8);
  border-color: rgba(248, 113, 113, 0.3);
  transform: scale(1.05);
}

/* 工具栏样式 */
.toolbar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
  padding: 16px 20px;
  background: linear-gradient(
    135deg,
    rgba(99, 102, 241, 0.05) 0%,
    rgba(168, 85, 247, 0.05) 100%
  );
  border-radius: 12px;
  border: 1px solid rgba(99, 102, 241, 0.1);
}

@media (max-width: 640px) {
  .toolbar {
    flex-direction: column;
    gap: 16px;
    align-items: stretch;
  }
}

.toolbar input[type="checkbox"] {
  margin-right: 8px;
  transform: scale(1.3);
  accent-color: #6366f1;
}

.toolbar label {
  font-weight: 600;
  color: #374151;
  display: flex;
  align-items: center;
  cursor: pointer;
  transition: color 0.2s ease;
}

.toolbar label:hover {
  color: #6366f1;
}

/* 排序选项 */
.sort-options {
  display: flex;
  align-items: center;
  gap: 12px;
  flex-wrap: wrap;
}

@media (max-width: 640px) {
  .sort-options {
    justify-content: center;
  }
}

.sort-options label {
  font-size: 14px;
  font-weight: 500;
  color: #6b7280;
}

.sort-options select {
  padding: 8px 12px;
  border: 2px solid #e5e7eb;
  background: white;
  color: #374151;
  border-radius: 8px;
  font-size: 14px;
  transition: all 0.2s ease;
  cursor: pointer;
}

.sort-options select:focus {
  outline: none;
  border-color: #6366f1;
  box-shadow: 0 0 0 3px rgba(99, 102, 241, 0.1);
}

.sort-options select:hover {
  border-color: #9ca3af;
}

/* 文件列表容器 */
.movies {
  margin-top: 16px;

  /* 修改文件列表容器 */
  flex: 1; /* 关键修改：占据剩余空间 */
  min-height: 200px; /* 最小高度保障 */
  /* height: 400px; */

  overflow-y: auto;
  overflow-x: hidden; /* 防止出现横向滚动条 */
  border: 2px solid #f1f5f9;
  border-radius: 16px;
  padding: 0;
  background: linear-gradient(135deg, #ffffff 0%, #f8fafc 100%);
  box-shadow: inset 0 2px 4px rgba(0, 0, 0, 0.06);
}

@media (max-width: 480px) {
  .movies {
    height: 300px;
  }
}

/* 自定义滚动条 */
.movies::-webkit-scrollbar {
  width: 8px;
}

.movies::-webkit-scrollbar-track {
  background: #f1f5f9;
  border-radius: 8px;
}

.movies::-webkit-scrollbar-thumb {
  background: linear-gradient(135deg, #cbd5e1 0%, #94a3b8 100%);
  border-radius: 8px;
  transition: background 0.2s ease;
}

.movies::-webkit-scrollbar-thumb:hover {
  background: linear-gradient(135deg, #94a3b8 0%, #64748b 100%);
}

/* 文件列表项 */
.movies li {
  display: flex;
  align-items: center;
  padding: 16px 20px;
  border-bottom: 1px solid rgba(226, 232, 240, 0.6);
  font-size: 14px;
  color: #334155;
  transition: all 0.2s ease;
  cursor: pointer;
  margin-right: -4px; /* 预留移动空间 */
  overflow: hidden; /* 防止子元素溢出 */
}

.movies li:hover {
  background: linear-gradient(
    135deg,
    rgba(99, 102, 241, 0.05) 0%,
    rgba(168, 85, 247, 0.05) 100%
  );
  transform: translateX(4px);
  margin-right: 0; /* hover 时恢复正常边距 */
}

.movies li:last-child {
  border-bottom: none;
}

.movies li input[type="checkbox"] {
  margin-right: 16px;
  transform: scale(1.2);
  accent-color: #6366f1;
}

.movies li .icon {
  margin-right: 12px;
  font-size: 1.5em;
  filter: drop-shadow(0 1px 2px rgba(0, 0, 0, 0.1));
}

.movies li .file-name {
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  flex-grow: 1;
}

.movies li .file-info {
  margin-left: auto;
  color: #64748b;
  font-size: 12px;
  font-weight: 500;
  padding: 4px 8px;
  background: rgba(248, 250, 252, 0.8);
  border-radius: 6px;
  border: 1px solid rgba(226, 232, 240, 0.5);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  max-width: 30%;
}

/* 底部操作区 */
.footer {
  /* margin-top: 24px; */
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 16px;
  padding-top: 20px;
  border-top: 1px solid rgba(226, 232, 240, 0.6);

  /* 确保底部区域始终可见 */
  flex-shrink: 0; /* 防止被压缩 */
  margin-top: auto; /* 自动推到底部 */
}

@media (max-width: 480px) {
  .movies {
    min-height: 150px; /* 移动端适当减小最小高度 */
  }

  .footer {
    flex-direction: column;
    gap: 12px;
    padding: 16px 0 0;
  }
}

/* 现代化按钮样式 */
.btn.el-button {
  padding: 14px 28px;
  border: none;
  border-radius: 12px;
  cursor: pointer;
  font-size: 16px;
  font-weight: 600;
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  position: relative;
  overflow: hidden;
}

/* 主要按钮样式 */
.btn.el-button--primary {
  background: linear-gradient(135deg, #6366f1 0%, #8b5cf6 100%);
  color: white;
  box-shadow: 0 4px 14px 0 rgba(99, 102, 241, 0.3);
}

/* 次要按钮样式 */
.btn.el-button--secondary {
  background: linear-gradient(135deg, #f8fafc 0%, #e2e8f0 100%);
  color: #475569;
  border: 2px solid #cbd5e1;
  box-shadow: 0 2px 8px 0 rgba(71, 85, 105, 0.1);
}

.btn.el-button::before {
  content: '';
  position: absolute;
  top: 0;
  left: -100%;
  width: 100%;
  height: 100%;
  background: linear-gradient(90deg, transparent, rgba(255, 255, 255, 0.2), transparent);
  transition: left 0.5s;
}

.btn.el-button--primary:hover {
  background: linear-gradient(135deg, #5b21b6 0%, #7c3aed 100%);
  box-shadow: 0 8px 25px 0 rgba(99, 102, 241, 0.4);
  transform: translateY(-2px);
}

.btn.el-button--secondary:hover {
  background: linear-gradient(135deg, #e2e8f0 0%, #cbd5e1 100%);
  border-color: #94a3b8;
  box-shadow: 0 4px 12px 0 rgba(71, 85, 105, 0.15);
  transform: translateY(-2px);
}

.btn.el-button:hover::before {
  left: 100%;
}

.btn.el-button:active {
  transform: translateY(0);
  box-shadow: 0 4px 14px 0 rgba(99, 102, 241, 0.3);
}

@media (max-width: 480px) {
  .btn.el-button {
    width: 100%;
    padding: 16px;
    font-size: 18px;
  }
}

/* 新增：修复移动端显示器顶部工具栏布局问题 */
@media (max-width: 768px) {
  .toolbar {
    flex-direction: column; /* 工具栏整体改为垂直布局 */
    gap: 12px;
    align-items: stretch; /* 子元素拉伸填充宽度 */
  }

  .sort-options {
    display: flex;
    flex-direction: column; /* 关键：排序选项垂直排列 */
    gap: 8px; /* 增加垂直间距 */
    width: 100%; /* 确保充分利用可用宽度 */
  }

  .sort-options label[for="sort-by"] {
    margin-bottom: 4px; /* 给"排序方式"标签一点下边距 */
    font-size: 14px; /* 可选：调整字体大小适应小屏幕 */
  }

  .sort-options select {
    width: 100%; /* 下拉框宽度填满容器 */
    box-sizing: border-box; /* 确保padding和border包含在宽度内 */
  }
}

/* aria2连接状态样式 */
.connection-status {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 12px 16px;
  margin: 16px 0;
  background: #f8f9fa;
  border-radius: 8px;
  border: 1px solid #e9ecef;
}

.status-indicator {
  display: flex;
  align-items: center;
  gap: 8px;
}

.status-dot {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  transition: all 0.3s ease;
}

.status-dot.connected {
  background: #52c41a;
  box-shadow: 0 0 0 2px rgba(82, 196, 26, 0.2);
  animation: pulse 2s infinite;
}

.status-dot.disconnected {
  background: #ff4d4f;
  box-shadow: 0 0 0 2px rgba(255, 77, 79, 0.2);
}

.status-dot.testing {
  background: #1890ff;
  box-shadow: 0 0 0 2px rgba(24, 144, 255, 0.2);
  animation: pulse 1s infinite;
}

.status-dot.unknown {
  background: #d9d9d9;
}

.status-text {
  font-size: 14px;
  color: #666;
  font-weight: 500;
}

.test-btn {
  padding: 6px 12px;
  font-size: 12px;
  border: 1px solid #d9d9d9;
  border-radius: 4px;
  background: white;
  color: #666;
  cursor: pointer;
  transition: all 0.2s ease;
}

.test-btn:hover:not(:disabled) {
  border-color: #409eff;
  color: #409eff;
}

.test-btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

@keyframes pulse {
  0% {
    transform: scale(1);
    opacity: 1;
  }
  50% {
    transform: scale(1.1);
    opacity: 0.7;
  }
  100% {
    transform: scale(1);
    opacity: 1;
  }
}
</style>
