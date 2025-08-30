<template>
  <div class="file-uploader">
    <el-card class="upload-card">
      <template #header>
        <div class="card-header">
          <span>文件上传与解析</span>
        </div>
      </template>
      
      <!-- 文件上传区域 -->
      <el-upload
        class="upload-demo"
        drag
        :auto-upload="false"
        :on-change="handleFileChange"
        :show-file-list="false"
        accept=".xlsx,.xls,.csv"
      >
        <el-icon class="el-icon--upload"><upload-filled /></el-icon>
        <div class="el-upload__text">
          将Excel或CSV文件拖拽到此处，或<em>点击上传</em>
        </div>
        <template #tip>
          <div class="el-upload__tip">
            支持 .xlsx, .xls, .csv 格式文件
          </div>
        </template>
      </el-upload>

      <!-- 文件信息显示 -->
      <div v-if="currentFile" class="file-info">
        <el-alert
          :title="`已选择文件: ${currentFile.name}`"
          type="info"
          :closable="false"
        />
        <div class="file-actions">
          <el-button type="primary" @click="parseFile" :loading="parsing">
            解析文件
          </el-button>
          <el-button @click="clearFile">清除</el-button>
        </div>
      </div>

      <!-- 解析进度 -->
      <div v-if="parsing" class="parsing-progress">
        <el-progress :percentage="100" :indeterminate="true" />
        <p>正在解析文件...</p>
      </div>
    </el-card>

    <!-- 数据预览 -->
    <el-card v-if="parsedData.length > 0" class="data-preview">
      <template #header>
        <div class="card-header">
          <span>数据预览 (共 {{ parsedData.length }} 行)</span>
          <el-button type="primary" size="small" @click="exportData">
            导出JSON
          </el-button>
        </div>
      </template>
      
      <el-table
        :data="displayData"
        style="width: 100%"
        max-height="400"
        stripe
        border
      >
        <el-table-column
          v-for="(column, index) in columns"
          :key="index"
          :prop="column"
          :label="column"
          :width="150"
          show-overflow-tooltip
        />
      </el-table>
      
      <!-- 分页 -->
      <div class="pagination-wrapper" v-if="parsedData.length > pageSize">
        <el-pagination
          v-model:current-page="currentPage"
          :page-size="pageSize"
          :total="parsedData.length"
          layout="prev, pager, next, jumper, total"
          @current-change="handlePageChange"
        />
      </div>
    </el-card>

    <!-- 错误信息 -->
    <el-alert
      v-if="errorMessage"
      :title="errorMessage"
      type="error"
      show-icon
      :closable="false"
      class="error-alert"
    />
  </div>
</template>

<script setup>
import { ref, computed } from 'vue'
import { ElMessage } from 'element-plus'
import { UploadFilled } from '@element-plus/icons-vue'
import * as XLSX from 'xlsx'

// 响应式数据
const currentFile = ref(null)
const parsing = ref(false)
const parsedData = ref([])
const columns = ref([])
const errorMessage = ref('')
const currentPage = ref(1)
const pageSize = ref(50)

// 计算属性
const displayData = computed(() => {
  const start = (currentPage.value - 1) * pageSize.value
  const end = start + pageSize.value
  return parsedData.value.slice(start, end)
})

// 文件选择处理
const handleFileChange = (file) => {
  currentFile.value = file.raw
  errorMessage.value = ''
  parsedData.value = []
  columns.value = []
  currentPage.value = 1
}

// 清除文件
const clearFile = () => {
  currentFile.value = null
  parsedData.value = []
  columns.value = []
  errorMessage.value = ''
  currentPage.value = 1
}

// 解析CSV文件
const parseCSV = (text) => {
  const lines = text.split('\n').filter(line => line.trim())
  if (lines.length === 0) {
    throw new Error('CSV文件为空')
  }

  // 解析CSV行（简单实现，支持逗号分隔）
  const parseCSVLine = (line) => {
    const result = []
    let current = ''
    let inQuotes = false
    
    for (let i = 0; i < line.length; i++) {
      const char = line[i]
      
      if (char === '"') {
        inQuotes = !inQuotes
      } else if (char === ',' && !inQuotes) {
        result.push(current.trim())
        current = ''
      } else {
        current += char
      }
    }
    
    result.push(current.trim())
    return result
  }

  const headers = parseCSVLine(lines[0])
  const data = []

  for (let i = 1; i < lines.length; i++) {
    const values = parseCSVLine(lines[i])
    const row = {}
    
    headers.forEach((header, index) => {
      row[header] = values[index] || ''
    })
    
    data.push(row)
  }

  return { headers, data }
}

// 解析Excel文件
const parseExcel = (arrayBuffer) => {
  const workbook = XLSX.read(arrayBuffer, { type: 'array' })
  const firstSheetName = workbook.SheetNames[0]
  const worksheet = workbook.Sheets[firstSheetName]
  
  // 转换为JSON格式
  const jsonData = XLSX.utils.sheet_to_json(worksheet, { header: 1 })
  
  if (jsonData.length === 0) {
    throw new Error('Excel文件为空')
  }

  const headers = jsonData[0].map(header => String(header || ''))
  const data = []

  for (let i = 1; i < jsonData.length; i++) {
    const row = {}
    headers.forEach((header, index) => {
      row[header] = jsonData[i][index] || ''
    })
    data.push(row)
  }

  return { headers, data }
}

// 解析文件主函数
const parseFile = async () => {
  if (!currentFile.value) {
    ElMessage.error('请先选择文件')
    return
  }

  parsing.value = true
  errorMessage.value = ''

  try {
    const fileExtension = currentFile.value.name.split('.').pop().toLowerCase()
    let result

    if (fileExtension === 'csv') {
      // 解析CSV文件
      const text = await readFileAsText(currentFile.value)
      result = parseCSV(text)
    } else if (['xlsx', 'xls'].includes(fileExtension)) {
      // 解析Excel文件
      const arrayBuffer = await readFileAsArrayBuffer(currentFile.value)
      result = parseExcel(arrayBuffer)
    } else {
      throw new Error('不支持的文件格式')
    }

    columns.value = result.headers
    parsedData.value = result.data
    currentPage.value = 1

    ElMessage.success(`文件解析成功！共解析 ${result.data.length} 行数据`)
  } catch (error) {
    console.error('文件解析错误:', error)
    errorMessage.value = `解析失败: ${error.message}`
    ElMessage.error('文件解析失败')
  } finally {
    parsing.value = false
  }
}

// 文件读取辅助函数
const readFileAsText = (file) => {
  return new Promise((resolve, reject) => {
    const reader = new FileReader()
    reader.onload = (e) => resolve(e.target.result)
    reader.onerror = reject
    reader.readAsText(file, 'UTF-8')
  })
}

const readFileAsArrayBuffer = (file) => {
  return new Promise((resolve, reject) => {
    const reader = new FileReader()
    reader.onload = (e) => resolve(e.target.result)
    reader.onerror = reject
    reader.readAsArrayBuffer(file)
  })
}

// 分页处理
const handlePageChange = (page) => {
  currentPage.value = page
}

// 导出数据
const exportData = () => {
  const dataStr = JSON.stringify(parsedData.value, null, 2)
  const dataBlob = new Blob([dataStr], { type: 'application/json' })
  
  const link = document.createElement('a')
  link.href = URL.createObjectURL(dataBlob)
  link.download = `parsed_data_${new Date().getTime()}.json`
  link.click()
  
  ElMessage.success('数据导出成功')
}
</script>

<style scoped>
.file-uploader {
  padding: 20px;
  max-width: 1200px;
  margin: 0 auto;
}

.upload-card {
  margin-bottom: 20px;
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.upload-demo {
  margin-bottom: 20px;
}

.file-info {
  margin-top: 20px;
}

.file-actions {
  margin-top: 15px;
  display: flex;
  gap: 10px;
}

.parsing-progress {
  margin-top: 20px;
  text-align: center;
}

.data-preview {
  margin-top: 20px;
}

.pagination-wrapper {
  margin-top: 20px;
  display: flex;
  justify-content: center;
}

.error-alert {
  margin-top: 20px;
}

:deep(.el-upload-dragger) {
  width: 100%;
}

:deep(.el-table) {
  font-size: 14px;
}

:deep(.el-table th) {
  background-color: #f5f7fa;
}
</style>
