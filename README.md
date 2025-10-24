# H5媒体应用

一个基于Vue3的H5应用，支持拍照上传和语音转文字功能。

## 功能特性

### 📷 拍照上传
- 支持调用设备摄像头拍照
- 实时图片预览
- 图片上传到服务器
- 上传进度显示
- 文件大小限制（5MB）
- 支持常见图片格式

### 🎤 语音转文字
- 按住说话进行录音
- 实时语音识别转文字
- 音频波形可视化
- 录音时长显示
- 支持中文语音识别
- 文本复制功能

## 技术栈

- **前端框架**: Vue 3
- **构建工具**: Vite
- **HTTP客户端**: Axios
- **语音识别**: Web Speech API
- **样式**: 原生CSS + 渐变设计

## 快速开始

### 安装依赖

```bash
npm install
```

### 开发模式

```bash
npm run dev
```

应用将在 `http://localhost:3000` 启动

### 构建生产版本

```bash
npm run build
```

### 预览生产版本

```bash
npm run preview
```

## 浏览器支持

### 拍照功能
- 所有现代浏览器
- 需要HTTPS环境（生产环境）

### 语音转文字
- Chrome 25+
- Edge 79+
- Safari 14.1+
- Firefox 不支持（使用Web Speech API）

## 使用说明

### 拍照上传
1. 点击"📷 拍照"按钮
2. 允许浏览器访问摄像头
3. 拍照后预览图片
4. 点击"确认上传"上传到服务器
5. 查看上传结果

### 语音转文字
1. 按住"🎤 按住说话"按钮
2. 开始说话（支持中文）
3. 松开按钮结束录音
4. 查看转录结果
5. 可以复制文本或清除结果

## 服务器端配置

### 图片上传接口

需要配置服务器端接口 `/api/upload` 来处理图片上传：

```javascript
// 示例：Express.js 服务器
const multer = require('multer');
const upload = multer({ dest: 'uploads/' });

app.post('/api/upload', upload.single('image'), (req, res) => {
  // 处理上传的图片
  const imageUrl = `/uploads/${req.file.filename}`;
  res.json({ imageUrl });
});
```

## 注意事项

1. **HTTPS要求**: 摄像头和麦克风功能需要HTTPS环境
2. **权限授权**: 首次使用需要授权摄像头和麦克风权限
3. **浏览器兼容性**: 语音识别功能需要支持的浏览器
4. **文件大小**: 图片上传限制为5MB
5. **网络环境**: 需要稳定的网络连接进行上传

## 开发说明

### 项目结构

```
src/
├── components/
│   ├── CameraUpload.vue    # 拍照上传组件
│   └── VoiceToText.vue     # 语音转文字组件
├── App.vue                 # 主应用组件
└── main.js                 # 应用入口
```

### 自定义配置

- 修改上传接口地址：在 `CameraUpload.vue` 中修改 `/api/upload`
- 调整文件大小限制：修改 `handleFileSelect` 方法中的大小检查
- 更改语音识别语言：修改 `recognition.lang` 属性

## 许可证

MIT License
