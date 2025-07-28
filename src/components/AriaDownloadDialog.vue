<template>
  <div style="width: 60%" v-if="show" class="dialog">
    <h2>请勾选你要下载的</h2>
    <div class="close" @click="close">×</div>
    <div class="toolbar">
      <input @change="onCheckAll" style="margin: 10px 10px 0 0" type="checkbox" id="checkbox" v-model="checkedAll">全选
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
        <span>{{ item.name }}</span>
        <span class="file-info">{{ formatFileInfo(item) }}</span>
      </li>
    </ul>
    <div class="footer">
      <div class="btn el-button el-button--primary" @click="pushBefore">推送到aria2</div>
    </div>
  </div>
</template>

<script setup>
import { ref, watch } from 'vue'
import { getDownload, pushToAria, getList } from '../api'

const props = defineProps({
  show: Boolean
})
const emits = defineEmits(['update:show', 'msg'])

const list = ref([])
const selected = ref([])
const checkedAll = ref(false)
const selectedItems = ref([]) // 选中的项目
const isForbidden = ref(false) // 按钮是否禁用，防抖

const sortBy = ref('name') // Default sort by name
const sortDirection = ref('asc') // Default sort direction

watch(
  () => props.show,
  (val) => {
    if (val) {
      const tempList = []
      let parent_id = window.location.href.split('/').pop()
      if (parent_id == 'all') parent_id = ''
      emits('msg', '开始加载文件列表，请稍等')
      getList(parent_id).then(res => {
        res.files.forEach(item => {
          tempList.push({ id: item.id, name: item.name, type: item.kind, created_time: item.created_time, modified_time: item.modified_time,size: item.size,file_category: item.file_category })
        })
        list.value = tempList
        sortList(); // Apply default sort after loading
      })
    }
  }
)


const close = () => {
  selected.value = []
  checkedAll.value = false
  isForbidden.value = false
  emits('update:show', false)
}

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
  emits('msg', '开始获取文件内容');
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
    emits('msg', `正在扫描第 ${processedCount} 个文件夹: ${currentFolder.name}`);
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
      emits('msg', `获取文件夹 ${currentFolder.name} 内容失败`);
      console.error(e);
    }
  }

  selectedItems.value = allFiles;
  emits('msg', `文件获取完毕，共${allFiles.length}个文件。`);
}

const pushBefore = async () => {
  if (!isForbidden.value) {
    isForbidden.value = true
    await getAllList()
    push()
  } else {
    emits('msg', '已经开始推送了')
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
    emits('msg', '请先配置aria2')
    close()
    return
  }
  console.log(`共${selectedItems.value.length}个项目`);
  let testIndex = 0
  for (let item of selectedItems.value) {
    getDownload(item.id).then((res) => {

      if (res.error_description) {
        emits('msg', `失败原因: ${res.error_description} 请刷新！`)
        return
      }
      emits('msg', `第${testIndex + 1}个项目下载链接获取成功`)
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
        if (ariares.result) {
          success++
        } else {
          console.log(ariares);
          console.log(ariaData);
          errorMSG = ariares.error.message === 'Unauthorized' ? '密钥不对' : '推送失败'
          fail++
        }
      }).catch((e) => {
        console.log(ariares);
        console.log(ariaData);
        errorMSG = `${e.statusText} 请检测配置`
        emits('msg', `失败原因: ${e.statusText}`)
        fail++
      }).finally(() => {
        total--
        if (total === 0) {
          emits('msg', `成功：${success} 失败: ${fail} ${fail !== 0 ? '失败原因' + errorMSG : ''}`)
          console.info(`成功：${success} 失败: ${fail} ${fail !== 0 ? '失败原因' + errorMSG : ''}`);
          if (retryList.length > 0) {
            console.log(retryList);
            emits('msg', `即将重试${retryList.length}个项目`)
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
  max-width: 700px;
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

.toolbar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 15px;
  padding-bottom: 10px;
  border-bottom: 1px solid #eee;
}

.toolbar input[type="checkbox"] {
  margin-right: 8px;
  transform: scale(1.2);
}

.sort-options button {
  margin-left: 10px;
  padding: 8px 15px;
  border: 1px solid #dcdfe6;
  background-color: #f4f4f5;
  color: #606266;
  cursor: pointer;
  border-radius: 4px;
  transition: all 0.3s ease;
}

.sort-options button:hover {
  background-color: #e9e9eb;
  border-color: #d3d4d6;
  color: #303133;
}

.movies {
  margin-top: 10px;
  height: 400px;
  overflow-y: auto;
  border: 1px solid #ebeef5;
  border-radius: 4px;
  padding: 10px;
  background-color: #fdfdfd;
}

.movies li {
  display: flex;
  align-items: center;
  padding: 8px 0;
  border-bottom: 1px dashed #f0f0f0;
  font-size: 14px;
  color: #303133;
}

.movies li .file-info {
  margin-left: auto;
  color: #606266;
  font-size: 12px;
}

.movies li:last-child {
  border-bottom: none;
}

.movies li input[type="checkbox"] {
  margin-right: 10px;
  transform: scale(1.1);
}

.movies li .icon {
  margin-right: 8px;
  font-size: 1.2em;
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
</style>
