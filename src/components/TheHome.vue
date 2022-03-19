<template>
  <div>
    <label>
      Average Span
      <input
        v-model="averageSpan"
        type="number"
        @change="draw"
      >
    </label>
    <canvas
      id="canvas"
      class="canvas">
    </canvas>

    <h3>Reactions</h3>
    <div
      v-for="(direction, index) in reactions"
      :key="index"
    >
      {{ index }}
      <span
        v-for="(reaction, index) in direction"
        :key="'r' + index"
        :class="`chart ${index}`"
        :style="`border-top-width:${reaction}px`"
      >
        {{ reaction }}%
      </span>
    </div>

    <h3>Diffs reactions (by unit)</h3>
    <span
      v-for="diff in diffsByUnit"
      :key="'du' + diff.diff"
      :class="`chart ${diff.reaction > 0 ? 'positive': 'negative'}`"
      :style="`border-top-width:${Math.abs(diff.reaction)}px`"
    >
      {{ diff.diff }}%|
    </span>

    <h3>Diffs reactions (by streak)</h3>
    <span
      v-for="diff in diffsByStreak"
      :key="'ds' + diff.diff"
      :class="`chart ${diff.reaction > 0 ? 'positive': 'negative'}`"
      :style="`border-top-width:${Math.abs(diff.reaction)}px`"
    >
      {{ diff.diff }}%|
    </span>

    <h3>Exponentials</h3>
    <div
      v-for="(reaction, exp) in exponentials"
      :key="'exp' + exp"
    >
      {{ `${exp} ${reaction.diff}% ${reaction.count.positive}% positive ${reaction.count.negative}% negative` }}
    </div>

    <h3>Streaks</h3>
    <span
      v-for="streak in streaks"
      :key="'s' + streak.count"
      class="chart positive"
      :style="`border-top-width:${streak.percentage}px`"
    >
      {{ streak.count }}|
    </span>

    <h3>Levels</h3>
    <div
      v-for="(level, index) in levels"
      :key="'l' + index"
    >
      {{ `${index}: ${level}`}}
    </div>
  </div>
</template>

<script>
export default {
  name: 'TheHome',
  data () {
    return {
      data: [],
      averageSpan: 100
    }
  },
  computed: {
    reactions () {
      let reactions = {
        bull: {
          positive: 0,
          negative: 0,
          neutral: 0
        },
        bear: {
          positive: 0,
          negative: 0,
          neutral: 0
        }
      }
      let lastPercentageDiff = 0
      let last = this.data[0]
      this.data.forEach(d => {
        const percentageDiff = 100 - 100 / last * d
        if (lastPercentageDiff > 0) {
          if (percentageDiff < 0) reactions.bull.negative++
          else if (percentageDiff > 0) reactions.bull.positive++
          else reactions.bull.neutral++
        }
        if (lastPercentageDiff < 0) {
          if (percentageDiff < 0) reactions.bear.negative++
          else if (percentageDiff > 0) reactions.bear.positive++
          else reactions.bear.neutral++
        }
        lastPercentageDiff = percentageDiff
        last = d
      })

      const bullTotal = reactions.bull.positive + reactions.bull.neutral + reactions.bull.negative
      reactions.bull.positive = Math.round(reactions.bull.positive / bullTotal * 100)
      reactions.bull.neutral = Math.round(reactions.bull.neutral / bullTotal * 100)
      reactions.bull.negative = Math.round(reactions.bull.negative / bullTotal * 100)
      const bearTotal = reactions.bear.positive + reactions.bear.neutral + reactions.bear.negative
      reactions.bear.positive = Math.round(reactions.bear.positive / bearTotal * 100)
      reactions.bear.neutral = Math.round(reactions.bear.neutral / bearTotal * 100)
      reactions.bear.negative = Math.round(reactions.bear.negative / bearTotal * 100)
      return reactions
    },
    streaks () {
      let last = this.data[0]
      let bullish = true
      let directions = [0]
      this.data.forEach(d => {
        const percentageDiff = 100 - 100 / last * d

        if (bullish) {
          if (percentageDiff >= 0) directions[directions.length - 1]++
          else {
            bullish = false
            directions.push(0)
          }
        } else if (!bullish) {
          if (percentageDiff <= 0) directions[directions.length - 1]--
          else {
            bullish = true
            directions.push(0)
          }
        }
        last = d
      })

      let streaks = {}
      directions.forEach(s => {
        if (streaks[s]) streaks[s]++
        else streaks[s] = 1
      })
      let total = 0
      for (let s in streaks) total += streaks[s]
      let sortedStreaks = []
      for (let s in streaks) {
        const percentage = Math.round(100 / total * streaks[s] * 10) / 10
        sortedStreaks.push({ count: s, percentage })
      }
      return sortedStreaks.sort((a, b) => parseInt(a.count) < parseInt(b.count))
    },
    levels () {
      let levels = {}
      this.data.forEach(d => {
        const round = Math.pow(10, Math.floor(Math.log10(d)) - 1)
        d = Math.round(d / round) * round
        levels[d] = levels[d] ? levels[d] + 1 : 1
      })
      return levels
    },
    diffsByUnit () {
      let diffs = {}
      let last = this.data[0]
      let lastDiff = 0
      this.data.forEach(d => {
        const diff = Math.round((d - last) / last * 100)
        if (diffs[lastDiff]) diffs[lastDiff].push(diff)
        else diffs[lastDiff] = [diff]
        lastDiff = diff
        last = d
      })

      let sortedDiffs = []
      for (let diff in diffs) {
        const reaction = Math.round(
          diffs[diff].reduce((a, b) => a + b) / diffs[diff].length
        )
        sortedDiffs.push({ diff, reaction })
      }
      return sortedDiffs.sort((a, b) => a.diff - b.diff)
    },
    diffsByStreak () {
      let diffs = {}
      let last = this.data[0]
      let lastDiff = 1
      this.data.forEach(d => {
        const diff = Math.round((d - last) / last * 100)
        if ((diff > 0 && lastDiff > 0) || (diff < 0 && lastDiff < 0)) {
          lastDiff += diff
        } else if ((diff < 0 && lastDiff > 0) || (diff > 0 && lastDiff < 0)) {
          if (diffs[lastDiff]) diffs[lastDiff].push(diff)
          else diffs[lastDiff] = [diff]
          lastDiff = diff
        }
        last = d
      })

      let sortedDiffs = []
      for (let diff in diffs) {
        const reaction = Math.round(
          diffs[diff].reduce((a, b) => a + b) / diffs[diff].length
        )
        sortedDiffs.push({ diff, reaction })
      }
      return sortedDiffs.sort((a, b) => a.diff - b.diff)
    },
    exponentials () {
      let beforeLastDiff = 0
      let lastDiff = 0
      let last = this.data[0]
      let exp = {
        positive: { diff: 0, count: { positive: 0, negative: 0 } },
        negative: { diff: 0, count: { positive: 0, negative: 0 } }
      }
      this.data.forEach(d => {
        const diff = (d - last) / last * 100
        if (beforeLastDiff > 0 && lastDiff > beforeLastDiff) {
          exp.positive.diff += diff
          if (diff > 0) exp.positive.count.positive++
          else if (diff < 0) exp.positive.count.negative++
        } else if (beforeLastDiff < 0 && lastDiff < beforeLastDiff) {
          exp.negative.diff += diff
          if (diff > 0) exp.negative.count.positive++
          else if (diff < 0) exp.negative.count.negative++
        }
        last = d
        beforeLastDiff = lastDiff
        lastDiff = diff
      })

      for (let i in exp) {
        exp[i].diff = Math.round(exp[i].diff)
        const total = exp[i].count.positive + exp[i].count.negative
        for (let j in exp[i].count) {
          exp[i].count[j] = Math.round(
            exp[i].count[j] / total * 100
          )
        }
      }
      return exp
    }
  },
  created () {
    this.getData()
  },
  methods: {
    getData (year) {
      const xhttp = new XMLHttpRequest()
      xhttp.open('GET', `data/1d 2018-2022.csv`, true)
      xhttp.responseType = 'text'
      xhttp.onload = () => {
        this.data = xhttp.responseText.split(/\r?\n|\r/)
        this.data.shift()
        this.data.pop()
        this.data = this.data.map(d => {
          d = d.split(',"')
          return parseFloat(d[1].replace(/,|"/g, ''))
        }).reverse()
        this.draw()
        this.bot()
      }
      xhttp.send()
    },
    draw () {
      const c = document.getElementById('canvas')
      let min = Math.log(this.data[0])
      let max = Math.log(this.data[0])
      let data = this.data.map(d => {
        d = Math.log(d)
        if (d < min) min = d
        if (d > max) max = d
        return d
      })
      const scaleY = c.clientHeight / 100
      const scaleX = c.clientWidth / this.data.length
      const percent = 100 / (max - min) * scaleY
      data = data.map(d => {
        return (d - min) * percent
      })

      c.width = this.data.length * scaleX
      c.height = 100 * scaleY
      const ctx = c.getContext('2d')
      ctx.lineWidth = 1

      this.drawPrices(c, ctx, scaleX, data)
      this.drawAverage(c, ctx, scaleX, data)
    },
    drawPrices (c, ctx, scaleX, data) {
      ctx.beginPath()
      ctx.moveTo(0, c.height - data[0])
      for (let i = 1; i < data.length - 1; i++) {
        ctx.lineTo(i * scaleX, c.height - data[i])
        const d = data[i]
        if (d < data[i - 1]) {
          ctx.strokeStyle = '#f00'
          ctx.stroke()
          if (d < data[i + 1]) {
            ctx.beginPath()
            ctx.moveTo(i * scaleX, c.height - d)
            ctx.strokeStyle = '#0a0'
            ctx.lineTo(data.length * scaleX, c.height - d)
            ctx.stroke()
            ctx.beginPath()
          }
        } else if (d > data[i - 1]) {
          ctx.strokeStyle = '#0f0'
          ctx.stroke()
          if (d > data[i + 1]) {
            ctx.beginPath()
            ctx.moveTo(i * scaleX, c.height - d)
            ctx.strokeStyle = '#a00'
            ctx.lineTo(data.length * scaleX, c.height - d)
            ctx.stroke()
            ctx.beginPath()
          }
        } else {
          ctx.strokeStyle = '#00f'
          ctx.stroke()
        }
        ctx.moveTo(i * scaleX, c.height - d)
      }
      ctx.stroke()
    },
    drawAverage (c, ctx, scaleX, data) {
      ctx.beginPath()
      ctx.moveTo(0, c.height - data[0])
      ctx.strokeStyle = '#00f'
      for (let i = this.averageSpan; i < data.length - 1; i++) {
        let sum = 0
        for (let j = 0; j < this.averageSpan; j++) {
          sum += data[i - j]
        }
        const average = sum / this.averageSpan
        ctx.lineTo(i * scaleX, c.height - average)
        ctx.moveTo(i * scaleX, c.height - average)
      }
      ctx.stroke()
    },
    bot () {
      // settings
      let reinvestPercentage = 0 / 100
      let standardBid = 100

      let investments = 0
      let investmentsTotal = 0
      let assets = 0
      let gains = 0
      let reinvestments = 0
      let fee = 0.01 / 100
      let investmentValue = 0
      const acceptHigherMean = true
      const trades = []
      this.data.forEach((price, i) => {
        investmentValue = assets * price
        investmentValue -= investmentValue * fee
        const average = this.getAverage(this.data, i, 100)
        if (investmentValue > (investments + reinvestments) * Math.max(1.1, Math.pow(price / average, 2))) {
          const gain = investmentValue - investments
          gains += gain
          investmentsTotal += investments
          investments = 0
          reinvestments = 0
          assets = 0
          trades.push({
            date: Date.now(),
            pair: '',
            price,
            value: investmentValue,
            amount: assets,
            type: 'sell',
            gain
          })
        }

        const mean = investments / assets
        if (acceptHigherMean || !mean || price < mean) {
          let reinvestment = 0
          if (price < mean) {
            reinvestment = reinvestPercentage * gains
            gains -= reinvestment
            reinvestments += reinvestment
          }
          let bid = Math.min(standardBid * Math.pow(average / price, 2), 2 * standardBid) || standardBid
          investments += bid
          const value = bid + reinvestment
          let amount = value / price
          amount = amount - amount * fee
          assets += amount
          trades.push({
            date: Date.now(),
            pair: '',
            price,
            value,
            amount,
            type: 'buy'
          })
        }
        
        if (i && i % 365 == 0) {
          const yearlyBuy = trades.slice(-365).filter(t => t.type === 'buy')
          const yearlySell = trades.slice(-365).filter(t => t.type === 'sell')
          const yearlyGain = Math.floor(yearlySell.reduce((a, t) => a + t.gain, 0))
          const yearlyInvestment = yearlyBuy.reduce((a, t) => a + t.value, 0)
          console.log(
            Math.ceil(i / 365),
            'gains: ' + yearlyGain + ' ' + Math.round(yearlyGain / yearlyInvestment * 100) + '%',
            'investments: ' + Math.floor(investments),
            'asset value: ' + Math.floor(investmentValue),
            'mean price: ' + investments / assets,
            'sell orders: ' + yearlySell.length
          )
        }
      })
      console.log(
        'gains: ' + Math.floor(gains) + ' ' + Math.round(gains / investmentsTotal * 100) + '%',
        'investments: ' + Math.floor(investments),
        'asset value: ' + Math.floor(investmentValue),
        'mean price: ' + investments / assets,
        'sell orders: ' + trades.filter(t => t.type === 'sell').length
      )
    },
    getAverage (data, index, span) {
      let sum = 0
      if (index < span) {
        span += (index - span)
      }
      for (let j = index - span + 1; j <= index; j++) {
        sum += data[j]
      }
      return sum / span
    }
  }
}
</script>

<style scoped>
.canvas {
  width: 100%;
  height: 100vh;
  background-color: #666;
}
.chart {
  display: inline-block;
  border-style: solid;
  border-width: 0;
}
.positive {
  border-color: #0f0;
}
.neutral {
  border-color: #ff0;
}
.negative {
  border-color: #f00;
}
</style>
