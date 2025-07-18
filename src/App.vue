<template>
  <div class="min-h-screen gradient-bg">
    <!-- 導航欄 -->
    <div class="navbar glass-card shadow-lg mb-6">
      <div class="navbar-start">
        <h1 class="text-2xl font-bold text-primary flex items-center gap-3">
          YouBike 站點查詢系統
        </h1>
      </div>
    </div>

    <main class="container mx-auto px-4 max-w-7xl">
      <!-- 搜尋和統計區域 -->
      <div class="card glass-card shadow-xl mb-6">
        <div class="card-body">
          <!-- 搜尋控制 -->
          <div class="flex flex-wrap gap-4 mb-6">
            <div class="form-control flex-1 min-w-64">
              <div class="input-group">
                <input 
                  v-model="searchKeyword" 
                  type="text" 
                  placeholder="搜尋站點名稱或地址..."
                  class="input input-bordered w-full"
                  @input="filterStations"
                >
                <button 
                  v-if="searchKeyword" 
                  @click="clearSearch" 
                  class="btn btn-square btn-outline"
                >
                  ✕
                </button>
              </div>
            </div>
            
            <div class="form-control">
              <select v-model="selectedArea" @change="filterStations" class="select select-bordered">
                <option value="">所有區域</option>
                <option v-for="area in areas" :key="area" :value="area">
                  {{ area }}
                </option>
              </select>
            </div>

            <!-- 收藏功能按鈕 -->
            <div class="form-control">
              <button 
                @click="showFavoritesOnly = !showFavoritesOnly" 
                class="btn btn-outline btn-primary"
                :class="showFavoritesOnly ? 'btn-active' : ''"
              >
                <span class="text-lg">{{ showFavoritesOnly ? '💙' : '🤍' }}</span>
                {{ showFavoritesOnly ? '顯示全部' : '只看收藏' }}
                <div class="badge badge-primary ml-2">{{ favoriteStations.length }}</div>
              </button>
            </div>
            
            <button @click="refreshData" class="btn btn-primary" :disabled="loading">
              <span v-if="loading" class="loading loading-spinner loading-sm"></span>
              <span v-else class="text-lg">🔄</span>
              {{ loading ? '更新中...' : '重新整理' }}
            </button>
          </div>

          <!-- 統計資訊 -->
          <div class="stats shadow">
            <div class="stat">
              <div class="stat-figure text-primary text-3xl">📍</div>
              <div class="stat-title">站點數量</div>
              <div class="stat-value text-primary">{{ filteredStations.length }}</div>
            </div>
            <div class="stat">
              <div class="stat-figure text-success text-3xl">🚲</div>
              <div class="stat-title">可借車輛</div>
              <div class="stat-value text-success">{{ totalBikes }}</div>
            </div>
            <div class="stat">
              <div class="stat-figure text-info text-3xl">🅿️</div>
              <div class="stat-title">可還空位</div>
              <div class="stat-value text-info">{{ totalSpaces }}</div>
            </div>
            <div class="stat">
              <div class="stat-figure text-accent text-3xl">⭐</div>
              <div class="stat-title">收藏站點</div>
              <div class="stat-value text-accent">{{ favoriteStations.length }}</div>
            </div>
          </div>
        </div>
      </div>

      <!-- 載入狀態 -->
      <div v-if="loading" class="flex justify-center items-center py-20">
        <div class="text-center">
          <span class="loading loading-spinner loading-lg text-primary"></span>
          <p class="mt-4 text-lg">載入站點資料中...</p>
        </div>
      </div>

      <!-- 錯誤狀態 -->
      <div v-else-if="error" class="alert alert-error">
        <svg xmlns="http://www.w3.org/2000/svg" class="stroke-current shrink-0 h-6 w-6" fill="none" viewBox="0 0 24 24">
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M10 14l2-2m0 0l2-2m-2 2l-2-2m2 2l2 2m7-2a9 9 0 11-18 0 9 9 0 0118 0z" />
        </svg>
        <span>{{ error }}</span>
        <button @click="fetchStations" class="btn btn-sm">重新載入</button>
      </div>

      <!-- 無結果 -->
      <div v-else-if="filteredStations.length === 0" class="alert alert-info">
        <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" class="stroke-current shrink-0 w-6 h-6">
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M13 16h-1v-4h-1m1-4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z"></path>
        </svg>
        <span>{{ showFavoritesOnly ? '尚未收藏任何站點' : '找不到符合條件的站點' }}</span>
      </div>

      <!-- 站點卡片網格 -->
      <div v-else class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4 gap-6">
        <div 
          v-for="station in paginatedStations" 
          :key="station.sno"
          class="card glass-card shadow-xl hover:shadow-2xl transition-all duration-300 transform hover:-translate-y-1"
          :class="getStationCardClass(station)"
        >
          <div class="card-body p-5">
            <!-- 站點標題 -->
            <div class="mb-3">
              <h3 class="card-title text-lg leading-tight">{{ station.sna }}</h3>
            </div>
            
            <!-- 營運狀態 -->
            <div class="flex justify-between items-center mb-4">
              <div class="badge" :class="getStatusBadgeClass(station.act)">
                {{ station.act === '1' ? '營運中' : '暫停' }}
              </div>
              <div class="text-sm opacity-70">{{ station.sarea }}</div>
            </div>
            
            <!-- 地址 -->
            <div class="text-sm opacity-80 mb-4 line-clamp-2">
              📍 {{ station.ar }}
            </div>

            <!-- 車輛資訊 -->
            <div class="grid grid-cols-3 gap-2 mb-4">
              <div class="text-center">
                <div class="text-2xl font-bold text-success">{{ station.available_rent_bikes }}</div>
                <div class="text-xs opacity-70">可借</div>
              </div>
              <div class="text-center">
                <div class="text-2xl font-bold text-info">{{ station.available_return_bikes }}</div>
                <div class="text-xs opacity-70">可還</div>
              </div>
              <div class="text-center">
                <div class="text-2xl font-bold opacity-50">{{ station.Quantity }}</div>
                <div class="text-xs opacity-70">總數</div>
              </div>
            </div>

            <!-- 收藏按鈕 -->
            <div class="mb-4">
              <button 
                @click="toggleFavorite(station)" 
                class="btn btn-sm w-full"
                :class="isFavorite(station.sno) ? 'btn-error' : 'btn-outline btn-primary'"
              >
                {{ isFavorite(station.sno) ? '取消收藏' : '收藏' }}
              </button>
            </div>

            <!-- 更新時間 -->
            <div class="text-xs opacity-50 text-center border-t border-base-300 pt-2">
              🕒 {{ formatTime(station.mday) }}
            </div>
          </div>
        </div>
      </div>

      <!-- 分頁 -->
      <div v-if="totalPages > 1" class="flex justify-center mt-8">
        <div class="join">
          <button 
            @click="currentPage = 1" 
            :disabled="currentPage === 1"
            class="join-item btn"
          >
            ⟸
          </button>
          <button 
            @click="currentPage--" 
            :disabled="currentPage === 1"
            class="join-item btn"
          >
            ⟨
          </button>
          
          <button class="join-item btn btn-active">
            第 {{ currentPage }} 頁，共 {{ totalPages }} 頁
          </button>
          
          <button 
            @click="currentPage++" 
            :disabled="currentPage === totalPages"
            class="join-item btn"
          >
            ⟩
          </button>
          <button 
            @click="currentPage = totalPages" 
            :disabled="currentPage === totalPages"
            class="join-item btn"
          >
            ⟹
          </button>
        </div>
      </div>
    </main>

    <!-- Footer -->
    <footer class="footer footer-center p-10 text-base-content/70 mt-20">
      <div>
        <p>© 2025 YouBike 站點查詢系統 - Vue.js 練習專案</p>
        <p>資料來源：台北市政府開放資料平台</p>
      </div>
    </footer>
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
    const itemsPerPage = 16
    const favoriteStations = ref([])
    const showFavoritesOnly = ref(false)

    // 初始化收藏資料
    const loadFavorites = () => {
      const saved = localStorage.getItem('youbike-favorites')
      if (saved) {
        favoriteStations.value = JSON.parse(saved)
      }
    }

    // 儲存收藏資料
    const saveFavorites = () => {
      localStorage.setItem('youbike-favorites', JSON.stringify(favoriteStations.value))
    }

    // 收藏功能
    const toggleFavorite = (station) => {
      const index = favoriteStations.value.findIndex(fav => fav.sno === station.sno)
      if (index > -1) {
        favoriteStations.value.splice(index, 1)
      } else {
        favoriteStations.value.push({
          sno: station.sno,
          sna: station.sna,
          sarea: station.sarea
        })
      }
      saveFavorites()
    }

    const isFavorite = (sno) => {
      return favoriteStations.value.some(fav => fav.sno === sno)
    }

    // 計算屬性
    const areas = computed(() => {
      const areaSet = new Set(stations.value.map(station => station.sarea))
      return Array.from(areaSet).sort()
    })

    const filteredStations = computed(() => {
      let filtered = stations.value

      // 如果只顯示收藏，先過濾收藏站點
      if (showFavoritesOnly.value) {
        const favoriteSnos = favoriteStations.value.map(fav => fav.sno)
        filtered = filtered.filter(station => favoriteSnos.includes(station.sno))
      }

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
      return filteredStations.value.reduce((sum, station) => sum + parseInt(station.available_rent_bikes || 0), 0)
    })

    const totalSpaces = computed(() => {
      return filteredStations.value.reduce((sum, station) => sum + parseInt(station.available_return_bikes || 0), 0)
    })

    // 方法
    const fetchStations = async () => {
      loading.value = true
      error.value = ''
      
      try {
        const response = await axios.get(
          'https://tcgbusfs.blob.core.windows.net/dotapp/youbike/v2/youbike_immediate.json'
        )
        
        if (Array.isArray(response.data)) {
          stations.value = response.data
        } else {
          throw new Error('API 回傳格式錯誤')
        }
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

    const getStationCardClass = (station) => {
      if (station.act !== '1') return 'border-l-4 border-l-error'
      if (parseInt(station.available_rent_bikes || 0) === 0) return 'border-l-4 border-l-warning'
      if (parseInt(station.available_return_bikes || 0) === 0) return 'border-l-4 border-l-info'
      return 'border-l-4 border-l-success'
    }

    const getStatusBadgeClass = (act) => {
      return act === '1' ? 'badge-success' : 'badge-error'
    }

    const formatTime = (timestamp) => {
      try {
        if (!timestamp) return '無資料'
        
        if (timestamp.includes('-') && timestamp.includes(':')) {
          const date = new Date(timestamp)
          return date.toLocaleString('zh-TW', {
            month: '2-digit',
            day: '2-digit',
            hour: '2-digit',
            minute: '2-digit'
          })
        }
        
        const date = new Date(timestamp)
        return date.toLocaleString('zh-TW', {
          month: '2-digit',
          day: '2-digit',
          hour: '2-digit',
          minute: '2-digit'
        })
      } catch (e) {
        return '時間格式錯誤'
      }
    }

    // 監聽篩選變化，重置頁碼
    watch([searchKeyword, selectedArea, showFavoritesOnly], () => {
      currentPage.value = 1
    })

    // 初始化
    onMounted(() => {
      loadFavorites()
      fetchStations()
    })

    return {
      stations,
      searchKeyword,
      selectedArea,
      loading,
      error,
      currentPage,
      favoriteStations,
      showFavoritesOnly,
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
      toggleFavorite,
      isFavorite,
      getStationCardClass,
      getStatusBadgeClass,
      formatTime
    }
  }
}
</script>

<style scoped>
.line-clamp-2 {
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  overflow: hidden;
}
</style>
