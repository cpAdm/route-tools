<template>
  <div class="container">
    <div class="map-container">
      <div id="map"></div>
    </div>

    <div class="info-container">
      <h3>Points</h3>

      <p v-if="!points.length">Add points by clicking on the map.</p>

      <ol>
        <li v-for="(point, index) in points" :key="point.id">
          <input v-model="point.name" />
          (Lat: {{ point.lat.toFixed(5) }}, Lng: {{ point.lng.toFixed(5) }})
          <button :disabled="index === 0" @click="moveUp(point.id)">Up</button>
          <button :disabled="index === points.length - 1" @click="moveDown(point.id)">Down</button>
          <button @click="deletePoint(point.id)">Delete</button>
        </li>
      </ol>

      <p v-if="totalDistance">Total distance: {{ totalDistance.toFixed(0) }} m</p>

      <button v-if="points.length" @click="clearPoints">Clear points</button>

      <h3>Helicopter Route (SVG)</h3>

      <div class="route-image">
        <p v-if="!svgData">At least two points required.</p>

        <svg
          v-else
          :height="svgHeight"
          :viewBox="svgData.viewBox"
          :width="svgWidth"
          xmlns="http://www.w3.org/2000/svg"
        >
          <defs>
            <marker
              id="arrowhead"
              markerHeight="7"
              markerWidth="10"
              orient="auto"
              refX="0"
              refY="3.5"
            >
              <polygon fill="purple" points="0 0, 10 3.5, 0 7" />
            </marker>
          </defs>

          <!-- North indicator -->
          <text fill="black" font-size="16" text-anchor="middle" x="250" y="20">N</text>

          <!-- Center marker -->
          <circle :cx="svgData.center.x" :cy="svgData.center.y" fill="black" r="5" />

          <!-- Arrows: one per segment -->
          <template v-for="(seg, index) in svgData.segments" :key="index">
            <line
              :x1="svgData.center.x"
              :x2="seg.x2"
              :y1="svgData.center.y"
              :y2="seg.y2"
              marker-end="url(#arrowhead)"
              stroke="purple"
              stroke-width="2"
            />

            <text
              :x="seg.x2 + (svgData.center.x - seg.x2 > 0 ? -20 : 15)"
              :y="seg.y2 + (svgData.center.y - seg.y2 > 0 ? -15 : 20)"
              fill="black"
              font-size="14"
              text-anchor="start"
            >
              <!--              Pt -->
              {{ seg.pointIndex }}
              <!--              : {{ seg.bearing.toFixed(0) }}° ({{ seg.distance.toFixed(0) }}m)-->
            </text>
          </template>

          <!-- Scale bar -->
          <g v-if="scaleBar">
            <line
              :x1="scaleBar.x1"
              :x2="scaleBar.x2"
              :y1="scaleBar.y1"
              :y2="scaleBar.y2"
              stroke="black"
              stroke-width="2"
            />
            <text
              :x="(scaleBar.x1 + scaleBar.x2) / 2"
              :y="scaleBar.y1 - 5"
              fill="black"
              font-size="12"
              text-anchor="middle"
            >
              {{ scaleBar.distance.toFixed(0) }} m
            </text>
          </g>
        </svg>
      </div>
    </div>
  </div>
</template>

<script lang="ts" setup>
import { computed, onMounted, watch } from 'vue'
import leaflet from 'leaflet'
import 'leaflet/dist/leaflet.css'
import { useStorage } from '@vueuse/core'

type LatLng = { lat: number; lng: number }
type Point = LatLng & {
  id: string
  name: string
}

const points = useStorage<Point[]>('points', [])

const mapLocation = useStorage<LatLng>('mapLocation', { lat: 52.24, lng: 6.865 })
const mapZoom = useStorage<number>('mapZoom', 15)

const svgWidth = 500
const svgHeight = 400

let map: leaflet.Map
let markers: leaflet.Marker[] = []
let route: leaflet.Polyline | null = null

const initMap = () => {
  map = leaflet.map('map').setView([mapLocation.value.lat, mapLocation.value.lng], mapZoom.value)
  leaflet
    .tileLayer('https://tile.openstreetmap.org/{z}/{x}/{y}.png', {
      attribution: '&copy; <a href="http://www.openstreetmap.org/copyright">OpenStreetMap</a>',
    })
    .addTo(map)

  // When the map moves, update saved location and zoom.
  map.on('moveend', () => {
    const center = map.getCenter()
    mapLocation.value = { lat: center.lat, lng: center.lng }
  })
  map.on('zoomend', () => {
    mapZoom.value = map.getZoom()
  })

  // Load stored points to the map
  updateMap()

  map.on('click', (e: leaflet.LeafletMouseEvent) => {
    const { lat, lng } = e.latlng
    const newIndex = points.value.length + 1
    const newPoint: Point = { id: crypto.randomUUID(), lat, lng, name: `Point ${newIndex}` }
    points.value.push(newPoint)
  })
}

const escapeHTML = (content: string) => new Option(content).innerHTML

const addDraggableMarker = (point: Point) => {
  const marker = leaflet.marker([point.lat, point.lng], { draggable: true }).addTo(map)
  marker.bindTooltip(escapeHTML(point.name), { permanent: true })

  marker.on('dragend', (e) => {
    const { lat, lng } = (e.target as leaflet.Marker).getLatLng()
    const index = points.value.findIndex((pt) => pt.id === point.id)
    if (index !== -1) {
      points.value[index] = { ...points.value[index], lat, lng }
      updateRoute()
    }
  })
  markers.push(marker)
}

const updateMap = () => {
  updateMarkers()
  updateRoute()
}

const updateMarkers = () => {
  markers.forEach((marker) => map.removeLayer(marker))
  markers = []
  points.value.forEach((pt) => addDraggableMarker(pt))
}

const updateRoute = () => {
  if (route) {
    map.removeLayer(route)
    route = null
  }

  if (points.value.length < 1) return
  const latlngs = points.value.map((p) => [p.lat, p.lng]) as leaflet.LatLngExpression[]
  route = leaflet.polyline(latlngs, { color: 'purple' }).addTo(map)
}

const moveUp = (id: string) => {
  const idx = points.value.findIndex((pt) => pt.id === id)
  if (idx <= 0) return
  ;[points.value[idx - 1], points.value[idx]] = [points.value[idx], points.value[idx - 1]]
}

const moveDown = (id: string) => {
  const idx = points.value.findIndex((pt) => pt.id === id)
  if (idx === -1 || idx === points.value.length - 1) return
  ;[points.value[idx + 1], points.value[idx]] = [points.value[idx], points.value[idx + 1]]
}

const deletePoint = (id: string) => {
  points.value.splice(
    points.value.findIndex((pt) => pt.id === id),
    1,
  )
}

const clearPoints = () => {
  if (confirm('Are you sure you want to DELETE all points?')) {
    points.value = []
  }
}

const totalDistance = computed(() => {
  if (points.value.length < 2) return 0
  let sum = 0
  for (let i = 0; i < points.value.length - 1; i++) {
    sum += haversineDistance(points.value[i], points.value[i + 1])
  }
  return sum
})

// Compute SVG data based on segments between consecutive points.
// Each segment is drawn as an arrow from the SVG center.
const svgData = computed(() => {
  if (points.value.length < 2) return null

  // Compute each segment (from point i to i+1)
  const segmentsRaw: { distance: number; bearing: number; pointIndex: number }[] = []
  for (let i = 0; i < points.value.length - 1; i++) {
    const p1 = points.value[i]
    const p2 = points.value[i + 1]
    const distance = haversineDistance(p1, p2)
    const bearing = getBearing(p1, p2)
    segmentsRaw.push({ distance, bearing, pointIndex: i + 1 })
  }

  // Use the maximum segment distance to determine a scale factor.
  const maxDistance = Math.max(...segmentsRaw.map((s) => s.distance))
  const maxLineLength = 0.6 * Math.min(svgHeight, svgWidth)
  const scaleFactor = maxDistance > 0 ? maxLineLength / maxDistance : 1

  const center = { x: svgWidth / 2, y: svgHeight / 2 }
  const segments = segmentsRaw.map((seg) => {
    const rad = toRad(seg.bearing)
    const len = seg.distance * scaleFactor
    const x2 = center.x + len * Math.sin(rad)
    const y2 = center.y - len * Math.cos(rad)
    return { ...seg, x2, y2 }
  })

  return { viewBox: `0 0 ${svgWidth} ${svgHeight}`, center, segments, scaleFactor }
})

// Compute scale bar for the SVG.
// We choose a fixed pixel length and compute its corresponding distance.
const scaleBar = computed(() => {
  if (!svgData.value) return null

  const barPixelLength = 100
  // Using the scaleFactor computed in svgData:
  const distanceForBar = barPixelLength / svgData.value.scaleFactor
  // Place the scale bar at the bottom center.
  const x1 = (svgWidth - barPixelLength) / 2
  const y1 = svgHeight - 20
  const x2 = x1 + barPixelLength
  const y2 = y1
  return { x1, y1, x2, y2, distance: distanceForBar }
})

const toRad = (deg: number) => (deg * Math.PI) / 180
const toDeg = (rad: number) => (rad * 180) / Math.PI

const getBearing = (point1: LatLng, point2: LatLng): number => {
  const phi1 = toRad(point1.lat)
  const phi2 = toRad(point2.lat)

  const deltaLng = toRad(point2.lng - point1.lng)

  const y = Math.sin(deltaLng) * Math.cos(phi2)
  const x = Math.cos(phi1) * Math.sin(phi2) - Math.sin(phi1) * Math.cos(phi2) * Math.cos(deltaLng)

  const bearing = toDeg(Math.atan2(y, x))
  return (bearing + 360) % 360
}

const haversineDistance = (point1: LatLng, point2: LatLng): number => {
  const R = 6371000
  const dLat = toRad(point2.lat - point1.lat)
  const dLng = toRad(point2.lng - point1.lng)
  const a =
    Math.sin(dLat / 2) ** 2 +
    Math.cos(toRad(point1.lat)) * Math.cos(toRad(point2.lat)) * Math.sin(dLng / 2) ** 2
  const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a))
  return R * c
}

// Update the map when points change
watch(
  points,
  () => {
    updateMap()
  },
  { deep: true },
)

onMounted(() => {
  initMap()
})
</script>

<style scoped>
.container {
  display: flex;
  flex-direction: row;
  gap: 1rem;
}

.map-container {
  flex: 1;
}

.info-container {
  flex: 1;
}

#map {
  width: 100%;
  height: 80vh;
}

.route-image {
  border: 1px solid #ccc;
}

li {
  margin-bottom: 0.5rem;
}

/* Responsive: one column on narrow screens */
@media (max-width: 768px) {
  .container {
    flex-direction: column;
  }
}
</style>
