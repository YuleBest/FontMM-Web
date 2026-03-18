<script setup lang="ts">
import { ref, reactive, onMounted, computed } from 'vue'
import JSZip from 'jszip'
import { Button } from '@/components/ui/button'
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from '@/components/ui/card'
import { Progress } from '@/components/ui/progress'
import { Badge } from '@/components/ui/badge'
import {
  Download,
  Upload,
  FileText,
  Package,
  CheckCircle2,
  XCircle,
  AlertTriangle,
  Info,
  X,
  RotateCcw,
  Type,
  Trash2,
  HardDriveUpload,
  ShieldCheck,
  Loader2,
  Code,
  FormInput,
} from 'lucide-vue-next'

// --- 配置常量 ---
const TEMPLATE_VERSION = 'V1.3-51a0be'
const TEMPLATE_HASH = '51a0be' // 模板文件 SHA-256 前缀
const TEMPLATE_FILE = `https://fontmm.files.21acg.cn/template-${TEMPLATE_VERSION}.zip`
const TEMPLATE_FILENAME = `template-${TEMPLATE_VERSION}.zip`
const COMP_LEVEL = 9

// --- 状态管理 ---
const isProcessing = ref(false)
const packProgress = ref(0)
const isVerifying = ref(false)
const templateDragOver = ref(false)

// Toast 通知系统
interface ToastMessage {
  id: number
  type: string
  message: string
  leaving: boolean
}
let toastIdCounter = 0
const toastMessages = ref<Array<ToastMessage>>([])

const addToast = (type: string, message: string) => {
  const id = ++toastIdCounter
  toastMessages.value.push({ id, type, message, leaving: false })
  // 最多保留 4 条
  if (toastMessages.value.length > 4) {
    const firstToast = toastMessages.value[0]
    if (firstToast) removeToast(firstToast.id)
  }
  // 自动消失
  setTimeout(() => removeToast(id), 4000)
}

const removeToast = (id: number) => {
  const idx = toastMessages.value.findIndex((t) => t.id === id)
  if (idx === -1) return
  const toast = toastMessages.value[idx]
  if (toast) toast.leaving = true
  setTimeout(() => {
    toastMessages.value = toastMessages.value.filter((t) => t.id !== id)
  }, 300)
}

// 文件上传状态
const uploadedFiles = reactive({
  ch: null as File | null,
  tc: null as File | null,
  en: null as File | null,
})

// 拖动状态
const dragStates = reactive({
  ch: false,
  tc: false,
  en: false,
})

// 字体预览状态
const fontPreviewNames = reactive({
  ch: '',
  tc: '',
  en: '',
})

const loadFontForPreview = async (type: 'ch' | 'tc' | 'en', file: File) => {
  try {
    const fontName = `PreviewFont-${type}-${Date.now()}`
    const reader = new FileReader()
    reader.onload = async (e) => {
      const arrayBuffer = e.target?.result as ArrayBuffer
      if (!arrayBuffer) return

      const fontFace = new FontFace(fontName, arrayBuffer)
      await fontFace.load()
      document.fonts.add(fontFace)
      fontPreviewNames[type] = fontName
    }
    reader.readAsArrayBuffer(file)
  } catch (error) {
    console.error(`加载预览字体 ${type} 失败:`, error)
  }
}

const cleanupFontPreview = (type: 'ch' | 'tc' | 'en') => {
  fontPreviewNames[type] = ''
}

// module.prop 结构化编辑
interface ModulePropFields {
  id: string
  name: string
  version: string
  versionCode: string
  author: string
  description: string
}

const modulePropFields = reactive<ModulePropFields>({
  id: 'MyFontModule',
  name: '我的字体模块',
  version: '1.0',
  versionCode: '10',
  author: 'Yule',
  description: '我的字体模块',
})

// 序列化为 key=value 格式
const modulePropContent = computed(() => {
  return [
    `id=${modulePropFields.id}`,
    `name=${modulePropFields.name}`,
    `version=${modulePropFields.version}`,
    `versionCode=${modulePropFields.versionCode}`,
    `author=${modulePropFields.author}`,
    `description=${modulePropFields.description}`,
  ].join('\n')
})

// 源代码模式
const sourceMode = ref(false)
const rawModuleProp = ref('')

const toggleSourceMode = () => {
  if (sourceMode.value) {
    // 从源代码切回表单：解析 textarea 内容回写到 fields
    parseModuleProp(rawModuleProp.value, true)
    sourceMode.value = false
  } else {
    // 从表单切到源代码：序列化 fields 到 textarea
    rawModuleProp.value = modulePropContent.value
    sourceMode.value = true
  }
}

const templateZip = ref<JSZip | null>(null)
const originalTemplateData = ref<ArrayBuffer | null>(null)

// 打包步骤状态
const packSteps = computed(() => [
  { label: '加载模板', done: packProgress.value >= 20 },
  { label: '添加字体', done: packProgress.value >= 50 },
  { label: '写入配置', done: packProgress.value >= 80 },
  { label: '压缩打包', done: packProgress.value >= 100 },
])

// --- 辅助函数 ---
const formatFileSize = (bytes: number): string => {
  if (bytes < 1024) return bytes + ' B'
  if (bytes < 1024 * 1024) return (bytes / 1024).toFixed(2) + ' KB'
  return (bytes / (1024 * 1024)).toFixed(2) + ' MB'
}

// --- Hash 校验 ---
const computeHash = async (buffer: ArrayBuffer): Promise<string> => {
  const hashBuffer = await crypto.subtle.digest('SHA-256', buffer)
  const hashArray = Array.from(new Uint8Array(hashBuffer))
  return hashArray.map((b) => b.toString(16).padStart(2, '0')).join('')
}

const verifyTemplateHash = async (buffer: ArrayBuffer): Promise<boolean> => {
  const hash = await computeHash(buffer)
  const matches = hash.startsWith(TEMPLATE_HASH)
  if (!matches) {
    console.warn(`Hash 不匹配: 期望前缀 ${TEMPLATE_HASH}, 实际 ${hash.substring(0, 12)}...`)
  }
  return matches
}

// --- 模板下载功能（下载到用户磁盘）---
const downloadTemplate = () => {
  try {
    addToast('info', '开始下载模板...')

    // 直接使用浏览器原生行为下载，避免 CORS 和大文件内存占用
    const a = document.createElement('a')
    a.href = TEMPLATE_FILE
    a.target = '_blank'
    a.download = TEMPLATE_FILENAME
    document.body.appendChild(a)
    a.click()
    document.body.removeChild(a)

    setTimeout(() => {
      addToast('success', '如果下载完成，请在下方拖入该压缩包')
    }, 1500)
  } catch (error: unknown) {
    const msg = error instanceof Error ? error.message : String(error)
    addToast('error', `触发下载失败：${msg}`)
  }
}

// --- 模板上传功能（从用户磁盘加载 + hash 校验）---
const loadTemplateFromFile = async (file: File) => {
  try {
    if (!file.name.endsWith('.zip')) {
      addToast('error', '请上传 .zip 格式的模板文件')
      return
    }

    isVerifying.value = true
    addToast('info', '正在校验模板文件...')

    const arrayBuffer = await file.arrayBuffer()

    // 校验 hash
    const valid = await verifyTemplateHash(arrayBuffer)
    if (!valid) {
      isVerifying.value = false
      addToast('error', `模板 Hash 校验失败，请确认文件为 ${TEMPLATE_FILENAME}`)
      return
    }

    originalTemplateData.value = arrayBuffer
    templateZip.value = await JSZip.loadAsync(arrayBuffer)

    addToast('success', `模板加载成功 (${formatFileSize(file.size)})`)

    await extractModuleProp()
    isVerifying.value = false
  } catch (error: unknown) {
    const msg = error instanceof Error ? error.message : String(error)
    addToast('error', `模板加载失败：${msg}`)
    isVerifying.value = false
  }
}

const handleTemplateUpload = (event: Event) => {
  const target = event.target as HTMLInputElement
  const file = target.files?.[0]
  if (file) loadTemplateFromFile(file)
}

const handleTemplateDrop = (event: DragEvent) => {
  event.preventDefault()
  templateDragOver.value = false
  const file = event.dataTransfer?.files[0]
  if (file) loadTemplateFromFile(file)
}

const triggerTemplateInput = () => {
  const input = document.getElementById('file-template') as HTMLInputElement
  input?.click()
}

const extractModuleProp = async () => {
  if (!templateZip.value) return

  try {
    let modulePropFile = templateZip.value.file('module.prop')

    if (!modulePropFile) {
      modulePropFile = templateZip.value.file('module.prop.txt')
    }

    if (!modulePropFile) {
      templateZip.value.forEach((relativePath: string, file: JSZip.JSZipObject) => {
        if (file.name === 'module.prop' && !modulePropFile) {
          modulePropFile = file
        }
      })
    }

    if (modulePropFile) {
      const content = await modulePropFile.async('string')
      parseModuleProp(content)
      addToast('info', '已加载 module.prop 内容')
    } else {
      addToast('warn', '模板中未找到 module.prop，使用默认值')
      resetModuleProp()
    }
  } catch (error: unknown) {
    const msg = error instanceof Error ? error.message : String(error)
    addToast('error', `读取 module.prop 失败：${msg}`)
    resetModuleProp()
  }
}

const parseModuleProp = (content: string, clearFirst = false) => {
  if (clearFirst) {
    Object.keys(modulePropFields).forEach((key) => {
      ;(modulePropFields as Record<string, string>)[key] = ''
    })
  }
  const lines = content.split('\n')
  for (const line of lines) {
    const trimmed = line.trim()
    if (!trimmed || trimmed.startsWith('#')) continue
    const eqIdx = trimmed.indexOf('=')
    if (eqIdx === -1) continue
    const key = trimmed.substring(0, eqIdx).trim()
    const value = trimmed.substring(eqIdx + 1).trim()
    if (key in modulePropFields) {
      ;(modulePropFields as Record<string, string>)[key] = value
    }
  }
}

const resetModuleProp = () => {
  modulePropFields.id = 'MyFontModule'
  modulePropFields.name = '我的字体模块'
  modulePropFields.version = '1.0'
  modulePropFields.versionCode = '10'
  modulePropFields.author = 'Yule'
  modulePropFields.description = '我的字体模块'
  addToast('info', '已重置为默认值')
}

// --- 文件上传处理 ---
const handleFileUpload = async (event: Event, type: 'ch' | 'tc' | 'en') => {
  const target = event.target as HTMLInputElement
  const file = target.files?.[0]
  if (file) {
    uploadedFiles[type] = file
    addToast('success', `已上传 ${getTypeName(type)} (${formatFileSize(file.size)})`)
    await loadFontForPreview(type, file)
  }
}

const handleDrop = async (event: DragEvent, type: 'ch' | 'tc' | 'en') => {
  event.preventDefault()
  dragStates[type] = false
  const file = event.dataTransfer?.files[0]
  if (file && (file.name.endsWith('.ttf') || file.name.endsWith('.otf'))) {
    uploadedFiles[type] = file
    addToast('success', `已上传 ${getTypeName(type)} (${formatFileSize(file.size)})`)
    await loadFontForPreview(type, file)
  } else {
    addToast('error', '请拖入 .ttf 或 .otf 格式的字体文件')
  }
}

const handleDragOver = (event: DragEvent, type: 'ch' | 'tc' | 'en') => {
  event.preventDefault()
  dragStates[type] = true
}

const handleDragLeave = (type: 'ch' | 'tc' | 'en') => {
  dragStates[type] = false
}

const removeFile = (type: 'ch' | 'tc' | 'en') => {
  uploadedFiles[type] = null
  cleanupFontPreview(type)
  addToast('info', `已移除 ${getTypeName(type)}`)
}

const getTypeName = (type: 'ch' | 'tc' | 'en') => {
  const names = { ch: '中文简体', tc: '中文繁体', en: '英文&数字' }
  return names[type]
}

const triggerFileInput = (type: 'ch' | 'tc' | 'en') => {
  const input = document.getElementById(`file-${type}`) as HTMLInputElement
  input?.click()
}

// --- 打包处理 ---
const startPacking = async () => {
  if (!uploadedFiles.ch) {
    addToast('error', '请先上传核心字体 ch.ttf')
    return
  }

  if (!templateZip.value) {
    addToast('error', '请先下载模板')
    return
  }

  // 1. 同步编辑器数据（如果处于源码模式）
  if (sourceMode.value) {
    parseModuleProp(rawModuleProp.value, true)
  }

  // 2. 验证 module.prop 字段
  const emptyKeys: Record<string, string> = {
    id: 'ID',
    name: '名称',
    version: '版本号',
    versionCode: '版本代码',
    author: '作者',
    description: '描述',
  }

  for (const [key, label] of Object.entries(emptyKeys)) {
    const val = modulePropFields[key as keyof typeof modulePropFields]
    if (typeof val === 'string' && !val.trim()) {
      addToast('error', `模块信息错误：[${label}] 不能为空`)
      return
    }
  }

  try {
    isProcessing.value = true
    packProgress.value = 0

    addToast('info', '正在初始化构建环境...')

    if (!originalTemplateData.value) {
      addToast('error', '模板数据丢失，请重新下载模板')
      isProcessing.value = false
      return
    }

    const targetZip = await JSZip.loadAsync(originalTemplateData.value)

    addToast('info', '正在同步配置文件和字体...')

    // --- 1. 注入 module.prop 到压缩包根目录 ---
    const propContent = Object.entries(modulePropFields)
      .map(([k, v]) => `${k}=${v}`)
      .join('\n')
    targetZip.file('module.prop', propContent)

    // --- 2. 注入字体文件到 /ttf/ 目录 ---
    const ttfFolder = targetZip.folder('ttf')
    if (ttfFolder) {
      if (uploadedFiles.ch) ttfFolder.file('ch.ttf', uploadedFiles.ch)
      if (uploadedFiles.tc) ttfFolder.file('tc.ttf', uploadedFiles.tc)
      if (uploadedFiles.en) ttfFolder.file('en.ttf', uploadedFiles.en)
    }

    // --- 调试信息：输出 JSZip 的详细结构到控制台 ---
    console.log('=== 将要打包的 JSZip 文件列表 ===')
    targetZip.forEach((relativePath) => {
      console.log(relativePath)
    })
    console.log('==================================')

    const content = await targetZip.generateAsync(
      {
        type: 'blob',
        compression: 'DEFLATE',
        compressionOptions: {
          level: COMP_LEVEL,
        },
        streamFiles: false,
      },
      (metadata) => {
        packProgress.value = 50 + Math.round(metadata.percent * 0.5)
      },
    )

    addToast('success', '资源已成功注入压缩包')

    await downloadModuleZip(content)

    isProcessing.value = false
    packProgress.value = 100
    addToast('success', '🎉 打包完成！文件已自动下载')
  } catch (error: unknown) {
    const msg = error instanceof Error ? error.message : String(error)
    addToast('error', `打包失败：${msg}`)
    isProcessing.value = false
  }
}

const downloadModuleZip = async (blob: Blob) => {
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url

  try {
    const arrayBuffer = await blob.arrayBuffer()
    const hash = await computeHash(arrayBuffer)
    const shortHash = hash.substring(0, 6)
    const { name, version } = modulePropFields
    // 移除非法文件名字符
    const safeName = name.replace(/[\\/:*?"<>|]/g, '_') || 'FontModule'
    const safeVersion = version.replace(/[\\/:*?"<>|]/g, '_') || '1.0'

    a.download = `${safeName}-${safeVersion}-${shortHash}.zip`
  } catch (error) {
    console.warn('Hash 计算失败，将使用默认文件名', error)
    a.download = 'MODULE.zip'
  }

  document.body.appendChild(a)
  a.click()
  document.body.removeChild(a)
  URL.revokeObjectURL(url)
}

// --- 生命周期 ---
onMounted(() => {
  addToast('info', '请下载或上传模板文件以开始')
})
</script>

<template>
  <div class="min-h-screen bg-background relative overflow-hidden">
    <!-- 背景装饰 -->
    <div class="fixed inset-0 pointer-events-none overflow-hidden -z-10">
      <div
        class="absolute -top-40 -right-40 w-80 h-80 rounded-full bg-primary/5 blur-3xl animate-blob"
      ></div>
      <div
        class="absolute top-1/3 -left-40 w-96 h-96 rounded-full bg-primary/5 blur-3xl animate-blob animation-delay-2000"
      ></div>
      <div
        class="absolute -bottom-40 right-1/4 w-72 h-72 rounded-full bg-primary/5 blur-3xl animate-blob animation-delay-4000"
      ></div>
    </div>

    <div class="max-w-3xl mx-auto px-4 py-8 sm:py-12 space-y-6">
      <!-- Hero 区域 -->
      <div
        class="relative rounded-2xl overflow-hidden bg-linear-to-br from-violet-600 via-purple-600 to-indigo-700 text-white shadow-2xl shadow-purple-500/20 animate-card-enter"
      >
        <!-- Hero 装饰 -->
        <div class="absolute inset-0 pointer-events-none">
          <div class="absolute -top-12 -right-12 w-48 h-48 rounded-full bg-white/10 blur-2xl"></div>
          <div class="absolute bottom-0 left-0 w-64 h-32 rounded-full bg-black/10 blur-2xl"></div>
        </div>

        <div class="relative px-5 py-6 sm:px-10 sm:py-10 text-center">
          <div
            class="inline-flex items-center justify-center w-12 h-12 sm:w-16 sm:h-16 rounded-xl sm:rounded-2xl bg-white/15 backdrop-blur-sm mb-3 sm:mb-4"
          >
            <Type :size="28" class="sm:hidden" :stroke-width="1.8" />
            <Type :size="32" class="hidden sm:block" :stroke-width="1.8" />
          </div>
          <h1 class="text-2xl sm:text-4xl font-extrabold tracking-tight">FontMM</h1>
          <p class="text-white/70 text-xs sm:text-sm mt-1 font-medium">For ColorOS 16</p>
          <Badge
            class="mt-3 sm:mt-4 bg-white/15 border-white/20 text-white/90 hover:bg-white/20 transition-colors text-[11px] sm:text-xs"
          >
            模板版本 {{ TEMPLATE_VERSION }}
          </Badge>
        </div>
      </div>

      <!-- Step 1: 加载模板 -->
      <div class="animate-card-enter" style="animation-delay: 0.08s">
        <Card class="group border-border/60 shadow-sm hover:shadow-md transition-all duration-300">
          <CardHeader class="pb-3 select-none">
            <div class="flex items-center gap-3">
              <div
                class="flex items-center justify-center w-8 h-8 rounded-lg bg-primary text-primary-foreground shadow-md shadow-primary/30 text-sm font-bold transition-colors"
              >
                1
              </div>
              <div class="flex-1 min-w-0">
                <CardTitle class="text-lg flex items-center gap-2">
                  <Download :size="18" class="text-primary shrink-0" />
                  加载模板
                </CardTitle>
                <CardDescription>下载模板文件到本地，或上传已有的模板</CardDescription>
              </div>
              <Badge
                v-if="templateZip"
                class="bg-green-100 text-green-700 border-green-200 dark:bg-green-900/40 dark:text-green-400 dark:border-green-800"
              >
                <CheckCircle2 :size="12" class="mr-1" />
                已就绪
              </Badge>
            </div>
          </CardHeader>
          <CardContent class="space-y-4 pt-1">
            <!-- 下载按钮 -->
            <div class="flex flex-col sm:flex-row gap-3 min-h-[44px]">
              <Button
                @click="downloadTemplate"
                :disabled="isProcessing"
                variant="outline"
                size="lg"
                class="flex-1 cursor-pointer h-auto py-3 sm:py-2"
              >
                <div class="flex flex-col items-center">
                  <div class="flex items-center">
                    <Download :size="16" class="mr-2" />
                    CDN 直链 (推荐)
                  </div>
                  <span class="text-[10px] text-muted-foreground mt-0.5">高速主线路</span>
                </div>
              </Button>

              <Button
                as="a"
                href="https://www.123865.com/s/iBeVVv-3akV"
                target="_blank"
                rel="noopener noreferrer"
                :disabled="isProcessing"
                variant="outline"
                size="lg"
                class="flex-1 cursor-pointer h-auto py-3 sm:py-2"
              >
                <div class="flex flex-col items-center">
                  <div class="flex items-center">
                    <Download :size="16" class="mr-2" />
                    123 云盘 (备用)
                  </div>
                  <span class="text-[10px] text-muted-foreground mt-0.5">提取码: 无</span>
                </div>
              </Button>
            </div>

            <!-- 上传模板区域 -->
            <input
              id="file-template"
              type="file"
              accept=".zip"
              @change="handleTemplateUpload"
              class="hidden"
            />
            <div v-if="!templateZip">
              <div
                @click="triggerTemplateInput"
                @drop="handleTemplateDrop"
                @dragover.prevent="templateDragOver = true"
                @dragleave="templateDragOver = false"
                :class="[
                  'flex flex-col items-center justify-center gap-2 p-6 rounded-xl border-2 border-dashed cursor-pointer transition-all duration-200',
                  templateDragOver
                    ? 'border-primary bg-primary/5 animate-drop-pulse'
                    : 'border-border hover:border-primary/50 hover:bg-accent/50',
                ]"
              >
                <Loader2 v-if="isVerifying" :size="24" class="text-primary animate-spin" />
                <HardDriveUpload v-else :size="24" class="text-muted-foreground" />
                <span class="text-sm text-muted-foreground text-center">
                  {{ isVerifying ? '正在校验 Hash...' : '点击或拖放模板 ZIP 文件' }}
                </span>
                <span class="text-[11px] text-muted-foreground/60 flex items-center gap-1">
                  <ShieldCheck :size="12" />
                  上传后自动校验 SHA-256
                </span>
              </div>
            </div>
            <!-- 模板已加载状态 -->
            <div
              v-else
              class="flex items-center gap-3 p-4 rounded-xl bg-green-50 dark:bg-green-900/20 border border-green-200 dark:border-green-800"
            >
              <CheckCircle2 :size="20" class="text-green-600 dark:text-green-400 shrink-0" />
              <div class="flex-1 min-w-0">
                <p class="text-sm font-medium text-green-700 dark:text-green-300">模板已加载</p>
                <p class="text-xs text-green-600/70 dark:text-green-400/70">Hash 校验通过</p>
              </div>
              <Badge
                class="bg-green-100 dark:bg-green-800/50 text-green-700 dark:text-green-300 border-green-200 dark:border-green-700"
              >
                {{ TEMPLATE_VERSION }}
              </Badge>
            </div>
          </CardContent>
        </Card>
      </div>

      <!-- Step 2: 上传字体文件 -->
      <div class="animate-card-enter" style="animation-delay: 0.16s">
        <Card class="group border-border/60 shadow-sm hover:shadow-md transition-all duration-300">
          <CardHeader class="pb-3 select-none">
            <div class="flex items-center gap-3">
              <div
                class="flex items-center justify-center w-8 h-8 rounded-lg bg-primary text-primary-foreground shadow-md shadow-primary/30 text-sm font-bold transition-colors"
              >
                2
              </div>
              <div class="flex-1 min-w-0">
                <CardTitle class="text-lg flex items-center gap-2">
                  <Upload :size="18" class="text-primary shrink-0" />
                  上传字体文件
                </CardTitle>
                <CardDescription>拖放或点击选择需要打包的字体文件</CardDescription>
              </div>
            </div>
          </CardHeader>
          <CardContent class="pt-1">
            <div class="grid gap-3 sm:gap-4 grid-cols-1 sm:grid-cols-2 lg:grid-cols-3">
              <!-- ch.ttf -->
              <div class="space-y-2">
                <label class="text-sm font-medium flex items-center gap-2">
                  中文简体 (ch.ttf)
                  <Badge variant="destructive" class="text-[10px] px-1.5 py-0">必选</Badge>
                </label>
                <p class="text-[11px] text-muted-foreground leading-tight">
                  核心字体，作为模块的基础资源
                </p>
                <input
                  :id="'file-ch'"
                  type="file"
                  accept=".ttf,.otf"
                  @change="(e) => handleFileUpload(e, 'ch')"
                  class="hidden"
                />
                <!-- Drop zone -->
                <div
                  v-if="!uploadedFiles.ch"
                  @click="triggerFileInput('ch')"
                  @drop="(e) => handleDrop(e, 'ch')"
                  @dragover="(e) => handleDragOver(e, 'ch')"
                  @dragleave="handleDragLeave('ch')"
                  :class="[
                    'flex flex-col items-center justify-center gap-1.5 sm:gap-2 p-4 sm:p-5 rounded-xl border-2 border-dashed cursor-pointer transition-all duration-200',
                    dragStates.ch
                      ? 'border-primary bg-primary/5 animate-drop-pulse'
                      : 'border-border hover:border-primary/50 hover:bg-accent/50',
                  ]"
                >
                  <Upload :size="20" class="text-muted-foreground" />
                  <span class="text-xs text-muted-foreground text-center">点击或拖放文件</span>
                </div>
                <!-- File uploaded state -->
                <div
                  v-else
                  class="flex items-center gap-2 p-3 rounded-xl bg-green-50 dark:bg-green-900/20 border border-green-200 dark:border-green-800"
                >
                  <CheckCircle2 :size="16" class="text-green-600 dark:text-green-400 shrink-0" />
                  <div class="flex-1 min-w-0">
                    <p class="text-xs font-medium text-green-700 dark:text-green-300 truncate">
                      {{ uploadedFiles.ch.name }}
                    </p>
                    <p class="text-[10px] text-green-600/70 dark:text-green-400/70">
                      {{ formatFileSize(uploadedFiles.ch.size) }}
                    </p>
                  </div>
                  <button
                    @click="removeFile('ch')"
                    class="p-1 rounded-md hover:bg-green-100 dark:hover:bg-green-800/50 transition-colors cursor-pointer"
                  >
                    <Trash2 :size="14" class="text-green-600 dark:text-green-400" />
                  </button>
                </div>
              </div>

              <!-- tc.ttf -->
              <div class="space-y-2">
                <label class="text-sm font-medium flex items-center gap-2">
                  中文繁体 (tc.ttf)
                  <Badge variant="outline" class="text-[10px] px-1.5 py-0">可选</Badge>
                </label>
                <p class="text-[11px] text-muted-foreground leading-tight">
                  未上传时将自动使用 ch.ttf 替换
                </p>
                <input
                  :id="'file-tc'"
                  type="file"
                  accept=".ttf,.otf"
                  @change="(e) => handleFileUpload(e, 'tc')"
                  class="hidden"
                />
                <div
                  v-if="!uploadedFiles.tc"
                  @click="triggerFileInput('tc')"
                  @drop="(e) => handleDrop(e, 'tc')"
                  @dragover="(e) => handleDragOver(e, 'tc')"
                  @dragleave="handleDragLeave('tc')"
                  :class="[
                    'flex flex-col items-center justify-center gap-1.5 sm:gap-2 p-4 sm:p-5 rounded-xl border-2 border-dashed cursor-pointer transition-all duration-200',
                    dragStates.tc
                      ? 'border-primary bg-primary/5 animate-drop-pulse'
                      : 'border-border hover:border-primary/50 hover:bg-accent/50',
                  ]"
                >
                  <Upload :size="20" class="text-muted-foreground" />
                  <span class="text-xs text-muted-foreground text-center">点击或拖放文件</span>
                </div>
                <div
                  v-else
                  class="flex items-center gap-2 p-3 rounded-xl bg-green-50 dark:bg-green-900/20 border border-green-200 dark:border-green-800"
                >
                  <CheckCircle2 :size="16" class="text-green-600 dark:text-green-400 shrink-0" />
                  <div class="flex-1 min-w-0">
                    <p class="text-xs font-medium text-green-700 dark:text-green-300 truncate">
                      {{ uploadedFiles.tc.name }}
                    </p>
                    <p class="text-[10px] text-green-600/70 dark:text-green-400/70">
                      {{ formatFileSize(uploadedFiles.tc.size) }}
                    </p>
                  </div>
                  <button
                    @click="removeFile('tc')"
                    class="p-1 rounded-md hover:bg-green-100 dark:hover:bg-green-800/50 transition-colors cursor-pointer"
                  >
                    <Trash2 :size="14" class="text-green-600 dark:text-green-400" />
                  </button>
                </div>
              </div>

              <!-- en.ttf -->
              <div class="space-y-2">
                <label class="text-sm font-medium flex items-center gap-2">
                  英文&数字 (en.ttf)
                  <Badge variant="outline" class="text-[10px] px-1.5 py-0">可选</Badge>
                </label>
                <p class="text-[11px] text-muted-foreground leading-tight">
                  未上传时将自动使用 ch.ttf 替换
                </p>
                <input
                  :id="'file-en'"
                  type="file"
                  accept=".ttf,.otf"
                  @change="(e) => handleFileUpload(e, 'en')"
                  class="hidden"
                />
                <div
                  v-if="!uploadedFiles.en"
                  @click="triggerFileInput('en')"
                  @drop="(e) => handleDrop(e, 'en')"
                  @dragover="(e) => handleDragOver(e, 'en')"
                  @dragleave="handleDragLeave('en')"
                  :class="[
                    'flex flex-col items-center justify-center gap-1.5 sm:gap-2 p-4 sm:p-5 rounded-xl border-2 border-dashed hacker transition-all duration-200',
                    dragStates.en
                      ? 'border-primary bg-primary/5 animate-drop-pulse'
                      : 'border-border hover:border-primary/50 hover:bg-accent/50',
                  ]"
                >
                  <Upload :size="20" class="text-muted-foreground" />
                  <span class="text-xs text-muted-foreground text-center">点击或拖放文件</span>
                </div>
                <div
                  v-else
                  class="flex items-center gap-2 p-3 rounded-xl bg-green-50 dark:bg-green-900/20 border border-green-200 dark:border-green-800"
                >
                  <CheckCircle2 :size="16" class="text-green-600 dark:text-green-400 shrink-0" />
                  <div class="flex-1 min-w-0">
                    <p class="text-xs font-medium text-green-700 dark:text-green-300 truncate">
                      {{ uploadedFiles.en.name }}
                    </p>
                    <p class="text-[10px] text-green-600/70 dark:text-green-400/70">
                      {{ formatFileSize(uploadedFiles.en.size) }}
                    </p>
                  </div>
                  <button
                    @click="removeFile('en')"
                    class="p-1 rounded-md hover:bg-green-100 dark:hover:bg-green-800/50 transition-colors cursor-pointer"
                  >
                    <Trash2 :size="14" class="text-green-600 dark:text-green-400" />
                  </button>
                </div>
              </div>
            </div>

            <!-- Font Preview Panel -->
            <div
              v-if="uploadedFiles.ch || uploadedFiles.tc || uploadedFiles.en"
              class="mt-6 p-4 rounded-xl border border-primary/20 bg-primary/5 space-y-4 animate-in fade-in slide-in-from-top-2 duration-500"
            >
              <div class="flex items-center justify-between">
                <div class="flex items-center gap-2">
                  <Type :size="18" class="text-primary" />
                  <span class="text-sm font-bold">字体预览</span>
                </div>
                <div class="flex gap-2">
                  <Badge variant="secondary" class="text-[10px] font-normal opacity-70">
                    仅供效果展示
                  </Badge>
                </div>
              </div>

              <!-- Fallback Tip -->
              <div
                v-if="uploadedFiles.ch && (!uploadedFiles.tc || !uploadedFiles.en)"
                class="flex items-start gap-2 p-2.5 rounded-lg bg-orange-500/5 border border-orange-500/20 text-orange-600 dark:text-orange-400 text-[11px] leading-tight"
              >
                <Info :size="14" class="shrink-0 mt-0.5" />
                <p>
                  温馨提示：由于您未上传
                  {{
                    !uploadedFiles.tc && !uploadedFiles.en
                      ? '繁体和英文'
                      : !uploadedFiles.tc
                        ? '繁体'
                        : '英文'
                  }}
                  字体，打包程序将自动使用
                  <span class="font-bold underline">中文简体</span> 字体进行填充。
                </p>
              </div>

              <div class="grid gap-4">
                <!-- ch preview -->
                <div v-if="uploadedFiles.ch" class="space-y-2">
                  <div class="flex items-center justify-between">
                    <p class="text-[10px] text-muted-foreground uppercase tracking-wider font-bold">
                      中文简体 (ch)
                    </p>
                    <span class="text-[10px] text-primary/60 font-mono">{{
                      fontPreviewNames.ch ? '已载入' : '加载中...'
                    }}</span>
                  </div>
                  <div
                    :style="{ fontFamily: fontPreviewNames.ch || 'inherit' }"
                    class="p-3 sm:p-4 rounded-lg bg-background border border-border/50 text-xl sm:text-2xl break-all transition-all"
                  >
                    永和九年，岁在癸丑，暮春之初。
                  </div>
                </div>

                <!-- tc preview (shown if ch exists) -->
                <div v-if="uploadedFiles.ch || uploadedFiles.tc" class="space-y-2">
                  <div class="flex items-center justify-between">
                    <div class="flex items-center gap-2">
                      <p
                        class="text-[10px] text-muted-foreground uppercase tracking-wider font-bold"
                      >
                        中文繁体 (tc)
                      </p>
                      <Badge
                        v-if="!uploadedFiles.tc"
                        variant="outline"
                        class="h-4 text-[9px] px-1 border-orange-500/30 text-orange-500"
                      >
                        使用简体回退
                      </Badge>
                    </div>
                    <span class="text-[10px] text-primary/60 font-mono">
                      {{
                        uploadedFiles.tc
                          ? fontPreviewNames.tc
                            ? '已载入'
                            : '加载中...'
                          : '回退模式'
                      }}
                    </span>
                  </div>
                  <div
                    :style="{
                      fontFamily: fontPreviewNames.tc || fontPreviewNames.ch || 'inherit',
                    }"
                    class="p-3 sm:p-4 rounded-lg bg-background border border-border/50 text-xl sm:text-2xl break-all transition-all"
                  >
                    憂郁的臺灣烏龜，尋找綠色龜殼。
                  </div>
                </div>

                <!-- en preview (shown if ch exists) -->
                <div v-if="uploadedFiles.ch || uploadedFiles.en" class="space-y-2">
                  <div class="flex items-center justify-between">
                    <div class="flex items-center gap-2">
                      <p
                        class="text-[10px] text-muted-foreground uppercase tracking-wider font-bold"
                      >
                        英文 & 数字 (en)
                      </p>
                      <Badge
                        v-if="!uploadedFiles.en"
                        variant="outline"
                        class="h-4 text-[9px] px-1 border-orange-500/30 text-orange-500"
                      >
                        使用简体回退
                      </Badge>
                    </div>
                    <span class="text-[10px] text-primary/60 font-mono">
                      {{
                        uploadedFiles.en
                          ? fontPreviewNames.en
                            ? '已载入'
                            : '加载中...'
                          : '回退模式'
                      }}
                    </span>
                  </div>
                  <div
                    :style="{
                      fontFamily: fontPreviewNames.en || fontPreviewNames.ch || 'inherit',
                    }"
                    class="p-3 sm:p-4 rounded-lg bg-background border border-border/50 text-xl sm:text-2xl break-all transition-all"
                  >
                    ABCDEFG abcdefg 1234567890
                  </div>
                </div>
              </div>
            </div>
          </CardContent>
        </Card>
      </div>

      <!-- Step 3: 编辑 module.prop -->
      <div class="animate-card-enter" style="animation-delay: 0.24s">
        <Card class="group border-border/60 shadow-sm hover:shadow-md transition-all duration-300">
          <CardHeader class="pb-3 select-none">
            <div class="flex items-center gap-3">
              <div
                class="flex items-center justify-center w-8 h-8 rounded-lg bg-primary text-primary-foreground shadow-md shadow-primary/30 text-sm font-bold transition-colors"
              >
                3
              </div>
              <div class="flex-1 min-w-0">
                <CardTitle class="text-lg flex items-center gap-2">
                  <FileText :size="18" class="text-primary shrink-0" />
                  编辑模块信息
                </CardTitle>
                <CardDescription>可选步骤，建议更改</CardDescription>
              </div>
              <div class="flex items-center gap-1 z-10 shrink-0" @click.stop>
                <Button
                  variant="ghost"
                  size="sm"
                  @click="toggleSourceMode"
                  class="cursor-pointer text-muted-foreground hover:text-primary px-2 sm:px-3"
                >
                  <Code v-if="!sourceMode" :size="14" class="sm:mr-1" />
                  <FormInput v-else :size="14" class="sm:mr-1" />
                  <span class="hidden sm:inline">{{ sourceMode ? '表单' : '源码' }}</span>
                </Button>
                <Button
                  variant="ghost"
                  size="sm"
                  @click="resetModuleProp"
                  class="cursor-pointer text-muted-foreground hover:text-primary px-2 sm:px-3"
                >
                  <RotateCcw :size="14" class="sm:mr-1" />
                  <span class="hidden sm:inline">重置</span>
                </Button>
              </div>
            </div>
          </CardHeader>
          <CardContent class="pt-1">
            <!-- 源代码模式 -->
            <div v-if="sourceMode">
              <textarea
                v-model="rawModuleProp"
                rows="7"
                class="w-full rounded-xl border border-input bg-muted/40 dark:bg-muted/20 px-3 sm:px-4 py-2.5 sm:py-3 text-xs sm:text-sm font-mono leading-relaxed ring-offset-background placeholder:text-muted-foreground focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2 resize-y min-h-[120px] sm:min-h-[160px] transition-colors"
                placeholder="id=MyFontModule&#10;name=我的字体模块&#10;..."
                spellcheck="false"
              ></textarea>
              <p class="text-xs text-muted-foreground mt-2 flex items-center gap-1">
                <Info :size="12" />
                切换回表单模式时会自动解析内容
              </p>
            </div>
            <!-- 表单模式 -->
            <div v-else class="grid gap-3 sm:gap-4 grid-cols-1 sm:grid-cols-2">
              <!-- id -->
              <div class="space-y-1.5">
                <label
                  class="text-xs sm:text-sm font-medium text-foreground flex items-center gap-1"
                >
                  ID
                  <span class="text-destructive">*</span>
                </label>
                <input
                  v-model="modulePropFields.id"
                  type="text"
                  class="w-full rounded-lg border border-input bg-muted/40 dark:bg-muted/20 px-3 py-2 text-sm font-mono ring-offset-background placeholder:text-muted-foreground focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2 transition-colors"
                  placeholder="MyFontModule"
                />
                <p class="text-[11px] text-muted-foreground">模块唯一标识，仅英文和数字</p>
              </div>
              <!-- name -->
              <div class="space-y-1.5">
                <label
                  class="text-xs sm:text-sm font-medium text-foreground flex items-center gap-1"
                >
                  名称
                  <span class="text-destructive">*</span>
                </label>
                <input
                  v-model="modulePropFields.name"
                  type="text"
                  class="w-full rounded-lg border border-input bg-muted/40 dark:bg-muted/20 px-3 py-2 text-sm ring-offset-background placeholder:text-muted-foreground focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2 transition-colors"
                  placeholder="我的字体模块"
                />
                <p class="text-[11px] text-muted-foreground">显示在模块管理器中的名称</p>
              </div>
              <!-- version -->
              <div class="space-y-1.5">
                <label
                  class="text-xs sm:text-sm font-medium text-foreground flex items-center gap-1"
                >
                  版本号
                  <span class="text-destructive">*</span>
                </label>
                <input
                  v-model="modulePropFields.version"
                  type="text"
                  class="w-full rounded-lg border border-input bg-muted/40 dark:bg-muted/20 px-3 py-2 text-sm font-mono ring-offset-background placeholder:text-muted-foreground focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2 transition-colors"
                  placeholder="1.0"
                />
              </div>
              <!-- versionCode -->
              <div class="space-y-1.5">
                <label
                  class="text-xs sm:text-sm font-medium text-foreground flex items-center gap-1"
                >
                  版本代码
                  <span class="text-destructive">*</span>
                </label>
                <input
                  v-model="modulePropFields.versionCode"
                  type="number"
                  min="1"
                  class="w-full rounded-lg border border-input bg-muted/40 dark:bg-muted/20 px-3 py-2 text-sm font-mono ring-offset-background placeholder:text-muted-foreground focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2 transition-colors"
                  placeholder="10"
                />
                <p class="text-[11px] text-muted-foreground">整数，每次更新递增</p>
              </div>
              <!-- author -->
              <div class="space-y-1.5">
                <label
                  class="text-xs sm:text-sm font-medium text-foreground flex items-center gap-1"
                >
                  作者
                  <span class="text-destructive">*</span>
                </label>
                <input
                  v-model="modulePropFields.author"
                  type="text"
                  class="w-full rounded-lg border border-input bg-muted/40 dark:bg-muted/20 px-3 py-2 text-sm ring-offset-background placeholder:text-muted-foreground focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2 transition-colors"
                  placeholder="Yule"
                />
              </div>
              <!-- description -->
              <div class="space-y-1.5 sm:col-span-2">
                <label
                  class="text-xs sm:text-sm font-medium text-foreground flex items-center gap-1"
                >
                  描述
                  <span class="text-destructive">*</span>
                </label>
                <input
                  v-model="modulePropFields.description"
                  type="text"
                  class="w-full rounded-lg border border-input bg-muted/40 dark:bg-muted/20 px-3 py-2 text-sm ring-offset-background placeholder:text-muted-foreground focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2 transition-colors"
                  placeholder="我的字体模块"
                />
              </div>
            </div>
          </CardContent>
        </Card>
      </div>

      <!-- Step 4: 打包 -->
      <div class="animate-card-enter" style="animation-delay: 0.32s">
        <Card class="group border-border/60 shadow-sm hover:shadow-md transition-all duration-300">
          <CardHeader class="pb-3 select-none">
            <div class="flex items-center gap-3">
              <div
                class="flex items-center justify-center w-8 h-8 rounded-lg bg-primary text-primary-foreground shadow-md shadow-primary/30 text-sm font-bold transition-colors"
              >
                4
              </div>
              <div class="flex-1 min-w-0">
                <CardTitle class="text-lg flex items-center gap-2">
                  <Package :size="18" class="text-primary shrink-0" />
                  开始打包
                </CardTitle>
                <CardDescription>生成 MODULE.zip 文件</CardDescription>
              </div>
            </div>
          </CardHeader>
          <CardContent class="space-y-4 pt-1">
            <Button
              @click="startPacking"
              :disabled="isProcessing || !templateZip || !uploadedFiles.ch"
              size="lg"
              class="w-full text-base font-semibold cursor-pointer"
            >
              <Package v-if="!isProcessing" :size="18" class="mr-2" />
              {{ isProcessing ? '正在打包...' : packProgress >= 100 ? '再次打包' : '开始打包' }}
            </Button>

            <!-- Stepper 进度指示器 -->
            <div v-if="isProcessing || packProgress >= 100" class="pt-2">
              <div class="flex items-center justify-between mb-3">
                <span class="text-sm text-muted-foreground">处理进度</span>
                <span class="text-sm font-mono font-semibold text-primary"
                  >{{ packProgress }}%</span
                >
              </div>
              <Progress :model-value="packProgress" class="h-2 mb-4" />

              <!-- Horizontal stepper -->
              <div class="flex items-center">
                <template v-for="(step, index) in packSteps" :key="index">
                  <!-- Step node -->
                  <div class="flex flex-col items-center flex-1 min-w-0">
                    <div
                      :class="[
                        'w-6 h-6 sm:w-8 sm:h-8 rounded-full flex items-center justify-center text-[10px] sm:text-xs font-bold transition-all duration-500',
                        step.done
                          ? 'bg-primary text-primary-foreground shadow-md shadow-primary/30'
                          : 'bg-muted text-muted-foreground',
                      ]"
                    >
                      <CheckCircle2 v-if="step.done" :size="14" class="sm:hidden" />
                      <CheckCircle2 v-if="step.done" :size="16" class="hidden sm:block" />
                      <span v-if="!step.done">{{ index + 1 }}</span>
                    </div>
                    <span
                      :class="[
                        'text-[10px] sm:text-[11px] mt-1 sm:mt-1.5 font-medium transition-colors text-center truncate w-full',
                        step.done ? 'text-primary' : 'text-muted-foreground',
                      ]"
                    >
                      {{ step.label }}
                    </span>
                  </div>
                  <!-- Connector line -->
                  <div
                    v-if="index < packSteps.length - 1"
                    :class="[
                      'h-[2px] flex-1 -mt-4 sm:-mt-5 mx-0.5 sm:mx-1 rounded-full transition-all duration-500',
                      packSteps[index + 1]?.done ? 'bg-primary' : 'bg-muted',
                    ]"
                  ></div>
                </template>
              </div>
            </div>
          </CardContent>
        </Card>
      </div>

      <!-- Footer -->
      <footer
        class="text-center py-6 text-xs text-muted-foreground animate-card-enter"
        style="animation-delay: 0.4s"
      >
        <p>FontMM &copy; {{ new Date().getFullYear() }} · By 于乐 Yule</p>
        <p class="mt-1 opacity-70">纯前端打包，文件不会上传到服务器</p>
      </footer>
    </div>

    <!-- Toast 通知容器 -->
    <div
      class="fixed bottom-3 left-3 right-3 sm:left-auto sm:bottom-4 sm:right-4 z-50 flex flex-col gap-2 max-w-sm w-auto sm:w-full pointer-events-none"
    >
      <div
        v-for="toast in toastMessages"
        :key="toast.id"
        :class="[
          'pointer-events-auto flex items-start gap-2.5 px-4 py-3 rounded-xl shadow-lg border backdrop-blur-sm text-sm',
          toast.leaving ? 'animate-toast-out' : 'animate-toast-in',
          toast.type === 'success'
            ? 'bg-green-50/95 dark:bg-green-900/80 text-green-800 dark:text-green-200 border-green-200 dark:border-green-700'
            : toast.type === 'error'
              ? 'bg-red-50/95 dark:bg-red-900/80 text-red-800 dark:text-red-200 border-red-200 dark:border-red-700'
              : toast.type === 'warn'
                ? 'bg-yellow-50/95 dark:bg-yellow-900/80 text-yellow-800 dark:text-yellow-200 border-yellow-200 dark:border-yellow-700'
                : 'bg-blue-50/95 dark:bg-blue-900/80 text-blue-800 dark:text-blue-200 border-blue-200 dark:border-blue-700',
        ]"
      >
        <component
          :is="
            toast.type === 'success'
              ? CheckCircle2
              : toast.type === 'error'
                ? XCircle
                : toast.type === 'warn'
                  ? AlertTriangle
                  : Info
          "
          :size="16"
          class="shrink-0 mt-0.5"
        />
        <span class="flex-1 leading-snug">{{ toast.message }}</span>
        <button
          @click="removeToast(toast.id)"
          class="shrink-0 p-0.5 rounded-md hover:bg-black/5 dark:hover:bg-white/10 transition-colors cursor-pointer"
        >
          <X :size="14" />
        </button>
      </div>
    </div>
  </div>
</template>
