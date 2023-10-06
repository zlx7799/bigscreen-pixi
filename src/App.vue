<!--
 * @Description: 
 * @Author: zhoulx
 * @Date: 2023-09-25 08:47:24
 * @LastEditors: zhoulx
 * @LastEditTime: 2023-10-07 06:55:59
-->
<script setup lang="ts">
import DisplayResult from './components/DisplayResult/Index.vue'
import { onMounted, ref, toValue, computed,reactive } from 'vue'

let ballList = ref([
  {
    inspectRound: 1,
    questionNums: 10,
    inspectProgress: 1
  },
  {
    inspectRound: 2,
    questionNums: 10,
    inspectProgress: 0.55
  },
  {
    inspectRound: 3,
    questionNums: 10,
    inspectProgress: 0.45
  },
  {
    inspectRound: 4,
    questionNums: 10,
    inspectProgress: 0.75
  },
  {
    inspectRound: 5,
    questionNums: 10,
    inspectProgress: 0.85
  }
])
let pieList = ref([
  {
    inspectRound: 1,
    departmentNums: 10
  },
  {
    inspectRound: 2,
    departmentNums: 10
  },
  {
    inspectRound: 3,
    departmentNums: 10
  },
  {
    inspectRound: 4,
    departmentNums: 10
  },
  {
    inspectRound: 5,
    departmentNums: 10
  }
])
const addData = () => {
  ballList.value.push({
    inspectRound: ballList.value.length + 1,
    questionNums: 10,
    inspectProgress: Math.random()
  })
  pieList.value.push({
    inspectRound: pieList.value.length + 1,
    departmentNums: 10
  })
}

const resetData = () => {
  ballList.value = [
    {
      inspectRound: 1,
      questionNums: 10,
      inspectProgress: 1
    },
    {
      inspectRound: 2,
      questionNums: 10,
      inspectProgress: 0.54
    },
    {
      inspectRound: 3,
      questionNums: 10,
      inspectProgress: 0.41
    },
    {
      inspectRound: 4,
      questionNums: 10,
      inspectProgress: 0.75
    },
    {
      inspectRound: 5,
      questionNums: 10,
      inspectProgress: 0.85
    }
  ]
  pieList.value = [
    {
      inspectRound: 1,
      departmentNums: 10
    },
    {
      inspectRound: 2,
      departmentNums: 10
    },
    {
      inspectRound: 3,
      departmentNums: 10
    },
    {
      inspectRound: 4,
      departmentNums: 10
    },
    {
      inspectRound: 5,
      departmentNums: 10
    }
  ]
}
let width = ref(1302)
let height = ref(900)
let wrapperStyle = computed(() => {
  return {
    width: `${width.value}px`,
    height: `${height.value}px`,
  }
})
</script>

<template>
  <!-- <div class="wrapper" :style="wrapperStyle"> -->
  <div class="wrapper" >
    <DisplayResult :ball-list="ballList" :pie-list="pieList" @click-ball="(val) => console.log(val)" @click-center-ball="(val) => console.log(val)" />
  </div>
  <!-- <div class="btn-container">
    <div class="btn" @click="addData">增加数据</div>
    <div class="btn" @click="resetData">重置数据</div>
    <div class="btn" @click="width *= 2; height *= 2; console.log(width, height)">放大</div>
    <div class="btn" @click="width /= 2; height /= 2">缩小</div>
  </div> -->
</template>

<style scoped>
.wrapper {
  /* width: 651px;
  height: 450px; */
  /* width: 651px;
  height: 600px; */
  /* width: 1302px;
  height: 900px; */
  /* width: 2144px;
  height: 1922px; */
  /* margin: auto; */
  width: 100vw;
  height: 100vh;
}
.btn-container {
  width: 300px;
  height: 400px;
  position: fixed;
  left: 50%;
  top: 100px;
  position: fixed;
  top: 10px;
  left: 10px;
}
.btn-container .btn {
  cursor: pointer;
}
</style>
