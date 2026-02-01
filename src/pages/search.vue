<script setup>
import { useRouter } from 'vue-router'

const router = useRouter()
const searchVal = ref('')
/** 完整门店列表（带坐标），用于模糊查询 */
const allStores = ref([])

/** 请求门店列表，返回带坐标的门店 */
function fetchStoreList() {
  return fetch('/store.json')
    .then(res => res.json())
    .then((json) => {
      const data = json.data || []
      const list = data.filter(
        d => d.coordinates?.longitude != null && d.coordinates?.latitude != null,
      )
      allStores.value = list
    })
}

/** 根据关键词在 name、city、streetAddressLine1、streetAddressLine3 中模糊匹配 */
function matchStore(keyword, store) {
  const k = keyword.toLowerCase()
  const name = (store.name ?? '').toLowerCase()
  const city = (store.address?.city ?? '').toLowerCase()
  const line1 = (store.address?.streetAddressLine1 ?? '').toLowerCase()
  const line3 = (store.address?.streetAddressLine3 ?? '').toLowerCase()
  return name.includes(k) || city.includes(k) || line1.includes(k) || line3.includes(k)
}

/** 当前搜索关键词（防抖后），用于过滤列表 */
const searchKeyword = ref('')
const setSearchKeyword = useDebounceFn((val) => {
  searchKeyword.value = (val ?? '').trim()
}, 300)

watch(searchVal, (val) => {
  setSearchKeyword(val)
})

/** 模糊查询后的门店列表 */
const storeList = computed(() => {
  const keyword = searchKeyword.value
  if (!keyword)
    return allStores.value
  return allStores.value.filter(store => matchStore(keyword, store))
})

/** 点击门店：跳转地图页并打开该门店详情弹层 */
function goToMapWithStore(store) {
  router.push({ path: '/', query: { storeId: store.id } })
}

onMounted(() => {
  fetchStoreList()
})
</script>

<template>
  <div>
    <div
      class="h-12 w-full flex items-center justify-between gap-2 bg-white p-2 dark:bg-[#1b1b1b]"
      border="b gray-200 dark:border-gray-700"
      fixed
      left-0
      right-0
      top-0
      z-10
    >
      <div class="icon-link" @click="router.push('/')">
        <div class="i-carbon-return text-xl text-gray-800 dark:text-gray-200" />
      </div>
      <div class="flex items-center gap-2" @click="router.push('/')">
        <div class="i-line-md-coffee-loop text-2xl text-gray-800 dark:text-gray-200" />
        <div class="text-gray-900 font-bold dark:text-gray-100">
          coffee
        </div>
      </div>
      <div class="flex items-center gap-1">
        <div class="icon-link" @click="toggleDark()">
          <div v-if="isDark" class="i-carbon-sun text-xl text-amber-400" />
          <div v-else class="i-carbon-moon text-xl text-gray-600" />
        </div>
      </div>
    </div>
    <div
      class="w-full flex justify-center bg-white dark:bg-[#1b1b1b]"
      fixed
      left-0
      right-0
      top-12
      z-10
    >
      <van-search v-model="searchVal" style="width: 100%" placeholder="请输入搜索关键字" />
    </div>
    <div class="m3 my2 bg-white pt-26 space-y-2 dark:bg-[#1b1b1b]">
      <div
        v-for="store in storeList"
        :key="store.id"
        class="cursor-pointer border-b border-gray-200 pb-2 dark:border-gray-700 active:bg-gray-100 dark:active:bg-gray-800"
        @click="goToMapWithStore(store)"
      >
        <p class="text-gray-900 font-bold dark:text-gray-100">
          {{ store.name }}
        </p>
        <div class="my1 text-xs text-gray-600 dark:text-gray-400">
          {{ store.address?.streetAddressLine3 || store.address?.city || '—' }}
        </div>
        <div class="flex items-center gap-2 text-xs">
          <div class="text-amber-600 dark:text-amber-400">
            {{ store.address?.city || '—' }}
          </div>
          <div v-if="store.features?.length" class="text-green-700 font-bold dark:text-green-400">
            {{ store.features.join(' ') }}
          </div>
        </div>
      </div>
      <div v-if="searchVal && storeList.length === 0" class="py-4 text-center text-sm text-gray-500 dark:text-gray-400">
        未找到相关门店
      </div>
    </div>
  </div>
</template>
