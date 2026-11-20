<template>
  <div>
    <!-- Tags display -->
    <div class="flex flex-wrap gap-1.5 rounded-lg border border-gray-200 bg-white p-2 dark:border-dark-600 dark:bg-dark-800 min-h-[2.5rem]">
      <span
        v-for="(model, idx) in models"
        :key="idx"
        class="inline-flex items-center gap-1 rounded-md px-2 py-0.5 text-sm"
        :class="getPlatformTagClass(props.platform || '')"
      >
        {{ model }}
        <button
          type="button"
          @click="removeModel(idx)"
          class="ml-0.5 rounded-full p-0.5 hover:bg-primary-200 dark:hover:bg-primary-800"
        >
          <Icon name="x" size="xs" />
        </button>
      </span>
      <input
        ref="inputRef"
        v-model="inputValue"
        type="text"
        class="flex-1 min-w-[120px] border-none bg-transparent text-sm outline-none placeholder:text-gray-400 dark:text-white"
        :placeholder="models.length === 0 ? placeholder : ''"
        @keydown.enter="handleEnter"
        @keydown.tab="handleTab"
        @keydown.backspace="handleBackspace"
        @paste="handlePaste"
      />
      <div v-if="suggestions?.length" class="order-last w-full rounded-md border border-gray-200 bg-white p-2 dark:border-dark-600 dark:bg-dark-800">
        <div class="mb-1 text-[11px] text-gray-500 dark:text-gray-400">
          {{ t('admin.channels.form.groupModelAllowlist', 'Group allowlist models') }}
          <span>({{ filteredSuggestions.length }}/{{ suggestions.length }})</span>
        </div>
        <div class="max-h-40 overflow-auto">
        <label
          v-for="model in filteredSuggestions"
          :key="model"
          class="flex cursor-pointer items-center gap-2 rounded px-1.5 py-1 text-xs hover:bg-gray-100 dark:hover:bg-dark-700"
        >
          <input
            type="checkbox"
            :checked="models.includes(model)"
            class="h-3.5 w-3.5 rounded border-gray-300 text-primary-600 focus:ring-primary-500"
            @change="toggleSuggestion(model, ($event.target as HTMLInputElement).checked)"
          />
          <span>{{ model }}</span>
        </label>
        <div v-if="filteredSuggestions.length === 0" class="px-1.5 py-2 text-xs text-gray-400">
          {{ t('common.noResults', 'No matching models') }}
        </div>
        </div>
      </div>
    </div>
    <p class="mt-1 text-xs text-gray-400">
      {{ props.suggestions?.length ? t('admin.channels.form.modelSuggestionHint', 'Check group models to add them to this pricing entry, or type a custom model.') : t('admin.channels.form.modelInputHint', 'Press Enter to add, supports paste for batch import.') }}
    </p>
  </div>
</template>

<script setup lang="ts">
import { computed, ref } from 'vue'
import { useI18n } from 'vue-i18n'
import Icon from '@/components/icons/Icon.vue'
import { getPlatformTagClass } from './types'

const { t } = useI18n()

const props = defineProps<{
  models: string[]
  placeholder?: string
  platform?: string
  suggestions?: string[]
}>()

const emit = defineEmits<{
  'update:models': [models: string[]]
}>()

const inputValue = ref('')
const inputRef = ref<HTMLInputElement>()
const filteredSuggestions = computed(() => {
  const query = inputValue.value.trim().toLowerCase()
  return (props.suggestions || []).filter(model => !query || model.toLowerCase().includes(query))
})

function toggleSuggestion(model: string, checked: boolean) {
  const models = checked
    ? [...props.models, model]
    : props.models.filter(item => item !== model)
  emit('update:models', [...new Set(models)])
}

function addModel() {
  const val = inputValue.value.trim()
  if (!val) return
  if (!props.models.includes(val)) {
    emit('update:models', [...props.models, val])
  }
  inputValue.value = ''
}

function handleEnter(event: KeyboardEvent) {
  if (event.isComposing) return
  event.preventDefault()
  addModel()
}

function handleTab(event: KeyboardEvent) {
  if (event.isComposing || !inputValue.value.trim()) return
  event.preventDefault()
  addModel()
}

function removeModel(idx: number) {
  const newModels = [...props.models]
  newModels.splice(idx, 1)
  emit('update:models', newModels)
}

function handleBackspace(event: KeyboardEvent) {
  if (event.isComposing) return
  if (inputValue.value === '' && props.models.length > 0) {
    removeModel(props.models.length - 1)
  }
}

function handlePaste(e: ClipboardEvent) {
  e.preventDefault()
  const text = e.clipboardData?.getData('text') || ''
  const items = text.split(/[,\n;]+/).map(s => s.trim()).filter(Boolean)
  if (items.length === 0) return
  const unique = [...new Set([...props.models, ...items])]
  emit('update:models', unique)
  inputValue.value = ''
}
</script>
