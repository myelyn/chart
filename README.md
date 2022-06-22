### 词云

![wordCloud](https://raw.githubusercontent.com/myelyn/charts/master/images/wordCloud.png)

###### 背景
客户期望将四个维度的标签显示在一个词云图里，UI出的设计图如上。

查了一些资料，对比之后决定用echarts的wordcloud插件来做，但是这个插件本身并不支持分维度展示。

为了实现这个效果，开动脑筋想了想办法，技术上其实不难，但是用到了一些小技巧，还挺好玩的，记录一下。

###### 思路
1.每个维度做成一个独立的词云，总共就是四个小词云。
2.把人物图按颜色区块切成四块，作为小词云的蒙板，每个小词云通过定位拼在一起。
3.这些蒙板的作用类似于ps中的蒙板，只用来限定词云渲染的形状，并不能当作背景来显示。
4.将整个人物背景当通过echarts的异型柱状图（pictorialBar）定位拼上去。

###### 步骤
1.首先做准备工作: 切图。这个切图非常重要，必须亲自切。
![wordCloud](https://raw.githubusercontent.com/myelyn/charts/master/images/wordcloud0.png)

如图所示，用标尺把人物图分成四个小区块，每个小区块都包含了一个完整的色块，用多边形套索将色块勾选出来放在一个单独的图层，整个人物也是一个独立图层，共5个图层。
![wordCloud](https://raw.githubusercontent.com/myelyn/charts/master/images/wordcloud1.png)
先切第一个区块，把另外四个图层隐藏，用裁切工具裁出第一块区域，然后存储为web通用格式。
![wordCloud](https://raw.githubusercontent.com/myelyn/charts/master/images/wordcloud2.png)
这里注意，由于这个切图是作为蒙板使用，只是用来限定词云的渲染形状，这张图并不会显示在图表里，所以这张图的画质并不重要，格式我选择了png-8，颜色改为2，这样即使失真严重也不会影响词云显示效果，并且图片尺寸只有1K多。为了方便，我把这张图转成了base64，建立一个js数组将base64保存起来备用。
![wordCloud](https://raw.githubusercontent.com/myelyn/charts/master/images/wordcloud3.png)
保存好第一张图后，通过历史记录撤销刚才的裁切操作，返回到上一步。将第二个色块的图层显示出来，其它的图层隐藏，切出第二个色块。
用同样的方法，切出第三、第四个色块。
最后再保存一张完整的人物图。
完整的人物图是作为异型柱状图显示出来，所以画质要求高一点。这里可以选择png-8，颜色32，具体配置可以根据实际显示效果来调整。
全部5张图需要用同一个psd源文件切出来，这样在用echarts渲染时定位就很方便了。

2.图片全部准备好之后，还要计算一下每个小色块顶部距离整个图片顶部的百分比，以及每个小色块高度所占整个图片的百分比。同样用数组保存起来。
``` 
  // imageInfo.js

  // 实际项目中是根据sex属性的不同值（men、women、unknow）来动态显示不同的图片，所以这里存了三套不同的位置信息
  export const maskPosition = {
    men: [
      {
        height: '33%',
        top: '0'
      }, {
        height: '32.6%',
        top: '23%'
      }, {
        height: '28.2%',
        top: '46.5%'
      }, {
        height: '30%',
        top: '70%'
      }
    ],
    women: [
      {
        height: '33%',
        top: '0'
      }, {
        height: '32%',
        top: '24.9%'
      }, {
        height: '31%',
        top: '43.5%'
      }, {
        height: '28.5%',
        top: '71.5%'
      }
    ],
    unknow: [
      {
        height: '34%',
        top: '0'
      }, {
        height: '33%',
        top: '23%'
      }, {
        height: '35%',
        top: '40%'
      }, {
        height: '30.5%',
        top: '69%'
      }
    ]
  }
  // 省略
  export const imgSrcList = {...}
  export const bgImgSrc = {...}
``` 

3. 生成echarts option
 
```
// echartOpt.js
// imgSrcList: 四张蒙板图的base64数组; bgImgSrc: 整个人物图的base64字符串; maskPosition: 蒙板图的位置信息; wordCloudData: 从后端获取到的四个维度的标签数组
 export const getWordCloudOptions = (imgSrcList, bgImgSrc, maskPosition, wordCloudData) => {
  const arrAverage = []
  // 算一下每个维度的标签对应数值的平均值，因为根据设计图，高于词云中平均值的标签文字opacity显示为1，低于平均值的opacity为0.7
  wordCloudData.forEach(obj => {
    const sum = obj.data.reduce((result, cur) => {
      result += cur.value
      return result
    }, 0)
    arrAverage.push(sum / obj.data.length)
  })
  return new Promise((resolve) => {
    const category = ['']
    // 首先定义整个词云的背景
    const series = [{
      type: 'pictorialBar', 
      symbol: 'image://' + bgImgSrc,
      barWidth: '100%',
      symbolRepeat: false,
      z: -180,
      data: [1],
      tooltip: {
        show: false
      }
    }]
    // 词云公共配置
    const comWordCloudOpts = {
      type: 'wordCloud',
      gridSize: 1,
      sizeRange: [8, 14],
      rotationRange: [0, 0],
      drawOutOfBound: false,
      width: '100%',
      left: 'center'
    }
    let count = 0
    // 遍历四张蒙板图，生成对应的option
    imgSrcList.forEach((s, i) => {
      const img = new Image()
      img.src = s
      series.unshift({
        ...comWordCloudOpts,
        textStyle: {
          normal: {
            fontWeight: 600,
            color: function (v) {
              return v.value > arrAverage[i] ? '#ffffff' : 'rgba(255,255,255,0.7)'
            }
          },
          emphasis: {
            color: '#3C66FA',
            textBorderColor: '#fff',
            textBorderWidth: 4
          }
        },
        maskImage: img,
        data: wordCloudData[i]['data'],
        ...maskPosition[i]
      })
      img.onload = () => {
        count++
        if (count === imgSrcList.length) {
          const chartOptions = {
            xAxis: {
              show: false
            },
            yAxis: {
              data: category,
              show: false
            },
            tooltip: {
              show: true
            },
            grid: [{
              left: 0,
              bottom: 0,
              top: 0,
              right: 0
            }],
            series: series
          }
          resolve(chartOptions)
        }
      }
    })
  })
}

```

4.引用插件，生成词云，完成。

```
// wordCloud.vue

// template
<div id="wordcloud"></div>

// script
import echarts from 'echarts'
import 'echarts-wordcloud/dist/echarts-wordcloud.min.js'
import { bgImgSrc, maskImgSrc, maskPosition } from './_source/imageInfo.js'
import { getWordCloudOptions } from './_source/echartOpt.js'

this.wordCloudChart = echarts.init(document.getElementById('wordcloud'))
getWordCloudOptions(maskImgSrc[this.sex], bgImgSrc[this.sex], maskPosition[this.sex], this.tagList).then(options => {
  this.wordCloudChart.setOption(options)
  this.wordCloudChart.on('click', () => {
    this.$emit('tagclick')
  })
})

```

### 链路图

![links](https://raw.githubusercontent.com/myelyn/charts/master/images/links.png)

这个图的UI改了好几版，最早的那版我是用echarts的graph（关系图）来做的，后来客户突然发现实际情况更复杂，于是改成了这样。这就不适合用关系图了，我决定自己画一下。

自己画的话，想过两种方案，一种是用svg，另一种是直接用css写布局和动画，再用js画路径。

这个图我感觉用第二种会更简单，于是开始写。

这个图层级和节点都是动态的，遍历接口数据生成层级和节点，层级用flex纵向排列，节点用flex默认排列，设置justify-content为space-around，便完成了基本的布局。

```
<template>
  <div class="chart">
    <ul class="stage-container">
      <li class="stage" v-for="stage in chartData" :style="{height: 100/chartData.length+'%'}">
        <span class="stage-name">{{stage.name}}</span>
        <span class="dot"></span>
        <span class="dashed"></span>
        <ul class="target-container">
          <li v-for="target in stage.targets" :class="['target-item', `target-item-${target.level}`, target.reach ? 'target-reach' : '']">
            <span class="target-name">{{target.name}}</span>
          </li>
        </ul>
      </li>
    </ul>
  </div>
</template>
```

接下来是节点高亮和连线。接口返回的数据用了一个reach字段来标记达成的节点，也就是需要连线并且高亮显示的节点，连线上的箭头方向默认从低层级指向高层级，同一层级内如果有多个高亮节点，则再返回一个数组来告知连线的顺序。

为了方便后面画线时js获取数据，在template渲染时，将这个连线顺序简单计算后保存在data-draw-index中，将连线上面显示的percent保存在data-percent中。

同时，在vue模板中，判断这个节点有没有reach字段，如果有，则增加class名称'target-reach'，然后在dom渲染完成之后，也就是在vue的nextTick回调函数中，通过这个class名称查找到所有高亮节点，同时根据dataset.drawIndex来获取连线的顺序。

这样就得到了这些需要连线的节点的Dom元素组成的数组，并且可以按连线顺序来排列。

遍历这个Dom元素数组, 用getBoundingClientRect()得到每一个Dom元素的位置信息，用pos保存，同时通过dataset.percent得到这个元素的连线上显示的百分比，用percent保存，组成新的nodeList。如下：

```
  const nodeList = []  
  const domList = document.querySelectorAll('.target-percent')
  domList.forEach(n => {
    nodeList.push({
      pos: n.getBoundingClientRect(),
      percent: n.dataset.percent
    })
  })
```

循环nodeList，将第0和1项、第1和2项...调用drawLine实现两两连线
```
  let prev = null
  nodeList.forEach(cur => {
    if (prev) {
      // 实际项目中这里会分两种情况，虚线连接会有percent，实现连线没有
      this.drawLine(prev, cur, 'dashed', cur.percent)
    }
    prev = cur
  })

```

下面实现画线的方法，因为通过prev和cur传过来了两个点的坐标，用三角函数就可以算出连线的旋转角度和长度。
```

  drawLine (prev, cur, type, percent) {
    const line = document.createElement('div')
    line.setAttribute('class', `chart-links-stage-line chart-links-stage-line-${type}`)

    const [startX, startY, endX, endY] = [prev.x + 1 / 2 * prev.width, prev.y + 1 / 2 * prev.height, cur.x + 1 / 2 * cur.width, cur.y + 1 / 2 * cur.height]

    const l = Math.sqrt(Math.pow((endX - startX), 2) + Math.pow((endY - startY), 2))
    const dx = Math.abs(endX - startX)
    const dy = Math.abs(endY - startY)

    let r = Math.atan(dy / dx) * 180 / Math.PI
    if (endX < startX) {
      r = 180 - r
    }
    line.style.left = startX + 'px'
    line.style.top = document.documentElement.scrollTop + startY + 'px'
    line.style.width = l + 'px'
    line.style.transform = `rotate(${r}deg)`
    const isVertical = Math.abs(r - 90) < 1
    if (type === 'dashed') {
      line.innerHTML = `<span style="transform: rotate(${isVertical ? -90 : r > 90 ? 180 : 0}deg) translate(${isVertical ? 'calc(50% - 5px), calc(-50% + 3px)' : r > 90 ? '50%,-1px' : '-50%, calc(-50% - 10px)'})">${formatNum(+percent, 2, true)}</span>`
    }
    document.body.appendChild(line)
  }
```
最后，监听一下resize行为，触发resize时将线段销毁重新渲染。

### 漏斗

![funnel](https://raw.githubusercontent.com/myelyn/charts/master/images/funnel.png)

这个图选择了用原生svg来实现，涉及到的svg知识很简单，就是一些方块和渐变。


```
<style>
  svg {
    border: 1px solid blue;
  }
  .svg1 {
      width: calc(20vw);
      height: calc(20vh);
  }
</style>
```

```
<div style="width: 600px; height:600px;">
  <svg class="svg1" viewBox="0 0 100 200" preserveAspectRatio="none">
    <defs>
    <linearGradient id="funnel-bar1">
      <stop offset="0%" stop-color="#FFAE64" />
      <stop offset="100%" stop-color="#FFE588" />
    </linearGradient>
    <linearGradient id="shadow-funnel" x1="0" y1="0" x2="0" y2="1">
      <stop offset="0%" stop-color="#8CCFFF" stop-opacity=".2" />
      <stop offset="100%" stop-color="rgba(165,144, 251, 0)" />
    </linearGradient>
    </defs>
    <g id="funnel">
      <rect x="0" y="0" width="100" height="14" fill="url(#funnel-bar1)"></rect>
      <path d="M0 14 H100 L70 100 H0 Z" stroke="none" fill="url(#shadow-funnel)"/>
      <rect x="0" y="100" width="70" height="14" fill="url(#funnel-bar1)"></rect>
      <path d="M0 114 H70 L30 200 H0 Z" stroke="none" fill="url(#shadow-funnel)"/>
      <rect x="0" y="200" width="30" height="15" fill="url(#funnel-bar1)"></rect>
    </g>
</svg>
<svg width="200" height="200">
  <use href="#funnel"></use>
</svg>
</div>

```

这是做预研时简单画的一个图，项目中并没有用viewBox，因为会导致文字变形。目前没有找到好的解决办法。有时间记得再研究一下。

实际项目中，根据数据动态计算了这些方块和路径的坐标，然后做了一些自适应的处理。

