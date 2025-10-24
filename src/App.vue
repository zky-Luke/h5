<template>
  <div class="app">
    <header class="header">
      <h1>H5媒体应用</h1>
    </header>
    
    <main class="main">
      <!-- 拍照上传组件 -->
      <CameraUpload @upload-success="handleUploadSuccess" />
      
      <!-- 语音转文字组件 -->
      <VoiceToText @transcription="handleTranscription" />
      
      <!-- 结果显示区域 -->
      <div class="results">
        <div v-if="uploadedImage" class="result-item">
          <h3>上传的图片</h3>
          <img :src="uploadedImage" alt="上传的图片" class="uploaded-image" />
        </div>
        
        <div v-if="transcriptionText" class="result-item">
          <h3>语音转文字结果</h3>
          <p class="transcription-text">{{ transcriptionText }}</p>
        </div>
      </div>
    </main>
  </div>
</template>

<script>
import CameraUpload from './components/CameraUpload.vue'
import VoiceToText from './components/VoiceToText.vue'

export default {
  name: 'App',
  components: {
    CameraUpload,
    VoiceToText
  },
  data() {
    return {
      uploadedImage: null,
      transcriptionText: ''
    }
  },
  methods: {
    handleUploadSuccess(imageUrl) {
      this.uploadedImage = imageUrl
    },
    handleTranscription(text) {
      this.transcriptionText = text
    }
  }
}
</script>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  min-height: 100vh;
}

.app {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
}

.header {
  background: rgba(255, 255, 255, 0.1);
  backdrop-filter: blur(10px);
  padding: 1rem;
  text-align: center;
  color: white;
}

.header h1 {
  font-size: 1.5rem;
  font-weight: 600;
}

.main {
  flex: 1;
  padding: 2rem;
  display: flex;
  flex-direction: column;
  gap: 2rem;
}

.results {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.result-item {
  background: rgba(255, 255, 255, 0.9);
  border-radius: 12px;
  padding: 1rem;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}

.result-item h3 {
  color: #333;
  margin-bottom: 0.5rem;
  font-size: 1.1rem;
}

.uploaded-image {
  max-width: 100%;
  height: auto;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.transcription-text {
  color: #555;
  line-height: 1.6;
  font-size: 1rem;
}

@media (max-width: 768px) {
  .main {
    padding: 1rem;
  }
}
</style>
