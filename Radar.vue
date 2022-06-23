<template>
  <div class="radar-wrapper">
     <svg
      xmlns="http://www.w3.org/2000/svg"
      x="0" y="0"
      :width="radius * 2 + gap * 2"
      :height="radius * 2 + gap * 2"
      :viewBox="`0 0 ${radius * 2 + gap * 2} ${radius * 2 + gap * 2}`"
      :style="{ background: radarBg }" >

      <!-- 整个雷达的辐射集合 -->
      <g :transform="`rotate(${rotate - 90 - angle / 2}, ${radius + gap}, ${radius + gap})`" >
        <!-- 扇形扫描区域-->
        <circle
          v-if="showScanBg"
          id="circle-1"
          :r="radius / 2"
          :cx="radius + gap"
          :cy="radius + gap"
          fill="transparent"
          :stroke="scanBgColor"
          :stroke-width="radius"
          :stroke-dasharray="radius * 3.1415926"
          :stroke-dashoffset="radius * 3.1415926 / 360 * (360 - angle)"
          />

        <!-- 刻度标注 -->
        <g id="callout-up">
          <g
            v-for="(i, j) in callout"
            :key="j">
            <line
              :x1="i.x1"
              :y1="i.y1"
              :x2="i.x2"
              :y2="i.y2"
              :stroke="color" stroke-width="2" stroke-linecap="butt"
              />
            <text
              :x="i.tx"
              :y="i.ty"
              :fill="color"
              :font-size="fontSize">{{ i.t }}</text>
          </g>
        </g>

        <!-- 刻度标注 -->
        <g id="callout-down"
          v-show="angle < 360 && angle > 0"
          :transform="`rotate(${angle}, ${radius + gap}, ${radius + gap})`">
          <g
            v-for="(i, j) in callout"
            :key="j">
            <line
              :x1="i.x1"
              :y1="i.y1"
              :x2="i.x2"
              :y2="i.y2_down"
              :stroke="color" stroke-width="2" stroke-linecap="butt"
              />
            <text
              :x="i.tx_down"
              :y="i.ty_down"
              :transform="`rotate(180, ${i.tx_down}, ${i.ty_down})`"
              :fill="color"
              :font-size="fontSize">{{ i.t }}</text>
          </g>
        </g>

        <!-- 圆弧扫描 -->
        <path
          v-for="(i, j) in arcPaths"
          :key="'arc' + j"
          :d="i"
          fill="none"
          :stroke="color"
          stroke-width="2"
          />

        <!-- 线段扫描 -->
        <path
          v-for="(i, j) in scanPath"
          :key="'scan' + j"
          :d="i"
          fill="none"
          :stroke="color"
          stroke-width="2"
          />

        <!-- 扫描度数 -->
        <text
          v-for="(i, j) in scanAngleText"
          :key="j"
          :x="i.x"
          :y="i.y"
          :fill="color"
          :transform="`rotate(${i.angle - 90}, ${i.x}, ${i.y})`"
          :font-size="fontSize">{{ i.t }}</text>

        <!-- 扫描频率线 -->
        <path
          :d="scanLine"
          fill="none"
          :stroke="color"
          stroke-width="4"/>
      </g>
    </svg>
  </div>
</template>

<script>
/* eslint-disable */
/**
* 获取文本px宽度
* @param font{String}: 字体样式
**/
const fontSize = '12px'
String.prototype.pxWidth = function (font = `normal ${fontSize} SiYuan`) {
  // re-use canvas object for better performance
  const canvas = String.prototype.pxWidth.canvas ||
    (String.prototype.pxWidth.canvas = document.createElement("canvas"))
  const context = canvas.getContext('2d')

  font && (context.font = font)
  const metrics = context.measureText(this)

  return metrics.width;
}

export default {
  name: 'Radar',

  props: {
    // 雷达扫描区域法线，相对正北方向旋转的角度
    rotate: { default: 130, type: Number },

    // 扫描区域夹角
    angle: { default: 80, type: Number },

    // 真实辐射距离（km），用来标记刻度
    radiusReal: { default: 3000, type: Number },

    // svg内雷达图形半径
    radius: { default: 180, type: Number },

    // svg的padding
    gap: { default: 40, type: Number },

    // 刻度高度
    height: { default: 5, type: Number },

    // 几个刻度， 需要考虑公里数文字显示是否被遮挡
    count: { default: 3, type: Number },

    // 图谱颜色
    color: { default: 'rgb(38, 222, 117)', type: String },

    // 扫描快慢
    speed: { default: 10, type: Number },

    // 是否展示扇形
    showScanBg: { default: false, type: Boolean },
    scanBgColor: { default: '#eee', type: String },

    // svg bg color
    radarBg: { default: 'transparent', type: String }
  },

  data () {
    return {
      scanLine: '', // 扫描线
      timer: {}, // 扫描定时器
      fontSize
    }
  },

  computed: {
    // 辐射公里数刻度
    callout () {
      const {
        radiusReal,
        radius,
        gap,
        height,
        count
      } = this

      const theR = radius + gap

      // output 可控个数坐标段数
      const calloutRPer = Math.ceil(radiusReal / count)
      const calloutPer = radius / count
      const calloutStart = theR

      const arr = []
      Array.from( { length: count } ).forEach( (i, j) => {
        const x = calloutStart + calloutPer * j
        const t = (calloutRPer * j).toFixed(0) + '公里'
        arr.push( {
          x1: x,
          y1: theR,
          x2: x,
          y2: theR - height,
          y2_down: theR + height,
          tx: x - String(t).pxWidth() / 2,
          tx_down: x + String(t).pxWidth() / 2,
          ty: theR - height * 1.5,
          ty_down: theR + height * 1.5,
          t
        } )
      } )
      arr.shift() // 不显示第一个

      return arr
    },

    // 弧线
    arcPaths () {
      const {
        radius,
        gap,
        count,
        angle
      } = this

      const origin = radius + gap

      const arr = []
      Array.from( { length: count + 1 } ).forEach( (i, j) => {
        arr.push( this.arcPathConvert( { origin, r: radius / count * j, angle } ) )
      } )
      // 不画第一个
      arr.shift()

      return arr
    },

    // 计算将扇形分割成几份
    scanPathCount () {
      const { angle } = this
      // angle / i(26~30), 取最小余数的i
      const items = [26, 27, 28, 29, 30]
      let b = 30 // 默认最大余数
      items.forEach(i => {
        if (angle % i < b) {
          b = i
        }
      })

      return Math.floor(angle / b)
    },

    // 绘制扫描半径的填充
    scanPath () {
      const {
        radius,
        gap,
        angle,
        height
      } = this

      const origin = radius + gap
      const r = radius + height
      const count = this.scanPathCount
      const perAngle = angle / count
      const arr = []
      // 首
      arr.push( this.scanPathConvert( { r, origin, angle: 0 } ) )

      Array.from( { length: count } ).forEach( (i, j) => {
        arr.push( this.scanPathConvert( { r, origin, angle: perAngle * j } ) )
      } )

      // 尾
      arr.push( this.scanPathConvert( { r, origin, angle } ) )

      return arr
    },

    scanAngleText () {
      const {
        radius,
        angle,
        rotate
      } = this

      const startAngle = rotate - angle / 2
      const count = this.scanPathCount
      const perAngle = angle / count
      const arr = []
      Array.from( { length: count } ).forEach( (i, j) => {
        arr.push( this.scanAngleTextConvert( {
          r: radius,
          angle: perAngle * j,
          t: (startAngle + perAngle * j).toFixed(1) + '度'
          } )
        )
      } )

      // 手动添加结尾
      arr.push( this.scanAngleTextConvert( {
        r: radius,
        angle,
        t: startAngle + Number(angle) + '度'
      } ) )


      // 满一圈去掉第一个
      if (angle === 360) {
        arr.shift()
      }

      return arr
    }
  },

  mounted () {
    this.scanAction()
  },

  methods: {
    // 开始扫描
    scanAction () {
      clearInterval(this.timer)

      let angle = 0
      this.timer = setInterval(() => {
        if (angle <= this.angle) {
          this.scanLine = this.scanPathConvert( {
            angle,
            r: this.radius,
            origin: this.radius + this.gap
          } )
          angle++
        } else {
          angle = 0
        }
      }, this.speed)
    },

    // **数据编辑**
    // ----------------------------------------------------------------

    arcPathConvert ({ r: theR, origin, angle }) {
      // 转换极坐标
      let x = 0
      let y = 0
      if (angle <= 90) {
        const rad = angle * (Math.PI / 180)
        x = ( theR - Math.abs(Math.cos( rad ) * theR) ) * -1
        y = Math.abs(Math.sin( rad ) * theR ) * 1
      }

      if (angle > 90 && angle <= 180) {
        const rad = (angle - 90) * (Math.PI / 180)
        x = -( Math.abs(Math.sin( rad ) * theR ) * 1 + theR )
        y = Math.abs(Math.cos( rad ) * theR ) * 1
      }

      if (angle > 180 && angle <= 270) {
        const rad = (angle - 180) * (Math.PI / 180)
        x = -( Math.abs( Math.cos( rad ) * theR ) * 1 + theR )
        y = Math.abs( Math.sin( rad ) * theR ) * -1
      }

      if (angle > 270) {
        const rad = (angle - 270) * (Math.PI / 180)
        x = (theR - Math.abs( Math.sin( rad ) * theR ) ) * -1
        y = Math.abs(Math.cos( rad ) * theR ) * -1
      }

      if (angle === 360) {
        // 此时，需要画两个半圆合并
        return `M${origin},${origin}
          m${theR},0
          a${theR},${theR} 0 0,1 ${-theR * 2},${0}
          a${theR},${theR} 0 0,1 ${theR * 2},${0}
          `
      }

      // deg > 180 ？大弧 1 ： 小弧 0
      const largeArc = Number(angle > 180)
      return `M${origin},${origin}
        m${theR},0
        a${theR},${theR} 0 ${largeArc},1
        ${x},${y}`
    },

    scanPathConvert ({ r: theR, origin, angle }) {
      const rad = angle * (Math.PI / 180)

      // 转换极坐标
      const x = Math.abs( Math.cos(rad) * theR ) * (angle > 90 && angle < 270 ? -1 : 1)
      const y = Math.abs( Math.sin(rad) * theR ) * (angle > 180 ? -1 : 1)

      return `M${origin},${origin} l${x},${y}`
    },

    scanAngleTextConvert ({ r: theR, angle, t }) {
      const angleTextOffset = 18 // 偏移度数文字距离最外弧度的距离
      const rIn = theR + angleTextOffset
      const rOut = theR + this.gap
      const rad = angle * (Math.PI / 180)

      const x = Math.abs( Math.cos(rad) * rIn) * (angle > 90 && angle < 270 ? -1 : 1) + rOut
      const y = Math.abs( Math.sin(rad) * rIn) * (angle > 180 ? -1 : 1) + rOut

      // 计算文字相对刻度偏移
      const tL = String(t).pxWidth() // 文字长度
      const l2 = Math.tan(rad) * rIn
      const l3 = l2 + tL / 2
      const y1 = Math.cos(rad) * l3

      const l5 = Math.sin(rad) * tL / 2
      const l6 = Math.cos(rad) * rIn
      const x1 = l6 - l5

      return {
        x: x1 + rOut,
        y: y1 + rOut,
        t,
        angle
      }
    }
  }
}
</script>

<style scoped lang="scss">
  .radar-wrapper {
    position: relative;
    display: inline-block;
    font-size: 0;
  }
</style>
