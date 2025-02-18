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
import { ref, computed, onMounted } from 'vue'
import { Chart } from 'chart.js/auto'

const PROXY_URL = 'https://cors-anywhere.herokuapp.com/'
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
  props: ['data'],
  mounted() {
    new Chart(this.$el, {
      type: 'line',
      data: {
        labels: this.data.labels,
        datasets: [{
          data: this.data.values,
          borderColor: this.data.change > 0 ? '#4CAF50' : '#f44336',
          borderWidth: 1.5,
          tension: 0.4,
          fill: false
        }]
      },
      options: {
        responsive: true,
        maintainAspectRatio: false,
        plugins: { legend: { display: false } },
        scales: { x: { display: false }, y: { display: false } }
      }
    })
  },
  template: '<canvas class="sparkline" width="100" height="30"></canvas>'
}

const fetchTop40ByVolume = async () => {
  try {
    const response = await fetch(`${PROXY_URL}${BINANCE_URL}/ticker/24hr`)
    const allTickers = await response.json()
    
    return allTickers
      .filter(t => t.symbol.endsWith('USDT'))
      .sort((a, b) => b.quoteVolume - a.quoteVolume)
      .slice(0, 40)
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
      `${PROXY_URL}${BINANCE_URL}/klines?` + 
      new URLSearchParams({
        symbol,
        interval,
        limit
      })
    )
    
    const data = await response.json()
    return data.map(d => parseFloat(d[4]))
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
    const top40 = await fetchTop40ByVolume()
    
    const tickersWithHistory = await Promise.all(
      top40.map(async ticker => {
        const currentPrice = parseFloat(ticker.lastPrice)
        const [sparkline, weekly, monthly, yearly, allTime] = await Promise.all([
          fetchHistoricalPrice(ticker.symbol, '24h'),
          fetchHistoricalPrice(ticker.symbol, '1w'),
          fetchHistoricalPrice(ticker.symbol, '1M'),
          fetchHistoricalPrice(ticker.symbol, '1y'),
          fetchHistoricalPrice(ticker.symbol, 'all')
        ])

        return {
          ...ticker,
          lastPrice: currentPrice,
          sparkline,
          changes: {
            '24h': calculateChange(sparkline[0], currentPrice),
            '1w': calculateChange(weekly[0], currentPrice),
            '1M': calculateChange(monthly[0], currentPrice),
            '1y': calculateChange(yearly[0], currentPrice),
            'all': calculateChange(allTime[0], currentPrice)
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