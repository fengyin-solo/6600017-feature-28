<template>
  <div class="flex h-screen">
    <!-- Sidebar -->
    <div class="w-72 bg-gray-900 p-4 flex flex-col gap-4 overflow-y-auto">
      <h1 class="text-xl font-bold text-blue-400">天文星图渲染器</h1>

      <!-- Search -->
      <div>
        <input v-model="store.searchQuery" placeholder="搜索天体..." class="w-full bg-gray-800 rounded px-3 py-2 text-sm" />
        <div v-if="store.filteredStars.length" class="mt-1">
          <div v-for="s in store.filteredStars" :key="s.name"
            @click="store.selectedStar = s"
            class="bg-gray-800 p-2 rounded mt-1 cursor-pointer hover:bg-gray-700 text-sm">
            {{ s.name }} <span class="text-gray-400">mag {{ s.mag }}</span>
          </div>
        </div>
      </div>

      <!-- Time Travel -->
      <div>
        <label class="text-gray-400 text-xs">时间旅行</label>
        <input type="datetime-local" v-model="dateStr" @input="updateDate"
          class="w-full bg-gray-800 rounded px-3 py-2 text-sm" />
      </div>

      <!-- Location Presets -->
      <div>
        <label class="text-gray-400 text-xs">
          纬度: {{ store.latitude.toFixed(1) }}°
          <span v-if="presetLabel" class="text-blue-400 ml-1">· {{ presetLabel }}</span>
        </label>
        <div class="mt-1 mb-1">
          <div class="text-gray-500 text-xs mb-0.5">北半球</div>
          <div class="flex flex-wrap gap-1">
            <button v-for="loc in northernLocations" :key="loc.name"
              @click="applyPreset(loc)"
              :class="[
                'px-2 py-0.5 rounded text-xs transition-colors',
                isPresetActive(loc)
                  ? 'bg-blue-600 text-white'
                  : 'bg-gray-800 text-gray-300 hover:bg-gray-700'
              ]">
              {{ loc.name }}
            </button>
          </div>
          <div class="text-gray-500 text-xs mb-0.5 mt-2">南半球</div>
          <div class="flex flex-wrap gap-1">
            <button v-for="loc in southernLocations" :key="loc.name"
              @click="applyPreset(loc)"
              :class="[
                'px-2 py-0.5 rounded text-xs transition-colors',
                isPresetActive(loc)
                  ? 'bg-blue-600 text-white'
                  : 'bg-gray-800 text-gray-300 hover:bg-gray-700'
              ]">
              {{ loc.name }}
            </button>
          </div>
        </div>
        <input type="range" :value="store.latitude" @input="onLatitudeInput" min="-90" max="90" step="0.1" class="w-full" />
      </div>

      <!-- Zoom -->
      <div>
        <label class="text-gray-400 text-xs">缩放: {{ store.zoom.toFixed(1) }}x</label>
        <input type="range" v-model.number="store.zoom" min="0.3" max="3" step="0.1" class="w-full" />
      </div>

      <!-- Toggles -->
      <div class="flex flex-col gap-2">
        <label class="flex items-center gap-2 text-sm">
          <input type="checkbox" v-model="store.showLabels" /> 星名标签
        </label>
        <label class="flex items-center gap-2 text-sm">
          <input type="checkbox" v-model="store.showConstLines" /> 星座连线
        </label>
        <label class="flex items-center gap-2 text-sm">
          <input type="checkbox" v-model="store.showGrid" /> 坐标网格
        </label>
      </div>

      <!-- Star Info -->
      <div v-if="store.selectedStar" class="bg-gray-800 rounded-xl p-3">
        <h3 class="text-amber-400 font-bold">{{ store.selectedStar.name }}</h3>
        <div class="text-xs text-gray-300 mt-2 space-y-1">
          <p>赤经: {{ store.selectedStar.ra.toFixed(2) }}h</p>
          <p>赤纬: {{ store.selectedStar.dec.toFixed(2) }}°</p>
          <p>视星等: {{ store.selectedStar.mag }}</p>
          <p>光谱型: {{ store.selectedStar.spectral }}</p>
        </div>
      </div>

      <!-- Constellation list -->
      <div class="text-xs">
        <h4 class="text-gray-400 mb-1">可见星座</h4>
        <div v-for="c in store.CONSTELLATIONS" :key="c.name" class="py-1 text-gray-300">
          {{ c.nameCn }} <span class="text-gray-500">({{ c.name }})</span>
        </div>
      </div>

      <div class="text-xs text-gray-500 mt-auto">
        LST: {{ store.localSiderealTime.toFixed(2) }}h
      </div>
    </div>

    <!-- Sky Canvas -->
    <div class="flex-1 relative">
      <StarCanvas />
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed } from 'vue'
import { useSkyStore } from './store/sky'
import StarCanvas from './components/StarCanvas.vue'

const store = useSkyStore()
const dateStr = ref(new Date().toISOString().slice(0, 16))
function updateDate() { store.viewDate = new Date(dateStr.value) }

interface LocationPreset {
  name: string
  latitude: number
}

const northernLocations: LocationPreset[] = [
  { name: '北京', latitude: 39.9 },
  { name: '东京', latitude: 35.7 },
  { name: '纽约', latitude: 40.7 },
  { name: '伦敦', latitude: 51.5 },
  { name: '莫斯科', latitude: 55.8 },
  { name: '开罗', latitude: 30.0 },
  { name: '赤道', latitude: 0.0 },
]

const southernLocations: LocationPreset[] = [
  { name: '悉尼', latitude: -33.9 },
  { name: '墨尔本', latitude: -37.8 },
  { name: '惠灵顿', latitude: -41.3 },
  { name: '圣保罗', latitude: -23.5 },
  { name: '布宜诺斯艾利斯', latitude: -34.6 },
  { name: '开普敦', latitude: -33.9 },
]

const allPresets = computed(() => [...northernLocations, ...southernLocations])

const selectedPresetName = ref('北京')

const presetLabel = computed(() => {
  if (!selectedPresetName.value) return ''
  const preset = allPresets.value.find(p => p.name === selectedPresetName.value)
  if (!preset) return ''
  if (Math.abs(store.latitude - preset.latitude) < 0.05) return preset.name
  return `${preset.name} ±${Math.abs(store.latitude - preset.latitude).toFixed(1)}°`
})

function isPresetActive(preset: LocationPreset): boolean {
  if (selectedPresetName.value !== preset.name) return false
  return Math.abs(store.latitude - preset.latitude) <= 5
}

function applyPreset(preset: LocationPreset) {
  store.latitude = preset.latitude
  selectedPresetName.value = preset.name
}

function onLatitudeInput(e: Event) {
  const val = parseFloat((e.target as HTMLInputElement).value)
  if (!isNaN(val)) {
    store.latitude = Math.max(-90, Math.min(90, val))
  }
  if (selectedPresetName.value) {
    const preset = allPresets.value.find(p => p.name === selectedPresetName.value)
    if (preset && Math.abs(store.latitude - preset.latitude) > 5) {
      selectedPresetName.value = ''
    }
  }
}
</script>
