<script setup>
import { ref } from 'vue'
import axios from 'axios'
import Clipboard from 'clipboard'

const files = ref([])
const fileInput = ref()

// 上传文件
const API_URL = 'http://localhost:3000/upload'
const uploadFiles = async () => {
  if (!files.value.length) return
  const formData = new FormData()
  files.value.forEach((item) => {
    if (!item.uploaded || !item.loading) {
      formData.append('files', item.file)
    }
  })
  const { data } = await axios.post(API_URL, formData)
  if (!data || data.code !== 200) {
    // 上传失败
    return
  }
  data.data.forEach((item, index) => {
    files.value[index] = {
      ...files.value[index],
      url: item.url,
      loading: false,
      uploaded: false
    }
  })

  // 解决上传后会闪现一下复制链接按钮
  const timer = setTimeout(() => {
    files.value = data.data.map((item, index) => {
      return {
        ...files.value[index],
        url: item.url,
        loading: false,
        uploaded: true
      }
    })
    clearTimeout(timer)
  }, 300)
}

// 打开文件输入框
const openFileInput = () => {
  fileInput.value && fileInput.value.click()
}

// 监听文件输入框变换 -> 上传文件
const handleFileInputChange = (e) => {
  for (const file of e.target.files) {
    const blobUrl = URL.createObjectURL(file)
    files.value.unshift({
      url: blobUrl,
      name: file.name,
      file,
      copyTips: '点击复制链接',
      loading: true,
      uploaded: false
    })
  }
  uploadFiles(e.target.files)
  fileInput.value.value = null
}

// 粘贴监听 -> 上传文件
const handlePaste = (event) => {
  let items = (event.clipboardData || event.originalEvent.clipboardData).items
  for (const item of items) {
    if (item.kind === 'file' && /image/.test(item.type)) {
      const file = item.getAsFile();
      // 非文件不处理
      if (!file) return
      const blobUrl = URL.createObjectURL(file)
      files.value.unshift({
        url: blobUrl,
        name: file.name,
        file,
        copyTips: '点击复制链接',
        loading: true,
        uploaded: false
      })
    }
  }
  if (!files.value.length) return
  // 上传文件
  uploadFiles()
}

// 托放
const handleDrop = (e) => {
  e.preventDefault()
  for (const file of e.dataTransfer.files) {
    if (/^image/.test(file.type)) {
      const blobUrl = URL.createObjectURL(file)
      files.value.unshift({
        url: blobUrl,
        name: file.name,
        file,
        copyTips: '点击复制链接',
        loading: true,
        uploaded: false
      })
    }
  }
  uploadFiles(e.target.files)
}

const handleDragOver = (e) => {
  e.preventDefault()
}

// 复制链接
const copyUrl = async (index, text, e) => {
  handleClipboard(text, e)
  files.value[index].copyTips = '链接已复制'
}

// 重置复制链接
const handleHoverIn = (index) => {
  files.value[index].copyTips = '点击复制链接'
}

// 复制
function handleClipboard(text, event) {
  const clipboard = new Clipboard(event.target, {
    text: () => text
  })
  
  clipboard.onClick(event)
  return new Promise((resolve, reject) => {
    clipboard.on('success', () => {
      clipboard.destroy()
      resolve(true)
    })
    clipboard.on('error', (e) => {
      clipboard.destroy()
      reject(e)
    })
  })
}
</script>

<template>
  <div class="container" @paste="handlePaste">
    <div
      class="uploader"
      @drop="handleDrop"
      @dragover="handleDragOver"
    >
      <!-- <img src="/upload.svg" alt=""> -->
      <svg t="1710830947689" class="icon" viewBox="0 0 1024 1024" version="1.1" xmlns="http://www.w3.org/2000/svg" p-id="10184" xmlns:xlink="http://www.w3.org/1999/xlink"><path d="M782.116 391.047a235.947 235.947 0 0 0-38.066 3.087c-48.551-101.58-152.204-171.759-272.3-171.759-156.248 0-284.699 118.785-300.099 270.937-0.491 0-0.984-0.028-1.528-0.028-90.986 0-164.769 73.755-164.769 164.739 0 78.289 54.614 143.798 127.85 160.563h275.032V614.359h-84.432L512 382.119 700.251 614.36h-84.433v204.227h305.835c58.763-43.034 96.992-112.557 96.992-190.98 0.001-130.636-105.894-236.56-236.529-236.56z" fill="currentColor" p-id="10185"></path></svg>
      <div>您可以将图像拖拽至此处</div>
    </div>
    <div class="uploader-tips">
      或者您也可以
      <div class="btn" @click="openFileInput()">
        选择本地图片上传
      </div>
      或者按下
      <div class="btn">Ctrl</div>
      +
      <div class="btn">V</div>
    </div>
    <ul class="previewer">
      <li
        v-for="(item, index) in files"
        :key="index"
        :class="{ 'is-loading': item.loading }"
      >
        <div class="previewer-inner">
          <header></header>
          <img :src="item.url" :alt="item.name">
          <p>{{ item.name }}</p>
        </div>
        <div
          class="previewer-item-mask"
          @mouseenter="handleHoverIn(index)"
          @click="copyUrl(index, item.url, $event)"
        >
          <img v-show="item.loading" src="/loading.svg" class="icon-loading">
          <a v-show="item.uploaded" class="btn">{{ item.copyTips }}</a>
        </div>
      </li>
    </ul>
    <input
      multiple
      ref="fileInput"
      type="file"
      style="display: none"
      accept="image/*"
      @change="handleFileInputChange"
    />
  </div>
</template>

<style scoped>
.container {
  width: 100%;
  min-height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-direction: column;
}
/* 上传区域 */
.uploader {
  border: var(--border-style);
  min-height: 300px;
  width: calc(100% - 24px);
  max-width: 800px;
  border-radius: 6px;
  cursor: pointer;
  transition: var(--transition);
  display: flex;
  align-items: center;
  justify-content: center;
  flex-direction: column;
  color: var(--border-color);
}
.uploader svg {
  width: 200px;
  transition: var(--transition);
}
.uploader:hover {
  color: #535bf2;
}

.uploader:hover {
  border-color: #535bf2;
}

.uploader-tips {
  margin: 24px 12px;
}

/* 预览区域 */
.previewer {
  padding: 0;
  margin: 0;
  display: flex;
  flex-wrap: wrap;
  width: 100%;
}
.previewer li {
  list-style: none;
  width: calc(100% / 6);
  padding: 12px;
  position: relative;
  z-index: 0;
  transition: var(--transition);
}
.previewer li:hover .previewer-item-mask {
  opacity: 1;
}
.previewer li.is-loading .previewer-item-mask {
  opacity: 1;
}
.previewer-item-mask {
  position: absolute;
  top: 50%;
  left: 50%;
  z-index: 1;
  background-color: rgba(0, 0, 0, .3);
  width: calc(100% - 24px);
  height: calc(100% - 24px);
  transform: translate(-50%, -50%);
  border-radius: 6px;
  display: flex;
  align-items: center;
  justify-content: center;
  opacity: 0;
  transition: var(--transition);
  cursor: pointer;
}
.icon-loading {
  width: 50px;
  height: 50px;
  display: block;
  z-index: 2;
  animation: loading 1s linear infinite;
}
.previewer li:hover .previewer-inner {
  border-color: #535bf2;
}
.previewer-inner {
  height: 100%;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  border: 1px solid #ccc;
  border-radius: 6px;
  overflow: hidden;
  transition: var(--transition);
}
.previewer header {
  height: 0;
  visibility: hidden;
}
.previewer img {
  display: block;
  width: 100%;
  margin-bottom: 6px;
  transform: translateY(-1px);
}
.previewer p {
  word-break: break-all;
  padding: 6px;
  font-size: 12px;
}

.btn {
  background-color: #535bf2;
  color: white;
  padding: 6px 8px;
  border-radius: 6px;
  user-select: none;
  cursor: pointer;
  display: inline-block;
  min-width: 40px;
  text-align: center;
}

/* 加载动画 */
@keyframes loading {
  0% {
    transform: rotate(0deg);
  }
  100% {
    transform: rotate(360deg);
  }
}
@media screen and (max-width: 1200px) {
  .previewer li {
    width: calc(100% / 4);
  }
}
@media screen and (max-width: 768px) {
  .uploader {
    min-height: 200px;
  }
  .uploader svg {
    width: 120px;
  }
  .previewer li {
    width: calc(100% / 2);
  }
}
</style>
