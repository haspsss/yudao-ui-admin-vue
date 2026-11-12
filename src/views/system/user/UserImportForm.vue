<template>
  <Dialog v-model="dialogVisible" title="用户异步导入" width="500">
    <!-- 尚未创建任务时，显示上传区域 -->
    <el-upload v-if="importTaskId === null" ref="uploadRef" v-model:file-list="fileList" :action="importUrl"
      :auto-upload="false" :disabled="formLoading" :headers="uploadHeaders" :limit="1" :on-error="submitFormError"
      :on-exceed="handleExceed" :on-success="submitFormSuccess" accept=".xlsx, .xls" drag>
      <Icon icon="ep:upload" />

      <div class="el-upload__text">
        将文件拖到此处，或<em>点击上传</em>
      </div>

      <template #tip>
        <div class="el-upload__tip text-center">
          <span>仅允许导入 xls、xlsx 格式文件。</span>

          <el-link :underline="false" style="font-size: 12px; vertical-align: baseline" type="primary"
            @click="importTemplate">
            下载模板
          </el-link>
        </div>
      </template>
    </el-upload>

    <!-- 创建任务后，显示导入进度 -->
    <div v-if="importTaskId !== null" class="import-progress">
      <el-alert :title="`异步导入任务 #${importTaskId}`" type="info" :closable="false" show-icon />

      <div v-if="importTask" class="progress-content">
        <div class="progress-title">
          <span>导入进度</span>
          <span>{{ importTask.processedCount }} / {{ importTask.totalCount }}</span>
        </div>

        <el-progress :percentage="importPercentage" :status="progressStatus" :stroke-width="16" />

        <el-row :gutter="12" class="import-statistics">
          <el-col :span="8">
            <el-statistic title="总数量" :value="importTask.totalCount" />
          </el-col>

          <el-col :span="8">
            <el-statistic title="成功数量" :value="importTask.successCount" />
          </el-col>

          <el-col :span="8">
            <el-statistic title="失败数量" :value="importTask.failCount" />
          </el-col>
        </el-row>

        <el-alert v-if="importTask.status === 0" title="任务等待执行中" type="info" :closable="false" />

        <el-alert v-else-if="importTask.status === 1" title="正在后台导入用户，请稍候..." type="info" :closable="false" />

        <el-alert v-else-if="importTask.status === 2" title="导入任务已完成" type="success" :closable="false" show-icon />

        <el-alert v-else-if="importTask.status === 3" :title="importTask.errorMessage || '导入任务执行失败'" type="error"
          :closable="false" show-icon />
      </div>

      <div v-else class="progress-content">
        <el-text>正在获取任务进度...</el-text>
      </div>
    </div>

    <template #footer>
      <el-button v-if="importTaskId === null" :loading="formLoading" type="primary" @click="submitForm">
        开始导入
      </el-button>

      <el-button v-if="importTaskId !== null" :disabled="!isTaskFinished" type="primary" @click="finishImport">
        完成
      </el-button>

      <el-button @click="dialogVisible = false">
        {{ importTaskId === null ? '取消' : '关闭' }}
      </el-button>
    </template>
  </Dialog>
</template>

<script lang="ts" setup>
import * as UserApi from '@/api/system/user'
import { getAccessToken, getTenantId } from '@/utils/auth'
import download from '@/utils/download'

defineOptions({ name: 'SystemUserImportForm' })

const message = useMessage()
const emits = defineEmits(['success'])

// 弹窗状态
const dialogVisible = ref(false)
const formLoading = ref(false)

// 上传组件
const uploadRef = ref()
const fileList = ref([])
const uploadHeaders = ref()

// 异步上传接口
const importUrl =
  import.meta.env.VITE_BASE_URL +
  import.meta.env.VITE_API_URL +
  '/system/user/import-async'

// 当前任务 ID
const importTaskId = ref<number | null>(null)

// 当前任务详情
const importTask = ref<UserApi.UserImportTaskVO | null>(null)

// 轮询定时器
let importTimer: ReturnType<typeof setInterval> | null = null

// 防止重复发送进度查询请求
let polling = false

// 标记当前轮询会话，防止旧请求影响新任务
let pollingVersion = 0

// 避免同一任务重复弹出完成消息
let resultNotified = false

/** 当前导入百分比 */
const importPercentage = computed(() => {
  if (!importTask.value) {
    return 0
  }

  const total = importTask.value.totalCount
  const processed = importTask.value.processedCount

  if (total <= 0) {
    return importTask.value.status === 2 ? 100 : 0
  }

  return Math.min(100, Math.round((processed / total) * 100))
})

/** 进度条状态 */
const progressStatus = computed(() => {
  if (importTask.value?.status === 2) {
    return 'success'
  }

  if (importTask.value?.status === 3) {
    return 'exception'
  }

  return undefined
})

/** 任务是否结束 */
const isTaskFinished = computed(() => {
  return importTask.value?.status === 2 ||
    importTask.value?.status === 3
})

/** 停止进度轮询 */
const stopImportPolling = () => {
  pollingVersion++

  if (importTimer !== null) {
    clearInterval(importTimer)
    importTimer = null
  }
}

/** 查询导入任务进度 */
const refreshImportTask = async () => {
  if (importTaskId.value === null || polling) {
    return
  }

  const currentTaskId = importTaskId.value
  const currentVersion = pollingVersion

  polling = true

  try {
    const task = await UserApi.getUserImportTask(currentTaskId)

    // 弹窗关闭、任务切换或轮询停止后，忽略旧响应
    if (
      currentVersion !== pollingVersion ||
      currentTaskId !== importTaskId.value
    ) {
      return
    }

    importTask.value = task

    // 状态 2：完成，状态 3：失败
    if (task.status === 2 || task.status === 3) {
      stopImportPolling()

      if (!resultNotified) {
        resultNotified = true

        if (task.status === 2) {
          message.success(
            `导入结束：成功 ${task.successCount} 条，失败 ${task.failCount} 条`
          )

          // 后台完成后刷新用户列表
          emits('success')
        } else {
          message.error(
            task.errorMessage || '异步导入任务执行失败'
          )
        }
      }
    }
  } catch (error) {
    if (currentVersion === pollingVersion) {
      stopImportPolling()
      message.error('查询导入进度失败，请关闭弹窗后重新查看任务')
    }
  } finally {
    polling = false
  }
}

/** 开始每秒查询一次 */
const startImportPolling = () => {
  stopImportPolling()

  // 先创建定时器，避免第一次查询立即完成时残留定时器
  importTimer = setInterval(() => {
    void refreshImportTask()
  }, 1000)

  // 创建任务后立即查询一次
  void refreshImportTask()
}

/** 打开导入弹窗 */
const open = () => {
  stopImportPolling()

  dialogVisible.value = true
  formLoading.value = false

  importTaskId.value = null
  importTask.value = null
  resultNotified = false

  fileList.value = []

  void resetForm()
}

defineExpose({ open })

/** 提交 Excel */
const submitForm = () => {
  if (fileList.value.length === 0) {
    message.error('请上传文件')
    return
  }

  if (formLoading.value || importTaskId.value !== null) {
    return
  }

  uploadHeaders.value = {
    Authorization: 'Bearer ' + getAccessToken(),
    'tenant-id': getTenantId()
  }

  formLoading.value = true
  uploadRef.value?.submit()
}

/** Excel 上传成功 */
const submitFormSuccess = (response: any) => {
  formLoading.value = false

  if (response.code !== 0) {
    message.error(response.msg || '创建导入任务失败')
    return
  }

  const taskId = Number(response.data)

  if (!Number.isSafeInteger(taskId) || taskId <= 0) {
    message.error('后端返回的任务 ID 无效')
    return
  }

  importTaskId.value = taskId
  importTask.value = null
  resultNotified = false

  message.success(`异步导入任务创建成功，任务 ID：${taskId}`)

  // 开始查询后台执行进度
  startImportPolling()
}

/** 上传失败 */
const submitFormError = () => {
  formLoading.value = false
  message.error('上传失败，请检查网络或后台服务')
}

/** 重置上传组件 */
const resetForm = async () => {
  formLoading.value = false

  await nextTick()
  uploadRef.value?.clearFiles()
}

/** 超出文件数量限制 */
const handleExceed = () => {
  message.error('最多只能上传一个文件！')
}

/** 下载 Excel 模板 */
const importTemplate = async () => {
  const res = await UserApi.importUserTemplate()
  download.excel(res, '用户导入模版.xls')
}

/** 任务完成后关闭 */
const finishImport = () => {
  if (!isTaskFinished.value) {
    return
  }

  dialogVisible.value = false
}

/** 关闭弹窗时停止轮询 */
watch(dialogVisible, (visible) => {
  if (!visible) {
    stopImportPolling()
  }
})

/** 组件卸载时清理定时器 */
onUnmounted(() => {
  stopImportPolling()
})
</script>

<style scoped>
.import-progress {
  padding: 8px 0;
}

.progress-content {
  margin-top: 20px;
}

.progress-title {
  display: flex;
  justify-content: space-between;
  margin-bottom: 12px;
  font-size: 14px;
}

.import-statistics {
  margin: 24px 0;
  text-align: center;
}
</style>
