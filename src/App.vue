<template>
  <div id="app">
    <h1>📸 照片墙 / 视频日记</h1>
    
    <!-- 上传区 -->
    <div class="upload-box">
      <input
        type="file"
        accept="image/*,video/mp4,video/quicktime"
        @change="handleFileSelect"
      />
      <button @click="uploadFile" :disabled="!selectedFile">上传</button>
    </div>

    <!-- 加载提示 -->
    <p v-if="loading" class="loading">加载中...</p>

    <!-- 媒体网格 -->
    <div class="grid">
      <div
        v-for="item in mediaList"
        :key="item.id"
        class="media-item"
      >
        <img
          v-if="item.file_type === 'image'"
          :src="getPublicUrl(item.file_path)"
          @click="openPreview(item)"
        />
        <video
          v-else
          :src="getPublicUrl(item.file_path)"
          controls
          @click="openPreview(item)"
        ></video>

        <button @click="deleteMedia(item.id)" class="delete-btn">×</button>
      </div>
    </div>

    <!-- 预览模态框 -->
    <div v-if="previewItem" class="modal" @click="previewItem = null">
      <div class="modal-content" @click.stop>
        <button class="close-btn" @click="previewItem = null">×</button>
        <img
          v-if="previewItem?.file_type === 'image'"
          :src="getPublicUrl(previewItem.file_path)"
        />
        <video
          v-else
          :src="getPublicUrl(previewItem.file_path)"
          controls
          autoplay
        ></video>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import { createClient } from '@supabase/supabase-js'

// 初始化 Supabase（替换为你的值）
const supabase = createClient(
  'https://bqpcvrhmxhrmpqraffdl.supabase.co', // ← 替换
  'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6ImJxcGN2cmhteGhybXBxcmFmZmRsIiwicm9sZSI6ImFub24iLCJpYXQiOjE3NjUwODI5OTMsImV4cCI6MjA4MDY1ODk5M30.YsB1P01YqJtE5G3wvFt2snFuw_3-XpT2Q1f7MJaQLzE'                    // ← 替换
)

const mediaList = ref([])
const selectedFile = ref(null)
const previewItem = ref(null)
const loading = ref(false)

// 获取所有媒体
const fetchMedia = async () => {
  loading.value = true
  const { data, error } = await supabase
    .from('media')
    .select('*')
    .order('created_at', { ascending: false })

  if (error) console.error('获取失败:', error)
  else mediaList.value = data || []
  loading.value = false
}

// 选择文件
const handleFileSelect = (e) => {
  selectedFile.value = e.target.files[0]
}

// 上传文件
const uploadFile = async () => {
  if (!selectedFile.value) return

  const file = selectedFile.value
  const fileName = `${Date.now()}-${file.name}`
  const filePath = `public/${fileName}`

  // 上传到 Storage
  const { error: uploadError } = await supabase.storage
    .from('media')
    .upload(filePath, file)

  if (uploadError) {
    alert('上传失败: ' + uploadError.message)
    return
  }

  // 判断类型
  const fileType = file.type.startsWith('image') ? 'image' : 'video'

  // 插入数据库
  const { error: dbError } = await supabase
    .from('media')
    .insert({ file_path: filePath, file_type: fileType })

  if (dbError) {
    alert('保存失败: ' + dbError.message)
  } else {
    await fetchMedia()
    selectedFile.value = null
  }
}

// 删除
const deleteMedia = async (id) => {
  if (!confirm('确定删除？')) return

  // 删除 Storage 文件
  const item = mediaList.value.find(m => m.id === id)
  await supabase.storage.from('media').remove([item.file_path])

  // 删除数据库记录
  await supabase.from('media').delete().eq('id', id)
  await fetchMedia()
}

// 工具函数
const getPublicUrl = (path) => {
  const { data } = supabase.storage.from('media').getPublicUrl(path)
  return data.publicUrl
}

const openPreview = (item) => {
  previewItem.value = item
}

// 初始化
onMounted(() => {
  fetchMedia()
})
</script>

<style>
* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, sans-serif;
  background: #f5f5f5;
  padding: 20px;
}

#app {
  max-width: 1200px;
  margin: 0 auto;
}

h1 {
  text-align: center;
  margin-bottom: 20px;
  color: #333;
}

.upload-box {
  display: flex;
  gap: 10px;
  margin-bottom: 20px;
  justify-content: center;
}

.upload-box input,
.upload-box button {
  padding: 8px 12px;
  font-size: 16px;
}

.grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(180px, 1fr));
  gap: 12px;
}

.media-item {
  position: relative;
  aspect-ratio: 1 / 1;
  background: #eee;
  border-radius: 8px;
  overflow: hidden;
  cursor: pointer;
}

.media-item img,
.media-item video {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.delete-btn {
  position: absolute;
  top: 4px;
  right: 4px;
  width: 24px;
  height: 24px;
  background: rgba(255, 0, 0, 0.7);
  color: white;
  border: none;
  border-radius: 50%;
  font-weight: bold;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
}

.loading {
  text-align: center;
  color: #666;
  margin: 20px 0;
}

.modal {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0, 0, 0, 0.9);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.modal-content {
  position: relative;
  max-width: 90vw;
  max-height: 90vh;
}

.modal-content img,
.modal-content video {
  max-width: 100%;
  max-height: 80vh;
  object-fit: contain;
}

.close-btn {
  position: absolute;
  top: -40px;
  right: 0;
  background: none;
  border: none;
  color: white;
  font-size: 32px;
  cursor: pointer;
}
</style>
