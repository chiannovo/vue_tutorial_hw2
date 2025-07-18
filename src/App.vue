<template>
  <div class="app">
    <header class="header">
      <h1>
        <i class="bike-icon">🚲</i>
        YouBike 站點查詢系統
      </h1>
    </header>

    <main class="main-content">
      <div class="search-section">
        <div class="search-controls">
          <div class="input-group">
            <input 
              v-model="searchKeyword" 
              type="text" 
              placeholder="搜尋站點名稱..."
              class="search-input"
              @input="filterStations"
            >
            <button @click="clearSearch" class="clear-btn" v-if="searchKeyword">
              ✕
            </button>
          </div>
          
          <div class="filter-group">
            <select v-model="selectedArea" @change="filterStations" class="area-select">
              <option value="">所有區域</option>
              <option v-for="area in areas" :key="area" :value="area">
                {{ area }}
              </option>
            </select>
            
            <button @click="refreshData" class="refresh-btn" :disabled="loading">
              <span v-if="loading">⟳</span>
              <span v-else>🔄</span>
              {{ loading ? '更新中...' : '重新整理' }}
            </button>
          </div>
        </div>

        <div class="stats">
          <div class="stat-item">
            <span class="stat-number">{{ filteredStations.length }}</span>
            <span class="stat-label">站點數量</span>
          </div>
          <div class="stat-item">
            <span class="stat-number">{{ totalBikes }}</span>
            <span class="stat-label">可借車輛</span>
          </div>
          <div class="stat-item">
            <span class="stat-number">{{ totalSpaces }}</span>
            <span class="stat-label">可還空位</span>
          </div>
        </div>
      </div>

      <div class="stations-section">
        <div v-if="loading" class="loading">
          <div class="loading-spinner"></div>
          <p>載入站點資料中...</p>
        </div>

        <div v-else-if="error" class="error">
          <p>{{ error }}</p>
          <button @click="fetchStations" class="retry-btn">重新載入</button>
        </div>

        <div v-else-if="filteredStations.length === 0" class="no-results">
          <p>找不到符合條件的站點</p>
        </div>

        <div v-else class="stations-grid">
          <div 
            v-for="station in paginatedStations" 
            :key="station.sno"
            class="station-card"
            :class="getStationClass(station)"
          >
            <div class="station-header">
              <h3 class="station-name">{{ station.sna }}</h3>
              <span class="station-status" :class="getStatusClass(station.act)">
                {{ station.act === '1' ? '營運中' : '暫停' }}
              </span>
            </div>
            
            <div class="station-info">
              <div class="info-row">
                <span class="info-label">區域：</span>
                <span class="info-value">{{ station.sarea }}</span>
              </div>
              <div class="info-row">
                <span class="info-label">地址：</span>
                <span class="info-value">{{ station.ar }}</span>
              </div>
            </div>

            <div class="station-bikes">
              <div class="bike-info">
                <div class="bike-count available">
                  <span class="count">{{ station.sbi }}</span>
                  <span class="label">可借</span>
                </div>
                <div class="bike-count spaces">
                  <span class="count">{{ station.bemp }}</span>
                  <span class="label">可還</span>
                </div>
                <div class="bike-count total">
                  <span class="count">{{ station.tot }}</span>
                  <span class="label">總數</span>
                </div>
              </div>
            </div>

            <div class="station-footer">
              <small class="update-time">
                更新時間：{{ formatTime(station.mday) }}
              </small>
            </div>
          </div>
        </div>

        <div v-if="totalPages > 1" class="pagination">
          <button 
            @click="currentPage = 1" 
            :disabled="currentPage === 1"
            class="page-btn"
          >
            ⟸
          </button>
          <button 
            @click="currentPage--" 
            :disabled="currentPage === 1"
            class="page-btn"
          >
            ⟨
          </button>
          
          <span class="page-info">
            第 {{ currentPage }} 頁，共 {{ totalPages }} 頁
          </span>
          
          <button 
            @click="currentPage++" 
            :disabled="currentPage === totalPages"
            class="page-btn"
          >
            ⟩
          </button>
          <button 
            @click="currentPage = totalPages" 
            :disabled="currentPage === totalPages"
            class="page-btn"
          >
            ⟹
          </button>
        </div>
      </div>
    </main>
  </div>
</template>

<script>
import { ref, computed, onMounted, watch } from 'vue'
import axios from 'axios'

export default {
  name: 'App',
  setup() {
    // 響應式數據
    const stations = ref([])
    const searchKeyword = ref('')
    const selectedArea = ref('')
    const loading = ref(false)
    const error = ref('')
    const currentPage = ref(1)
    const itemsPerPage = 12

    // 計算屬性
    const areas = computed(() => {
      const areaSet = new Set(stations.value.map(station => station.sarea))
      return Array.from(areaSet).sort()
    })

    const filteredStations = computed(() => {
      let filtered = stations.value

      if (searchKeyword.value) {
        const keyword = searchKeyword.value.toLowerCase()
        filtered = filtered.filter(station => 
          station.sna.toLowerCase().includes(keyword) ||
          station.ar.toLowerCase().includes(keyword)
        )
      }

      if (selectedArea.value) {
        filtered = filtered.filter(station => station.sarea === selectedArea.value)
      }

      return filtered
    })

    const totalPages = computed(() => {
      return Math.ceil(filteredStations.value.length / itemsPerPage)
    })

    const paginatedStations = computed(() => {
      const start = (currentPage.value - 1) * itemsPerPage
      const end = start + itemsPerPage
      return filteredStations.value.slice(start, end)
    })

    const totalBikes = computed(() => {
      return filteredStations.value.reduce((sum, station) => sum + parseInt(station.sbi), 0)
    })

    const totalSpaces = computed(() => {
      return filteredStations.value.reduce((sum, station) => sum + parseInt(station.bemp), 0)
    })

    // 方法
    const fetchStations = async () => {
      loading.value = true
      error.value = ''
      
      try {
        const response = await axios.get(
          'https://data.taipei/api/v1/dataset/c6bc8aed-557d-41d5-bfb1-8da24f78f2fb?scope=resourceAquire'
        )
        stations.value = response.data.result.results
      } catch (err) {
        error.value = '無法載入站點資料，請檢查網路連線'
        console.error('API錯誤:', err)
      } finally {
        loading.value = false
      }
    }

    const filterStations = () => {
      currentPage.value = 1
    }

    const clearSearch = () => {
      searchKeyword.value = ''
      filterStations()
    }

    const refreshData = () => {
      fetchStations()
    }

    const getStationClass = (station) => {
      if (station.act !== '1') return 'inactive'
      if (parseInt(station.sbi) === 0) return 'no-bikes'
      if (parseInt(station.bemp) === 0) return 'no-spaces'
      return 'active'
    }

    const getStatusClass = (act) => {
      return act === '1' ? 'status-active' : 'status-inactive'
    }

    const formatTime = (timestamp) => {
      const date = new Date(timestamp)
      return date.toLocaleString('zh-TW', {
        month: '2-digit',
        day: '2-digit',
        hour: '2-digit',
        minute: '2-digit'
      })
    }

    // 監聽篩選變化，重置頁碼
    watch([searchKeyword, selectedArea], () => {
      currentPage.value = 1
    })

    // 初始化
    onMounted(() => {
      fetchStations()
    })

    return {
      stations,
      searchKeyword,
      selectedArea,
      loading,
      error,
      currentPage,
      areas,
      filteredStations,
      totalPages,
      paginatedStations,
      totalBikes,
      totalSpaces,
      fetchStations,
      filterStations,
      clearSearch,
      refreshData,
      getStationClass,
      getStatusClass,
      formatTime
    }
  }
}
</script>

<style scoped>
.app {
  min-height: 100vh;
  padding: 20px;
}

.header {
  text-align: center;
  margin-bottom: 30px;
}

.header h1 {
  color: white;
  font-size: 2.5em;
  text-shadow: 2px 2px 4px rgba(0,0,0,0.3);
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 15px;
}

.bike-icon {
  font-size: 1.2em;
}

.main-content {
  max-width: 1200px;
  margin: 0 auto;
}

.search-section {
  background: rgba(255,255,255,0.95);
  padding: 25px;
  border-radius: 15px;
  box-shadow: 0 8px 32px rgba(0,0,0,0.1);
  margin-bottom: 25px;
}

.search-controls {
  display: flex;
  flex-wrap: wrap;
  gap: 15px;
  margin-bottom: 20px;
}

.input-group {
  position: relative;
  flex: 1;
  min-width: 250px;
}

.search-input {
  width: 100%;
  padding: 12px 40px 12px 15px;
  border: 2px solid #e1e5e9;
  border-radius: 8px;
  font-size: 16px;
  transition: border-color 0.3s;
}

.search-input:focus {
  outline: none;
  border-color: #667eea;
}

.clear-btn {
  position: absolute;
  right: 10px;
  top: 50%;
  transform: translateY(-50%);
  background: none;
  border: none;
  color: #999;
  cursor: pointer;
  font-size: 18px;
}

.filter-group {
  display: flex;
  gap: 10px;
}

.area-select {
  padding: 12px 15px;
  border: 2px solid #e1e5e9;
  border-radius: 8px;
  font-size: 16px;
  min-width: 140px;
}

.refresh-btn {
  padding: 12px 20px;
  background: #667eea;
  color: white;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  font-size: 16px;
  transition: background 0.3s;
  display: flex;
  align-items: center;
  gap: 8px;
}

.refresh-btn:hover:not(:disabled) {
  background: #5a67d8;
}

.refresh-btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.stats {
  display: flex;
  justify-content: space-around;
  text-align: center;
}

.stat-item {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.stat-number {
  font-size: 2em;
  font-weight: bold;
  color: #667eea;
}

.stat-label {
  color: #666;
  margin-top: 5px;
}

.loading {
  text-align: center;
  padding: 40px;
  color: white;
}

.loading-spinner {
  width: 40px;
  height: 40px;
  border: 4px solid rgba(255,255,255,0.3);
  border-top: 4px solid white;
  border-radius: 50%;
  animation: spin 1s linear infinite;
  margin: 0 auto 20px;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

.error {
  text-align: center;
  padding: 40px;
  color: white;
}

.retry-btn {
  margin-top: 15px;
  padding: 10px 20px;
  background: #ff6b6b;
  color: white;
  border: none;
  border-radius: 5px;
  cursor: pointer;
}

.no-results {
  text-align: center;
  padding: 40px;
  color: white;
  font-size: 1.2em;
}

.stations-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(350px, 1fr));
  gap: 20px;
}

.station-card {
  background: white;
  border-radius: 12px;
  padding: 20px;
  box-shadow: 0 4px 12px rgba(0,0,0,0.1);
  transition: transform 0.2s, box-shadow 0.2s;
}

.station-card:hover {
  transform: translateY(-2px);
  box-shadow: 0 8px 25px rgba(0,0,0,0.15);
}

.station-card.inactive {
  opacity: 0.6;
  border-left: 4px solid #ff6b6b;
}

.station-card.no-bikes {
  border-left: 4px solid #ffa726;
}

.station-card.no-spaces {
  border-left: 4px solid #66bb6a;
}

.station-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 15px;
}

.station-name {
  color: #333;
  font-size: 1.2em;
  margin: 0;
}

.station-status {
  padding: 4px 12px;
  border-radius: 20px;
  font-size: 0.9em;
  font-weight: bold;
}

.status-active {
  background: #e8f5e8;
  color: #2e7d32;
}

.status-inactive {
  background: #ffebee;
  color: #c62828;
}

.station-info {
  margin-bottom: 15px;
}

.info-row {
  margin-bottom: 8px;
  font-size: 0.95em;
}

.info-label {
  color: #666;
  font-weight: 500;
}

.info-value {
  color: #333;
}

.bike-info {
  display: flex;
  justify-content: space-between;
  margin-bottom: 15px;
}

.bike-count {
  text-align: center;
  flex: 1;
}

.bike-count .count {
  display: block;
  font-size: 1.8em;
  font-weight: bold;
  margin-bottom: 5px;
}

.available .count {
  color: #4caf50;
}

.spaces .count {
  color: #2196f3;
}

.total .count {
  color: #9e9e9e;
}

.bike-count .label {
  font-size: 0.9em;
  color: #666;
}

.station-footer {
  border-top: 1px solid #eee;
  padding-top: 10px;
}

.update-time {
  color: #999;
  font-size: 0.85em;
}

.pagination {
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 15px;
  margin-top: 30px;
  padding: 20px;
}

.page-btn {
  padding: 8px 12px;
  background: white;
  border: 1px solid #ddd;
  border-radius: 5px;
  cursor: pointer;
  transition: background 0.3s;
}

.page-btn:hover:not(:disabled) {
  background: #f0f0f0;
}

.page-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.page-info {
  color: white;
  font-weight: 500;
}

/* 響應式設計 */
@media (max-width: 768px) {
  .app {
    padding: 10px;
  }
  
  .header h1 {
    font-size: 2em;
    flex-direction: column;
    gap: 10px;
  }
  
  .search-controls {
    flex-direction: column;
  }
  
  .filter-group {
    flex-direction: column;
  }
  
  .stats {
    flex-direction: column;
    gap: 15px;
  }
  
  .stations-grid {
    grid-template-columns: 1fr;
  }
  
  .bike-info {
    flex-direction: column;
    gap: 10px;
  }
  
  .pagination {
    flex-wrap: wrap;
    gap: 10px;
  }
}
</style>
