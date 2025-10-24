<template>
  <div class="voice-to-text">
    <div class="voice-section">
      <h2>语音录制与播放</h2>
      
      <!-- 录音按钮 -->
      <button 
        @mousedown="startRecording"
        @mouseup="stopRecording"
        @mouseleave="stopRecording"
        @touchstart="startRecording"
        @touchend="stopRecording"
        class="record-btn"
        :class="{ 
          'recording': isRecording,
          'disabled': !isSupported 
        }"
        :disabled="!isSupported"
      >
        <span v-if="!isRecording">🎤 按住说话</span>
        <span v-else>🔴 录音中...</span>
      </button>
      
      <!-- 录音状态指示 -->
      <div v-if="isRecording" class="recording-indicator">
        <div class="pulse-dot"></div>
        <span>正在录音...</span>
      </div>
      
      <!-- 录音波形可视化 -->
      <div v-if="isRecording" class="audio-visualizer">
        <div 
          v-for="(bar, index) in audioBars" 
          :key="index"
          class="audio-bar"
          :style="{ height: bar + '%' }"
        ></div>
      </div>
      
      <!-- 录音时长显示 -->
      <div v-if="isRecording" class="recording-timer">
        <span class="timer-text">{{ formatTime(recordingTime) }}</span>
      </div>
      
      <!-- 播放按钮 -->
      <button 
        v-if="audioBlob && !isRecording"
        @click="togglePlayback"
        class="play-btn"
        :class="{ 'playing': isPlaying }"
      >
        <span v-if="!isPlaying">▶️ 播放录音</span>
        <span v-else>⏸️ 播放中...</span>
      </button>
      
      <!-- 播放状态指示 -->
      <div v-if="isPlaying" class="playing-indicator">
        <div class="pulse-dot"></div>
        <span>正在播放录音...</span>
      </div>
      
      <!-- 播放进度 -->
      <div v-if="isPlaying" class="playing-progress">
        <div class="progress-bar">
          <div class="progress-fill" :style="{ width: progress + '%' }"></div>
        </div>
        <span class="progress-text">{{ formatTime(currentTime) }} / {{ formatTime(duration) }}</span>
      </div>
      
      <!-- 播放波形可视化 -->
      <div v-if="isPlaying" class="audio-visualizer">
        <div 
          v-for="(bar, index) in audioBars" 
          :key="index"
          class="audio-bar"
          :style="{ height: bar + '%' }"
        ></div>
      </div>
      
      <!-- 录音控制按钮 -->
      <div v-if="audioBlob && !isRecording" class="recording-controls">
        <button @click="deleteRecording" class="delete-btn">🗑️ 删除录音</button>
        <button @click="downloadRecording" class="download-btn">💾 下载录音</button>
      </div>
      
      <!-- 状态消息 -->
      <div v-if="message" class="message" :class="messageType">
        {{ message }}
      </div>
      
      <!-- 浏览器支持提示 -->
      <div v-if="!isSupported" class="support-warning">
        <p>⚠️ 您的浏览器不支持录音功能</p>
        <p>请使用Chrome、Edge或Safari浏览器</p>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'VoiceToText',
  data() {
    return {
      // 录音相关
      isRecording: false,
      isSupported: false,
      mediaRecorder: null,
      audioBlob: null,
      audioUrl: null,
      recordingTime: 0,
      recordingTimer: null,
      
      // 播放相关
      isPlaying: false,
      audioElement: null,
      currentTime: 0,
      duration: 0,
      progress: 0,
      progressTimer: null,
      
      // 可视化相关
      audioBars: Array(20).fill(0),
      audioContext: null,
      analyser: null,
      animationId: null,
      
      // 状态相关
      message: '',
      messageType: ''
    }
  },
  mounted() {
    this.checkSupport()
  },
  beforeUnmount() {
    this.cleanup()
  },
  methods: {
    checkSupport() {
      // 检查录音支持
      this.isSupported = navigator.mediaDevices && navigator.mediaDevices.getUserMedia
    },
    
    async startRecording() {
      if (!this.isSupported || this.isRecording) return
      
      try {
        // 请求麦克风权限
        const stream = await navigator.mediaDevices.getUserMedia({ 
          audio: {
            echoCancellation: true,
            noiseSuppression: true,
            sampleRate: 44100
          } 
        })
        
        // 创建 MediaRecorder
        this.mediaRecorder = new MediaRecorder(stream, {
          mimeType: 'audio/webm;codecs=opus'
        })
        
        const chunks = []
        
        this.mediaRecorder.ondataavailable = (event) => {
          if (event.data.size > 0) {
            chunks.push(event.data)
          }
        }
        
        this.mediaRecorder.onstop = () => {
          this.audioBlob = new Blob(chunks, { type: 'audio/webm' })
          this.audioUrl = URL.createObjectURL(this.audioBlob)
          this.showMessage('录音完成', 'success')
          
          // 停止所有音频轨道
          stream.getTracks().forEach(track => track.stop())
        }
        
        // 开始录音
        this.mediaRecorder.start()
        this.isRecording = true
        this.recordingTime = 0
        
        // 开始计时
        this.recordingTimer = setInterval(() => {
          this.recordingTime++
        }, 1000)
        
        // 开始音频可视化
        this.startAudioVisualization()
        
        this.showMessage('开始录音...', 'info')
        
      } catch (error) {
        console.error('录音失败:', error)
        this.showMessage('无法访问麦克风，请检查权限设置', 'error')
      }
    },
    
    stopRecording() {
      if (!this.isRecording || !this.mediaRecorder) return
      
      this.mediaRecorder.stop()
      this.isRecording = false
      
      // 停止计时
      if (this.recordingTimer) {
        clearInterval(this.recordingTimer)
        this.recordingTimer = null
      }
      
      // 停止音频可视化
      this.stopAudioVisualization()
      
      this.showMessage('录音已停止', 'info')
    },
    
    togglePlayback() {
      if (!this.audioBlob) return
      
      if (this.isPlaying) {
        this.stopPlayback()
      } else {
        this.startPlayback()
      }
    },
    
    startPlayback() {
      if (this.isPlaying || !this.audioUrl) return
      
      try {
        // 创建音频元素
        this.audioElement = new Audio(this.audioUrl)
        
        this.audioElement.onloadedmetadata = () => {
          this.duration = Math.floor(this.audioElement.duration)
        }
        
        this.audioElement.ontimeupdate = () => {
          this.currentTime = Math.floor(this.audioElement.currentTime)
          this.progress = (this.currentTime / this.duration) * 100
        }
        
        this.audioElement.onended = () => {
          this.isPlaying = false
          this.stopAudioVisualization()
          this.stopProgressTracking()
          this.showMessage('播放完成', 'success')
        }
        
        this.audioElement.onerror = (error) => {
          console.error('播放错误:', error)
          this.showMessage('播放失败，请重试', 'error')
          this.isPlaying = false
          this.stopAudioVisualization()
          this.stopProgressTracking()
        }
        
        // 开始播放
        this.audioElement.play()
        this.isPlaying = true
        this.startAudioVisualization()
        this.startProgressTracking()
        
        this.showMessage('开始播放录音...', 'info')
        
      } catch (error) {
        console.error('播放失败:', error)
        this.showMessage('无法播放录音，请重试', 'error')
      }
    },
    
    stopPlayback() {
      if (!this.isPlaying || !this.audioElement) return
      
      this.audioElement.pause()
      this.audioElement.currentTime = 0
      this.isPlaying = false
      this.stopAudioVisualization()
      this.stopProgressTracking()
      this.showMessage('播放已停止', 'info')
    },
    
    startProgressTracking() {
      this.currentTime = 0
      this.progress = 0
      
      this.progressTimer = setInterval(() => {
        if (this.isPlaying && this.audioElement) {
          this.currentTime = Math.floor(this.audioElement.currentTime)
          this.progress = (this.currentTime / this.duration) * 100
        }
      }, 100)
    },
    
    stopProgressTracking() {
      if (this.progressTimer) {
        clearInterval(this.progressTimer)
        this.progressTimer = null
      }
      this.currentTime = 0
      this.progress = 0
    },
    
    async startAudioVisualization() {
      try {
        // 创建音频上下文
        this.audioContext = new (window.AudioContext || window.webkitAudioContext)()
        this.analyser = this.audioContext.createAnalyser()
        this.analyser.fftSize = 256
        
        const bufferLength = this.analyser.frequencyBinCount
        const dataArray = new Uint8Array(bufferLength)
        
        const updateBars = () => {
          if (!this.isRecording && !this.isPlaying) return
          
          this.analyser.getByteFrequencyData(dataArray)
          
          // 将频率数据转换为音频条高度
          for (let i = 0; i < this.audioBars.length; i++) {
            const dataIndex = Math.floor(i * bufferLength / this.audioBars.length)
            this.audioBars[i] = (dataArray[dataIndex] / 255) * 100
          }
          
          this.animationId = requestAnimationFrame(updateBars)
        }
        
        updateBars()
        
      } catch (error) {
        console.error('音频可视化失败:', error)
        // 如果音频可视化失败，使用模拟效果
        this.startSimulatedVisualization()
      }
    },
    
    startSimulatedVisualization() {
      const updateBars = () => {
        if (!this.isRecording && !this.isPlaying) return
        
        // 随机生成音频条高度
        for (let i = 0; i < this.audioBars.length; i++) {
          this.audioBars[i] = Math.random() * 100
        }
        
        this.animationId = requestAnimationFrame(updateBars)
      }
      
      updateBars()
    },
    
    stopAudioVisualization() {
      if (this.animationId) {
        cancelAnimationFrame(this.animationId)
        this.animationId = null
      }
      
      // 重置音频条
      this.audioBars = Array(20).fill(0)
    },
    
    deleteRecording() {
      if (this.audioUrl) {
        URL.revokeObjectURL(this.audioUrl)
      }
      this.audioBlob = null
      this.audioUrl = null
      this.audioElement = null
      this.isPlaying = false
      this.stopAudioVisualization()
      this.stopProgressTracking()
      this.showMessage('录音已删除', 'info')
    },
    
    downloadRecording() {
      if (!this.audioBlob) return
      
      const url = URL.createObjectURL(this.audioBlob)
      const a = document.createElement('a')
      a.href = url
      a.download = `recording-${new Date().getTime()}.webm`
      document.body.appendChild(a)
      a.click()
      document.body.removeChild(a)
      URL.revokeObjectURL(url)
      
      this.showMessage('录音已下载', 'success')
    },
    
    formatTime(seconds) {
      const mins = Math.floor(seconds / 60)
      const secs = seconds % 60
      return `${mins.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`
    },
    
    showMessage(text, type) {
      this.message = text
      this.messageType = type
      
      setTimeout(() => {
        this.message = ''
      }, 3000)
    },
    
    cleanup() {
      this.stopRecording()
      this.stopPlayback()
      this.stopAudioVisualization()
      
      if (this.audioUrl) {
        URL.revokeObjectURL(this.audioUrl)
      }
      
      if (this.audioContext) {
        this.audioContext.close()
      }
    }
  }
}
</script>

<style scoped>
.voice-to-text {
  background: rgba(255, 255, 255, 0.9);
  border-radius: 12px;
  padding: 1.5rem;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}

.voice-section h2 {
  color: #333;
  margin-bottom: 1rem;
  font-size: 1.2rem;
  text-align: center;
}

/* 录音按钮样式 */
.record-btn {
  width: 100%;
  padding: 1.5rem;
  background: linear-gradient(135deg, #2196F3 0%, #1976D2 100%);
  color: white;
  border: none;
  border-radius: 12px;
  font-size: 1.1rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s ease;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 0.5rem;
  user-select: none;
  -webkit-user-select: none;
  -webkit-touch-callout: none;
  margin-bottom: 1rem;
}

.record-btn:hover:not(:disabled) {
  transform: translateY(-2px);
  box-shadow: 0 6px 20px rgba(33, 150, 243, 0.4);
}

.record-btn.recording {
  background: linear-gradient(135deg, #f44336 0%, #d32f2f 100%);
  animation: pulse 1.5s infinite;
}

.record-btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

/* 播放按钮样式 */
.play-btn {
  width: 100%;
  padding: 1.5rem;
  background: linear-gradient(135deg, #4CAF50 0%, #45a049 100%);
  color: white;
  border: none;
  border-radius: 12px;
  font-size: 1.1rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s ease;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 0.5rem;
  user-select: none;
  -webkit-user-select: none;
  -webkit-touch-callout: none;
  margin-bottom: 1rem;
}

.play-btn:hover:not(:disabled) {
  transform: translateY(-2px);
  box-shadow: 0 6px 20px rgba(76, 175, 80, 0.4);
}

.play-btn.playing {
  background: linear-gradient(135deg, #ff9800 0%, #f57c00 100%);
  animation: pulse 1.5s infinite;
}

@keyframes pulse {
  0% { transform: scale(1); }
  50% { transform: scale(1.05); }
  100% { transform: scale(1); }
}

/* 录音状态指示 */
.recording-indicator {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 0.5rem;
  margin-top: 1rem;
  color: #f44336;
  font-weight: 600;
}

.playing-indicator {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 0.5rem;
  margin-top: 1rem;
  color: #ff9800;
  font-weight: 600;
}

.pulse-dot {
  width: 12px;
  height: 12px;
  background: currentColor;
  border-radius: 50%;
  animation: pulse-dot 1s infinite;
}

@keyframes pulse-dot {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.3; }
}

/* 录音计时器 */
.recording-timer {
  display: flex;
  align-items: center;
  justify-content: center;
  margin-top: 1rem;
}

.timer-text {
  font-size: 1.2rem;
  font-weight: 600;
  color: #f44336;
  font-family: 'Courier New', monospace;
  background: rgba(244, 67, 54, 0.1);
  padding: 0.5rem 1rem;
  border-radius: 6px;
  border: 2px solid #f44336;
}

/* 播放进度 */
.playing-progress {
  margin-top: 1rem;
}

.progress-bar {
  width: 100%;
  height: 6px;
  background: #e0e0e0;
  border-radius: 3px;
  overflow: hidden;
  margin-bottom: 0.5rem;
}

.progress-fill {
  height: 100%;
  background: linear-gradient(90deg, #4CAF50, #8BC34A);
  border-radius: 3px;
  transition: width 0.3s ease;
}

.progress-text {
  text-align: center;
  font-size: 0.9rem;
  color: #666;
  font-family: 'Courier New', monospace;
}

/* 音频可视化 */
.audio-visualizer {
  display: flex;
  align-items: end;
  justify-content: center;
  gap: 2px;
  height: 60px;
  margin: 1rem 0;
  padding: 0 1rem;
}

.audio-bar {
  width: 4px;
  background: linear-gradient(to top, #4CAF50, #8BC34A);
  border-radius: 2px;
  transition: height 0.1s ease;
  min-height: 2px;
}

/* 录音控制按钮 */
.recording-controls {
  display: flex;
  gap: 0.5rem;
  margin-top: 1rem;
}

.delete-btn, .download-btn {
  flex: 1;
  padding: 0.75rem 1rem;
  border: none;
  border-radius: 6px;
  font-size: 0.9rem;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.3s ease;
}

.delete-btn {
  background: #f44336;
  color: white;
}

.delete-btn:hover {
  background: #d32f2f;
  transform: translateY(-1px);
}

.download-btn {
  background: #2196F3;
  color: white;
}

.download-btn:hover {
  background: #1976D2;
  transform: translateY(-1px);
}

/* 状态消息 */
.message {
  margin-top: 1rem;
  padding: 0.75rem;
  border-radius: 6px;
  text-align: center;
  font-weight: 500;
}

.message.info {
  background: #d1ecf1;
  color: #0c5460;
  border: 1px solid #bee5eb;
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

/* 浏览器支持提示 */
.support-warning {
  margin-top: 1rem;
  padding: 1rem;
  background: #fff3cd;
  border: 1px solid #ffeaa7;
  border-radius: 6px;
  text-align: center;
  color: #856404;
}

.support-warning p {
  margin: 0.25rem 0;
}

/* 响应式设计 */
@media (max-width: 768px) {
  .recording-controls {
    flex-direction: column;
  }
  
  .delete-btn, .download-btn {
    width: 100%;
  }
  
  .voice-to-text {
    padding: 1rem;
  }
  
  .record-btn, .play-btn {
    padding: 1.2rem;
    font-size: 1rem;
  }
}

/* 触摸设备优化 */
@media (hover: none) and (pointer: coarse) {
  .record-btn, .play-btn {
    padding: 1.8rem;
    font-size: 1.2rem;
  }
  
  .record-btn:active {
    transform: scale(0.98);
  }
}
</style>