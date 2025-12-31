<script setup lang="ts">
import { ref, reactive, onMounted } from 'vue'
import { ElMessage } from 'element-plus'
import { UploadFilled, VideoPlay, Download } from '@element-plus/icons-vue'

type UploadFile = { name: string; raw: File }

const mode = ref<'images' | 'video'>('images')
const fileList = ref<UploadFile[]>([])
const processing = ref(false)
const previewUrl = ref('')

const settings = reactive({
  // images mode
  durationPerImage: 2, // seconds
  transition: 'pan', // 'pan' | 'zoom' | 'none'
  // video mode
  startTime: 0,
  duration: 5,
  fps: 25,
  outputScale: 1, // 1 = source size
})

const canvasRef = ref<HTMLCanvasElement | null>(null)

const handleFiles = (files: File[]) => {
  if (!files || !files.length) return
  const f = files[0]
  if (f.type.startsWith('image/')) {
    mode.value = 'images'
    fileList.value = Array.from(files).map((file) => ({ name: file.name, raw: file }))
    previewUrl.value = URL.createObjectURL(fileList.value[0].raw)
  } else if (f.type.startsWith('video/')) {
    mode.value = 'video'
    fileList.value = [{ name: f.name, raw: f }]
    previewUrl.value = URL.createObjectURL(f)
  } else {
    ElMessage.error('仅支持图片或视频文件')
  }
}

// wrapper for upload change event to normalize files
const onUploadChange = (file: any, fileListNew: any) => {
  const arr = (fileListNew || []).map((f: any) => f.raw || f.file || f)
  handleFiles(arr)
}

const clearAll = () => {
  fileList.value = []
  previewUrl.value && URL.revokeObjectURL(previewUrl.value)
  previewUrl.value = ''
}

// 动画实现：针对图片序列的 Ken Burns 风格，和视频的画布捕获
const generateVideo = async () => {
  if (!fileList.value.length) {
    ElMessage.warning('请先上传图片或视频')
    return
  }
  if (!canvasRef.value) {
    ElMessage.error('Canvas 未准备好')
    return
  }

  processing.value = true
  try {
    const canvas = canvasRef.value
    const ctx = canvas.getContext('2d')
    if (!ctx) throw new Error('Canvas not supported')

    // 设置输出尺寸（以第一张图片或视频分辨率为基础）
    let outWidth = 1080
    let outHeight = 1920

    if (mode.value === 'images') {
      // 读取第一张图片以确定尺寸
      const img = await loadImage(fileList.value[0].raw)
      outWidth = img.width * settings.outputScale
      outHeight = img.height * settings.outputScale
      canvas.width = outWidth
      canvas.height = outHeight

      const stream = canvas.captureStream(settings.fps)
      const chunks: Blob[] = []
      let mime = 'video/webm'
      if (MediaRecorder.isTypeSupported('video/webm;codecs=vp9')) {
        mime = 'video/webm;codecs=vp9'
      } else if (MediaRecorder.isTypeSupported('video/webm;codecs=vp8')) {
        mime = 'video/webm;codecs=vp8'
      }

      const recorder = new MediaRecorder(stream, { mimeType: mime })
      recorder.ondataavailable = (e) => { if (e.data && e.data.size) chunks.push(e.data) }
      recorder.start()

      // 动画循环每张图片
      for (const file of fileList.value) {
        const img = await loadImage(file.raw)
        const frames = Math.max(1, Math.round(settings.durationPerImage * settings.fps))
        for (let f = 0; f < frames; f++) {
          // 计算动画参数（简单线性 pan/zoom）
          ctx.clearRect(0, 0, canvas.width, canvas.height)
          const t = f / frames
          const sx = settings.transition === 'zoom' ? 1 + 0.1 * t : 1
          const sy = sx
          const dx = (canvas.width - img.width * sx) / 2
          const dy = (canvas.height - img.height * sy) / 2
          ctx.save()
          ctx.translate(dx, dy)
          ctx.scale(sx, sy)
          ctx.drawImage(img, 0, 0)
          ctx.restore()
          await waitNextFrame()
        }
      }

      // stop and assemble
      recorder.stop()
      await waitForRecorderStop(recorder)
      const blob = new Blob(chunks, { type: mime })
      downloadBlob(blob, `dynamic-wallpaper-${Date.now()}.webm`)
      ElMessage.success('视频生成完成')
    } else {
      // video mode: draw video frames to canvas and record (trim)
      const vFile = fileList.value[0].raw
      const video = document.createElement('video')
      video.src = URL.createObjectURL(vFile)
      video.muted = true
      video.playsInline = true
      await new Promise((r) => video.addEventListener('loadedmetadata', r, { once: true }))
      outWidth = video.videoWidth * settings.outputScale
      outHeight = video.videoHeight * settings.outputScale
      canvas.width = outWidth
      canvas.height = outHeight

      const stream = canvas.captureStream(settings.fps)
      const chunks: Blob[] = []
      const mime = MediaRecorder.isTypeSupported('video/webm;codecs=vp9') ? 'video/webm;codecs=vp9' : 'video/webm'
      const recorder = new MediaRecorder(stream, { mimeType: mime })
      recorder.ondataavailable = (e) => { if (e.data && e.data.size) chunks.push(e.data) }
      recorder.start(100)

      // play trimmed segment
      video.currentTime = settings.startTime
      video.play().catch(() => {})
      const endTime = settings.startTime + settings.duration
      // draw loop
      await new Promise<void>((resolve) => {
        const draw = () => {
          if (video.ended || video.currentTime >= endTime) {
            video.pause()
            resolve()
            return
          }
          ctx.clearRect(0, 0, canvas.width, canvas.height)
          ctx.drawImage(video, 0, 0, canvas.width, canvas.height)
          requestAnimationFrame(draw)
        }
        draw()
      })

      recorder.stop()
      await waitForRecorderStop(recorder)
      const blob = new Blob(chunks, { type: mime })
      downloadBlob(blob, `dynamic-wallpaper-${Date.now()}.webm`)
      URL.revokeObjectURL(video.src)
      ElMessage.success('视频生成完成')
    }
  } catch (e: any) {
    console.error(e)
    ElMessage.error('生成失败：' + (e?.message || e))
  } finally {
    processing.value = false
  }
}

// helpers
const loadImage = (file: File): Promise<HTMLImageElement> => {
  return new Promise((resolve, reject) => {
    const url = URL.createObjectURL(file)
    const img = new Image()
    img.onload = () => { URL.revokeObjectURL(url); resolve(img) }
    img.onerror = (e) => { URL.revokeObjectURL(url); reject(e) }
    img.src = url
  })
}

const waitNextFrame = () => new Promise((r) => requestAnimationFrame(() => r(null)))

const waitForRecorderStop = (rec: MediaRecorder) => {
  return new Promise<void>((resolve) => {
    rec.addEventListener('stop', () => resolve(), { once: true })
  })
}

const downloadBlob = (blob: Blob, filename: string) => {
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  a.download = filename
  a.click()
  setTimeout(() => URL.revokeObjectURL(url), 200)
}
</script>

<template>
  <div class="page" style="padding:16px;">
    <div class="container" style="max-width:1000px;margin:0 auto;">
      <div class="controls" style="display:flex;gap:12px;flex-wrap:wrap;">
        <el-segmented v-model="mode" :options="[{label:'图片序列',value:'images'},{label:'视频',value:'video'}]" />
        <el-upload
          class="upload"
          drag
          :auto-upload="false"
          :show-file-list="false"
          accept="image/*,video/*"
          multiple
          :on-change="onUploadChange"
        >
          <el-icon><UploadFilled /></el-icon>
          <div>拖拽或点击上传图片序列或视频</div>
        </el-upload>
        <el-button type="warning" @click="clearAll">清空</el-button>
      </div>

      <div style="display:flex;gap:16px;margin-top:16px;flex-wrap:wrap;">
        <div style="flex:1;min-width:260px;">
          <div style="background:#fff;padding:12px;border-radius:8px;">
            <div style="font-weight:600;margin-bottom:8px;">设置</div>
            <div v-if="mode==='images'">
              <div style="margin-bottom:8px;">每张时长（秒）</div>
              <el-slider v-model="settings.durationPerImage" :min="0.5" :max="10" :step="0.1" />
              <div style="margin-top:8px;">过渡</div>
              <el-radio-group v-model="settings.transition">
                <el-radio-button label="pan">平移</el-radio-button>
                <el-radio-button label="zoom">缩放</el-radio-button>
                <el-radio-button label="none">无</el-radio-button>
              </el-radio-group>
            </div>
            <div v-else>
              <div style="margin-bottom:8px;">开始时间（秒）</div>
              <el-input-number v-model="settings.startTime" :min="0" :step="0.5" />
              <div style="margin-top:8px;margin-bottom:8px;">时长（秒）</div>
              <el-input-number v-model="settings.duration" :min="1" :step="0.5" />
            </div>
            <div style="margin-top:12px;">
              <div>帧率 (FPS)</div>
              <el-slider v-model="settings.fps" :min="10" :max="30" :step="1" />
            </div>
            <div style="margin-top:12px;">
              <el-switch v-model="settings.outputScale" active-text="高清" inactive-text="原始" />
            </div>
            <div style="margin-top:12px;">
              <el-button type="primary" :loading="processing" @click="generateVideo">
                <el-icon><VideoPlay /></el-icon> 生成动态壁纸视频
              </el-button>
            </div>
          </div>
        </div>

        <div style="flex:1;min-width:300px;">
          <div style="background:#fff;padding:12px;border-radius:8px;">
            <div style="font-weight:600;margin-bottom:8px;">预览</div>
            <div v-if="previewUrl" style="margin-bottom:8px;">
              <template v-if="mode==='video'">
                <video :src="previewUrl" controls playsinline webkit-playsinline style="width:100%;border-radius:8px;"></video>
              </template>
              <template v-else>
                <img :src="previewUrl" style="width:100%;border-radius:8px;" />
              </template>
            </div>
            <div style="margin-top:8px;">
              <canvas ref="canvasRef" style="width:100%;background:#000;border-radius:8px;"></canvas>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.upload { display:flex; align-items:center; gap:8px; padding:12px; background:#fff; border-radius:8px; }
@media (max-width:768px){
  .controls { flex-direction:column; }
}
</style>


