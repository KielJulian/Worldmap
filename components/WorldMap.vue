<template>
  <div class="world-map" ref="mapContainer">
    <div v-if="loading" class="loading">Loading map...</div>
    <div v-else class="map-svg-container" ref="svgContainer">
      <svg :viewBox="`${viewBox.x} ${viewBox.y} ${viewBox.width} ${viewBox.height}`" width="100%" height="100%">
        <!-- Render each country path -->
        <path
          v-for="country in countriesData"
          :key="country.id"
          :id="country.id"
          :d="country.path"
          :class="[
            'country',
            { 'highlighted': props.currentCountry === country.id },
            { 'correct': props.correctAnswers.includes(country.id) }
          ]"
          @click="handleCountryClick(country.id)"
          @mouseover="hoveredCountry = country.id"
          @mouseleave="hoveredCountry = null"
        />
        
        <!-- Label for the hovered/selected country -->
        <g v-if="hoveredCountry || props.currentCountry" class="country-label">
          <text
            v-if="hoveredCountry"
            :x="getCountryPosition(hoveredCountry).x"
            :y="getCountryPosition(hoveredCountry).y"
            class="country-text"
          >
            ID: {{ hoveredCountry }}
          </text>
        </g>
      </svg>
    </div>
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
const hoveredCountry = ref(null);

// Countries data with IDs and path data
const countriesData = ref([
  {
    id: 'japan',
    name: 'Japan',
    alternatives: ['nippon', 'nihon'],
    path: 'M1397.7 94.5994L1397.1 89.4994L1393.5 82.8994L1394.8 76.3994L1392 71.2994L1383.8 62.3994L1379 63.5994L1379.2 67.5994L1384.3 74.7994L1385.3 82.5994L1383.6 85.0994L1379.1 91.4994L1374 88.3994V99.7994L1367.7 98.4994L1358 100.399L1356 104.799L1352.1 108.099L1351 112.199L1346.7 114.199L1350.7 118.599L1354.8 120.399L1355.7 126.199L1359.3 128.699L1361.8 125.999L1361 115.099L1353.6 110.399L1359.8 110.199L1364.9 107.199L1373.5 105.799L1375.9 110.599L1380.6 113.099L1385 105.699L1394.2 105.299L1399.6 102.299L1400.2 97.7994L1397.7 94.5994ZM1374.1 110.9L1372.5 108.7L1367.9 107.3L1366.9 110L1363.5 109.2L1362.2 113.1L1363.4 116.1L1367.7 117.9L1367.6 114.2L1369.7 112.7L1372.8 114.8L1374.1 110.9Z',
    center: { x: 1375, y: 95 }
  },
  {
    id: 'germany',
    name: 'Germany',
    alternatives: ['deutschland', 'federal republic of germany'],
    path: 'M678.4 11.1L679.5 11.6L680.9 11.5L683.3 13.1L690.5 14.3L688.1 18.5L687.7 23L686.4 24.1L684.1 23.5L684.3 25.1L680.7 28.6V31.5L683.1 30.5L684.9 33.2L684.8 35L686.3 37.4L684.6 39.3L686.1 44.2L688.9 45L688.4 47.7L683.9 51.3L673.7 49.6L666.3 51.7L665.7 55.5L659.8 56.3L654 53.4L652.1 54.8L642.6 51.9L640.6 49.5L643.3 45.7L644.3 33.1L639.2 26.5L635.5 23.3L627.9 20.9L627.5 16.3L634 15L642.3 16.6L640.8 9.5L645.5 12.2L656.9 7.40001L658.4 2.3L662.6 1L663.4 3.20001L665.6 3.3L668 5.8L671.5 8.70001L674 8.20001L678.4 11.1Z',
    center: { x: 660, y: 30 }
  },
  {
    id: 'usa',
    name: 'United States',
    alternatives: ['usa', 'us', 'united states of america', 'america'],
    path: 'M203.5 20.6L197.4 22.6L192.7 25.1L188.1 27.8L187.6 28.7L193.3 27.4L195.4 29.5L200 28L204.9 25.9L210.3 23.8L207.2 27.1L209.7 27.9L212.2 30.3L217.3 28.9L222.4 28.4L222.7 30.2L224.2 30.4L225.4 30.6L226.9 33.1L222.2 33.7H222.1L218.4 33L213.9 34.2L210.2 34.8L205.5 38.9L202.5 41.2L202.9 41.9L208.4 37.8H209.1L204.4 42.7L201.5 47.1L199 50.7L198.4 53.8L197.6 55.3L197 57L197.1 60.3L197.4 60.8L199.2 60.7L200.8 60L202.2 59.2L205.5 56.1L207.3 51.9L207.2 48L208.6 45.3L211.2 42.2L213.3 40L216 38.5L215.6 40.6L217.8 37.5L219.1 36.9L220.8 34.5L224.6 35.8L227.4 38.2L226.6 41.1L225 44L221.2 46.5L220.8 48.1H221.8L226.1 45.4L227.7 46L227.2 49.7L226.5 52.3L222.8 55.8L220.8 58L218.1 60.4L220.8 61.7L223.3 62.1L227.3 61.2L231 59.5L234 58.6L238.6 56.8L244.4 53L244.5 52.4L244.8 50.5L247.5 49.7L251.4 50L255.4 50.5L260 48.4L260.6 45.9L260.4 45L267.2 40.6L269.9 39.5L277.7 39.4H287L288.1 37.9L289.8 37.6L292.3 36.6L295.1 33.7L298.3 28.8L303.8 24.1L304.9 25.7L308.6 24.7L310.2 26.5L307.3 35L309.5 38.6L309.7 40.7L303.3 43.7L297.3 45.9L291.3 47.8L287.3 51.6L286 53L284.8 56.4L285.5 59.7L287.6 59.9L287.8 57.6L288.9 59L287.9 60.8L284.1 61.8L281.6 61.7L277.4 62.8L275.1 63.1L272 63.4L267 65.3L275.1 64.1L276.2 65.3L268.3 67.2H265L265.4 66.4L263.3 68.2L264.7 68.5L262.2 73.1L256.9 78L257 76.3L255.9 76L254.7 74.4V77.9L255.7 79L255.1 81.4L252.7 83.9L248.2 89L247.8 88.8L250.7 84.4L248.7 82L249.7 76.6L247.8 79.4V83.5L244.6 82.5L247.6 84.5L246.1 90.6L247.5 91.1V93.3L246.5 99.7L241.9 104.4L235.8 106.3L231.4 110.1L228.6 110.5L225.2 112.9L223.9 115L217 119.2L213.2 122.3L209.7 126.1L207.8 130.6V135.1L208.4 140.6L209.9 145.1L209.4 147.9L210.7 155.3L209.7 159.7L209.1 162.2L207.1 166.1L205.3 166.9L202.7 166.1L202.3 163.3L200.5 161.8L198.5 156.3L196.9 151.4L196.5 148.9L198.5 144.6L197.7 141.1L194.6 135.7L192.7 134.7L186.6 137.7L185.7 137.3L183.7 134.3L180.7 132.7L174.3 133.6L169.7 132.8L165.4 133.3L162.9 134.3L163.5 136L162.8 138.6L163.6 139.9L162.4 140.7L160.6 139.8L158.3 141L154.4 140.8L151.1 137.4L146.2 138.2L142.6 136.7L139.1 137.2L134.1 138.7L128 143.4L121.9 146.2L118.2 149.2L116.3 152.1L115.3 156.6L114.9 159.6L115.5 161.8L113.3 162L109.7 160.6L105.8 158.6L104.9 155.6L104.7 151.1L102.3 147.5L101.4 143.7L99.8 139.3L96.6 136.7L92.1 136.9L87.3 141.9L83.3 140L81 138.1L80.6 134.5L79.8 131.2L77.4 128.4L75.3 126.3L74 124H64.6L63.8 126.7H59.5H48.7L37.8 122.2L30.8 119.1L31.7 117.8L24.6 118.5L18.3 119L18.6 115.8L16.5 112.1L14.3 111.3L14.4 109.5L11.5 109.1L10.3 107.4L5.5 106.8L4.60001 105.7L5.39999 102.2L2.89999 95.8L2.39999 86.9L3.29999 85.4L2 83.3L0.5 77.9L2.29999 72.7L1.39999 69.2L5.29999 63.9L8.10001 58.5L9.20001 53.6L14.7 47.6L18.7 41.9L22.7 36.2L27 27.7L28.8 22.4L29.2 19.5L30.6 18.2L36.4 20.4L35.4 26.3L37.6 24.6L40.1 19.5L41.7 14.4H55.8H70.5H75.3H90.4H105.1H119.9H134.8H151.6H168.6H178.8L180.1 12H181.8L180.9 15.4L181.9 16.4L185.2 16.8L189.8 17.8L193.7 19.7L198.1 18.9L203.5 20.6Z',
    center: { x: 175, y: 90 }
  }
]);

// SVG viewport state
const viewBox = ref({
  x: 0,
  y: 0,
  width: 1401,
  height: 167,
  originalWidth: 1401,
  originalHeight: 167
});

// Function to get country center position for label
function getCountryPosition(countryId) {
  const country = countriesData.value.find(c => c.id === countryId);
  return country ? country.center : { x: 0, y: 0 };
}

// Handle country click
function handleCountryClick(countryId) {
  emit('countrySelected', countryId);
}

// Focus on a specific country by ID
function focusOnCountry(countryId) {
  const country = countriesData.value.find(c => c.id === countryId);
  if (!country) return;
  
  // Use the country's center point for focusing
  const centerX = country.center.x;
  const centerY = country.center.y;
  
  // Calculate new viewBox - center the country and zoom to 40% of original width
  let newWidth = viewBox.value.originalWidth / 2.5;
  let newHeight = viewBox.value.originalHeight / 2.5;
  
  // Center the viewBox on the country
  const newX = centerX - newWidth / 2;
  const newY = centerY - newHeight / 2;
  
  // Animate the transition
  animateViewBox(
    viewBox.value.x, viewBox.value.y, viewBox.value.width, viewBox.value.height,
    newX, newY, newWidth, newHeight
  );
}

// Animate the viewBox change
function animateViewBox(startX, startY, startWidth, startHeight, endX, endY, endWidth, endHeight) {
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
  animateViewBox(
    viewBox.value.x, viewBox.value.y, viewBox.value.width, viewBox.value.height,
    0, 0, viewBox.value.originalWidth, viewBox.value.originalHeight
  );
}

// Watch for changes to props and update the map accordingly
watch(() => props.currentCountry, () => {
  if (props.currentCountry) {
    hoveredCountry.value = props.currentCountry;
  }
});

// Export country data for use in the parent component
const countryData = computed(() => countriesData.value.map(country => ({
  id: country.id,
  name: country.name,
  alternatives: country.alternatives
})));

onMounted(() => {
  loading.value = false;
  
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
  countries: countryData,
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
  padding-left: 4rem;
  padding-right: 4rem;
}

@media (max-width: 768px) {
  .map-svg-container {
    padding-left: 1rem;
    padding-right: 1rem;
  }
}

svg {
  max-width: 100%;
  max-height: 100%;
  overflow: visible;
}

.country {
  fill: var(--color-country-default);
  stroke: var(--gray-6);
  stroke-width: 1;
  cursor: pointer;
  transition: fill 0.3s ease;
}

.country:hover {
  fill: var(--color-country-hover);
}

.country.highlighted {
  stroke: var(--color-country-highlighted);
  stroke-width: 2;
  filter: drop-shadow(0 0 3px var(--color-primary));
}

.country.correct {
  fill: var(--color-country-correct);
}

.country-text {
  fill: white;
  font-size: 12px;
  font-weight: bold;
  stroke: black;
  stroke-width: 0.5px;
  paint-order: stroke;
  text-anchor: middle;
  pointer-events: none;
}
</style>