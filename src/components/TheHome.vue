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
      let last = this.data[0]?.close
      this.data.forEach(line => {
        const d = line.close
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
      let last = this.data[0]?.close
      let bullish = true
      let directions = [0]
      this.data.forEach(line => {
        const d = line.close
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
      this.data.forEach(line => {
        let d = line.close
        const round = Math.pow(10, Math.floor(Math.log10(d)) - 1)
        d = Math.round(d / round) * round
        levels[d] = levels[d] ? levels[d] + 1 : 1
      })
      return levels
    },
    diffsByUnit () {
      let diffs = {}
      let last = this.data[0]?.close
      let lastDiff = 0
      this.data.forEach(line => {
        const d = line.close
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
      let last = this.data[0]?.close
      let lastDiff = 1
      this.data.forEach(line => {
        const d = line.close
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
      let last = this.data[0]?.close
      let exp = {
        positive: { diff: 0, count: { positive: 0, negative: 0 } },
        negative: { diff: 0, count: { positive: 0, negative: 0 } }
      }
      this.data.forEach(line => {
        const d = line.close
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
    getData () {
      const xhttp = new XMLHttpRequest()
      xhttp.open('GET', `data/ETH.csv`, true)
      xhttp.responseType = 'text'
      xhttp.onload = () => {
        this.data = xhttp.responseText.split(/\r?\n|\r/)
        this.data.shift()
        this.data.pop()
        this.data = this.data.map(d => {
          d = d.split(',')
          return {
            date: d[0],
            open: parseFloat(d[1].replace(/,|"/g, '')),
            high: parseFloat(d[2].replace(/,|"/g, '')),
            low: parseFloat(d[3].replace(/,|"/g, '')),
            close: parseFloat(d[4].replace(/,|"/g, '')),
          }
        })//.reverse()
        this.draw()
        this.bot()
      }
      xhttp.send()
    },
    draw () {
      const c = document.getElementById('canvas')
      let min = Math.log(this.data[0].close)
      let max = Math.log(this.data[0].close)
      let data = this.data.map(line => {
        const d = Math.log(line.close)
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
      console.log('')
      // settings
      let reinvestPercentage = 1 / 100
      let reinvestmentBid = 0
      let standardBid = 100

      let investments = 0
      let investmentsMax = 0
      let assets = 0
      let gains = 0
      let reinvestments = 0
      let fee = 0.1 / 100
      let investmentValue = 0
      let totalInvested = 0
      let buyCounter = 0
      let buyCounterMax = 0
      const acceptHigherMean = true
      const trades = []
      this.data.forEach((line, i) => {
        const price = line.close
        investmentValue = assets * price
        investmentValue -= investmentValue * fee
        const span = 20
        const average = this.getAverage(this.data, span, i)
        const average2 = this.getAverage(this.data, span, i - 10)
        const strength = average / average2
        totalInvested = investments + reinvestments
        if (investmentValue > totalInvested * Math.min(Math.max(1.1, price / average * strength), 1.5)) {
          const gain = investmentValue - totalInvested
          reinvestmentBid += reinvestPercentage * gain
          gains += investmentValue - investments
          investmentsMax = Math.max(investments, investmentsMax)
          buyCounterMax = Math.max(buyCounter, buyCounterMax)
          trades.push({
            date: Date.now(),
            pair: '',
            price,
            value: investmentValue,
            amount: assets,
            type: 'sell',
            gain
          })
          investments = 0
          reinvestments = 0
          assets = 0
          buyCounter = 0
        }

        const mean = totalInvested / assets
        if (acceptHigherMean || !mean || price < mean) {
          let reinvestment = 0
          if (reinvestmentBid && gains > reinvestmentBid) {
            reinvestment = Math.min(reinvestmentBid / 2 * Math.pow(average / price, 5), reinvestmentBid)
            gains -= reinvestment
            reinvestments += reinvestment
          }

          let bid = Math.min(standardBid * Math.pow(average / price, 5), 2 * standardBid) || standardBid
          investments += bid

          const value = bid + reinvestment
          let amount = value / price
          amount = amount - amount * fee
          assets += amount
          buyCounter++
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
          const yearlySell = trades.slice(-365).filter(t => t.type === 'sell')
          const yearlyGain = Math.floor(yearlySell.reduce((a, t) => a + t.gain, 0))
          console.log(
            line.date,
            'gains: ' + yearlyGain + ' ' + Math.round(yearlyGain / investmentsMax * 100) + '%',
            'investment max: ' + Math.floor(investmentsMax),
            'investments: ' + Math.floor(investments) + '+' +  Math.floor(reinvestments),
            'asset value: ' + Math.floor(investmentValue),
            'mean price: ' + Math.floor(totalInvested / assets),
            'sell orders: ' + yearlySell.length
          )
        }
      })
      totalInvested = investments + reinvestments
      console.log(
        'TOTAL => ' + Math.floor(gains + reinvestments),
        'gains: ' + Math.floor(gains) + ' ' + Math.round(gains / investmentsMax * 100) + '%',
        'investment max: ' + Math.floor(investmentsMax),
        buyCounterMax,
        'investments: ' + Math.floor(investments) + '+' +  Math.floor(reinvestments),
        'asset value: ' + Math.floor(investmentValue),
        'mean price: ' + Math.floor(totalInvested / assets),
        'sell orders: ' + trades.filter(t => t.type === 'sell').length,
        'reinvestmentBid: ' + Math.floor(reinvestmentBid)
      )
    },
    getAverage (data, span, index) {
      index = Math.min(data.length, Math.max(0, index + 1))
      const range = data.slice(Math.max(index - span, 0), index)
      const sum = range.reduce((a, price) => a + price.close, 0)
      return sum / range.length
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
