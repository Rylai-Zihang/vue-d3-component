<template lang="html">
  <g class="axisNGrid">
  </g>
</template>

<script>
import * as d3 from 'd3'

/**
 * Default option of line
 * @type {Object}
 */
const DEFAULTLINE = {
  // stroke: {string},
  // opacity: {number},
  // lineWidth: {number},
  lineDash: [4, 4]
}

/**
 * Default option of grid
 * @type {Object}
 */
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

/**
 * Default option of label
 * @type {Object}
 */
const DEFAULTLABEL = {
  offset: 0,
  textStyle: {
    textAlign: 'center', // 文本对齐方向，可取值为： start middle end
    fill: '#404040', // 文本的颜色
    fontSize: '12', // 文本大小
    fontWeight: 'bold', // 文本粗细
    rotate: 0 // 旋转角度
  },
  formatter (item, index) {}
}

/**
 * Default option of grid
 * @type {Object}
 */
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

/**
 * Props of vue instance
 * @type {Object}
 */
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
    /**
     * Data length
     * @return {[type]} [description]
     */
    dataLength () {
      return this.data.length
    },
    /**
     * Size of parent node
     * @return { width, height }
    */
    size () {
      const parentEle = this.$el.parentNode
      const width = parentEle.getAttribute('width')
      const height = parentEle.getAttribute('height')
      return {width, height}
    },
    /**
     * Offset of axis component according to the position option.
     * @return { top, left }
     */
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
    /**
     * Max value of scale range
     * @return {[type]} [description]
     */
    rangeMax () {
      const range = this.scale.range()
      return d3.max(range)
    },
    /**
     * Max count of ticks.
     * Determined by range & min gap.
     * @return {[type]} [description]
     */
    maxTickCnt () {
      const {rangeMax, minGap} = this
      return rangeMax / minGap
    },
    /**
     * Count of ticks in chart
     * @return {[type]} [description]
     */
    tickCnt () {
      const { dataLength, maxTickCnt, autoCount } = this
      let tickCnt = 0
      // if tick count is not assigned
      if (autoCount) {
        tickCnt = dataLength > maxTickCnt ? maxTickCnt : dataLength
      } else { // tick count is determined
        tickCnt = this.tickCount
      }
      return tickCnt
    },
    /**
     * Interval between two displayed ticks
     * @return {[type]} [description]
     */
    interval () {
      const {dataLength, tickCnt} = this
      return Math.ceil(dataLength / tickCnt)
    },
    /**
     * Array of displayed ticks
     * @return {Array} [description]
     */
    tickValues () {
      const {data, interval} = this
      return data.filter((d, i) => i % interval === 0)
    },
    /**
     * Built-in formatter for type if time
     * @return {Function} Tick format funciton
     */
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
    /**
     * Rotate degree of title text.
     * Determined by axis position.
     * @return {[type]} [description]
     */
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
    /**
     * Translate of title text.
     * Determined by title position and axis position option.
     * @return { top, left }
     */
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
    /**
     * Size of grid tick.
     * @return {[type]} [description]
     */
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
    /**
     * Create axis (including title & grid)
     */
    createAxis () {
      const {titleOpt, gridOpt, gridSize} = this
      // Create axis
      const axis = d3.select(this.$el).append('g').attr('class', 'axis')
      const axisFn = this.getFn()
      const fn = this.getAxis()(axisFn)
      axis.call(fn)
      // set label style
      this.setLabelStyle()
      // Create axis title
      if (titleOpt.enabled) {
        this.setAxisTitle()
      }
      // Create axis grid
      if (gridOpt.enabled) {
        const grid = d3.select(this.$el).append('g').attr('class', 'grid')
        grid.call(fn.tickSize(gridSize).tickFormat(''))
        this.setGridStyle()
      }
    },
    /**
     * Get initial axis function
     * @return {Function} [description]
     */
    getFn () {
      const {tickOrient, scale} = this
      let axisFn = d3['axis' + tickOrient](scale)
      return axisFn
    },
    /**
     * Return the composition of three funcitons.
     * Including setTickFormat, setTickPadding and getFn.
     * @return {Function} [description]
     */
    getAxis () {
      const { type, setTickPadding, setTickFormat } = this
      const { flow } = this.$lodash
      let getFn = null
      switch (type) {
        case 'time':
          getFn = this.getTimeAxis
          break
        case 'category':
          getFn = this.getCategoryAxis
          break
        default:
          break
      }
      return flow(getFn, setTickPadding, setTickFormat)
    },
    /**
     * Get axis for time axis
     * @param  {Function} axisFn [description]
     * @return {Function}        [description]
     */
    getTimeAxis (axisFn) {
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
    /**
     * Get axis for category axis
     * @param  {Function} axisFn [description]
     * @return {Function}        [description]
     */
    getCategoryAxis (axisFn) {
      const { tickValues } = this
      axisFn.tickValues(tickValues)
      return axisFn
    },
    /**
     * Format axis using built-in formatter or self-defined formatter
     * @param  {Function} axisFn [description]
     * @return {Function}         [description]
     */
    setTickFormat (axisFn) {
      const { labelOpt, builtInFormatter } = this
      const { format } = labelOpt
      let tickFormatter = null
      // Self-defined formatter
      if (Object.prototype.toString.call(format) === '[object Function]') {
        tickFormatter = format
      } else {
        tickFormatter = builtInFormatter
      }
      axisFn.tickFormat(tickFormatter)
      return axisFn
    },
    /**
     * Set offset between axis and label text
     * @param {Function} axisFn [description]
     */
    setTickPadding (axisFn) {
      const { offset } = this.labelOpt
      axisFn.tickPadding(offset)
      return axisFn
    },
    /**
     * Set label style
    */
    setLabelStyle () {
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
    /**
     * Set axis title
     */
    setAxisTitle () {
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
    /**
     * Set Grid Style
     */
    setGridStyle () {
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
