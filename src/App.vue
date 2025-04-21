<template>
  <div class="container">
    <h2>订单列表</h2>
    <!-- 新增隐藏列按钮 -->
    <button @click="toggleColumnsVisibility" class="query-btn">
      {{ isColumnsVisible ? '隐藏列' : '显示列' }}
    </button>
    <!-- 新增查询条件12 -->
    <div class="search-condition">
      <label for="status">订单状态：</label>
      <select v-model="selectedStatus">
        <option value="-1">全部</option>
        <option value="1">已定稿</option>
        <option value="0">未定稿</option>
      </select>
      <button @click="handleSearch" class="query-btn">查询</button>
    </div>
    <div class="table-container">
      <table>
        <thead>
        <tr>
          <th v-show="isColumnsVisible">
            <input type="checkbox" v-model="isAllChecked" @change="handleSelectAll" />
          </th>
          <th v-show="isColumnsVisible">ID</th>
          <th>订单编号</th>
          <th v-show="isColumnsVisible">样式</th>
          <th v-show="isColumnsVisible">模板名称</th>
          <th v-show="isColumnsVisible">订单状态</th> <!-- 新增订单状态列 -->
          <th v-show="isColumnsVisible">操作</th>
          <th v-show="isColumnsVisible">导出次数</th>
        </tr>
        </thead>
        <tbody>
        <tr v-for="item in tableData" :key="item.id">
          <td v-show="isColumnsVisible">
            <input type="checkbox" v-model="item.checked" />
          </td>
          <td v-show="isColumnsVisible">{{ item.id }}</td>
          <td>{{ item.orderSn || '暂无' }}</td>
          <td v-show="isColumnsVisible">{{ item.styleName }}</td>
          <td v-show="isColumnsVisible">{{ item.templateName }}</td>
          <td v-show="isColumnsVisible">{{ item.status ? '已定稿' : '未定稿' }}</td> <!-- 显示订单状态 -->
          <td v-show="isColumnsVisible">
            <button
                @click="handleExport(item)"
                class="export-btn
                {{ item.exportCount > 0 ? 'exported-btn' : '' }}
                {{ item.exporting ? 'exporting-btn' : '' }}"
                :disabled="item.exporting"
            >
              {{ item.exporting ? '导出中...' : '导出' }}
            </button>
          </td>
          <!-- 显示导出次数 -->
          <td v-show="isColumnsVisible">{{ item.exportCount || 0 }}</td>
        </tr>
        </tbody>
      </table>
    </div>
    <button @click="handleBatchExport" class="query-btn">一键导出</button>
    <div class="pagination">
      <button :disabled="currentPage === 1" @click="handlePageChange(currentPage - 1)">上一页</button>
      <span>第 {{ currentPage }} 页 / 共 {{ totalPages }} 页</span>
      <button :disabled="currentPage === totalPages" @click="handlePageChange(currentPage + 1)">下一页</button>
      <!-- 新增输入页码和跳转按钮 -->
      <input
          v-model="inputPage"
          type="number"
          min="1"
          :max="totalPages"
          placeholder="输入页码"
          @keydown.enter="handleGoToPage"
      />
      <button @click="handleGoToPage">跳转</button>
    </div>

    <!-- 新增批量查询区域 -->
    <div class="batch-query">
      <h3>批量查询订单</h3>
      <textarea
          v-model="orderSns"
          placeholder="请输入订单编号，每行一个
例如：
2458017265067963565
4221494571529979437
2457041988150466597"
          rows="5"
      ></textarea>
      <button @click="handleBatchQuery" class="query-btn">查询订单</button>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import axios from 'axios'

const tableData = ref([])
const currentPage = ref(1)
const pageSize = ref(50)
const totalPages = ref(0)
const total = ref(0)
const orderSns = ref('') // 新增订单编号输入框的值
const inputPage = ref('') // 新增输入页码的值
const selectedStatus = ref('-1') // 新增所选订单状态
const isAllChecked = ref(false) // 全选状态
// 新增响应式变量控制列显示状态
const isColumnsVisible = ref(true)

const fetchData = async (page = 1, orderSnList = null, status = '') => {
  try {
    const params = {
      page,
      size: pageSize.value
    }

    // 如果有订单编号列表，作为一个数组添加到请求参数中
    if (orderSnList) {
      params.orderSns = orderSnList // 直接传递数组
    }

    params.status = status

    // 使用 axios 的 paramsSerializer 来自定义参数序列化
    const response = await axios.get('https://ai.cbbeny.cn/api/api/v1/custom/customs', {
      params,
      paramsSerializer: {
        indexes: null // 这将确保数组被序列化为 orderSns=value1,value2,value3 的形式
      }
    })

    if (response.data.code === '0000') {
      const { data } = response.data
      tableData.value = data.data.map(item => ({ ...item, checked: false, exportCount: item.exportCount || 0}))
      totalPages.value = data.pages
      total.value = data.total
      currentPage.value = data.page
      isAllChecked.value = false
    }
  } catch (error) {
    console.error('获取数据失败:', error)
  }
}

// 新增切换列显示状态的方法
const toggleColumnsVisibility = () => {
  isColumnsVisible.value = !isColumnsVisible.value
}

const handlePageChange = (page) => {
  // 如果有订单号查询条件，翻页时保持查询条件
  const orderSnList = orderSns.value ? orderSns.value.split('\n').filter(sn => sn.trim()) : null
  fetchData(page, orderSnList, selectedStatus.value)
}

const handleBatchQuery = () => {
  // 将文本框中的内容按行分割，并过滤掉空行
  const orderSnList = orderSns.value.split('\n').filter(sn => sn.trim())
  if (orderSnList.length === 0) {
    alert('请输入订单编号')
    return
  }
  // 重置到第一页并带着查询条件请求数据
  fetchData(1, orderSnList, selectedStatus.value)
}

const handleExport = async (item) => {
  // 添加导出状态标记
  item.exporting = true

  try {
    const response = await axios.post(
        'https://ai.cbbeny.cn/api/office/generate-excel',
        item,
        {
          responseType: 'blob' // 设置响应类型为blob，用于处理文件下载
        }
    )
    item.exportCount += 1
    // 创建下载链接
    const blob = new Blob([response.data], {
      type: response.headers['content-type']
    })
    const url = window.URL.createObjectURL(blob)
    const link = document.createElement('a')
    link.href = url
    // 从响应头获取文件名，如果没有则使用默认名称
    console.log('文件名:', response.headers['content-disposition']) // 打印响应头看看具体内容
    const filename = response.headers['content-disposition']?.split('filename=')[1] || 'export.xlsx'
    link.setAttribute('download', filename)
    document.body.appendChild(link)
    link.click()
    document.body.removeChild(link)
    window.URL.revokeObjectURL(url)
  } catch (error) {
    console.error('导出失败:', error)
    alert('导出失败，请重试')
  } finally {
    item.exporting = false
  }
}

// 新增跳转页码处理函数
const handleGoToPage = () => {
  const page = parseInt(inputPage.value)
  if (isNaN(page) || page < 1 || page > totalPages.value) {
    alert('请输入有效的页码')
    return
  }
  const orderSnList = orderSns.value ? orderSns.value.split('\n').filter(sn => sn.trim()) : null
  fetchData(page, orderSnList, selectedStatus.value)
  currentPage.value = page
  inputPage.value = ''
}

// 新增查询处理函数
const handleSearch = () => {
  const orderSnList = orderSns.value ? orderSns.value.split('\n').filter(sn => sn.trim()) : null
  fetchData(1, orderSnList, selectedStatus.value)
}

// 处理全选
const handleSelectAll = () => {
  tableData.value.forEach(item => {
    item.checked = isAllChecked.value
  })
}

// 一键导出
const handleBatchExport = async () => {
  const selectedItems = tableData.value.filter(item => item.checked)
  if (selectedItems.length === 0) {
    alert('请选择要导出的订单')
    return
  }

  selectedItems.forEach(item => {
    item.exporting = true
  })

  try {
    for (const item of selectedItems) {
      const response = await axios.post(
          'https://ai.cbbeny.cn/api/office/generate-excel',
          item,
          {
            responseType: 'blob' // 设置响应类型为blob，用于处理文件下载
          }
      )

      // 创建下载链接
      const blob = new Blob([response.data], {
        type: response.headers['content-type']
      })
      const url = window.URL.createObjectURL(blob)
      const link = document.createElement('a')
      link.href = url
      // 从响应头获取文件名，如果没有则使用默认名称
      const filename = response.headers['content-disposition']?.split('filename=')[1] || 'export.xlsx'
      link.setAttribute('download', filename)
      document.body.appendChild(link)
      link.click()
      document.body.removeChild(link)
      window.URL.revokeObjectURL(url)
    }
  } catch (error) {
    console.error('导出失败:', error)
    alert('导出失败，请重试')
  } finally {
    selectedItems.forEach(item => {
      item.exporting = false
    })
  }
}

onMounted(() => {
  fetchData(1)
})
</script>

<style scoped>
.container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
}

.container button:first-child {
  margin-bottom: 15px;
}

.table-container {
  margin: 20px 0;
  overflow-x: auto;
}

table {
  width: 100%;
  border-collapse: collapse;
  background-color: white;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.2);
}

th,
td {
  padding: 12px;
  text-align: left;
  border-bottom: 1px solid #ddd;
}

th {
  background-color: #f5f5f5;
  font-weight: bold;
}

tr:hover {
  background-color: #f5f5f5;
}

.pagination {
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 20px;
  margin-top: 20px;
}

button {
  padding: 8px 16px;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  transition: background-color 0.3s;
}

button:disabled {
  background-color: #cccccc;
  cursor: not-allowed;
}

.pagination button {
  background-color: #4CAF50;
}

.pagination button:hover:not(:disabled) {
  background-color: #45a049;
}

.batch-query {
  margin-top: 30px;
  padding: 20px;
  background-color: #f9f9f9;
  border-radius: 8px;
}

.batch-query h3 {
  margin-bottom: 15px;
  color: #333;
}

textarea {
  width: 100%;
  padding: 12px;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-family: monospace;
  resize: vertical;
  margin-bottom: 15px;
}

.query-btn {
  background-color: #2196F3;
  margin-top: 10px;
}

.query-btn:hover:not(:disabled) {
  background-color: #1976D2;
}

.export-btn {
  background-color: #2196F3;
}

.exported-btn {
  background-color: #6ef55f;
}

td {
  vertical-align: middle;
}

.search-condition {
  margin-bottom: 20px;
  display: flex;
  align-items: center;
  gap: 10px;
}
</style>