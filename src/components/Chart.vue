<template>
  <div class="chart">
    <svg width="100%" height="100%">
      <g class="container" transform="translate(50, 50)" width="700" height="300">
        <rect width="700" height="300" fill="#42b983"></rect>
        <axes :scale="scale" :data="categoryData" type="category" position="left" tickOrient="Left" :title="title" :grid="grid"/>
        <axes :scale="timeScale" :data="timeData" type="time" period="day" position="bottom" :title="title"/>
        <!-- <axes :scale="timeScale" :data="timeData" type="time" period="day" position="top" tickOrient="Top" :title="title" :grid="grid"/> -->
        <axes :scale="scale" :data="categoryData" type="category" position="right" tickOrient="Right" :title="title"/>
      </g>
    </svg>
  </div>
</template>

<script>
import * as d3 from 'd3'
import axes from './Axes'
export default {
  name: 'Chart',
  components: { axes },
  created () {},
  mounted () {},
  data () {
    return {
      categoryData: [1, 2, 3, 4, 5],
      timeData: [new Date(2017, 12, 14), new Date(2017, 12, 15), new Date(2017, 12, 16), new Date(2017, 12, 17), new Date(2017, 12, 18), new Date(2017, 12, 19), new Date(2017, 12, 20), new Date(2017, 12, 21)],
      type: 'time',
      period: '',
      title: {
        text: 'value'
      },
      grid: {
        enabled: true
      }
    }
  },
  computed: {
    scale: function () {
      return d3.scaleLinear().domain([0, 5]).range([300, 0])
    },
    timeScale: function () {
      return d3.scaleTime().domain([new Date(2017, 12, 14), new Date(2017, 12, 21)]).range([0, 700])
    }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
.chart {
  width: 800px;
  height: 400px;
  margin: 0 auto;
}
svg {
  background-color: aquamarine;
}
</style>
