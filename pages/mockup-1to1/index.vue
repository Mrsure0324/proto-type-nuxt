<script setup lang="ts">
import { ref, reactive, onMounted, onBeforeUnmount } from 'vue'
import { ElMessage } from 'element-plus'

const framePath = '/image/proto/default1.png' // mockup default frame
const bgPlaceholder = '/image/background/magazine.jpg' // blurred background placeholder

const file = ref<File | null>(null)
const videoUrl = ref('')
const videoEl = ref<HTMLVideoElement | null>(null)
const canvasEl = ref<HTMLCanvasElement | null>(null)
const frameImg = ref<HTMLImageElement | null>(null)

const settings = reactive({
  // output 1:1 size (px)
  outputSize: 1080,
  // screen area inside the frame (percentages of canvas)
  screen: { left: 0.06, top: 0.06, width: 0.88, height: 0.88 },
  // adjustments for the video inside the screen
  scale: 1,
  offsetX: 0,
  offsetY: 0,
  fps: 25,
  duration: 0, // 0 = use video duration (or min 3s)
  blurAmount: 18,
})

let rafId: number | null = null
const debugShowVideo = ref(false)

const handleUpload = (raw: File) => {
  if (!raw || !raw.type.startsWith('video/')) {
    ElMessage.error('请上传视频文件 (MP4/MOV 等)')
    return
  }
  file.value = raw
  if (videoUrl.value) URL.revokeObjectURL(videoUrl.value)
  videoUrl.value = URL.createObjectURL(raw)
  if (videoEl.value) {
    videoEl.value.src = videoUrl.value
    videoEl.value.load()
    // draw first frame when metadata available
    videoEl.value.addEventListener('loadedmetadata', async () => {
      console.log('video loadedmetadata', videoEl.value?.videoWidth, videoEl.value?.videoHeight, videoEl.value?.duration)
      ElMessage.info('视频已加载元数据')
      await seekAndDrawFirstFrame()
    }, { once: true })
    videoEl.value.addEventListener('error', (e) => {
      console.error('video error', e)
      ElMessage.error('视频加载出错，浏览器可能不支持该编码或存在 CORS 限制')
    })
  }
}

const seekAndDrawFirstFrame = async () => {
  if (!videoEl.value || !canvasEl.value) return
  const v = videoEl.value
  try {
    await new Promise<void>((res) => {
      if (v.readyState >= 2) res()
      else v.addEventListener('loadedmetadata', () => res(), { once: true })
    })
    v.currentTime = 0
    await new Promise<void>((res) => v.addEventListener('seeked', () => res(), { once: true }))
    const canvas = canvasEl.value
    const ctx = canvas.getContext('2d')
    if (!ctx) return
    setupCanvasSize(canvas)
    // try draw; if video dimensions are zero, wait briefly and retry
    drawFrame(ctx)
    if ((v.videoWidth || 0) === 0 || (v.videoHeight || 0) === 0) {
      await new Promise((r) => setTimeout(r, 200))
      drawFrame(ctx)
    }
  } catch (e) {
    // ignore
  }
}

const setupCanvasSize = (canvas: HTMLCanvasElement) => {
  const size = settings.outputSize
  const dpr = (window && window.devicePixelRatio) ? window.devicePixelRatio : 1
  canvas.style.width = `${size}px`
  canvas.style.height = `${size}px`
  canvas.width = Math.round(size * dpr)
  canvas.height = Math.round(size * dpr)
}

const drawFrame = (ctx: CanvasRenderingContext2D) => {
  if (!canvasEl.value) return
  const canvas = canvasEl.value
  const dpr = (window && window.devicePixelRatio) ? window.devicePixelRatio : 1
  const w = canvas.width
  const h = canvas.height
  // clear
  ctx.setTransform(1,0,0,1,0,0) // reset
  ctx.clearRect(0,0,w,h)
  // draw blurred background image scaled to cover
  const bg = new Image()
  bg.src = bgPlaceholder
  bg.onload = () => {
    ctx.save()
    ctx.filter = `blur(${settings.blurAmount}px)`
    // draw cover
    const bw = bg.naturalWidth, bh = bg.naturalHeight
    const scale = Math.max(w / bw, h / bh)
    const bwScaled = bw * scale, bhScaled = bh * scale
    ctx.drawImage(bg, (w - bwScaled)/2, (h - bhScaled)/2, bwScaled, bhScaled)
    ctx.restore()
    // draw screen mask
    const sx = Math.round(settings.screen.left * w)
    const sy = Math.round(settings.screen.top * h)
    const sw = Math.round(settings.screen.width * w)
    const sh = Math.round(settings.screen.height * h)
    // black screen area
    ctx.fillStyle = '#000'
    roundRect(ctx, sx, sy, sw, sh, Math.round(w*0.05))
    ctx.fill()
    // draw video inside screen if available
    if (videoEl.value && videoEl.value.readyState >= 2) {
      const vid = videoEl.value
      const iw = vid.videoWidth, ih = vid.videoHeight
      const fitScale = Math.max(sw / iw, sh / ih) * settings.scale
      const dw = iw * fitScale, dh = ih * fitScale
      const dx = sx + (sw - dw)/2 + settings.offsetX
      const dy = sy + (sh - dh)/2 + settings.offsetY
      try { ctx.drawImage(vid, 0,0,iw,ih, dx, dy, dw, dh) } catch(e) {}
    }
    // draw frame overlay on top if loaded
    if (frameImg.value && frameImg.value.complete) {
      try { ctx.drawImage(frameImg.value, 0, 0, w, h) } catch(e) {}
    }
  }
}

const roundRect = (ctx: CanvasRenderingContext2D, x:number,y:number,w:number,h:number,r:number) => {
  ctx.beginPath()
  ctx.moveTo(x+r,y)
  ctx.arcTo(x+w,y,x+w,y+h,r)
  ctx.arcTo(x+w,y+h,x,y+h,r)
  ctx.arcTo(x,y+h,x,y,r)
  ctx.arcTo(x,y,x+w,y,r)
  ctx.closePath()
}

const playPreview = async () => {
  if (!videoEl.value || !canvasEl.value) return
  try {
    await videoEl.value.play()
  } catch (e) {
    // autoplay may be blocked
  }
  const canvas = canvasEl.value
  const ctx = canvas.getContext('2d')
  if (!ctx) return
  // ensure canvas size
  setupCanvasSize(canvas)
  const loop = () => {
    drawFrame(ctx)
    rafId = requestAnimationFrame(loop)
  }
  rafId = requestAnimationFrame(loop)
}

const stopPreview = () => {
  if (videoEl.value) {
    videoEl.value.pause()
    videoEl.value.currentTime = 0
  }
  if (rafId) { cancelAnimationFrame(rafId); rafId = null }
}

const record = async () => {
  if (!file.value || !videoEl.value || !canvasEl.value) {
    ElMessage.error('请先上传视频')
    return
  }
  processing.value = true
  try {
    const canvas = canvasEl.value
    const ctx = canvas.getContext('2d')
    if (!ctx) throw new Error('Canvas not supported')
    setupCanvasSize(canvas)
    // recorder
    const stream = canvas.captureStream(settings.fps)
    const chunks: Blob[] = []
    const mime = MediaRecorder.isTypeSupported('video/webm;codecs=vp9') ? 'video/webm;codecs=vp9' : 'video/webm'
    const recorder = new MediaRecorder(stream, { mimeType: mime })
    recorder.ondataavailable = (e) => { if (e.data && e.data.size) chunks.push(e.data) }
    recorder.start(100)
    // play video
    const vid = videoEl.value
    vid.currentTime = 0
    vid.muted = true
    await vid.play().catch(()=>{})
    // draw loop using fixed interval to match fps
    const frameInterval = 1000 / Math.max(1, settings.fps)
    let elapsed = 0
    const duration = settings.duration > 0 ? settings.duration : (vid.duration || 3)
    await new Promise<void>((resolve) => {
      const id = setInterval(() => {
        drawFrame(ctx)
        elapsed += frameInterval/1000
        if (elapsed >= duration || vid.ended) {
          clearInterval(id)
          resolve()
        }
      }, frameInterval)
    })
    recorder.stop()
    await new Promise<void>((res) => recorder.addEventListener('stop', () => res(), { once: true }))
    const blob = new Blob(chunks, { type: mime })
    const url = URL.createObjectURL(blob)
    const a = document.createElement('a')
    a.href = url
    a.download = `mockup-1to1-${Date.now()}.webm`
    a.click()
    setTimeout(() => URL.revokeObjectURL(url), 200)
    ElMessage.success('导出完成')
    vid.pause()
    vid.currentTime = 0
  } catch (e: any) {
    console.error(e)
    ElMessage.error('导出失败：' + (e?.message || e))
  } finally {
    processing.value = false
  }
}

onMounted(() => {
  const img = new Image()
  img.src = framePath
  img.onload = () => { frameImg.value = img }
})

onBeforeUnmount(() => {
  if (rafId) cancelAnimationFrame(rafId)
  if (videoUrl.value) URL.revokeObjectURL(videoUrl.value)
})
</script>

<template>
  <div style="padding:16px;max-width:900px;margin:0 auto;">
    <div style="display:flex;gap:12px;align-items:center;margin-bottom:12px;">
      <input type="file" accept="video/*" @change="(e) => handleUpload((e.target as HTMLInputElement).files?.[0] as File)" />
      <el-button type="primary" :loading="processing" @click="record">导出 1:1 展示视频</el-button>
      <el-button @click="playPreview">播放预览</el-button>
      <el-button @click="stopPreview">停止</el-button>
      <el-checkbox v-model="debugShowVideo" style="margin-left:12px;">显示 video 元素（调试）</el-checkbox>
    </div>

    <div style="display:flex;gap:16px;flex-wrap:wrap;">
      <div style="flex:1;min-width:260px;">
        <div style="background:#fff;padding:12px;border-radius:8px;">
          <div style="font-weight:600;margin-bottom:8px;">设置</div>
          <div style="margin-bottom:8px;">输出尺寸：{{ settings.outputSize }}px (1:1)</div>
          <el-select v-model="settings.outputSize" style="width:160px;">
            <el-option :value="540" label="540"></el-option>
            <el-option :value="720" label="720"></el-option>
            <el-option :value="1080" label="1080"></el-option>
          </el-select>
          <div style="margin-top:12px;">屏幕区域（百分比）</div>
          <div style="display:flex;gap:8px;margin-top:8px;">
            <div style="flex:1"><div>left</div><el-slider v-model="settings.screen.left" :min="0" :max="0.4" :step="0.01" /></div>
            <div style="flex:1"><div>top</div><el-slider v-model="settings.screen.top" :min="0" :max="0.4" :step="0.01" /></div>
          </div>
          <div style="display:flex;gap:8px;margin-top:8px;">
            <div style="flex:1"><div>width</div><el-slider v-model="settings.screen.width" :min="0.5" :max="0.95" :step="0.01" /></div>
            <div style="flex:1"><div>height</div><el-slider v-model="settings.screen.height" :min="0.5" :max="0.95" :step="0.01" /></div>
          </div>
          <div style="margin-top:12px;">视频缩放与位移</div>
          <div style="display:flex;gap:8px;margin-top:8px;">
            <div style="flex:1"><div>缩放</div><el-slider v-model="settings.scale" :min="0.5" :max="2" :step="0.01" /></div>
          </div>
          <div style="display:flex;gap:8px;margin-top:8px;">
            <div style="flex:1"><div>offsetX</div><el-slider v-model="settings.offsetX" :min="-500" :max="500" /></div>
            <div style="flex:1"><div>offsetY</div><el-slider v-model="settings.offsetY" :min="-500" :max="500" /></div>
          </div>
          <div style="margin-top:12px;">
            <div>模糊背景强度</div>
            <el-slider v-model="settings.blurAmount" :min="0" :max="40" />
          </div>
        </div>
      </div>

      <div style="flex:1;min-width:300px;">
        <div style="background:#fff;padding:12px;border-radius:8px;">
          <div style="font-weight:600;margin-bottom:8px;">预览 (1:1)</div>
          <div style="width:100%;display:flex;justify-content:center;">
            <div style="width:360px;height:360px;border-radius:24px;overflow:hidden;background:#111;display:flex;align-items:center;justify-content:center;">
              <canvas ref="canvasEl" style="width:100%;height:100%;display:block;"></canvas>
              <video ref="videoEl" :src="videoUrl" :style="{ display: debugShowVideo ? 'block' : 'none', width: '100%' }" controls></video>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.el-select { margin-top: 6px; }
</style>


