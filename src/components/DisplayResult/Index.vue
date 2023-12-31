<script setup lang="ts">
import { onMounted, ref, toValue, computed, watch, onUnmounted } from 'vue'
import { Application, Sprite, Graphics, Container, Ticker } from 'pixi.js'
import StageBgImg from '@/assets/bg.png'
import unselectedBall from '@/assets/unselected.png'
import DisplayBall from './DisplayBall.vue'
import { isEqual, cloneDeep, debounce } from 'lodash'
const emit = defineEmits(['click-ball', 'click-center-ball'])
type DOMRef = HTMLElement | null
type CanvasDomRef = HTMLCanvasElement | null
interface BallItem {
  inspectRound: number
  questionNums: number
  inspectProgress: number
  index?: number
}

interface PieItem {
  inspectRound: number
  departmentNums: number
  index?: number
}

interface Props {
  centerText?: string
  ballList?: Array<BallItem>
  pieList?: Array<PieItem>
  allDepartmentNums?: number
}

interface Point {
  x: number
  y: number
}

const props = withDefaults(defineProps<Props>(), {
  centerText: `十二届总巡察`,
  allDepartmentNums: 120,
  ballList: () => [],
  pieList: () => []
})
const displayContainer = ref<DOMRef>(null)
const wrapper = ref<DOMRef>(null)
const displayCanvas = ref<CanvasDomRef>(null)
const STAGE_BG_COLOR = '#010416'
const CENTER_CIRCLE_BG_COLOR = '#000000'
let containerWidth: number
let containerHeight: number
let app: Application
let ctx: CanvasRenderingContext2D | null
let centerPos = ref<Point>({
  x: 0,
  y: 0
})

const outermostLayerCircleScaleColor = '#091A58'
const outermostLayerCircleScaleWidth = 3
let outermostLayerOfCenterCircleRadius: number
let outermostLayerOfCenterCircleScaleLength: number
interface outermostLayerCircleAnimateProgressInterface {
  graphics: Graphics
  curNum: number
  degUnit: number
}
let outermostLayerCircleAnimateProgress: outermostLayerCircleAnimateProgressInterface = {
  graphics: new Graphics(),
  curNum: 0,
  degUnit: 0
}

const innerLayerCircleScaleColor = '#27498D'
const innerLayerCircleScaleBGColor = '#030A4D'
let innerLayerOfCenterStaticCircleRadius = ref<number>(0)
let innerLayerOfCenterStaticCircleScaleLength: number

let centerBlackCircleRadius: number

const PIE_COLOR_LIST = [
  '#280090',
  '#2800B7',
  '#1409AF',
  '#0253C5',
  '#0D63DB',
  '#147FE8',
  '#208AF2',
  '#23A4E6',
  '#23B7D4',
  '#23C9C6',
  '#28CAB2',
  '#2DC796'
]
const PIE_DEFAULT_COLOR = '#061E4E'
let pieWidth: number
let pieRadius: number
interface Pie {
  startAngle: number
  endAngle: number
  proportion: number
  arcCenterPoint: Point
}
let pieInfoList: Pie[] = []
let lastPieEndAngle = 0
let lastPercent = 0

let ringWidth: number
let ringRadius: number
const CONIC_RING_START_COLOR = 'rgba(1, 10, 32, 0.2)'
const CONIC_RING_END_COLOR = 'rgba(59, 194, 255, 1)'
interface centerRadialGradientProgressInterface {
  arc: Graphics
  curNum: number
  splitNum: number
  radius: number
  middleColor: string[][],
  ticker?: Ticker
}

let centerRadialGradientProgress: centerRadialGradientProgressInterface = {
  arc: new Graphics(),
  curNum: 0,
  splitNum: 0,
  radius: 0,
  middleColor: [[], [], [], []]
}

let ballWidth = ref<number>()
let centerBallWidth = ref<number>()
let percent = ref<number>(0)
let pointerContainerStyle = computed(() => {
  return {
    width: '4px',
    height: `${
      (innerLayerOfCenterStaticCircleRadius.value + innerLayerOfCenterStaticCircleScaleLength / 2) * 2
    }px`,
    transform: `rotate(${360 * percent.value}deg)`
  }
})
let pointerStyle = ref<any>({})

let posList = ref<Array<Point>>()

let centerBallStyle = computed(() => {
  let ballWidthValue = centerBallWidth.value || 0
  let padding = (ballWidthValue - (ballWidthValue / 5) * 3) / 2
  return {
    width: `${ballWidthValue}px`,
    height: `${ballWidthValue}px`,
    top: `${centerPos.value.y}px`,
    left: `${centerPos.value.x}px`,
    marginLeft: `${-ballWidthValue / 2}px`,
    marginTop: `${-ballWidthValue / 2}px`,
    fontSize: `${ballWidthValue / 5}px`,
    backgroundImage: `url(${unselectedBall})`,
    backgroundSize: '100% 100%',
    padding: `0px ${padding}px`
  }
})

/**
 * @description: 获取元素宽高
 * @param {*} el
 * @return {*}
 */
const getElSize = (el: DOMRef) => {
  return {
    width: el?.clientWidth,
    height: el?.clientHeight
  }
}

/**
 * @description: 初始化舞台
 * @return {*}
 */
const initApplication = () => {
  let resolution = window.devicePixelRatio
  app = new Application({
    width: containerWidth / resolution,
    height: containerHeight / resolution,
    background: STAGE_BG_COLOR,
    antialias: true, // 消除锯齿
    resolution: resolution, // 像素设置  模糊的处理
    autoDensity: true // 这属性很关键 模糊的处理
  })
  app.renderer.resize(containerWidth, containerHeight)
  displayContainer.value?.appendChild(app.view as HTMLCanvasElement)
}

/**
 * @description: 初始化canvas
 * @return {*}
 */
const initCanvas = () => {
  let canvas = toValue(displayCanvas)
  if (!canvas) {
    return
  }
  canvas.width = containerWidth
  canvas.height = containerHeight
  ctx = canvas.getContext('2d')
  if (window.devicePixelRatio && ctx) {
    const width = ctx.canvas.width
    const height = ctx.canvas.height
    ctx.canvas.style.width = width + 'px'
    ctx.canvas.style.height = height + 'px'
    ctx.canvas.width = width * window.devicePixelRatio
    ctx.canvas.height = height * window.devicePixelRatio
    ctx.scale(window.devicePixelRatio, window.devicePixelRatio)
  }
  // 逆时针旋转45度
  ctx?.translate(centerPos.value.x, centerPos.value.y)
  ctx?.rotate(-Math.PI / 2)
  ctx?.translate(-centerPos.value.x, -centerPos.value.y)
  canvas.onmousemove = canvasMouseMove
}

/**
 * @description: 初始化背景
 * @return {*}
 */
const initBg = () => {
  const stageBg = Sprite.from(StageBgImg)
  stageBg.width = containerWidth
  stageBg.height = containerHeight
  app.stage.addChild(stageBg)
}

/**
 * @description: 背景的星星类
 * @return {*}
 */
class Star extends Graphics {
  distance: number
  speed: number
  currentSize: number
  fillColor: string
  maxSize: number
  constructor(size: number, color: string) {
    super()
    this.beginFill(color)
    this.drawCircle(0, 0, size)
    this.endFill()

    this.angle = Math.random() * Math.PI * 2 // 随机初始角度
    this.distance = 0
    this.speed = 0
    this.currentSize = size
    this.fillColor = color
    this.maxSize = size
  }
  updateRadius(size: number) {
    this.clear() // 清除之前的绘制
    this.beginFill(this.fillColor)
    this.drawCircle(0, 0, size) // 绘制新的圆形
    this.endFill()
  }
}

const initStarBg = () => {
  const numStars = 150
  let stars: Array<Star> = []

  // 随机生成星星
  for (let i = 0; i < numStars; i++) {
    const size = Math.random() * 2 + 1 // 随机大小
    const color = PIE_COLOR_LIST[Math.round(Math.random() * 12)]
    const star = new Star(size, color)
    star.maxSize = Math.random() * 4 + 2
    star.x = Math.random() * containerWidth
    star.y = Math.random() * containerHeight
    stars.push(star)
    star.distance = Math.sqrt((star.x - centerPos.value.x / 2) ** 2 + (star.y - centerPos.value.y / 2) ** 2)
    app.stage.addChild(star)
  }

  app.ticker.add(() => {
    const radius = Math.max(centerPos.value.x, centerPos.value.y)

    for (let i = 0; i < numStars; i++) {
      const star = stars[i]
      const speed = 0.01
      const distance = 0.1 + star.speed // 随机距离中心的半径
      if (star.distance > radius) {
        star.distance = Math.random() * radius
        star.speed = 0
        star.currentSize = 0
      }

      star.speed += speed // 更新速度
      star.distance += distance // 更新距离

      const x = centerPos.value.x + Math.cos(star.angle) * star.distance
      const y = centerPos.value.y + Math.sin(star.angle) * star.distance

      star.position.set(x, y) // 设置新位置

      // 根据距离中心的距离来调整星星的大小
      const sizeRatio = star.distance / radius
      if (sizeRatio * star.maxSize > star.currentSize) {
        star.currentSize = sizeRatio * star.maxSize // 根据比例计算星星的当前大小
        star.updateRadius(star.currentSize)
      }
      // 设置星星的透明度，随着距离中心的距离增加而逐渐消失
      const alpha = 1 - sizeRatio
      star.alpha = alpha
    }
  })
}

/**
 * @description: 初始化数据
 * @return {*}
 */
const initData = () => {
  let { width = 0, height = 0 } = getElSize(toValue(displayContainer))
  containerWidth = width
  containerHeight = height
  centerPos.value = {
    x: containerWidth / 2,
    y: containerHeight / 2
  }
  outermostLayerOfCenterCircleRadius = Math.ceil((containerHeight * 0.5) / 2)
  outermostLayerOfCenterCircleScaleLength = Math.ceil(outermostLayerOfCenterCircleRadius * 0.1) // 最外层的刻度 暂时就是按半径的10%来算 后面的数据 基本以这个为基数

  innerLayerOfCenterStaticCircleRadius.value =
    outermostLayerOfCenterCircleRadius - outermostLayerOfCenterCircleScaleLength * 2 // 最外层和第二层的静态刻度之间间隙占了半径的5%
  innerLayerOfCenterStaticCircleScaleLength = Math.ceil(outermostLayerOfCenterCircleScaleLength / 3) // 第二层静态刻度 占了 3.3% 可以改 乘系数
  centerBlackCircleRadius =
    innerLayerOfCenterStaticCircleRadius.value - outermostLayerOfCenterCircleScaleLength // 第二层与中间圆的间隙占了5% 黑色背底拿上一层的半径减去5%

  pieWidth = outermostLayerOfCenterCircleScaleLength // 饼图的宽度占了 10% 可以改 乘系数
  pieRadius = centerBlackCircleRadius - pieWidth / 2 - outermostLayerOfCenterCircleScaleLength * 0.5 // 饼的半径在中间

  ringWidth = outermostLayerOfCenterCircleScaleLength * 2 // 渐变色环的宽度半径的 20% 可以改
  ringRadius =
    pieRadius - pieWidth / 2 - outermostLayerOfCenterCircleScaleLength / 2 - ringWidth / 2 // 拿饼的半径一层一层往下算 环的半径也在中间
  let tmpWidth = (ringRadius - ringWidth / 2 - outermostLayerOfCenterCircleScaleLength / 2) * 2
  ballWidth.value = tmpWidth > 90 ? 90 :  tmpWidth// 球的宽度
  centerBallWidth.value = (ringRadius - ringWidth / 2 - outermostLayerOfCenterCircleScaleLength / 2) * 2 // 中心球的宽度
  percent.value =
    props.pieList.reduce((sum, item) => sum + item.departmentNums, 0) / props.allDepartmentNums // 百分比

  pointerStyle.value = {
    height: `${
      innerLayerOfCenterStaticCircleRadius.value -
      ringRadius +
      ringWidth / 2 +
      innerLayerOfCenterStaticCircleScaleLength / 2
    }px`,
    backgroundColor: CONIC_RING_END_COLOR
  }
}

/**
 * @description: 绘制最外层圆
 * @return {*}
 */
const initOutermostLayerCircle = () => {
  const graphics = new Graphics()
  graphics.lineStyle(outermostLayerCircleScaleWidth, outermostLayerCircleScaleColor)
  const degUnit = (Math.PI * 2) / 100 // 3.6度
  for (let i = 0; i < 100; i++) {
    const angle = i * degUnit
    const startX =
    centerPos.value.x +
      (outermostLayerOfCenterCircleRadius - outermostLayerOfCenterCircleScaleLength) *
        Math.cos(angle)
    const startY =
    centerPos.value.y +
      (outermostLayerOfCenterCircleRadius - outermostLayerOfCenterCircleScaleLength) *
        Math.sin(angle)
    const endX = centerPos.value.x + outermostLayerOfCenterCircleRadius * Math.cos(angle)
    const endY = centerPos.value.y + outermostLayerOfCenterCircleRadius * Math.sin(angle)

    graphics.moveTo(startX, startY)
    graphics.lineTo(endX, endY)
  }
  app.stage.addChild(graphics)
}

/**
 * @description: 绘制最外层圆的动画
 * @return {*}
 */
const initOutermostLayerCircleAnimate = () => {
  const graphics = outermostLayerCircleAnimateProgress.graphics
    ? outermostLayerCircleAnimateProgress.graphics
    : new Graphics()
  const degUnit = (Math.PI * 2) / 100 // 3.6度
  const container = new Container()
  app.stage.addChild(container)
  container.x = centerPos.value.x
  container.y = centerPos.value.y
  container.rotation = -Math.PI / 2

  let i = 0
  let curNum = Math.ceil(percent.value * 100)
  let ticker = new Ticker()
  ticker.autoStart = true
  ticker.add(() => {
    if (i > curNum) {
      ticker.stop()
      return false
    }
    const angle = i * degUnit
    graphics.lineStyle(outermostLayerCircleScaleWidth, PIE_COLOR_LIST[Math.floor(i / 10)])
    const startX =
      (outermostLayerOfCenterCircleRadius - outermostLayerOfCenterCircleScaleLength) *
      Math.cos(angle)
    const startY =
      (outermostLayerOfCenterCircleRadius - outermostLayerOfCenterCircleScaleLength) *
      Math.sin(angle)
    const endX = outermostLayerOfCenterCircleRadius * Math.cos(angle)
    const endY = outermostLayerOfCenterCircleRadius * Math.sin(angle)

    graphics.moveTo(startX, startY)
    graphics.lineTo(endX, endY)
    i++
  })
  container.addChild(graphics)
  outermostLayerCircleAnimateProgress = {
    graphics,
    curNum,
    degUnit
  }
}
/**
 * @description: 更新最外层圆的动画
 * @return {*}
 */
const updateOutermostLayerCircleAnimate = () => {
  const { graphics, curNum: num, degUnit } = outermostLayerCircleAnimateProgress
  let i = num
  let curNum = Math.ceil(percent.value * 100)
  let diff = curNum - i
  let fps = Math.floor(60 / diff) // 计算画一个格要经过多少帧
  let count = fps - 1
  let ticker = new Ticker()
  ticker.autoStart = true
  ticker.add(() => {
    count++
    if (count == fps) {
      count = 0
    } else {
      return false
    }
    if (i > curNum) {
      ticker.stop()
      return false
    }
    const angle = i * degUnit
    graphics.lineStyle(outermostLayerCircleScaleWidth, PIE_COLOR_LIST[Math.floor(i / 10)])
    const startX =
      (outermostLayerOfCenterCircleRadius - outermostLayerOfCenterCircleScaleLength) *
      Math.cos(angle)
    const startY =
      (outermostLayerOfCenterCircleRadius - outermostLayerOfCenterCircleScaleLength) *
      Math.sin(angle)
    const endX = outermostLayerOfCenterCircleRadius * Math.cos(angle)
    const endY = outermostLayerOfCenterCircleRadius * Math.sin(angle)

    graphics.moveTo(startX, startY)
    graphics.lineTo(endX, endY)
    i++
  })
  outermostLayerCircleAnimateProgress = {
    graphics,
    curNum,
    degUnit
  }
}
/**
 * @description: 绘制内层静态圆
 * @return {*}
 */
const initInnerLayerStaticCircle = () => {
  const graphics = new Graphics()
  // 绘制内层静态圆 背景色
  graphics.lineStyle(innerLayerOfCenterStaticCircleScaleLength * 2, innerLayerCircleScaleBGColor)
  graphics.drawCircle(
    centerPos.value.x,
    centerPos.value.y,
    innerLayerOfCenterStaticCircleRadius.value - innerLayerOfCenterStaticCircleScaleLength / 2
  )

  graphics.lineStyle(outermostLayerCircleScaleWidth, innerLayerCircleScaleColor)
  const degUnit = (Math.PI * 2) / 60 // 6度
  for (let i = 0; i < 100; i++) {
    const angle = i * degUnit
    const startX =
    centerPos.value.x +
      (innerLayerOfCenterStaticCircleRadius.value - innerLayerOfCenterStaticCircleScaleLength) *
        Math.cos(angle)
    const startY =
    centerPos.value.y +
      (innerLayerOfCenterStaticCircleRadius.value - innerLayerOfCenterStaticCircleScaleLength) *
        Math.sin(angle)
    const endX = centerPos.value.x + innerLayerOfCenterStaticCircleRadius.value * Math.cos(angle)
    const endY = centerPos.value.y + innerLayerOfCenterStaticCircleRadius.value * Math.sin(angle)

    graphics.moveTo(startX, startY)
    graphics.lineTo(endX, endY)
  }
  app.stage.addChild(graphics)
}

/**
 * @description: 绘制中间黑色背景,因为饼图,包括中心 都有黑色的环, 直接在中间填充一个黑色圆就好
 * @return {*}
 */
const initCenterBlackBg = () => {
  const circle = new Graphics()
  circle.beginFill(CENTER_CIRCLE_BG_COLOR)
  circle.drawCircle(centerPos.value.x, centerPos.value.y, centerBlackCircleRadius)
  circle.endFill()
  app.stage.addChild(circle)
}

const initPieBg = () => {
  const pieBg = new Graphics()
  pieBg.lineStyle(pieWidth, PIE_DEFAULT_COLOR)
  pieBg.drawCircle(centerPos.value.x, centerPos.value.y, pieRadius)
  app.stage.addChild(pieBg)
}

const canvasDrawArc = (radius: number, startAngle: number, endAngle: number, color: string) => {
  if (ctx) {
    ctx.beginPath()
    ctx.arc(centerPos.value.x, centerPos.value.y, radius, startAngle, endAngle)
    ctx.strokeStyle = color
    ctx.lineWidth = pieWidth
    ctx.stroke()
    ctx.closePath()
  }
}

/**
 * @description: 初始化饼图
 * @return {*}
 */
const initPie = () => {
  const { allDepartmentNums, pieList } = props
  const intervalNums = pieList.length + 1 // 每个饼图中间的黑色间隔总数
  const intervalAngle = Math.PI / 120
  let startAngle = 0
  const allAngle = Math.PI * 2 - intervalNums * intervalAngle
  pieInfoList = []
  pieList.forEach((pie: PieItem, index: number) => {
    let proportion = pie.departmentNums / allDepartmentNums
    const curAngle = proportion * allAngle
    startAngle = startAngle + intervalAngle
    let endAngle = startAngle + curAngle
    let arcCenterPoint = getCenterPointPos(centerPos.value, { x: centerPos.value.x, y: centerPos.value.y - pieRadius}, startAngle + (endAngle - startAngle) / 2)
    pieInfoList.push({ startAngle, endAngle, proportion, arcCenterPoint })
    startAngle = endAngle
  })

  let drawAngleUnit = Math.PI * 2 * (percent.value - lastPercent) / 60
  let currentEndAngle = lastPieEndAngle
  function drawPie() {
    currentEndAngle+=drawAngleUnit
    if (currentEndAngle >= Math.PI * 2 * percent.value) {
      lastPieEndAngle = currentEndAngle
      lastPercent = percent.value
      return  
    }
    ctx?.clearRect(0, 0, containerWidth, containerHeight)
    for (let i = 0; i < pieInfoList.length; i ++ ){
      let flag = false
      canvasDrawArc(pieRadius, pieInfoList[i].startAngle - intervalAngle, pieInfoList[i].startAngle, '#000')
      let end = pieInfoList[i].endAngle
      if(end > currentEndAngle){
        end = currentEndAngle
        flag = true
      }
      if (pieInfoList[i].startAngle > end){
        break;
      }
      canvasDrawArc(pieRadius, pieInfoList[i].startAngle, end, PIE_COLOR_LIST[i])
      if(flag){
        break;
      }
    }
    window.requestAnimationFrame(drawPie)
  }
  window.requestAnimationFrame(drawPie)

}
const canvasMouseMove = (e: any) => {
  let point = {
    x: e.offsetX,
    y: e.offsetY
  }
  let pointToCenterDistance = Math.sqrt(
    Math.pow(point.x - centerPos.value.x, 2) + Math.pow(point.y - centerPos.value.y, 2)
  ) // 鼠标到圆心的距离
  let radian = -Math.atan2(point.x - centerPos.value.x, point.y - centerPos.value.y) + Math.PI// 计算弧度
  let startDistance = pieRadius - pieWidth / 2
  let endDistance = pieRadius + pieWidth / 2
  if (pointToCenterDistance > outermostLayerOfCenterCircleRadius) {
    return
  }
  if (pointToCenterDistance > startDistance && pointToCenterDistance < endDistance){
    if(pieInfoList.length > 0){
      let index = pieInfoList.findIndex(pie => radian > pie.startAngle && radian < pie.endAngle)
      if(index != -1){
        // console.log('%c point2', 'color: red', point, );
        // console.log('%c arcCenterPoint', 'color: red', arcCenterPoint, );
        // handleShowPieInfoWindow({...props.pieList[index], index}, arcCenterPoint) // 用弹框的位置明显看出中心点的位置来校验算的对不对
        handleShowPieInfoWindow({...props.pieList[index], index}, point)
      } else {
        handleHideBallInfoWindow('pie')
      }
    }
  } else {
    handleHideBallInfoWindow('pie')
  }
}

const getCenterPointPos = (center: Point, centerTop: Point, angle: number) => {
  let x = center.x + (centerTop.x - center.x) * Math.cos(angle) - (centerTop.y - center.y) * Math.sin(angle)
  let y = center.y + (centerTop.x - center.x) * Math.sin(angle) + (centerTop.y - center.y) * Math.cos(angle)
  return { x, y }
}

const drawRing = (
  i: number,
  middleColor: string[][],
  curNum: number,
  radius: number,
  splitNum: number
) => {
  const arc = centerRadialGradientProgress.arc ? centerRadialGradientProgress.arc : new Graphics()
  arc.lineStyle(
    ringWidth,
    `rgba(${middleColor[0][i]}, ${middleColor[1][i]}, ${middleColor[2][i]}, ${middleColor[3][i]})`
  )
  if (i == curNum - 1) {
    arc.arc(
      centerPos.value.x,
      centerPos.value.y,
      radius,
      -Math.PI / 2 + ((Math.PI * 2) / splitNum) * i,
      -Math.PI / 2 + ((Math.PI * 2) / splitNum) * (i + 1)
    )
  } else {
    arc.arc(
      centerPos.value.x,
      centerPos.value.y,
      radius,
      -Math.PI / 2 + ((Math.PI * 2) / splitNum) * i,
      -Math.PI / 2 + ((Math.PI * 2) / splitNum) * (i + 2)
    )
  }
  app.stage.addChild(arc)
}
/**
 * @description: 初始化中间渐变色圆环
 * @return {*}
 */
const initCenterRadialGradient = (radius: number, begin: string, end: string) => {
  const splitNum = 200 // 分割的份数
  // 把颜色分段
  const beginColor = begin
    .slice(5, -1)
    .split(',')
    .map((item) => Number(item))
  const endColor = end
    .slice(5, -1)
    .split(',')
    .map((item) => Number(item))

  // 分段后的颜色储存在这个数组
  const middleColor: string[][] = [[], [], [], []]
  // 循环rgba四种
  beginColor.forEach((item, index) => {
    // 当前的值每段颜色之间的间隔
    const differ = (endColor[index] - item) / (splitNum - 1)
    // 循环分段数的次数
    for (let i = 0; i < splitNum; i++) {
      // 每次加上这个间隔
      middleColor[index].push((item + differ * i).toFixed(2))
    }
  })
  let curNum = Math.floor(percent.value * splitNum)

  let i = 0
  let drawCount = Math.floor(curNum / 60)
  centerRadialGradientProgress.ticker = new Ticker()
  centerRadialGradientProgress.ticker.autoStart = true
  centerRadialGradientProgress.ticker.add(() => {
    if (i == curNum) {
      centerRadialGradientProgress.ticker?.stop()
      return false
    }
    if (drawCount == 1) {
      drawRing(i, middleColor, curNum, radius, splitNum)
      i++
    } else {
      for (let j = 0; j < drawCount; j++) {
        if (i == curNum) {
          centerRadialGradientProgress.ticker?.stop()
          return false
        }
        drawRing(i, middleColor, curNum, radius, splitNum)
        i++
      }
    }
  })
  centerRadialGradientProgress = {
    ...centerRadialGradientProgress,
    arc: centerRadialGradientProgress.arc,
    middleColor,
    splitNum,
    radius,
    curNum
  }
}

const updateCenterRadialGradient = () => {
  const { curNum: num, splitNum, middleColor, radius } = centerRadialGradientProgress
  let i = num - 1
  let curNum = Math.floor(percent.value * splitNum)

  let diff = curNum - i
  let fps = Math.floor(60 / diff) // 计算画一个格要经过多少帧
  let count = fps - 1

  let drawCount = Math.floor(curNum / 60)
  centerRadialGradientProgress.ticker = new Ticker()
  centerRadialGradientProgress.ticker.autoStart = true
  centerRadialGradientProgress.ticker.add(() => {
    count++
    if (count == fps) {
      count = 0
    } else {
      return false
    }
    if (i == curNum) {
      centerRadialGradientProgress.ticker?.stop()
      return false
    }
    if (drawCount == 1) {
      drawRing(i, middleColor, curNum, radius, splitNum)
      i++
    } else {
      for (let j = 0; j < drawCount; j++) {
        if (i == curNum) {
          centerRadialGradientProgress.ticker?.stop()
          return false
        }
        drawRing(i, middleColor, curNum, radius, splitNum)
        i++
      }
    }
  })
  centerRadialGradientProgress = {
    ...centerRadialGradientProgress,
    arc: centerRadialGradientProgress.arc,
    middleColor,
    splitNum,
    radius,
    curNum
  }
}

const generateCircles = (center: Point, minRadius: number, ballWidth: number) => {
  // 输入参数
  const CENTER = center
  const ballNum = props.ballList.length
  // 计算每个扇区角度
  const ANGLE = (Math.PI * 2) / ballNum

  const startAngle = -Math.PI / 4

  // 保存生成坐标
  const POS = []
  for (let i = 0; i < ballNum; i++) {
    const angle = startAngle + ANGLE * i
    let maxRadius = containerHeight / 2 - ballWidth / 2
    if (
      (angle > -(-Math.PI) / 4 && angle < Math.PI / 4) ||
      (angle > (-(-Math.PI) * 3) / 4 && angle < (Math.PI * 5) / 4)
    ) {
      maxRadius = containerWidth / 2 - ballWidth / 2
    }
    const RADIUS = minRadius + (maxRadius - minRadius) * Math.random()
    // 计算圆球角度

    // 初始化坐标
    let x = CENTER.x + RADIUS * Math.cos(angle)
    let y = CENTER.y + RADIUS * Math.sin(angle)

    // 添加坐标
    POS.push({ x, y })
  }
  return POS
}
const infoWindowInfo = ref<any>({})
const handleClickBall = (ballInfo: BallItem) => {
  emit('click-ball', ballInfo)
}

const handleShowBallInfoWindow = (ballInfo: BallItem) => {
  let index = ballInfo.index
  if (index || index === 0) {
    infoWindowInfo.value = {
      ...ballInfo,
      x: posList.value![index].x,
      y: posList.value![index].y,
      visible: true,
      text: `第${ballInfo.inspectRound}轮`,
      type: 'ball'
    }
  }
}

const handleShowPieInfoWindow = (pieInfo: PieItem, pos: Point) => {
  let index = pieInfo.index
  currentShowPieIndex = index! + 1 // 增加体验感,如果悬浮 从悬浮之后的开始继续轮询
  pauseLoopShowPieInfoWindow()
  if (index || index === 0) {
    infoWindowInfo.value = {
      ...pieInfo,
      x: pos.x,
      y: pos.y,
      visible: true,
      text: `第${pieInfo.inspectRound}轮`,
      type: 'pie'
    }
  }
}
const infoWindowStyle = computed(() => {
  if (infoWindowInfo.value.x && infoWindowInfo.value.y && ballWidth.value) {
    // if (infoWindowInfo.value.type == 'ball'){
      return {
        top: `${infoWindowInfo.value.y - ballWidth.value / 2}px`,
        left: `${infoWindowInfo.value.x + ballWidth.value / 2 + 10}px`
      }
    // } else {
    //   return {
    //     top: `${infoWindowInfo.value.y - pieWidth}px`,
    //     left: `${infoWindowInfo.value.x + pieWidth}px`
    //   }
    // }
  } else {
    return {}
  }
})
const handleHideBallInfoWindow = (source: string) => {
  if (source == 'pie' && !intervalTimer){
    openLoopShowPieInfoWindow()
  }
  infoWindowInfo.value = {
    visible: false
  }
}

const infoWindowInfoLoop = ref<any>({})

const handleShowPieInfoLoopWindow = (pieInfo: PieItem, pos: Point) => {
  console.log('%c pieInfo, pos', 'color: red', pieInfo, pos, );
  let index = pieInfo.index
  if (index || index === 0) {
    infoWindowInfoLoop.value = {
      ...pieInfo,
      x: pos.x,
      y: pos.y,
      visible: true,
      text: `第${pieInfo.inspectRound}轮`,
      type: 'pie'
    }
  }
}
const infoWindowLoopStyle = computed(() => {
  if (infoWindowInfoLoop.value.x && infoWindowInfoLoop.value.y) {
      return {
        top: `${infoWindowInfoLoop.value.y}px`,
        left: `${infoWindowInfoLoop.value.x}px`
      }
  } else {
    return {}
  }
})
let intervalTimer: number | null = null;
let currentShowPieIndex = 0;
const beginLoopShowPieInfoWindow = () => {
  intervalTimer = setInterval(() => {
    handleShowPieInfoLoopWindow( { ...props.pieList[currentShowPieIndex], index: currentShowPieIndex }, pieInfoList[currentShowPieIndex].arcCenterPoint)
    currentShowPieIndex++
    if (currentShowPieIndex >= pieInfoList.length){
      currentShowPieIndex = 0
    }
  }, 2000)
}
const pauseLoopShowPieInfoWindow = () => {
  infoWindowInfoLoop.value.visible = false
  clearInterval(intervalTimer!)
  intervalTimer = null
}
const openLoopShowPieInfoWindow = () => {
  beginLoopShowPieInfoWindow()
}

watch(
  () => props.ballList,
  (newVal, oldVal) => {
    if (!newVal || !oldVal) {
      return
    }
    if (ballWidth.value) {
      posList.value = generateCircles(
        centerPos.value,
        outermostLayerOfCenterCircleRadius + ballWidth.value,
        ballWidth.value
      )
    }
  },
  { deep: true }
)

watch(
  () => cloneDeep(props.pieList),
  (newVal, oldVal) => {
    if (!newVal || !oldVal) {
      return
    }
    if (
      newVal.length > oldVal.length &&
      isEqual(newVal.slice(0, oldVal.length), oldVal) &&
      ballWidth.value
    ) {
      // 数据是增加的情况,动画增加
      percent.value =
        props.pieList.reduce((sum, item) => sum + item.departmentNums, 0) / props.allDepartmentNums // 更新指针的变化
      updateOutermostLayerCircleAnimate()
      updateCenterRadialGradient()
      initPie()
    } else {
      // 重新画
      percent.value =
        props.pieList.reduce((sum, item) => sum + item.departmentNums, 0) / props.allDepartmentNums // 更新指针的变化
      // 重新画最外层刻度线
      outermostLayerCircleAnimateProgress.graphics.clear()
      initOutermostLayerCircleAnimate()

      centerRadialGradientProgress.arc.clear()
      initCenterRadialGradient(ringRadius, CONIC_RING_START_COLOR, CONIC_RING_END_COLOR)
      ctx?.clearRect(0, 0, containerWidth, containerHeight)
      lastPercent = 0
      lastPieEndAngle = 0
      initPie()
    }
  },
  { deep: true }
)

let isFirst = true
const handleResize = debounce((entries: Array<ResizeObserverEntry>) => {
  if (isFirst){
    isFirst = false
    return
  }
  initData()
  app.renderer.resize(containerWidth, containerHeight)
  app.stage.removeChildren()
  if(centerRadialGradientProgress.ticker?.started){
    centerRadialGradientProgress.ticker?.stop()
    centerRadialGradientProgress.arc.clear()
  }
  centerRadialGradientProgress = {
    arc: new Graphics(),
    curNum: 0,
    splitNum: 0,
    radius: 0,
    middleColor: [[], [], [], []]
  }
  outermostLayerCircleAnimateProgress = {
    graphics: new Graphics(),
    curNum: 0,
    degUnit: 0
  }
  lastPercent = 0
  lastPieEndAngle = 0
  initCanvas()
  initStarBg()
  initBg()
  initOutermostLayerCircle()
  initOutermostLayerCircleAnimate()
  initInnerLayerStaticCircle()
  initCenterBlackBg()
  initPieBg()
  initPie()
  initCenterRadialGradient(ringRadius, CONIC_RING_START_COLOR, CONIC_RING_END_COLOR)
  if (ballWidth.value) {
    posList.value = generateCircles(
      centerPos.value,
      outermostLayerOfCenterCircleRadius + ballWidth.value,
      ballWidth.value
    )
  }
}, 300)

const resizeObserver = new ResizeObserver(handleResize);


onMounted(() => {
  initData()
  initApplication()
  initCanvas()
  initStarBg()
  initBg()
  initOutermostLayerCircle()
  initOutermostLayerCircleAnimate()
  initInnerLayerStaticCircle()
  initCenterBlackBg()
  initPieBg()
  initPie()
  initCenterRadialGradient(ringRadius, CONIC_RING_START_COLOR, CONIC_RING_END_COLOR)
  beginLoopShowPieInfoWindow()
  if (ballWidth.value) {
    posList.value = generateCircles(
      centerPos.value,
      outermostLayerOfCenterCircleRadius + ballWidth.value,
      ballWidth.value
    )
  }
  resizeObserver.observe(wrapper.value as Element)
})
onUnmounted(() => {
  if (resizeObserver) {
    resizeObserver.disconnect();
  }
});

</script>

<template>
  <div class="wrapper" ref="wrapper">
    <div class="display-container" ref="displayContainer"></div>
    <canvas class="display-canvas" ref="displayCanvas"></canvas>
    <div class="center-ball" :style="centerBallStyle" @click="emit('click-center-ball')">
      {{ centerText }}
    </div>
    <div class="pointer-container" :style="pointerContainerStyle">
      <div class="pointer" :style="pointerStyle"></div>
      <div class="pointer-transparent"></div>
    </div>
    <template v-for="(item, index) in ballList" :key="index">
      <display-ball
        :ball-info="item"
        :ball-width="ballWidth || 0"
        :pos="posList && posList[index]"
        @click-ball="(ballInfo) => handleClickBall({ ...ballInfo, index })"
        @mouse-move-ball="(ballInfo) => handleShowBallInfoWindow({ ...ballInfo, index })"
        @mouse-leave-ball="() => handleHideBallInfoWindow('ball')"
      ></display-ball>
    </template>
    <div class="info-window" v-if="infoWindowInfo.visible" :style="infoWindowStyle">
      <div class="title">{{ infoWindowInfo.text }}</div>
      <template v-if="infoWindowInfo.type == 'ball'">
        <div class="row">
          <div class="label"><span class="circle"></span>问题数</div>
          <div class="content">{{ infoWindowInfo.questionNums }}</div>
        </div>
        <div class="row">
          <div class="label"><span class="circle"></span>百分比</div>
          <div class="content">{{ (infoWindowInfo.inspectProgress * 100).toFixed(0) }}%</div>
        </div>
      </template>
      <template v-else>
        <div class="row">
          <div class="label"><span class="circle"></span>部门数</div>
          <div class="content">{{ infoWindowInfo.departmentNums }}</div>
        </div>
      </template>
    </div>
    <div class="info-window" v-show="infoWindowInfoLoop.visible" :style="infoWindowLoopStyle">
      <div class="title">{{ infoWindowInfoLoop.text }}</div>
      <div class="row">
        <div class="label"><span class="circle"></span>部门数</div>
        <div class="content">{{ infoWindowInfoLoop.departmentNums }}</div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.wrapper {
  width: 100%;
  height: 100%;
  position: relative;
}
.display-container {
  width: 100%;
  height: 100%;
  position: absolute;
  z-index: 1;
}
.display-canvas {
  width: 100%;
  height: 100%;
  position: absolute;
  z-index: 2;
}
.pointer-container {
  position: absolute;
  top: 0;
  left: 0;
  bottom: 0;
  right: 0;
  margin: auto;
  z-index: 4;
  transition: all 1s ease;
}
.pointer-container .pointer {
  width: 100%;
}

.center-ball {
  position: absolute;
  display: flex;
  flex-direction: row;
  flex-wrap: wrap;
  justify-content: center;
  align-items: center;
  font-size: 16px;
  color: #fff;
  z-index: 3;
  box-sizing: border-box;
  cursor: pointer;
}
.info-window {
  position: absolute;
  width: 160px;
  border: 1px solid #53485c;
  border-radius: 4px;
  background-color: #000224;
  box-sizing: border-box;
  padding: 5px 10px;
  z-index: 999;
}
.info-window .title {
  font-size: 16px;
  color: #fff;
}
.info-window .row {
  font-size: 12px;
  color: #fff;
  display: flex;
  align-items: center;
  justify-content: space-between;
}
.info-window .row .circle {
  display: inline-block;
  margin-right: 8px;
  width: 4px;
  height: 4px;
  background-color: #783e03;
  border-radius: 100%;
}

.info-window .row .content {
  color: #783e03;
}
</style>
