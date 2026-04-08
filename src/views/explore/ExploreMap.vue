<script setup lang="ts">
import { onMounted, onUnmounted, ref, watch } from 'vue'
import { useRouter } from 'vue-router'
import TabBar from '../../components/ui/TabBar.vue'
import SocialMatchChat from '../social/SocialMatchChat.vue'
import { poiList } from '../../lib/mockData'
import { loadAMap, type AMapGlobal, type AMapMapLike, type AMapMarkerLike } from '../../lib/amapLoader'
import { MAP_CONFIG, assertMapConfig } from '../../config/mapConfig'

const router = useRouter()
const mapContainer = ref<HTMLDivElement | null>(null)
const amap = ref<AMapGlobal | null>(null)
const map = ref<AMapMapLike | null>(null)
const poiMarkers = ref<Map<string, AMapMarkerLike>>(new Map())
const mapError = ref('')
const showSearch = ref(false)
const searchQuery = ref('')
const activeMood = ref<string | null>(null)
const showFilter = ref(false)
const showMatchChat = ref(false)

const MAP_CENTER = MAP_CONFIG.DEFAULT_CENTER
const MAP_ZOOM = MAP_CONFIG.DEFAULT_ZOOM

const poiCoords: Record<string, [number, number]> = {
  'harbor-light': [121.4489, 31.2295],
  glasshouse: [121.4375, 31.1889],
  'midnight-pool': [121.491, 31.2345],
}

const moodFilters = [
  { key: 'healing', label: '治愈', color: '#6BBFA3' },
  { key: 'energy', label: '活力', color: '#E8A44A' },
  { key: 'romantic', label: '浪漫', color: '#D4788C' },
  { key: 'creative', label: '创意', color: '#9B8EC4' },
  { key: 'social', label: '社交', color: '#D49A5A' },
  { key: 'cozy', label: '温馨', color: '#B89A80' },
]

const moodColorMap: Record<string, string> = {
  healing: '#6BBFA3',
  energy: '#E8A44A',
  romantic: '#D4788C',
  creative: '#9B8EC4',
  social: '#D49A5A',
  cozy: '#B89A80',
}

const poiMoodMap: Record<string, string> = {
  'harbor-light': 'healing',
  glasshouse: 'creative',
  'midnight-pool': 'romantic',
}

const getPoiColor = (id: string) => moodColorMap[poiMoodMap[id] ?? 'healing'] ?? '#8E8E93'

const openPoi = (id: string) => {
  void router.push(`/poi/${id}`)
}

const toggleMood = (key: string) => {
  activeMood.value = activeMood.value === key ? null : key
}

const openMatchChat = () => {
  showSearch.value = false
  showFilter.value = false
  showMatchChat.value = true
}

const closeMatchChat = () => {
  showMatchChat.value = false
}

const clearPoiMarkers = () => {
  poiMarkers.value.forEach((marker) => marker.setMap(null))
  poiMarkers.value.clear()
}

const createPoiMarker = (poi: typeof poiList[number]) => {
  if (!map.value || !amap.value) return

  const coords = poiCoords[poi.id] ?? MAP_CENTER
  const color = getPoiColor(poi.id)

  const el = document.createElement('div')
  el.className = 'poi-marker'
  el.style.cssText = `
    width: 32px;
    height: 32px;
    border-radius: 50%;
    background: ${color};
    border: 3px solid white;
    box-shadow: 0 2px 8px rgba(0,0,0,0.2);
    cursor: pointer;
    transition: transform 0.2s;
  `
  el.addEventListener('mouseenter', () => { el.style.transform = 'scale(1.3)' })
  el.addEventListener('mouseleave', () => { el.style.transform = 'scale(1)' })
  el.addEventListener('click', () => openPoi(poi.id))

  const marker = new amap.value.Marker({
    position: coords,
    content: el,
    anchor: 'bottom-center',
    title: poi.name,
  })

  marker.setMap(map.value)
  poiMarkers.value.set(poi.id, marker)
}

const renderFilteredPois = () => {
  clearPoiMarkers()
  const filtered = activeMood.value
    ? poiList.filter((poi) => poiMoodMap[poi.id] === activeMood.value)
    : poiList

  filtered.forEach(createPoiMarker)
}

watch(activeMood, () => {
  renderFilteredPois()
})

onMounted(async () => {
  if (!mapContainer.value) return

  try {
    assertMapConfig()
    amap.value = await loadAMap()

    const instance = new amap.value.Map(mapContainer.value, {
      center: MAP_CENTER,
      zoom: MAP_ZOOM,
      viewMode: MAP_CONFIG.DEFAULT_VIEW_MODE,
      zooms: [3, 20],
    })

    map.value = instance

    if (instance.plugin && amap.value.ToolBar && (instance as any).addControl) {
      instance.plugin(['AMap.ToolBar'], () => {
        if (!map.value || !amap.value?.ToolBar) return
        ;(map.value as any).addControl(new amap.value.ToolBar({ position: 'RB' }))
      })
    }

    mapError.value = ''
    renderFilteredPois()
  } catch (error) {
    mapError.value = error instanceof Error ? error.message : '地图加载失败，请检查高德配置。'
  }
})

onUnmounted(() => {
  clearPoiMarkers()
  if (map.value) {
    map.value.destroy()
    map.value = null
  }
})

const locateUser = () => {
  if (!map.value) return
  if (map.value.setZoomAndCenter) {
    map.value.setZoomAndCenter(15, MAP_CENTER)
    return
  }
  map.value.setCenter(MAP_CENTER)
  map.value.setZoom(15)
}
</script>

<template>
  <main class="device-shell" style="overflow: visible;">
    <div class="relative h-[100dvh] w-full overflow-hidden">
      <!-- AMap 地图容器 -->
      <div ref="mapContainer" class="absolute inset-0 z-0" />
      <div
        v-if="mapError"
        class="absolute inset-x-4 top-[calc(env(safe-area-inset-top)+64px)] z-20 rounded-2xl bg-white/95 px-4 py-3 text-[13px] text-[#B42318] shadow-[0_2px_10px_rgba(0,0,0,0.08)]"
      >
        {{ mapError }}
      </div>

      <div class="absolute right-4 top-[calc(env(safe-area-inset-top)+16px)] z-20 flex flex-col gap-3">
        <div class="relative">
          <input
            v-if="showSearch"
            v-model="searchQuery"
            type="text"
            class="absolute right-12 top-1/2 z-30 h-9 w-56 -translate-y-1/2 rounded-[18px] bg-white px-3 text-[14px] text-[#1D1D1f] shadow-[0_1px_3px_rgba(0,0,0,0.06)] outline-none"
            style="letter-spacing: -0.224px"
            placeholder="搜索..."
          />
          <button
            type="button"
            class="relative z-20 flex h-9 w-9 items-center justify-center rounded-full bg-white shadow-[0_1px_3px_rgba(0,0,0,0.06)]"
            @click="showSearch = !showSearch; showFilter = false"
          >
            <span class="material-symbols-outlined text-[18px] text-[#6C6C6C]">
              {{ showSearch ? 'close' : 'search' }}
            </span>
          </button>
        </div>

        <div class="relative">
          <button
            type="button"
            class="relative z-20 flex h-9 w-9 items-center justify-center rounded-full bg-white shadow-[0_1px_3px_rgba(0,0,0,0.06)]"
            @click.stop="showFilter = !showFilter; showSearch = false"
          >
            <span class="material-symbols-outlined text-[18px] text-[#6C6C6C]">palette</span>
          </button>

          <div
            v-if="showFilter"
            class="absolute right-12 top-1/2 z-30 flex -translate-y-1/2 gap-2 rounded-[12px] bg-white px-3 py-2.5 shadow-[0_1px_3px_rgba(0,0,0,0.06)]"
            @click.stop
          >
            <button
              v-for="mood in moodFilters"
              :key="mood.key"
              type="button"
              class="h-5 w-5 rounded-full transition-all duration-200"
              :class="activeMood === mood.key ? 'ring-2 ring-[#1D1D1F] ring-offset-1' : 'opacity-60 hover:opacity-100'"
              :style="{ backgroundColor: mood.color }"
              :aria-label="mood.label"
              :title="mood.label"
              @click.stop="toggleMood(mood.key)"
            />
          </div>
        </div>
      </div>

      <div
        v-if="showSearch || showFilter"
        class="absolute inset-0 z-[15]"
        @click="showSearch = false; showFilter = false"
      />

      <button
        type="button"
        class="absolute left-4 top-[calc(env(safe-area-inset-top)+16px)] z-20 flex h-9 w-9 items-center justify-center rounded-full bg-white shadow-[0_1px_3px_rgba(0,0,0,0.06)]"
        @click="locateUser"
      >
        <span class="material-symbols-outlined text-[18px] text-[#6C6C6C]">my_location</span>
      </button>

      <button
        type="button"
        class="absolute right-4 bottom-[calc(72px+env(safe-area-inset-bottom)+12px)] z-20 flex h-11 items-center gap-2 rounded-full bg-[#1D1D1F] pl-4 pr-5 shadow-[0_2px_8px_rgba(0,0,0,0.12)] transition-transform active:scale-95"
        @click="openMatchChat"
      >
        <span class="material-symbols-outlined text-[18px] text-white">forum</span>
        <span class="text-[13px] font-medium text-white" style="letter-spacing: -0.224px">开始社交</span>
      </button>

      <SocialMatchChat :visible="showMatchChat" @close="closeMatchChat" />

      <TabBar />
    </div>
  </main>
</template>

<style scoped>
:deep(.device-shell) {
  overflow: visible;
}
</style>
