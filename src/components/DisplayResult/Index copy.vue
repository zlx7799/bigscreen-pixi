<script setup lang="ts">
import { onMounted, ref, toValue } from 'vue'
import { Application, Sprite, Graphics, Text, TextStyle, Container, Ticker, Texture} from 'pixi.js'
import gsap from 'gsap'
import StageBgImg from '@/assets/bg.png'
import selectedBall from '@/assets/selected.png'
import unselectedBall from '@/assets/unselected.png'
const emit = defineEmits(['click-ball', 'click-center-ball'])
type DOMRef = HTMLElement | null

interface BallItem {
  inspectRound: number
  questionNums: number
  inspectProgress: number
}

interface PieItem {
  inspectRound: number
  departmentNums: number
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
  centerText: 
`
十二届
总巡察
`,
  allDepartmentNums: 120,
  ballList: () => [
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
    },
  ],
  pieList: () => [
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
    },
    // {
    //   inspectRound: 6,
    //   departmentNums: 10
    // },
    // {
    //   inspectRound: 7,
    //   departmentNums: 10
    // },
    // {
    //   inspectRound: 8,
    //   departmentNums: 10
    // },
    // {
    //   inspectRound: 9,
    //   departmentNums: 10
    // },
    // {
    //   inspectRound: 10,
    //   departmentNums: 10
    // },
    // {
    //   inspectRound: 11,
    //   departmentNums: 10
    // },
    // {
    //   inspectRound: 12,
    //   departmentNums: 10
    // },
  ]
})

const displayContainer = ref<DOMRef>(null)
const STAGE_BG_COLOR = '#010416'
const CENTER_CIRCLE_BG_COLOR = '#000000'
let containerWidth: number
let containerHeight: number
let app: Application
let centerPos: Point

const outermostLayerCircleScaleColor = '#091A58'
const outermostLayerCircleScaleWidth = 3
let outermostLayerOfCenterCircleRadius: number
let outermostLayerOfCenterCircleScaleLength: number

const innerLayerCircleScaleColor = '#27498D'
const innerLayerCircleScaleBGColor = '#030A4D'
let innerLayerOfCenterStaticCircleRadius: number
let innerLayerOfCenterStaticCircleScaleLength: number

let centerBlackCircleRadius: number

const PIE_COLOR_LIST = ['#280090', '#2800B7', '#1409AF', '#0253C5', '#0D63DB', '#147FE8', '#208AF2', '#23A4E6', '#23B7D4', '#23C9C6', '#28CAB2', '#2DC796']
const PIE_DEFAULT_COLOR = '#061E4E'
let pieWidth: number  
let pieRadius: number

let ringWidth: number
let ringRadius: number
const CONIC_RING_START_COLOR = 'rgba(1, 10, 32, 0.2)'
const CONIC_RING_END_COLOR = 'rgba(59, 194, 255, 1)'

let ballwidth: number
const animateTime = 1 // 秒
let percent: number

let ballInfoWindow: InfoWindow
let pieInfoWindow: InfoWindow

const UNSELECTED_BALL_COLOR = '#69D5E7'
const SELECTED_BALL_COLOR = '#F4B23C'
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
  let { width = 0, height = 0 } = getElSize(toValue(displayContainer))
  containerWidth = width
  containerHeight = height
  let resolution = 10 
  app = new Application({
    width: width / resolution,
    height: height / resolution,
    background: STAGE_BG_COLOR,
    antialias: true, // 消除锯齿
    resolution: resolution, // 像素设置  模糊的处理
    autoDensity: true // 这属性很关键 模糊的处理
  })
  app.renderer.resize(width, height);
  displayContainer.value?.appendChild(app.view as HTMLCanvasElement)
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
 * @description: 初始化数据
 * @return {*}
 */
const initData = () => {
  centerPos = {
    x: containerWidth / 2,
    y: containerHeight / 2
  }
  outermostLayerOfCenterCircleRadius = Math.ceil((containerHeight * 0.5) / 2)
  outermostLayerOfCenterCircleScaleLength = Math.ceil(outermostLayerOfCenterCircleRadius * 0.1) // 最外层的刻度 暂时就是按半径的10%来算

  innerLayerOfCenterStaticCircleRadius =
    outermostLayerOfCenterCircleRadius - outermostLayerOfCenterCircleScaleLength * 2
  innerLayerOfCenterStaticCircleScaleLength = Math.ceil(outermostLayerOfCenterCircleScaleLength / 3)

  centerBlackCircleRadius = innerLayerOfCenterStaticCircleRadius - innerLayerOfCenterStaticCircleScaleLength * 3

  pieWidth = outermostLayerOfCenterCircleScaleLength
  pieRadius = centerBlackCircleRadius - pieWidth / 2 - 5

  ringWidth = pieWidth * 1.7
  ringRadius = pieRadius - ringWidth - 15

  ballwidth = ringRadius + 30

  percent = props.pieList.reduce((sum, item) => sum + item.departmentNums, 0) / props.allDepartmentNums 

}

/**
 * @description: 绘制最外层圆
 * @return {*}
 */
const initOutermostLayerCircle = () => {
  const graphics = new Graphics()
  graphics.lineStyle(outermostLayerCircleScaleWidth, outermostLayerCircleScaleColor)
  const degUnit = Math.PI * 2 / 100 // 3.6度
  for (let i = 0; i < 100; i++) {
    const angle = i * degUnit
    const startX = centerPos.x + (outermostLayerOfCenterCircleRadius - outermostLayerOfCenterCircleScaleLength) * Math.cos(angle)
    const startY = centerPos.y + (outermostLayerOfCenterCircleRadius - outermostLayerOfCenterCircleScaleLength) * Math.sin(angle)
    const endX = centerPos.x + outermostLayerOfCenterCircleRadius * Math.cos(angle)
    const endY = centerPos.y + outermostLayerOfCenterCircleRadius * Math.sin(angle)
    
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
  const graphics = new Graphics()
  const degUnit = Math.PI * 2 / 100 // 3.6度
  // for (let i = 0; i < 100; i++) {
  //   const angle = i * degUnit
  //   const startX = centerPos.x + (outermostLayerOfCenterCircleRadius - outermostLayerOfCenterCircleScaleLength) * Math.cos(angle)
  //   const startY = centerPos.y + (outermostLayerOfCenterCircleRadius - outermostLayerOfCenterCircleScaleLength) * Math.sin(angle)
  //   const endX = centerPos.x + outermostLayerOfCenterCircleRadius * Math.cos(angle)
  //   const endY = centerPos.y + outermostLayerOfCenterCircleRadius * Math.sin(angle)
    
  //   graphics.moveTo(startX, startY)
  //   graphics.lineTo(endX, endY)
  // }
  const container = new Container()
  app.stage.addChild(container)
  container.x = centerPos.x
  container.y = centerPos.y
  container.rotation = -Math.PI / 2 

  let i = 0
  let curNum = Math.ceil(percent * 100)
  let ticker = new Ticker()
  ticker.autoStart = true
  ticker.add(() => {
    if(i > curNum) {
      return false
    }
    const angle = i * degUnit
    graphics.lineStyle(outermostLayerCircleScaleWidth, PIE_COLOR_LIST[Math.floor(i / 10)])
    const startX = (outermostLayerOfCenterCircleRadius - outermostLayerOfCenterCircleScaleLength) * Math.cos(angle)
    const startY = (outermostLayerOfCenterCircleRadius - outermostLayerOfCenterCircleScaleLength) * Math.sin(angle)
    const endX = outermostLayerOfCenterCircleRadius * Math.cos(angle)
    const endY = outermostLayerOfCenterCircleRadius * Math.sin(angle)
    
    graphics.moveTo(startX, startY)
    graphics.lineTo(endX, endY)
    i++
  })
  container.addChild(graphics)
}

/**
 * @description: 绘制内层静态圆
 * @return {*}
 */
const initInnerLayerStaticCircle = () => {
  const graphics = new Graphics()
  // 绘制内层静态圆 背景色
  graphics.lineStyle(innerLayerOfCenterStaticCircleScaleLength * 2, innerLayerCircleScaleBGColor)
  graphics.drawCircle(centerPos.x, centerPos.y, innerLayerOfCenterStaticCircleRadius - innerLayerOfCenterStaticCircleScaleLength / 2)

  graphics.lineStyle(outermostLayerCircleScaleWidth, innerLayerCircleScaleColor)
  const degUnit = Math.PI * 2 / 60 // 6度
  for (let i = 0; i < 100; i++) {
    const angle = i * degUnit
    const startX = centerPos.x + (innerLayerOfCenterStaticCircleRadius - innerLayerOfCenterStaticCircleScaleLength) * Math.cos(angle)
    const startY = centerPos.y + (innerLayerOfCenterStaticCircleRadius - innerLayerOfCenterStaticCircleScaleLength) * Math.sin(angle)
    const endX = centerPos.x + innerLayerOfCenterStaticCircleRadius * Math.cos(angle)
    const endY = centerPos.y + innerLayerOfCenterStaticCircleRadius * Math.sin(angle)
    
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
  circle.drawCircle(centerPos.x, centerPos.y, centerBlackCircleRadius)
  circle.endFill()
  app.stage.addChild(circle)
}

/**
 * @description: 初始化饼图
 * @return {*}
 */
const initPie = () => {
  const { allDepartmentNums, pieList } = props
  const intervalNums = pieList.length + 1 // 每个饼图中间的黑色间隔总数
  const intervalAngle = Math.PI / 120
  // let startAngle = -Math.PI / 2
  let startAngle = 0
  const allAngle = Math.PI * 2 - intervalNums * intervalAngle
  interface Pie {
    startAngle: number,
    endAngle: number,
    proportion: number
  }
  // 直接画圆弧从水平方向是0度,从-PI/2开始画的画 有一些奇奇怪怪的问题,借助,container旋转,再去画
  const container = new Container()
  app.stage.addChild(container)
  container.x = centerPos.x
  container.y = centerPos.y
  container.rotation = -Math.PI / 2 

  // function onPointerOver(e) {
  //   console.log('%c e', 'color: red', 1111, e);
  //   // 鼠标悬浮时执行
  //   // pieInfoWindow.show(POS[index].x + 10 + ballwidth / 2, POS[index].y - ballwidth / 2, ball)
  // }

  // function onPointerOut() {
  //   // 鼠标移出时执行
  //   pieInfoWindow.hide()
  // }
  // container.interactive = true
  // container
  //   .on('pointerover', onPointerOver)
  //   .on('pointerout', onPointerOut)
    
  let pieInfoList: Pie[]= []
  pieList.forEach((pie: PieItem, index: number) => {
    let proportion = parseFloat((pie.departmentNums / allDepartmentNums).toFixed(4))
    const curAngle = proportion * allAngle
    startAngle = startAngle + intervalAngle
    const endAngle = startAngle + curAngle
    pieInfoList.push({startAngle, endAngle, proportion})
    // const arc = new Graphics()
    // arc.lineStyle(pieWidth, PIE_COLOR_LIST[index])
    // arc.arc(centerPos.x, centerPos.y, pieRadius, startAngle, endAngle)
    // app.stage.addChild(arc)
    
    startAngle = endAngle
  })

  let tl = gsap.timeline()
  pieInfoList.forEach((pie, index) => {
    const arc = new Graphics()
    tl.to(arc, {
      duration: animateTime * pie.proportion,
      endAngle: pie.endAngle,
      onUpdate: () => {
        arc.lineStyle(pieWidth, PIE_COLOR_LIST[index])
        arc.arc(0, 0, pieRadius, pie.startAngle, (arc as any).endAngle)
      }
    })
    container.addChild(arc)
  })

  if (pieList.length != 12){
    const lastArc = new Graphics()
    lastArc.lineStyle(pieWidth, PIE_DEFAULT_COLOR)
    lastArc.arc(0, 0, pieRadius, startAngle + intervalAngle,  Math.PI * 2)
    container.addChild(lastArc)
  }
}

/**
 * @description: 初始化中间渐变色圆环
 * @return {*}
 */
const initCenterRadialGradient = (radius: number, begin: string, end: string) => {
  const splitNum = 200 // 分割的份数
  // 把颜色分段
  const beginColor = begin.slice(5, -1).split(',').map(item => Number(item))
  const endColor = end.slice(5, -1).split(',').map(item => Number(item))

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
  let curNum = Math.floor(percent * splitNum)
  // for (let i =0; i < curNum; i++ ){
  //   const arc = new Graphics()
  //   arc.lineStyle(ringWidth, `rgba(${middleColor[0][i]}, ${middleColor[1][i]}, ${middleColor[2][i]}, ${middleColor[3][i]})`)
  //   if(i == splitNum - 1){
  //     arc.arc(centerPos.x, centerPos.y, radius, -Math.PI / 2  + Math.PI * 2 / splitNum * i, -Math.PI / 2  + Math.PI * 2  / splitNum * (i + 1))
  //   } else {
  //     arc.arc(centerPos.x, centerPos.y, radius, -Math.PI / 2  + Math.PI * 2 / splitNum * i, -Math.PI / 2  + Math.PI * 2  / splitNum * (i + 2))
  //   }
  //   app.stage.addChild(arc)
  // }
  function drawRing(i: number){
    const arc = new Graphics()
    arc.lineStyle(ringWidth, `rgba(${middleColor[0][i]}, ${middleColor[1][i]}, ${middleColor[2][i]}, ${middleColor[3][i]})`)
    if(i == curNum - 1){
      arc.arc(centerPos.x, centerPos.y, radius, -Math.PI / 2  + Math.PI * 2 / splitNum * i, -Math.PI / 2  + Math.PI * 2  / splitNum * (i + 1))
    } else {
      arc.arc(centerPos.x, centerPos.y, radius, -Math.PI / 2  + Math.PI * 2 / splitNum * i, -Math.PI / 2  + Math.PI * 2  / splitNum * (i + 2))
    }
    app.stage.addChild(arc)
  }
  
  let i = 0
  let drawCount = Math.ceil(curNum / 60)
  let ticker = new Ticker()
  ticker.autoStart = true
  ticker.add(() => {
    if (i == curNum - 2){
      return false
    }
    if(drawCount == 1){
      drawRing(i)
      i++
    } else {
      for (let j = 0; j < drawCount; j++){
        if(i == curNum){
          // app.ticker.stop()
          // break; 
          return false

        }
        drawRing(i)
        i++
      }
    }
  })
}
const initCenterBall = () => {
  const centerBallSprite = Sprite.from(unselectedBall)
  centerBallSprite.width = ballwidth
  centerBallSprite.height = ballwidth
  centerBallSprite.x = centerPos.x
  centerBallSprite.y = centerPos.y
  centerBallSprite.anchor.x = 0.5
  centerBallSprite.anchor.y = 0.5
  app.stage.addChild(centerBallSprite)

  //  中间文字
  const centerTextStyle = new TextStyle({
    fontFamily: '宋体',
    fontSize: ballwidth / 6,
    fill: '#ffffff',
  })
  const centerText = new Text(props.centerText, centerTextStyle)
  centerText.x = centerPos.x
  centerText.y = centerPos.y
  centerText.anchor.set(0.5)
  app.stage.addChild(centerText)
}

const initPointer = () => {
  const line = new Graphics()
  const pointerLength = innerLayerOfCenterStaticCircleRadius + innerLayerOfCenterStaticCircleScaleLength / 2 + 4 - ringRadius + ringWidth / 2
  line.lineStyle(4, CONIC_RING_END_COLOR)
  line.moveTo(0, 0)
  line.lineTo(0, pointerLength)
  const extendProportion = (ringRadius - ringWidth / 2) / pointerLength // 计算一下外面的锚点占的比例
  const renderer = app.renderer
  const texture = renderer.generateTexture(line) // 将Graphics 转化为纹理, 画出来的线不能旋转,所以转换一下
  const sprite = new Sprite(texture)
  sprite.anchor.set(0.5, 1 + extendProportion)
  sprite.position.set(centerPos.x, centerPos.y)
  app.stage.addChild(sprite)

  gsap.to(sprite, {duration: animateTime, percent, onUpdate: () => {
    sprite.rotation = Math.PI * 2 * (sprite as any).percent
  }})
}

const getDistance = (p1: Point, p2: Point) => {
  return Math.sqrt((p1.x - p2.x)**2 + (p1.y - p2.y)**2);
}

const generateCircles = (center: Point, minRadius: number, maxRadius: number, ballwidth: number) => {

  // 输入参数
  const CENTER = center; 

  // 计算每个扇区角度
  const ANGLE = Math.PI * 2 / 12;

  const startAngle = -Math.PI / 2 + ANGLE

  // 保存生成坐标
  const POS = []; 

  console.log('%c maxRadius', 'color: red', maxRadius, minRadius);
  for (let i = 0; i < 12; i++) {
    const RADIUS = minRadius + (maxRadius - minRadius) * Math.random();
    // 计算圆球角度
    // const angle = ANGLE * i + (Math.random() - 0.5) * Math.PI / 3;
    const angle = startAngle + ANGLE * i ;
    
    // 初始化坐标
    let x = CENTER.x + RADIUS * Math.cos(angle);
    let y = CENTER.y + RADIUS * Math.sin(angle);

    // // 检查与上一个圆球距离
    // if(i > 0){
    //   const distToCircle = getDistance(POS[i - 1], {x, y});
    //   if (distToCircle < ballwidth) {
    //     // 调整距离
    //     const adj = (ballwidth * 2) - distToCircle + 10;
    //     x += adj * Math.cos(angle);
    //     y += adj * Math.sin(angle);
    //   }
    // }

    // 添加坐标
    POS.push({x, y});
  }
  return POS;
}

function getRandomInt(min: number, max: number){
  return Math.floor(Math.random()*(max-min+1))+min;
}

const drawInfoWindow = (x: number, y: number, round: PieItem | BallItem) => {
  let roundBox = new Graphics()
  roundBox.lineStyle(2, '#53485C')
  roundBox.beginFill('#000224')
  roundBox.drawRoundedRect(x, y, 200, 150, 10)
  roundBox.endFill()
  app.stage.addChild(roundBox)
}

class InfoWindow {
  app: Application
  roundBox: Graphics
  type: string
  container: Container
  questionNumText?: Text
  percentValueText?: Text
  roundText?: Text
  departmentValueText?: Text
  constructor(
    app: Application,
    type: string
  ) {
    this.app = app
    this.roundBox = new Graphics()
    this.container = new Container()
    this.type = type
    this.init()
  }
  init() {
    let { app, roundBox, container, type } = this
    container.visible = false
    app.stage.addChild(container)
    roundBox.lineStyle(2, '#53485C')
    roundBox.beginFill('#000224')
    if (type == 'ball'){
      roundBox.drawRoundedRect(0, 0, 180, 120, 10)
    } else {
      roundBox.drawRoundedRect(0, 0, 180, 80, 10)
    }
    roundBox.endFill()
    container.addChild(roundBox)
    let textStyle = new TextStyle({
      fontSize: 20,
      fill: '#fff'
    })
    let textStyle2 = new TextStyle({
      fontSize: 16,
      fill: '#fff'
    })
    let textStyle3 = new TextStyle({
      fontSize: 16,
      fill: '#783E03'
    })
    let roundText = new Text('第一轮', textStyle)
    roundText.position.set(20, 18)
    if (type == 'ball'){
      let questionText = new Text('问题数', textStyle2)
      questionText.position.set(20, 46)
      let percentText = new Text('百分比', textStyle2)
      percentText.position.set(20, 72)
      let questionNumText = new Text('100', textStyle3)
      questionNumText.position.set(100, 46)
      let percentValueText = new Text('100%', textStyle3)
      percentValueText.position.set(100, 72)
      this.questionNumText = questionNumText
      this.percentValueText = percentValueText
      container.addChild(questionText)
      container.addChild(percentText)
      container.addChild(questionNumText)
      container.addChild(percentValueText)
    } else {
      let departmentText = new Text('部门数', textStyle2)
      departmentText.position.set(20, 46)
      let departmentValueText = new Text('100', textStyle3)
      departmentValueText.position.set(100, 46)
      container.addChild(departmentText)
      container.addChild(departmentValueText)
      this.departmentValueText = departmentValueText
    }
    this.roundText = roundText
    container.addChild(roundText)
    
  }
  show(x: number, y: number, item: BallItem | PieItem){
    let {container, questionNumText, percentValueText, roundText, departmentValueText, type} = this
    container.position.set(x, y)
    // container.x = x 
    // container.y = y 
    roundText.text = `第${item.inspectRound}轮`
    if(type == 'ball'){
      questionNumText.text = item.questionNums
      percentValueText.text = `${(item.inspectProgress * 100).toFixed(0)}%`
    } else {
      departmentValueText.text = item.departmentValueText
    }
    container.visible = true
  }
  hide() {
    let {container} = this
    container.visible = false
  }
}

/**
 * @description: 画全部球
 * @return {*}
 */
const initAllBall = () => {
  const POS = generateCircles(centerPos, outermostLayerOfCenterCircleRadius + ballwidth, centerPos.y - ballwidth / 2, ballwidth)
  props.ballList.forEach((ball, index) => {
    const container = new Container()
    app.stage.addChild(container)
    container.x = POS[index].x
    container.y = POS[index].y

    const centerBallSprite = Sprite.from(unselectedBall)
    centerBallSprite.width = ballwidth
    centerBallSprite.height = ballwidth
    centerBallSprite.anchor.x = 0.5
    centerBallSprite.anchor.y = 0.5
    container.addChild(centerBallSprite)

    //  中间文字
    const centerTextStyle = new TextStyle({
      fontFamily: '宋体',
      fontSize: ballwidth / 6,
      fill: '#ffffff',
    })
    const centerText = new Text(`第${index +1}轮`, centerTextStyle)
    centerText.anchor.set(0.5)
    container.addChild(centerText)

    // 进度
    const arc = new Graphics()
    arc.lineStyle(4, UNSELECTED_BALL_COLOR)
    arc.arc(0, 0, ballwidth / 2, -Math.PI / 2, -Math.PI / 2 +Math.PI * 2 * ball.inspectProgress)
    container.addChild(arc)

    let ballAnimate = gsap.to(container, {
      y: POS[index].y +  getRandomInt(3, 8), 
      duration: 1,
      repeat: -1, // 无限循环
      yoyo: true, // 往返运动
      ease: 'sine.inOut' // 函数式缓动
    })

    function onPointerOver() {
      // 鼠标悬浮时执行
      ballAnimate.pause()
      arc.lineStyle(4, SELECTED_BALL_COLOR)
      arc.arc(0, 0, ballwidth / 2, -Math.PI / 2, -Math.PI / 2 +Math.PI * 2 * ball.inspectProgress)
      container.addChild(arc)
      centerBallSprite.texture = Texture.from(selectedBall)
      ballInfoWindow.show(POS[index].x + 10 + ballwidth / 2, POS[index].y - ballwidth / 2, ball)
    }

    function onPointerOut() {
      // 鼠标移出时执行
      ballAnimate.resume()
      arc.lineStyle(4, UNSELECTED_BALL_COLOR)
      arc.arc(0, 0, ballwidth / 2, -Math.PI / 2, -Math.PI / 2 +Math.PI * 2 * ball.inspectProgress)
      container.addChild(arc)
      centerBallSprite.texture = Texture.from(unselectedBall)
      ballInfoWindow.hide()
    }

    function onPointClick() {
      // 鼠标移出时执行
      emit('click-ball', {ball})
    }

    container.interactive = true;
    container
      .on('pointerover', onPointerOver)
      .on('pointerout', onPointerOut)
      .on('pointerdown', onPointClick)

  })
}

onMounted(() => {
  initApplication()
  initBg()
  initData()
  initOutermostLayerCircle()
  initInnerLayerStaticCircle()
  initCenterBlackBg()
  initPie()
  initCenterRadialGradient(ringRadius, CONIC_RING_START_COLOR, CONIC_RING_END_COLOR)
  initCenterBall()
  initPointer()
  initOutermostLayerCircleAnimate()
  initAllBall()
  // drawInfoWindow(10, 10, props.ballList[0])
  ballInfoWindow = new InfoWindow(app, 'ball')
  pieInfoWindow = new InfoWindow(app, 'pie')
})
</script>

<template>
  <div class="display-container" ref="displayContainer"></div>
</template>

<style scoped>
  .display-container {
    width: 100%;
    height: 100%;
  }
</style>
