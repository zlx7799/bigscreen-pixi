<script setup lang="ts">
import { onMounted, ref, toValue, computed, watch } from 'vue'
import selectedBall from '@/assets/selected.png'
import unselectedBall from '@/assets/unselected.png'
const emit = defineEmits(['click-ball', 'mouse-move-ball', 'mouse-leave-ball'])
type CanvasDomRef = HTMLCanvasElement | null
interface BallItem {
  inspectRound: number
  questionNums: number
  inspectProgress: number
}

interface Point {
  x: number
  y: number
}

interface Props {
  ballInfo: BallItem
  ballWidth: number
  pos: Point
}

const props = withDefaults(defineProps<Props>(), {
  pos: () => ({x: 0, y: 0})
})

let lineWidth = 2
let isMouseMove = ref<Boolean>(false)
let text = computed(() => {
  return `第${props.ballInfo.inspectRound}轮`
})
let ballContainerStyle = computed(() => {
  let ballWidthValue = props.ballWidth
  let style = {
    top: `${props.pos.y}px`,
    left: `${props.pos.x}px`,
    marginLeft: `${-ballWidthValue / 2}px`,
    marginTop: `${-ballWidthValue / 2}px`,
    width: `${ballWidthValue + lineWidth}px`,
    height: `${ballWidthValue + lineWidth}px`,
    'animation-play-state': 'running'
  }
  if (isMouseMove.value){
    style['animation-play-state'] = 'paused'
  } else {
    style['animation-play-state'] = 'running'
   }
  return style
})




let ballCanvas = ref<CanvasDomRef>(null)
let ctx: CanvasRenderingContext2D | null

/**
 * @description: 初始化canvas
 * @return {*}
 */
 const initCanvas = () => {
  let canvas = toValue(ballCanvas)
  if (!canvas) {
    return
  }
  canvas.width = props.ballWidth + lineWidth * 2
  canvas.height = props.ballWidth + lineWidth * 2
  ctx = canvas.getContext('2d')
  if (window.devicePixelRatio && ctx) {
    const width = ctx.canvas.width
    const height = ctx.canvas.height
    ctx.canvas.style.width = width + 'px'
    ctx.canvas.style.height = height + 'px'
    ctx.canvas.width = width * window.devicePixelRatio * 2
    ctx.canvas.height = height * window.devicePixelRatio * 2
    ctx.scale(window.devicePixelRatio * 2, window.devicePixelRatio * 2)
  }
}

let ballStyle = computed(() => {
  let ballWidthValue = props.ballWidth
  let url = unselectedBall
  if(isMouseMove.value){
    url = selectedBall
  }
  return {
    width: `${ballWidthValue}px`,
    height: `${ballWidthValue}px`,
    top: `${lineWidth}px`,
    left: `${lineWidth}px`,
    backgroundImage: `url(${url})`,
    backgroundSize: '100% 100%',
  }
})

const UNSELECTED_BALL_COLOR = '#69D5E7'
const SELECTED_BALL_COLOR = '#F4B23C'
const initRing = () => {
  if(ctx){
    let centerPos = {
      x: (props.ballWidth + lineWidth * 2) / 2,
      y: (props.ballWidth + lineWidth * 2) / 2,
    }
    ctx.clearRect(0, 0, props.ballWidth, props.ballWidth)
    ctx.beginPath()
    ctx.arc(centerPos.x, centerPos.y, (props.ballWidth + lineWidth) / 2 , -Math.PI / 2, -Math.PI / 2 + Math.PI * 2 * props.ballInfo.inspectProgress)
    ctx.strokeStyle = isMouseMove.value ? SELECTED_BALL_COLOR : UNSELECTED_BALL_COLOR
    ctx.lineWidth = lineWidth
    ctx.stroke()
    ctx.closePath()
  }
}
watch(() => props.ballWidth, (newVal, oldVal) => {
  if(newVal > 0 && ballCanvas.value){
    initCanvas()
    initRing()
  }
}, {immediate: true})

watch(isMouseMove, (newVal) => {
  initRing()
  if(newVal){
    emit('mouse-move-ball', props.ballInfo)
  } else {
    emit('mouse-leave-ball')
  }
})
onMounted(() => {
  if(ballCanvas.value && props.ballWidth){
    initCanvas()
  }
  initRing()
})
</script>
<template>
  <div class="ball-container" :style="ballContainerStyle">
    <canvas ref="ballCanvas" class="ball-canvas"></canvas>
    <div class="ball" :style="ballStyle" @mousemove="isMouseMove = true;" @mouseleave="isMouseMove = false;" @click="emit('click-ball', props.ballInfo)">
      {{ text }}
    </div>
  </div>
</template>

<style scoped>
@keyframes moveUpDown {
  0% {
    transform: translateY(0);
  }
  50% {
    transform: translateY(3px);
  }
  100% {
    transform: translateY(0);
  }
}

.ball-container{
  position: absolute;
  display: flex;
  justify-content: center;
  align-items: center;
  transition: all 1s ease;
  animation: moveUpDown 1s ease-in-out infinite;
  z-index: 4;
}
.ball-canvas{
  position: absolute;
  top: 0;
  left: 0;
  z-index: 4;
}
.ball {
  position: absolute;
  display: flex;
  justify-content: center;
  align-items: center;
  font-size: 12px;
  cursor: pointer;
  z-index: 4;
  color: #fff;
}
</style>
