<template>
  <div class="table-wrapper">
    <table class="w-full min-w-max border-collapse text-sm">
        <thead>
          <tr class="border-b border-gray-100 bg-gray-50/50 text-xs font-medium uppercase tracking-wide text-gray-500 dark:border-dark-700 dark:bg-dark-800/50 dark:text-gray-400">
            <th class="w-[140px] px-4 py-3 text-left">{{ columns.name }}</th>
            <th class="w-[180px] px-4 py-3 text-left">{{ columns.description }}</th>
            <th class="w-[120px] px-4 py-3 text-left">{{ columns.platform }}</th>
            <th class="w-[200px] px-4 py-3 text-left">{{ columns.groups }}</th>
            <th class="w-[180px] px-4 py-3 text-left">{{ columns.model }}</th>
            <th class="w-[90px] px-4 py-3 text-left">{{ columns.billingMode }}</th>
            <th class="w-[90px] px-4 py-3 text-right">{{ columns.inputPrice }}</th>
            <th class="w-[90px] px-4 py-3 text-right">{{ columns.outputPrice }}</th>
            <th class="w-[90px] px-4 py-3 text-right">{{ columns.cacheWritePrice }}</th>
            <th class="w-[90px] px-4 py-3 text-right">{{ columns.cacheReadPrice }}</th>
            <th class="w-[90px] px-4 py-3 text-right">{{ columns.perRequestPrice }}</th>
            <th class="w-[80px] px-4 py-3 text-center">{{ columns.tiers }}</th>
          </tr>
        </thead>
        <tbody v-if="loading">
          <tr>
            <td colspan="12" class="py-10 text-center">
              <Icon name="refresh" size="lg" class="inline-block animate-spin text-gray-400" />
            </td>
          </tr>
        </tbody>
        <tbody v-else-if="rows.length === 0">
          <tr>
            <td colspan="12" class="py-12 text-center">
              <Icon name="inbox" size="xl" class="mx-auto mb-3 h-12 w-12 text-gray-400" />
              <p class="text-sm text-gray-500 dark:text-gray-400">{{ emptyLabel }}</p>
            </td>
          </tr>
        </tbody>
        <tbody v-else>
          <tr
            v-for="(row, idx) in rows"
            :key="idx"
            class="border-b border-gray-100 transition-colors hover:bg-gray-50/40 dark:border-dark-700/50 dark:hover:bg-dark-800/40"
          >
            <!-- 渠道名 -->
            <td class="px-4 py-3 align-top font-medium text-gray-900 dark:text-white">
              {{ row.channelName }}
            </td>

            <!-- 描述 -->
            <td class="px-4 py-3 align-top text-xs text-gray-500 dark:text-gray-400">
              <template v-if="row.channelDescription">{{ row.channelDescription }}</template>
              <span v-else class="text-gray-400">-</span>
            </td>

            <!-- 平台徽章 -->
            <td class="align-top px-4 py-3">
              <span
                :class="[
                  'inline-flex items-center gap-1 rounded-md border px-2 py-0.5 text-[11px] font-medium uppercase',
                  platformBadgeClass(row.section.platform),
                ]"
              >
                <PlatformIcon :platform="row.section.platform as GroupPlatform" size="xs" />
                {{ row.section.platform }}
              </span>
            </td>

            <!-- 分组：专属分组在前（紫色 shield 行），公开分组在后（灰色 globe 行）。 -->
            <td class="align-top px-4 py-3">
              <div class="flex flex-col gap-1.5">
                <div
                  v-if="exclusiveGroups(row.section).length > 0"
                  class="flex flex-wrap items-center gap-1.5"
                >
                  <span
                    class="inline-flex items-center gap-0.5 text-[10px] font-medium uppercase text-purple-600 dark:text-purple-400"
                    :title="t('availableChannels.exclusiveTooltip')"
                  >
                    <Icon name="shield" size="xs" class="h-3 w-3" />
                    {{ t('availableChannels.exclusive') }}
                  </span>
                  <GroupBadge
                    v-for="g in exclusiveGroups(row.section)"
                    :key="`ex-${g.id}`"
                    :name="g.name"
                    :platform="g.platform as GroupPlatform"
                    :subscription-type="(g.subscription_type || 'standard') as SubscriptionType"
                    :rate-multiplier="g.rate_multiplier"
                    :user-rate-multiplier="userGroupRates[g.id] ?? null"
                    always-show-rate
                  />
                </div>
                <div
                  v-if="publicGroups(row.section).length > 0"
                  class="flex flex-wrap items-center gap-1.5"
                >
                  <span
                    class="inline-flex items-center gap-0.5 text-[10px] font-medium uppercase text-gray-500 dark:text-gray-400"
                    :title="t('availableChannels.publicTooltip')"
                  >
                    <Icon name="globe" size="xs" class="h-3 w-3" />
                    {{ t('availableChannels.public') }}
                  </span>
                  <GroupBadge
                    v-for="g in publicGroups(row.section)"
                    :key="`pub-${g.id}`"
                    :name="g.name"
                    :platform="g.platform as GroupPlatform"
                    :subscription-type="(g.subscription_type || 'standard') as SubscriptionType"
                    :rate-multiplier="g.rate_multiplier"
                    :user-rate-multiplier="userGroupRates[g.id] ?? null"
                    always-show-rate
                  />
                </div>
                <span v-if="row.section.groups.length === 0" class="text-xs text-gray-400">-</span>
              </div>
            </td>

            <!-- 模型名 -->
            <td class="align-top px-4 py-3">
              <template v-if="row.model">
                <div class="flex items-center gap-1.5">
                  <PlatformIcon
                    v-if="!row.model.platform"
                    :platform="row.section.platform as GroupPlatform"
                    size="xs"
                  />
                  <span class="font-medium text-gray-900 dark:text-white">{{ row.model.name }}</span>
                </div>
              </template>
              <span v-else class="text-xs text-gray-400">{{ noModelsLabel }}</span>
            </td>

            <!-- 计费模式 -->
            <td class="align-top px-4 py-3">
              <span
                v-if="row.model?.pricing"
                class="inline-flex items-center rounded px-1.5 py-0.5 text-[11px] font-medium"
                :class="billingModeBadgeClass(row.model.pricing.billing_mode)"
              >
                {{ billingModeLabel(row.model.pricing.billing_mode) }}
              </span>
              <span v-else class="text-xs text-gray-400">{{ noPricingLabel }}</span>
            </td>

            <!-- 输入价 / 1M token -->
            <td class="align-top px-4 py-3 text-right">
              <span v-if="isTokenMode(row.model)" class="font-mono text-xs text-gray-700 dark:text-gray-300">
                {{ formatScaled(row.model!.pricing!.input_price, perMillionScale) }}
              </span>
              <span v-else class="text-xs text-gray-400">-</span>
            </td>

            <!-- 输出价 / 1M token -->
            <td class="align-top px-4 py-3 text-right">
              <span v-if="isTokenMode(row.model)" class="font-mono text-xs text-gray-700 dark:text-gray-300">
                {{ formatScaled(row.model!.pricing!.output_price, perMillionScale) }}
              </span>
              <span v-else class="text-xs text-gray-400">-</span>
            </td>

            <!-- 缓存写入价 / 1M token -->
            <td class="align-top px-4 py-3 text-right">
              <span v-if="isTokenMode(row.model)" class="font-mono text-xs text-gray-700 dark:text-gray-300">
                {{ formatScaled(row.model!.pricing!.cache_write_price, perMillionScale) }}
              </span>
              <span v-else class="text-xs text-gray-400">-</span>
            </td>

            <!-- 缓存读取价 / 1M token -->
            <td class="align-top px-4 py-3 text-right">
              <span v-if="isTokenMode(row.model)" class="font-mono text-xs text-gray-700 dark:text-gray-300">
                {{ formatScaled(row.model!.pricing!.cache_read_price, perMillionScale) }}
              </span>
              <span v-else class="text-xs text-gray-400">-</span>
            </td>

            <!-- 单次价（按次/按图片） -->
            <td class="align-top px-4 py-3 text-right">
              <span v-if="isPerRequestMode(row.model) || isImageMode(row.model)" class="font-mono text-xs text-gray-700 dark:text-gray-300">
                {{ formatScaled(row.model!.pricing!.per_request_price ?? row.model!.pricing!.image_output_price, 1) }}
              </span>
              <span v-else-if="isTokenMode(row.model) && row.model!.pricing!.image_output_price != null && row.model!.pricing!.image_output_price > 0" class="font-mono text-xs text-gray-700 dark:text-gray-300">
                {{ formatScaled(row.model!.pricing!.image_output_price, perMillionScale) }}
              </span>
              <span v-else class="text-xs text-gray-400">-</span>
            </td>

            <!-- 阶梯定价 -->
            <td class="align-top px-4 py-3 text-center">
              <span
                v-if="row.model?.pricing?.intervals && row.model.pricing.intervals.length > 0"
                class="inline-flex cursor-help items-center gap-0.5 rounded bg-amber-100 px-1.5 py-0.5 text-[11px] font-medium text-amber-700 transition-colors hover:bg-amber-200 dark:bg-amber-900/30 dark:text-amber-400 dark:hover:bg-amber-900/50"
                @mouseenter="showTiersPopover($event, row.model!)"
                @mouseleave="hideTiersPopover"
              >
                <Icon name="chartBar" size="xs" class="h-3 w-3" />
                {{ t('availableChannels.pricing.tiersCount', { count: row.model.pricing.intervals.length }) }}
              </span>
              <span v-else class="text-xs text-gray-400">-</span>
            </td>
          </tr>
        </tbody>
      </table>

    <!-- 阶梯定价详情 Popover（Teleport 到 body 避免被 overflow-hidden 裁剪） -->
    <Teleport to="body">
      <div
        v-show="tiersPopover.visible"
        ref="tiersPopoverEl"
        role="tooltip"
        class="pointer-events-none fixed z-[99999] w-80 max-w-[min(22rem,calc(100vw-1rem))] rounded-lg border border-gray-200 bg-white text-xs shadow-xl dark:border-dark-600 dark:bg-dark-800"
        :style="tiersPopover.style"
      >
        <div class="flex items-center justify-between gap-2 rounded-t-lg border-b border-gray-100 bg-amber-50 px-3 py-2 dark:border-dark-600 dark:bg-amber-900/20">
          <span class="truncate font-semibold text-amber-800 dark:text-amber-400">
            {{ tiersPopover.modelName }}
          </span>
          <span class="flex-shrink-0 text-[10px] uppercase tracking-wide text-amber-600 dark:text-amber-500">
            {{ t('availableChannels.pricing.intervals') }}
          </span>
        </div>
        <div class="p-3">
          <div class="space-y-1.5">
            <div
              v-for="(iv, idx) in tiersPopover.intervals"
              :key="idx"
              class="flex justify-between gap-2 text-[11px]"
            >
              <span class="text-gray-500 dark:text-gray-400">
                <template v-if="iv.tier_label">{{ iv.tier_label }}</template>
                <template v-else>{{ formatRange(iv.min_tokens, iv.max_tokens) }}</template>
              </span>
              <span class="font-mono text-gray-700 dark:text-gray-300">{{ formatInterval(iv, tiersPopover.billingMode) }}</span>
            </div>
          </div>
        </div>
      </div>
    </Teleport>
  </div>
</template>

<script lang="ts">
import type { UserChannelPlatformSection, UserSupportedModel } from '@/api/channels'

/** 扁平化的模型行：每个模型一行，包含渠道名/描述/平台/分组等完整信息。 */
export interface ChannelModelRow {
  channelName: string
  channelDescription: string
  section: UserChannelPlatformSection
  model: UserSupportedModel | null
}
</script>

<script setup lang="ts">
import { nextTick, onBeforeUnmount, ref } from 'vue'
import { useI18n } from 'vue-i18n'
import Icon from '@/components/icons/Icon.vue'
import PlatformIcon from '@/components/common/PlatformIcon.vue'
import GroupBadge from '@/components/common/GroupBadge.vue'
import { formatScaled } from '@/utils/pricing'
import type { UserPricingInterval } from '@/api/channels'
import type { GroupPlatform, SubscriptionType } from '@/types'
import { platformBadgeClass } from '@/utils/platformColors'
import {
  BILLING_MODE_TOKEN,
  BILLING_MODE_PER_REQUEST,
  BILLING_MODE_IMAGE,
  type BillingMode
} from '@/constants/channel'

const props = defineProps<{
  columns: {
    name: string
    description: string
    platform: string
    groups: string
    model: string
    billingMode: string
    inputPrice: string
    outputPrice: string
    cacheWritePrice: string
    cacheReadPrice: string
    perRequestPrice: string
    tiers: string
    supportedModels: string
  }
  rows: ChannelModelRow[]
  loading: boolean
  pricingKeyPrefix: string
  noPricingLabel: string
  noModelsLabel: string
  emptyLabel: string
  /** 用户专属倍率（group_id → multiplier）；无专属时由 GroupBadge 仅显示默认倍率。 */
  userGroupRates: Record<number, number>
}>()

// Suppress unused warning — props is accessed via template automatically but
// the explicit reference here keeps the linter from flagging userGroupRates.
void props.userGroupRates

const { t } = useI18n()

/** 按 token 定价展示时的换算单位：每百万 token。 */
const perMillionScale = 1_000_000

// ── 分组分类 ──────────────────────────────────────────────────────
function exclusiveGroups(section: UserChannelPlatformSection) {
  return section.groups.filter((g) => g.is_exclusive)
}

function publicGroups(section: UserChannelPlatformSection) {
  return section.groups.filter((g) => !g.is_exclusive)
}

// ── 计费模式辅助 ──────────────────────────────────────────────────
function isTokenMode(model: UserSupportedModel | null): boolean {
  return !!model?.pricing && model.pricing.billing_mode === BILLING_MODE_TOKEN
}

function isPerRequestMode(model: UserSupportedModel | null): boolean {
  return !!model?.pricing && model.pricing.billing_mode === BILLING_MODE_PER_REQUEST
}

function isImageMode(model: UserSupportedModel | null): boolean {
  return !!model?.pricing && model.pricing.billing_mode === BILLING_MODE_IMAGE
}

function billingModeLabel(mode: BillingMode): string {
  switch (mode) {
    case BILLING_MODE_TOKEN:
      return t('availableChannels.pricing.billingModeToken')
    case BILLING_MODE_PER_REQUEST:
      return t('availableChannels.pricing.billingModePerRequest')
    case BILLING_MODE_IMAGE:
      return t('availableChannels.pricing.billingModeImage')
    default:
      return '-'
  }
}

function billingModeBadgeClass(mode: BillingMode): string {
  switch (mode) {
    case BILLING_MODE_TOKEN:
      return 'bg-blue-100 text-blue-700 dark:bg-blue-900/30 dark:text-blue-400'
    case BILLING_MODE_PER_REQUEST:
      return 'bg-emerald-100 text-emerald-700 dark:bg-emerald-900/30 dark:text-emerald-400'
    case BILLING_MODE_IMAGE:
      return 'bg-purple-100 text-purple-700 dark:bg-purple-900/30 dark:text-purple-400'
    default:
      return 'bg-gray-100 text-gray-600 dark:bg-dark-700 dark:text-gray-400'
  }
}

// ── 阶梯定价格式化 ────────────────────────────────────────────────
function formatRange(min: number, max: number | null): string {
  const maxLabel = max == null ? '∞' : String(max)
  return `(${min}, ${maxLabel}]`
}

function formatInterval(iv: UserPricingInterval, mode: BillingMode): string {
  if (mode === BILLING_MODE_PER_REQUEST || mode === BILLING_MODE_IMAGE) {
    return formatScaled(iv.per_request_price, 1)
  }
  const input = formatScaled(iv.input_price, perMillionScale)
  const output = formatScaled(iv.output_price, perMillionScale)
  return `${input} / ${output}`
}

// ── 阶梯定价 Popover ──────────────────────────────────────────────
// 单实例 popover：鼠标移入"阶梯"徽章时显示对应模型的区间定价明细。
// Teleport 到 body + fixed 定位，避免被表格 overflow-hidden 裁剪。
const tiersPopoverEl = ref<HTMLElement | null>(null)
const tiersPopover = ref<{
  visible: boolean
  modelName: string
  intervals: UserPricingInterval[]
  billingMode: BillingMode
  style: Record<string, string>
}>({
  visible: false,
  modelName: '',
  intervals: [],
  billingMode: BILLING_MODE_TOKEN,
  style: { top: '0px', left: '0px' },
})

function showTiersPopover(event: MouseEvent, model: UserSupportedModel) {
  if (!model.pricing?.intervals || model.pricing.intervals.length === 0) return
  tiersPopover.value = {
    visible: true,
    modelName: model.name,
    intervals: model.pricing.intervals,
    billingMode: model.pricing.billing_mode,
    style: tiersPopover.value.style,
  }
  nextTick(() => {
    updateTiersPosition(event.target as HTMLElement)
    window.addEventListener('scroll', onTiersScroll, true)
    window.addEventListener('resize', onTiersScroll)
  })
}

function updateTiersPosition(trigger: HTMLElement | null) {
  if (!trigger) return
  const rect = trigger.getBoundingClientRect()
  const margin = 8
  const popover = tiersPopoverEl.value
  const popWidth = popover?.offsetWidth ?? 320
  const popHeight = popover?.offsetHeight ?? 200
  const vw = window.innerWidth
  const vh = window.innerHeight

  let top = rect.bottom + margin
  if (top + popHeight > vh - margin) {
    top = Math.max(margin, rect.top - popHeight - margin)
  }

  let left = rect.left + rect.width / 2 - popWidth / 2
  if (left < margin) left = margin
  if (left + popWidth > vw - margin) left = vw - margin - popWidth

  tiersPopover.value.style = {
    top: `${Math.round(top)}px`,
    left: `${Math.round(left)}px`,
  }
}

function onTiersScroll() {
  if (!tiersPopover.value.visible) return
  // 滚动时隐藏 popover，避免定位错乱（简化处理）。
  hideTiersPopover()
}

function hideTiersPopover() {
  tiersPopover.value.visible = false
  window.removeEventListener('scroll', onTiersScroll, true)
  window.removeEventListener('resize', onTiersScroll)
}

onBeforeUnmount(() => {
  window.removeEventListener('scroll', onTiersScroll, true)
  window.removeEventListener('resize', onTiersScroll)
})
</script>
