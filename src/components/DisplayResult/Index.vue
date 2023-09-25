<script setup lang="ts">
import { onMounted, ref, toValue } from "vue";
type DOMRef = HTMLElement | null

const displayContainer = ref<DOMRef>(null)
/**
 * @description: 获取元素宽高
 * @param {*} el
 * @return {*}
 */
const getElSize = (el: DOMRef) => {
  return {
    width: el?.clientWidth,
    height: el?.clientHeight,
  }
}

onMounted(() => {
  getElSize(toValue(displayContainer))
})

</script>

<template>
  <div class="display-container" ref="displayContainer">
  </div>
</template>

<style scoped>
.display-container {
  width: 100%;
  height: 100%;
}
</style>
