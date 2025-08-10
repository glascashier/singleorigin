<template>


<div
      ref="container"
      class="relative min-h-screen touch-pan-y select-none"
      @touchstart.passive="onTouchStart"
      @touchmove.prevent="onTouchMove"
      @touchend="onTouchEnd"
      @touchcancel="onTouchEnd"
    >
 
   <!-- 상단 고정 프리뷰 영역 -->
<div v-show="showPreview" class="fixed left-0 right-0 top-0 z-50 bg-white border-b">
  <div class="w-full max-w-screen-sm mx-auto p-2">
    <div class="relative w-full h-[40vh] overflow-hidden rounded">
      <!-- 이미지 -->
      <NuxtImg
        v-if="imgReady && currentSrc"
        :key="currentSrc"
        :src="currentSrc"
        class="absolute inset-0 w-full h-full object-cover"
        :modifiers="{ width: 1600, height: undefined, quality: 75, format: 'webp' }"
        sizes="(max-width: 768px) 100vw, 600px"
        placeholder
        preload
        fetchpriority="high"
        decoding="async"
      />
      <!-- 로딩 플레이스홀더 -->
      <div v-else class="absolute inset-0 grid place-items-center bg-neutral-100">
        <div class="animate-pulse text-sm text-neutral-500">Loading…</div>
      </div>
    </div>

    <div class="mt-2 text-sm">
      <div class="font-medium">{{ activePreview?.title }}</div>
      <div class="opacity-70">{{ activePreview?.subtitle }}</div>
    </div>
  </div>
</div>


  
      <!-- 포트폴리오 리스트 (table 사용) -->
      <div class="pt-[calc(40vh+12px)]"> <!-- 프리뷰 높이만큼 아래로 밀어줌 -->
        <table class="w-full table-fixed">
          <tbody>
            <tr
              v-for="(item, i) in items"
              :key="item.id"
              :ref="el => (rowRefs[i] = el as HTMLTableRowElement)"
              class="border-b"
            >
              <td class="p-4 text-base">
                <div class="font-medium">{{ item.title }}</div>
                <div class="text-sm opacity-70">{{ item.subtitle }}</div>
              </td>
            </tr>
          </tbody>
        </table>
      </div>
  
      <!-- 터치 바 -->
      <div
        v-show="isActive"
        class="fixed left-0 right-0 z-40 pointer-events-none"
        :style="{ top: `${barY - 1}px` }"
      >
        <div class="h-[2px] bg-black/80"></div>
      </div>
    </div>
  </template>
  
  <script setup lang="ts">
  import { onMounted, onBeforeUnmount, reactive, ref, computed, watch } from 'vue'
  import { useImage } from '#image' // ✅ Nuxt Image helper
  
  type Item = {
    id: string
    title: string
    subtitle?: string
    image: string // ex) '/images/p1.jpg' (public/ 아래)
  }
  
  const items = reactive<Item[]>([
    { id: 'p1',  title: 'Project 1', subtitle: '2024', image: '/images/p1.jpg' },
    { id: 'p2',  title: 'Project 2', subtitle: '2023', image: '/images/p2.jpg' },
    { id: 'p3',  title: 'Project 3', subtitle: '2022', image: '/images/p3.jpg' },
    { id: 'p4',  title: 'Project 4', subtitle: '2022', image: '/images/p4.jpg' },
    { id: 'p5',  title: 'Project 5', subtitle: '2021', image: '/images/p5.jpg' },
    { id: 'p6',  title: 'Project 6', subtitle: '2021', image: '/images/p6.jpg' },
    { id: 'p7',  title: 'Project 7', subtitle: '2020', image: '/images/p7.jpg' },
    { id: 'p8',  title: 'Project 8', subtitle: '2020', image: '/images/p8.jpg' },
    { id: 'p9',  title: 'Project 9', subtitle: '2019', image: '/images/p9.jpg' },
    { id: 'p10', title: 'Project 10', subtitle: '2019', image: '/images/p10.jpg' },
  ])
  
  const container = ref<HTMLElement | null>(null)
  const rowRefs = reactive<HTMLTableRowElement[]>([])
  
  const isTouching = ref(false)
  const isActive = ref(false)
  const showPreview = ref(false)
  const barY = ref(0)
  let longPressTimer: number | null = null
  const LONG_PRESS_MS = 300
  
  const activeIndex = ref<number | null>(null)
  const activePreview = computed(() =>
    activeIndex.value != null ? items[activeIndex.value] : null
  )
  
  const imgReady = ref(false)
  const currentSrc = ref<string | null>(null)
  let loadToken = 0
  let switchTimer: number | null = null
  const SWITCH_DELAY = 80
  
  // Nuxt Image URL 생성기
  const { $img } = useNuxtApp()
  const modifiers = { width: 1600, quality: 75, format: 'webp' } as const
function resolveImageUrl(src: string) {
  return $img(src, modifiers)
}
  
  // Nuxt Image URL로 프리로드
  function preloadImage(src: string): Promise<void> {
    return new Promise((resolve, reject) => {
      if (import.meta.server) return resolve()
      const u = resolveImageUrl(src)
      const i = new Image()
      i.onload = () => resolve()
      i.onerror = reject
      i.src = u
    })
  }
  
  // 인접 항목 선프리로드(±2)
  async function preloadAround(center: number, radius = 2) {
    if (import.meta.server) return
    const tasks: Promise<void>[] = []
    for (let d = -radius; d <= radius; d++) {
      if (d === 0) continue
      const idx = center + d
      if (idx < 0 || idx >= items.length) continue
      tasks.push(preloadImage(items[idx].image).catch(() => {}))
    }
    await Promise.allSettled(tasks)
  }
  
  watch(
    () => activeIndex.value,
    (idx) => {
      if (idx == null) return
      const next = items[idx]
      if (!next) return
      if (import.meta.server) return
  
      if (switchTimer) {
        clearTimeout(switchTimer)
        switchTimer = null
      }
  
      switchTimer = window.setTimeout(async () => {
        const token = ++loadToken
        imgReady.value = false
  
        try {
          await preloadImage(next.image)
          if (token !== loadToken) return
          currentSrc.value = resolveImageUrl(next.image) // ✅ 최종 URL 세팅
          imgReady.value = true
  
          // 인접 항목 선프리로드 (백그라운드)
          preloadAround(idx).catch(() => {})
        } catch {
          if (token !== loadToken) return
          imgReady.value = false
        }
      }, SWITCH_DELAY)
    },
    { immediate: false }
  )
  
  // Y에 닿는 행 찾기
  function findRowIndexAtY(clientY: number): number | null {
    for (let i = 0; i < rowRefs.length; i++) {
      const row = rowRefs[i]
      if (!row) continue
      const rect = row.getBoundingClientRect()
      if (clientY >= rect.top && clientY <= rect.bottom) {
        return i
      }
    }
    return null
  }
  
  function startLongPress(y: number) {
    isActive.value = true
    showPreview.value = true
    barY.value = y
    const idx = findRowIndexAtY(y)
    activeIndex.value = idx
  }
  
  function clearLongPressTimer() {
    if (longPressTimer) {
      window.clearTimeout(longPressTimer)
      longPressTimer = null
    }
  }
  
  function onTouchStart(e: TouchEvent) {
    if (!e.touches.length) return
    isTouching.value = true
    const y = e.touches[0].clientY
    clearLongPressTimer()
    longPressTimer = window.setTimeout(() => {
      try { e.preventDefault() } catch {}
      startLongPress(y)
    }, LONG_PRESS_MS)
  }
  
  function onTouchMove(e: TouchEvent) {
    if (!e.touches.length) return
    const y = e.touches[0].clientY
    if (!isActive.value && isTouching.value) return
    if (!isActive.value) return
    barY.value = y
    const idx = findRowIndexAtY(y)
    if (idx !== activeIndex.value) activeIndex.value = idx
  }
  
  function onTouchEnd() {
    isTouching.value = false
    clearLongPressTimer()
    if (isActive.value) {
      isActive.value = false
      showPreview.value = false
      activeIndex.value = null
      currentSrc.value = null
      imgReady.value = false
      loadToken++
    }
  }
  
  onMounted(() => {
    window.addEventListener('scroll', onTouchEnd, { passive: true })
  })
  onBeforeUnmount(() => {
    window.removeEventListener('scroll', onTouchEnd)
    clearLongPressTimer()
  })
  </script>
  
  
  <style scoped>
  /* 필요시 커스텀 스타일 추가 */
  </style>
  