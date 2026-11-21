<template>
  <div class="grid gap-3 lg:grid-cols-[16rem_minmax(0,1fr)]">
    <aside class="self-start rounded-lg border border-gray-200 bg-white p-2 dark:border-dark-600 dark:bg-dark-900">
      <div class="mb-2 flex items-center justify-between px-1">
        <span class="text-xs font-medium text-gray-500 dark:text-gray-400">
          {{ t('admin.channels.form.models') }}
        </span>
        <div class="flex items-center gap-1">
          <span class="text-[11px] text-gray-400">{{ modelRows.length }}</span>
          <button
            v-if="activeModel"
            type="button"
            class="rounded p-1 text-gray-400 hover:bg-red-50 hover:text-red-500 dark:hover:bg-red-900/20"
            :aria-label="t('admin.channels.form.removeModel', { model: activeModel })"
            :title="t('admin.channels.form.removeModel', { model: activeModel })"
            @click="removeActiveModel"
          >
            <Icon name="trash" size="sm" />
          </button>
        </div>
      </div>
      <div v-if="modelRows.length" class="max-h-72 space-y-1 overflow-auto">
        <button
          v-for="model in modelRows"
          :key="model"
          type="button"
          class="flex w-full items-center justify-between gap-2 rounded-md px-2 py-2 text-left text-xs transition-colors"
          :class="activeModel === model
            ? 'bg-primary-50 text-primary-700 dark:bg-primary-900/20 dark:text-primary-300'
            : 'text-gray-600 hover:bg-gray-50 dark:text-gray-300 dark:hover:bg-dark-800'"
          @click="selectModel(model)"
        >
          <span class="min-w-0 break-all">{{ model }}</span>
          <span
            class="shrink-0 rounded px-1.5 py-0.5 text-[10px]"
            :class="hasPricing(model)
              ? 'bg-emerald-50 text-emerald-700 dark:bg-emerald-900/20 dark:text-emerald-300'
              : 'bg-gray-100 text-gray-500 dark:bg-dark-700 dark:text-gray-400'"
          >
            {{ hasPricing(model) ? t('admin.channels.form.priced', '已定价') : t('admin.channels.form.unpriced', '未定价') }}
          </span>
        </button>
      </div>
      <div v-else class="rounded-md border border-dashed border-gray-200 px-2 py-4 text-center text-xs text-gray-400 dark:border-dark-600">
        {{ t('admin.channels.form.noModelCandidates', '分组账号未保存白名单模型') }}
      </div>

      <button
        type="button"
        class="mt-2 w-full rounded-md border border-dashed border-gray-300 px-2 py-2 text-xs text-gray-500 hover:border-primary-400 hover:text-primary-600 dark:border-dark-600 dark:text-gray-400"
        @click="addCustomPricing"
      >
        + {{ t('admin.channels.form.addCustomModel', '添加自定义模型') }}
      </button>
      <p v-if="props.modelCandidates.length" class="mt-2 px-1 text-[11px] text-gray-400">
        {{ t('admin.channels.form.savedAccountAllowlistHint', '模型来自所选分组账号已保存的白名单。') }}
      </p>
    </aside>

    <div v-if="activeEntry" class="min-w-0">
      <div v-if="activeModel" class="mb-2 flex items-center justify-between gap-2">
        <div class="min-w-0 break-all text-sm font-medium text-gray-700 dark:text-gray-200">
          {{ activeModel }}
        </div>
        <button
          type="button"
          class="shrink-0 rounded-md border border-red-200 p-2 text-red-500 hover:bg-red-50 dark:border-red-900/50 dark:hover:bg-red-900/20"
          :aria-label="t('admin.channels.form.removeModel', { model: activeModel })"
          :title="t('admin.channels.form.removeModel', { model: activeModel })"
          @click="removeActiveModel"
        >
          <Icon name="trash" size="sm" />
        </button>
      </div>
      <PricingEntryCard
        :entry="activeEntry"
        :platform="props.platform"
        :model-candidates="props.modelCandidates"
        :hide-model-selector="!!activeModel"
        hide-remove
        :hide-token-intervals="props.hideTokenIntervals"
        :enable-time-pricing="props.enableTimePricing"
        :enable-tier-multipliers="props.enableTierMultipliers"
        @update="updateActiveEntry"
      />
    </div>
    <div v-else class="flex min-h-40 items-center justify-center rounded-lg border border-dashed border-gray-200 text-sm text-gray-400 dark:border-dark-600">
      {{ t('admin.channels.form.selectModelToPrice', '选择左侧模型以设置价格') }}
    </div>
  </div>
</template>

<script setup lang="ts">
import { computed, ref, watch } from 'vue'
import { useI18n } from 'vue-i18n'
import Icon from '@/components/icons/Icon.vue'
import PricingEntryCard from './PricingEntryCard.vue'
import { createDefaultTimePricingForm, type PricingFormEntry } from './types'

const { t } = useI18n()

const props = withDefaults(defineProps<{
  entries: PricingFormEntry[]
  modelCandidates: string[]
  excludedModels?: string[]
  platform?: string
  hideTokenIntervals?: boolean
  enableTimePricing?: boolean
  enableTierMultipliers?: boolean
}>(), {
  hideTokenIntervals: false,
  enableTimePricing: false,
  enableTierMultipliers: false,
})

const emit = defineEmits<{
  'update:entries': [entries: PricingFormEntry[]]
  'update:excludedModels': [models: string[]]
}>()

const initialModels = props.entries.flatMap(entry => entry.models)
const activeModel = ref(initialModels[0] || props.modelCandidates[0] || '')
const customMode = ref(!activeModel.value && props.entries.some(entry => entry.models.length === 0))
const excludedModels = computed(() => new Set(props.excludedModels || []))

const modelRows = computed(() => [...new Set([
  ...props.modelCandidates,
  ...props.entries.flatMap(entry => entry.models),
])].filter(model => !excludedModels.value.has(model)))

function createEntry(models: string[]): PricingFormEntry {
  return {
    models,
    billing_mode: 'token',
    input_price: null,
    output_price: null,
    cache_write_price: null,
    cache_write_1h_price: null,
    cache_write_12h_price: null,
    cache_read_price: null,
    fast_multiplier: null,
    flex_multiplier: null,
    ultrafast_multiplier: null,
    reasoning_effort_multipliers: null,
    image_input_price: null,
    image_output_price: null,
    per_request_price: null,
    intervals: [],
    time_pricing: createDefaultTimePricingForm(),
  }
}

function hasPricing(model: string) {
  return props.entries.some(entry => entry.models.includes(model))
}

const activeEntry = computed<PricingFormEntry | null>(() => {
  if (activeModel.value) {
    const entry = props.entries.find(item => item.models.includes(activeModel.value))
    return entry ? { ...entry, models: [activeModel.value] } : createEntry([activeModel.value])
  }
  if (customMode.value) {
    return props.entries.find(item => item.models.length === 0) || createEntry([])
  }
  return null
})

watch(() => props.modelCandidates, candidates => {
  if (!activeModel.value && candidates.length) activeModel.value = candidates[0]
}, { deep: true })

function selectModel(model: string) {
  activeModel.value = model
  customMode.value = false
}

function addCustomPricing() {
  activeModel.value = ''
  customMode.value = true
}

function splitEntriesByModel(entries: PricingFormEntry[]) {
  return entries.flatMap(entry => entry.models.length
    ? entry.models.map(model => ({ ...entry, models: [model] }))
    : [{ ...entry }])
}

function updateActiveEntry(updated: PricingFormEntry) {
  if (customMode.value) {
    const modelNames = [...new Set(updated.models.map(model => model.trim()).filter(Boolean))]
    if (modelNames.length > 0) {
      emit('update:excludedModels', (props.excludedModels || []).filter(model => !modelNames.includes(model)))
    }
    const remaining = props.entries.filter(entry => entry.models.length > 0)
    const additions = modelNames.length
      ? modelNames.map(model => ({ ...updated, models: [model] }))
      : [{ ...updated, models: [] }]
    emit('update:entries', [...remaining, ...additions])
    if (modelNames.length) {
      activeModel.value = modelNames[0]
      customMode.value = false
    }
    return
  }

  const model = activeModel.value
  if (!model) return
  const entries = splitEntriesByModel(props.entries)
  const index = entries.findIndex(entry => entry.models[0] === model)
  const normalized = { ...updated, models: [model] }
  if (index >= 0) entries.splice(index, 1, normalized)
  else entries.push(normalized)
  emit('update:entries', entries)
}

function removeActiveModel() {
  if (customMode.value && !activeModel.value) {
    emit('update:entries', props.entries.filter(entry => entry.models.length > 0))
    customMode.value = false
    return
  }
  const model = activeModel.value
  if (!model) return
  emit('update:excludedModels', [...new Set([...excludedModels.value, model])])
  emit('update:entries', splitEntriesByModel(props.entries).filter(entry => entry.models[0] !== model))
  const nextModel = modelRows.value.find(candidate => candidate !== model) || ''
  activeModel.value = nextModel
  customMode.value = false
}
</script>
