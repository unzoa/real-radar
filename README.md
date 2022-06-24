# real-radar

> 一个真实雷达调节图形化组件。🎃注意🎃：是真实雷达数据！

<img src="https://raw.githubusercontent.com/unzoa/real-radar/main/img.png" width="500px" />

## 安装

```bash
npm i real-radar
```

## props

参数|说明|类型|默认值|可选值
:---|:---|:---|:---|:---|
angle|扫描区域夹角|Number|80|0~360
radius|svg内雷达图形半径|Number|180|1~Infinity
rotate|雷达扫描区域法线，相对正北方向旋转的角度|Number|130|0~360
gap|svg的padding|Number|40|0～radius
height|刻度高度|Number|5|-
radiusReal|真实辐射距离（km），用来标记刻度|Number|3000|-
count|几个刻度、弧|Number|3|0～
color|图谱颜色|String|rgb(38, 222, 117)|-
speed|扫描快慢(ms)|Number|10|-
showScanBg|是否展示扇形|Boolean|false|true
scanBgColor|展示扇形底色|String|#eee|-
radarBg|svg bg color|String|transparent|-

## 注意

1. 标注字体12px

> 标注需要让文字向前移动一般距离，处在刻度中心，就需要知道字体的长度

```js
/**
* 获取文本px宽度
* @param font{String}: normal(正常字体) 字体大小 字体名称
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
```

2. 图形内设置了padding，对应着props参数重gap

3. 扇形均分

> 配置均分最大为30度，想要均分的舒服些，这里取26～30中计算哪个分起来更舒服

```js
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
```
