<template>
  <div class="app-container">
    <header>
      <h1>World Map Quiz</h1>
    </header>
    <main>
      <div class="game-info">
        <div class="timer">Time: {{ formatTime(timeElapsed) }}</div>
        <div class="score">Score: {{ score }}</div>
      </div>

      <div class="map-container">
        <WorldMap 
          ref="worldMapRef"
          :currentCountry="currentCountry"
          :correctAnswers="correctAnswers"
          @countrySelected="handleCountrySelect"
        />
        <div v-if="!isGameActive" class="game-overlay">
          <button @click="startGame" class="btn btn-primary">Start Game</button>
        </div>
      </div>

      <div class="answer-section">
        <div class="input-group">
          <div class="autocomplete">
            <input 
              v-model="userAnswer" 
              @input="updateSuggestions"
              @keyup.enter="checkAnswer"
              @keyup.down="navigateSuggestion(1)"
              @keyup.up="navigateSuggestion(-1)"
              placeholder="Type country name..."
              class="answer-input"
              :disabled="!isGameActive"
            />
            <div v-if="countrySuggestions.length > 0 && isGameActive" class="suggestions">
              <div 
                v-for="(suggestion, index) in countrySuggestions" 
                :key="index"
                @click="selectSuggestion(suggestion)"
                class="suggestion-item"
                :class="{ 'suggestion-active': index === activeSuggestionIndex }"
              >
                {{ suggestion.charAt(0).toUpperCase() + suggestion.slice(1) }}
              </div>
            </div>
          </div>
          <button 
            v-if="isGameActive" 
            @click="skipCountry" 
            class="btn btn-secondary skip-btn"
            title="Skip this country (no penalty)"
          >
            Skip
          </button>
        </div>
        <div v-if="feedback" class="feedback" :class="feedbackClass">
          {{ feedback }}
        </div>
      </div>
      
      <!-- Game Summary Popup (shown when game is complete) -->
      <div v-if="showGameSummary" class="game-summary-overlay">
        <div class="game-summary">
          <h2>Game Complete!</h2>
          <div class="summary-time">
            Time: {{ formatTime(timeElapsed) }}
          </div>
          
          <div class="country-attempts-list">
            <h3>Attempts Per Country</h3>
            <div class="country-attempt" v-for="(attempts, id) in countryStats" :key="id">
              <div class="country-name">
                {{ countryMap[id]?.name.charAt(0).toUpperCase() + countryMap[id]?.name.slice(1) }}
              </div>
              <div class="country-attempt-count" :class="getAttemptsClass(attempts)">
                {{ attempts }} attempt{{ attempts !== 1 ? 's' : '' }}
              </div>
            </div>
          </div>
          
          <div class="summary-stats">
            <div class="summary-stat">
              <div class="stat-value">{{ score }}</div>
              <div class="stat-label">Score</div>
            </div>
            <div class="summary-stat">
              <div class="stat-value">{{ correctAnswers.length }}</div>
              <div class="stat-label">Countries</div>
            </div>
            <div class="summary-stat">
              <div class="stat-value">{{ 
                Math.round((correctAnswers.length / (countryMap ? Object.keys(countryMap).length : 1)) * 100) 
              }}%</div>
              <div class="stat-label">Completion</div>
            </div>
          </div>
          
          <button @click="closeSummary" class="btn btn-primary mt-4">Play Again</button>
        </div>
      </div>
    </main>
  </div>
</template>

<script setup>
import { ref, onUnmounted, onMounted } from 'vue';
import WorldMap from './components/WorldMap.vue';

// Add a meta tag to prevent search engines from indexing the site during development
useHead({
  meta: [
    { name: 'robots', content: 'noindex, nofollow' }
  ]
});

// Game state
const isGameActive = ref(false);
const timeElapsed = ref(0); // Time elapsed in seconds
const score = ref(0);
const userAnswer = ref('');
const feedback = ref('');
const feedbackClass = ref('');
const currentCountry = ref(null);
const countries = ref([]);
const correctAnswers = ref([]);
const currentCountryAttempts = ref(0); // Track attempts for the current country
const countrySuggestions = ref([]); // Dropdown suggestions
const activeSuggestionIndex = ref(-1); // Index of the active suggestion
const showGameSummary = ref(false); // Control the end game popup
const countryStats = ref({}); // Track attempts per country
let timerInterval = null;

// Format time as MM:SS
function formatTime(seconds) {
  const mins = Math.floor(seconds / 60);
  const secs = seconds % 60;
  return `${mins}:${secs.toString().padStart(2, '0')}`;
}

// Start the game
function startGame() {
  isGameActive.value = true;
  timeElapsed.value = 0;
  score.value = 0;
  userAnswer.value = '';
  feedback.value = '';
  correctAnswers.value = [];
  countryStats.value = {};
  showGameSummary.value = false;
  
  // Make sure country data is initialized
  initCountryMap();
  
  // Select a random country to start
  selectRandomCountry();
  
  // Start timer (counting up)
  timerInterval = setInterval(() => {
    timeElapsed.value++;
  }, 1000);
}

// End the game and show the summary
function endGame() {
  isGameActive.value = false;
  
  if (timerInterval) {
    clearInterval(timerInterval);
    timerInterval = null;
  }
  
  // Check if all countries were guessed
  const allCountriesGuessed = correctAnswers.value.length === Object.keys(countryMap.value).length;
  
  // Show game summary popup
  showGameSummary.value = true;
  
  if (allCountriesGuessed) {
    feedback.value = `Congratulations! You've guessed all countries!`;
  } else {
    feedback.value = `Game Over!`;
  }
  
  feedbackClass.value = allCountriesGuessed ? 'correct' : 'neutral';
}

// Close the summary and reset for a new game
function closeSummary() {
  showGameSummary.value = false;
  startGame();
}

// Get class based on number of attempts
function getAttemptsClass(attempts) {
  if (attempts === 1) return 'perfect';
  if (attempts <= 3) return 'good';
  return 'challenging';
}

// Handle country selection from the map
function handleCountrySelect(countryId) {
  if (!isGameActive.value) return;
  
  // For debugging - allows clicking to select a country
  console.log('Country selected:', countryId);
}

// Get a reference to the WorldMap component to access its data
const worldMapRef = ref(null);
const countryMap = ref({});

// Select a random country for the player to guess
function selectRandomCountry() {
  // Get countries that haven't been guessed yet
  const availableCountries = Object.keys(countryMap.value).filter(
    id => !correctAnswers.value.includes(id)
  );
  
  if (availableCountries.length === 0) {
    // All countries have been guessed
    endGame();
    return;
  }
  
  const randomIndex = Math.floor(Math.random() * availableCountries.length);
  currentCountry.value = availableCountries[randomIndex];
  currentCountryAttempts.value = 0; // Reset attempts for the new country
  console.log('New country to guess:', currentCountry.value, countryMap.value[currentCountry.value]);
}

// Initialize the map of country IDs to names and alternatives
function initCountryMap() {
  if (!worldMapRef.value?.countries) return;
  
  worldMapRef.value.countries.forEach(country => {
    countryMap.value[country.id] = {
      name: country.name.toLowerCase(),
      alternatives: country.alternatives.map(alt => alt.toLowerCase())
    };
  });
  
  console.log('Country map initialized:', countryMap.value);
}

// Check the user's answer
function checkAnswer() {
  if (!isGameActive.value || !currentCountry.value) return;
  
  // Convert to lowercase and trim whitespace for comparison
  const answer = userAnswer.value.toLowerCase().trim();
  
  if (answer.length === 0) return;
  
  const country = countryMap.value[currentCountry.value];
  const correctName = country.name;
  const allValidAnswers = [correctName, ...country.alternatives];
  
  // Track attempts for this country
  if (!countryStats.value[currentCountry.value]) {
    countryStats.value[currentCountry.value] = 0;
  }
  countryStats.value[currentCountry.value]++;
  
  // Check if the answer matches the name or any alternative
  const correct = allValidAnswers.includes(answer);
  
  if (correct) {
    // Award points (no time bonus anymore since we're counting up)
    const pointsAwarded = 10;
    
    score.value += pointsAwarded;
    feedback.value = `Correct!`;
    feedbackClass.value = 'correct';
    correctAnswers.value.push(currentCountry.value);
    
    // If all countries guessed, end the game
    if (correctAnswers.value.length === Object.keys(countryMap.value).length) {
      endGame();
    } else {
      selectRandomCountry(); // Pick a new country
    }
    
    // Clear suggestion dropdown
    countrySuggestions.value = [];
  } else {
    currentCountryAttempts.value++;
    
    feedback.value = `Try again!`;
    feedbackClass.value = 'incorrect';
  }
  
  userAnswer.value = '';
  
  // Clear feedback after 2 seconds
  setTimeout(() => {
    feedback.value = '';
  }, 2000);
}

// Update suggestions based on input
function updateSuggestions() {
  if (!currentCountry.value || !isGameActive.value || userAnswer.value.length < 1) {
    countrySuggestions.value = [];
    activeSuggestionIndex.value = -1;
    return;
  }
  
  const input = userAnswer.value.toLowerCase().trim();
  
  // Get all country names and alternatives
  const allCountryNames = [];
  Object.values(countryMap.value).forEach(country => {
    allCountryNames.push(country.name);
    country.alternatives.forEach(alt => allCountryNames.push(alt));
  });
  
  // Filter suggestions based on input
  countrySuggestions.value = allCountryNames.filter(name => 
    name.toLowerCase().startsWith(input)
  );
  
  activeSuggestionIndex.value = -1;
}

// Navigate through suggestions with arrow keys
function navigateSuggestion(direction) {
  if (countrySuggestions.value.length === 0) return;
  
  const newIndex = activeSuggestionIndex.value + direction;
  
  if (newIndex >= countrySuggestions.value.length) {
    activeSuggestionIndex.value = 0;
  } else if (newIndex < 0) {
    activeSuggestionIndex.value = countrySuggestions.value.length - 1;
  } else {
    activeSuggestionIndex.value = newIndex;
  }
  
  // Update the input with the selected suggestion
  userAnswer.value = countrySuggestions.value[activeSuggestionIndex.value];
}

// Select a suggestion from the dropdown
function selectSuggestion(suggestion) {
  userAnswer.value = suggestion;
  countrySuggestions.value = [];
  checkAnswer();
}

// Skip current country without penalty
function skipCountry() {
  if (!isGameActive.value || !currentCountry.value) return;
  
  const country = countryMap.value[currentCountry.value];
  const correctName = country.name;
  
  // Record the skip in stats
  if (!countryStats.value[currentCountry.value]) {
    countryStats.value[currentCountry.value] = 0;
  }
  countryStats.value[currentCountry.value] += 5; // Counting as 5 attempts for a skip
  
  feedback.value = `Skipped. The answer was "${correctName.charAt(0).toUpperCase() + correctName.slice(1)}".`;
  feedbackClass.value = 'neutral';
  
  // Move to the next country
  correctAnswers.value.push(currentCountry.value);
  selectRandomCountry();
  
  // Clear feedback after 2 seconds
  setTimeout(() => {
    feedback.value = '';
  }, 2000);
}

// Initialize map data on component mount
onMounted(() => {
  // Wait for the world map component to load
  setTimeout(() => {
    initCountryMap();
  }, 500);
});

// Clean up on component unmount
onUnmounted(() => {
  if (timerInterval) {
    clearInterval(timerInterval);
  }
});
</script>

<style>
.app-container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 1rem;
  font-family: Arial, sans-serif;
}

@media (max-width: 768px) {
  .app-container {
    padding: 0.5rem;
  }
  
  h1 {
    font-size: 1.5rem;
  }
}

header {
  text-align: center;
  margin-bottom: 2rem;
}

.game-info {
  display: flex;
  justify-content: space-between;
  margin-bottom: 1rem;
  flex-wrap: wrap;
}

.timer, .score {
  font-size: 1.5rem;
  font-weight: bold;
}

@media (max-width: 480px) {
  .timer, .score {
    font-size: 1.2rem;
  }
}

.map-container {
  width: 100%;
  height: 500px;
  border: 1px solid #ccc;
  position: relative;
  margin-bottom: 1rem;
  background-color: #f0f0f0;
}

@media (max-width: 768px) {
  .map-container {
    height: 400px;
  }
}

@media (max-width: 480px) {
  .map-container {
    height: 300px;
  }
}

.game-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  background-color: rgba(0, 0, 0, 0.1);
}

.answer-section {
  margin-top: 1rem;
}

.answer-input {
  flex-grow: 1;
  padding: 0.75rem;
  font-size: 1rem;
  border: 1px solid #ccc;
  border-radius: 4px;
  margin-bottom: 1rem;
}

.btn {
  padding: 0.75rem 1.5rem;
  font-size: 1rem;
  background-color: #3498db;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.btn:hover {
  background-color: #2980b9;
}

.btn-secondary {
  background-color: #95a5a6;
  color: white;
}

.btn-secondary:hover {
  background-color: #7f8c8d;
}

.input-group {
  display: flex;
  gap: 0.5rem;
}

.skip-btn {
  flex-shrink: 0;
}

.feedback {
  padding: 0.5rem;
  margin-top: 0.5rem;
  border-radius: 4px;
  text-align: center;
  font-weight: bold;
}

.feedback.correct {
  background-color: #d4edda;
  color: #155724;
}

.feedback.incorrect {
  background-color: #f8d7da;
  color: #721c24;
}

.feedback.neutral {
  background-color: #e2e3e5;
  color: #383d41;
}

.game-stats {
  margin-top: 2rem;
  padding: 1.5rem;
  background-color: #f8f9fa;
  border-radius: 8px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
}

.game-stats h2 {
  text-align: center;
  margin-bottom: 1.5rem;
  color: #333;
}

.stats-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(150px, 1fr));
  gap: 1.5rem;
}

.stat-item {
  text-align: center;
  padding: 1rem;
  background-color: white;
  border-radius: 8px;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.05);
}

.stat-value {
  font-size: 2rem;
  font-weight: bold;
  color: #3498db;
  margin-bottom: 0.5rem;
}

.stat-label {
  font-size: 0.9rem;
  color: #666;
  text-transform: uppercase;
  letter-spacing: 1px;
}

.mt-4 {
  margin-top: 2rem;
}

/* Autocomplete Dropdown */
.autocomplete {
  position: relative;
  flex-grow: 1;
}

.suggestions {
  position: absolute;
  top: 100%;
  left: 0;
  width: 100%;
  max-height: 200px;
  overflow-y: auto;
  background-color: white;
  border: 1px solid #ccc;
  border-radius: 4px;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
  z-index: 10;
}

.suggestion-item {
  padding: 0.75rem;
  cursor: pointer;
}

.suggestion-item:hover,
.suggestion-active {
  background-color: #f0f0f0;
}

/* Game Summary Overlay */
.game-summary-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.7);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 100;
}

.game-summary {
  background-color: white;
  border-radius: 8px;
  padding: 2rem;
  width: 90%;
  max-width: 600px;
  max-height: 80vh;
  overflow-y: auto;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.3);
}

.game-summary h2 {
  text-align: center;
  margin-bottom: 1.5rem;
  color: #333;
}

.summary-time {
  font-size: 2rem;
  text-align: center;
  font-weight: bold;
  margin-bottom: 2rem;
  color: #3498db;
}

.country-attempts-list {
  margin-bottom: 2rem;
}

.country-attempts-list h3 {
  margin-bottom: 1rem;
  text-align: center;
}

.country-attempt {
  display: flex;
  justify-content: space-between;
  padding: 0.75rem 0;
  border-bottom: 1px solid #eee;
}

.country-name {
  font-weight: bold;
}

.country-attempt-count {
  padding: 0.25rem 0.5rem;
  border-radius: 4px;
}

.country-attempt-count.perfect {
  background-color: #d4edda;
  color: #155724;
}

.country-attempt-count.good {
  background-color: #fff3cd;
  color: #856404;
}

.country-attempt-count.challenging {
  background-color: #f8d7da;
  color: #721c24;
}

.summary-stats {
  display: flex;
  justify-content: space-around;
  margin-bottom: 2rem;
}

.summary-stat {
  text-align: center;
}

.summary-stat .stat-value {
  font-size: 1.5rem;
  font-weight: bold;
  color: #3498db;
}
</style>
