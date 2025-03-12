<template>
  <Analytics />
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
        </div>
        
        <!-- Feedback display above input -->
        <div v-if="isGameActive && feedback" class="feedback-container">
          <div class="feedback-above" :class="feedbackClass">
            {{ feedback }}
          </div>
        </div>
        
        <!-- Answer input inside map container -->
        <div v-if="isGameActive" class="answer-popup">
          <div class="autocomplete">
            <div class="input-wrapper">
              <input 
                ref="inputRef"
                v-model="userAnswer" 
                @input="updateAutocomplete"
                @keyup.enter="checkAnswer"
                @keydown.tab.prevent="acceptAutocomplete"
                @keydown.right="acceptAutocomplete"
                placeholder="Type a country name (or 'skip' to skip)..."
                class="answer-input"
                :disabled="!isGameActive"
                autofocus
              />
              <input 
                type="text" 
                :value="autocompleteText" 
                class="autocomplete-suggestion"
                disabled
                aria-hidden="true"
              />
            </div>
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
            <h3>Country Performance</h3>
            <div class="country-attempt" 
                v-for="id in Object.keys(countryStats).sort((a, b) => (countryTimes[a] || 0) - (countryTimes[b] || 0))" 
                :key="id">
              <div class="country-name">
                {{ countryMap[id]?.name.charAt(0).toUpperCase() + countryMap[id]?.name.slice(1) }}
              </div>
              <div class="country-stats">
                <div class="country-attempt-count" :class="getAttemptsClass(countryStats[id])">
                  {{ countryStats[id] }} attempt{{ countryStats[id] !== 1 ? 's' : '' }}
                </div>
                <div class="country-time">
                  {{ formatTime(countryTimes[id] || 0) }}
                </div>
              </div>
            </div>
          </div>
          
          <div class="summary-stats">
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
            <div class="summary-stat">
              <div class="stat-value">{{ formatTime(Object.values(countryTimes).reduce((sum, time) => sum + time, 0)) }}</div>
              <div class="stat-label">Total Answer Time</div>
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
import { Analytics } from '@vercel/analytics/nuxt';

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
const userAnswer = ref('');
const feedback = ref('');
const feedbackClass = ref('');
const currentCountry = ref(null);
const countries = ref([]);
const correctAnswers = ref([]);
const currentCountryAttempts = ref(0); // Track attempts for the current country
const autocompleteText = ref(''); // Inline autocomplete suggestion
const inputRef = ref(null); // Reference to the input element
const showGameSummary = ref(false); // Control the end game popup
const countryStats = ref({}); // Track attempts per country
const countryTimes = ref({}); // Track time spent on each country in seconds
const currentCountryStartTime = ref(0); // When the current country was started
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
  userAnswer.value = '';
  feedback.value = '';
  correctAnswers.value = [];
  countryStats.value = {};
  countryTimes.value = {};
  currentCountryStartTime.value = 0;
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
  currentCountryStartTime.value = timeElapsed.value; // Store when this country was presented
  
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
  
  // Check if user wants to skip this country
  if (answer === 'skip') {
    skipCountry();
    return;
  }
  
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
    // Calculate time spent on this country
    const timeSpent = timeElapsed.value - currentCountryStartTime.value;
    countryTimes.value[currentCountry.value] = timeSpent;
    
    feedback.value = `Correct! Time: ${formatTime(timeSpent)}`;
    feedbackClass.value = 'correct';
    correctAnswers.value.push(currentCountry.value);
    
    // If all countries guessed, end the game
    if (correctAnswers.value.length === Object.keys(countryMap.value).length) {
      endGame();
    } else {
      selectRandomCountry(); // Pick a new country
    }
  } else {
    currentCountryAttempts.value++;
    
    feedback.value = `Try again!`;
    feedbackClass.value = 'incorrect';
  }
  
  // Clear input and autocomplete
  userAnswer.value = '';
  autocompleteText.value = '';
  
  // Clear feedback after 2 seconds
  setTimeout(() => {
    feedback.value = '';
  }, 2000);
}

// Update autocomplete suggestion based on input
function updateAutocomplete() {
  if (!currentCountry.value || !isGameActive.value || userAnswer.value.length < 1) {
    autocompleteText.value = '';
    return;
  }
  
  const input = userAnswer.value.toLowerCase().trim();
  
  // Get all country names and alternatives
  const allCountryNames = [];
  Object.values(countryMap.value).forEach(country => {
    allCountryNames.push(country.name);
    country.alternatives.forEach(alt => allCountryNames.push(alt));
  });
  
  // Find a match that starts with the input
  const match = allCountryNames.find(name => 
    name.toLowerCase().startsWith(input)
  );
  
  if (match) {
    // Set autocomplete text to the full suggestion
    // This will show in the background of the input
    autocompleteText.value = match;
  } else {
    autocompleteText.value = '';
  }
}

// Accept the autocomplete suggestion
function acceptAutocomplete(event) {
  // If we have a valid autocomplete suggestion
  if (autocompleteText.value && userAnswer.value.length > 0) {
    // Prevent tab key from changing focus
    if (event.key === 'Tab') {
      event.preventDefault();
    }
    
    // Set the user input to the full autocomplete suggestion
    userAnswer.value = autocompleteText.value;
    
    // Position cursor at the end of the input
    setTimeout(() => {
      if (inputRef.value) {
        inputRef.value.selectionStart = userAnswer.value.length;
        inputRef.value.selectionEnd = userAnswer.value.length;
        inputRef.value.focus();
      }
    }, 0);
    
    // Clear the autocomplete suggestion to avoid showing the same text twice
    autocompleteText.value = '';
    
    // Immediately search for a new autocomplete suggestion based on the now-complete word
    setTimeout(updateAutocomplete, 10);
  }
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
  
  // Record time spent
  const timeSpent = timeElapsed.value - currentCountryStartTime.value;
  countryTimes.value[currentCountry.value] = timeSpent;
  
  feedback.value = `Skipped. The answer was "${correctName.charAt(0).toUpperCase() + correctName.slice(1)}".`;
  feedbackClass.value = 'neutral';
  
  // Move to the next country
  correctAnswers.value.push(currentCountry.value);
  selectRandomCountry();
  
  // Clear input and autocomplete
  userAnswer.value = '';
  autocompleteText.value = '';
  
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

// Initialize Vercel Analytics
inject();

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
  background-color: transparent;
  border-radius: var(--border-radius-md);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.4);
  z-index: 5;
}

.answer-popup {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: var(--spacing-md);
}

.autocomplete {
  position: relative;
  width: 100%;
  max-width: 500px;
}

.input-wrapper {
  position: relative;
  display: flex;
  align-items: center;
  width: 100%;
}

.answer-input {
  position: relative;
  width: 100%;
  padding: var(--spacing-md);
  font-size: 1rem;
  border: 1px solid var(--color-border);
  border-radius: var(--border-radius-sm);
  color: var(--color-text-primary);
  background: transparent;
  z-index: 2;
  transition: border-color 0.2s, box-shadow 0.2s;
}

.answer-input:focus {
  outline: none;
  border-color: var(--color-primary);
  box-shadow: 0 0 0 2px rgba(33, 150, 243, 0.3);
}

.autocomplete-suggestion {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  padding: var(--spacing-md);
  font-size: 1rem;
  border: 1px solid transparent;
  border-radius: var(--border-radius-sm);
  color: var(--color-text-secondary-light, #666);
  background-color: transparent;
  z-index: 1;
  pointer-events: none;
}

/* Feedback messages */
.feedback {
  padding: var(--spacing-sm);
  margin-top: var(--spacing-sm);
  border-radius: var(--border-radius-sm);
  text-align: center;
  font-weight: bold;
}

/* Feedback container - independent of the input field */
.feedback-container {
  position: absolute;
  bottom: calc(10% + 80px); /* Position above the input field */
  left: 50%;
  transform: translateX(-50%);
  width: 80%;
  max-width: 600px;
  z-index: 6;
}

/* Feedback above input */
.feedback-above {
  text-align: center;
  font-weight: bold;
  padding: var(--spacing-sm);
  text-shadow: 0 0 4px rgba(0, 0, 0, 0.8);
}

.feedback.correct {
  background-color: rgba(67, 160, 71, 0.2);
  color: var(--green-3);
}

.feedback-above.correct {
  color: var(--green-3);
}

.feedback.incorrect {
  background-color: rgba(229, 57, 53, 0.2);
  color: var(--red-3);
}

.feedback-above.incorrect {
  color: var(--red-3);
}

.feedback.neutral {
  background-color: rgba(33, 33, 33, 0.2);
  color: var(--gray-3);
}

.feedback-above.neutral {
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

.country-stats {
  display: flex;
  gap: var(--spacing-md);
}

.country-attempt-count {
  padding: var(--spacing-xs) var(--spacing-sm);
  border-radius: var(--border-radius-sm);
}

.country-time {
  font-family: monospace;
  padding: var(--spacing-xs) var(--spacing-sm);
  color: var(--color-text-primary);
  background-color: rgba(33, 33, 33, 0.2);
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
    font-size: 0.9rem;
  }
  
  .answer-input, .autocomplete-suggestion {
    padding: var(--spacing-sm);
    font-size: 0.9rem;
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
  
  .feedback-container {
    bottom: calc(6% + 70px);
    width: 90%;
  }
  
  .feedback-above {
    font-size: 0.9rem;
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