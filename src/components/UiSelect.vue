<script setup>
import { ref, computed, onMounted, onBeforeUnmount, nextTick } from 'vue'

const props = defineProps({
  modelValue: { type: String, default: '' },
  options: { type: Array, default: () => [] },
  placeholder: { type: String, default: '' },
  disabled: { type: Boolean, default: false },
})
const emit = defineEmits(['update:modelValue'])

const open = ref(false)
const hoverIndex = ref(-1)
const container = ref(null)
const buttonEl = ref(null)

const valueLabel = computed(() => {
  const f = props.options.find(o => String(o.value) === String(props.modelValue))
  return f ? f.label : props.placeholder || ''
})

const toggle = () => { if (!props.disabled) open.value = !open.value }
const close = () => { open.value = false; hoverIndex.value = -1 }
const selectIndex = (i) => {
  const o = props.options[i]
  if (!o) return
  emit('update:modelValue', String(o.value))
  nextTick(() => {
    buttonEl.value && typeof buttonEl.value.blur === 'function' && buttonEl.value.blur()
    close()
  })
}
const onKey = (e) => {
  if (props.disabled) return
  if (!open.value && (e.key === 'ArrowDown' || e.key === 'ArrowUp' || e.key === 'Enter' || e.key === ' ')) { open.value = true; e.preventDefault(); return }
  if (e.key === 'Escape') { close(); return }
  if (e.key === 'ArrowDown') { hoverIndex.value = Math.min(props.options.length - 1, hoverIndex.value + 1); e.preventDefault(); }
  if (e.key === 'ArrowUp') { hoverIndex.value = Math.max(0, hoverIndex.value - 1); e.preventDefault(); }
  if (e.key === 'Enter') { if (hoverIndex.value >= 0) selectIndex(hoverIndex.value); e.preventDefault(); }
}

const onDocClick = (e) => { if (!container.value) return; if (!container.value.contains(e.target)) close() }
onMounted(() => { document.addEventListener('click', onDocClick) })
onBeforeUnmount(() => { document.removeEventListener('click', onDocClick) })
</script>

<template>
  <div ref="container" class="ui-select" :class="{ open, disabled }" tabindex="0" role="combobox" :aria-expanded="open" :aria-disabled="disabled" @keydown="onKey">
    <button ref="buttonEl" class="ui-select__button" type="button" @click="toggle" :disabled="disabled">
      <span class="ui-select__label">{{ valueLabel }}</span>
      <span class="ui-select__chev" aria-hidden="true"></span>
    </button>
    <div v-if="open" class="ui-select__menu">
      <ul role="listbox">
        <li v-for="(o,i) in options" :key="String(o.value)" role="option"
            :aria-selected="String(modelValue) === String(o.value)"
            class="ui-select__option"
            :class="{ active: i === hoverIndex, selected: String(modelValue) === String(o.value) }"
            @mouseenter="hoverIndex = i" @mouseleave="hoverIndex = -1" @click="selectIndex(i)">
          <span class="ui-select__text">{{ o.label }}</span>
          <span class="ui-select__tick" v-if="String(modelValue) === String(o.value)">✓</span>
        </li>
      </ul>
    </div>
  </div>
  
</template>

<style scoped>
.ui-select { position: relative; }
.ui-select__button { width: 100%; display: flex; align-items: center; justify-content: space-between; background: var(--input-bg); color: var(--text); border: 1px solid var(--input-border); border-radius: 12px; padding: 14px 16px; height: var(--input-h); box-shadow: 0 2px 6px rgba(0,0,0,0.06); transition: border-color .2s ease, box-shadow .2s ease, transform .2s ease; box-sizing: border-box; line-height: 1.25; }
.ui-select__label { display: inline-block; max-width: calc(100% - 28px); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.ui-select__button:hover { transform: translateY(-1px); }
.ui-select__button:focus { outline: none; border-color: var(--accent); box-shadow: 0 0 0 3px rgba(255,204,0,0.2); }
.ui-select__chev { width: 10px; height: 10px; border: 2px solid var(--muted); border-left-color: transparent; border-top-color: transparent; transform: rotate(45deg); transition: transform .22s ease, border-color .22s ease; }
.open .ui-select__chev { border-color: var(--accent); border-left-color: transparent; border-top-color: transparent; transform: rotate(225deg); }
.disabled .ui-select__button { opacity: .6; cursor: not-allowed; }

.ui-select__menu { position: absolute; left: 0; right: 0; margin-top: 8px; background: var(--card-bg); border: 1px solid var(--card-border); border-radius: 12px; box-shadow: 0 16px 36px var(--shadow-color); overflow: hidden; animation: menuShow .16s ease-out both; z-index: 20; }
.ui-select__menu ul { list-style: none; margin: 0; padding: 6px; max-height: 240px; overflow: auto; }
.ui-select__option { display: flex; align-items: center; justify-content: space-between; gap: 10px; padding: 10px 12px; border-radius: 8px; color: var(--text); cursor: pointer; transition: background-color .15s ease, transform .15s ease; }
.ui-select__option:hover { transform: translateY(-1px); background: var(--thead-bg); }
.ui-select__option.active { background: var(--thead-bg); }
.ui-select__option.selected { background: var(--thead-bg); outline: 2px solid rgba(255,204,0,0.25); }
.ui-select__tick { color: var(--accent); font-weight: 700; }

@keyframes menuShow { from { opacity: 0; transform: translateY(4px) scale(.98); } to { opacity: 1; transform: translateY(0) scale(1); } }
</style>
