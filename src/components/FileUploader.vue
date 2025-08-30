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

    <!-- 数据统计 -->
    <el-card v-if="parsedData.length > 0" class="statistics-card">
      <template #header>
        <div class="card-header">
          <span>账单统计</span>
        </div>
      </template>
      
      <el-row :gutter="20">
        <el-col :span="6">
          <div class="stat-item">
            <div class="stat-value">{{ parsedData.length }}</div>
            <div class="stat-label">总交易数</div>
          </div>
        </el-col>
        <el-col :span="6">
          <div class="stat-item income">
            <div class="stat-value">{{ totalIncome.toFixed(2) }}</div>
            <div class="stat-label">总收入 (元)</div>
          </div>
        </el-col>
        <el-col :span="6">
          <div class="stat-item expense">
            <div class="stat-value">{{ totalExpense.toFixed(2) }}</div>
            <div class="stat-label">总支出 (元)</div>
          </div>
        </el-col>
        <el-col :span="6">
          <div class="stat-item net" :class="{ positive: netAmount >= 0, negative: netAmount < 0 }">
            <div class="stat-value">{{ netAmount.toFixed(2) }}</div>
            <div class="stat-label">净收入 (元)</div>
          </div>
        </el-col>
      </el-row>
    </el-card>

    <!-- 数据预览 -->
    <el-card v-if="parsedData.length > 0" class="data-preview">
      <template #header>
        <div class="card-header">
          <span>明细数据 (共 {{ parsedData.length }} 条记录)</span>
          <el-button type="primary" size="small" @click="exportData">
            导出JSON
          </el-button>
        </div>
      </template>
      
      <el-table
        :data="displayData"
        style="width: 100%"
        max-height="500"
        stripe
        border
      >
        <el-table-column
          prop="交易时间"
          label="交易时间"
          width="180"
          show-overflow-tooltip
        />
        <el-table-column
          prop="交易内容"
          label="交易内容"
          min-width="200"
          show-overflow-tooltip
        />
        <el-table-column
          prop="收款人/付款方"
          label="收款人/付款方"
          width="150"
          show-overflow-tooltip
        />
        <el-table-column
          prop="金额"
          label="金额"
          width="120"
          align="right"
        >
          <template #default="scope">
            <span class="amount-text">{{ scope.row['金额'] }}</span>
          </template>
        </el-table-column>
        <el-table-column
          prop="收支类型"
          label="收支类型"
          width="100"
          align="center"
        >
          <template #default="scope">
            <el-tag 
              :type="scope.row['收支类型'] === '收入' ? 'success' : 'danger'"
              size="small"
            >
              {{ scope.row['收支类型'] }}
            </el-tag>
          </template>
        </el-table-column>
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

// 统计数据计算
const totalIncome = computed(() => {
  return parsedData.value
    .filter(item => item['收支类型'] === '收入')
    .reduce((sum, item) => {
      const amount = parseFloat(item['金额']?.replace(/[^\d.-]/g, '') || 0)
      return sum + (isNaN(amount) ? 0 : amount)
    }, 0)
})

const totalExpense = computed(() => {
  return parsedData.value
    .filter(item => item['收支类型'] === '支出')
    .reduce((sum, item) => {
      const amount = parseFloat(item['金额']?.replace(/[^\d.-]/g, '') || 0)
      return sum + (isNaN(amount) ? 0 : amount)
    }, 0)
})

const netAmount = computed(() => {
  return totalIncome.value - totalExpense.value
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

// 智能识别账单明细的开始行
const findDetailStartRow = (data, headers) => {
  // 常见的明细标识关键词
  const detailKeywords = ['交易时间', '交易日期', '日期', '时间', '商户', '收款方', '付款方', '金额', '收支', '类型', '备注', '说明']
  
  for (let i = 0; i < Math.min(data.length, 20); i++) { // 只检查前20行
    const row = data[i]
    const rowValues = Object.values(row).join('').toLowerCase()
    
    // 检查是否包含明细关键词
    const hasDetailKeywords = detailKeywords.some(keyword => 
      rowValues.includes(keyword) || 
      Object.keys(row).some(key => key.includes(keyword))
    )
    
    if (hasDetailKeywords) {
      return i
    }
  }
  
  // 如果没找到明细标识，假设第一行是表头，第二行开始是明细
  return 1
}

// 智能映射字段名
const mapFieldNames = (headers) => {
  const fieldMapping = {
    // 交易时间相关
    '交易时间': 'transactionTime',
    '交易日期': 'transactionTime', 
    '日期': 'transactionTime',
    '时间': 'transactionTime',
    '成交时间': 'transactionTime',
    
    // 交易内容相关
    '交易说明': 'description',
    '商品说明': 'description',
    '交易备注': 'description',
    '备注': 'description',
    '说明': 'description',
    '商户名称': 'description',
    '交易对方': 'description',
    '内容': 'description',
    
    // 收款人相关
    '收款方': 'payee',
    '收款人': 'payee',
    '付款方': 'payee',
    '交易对方': 'payee',
    '对方': 'payee',
    '商户': 'payee',
    
    // 金额相关
    '金额': 'amount',
    '交易金额': 'amount',
    '收入金额': 'amount',
    '支出金额': 'amount',
    '金额(元)': 'amount',
    
    // 收支类型相关
    '收支': 'type',
    '类型': 'type',
    '收支类型': 'type',
    '交易类型': 'type',
    '收入': 'type',
    '支出': 'type'
  }
  
  const mappedHeaders = {}
  headers.forEach(header => {
    const cleanHeader = header.trim()
    const mappedName = fieldMapping[cleanHeader] || cleanHeader
    mappedHeaders[cleanHeader] = mappedName
  })
  
  return mappedHeaders
}

// 解析并标准化账单数据
const parseBillData = (rawData, headers) => {
  const detailStartRow = findDetailStartRow(rawData, headers)
  const fieldMapping = mapFieldNames(headers)
  
  // 提取明细数据（跳过总结行）
  const detailData = rawData.slice(detailStartRow)
  
  const standardizedData = detailData.map((row, index) => {
    const standardRow = {
      id: index + 1,
      transactionTime: '',
      description: '',
      payee: '',
      amount: '',
      type: '',
      originalData: row // 保留原始数据用于调试
    }
    
    // 映射字段
    Object.keys(row).forEach(key => {
      const value = String(row[key] || '').trim()
      if (!value) return
      
      const mappedField = fieldMapping[key]
      
      if (mappedField === 'transactionTime') {
        standardRow.transactionTime = value
      } else if (mappedField === 'description') {
        standardRow.description = value
      } else if (mappedField === 'payee') {
        standardRow.payee = value
      } else if (mappedField === 'amount') {
        // 清理金额格式
        const cleanAmount = value.replace(/[￥¥,，\s]/g, '')
        standardRow.amount = cleanAmount
      } else if (mappedField === 'type') {
        // 标准化收支类型
        if (value.includes('收入') || value.includes('入账') || value.includes('+')) {
          standardRow.type = '收入'
        } else if (value.includes('支出') || value.includes('出账') || value.includes('-')) {
          standardRow.type = '支出'
        } else {
          standardRow.type = value
        }
      }
    })
    
    // 如果没有明确的收支类型，根据金额判断
    if (!standardRow.type && standardRow.amount) {
      const amount = parseFloat(standardRow.amount.replace(/[^\d.-]/g, ''))
      if (!isNaN(amount)) {
        standardRow.type = amount >= 0 ? '收入' : '支出'
        standardRow.amount = Math.abs(amount).toString()
      }
    }
    
    return standardRow
  }).filter(row => 
    // 过滤掉空行或无效数据
    row.transactionTime || row.description || row.amount
  )
  
  return standardizedData
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

    // 使用智能账单解析
    const billData = parseBillData(result.data, result.headers)
    
    // 设置标准化的列名
    columns.value = ['交易时间', '交易内容', '收款人/付款方', '金额', '收支类型']
    parsedData.value = billData.map(item => ({
      '交易时间': item.transactionTime,
      '交易内容': item.description,
      '收款人/付款方': item.payee,
      '金额': item.amount,
      '收支类型': item.type
    }))
    
    currentPage.value = 1

    ElMessage.success(`账单解析成功！共解析 ${billData.length} 条明细记录`)
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
  // 计算统计信息
  const totalRecords = parsedData.value.length
  const incomeRecords = parsedData.value.filter(item => item['收支类型'] === '收入')
  const expenseRecords = parsedData.value.filter(item => item['收支类型'] === '支出')
  
  const totalIncome = incomeRecords.reduce((sum, item) => {
    const amount = parseFloat(item['金额']?.replace(/[^\d.-]/g, '') || 0)
    return sum + (isNaN(amount) ? 0 : amount)
  }, 0)
  
  const totalExpense = expenseRecords.reduce((sum, item) => {
    const amount = parseFloat(item['金额']?.replace(/[^\d.-]/g, '') || 0)
    return sum + (isNaN(amount) ? 0 : amount)
  }, 0)
  
  // 构建标准化的导出数据
  const exportObject = {
    metadata: {
      exportTime: new Date().toISOString(),
      fileName: currentFile.value?.name || 'unknown',
      totalRecords: totalRecords,
      summary: {
        totalIncome: totalIncome,
        totalExpense: totalExpense,
        netAmount: totalIncome - totalExpense,
        incomeRecords: incomeRecords.length,
        expenseRecords: expenseRecords.length
      }
    },
    transactions: parsedData.value.map((item, index) => ({
      id: index + 1,
      transactionTime: item['交易时间'] || '',
      description: item['交易内容'] || '',
      payee: item['收款人/付款方'] || '',
      amount: parseFloat(item['金额']?.replace(/[^\d.-]/g, '') || 0),
      type: item['收支类型'] || '',
      originalAmount: item['金额'] || ''
    }))
  }
  
  const dataStr = JSON.stringify(exportObject, null, 2)
  const dataBlob = new Blob([dataStr], { type: 'application/json' })
  
  const link = document.createElement('a')
  link.href = URL.createObjectURL(dataBlob)
  link.download = `bill_analysis_${new Date().getTime()}.json`
  link.click()
  
  ElMessage.success(`数据导出成功！共${totalRecords}条记录，收入${totalIncome.toFixed(2)}元，支出${totalExpense.toFixed(2)}元`)
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

.statistics-card {
  margin-bottom: 20px;
}

.stat-item {
  text-align: center;
  padding: 20px;
  border-radius: 8px;
  background: #f8f9fa;
  transition: all 0.3s ease;
}

.stat-item:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
}

.stat-item.income {
  background: linear-gradient(135deg, #e8f5e8 0%, #c8e6c9 100%);
  border-left: 4px solid #4caf50;
}

.stat-item.expense {
  background: linear-gradient(135deg, #ffeaea 0%, #ffcdd2 100%);
  border-left: 4px solid #f44336;
}

.stat-item.net.positive {
  background: linear-gradient(135deg, #e3f2fd 0%, #bbdefb 100%);
  border-left: 4px solid #2196f3;
}

.stat-item.net.negative {
  background: linear-gradient(135deg, #fff3e0 0%, #ffe0b2 100%);
  border-left: 4px solid #ff9800;
}

.stat-value {
  font-size: 24px;
  font-weight: bold;
  color: #333;
  margin-bottom: 5px;
}

.stat-label {
  font-size: 14px;
  color: #666;
  font-weight: 500;
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

.amount-text {
  font-weight: 600;
  font-family: 'Monaco', 'Menlo', monospace;
}
</style>
