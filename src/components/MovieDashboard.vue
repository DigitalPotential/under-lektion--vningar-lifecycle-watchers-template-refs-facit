<script setup>
import { ref, onMounted, onBeforeMount, onUnmounted, watch, computed } from 'vue'

// ==================== API KONFIGURATION ====================
const API_KEY = '3deda79e'
const API_URL = 'http://www.omdbapi.com'

// ==================== STATE MANAGEMENT ====================
/**
 * Huvudsaklig state för applikationen
 * movies: Lista med filmer från API
 * isLoading: Visar laddningstillstånd
 * errorMessage: Visar eventuella felmeddelanden
 */
const movies = ref([])
const isLoading = ref(false)
const errorMessage = ref('')

/**
 * Användarens preferenser som sparas i localStorage
 * category: Vald filmkategori (action/comedy/drama)
 * darkMode: Om dark mode är aktiverat
 */
const userPreferences = ref({
  category: 'action',
  darkMode: false
})

// ==================== SÖK OCH STATISTIK ====================
/**
 * State för sökfunktionalitet och statistik
 * searchQuery: Användarens sökning
 * searchHistory: Lista med tidigare sökningar
 * viewStats: Statistik över användning
 */
const searchQuery = ref('')
const searchHistory = ref([])
const viewStats = ref({
  totalSearches: 0,
  mostSearchedGenre: '',
  viewedMovies: new Set()
})

// ==================== COMPUTED PROPERTIES ====================
/**
 * Filtrerar filmer baserat på sökfrågan
 * Returnerar alla filmer om ingen sökning finns
 * Filtrerar på filmtitel om det finns en sökning
 */
const filteredMovies = computed(() => 
  !searchQuery.value 
    ? movies.value 
    : movies.value.filter(movie => 
        movie.Title.toLowerCase().includes(searchQuery.value.toLowerCase())
      )
)

// ==================== TEMPLATE REFS ====================
/**
 * Referenser till DOM-element för keyboard navigation
 * rootEl: Huvudcontainern
 * searchInput: Sökfältet
 * movieCards: Lista med filmkort
 * currentFocusIndex: Index för fokuserat filmkort
 */
const rootEl = ref(null)
const searchInput = ref(null)
const movieCards = ref([])
const currentFocusIndex = ref(-1)

// ==================== KEYBOARD NAVIGATION ====================
/**
 * Sätter upp keyboard navigation
 * '/' - Fokuserar sökfältet
 * Piltangenter - Navigerar mellan filmkort
 */
const setupKeyboardNavigation = () => {
  rootEl.value?.addEventListener('keydown', (e) => {
    // Snabbkommando för sökning
    if (e.key === '/' && document.activeElement !== searchInput.value) {
      e.preventDefault()
      searchInput.value?.focus()
    }

    // Piltangenter för navigation mellan filmkort
    if (e.key === 'ArrowDown' || e.key === 'ArrowUp') {
      e.preventDefault()
      
      const direction = e.key === 'ArrowDown' ? 1 : -1
      const cards = movieCards.value.filter(card => card)
      
      if (cards.length === 0) return
      
      if (currentFocusIndex.value === -1) {
        currentFocusIndex.value = direction === 1 ? 0 : cards.length - 1
      } else {
        currentFocusIndex.value = (currentFocusIndex.value + direction + cards.length) % cards.length
      }
      
      cards[currentFocusIndex.value]?.focus()
    }
  })
}

// ==================== API FUNKTIONER ====================
/**
 * Hämtar filmer från OMDB API
 * @param {string} category - Filmkategori att söka efter
 */
const getMovies = async (category) => {
  try {
    const res = await fetch(`${API_URL}/?s=${category}&apikey=${API_KEY}`)
    if (!res.ok) throw new Error('Nätverksfel')
    const data = await res.json()
    return data.Response === 'True' ? data : { Error: data.Error || 'Inga resultat hittades' }
  } catch (error) {
    throw new Error('Kunde inte hämta filmer: ' + error.message)
  }
}

// ==================== SÖKFUNKTIONER ====================
/**
 * Uppdaterar sökhistorik och statistik
 */
const updateSearchHistory = (query, category) => {
  searchHistory.value.push({ query, category })
  viewStats.value.totalSearches++
}

const updateMostSearchedGenre = () => {
  const categoryCounts = searchHistory.value.reduce((acc, search) => {
    acc[search.category] = (acc[search.category] || 0) + 1
    return acc
  }, {})
  
  viewStats.value.mostSearchedGenre = Object.entries(categoryCounts)
    .sort((a, b) => b[1] - a[1])[0]?.[0] || ''
}

const handleSearch = () => {
  const query = searchQuery.value.trim()
  const lastSearch = searchHistory.value[searchHistory.value.length - 1]?.query
  
  if (query && filteredMovies.value.length > 0 && query !== lastSearch) {
    updateSearchHistory(query, userPreferences.value.category)
    updateMostSearchedGenre()
    saveViewStats()
  }
}

// ==================== LIFECYCLE HOOKS ====================
/**
 * onBeforeMount: Körs innan komponenten monteras
 * - Hämtar sparade preferenser från localStorage
 */
onBeforeMount(() => {
  userPreferences.value = storage.get('movieDashboardPrefs', userPreferences.value)
  const savedStats = storage.get('movieViewStats')
  if (savedStats) {
    viewStats.value = {
      ...savedStats,
      viewedMovies: new Set(savedStats.viewedMovies)
    }
  }
})

/**
 * onMounted: Körs när komponenten har monterats
 * - Sätter upp keyboard navigation
 * - Laddar initiala filmer
 */
onMounted(() => {
  setupKeyboardNavigation()
  loadMovies()
})

/**
 * onUnmounted: Körs när komponenten avmonteras
 * - Städar upp event listeners
 * - Sparar användarens preferenser
 */
onUnmounted(() => {
  rootEl.value?.removeEventListener('keydown', setupKeyboardNavigation)
  savePreferences()
  saveViewStats()
  errorMessage.value = ''
})

// ==================== WATCHERS ====================
/**
 * Övervakar ändringar i kategori
 * - Laddar nya filmer
 * - Återställer sökning
 * - Sparar preferenser
 */
watch(() => userPreferences.value.category, async () => {
  await loadMovies()
  searchQuery.value = ''
  savePreferences()
})

watch(() => userPreferences.value.darkMode, savePreferences)

// ==================== DATA LADDNING OCH SPARANDE ====================
const loadMovies = async () => {
  isLoading.value = true
  errorMessage.value = ''
  
  try {
    const data = await getMovies(userPreferences.value.category)
    movies.value = data.Search || []
    if (!data.Search) errorMessage.value = data.Error
  } catch (error) {
    errorMessage.value = error.message
  } finally {
    isLoading.value = false
  }
}

// LocalStorage hantering
const storage = {
  get: (key, defaultValue = null) => {
    try {
      return JSON.parse(localStorage.getItem(key)) || defaultValue
    } catch {
      return defaultValue
    }
  },
  set: (key, value) => localStorage.setItem(key, JSON.stringify(value))
}

const savePreferences = () => storage.set('movieDashboardPrefs', userPreferences.value)
const saveViewStats = () => storage.set('movieViewStats', {
  ...viewStats.value,
  viewedMovies: Array.from(viewStats.value.viewedMovies)
})
</script>

<template>
  <div 
    ref="rootEl"
    :class="{ 'dark-mode': userPreferences.darkMode }" 
    tabindex="0"
  >
    <div class="header">
      <h1>Movie Dashboard</h1>
      <select v-model="userPreferences.category">
        <option value="action">Action</option>
        <option value="comedy">Comedy</option>
        <option value="drama">Drama</option>
      </select>
      <button @click="userPreferences.darkMode = !userPreferences.darkMode">
        Toggle Dark Mode
      </button>
    </div>

    <div class="search-section">
      <input 
        ref="searchInput"
        v-model="searchQuery"
        type="text" 
        placeholder="Tryck '/' eller 'S' för att söka..."
        class="search-input"
        @keyup.enter="handleSearch"
      >
      <div class="search-history">
        <h3>Senaste sökningar:</h3>
        <ul>
          <li v-for="search in searchHistory.slice(-5)" :key="search.query">
            {{ search.query }} ({{ search.category }})
          </li>
        </ul>
      </div>
    </div>

    <div class="stats">
      <p>Antal sökningar: {{ viewStats.totalSearches }}</p>
      <p>Mest sökta genre: {{ viewStats.mostSearchedGenre || 'Ingen data än' }}</p>
    </div>

    <div v-if="errorMessage" class="error">
      {{ errorMessage }}
    </div>

    <div v-if="isLoading" class="loading">
      Laddar filmer...
    </div>

    <div v-else class="movie-grid">
      <div 
        v-for="(movie, index) in filteredMovies" 
        :key="movie.imdbID" 
        :ref="el => movieCards[index] = el"
        class="movie-card"
        tabindex="0"
        :class="{ 'focused': index === currentFocusIndex }"
      >
        <img :src="movie.Poster" :alt="movie.Title">
        <h3>{{ movie.Title }}</h3>
        <p>{{ movie.Year }}</p>
      </div>
    </div>
  </div>
</template>

<style scoped>
.header {
  padding: 1rem;
  display: flex;
  gap: 1rem;
  align-items: center;
}

.movie-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
  gap: 1rem;
  padding: 1rem;
}

.movie-card {
  border: 1px solid #ddd;
  padding: 1rem;
  border-radius: 8px;
  cursor: pointer;
  outline: none; /* Ta bort default outline */
  transition: all 0.2s ease;
}

.movie-card img {
  width: 100%;
  height: auto;
}

.error {
  color: red;
  padding: 1rem;
}

.loading {
  padding: 1rem;
  text-align: center;
}

.dark-mode {
  background-color: #333;
  color: white;
}

.dark-mode .movie-card {
  background-color: #444;
  border-color: #555;
}

.movie-card:focus,
.movie-card.focused {
  border-color: #4CAF50;
  box-shadow: 0 0 0 2px rgba(76, 175, 80, 0.2);
  transform: translateY(-2px);
}

.dark-mode .movie-card:focus,
.dark-mode .movie-card.focused {
  border-color: #81C784;
  box-shadow: 0 0 0 2px rgba(129, 199, 132, 0.2);
}

.search-input:focus {
  outline: 2px solid #4CAF50;
  outline-offset: 2px;
}

.dark-mode .search-input:focus {
  outline-color: #81C784;
}
</style> 