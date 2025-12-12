<script setup lang="ts">
import { ref, reactive, watch } from 'vue'
import { ElMessage } from 'element-plus'
import { UploadFilled } from '@element-plus/icons-vue'

type UploadFile = {
  name: string
  url: string
  raw: File
}

const fileList = ref<UploadFile[]>([])
const processing = ref(false)
const previewLoading = ref(false)
const lastSize = ref<{ w: number; h: number } | null>(null)
const currentIndex = ref(0)
const previewCache = ref<Record<string, { url: string; width: number; height: number }>>({})

const watermark = reactive({
  text: '像素潮汐  |  PIXEL TIDE  |  DESIGN STUDIO',
  fontSize: 100,
  color: '#ffffff',
  opacity: 0.6,
  angle: -30,
  gapX: 220,
  gapY: 130,
  tile: false,
  offsetX: 0,
  offsetY: -250,
  // 导出设置
  outputScale: 2, // 0.5倍导出
  limit1080p: true, // 限制在1080p以内
})

const previewUrl = ref('')

const handleChange = (uploadFile: any) => {
  const raw = uploadFile.raw as File
  if (!raw) return
  const url = URL.createObjectURL(raw)
  fileList.value = [{ name: raw.name, url, raw }]
  previewUrl.value = ''
  currentIndex.value = 0
  previewCache.value = {}
  updatePreviewDebounced()
}

const handleMultiple = (uploadFiles: any[]) => {
  fileList.value = uploadFiles.map((f) => ({
    name: f.name,
    url: URL.createObjectURL(f.raw),
    raw: f.raw,
  }))
  previewUrl.value = ''
  currentIndex.value = 0
  previewCache.value = {}
  updatePreviewDebounced()
}

const loadImage = (file: File): Promise<HTMLImageElement> => {
  return new Promise((resolve, reject) => {
    const img = new Image()
    img.crossOrigin = 'anonymous'
    img.onload = () => resolve(img)
    img.onerror = (e) => reject(e)
    img.src = URL.createObjectURL(file)
  })
}

const drawWatermark = async (
  file: File
): Promise<{ blob: Blob; width: number; height: number }> => {
  const img = await loadImage(file)
  const w = img.naturalWidth
  const h = img.naturalHeight
  const baseScale = watermark.outputScale || 1
  const limitScale = watermark.limit1080p ? Math.min(1920 / w, 1080 / h, 1) : 1
  const scale = Math.max(baseScale * limitScale, 0.05)
  const targetW = Math.max(1, Math.round(w * scale))
  const targetH = Math.max(1, Math.round(h * scale))

  const canvas = document.createElement('canvas')
  canvas.width = targetW
  canvas.height = targetH
  const ctx = canvas.getContext('2d')
  if (!ctx) throw new Error('Canvas not supported')

  ctx.drawImage(img, 0, 0, targetW, targetH)

  ctx.save()
  ctx.globalAlpha = watermark.opacity
  ctx.fillStyle = watermark.color
  const fontPx = watermark.fontSize * scale
  ctx.font = `${fontPx}px sans-serif`
  ctx.textAlign = 'center'
  ctx.textBaseline = 'middle'

  if (watermark.tile) {
    // 平铺水印：以画布中心为原点旋转，然后平铺
    ctx.translate(targetW / 2, targetH / 2)
    ctx.rotate((watermark.angle * Math.PI) / 180)
    ctx.translate(-targetW / 2, -targetH / 2)

    const gapX = watermark.gapX * scale
    const gapY = watermark.gapY * scale
    const startX = -targetW
    const endX = targetW * 2
    const startY = -targetH
    const endY = targetH * 2
    for (let x = startX; x <= endX; x += gapX) {
      for (let y = startY; y <= endY; y += gapY) {
        ctx.fillText(watermark.text, x, y)
      }
    }
  } else {
    // 单行水印：默认靠下居中，可微调偏移
    const padding = 60 * scale
    const x = targetW / 2 + watermark.offsetX * scale
    const y = targetH - padding + watermark.offsetY * scale
    ctx.fillText(watermark.text, x, y)
  }

  ctx.restore()

  return new Promise((resolve, reject) => {
    canvas.toBlob((blob) => {
      if (!blob) return reject(new Error('导出失败'))
      resolve({ blob, width: targetW, height: targetH })
    }, 'image/png')
  })
}

const debounce = (fn: (...args: any[]) => void, delay = 300) => {
  let timer: any
  return (...args: any[]) => {
    clearTimeout(timer)
    timer = setTimeout(() => fn(...args), delay)
  }
}

const updatePreview = async () => {
  if (!fileList.value.length) {
    previewUrl.value = ''
    lastSize.value = null
    return
  }
  previewLoading.value = true
  try {
    const target = fileList.value[currentIndex.value] || fileList.value[0]
    const cacheKey = target.name
    if (previewCache.value[cacheKey]) {
      previewUrl.value = previewCache.value[cacheKey].url
      lastSize.value = {
        w: previewCache.value[cacheKey].width,
        h: previewCache.value[cacheKey].height,
      }
    } else {
      const { blob, width, height } = await drawWatermark(target.raw)
      const url = URL.createObjectURL(blob)
      previewCache.value[cacheKey] = { url, width, height }
      previewUrl.value = url
      lastSize.value = { w: width, h: height }
    }
  } catch (e) {
    console.error(e)
    ElMessage.error('预览生成失败')
  } finally {
    previewLoading.value = false
  }
}

const updatePreviewDebounced = debounce(updatePreview, 300)

const downloadBlob = (blob: Blob, name: string) => {
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  a.download = name.endsWith('.png') ? name : `${name}.png`
  a.click()
  URL.revokeObjectURL(url)
}

const preview = async () => {
  if (!fileList.value.length) return ElMessage.warning('请先上传图片')
  await updatePreview()
  ElMessage.success('预览生成完成')
}

const clearAll = () => {
  fileList.value = []
  previewUrl.value = ''
  lastSize.value = null
  previewCache.value = {}
  currentIndex.value = 0
  ElMessage.success('已清空图片')
}

watch(
  () => ({ ...watermark }),
  () => updatePreviewDebounced(),
  { deep: true }
)

const batchExport = async () => {
  if (!fileList.value.length) return ElMessage.warning('请先上传图片')
  processing.value = true
  try {
    for (const f of fileList.value) {
      const { blob } = await drawWatermark(f.raw)
      downloadBlob(blob, `watermark-${f.name}`)
    }
    ElMessage.success('批量导出完成')
  } catch (e) {
    console.error(e)
    ElMessage.error('导出失败')
  } finally {
    processing.value = false
  }
}
</script>

<template>
  <div class="page">
    <div class="panel">
      <div class="left">
        <div class="section-title">上传图片</div>
        <el-upload
          class="upload-block"
          drag
          :auto-upload="false"
          :show-file-list="false"
          accept="image/*"
          :limit="1"
          :on-change="handleChange"
        >
          <el-icon class="el-icon--upload"><upload-filled /></el-icon>
          <div class="el-upload__text">拖拽或点击上传单张预览</div>
        </el-upload>

        <div class="section-title">批量上传</div>
        <el-upload
          class="upload-block"
          drag
          multiple
          :auto-upload="false"
          :show-file-list="false"
          accept="image/*"
          :on-change="(_, files) => handleMultiple(files)"
        >
          <el-icon class="el-icon--upload"><upload-filled /></el-icon>
          <div class="el-upload__text">拖拽或点击上传多张图片</div>
        </el-upload>

        <div class="section-title">水印设置</div>
        <div class="form">
          <div class="form-item">
            <div class="label">文本</div>
            <el-input v-model="watermark.text" placeholder="请输入水印文本" />
          </div>
          <div class="form-item">
            <div class="label">模式</div>
            <el-switch v-model="watermark.tile" active-text="平铺" inactive-text="单行" />
          </div>
          <div class="form-item two-col">
            <div>
              <div class="label">字号</div>
              <el-slider v-model="watermark.fontSize" :min="10" :max="200" />
            </div>
            <div>
              <div class="label">透明度</div>
              <el-slider v-model="watermark.opacity" :min="0.05" :max="1" :step="0.05" />
            </div>
          </div>
          <template v-if="watermark.tile">
            <div class="form-item two-col">
              <div>
                <div class="label">间距 X</div>
                <el-slider v-model="watermark.gapX" :min="40" :max="800" />
              </div>
              <div>
                <div class="label">间距 Y</div>
                <el-slider v-model="watermark.gapY" :min="40" :max="800" />
              </div>
            </div>
            <div class="form-item two-col">
              <div>
                <div class="label">角度</div>
                <el-slider v-model="watermark.angle" :min="-180" :max="180" />
              </div>
              <div>
                <div class="label">颜色</div>
                <el-color-picker v-model="watermark.color" show-alpha />
              </div>
            </div>
          </template>
          <template v-else>
            <div class="form-item two-col">
              <div>
                <div class="label">偏移 X</div>
                <el-slider v-model="watermark.offsetX" :min="-800" :max="800" />
              </div>
              <div>
                <div class="label">偏移 Y</div>
                <el-slider v-model="watermark.offsetY" :min="-800" :max="800" />
              </div>
            </div>
            <div class="form-item two-col">
              <div>
                <div class="label">颜色</div>
                <el-color-picker v-model="watermark.color" show-alpha />
              </div>
              <div>
                <div class="label">角度（单行不旋转，保留设置用于切换平铺）</div>
                <el-slider v-model="watermark.angle" :min="-90" :max="90" />
              </div>
            </div>
          </template>

          <div class="section-title export-title">导出设置</div>
          <div class="form-item">
            <div class="label">缩放倍数（相对原图）</div>
            <el-slider v-model="watermark.outputScale" :min="0.1" :max="3" :step="0.05" />
          </div>
          <div class="form-item">
            <el-switch v-model="watermark.limit1080p" active-text="限制在 1080p 以内" inactive-text="不限制" />
            <div class="hint">开启后会按最长边不超过 1080p（约 1920×1080）再乘以缩放倍数</div>
          </div>
          <div class="export-size" v-if="lastSize">
            导出分辨率：{{ lastSize.w }} × {{ lastSize.h }}
          </div>
        </div>

        <div class="actions">
          <el-button type="primary" :loading="processing" @click="preview">生成预览</el-button>
          <el-button type="success" :loading="processing" @click="batchExport">批量导出</el-button>
          <el-button type="warning" plain @click="clearAll">清空图片</el-button>
        </div>
      </div>

      <div class="right">
        <div class="section-title">预览</div>
        <div class="preview">
          <div v-if="previewUrl" class="preview-img">
            <img :src="previewUrl" alt="preview" />
            <div class="preview-nav" v-if="fileList.length > 1">
              <el-button size="small" @click="currentIndex = (currentIndex - 1 + fileList.length) % fileList.length; updatePreview()">
                上一张
              </el-button>
              <div class="preview-counter">{{ currentIndex + 1 }} / {{ fileList.length }}</div>
              <el-button size="small" @click="currentIndex = (currentIndex + 1) % fileList.length; updatePreview()">
                下一张
              </el-button>
            </div>
          </div>
          <div v-else class="placeholder">请上传图片并点击生成预览</div>
        </div>
        <div class="list" v-if="fileList.length > 0">
          <div class="list-title">已选文件（{{ fileList.length }}）</div>
          <div class="list-items">
            <div
              v-for="(item, idx) in fileList"
              :key="item.url"
              class="file-item"
              :class="{ active: idx === currentIndex }"
              @click="currentIndex = idx; updatePreview()"
            >
              <div class="name">{{ item.name }}</div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.page {
  display: flex;
  justify-content: center;
  padding: 24px;
}

.panel {
  width: 1200px;
  display: grid;
  grid-template-columns: 420px 1fr;
  gap: 24px;
}

.left,
.right {
  background: #fff;
  border-radius: 16px;
  padding: 20px;
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.06);
  box-sizing: border-box;
}

.section-title {
  font-weight: 700;
  margin: 12px 0 8px;
}

.upload-block {
  width: 100%;
}

.form {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.form-item {
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.two-col {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 12px;
}

.label {
  font-size: 13px;
  color: #606266;
}

.hint {
  margin-top: 4px;
  font-size: 12px;
  color: #909399;
}

.export-title {
  margin-top: 6px;
}

.export-size {
  margin-top: 4px;
  font-size: 13px;
  color: #303133;
}

.actions {
  margin-top: 12px;
  display: flex;
  gap: 12px;
}

.preview {
  border: 1px dashed #e5e7eb;
  border-radius: 12px;
  min-height: 320px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: #fafafa;
}

.preview-img {
  max-width: 100%;
  max-height: 480px;
}

.preview-img img {
  max-width: 100%;
  max-height: 480px;
  display: block;
  border-radius: 12px;
}

.placeholder {
  color: #909399;
}

.list {
  margin-top: 16px;
}

.list-title {
  font-weight: 600;
  margin-bottom: 8px;
}

.list-items {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
}

.file-item {
  padding: 6px 10px;
  border-radius: 8px;
  background: #f2f3f5;
  font-size: 13px;
}

.file-item {
  cursor: pointer;
  border: 1px solid transparent;
}

.file-item.active {
  border-color: #409eff;
  background: #ecf5ff;
}

.preview-nav {
  margin-top: 8px;
  display: flex;
  align-items: center;
  gap: 8px;
  justify-content: center;
}

.preview-counter {
  font-size: 12px;
  color: #606266;
}
</style>

