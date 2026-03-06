<script setup>
import { computed, onMounted, ref } from 'vue'

const endpoint = import.meta.env.VITE_KEYS_ENDPOINT || '/api/v1/keys'
const usageEndpoint =
  import.meta.env.VITE_USAGE_ENDPOINT || '/api/v1/usage/dashboard/api-keys-usage'
const token =
  'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX2lkIjoxODAsImVtYWlsIjoiMTQ1Njc5OTM3OUBxcS5jb20iLCJyb2xlIjoidXNlciIsInRva2VuX3ZlcnNpb24iOjAsImV4cCI6MTc3Mjg2ODY3OSwibmJmIjoxNzcyNzgyMjc5LCJpYXQiOjE3NzI3ODIyNzl9.80IQGkAKFIGo0vfGqYJ5dz5oHjFTGbqhUwD6aFdMb0o'

const loading = ref(false)
const usageLoading = ref(false)
const errorMessage = ref('')
const items = ref([])
const selectedKeyId = ref('')
const usageStatsMap = ref({})

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

const numberFormatter = new Intl.NumberFormat('zh-CN', {
  minimumFractionDigits: 0,
  maximumFractionDigits: 6,
})

const formatAmount = (value) => numberFormatter.format(Number(value || 0))

const fetchUsageStats = async (apiKeyIds) => {
  if (!apiKeyIds.length) {
    usageStatsMap.value = {}
    return
  }

  usageLoading.value = true

  try {
    const response = await fetch(usageEndpoint, {
      method: 'POST',
      headers: {
        Accept: 'application/json',
        'Content-Type': 'application/json',
        Authorization: `Bearer ${token}`,
      },
      body: JSON.stringify({
        api_key_ids: apiKeyIds,
      }),
    })

    if (!response.ok) {
      throw new Error(`统计请求失败，状态码 ${response.status}`)
    }

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

const fetchKeyOptions = async () => {
  loading.value = true
  errorMessage.value = ''

  try {
    const query = new URLSearchParams({
      page: '1',
      page_size: '10',
      timezone: 'Asia/Shanghai',
    })

    const headers = {
      Accept: 'application/json',
    }

    if (token) {
      headers.Authorization = `Bearer ${token}`
    }

    const response = await fetch(`${endpoint}?${query.toString()}`, {
      method: 'GET',
      headers,
    })

    if (!response.ok) {
      throw new Error(`请求失败，状态码 ${response.status}`)
    }

    const payload = await response.json()
    const fetchedItems = payload?.data?.items

    if (payload?.code !== 0 || !Array.isArray(fetchedItems)) {
      throw new Error(payload?.message || '接口返回格式不符合预期')
    }

    items.value = fetchedItems
    selectedKeyId.value = fetchedItems[0]?.id ? String(fetchedItems[0].id) : ''

    const apiKeyIds = fetchedItems.map((item) => item.id).filter((id) => Number.isFinite(id))
    await fetchUsageStats(apiKeyIds)
  } catch (error) {
    errorMessage.value = error instanceof Error ? error.message : '请求失败'
  } finally {
    loading.value = false
  }
}

onMounted(() => {
  fetchKeyOptions()
})
</script>

<template>
  <div class="panel">
    <h1>Key 选择</h1>

    <div class="row">
      <label for="key-select">Key 名称</label>
      <select id="key-select" v-model="selectedKeyId" :disabled="loading || !nameOptions.length">
        <option value="" disabled>
          {{ loading ? '加载中...' : '请选择 Key' }}
        </option>
        <option v-for="option in nameOptions" :key="option.id" :value="option.id">
          {{ option.name }}
        </option>
      </select>
      <button type="button" :disabled="loading" @click="fetchKeyOptions">刷新</button>
    </div>

    <p v-if="errorMessage" class="error">{{ errorMessage }}</p>

    <div v-else class="stats">
      <div class="stat-item">
        <span class="stat-label">今日消费</span>
        <strong class="stat-value">
          {{ usageLoading ? '加载中...' : formatAmount(selectedUsage?.today_actual_cost) }}
        </strong>
      </div>
      <div class="stat-item">
        <span class="stat-label">总消费</span>
        <strong class="stat-value">
          {{ usageLoading ? '加载中...' : formatAmount(selectedUsage?.total_actual_cost) }}
        </strong>
      </div>
      <div class="stat-item">
        <span class="stat-label">最大可消费</span>
        <strong class="stat-value">{{ quotaDisplay }}</strong>
      </div>
    </div>
  </div>
</template>

<style scoped lang="less">
.panel {
  width: min(640px, 100%);
  padding: 28px;
  border-radius: 16px;
  background: #ffffff;
  box-shadow: 0 18px 40px rgba(15, 23, 42, 0.08);
}

h1 {
  margin: 0 0 16px;
  font-size: 24px;
  color: #0f172a;
}

.row {
  display: grid;
  grid-template-columns: 92px 1fr auto;
  gap: 12px;
  align-items: center;
}

label {
  font-size: 14px;
  color: #334155;
}

select {
  height: 40px;
  padding: 0 12px;
  border: 1px solid #cbd5e1;
  border-radius: 10px;
  font-size: 14px;
}

button {
  height: 40px;
  padding: 0 14px;
  border: 0;
  border-radius: 10px;
  background: #2563eb;
  color: #fff;
  cursor: pointer;
}

button:disabled {
  cursor: not-allowed;
  opacity: 0.6;
}

.error {
  margin: 12px 0 0;
  color: #b91c1c;
  font-size: 14px;
}

.stats {
  margin-top: 16px;
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  gap: 12px;
}

.stat-item {
  border: 1px solid #e2e8f0;
  border-radius: 10px;
  padding: 12px;
  background: #f8fafc;
}

.stat-label {
  display: block;
  color: #64748b;
  font-size: 13px;
  margin-bottom: 6px;
}

.stat-value {
  color: #0f172a;
  font-size: 18px;
  line-height: 1.2;
}

@media (max-width: 640px) {
  .row {
    grid-template-columns: 1fr;
  }

  .stats {
    grid-template-columns: 1fr;
  }
}
</style>
