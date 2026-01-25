<script setup>
import AMapLoader from '@amap/amap-jsapi-loader'

const map = ref(null)

onMounted(() => {
  window._AMapSecurityConfig = {
    securityJsCode: 'bf61b5b8457d13555c3d9e36321102b1',
  }
  AMapLoader.load({
    key: 'bf61b5b8457d13555c3d9e36321102b1',
    version: '2.0',
    plugins: ['AMap.Scale'],
  })
    .then((Amap) => {
      map.value = new Amap.Map('container', {
        viewMode: '3D',
        zoom: 11,
        center: [116.397428, 39.90923],
      })
    })
    .catch((e) => {
      console.error(e)
    })
})
</script>

<template>
  <div id="container" />
</template>

<style scoped>
#container {
  padding: 0px;
  margin: 0px;
  width: 100%;
  height: 800px;
}
</style>
