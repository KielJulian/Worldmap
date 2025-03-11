<template>
  <div class="world-map" ref="mapContainer">
    <div v-if="loading" class="loading">Loading map...</div>
    <div v-else class="map-svg-container" ref="svgContainer" v-html="svgContent"></div>
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
const svgContainer = ref(null);
const loading = ref(true);
const svgContent = ref('');
const countryData = ref([
  { id: 'usa', name: 'United States', alternatives: ['usa', 'us', 'united states of america', 'america'] },
  { id: 'germany', name: 'Germany', alternatives: ['deutschland', 'federal republic of germany'] },
  { id: 'japan', name: 'Japan', alternatives: ['nippon', 'nihon'] }
]);

// SVG viewport state
const viewBox = ref({
  x: 0,
  y: 0,
  width: 1000,
  height: 500,
  originalWidth: 1000,
  originalHeight: 500
});

// Load the SVG content
async function loadSvg() {
  try {
    loading.value = true;
    const response = await fetch('/svg/world-map.svg');
    svgContent.value = await response.text();
    loading.value = false;
    
    // After the SVG is loaded, apply classes and add event listeners
    setTimeout(() => {
      initSvg();
      applyCountryClasses();
      addCountryEventListeners();
    }, 100);
  } catch (error) {
    console.error('Error loading SVG:', error);
    loading.value = false;
  }
}

// Initialize the SVG and capture its original dimensions
function initSvg() {
  const svgElement = svgContainer.value?.querySelector('svg');
  if (!svgElement) return;
  
  // Store original viewBox
  const originalViewBox = svgElement.getAttribute('viewBox');
  if (originalViewBox) {
    const [x, y, width, height] = originalViewBox.split(' ').map(n => parseFloat(n));
    viewBox.value = {
      x, y, width, height,
      originalWidth: width,
      originalHeight: height
    };
  }
  
  // Make the SVG take 100% of container
  svgElement.setAttribute('width', '100%');
  svgElement.setAttribute('height', '100%');
  svgElement.style.display = 'block';
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

// Focus on a specific country by ID
function focusOnCountry(countryId) {
  const countryElement = document.getElementById(countryId);
  if (!countryElement) return;
  
  // Get the country's bounding box
  const bbox = countryElement.getBBox();
  
  // Calculate new viewBox - center the country and zoom to 40% of screen width
  const padding = 10; // Add some padding around the country
  let newWidth = bbox.width * 2.5; // Expand to show context around country
  let newHeight = bbox.height * 2.5;
  
  // Ensure minimum size for very small countries
  newWidth = Math.max(newWidth, viewBox.value.originalWidth / 5);
  newHeight = Math.max(newHeight, viewBox.value.originalHeight / 5);
  
  // Center the viewBox on the country
  const newX = bbox.x - (newWidth - bbox.width) / 2;
  const newY = bbox.y - (newHeight - bbox.height) / 2;
  
  // Apply the new viewBox to the SVG
  const svgElement = svgContainer.value?.querySelector('svg');
  if (svgElement) {
    // Animate the transition
    animateViewBox(
      viewBox.value.x, viewBox.value.y, viewBox.value.width, viewBox.value.height,
      newX, newY, newWidth, newHeight
    );
  }
}

// Animate the viewBox change
function animateViewBox(startX, startY, startWidth, startHeight, endX, endY, endWidth, endHeight) {
  const svgElement = svgContainer.value?.querySelector('svg');
  if (!svgElement) return;
  
  const duration = 500; // Animation duration in ms
  const startTime = performance.now();
  
  function animate(currentTime) {
    const elapsedTime = currentTime - startTime;
    const progress = Math.min(elapsedTime / duration, 1);
    
    // Ease in-out function
    const easeProgress = progress < 0.5
      ? 2 * progress * progress
      : 1 - Math.pow(-2 * progress + 2, 2) / 2;
    
    // Calculate current viewBox values
    const currentX = startX + (endX - startX) * easeProgress;
    const currentY = startY + (endY - startY) * easeProgress;
    const currentWidth = startWidth + (endWidth - startWidth) * easeProgress;
    const currentHeight = startHeight + (endHeight - startHeight) * easeProgress;
    
    // Update viewBox
    svgElement.setAttribute('viewBox', `${currentX} ${currentY} ${currentWidth} ${currentHeight}`);
    viewBox.value = { 
      x: currentX, y: currentY, width: currentWidth, height: currentHeight,
      originalWidth: viewBox.value.originalWidth,
      originalHeight: viewBox.value.originalHeight
    };
    
    // Continue animation if not complete
    if (progress < 1) {
      requestAnimationFrame(animate);
    }
  }
  
  requestAnimationFrame(animate);
}

// Reset zoom to show the entire map
function resetZoom() {
  const svgElement = svgContainer.value?.querySelector('svg');
  if (!svgElement) return;
  
  animateViewBox(
    viewBox.value.x, viewBox.value.y, viewBox.value.width, viewBox.value.height,
    0, 0, viewBox.value.originalWidth, viewBox.value.originalHeight
  );
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
  
  // Add mobile touch support
  if (mapContainer.value) {
    mapContainer.value.addEventListener('touchstart', handleTouchStart, { passive: false });
    mapContainer.value.addEventListener('touchmove', handleTouchMove, { passive: false });
    mapContainer.value.addEventListener('touchend', handleTouchEnd, { passive: false });
  }
});

// Touch event handling for mobile
let touchStartX = 0;
let touchStartY = 0;
let initialTouchDistance = 0;
let initialViewBox = null;

function handleTouchStart(event) {
  if (event.touches.length === 1) {
    // Single touch for panning
    touchStartX = event.touches[0].clientX;
    touchStartY = event.touches[0].clientY;
    initialViewBox = { ...viewBox.value };
  } else if (event.touches.length === 2) {
    // Two finger touch for pinch zoom
    initialTouchDistance = Math.hypot(
      event.touches[0].clientX - event.touches[1].clientX,
      event.touches[0].clientY - event.touches[1].clientY
    );
    initialViewBox = { ...viewBox.value };
  }
}

function handleTouchMove(event) {
  if (event.touches.length === 1 && initialViewBox) {
    // Handle panning
    event.preventDefault();
    const dx = event.touches[0].clientX - touchStartX;
    const dy = event.touches[0].clientY - touchStartY;
    
    // Convert screen pixels to SVG units
    const svgElement = svgContainer.value?.querySelector('svg');
    if (svgElement) {
      const svgRect = svgElement.getBoundingClientRect();
      const scaleX = viewBox.value.width / svgRect.width;
      const scaleY = viewBox.value.height / svgRect.height;
      
      // Update viewBox
      const newX = initialViewBox.x - dx * scaleX;
      const newY = initialViewBox.y - dy * scaleY;
      
      svgElement.setAttribute('viewBox', `${newX} ${newY} ${viewBox.value.width} ${viewBox.value.height}`);
      viewBox.value.x = newX;
      viewBox.value.y = newY;
    }
  } else if (event.touches.length === 2 && initialViewBox) {
    // Handle pinch zoom
    event.preventDefault();
    const currentDistance = Math.hypot(
      event.touches[0].clientX - event.touches[1].clientX,
      event.touches[0].clientY - event.touches[1].clientY
    );
    
    const scale = initialTouchDistance / currentDistance;
    
    // Calculate zoom center (midpoint between the two touches)
    const touchMidX = (event.touches[0].clientX + event.touches[1].clientX) / 2;
    const touchMidY = (event.touches[0].clientY + event.touches[1].clientY) / 2;
    
    // Convert touch center to SVG coordinates
    const svgElement = svgContainer.value?.querySelector('svg');
    if (svgElement) {
      const svgRect = svgElement.getBoundingClientRect();
      const svgCenterX = (touchMidX - svgRect.left) / svgRect.width * initialViewBox.width + initialViewBox.x;
      const svgCenterY = (touchMidY - svgRect.top) / svgRect.height * initialViewBox.height + initialViewBox.y;
      
      // Calculate new viewBox dimensions
      const newWidth = initialViewBox.width * scale;
      const newHeight = initialViewBox.height * scale;
      
      // Calculate new viewBox position to maintain the zoom center
      const newX = svgCenterX - newWidth / 2;
      const newY = svgCenterY - newHeight / 2;
      
      // Update the viewBox
      svgElement.setAttribute('viewBox', `${newX} ${newY} ${newWidth} ${newHeight}`);
      viewBox.value = { 
        x: newX, y: newY, width: newWidth, height: newHeight,
        originalWidth: viewBox.value.originalWidth,
        originalHeight: viewBox.value.originalHeight
      };
    }
  }
}

function handleTouchEnd() {
  initialViewBox = null;
}

defineExpose({
  countries,
  focusOnCountry,
  resetZoom
});
</script>

<style scoped>
.world-map {
  width: 100%;
  height: 100%;
  position: relative;
  overflow: hidden;
  touch-action: none; /* Disable browser touch actions for custom handling */
  user-select: none; /* Prevent text selection during interaction */
}

.loading {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  font-size: 1.5rem;
  color: var(--color-text-primary);
  background-color: rgba(0, 0, 0, 0.4);
  padding: var(--spacing-md);
  border-radius: var(--border-radius-md);
}

.map-svg-container {
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
}

:deep(svg) {
  max-width: 100%;
  max-height: 100%;
  overflow: visible;
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
  filter: drop-shadow(0 0 3px var(--color-primary));
}

:deep(.country.correct) {
  fill: var(--color-country-correct);
}
</style>