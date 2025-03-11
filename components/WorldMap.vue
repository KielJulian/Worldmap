<template>
  <div class="world-map" ref="mapContainer">
    <div v-if="loading" class="loading">Loading map...</div>
    <div v-else class="map-svg-container" v-html="svgContent"></div>
  </div>
</template>

<script setup>
import { ref, onMounted, computed, watch } from 'vue';

const props = defineProps({
  currentCountry: {
    type: String,
    default: null
  },
  correctAnswers: {
    type: Array,
    default: () => []
  }
});

const emit = defineEmits(['countrySelected']);

const mapContainer = ref(null);
const loading = ref(true);
const svgContent = ref('');
const countryData = ref([
  { id: 'usa', name: 'United States', alternatives: ['usa', 'us', 'united states of america', 'america'] },
  { id: 'germany', name: 'Germany', alternatives: ['deutschland', 'federal republic of germany'] },
  { id: 'japan', name: 'Japan', alternatives: ['nippon', 'nihon'] }
]);

// Load the SVG content
async function loadSvg() {
  try {
    loading.value = true;
    const response = await fetch('/svg/world-map.svg');
    svgContent.value = await response.text();
    loading.value = false;
    
    // After the SVG is loaded, apply classes and add event listeners
    setTimeout(() => {
      applyCountryClasses();
      addCountryEventListeners();
    }, 100);
  } catch (error) {
    console.error('Error loading SVG:', error);
    loading.value = false;
  }
}

// Apply CSS classes to countries based on game state
function applyCountryClasses() {
  // Reset all countries first
  document.querySelectorAll('.map-svg-container path').forEach(path => {
    path.classList.remove('highlighted', 'correct');
    path.classList.add('country');
  });
  
  // Highlight current country to guess
  if (props.currentCountry) {
    const currentCountryEl = document.getElementById(props.currentCountry);
    if (currentCountryEl) {
      currentCountryEl.classList.add('highlighted');
    }
  }
  
  // Mark correctly guessed countries
  props.correctAnswers.forEach(countryId => {
    const countryEl = document.getElementById(countryId);
    if (countryEl) {
      countryEl.classList.add('correct');
    }
  });
}

// Add click event listeners to countries
function addCountryEventListeners() {
  document.querySelectorAll('.map-svg-container path').forEach(path => {
    path.addEventListener('click', () => {
      emit('countrySelected', path.id);
    });
  });
}

// Watch for changes to props and update the map accordingly
watch(() => props.currentCountry, () => {
  applyCountryClasses();
});

watch(() => props.correctAnswers, () => {
  applyCountryClasses();
}, { deep: true });

// Export country data for use in the parent component
const countries = computed(() => countryData.value);

onMounted(() => {
  loadSvg();
});

defineExpose({
  countries
});
</script>

<style scoped>
.world-map {
  width: 100%;
  height: 100%;
  position: relative;
}

.loading {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  font-size: 1.5rem;
  color: #555;
}

.map-svg-container {
  width: 100%;
  height: 100%;
}

:deep(.country) {
  fill: var(--color-country-default);
  stroke: var(--gray-6);
  stroke-width: 1;
  cursor: pointer;
  transition: fill 0.3s ease;
}

:deep(.country:hover) {
  fill: var(--color-country-hover);
}

:deep(.country.highlighted) {
  stroke: var(--color-country-highlighted);
  stroke-width: 2;
}

:deep(.country.correct) {
  fill: var(--color-country-correct);
}
</style>