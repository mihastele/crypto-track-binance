<template>
  <div class="container py-5">
    <div class="crypto-header mb-5 text-center">
      <h1 class="display-4 fw-bold mb-3">📈 Crypto Market Movers</h1>
      <p class="lead text-muted">Tracking Top 40 Cryptocurrencies on Binance</p>
      
      <div class="time-filter mb-4">
        <select v-model="selectedTimeFrame" class="form-select form-select-lg w-25 mx-auto shadow">
          <option value="24h">24 Hours</option>
          <option value="1w">1 Week</option>
          <option value="1M">1 Month</option>
          <option value="1y">1 Year</option>
          <option value="all">All Time</option>
        </select>
      </div>
    </div>

    <div v-if="loading" class="text-center py-5">
      <div class="spinner-border text-primary" role="status">
        <span class="visually-hidden">Loading...</span>
      </div>
      <p class="mt-3 text-muted">Fetching live crypto data...</p>
    </div>

    <div v-else-if="error" class="alert alert-danger mx-auto" style="max-width: 600px">
      ⚠️ {{ error }}
    </div>

    <div v-else class="card shadow-lg">
      <div class="card-body p-0">
        <div class="table-responsive">
          <table class="table table-hover align-middle mb-0">
            <thead class="table-light">
              <tr>
                <th class="ps-4">Coin</th>
                <th>Price (USDT)</th>
                <th>24h Chart</th>
                <th>Change</th>
                <th class="pe-4">Market Cap</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="ticker in sortedTickers" :key="ticker.symbol">
                <td class="ps-4">
                  <div class="d-flex align-items-center">
                    <img :src="`https://coinicons-api.vercel.app/api/icon/${ticker.symbol.replace('USDT', '').toLowerCase()}`" 
                         class="rounded-circle me-3" width="32" height="32"
                         onerror="this.src='https://via.placeholder.com/32'">
                    <div>
                      <div class="fw-bold">{{ ticker.symbol.replace('USDT', '') }}</div>
                      <div class="text-muted small">{{ ticker.symbol }}</div>
                    </div>
                  </div>
                </td>
                
                <td>
                  <span class="fw-bold">${{ parseFloat(ticker.lastPrice).toFixed(2) }}</span>
                </td>

                <td style="width: 120px">
                  <SparklineChart :data="generateSparklineData(ticker)" />
                </td>

                <td>
                  <span :class="['badge', getChangeClass(ticker.changes[selectedTimeFrame])]">
                    <i :class="['fas', ticker.changes[selectedTimeFrame] > 0 ? 'fa-arrow-up' : 'fa-arrow-down']"></i>
                    {{ ticker.changes[selectedTimeFrame].toFixed(2) }}%
                  </span>
                </td>

                <td class="pe-4">
                  ${{ (ticker.quoteVolume / 1000000).toFixed(1) }}M
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, watch } from 'vue'
import { Chart } from 'chart.js/auto'

const BINANCE_URL = 'https://api.binance.com/api/v3'

const tickers = ref([])
const selectedTimeFrame = ref('24h')
const loading = ref(true)
const error = ref(null)

const sortedTickers = computed(() => {
  return [...tickers.value].sort((a, b) => 
    a.changes[selectedTimeFrame.value] - b.changes[selectedTimeFrame.value]
  )
})

const SparklineChart = {
  /* Keep existing SparklineChart implementation */
}

const fetchTop25ByVolume = async () => {
  try {
    const response = await fetch(`${BINANCE_URL}/ticker/24hr`)
    const allTickers = await response.json()
    
    return allTickers
      .filter(t => t.symbol.endsWith('USDT'))
      .sort((a, b) => b.quoteVolume - a.quoteVolume)
      .slice(0, 25) // Changed to top 25
  } catch (err) {
    error.value = 'Failed to fetch initial data'
    throw err
  }
}

const fetchHistoricalPrice = async (symbol, period) => {
  try {
    const intervals = {
      '24h': { interval: '1h', limit: 24 },
      '1w': { interval: '1d', limit: 7 },
      '1M': { interval: '1d', limit: 30 },
      '1y': { interval: '1w', limit: 52 },
      'all': { interval: '1M', limit: 1000 }
    }

    const { interval, limit } = intervals[period]
    const response = await fetch(
      `${BINANCE_URL}/klines?` + 
      new URLSearchParams({ symbol, interval, limit })
    )
    
    return (await response.json()).map(d => parseFloat(d[4]))
  } catch (err) {
    console.error(`Failed to fetch ${period} data for ${symbol}:`, err)
    return []
  }
}

const calculateChange = (historicalPrice, currentPrice) => {
  if (!historicalPrice || historicalPrice === 0) return 0
  return ((currentPrice - historicalPrice) / historicalPrice) * 100
}

onMounted(async () => {
  try {
    const top25 = await fetchTop25ByVolume()
    
    const tickersWithHistory = await Promise.all(
      top25.map(async ticker => {
        const currentPrice = parseFloat(ticker.lastPrice)
        const sparkline = await fetchHistoricalPrice(ticker.symbol, '24h')

        return {
          ...ticker,
          lastPrice: currentPrice,
          sparkline,
          changes: {
            '24h': calculateChange(sparkline[0], currentPrice)
          }
        }
      })
    )

    tickers.value = tickersWithHistory
    loading.value = false
  } catch (err) {
    error.value = 'Failed to load market data. Please try again later.'
    loading.value = false
  }
})

const generateSparklineData = (ticker) => {
  // Get data for current time frame
  const timeFrameData = {
    '24h': ticker.sparkline,
    '1w': ticker.weeklySparkline || [],
    '1M': ticker.monthlySparkline || [],
    '1y': ticker.yearlySparkline || [],
    'all': ticker.allTimeSparkline || []
  }

  const values = timeFrameData[selectedTimeFrame.value]
  const labels = Array.from({ length: values.length }, (_, i) => i + 1)
  
  return {
    labels,
    values,
    change: ticker.changes[selectedTimeFrame.value]
  }
}


watch(selectedTimeFrame, async (newTimeFrame) => {
  if (newTimeFrame === '24h') return

  const needsFetch = tickers.value.some(t => 
    !t[`${newTimeFrame}Sparkline`] && !t.changes[newTimeFrame]
  )

  if (!needsFetch) return

  try {
    loading.value = true
    const updated = await Promise.all(
      tickers.value.map(async ticker => {
        if (ticker[`${newTimeFrame}Sparkline`]) return ticker
        
        const historical = await fetchHistoricalPrice(ticker.symbol, newTimeFrame)
        return {
          ...ticker,
          [`${newTimeFrame}Sparkline`]: historical,
          changes: {
            ...ticker.changes,
            [newTimeFrame]: calculateChange(historical[0], ticker.lastPrice)
          }
        }
      })
    )
    
    tickers.value = updated
  } catch (err) {
    error.value = `Failed to load ${newTimeFrame} data`
  } finally {
    loading.value = false
  }
})

const getChangeClass = (change) => change > 0 ? 'positive' : 'negative'
</script>

<style>
@import url('https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600;700&display=swap');
@import url('https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css');

body {
  font-family: 'Inter', sans-serif;
  background-color: #f8f9fa;
}

.crypto-header {
  background: linear-gradient(135deg, #6366f1 0%, #a855f7 100%);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
}

.table-hover tbody tr:hover {
  background-color: rgba(99, 102, 241, 0.05);
}

.badge {
  padding: 0.5em 0.8em;
  border-radius: 8px;
  font-size: 0.9em;
  transition: all 0.2s ease;
}

.badge.positive {
  background-color: #e8f5e9;
  color: #4CAF50;
}

.badge.negative {
  background-color: #ffebee;
  color: #f44336;
}

.sparkline {
  max-width: 100px;
  height: 30px !important;
}

.form-select {
  transition: all 0.3s ease;
  border: 2px solid #e0e0e0;
}

.form-select:focus {
  border-color: #6366f1;
  box-shadow: 0 0 0 3px rgba(99, 102, 241, 0.25);
}

.card {
  border-radius: 1rem;
  overflow: hidden;
}

.table th {
  font-weight: 600;
  text-transform: uppercase;
  font-size: 0.85em;
  letter-spacing: 0.5px;
}
</style>