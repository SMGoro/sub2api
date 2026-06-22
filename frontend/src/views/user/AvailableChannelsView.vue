<template>
  <AppLayout>
    <TablePageLayout>
      <template #filters>
        <div class="card">
          <div class="px-6 py-4">
            <div class="flex flex-wrap items-end gap-4">
              <!-- 文本搜索 -->
              <div class="min-w-[200px] flex-1">
                <label class="input-label">{{ t('availableChannels.searchPlaceholder') }}</label>
                <div class="relative">
                  <Icon
                    name="search"
                    size="md"
                    class="absolute left-3 top-1/2 -translate-y-1/2 text-gray-400 dark:text-gray-500"
                  />
                  <input
                    v-model="searchQuery"
                    type="text"
                    :placeholder="t('availableChannels.searchPlaceholder')"
                    class="input pl-10"
                  />
                </div>
              </div>

              <!-- 平台筛选 -->
              <div class="min-w-[160px]">
                <label class="input-label">{{ t('availableChannels.filters.platform') }}</label>
                <Select
                  v-model="filterPlatform"
                  :options="platformOptions"
                  :placeholder="t('availableChannels.filters.allPlatforms')"
                  clearable
                  @change="onFilterChange"
                />
              </div>

              <!-- 分组筛选 -->
              <div class="min-w-[160px]">
                <label class="input-label">{{ t('availableChannels.filters.group') }}</label>
                <Select
                  v-model="filterGroup"
                  :options="groupOptions"
                  :placeholder="t('availableChannels.filters.allGroups')"
                  clearable
                  @change="onFilterChange"
                />
              </div>

              <!-- 计费模式筛选 -->
              <div class="min-w-[140px]">
                <label class="input-label">{{ t('availableChannels.filters.billingMode') }}</label>
                <Select
                  v-model="filterBillingMode"
                  :options="billingModeOptions"
                  :placeholder="t('availableChannels.filters.allBillingModes')"
                  clearable
                  @change="onFilterChange"
                />
              </div>

              <!-- 操作按钮 -->
              <div class="ml-auto flex items-center gap-3">
                <button @click="resetFilters" class="btn btn-secondary">
                  {{ t('common.reset') }}
                </button>
                <button
                  @click="loadChannels"
                  :disabled="loading"
                  class="btn btn-secondary"
                  :title="t('common.refresh', 'Refresh')"
                >
                  <Icon name="refresh" size="md" :class="loading ? 'animate-spin' : ''" />
                </button>
              </div>
            </div>
          </div>
        </div>
      </template>

      <template #table>
        <AvailableChannelsTable
          :columns="columnLabels"
          :rows="paginatedRows"
          :loading="loading"
          :user-group-rates="userGroupRates"
          pricing-key-prefix="availableChannels.pricing"
          :no-pricing-label="t('availableChannels.noPricing')"
          :no-models-label="t('availableChannels.noModels')"
          :empty-label="t('availableChannels.empty')"
        />
      </template>

      <template #pagination>
        <Pagination
          v-if="pagination.total > 0"
          :page="pagination.page"
          :total="pagination.total"
          :page-size="pagination.page_size"
          :page-size-options="CHANNELS_PAGE_SIZE_OPTIONS"
          @update:page="handlePageChange"
          @update:pageSize="handlePageSizeChange"
        />
      </template>
    </TablePageLayout>
  </AppLayout>
</template>

<script setup lang="ts">
import { computed, onMounted, reactive, ref, watch } from 'vue'
import { useI18n } from 'vue-i18n'
import AppLayout from '@/components/layout/AppLayout.vue'
import TablePageLayout from '@/components/layout/TablePageLayout.vue'
import Icon from '@/components/icons/Icon.vue'
import Select from '@/components/common/Select.vue'
import Pagination from '@/components/common/Pagination.vue'
import AvailableChannelsTable, { type ChannelModelRow } from '@/components/channels/AvailableChannelsTable.vue'
import userChannelsAPI, { type UserAvailableChannel } from '@/api/channels'
import userGroupsAPI from '@/api/groups'
import { useAppStore } from '@/stores/app'
import { extractApiErrorMessage } from '@/utils/apiError'
import {
  BILLING_MODE_TOKEN,
  BILLING_MODE_PER_REQUEST,
  BILLING_MODE_IMAGE,
  type BillingMode
} from '@/constants/channel'

const { t } = useI18n()
const appStore = useAppStore()

const channels = ref<UserAvailableChannel[]>([])
const userGroupRates = ref<Record<number, number>>({})
const loading = ref(false)

// ── 筛选状态 ──────────────────────────────────────────────────────
const searchQuery = ref('')
const filterPlatform = ref<string | null>(null)
const filterGroup = ref<string | null>(null)
const filterBillingMode = ref<string | null>(null)

// ── 分页状态 ──────────────────────────────────────────────────────
// 按模型行分页（每个模型一行），独立持久化，避免与"使用记录"等视图共享页大小。
const CHANNELS_PAGE_SIZE_KEY = 'available-channels-page-size'
const CHANNELS_PAGE_SIZE_OPTIONS = [10, 20, 50, 100]

function getChannelPageSize(): number {
  try {
    const stored = localStorage.getItem(CHANNELS_PAGE_SIZE_KEY)
    if (stored !== null) {
      const parsed = Number(stored)
      if (Number.isFinite(parsed) && CHANNELS_PAGE_SIZE_OPTIONS.includes(parsed)) {
        return parsed
      }
    }
  } catch { /* ignore */ }
  return 20
}

const pagination = reactive({
  page: 1,
  page_size: getChannelPageSize(),
  total: 0,
})

const columnLabels = computed(() => ({
  name: t('availableChannels.columns.name'),
  description: t('availableChannels.columns.description'),
  platform: t('availableChannels.columns.platform'),
  groups: t('availableChannels.columns.groups'),
  model: t('availableChannels.columns.model'),
  billingMode: t('availableChannels.columns.billingMode'),
  inputPrice: t('availableChannels.columns.inputPrice'),
  outputPrice: t('availableChannels.columns.outputPrice'),
  cacheWritePrice: t('availableChannels.columns.cacheWritePrice'),
  cacheReadPrice: t('availableChannels.columns.cacheReadPrice'),
  perRequestPrice: t('availableChannels.columns.perRequestPrice'),
  tiers: t('availableChannels.columns.tiers'),
  supportedModels: t('availableChannels.columns.supportedModels'),
}))

// ── 筛选选项（从数据中提取唯一值） ────────────────────────────────
const platformOptions = computed(() => {
  const set = new Set<string>()
  for (const ch of channels.value) {
    for (const sec of ch.platforms) {
      set.add(sec.platform)
    }
  }
  return Array.from(set).sort().map((p) => ({ value: p, label: p }))
})

const groupOptions = computed(() => {
  const set = new Set<string>()
  for (const ch of channels.value) {
    for (const sec of ch.platforms) {
      for (const g of sec.groups) {
        set.add(g.name)
      }
    }
  }
  return Array.from(set).sort().map((g) => ({ value: g, label: g }))
})

const billingModeOptions = computed(() => [
  { value: BILLING_MODE_TOKEN, label: t('availableChannels.pricing.billingModeToken') },
  { value: BILLING_MODE_PER_REQUEST, label: t('availableChannels.pricing.billingModePerRequest') },
  { value: BILLING_MODE_IMAGE, label: t('availableChannels.pricing.billingModeImage') },
])

// ── 筛选逻辑 ──────────────────────────────────────────────────────
// 多维度筛选：文本搜索 + 平台 + 分组 + 计费模式。
// - 文本搜索命中渠道名/描述 → 保留整个渠道（但仍受其他维度筛选约束）
// - 否则在 section 内按 platform/group/model 过滤，且仅保留匹配的模型
const filteredChannels = computed<UserAvailableChannel[]>(() => {
  const q = searchQuery.value.trim().toLowerCase()
  const plat = filterPlatform.value
  const grp = filterGroup.value
  const bm = filterBillingMode.value as BillingMode | null

  // 无任何筛选时直接返回
  if (!q && !plat && !grp && !bm) return channels.value

  return channels.value
    .map((ch) => {
      // 文本搜索：渠道名或描述命中 → 整个渠道保留（跳过 section/model 级文本过滤）
      const channelTextHit =
        !q ||
        ch.name.toLowerCase().includes(q) ||
        (ch.description || '').toLowerCase().includes(q)

      const matchingSections = ch.platforms.filter((p) => {
        // 平台筛选
        if (plat && p.platform !== plat) return false
        // 分组筛选
        if (grp && !p.groups.some((g) => g.name === grp)) return false
        // 计费模式筛选：section 内至少有一个匹配模型
        if (bm && !p.supported_models.some((m) => m.pricing?.billing_mode === bm)) return false
        return true
      })

      if (matchingSections.length === 0) return null

      // 构建过滤后的 sections，按需过滤模型
      const sections = matchingSections.map((p) => {
        let models = p.supported_models

        // 计费模式：仅保留匹配模型
        if (bm) {
          models = models.filter((m) => m.pricing?.billing_mode === bm)
        }

        // 文本搜索：渠道名/描述未命中时，在 section 内按模型名/平台名/分组名过滤
        if (!channelTextHit) {
          models = models.filter(
            (m) =>
              m.name.toLowerCase().includes(q) ||
              p.platform.toLowerCase().includes(q) ||
              p.groups.some((g) => g.name.toLowerCase().includes(q)),
          )
        }

        return { ...p, supported_models: models }
      }).filter((p) => p.supported_models.length > 0 || channelTextHit)

      if (sections.length === 0) return null

      return { ...ch, platforms: sections }
    })
    .filter((ch): ch is UserAvailableChannel => ch !== null)
})

// ── 扁平化为模型行 ────────────────────────────────────────────────
// 将渠道 → 平台 section → 模型的三层嵌套展开为扁平的模型行数组，
// 每个模型一行，便于按行分页和渲染。
const flatRows = computed<ChannelModelRow[]>(() => {
  const result: ChannelModelRow[] = []
  for (const ch of filteredChannels.value) {
    for (const sec of ch.platforms) {
      if (sec.supported_models.length === 0) {
        result.push({
          channelName: ch.name,
          channelDescription: ch.description || '',
          section: sec,
          model: null,
        })
      } else {
        for (const model of sec.supported_models) {
          result.push({
            channelName: ch.name,
            channelDescription: ch.description || '',
            section: sec,
            model,
          })
        }
      }
    }
  }
  return result
})

// ── 分页（客户端，按模型行分页） ──────────────────────────────────
const paginatedRows = computed<ChannelModelRow[]>(() => {
  const start = (pagination.page - 1) * pagination.page_size
  return flatRows.value.slice(start, start + pagination.page_size)
})

// 筛选变化时重置分页并更新总数
watch(
  flatRows,
  (list) => {
    pagination.total = list.length
    const maxPage = Math.max(1, Math.ceil(list.length / pagination.page_size))
    if (pagination.page > maxPage) pagination.page = maxPage
  },
  { immediate: true },
)

function onFilterChange() {
  pagination.page = 1
}

function resetFilters() {
  searchQuery.value = ''
  filterPlatform.value = null
  filterGroup.value = null
  filterBillingMode.value = null
  pagination.page = 1
}

function handlePageChange(page: number) {
  pagination.page = page
}

function handlePageSizeChange(pageSize: number) {
  pagination.page_size = CHANNELS_PAGE_SIZE_OPTIONS.includes(pageSize) ? pageSize : 20
  try {
    localStorage.setItem(CHANNELS_PAGE_SIZE_KEY, String(pagination.page_size))
  } catch { /* ignore */ }
  pagination.page = 1
}

async function loadChannels() {
  loading.value = true
  try {
    const [list, rates] = await Promise.all([
      userChannelsAPI.getAvailable(),
      userGroupsAPI.getUserGroupRates().catch((err: unknown) => {
        console.error('Failed to load user group rates:', err)
        return {} as Record<number, number>
      }),
    ])
    channels.value = list
    userGroupRates.value = rates
    pagination.page = 1
  } catch (err: unknown) {
    appStore.showError(extractApiErrorMessage(err, t('common.error')))
  } finally {
    loading.value = false
  }
}

onMounted(loadChannels)
</script>
