<template>
  <canvas id="canvas"></canvas>
</template>

<script>
import _ from 'lodash'

function drawLine (context, color, telemetry) {
  context.beginPath()
  telemetry.forEach((d, i) => context.lineTo(d[1] + context.canvas.clientWidth, (context.canvas.clientHeight / 2) + d[0]))
  context.strokeStyle = color
  context.stroke()
  context.closePath()
}

function drawStraightLine (context, color, pos) {
  context.beginPath()
  context.moveTo(0, context.canvas.clientHeight / 2 + pos[0])
  context.lineTo(375, context.canvas.clientHeight / 2 + pos[0])
  context.strokeStyle = color
  context.stroke()
  context.closePath()
}

function drawMarker (context, color, pos) {
  context.beginPath()
  context.moveTo(pos[1] + context.canvas.clientWidth, 0)
  context.lineTo(pos[1] + context.canvas.clientWidth, context.canvas.clientHeight)
  context.strokeStyle = color
  context.stroke()
  context.closePath()
}

function drawCounter (context, color, count) {
  context.font = '32px monospace'
  context.fillText(count, 10, 32)
}

function standardDeviation (avg, values) {
  var squareDiffs = values.map((val) => Math.pow(val - avg, '2'))
  var avgSquareDiff = average(squareDiffs)
  var stdDev = Math.sqrt(avgSquareDiff)
  return stdDev
}

function average (data) {
  var sum = data.reduce((sum, val) => sum + val, 0)
  var avg = sum / data.length
  return avg
}

function getPoint (mean, d, i, len) {
  return [(d - mean) * 30, i - len]
}

function getLine (mean, telemetry) {
  return telemetry.map((d, i) => getPoint(mean, d, i, telemetry.length))
}

export default {
  data () {
    return {
      canvas: {
        height: 600,
        width: 375
      },
      telemetry: [],
      count: 0
    }
  },

  attached () {
    window.addEventListener('devicemotion', this.motion, true)
    this.$watch('telemetry', this.redraw)

    this.$el.width = this.canvas.width * 2
    this.$el.height = this.canvas.height * 2
    this.$el.style.width = `${this.canvas.width}px`
    this.$el.style.height = `${this.canvas.height}px`
    this.context.scale(2, 2)
  },

  computed: {
    context () {
      return this.$el.getContext('2d')
    }
  },

  methods: {
    incr: _.debounce(function () {
      this.count++
    }, 1000),

    motion: function (e) {
      let {x, y, z} = e.acceleration

      let zs = _.map(this.telemetry, 'z')
      let zmean = average(zs)
      let zstd = standardDeviation(zmean, zs)

      let isAnomaly = Math.abs(Math.abs(z) - Math.abs(zmean)) > zstd

      if (isAnomaly) {
        this.incr()
      }

      this.telemetry.push({
        x, y, z, isAnomaly, timeStamp: e.timeStamp
      })

      // limiting to 10000 measurements, for the sake of memory I guess
      this.telemetry = _.takeRight(this.telemetry, 10000)
    },

    redraw: function () {
      window.requestAnimationFrame(() => {
        let zs = _.map(this.telemetry, 'z')
        let zmean = average(zs)
        let zstd = standardDeviation(zmean, zs)

        this.context.clearRect(0, 0, this.canvas.width, this.canvas.height)

        this.telemetry.forEach((t, i) => {
          if (t.isAnomaly) drawMarker(this.context, '#fdd', getPoint(zmean, 0, i, this.telemetry.length))
        })

        drawStraightLine(this.context, '#ccc', getPoint(zmean, [zmean + zstd]))
        drawStraightLine(this.context, '#ccc', getPoint(zmean, [zmean - zstd]))
        drawStraightLine(this.context, '#ddd', getPoint(zmean, [zmean]))

        drawLine(this.context, '#444', getLine(zmean, zs))
        drawCounter(this.context, '#333', this.count)
      })
    }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
h1 {
  color: #42b983;
}
</style>
