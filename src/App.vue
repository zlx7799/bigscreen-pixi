<!--
 * @Description: 
 * @Author: zhoulx
 * @Date: 2023-09-25 08:47:24
 * @LastEditors: zhoulx
 * @LastEditTime: 2023-10-04 08:08:53
-->
<script setup lang="ts">
import DisplayResult from './components/DisplayResult/Index.vue'
import { onMounted, ref, toValue, computed } from 'vue'

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
    inspectRound: ballList.value.length + 1,
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
</script>

<template>
  <div class="wrapper">
    <DisplayResult :ball-list="ballList" :pie-list="pieList" />
  </div>
  <div class="btn-container">
    <div class="btn" @click="addData">增加数据</div>
    <div class="btn" @click="resetData">重置数据</div>
  </div>
</template>

<style scoped>
.wrapper {
  width: 651px;
  height: 450px;
  /* width: 1302px;
  height: 900px; */
  /* width: 2144px;
  height: 1922px; */
  margin: auto;
}
.btn-container {
  width: 300px;
  height: 400px;
  position: fixed;
  left: 50%;
  top: 100px;
}
.btn-container .btn {
  cursor: pointer;
}
</style>
