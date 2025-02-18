<template>
  <div class="container">
    <h1>Crypto Market Movers (Top 40)</h1>
    <div class="controls">
      <select v-model="selectedTimeFrame" class="time-select">
        <option value="24h">24 Hours</option>
        <option value="1w">1 Week</option>
        <option value="1M">1 Month</option>
        <option value="1y">1 Year</option>
        <option value="all">All Time</option>
      </select>
    </div>

    <div v-if="loading" class="loading">Loading data...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <table v-else>
      <thead>
        <tr>
          <th>Coin</th>
          <th>Price (USDT)</th>
          <th>Change (%)</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="ticker in sortedTickers" :key="ticker.symbol">
          <td>{{ ticker.symbol.replace('USDT', '') }}</td>
          <td>${{ parseFloat(ticker.lastPrice).toFixed(2) }}</td>
          <td :class="getChangeClass(getChange(ticker, selectedTimeFrame))">
            {{ getChange(ticker, selectedTimeFrame).toFixed(2) }}%
          </td>
        </tr>
      </tbody>
    </table>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue'

const tickers = ref([])
const selectedTimeFrame = ref('24h')
const loading = ref(true)
const error = ref(null)

const sortedTickers = computed(() => {
  return [...tickers.value].sort((a, b) => {
    const aChange = getChange(a, selectedTimeFrame.value)
    const bChange = getChange(b, selectedTimeFrame.value)
    return aChange - bChange
  })
})

onMounted(async () => {
  try {
    const tickersResponse = await fetch('https://api.binance.com/api/v3/ticker/24hr')
    const allTickers = await tickersResponse.json()
    
    const filtered = allTickers
      .filter(t => t.symbol.endsWith('USDT'))
      .sort((a, b) => b.quoteVolume - a.quoteVolume)
      .slice(0, 40)

    for (const ticker of filtered) {
      const symbol = ticker.symbol
      const currentPrice = parseFloat(ticker.lastPrice)
      
      const [weekly, monthly, yearly, allTime] = await Promise.all([
        fetchHistoricalData(symbol, '1w'),
        fetchHistoricalData(symbol, '1M'),
        fetchHistoricalData(symbol, '1y'),
        fetchHistoricalData(symbol, 'all')
      ])

      ticker.weekChange = calculateChange(weekly, currentPrice)
      ticker.monthChange = calculateChange(monthly, currentPrice)
      ticker.yearChange = calculateChange(yearly, currentPrice)
      ticker.allTimeChange = calculateChange(allTime, currentPrice)
    }

    tickers.value = filtered
    loading.value = false
  } catch (err) {
    error.value = 'Failed to fetch data. Please try again later.'
    loading.value = false
    console.error(err)
  }
})

async function fetchHistoricalData(symbol, period) {
  try {
    const intervalMap = {
      '1w': '1d',
      '1M': '1d',
      '1y': '1w',
      'all': '1d'
    }

    let url = `https://api.binance.com/api/v3/klines?symbol=${symbol}&interval=${intervalMap[period]}&limit=1`

    if (period !== 'all') {
      const now = Date.now()
      const startTime = now - ({
        '1w': 7 * 24 * 60 * 60 * 1000,
        '1M': 30 * 24 * 60 * 60 * 1000,
        '1y': 365 * 24 * 60 * 60 * 1000
      }[period])
      url += `&startTime=${startTime}`
    }

    const response = await fetch(url)
    const data = await response.json()
    return data.length > 0 ? parseFloat(data[0][4]) : null
  } catch (err) {
    console.error(`Error fetching ${period} data for ${symbol}:`, err)
    return null
  }
}

function calculateChange(historicalPrice, currentPrice) {
  if (!historicalPrice || historicalPrice === 0) return 0
  return ((currentPrice - historicalPrice) / historicalPrice) * 100
}

function getChange(ticker, timeFrame) {
  switch (timeFrame) {
    case '24h': return parseFloat(ticker.priceChangePercent)
    case '1w': return ticker.weekChange
    case '1M': return ticker.monthChange
    case '1y': return ticker.yearChange
    case 'all': return ticker.allTimeChange
    default: return 0
  }
}

function getChangeClass(change) {
  return change > 0 ? 'positive' : 'negative'
}
</script>

<style>
/* Same styles as previous version */
.container {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
  font-family: Arial, sans-serif;
}

.controls {
  margin: 20px 0;
}

.time-select {
  padding: 8px;
  font-size: 16px;
}

table {
  width: 100%;
  border-collapse: collapse;
  margin-top: 20px;
}

th, td {
  padding: 12px;
  text-align: left;
  border-bottom: 1px solid #ddd;
}

th {
  background-color: #f5f5f5;
}

.positive {
  color: #4CAF50;
}

.negative {
  color: #f44336;
}

.loading, .error {
  padding: 20px;
  text-align: center;
  font-size: 18px;
}

.error {
  color: #f44336;
}
</style>