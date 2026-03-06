<script setup>
import { computed, onBeforeUnmount, onMounted, ref } from 'vue'

const keysEndpoint = import.meta.env.VITE_KEYS_ENDPOINT || '/api/v1/keys'
const usageStatsEndpoint =
  import.meta.env.VITE_USAGE_ENDPOINT || '/api/v1/usage/dashboard/api-keys-usage'
const usageDetailsEndpoint = import.meta.env.VITE_USAGE_DETAILS_ENDPOINT || '/api/v1/usage'

const token =
  'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX2lkIjoxODAsImVtYWlsIjoiMTQ1Njc5OTM3OUBxcS5jb20iLCJyb2xlIjoidXNlciIsInRva2VuX3ZlcnNpb24iOjAsImV4cCI6MTc3Mjg2ODY3OSwibmJmIjoxNzcyNzgyMjc5LCJpYXQiOjE3NzI3ODIyNzl9.80IQGkAKFIGo0vfGqYJ5dz5oHjFTGbqhUwD6aFdMb0o'

const timezone = 'Asia/Shanghai'
const fixedStartDate = '2026-03-01'
const fixedDetailsPage = '1'
const fixedDetailsPageSize = '100'

const loading = ref(false)
const usageLoading = ref(false)
const detailsLoading = ref(false)
const errorMessage = ref('')
const detailsErrorMessage = ref('')
const items = ref([])
const selectedKeyId = ref('')
const usageStatsMap = ref({})
const usageItems = ref([])
const usageTotal = ref(0)
const usageEndDate = ref('')
const activeCostRow = ref(null)
const costTooltipPosition = ref({ left: 0, top: 0 })
const activeTokenRow = ref(null)
const tokenTooltipPosition = ref({ left: 0, top: 0 })
let hideTooltipTimer = null
let hideTokenTooltipTimer = null

const numberFormatter = new Intl.NumberFormat('zh-CN', {
  minimumFractionDigits: 0,
  maximumFractionDigits: 6,
})

const getShanghaiDate = (date = new Date()) =>
  new Intl.DateTimeFormat('sv-SE', {
    timeZone: timezone,
    year: 'numeric',
    month: '2-digit',
    day: '2-digit',
  }).format(date)

const formatAmount = (value) => numberFormatter.format(Number(value || 0))
const formatUsd = (value) => `$${Number(value || 0).toFixed(6)}`

const formatSeconds = (ms) => `${(Number(ms || 0) / 1000).toFixed(2)}s`

const formatDateTime = (value) => {
  if (!value) return '-'
  const date = new Date(value)
  if (Number.isNaN(date.getTime())) return '-'
  return date.toLocaleString('zh-CN', { timeZone: timezone, hour12: false })
}

const formatRequestType = (type) => {
  if (!type) return '-'
  return type === 'stream' ? '流式' : type
}

const formatTokenCount = (value) => {
  const num = Number(value || 0)
  if (num >= 1000000) return `${(num / 1000000).toFixed(1)}M`
  if (num >= 1000) return `${(num / 1000).toFixed(1)}K`
  return String(num)
}

const nameOptions = computed(() =>
  items.value.map((item) => ({
    id: String(item.id),
    name: item.name || `未命名-${item.id}`,
  })),
)

const selectedKey = computed(() =>
  items.value.find((item) => String(item.id) === selectedKeyId.value),
)

const selectedUsage = computed(() => usageStatsMap.value[selectedKeyId.value] || null)

const quotaDisplay = computed(() => {
  const quota = Number(selectedKey.value?.quota ?? 0)
  return quota === 0 ? '无限限制额度' : formatAmount(quota)
})

const summaryCards = computed(() => [
  {
    label: '今日消费',
    value: usageLoading.value ? '加载中...' : `$${formatAmount(selectedUsage.value?.today_actual_cost)}`,
  },
  {
    label: '总消费',
    value: usageLoading.value ? '加载中...' : `$${formatAmount(selectedUsage.value?.total_actual_cost)}`,
  },
  { label: '最大额度', value: quotaDisplay.value },
])

const costTooltipStyle = computed(() => ({
  left: `${costTooltipPosition.value.left}px`,
  top: `${costTooltipPosition.value.top}px`,
}))

const tokenTooltipStyle = computed(() => ({
  left: `${tokenTooltipPosition.value.left}px`,
  top: `${tokenTooltipPosition.value.top}px`,
}))

const clearHideTooltipTimer = () => {
  if (hideTooltipTimer) {
    clearTimeout(hideTooltipTimer)
    hideTooltipTimer = null
  }
}

const clearTokenHideTooltipTimer = () => {
  if (hideTokenTooltipTimer) {
    clearTimeout(hideTokenTooltipTimer)
    hideTokenTooltipTimer = null
  }
}

const updateCostTooltipPosition = (target) => {
  const rect = target.getBoundingClientRect()
  const tooltipWidth = 220
  const gap = 10
  let left = rect.right + gap

  if (left + tooltipWidth > window.innerWidth - 12) {
    left = rect.left - tooltipWidth - gap
  }

  const top = rect.top + rect.height / 2
  costTooltipPosition.value = {
    left: Math.max(12, left),
    top: Math.max(20, Math.min(window.innerHeight - 20, top)),
  }
}

const showCostTooltip = (event, row) => {
  clearHideTooltipTimer()
  activeTokenRow.value = null
  activeCostRow.value = row
  updateCostTooltipPosition(event.currentTarget)
}

const scheduleHideCostTooltip = () => {
  clearHideTooltipTimer()
  hideTooltipTimer = setTimeout(() => {
    activeCostRow.value = null
  }, 120)
}

const keepCostTooltip = () => {
  clearHideTooltipTimer()
}

const hideCostTooltip = () => {
  clearHideTooltipTimer()
  activeCostRow.value = null
}

const updateTokenTooltipPosition = (target) => {
  const rect = target.getBoundingClientRect()
  const tooltipWidth = 220
  const gap = 10
  let left = rect.right + gap

  if (left + tooltipWidth > window.innerWidth - 12) {
    left = rect.left - tooltipWidth - gap
  }

  const top = rect.top + rect.height / 2
  tokenTooltipPosition.value = {
    left: Math.max(12, left),
    top: Math.max(20, Math.min(window.innerHeight - 20, top)),
  }
}

const showTokenTooltip = (event, row) => {
  clearTokenHideTooltipTimer()
  activeCostRow.value = null
  activeTokenRow.value = row
  updateTokenTooltipPosition(event.currentTarget)
}

const scheduleHideTokenTooltip = () => {
  clearTokenHideTooltipTimer()
  hideTokenTooltipTimer = setTimeout(() => {
    activeTokenRow.value = null
  }, 120)
}

const keepTokenTooltip = () => {
  clearTokenHideTooltipTimer()
}

const hideTokenTooltip = () => {
  clearTokenHideTooltipTimer()
  activeTokenRow.value = null
}

const fetchUsageStats = async (apiKeyIds) => {
  if (!apiKeyIds.length) {
    usageStatsMap.value = {}
    return
  }

  usageLoading.value = true
  try {
    const response = await fetch(usageStatsEndpoint, {
      method: 'POST',
      headers: {
        Accept: 'application/json',
        'Content-Type': 'application/json',
        Authorization: `Bearer ${token}`,
      },
      body: JSON.stringify({ api_key_ids: apiKeyIds }),
    })

    if (!response.ok) throw new Error(`统计请求失败，状态码 ${response.status}`)

    const payload = await response.json()
    const stats = payload?.data?.stats
    if (payload?.code !== 0 || !stats || typeof stats !== 'object') {
      throw new Error(payload?.message || '统计接口返回格式不符合预期')
    }

    usageStatsMap.value = stats
  } catch (error) {
    errorMessage.value = error instanceof Error ? error.message : '统计请求失败'
  } finally {
    usageLoading.value = false
  }
}

const fetchUsageDetails = async (apiKeyId) => {
  if (!apiKeyId) {
    usageItems.value = []
    usageTotal.value = 0
    return
  }

  detailsLoading.value = true
  detailsErrorMessage.value = ''
  usageEndDate.value = getShanghaiDate()

  try {
    const query = new URLSearchParams({
      page: fixedDetailsPage,
      page_size: fixedDetailsPageSize,
      api_key_id: String(apiKeyId),
      start_date: fixedStartDate,
      end_date: usageEndDate.value,
      timezone,
    })

    const response = await fetch(`${usageDetailsEndpoint}?${query.toString()}`, {
      method: 'GET',
      headers: {
        Accept: 'application/json',
        Authorization: `Bearer ${token}`,
      },
    })

    if (!response.ok) throw new Error(`明细请求失败，状态码 ${response.status}`)

    const payload = await response.json()
    const fetchedItems = payload?.data?.items
    if (payload?.code !== 0 || !Array.isArray(fetchedItems)) {
      throw new Error(payload?.message || '明细接口返回格式不符合预期')
    }

    usageItems.value = fetchedItems
    usageTotal.value = Number(payload?.data?.total || 0)
  } catch (error) {
    detailsErrorMessage.value = error instanceof Error ? error.message : '明细请求失败'
  } finally {
    detailsLoading.value = false
  }
}

const fetchKeyOptions = async () => {
  loading.value = true
  errorMessage.value = ''

  try {
    const query = new URLSearchParams({
      page: '1',
      page_size: '10',
      timezone,
    })

    const response = await fetch(`${keysEndpoint}?${query.toString()}`, {
      method: 'GET',
      headers: {
        Accept: 'application/json',
        Authorization: `Bearer ${token}`,
      },
    })

    if (!response.ok) throw new Error(`请求失败，状态码 ${response.status}`)

    const payload = await response.json()
    const fetchedItems = payload?.data?.items
    if (payload?.code !== 0 || !Array.isArray(fetchedItems)) {
      throw new Error(payload?.message || '接口返回格式不符合预期')
    }

    items.value = fetchedItems
    const previousSelectedKeyId = selectedKeyId.value
    const hasPreviousKey = fetchedItems.some((item) => String(item.id) === previousSelectedKeyId)
    selectedKeyId.value = hasPreviousKey
      ? previousSelectedKeyId
      : fetchedItems[0]?.id
        ? String(fetchedItems[0].id)
        : ''

    const apiKeyIds = fetchedItems.map((item) => Number(item.id)).filter((id) => Number.isFinite(id))
    await fetchUsageStats(apiKeyIds)
    await fetchUsageDetails(selectedKeyId.value)
  } catch (error) {
    errorMessage.value = error instanceof Error ? error.message : '请求失败'
  } finally {
    loading.value = false
  }
}

const handleKeyChange = () => {
  fetchUsageDetails(selectedKeyId.value)
}

onMounted(() => {
  fetchKeyOptions()
})

onBeforeUnmount(() => {
  clearHideTooltipTimer()
  clearTokenHideTooltipTimer()
})
</script>

<template>
  <div class="usage-page">
    <header class="topbar">
      <div>
        <h1>调用明细</h1>
      </div>
      <div class="filters">
        <label for="key-select">API 密钥</label>
        <select
          id="key-select"
          v-model="selectedKeyId"
          :disabled="loading || !nameOptions.length"
          @change="handleKeyChange"
        >
          <option value="" disabled>{{ loading ? '加载中...' : '请选择 Key' }}</option>
          <option v-for="option in nameOptions" :key="option.id" :value="option.id">
            {{ option.name }}
          </option>
        </select>
        <button type="button" :disabled="loading" @click="fetchKeyOptions">刷新</button>
      </div>
    </header>

    <p v-if="errorMessage" class="error">{{ errorMessage }}</p>

    <section class="summary">
      <div v-for="card in summaryCards" :key="card.label" class="summary-item">
        <span>{{ card.label }}</span>
        <strong>{{ card.value }}</strong>
      </div>
    </section>

    <section class="table-card">
      <div class="table-title">
        <span>明细记录</span>
        <span>共 {{ usageTotal }} 条</span>
      </div>

      <div class="table-wrap">
        <table>
          <thead>
            <tr>
              <th>API 密钥</th>
              <th>模型</th>
              <th>推理强度</th>
              <th>类型</th>
              <th>TOKEN</th>
              <th>费用</th>
              <th>首 TOKEN</th>
              <th>耗时</th>
              <th>时间</th>
              <th>USER-AGENT</th>
            </tr>
          </thead>
          <tbody>
            <tr v-if="detailsErrorMessage">
              <td colspan="10" class="table-msg table-error">{{ detailsErrorMessage }}</td>
            </tr>
            <tr v-else-if="!detailsLoading && !usageItems.length">
              <td colspan="10" class="table-msg">暂无明细</td>
            </tr>
            <tr v-for="row in usageItems" :key="row.id">
              <td>{{ row.api_key?.name || selectedKey?.name || '-' }}</td>
              <td>{{ row.model || '-' }}</td>
              <td>{{ row.reasoning_effort || '-' }}</td>
              <td>
                <span class="type-tag">{{ formatRequestType(row.request_type) }}</span>
              </td>
              <td>
                <div
                  class="token-cell"
                  tabindex="0"
                  @mouseenter="showTokenTooltip($event, row)"
                  @mouseleave="scheduleHideTokenTooltip"
                  @focusin="showTokenTooltip($event, row)"
                  @focusout="scheduleHideTokenTooltip"
                >
                  <div>
                    <div class="token-line">
                      <span class="token-down">↓ {{ row.input_tokens || 0 }}</span>
                      <span class="token-up">↑ {{ row.output_tokens || 0 }}</span>
                    </div>
                    <div class="token-line token-sub">
                      <span class="token-read">⟲ {{ formatTokenCount(row.cache_read_tokens) }}</span>
                      <span class="token-write">✎ {{ formatTokenCount(row.cache_creation_tokens) }}</span>
                    </div>
                  </div>
                  <span class="token-info" aria-hidden="true">i</span>
                </div>
              </td>
              <td>
                <div
                  class="cost-cell"
                  tabindex="0"
                  @mouseenter="showCostTooltip($event, row)"
                  @mouseleave="scheduleHideCostTooltip"
                  @focusin="showCostTooltip($event, row)"
                  @focusout="scheduleHideCostTooltip"
                >
                  <span class="cost">{{ formatUsd(row.actual_cost) }}</span>
                  <span class="cost-info" aria-hidden="true">i</span>
                </div>
              </td>
              <td>{{ formatSeconds(row.first_token_ms) }}</td>
              <td>{{ formatSeconds(row.duration_ms) }}</td>
              <td>{{ formatDateTime(row.created_at) }}</td>
              <td>{{ row.user_agent || '-' }}</td>
            </tr>
          </tbody>
        </table>
      </div>

      <p v-if="detailsLoading" class="loading-tip">明细加载中...</p>
    </section>

    <Teleport to="body">
      <div
        v-if="activeCostRow"
        class="cost-tooltip-portal"
        :style="costTooltipStyle"
        @mouseenter="keepCostTooltip"
        @mouseleave="hideCostTooltip"
      >
        <div class="tooltip-title">成本明细</div>
        <div class="tooltip-row">
          <span>输入成本</span>
          <span>{{ formatUsd(activeCostRow.input_cost) }}</span>
        </div>
        <div class="tooltip-row">
          <span>输出成本</span>
          <span>{{ formatUsd(activeCostRow.output_cost) }}</span>
        </div>
        <div class="tooltip-row">
          <span>缓存创建成本</span>
          <span>{{ formatUsd(activeCostRow.cache_creation_cost) }}</span>
        </div>
        <div class="tooltip-row">
          <span>缓存读取成本</span>
          <span>{{ formatUsd(activeCostRow.cache_read_cost) }}</span>
        </div>
        <div class="tooltip-row">
          <span>倍率</span>
          <span>{{ Number(activeCostRow.rate_multiplier || 1).toFixed(2) }}x</span>
        </div>
        <div class="tooltip-row">
          <span>原始</span>
          <span>{{ formatUsd(activeCostRow.total_cost) }}</span>
        </div>
        <div class="tooltip-row total">
          <span>计费</span>
          <span>{{ formatUsd(activeCostRow.actual_cost) }}</span>
        </div>
      </div>
    </Teleport>

    <Teleport to="body">
      <div
        v-if="activeTokenRow"
        class="cost-tooltip-portal"
        :style="tokenTooltipStyle"
        @mouseenter="keepTokenTooltip"
        @mouseleave="hideTokenTooltip"
      >
        <div class="tooltip-title">Token 明细</div>
        <div class="tooltip-row">
          <span>输入 Token</span>
          <span>{{ Number(activeTokenRow.input_tokens || 0) }}</span>
        </div>
        <div class="tooltip-row">
          <span>输出 Token</span>
          <span>{{ Number(activeTokenRow.output_tokens || 0) }}</span>
        </div>
        <div class="tooltip-row">
          <span>缓存创建 Token</span>
          <span>{{ Number(activeTokenRow.cache_creation_tokens || 0) }}</span>
        </div>
        <div class="tooltip-row">
          <span>缓存读取 Token</span>
          <span>{{ Number(activeTokenRow.cache_read_tokens || 0) }}</span>
        </div>
        <div class="tooltip-row total token-total">
          <span>总 Token</span>
          <span>{{
            Number(activeTokenRow.input_tokens || 0) +
            Number(activeTokenRow.output_tokens || 0) +
            Number(activeTokenRow.cache_creation_tokens || 0) +
            Number(activeTokenRow.cache_read_tokens || 0)
          }}</span>
        </div>
      </div>
    </Teleport>
  </div>
</template>

<style scoped lang="less">
.usage-page {
  width: min(1600px, 100%);
  margin: 0 auto;
}

.topbar {
  display: flex;
  justify-content: space-between;
  gap: 20px;
  align-items: flex-end;
  margin-bottom: 14px;
}

h1 {
  margin: 0;
  font-size: 26px;
  color: #1f2937;
}

.filters {
  display: flex;
  align-items: center;
  gap: 10px;
}

label {
  color: #475569;
  font-size: 14px;
  white-space: nowrap;
}

select {
  height: 38px;
  min-width: 220px;
  border: 1px solid #d0d7e2;
  border-radius: 8px;
  padding: 0 10px;
  font-size: 14px;
  background: #fff;
}

button {
  height: 38px;
  border: 0;
  border-radius: 8px;
  padding: 0 16px;
  background: #2563eb;
  color: #fff;
  cursor: pointer;
}

button:disabled {
  cursor: not-allowed;
  opacity: 0.6;
}

.error {
  margin: 6px 0 0;
  color: #b91c1c;
  font-size: 14px;
}

.summary {
  display: grid;
  grid-template-columns: repeat(3, minmax(220px, 260px));
  justify-content: flex-start;
  gap: 14px;
  margin: 14px 0 16px;
}

.summary-item {
  background: linear-gradient(180deg, #ffffff 0%, #f8fbff 100%);
  border: 1px solid #c9d3e3;
  border-left: 4px solid #3b82f6;
  border-radius: 12px;
  padding: 12px 14px;
  box-shadow: 0 4px 12px rgba(15, 23, 42, 0.06);
}

.summary-item span {
  display: block;
  color: #4b5563;
  font-size: 12px;
  margin-bottom: 6px;
}

.summary-item strong {
  font-size: 24px;
  color: #111827;
  line-height: 1.15;
}

.table-card {
  background: #fff;
  border: 1px solid #d9e0ea;
  border-radius: 16px;
  overflow: hidden;
}

.table-title {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 12px 16px;
  border-bottom: 1px solid #e6ebf2;
  color: #475569;
  font-size: 14px;
}

.table-wrap {
  overflow: auto;
}

table {
  width: 100%;
  min-width: 1320px;
  border-collapse: collapse;
}

th {
  background: #f2f4f7;
  color: #1f2937;
  font-size: 14px;
  font-weight: 600;
  text-align: left;
  padding: 14px 18px;
  white-space: nowrap;
}

td {
  padding: 16px 18px;
  border-top: 1px solid #e9edf3;
  color: #374151;
  font-size: 14px;
  vertical-align: middle;
}

.type-tag {
  display: inline-flex;
  align-items: center;
  padding: 2px 8px;
  height: 22px;
  border-radius: 6px;
  background: #dbeafe;
  color: #1d4ed8;
  font-size: 12px;
  font-weight: 600;
}

.token-line {
  display: flex;
  align-items: center;
  gap: 10px;
  white-space: nowrap;
}

.token-cell {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  outline: none;
}

.token-sub {
  margin-top: 4px;
}

.token-down {
  color: #059669;
}

.token-up {
  color: #7c3aed;
}

.token-read {
  color: #0369a1;
}

.token-write {
  color: #ea580c;
}

.token-info {
  width: 14px;
  height: 14px;
  border-radius: 50%;
  border: 1px solid #cbd5e1;
  color: #64748b;
  font-size: 10px;
  line-height: 12px;
  text-align: center;
  display: inline-block;
}

.cost {
  color: #059669;
  font-weight: 600;
}

.cost-cell {
  position: relative;
  display: inline-flex;
  align-items: center;
  gap: 6px;
  outline: none;
}

.cost-info {
  width: 14px;
  height: 14px;
  border-radius: 50%;
  border: 1px solid #cbd5e1;
  color: #64748b;
  font-size: 10px;
  line-height: 12px;
  text-align: center;
  display: inline-block;
}

.cost-tooltip-portal {
  position: fixed;
  transform: translateY(-50%);
  min-width: 190px;
  background: #0f172a;
  border: 1px solid #1e293b;
  border-radius: 10px;
  padding: 10px;
  z-index: 20;
  box-shadow: 0 10px 22px rgba(2, 6, 23, 0.35);
  pointer-events: auto;
}

.tooltip-title {
  color: #e2e8f0;
  font-size: 13px;
  font-weight: 600;
  margin-bottom: 6px;
}

.tooltip-row {
  display: flex;
  justify-content: space-between;
  gap: 10px;
  color: #cbd5e1;
  font-size: 12px;
  line-height: 1.4;
}

.tooltip-row.total {
  margin-top: 6px;
  padding-top: 6px;
  border-top: 1px solid #334155;
  color: #86efac;
  font-weight: 600;
}

.tooltip-row.total.token-total {
  color: #60a5fa;
}

.table-msg {
  text-align: center;
  color: #64748b;
  padding: 20px;
}

.table-error {
  color: #b91c1c;
}

.loading-tip {
  margin: 0;
  padding: 10px 16px 14px;
  color: #64748b;
  font-size: 13px;
}

@media (max-width: 900px) {
  .topbar {
    flex-direction: column;
    align-items: stretch;
  }

  .filters {
    flex-wrap: wrap;
  }

  .summary {
    grid-template-columns: 1fr;
  }
}
</style>
