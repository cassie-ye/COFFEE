<script setup>
/** 高德地图实例等用 shallowRef，避免 Vue 深度代理导致 SDK 内部 postMessage 克隆报错 */
const map = shallowRef(null)
const currentLocationMarker = shallowRef(null)
const geolocation = shallowRef(null)
const Amap = shallowRef(null)
const show = ref(false)
const authorPopupShow = ref(false)
/** 作者弹层内头像是否已加载完成，用于显示 loading */
const authorImgLoaded = ref(false)
const authorImgRef = ref(null)
const list = ref([])
const currentStore = ref({})
const pointList = ref([])
/** 当前视野内的点位，随视口变化更新 */
const pointsInBounds = ref([])
/** 点聚合实例，用于视口变化时先销毁再重建（shallowRef 避免被深度代理） */
const clusterRef = shallowRef(null)
/** 当前定位 [lng, lat]，定位成功后写入，用于计算最近门店 */
const currentLocation = ref(null)

/** 主题切换中（等地图样式完全切换后再隐藏，避免导航栏已变但地图还未变） */
const themeChanging = ref(false)
/** 首屏 loading：进入页即显示，定位完成且地图 complete 后结束 */
const firstScreenLoading = ref(true)
const mapReady = ref(false)
const geolocationDone = ref(false)
function tryEndFirstScreenLoading() {
  if (mapReady.value && geolocationDone.value)
    firstScreenLoading.value = false
}

/** 百度地图：标点链接（打开后可点「驾车」导航；api 子域 direction 已不可用，用 marker） */
const baiduNavUrl = computed(() => {
  const c = currentStore.value?.coordinates
  const lat = c?.latitude
  const lng = c?.longitude
  const name = currentStore.value?.name || '星巴克'
  const address = currentStore.value?.address?.streetAddressLine3 || ''
  if (lat != null && lng != null) {
    const params = new URLSearchParams({
      location: `${lat},${lng}`,
      title: name,
      content: address || name,
      output: 'html',
      coord_type: 'gcj02',
      src: 'webapp.starbucks.coffee',
    })
    return `https://api.map.baidu.com/marker?${params.toString()}`
  }
  const query = [name, address].filter(Boolean).join(' ')
  return query ? `https://map.baidu.com/search?query=${encodeURIComponent(query)}` : ''
})

/** 高德地图导航 URL（to=经度,纬度,名称） */
const amapNavUrl = computed(() => {
  const c = currentStore.value?.coordinates
  const lng = c?.longitude
  const lat = c?.latitude
  if (lng == null || lat == null)
    return ''
  const name = currentStore.value?.name || '星巴克'
  return `https://uri.amap.com/navigation?to=${lng},${lat},${encodeURIComponent(name)}`
})
const route = useRoute()

/** 搜索弹层（替代 router.push('/search')，避免 GitHub Pages 下路由切换慢） */
const searchPopupShow = ref(false)
const searchVal = ref('')
const searchKeyword = ref('')
const setSearchKeyword = useDebounceFn((val) => {
  searchKeyword.value = (val ?? '').trim()
}, 300)
watch(searchVal, val => setSearchKeyword(val))

function matchStore(keyword, store) {
  const k = keyword.toLowerCase()
  const name = (store.name ?? '').toLowerCase()
  const city = (store.address?.city ?? '').toLowerCase()
  const line1 = (store.address?.streetAddressLine1 ?? '').toLowerCase()
  const line3 = (store.address?.streetAddressLine3 ?? '').toLowerCase()
  return name.includes(k) || city.includes(k) || line1.includes(k) || line3.includes(k)
}

const searchFilteredList = computed(() => {
  const keyword = searchKeyword.value
  if (!keyword)
    return list.value
  return list.value.filter(store => matchStore(keyword, store))
})

/** Vant List 分批加载：每批条数 */
const SEARCH_PAGE_SIZE = 10
/** 搜索弹层内实际展示的门店（分批 push） */
const searchDisplayList = ref([])
const searchListLoading = ref(false)
const searchListFinished = ref(false)
/** 列表加载失败（如拉取门店列表失败），点击可重试 */
const searchListError = ref(false)

/** 重置为只展示前 20 条（防抖结束或弹层打开时调用） */
function resetSearchDisplayList() {
  const full = searchFilteredList.value
  searchDisplayList.value = full.slice(0, SEARCH_PAGE_SIZE)
  searchListFinished.value = full.length <= SEARCH_PAGE_SIZE
}

function onSearchListLoad() {
  if (!searchKeyword.value) {
    searchListLoading.value = false
    return
  }
  if (list.value.length === 0) {
    searchListLoading.value = true
    searchListError.value = false
    fetchStoreList()
      .then(({ storeList: stores }) => {
        list.value = stores
        resetSearchDisplayList()
      })
      .catch(() => {
        searchListError.value = true
      })
      .finally(() => {
        searchListLoading.value = false
      })
    return
  }
  searchListLoading.value = true
  searchListError.value = false
  const full = searchFilteredList.value
  const start = searchDisplayList.value.length
  const nextBatch = full.slice(start, start + SEARCH_PAGE_SIZE)
  for (const item of nextBatch) searchDisplayList.value.push(item)
  searchListLoading.value = false
  if (searchDisplayList.value.length >= full.length)
    searchListFinished.value = true
}

/** 弹层打开时：清空关键词，默认展示最近 10 家 */
watch(searchPopupShow, (open) => {
  if (open) {
    searchVal.value = ''
    nextTick(resetSearchDisplayList)
  }
})
/** 搜索防抖结束后：只展示前 20 条，滚动再加载更多 */
watch(searchKeyword, () => {
  if (searchPopupShow.value)
    nextTick(resetSearchDisplayList)
})

/** 作者弹层打开时重置头像 loading；若图片已缓存则不再触发 load，需在 nextTick 检查 complete */
watch(authorPopupShow, (open) => {
  if (open) {
    authorImgLoaded.value = false
    nextTick(() => {
      if (authorImgRef.value?.complete)
        authorImgLoaded.value = true
    })
  }
})

function openSearchPopup() {
  searchPopupShow.value = true
  searchListError.value = false
  if (!list.value.length) {
    fetchStoreList()
      .then(({ storeList: stores }) => {
        list.value = stores
        resetSearchDisplayList()
      })
      .catch(() => {
        searchListError.value = true
      })
  }
}

function onSelectStoreFromSearch(store) {
  searchPopupShow.value = false
  currentStore.value = store
  if (
    map.value
    && store?.coordinates?.longitude != null
    && store?.coordinates?.latitude != null
  ) {
    map.value.setZoomAndCenter(
      18,
      [store.coordinates.longitude, store.coordinates.latitude],
      false,
    )
  }
  nextTick(() => {
    setTimeout(() => {
      show.value = true
    }, 2000)
  })
}

/** 初始化地图（动态 import 避免 SSG 时 Node 无 window） */
async function initMap() {
  const { default: AMapLoader } = await import('@amap/amap-jsapi-loader')
  const amapKey = import.meta.env.VITE_AMAP_KEY
  if (!amapKey)
    return Promise.reject(new Error('Missing VITE_AMAP_KEY'))
  window._AMapSecurityConfig = { securityJsCode: amapKey }
  return AMapLoader.load({
    key: amapKey,
    version: '2.0',
    plugins: ['AMap.Scale', 'AMap.MarkerCluster', 'AMap.Geolocation'],
  })
}

/** 初始化定位：创建 Geolocation 实例并请求当前位置，成功后居中并添加蓝色标记 */
function initGeolocation(AmapInstance) {
  AmapInstance.plugin('AMap.Geolocation', () => {
    geolocation.value = new AmapInstance.Geolocation({
      enableHighAccuracy: true,
      timeout: 10000,
      offset: [10, 20],
      zoomToAccuracy: true,
      position: 'RB',
    })

    geolocation.value.getCurrentPosition((status, result) => {
      if (status === 'complete') {
        onGeolocationComplete(AmapInstance, result)
      }
      else {
        onGeolocationError(result)
      }
    })
  })
}

/** 从定位结果中解析出 [lng, lat] 中心点 */
function parsePositionToCenter(position) {
  if (!position)
    return null
  const lng = position.lng ?? position.longitude
  const lat = position.lat ?? position.latitude
  if (lng == null || lat == null)
    return null
  return [lng, lat]
}

/** 设置地图中心 */
function setMapCenter(center) {
  if (map.value && center)
    map.value.setCenter(center)
}

/** 移除旧标记并在 center 处创建蓝色当前位置标记（雷达波纹效果） */
function createLocationMarker(AmapInstance, center) {
  if (!map.value || !center)
    return
  if (currentLocationMarker.value) {
    currentLocationMarker.value.setMap(null)
  }
  const markerContent = `
    <div class="location-radar-wrap">
      <span class="location-radar-ring location-radar-ring-1"></span>
      <span class="location-radar-ring location-radar-ring-2"></span>
      <span class="location-radar-ring location-radar-ring-3"></span>
      <span class="location-radar-dot"></span>
    </div>
  `
  const marker = new AmapInstance.Marker({
    position: center,
    content: markerContent,
    offset: new AmapInstance.Pixel(-20, -20),
  })
  marker.setMap(map.value)
  currentLocationMarker.value = marker
}

/** 定位成功：若当前是从搜索进入查看某门店（有 storeId），只添加当前位置标记不移动地图中心，避免覆盖门店视图 */
function onGeolocationComplete(AmapInstance, data) {
  console.warn('定位成功：', data)
  const position = data.position || data.location
  const center = parsePositionToCenter(position)
  if (!center || !map.value)
    return
  currentLocation.value = center
  const fromSearchStore = !!route.query.storeId
  if (!fromSearchStore)
    setMapCenter(center)
  createLocationMarker(AmapInstance, center)
  geolocationDone.value = true
  tryEndFirstScreenLoading()

  // 定位后视口变化会触发 moveend，由 loadAndRenderStores 绑定的 updateClusterByViewport 更新聚合
  if (pointList.value?.length && Amap.value)
    updateClusterByViewport(Amap.value)
}

/** 定位失败 */
function onGeolocationError(data) {
  console.error('定位失败：', data)
  geolocationDone.value = true
  tryEndFirstScreenLoading()
}

/** 两经纬度点之间的球面距离（km），Haversine 公式 */
function haversineKm(lat1, lng1, lat2, lng2) {
  const R = 6371
  const toRad = d => (d * Math.PI) / 180
  const dLat = toRad(lat2 - lat1)
  const dLng = toRad(lng2 - lng1)
  const a
    = Math.sin(dLat / 2) ** 2
      + Math.cos(toRad(lat1)) * Math.cos(toRad(lat2)) * Math.sin(dLng / 2) ** 2
  const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a))
  return R * c
}

/** 距离当前定位最近的 10 家门店（需先定位成功且有门店数据），带距离 km */
const nearestStoresWithDistance = computed(() => {
  const loc = currentLocation.value
  const stores = list.value
  if (!loc?.length || loc.length < 2 || !stores?.length)
    return []
  const [lng, lat] = loc
  return [...stores]
    .filter(s => s.coordinates?.latitude != null && s.coordinates?.longitude != null)
    .map(store => ({
      store,
      distance:
        Math.round(
          haversineKm(lat, lng, store.coordinates.latitude, store.coordinates.longitude)
          * 10,
        ) / 10,
    }))
    .sort((a, b) => a.distance - b.distance)
    .slice(0, 10)
})

/** 搜索弹层列表实际展示的数据：无关键词时默认展示「离你最近」10 家（带距离），有关键词时展示搜索结果（分批，也带距离） */
const searchListDisplay = computed(() => {
  if (searchKeyword.value) {
    const loc = currentLocation.value
    return searchDisplayList.value
      .map((store) => {
        let distance
        if (
          loc?.length >= 2
          && store.coordinates?.latitude != null
          && store.coordinates?.longitude != null
        ) {
          distance
            = Math.round(
              haversineKm(
                loc[1],
                loc[0],
                store.coordinates.latitude,
                store.coordinates.longitude,
              ) * 10,
            ) / 10
        }
        return { store, distance }
      })
      .sort((a, b) => {
        const da = a.distance
        const db = b.distance
        if (da == null && db == null)
          return 0
        if (da == null)
          return 1
        if (db == null)
          return -1
        return da - db
      })
  }
  return nearestStoresWithDistance.value
})
/** 无关键词时列表已「加载完」（只展示最近 10 家，不需 load more） */
const searchListFinishedEffective = computed(() =>
  searchKeyword.value ? searchListFinished.value : true,
)

/** 请求门店列表，返回带坐标的门店与点位数组 */
function fetchStoreList() {
  return fetch('/store.json')
    .then(res => res.json())
    .then((json) => {
      const data = json.data || []
      const storeList = data.filter(
        d => d.coordinates?.longitude != null && d.coordinates?.latitude != null,
      )
      const points = storeList.map(d => ({
        lnglat: [d.coordinates.longitude, d.coordinates.latitude],
        id: d.id,
      }))
      return { storeList, points }
    })
}

/** 聚合点渲染：星巴克绿色系渐变 */
function createRenderClusterMarker(AmapInstance, totalCount) {
  return function (context) {
    const factor = (context.count / totalCount) ** (1 / 18)
    const div = document.createElement('div')
    const Hue = 140 + factor * 18
    const lightness = 50 - factor * 30
    div.style.backgroundColor = `hsla(${Hue},100%,${lightness}%,0.7)`
    const size = Math.round(30 + (context.count / totalCount) ** (1 / 5) * 20)
    div.style.width = div.style.height = `${size}px`
    div.style.border = `solid 1px hsl(${Hue},100%,${lightness}%)`
    div.style.borderRadius = `${size / 2}px`
    div.style.boxShadow = `0 0 5px hsla(${Hue},100%,${lightness + 20}%,0.5)`
    div.innerHTML = context.count
    div.style.lineHeight = `${size}px`
    div.style.color = '#ffffff'
    div.style.fontSize = '14px'
    div.style.textAlign = 'center'
    context.marker.setOffset(new AmapInstance.Pixel(-size / 2, -size / 2))
    context.marker.setContent(div)
  }
}

/** 单点渲染：星巴克主色 + 点击打开门店弹层 */
function createRenderStoreMarker(AmapInstance) {
  return function (context) {
    const content
      = '<div style="background-color: rgba(0, 100, 64, 0.8); height: 18px; width: 18px; border: 2px solid rgb(0, 100, 64); border-radius: 12px; box-shadow: rgba(0, 100, 64, 0.6) 0px 0px 4px;"></div>'
    context.marker.setContent(content)
    context.marker.setOffset(new AmapInstance.Pixel(-9, -9))

    const pointData = context.data?.[0]
    context.marker.on('click', (e) => {
      const store = pointData?.id
        ? list.value.find(item => item.id === pointData.id)
        : null
      if (store)
        currentStore.value = store
      if (e.originEvent) {
        e.originEvent.stopPropagation()
        e.originEvent.preventDefault()
      }
      nextTick(() => {
        show.value = true
      })
    })
  }
}

/** 地图平滑缩放到指定中心与级别 */
function animateMapToZoomAndCenter(targetCenter, zoomDelta = 2, duration = 800) {
  if (!map.value || !targetCenter)
    return
  const currentZoom = map.value.getZoom()
  const currentCenter = map.value.getCenter()
  const targetZoom = Math.min(currentZoom + zoomDelta, 18)
  const easeOutCubic = t => 1 - (1 - t) ** 3
  const startTime = Date.now()
  const startCenter = [currentCenter.lng, currentCenter.lat]
  const centerDiff = [targetCenter[0] - startCenter[0], targetCenter[1] - startCenter[1]]
  const zoomDiff = targetZoom - currentZoom

  const animate = () => {
    const elapsed = Date.now() - startTime
    const progress = Math.min(elapsed / duration, 1)
    const eased = easeOutCubic(progress)
    map.value.setZoomAndCenter(
      currentZoom + zoomDiff * eased,
      [startCenter[0] + centerDiff[0] * eased, startCenter[1] + centerDiff[1] * eased],
      false,
    )
    if (progress < 1) {
      requestAnimationFrame(animate)
    }
    else {
      map.value.setZoomAndCenter(targetZoom, targetCenter, false)
    }
  }
  requestAnimationFrame(animate)
}

/** 为聚合实例绑定点击放大：仅聚合点触发平滑缩放，单点不缩放 */
function setupClusterClickZoom(cluster) {
  cluster.on('click', (e) => {
    const clusterCount = e.clusterData?.length ?? 0
    if (clusterCount <= 1)
      return
    const center = e.lnglat ? [e.lnglat.lng, e.lnglat.lat] : null
    if (center)
      animateMapToZoomAndCenter(center)
  })
}

/** 根据当前视口 bounds 过滤点位，销毁旧聚合并只渲染视野内的点聚合 */
function updateClusterByViewport(AmapInstance) {
  if (!map.value || !pointList.value?.length)
    return
  const b = map.value.getBounds()
  if (!b)
    return
  const inBounds = pointList.value.filter((point) => {
    const ll = point.lnglat
    const lng = Array.isArray(ll) ? ll[0] : ll?.lng ?? ll?.longitude
    const lat = Array.isArray(ll) ? ll[1] : ll?.lat ?? ll?.latitude
    return lng != null && lat != null && b.contains(new AmapInstance.LngLat(lng, lat))
  })
  pointsInBounds.value = inBounds

  if (clusterRef.value) {
    clusterRef.value.setMap(null)
    clusterRef.value = null
  }
  if (!inBounds.length)
    return
  const myList = inBounds.map(point => ({
    lnglat: Array.isArray(point.lnglat)
      ? point.lnglat
      : [
          point.lnglat?.lng ?? point.lnglat?.longitude,
          point.lnglat?.lat ?? point.lnglat?.latitude,
        ],
    id: point.id,
  }))
  const gridSize = 60
  const cluster = new AmapInstance.MarkerCluster(map.value, myList, {
    gridSize,
    renderClusterMarker: createRenderClusterMarker(AmapInstance, myList.length),
    renderMarker: createRenderStoreMarker(AmapInstance),
  })
  setupClusterClickZoom(cluster)
  clusterRef.value = cluster
}

/** 是否已绑定视口变化监听，避免重复绑定 */
let viewportListenersBound = false

/** 加载门店数据，并按当前视口渲染点聚合；绑定 moveend/zoomend 后随视口更新聚合 */
function loadAndRenderStores(AmapInstance) {
  fetchStoreList()
    .then(({ storeList, points }) => {
      list.value = storeList
      pointList.value = points
      updateClusterByViewport(AmapInstance)
      if (map.value && !viewportListenersBound) {
        viewportListenersBound = true
        const onViewportChange = useDebounceFn(
          () => updateClusterByViewport(Amap.value),
          150,
        )
        map.value.on('moveend', onViewportChange)
        map.value.on('zoomend', onViewportChange)
      }
    })
    .catch(e => console.error(e))
}

/** 根据路由 query.storeId 打开对应门店详情弹层（从搜索页跳转时）：先放大地图到该点再出 popup */
watch(
  [() => route.query.storeId, list],
  ([storeId, listVal]) => {
    if (!storeId || !listVal?.length)
      return
    const store = listVal.find(item => item.id === storeId)
    if (
      !store?.coordinates
      || store.coordinates.longitude == null
      || store.coordinates.latitude == null
    ) {
      return
    }
    currentStore.value = store
    const center = [store.coordinates.longitude, store.coordinates.latitude]
    if (map.value) {
      // 放大到最大级别（18）并居中到该门店，便于看清点位
      map.value.setZoomAndCenter(18, center, false)
      // 等地图移动/缩放完成后再弹出详情
      const onMoveEnd = () => {
        map.value?.off('moveend', onMoveEnd)
        nextTick(() => {
          show.value = true
        })
      }
      map.value.on('moveend', onMoveEnd)
      // 若地图已稳定（无动画）则延迟兜底打开 popup
      setTimeout(() => {
        if (!show.value)
          onMoveEnd()
      }, 900)
    }
    else {
      nextTick(() => {
        show.value = true
      })
    }
  },
  { immediate: true },
)

/** 根据 isDark 设置地图底图样式（深色/标准）；返回 Promise，地图样式切换完成后再 resolve（高德无完成回调，用延迟近似） */
function applyMapStyle() {
  if (!map.value)
    return Promise.resolve()
  const style = isDark.value ? 'amap://styles/dark' : 'amap://styles/normal'
  return new Promise((resolve) => {
    nextTick(() => {
      map.value?.setMapStyle(style)
      setTimeout(resolve, 1800)
    })
  })
}

function onThemeToggle() {
  themeChanging.value = true
  nextTick(() => {
    toggleDark()
  })
}

onMounted(() => {
  initMap()
    .then((AmapInstance) => {
      Amap.value = AmapInstance
      map.value = new AmapInstance.Map('container', {
        viewMode: '3D',
        zoom: 11,
        mapStyle: isDark.value ? 'amap://styles/dark' : 'amap://styles/normal',
      })

      initGeolocation(AmapInstance)

      map.value.on('complete', () => {
        const run = () => loadAndRenderStores(AmapInstance)
        if (typeof requestIdleCallback !== 'undefined') {
          requestIdleCallback(run, { timeout: 500 })
        }
        else {
          setTimeout(run, 0)
        }
        mapReady.value = true
        tryEndFirstScreenLoading()
      })
    })
    .catch((e) => {
      console.error(e)
      firstScreenLoading.value = false
    })
})

watch(isDark, () => {
  const p = applyMapStyle()
  if (p) {
    p.finally(() => {
      themeChanging.value = false
    })
  }
  else {
    themeChanging.value = false
  }
})

/** 定位到当前位置：请求定位后居中并更新蓝色标记，复用 onGeolocationComplete / onGeolocationError */
function locateToCurrentPosition() {
  if (!geolocation.value || !map.value)
    return
  geolocation.value.getCurrentPosition((status, result) => {
    if (status === 'complete') {
      onGeolocationComplete(Amap.value, result)
    }
    else {
      onGeolocationError(result)
    }
  })
}

/** 缩放/视野范围选项（米），用于 picker 选择 */
const columns = [
  { text: '1~2km', value: 1500 },
  { text: '2~5km', value: 3500 },
  { text: '5~10km', value: 7500 },
  { text: '10~20km', value: 15000 },
  { text: '20~50km', value: 35000 },
  { text: '50~100km', value: 75000 },
  { text: '100~200km', value: 150000 },
  { text: '全国', value: 5000000 },
]
/** 视野范围 value（米）对应的高德 zoom 级别（中心不变，只改缩放） */
const scaleValueToZoom = {
  1500: 16,
  3500: 15,
  7500: 14,
  15000: 13,
  35000: 12,
  75000: 11,
  150000: 10,
  5000000: 4,
}
const fieldValue = ref('')
const showPicker = ref(false)
const pickerValue = ref([])
function onConfirm({ selectedValues, selectedOptions }) {
  showPicker.value = false
  pickerValue.value = selectedValues
  fieldValue.value = selectedOptions[0].text
  if (!map.value)
    return
  const value = selectedOptions[0]?.value
  const zoom = scaleValueToZoom[value] ?? map.value.getZoom()
  const center = map.value.getCenter()
  const centerArr = center ? [center.lng, center.lat] : undefined
  if (centerArr)
    map.value.setZoomAndCenter(zoom, centerArr, true)
}
</script>

<template>
  <div>
    <!-- 首屏 loading：定位完成且地图 complete 后消失 -->
    <Transition name="fade">
      <div
        v-show="firstScreenLoading"
        class="fixed inset-0 z-30 flex flex-col items-center justify-center gap-4 bg-white dark:bg-[#1b1b1b]"
      >
        <div class="i-line-md-loading-loop text-5xl text-[#00704A] dark:text-green-400" />
        <div class="text-sm text-gray-600 dark:text-gray-400">
          加载地图与定位…
        </div>
      </div>
    </Transition>
    <!-- 主题切换中：等地图样式完全切换后再隐藏 -->
    <Transition name="fade">
      <div
        v-show="themeChanging"
        class="fixed inset-0 z-40 flex flex-col items-center justify-center gap-4 bg-black/50"
      >
        <div class="i-line-md-loading-loop text-5xl text-white" />
        <div class="text-sm text-white/90">
          切换主题中…
        </div>
      </div>
    </Transition>
    <div
      class="h-12 w-full flex items-center justify-between gap-2 bg-white p-2 dark:bg-[#1b1b1b]"
      border="b gray-200 dark:border-gray-700"
      fixed
      left-0
      right-0
      top-0
      z-10
    >
      <!-- eslint-disable-next-line uno/order -- 大点击区与 active 反馈顺序 -->
      <div class="flex items-center gap-1">
        <div class="icon-link" @click="openSearchPopup">
          <div class="i-carbon-search text-xl text-gray-800 dark:text-gray-200" />
        </div>
        <div class="icon-link" @click="authorPopupShow = true">
          <div class="i-carbon-user-favorite text-xl text-gray-800 dark:text-gray-200" />
        </div>
      </div>

      <div class="flex items-center gap-2">
        <div class="i-line-md-coffee-loop text-2xl text-gray-800 dark:text-gray-200" />
        <div class="text-gray-900 font-bold dark:text-gray-100">
          coffee
        </div>
      </div>
      <div class="flex items-center gap-1">
        <div class="icon-link" @click="onThemeToggle">
          <div v-if="isDark" class="i-carbon-sun text-xl text-amber-400" />
          <div v-else class="i-carbon-moon text-xl text-gray-600" />
        </div>
        <!-- 定位：大点击区 + 按下反馈 -->
        <div class="icon-link" @click="locateToCurrentPosition">
          <div class="i-carbon-location-heart text-xl text-gray-700 dark:text-gray-300" />
        </div>
        <div class="icon-link" @click="showPicker = true">
          <div class="i-carbon-zoom-area text-xl text-gray-700 dark:text-gray-300" />
        </div>
      </div>
    </div>
    <div class="relative mt-3rem h-[calc(100vh-3rem)] w-full">
      <div id="container" class="absolute inset-0" />
      <!-- 当前视野内门店数：毛玻璃卡片 -->
      <Transition name="slide-up">
        <div
          v-show="pointList.length > 0"
          class="absolute bottom-2 right-2 z-20 flex items-center gap-1 border border-white/30 rounded-lg bg-white/40 px-2 shadow-md backdrop-blur-sm dark:border-gray-500/30 dark:bg-gray-900/40"
        >
          <div class="min-w-0">
            <div class="flex items-baseline gap-1.5">
              <span
                class="text-lg text-[#00704A] font-bold tabular-nums dark:text-green-400"
              >
                {{ pointsInBounds.length }}
              </span>
              <span class="text-xs text-gray-700 dark:text-gray-300"> 家门店 </span>
            </div>
          </div>
        </div>
      </Transition>
    </div>
    <van-popup v-model:show="show" :style="{ padding: '20px' }" position="bottom" round>
      <div class="text-xl text-gray-900 font-bold dark:text-gray-100">
        星巴克 ({{ currentStore.name || "--" }})
      </div>
      <div class="mt-2 flex items-start gap-2 text-gray-700 dark:text-gray-300">
        <div class="i-carbon-location mt0.6" />
        <div class="flex-1">
          {{ currentStore.address.streetAddressLine3 || "--" }}
        </div>
      </div>
      <div class="mt-2 flex items-center gap-2 text-gray-700 dark:text-gray-300">
        <div class="i-carbon-time" />
        营业时间: {{ currentStore.today.openTime }}-{{ currentStore.today.closeTime }}
      </div>
      <div class="mt-2 flex items-center gap-2 text-gray-700 dark:text-gray-300">
        <div
          :class="
            currentStore.hasArtwork ? 'i-carbon-chart-bubble-packed' : 'i-carbon-heat-map'
          "
        />
        {{ currentStore.hasArtwork ? "特色艺术装饰/建筑设计" : "标准门店" }}
      </div>
      <div class="mt-2">
        <div class="mb-2 flex items-center gap-2 text-gray-700 dark:text-gray-300">
          <div class="i-carbon-tag-group" />
          门店标签
        </div>
        <div class="flex flex-wrap gap-2">
          <div
            v-for="i in currentStore.features"
            :key="i"
            class="border border-gray-200 rounded-lg bg-gray-50 p-2 text-xs dark:border-gray-600 dark:bg-gray-800"
          >
            <div class="text-green-600 font-bold dark:text-green-400">
              {{ i }}
            </div>
          </div>
        </div>
      </div>
      <div class="mt-4">
        <div
          class="mb-3 flex items-center gap-1.5 text-sm text-gray-600 font-semibold dark:text-gray-400"
        >
          <div class="i-carbon-map text-base text-[#00704A] dark:text-green-400" />
          <span>到这去喝咖啡</span>
        </div>
        <div class="grid grid-cols-2 gap-3">
          <a
            :href="baiduNavUrl || '#'"
            class="group map-nav-card flex items-center gap-2.5 border border-gray-200 rounded-xl bg-white px-3 py-1.5 text-left shadow-sm transition-all dark:border-gray-600 hover:border-amber-400 dark:bg-gray-800 hover:bg-amber-50 hover:shadow dark:hover:border-amber-500 dark:hover:bg-amber-900/20"
            target="_blank"
            rel="noopener noreferrer"
            @click="!baiduNavUrl && $event.preventDefault()"
          >
            <div
              class="h-7 w-7 flex shrink-0 items-center justify-center rounded-lg bg-blue-100 text-blue-600 dark:bg-blue-900/50 dark:text-blue-400"
            >
              <div class="i-carbon-location-filled" />
            </div>
            <div class="min-w-0 flex-1">
              <div class="text-xs text-gray-800 font-semibold dark:text-gray-200">
                百度地图
              </div>
              <div class="mt-0.5 text-xs text-gray-500 dark:text-gray-400">打开导航</div>
            </div>
            <div
              class="i-carbon-arrow-right text-gray-400 opacity-0 transition-opacity dark:text-gray-500 group-hover:opacity-100"
            />
          </a>
          <a
            :href="amapNavUrl || '#'"
            class="map-nav-card group flex items-center gap-2.5 border border-gray-200 rounded-xl bg-white px-3 py-1.5 text-left shadow-sm transition-all dark:border-gray-600 hover:border-amber-400 dark:bg-gray-800 hover:bg-amber-50 hover:shadow dark:hover:border-amber-500 dark:hover:bg-amber-900/20"
            target="_blank"
            rel="noopener noreferrer"
            @click="!amapNavUrl && $event.preventDefault()"
          >
            <div
              class="h-7 w-7 flex shrink-0 items-center justify-center rounded-lg bg-green-100 text-green-600 dark:bg-green-900/50 dark:text-green-400"
            >
              <div class="i-carbon-location-filled" />
            </div>
            <div class="min-w-0 flex-1">
              <div class="text-xs text-gray-800 font-semibold dark:text-gray-200">
                高德地图
              </div>
              <div class="mt-0.5 text-xs text-gray-500 dark:text-gray-400">打开导航</div>
            </div>
            <div
              class="i-carbon-arrow-right text-gray-400 opacity-0 transition-opacity dark:text-gray-500 group-hover:opacity-100"
            />
          </a>
        </div>
      </div>
    </van-popup>
    <!-- 搜索弹层：全屏从右滑出，替代路由 /search，避免 GitHub Pages 下 router.push 慢 -->
    <van-popup
      v-model:show="searchPopupShow"
      position="right"
      :style="{ width: '100%', height: '100%' }"
      class="search-popup"
    >
      <div class="h-full flex flex-col bg-white dark:bg-[#1b1b1b]">
        <div
          class="h-12 w-full flex shrink-0 items-center justify-between gap-2 border-b border-gray-200 p-2 dark:border-gray-700"
        >
          <div class="icon-link" @click="searchPopupShow = false">
            <div class="i-carbon-arrow-left text-xl text-gray-800 dark:text-gray-200" />
          </div>
          <div class="flex items-center gap-2">
            <div
              class="i-line-md-coffee-loop text-2xl text-gray-800 dark:text-gray-200"
            />
            <div class="text-gray-900 font-bold dark:text-gray-100">
              搜索门店
            </div>
          </div>
          <div class="icon-link" @click="onThemeToggle">
            <div v-if="isDark" class="i-carbon-sun text-xl text-amber-400" />
            <div v-else class="i-carbon-moon text-xl text-gray-600" />
          </div>
        </div>
        <div class="shrink-0">
          <van-search v-model="searchVal" placeholder="请输入搜索关键字" />
        </div>
        <div class="min-h-0 flex-1 overflow-y-auto px-3 pb-4 pt-2">
          <van-list
            v-model:loading="searchListLoading"
            v-model:error="searchListError"
            error-text="请求失败，点击重新加载"
            :finished="searchListFinishedEffective"
            finished-text="没有更多了"
            :immediate-check="true"
            @load="onSearchListLoad"
          >
            <div
              v-for="(item, index) in searchListDisplay"
              :key="item.store.id"
              class="cursor-pointer border-b border-gray-200 py-3 dark:border-gray-700 active:bg-gray-100 dark:active:bg-gray-800"
              @click="onSelectStoreFromSearch(item.store)"
            >
              <p class="text-gray-900 font-bold dark:text-gray-100">
                {{ item.store.name }}
              </p>
              <div class="mt-1 text-xs text-gray-600 dark:text-gray-400">
                {{
                  item.store.address?.streetAddressLine3
                    || item.store.address?.city
                    || "—"
                }}
              </div>
              <div class="mt-1 flex items-center gap-2 text-xs">
                <span
                  v-if="item.distance != null"
                  class="text-amber-600 dark:text-amber-400"
                >{{
                  index === 0 ? `离您最近 ${item.distance}` : item.distance
                }}
                  km</span>
                <div
                  v-if="item.store.features?.length"
                  class="text-green-700 font-bold dark:text-green-400"
                >
                  {{ item.store.features.join(" ") }}
                </div>
              </div>
            </div>
          </van-list>
          <div
            v-if="
              searchListFinished
                && searchDisplayList.length === 0
                && searchFilteredList.length === 0
            "
            class="py-8 text-center text-sm text-gray-500 dark:text-gray-400"
          >
            {{ list.length ? "未找到相关门店" : "加载中…" }}
          </div>
        </div>
      </div>
    </van-popup>
    <van-popup v-model:show="authorPopupShow" position="center" round>
      <div
        class="w-60 flex flex-col justify-center bg-#fff p3 py6 dark:bg-#1b1b1b dark:text-gray-200"
      >
        <div class="relative m-auto h-24 w-24">
          <div
            v-show="!authorImgLoaded"
            class="absolute inset-0 flex items-center justify-center rounded-2 bg-gray-200 dark:bg-gray-700"
          >
            <div class="i-line-md-loading-loop text-3xl text-gray-400 dark:text-gray-500" />
          </div>
          <img
            ref="authorImgRef"
            class="h-24 w-24 rounded-2"
            :class="{ 'opacity-0': !authorImgLoaded }"
            src="/me.jpg"
            alt=""
            loading="lazy"
            @load="authorImgLoaded = true"
          >
        </div>
        <div mt3>
          Author: <span color-pink-3 font-bold>Joie Pink</span>
        </div>
        <div>Bio: <span text-sm>他山之石, 可以攻玉</span></div>
        <div>Base: <span text-sm>Ningbo, ZheJiang, China</span></div>
        <div>
          Find me:
          <div class="flex items-center gap-1">
            <a
              href="https://github.com/JoiePink"
              target="_blank"
              rel="noopener noreferrer"
              class="icon-link"
            >
              <div
                class="i-carbon-logo-github text-xl text-gray-800 dark:text-gray-200"
              />
            </a>
            <a
              href="https://juejin.cn/user/3265984436383947"
              target="_blank"
              rel="noopener noreferrer"
              class="icon-link"
            >
              <div
                class="i-simple-icons-juejin text-xl text-gray-800 dark:text-gray-200"
              />
            </a>
            <a
              href="https://www.xiaohongshu.com/user/profile/65e54a70000000000b0327f0"
              target="_blank"
              rel="noopener noreferrer"
              class="icon-link"
            >
              <div
                class="i-arcticons-xiaohongshu-rednote text-xl text-gray-800 dark:text-gray-200"
              />
            </a>
            <a
              href="https://www.npmjs.com/~joie_pink"
              target="_blank"
              rel="noopener noreferrer"
              class="icon-link"
            >
              <div class="i-carbon-logo-npm text-xl text-gray-800 dark:text-gray-200" />
            </a>
          </div>
        </div>
      </div>
    </van-popup>

    <van-popup v-model:show="showPicker" destroy-on-close round position="bottom">
      <van-picker
        :model-value="pickerValue"
        :columns="columns"
        @cancel="showPicker = false"
        @confirm="onConfirm"
      />
    </van-popup>
  </div>
</template>

<style scoped></style>

<style>
/* 首屏 loading 淡出 */
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.2s ease;
}
.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}

/* 视野内门店数卡片滑入 */
.slide-up-enter-active,
.slide-up-leave-active {
  transition:
    opacity 0.25s ease,
    transform 0.25s ease;
}
.slide-up-enter-from,
.slide-up-leave-to {
  opacity: 0;
  transform: translateY(8px);
}

/* 当前位置雷达波纹效果（非 scoped：标记由 AMap 动态插入） */
.location-radar-wrap {
  position: relative;
  width: 40px;
  height: 40px;
  display: flex;
  align-items: center;
  justify-content: center;
}
.location-radar-dot {
  position: absolute;
  width: 14px;
  height: 14px;
  background: rgba(0, 123, 255, 0.95);
  border: 3px solid #fff;
  border-radius: 50%;
  box-shadow: 0 0 10px rgba(0, 123, 255, 0.6);
  z-index: 2;
}
.location-radar-ring {
  position: absolute;
  width: 20px;
  height: 20px;
  border: 2px solid rgba(0, 123, 255, 0.7);
  border-radius: 50%;
  animation: location-radar-ripple 2s ease-out infinite;
  z-index: 1;
}
.location-radar-ring-1 {
  animation-delay: 0s;
}
.location-radar-ring-2 {
  animation-delay: 0.66s;
}
.location-radar-ring-3 {
  animation-delay: 1.33s;
}
@keyframes location-radar-ripple {
  0% {
    transform: scale(0.5);
    opacity: 0.9;
    border-width: 2px;
  }
  100% {
    transform: scale(2.2);
    opacity: 0;
    border-width: 1px;
  }
}
</style>
