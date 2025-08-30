<template>
  <div class="statistics-view">
    <!-- 统计概览 -->
    <el-card class="overview-card">
      <template #header>
        <div class="card-header">
          <span>统计概览</span>
          <el-button-group size="small">
            <el-button 
              :type="viewMode === 'charts' ? 'primary' : ''"
              @click="viewMode = 'charts'"
            >
              图表视图
            </el-button>
            <el-button 
              :type="viewMode === 'tables' ? 'primary' : ''"
              @click="viewMode = 'tables'"
            >
              表格视图
            </el-button>
          </el-button-group>
        </div>
      </template>

      <!-- 图表视图 -->
      <div v-if="viewMode === 'charts'" class="charts-container">
        <el-row :gutter="20">
          <!-- 收支饼图 -->
          <el-col :span="12">
            <div class="chart-wrapper">
              <h4>收支分布</h4>
              <Pie :data="incomeExpenseData" :options="pieOptions" />
            </div>
          </el-col>
          
          <!-- 平台对比柱状图 -->
          <el-col :span="12">
            <div class="chart-wrapper">
              <h4>平台对比</h4>
              <Bar :data="platformComparisonData" :options="barOptions" />
            </div>
          </el-col>
        </el-row>

        <el-row :gutter="20" style="margin-top: 20px;">
          <!-- 时间趋势图 -->
          <el-col :span="24">
            <div class="chart-wrapper">
              <h4>交易趋势 (按日)</h4>
              <Line :data="timeTrendData" :options="lineOptions" />
            </div>
          </el-col>
        </el-row>

        <el-row :gutter="20" style="margin-top: 20px;">
          <!-- 分类统计 -->
          <el-col :span="6">
            <div class="chart-wrapper">
              <h4>支出分类 (Top 8)</h4>
              <Doughnut :data="categoryData" :options="doughnutOptions" />
            </div>
          </el-col>
          
          <!-- 交易类型统计 -->
          <el-col :span="6">
            <div class="chart-wrapper">
              <h4>交易类型分布</h4>
              <Pie :data="transactionTypeData" :options="pieOptions" />
            </div>
          </el-col>
          
          <!-- 支付方式统计 -->
          <el-col :span="6">
            <div class="chart-wrapper">
              <h4>支付方式分布</h4>
              <Pie :data="paymentMethodData" :options="pieOptions" />
            </div>
          </el-col>
          
          <!-- 金额分布 -->
          <el-col :span="6">
            <div class="chart-wrapper">
              <h4>交易金额分布</h4>
              <Bar :data="amountDistributionData" :options="amountBarOptions" />
            </div>
          </el-col>
        </el-row>
      </div>

      <!-- 表格视图 -->
      <div v-if="viewMode === 'tables'" class="tables-container">
        <!-- 平台统计表 -->
        <div class="table-section">
          <div class="table-header">
            <h4>平台统计</h4>
            <el-tag size="small" type="info">{{ platformStats.length }} 个平台</el-tag>
          </div>
          <el-table :data="platformStats" stripe border class="stats-table">
            <el-table-column prop="platform" label="平台" width="120">
              <template #default="scope">
                <el-tag :type="getPlatformTagType(scope.row.platform)">
                  {{ scope.row.platform }}
                </el-tag>
              </template>
            </el-table-column>
            <el-table-column prop="count" label="交易数" align="right" width="100" />
            <el-table-column prop="totalAmount" label="总金额 (元)" align="right" width="150">
              <template #default="scope">
                <span class="amount-text">{{ scope.row.totalAmount.toFixed(2) }}</span>
              </template>
            </el-table-column>
            <el-table-column prop="avgAmount" label="平均金额 (元)" align="right" width="150">
              <template #default="scope">
                <span class="amount-text">{{ scope.row.avgAmount.toFixed(2) }}</span>
              </template>
            </el-table-column>
            <el-table-column label="占比" align="right" width="100">
              <template #default="scope">
                <span class="percentage-text">{{ ((scope.row.totalAmount / totalAmount) * 100).toFixed(1) }}%</span>
              </template>
            </el-table-column>
            <el-table-column prop="totalAmount" label="金额条形图" min-width="200">
              <template #default="scope">
                <div class="progress-bar">
                  <div 
                    class="progress-fill" 
                    :style="{ 
                      width: ((scope.row.totalAmount / maxPlatformAmount) * 100) + '%',
                      backgroundColor: getPlatformColor(scope.row.platform)
                    }"
                  ></div>
                </div>
              </template>
            </el-table-column>
          </el-table>
        </div>

        <!-- 交易类型统计表 -->
        <div class="table-section">
          <div class="table-header">
            <h4>交易类型统计</h4>
            <el-tag size="small" type="info">Top {{ Math.min(transactionTypeStats.length, 10) }} 类型</el-tag>
          </div>
          <el-table :data="transactionTypeStats.slice(0, 10)" stripe border class="stats-table">
            <el-table-column prop="type" label="交易类型" min-width="150" show-overflow-tooltip />
            <el-table-column prop="count" label="交易次数" align="right" width="100" />
            <el-table-column prop="amount" label="总金额 (元)" align="right" width="150">
              <template #default="scope">
                <span class="amount-text">{{ scope.row.amount.toFixed(2) }}</span>
              </template>
            </el-table-column>
            <el-table-column label="平均金额 (元)" align="right" width="150">
              <template #default="scope">
                <span class="amount-text">{{ (scope.row.amount / scope.row.count).toFixed(2) }}</span>
              </template>
            </el-table-column>
            <el-table-column label="占比" align="right" width="100">
              <template #default="scope">
                <span class="percentage-text">{{ ((scope.row.amount / totalAmount) * 100).toFixed(1) }}%</span>
              </template>
            </el-table-column>
            <el-table-column prop="amount" label="金额条形图" min-width="200">
              <template #default="scope">
                <div class="progress-bar">
                  <div 
                    class="progress-fill transaction-type-bar" 
                    :style="{ width: ((scope.row.amount / maxTransactionTypeAmount) * 100) + '%' }"
                  ></div>
                </div>
              </template>
            </el-table-column>
          </el-table>
        </div>

        <!-- 支付方式统计表 -->
        <div class="table-section">
          <div class="table-header">
            <h4>支付方式统计</h4>
            <el-tag size="small" type="info">Top {{ Math.min(paymentMethodStats.length, 10) }} 方式</el-tag>
          </div>
          <el-table :data="paymentMethodStats.slice(0, 10)" stripe border class="stats-table">
            <el-table-column prop="method" label="支付方式" min-width="150" show-overflow-tooltip />
            <el-table-column prop="count" label="使用次数" align="right" width="100" />
            <el-table-column prop="amount" label="总金额 (元)" align="right" width="150">
              <template #default="scope">
                <span class="amount-text">{{ scope.row.amount.toFixed(2) }}</span>
              </template>
            </el-table-column>
            <el-table-column label="平均金额 (元)" align="right" width="150">
              <template #default="scope">
                <span class="amount-text">{{ (scope.row.amount / scope.row.count).toFixed(2) }}</span>
              </template>
            </el-table-column>
            <el-table-column label="占比" align="right" width="100">
              <template #default="scope">
                <span class="percentage-text">{{ ((scope.row.amount / totalAmount) * 100).toFixed(1) }}%</span>
              </template>
            </el-table-column>
            <el-table-column prop="amount" label="金额条形图" min-width="200">
              <template #default="scope">
                <div class="progress-bar">
                  <div 
                    class="progress-fill payment-method-bar" 
                    :style="{ width: ((scope.row.amount / maxPaymentMethodAmount) * 100) + '%' }"
                  ></div>
                </div>
              </template>
            </el-table-column>
          </el-table>
        </div>

        <!-- 分类统计表 -->
        <div class="table-section">
          <div class="table-header">
            <h4>消费分类统计</h4>
            <el-tag size="small" type="info">Top {{ Math.min(categoryStats.length, 15) }} 分类</el-tag>
          </div>
          <el-table :data="categoryStats.slice(0, 15)" stripe border class="stats-table">
            <el-table-column prop="category" label="消费分类" min-width="200" show-overflow-tooltip />
            <el-table-column prop="count" label="交易次数" align="right" width="100" />
            <el-table-column prop="amount" label="总金额 (元)" align="right" width="150">
              <template #default="scope">
                <span class="amount-text">{{ scope.row.amount.toFixed(2) }}</span>
              </template>
            </el-table-column>
            <el-table-column label="平均金额 (元)" align="right" width="150">
              <template #default="scope">
                <span class="amount-text">{{ (scope.row.amount / scope.row.count).toFixed(2) }}</span>
              </template>
            </el-table-column>
            <el-table-column label="占比" align="right" width="100">
              <template #default="scope">
                <span class="percentage-text">{{ ((scope.row.amount / totalAmount) * 100).toFixed(1) }}%</span>
              </template>
            </el-table-column>
            <el-table-column prop="amount" label="金额条形图" min-width="200">
              <template #default="scope">
                <div class="progress-bar">
                  <div 
                    class="progress-fill category-bar" 
                    :style="{ width: ((scope.row.amount / maxCategoryAmount) * 100) + '%' }"
                  ></div>
                </div>
              </template>
            </el-table-column>
          </el-table>
        </div>

        <!-- 每日交易统计表 -->
        <div class="table-section">
          <div class="table-header">
            <h4>每日交易统计</h4>
            <el-tag size="small" type="info">{{ dailyStats.length }} 天数据</el-tag>
          </div>
          <el-table :data="dailyStats" stripe border class="stats-table" max-height="500">
            <el-table-column prop="date" label="日期" width="120" fixed="left" />
            <el-table-column prop="count" label="交易数" align="right" width="100" />
            <el-table-column prop="income" label="收入 (元)" align="right" width="120">
              <template #default="scope">
                <span class="income-text">{{ scope.row.income.toFixed(2) }}</span>
              </template>
            </el-table-column>
            <el-table-column prop="expense" label="支出 (元)" align="right" width="120">
              <template #default="scope">
                <span class="expense-text">{{ scope.row.expense.toFixed(2) }}</span>
              </template>
            </el-table-column>
            <el-table-column prop="net" label="净收入 (元)" align="right" width="120">
              <template #default="scope">
                <span :class="scope.row.net >= 0 ? 'income-text' : 'expense-text'">
                  {{ scope.row.net.toFixed(2) }}
                </span>
              </template>
            </el-table-column>
            <el-table-column label="收支趋势" min-width="200">
              <template #default="scope">
                <div class="daily-trend-bar">
                  <div class="trend-income" :style="{ width: getTrendWidth(scope.row.income, 'income') }"></div>
                  <div class="trend-expense" :style="{ width: getTrendWidth(scope.row.expense, 'expense') }"></div>
                </div>
              </template>
            </el-table-column>
          </el-table>
        </div>
      </div>
    </el-card>
  </div>
</template>

<script setup>
import { ref, computed, watch } from 'vue'
import {
  Chart as ChartJS,
  CategoryScale,
  LinearScale,
  PointElement,
  LineElement,
  BarElement,
  Title,
  Tooltip,
  Legend,
  ArcElement
} from 'chart.js'
import { Line, Bar, Pie, Doughnut } from 'vue-chartjs'

// 注册Chart.js组件
ChartJS.register(
  CategoryScale,
  LinearScale,
  PointElement,
  LineElement,
  BarElement,
  Title,
  Tooltip,
  Legend,
  ArcElement
)

// Props
const props = defineProps({
  data: {
    type: Array,
    default: () => []
  },
  parseResults: {
    type: Array,
    default: () => []
  }
})

// 响应式数据
const viewMode = ref('charts')

// 计算统计数据
const processedData = computed(() => {
  return props.data.map(item => {
    const amount = parseFloat(item['金额']?.toString().replace(/[^\d.-]/g, '') || 0)
    const date = new Date(item['交易时间'] || Date.now())
    
    return {
      ...item,
      amount: Math.abs(amount),
      date: date,
      dateStr: date.toISOString().split('T')[0],
      category: item['交易内容'] || '其他',
      transactionType: item['交易类型'] || '其他',
      paymentMethod: item['支付方式'] || '其他',
      platform: item['来源文件']?.includes('支付宝') ? '支付宝' : 
                item['来源文件']?.includes('微信') ? '微信' : '其他',
      type: item['收支类型'] || '其他'
    }
  }).filter(item => !isNaN(item.amount))
})

// 收支分布数据
const incomeExpenseData = computed(() => {
  const income = processedData.value.filter(item => item.type === '收入')
  const expense = processedData.value.filter(item => item.type === '支出')
  const neutral = processedData.value.filter(item => item.type === '不计收支')
  
  const incomeAmount = income.reduce((sum, item) => sum + item.amount, 0)
  const expenseAmount = expense.reduce((sum, item) => sum + item.amount, 0)
  const neutralAmount = neutral.reduce((sum, item) => sum + item.amount, 0)
  
  const data = []
  const labels = []
  const backgroundColor = []
  
  if (incomeAmount > 0) {
    labels.push('收入')
    data.push(incomeAmount)
    backgroundColor.push('#4CAF50')
  }
  if (expenseAmount > 0) {
    labels.push('支出')
    data.push(expenseAmount)
    backgroundColor.push('#F44336')
  }
  if (neutralAmount > 0) {
    labels.push('不计收支')
    data.push(neutralAmount)
    backgroundColor.push('#9C27B0')
  }
  
  return {
    labels,
    datasets: [{
      data,
      backgroundColor,
      borderWidth: 2,
      borderColor: '#fff'
    }]
  }
})

// 平台对比数据
const platformComparisonData = computed(() => {
  const platforms = {}
  
  processedData.value.forEach(item => {
    if (!platforms[item.platform]) {
      platforms[item.platform] = { income: 0, expense: 0, neutral: 0 }
    }
    
    if (item.type === '收入') {
      platforms[item.platform].income += item.amount
    } else if (item.type === '支出') {
      platforms[item.platform].expense += item.amount
    } else {
      platforms[item.platform].neutral += item.amount
    }
  })
  
  const labels = Object.keys(platforms)
  const incomeData = labels.map(platform => platforms[platform].income)
  const expenseData = labels.map(platform => platforms[platform].expense)
  const neutralData = labels.map(platform => platforms[platform].neutral)
  
  const datasets = []
  if (incomeData.some(v => v > 0)) {
    datasets.push({
      label: '收入',
      data: incomeData,
      backgroundColor: '#4CAF50',
      borderColor: '#4CAF50',
      borderWidth: 1
    })
  }
  if (expenseData.some(v => v > 0)) {
    datasets.push({
      label: '支出',
      data: expenseData,
      backgroundColor: '#F44336',
      borderColor: '#F44336',
      borderWidth: 1
    })
  }
  if (neutralData.some(v => v > 0)) {
    datasets.push({
      label: '不计收支',
      data: neutralData,
      backgroundColor: '#9C27B0',
      borderColor: '#9C27B0',
      borderWidth: 1
    })
  }
  
  return { labels, datasets }
})

// 时间趋势数据
const timeTrendData = computed(() => {
  const dailyData = {}
  
  processedData.value.forEach(item => {
    const date = item.dateStr
    if (!dailyData[date]) {
      dailyData[date] = { income: 0, expense: 0 }
    }
    
    if (item.type === '收入') {
      dailyData[date].income += item.amount
    } else if (item.type === '支出') {
      dailyData[date].expense += item.amount
    }
  })
  
  const sortedDates = Object.keys(dailyData).sort()
  const incomeData = sortedDates.map(date => dailyData[date].income)
  const expenseData = sortedDates.map(date => dailyData[date].expense)
  
  return {
    labels: sortedDates,
    datasets: [
      {
        label: '收入',
        data: incomeData,
        borderColor: '#4CAF50',
        backgroundColor: 'rgba(76, 175, 80, 0.1)',
        tension: 0.4,
        fill: true
      },
      {
        label: '支出',
        data: expenseData,
        borderColor: '#F44336',
        backgroundColor: 'rgba(244, 67, 54, 0.1)',
        tension: 0.4,
        fill: true
      }
    ]
  }
})

// 分类统计数据
const categoryData = computed(() => {
  const categories = {}
  
  processedData.value
    .filter(item => item.type === '支出')
    .forEach(item => {
      const category = item.category.substring(0, 20) // 限制长度
      categories[category] = (categories[category] || 0) + item.amount
    })
  
  const sortedCategories = Object.entries(categories)
    .sort(([,a], [,b]) => b - a)
    .slice(0, 8)
  
  const labels = sortedCategories.map(([category]) => category)
  const data = sortedCategories.map(([, amount]) => amount)
  
  // 生成颜色
  const colors = [
    '#FF6384', '#36A2EB', '#FFCE56', '#4BC0C0', '#9966FF',
    '#FF9F40', '#FF6384', '#C9CBCF', '#4BC0C0', '#FF6384'
  ]
  
  return {
    labels,
    datasets: [{
      data,
      backgroundColor: colors.slice(0, data.length),
      borderWidth: 2,
      borderColor: '#fff'
    }]
  }
})

// 交易类型分布数据
const transactionTypeData = computed(() => {
  const transactionTypes = {}
  
  processedData.value.forEach(item => {
    if (item.transactionType && item.transactionType !== '其他') {
      const type = item.transactionType.substring(0, 15) // 限制长度
      transactionTypes[type] = (transactionTypes[type] || 0) + item.amount
    }
  })
  
  const sortedTypes = Object.entries(transactionTypes)
    .sort(([,a], [,b]) => b - a)
    .slice(0, 8) // 显示前8种类型
  
  const labels = sortedTypes.map(([type]) => type)
  const data = sortedTypes.map(([, amount]) => amount)
  
  // 生成颜色
  const colors = [
    '#FF6384', '#36A2EB', '#FFCE56', '#4BC0C0', '#9966FF',
    '#FF9F40', '#FF6384', '#C9CBCF'
  ]
  
  return {
    labels,
    datasets: [{
      data,
      backgroundColor: colors.slice(0, data.length),
      borderWidth: 2,
      borderColor: '#fff'
    }]
  }
})

// 支付方式分布数据
const paymentMethodData = computed(() => {
  const paymentMethods = {}
  
  processedData.value.forEach(item => {
    if (item.paymentMethod && item.paymentMethod !== '其他') {
      // 简化支付方式名称，去掉括号内容
      let method = item.paymentMethod.replace(/[()（）]/g, '').trim()
      // 限制长度
      method = method.substring(0, 12)
      paymentMethods[method] = (paymentMethods[method] || 0) + item.amount
    }
  })
  
  const sortedMethods = Object.entries(paymentMethods)
    .sort(([,a], [,b]) => b - a)
    .slice(0, 8) // 显示前8种支付方式
  
  const labels = sortedMethods.map(([method]) => method)
  const data = sortedMethods.map(([, amount]) => amount)
  
  // 生成颜色
  const colors = [
    '#8B5CF6', '#06B6D4', '#10B981', '#F59E0B', '#EF4444',
    '#6366F1', '#EC4899', '#84CC16'
  ]
  
  return {
    labels,
    datasets: [{
      data,
      backgroundColor: colors.slice(0, data.length),
      borderWidth: 2,
      borderColor: '#fff'
    }]
  }
})

// 金额分布数据
const amountDistributionData = computed(() => {
  const ranges = [
    { label: '0-10', min: 0, max: 10 },
    { label: '10-50', min: 10, max: 50 },
    { label: '50-100', min: 50, max: 100 },
    { label: '100-500', min: 100, max: 500 },
    { label: '500-1000', min: 500, max: 1000 },
    { label: '1000+', min: 1000, max: Infinity }
  ]
  
  const distribution = ranges.map(range => {
    const count = processedData.value.filter(item => 
      item.amount >= range.min && item.amount < range.max
    ).length
    return count
  })
  
  return {
    labels: ranges.map(r => r.label),
    datasets: [{
      label: '交易数量',
      data: distribution,
      backgroundColor: '#36A2EB',
      borderColor: '#36A2EB',
      borderWidth: 1
    }]
  }
})

// 平台统计表数据
const platformStats = computed(() => {
  const platforms = {}
  
  processedData.value.forEach(item => {
    if (!platforms[item.platform]) {
      platforms[item.platform] = { count: 0, totalAmount: 0 }
    }
    platforms[item.platform].count++
    platforms[item.platform].totalAmount += item.amount
  })
  
  return Object.entries(platforms).map(([platform, stats]) => ({
    platform,
    count: stats.count,
    totalAmount: stats.totalAmount,
    avgAmount: stats.totalAmount / stats.count
  })).sort((a, b) => b.totalAmount - a.totalAmount)
})

// 交易类型统计表数据
const transactionTypeStats = computed(() => {
  const transactionTypes = {}
  
  processedData.value.forEach(item => {
    if (item.transactionType && item.transactionType !== '其他') {
      const type = item.transactionType
      if (!transactionTypes[type]) {
        transactionTypes[type] = { count: 0, amount: 0 }
      }
      transactionTypes[type].count++
      transactionTypes[type].amount += item.amount
    }
  })
  
  return Object.entries(transactionTypes).map(([type, stats]) => ({
    type,
    count: stats.count,
    amount: stats.amount
  })).sort((a, b) => b.amount - a.amount)
})

// 支付方式统计表数据
const paymentMethodStats = computed(() => {
  const paymentMethods = {}
  
  processedData.value.forEach(item => {
    if (item.paymentMethod && item.paymentMethod !== '其他') {
      let method = item.paymentMethod.replace(/[()（）]/g, '').trim()
      method = method.substring(0, 12) // 限制长度
      if (!paymentMethods[method]) {
        paymentMethods[method] = { count: 0, amount: 0 }
      }
      paymentMethods[method].count++
      paymentMethods[method].amount += item.amount
    }
  })
  
  return Object.entries(paymentMethods).map(([method, stats]) => ({
    method,
    count: stats.count,
    amount: stats.amount
  })).sort((a, b) => b.amount - a.amount)
})

// 分类统计表数据
const categoryStats = computed(() => {
  const categories = {}
  
  processedData.value.forEach(item => {
    const category = item.category
    if (!categories[category]) {
      categories[category] = { count: 0, amount: 0 }
    }
    categories[category].count++
    categories[category].amount += item.amount
  })
  
  return Object.entries(categories).map(([category, stats]) => ({
    category,
    count: stats.count,
    amount: stats.amount
  })).sort((a, b) => b.amount - a.amount)
})

// 每日统计表数据
const dailyStats = computed(() => {
  const dailyData = {}
  
  processedData.value.forEach(item => {
    const date = item.dateStr
    if (!dailyData[date]) {
      dailyData[date] = { count: 0, income: 0, expense: 0 }
    }
    
    dailyData[date].count++
    if (item.type === '收入') {
      dailyData[date].income += item.amount
    } else if (item.type === '支出') {
      dailyData[date].expense += item.amount
    }
  })
  
  return Object.entries(dailyData).map(([date, stats]) => ({
    date,
    count: stats.count,
    income: stats.income,
    expense: stats.expense,
    net: stats.income - stats.expense
  })).sort((a, b) => b.date.localeCompare(a.date))
})

// 计算总金额
const totalAmount = computed(() => {
  return processedData.value.reduce((sum, item) => sum + item.amount, 0)
})

// 计算各类统计的最大值，用于条形图
const maxPlatformAmount = computed(() => {
  return Math.max(...platformStats.value.map(item => item.totalAmount), 1)
})

const maxTransactionTypeAmount = computed(() => {
  return Math.max(...transactionTypeStats.value.map(item => item.amount), 1)
})

const maxPaymentMethodAmount = computed(() => {
  return Math.max(...paymentMethodStats.value.map(item => item.amount), 1)
})

const maxCategoryAmount = computed(() => {
  return Math.max(...categoryStats.value.map(item => item.amount), 1)
})

const maxDailyAmount = computed(() => {
  const maxIncome = Math.max(...dailyStats.value.map(item => item.income), 0)
  const maxExpense = Math.max(...dailyStats.value.map(item => item.expense), 0)
  return Math.max(maxIncome, maxExpense, 1)
})

// 辅助函数
const getPlatformColor = (platform) => {
  switch (platform) {
    case '支付宝': return '#00A0E9'
    case '微信': return '#07C160'
    default: return '#909399'
  }
}

const getTrendWidth = (value, type) => {
  const percentage = (value / maxDailyAmount.value) * 100
  return Math.max(percentage, 2) + '%'
}

// 图表选项
const pieOptions = {
  responsive: true,
  maintainAspectRatio: false,
  plugins: {
    legend: {
      position: 'bottom'
    },
    tooltip: {
      callbacks: {
        label: function(context) {
          const total = context.dataset.data.reduce((a, b) => a + b, 0)
          const percentage = ((context.parsed / total) * 100).toFixed(1)
          return `${context.label}: ¥${context.parsed.toFixed(2)} (${percentage}%)`
        }
      }
    }
  }
}

const barOptions = {
  responsive: true,
  maintainAspectRatio: false,
  plugins: {
    legend: {
      position: 'top'
    },
    tooltip: {
      callbacks: {
        label: function(context) {
          return `${context.dataset.label}: ¥${context.parsed.y.toFixed(2)}`
        }
      }
    }
  },
  scales: {
    y: {
      beginAtZero: true,
      ticks: {
        callback: function(value) {
          return '¥' + value.toFixed(0)
        }
      }
    }
  }
}

const lineOptions = {
  responsive: true,
  maintainAspectRatio: false,
  plugins: {
    legend: {
      position: 'top'
    },
    tooltip: {
      callbacks: {
        label: function(context) {
          return `${context.dataset.label}: ¥${context.parsed.y.toFixed(2)}`
        }
      }
    }
  },
  scales: {
    y: {
      beginAtZero: true,
      ticks: {
        callback: function(value) {
          return '¥' + value.toFixed(0)
        }
      }
    }
  }
}

const doughnutOptions = {
  responsive: true,
  maintainAspectRatio: false,
  plugins: {
    legend: {
      position: 'right'
    },
    tooltip: {
      callbacks: {
        label: function(context) {
          return `${context.label}: ¥${context.parsed.toFixed(2)}`
        }
      }
    }
  }
}

const amountBarOptions = {
  responsive: true,
  maintainAspectRatio: false,
  plugins: {
    legend: {
      display: false
    }
  },
  scales: {
    y: {
      beginAtZero: true,
      title: {
        display: true,
        text: '交易数量'
      }
    },
    x: {
      title: {
        display: true,
        text: '金额区间 (元)'
      }
    }
  }
}

// 辅助函数
const getPlatformTagType = (platform) => {
  switch (platform) {
    case '支付宝': return 'success'
    case '微信': return 'primary'
    default: return 'info'
  }
}
</script>

<style scoped>
.statistics-view {
  padding: 20px;
}

.overview-card {
  margin-bottom: 20px;
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.charts-container {
  min-height: 400px;
}

.chart-wrapper {
  background: #fafafa;
  border-radius: 8px;
  padding: 20px;
  height: 400px;
}

.chart-wrapper h4 {
  margin: 0 0 20px 0;
  text-align: center;
  color: #333;
  font-weight: 600;
}

.tables-container h4 {
  margin: 0 0 15px 0;
  color: #333;
  font-weight: 600;
}

.amount-text {
  font-weight: 600;
  font-family: 'Monaco', 'Menlo', monospace;
  color: #333;
}

.income-text {
  font-weight: 600;
  font-family: 'Monaco', 'Menlo', monospace;
  color: #4CAF50;
}

.expense-text {
  font-weight: 600;
  font-family: 'Monaco', 'Menlo', monospace;
  color: #F44336;
}

.percentage-text {
  font-weight: 600;
  font-family: 'Monaco', 'Menlo', monospace;
  color: #606266;
}

/* 表格区域样式 */
.table-section {
  margin-bottom: 30px;
  background: #fff;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  overflow: hidden;
}

.table-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 20px 24px 16px;
  background: linear-gradient(135deg, #f8f9fa 0%, #e9ecef 100%);
  border-bottom: 1px solid #dee2e6;
}

.table-header h4 {
  margin: 0;
  color: #333;
  font-weight: 600;
  font-size: 16px;
}

.stats-table {
  margin: 0;
}

/* 条形图样式 */
.progress-bar {
  height: 20px;
  background-color: #f0f0f0;
  border-radius: 10px;
  overflow: hidden;
  position: relative;
}

.progress-fill {
  height: 100%;
  border-radius: 10px;
  transition: width 0.3s ease;
  position: relative;
}

.progress-fill.transaction-type-bar {
  background: linear-gradient(90deg, #667eea 0%, #764ba2 100%);
}

.progress-fill.payment-method-bar {
  background: linear-gradient(90deg, #8B5CF6 0%, #06B6D4 100%);
}

.progress-fill.category-bar {
  background: linear-gradient(90deg, #FF6B6B 0%, #4ECDC4 100%);
}

/* 每日趋势条形图 */
.daily-trend-bar {
  display: flex;
  height: 20px;
  background-color: #f0f0f0;
  border-radius: 10px;
  overflow: hidden;
}

.trend-income {
  background: linear-gradient(90deg, #4CAF50 0%, #8BC34A 100%);
  transition: width 0.3s ease;
  min-width: 2px;
}

.trend-expense {
  background: linear-gradient(90deg, #F44336 0%, #FF9800 100%);
  transition: width 0.3s ease;
  min-width: 2px;
}

:deep(.el-table) {
  font-size: 14px;
}

:deep(.el-table th) {
  background-color: #f8f9fa;
  font-weight: 600;
  color: #495057;
}

:deep(.el-table td) {
  padding: 12px 0;
}

:deep(.el-table--striped .el-table__body tr.el-table__row--striped td) {
  background-color: #fafbfc;
}

:deep(.el-table__header-wrapper) {
  border-radius: 0;
}

:deep(.el-table--border) {
  border: none;
}

:deep(.el-table--border td, .el-table--border th) {
  border-right: 1px solid #ebeef5;
}

:deep(.el-table--border th:first-child, .el-table--border td:first-child) {
  border-left: none;
}

:deep(.el-table--border th:last-child, .el-table--border td:last-child) {
  border-right: none;
}
</style>
