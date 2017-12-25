<template lang="html">
  <g class="axisNGrid">
  </g>
</template>

<script>
import * as d3 from 'd3'
const DEFAULTTITLE = {
  enabled: true,
  text: '',
  offset: 25,
  textStyle: {
    fill: '#404040',
    fontSize: '12',
    fontWeight: 'bold',
    rotate: 0
  },
  position: 'center'
}

const DEFAULTLINE = {
  // stroke: {string},
  // opacity: {number},
  // lineWidth: {number},
  lineDash: [4, 4]
}

const DEFAULTLABEL = {
  offset: 0,
  textStyle: {
    textAlign: 'center', // 文本对齐方向，可取值为： start middle end
    fill: '#404040', // 文本的颜色
    fontSize: '12', // 文本大小
    fontWeight: 'bold', // 文本粗细
    rotate: 0
  },
  formatter (item, index) {}
}

const DEFAULTGRID = {
  enabled: false,
  lineStyle: {
    stroke: '#d9d9d9', // 网格线的颜色
    opacity: 1, // 网格线的透明度，数值范围为 0 - 1
    width: 1, // 网格线的粗细
    lineDash: [4, 4] // 网格线的虚线配置，第一个参数描述虚线的实部占多少像素，第二个参数描述虚线的虚部占多少像素
  },
  hideFirstLine: false, // 是否隐藏第一条网格线，默认为 true
  hideLastLine: false // 是否隐藏最后一条网格线，默认为 true
}

const props = {
  data: {
    type: Array,
    required: true
  },
  scale: {
    type: Function,
    required: true
  },
  type: {
    type: String,
    required: true
  },
  position: {
    type: String,
    default: 'bottom'
  },
  period: {
    type: String,
    default: 'day'
  },
  tickOrient: {
    type: String,
    default: 'Bottom'
  },
  minGap: {
    type: Number,
    default: 50
  },
  autoCount: {
    type: Boolean,
    default: true
  },
  tickCount: {
    type: Number,
    default: 6
  },
  title: {
    type: Object,
    default: function () { return {} }
  },
  line: {
    type: Object,
    default: function () { return {} }
  },
  label: {
    type: Object,
    default: function () { return {} }
  },
  grid: {
    type: Object,
    default: function () { return {} }
  }
}
export default {
  name: 'axes',
  props,
  created () {},
  mounted () {
    this.createAxis()
  },
  data () {
    const {merge} = this.$lodash
    return {
      titleOpt: merge({}, DEFAULTTITLE, this.title),
      labelOpt: merge({}, DEFAULTLABEL, this.label),
      lineOpt: merge({}, DEFAULTLINE, this.line),
      gridOpt: merge({}, DEFAULTGRID, this.grid)
    }
  },
  computed: {
    dataLength () {
      return this.data.length
    },
    size () {
      const parentEle = this.$el.parentNode
      const width = parentEle.getAttribute('width')
      const height = parentEle.getAttribute('height')
      return {width, height}
    },
    offsetOpt () {
      const {width, height} = this.size
      const offsetOpt = {
        left: 0,
        top: 0
      }
      const { position } = this
      switch (position) {
        case 'bottom':
          offsetOpt.top = height
          break
        case 'right':
          offsetOpt.left = width
          break
        default:
          break
      }
      return offsetOpt
    },
    rangeMax () {
      const range = this.scale.range()
      return d3.max(range)
    },
    maxTickCnt () {
      const {rangeMax, minGap} = this
      return rangeMax / minGap
    },
    tickCnt () {
      const { dataLength, maxTickCnt, autoCount } = this
      let tickCnt = 0
      if (autoCount) {
        tickCnt = dataLength > maxTickCnt ? maxTickCnt : dataLength
      } else {
        tickCnt = this.tickCount
      }
      return tickCnt
    },
    interval () {
      const {dataLength, tickCnt} = this
      return Math.ceil(dataLength / tickCnt)
    },
    tickValues () {
      const {data, interval} = this
      return data.filter((d, i) => i % interval === 0)
    },
    builtInFormatter () {
      const { type, period } = this
      let formatter = null
      if (type === 'time') {
        switch (period) {
          case 'day':
            formatter = d3.timeFormat('%m-%d')
            break
          case 'week':
            formatter = function (d) { return d3.timeFormat('%m-%d')(d) + '当周' }
            break
          case 'month':
            formatter = d3.timeFormat('%Y-%m')
            break
        }
      } else {
        formatter = function (d) { return d }
      }
      return formatter
    },
    titleDegree () {
      const {position} = this
      let degree = 0
      switch (position) {
        case 'left':
          degree = -90
          break
        case 'right':
          degree = 90
          break
        default:
          break
      }
      return degree
    },
    titleTranslate () {
      const {titleOpt, position, size} = this
      const {offset} = titleOpt
      const {width, height} = size
      const tPosition = titleOpt.position
      const factorMap = {
        'start': 0,
        'center': 0.5,
        'end': 1
      }
      const positionFactor = factorMap[tPosition]
      let translate = {
        top: 0,
        left: 0
      }
      switch (position) {
        case 'top':
          translate.left = width * (1 - positionFactor)
          translate.top = -offset
          break
        case 'bottom':
          translate.left = width * (1 - positionFactor)
          translate.top = offset
          break
        case 'left':
          translate.left = -offset
          translate.top = height * positionFactor
          break
        case 'right':
          translate.left = offset
          translate.top = height * positionFactor
          break
        default:
          break
      }
      return translate
    },
    gridSize () {
      const {position, size} = this
      const {width, height} = size
      switch (position) {
        case 'top':
          return -height
        case 'bottom':
          return height
        case 'left':
          return -width
        case 'right':
          return width
        default:
          break
      }
    }
  },
  methods: {
    createAxis () {
      const {titleOpt, gridOpt, gridSize} = this
      const axis = d3.select(this.$el).append('g').attr('class', 'axis')
      const axisFn = this.createAxisFn()
      const fn = this.drawAxis()(axisFn)
      axis.call(fn)
      this.renderLabelStyle()
      if (titleOpt.enabled) {
        this.drawAxisTitle()
      }
      if (gridOpt.enabled) {
        const grid = d3.select(this.$el).append('g').attr('class', 'grid')
        grid.call(fn.tickSize(gridSize).tickFormat(''))
        this.renderGridStyle()
      }
    },
    createAxisFn () {
      const {tickOrient, scale} = this
      let axisFn = d3['axis' + tickOrient](scale)
      return axisFn
    },
    drawAxis () {
      const { type, renderTickPadding, formatTick } = this
      const { flow } = this.$lodash
      let typeFn = null
      switch (type) {
        case 'time':
          typeFn = this.drawTimeAxis
          break
        case 'category':
          typeFn = this.drawCategoryAxis
          break
        default:
          break
      }
      return flow(typeFn, renderTickPadding, formatTick)
    },
    drawTimeAxis (axisFn) {
      const { period, tickValues } = this
      switch (period) {
        case 'day':
          axisFn.tickValues(tickValues)
          break
        case 'week':
          axisFn.ticks(d3.timeMonday)
          break
        case 'month':
          axisFn.ticks(d3.timeMonth)
          break
        default:
          break
      }
      return axisFn
    },
    drawCategoryAxis (axisFn) {
      const { tickValues } = this
      axisFn.tickValues(tickValues)
      return axisFn
    },
    formatTick (axisFn) {
      const { labelOpt, builtInFormatter } = this
      const { format } = labelOpt
      let tickFormatter = null
      if (Object.prototype.toString.call(format) === '[object Function]') {
        tickFormatter = format
      } else {
        tickFormatter = builtInFormatter
      }
      axisFn.tickFormat(tickFormatter)
      return axisFn
    },
    renderTickPadding (axisFn) {
      const { offset } = this.labelOpt
      axisFn.tickPadding(offset)
      return axisFn
    },
    renderLabelStyle () {
      const { offsetOpt, labelOpt } = this
      const { textStyle } = labelOpt
      const { textAlign, fill, fontSize, fontWeight, rotate } = textStyle
      const axis = d3.select(this.$el).select('.axis')
      axis.selectAll('text')
      .attr('fill', fill)
      .style('text-anchor', textAlign)
      .style('font-size', fontSize)
      .style('font-weight', fontWeight)
      .attr('transform', function (d) {
        return 'rotate(' + rotate + ')'
      })
      axis.attr('transform', 'translate(' + offsetOpt.left + ',' + offsetOpt.top + ')')
    },
    drawAxisTitle () {
      const { titleOpt } = this
      const { text } = titleOpt
      const {
        fill,
        fontSize,
        fontWeight,
        rotate
      } = titleOpt.textStyle
      const { titleDegree, titleTranslate } = this
      const axis = d3.select(this.$el).select('.axis')
      axis.append('text')
      .attr('fill', fill)
      .style('font-size', fontSize)
      .style('font-weight', fontWeight)
      .attr('text-anchor', 'middle')
      .attr('transform', `translate(${titleTranslate.left},${titleTranslate.top})rotate(${rotate + titleDegree})`)
      .text(text)
    },
    renderGridStyle () {
      const { gridOpt } = this
      const { lineStyle, hideFirstLine, hideLastLine } = gridOpt
      const { stroke, opacity, width, lineDash } = lineStyle
      const grid = d3.select(this.$el).select('.grid')
      grid.selectAll('line')
      .attr('stroke', stroke)
      .attr('stroke-width', width)
      .attr('stroke-dasharray', lineDash.join(','))
      .style('opacity', opacity)
      const ticks = grid.selectAll('.tick')
      const gridSize = ticks.size()
      const filterFn = function (index) {
        return function (d, i) {
          return i === index
        }
      }
      if (hideFirstLine) {
        ticks.filter(filterFn(0)).remove()
      }
      if (hideLastLine) {
        ticks.filter(filterFn(gridSize - 1)).remove()
      }
    }
  }
}
</script>

<style lang="css">
</style>
