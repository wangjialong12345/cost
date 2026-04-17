<script setup>
import { computed, onBeforeUnmount, onMounted, ref } from 'vue'

const logEndpoint = 'https://cn.nyi.cn/api/log'
const keysEndpoint = 'https://cn.nyi.cn/api/keys'
const token = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOjQsInNpZ24iOiJlZWE3MDY2YjNhNTk0MTcxMzIyNGYwNDYxMGM5ZWUwMyIsInJvbGUiOiJ1c2VyIiwiZXhwIjoxNzc5MDg2ODYwLCJuYmYiOjE3NzY0MDg0NjAsImlhdCI6MTc3NjQwODQ2MH0.dmzpT-Xa2vf4nt34Ox0J4xE-tAGaWNDULJD89RE6Ppo'
const timezone = 'Asia/Shanghai'

const loading = ref(false)
const keysLoading = ref(false)
const detailsLoading = ref(false)
const detailsLoadingMore = ref(false)
const errorMessage = ref('')
const detailsErrorMessage = ref('')
const keys = ref([])
const selectedKeyId = ref('')
const usageItems = ref([])
const usageTotal = ref(0)
const detailsCurrentPage = ref(1)
const tableWrap = ref(null)
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

const formatRequestType = (stream) => (stream ? '流式' : '普通')

const formatTokenCount = (value) => {
  const num = Number(value || 0)
  if (num >= 1000000) return `${(num / 1000000).toFixed(1)}M`
  if (num >= 1000) return `${(num / 1000).toFixed(1)}K`
  return String(num)
}

const selectedKey = computed(() =>
  keys.value.find((k) => String(k.id) === selectedKeyId.value) || null,
)

const keyQuota = computed(() => ({
  amount: Number(selectedKey.value?.amount || 0),
  used: Number(selectedKey.value?.used || 0),
}))

const summaryCards = computed(() => [
  {
    label: '可用额度',
    value: keysLoading.value ? '加载中...' : `$${formatAmount(keyQuota.value.amount)}`,
  },
  {
    label: '已用额度',
    value: keysLoading.value ? '加载中...' : `$${formatAmount(keyQuota.value.used)}`,
  },
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

const handleTableScroll = () => {
  if (detailsLoading.value || detailsLoadingMore.value) return
  if (usageItems.value.length >= usageTotal.value) return
  const el = tableWrap.value
  if (!el) return
  if (el.scrollTop + el.clientHeight >= el.scrollHeight - 120) {
    fetchData(detailsCurrentPage.value + 1)
  }
}

const fetchKeys = async () => {
  keysLoading.value = true
  try {
    const params = new URLSearchParams({
      page: '1',
      size: '20',
      'sl[name]': 'true',
      'sl[token]': 'true',
      'sl[note]': 'true',
    })
    const response = await fetch(`${keysEndpoint}?${params}`, {
      headers: { Accept: 'application/json', Authorization: token },
    })
    if (!response.ok) throw new Error(`密钥请求失败，状态码 ${response.status}`)
    const payload = await response.json()
    if (payload?.code !== 0 || !Array.isArray(payload?.data?.records)) {
      throw new Error(payload?.msg || '密钥接口返回格式不符合预期')
    }
    const yunwuKeys = payload.data.records.filter((k) => k.name?.includes('云雾'))
    keys.value = yunwuKeys
    if (!selectedKeyId.value && yunwuKeys.length) {
      selectedKeyId.value = String(yunwuKeys[0].id)
    }
  } catch (error) {
    errorMessage.value = error instanceof Error ? error.message : '密钥请求失败'
  } finally {
    keysLoading.value = false
  }
}

const handleKeyChange = () => {
  fetchData()
}

const fetchData = async (page = 1) => {
  const isLoadMore = page > 1

  if (!isLoadMore) {
    detailsCurrentPage.value = 1
    usageItems.value = []
    usageTotal.value = 0
    loading.value = true
    detailsLoading.value = true
    errorMessage.value = ''
    detailsErrorMessage.value = ''
  } else {
    detailsLoadingMore.value = true
  }

  try {
    const params = new URLSearchParams({
      page: String(page),
      size: '100',
      'search[type]': '0',
      'sl[ip]': 'true',
      'sl[model]': 'true',
      'sl[reqPath]': 'true',
      'sl[detail]': 'true',
    })
    if (selectedKeyId.value) params.set('search[keyId]', selectedKeyId.value)

    const response = await fetch(`${logEndpoint}?${params}`, {
      headers: {
        Accept: 'application/json',
        Authorization: token,
      },
    })

    if (!response.ok) throw new Error(`请求失败，状态码 ${response.status}`)

    const payload = await response.json()
    if (payload?.code !== 0 || !Array.isArray(payload?.data?.records)) {
      throw new Error(payload?.msg || '接口返回格式不符合预期')
    }

    if (isLoadMore) {
      usageItems.value = [...usageItems.value, ...payload.data.records]
    } else {
      usageItems.value = payload.data.records
    }
    usageTotal.value = Number(payload.data.total || 0)
    detailsCurrentPage.value = page
  } catch (error) {
    const msg = error instanceof Error ? error.message : '请求失败'
    if (isLoadMore) {
      detailsErrorMessage.value = msg
    } else {
      errorMessage.value = msg
    }
  } finally {
    if (isLoadMore) {
      detailsLoadingMore.value = false
    } else {
      loading.value = false
      detailsLoading.value = false
    }
  }
}

let autoRefreshTimer = null

onMounted(async () => {
  await fetchKeys()
  fetchData()
  autoRefreshTimer = setInterval(async () => {
    await fetchKeys()
    fetchData()
  }, 30 * 1000)
})

onBeforeUnmount(() => {
  clearInterval(autoRefreshTimer)
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
        <label for="key-select">密钥</label>
        <select id="key-select" v-model="selectedKeyId" :disabled="keysLoading" @change="handleKeyChange">
          <option v-for="k in keys" :key="k.id" :value="String(k.id)">{{ k.name }}</option>
        </select>
        <button type="button" :disabled="loading || keysLoading" @click="fetchKeys(); fetchData()">刷新</button>
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

      <div ref="tableWrap" class="table-wrap" @scroll="handleTableScroll">
        <table class="pc-table">
          <thead>
            <tr>
              <th>KEY 名称</th>
              <th>时间</th>
              <th>模型</th>
              <th>渠道</th>
              <th>类型</th>
              <th>TOKEN</th>
              <th>费用</th>
              <th>首 TOKEN</th>
              <th>耗时</th>
              <th>IP</th>
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
              <td>{{ row.kName || '-' }}</td>
              <td>{{ formatDateTime(row.created) }}</td>
              <td>{{ row.model || '-' }}</td>
              <td>{{ row.cgName || '-' }}</td>
              <td>
                <span class="type-tag">{{ formatRequestType(row.stream) }}</span>
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
                      <span class="token-down">↓ {{ row.inputTokens || 0 }}</span>
                      <span class="token-up">↑ {{ row.outputTokens || 0 }}</span>
                    </div>
                    <div class="token-line token-sub">
                      <span class="token-read">⟲ {{ formatTokenCount(row.inputCacheTokens) }}</span>
                      <span class="token-write">✎ {{ formatTokenCount(row.createInputCacheTokens) }}</span>
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
                  <span class="cost">{{ formatUsd(row.changeAmount) }}</span>
                  <span class="cost-info" aria-hidden="true">i</span>
                </div>
              </td>
              <td>{{ formatSeconds(row.preMs) }}</td>
              <td>{{ formatSeconds(row.totalMs) }}</td>
              <td>{{ row.ip || '-' }}</td>
            </tr>
          </tbody>
        </table>

        <div class="mobile-list">
          <p v-if="detailsErrorMessage" class="table-msg table-error">{{ detailsErrorMessage }}</p>
          <p v-else-if="!detailsLoading && !usageItems.length" class="table-msg">暂无明细</p>
          <div v-for="row in usageItems" :key="'m-' + row.id" class="mobile-card">
            <div class="mobile-card-head">
              <span class="mobile-card-name">{{ row.kName || '-' }}</span>
              <span class="mobile-card-model">{{ row.model || '-' }}</span>
            </div>
            <div class="mobile-card-head">
              <span class="mobile-card-time">{{ formatDateTime(row.created) }}</span>
              <span class="type-tag">{{ formatRequestType(row.stream) }}</span>
            </div>
            <div class="mobile-card-tokens">
              <span class="token-down">↓ {{ row.inputTokens || 0 }}</span>
              <span class="token-up">↑ {{ row.outputTokens || 0 }}</span>
              <span class="token-read">⟲ {{ formatTokenCount(row.inputCacheTokens) }}</span>
              <span class="token-write">✎ {{ formatTokenCount(row.createInputCacheTokens) }}</span>
            </div>
            <div class="mobile-card-row">
              <span class="mobile-card-label">费用</span>
              <span class="cost">{{ formatUsd(row.changeAmount) }}</span>
              <span class="mobile-card-label">渠道</span>
              <span>{{ row.cgName || '-' }}</span>
            </div>
            <div class="mobile-card-row">
              <span class="mobile-card-label">首Token</span>
              <span>{{ formatSeconds(row.preMs) }}</span>
              <span class="mobile-card-label">耗时</span>
              <span>{{ formatSeconds(row.totalMs) }}</span>
            </div>
          </div>
        </div>
      </div>

      <p v-if="detailsLoading" class="loading-tip">明细加载中...</p>
      <p v-else-if="detailsLoadingMore" class="loading-tip">加载更多...</p>
      <p v-else-if="!detailsLoading && usageItems.length > 0 && usageItems.length >= usageTotal" class="loading-tip end-tip">已加载全部 {{ usageTotal }} 条</p>
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
          <span>{{ formatUsd(activeCostRow.info?.useInPrice) }}</span>
        </div>
        <div class="tooltip-row">
          <span>输出成本</span>
          <span>{{ formatUsd(activeCostRow.info?.useOutPrice) }}</span>
        </div>
        <div class="tooltip-row">
          <span>缓存创建成本</span>
          <span>{{ formatUsd(activeCostRow.info?.useCacheWritePrice) }}</span>
        </div>
        <div class="tooltip-row">
          <span>缓存读取成本</span>
          <span>{{ formatUsd(activeCostRow.info?.useCacheReadPrice) }}</span>
        </div>
        <div class="tooltip-row">
          <span>倍率</span>
          <span>{{ Number(activeCostRow.info?.ratio || 1).toFixed(2) }}x</span>
        </div>
        <div class="tooltip-row">
          <span>原始</span>
          <span>{{ formatUsd(activeCostRow.info?.breakdown?.total) }}</span>
        </div>
        <div class="tooltip-row total">
          <span>计费</span>
          <span>{{ formatUsd(activeCostRow.changeAmount) }}</span>
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
          <span>{{ Number(activeTokenRow.inputTokens || 0) }}</span>
        </div>
        <div class="tooltip-row">
          <span>输出 Token</span>
          <span>{{ Number(activeTokenRow.outputTokens || 0) }}</span>
        </div>
        <div class="tooltip-row">
          <span>缓存创建 Token</span>
          <span>{{ Number(activeTokenRow.createInputCacheTokens || 0) }}</span>
        </div>
        <div class="tooltip-row">
          <span>缓存读取 Token</span>
          <span>{{ Number(activeTokenRow.inputCacheTokens || 0) }}</span>
        </div>
        <div class="tooltip-row total token-total">
          <span>总 Token</span>
          <span>{{
            Number(activeTokenRow.inputTokens || 0) +
            Number(activeTokenRow.outputTokens || 0) +
            Number(activeTokenRow.createInputCacheTokens || 0) +
            Number(activeTokenRow.inputCacheTokens || 0)
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
  height: 100%;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  box-sizing: border-box;
}

.topbar {
  display: flex;
  justify-content: space-between;
  gap: 20px;
  align-items: flex-end;
  margin-bottom: 10px;
  flex-shrink: 0;
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
  grid-template-columns: repeat(2, minmax(220px, 260px));
  justify-content: flex-start;
  gap: 10px;
  margin: 10px 0 10px;
  flex-shrink: 0;
}

.summary-item {
  background: linear-gradient(180deg, #ffffff 0%, #f8fbff 100%);
  border: 1px solid #c9d3e3;
  border-left: 4px solid #3b82f6;
  border-radius: 12px;
  padding: 8px 14px;
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
  flex: 1;
  min-height: 0;
  display: flex;
  flex-direction: column;
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
  flex: 1;
  min-height: 0;
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

.end-tip {
  text-align: center;
  color: #94a3b8;
}

.mobile-list {
  display: none;
}

@media (max-width: 900px) {
  .usage-page {
    height: auto;
    overflow: visible;
    padding-bottom: 20px;
  }

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

  .table-card {
    flex: none;
    min-height: auto;
  }

  .table-wrap {
    overflow: visible;
    flex: none;
  }

  .pc-table {
    display: none;
  }

  .mobile-list {
    display: block;
    padding: 8px 12px 4px;
  }

  .mobile-card {
    border: 1px solid #e6ebf2;
    border-radius: 10px;
    padding: 10px 12px;
    margin-bottom: 8px;
    background: #f8fafc;
  }

  .mobile-card-head {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 6px;
  }

  .mobile-card-name {
    font-weight: 600;
    color: #1f2937;
    font-size: 14px;
  }

  .mobile-card-model {
    color: #475569;
    font-size: 13px;
  }

  .mobile-card-time {
    color: #64748b;
    font-size: 12px;
  }

  .mobile-card-tokens {
    display: flex;
    gap: 10px;
    flex-wrap: wrap;
    padding: 6px 0;
    border-top: 1px solid #e9edf3;
    border-bottom: 1px solid #e9edf3;
    margin: 4px 0;
    font-size: 13px;
  }

  .mobile-card-row {
    display: flex;
    align-items: center;
    gap: 6px;
    margin-top: 6px;
    font-size: 13px;
    color: #374151;
  }

  .mobile-card-label {
    color: #94a3b8;
    font-size: 12px;
    flex-shrink: 0;

    &:nth-child(3) {
      margin-left: auto;
    }
  }
}
</style>
