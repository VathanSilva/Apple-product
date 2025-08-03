<template>
  <div class="apple-product-page">

    <section class="product-showcase">
      <div class="showcase-container">
        <div class="gallery-wrapper">
          <div class="image-gallery" ref="mainCarousel">
            <div class="gallery-viewport">
              <div class="gallery-track" ref="carouselTrack" :style="{ transform: `translateX(-${currentSlide * 100}%)` }">
                <div 
                  v-for="(image, index) in productImages" 
                  :key="index"
                  class="gallery-slide" 
                  :data-slide-index="index"
                >
                  <div class="image-container">
                    <img :src="image.url" :alt="image.alt" />
                    <div 
                      class="measurement-overlay" 
                      :class="{ 'active': measurementMode }"
                      :data-slide="index"
                      @click="handleOverlayClick"
                      @touchstart="handleOverlayClick"
                    >
                      <div
                        v-for="circle in getCirclesForSlide(index)"
                        :key="circle.id"
                        class="measurement-point"
                        :class="{ 
                          'selected': selectedCircle?.id === circle.id,
                          'dragging': draggedCircle?.id === circle.id 
                        }"
                        :style="{ left: circle.x + 'px', top: circle.y + 'px' }"
                        @mousedown="handleCircleMouseDown($event, circle)"
                        @touchstart="handleCircleMouseDown($event, circle)"
                        @click.stop="selectCircle(circle)"
                      >
                        <div class="point-inner">
                          <span class="point-label">{{ circle.id }}</span>
                        </div>
                      </div>
                    </div>
                  </div>
                </div>
              </div>
            </div>
            
            <button class="gallery-nav prev" @click="prevSlide" :disabled="currentSlide === 0">
              <svg viewBox="0 0 24 24">
                <path d="M15.41 7.41L14 6l-6 6 6 6 1.41-1.41L10.83 12z"/>
              </svg>
            </button>
            <button class="gallery-nav next" @click="nextSlide" :disabled="currentSlide === totalSlides - 1">
              <svg viewBox="0 0 24 24">
                <path d="M8.59 16.59L10 18l6-6-6-6-1.41 1.41L13.17 12z"/>
              </svg>
            </button>
            
            <div class="gallery-thumbnails">
              <div
                v-for="(image, index) in productImages"
                :key="index"
                class="thumbnail"
                :class="{ active: index === currentSlide }"
                @click="goToSlide(index)"
              >
                <img :src="image.url" :alt="image.alt" />
              </div>
            </div>
          </div>
        </div>
        
        <div class="product-config">
          <div class="product-info">
            <h1 class="product-title">{{ product.title }}</h1>
            <p class="product-description">{{ product.description }}</p>
            <div class="product-price">From ${{ product.price }}</div>
          </div>

          <div class="config-section measurement-section">
            <div class="section-header">
              <h2 class="section-title">Precision Measurement</h2>
              <p class="section-description">Mark and measure specific points on your product with pixel-perfect accuracy</p>
            </div>
            
            <div class="control-group">
              <div class="toggle-control">
                <label class="toggle-label">
                  <span class="label-text">Measurement Mode</span>
                  <div 
                    class="toggle-switch" 
                    :class="{ active: measurementMode }"
                    @click="toggleMeasurementMode"
                  >
                    <div class="toggle-slider">
                      <div class="toggle-icon">
                        <svg v-if="!measurementMode" viewBox="0 0 24 24">
                          <path d="M12 2C6.48 2 2 6.48 2 12s4.48 10 10 10 10-4.48 10-10S17.52 2 12 2zm-2 15l-5-5 1.41-1.41L10 14.17l7.59-7.59L19 8l-9 9z"/>
                        </svg>
                        <svg v-else viewBox="0 0 24 24">
                          <path d="M19 6.41L17.59 5 12 10.59 6.41 5 5 6.41 10.59 12 5 17.59 6.41 19 12 13.41 17.59 19 19 17.59 13.41 12z"/>
                        </svg>
                      </div>
                    </div>
                  </div>
                </label>
              </div>
            </div>
            
            <div class="coordinate-panel" :class="{ disabled: !measurementMode }">
              <h3 class="panel-title">Coordinate Input</h3>
              <div class="coordinate-grid">
                <div class="input-field">
                  <label class="field-label">X Position</label>
                  <input 
                    type="number" 
                    class="field-input" 
                    v-model.number="xInput" 
                    placeholder="0" 
                    min="0"
                    :disabled="!measurementMode"
                    @input="updateFromInputs"
                  >
                  <span class="field-unit">px</span>
                </div>
                <div class="input-field">
                  <label class="field-label">Y Position</label>
                  <input 
                    type="number" 
                    class="field-input" 
                    v-model.number="yInput" 
                    placeholder="0" 
                    min="0"
                    :disabled="!measurementMode"
                    @input="updateFromInputs"
                  >
                  <span class="field-unit">px</span>
                </div>
              </div>
              <button 
                class="action-button primary" 
                :disabled="!measurementMode"
                @click="addCircleFromInputs"
              >
                <svg viewBox="0 0 24 24">
                  <path d="M19 13h-6v6h-2v-6H5v-2h6V5h2v6h6v2z"/>
                </svg>
                Add Measurement Point
              </button>
            </div>
            
            <div class="points-panel" v-if="Object.keys(circlesBySlide).length > 0">
              <h3 class="panel-title">Measurement Points</h3>
              <div class="points-list">
                <template v-for="slideIndex in Object.keys(circlesBySlide)" :key="slideIndex">
                  <div class="slide-group">
                    <div class="slide-header">
                      <span class="slide-label">Image {{ parseInt(slideIndex) + 1 }}</span>
                      <span class="point-count">{{ circlesBySlide[slideIndex].length }} points</span>
                    </div>
                    <div class="points-grid">
                      <div
                        v-for="circle in circlesBySlide[slideIndex]"
                        :key="circle.id"
                        class="point-item"
                        :class="{ selected: selectedCircle?.id === circle.id }"
                        @click="selectCircle(circle)"
                      >
                        <div class="point-info">
                          <div class="point-id">Point {{ circle.id }}</div>
                          <div class="point-coords">
                            {{ Math.round(circle.x) }}, {{ Math.round(circle.y) }}px
                          </div>
                          <div class="point-percentage">
                            {{ getPercentageCoords(circle).x }}%, {{ getPercentageCoords(circle).y }}%
                          </div>
                        </div>
                        <button class="delete-point" @click.stop="deleteCircle(circle.id)">
                          <svg viewBox="0 0 24 24">
                            <path d="M19 6.41L17.59 5 12 10.59 6.41 5 5 6.41 10.59 12 5 17.59 6.41 19 12 13.41 17.59 19 19 17.59 13.41 12z"/>
                          </svg>
                        </button>
                      </div>
                    </div>
                  </div>
                </template>
              </div>
            </div>
          </div>
          
          <div class="config-section material-section">
            <div class="section-header">
              <h2 class="section-title">Material Selection</h2>
              <p class="section-description">Choose from our premium materials crafted for performance and durability</p>
            </div>
            
            <div class="materials-grid">
              <div
                v-for="material in materials"
                :key="material.id"
                class="material-card"
                :class="{ selected: selectedMaterial === material.id }"
                @click="selectMaterial(material.id)"
              >
                <div class="material-content">
                  <h3 class="material-name">{{ material.name }}</h3>
                  <p class="material-description">{{ material.description }}</p>
                </div>
                <div class="material-indicator">
                  <div class="selection-check">
                    <svg viewBox="0 0 24 24">
                      <path d="M9 16.17L4.83 12l-1.42 1.41L9 19 21 7l-1.41-1.41z"/>
                    </svg>
                  </div>
                </div>
              </div>
            </div>
          </div>
          
          <div class="action-section">
            <button class="action-button large primary" @click="submitData">
              <svg viewBox="0 0 24 24">
                <path d="M9 16.17L4.83 12l-1.42 1.41L9 19 21 7l-1.41-1.41z"/>
              </svg>
              Submit Configuration
            </button>
            <button class="action-button large secondary" @click="addToCart">
              <svg viewBox="0 0 24 24">
                <path d="M7 18c-1.1 0-2 .9-2 2s.9 2 2 2 2-.9 2-2-.9-2-2-2zM1 2v2h2l3.6 7.59-1.35 2.45c-.16.28-.25.61-.25.96 0 1.1.9 2 2 2h12v-2H7.42c-.14 0-.25-.11-.25-.25l.03-.12L8.1 13h7.45c.75 0 1.41-.41 1.75-1.03L21.7 4H5.21l-.94-2H1zm16 16c-1.1 0-2 .9-2 2s.9 2 2 2 2-.9 2-2-.9-2-2-2z"/>
              </svg>
              Add to Cart
            </button>
          </div>
        </div>
      </div>
    </section>
  </div>
</template>

<script>
import { computed, onMounted, onUnmounted, reactive, ref } from 'vue'

export default {
  name: 'AppleProductPage',
  setup() {
    const currentSlide = ref(0)
    const measurementMode = ref(false)
    const circles = ref([])
    const selectedMaterial = ref(null)
    const selectedCircle = ref(null)
    const draggedCircle = ref(null)
    const circleCounter = ref(0)
    const xInput = ref('')
    const yInput = ref('')
    const isDragging = ref(false)
    
    const mainCarousel = ref(null)
    const carouselTrack = ref(null)
    
    const CIRCLE_RADIUS = 25 
    
    const product = reactive({
      title: 'iPhone 16 Pro',
      description: 'Your new iPhone 16 Pro Max. Just the way you want it.',
      price: 1199
    })
    
    const productImages = ref([
      {
        url: 'https://store.storeimages.cdn-apple.com/1/as-images.apple.com/is/iphone-16-pro-finish-select-202409-6-9inch_GEO_US?wid=5120&hei=2880&fmt=webp&qlt=90&.v=eUdsd0dIb3VUOXdtWkY0VFUwVE8vbEdkZHNlSjBQRklnaFB2d3I5MW94NUZqUFNQc3E5VDh2SEx1ZlJpSjNkR0FOL1haWCt6TDJ0UWlLb09XajVNdENYR1ZZZnEyMVlVQUliTThGMjNyaFFDdURYTkVwaGQzbSsxZFlGem8yMnNkU3dydy9HM2lySFhSSEZkUTBtb0tR&traceId=1',
        alt: 'iPhone 16 Pro'
      },
      {
        url: 'https://store.storeimages.cdn-apple.com/1/as-images.apple.com/is/iphone-16-pro-finish-select-202409-6-9inch_AV1?wid=5120&hei=2880&fmt=webp&qlt=90&.v=eUdsd0dIb3VUOXdtWkY0VFUwVE8vbEdkZHNlSjBQRklnaFB2d3I5MW94NENxZ2Y2UndFVkhoZG1DQ0NWVTFWa2xjZnhHRHJyenVmME5KTm9Sd1ZaU3NqbWRhTGpRM2xxVWJRWUhSaDlCQ3FBdHR2dFhwdGUzbmtydVhBa0gxTE5GQlRRRytmTHd2bTlDU2RZMUtqVTRR&traceId=1',
        alt: 'iPhone 16 Pro'
      },
      {
        url: 'https://store.storeimages.cdn-apple.com/1/as-images.apple.com/is/iphone-16-pro-finish-select-202409-6-9inch_AV2?wid=5120&hei=2880&fmt=webp&qlt=90&.v=eUdsd0dIb3VUOXdtWkY0VFUwVE8vbEdkZHNlSjBQRklnaFB2d3I5MW94NUYwNEIwcFZiSzRpTGtjOVdwSCt4N2xjZnhHRHJyenVmME5KTm9Sd1ZaU3NqbWRhTGpRM2xxVWJRWUhSaDlCQ29INlpMOUhoWTNUbUIrQVc3a2xXUGdRRXB6YUhNcms5Z1pYZXdkbUhocnBn&traceId=1',
        alt: 'iPhone 16 Pro'
      },
      {
        url: 'https://store.storeimages.cdn-apple.com/1/as-images.apple.com/is/iphone-16-pro-finish-select-202409-6-9inch_AV3?wid=5120&hei=2880&fmt=webp&qlt=90&.v=eUdsd0dIb3VUOXdtWkY0VFUwVE8vbEdkZHNlSjBQRklnaFB2d3I5MW94NmMyUjc1U3JwZ2VscGtUR0tSYnFmSGxjZnhHRHJyenVmME5KTm9Sd1ZaU3NqbWRhTGpRM2xxVWJRWUhSaDlCQ294WmV3am9pcVFFaG9jVWxZNFpSejdsWG55WFNZREIrcHRwdlRvMGw2S3BR&traceId=1',
        alt: 'iPhone 16 Pro'
      }
    ])
    
    const materials = ref([
      {
        id: 'aluminum',
        name: 'Premium Aluminum',
        description: 'Lightweight aerospace-grade aluminum with anodized finish for exceptional durability and style.'
      },
      {
        id: 'steel',
        name: 'Stainless Steel',
        description: 'Medical-grade stainless steel offering superior strength and corrosion resistance.'
      },
      {
        id: 'titanium',
        name: 'Titanium Alloy',
        description: 'Ultra-lightweight titanium alloy with unmatched strength-to-weight ratio.'
      },
      {
        id: 'carbon-fiber',
        name: 'Carbon Fiber',
        description: 'Advanced carbon fiber composite offering maximum strength with minimal weight.'
      }
    ])
    
    const totalSlides = computed(() => productImages.value.length)
    
    const circlesBySlide = computed(() => {
      const grouped = {}
      circles.value.forEach(circle => {
        if (!grouped[circle.slideIndex]) {
          grouped[circle.slideIndex] = []
        }
        grouped[circle.slideIndex].push(circle)
      })
      return grouped
    })
    
    const getEventCoords = (event) => {
      if (event.touches && event.touches.length > 0) {
        return {
          x: event.touches[0].clientX,
          y: event.touches[0].clientY
        }
      }
      return {
        x: event.clientX,
        y: event.clientY
      }
    }
    
    const constrainCoordinates = (x, y, overlayRect) => {
      const constrainedX = Math.max(CIRCLE_RADIUS, Math.min(overlayRect.width - CIRCLE_RADIUS, x))
      const constrainedY = Math.max(CIRCLE_RADIUS, Math.min(overlayRect.height - CIRCLE_RADIUS, y))
      return { x: constrainedX, y: constrainedY }
    }
    
    const goToSlide = (slideIndex) => {
      currentSlide.value = slideIndex
    }
    
    const nextSlide = () => {
      if (currentSlide.value < totalSlides.value - 1) {
        currentSlide.value++
      }
    }
    
    const prevSlide = () => {
      if (currentSlide.value > 0) {
        currentSlide.value--
      }
    }
    
    const toggleMeasurementMode = () => {
      measurementMode.value = !measurementMode.value
      if (!measurementMode.value) {
        selectedCircle.value = null
        xInput.value = ''
        yInput.value = ''
      }
    }
    
    const getCirclesForSlide = (slideIndex) => {
      return circles.value.filter(circle => circle.slideIndex === slideIndex)
    }
    
    const getPercentageCoords = (circle) => {
      const overlay = document.querySelector(`.measurement-overlay[data-slide="${circle.slideIndex}"]`)
      if (!overlay) return { x: 0, y: 0 }
      
      const rect = overlay.getBoundingClientRect()
      return {
        x: parseFloat(((circle.x / rect.width) * 100).toFixed(1)),
        y: parseFloat(((circle.y / rect.height) * 100).toFixed(1))
      }
    }
    
    const checkCollision = (x, y, slideIndex, excludeId = null) => {
      const minDistance = CIRCLE_RADIUS * 2 + 8
      return circles.value.some(circle => {
        if (circle.id === excludeId || circle.slideIndex !== slideIndex) return false
        const distance = Math.sqrt(Math.pow(circle.x - x, 2) + Math.pow(circle.y - y, 2))
        return distance < minDistance
      })
    }
    
    const addCircle = (x, y, slideIndex = currentSlide.value) => {
      const overlay = document.querySelector(`.measurement-overlay[data-slide="${slideIndex}"]`)
      if (!overlay) return null
      
      const rect = overlay.getBoundingClientRect()
      const constrained = constrainCoordinates(x, y, rect)
      
      if (checkCollision(constrained.x, constrained.y, slideIndex)) {
        let found = false
        let attempts = 0
        const maxAttempts = 20
        
        while (!found && attempts < maxAttempts) {
          const offsetX = (Math.random() - 0.5) * 100
          const offsetY = (Math.random() - 0.5) * 100
          const newX = constrained.x + offsetX
          const newY = constrained.y + offsetY
          const newConstrained = constrainCoordinates(newX, newY, rect)
          
          if (!checkCollision(newConstrained.x, newConstrained.y, slideIndex)) {
            constrained.x = newConstrained.x
            constrained.y = newConstrained.y
            found = true
          }
          attempts++
        }
        
        if (!found) {
          alert('Cannot place point here - try a different location with more space.')
          return null
        }
      }
      
      const circle = {
        id: ++circleCounter.value,
        x: constrained.x,
        y: constrained.y,
        slideIndex: slideIndex
      }
      
      circles.value.push(circle)
      selectCircle(circle)
      return circle
    }
    
    const addCircleFromInputs = () => {
      if (!measurementMode.value) return
      
      const x = parseInt(xInput.value) || 100
      const y = parseInt(yInput.value) || 100
      addCircle(x, y, currentSlide.value)
    }
    
    const selectCircle = (circle) => {
      selectedCircle.value = circle
      xInput.value = Math.round(circle.x)
      yInput.value = Math.round(circle.y)
      currentSlide.value = circle.slideIndex
    }
    
    const deleteCircle = (circleId) => {
      const index = circles.value.findIndex(c => c.id === circleId)
      if (index !== -1) {
        circles.value.splice(index, 1)
        
        if (selectedCircle.value?.id === circleId) {
          selectedCircle.value = null
          xInput.value = ''
          yInput.value = ''
        }
      }
    }
    
    const selectMaterial = (materialId) => {
      selectedMaterial.value = materialId
    }
    
    const handleOverlayClick = (event) => {
      if (!measurementMode.value || isDragging.value) return
      
      const overlay = event.currentTarget
      const slideIndex = parseInt(overlay.dataset.slide)
      
      if (slideIndex !== currentSlide.value) return
      
      const rect = overlay.getBoundingClientRect()
      const coords = getEventCoords(event)
      const x = coords.x - rect.left
      const y = coords.y - rect.top
      
      addCircle(x, y, slideIndex)
    }
    
    const updateFromInputs = () => {
      if (selectedCircle.value && measurementMode.value) {
        const x = parseInt(xInput.value) || 0
        const y = parseInt(yInput.value) || 0
        
        const overlay = document.querySelector(`.measurement-overlay[data-slide="${selectedCircle.value.slideIndex}"]`)
        if (!overlay) return
        
        const rect = overlay.getBoundingClientRect()
        const constrained = constrainCoordinates(x, y, rect)
        
        if (!checkCollision(constrained.x, constrained.y, selectedCircle.value.slideIndex, selectedCircle.value.id)) {
          selectedCircle.value.x = constrained.x
          selectedCircle.value.y = constrained.y
          
          xInput.value = Math.round(constrained.x)
          yInput.value = Math.round(constrained.y)
        } else {
          xInput.value = Math.round(selectedCircle.value.x)
          yInput.value = Math.round(selectedCircle.value.y)
        }
      }
    }
    
    const handleCircleMouseDown = (event, circle) => {
      event.preventDefault()
      event.stopPropagation()
      
      isDragging.value = true
      draggedCircle.value = circle
      selectCircle(circle)
      
      const overlay = event.target.closest('.measurement-overlay')
      const rect = overlay.getBoundingClientRect()
      
      const startCoords = getEventCoords(event)
      const startX = startCoords.x
      const startY = startCoords.y
      const initialX = circle.x
      const initialY = circle.y
      
      const onMove = (e) => {
        if (!isDragging.value || draggedCircle.value !== circle) return
        
        const coords = getEventCoords(e)
        const deltaX = coords.x - startX
        const deltaY = coords.y - startY
        
        let newX = initialX + deltaX
        let newY = initialY + deltaY
        
        const constrained = constrainCoordinates(newX, newY, rect)
        
        if (!checkCollision(constrained.x, constrained.y, circle.slideIndex, circle.id)) {
          circle.x = constrained.x
          circle.y = constrained.y
          xInput.value = Math.round(constrained.x)
          yInput.value = Math.round(constrained.y)
        }
      }
      
      const onTouchMove = (e) => {
        e.preventDefault()
        onMove(e)
      }
      
      const onEnd = () => {
        if (draggedCircle.value === circle) {
          isDragging.value = false
          draggedCircle.value = null
        }
        
        document.removeEventListener('mousemove', onMove)
        document.removeEventListener('mouseup', onEnd)
        document.removeEventListener('touchmove', onTouchMove)
        document.removeEventListener('touchend', onEnd)
      }
      
      document.addEventListener('mousemove', onMove)
      document.addEventListener('mouseup', onEnd)
      document.addEventListener('touchmove', onTouchMove, { passive: false })
      document.addEventListener('touchend', onEnd)
    }
    
    const submitData = () => {
      const outputData = {
        product: {
          title: product.title,
          price: product.price
        },
        measurements: circles.value.map(circle => ({
          id: circle.id,
          slideIndex: circle.slideIndex,
          coordinates: {
            pixels: {
              x: Math.round(circle.x),
              y: Math.round(circle.y)
            },
            percentage: getPercentageCoords(circle)
          }
        })),
        selectedMaterial: selectedMaterial.value,
        currentSlide: currentSlide.value,
        totalSlides: totalSlides.value,
        timestamp: new Date().toISOString()
      }
      
      console.log('Product Configuration Data:', outputData)
      alert(`Data submitted! Check console for details.\n\n${JSON.stringify(outputData, null, 2)}`);
    }
    
    const addToCart = () => {
      if (selectedMaterial.value) {
        alert(`Added ${product.title} with ${materials.value.find(m => m.id === selectedMaterial.value)?.name} to cart!`)
      } else {
        alert('Please select a material before adding to cart.')
      }
    }
    
    let touchStartX = 0
    let touchStartY = 0
    let isSwipeGesture = false
    
    const handleTouchStart = (e) => {
      touchStartX = e.touches[0].clientX
      touchStartY = e.touches[0].clientY
      isSwipeGesture = true
    }
    
    const handleTouchMove = (e) => {
      if (!isSwipeGesture) return
      e.preventDefault()
    }
    
    const handleTouchEnd = (e) => {
      if (!isSwipeGesture) return
      
      const endX = e.changedTouches[0].clientX
      const endY = e.changedTouches[0].clientY
      const diffX = touchStartX - endX
      const diffY = Math.abs(touchStartY - endY)
      
      if (Math.abs(diffX) > 50 && diffY < 100) {
        if (diffX > 0) {
          nextSlide()
        } else {
          prevSlide()
        }
      }
      
      isSwipeGesture = false
    }
    
    onMounted(() => {
      const track = carouselTrack.value
      if (track) {
        track.addEventListener('touchstart', handleTouchStart)
        track.addEventListener('touchmove', handleTouchMove)
        track.addEventListener('touchend', handleTouchEnd)
      }
    })
    
    onUnmounted(() => {
      const track = carouselTrack.value
      if (track) {
        track.removeEventListener('touchstart', handleTouchStart)
        track.removeEventListener('touchmove', handleTouchMove)
        track.removeEventListener('touchend', handleTouchEnd)
      }
    })
    
    return {
      currentSlide,
      measurementMode,
      circles,
      selectedMaterial,
      selectedCircle,
      draggedCircle,
      xInput,
      yInput,
      
      mainCarousel,
      carouselTrack,
      
      product,
      productImages,
      materials,
      
      totalSlides,
      circlesBySlide,
      
      goToSlide,
      nextSlide,
      prevSlide,
      toggleMeasurementMode,
      getCirclesForSlide,
      getPercentageCoords,
      addCircleFromInputs,
      selectCircle,
      deleteCircle,
      selectMaterial,
      handleOverlayClick,
      updateFromInputs,
      handleCircleMouseDown,
      submitData,
      addToCart
    }
  }
}
</script>

<style scoped>
.apple-product-page {
  font-family: -apple-system, BlinkMacSystemFont, 'SF Pro Display', 'Segoe UI', Roboto, sans-serif;
  background: #ffffff;
  min-height: 100vh;
}

.hero-section {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  padding: 120px 20px 80px;
  text-align: center;
  position: relative;
  overflow: hidden;
}

.hero-section::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: radial-gradient(circle at 30% 20%, rgba(255,255,255,0.1) 0%, transparent 50%);
}

.hero-content {
  max-width: 800px;
  margin: 0 auto;
  position: relative;
  z-index: 1;
}

.hero-title {
  font-size: clamp(2.5rem, 5vw, 4rem);
  font-weight: 700;
  margin: 0 0 20px 0;
  line-height: 1.1;
  letter-spacing: -0.02em;
}

.hero-subtitle {
  font-size: clamp(1.125rem, 2vw, 1.375rem);
  font-weight: 400;
  margin: 0 0 30px 0;
  opacity: 0.9;
  line-height: 1.4;
}

.hero-price {
  font-size: clamp(1.5rem, 3vw, 2rem);
  font-weight: 600;
  opacity: 0.95;
}

.product-showcase {
  padding: 40px 20px;
}

.showcase-container {
  max-width: 1400px;
  margin: 0 auto;
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 20px;
  align-items: start;
}

.gallery-wrapper {
  position: sticky;
  top: 16px;
}

.image-gallery {
  background: white;
  border-radius: 24px;
  padding: 20px;
  box-shadow: 0 8px 60px rgba(0, 0, 0, 0.08);
  position: relative;
}

.gallery-viewport {
  position: relative;
  overflow: hidden;
  border-radius: 16px;
  aspect-ratio: 1.25;
  background: #f8f9fa;
}

.gallery-track {
  display: flex;
  transition: transform 0.6s cubic-bezier(0.16, 1, 0.3, 1);
  height: 100%;
}

.gallery-slide {
  width: 100%;
  height: 100%;
  flex-shrink: 0;
  position: relative;
}

.image-container {
  position: relative;
  width: 100%;
  height: 100%;
}

.gallery-slide img {
  width: 100%;
  height: 100%;
  object-fit: cover;
  border-radius: 12px;
}

.measurement-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
  z-index: 10;
  border-radius: 12px;
}

.measurement-overlay.active {
  pointer-events: all;
  cursor: crosshair;
  touch-action: none; 
}

.measurement-point {
  position: absolute;
  width: 50px;
  height: 50px;
  cursor: move;
  transform: translate(-50%, -50%);
  transition: all 0.3s cubic-bezier(0.16, 1, 0.3, 1);
  z-index: 15;
  touch-action: none; 
}

.point-inner {
  width: 100%;
  height: 100%;
  background: transparent; 
  border: 3px solid rgba(0, 122, 255, 0.9); 
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: 0 4px 20px rgba(0, 122, 255, 0.3);
}

.point-label {
  color: rgba(0, 122, 255, 0.9); 
  font-size: 12px;
  font-weight: 700;
}

.measurement-point:hover .point-inner {
  background: transparent; 
  border-color: rgba(0, 122, 255, 1);
  box-shadow: 0 8px 30px rgba(0, 122, 255, 0.5);
}

.measurement-point.selected .point-inner {
  background: transparent;
  border-color: rgba(255, 59, 48, 0.9); 
  box-shadow: 0 4px 20px rgba(255, 59, 48, 0.4);
}

.measurement-point.selected .point-label {
  color: rgba(255, 59, 48, 0.9);
}

.measurement-point.dragging .point-inner {
  background: transparent; 
  border-color: rgba(255, 59, 48, 1);
  box-shadow: 0 12px 40px rgba(255, 59, 48, 0.6);
}

.measurement-point.dragging .point-label {
  color: rgba(255, 59, 48, 1);
}

.gallery-nav {
  position: absolute;
  top: 50%;
  transform: translateY(-50%);
  background: rgba(255, 255, 255, 0.95);
  border: none;
  width: 48px;
  height: 48px;
  border-radius: 50%;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.3s ease;
  backdrop-filter: blur(20px);
  z-index: 20;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.1);
}

.gallery-nav:hover:not(:disabled) {
  background: white;
  transform: translateY(-50%) scale(1.05);
  box-shadow: 0 8px 30px rgba(0, 0, 0, 0.15);
}

.gallery-nav:disabled {
  opacity: 0.3;
  cursor: not-allowed;
}

.gallery-nav.prev {
  left: 20px;
}

.gallery-nav.next {
  right: 20px;
}

.gallery-nav svg {
  width: 24px;
  height: 24px;
  fill: #1d1d1f;
}

.gallery-thumbnails {
  display: flex;
  gap: 12px;
  margin-top: 20px;
  justify-content: center;
}

.thumbnail {
  width: 60px;
  height: 60px;
  border-radius: 12px;
  overflow: hidden;
  cursor: pointer;
  transition: all 0.3s ease;
  border: 2px solid transparent;
}

.thumbnail.active {
  border-color: #007aff;
  transform: scale(1.05);
}

.thumbnail img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.product-info {
  padding-bottom: 32px;
  border-bottom: 1px solid #e5e5ea;
}

.product-title {
  font-size: clamp(2rem, 4vw, 2.5rem);
  font-weight: 700;
  color: #1d1d1f;
  margin: 0 0 16px 0;
  line-height: 1.1;
  letter-spacing: -0.02em;
}

.product-description {
  font-size: 1.125rem;
  color: #86868b;
  margin: 0 0 20px 0;
  line-height: 1.5;
}

.product-price {
  font-size: 1.5rem;
  font-weight: 600;
  color: #1d1d1f;
}

.product-config {
  display: flex;
  flex-direction: column;
  gap: 40px;
}

.config-section {
  background: white;
  border-radius: 20px;
  padding: 32px;
  box-shadow: 0 4px 30px rgba(0, 0, 0, 0.06);
}

.section-header {
  margin-bottom: 32px;
}

.section-title {
  font-size: 2rem;
  font-weight: 700;
  color: #1d1d1f;
  margin: 0 0 12px 0;
  letter-spacing: -0.01em;
}

.section-description {
  font-size: 1.125rem;
  color: #86868b;
  margin: 0;
  line-height: 1.5;
}

.control-group {
  margin-bottom: 32px;
}

.toggle-control {
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.toggle-label {
  display: flex;
  align-items: center;
  justify-content: space-between;
  width: 100%;
  cursor: pointer;
}

.label-text {
  font-size: 1.125rem;
  font-weight: 600;
  color: #1d1d1f;
}

.toggle-switch {
  position: relative;
  width: 70px;
  height: 40px;
  background: #e5e5ea;
  border-radius: 20px;
  cursor: pointer;
  transition: all 0.3s cubic-bezier(0.16, 1, 0.3, 1);
}

.toggle-switch.active {
  background: #007aff;
}

.toggle-slider {
  position: absolute;
  top: 3px;
  left: 3px;
  width: 34px;
  height: 34px;
  background: white;
  border-radius: 50%;
  transition: all 0.3s cubic-bezier(0.16, 1, 0.3, 1);
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.15);
  display: flex;
  align-items: center;
  justify-content: center;
}

.toggle-switch.active .toggle-slider {
  transform: translateX(30px);
}

.toggle-icon {
  width: 16px;
  height: 16px;
}

.toggle-icon svg {
  width: 100%;
  height: 100%;
  fill: #007aff;
}

.toggle-switch.active .toggle-icon svg {
  fill: #007aff;
}

.coordinate-panel {
  background: #f8f9fa;
  border-radius: 16px;
  padding: 24px;
  transition: opacity 0.3s ease;
}

.coordinate-panel.disabled {
  opacity: 0.5;
  pointer-events: none;
}

.panel-title {
  font-size: 1.25rem;
  font-weight: 600;
  color: #1d1d1f;
  margin: 0 0 20px 0;
}

.coordinate-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 16px;
  margin-bottom: 24px;
}

.input-field {
  position: relative;
}

.field-label {
  display: block;
  font-size: 0.875rem;
  font-weight: 600;
  color: #86868b;
  margin-bottom: 8px;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.field-input {
  width: 80%;
  padding: 12px 40px 12px 16px;
  border: 2px solid #e5e5ea;
  border-radius: 12px;
  font-size: 1rem;
  font-weight: 500;
  background: white;
  transition: all 0.3s ease;
}

.field-input:focus {
  outline: none;
  border-color: #007aff;
  box-shadow: 0 0 0 3px rgba(0, 122, 255, 0.1);
}

.field-input:disabled {
  background: #f5f5f7;
  color: #86868b;
}

.field-unit {
  position: absolute;
  right: 16px;
  top: 50%;
  transform: translateY(-50%);
  font-size: 0.875rem;
  color: #86868b;
  pointer-events: none;
  margin-top: 16px;
}

.action-button {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  padding: 12px 24px;
  border: none;
  border-radius: 12px;
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s cubic-bezier(0.16, 1, 0.3, 1);
  text-decoration: none;
}

.action-button svg {
  width: 20px;
  height: 20px;
}

.action-button.primary {
  background: #007aff;
  color: white;
}

.action-button.primary:hover:not(:disabled) {
  background: #0051d5;
  transform: translateY(-2px);
  box-shadow: 0 8px 30px rgba(0, 122, 255, 0.3);
}

.action-button.secondary {
  background: #f5f5f7;
  color: #1d1d1f;
}

.action-button.secondary:hover:not(:disabled) {
  background: #e5e5ea;
  transform: translateY(-2px);
}

.action-button.large {
  padding: 16px 32px;
  font-size: 1.125rem;
}

.action-button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
  transform: none !important;
}

.points-panel {
  background: #f8f9fa;
  border-radius: 16px;
  padding: 24px;
}

.points-list {
  display: flex;
  flex-direction: column;
  gap: 24px;
}

.slide-group {
  background: white;
  border-radius: 12px;
  padding: 20px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.04);
}

.slide-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 16px;
  padding-bottom: 12px;
  border-bottom: 1px solid #f0f0f0;
}

.slide-label {
  font-weight: 600;
  color: #1d1d1f;
}

.point-count {
  font-size: 0.875rem;
  color: #86868b;
}

.points-grid {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.point-item {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 12px 16px;
  background: #f8f9fa;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.3s ease;
  border: 2px solid transparent;
}

.point-item:hover {
  background: #f0f0f0;
}

.point-item.selected {
  background: rgba(0, 122, 255, 0.1);
  border-color: #007aff;
}

.point-info {
  flex: 1;
}

.point-id {
  font-weight: 600;
  color: #1d1d1f;
  font-size: 0.875rem;
}

.point-coords {
  color: #007aff;
  font-size: 0.875rem;
  font-weight: 500;
  margin: 2px 0;
}

.point-percentage {
  color: #86868b;
  font-size: 0.75rem;
}

.delete-point {
  background: #ff3b30;
  color: white;
  border: none;
  width: 28px;
  height: 28px;
  border-radius: 50%;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.3s ease;
}

.delete-point:hover {
  background: #d70015;
  transform: scale(1.1);
}

.delete-point svg {
  width: 14px;
  height: 14px;
}

.materials-grid {
  display: grid;
  grid-template-columns: 1fr;
  gap: 16px;
  margin-bottom: 32px;
}

.material-card {
  display: flex;
  align-items: center;
  padding: 20px;
  border: 2px solid #e5e5ea;
  border-radius: 16px;
  cursor: pointer;
  transition: all 0.3s cubic-bezier(0.16, 1, 0.3, 1);
  background: white;
  position: relative;
  overflow: hidden;
}

.material-card::before {
  content: '';
  position: absolute;
  top: 0;
  left: -100%;
  width: 100%;
  height: 100%;
  background: linear-gradient(90deg, transparent, rgba(0, 122, 255, 0.05), transparent);
  transition: left 0.6s ease;
}

.material-card:hover {
  border-color: #007aff;
  transform: translateY(-2px);
  box-shadow: 0 8px 30px rgba(0, 122, 255, 0.15);
}

.material-card:hover::before {
  left: 100%;
}

.material-card.selected {
  border-color: #007aff;
  background: linear-gradient(135deg, rgba(0, 122, 255, 0.05), rgba(0, 122, 255, 0.02));
  transform: translateY(-2px);
  box-shadow: 0 8px 30px rgba(0, 122, 255, 0.2);
}

.material-content {
  flex: 1;
}

.material-name {
  font-size: 1.125rem;
  font-weight: 600;
  color: #1d1d1f;
  margin: 0 0 8px 0;
}

.material-description {
  color: #86868b;
  margin: 0;
  font-size: 0.875rem;
  line-height: 1.4;
}

.material-indicator {
  margin-left: 16px;
}

.selection-check {
  width: 28px;
  height: 28px;
  border: 2px solid #e5e5ea;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.3s ease;
}

.material-card.selected .selection-check {
  background: #007aff;
  border-color: #007aff;
}

.selection-check svg {
  width: 16px;
  height: 16px;
  fill: white;
  opacity: 0;
  transition: opacity 0.3s ease;
}

.material-card.selected .selection-check svg {
  opacity: 1;
}

.action-section {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

@media (max-width: 1200px) {
  .showcase-container {
    grid-template-columns: 1fr;
    gap: 60px;
  }
  
  .gallery-wrapper {
    position: relative;
    top: 0;
  }
}

@media (max-width: 768px) {
  .hero-section {
    padding: 80px 20px 60px;
  }
  
  .product-showcase {
    padding: 60px 20px;
  }
  
  .showcase-container {
    gap: 40px;
  }
  
  .config-section {
    padding: 24px;
  }
  
  .coordinate-grid {
    grid-template-columns: 1fr;
  }
  
  .action-section {
    flex-direction: column;
  }
  
  .gallery-nav {
    width: 40px;
    height: 40px;
  }
  
  .gallery-nav svg {
    width: 20px;
    height: 20px;
  }
  
  .materials-grid {
    grid-template-columns: 1fr;
  }
  
  .measurement-point {
    width: 50px; 
    height: 50px;
  }
  
  .measurement-point::before {
    content: '';
    position: absolute;
    top: -15px;
    left: -15px;
    right: -15px;
    bottom: -15px;
    z-index: -1;
  }
  
  .point-inner {
    width: 100%;
    height: 100%;
  }
}

@media (max-width: 480px) {
  .gallery-nav {
    display: none;
  }
  
  .coordinate-grid {
    grid-template-columns: 1fr;
  }
  
  .thumbnail {
    width: 50px;
    height: 50px;
  }
  
  .measurement-point {
    width: 50px;
    height: 50px;
  }
  
  .measurement-point::before {
    top: -20px;
    left: -20px;
    right: -20px;
    bottom: -20px;
  }
}

@media (prefers-color-scheme: dark) {
  .apple-product-page {
    background: #ffffff;
    color: #1d1d1f;
  }
  
  .config-section,
  .image-gallery,
  .material-card,
  .slide-group {
    background: #ffffff;
  }
  
  .coordinate-panel,
  .points-panel {
    background: #f8f9fa;
  }
  
  .field-input {
    background: #ffffff;
    border-color: #e5e5ea;
    color: #1d1d1f;
  }
  
  .field-input:disabled {
    background: #f5f5f7;
  }
  
  .section-title,
  .material-name,
  .point-id,
  .label-text,
  .product-title {
    color: #1d1d1f;
  }
  
  .action-button.secondary {
    background: #f5f5f7;
    color: #1d1d1f;
  }
  
  .action-button.secondary:hover:not(:disabled) {
    background: #e5e5ea;
  }
}
</style>