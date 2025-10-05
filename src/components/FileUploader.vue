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
        :on-remove="handleFileRemove"
        :file-list="fileList"
        accept=".xlsx,.xls,.csv"
        multiple
      >
        <el-icon class="el-icon--upload"><upload-filled /></el-icon>
        <div class="el-upload__text">
          将Excel或CSV文件拖拽到此处，或<em>点击上传</em>
        </div>
        <template #tip>
          <div class="el-upload__tip">
            支持 .xlsx, .xls, .csv 格式文件，可同时上传多个文件
          </div>
        </template>
      </el-upload>

      <!-- 文件信息显示 -->
      <div v-if="selectedFiles.length > 0" class="file-info">
        <el-alert
          :title="`已选择 ${selectedFiles.length} 个文件`"
          type="info"
          :closable="false"
        />
        <div class="selected-files">
          <el-tag
            v-for="(file, index) in selectedFiles"
            :key="index"
            :type="getFileTagType(file.name)"
            closable
            @close="removeFile(index)"
            class="file-tag"
          >
            {{ file.name }}
          </el-tag>
        </div>
        <div class="file-actions">
          <el-button type="primary" @click="parseFiles" :loading="parsing">
            解析所有文件
          </el-button>
          <el-button @click="clearFiles">清除所有</el-button>
        </div>
      </div>

      <!-- 解析进度 -->
      <div v-if="parsing" class="parsing-progress">
        <el-progress :percentage="100" :indeterminate="true" />
        <p>正在解析文件...</p>
      </div>
    </el-card>

    <!-- 文件解析结果 -->
    <el-card v-if="parseResults.length > 1" class="parse-results-card">
      <template #header>
        <div class="card-header">
          <span>文件解析结果</span>
        </div>
      </template>
      
      <el-row :gutter="20">
        <el-col :span="8" v-for="(result, index) in parseResults" :key="index">
          <div class="file-result-item">
            <div class="file-result-header">
              <el-tag :type="getFileTagType(result.fileName)" size="small">
                {{ result.billType === 'alipay' ? '支付宝' : result.billType === 'wechat' ? '微信' : '通用' }}
              </el-tag>
              <span class="file-name">{{ getFileDisplayName(result.fileName) }}</span>
            </div>
            <div class="file-result-stats">
              <span class="record-count">{{ result.recordCount }} 条记录</span>
              <div class="encoding-info" v-if="fileEncodings[result.fileName]">
                <el-tag 
                  :type="getEncodingTagType(fileEncodings[result.fileName])" 
                  size="mini"
                  class="encoding-tag"
                >
                  {{ getEncodingDisplayText(fileEncodings[result.fileName]) }}
                </el-tag>
              </div>
            </div>
          </div>
        </el-col>
      </el-row>
    </el-card>

    <!-- 数据统计 -->
    <el-card v-if="parsedData.length > 0" class="statistics-card">
      <template #header>
        <div class="card-header">
          <span>账单统计</span>
        </div>
      </template>
      
      <el-row :gutter="20">
        <el-col :span="4">
          <div class="stat-item">
            <div class="stat-value">{{ parsedData.length }}</div>
            <div class="stat-label">总交易数</div>
          </div>
        </el-col>
        <el-col :span="4">
          <div class="stat-item income">
            <div class="stat-value">{{ totalIncome.toFixed(2) }}</div>
            <div class="stat-label">总收入 (元)</div>
          </div>
        </el-col>
        <el-col :span="4">
          <div class="stat-item expense">
            <div class="stat-value">{{ totalExpense.toFixed(2) }}</div>
            <div class="stat-label">总支出 (元)</div>
          </div>
        </el-col>
        <el-col :span="4" v-if="totalNeutral > 0">
          <div class="stat-item neutral">
            <div class="stat-value">{{ totalNeutral.toFixed(2) }}</div>
            <div class="stat-label">不计收支 (元)</div>
          </div>
        </el-col>
        <el-col :span="4">
          <div class="stat-item net" :class="{ positive: netAmount >= 0, negative: netAmount < 0 }">
            <div class="stat-value">{{ netAmount.toFixed(2) }}</div>
            <div class="stat-label">净收入 (元)</div>
          </div>
        </el-col>
      </el-row>
    </el-card>

    <!-- 统计视图 -->
    <StatisticsView 
      v-if="parsedData.length > 0" 
      :data="parsedData" 
      :parse-results="parseResults"
    />

    <!-- 数据预览 -->
    <el-card v-if="parsedData.length > 0" class="data-preview">
      <template #header>
        <div class="card-header">
          <span>明细数据 (共 {{ parsedData.length }} 条记录)</span>
          <div class="header-actions">
            <el-button-group size="small">
              <el-button
                :type="showTable ? 'primary' : ''"
                @click="showTable = !showTable"
              >
                {{ showTable ? '隐藏' : '显示' }}明细
              </el-button>
            </el-button-group>
            <el-select
              v-model="pageSize"
              size="small"
              style="margin-left: 10px; width: 120px;"
              :disabled="!showTable"
              placeholder="每页数量"
            >
              <el-option v-for="size in [10, 20, 50, 100, 200]" :key="size" :label="`${size} 条/页`" :value="size" />
            </el-select>
            <el-button type="primary" size="small" @click="exportData" style="margin-left: 10px;">
              导出JSON
            </el-button>
          </div>
        </div>
      </template>
      
      <el-table
        v-if="showTable"
        :data="displayData"
        style="width: 100%"
        max-height="500"
        stripe
        border
      >
        <el-table-column
          prop="交易时间"
          label="交易时间"
          min-width="120"
          show-overflow-tooltip
        />
        <el-table-column
          prop="交易类型"
          label="交易类型"
          min-width="100"
          show-overflow-tooltip
        >
          <template #default="scope">
            <el-tag 
              v-if="scope.row['交易类型']"
              type="info" 
              size="small"
              class="transaction-type-tag"
            >
              {{ scope.row['交易类型'] }}
            </el-tag>
            <span v-else class="no-data">-</span>
          </template>
        </el-table-column>
        <el-table-column
          prop="交易内容"
          label="交易内容"
          min-width="160"
          show-overflow-tooltip
        />
        <el-table-column
          prop="收款人/付款方"
          label="收款人/付款方"
          min-width="120"
          show-overflow-tooltip
        />
        <el-table-column
          prop="支付方式"
          label="支付方式"
          min-width="180"
          show-overflow-tooltip
        >
          <template #default="scope">
            <el-tag 
              v-if="scope.row['支付方式']"
              :type="getPaymentMethodTagType(scope.row['支付方式'])" 
              size="small"
              class="payment-method-tag"
            >
              {{ scope.row['支付方式'] }}
            </el-tag>
            <span v-else class="no-data">-</span>
          </template>
        </el-table-column>
        <el-table-column
          prop="金额"
          label="金额"
          min-width="80"
          align="right"
        >
          <template #default="scope">
            <span class="amount-text">{{ scope.row['金额'] }}</span>
          </template>
        </el-table-column>
        <el-table-column
          prop="收支类型"
          label="收支类型"
          min-width="80"
          align="center"
        >
          <template #default="scope">
            <el-tag 
              :type="getTagType(scope.row['收支类型'])"
              size="small"
            >
              {{ scope.row['收支类型'] }}
            </el-tag>
          </template>
        </el-table-column>
        <el-table-column
          v-if="parseResults.length > 1"
          prop="来源文件"
          label="来源文件"
          min-width="120"
          show-overflow-tooltip
        >
          <template #default="scope">
            <el-tag 
              :type="getFileTagType(scope.row['来源文件'])"
              size="small"
            >
              {{ scope.row['来源文件'] }}
            </el-tag>
          </template>
        </el-table-column>
      </el-table>
      
      <!-- 分页 -->
      <div class="pagination-wrapper" v-if="showTable && parsedData.length > pageSize">
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
import StatisticsView from './StatisticsView.vue'
import wechatMappingConfig from '../config/wechat-transaction-mapping.json'

// 响应式数据
const selectedFiles = ref([])
const fileList = ref([])
const parsing = ref(false)
const parsedData = ref([])
const columns = ref([])
const errorMessage = ref('')
const currentPage = ref(1)
const pageSize = ref(50)
const parseResults = ref([]) // 存储每个文件的解析结果
const showTable = ref(false) // 控制明细表格显示
const fileEncodings = ref({}) // 存储每个文件的编码信息

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

const totalNeutral = computed(() => {
  return parsedData.value
    .filter(item => item['收支类型'] === '不计收支')
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
  selectedFiles.value.push(file.raw)
  errorMessage.value = ''
}

// 文件移除处理
const handleFileRemove = (file) => {
  const index = selectedFiles.value.findIndex(f => f.name === file.name)
  if (index > -1) {
    selectedFiles.value.splice(index, 1)
  }
}

// 移除单个文件
const removeFile = (index) => {
  selectedFiles.value.splice(index, 1)
  fileList.value.splice(index, 1)
}

// 清除所有文件
const clearFiles = () => {
  selectedFiles.value = []
  fileList.value = []
  parsedData.value = []
  columns.value = []
  parseResults.value = []
  fileEncodings.value = {}
  errorMessage.value = ''
  currentPage.value = 1
}

// 获取文件标签类型
const getFileTagType = (fileName) => {
  const fileNameLower = fileName.toLowerCase()
  if (fileNameLower.includes('支付宝') || fileNameLower.includes('alipay')) {
    return 'success'
  }
  if (fileNameLower.includes('微信') || fileNameLower.includes('wechat')) {
    return 'primary'
  }
  return 'info'
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

  const headers = parseCSVLine(lines[22])
  const data = []

  for (let i = 23; i < lines.length; i++) {
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

  const headers = jsonData[16].map(header => String(header || ''))
  const data = []

  for (let i = 17; i < jsonData.length; i++) {
    const row = {}
    headers.forEach((header, index) => {
      row[header] = jsonData[i][index] || ''
    })
    data.push(row)
  }

  return { headers, data }
}

// 检测账单类型
const detectBillType = (data, fileName) => {
  const fileNameLower = fileName.toLowerCase()
  
  // 根据文件名判断
  if (fileNameLower.includes('支付宝') || fileNameLower.includes('alipay')) {
    return 'alipay'
  }
  if (fileNameLower.includes('微信') || fileNameLower.includes('wechat')) {
    return 'wechat'
  }
  
  // 根据内容判断
  const firstFewRows = data.slice(0, 10).map(row => 
    Object.values(row).join('').toLowerCase()
  ).join('')
  
  if (firstFewRows.includes('支付宝') || firstFewRows.includes('alipay')) {
    return 'alipay'
  }
  if (firstFewRows.includes('微信支付') || firstFewRows.includes('wechat')) {
    return 'wechat'
  }
  
  return 'generic'
}

// 智能识别账单明细的开始行
const findDetailStartRow = (data, headers, billType) => {
  // 针对特定平台的明细行识别
  if (billType === 'alipay') {
    // 支付宝：查找包含"交易时间"的行
    for (let i = 0; i < data.length; i++) {
      const row = data[i]
      const keys = Object.keys(row)
      if (keys.includes('交易时间') && keys.includes('交易分类') && keys.includes('收/支')) {
        return i
      }
    }
  } else if (billType === 'wechat') {
    // 微信：查找包含"交易时间"的行
    for (let i = 0; i < data.length; i++) {
      const row = data[i]
      const keys = Object.keys(row)
      if (keys.includes('交易时间') && keys.includes('交易类型') && keys.includes('收/支')) {
        return i
      }
    }
  }
  
  // 通用方法：常见的明细标识关键词
  const detailKeywords = ['交易时间', '交易日期', '日期', '时间', '商户', '收款方', '付款方', '金额', '收支', '类型', '备注', '说明']
  
  for (let i = 0; i < Math.min(data.length, 30); i++) { // 检查前30行
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
const mapFieldNames = (headers, billType) => {
  const fieldMapping = {
    // 交易时间相关
    '交易时间': 'transactionTime',
    '交易日期': 'transactionTime', 
    '日期': 'transactionTime',
    '时间': 'transactionTime',
    '成交时间': 'transactionTime',
    
    // 交易内容相关 - 支付宝
    '商品说明': 'description',
    '交易说明': 'description',
    '交易备注': 'description',
    '备注': 'description',
    '说明': 'description',
    '商户名称': 'description',
    '内容': 'description',
    
    // 交易内容相关 - 微信
    '商品': 'description',
    '交易类型': 'description', // 微信的交易类型也可以作为描述的一部分
    
    // 收款人相关 - 支付宝
    '交易对方': 'payee',
    '收款方': 'payee',
    '收款人': 'payee',
    '付款方': 'payee',
    '对方': 'payee',
    '商户': 'payee',
    
    // 金额相关
    '金额': 'amount',
    '交易金额': 'amount',
    '收入金额': 'amount',
    '支出金额': 'amount',
    '金额(元)': 'amount',
    
    // 收支类型相关
    '收/支': 'type',
    '收支': 'type',
    '类型': 'type',
    '收支类型': 'type',
    '收入': 'type',
    '支出': 'type',
    
    // 支付方式相关
    '收/付款方式': 'paymentMethod',
    '支付方式': 'paymentMethod',
    '付款方式': 'paymentMethod',
    '收款方式': 'paymentMethod'
  }
  
  // 根据账单类型调整映射策略
  if (billType === 'alipay') {
    // 支付宝特殊处理
    fieldMapping['交易分类'] = 'transactionType' // 支付宝的交易分类映射为交易类型
  } else if (billType === 'wechat') {
    // 微信特殊处理
    fieldMapping['交易类型'] = 'transactionType' // 微信的交易类型映射为交易类型
  }
  
  const mappedHeaders = {}
  headers.forEach(header => {
    const cleanHeader = header.trim()
    const mappedName = fieldMapping[cleanHeader] || cleanHeader
    mappedHeaders[cleanHeader] = mappedName
  })
  
  return mappedHeaders
}

// 构建微信交易类型映射表（从JSON配置生成）
const buildWechatTransactionTypeMapping = () => {
  const mapping = {}
  
  // 遍历每个分类，将商户名映射到分类
  Object.keys(wechatMappingConfig).forEach(category => {
    if (category !== '关键词推断' && Array.isArray(wechatMappingConfig[category])) {
      wechatMappingConfig[category].forEach(merchant => {
        mapping[merchant] = category
      })
    }
  })
  
  return mapping
}

// 生成映射表
const wechatTransactionTypeMapping = buildWechatTransactionTypeMapping()

// 根据交易对方智能映射交易类型
const mapTransactionTypeByPayee = (payee, originalType, billType) => {
  if (billType !== 'wechat' || !payee) {
    return originalType
  }
  
  // 清理交易对方名称，去除特殊字符和空格
  const cleanPayee = payee.replace(/[()（）\[\]【】\s]/g, '').trim()
  
  // 精确匹配
  if (wechatTransactionTypeMapping[cleanPayee]) {
    console.log(`精确匹配: ${payee} -> ${wechatTransactionTypeMapping[cleanPayee]}`)
    return wechatTransactionTypeMapping[cleanPayee]
  }
  
  // 模糊匹配 - 优先匹配更长的关键词
  const sortedKeywords = Object.keys(wechatTransactionTypeMapping).sort((a, b) => b.length - a.length)
  
  for (const keyword of sortedKeywords) {
    const cleanKeyword = keyword.replace(/[()（）\[\]【】\s]/g, '').trim()
    
    // 检查交易对方是否包含关键词
    if (cleanPayee.includes(cleanKeyword)) {
      console.log(`模糊匹配: ${payee} 包含 ${keyword} -> ${wechatTransactionTypeMapping[keyword]}`)
      return wechatTransactionTypeMapping[keyword]
    }
    
    // 检查关键词是否包含在交易对方中（适用于简化的商户名）
    if (cleanKeyword.includes(cleanPayee) && cleanPayee.length >= 2) {
      console.log(`反向匹配: ${keyword} 包含 ${payee} -> ${wechatTransactionTypeMapping[keyword]}`)
      return wechatTransactionTypeMapping[keyword]
    }
  }
  
  // 基于关键词的智能推断
  const payeeLower = cleanPayee.toLowerCase()
  const keywordInference = wechatMappingConfig['关键词推断']
  
  // 遍历所有分类的关键词进行匹配
  for (const [category, keywords] of Object.entries(keywordInference)) {
    if (Array.isArray(keywords)) {
      for (const keyword of keywords) {
        if (payeeLower.includes(keyword)) {
          console.log(`关键词推断: ${payee} 包含关键词 "${keyword}" -> ${category}`)
          return category
        }
      }
    }
  }
  
  // 如果没有匹配到，返回原始类型
  return originalType || '其他消费'
}

// 解析并标准化账单数据
const parseBillData = (rawData, headers, fileName) => {
  const billType = detectBillType(rawData, fileName)
  const detailStartRow = findDetailStartRow(rawData, headers, billType)
  const fieldMapping = mapFieldNames(headers, billType)
  
  console.log('检测到账单类型:', billType)
  console.log('明细开始行:', detailStartRow)
  console.log('字段映射:', fieldMapping)
  
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
      transactionType: '', // 交易类型字段
      paymentMethod: '', // 支付方式字段
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
        // 组合描述信息
        if (standardRow.description) {
          standardRow.description += ' - ' + value
        } else {
          standardRow.description = value
        }
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
        } else if (value.includes('不计收支') || value.includes('中性交易')) {
          standardRow.type = '不计收支'
        } else {
          standardRow.type = value
        }
      } else if (mappedField === 'transactionType') {
        standardRow.transactionType = value
      } else if (mappedField === 'paymentMethod') {
        // 清理和标准化支付方式信息
        standardRow.paymentMethod = value.replace(/[()（）]/g, '').trim()
      }
    })
    
    // 应用自定义交易类型映射（基于交易对方）
    standardRow.transactionType = mapTransactionTypeByPayee(
      standardRow.payee, 
      standardRow.transactionType, 
      billType
    )
    
    // 特殊处理：如果描述为空，使用交易类型作为描述
    if (!standardRow.description && standardRow.transactionType) {
      standardRow.description = standardRow.transactionType
    }
    
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
  
  return { data: standardizedData, billType }
}

// 解析单个文件
const parseSingleFile = async (file) => {
  const fileExtension = file.name.split('.').pop().toLowerCase()
  let result

  if (fileExtension === 'csv') {
    // 解析CSV文件
    const text = await readFileAsText(file)
    result = parseCSV(text)
  } else if (['xlsx', 'xls'].includes(fileExtension)) {
    // 解析Excel文件
    const arrayBuffer = await readFileAsArrayBuffer(file)
    result = parseExcel(arrayBuffer)
  } else {
    throw new Error(`不支持的文件格式: ${fileExtension}`)
  }

  // 使用智能账单解析
  const parseResult = parseBillData(result.data, result.headers, file.name)
  const billData = parseResult.data
  const billType = parseResult.billType
  
  return {
    fileName: file.name,
    billType,
    data: billData,
    recordCount: billData.length
  }
}

// 解析多个文件主函数
const parseFiles = async () => {
  if (selectedFiles.value.length === 0) {
    ElMessage.error('请先选择文件')
    return
  }

  parsing.value = true
  errorMessage.value = ''
  parseResults.value = []

  try {
    const results = []
    let totalRecords = 0
    
    // 并行解析所有文件
    const parsePromises = selectedFiles.value.map(file => parseSingleFile(file))
    const fileResults = await Promise.all(parsePromises)
    
    // 合并所有解析结果
    const allTransactions = []
    fileResults.forEach((result, index) => {
      results.push(result)
      totalRecords += result.recordCount
      
      // 为每条记录添加来源文件信息
      result.data.forEach(item => {
        allTransactions.push({
          ...item,
          sourceFile: result.fileName,
          sourcePlatform: result.billType
        })
      })
    })
    
    // 按时间排序（如果有时间信息）
    allTransactions.sort((a, b) => {
      const timeA = new Date(a.transactionTime || 0)
      const timeB = new Date(b.transactionTime || 0)
      return timeB - timeA // 降序排列，最新的在前
    })
    
    // 设置标准化的列名
    columns.value = ['交易时间', '交易类型', '交易内容', '收款人/付款方', '支付方式', '金额', '收支类型', '来源文件']
    parsedData.value = allTransactions.map(item => ({
      '交易时间': item.transactionTime,
      '交易类型': item.transactionType,
      '交易内容': item.description,
      '收款人/付款方': item.payee,
      '支付方式': item.paymentMethod,
      '金额': item.amount,
      '收支类型': item.type,
      '来源文件': getFileDisplayName(item.sourceFile)
    }))
    
    parseResults.value = results
    currentPage.value = 1

    // 生成解析成功消息
    const platformCounts = {}
    results.forEach(result => {
      const platformName = result.billType === 'alipay' ? '支付宝' : result.billType === 'wechat' ? '微信' : '通用'
      platformCounts[platformName] = (platformCounts[platformName] || 0) + result.recordCount
    })
    
    const platformSummary = Object.entries(platformCounts)
      .map(([platform, count]) => `${platform}: ${count}条`)
      .join('，')
    
    // 统计微信交易类型映射情况
    let mappingStats = ''
    if (results.some(r => r.billType === 'wechat')) {
      const wechatTransactions = allTransactions.filter(t => t.sourcePlatform === 'wechat')
      const mappedCount = wechatTransactions.filter(t => t.transactionType && t.transactionType !== '其他消费').length
      mappingStats = `，微信交易类型映射: ${mappedCount}/${wechatTransactions.length}`
    }
    
    ElMessage.success(`多文件解析成功！共解析 ${totalRecords} 条记录 (${platformSummary}${mappingStats})`)
  } catch (error) {
    console.error('文件解析错误:', error)
    errorMessage.value = `解析失败: ${error.message}`
    ElMessage.error('文件解析失败')
  } finally {
    parsing.value = false
  }
}

// 获取文件显示名称
const getFileDisplayName = (fileName) => {
  if (fileName.length > 20) {
    return fileName.substring(0, 10) + '...' + fileName.substring(fileName.length - 10)
  }
  return fileName
}

// 编码检测函数
const detectEncoding = (buffer) => {
  const bytes = new Uint8Array(buffer)
  
  // 检测BOM (Byte Order Mark)
  if (bytes.length >= 3 && bytes[0] === 0xEF && bytes[1] === 0xBB && bytes[2] === 0xBF) {
    return 'UTF-8'
  }
  if (bytes.length >= 2 && bytes[0] === 0xFF && bytes[1] === 0xFE) {
    return 'UTF-16LE'
  }
  if (bytes.length >= 2 && bytes[0] === 0xFE && bytes[1] === 0xFF) {
    return 'UTF-16BE'
  }
  
  // 检测UTF-8编码特征
  let utf8Score = 0
  let gbkScore = 0
  let totalBytes = Math.min(bytes.length, 1000) // 只检查前1000字节
  
  for (let i = 0; i < totalBytes; i++) {
    const byte = bytes[i]
    
    // ASCII字符 (0-127)
    if (byte <= 0x7F) {
      utf8Score += 1
      gbkScore += 1
      continue
    }
    
    // UTF-8 多字节序列检测
    if (byte >= 0xC0 && byte <= 0xDF && i + 1 < totalBytes) {
      // 2字节UTF-8序列
      const next = bytes[i + 1]
      if (next >= 0x80 && next <= 0xBF) {
        utf8Score += 3
        i++ // 跳过下一个字节
        continue
      }
    }
    
    if (byte >= 0xE0 && byte <= 0xEF && i + 2 < totalBytes) {
      // 3字节UTF-8序列
      const next1 = bytes[i + 1]
      const next2 = bytes[i + 2]
      if (next1 >= 0x80 && next1 <= 0xBF && next2 >= 0x80 && next2 <= 0xBF) {
        utf8Score += 4
        i += 2 // 跳过接下来的两个字节
        continue
      }
    }
    
    // GBK编码检测 (简化版)
    if (byte >= 0x81 && byte <= 0xFE && i + 1 < totalBytes) {
      const next = bytes[i + 1]
      if ((next >= 0x40 && next <= 0x7E) || (next >= 0x80 && next <= 0xFE)) {
        gbkScore += 2
        i++ // 跳过下一个字节
        continue
      }
    }
    
    // 其他高位字节，可能是GBK
    if (byte >= 0x80) {
      gbkScore += 1
    }
  }
  
  // 根据得分判断编码
  if (utf8Score > gbkScore * 1.2) {
    return 'UTF-8'
  } else if (gbkScore > utf8Score) {
    return 'GBK'
  }
  
  return 'UTF-8' // 默认UTF-8
}

// 验证文本内容是否正确解码
const validateTextContent = (text) => {
  // 检查是否包含大量乱码字符
  const invalidChars = text.match(/[�]/g)
  const invalidRatio = invalidChars ? invalidChars.length / text.length : 0
  
  // 如果乱码字符超过5%，认为解码失败
  if (invalidRatio > 0.05) {
    return false
  }
  
  // 检查是否包含常见的中文字符
  const chineseChars = text.match(/[\u4e00-\u9fff]/g)
  const hasValidContent = chineseChars || text.match(/[a-zA-Z0-9]/g)
  
  return hasValidContent !== null
}

// 增强的文件读取函数
const readFileAsText = async (file) => {
  // 支持的编码列表，按优先级排序
  const encodings = ['UTF-8', 'GBK', 'GB2312', 'UTF-16LE', 'UTF-16BE', 'ISO-8859-1']
  
  try {
    // 首先读取文件为ArrayBuffer进行编码检测
    const arrayBuffer = await readFileAsArrayBuffer(file)
    const detectedEncoding = detectEncoding(arrayBuffer)
    
    console.log(`检测到文件编码: ${detectedEncoding}`)
    
    // 尝试使用检测到的编码读取文件
    try {
      const text = await readFileWithEncoding(file, detectedEncoding)
      if (validateTextContent(text)) {
        console.log(`成功使用 ${detectedEncoding} 编码读取文件`)
        // 记录成功使用的编码
        fileEncodings.value[file.name] = {
          detected: detectedEncoding,
          used: detectedEncoding,
          success: true
        }
        return text
      }
    } catch (error) {
      console.warn(`使用检测编码 ${detectedEncoding} 读取失败:`, error.message)
    }
    
    // 如果检测的编码失败，尝试其他编码
    for (const encoding of encodings) {
      if (encoding === detectedEncoding) continue // 跳过已经尝试过的编码
      
      try {
        const text = await readFileWithEncoding(file, encoding)
        if (validateTextContent(text)) {
          console.log(`成功使用备用编码 ${encoding} 读取文件`)
          // 记录成功使用的编码
          fileEncodings.value[file.name] = {
            detected: detectedEncoding,
            used: encoding,
            success: true
          }
          return text
        }
      } catch (error) {
        console.warn(`编码 ${encoding} 读取失败:`, error.message)
        continue
      }
    }
    
    // 如果所有编码都失败，使用UTF-8作为最后的尝试
    console.warn('所有编码尝试失败，使用UTF-8强制读取')
    const text = await readFileWithEncoding(file, 'UTF-8')
    // 记录强制使用的编码
    fileEncodings.value[file.name] = {
      detected: detectedEncoding,
      used: 'UTF-8',
      success: false,
      fallback: true
    }
    return text
    
  } catch (error) {
    console.error('文件读取完全失败:', error)
    // 记录失败的编码信息
    fileEncodings.value[file.name] = {
      detected: 'unknown',
      used: 'none',
      success: false,
      error: error.message
    }
    throw new Error(`文件读取失败: ${error.message}`)
  }
}

// GBK编码转换 (简化版本，支持常用汉字)
const gbkToUtf8 = (buffer) => {
  const bytes = new Uint8Array(buffer)
  let result = ''
  
  for (let i = 0; i < bytes.length; i++) {
    const byte = bytes[i]
    
    if (byte <= 0x7F) {
      // ASCII字符
      result += String.fromCharCode(byte)
    } else if (byte >= 0x81 && byte <= 0xFE && i + 1 < bytes.length) {
      // GBK双字节字符
      const byte2 = bytes[i + 1]
      if ((byte2 >= 0x40 && byte2 <= 0x7E) || (byte2 >= 0x80 && byte2 <= 0xFE)) {
        // 这里使用一个简化的转换，实际应用中可能需要完整的GBK码表
        // 对于常见的中文字符，我们尝试使用TextDecoder的fallback
        try {
          const gbkBytes = new Uint8Array([byte, byte2])
          const decoder = new TextDecoder('gbk', { fatal: false })
          const char = decoder.decode(gbkBytes)
          if (char && char !== '�') {
            result += char
          } else {
            // 如果解码失败，保留原始字节
            result += String.fromCharCode(byte, byte2)
          }
        } catch {
          // 如果TextDecoder不支持GBK，使用备用方案
          result += String.fromCharCode(byte, byte2)
        }
        i++ // 跳过下一个字节
      } else {
        result += String.fromCharCode(byte)
      }
    } else {
      result += String.fromCharCode(byte)
    }
  }
  
  return result
}

// 使用指定编码读取文件
const readFileWithEncoding = async (file, encoding) => {
  return new Promise((resolve, reject) => {
    const reader = new FileReader()
    
    if (encoding === 'GBK' || encoding === 'GB2312') {
      // 对于GBK/GB2312编码，先读取为ArrayBuffer然后转换
      reader.onload = (e) => {
        try {
          const buffer = e.target.result
          
          // 尝试使用TextDecoder
          try {
            const decoder = new TextDecoder('gbk', { fatal: false })
            const text = decoder.decode(buffer)
            if (text && !text.includes('�')) {
              resolve(text)
              return
            }
          } catch (error) {
            console.warn('TextDecoder GBK解码失败，使用备用方案')
          }
          
          // 备用方案：简化的GBK转换
          const text = gbkToUtf8(buffer)
          resolve(text)
        } catch (error) {
          reject(new Error(`GBK编码解析失败: ${error.message}`))
        }
      }
      
      reader.onerror = () => {
        reject(new Error(`文件读取错误: ${reader.error?.message || '未知错误'}`))
      }
      
      reader.readAsArrayBuffer(file)
    } else {
      // 其他编码使用标准方法
      reader.onload = (e) => {
        try {
          resolve(e.target.result)
        } catch (error) {
          reject(new Error(`编码 ${encoding} 解析失败: ${error.message}`))
        }
      }
      
      reader.onerror = () => {
        reject(new Error(`文件读取错误: ${reader.error?.message || '未知错误'}`))
      }
      
      reader.onabort = () => {
        reject(new Error('文件读取被中断'))
      }
      
      try {
        // 对于一些浏览器不支持的编码，使用UTF-8作为备用
        const supportedEncodings = ['UTF-8', 'UTF-16LE', 'UTF-16BE', 'ISO-8859-1']
        const actualEncoding = supportedEncodings.includes(encoding) ? encoding : 'UTF-8'
        
        reader.readAsText(file, actualEncoding)
      } catch (error) {
        reject(new Error(`不支持的编码格式: ${encoding}`))
      }
    }
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

// 获取标签类型
const getTagType = (type) => {
  switch (type) {
    case '收入':
      return 'success'
    case '支出':
      return 'danger'
    case '不计收支':
      return 'info'
    default:
      return 'info'
  }
}

// 获取编码标签类型
const getEncodingTagType = (encodingInfo) => {
  if (!encodingInfo) return 'info'
  
  if (encodingInfo.success && encodingInfo.detected === encodingInfo.used) {
    return 'success' // 检测正确且成功
  } else if (encodingInfo.success && encodingInfo.detected !== encodingInfo.used) {
    return 'warning' // 检测错误但成功使用备用编码
  } else if (encodingInfo.fallback) {
    return 'warning' // 强制使用UTF-8
  } else {
    return 'danger' // 失败
  }
}

// 获取编码显示文本
const getEncodingDisplayText = (encodingInfo) => {
  if (!encodingInfo) return '未知编码'
  
  if (encodingInfo.success && encodingInfo.detected === encodingInfo.used) {
    return encodingInfo.used
  } else if (encodingInfo.success && encodingInfo.detected !== encodingInfo.used) {
    return `${encodingInfo.used} (检测: ${encodingInfo.detected})`
  } else if (encodingInfo.fallback) {
    return `${encodingInfo.used} (强制)`
  } else {
    return '解码失败'
  }
}

// 获取支付方式标签类型
const getPaymentMethodTagType = (paymentMethod) => {
  if (!paymentMethod) return 'info'
  
  const method = paymentMethod.toLowerCase()
  
  // 银行卡相关
  if (method.includes('银行') || method.includes('储蓄卡') || method.includes('信用卡')) {
    return 'primary'
  }
  // 余额相关
  if (method.includes('余额') || method.includes('零钱')) {
    return 'success'
  }
  // 理财相关
  if (method.includes('余额宝') || method.includes('理财') || method.includes('零钱通')) {
    return 'warning'
  }
  // 其他支付方式
  return 'info'
}

// 导出数据
const exportData = () => {
  // 计算统计信息
  const totalRecords = parsedData.value.length
  const incomeRecords = parsedData.value.filter(item => item['收支类型'] === '收入')
  const expenseRecords = parsedData.value.filter(item => item['收支类型'] === '支出')
  const neutralRecords = parsedData.value.filter(item => item['收支类型'] === '不计收支')
  
  const totalIncomeAmount = incomeRecords.reduce((sum, item) => {
    const amount = parseFloat(item['金额']?.replace(/[^\d.-]/g, '') || 0)
    return sum + (isNaN(amount) ? 0 : amount)
  }, 0)
  
  const totalExpenseAmount = expenseRecords.reduce((sum, item) => {
    const amount = parseFloat(item['金额']?.replace(/[^\d.-]/g, '') || 0)
    return sum + (isNaN(amount) ? 0 : amount)
  }, 0)
  
  const totalNeutralAmount = neutralRecords.reduce((sum, item) => {
    const amount = parseFloat(item['金额']?.replace(/[^\d.-]/g, '') || 0)
    return sum + (isNaN(amount) ? 0 : amount)
  }, 0)
  
  // 构建标准化的导出数据
  const exportObject = {
    metadata: {
      exportTime: new Date().toISOString(),
      fileCount: parseResults.value.length || 1,
      files: parseResults.value.length > 0 ? parseResults.value.map(r => ({
        fileName: r.fileName,
        billType: r.billType,
        recordCount: r.recordCount
      })) : [{ fileName: 'unknown', billType: 'unknown', recordCount: totalRecords }],
      totalRecords: totalRecords,
      summary: {
        totalIncome: totalIncomeAmount,
        totalExpense: totalExpenseAmount,
        totalNeutral: totalNeutralAmount,
        netAmount: totalIncomeAmount - totalExpenseAmount,
        incomeRecords: incomeRecords.length,
        expenseRecords: expenseRecords.length,
        neutralRecords: neutralRecords.length
      }
    },
    transactions: parsedData.value.map((item, index) => ({
      id: index + 1,
      transactionTime: item['交易时间'] || '',
      transactionType: item['交易类型'] || '',
      description: item['交易内容'] || '',
      payee: item['收款人/付款方'] || '',
      paymentMethod: item['支付方式'] || '',
      amount: parseFloat(item['金额']?.replace(/[^\d.-]/g, '') || 0),
      type: item['收支类型'] || '',
      sourceFile: item['来源文件'] || '',
      originalAmount: item['金额'] || ''
    }))
  }
  
  const dataStr = JSON.stringify(exportObject, null, 2)
  const dataBlob = new Blob([dataStr], { type: 'application/json' })
  
  const link = document.createElement('a')
  link.href = URL.createObjectURL(dataBlob)
  link.download = `bill_analysis_${new Date().getTime()}.json`
  link.click()
  
  const neutralText = totalNeutralAmount > 0 ? `，不计收支${totalNeutralAmount.toFixed(2)}元` : ''
  ElMessage.success(`数据导出成功！共${totalRecords}条记录，收入${totalIncomeAmount.toFixed(2)}元，支出${totalExpenseAmount.toFixed(2)}元${neutralText}`)
}
</script>

<style scoped>
.file-uploader {
  max-width: 100vw;
  margin-left: auto;
  margin-right: auto;
  padding-left: 32px;
  padding-right: 32px;
  padding-top: 20px;
  padding-bottom: 20px;
  box-sizing: border-box;
}

.upload-card {
  margin-bottom: 20px;
}

.parse-results-card {
  margin-bottom: 20px;
}

.statistics-card {
  margin-bottom: 20px;
}

.selected-files {
  margin: 15px 0;
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
}

.file-tag {
  margin: 2px;
}

.file-result-item {
  padding: 15px;
  border: 1px solid #e4e7ed;
  border-radius: 8px;
  background: #fafafa;
  transition: all 0.3s ease;
}

.file-result-item:hover {
  background: #f0f9ff;
  border-color: #409eff;
}

.file-result-header {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 8px;
}

.file-name {
  font-size: 14px;
  color: #606266;
  font-weight: 500;
}

.file-result-stats {
  text-align: center;
}

.record-count {
  font-size: 16px;
  font-weight: bold;
  color: #409eff;
}

.encoding-info {
  margin-top: 8px;
  text-align: center;
}

.encoding-tag {
  font-size: 11px;
}

.transaction-type-tag {
  background-color: #f0f9ff;
  border-color: #0ea5e9;
  color: #0c4a6e;
}

.no-data {
  color: #9ca3af;
  font-style: italic;
}

.payment-method-tag {
  font-size: 12px;
  max-width: 100%;
  word-break: break-all;
  white-space: normal;
  overflow: visible;
  text-overflow: unset;
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

.stat-item.neutral {
  background: linear-gradient(135deg, #f3e5f5 0%, #e1bee7 100%);
  border-left: 4px solid #9c27b0;
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

.header-actions {
  display: flex;
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
