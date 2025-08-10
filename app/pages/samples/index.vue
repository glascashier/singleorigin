<template>
    <div
      ref="container"
      :class="['relative select-none', { 'touch-none': isBlockingScroll }]"
      @touchstart="onTouchStart"
      @touchmove="onTouchMove"
      @touchend="onTouchEnd"
      @touchcancel="onTouchEnd"
    >
      <!-- 상단 고정 프리뷰 영역 -->
      <div v-show="showPreview" class="fixed left-0 right-0 bottom-0 z-50 bg-transparent border-b">
        <div class="w-full max-w-screen-sm mx-auto p-2">
          <div class="relative w-full h-[40vh] overflow-hidden rounded">
            <NuxtImg
              v-if="displaySrc"
              :key="displaySrc"
              :src="displaySrc"
              class="absolute inset-0 w-full h-full object-cover"
              :modifiers="{ width: 1600, quality: 75, format: 'webp' }"
              sizes="(max-width: 768px) 100vw, 600px"
              placeholder
              preload
              fetchpriority="high"
              decoding="async"
            />
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
  
      <!-- 포트폴리오 리스트 -->
      <div class="pt-3">
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
  import { ref, reactive, computed, watch } from 'vue'
  import { onMounted } from 'vue'

  type Item = {
    id: string
    title: string
    subtitle?: string
    image: string // '/images/p1.jpg' (public/ 아래)
  }
  
  const items = reactive<Item[]>([
    { id: 'p1',  title: 'Project 1',  subtitle: '2024', image: '/images/p1.jpg' },
    { id: 'p2',  title: 'Project 2',  subtitle: '2023', image: '/images/p2.jpg' },
    { id: 'p3',  title: 'Project 3',  subtitle: '2022', image: '/images/p3.jpg' },
    { id: 'p4',  title: 'Project 4',  subtitle: '2022', image: '/images/p4.jpg' },
    { id: 'p5',  title: 'Project 5',  subtitle: '2021', image: '/images/p5.jpg' },
    { id: 'p6',  title: 'Project 6',  subtitle: '2021', image: '/images/p6.jpg' },
    { id: 'p7',  title: 'Project 7',  subtitle: '2020', image: '/images/p7.jpg' },
    { id: 'p8',  title: 'Project 8',  subtitle: '2020', image: '/images/p8.jpg' },
    { id: 'p9',  title: 'Project 9',  subtitle: '2019', image: '/images/p9.jpg' },
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
  
  /** 스크롤 차단 상태 (롱프레스 안정) */
  const isBlockingScroll = ref(false)
  let startY = 0
  const MOVE_CANCEL_PX = 12 // 롱프레스 전 허용 이동치
  
  /** 프리뷰 이미지 표시용(현재 화면에 보이는 src) */
  const displaySrc = ref<string | null>(null)
  
  /** 프리로드/레이스 가드 */
  let inFlightFor: number | null = null
  let loadToken = 0
  let switchTimer: number | null = null
  const SWITCH_DELAY = 100
  
  // Nuxt Image 최적화 URL 생성기 ($img)
   /** Nuxt Image URL로 프리로드(캐시 워밍) */
   // --- 아래 유틸을 스크립트 하단 아무 곳에 추가하세요 ---

/** 원본 경로 프리로드(브라우저 캐시 워밍) */
function preload(src: string): Promise<void> {
  return new Promise((resolve, reject) => {
    if (import.meta.server) return resolve()
    const img = new Image()
    img.onload = () => resolve()
    img.onerror = reject
    img.src = src
  })
}

/** 네트워크 상태 체크 (느리거나 데이터세이버면 프리로드 생략) */
function shouldSkipPreload(): boolean {
  if (import.meta.server) return true
  // @ts-ignore
  const conn = navigator.connection || navigator.mozConnection || navigator.webkitConnection
  if (!conn) return false
  // 데이터 세이버 on, 또는 매우 느린 네트워크면 스킵
  // @ts-ignore
  if (conn.saveData) return true
  // @ts-ignore
  if (typeof conn.effectiveType === 'string' && /(^2g|^slow-2g)/.test(conn.effectiveType)) return true
  return false
}

/** requestIdleCallback 폴리필 */
const ric: (cb: () => void) => void =
  typeof window !== 'undefined' && 'requestIdleCallback' in window
    // @ts-ignore
    ? window.requestIdleCallback
    : (cb: () => void) => setTimeout(cb, 0)

/** 페이지 진입시 전체 워밍 (동시 3개) */
function warmupAllImages(concurrency = 3) {
  if (import.meta.server) return
  if (shouldSkipPreload()) return

  ric(() => {
    const queue = items.map(x => x.image)
    let running = 0

    const kick = () => {
      if (!queue.length || running >= concurrency) return
      const src = queue.shift()!
      running++
      preload(src).finally(() => {
        running--
        if (queue.length) kick()
      })
      // 채널 채우기
      if (running < concurrency) kick()
    }

    // 초기 시동
    for (let i = 0; i < concurrency; i++) kick()
  })
}

  
  /** 인접 항목 선프리로드(±2) */
  async function preloadAround(center: number, radius = 2) {
    if (import.meta.server) return
    const tasks: Promise<void>[] = []
    for (let d = -radius; d <= radius; d++) {
      if (d === 0) continue
      const idx = center + d
      if (idx < 0 || idx >= items.length) continue
      tasks.push(preload(items[idx].image).catch(() => {}))
    }
    await Promise.allSettled(tasks)
  }
  
  /** 바 위치와 맞닿는 행 찾기 */
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
  
  /** activeIndex 변경 → 프리로드 후 표시 교체(이전 이미지는 유지) */
  watch(
    () => activeIndex.value,
    (idx) => {
      if (idx == null || import.meta.server) return
  
      // 빠른 이동 스로틀
      if (switchTimer) { clearTimeout(switchTimer); switchTimer = null }
  
      switchTimer = window.setTimeout(async () => {
        // 중복 프리로드 방지
        if (inFlightFor === idx) return
        inFlightFor = idx
  
        const token = ++loadToken
        const next = items[idx]
        try {
          await preload(next.image)
          if (token !== loadToken) return
          // 로드 완료되면 그때 화면 교체 (스피너에 갇히지 않음)
          displaySrc.value = next.image
          // 주변도 백그라운드 프리로드
          preloadAround(idx).catch(() => {})
        } finally {
          if (inFlightFor === idx) inFlightFor = null
        }
      }, SWITCH_DELAY)
    },
    { immediate: false }
  )
  
  /** 롱프레스 시작 */
  function startLongPress(y: number) {
    isActive.value = true
    showPreview.value = true
    barY.value = y
    const idx = findRowIndexAtY(y)
    activeIndex.value = idx
  
    // 첫 진입에서 displaySrc가 없다면 즉시 한 번 프리로드 후 표시
    if (displaySrc.value == null && idx != null) {
      const first = items[idx]
      preload(first.image).then(() => { displaySrc.value = first.image })
    }
  }
  
  /** 터치 이벤트들 */
  function clearLongPressTimer() {
    if (longPressTimer) {
      window.clearTimeout(longPressTimer)
      longPressTimer = null
    }
  }
  
  function onTouchStart(e: TouchEvent) {
    if (!e.touches.length) return
    isTouching.value = true
    startY = e.touches[0].clientY
  
    // 터치 시작 즉시 스크롤 차단(세션 보호)
    isBlockingScroll.value = true
  
    clearLongPressTimer()
    longPressTimer = window.setTimeout(() => {
      try { e.preventDefault() } catch {}
      startLongPress(startY)
    }, LONG_PRESS_MS)
  }
  
  function onTouchMove(e: TouchEvent) {
    if (!e.touches.length) return
    const y = e.touches[0].clientY
  
    // 롱프레스 전: 크게 움직이면 롱프레스 취소 + 스크롤 허용
    if (!isActive.value) {
      if (Math.abs(y - startY) > MOVE_CANCEL_PX) {
        clearLongPressTimer()
        isBlockingScroll.value = false
      }
      return
    }
  
    // 롱프레스 활성: 바 이동
    barY.value = y
    const idx = findRowIndexAtY(y)
    if (idx !== activeIndex.value) activeIndex.value = idx
  }
  
  function onTouchEnd() {
    isTouching.value = false
    clearLongPressTimer()
    // 어떤 경우에도 터치 끝나면 스크롤 허용
    isBlockingScroll.value = false
  
    if (isActive.value) {
      isActive.value = false
      showPreview.value = false
      activeIndex.value = null
      // displaySrc는 유지(원하면 초기화 가능)
      // displaySrc.value = null
      loadToken++       // 진행 중 프리로드 무효화
      inFlightFor = null
    }
  }
  onMounted(() => {
  warmupAllImages(3)  // ✅ 페이지 진입 시 전부 프리로드(동시 3개)
})
  
  </script>
  
  <style scoped>
  /* 필요시 커스텀 스타일 추가 */
  </style>
  