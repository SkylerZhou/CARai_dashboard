<template>
  <div class="umap-viewer">
    <div class="file-upload-section">
      <div class="upload-grid">
        <div class="upload-item">
          <h3>UMAP File</h3>
          <div
            class="file-drop-zone"
            :class="{ 'drag-over': isDragOverUMAP, 'has-file': umapFile }"
            @drop="handleDropUMAP"
            @dragover.prevent="isDragOverUMAP = true"
            @dragleave="isDragOverUMAP = false"
            @dragend="isDragOverUMAP = false"
          >
            <input
              ref="umapFileInput"
              type="file"
              accept=".csv,.tsv,.txt,.json"
              @change="handleUMAPFileSelect"
              style="display: none"
            />
            <div class="drop-zone-content">
              <svg width="48" height="48" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                <path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4"></path>
                <polyline points="17 8 12 3 7 8"></polyline>
                <line x1="12" y1="3" x2="12" y2="15"></line>
              </svg>
              <p class="drop-zone-text" v-if="!umapFile">
                <strong>Drop your UMAP file here</strong> or
                <button @click="openUMAPFileDialog" class="btn-link">browse</button>
              </p>
              <p class="drop-zone-text" v-else>
                <strong>{{ umapFile.name }}</strong>
                <button @click="clearUMAPFile" class="btn-link">remove</button>
              </p>
              <p class="drop-zone-hint">Supports CSV, TSV, and JSON files</p>
            </div>
          </div>
        </div>
        <div class="upload-item">
          <h3>Metadata File (Optional)</h3>
          <div
            class="file-drop-zone"
            :class="{ 'drag-over': isDragOverMetadata, 'has-file': metadataFile }"
            @drop="handleDropMetadata"
            @dragover.prevent="isDragOverMetadata = true"
            @dragleave="isDragOverMetadata = false"
            @dragend="isDragOverMetadata = false"
          >
            <input
              ref="metadataFileInput"
              type="file"
              accept=".csv,.tsv,.txt,.json"
              @change="handleMetadataFileSelect"
              style="display: none"
            />
            <div class="drop-zone-content">
              <svg width="48" height="48" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                <path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4"></path>
                <polyline points="17 8 12 3 7 8"></polyline>
                <line x1="12" y1="3" x2="12" y2="15"></line>
              </svg>
              <p class="drop-zone-text" v-if="!metadataFile">
                <strong>Drop metadata file here</strong> or
                <button @click="openMetadataFileDialog" class="btn-link">browse</button>
              </p>
              <p class="drop-zone-text" v-else>
                <strong>{{ metadataFile.name }}</strong>
                <button @click="clearMetadataFile" class="btn-link">remove</button>
              </p>
              <p class="drop-zone-hint">Will be mapped using first column</p>
            </div>
          </div>
        </div>
      </div>
      <div class="load-button-container">
        <button 
          @click="loadViewer" 
          class="btn-load"
          :disabled="!umapFile || isLoading"
        >
          {{ isLoading ? 'Loading...' : 'Load Viewer' }}
        </button>
      </div>
      <div v-if="statusMessage" class="status-message" :class="statusType">
        {{ statusMessage }}
      </div>
    </div>
    <div class="controls" v-if="chartData.length > 0">
      <div class="control-group">
        <label for="colorBy">Color by:</label>
        <select id="colorBy" v-model="colorBy" @change="updateColors">
          <option value="none">None</option>
          <option v-for="category in availableCategories" :key="category" :value="category">
            {{ category }}
          </option>
        </select>
      </div>
      <div class="control-group">
        <label for="pointSize">Point Size:</label>
        <input
          id="pointSize"
          type="range"
          min="1"
          max="10"
          v-model.number="pointSize"
          @input="updatePointSize"
        />
        <span>{{ pointSize }}px</span>
      </div>
      <div class="control-group">
        <label for="opacity">Opacity:</label>
        <input
          id="opacity"
          type="range"
          min="0.1"
          max="1"
          step="0.1"
          v-model.number="opacity"
          @input="updateOpacity"
        />
        <span>{{ opacity }}</span>
      </div>
      <button @click="resetZoom" class="btn-reset">Reset Zoom</button>
      <button @click="generateSampleData" class="btn-sample">Generate Sample Data</button>
    </div>
    <div class="chart-container" ref="chartContainer" v-if="chartData.length > 0">
      <svg ref="svg" class="umap-svg" v-show="!useCanvas"></svg>
      <canvas ref="canvas" class="umap-canvas" v-show="useCanvas" @mousemove="handleCanvasMouseMove" @mouseleave="handleCanvasMouseLeave"></canvas>
      <div class="tooltip" ref="tooltip"></div>
    </div>
    <div class="legend" v-if="chartData.length > 0 && colorBy !== 'none' && legendItems.length > 0">
      <h4>Legend</h4>
      <div class="legend-items">
        <div
          v-for="item in legendItems"
          :key="item.label"
          class="legend-item"
        >
          <span
            class="legend-color"
            :style="{ backgroundColor: item.color }"
          ></span>
          <span class="legend-label">{{ item.label }}</span>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, watch, nextTick } from 'vue'
import * as d3 from 'd3'

const props = defineProps({
  data: {
    type: Array,
    default: () => []
  },
  width: {
    type: Number,
    default: 800
  },
  height: {
    type: Number,
    default: 600
  }
})

const svg = ref(null)
const canvas = ref(null)
const chartContainer = ref(null)
const tooltip = ref(null)
const umapFileInput = ref(null)
const metadataFileInput = ref(null)
const colorBy = ref('none')
const pointSize = ref(3)
const opacity = ref(0.6)
const isDragOverUMAP = ref(false)
const isDragOverMetadata = ref(false)
const statusMessage = ref('')
const statusType = ref('info')
const useCanvas = ref(false)
const maxSvgPoints = 5000 // Switch to canvas if more points
const umapFile = ref(null)
const metadataFile = ref(null)
const umapData = ref([])
const metadataData = ref([])
const isLoading = ref(false)

let chartData = ref([])
let availableCategories = ref([])
let legendItems = ref([])
let xScale = null
let yScale = null
let zoom = null
let g = null
let points = null
let canvasContext = null
let zoomTransform = d3.zoomIdentity
let animationFrameId = null
let isRendering = false

// Color schemes
const colorSchemes = {
  category10: d3.schemeCategory10,
  category20: d3.schemeCategory20,
  set3: d3.schemeSet3
}

const colorScale = d3.scaleOrdinal(colorSchemes.category10)

// Initialize or update data
watch(() => props.data, (newData) => {
  if (newData && newData.length > 0) {
    chartData.value = newData
    extractCategories()
    nextTick(() => {
      renderChart()
    })
  }
}, { immediate: true, deep: true })

onMounted(() => {
  if (props.data.length === 0) {
    // Don't auto-generate sample data, wait for user to load file
    statusMessage.value = 'Upload a UMAP file and optionally a metadata file, then click "Load Viewer"'
    statusType.value = 'info'
  } else {
    chartData.value = props.data
    extractCategories()
    nextTick(() => {
      renderChart()
    })
  }
})

function extractCategories() {
  if (chartData.value.length === 0) return
  
  const categories = new Set()
  const sample = chartData.value[0]
  
  // Check all keys in first row to determine which are categorical
  Object.keys(sample).forEach(key => {
    if (key === 'x' || key === 'y' || key === 'id') return
    
    const value = sample[key]
    // Include if it's a string, or if it's a number but has few unique values (likely categorical)
    if (typeof value === 'string') {
      categories.add(key)
    } else if (typeof value === 'number') {
      // Check if it's likely categorical (few unique values relative to data size)
      const uniqueValues = new Set(chartData.value.map(d => d[key])).size
      if (uniqueValues < Math.min(20, chartData.value.length * 0.1)) {
        categories.add(key)
      }
    }
  })
  
  // Prioritize common metadata fields
  const priorityFields = ['patient', 'sample', 'cluster', 'type', 'cell_type', 'treatment']
  const sortedCategories = Array.from(categories).sort((a, b) => {
    const aPriority = priorityFields.indexOf(a.toLowerCase())
    const bPriority = priorityFields.indexOf(b.toLowerCase())
    if (aPriority !== -1 && bPriority !== -1) return aPriority - bPriority
    if (aPriority !== -1) return -1
    if (bPriority !== -1) return 1
    return a.localeCompare(b)
  })
  
  availableCategories.value = sortedCategories
}

function generateSampleData() {
  // Generate sample UMAP-like data with clusters
  const n = 1000
  const clusters = [
    { center: [0, 0], n: 300, label: 'Cluster A' },
    { center: [5, 5], n: 250, label: 'Cluster B' },
    { center: [-5, 5], n: 200, label: 'Cluster C' },
    { center: [0, -5], n: 150, label: 'Cluster D' },
    { center: [5, -5], n: 100, label: 'Cluster E' }
  ]
  
  const data = []
  clusters.forEach((cluster, i) => {
    for (let j = 0; j < cluster.n; j++) {
      data.push({
        id: `cell_${i}_${j}`,
        x: cluster.center[0] + (Math.random() - 0.5) * 3,
        y: cluster.center[1] + (Math.random() - 0.5) * 3,
        cluster: cluster.label,
        type: ['Type1', 'Type2', 'Type3'][Math.floor(Math.random() * 3)],
        expression: Math.random() * 100
      })
    }
  })
  
  chartData.value = data
  extractCategories()
  nextTick(() => {
    renderChart()
  })
}

function renderChart() {
  if (chartData.value.length === 0) return

  // Decide rendering method based on data size
  const shouldUseCanvas = chartData.value.length > maxSvgPoints
  useCanvas.value = shouldUseCanvas

  // Wait for DOM to update before rendering
  nextTick(() => {
    // Additional wait to ensure elements are rendered
    setTimeout(() => {
      if (useCanvas.value) {
        // Ensure canvas is available
        if (!canvas.value) {
          console.warn('Canvas not available, retrying...', { 
            useCanvas: useCanvas.value,
            canvasExists: !!canvas.value 
          })
          setTimeout(() => renderChart(), 200)
          return
        }
        renderChartCanvas()
      } else {
        // Ensure SVG is available
        if (!svg.value) {
          console.warn('SVG not available, retrying...', {
            useCanvas: useCanvas.value,
            svgExists: !!svg.value
          })
          setTimeout(() => renderChart(), 200)
          return
        }
        renderChartSvg()
      }
    }, 100)
  })
}

function renderChartSvg() {
  if (!svg.value || chartData.value.length === 0) return

  // Clear previous render
  d3.select(svg.value).selectAll('*').remove()

  const margin = { top: 20, right: 20, bottom: 40, left: 40 }
  const innerWidth = props.width - margin.left - margin.right
  const innerHeight = props.height - margin.top - margin.bottom

  // Set SVG dimensions
  d3.select(svg.value)
    .attr('width', props.width)
    .attr('height', props.height)

  // Create main group
  g = d3.select(svg.value)
    .append('g')
    .attr('transform', `translate(${margin.left},${margin.top})`)

  // Create scales
  const xExtent = d3.extent(chartData.value, d => d.x)
  const yExtent = d3.extent(chartData.value, d => d.y)
  
  xScale = d3.scaleLinear()
    .domain(xExtent)
    .range([0, innerWidth])
    .nice()

  yScale = d3.scaleLinear()
    .domain(yExtent)
    .range([innerHeight, 0])
    .nice()

  // Create axes
  const xAxis = d3.axisBottom(xScale)
  const yAxis = d3.axisLeft(yScale)

  g.append('g')
    .attr('class', 'axis axis-x')
    .attr('transform', `translate(0,${innerHeight})`)
    .call(xAxis)
    .append('text')
    .attr('x', innerWidth / 2)
    .attr('y', 35)
    .attr('fill', 'currentColor')
    .style('text-anchor', 'middle')
    .text('UMAP 1')

  g.append('g')
    .attr('class', 'axis axis-y')
    .call(yAxis)
    .append('text')
    .attr('transform', 'rotate(-90)')
    .attr('y', -30)
    .attr('x', -innerHeight / 2)
    .attr('fill', 'currentColor')
    .style('text-anchor', 'middle')
    .text('UMAP 2')

  // Setup zoom behavior with throttling
  zoom = d3.zoom()
    .scaleExtent([0.1, 10])
    .on('zoom', (event) => {
      zoomTransform = event.transform
      
      if (animationFrameId) {
        cancelAnimationFrame(animationFrameId)
      }
      
      animationFrameId = requestAnimationFrame(() => {
        const newXScale = event.transform.rescaleX(xScale)
        const newYScale = event.transform.rescaleY(yScale)
        
        g.selectAll('.point')
          .attr('cx', d => newXScale(d.x))
          .attr('cy', d => newYScale(d.y))
        
        g.select('.axis-x').call(xAxis.scale(newXScale))
        g.select('.axis-y').call(yAxis.scale(newYScale))
      })
    })

  d3.select(svg.value).call(zoom)

  // Draw points
  updateColors()
}

function renderChartCanvas() {
  if (chartData.value.length === 0) {
    console.warn('Canvas render: no data', { dataLength: chartData.value.length })
    return
  }
  
  if (!canvas.value) {
    console.warn('Canvas render: canvas element not available', { 
      canvas: !!canvas.value, 
      dataLength: chartData.value.length,
      useCanvas: useCanvas.value 
    })
    // Retry after a short delay
    setTimeout(() => {
      if (canvas.value) {
        renderChartCanvas()
      }
    }, 100)
    return
  }

  const margin = { top: 20, right: 20, bottom: 40, left: 40 }
  const innerWidth = props.width - margin.left - margin.right
  const innerHeight = props.height - margin.top - margin.bottom

  // Set canvas dimensions
  canvas.value.width = props.width
  canvas.value.height = props.height
  canvasContext = canvas.value.getContext('2d')
  
  if (!canvasContext) {
    console.error('Failed to get canvas context')
    return
  }

  // Create scales
  const xExtent = d3.extent(chartData.value, d => d.x)
  const yExtent = d3.extent(chartData.value, d => d.y)
  
  // Validate extents
  if (!xExtent[0] || !xExtent[1] || !yExtent[0] || !yExtent[1] || 
      isNaN(xExtent[0]) || isNaN(xExtent[1]) || isNaN(yExtent[0]) || isNaN(yExtent[1])) {
    console.error('Invalid data extents:', { xExtent, yExtent, sampleData: chartData.value.slice(0, 3) })
    statusMessage.value = 'Error: Invalid coordinate data. Please check your file format.'
    statusType.value = 'error'
    return
  }
  
  xScale = d3.scaleLinear()
    .domain(xExtent)
    .range([0, innerWidth])
    .nice()

  yScale = d3.scaleLinear()
    .domain(yExtent)
    .range([innerHeight, 0])
    .nice()

  // Remove any existing axes overlay
  d3.select(chartContainer.value).selectAll('.axes-overlay').remove()
  
  // Create container div for SVG axes overlay
  const axesSvg = d3.select(chartContainer.value)
    .append('svg')
    .attr('class', 'axes-overlay')
    .style('position', 'absolute')
    .style('top', '0')
    .style('left', '0')
    .style('pointer-events', 'none')
    .attr('width', props.width)
    .attr('height', props.height)
  
  g = axesSvg.append('g')
    .attr('transform', `translate(${margin.left},${margin.top})`)

  // Create axes
  const xAxis = d3.axisBottom(xScale)
  const yAxis = d3.axisLeft(yScale)

  g.append('g')
    .attr('class', 'axis axis-x')
    .attr('transform', `translate(0,${innerHeight})`)
    .call(xAxis)
    .append('text')
    .attr('x', innerWidth / 2)
    .attr('y', 35)
    .attr('fill', 'currentColor')
    .style('text-anchor', 'middle')
    .text('UMAP 1')

  g.append('g')
    .attr('class', 'axis axis-y')
    .call(yAxis)
    .append('text')
    .attr('transform', 'rotate(-90)')
    .attr('y', -30)
    .attr('x', -innerHeight / 2)
    .attr('fill', 'currentColor')
    .style('text-anchor', 'middle')
    .text('UMAP 2')

  // Setup zoom behavior
  zoom = d3.zoom()
    .scaleExtent([0.1, 10])
    .on('zoom', (event) => {
      zoomTransform = event.transform
      
      if (animationFrameId) {
        cancelAnimationFrame(animationFrameId)
      }
      
      animationFrameId = requestAnimationFrame(() => {
        const newXScale = event.transform.rescaleX(xScale)
        const newYScale = event.transform.rescaleY(yScale)
        
        drawCanvasPoints(newXScale, newYScale)
        
        g.select('.axis-x').call(xAxis.scale(newXScale))
        g.select('.axis-y').call(yAxis.scale(newYScale))
      })
    })

  d3.select(canvas.value).call(zoom)

  // Initial render
  drawCanvasPoints(xScale, yScale)
}

function updateColors() {
  if (chartData.value.length === 0) return

  if (useCanvas.value) {
    updateColorsCanvas()
  } else {
    updateColorsSvg()
  }
}

function updateColorsSvg() {
  if (!g || chartData.value.length === 0) return

  // Remove existing points
  g.selectAll('.point').remove()

  let colorFn = () => '#333'
  let categories = []

  if (colorBy.value !== 'none' && chartData.value[0][colorBy.value] !== undefined) {
    categories = [...new Set(chartData.value.map(d => d[colorBy.value]))]
    colorScale.domain(categories)
    colorFn = d => colorScale(d[colorBy.value])
    
    // Update legend
    legendItems.value = categories.map(cat => ({
      label: cat,
      color: colorScale(cat)
    }))
  } else {
    legendItems.value = []
  }

  // Draw points
  points = g.selectAll('.point')
    .data(chartData.value)
    .enter()
    .append('circle')
    .attr('class', 'point')
    .attr('cx', d => xScale(d.x))
    .attr('cy', d => yScale(d.y))
    .attr('r', pointSize.value)
    .attr('fill', colorFn)
    .attr('opacity', opacity.value)
    .attr('stroke', '#fff')
    .attr('stroke-width', 0.5)
    .on('mouseover', handleMouseOver)
    .on('mouseout', handleMouseOut)
    .on('click', handleClick)
}

function updateColorsCanvas() {
  if (!canvasContext || chartData.value.length === 0) return

  let categories = []

  if (colorBy.value !== 'none' && chartData.value[0][colorBy.value] !== undefined) {
    categories = [...new Set(chartData.value.map(d => d[colorBy.value]))]
    colorScale.domain(categories)
    
    // Update legend
    legendItems.value = categories.map(cat => ({
      label: cat,
      color: colorScale(cat)
    }))
  } else {
    legendItems.value = []
  }

  // Redraw with new colors
  const currentXScale = zoomTransform.rescaleX(xScale)
  const currentYScale = zoomTransform.rescaleY(yScale)
  drawCanvasPoints(currentXScale, currentYScale)
}

function drawCanvasPoints(xScale, yScale) {
  if (!canvasContext || !canvas.value) {
    console.warn('drawCanvasPoints: missing context or canvas')
    return
  }
  
  if (!xScale || !yScale) {
    console.warn('drawCanvasPoints: missing scales')
    return
  }

  const margin = { top: 20, right: 20, bottom: 40, left: 40 }
  
  // Clear canvas
  canvasContext.clearRect(0, 0, canvas.value.width, canvas.value.height)
  
  // Fill background
  canvasContext.fillStyle = '#ffffff'
  canvasContext.fillRect(0, 0, canvas.value.width, canvas.value.height)
  
  // Translate to account for margins
  canvasContext.save()
  canvasContext.translate(margin.left, margin.top)

  // Get color function
  let colorFn = () => '#333333'
  if (colorBy.value !== 'none' && chartData.value[0] && chartData.value[0][colorBy.value] !== undefined) {
    colorFn = d => colorScale(d[colorBy.value])
  }

  const data = chartData.value
  
  if (data.length === 0) {
    console.warn('drawCanvasPoints: no data to render')
    canvasContext.restore()
    return
  }
  
  // Decimate if too many points (for very large datasets)
  let pointsToRender = data
  if (data.length > 50000) {
    // Sample every Nth point
    const step = Math.ceil(data.length / 50000)
    pointsToRender = data.filter((_, i) => i % step === 0)
  }

  canvasContext.globalAlpha = opacity.value
  canvasContext.strokeStyle = '#ffffff'
  canvasContext.lineWidth = 0.5

  const innerWidth = props.width - margin.left - margin.right
  const innerHeight = props.height - margin.top - margin.bottom

  let renderedCount = 0
  for (let i = 0; i < pointsToRender.length; i++) {
    const d = pointsToRender[i]
    
    // Validate data point
    if (d.x === undefined || d.y === undefined || isNaN(d.x) || isNaN(d.y)) {
      continue
    }
    
    const x = xScale(d.x)
    const y = yScale(d.y)
    
    // Validate scaled coordinates
    if (isNaN(x) || isNaN(y) || !isFinite(x) || !isFinite(y)) {
      continue
    }
    
    // Skip points outside viewport (basic culling)
    if (x < -10 || x > innerWidth + 10 || y < -10 || y > innerHeight + 10) {
      continue
    }

    const color = colorFn(d)
    canvasContext.fillStyle = color
    
    canvasContext.beginPath()
    canvasContext.arc(x, y, pointSize.value, 0, 2 * Math.PI)
    canvasContext.fill()
    canvasContext.stroke()
    renderedCount++
  }

  canvasContext.restore()
  canvasContext.globalAlpha = 1.0
  
  if (renderedCount === 0 && pointsToRender.length > 0) {
    console.warn('drawCanvasPoints: no points rendered', {
      totalPoints: data.length,
      pointsToRender: pointsToRender.length,
      samplePoint: pointsToRender[0],
      xScaleDomain: xScale.domain(),
      yScaleDomain: yScale.domain()
    })
  }
}

function updatePointSize() {
  if (useCanvas.value) {
    const currentXScale = zoomTransform.rescaleX(xScale)
    const currentYScale = zoomTransform.rescaleY(yScale)
    drawCanvasPoints(currentXScale, currentYScale)
  } else if (points) {
    points.attr('r', pointSize.value)
  }
}

function updateOpacity() {
  if (useCanvas.value) {
    const currentXScale = zoomTransform.rescaleX(xScale)
    const currentYScale = zoomTransform.rescaleY(yScale)
    drawCanvasPoints(currentXScale, currentYScale)
  } else if (points) {
    points.attr('opacity', opacity.value)
  }
}

function handleMouseOver(event, d) {
  d3.select(event.currentTarget)
    .attr('stroke-width', 2)
    .attr('stroke', '#000')
    .raise()

  const tooltipEl = d3.select(tooltip.value)
  const tooltipContent = Object.keys(d)
    .filter(key => key !== 'x' && key !== 'y')
    .map(key => `<strong>${key}:</strong> ${d[key]}`)
    .join('<br>')

  tooltipEl
    .style('opacity', 1)
    .html(tooltipContent)
    .style('left', (event.pageX + 10) + 'px')
    .style('top', (event.pageY - 10) + 'px')
}

function handleMouseOut(event) {
  d3.select(event.currentTarget)
    .attr('stroke-width', 0.5)
    .attr('stroke', '#fff')

  d3.select(tooltip.value).style('opacity', 0)
}

function handleClick(event, d) {
  console.log('Clicked point:', d)
}

function handleCanvasMouseMove(event) {
  if (!canvas.value || !canvasContext || chartData.value.length === 0) return
  
  const rect = canvas.value.getBoundingClientRect()
  const margin = { top: 20, right: 20, bottom: 40, left: 40 }
  
  // Calculate mouse position in canvas coordinates
  const scaleX = props.width / rect.width
  const scaleY = props.height / rect.height
  const x = (event.clientX - rect.left - margin.left) * scaleX
  const y = (event.clientY - rect.top - margin.top) * scaleY
  
  const currentXScale = zoomTransform.rescaleX(xScale)
  const currentYScale = zoomTransform.rescaleY(yScale)
  
  // Find nearest point (limit search for performance)
  let nearestPoint = null
  let minDist = Infinity
  const searchRadius = pointSize.value * 5
  
  // Only search through a subset for very large datasets
  const searchData = chartData.value.length > 10000 
    ? chartData.value.filter((_, i) => i % Math.ceil(chartData.value.length / 10000) === 0)
    : chartData.value
  
  for (const d of searchData) {
    const px = currentXScale(d.x)
    const py = currentYScale(d.y)
    const dist = Math.sqrt((x - px) ** 2 + (y - py) ** 2)
    
    if (dist < searchRadius && dist < minDist) {
      minDist = dist
      nearestPoint = d
    }
  }
  
  if (nearestPoint) {
    const tooltipEl = d3.select(tooltip.value)
    const tooltipContent = Object.keys(nearestPoint)
      .filter(key => key !== 'x' && key !== 'y')
      .map(key => `<strong>${key}:</strong> ${nearestPoint[key]}`)
      .join('<br>')
    
    tooltipEl
      .style('opacity', 1)
      .html(tooltipContent)
      .style('left', (event.pageX + 10) + 'px')
      .style('top', (event.pageY - 10) + 'px')
  } else {
    d3.select(tooltip.value).style('opacity', 0)
  }
}

function handleCanvasMouseLeave() {
  d3.select(tooltip.value).style('opacity', 0)
}

function resetZoom() {
  if (zoom) {
    const target = useCanvas.value ? canvas.value : svg.value
    if (target) {
      d3.select(target)
        .transition()
        .duration(750)
        .call(zoom.transform, d3.zoomIdentity)
      
      zoomTransform = d3.zoomIdentity
      
      nextTick(() => {
        if (useCanvas.value) {
          drawCanvasPoints(xScale, yScale)
        } else {
          updateColors()
        }
      })
    }
  }
}

function openUMAPFileDialog() {
  umapFileInput.value?.click()
}

function openMetadataFileDialog() {
  metadataFileInput.value?.click()
}

function handleUMAPFileSelect(event) {
  const file = event.target.files[0]
  if (file) {
    umapFile.value = file
    statusMessage.value = `UMAP file selected: ${file.name}`
    statusType.value = 'info'
  }
}

function handleMetadataFileSelect(event) {
  const file = event.target.files[0]
  if (file) {
    metadataFile.value = file
    statusMessage.value = `Metadata file selected: ${file.name}`
    statusType.value = 'info'
  }
}

function handleDropUMAP(event) {
  event.preventDefault()
  isDragOverUMAP.value = false
  
  const file = event.dataTransfer.files[0]
  if (file) {
    umapFile.value = file
    statusMessage.value = `UMAP file selected: ${file.name}`
    statusType.value = 'info'
  }
}

function handleDropMetadata(event) {
  event.preventDefault()
  isDragOverMetadata.value = false
  
  const file = event.dataTransfer.files[0]
  if (file) {
    metadataFile.value = file
    statusMessage.value = `Metadata file selected: ${file.name}`
    statusType.value = 'info'
  }
}

function clearUMAPFile() {
  umapFile.value = null
  umapData.value = []
  chartData.value = []
  if (umapFileInput.value) {
    umapFileInput.value.value = ''
  }
}

function clearMetadataFile() {
  metadataFile.value = null
  metadataData.value = []
  if (metadataFileInput.value) {
    metadataFileInput.value.value = ''
  }
}

async function loadViewer() {
  if (!umapFile.value) {
    statusMessage.value = 'Please upload a UMAP file first'
    statusType.value = 'error'
    return
  }

  isLoading.value = true
  statusMessage.value = 'Loading files...'
  statusType.value = 'info'

  try {
    // Load UMAP file
    await loadUMAPFile(umapFile.value)
    
    // Load metadata file if provided
    if (metadataFile.value) {
      await loadMetadataFile(metadataFile.value)
      // Map metadata to UMAP data
      mapMetadataToUMAP()
    }
    
    // Normalize and prepare data
    const normalizedData = normalizeData(umapData.value)
    
    if (normalizedData.length === 0) {
      throw new Error('No valid data points found in UMAP file')
    }
    
    chartData.value = normalizedData
    extractCategories()
    colorBy.value = 'none' // Reset color selection
    
    statusMessage.value = `Loaded ${normalizedData.length} data points successfully!`
    statusType.value = 'success'
    
    // Clear status message after 3 seconds
    setTimeout(() => {
      statusMessage.value = ''
    }, 3000)
    
    // Wait for DOM to update, then render
    nextTick(() => {
      setTimeout(() => {
        renderChart()
      }, 50)
    })
  } catch (error) {
    statusMessage.value = `Error loading files: ${error.message}`
    statusType.value = 'error'
    console.error('Error loading files:', error)
  } finally {
    isLoading.value = false
  }
}

function loadUMAPFile(file) {
  return new Promise((resolve, reject) => {
    const fileName = file.name.toLowerCase()
    const reader = new FileReader()
    
    reader.onload = (e) => {
      try {
        let data = null
        
        if (fileName.endsWith('.json')) {
          data = JSON.parse(e.target.result)
          if (!Array.isArray(data)) {
            throw new Error('JSON file must contain an array of objects')
          }
        } else {
          const text = e.target.result
          const delimiter = fileName.endsWith('.tsv') || fileName.endsWith('.txt') ? '\t' : ','
          data = parseDelimitedFile(text, delimiter)
        }
        
        umapData.value = data
        resolve()
      } catch (error) {
        reject(error)
      }
    }
    
    reader.onerror = () => {
      reject(new Error('Error reading UMAP file'))
    }
    
    reader.readAsText(file)
  })
}

function loadMetadataFile(file) {
  return new Promise((resolve, reject) => {
    const fileName = file.name.toLowerCase()
    const reader = new FileReader()
    
    reader.onload = (e) => {
      try {
        let data = null
        
        if (fileName.endsWith('.json')) {
          data = JSON.parse(e.target.result)
          if (!Array.isArray(data)) {
            throw new Error('JSON file must contain an array of objects')
          }
        } else {
          const text = e.target.result
          const delimiter = fileName.endsWith('.tsv') || fileName.endsWith('.txt') ? '\t' : ','
          data = parseDelimitedFile(text, delimiter)
        }
        
        metadataData.value = data
        resolve()
      } catch (error) {
        reject(error)
      }
    }
    
    reader.onerror = () => {
      reject(new Error('Error reading metadata file'))
    }
    
    reader.readAsText(file)
  })
}

function mapMetadataToUMAP() {
  if (metadataData.value.length === 0 || umapData.value.length === 0) {
    return
  }
  
  // Get first column name from metadata
  const metadataFirstRow = metadataData.value[0]
  const metadataKeys = Object.keys(metadataFirstRow)
  const metadataIdKey = metadataKeys[0] // First column
  
  // Get first column name from UMAP data
  const umapFirstRow = umapData.value[0]
  const umapKeys = Object.keys(umapFirstRow)
  const umapIdKey = umapKeys[0] // First column
  
  // Create a map of metadata by ID
  const metadataMap = new Map()
  metadataData.value.forEach(row => {
    const id = String(row[metadataIdKey])
    if (id) {
      metadataMap.set(id, row)
    }
  })
  
  // Merge metadata into UMAP data
  umapData.value.forEach(umapRow => {
    const id = String(umapRow[umapIdKey])
    if (id && metadataMap.has(id)) {
      const metadataRow = metadataMap.get(id)
      // Add all metadata columns except the ID column
      Object.keys(metadataRow).forEach(key => {
        if (key !== metadataIdKey) {
          umapRow[key] = metadataRow[key]
        }
      })
    }
  })
  
  statusMessage.value = `Mapped ${metadataMap.size} metadata entries to UMAP data`
  statusType.value = 'success'
}

function parseDelimitedFile(text, delimiter) {
  const lines = text.split('\n').filter(line => line.trim())
  if (lines.length === 0) return []
  
  // Parse header - optimized for common formats
  const headerLine = lines[0]
  const headers = headerLine.split(delimiter).map(h => {
    const trimmed = h.trim()
    return trimmed.replace(/^["']|["']$/g, '') // Remove quotes
  })
  
  // Pre-determine column indices for performance
  const xColIndex = headers.findIndex(h => 
    h === 'umapharmony_1' || h === 'UMAP_1' || h === 'umap_1' || h === 'x' || h === 'X'
  )
  const yColIndex = headers.findIndex(h => 
    h === 'umapharmony_2' || h === 'UMAP_2' || h === 'umap_2' || h === 'y' || h === 'Y'
  )
  const idColIndex = headers.findIndex(h => 
    h && (h.toLowerCase() === 'id' || h.toLowerCase().includes('cell') || h.toLowerCase().includes('barcode'))
  ) || 0 // Default to first column if no ID column found
  
  // Parse data rows with optimized parsing
  const data = []
  const numHeaders = headers.length
  
  for (let i = 1; i < lines.length; i++) {
    const line = lines[i]
    if (!line.trim()) continue
    
    const values = line.split(delimiter)
    if (values.length < numHeaders) continue
    
    const row = {}
    
    // Process all columns
    for (let j = 0; j < numHeaders; j++) {
      const header = headers[j]
      if (!header) continue // Skip empty headers
      
      let value = values[j]?.trim().replace(/^["']|["']$/g, '') || ''
      
      // Optimize number parsing - only parse if it looks like a number
      if (value && !isNaN(value) && value !== '') {
        const numValue = parseFloat(value)
        row[header] = isNaN(numValue) ? value : numValue
      } else {
        row[header] = value
      }
    }
    
    // Validate that we have coordinates
    if (xColIndex >= 0 && yColIndex >= 0) {
      const xVal = row[headers[xColIndex]]
      const yVal = row[headers[yColIndex]]
      if (xVal !== undefined && yVal !== undefined && !isNaN(xVal) && !isNaN(yVal)) {
        data.push(row)
      }
    } else {
      // Fallback: try to find numeric columns
      const xVal = row[headers[xColIndex]] ?? (typeof row[headers[1]] === 'number' ? row[headers[1]] : parseFloat(row[headers[1]]))
      const yVal = row[headers[yColIndex]] ?? (typeof row[headers[2]] === 'number' ? row[headers[2]] : parseFloat(row[headers[2]]))
      if (!isNaN(xVal) && !isNaN(yVal)) {
        data.push(row)
      }
    }
  }
  
  return data
}

function normalizeData(data) {
  if (!Array.isArray(data) || data.length === 0) return []
  
  // Optimized: prioritize umapharmony columns first (most common in this format)
  const xColumnNames = ['umapharmony_1', 'umap_1', 'UMAP_1', 'UMAP1', 'x', 'X', 'umap1', 'umap_x', 'dim1', 'dimension1']
  const yColumnNames = ['umapharmony_2', 'umap_2', 'UMAP_2', 'UMAP2', 'y', 'Y', 'umap2', 'umap_y', 'dim2', 'dimension2']
  
  const firstRow = data[0]
  const allKeys = Object.keys(firstRow)
  
  // Find x column (prioritize umapharmony_1)
  let xCol = xColumnNames.find(name => allKeys.includes(name))
  if (!xCol) {
    // Try to find column with 'umap' and '1' in name (case insensitive)
    xCol = allKeys.find(key => {
      const lower = key.toLowerCase()
      return (lower.includes('umap') && (lower.includes('1') || lower.endsWith('_1'))) ||
             lower === 'x' || lower.includes('dimension1') || lower.includes('dim1')
    })
  }
  
  // Find y column (prioritize umapharmony_2)
  let yCol = yColumnNames.find(name => allKeys.includes(name))
  if (!yCol) {
    // Try to find column with 'umap' and '2' in name (case insensitive)
    yCol = allKeys.find(key => {
      const lower = key.toLowerCase()
      return (lower.includes('umap') && (lower.includes('2') || lower.endsWith('_2'))) ||
             lower === 'y' || lower.includes('dimension2') || lower.includes('dim2')
    })
  }
  
  if (!xCol || !yCol) {
    // If still not found, try first two numeric columns
    const numericCols = allKeys.filter(key => {
      const val = firstRow[key]
      return val !== null && val !== undefined && val !== '' && 
             (typeof val === 'number' || !isNaN(parseFloat(val)))
    })
    
    if (numericCols.length >= 2) {
      xCol = numericCols[0]
      yCol = numericCols[1]
    } else {
      throw new Error(`Could not find UMAP coordinates. Expected columns named: umapharmony_1/umapharmony_2, UMAP_1/UMAP_2, or similar. Found columns: ${allKeys.join(', ')}`)
    }
  }
  
  // Find ID column - check first column or look for ID/barcode columns
  const idColumnNames = ['id', 'ID', 'cell_id', 'barcode', 'Barcode', 'cell_barcode']
  let idCol = idColumnNames.find(name => allKeys.includes(name))
  if (!idCol) {
    // Check if first column looks like an ID (not numeric)
    const firstKey = allKeys[0]
    const firstVal = firstRow[firstKey]
    if (firstVal !== null && firstVal !== undefined && 
        (typeof firstVal === 'string' || (typeof firstVal !== 'number' && isNaN(parseFloat(firstVal))))) {
      idCol = firstKey
    }
  }
  
  // Normalize data with optimized processing
  const normalized = []
  const xColKey = xCol
  const yColKey = yCol
  const idColKey = idCol
  
  for (let i = 0; i < data.length; i++) {
    const row = data[i]
    
    // Extract coordinates
    const x = typeof row[xColKey] === 'number' ? row[xColKey] : parseFloat(row[xColKey])
    const y = typeof row[yColKey] === 'number' ? row[yColKey] : parseFloat(row[yColKey])
    
    // Skip invalid coordinates
    if (isNaN(x) || isNaN(y) || !isFinite(x) || !isFinite(y)) {
      continue
    }
    
    // Extract ID
    let id = `point_${i}`
    if (idColKey && row[idColKey] !== undefined && row[idColKey] !== null && row[idColKey] !== '') {
      id = String(row[idColKey])
    } else if (row.id !== undefined) {
      id = String(row.id)
    } else if (row.ID !== undefined) {
      id = String(row.ID)
    }
    
    // Extract metadata from ID if it looks like "Patient1-Infusion_..."
    let patient = null
    let sample = null
    if (id.includes('-')) {
      const parts = id.split('-')
      if (parts.length >= 2) {
        patient = parts[0] // e.g., "Patient1"
        if (parts[1].includes('_')) {
          sample = parts[1].split('_')[0] // e.g., "Infusion"
        }
      }
    }
    
    const normalizedRow = {
      id,
      x,
      y
    }
    
    // Add extracted metadata if available
    if (patient) normalizedRow.patient = patient
    if (sample) normalizedRow.sample = sample
    
    // Copy all other columns (excluding coordinates and ID)
    for (const key of allKeys) {
      if (key !== xColKey && key !== yColKey && key !== idColKey && 
          key !== 'id' && key !== 'ID' && row[key] !== undefined && row[key] !== null && row[key] !== '') {
        normalizedRow[key] = row[key]
      }
    }
    
    normalized.push(normalizedRow)
  }
  
  if (normalized.length === 0) {
    throw new Error('No valid data points found after normalization. Please check your file format.')
  }
  
  return normalized
}
</script>

<style scoped>
.umap-viewer {
  display: flex;
  flex-direction: column;
  gap: 20px;
  padding: 20px;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, sans-serif;
}

.controls {
  display: flex;
  flex-wrap: wrap;
  gap: 20px;
  align-items: center;
  padding: 15px;
  background: #f5f5f5;
  border-radius: 8px;
}

.control-group {
  display: flex;
  align-items: center;
  gap: 10px;
}

.control-group label {
  font-weight: 500;
  color: #333;
}

.control-group select,
.control-group input[type="range"] {
  padding: 5px 10px;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 14px;
}

.control-group input[type="range"] {
  width: 100px;
}

.btn-reset,
.btn-sample {
  padding: 8px 16px;
  background: #007bff;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 14px;
  transition: background 0.2s;
}

.btn-reset:hover,
.btn-sample:hover {
  background: #0056b3;
}

.chart-container {
  position: relative;
  border: 1px solid #ddd;
  border-radius: 8px;
  background: white;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  overflow: hidden;
}

.umap-svg,
.umap-canvas {
  display: block;
  width: 100%;
  height: auto;
}

.umap-canvas {
  cursor: grab;
}

.umap-canvas:active {
  cursor: grabbing;
}

.axes-overlay {
  pointer-events: none;
  z-index: 1;
}

.tooltip {
  position: absolute;
  padding: 10px;
  background: rgba(0, 0, 0, 0.8);
  color: white;
  border-radius: 4px;
  pointer-events: none;
  opacity: 0;
  transition: opacity 0.2s;
  font-size: 12px;
  z-index: 1000;
  max-width: 200px;
}

.tooltip strong {
  color: #fff;
}

.legend {
  padding: 15px;
  background: #f9f9f9;
  border-radius: 8px;
  border: 1px solid #ddd;
}

.legend h4 {
  margin: 0 0 10px 0;
  font-size: 14px;
  color: #333;
}

.legend-items {
  display: flex;
  flex-wrap: wrap;
  gap: 15px;
}

.legend-item {
  display: flex;
  align-items: center;
  gap: 8px;
}

.legend-color {
  width: 16px;
  height: 16px;
  border-radius: 3px;
  border: 1px solid #ddd;
}

.legend-label {
  font-size: 12px;
  color: #666;
}

:deep(.axis) {
  font-size: 12px;
}

:deep(.axis path),
:deep(.axis line) {
  fill: none;
  stroke: #333;
  shape-rendering: crispEdges;
}

:deep(.axis text) {
  fill: #666;
}

.file-upload-section {
  margin-bottom: 20px;
}

.upload-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 20px;
  margin-bottom: 20px;
}

.upload-item h3 {
  margin: 0 0 10px 0;
  font-size: 16px;
  color: #333;
  font-weight: 600;
}

.file-drop-zone {
  border: 2px dashed #ccc;
  border-radius: 8px;
  padding: 30px;
  text-align: center;
  background: #fafafa;
  transition: all 0.3s ease;
  cursor: pointer;
  min-height: 150px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.file-drop-zone:hover,
.file-drop-zone.drag-over {
  border-color: #007bff;
  background: #f0f7ff;
}

.file-drop-zone.has-file {
  border-color: #28a745;
  background: #f0fff4;
}

.drop-zone-content {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 10px;
  color: #666;
}

.drop-zone-content svg {
  color: #999;
  margin-bottom: 10px;
}

.drop-zone-text {
  margin: 0;
  font-size: 16px;
}

.drop-zone-hint {
  margin: 0;
  font-size: 12px;
  color: #999;
}

.btn-link {
  background: none;
  border: none;
  color: #007bff;
  text-decoration: underline;
  cursor: pointer;
  font-size: inherit;
  padding: 0;
  font-weight: inherit;
}

.btn-link:hover {
  color: #0056b3;
}

.status-message {
  margin-top: 10px;
  padding: 12px 16px;
  border-radius: 4px;
  font-size: 14px;
}

.status-message.info {
  background: #e7f3ff;
  color: #0066cc;
  border: 1px solid #b3d9ff;
}

.status-message.success {
  background: #d4edda;
  color: #155724;
  border: 1px solid #c3e6cb;
}

.status-message.error {
  background: #f8d7da;
  color: #721c24;
  border: 1px solid #f5c6cb;
}

.load-button-container {
  display: flex;
  justify-content: center;
  margin: 20px 0;
}

.btn-load {
  padding: 12px 32px;
  background: #28a745;
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-size: 16px;
  font-weight: 600;
  transition: all 0.2s;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.btn-load:hover:not(:disabled) {
  background: #218838;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.15);
  transform: translateY(-1px);
}

.btn-load:disabled {
  background: #6c757d;
  cursor: not-allowed;
  opacity: 0.6;
}

@media (max-width: 768px) {
  .upload-grid {
    grid-template-columns: 1fr;
  }
}
</style>
