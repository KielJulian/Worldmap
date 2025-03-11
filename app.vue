<template>
  <div class="app-container">
    <header>
      <h1>World Map Quiz</h1>
    </header>
    <main>
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
        
        <!-- Game indicators and input over the map -->
        <div v-if="isGameActive" class="game-info">
          <div class="timer">Time: {{ formatTime(timeElapsed) }}</div>
          <div class="score">Score: {{ score }}</div>
        </div>
        
        <!-- Answer input inside map container -->
        <div v-if="isGameActive" class="answer-popup">
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
                autofocus
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
import { ref, onUnmounted, onMounted, watchEffect } from 'vue';
import WorldMap from './components/WorldMap.vue';

// Add a meta tag to prevent search engines from indexing the site during development
useHead({
  meta: [
    { name: 'robots', content: 'noindex, nofollow' },
    { name: 'viewport', content: 'width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no' }
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

// Function removed to improve performance
function focusOnCountry(countryId) {
  // No longer zooming for performance reasons
  return;
}

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

// Watch for country changes
watchEffect(() => {
  // No longer focusing on country with zoom to improve performance
});

// Initialize map data on component mount
onMounted(() => {
  // Wait for the world map component to load
  setTimeout(() => {
    initCountryMap();
  }, 500);
  
  // Handle screen orientation changes on mobile
  window.addEventListener('resize', handleResize);
  handleResize();
});

// Handle window resize events
function handleResize() {
  // Adjust for mobile
  if (window.innerWidth <= 768) {
    // Mobile-specific adjustments
    document.documentElement.style.setProperty('--map-height', '50vh');
  } else {
    // Desktop
    document.documentElement.style.setProperty('--map-height', '70vh');
  }
  
  // No longer focusing on countries for performance
}

// Clean up on component unmount
onUnmounted(() => {
  if (timerInterval) {
    clearInterval(timerInterval);
  }
  window.removeEventListener('resize', handleResize);
});
</script>

<style>
/* Main layout - fullscreen dark theme */
.app-container {
  width: 100%;
  height: 100vh;
  margin: 0;
  padding: 0;
  display: flex;
  flex-direction: column;
  font-family: Arial, sans-serif;
  background-color: var(--color-background);
  color: var(--color-text-primary);
  overflow: hidden;
}

header {
  text-align: center;
  padding: var(--spacing-md);
  background-color: var(--color-surface);
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.3);
  z-index: 5;
}

header h1 {
  margin: 0;
  font-size: 1.8rem;
  color: var(--color-primary);
}

main {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  position: relative;
}

/* Game information section */
.game-info {
  position: absolute;
  top: 10px;
  left: 0;
  right: 0;
  display: flex;
  justify-content: space-between;
  padding: var(--spacing-sm) var(--spacing-md);
  z-index: 4;
}

.timer, .score {
  font-size: 1.2rem;
  font-weight: bold;
  color: var(--color-text-primary);
  background-color: rgba(0, 0, 0, 0.5);
  padding: var(--spacing-xs) var(--spacing-md);
  border-radius: var(--border-radius-md);
  backdrop-filter: blur(4px);
}

/* Map container */
.map-container {
  flex: 1;
  width: 100%;
  height: var(--map-height, 70vh);
  position: relative;
  background-color: var(--color-background);
  overflow: hidden;
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
  background-color: rgba(0, 0, 0, 0.7);
  z-index: 10;
}

/* Answer input section */
.answer-popup {
  position: absolute;
  bottom: 10%;
  left: 50%;
  transform: translateX(-50%);
  width: 80%;
  max-width: 600px;
  padding: var(--spacing-md);
  background-color: var(--color-surface);
  border-radius: var(--border-radius-md);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.4);
  z-index: 5;
}

.input-group {
  display: flex;
  gap: var(--spacing-sm);
}

.autocomplete {
  position: relative;
  flex-grow: 1;
}

.answer-input {
  width: 100%;
  padding: var(--spacing-md);
  font-size: 1rem;
  border: 1px solid var(--color-border);
  border-radius: var(--border-radius-sm);
  color: var(--color-text-primary);
  background-color: var(--color-surface);
}

.answer-input:focus {
  outline: none;
  border-color: var(--color-primary);
  box-shadow: 0 0 0 2px rgba(33, 150, 243, 0.3);
}

.skip-btn {
  flex-shrink: 0;
}

/* Suggestion dropdown */
.suggestions {
  position: absolute;
  top: 100%;
  left: 0;
  width: 100%;
  max-height: 200px;
  overflow-y: auto;
  background-color: var(--color-surface);
  border: 1px solid var(--color-border);
  border-radius: var(--border-radius-sm);
  box-shadow: var(--shadow-md);
  z-index: 20;
}

.suggestion-item {
  padding: var(--spacing-md);
  cursor: pointer;
  transition: background-color 0.2s ease;
}

.suggestion-item:hover,
.suggestion-active {
  background-color: var(--gray-8);
}

/* Feedback messages */
.feedback {
  padding: var(--spacing-sm);
  margin-top: var(--spacing-sm);
  border-radius: var(--border-radius-sm);
  text-align: center;
  font-weight: bold;
}

.feedback.correct {
  background-color: rgba(67, 160, 71, 0.2);
  color: var(--green-3);
}

.feedback.incorrect {
  background-color: rgba(229, 57, 53, 0.2);
  color: var(--red-3);
}

.feedback.neutral {
  background-color: rgba(33, 33, 33, 0.2);
  color: var(--gray-3);
}

/* Buttons */
.btn {
  padding: var(--spacing-sm) var(--spacing-lg);
  font-size: 1rem;
  background-color: var(--color-primary);
  color: var(--color-primary-text);
  border: none;
  border-radius: var(--border-radius-sm);
  cursor: pointer;
  transition: background-color 0.2s ease;
}

.btn:hover {
  background-color: var(--color-primary-dark);
}

.btn-secondary {
  background-color: var(--color-secondary);
  color: var(--color-secondary-text);
}

.btn-secondary:hover {
  background-color: var(--color-secondary-dark);
}

/* Game summary popup */
.game-summary-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.8);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 100;
}

.game-summary {
  background-color: var(--color-surface);
  border-radius: var(--border-radius-md);
  padding: var(--spacing-xl);
  width: 90%;
  max-width: 600px;
  max-height: 80vh;
  overflow-y: auto;
  box-shadow: var(--shadow-lg);
}

.summary-time {
  font-size: 2rem;
  text-align: center;
  font-weight: bold;
  margin-bottom: var(--spacing-xl);
  color: var(--color-primary);
}

/* Country attempts section */
.country-attempts-list {
  margin-bottom: var(--spacing-xl);
}

.country-attempt {
  display: flex;
  justify-content: space-between;
  padding: var(--spacing-sm) 0;
  border-bottom: 1px solid var(--color-border);
}

.country-name {
  font-weight: bold;
  color: var(--color-text-primary);
}

.country-attempt-count {
  padding: var(--spacing-xs) var(--spacing-sm);
  border-radius: var(--border-radius-sm);
}

.country-attempt-count.perfect {
  background-color: rgba(67, 160, 71, 0.2);
  color: var(--green-3);
}

.country-attempt-count.good {
  background-color: rgba(253, 216, 53, 0.2);
  color: var(--yellow-3);
}

.country-attempt-count.challenging {
  background-color: rgba(229, 57, 53, 0.2);
  color: var(--red-3);
}

/* Summary stats */
.summary-stats {
  display: flex;
  justify-content: space-around;
  margin-bottom: var(--spacing-xl);
}

.summary-stat {
  text-align: center;
}

.summary-stat .stat-value {
  font-size: 1.5rem;
  font-weight: bold;
  color: var(--color-primary);
}

.game-summary h2,
.country-attempts-list h3 {
  text-align: center;
  margin-bottom: var(--spacing-md);
  color: var(--color-text-primary);
}

/* Utility classes */
.mt-4 {
  margin-top: var(--spacing-xl);
}

/* Responsive styles */
@media (max-width: 768px) {
  header h1 {
    font-size: 1.4rem;
  }
  
  .game-info {
    padding: var(--spacing-xs) var(--spacing-sm);
  }
  
  .timer, .score {
    font-size: 1rem;
  }
  
  .answer-input {
    padding: var(--spacing-sm);
  }
  
  .btn {
    padding: var(--spacing-sm) var(--spacing-md);
    font-size: 0.9rem;
  }
  
  .game-summary {
    padding: var(--spacing-md);
    width: 95%;
  }
  
  .summary-time {
    font-size: 1.5rem;
  }
  
  .answer-popup {
    width: 90%;
    bottom: 6%;
    padding: var(--spacing-sm);
  }
}

/* Fix for touch devices */
@media (hover: none) {
  .suggestion-item:hover {
    background-color: transparent;
  }
  
  .suggestion-item:active {
    background-color: var(--gray-8);
  }
}
</style>