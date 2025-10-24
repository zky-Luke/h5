<template>
  <div class="camera-upload">
    <div class="upload-section">
      <h2>拍照上传</h2>
      
      <!-- 拍照按钮 -->
      <button 
        @click="openCamera" 
        class="camera-btn"
        :disabled="isUploading"
      >
        <span v-if="!isUploading">📷 拍照</span>
        <span v-else>上传中...</span>
      </button>
      
      <!-- 隐藏的文件输入 -->
      <input 
        ref="fileInput"
        type="file" 
        accept="image/*" 
        capture="camera"
        @change="handleFileSelect"
        style="display: none"
      />
      
      <!-- 预览图片 -->
      <div v-if="previewImage" class="preview-container">
        <img :src="previewImage" alt="预览图片" class="preview-image" />
        <div class="preview-actions">
          <button @click="uploadImage" class="upload-btn" :disabled="isUploading">
            {{ isUploading ? '上传中...' : '确认上传' }}
          </button>
          <button @click="cancelPreview" class="cancel-btn">取消</button>
        </div>
      </div>
      
      <!-- 上传进度 -->
      <div v-if="isUploading" class="progress-container">
        <div class="progress-bar">
          <div class="progress-fill" :style="{ width: uploadProgress + '%' }"></div>
        </div>
        <p class="progress-text">{{ uploadProgress }}%</p>
      </div>
      
      <!-- 上传状态消息 -->
      <div v-if="uploadMessage" class="message" :class="messageType">
        {{ uploadMessage }}
      </div>
    </div>
  </div>
</template>

<script>
import axios from 'axios'

export default {
  name: 'CameraUpload',
  data() {
    return {
      previewImage: null,
      isUploading: false,
      uploadProgress: 0,
      uploadMessage: '',
      messageType: ''
    }
  },
  methods: {
    openCamera() {
      this.$refs.fileInput.click()
    },
    
    handleFileSelect(event) {
      const file = event.target.files[0]
      if (file) {
        // 检查文件类型
        if (!file.type.startsWith('image/')) {
          this.showMessage('请选择图片文件', 'error')
          return
        }
        
        // 检查文件大小 (限制为5MB)
        if (file.size > 5 * 1024 * 1024) {
          this.showMessage('图片大小不能超过5MB', 'error')
          return
        }
        
        // 创建预览
        const reader = new FileReader()
        reader.onload = (e) => {
          this.previewImage = e.target.result
        }
        reader.readAsDataURL(file)
      }
    },
    
    async uploadImage() {
      if (!this.previewImage) return
      
      this.isUploading = true
      this.uploadProgress = 0
      this.uploadMessage = ''
      
      try {
        // 将base64转换为blob
        const response = await fetch(this.previewImage)
        const blob = await response.blob()
        
        // 创建FormData
        const formData = new FormData()
        formData.append('image', blob, 'camera-image.jpg')
        
        // 上传到服务器
        const uploadResponse = await axios.post('/api/upload', formData, {
          headers: {
            'Content-Type': 'multipart/form-data'
          },
          onUploadProgress: (progressEvent) => {
            this.uploadProgress = Math.round(
              (progressEvent.loaded * 100) / progressEvent.total
            )
          }
        })
        
        // 上传成功
        this.showMessage('上传成功！', 'success')
        this.$emit('upload-success', uploadResponse.data.imageUrl)
        
        // 重置状态
        setTimeout(() => {
          this.resetUpload()
        }, 2000)
        
      } catch (error) {
        console.error('上传失败:', error)
        this.showMessage('上传失败，请重试', 'error')
        this.isUploading = false
      }
    },
    
    cancelPreview() {
      this.previewImage = null
      this.$refs.fileInput.value = ''
    },
    
    resetUpload() {
      this.previewImage = null
      this.isUploading = false
      this.uploadProgress = 0
      this.uploadMessage = ''
      this.$refs.fileInput.value = ''
    },
    
    showMessage(message, type) {
      this.uploadMessage = message
      this.messageType = type
      
      // 3秒后清除消息
      setTimeout(() => {
        this.uploadMessage = ''
      }, 3000)
    }
  }
}
</script>

<style scoped>
.camera-upload {
  background: rgba(255, 255, 255, 0.9);
  border-radius: 12px;
  padding: 1.5rem;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}

.upload-section h2 {
  color: #333;
  margin-bottom: 1rem;
  font-size: 1.2rem;
  text-align: center;
}

.camera-btn {
  width: 100%;
  padding: 1rem;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s ease;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 0.5rem;
}

.camera-btn:hover:not(:disabled) {
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(102, 126, 234, 0.4);
}

.camera-btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.preview-container {
  margin-top: 1rem;
  text-align: center;
}

.preview-image {
  max-width: 100%;
  max-height: 300px;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  margin-bottom: 1rem;
}

.preview-actions {
  display: flex;
  gap: 1rem;
  justify-content: center;
}

.upload-btn, .cancel-btn {
  padding: 0.75rem 1.5rem;
  border: none;
  border-radius: 6px;
  font-size: 0.9rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s ease;
}

.upload-btn {
  background: #4CAF50;
  color: white;
}

.upload-btn:hover:not(:disabled) {
  background: #45a049;
}

.upload-btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.cancel-btn {
  background: #f44336;
  color: white;
}

.cancel-btn:hover {
  background: #da190b;
}

.progress-container {
  margin-top: 1rem;
  text-align: center;
}

.progress-bar {
  width: 100%;
  height: 8px;
  background: #e0e0e0;
  border-radius: 4px;
  overflow: hidden;
  margin-bottom: 0.5rem;
}

.progress-fill {
  height: 100%;
  background: linear-gradient(90deg, #4CAF50, #45a049);
  transition: width 0.3s ease;
}

.progress-text {
  color: #666;
  font-size: 0.9rem;
}

.message {
  margin-top: 1rem;
  padding: 0.75rem;
  border-radius: 6px;
  text-align: center;
  font-weight: 500;
}

.message.success {
  background: #d4edda;
  color: #155724;
  border: 1px solid #c3e6cb;
}

.message.error {
  background: #f8d7da;
  color: #721c24;
  border: 1px solid #f5c6cb;
}

@media (max-width: 768px) {
  .preview-actions {
    flex-direction: column;
  }
  
  .upload-btn, .cancel-btn {
    width: 100%;
  }
}
</style>
