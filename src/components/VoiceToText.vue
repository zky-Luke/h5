<template>
  <div class="voice-to-text">
    <div class="voice-section">
      <h2>语音转文字</h2>
      
      <!-- 录音按钮 -->
      <button 
        @mousedown="startRecording"
        @mouseup="stopRecording"
        @touchstart="startRecording"
        @touchend="stopRecording"
        class="voice-btn"
        :class="{ 'recording': isRecording }"
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
      
      <!-- 录音时长 -->
      <div v-if="isRecording" class="recording-time">
        {{ formatTime(recordingTime) }}
      </div>
      
      <!-- 音频波形可视化 -->
      <div v-if="isRecording" class="audio-visualizer">
        <div 
          v-for="(bar, index) in audioBars" 
          :key="index"
          class="audio-bar"
          :style="{ height: bar + '%' }"
        ></div>
      </div>
      
      <!-- 转录结果 -->
      <div v-if="transcription" class="transcription-result">
        <h3>转录结果：</h3>
        <p class="transcription-text">{{ transcription }}</p>
        <div class="transcription-actions">
          <button @click="copyText" class="copy-btn">复制文本</button>
          <button @click="clearTranscription" class="clear-btn">清除</button>
        </div>
      </div>
      
      <!-- 状态消息 -->
      <div v-if="message" class="message" :class="messageType">
        {{ message }}
      </div>
      
      <!-- 浏览器支持提示 -->
      <div v-if="!isSupported" class="support-warning">
        <p>⚠️ 您的浏览器不支持语音识别功能</p>
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
      isRecording: false,
      isSupported: false,
      mediaRecorder: null,
      audioChunks: [],
      recordingTime: 0,
      recordingTimer: null,
      audioBars: Array(20).fill(0),
      audioContext: null,
      analyser: null,
      animationId: null,
      transcription: '',
      message: '',
      messageType: '',
      recognition: null
    }
  },
  mounted() {
    this.checkSupport()
    this.initSpeechRecognition()
  },
  beforeUnmount() {
    this.cleanup()
  },
  methods: {
    checkSupport() {
      // 检查语音识别支持
      this.isSupported = 'webkitSpeechRecognition' in window || 'SpeechRecognition' in window
    },
    
    initSpeechRecognition() {
      if (!this.isSupported) return
      
      const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition
      this.recognition = new SpeechRecognition()
      
      this.recognition.continuous = false
      this.recognition.interimResults = true
      this.recognition.lang = 'zh-CN'
      
      this.recognition.onstart = () => {
        console.log('语音识别开始')
      }
      
      this.recognition.onresult = (event) => {
        let finalTranscript = ''
        let interimTranscript = ''
        
        for (let i = event.resultIndex; i < event.results.length; i++) {
          const transcript = event.results[i][0].transcript
          if (event.results[i].isFinal) {
            finalTranscript += transcript
          } else {
            interimTranscript += transcript
          }
        }
        
        if (finalTranscript) {
          this.transcription = finalTranscript
          this.$emit('transcription', finalTranscript)
        }
      }
      
      this.recognition.onerror = (event) => {
        console.error('语音识别错误:', event.error)
        this.showMessage('语音识别出错，请重试', 'error')
        this.isRecording = false
      }
      
      this.recognition.onend = () => {
        this.isRecording = false
        this.stopAudioVisualization()
      }
    },
    
    async startRecording() {
      if (!this.isSupported || this.isRecording) return
      
      try {
        this.isRecording = true
        this.recordingTime = 0
        this.transcription = ''
        
        // 开始录音计时
        this.recordingTimer = setInterval(() => {
          this.recordingTime++
        }, 1000)
        
        // 开始语音识别
        this.recognition.start()
        
        // 开始音频可视化
        this.startAudioVisualization()
        
        this.showMessage('开始录音...', 'info')
        
      } catch (error) {
        console.error('开始录音失败:', error)
        this.showMessage('无法开始录音，请检查麦克风权限', 'error')
        this.isRecording = false
      }
    },
    
    stopRecording() {
      if (!this.isRecording) return
      
      this.isRecording = false
      
      // 停止录音计时
      if (this.recordingTimer) {
        clearInterval(this.recordingTimer)
        this.recordingTimer = null
      }
      
      // 停止语音识别
      if (this.recognition) {
        this.recognition.stop()
      }
      
      // 停止音频可视化
      this.stopAudioVisualization()
      
      this.showMessage('录音结束，正在处理...', 'info')
    },
    
    async startAudioVisualization() {
      try {
        const stream = await navigator.mediaDevices.getUserMedia({ audio: true })
        this.audioContext = new (window.AudioContext || window.webkitAudioContext)()
        this.analyser = this.audioContext.createAnalyser()
        const source = this.audioContext.createMediaStreamSource(stream)
        source.connect(this.analyser)
        
        this.analyser.fftSize = 32
        const bufferLength = this.analyser.frequencyBinCount
        const dataArray = new Uint8Array(bufferLength)
        
        const updateBars = () => {
          if (!this.isRecording) return
          
          this.analyser.getByteFrequencyData(dataArray)
          
          // 更新音频条
          for (let i = 0; i < this.audioBars.length; i++) {
            const value = dataArray[Math.floor(i * bufferLength / this.audioBars.length)]
            this.audioBars[i] = (value / 255) * 100
          }
          
          this.animationId = requestAnimationFrame(updateBars)
        }
        
        updateBars()
        
      } catch (error) {
        console.error('音频可视化失败:', error)
      }
    },
    
    stopAudioVisualization() {
      if (this.animationId) {
        cancelAnimationFrame(this.animationId)
        this.animationId = null
      }
      
      if (this.audioContext) {
        this.audioContext.close()
        this.audioContext = null
      }
      
      // 重置音频条
      this.audioBars = Array(20).fill(0)
    },
    
    formatTime(seconds) {
      const mins = Math.floor(seconds / 60)
      const secs = seconds % 60
      return `${mins.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`
    },
    
    copyText() {
      if (this.transcription) {
        navigator.clipboard.writeText(this.transcription).then(() => {
          this.showMessage('文本已复制到剪贴板', 'success')
        }).catch(() => {
          this.showMessage('复制失败', 'error')
        })
      }
    },
    
    clearTranscription() {
      this.transcription = ''
      this.message = ''
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
      this.stopAudioVisualization()
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

.voice-btn {
  width: 100%;
  padding: 1.5rem;
  background: linear-gradient(135deg, #ff6b6b 0%, #ee5a24 100%);
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
}

.voice-btn:hover:not(:disabled) {
  transform: translateY(-2px);
  box-shadow: 0 6px 20px rgba(255, 107, 107, 0.4);
}

.voice-btn.recording {
  background: linear-gradient(135deg, #ff4757 0%, #c44569 100%);
  animation: pulse 1.5s infinite;
}

.voice-btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

@keyframes pulse {
  0% { transform: scale(1); }
  50% { transform: scale(1.05); }
  100% { transform: scale(1); }
}

.recording-indicator {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 0.5rem;
  margin-top: 1rem;
  color: #ff4757;
  font-weight: 600;
}

.pulse-dot {
  width: 12px;
  height: 12px;
  background: #ff4757;
  border-radius: 50%;
  animation: pulse-dot 1s infinite;
}

@keyframes pulse-dot {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.3; }
}

.recording-time {
  text-align: center;
  margin-top: 0.5rem;
  font-size: 1.2rem;
  font-weight: 600;
  color: #333;
  font-family: 'Courier New', monospace;
}

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
  background: linear-gradient(to top, #ff6b6b, #ffa726);
  border-radius: 2px;
  transition: height 0.1s ease;
  min-height: 2px;
}

.transcription-result {
  margin-top: 1rem;
  padding: 1rem;
  background: #f8f9fa;
  border-radius: 8px;
  border-left: 4px solid #4CAF50;
}

.transcription-result h3 {
  color: #333;
  margin-bottom: 0.5rem;
  font-size: 1rem;
}

.transcription-text {
  color: #555;
  line-height: 1.6;
  margin-bottom: 1rem;
  font-size: 1rem;
}

.transcription-actions {
  display: flex;
  gap: 0.5rem;
}

.copy-btn, .clear-btn {
  padding: 0.5rem 1rem;
  border: none;
  border-radius: 6px;
  font-size: 0.9rem;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.3s ease;
}

.copy-btn {
  background: #4CAF50;
  color: white;
}

.copy-btn:hover {
  background: #45a049;
}

.clear-btn {
  background: #f44336;
  color: white;
}

.clear-btn:hover {
  background: #da190b;
}

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

@media (max-width: 768px) {
  .transcription-actions {
    flex-direction: column;
  }
  
  .copy-btn, .clear-btn {
    width: 100%;
  }
}
</style>
