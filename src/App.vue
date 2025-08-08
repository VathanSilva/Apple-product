<template>
  <div class="app-container">
    <div class="sidebar">
      <div class="canvas-container">
        <div class="canvas-wrapper" ref="canvasWrapper">
          <canvas 
            ref="previewCanvas" 
            class="preview-canvas"
            :width="canvasWidth"
            :height="canvasHeight"
            @wheel="handleWheel"
            @mousedown="handleMouseDown"
            @mousemove="handleMouseMove"
            @mouseup="handleMouseUp"
            @mouseleave="handleMouseUp"
            @touchstart="handleTouchStart"
            @touchmove="handleTouchMove"
            @touchend="handleTouchEnd"
            :style="{ 
              transform: `translate(${panX}px, ${panY}px) scale(${zoomLevel})`,
              cursor: isDragging ? 'grabbing' : 'grab'
            }"
          ></canvas>
        </div>
        
        <div class="zoom-controls">
          <button @click="zoomOut" class="zoom-btn" :disabled="zoomLevel <= 0.2">−</button>
          <span class="zoom-level">{{ Math.round(zoomLevel * 100) }}%</span>
          <button @click="zoomIn" class="zoom-btn" :disabled="zoomLevel >= 3">+</button>
          <button @click="resetZoom" class="reset-btn">Reset</button>
        </div>
        
        <div class="export-controls">
          <button @click="downloadPreview" class="export-btn">
            📥 Download Preview
          </button>
        </div>
      </div>
    </div>

    <div class="main-content">
      <div class="header">
        <h1 class="title">Maße. Eingeben.</h1>
        
        <div class="controls-row">
          <button 
            @click="toggleUnits" 
            class="unit-toggle-btn"
            :class="{ active: isMetric }"
          >
            {{ isMetric ? 'cm' : 'in' }}
          </button>
          
          <div class="upload-section">
            <input 
              type="file" 
              ref="imageUpload" 
              @change="handleImageUpload" 
              accept="image/*"
              style="display: none"
            />
            <button @click="$refs.imageUpload.click()" class="upload-image-btn">
              🖼️ Upload Image
            </button>
          </div>
        </div>
      </div>

      <div v-if="alertMessage" class="alert">
        {{ alertMessage }}
      </div>

      <div class="measurement-section">
        <div 
          v-for="(plate, index) in plates" 
          :key="plate.id"
          class="measurement-box"
          :class="{ 'dragging': draggedIndex === index, 'animated': animateChanges }"
          draggable="true"
          @dragstart="handleDragStart(index, $event)"
          @dragover="handleDragOver($event)"
          @drop="handleDrop(index, $event)"
          @dragend="handleDragEnd"
        >
          <div class="plate-row" :class="{ 'first-plate': index === 0 }">
            <div class="plate-number" :class="{ active: index === activePlateIndex }">
              {{ index + 1 }}
            </div>
            <div class="plate-inputs">
              <input 
                type="text" 
                v-model="plate.widthInput"
                @input="validateAndUpdatePlate(index, 'width', $event.target.value)"
                @blur="formatInput(index, 'width')"
                class="dimension-input" 
                :class="{ error: plate.widthError }"
                inputmode="decimal"
              />
              <span class="unit-text">{{ currentUnit }}</span>
              <span class="multiply-symbol">×</span>
              <input 
                type="text" 
                v-model="plate.heightInput"
                @input="validateAndUpdatePlate(index, 'height', $event.target.value)"
                @blur="formatInput(index, 'height')"
                class="dimension-input"
                :class="{ error: plate.heightError }"
                inputmode="decimal"
              />
              <span class="unit-text">{{ currentUnit }}</span>
              <button 
                v-if="plates.length > 1"
                @click="deletePlate(index)"
                class="remove-btn"
                :aria-label="'Delete plate ' + (index + 1)"
              >−</button>
            </div>
          </div>
          
          <div v-if="index !== 0" class="dimension-labels">
            <div class="label-group">
              <span class="label-title">Breite</span>
              <span class="label-range">{{ widthRange }}</span>
            </div>
            <div class="label-group">
              <span class="label-title">Höhe</span>
              <span class="label-range">{{ heightRange }}</span>
            </div>
          </div>

          <div v-if="index !== 0 && isMetric" style="gap: 9.6rem;" class="conversion-info">
            <span class="conversion-value">{{ (plate.width * 10).toFixed(0) }} mm</span>
            <span class="conversion-value">{{ (plate.height * 10).toFixed(0) }} mm</span>
          </div>
          <div v-if="index !== 0 && !isMetric" class="conversion-info">
            <span class="conversion-value">{{ convertToMetric(plate.width).toFixed(1) }} cm</span>
            <span class="conversion-value">{{ convertToMetric(plate.height).toFixed(1) }} cm</span>
          </div>
        </div>

        <button 
          v-if="plates.length < 10"
          @click="addPlate"
          class="add-plate-btn"
          :class="{ 'animated': animateChanges }"
        >
          Rückwand hinzufügen +
        </button>

        <div class="debug-info">
          <p>Total Width: {{ totalWidth.toFixed(1) }}{{ currentUnit }}</p>
          <p>Image Extension: {{ totalWidth > (isMetric ? 300 : 118) ? 'Mirrored' : 'Original' }}</p>
          <p>Image Status: {{ baseImage ? '✅ Loaded' : '❌ Loading...' }}</p>
          <p v-if="baseImage">Image Size: {{ baseImage.width }}x{{ baseImage.height }}px</p>
          <p>Active Plate: {{ activePlateIndex + 1 }}</p>
          <p>Units: {{ isMetric ? 'Metric (cm)' : 'Imperial (inches)' }}</p>
          <p>Zoom: {{ Math.round(zoomLevel * 100) }}%</p>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'GardenDesign',
  data() {
    return {
      plates: [],
      activePlateIndex: 0,
      alertMessage: '',
      canvasWidth: 400,
      canvasHeight: 300,
      baseImage: null,
      locale: 'de',
      isMetric: true,
      draggedIndex: null,
      animateChanges: false,
      persistentData: {
        plates: [],
        isMetric: true
      },
      zoomLevel: 1,
      panX: 0,
      panY: 0,
      isDragging: false,
      lastMouseX: 0,
      lastMouseY: 0,
      touches: []
    }
  },
  computed: {
    totalWidth() {
      return this.plates.reduce((sum, plate) => sum + plate.width, 0);
    },
    currentUnit() {
      return this.isMetric ? 'cm' : 'in';
    },
    widthRange() {
      if (this.isMetric) {
        return '20 - 300 cm';
      } else {
        return '8 - 118 in';
      }
    },
    heightRange() {
      if (this.isMetric) {
        return '30 - 128 cm';
      } else {
        return '12 - 50 in';
      }
    }
  },
  mounted() {
    this.detectLocale();
    this.loadFromMemory();
    this.initializeCanvas();
    this.loadImage();
    
    this.resizeHandler = this.throttle(this.handleResize, 100);
    window.addEventListener('resize', this.resizeHandler);
  },
  beforeUnmount() {
    window.removeEventListener('resize', this.resizeHandler);
  },
  methods: {
    throttle(func, limit) {
      let inThrottle;
      return function() {
        const args = arguments;
        const context = this;
        if (!inThrottle) {
          func.apply(context, args);
          inThrottle = true;
          setTimeout(() => inThrottle = false, limit);
        }
      }
    },

    detectLocale() {
      const userLang = navigator.language || navigator.userLanguage || 'en';
      this.locale = userLang.startsWith('de') ? 'de' : 'en';
      this.isMetric = userLang.startsWith('de') || userLang.startsWith('fr') || userLang.startsWith('it') || userLang.startsWith('es');
    },

    handleWheel(event) {
      event.preventDefault();
      
      const rect = this.$refs.canvasWrapper.getBoundingClientRect();
      const mouseX = event.clientX - rect.left;
      const mouseY = event.clientY - rect.top;
      
      const zoomFactor = event.deltaY > 0 ? 0.9 : 1.1;
      const newZoom = Math.max(0.2, Math.min(3, this.zoomLevel * zoomFactor));
      
      if (newZoom !== this.zoomLevel) {
        const zoomRatio = newZoom / this.zoomLevel;
        
        this.panX = mouseX - (mouseX - this.panX) * zoomRatio;
        this.panY = mouseY - (mouseY - this.panY) * zoomRatio;
        
        this.zoomLevel = newZoom;
      }
    },

    handleMouseDown(event) {
      this.isDragging = true;
      this.lastMouseX = event.clientX;
      this.lastMouseY = event.clientY;
      event.preventDefault();
    },

    handleMouseMove(event) {
      if (!this.isDragging) return;
      
      const deltaX = event.clientX - this.lastMouseX;
      const deltaY = event.clientY - this.lastMouseY;
      
      this.panX += deltaX;
      this.panY += deltaY;
      
      this.lastMouseX = event.clientX;
      this.lastMouseY = event.clientY;
    },

    handleMouseUp() {
      this.isDragging = false;
    },

    handleTouchStart(event) {
      event.preventDefault();
      this.touches = Array.from(event.touches);
      
      if (this.touches.length === 1) {
        this.isDragging = true;
        this.lastMouseX = this.touches[0].clientX;
        this.lastMouseY = this.touches[0].clientY;
      } else if (this.touches.length === 2) {
        this.isDragging = false;
      }
    },

    handleTouchMove(event) {
      event.preventDefault();
      const currentTouches = Array.from(event.touches);
      
      if (currentTouches.length === 1 && this.isDragging) {
        const deltaX = currentTouches[0].clientX - this.lastMouseX;
        const deltaY = currentTouches[0].clientY - this.lastMouseY;
        
        this.panX += deltaX;
        this.panY += deltaY;
        
        this.lastMouseX = currentTouches[0].clientX;
        this.lastMouseY = currentTouches[0].clientY;
      } else if (currentTouches.length === 2 && this.touches.length === 2) {
        const prevDistance = Math.hypot(
          this.touches[0].clientX - this.touches[1].clientX,
          this.touches[0].clientY - this.touches[1].clientY
        );
        
        const currentDistance = Math.hypot(
          currentTouches[0].clientX - currentTouches[1].clientX,
          currentTouches[0].clientY - currentTouches[1].clientY
        );
        
        if (prevDistance > 0) {
          const rect = this.$refs.canvasWrapper.getBoundingClientRect();
          const centerX = (currentTouches[0].clientX + currentTouches[1].clientX) / 2 - rect.left;
          const centerY = (currentTouches[0].clientY + currentTouches[1].clientY) / 2 - rect.top;
          
          const zoomFactor = currentDistance / prevDistance;
          const newZoom = Math.max(0.2, Math.min(3, this.zoomLevel * zoomFactor));
          
          if (newZoom !== this.zoomLevel) {
            const zoomRatio = newZoom / this.zoomLevel;
            
            this.panX = centerX - (centerX - this.panX) * zoomRatio;
            this.panY = centerY - (centerY - this.panY) * zoomRatio;
            
            this.zoomLevel = newZoom;
          }
        }
      }
      
      this.touches = currentTouches;
    },

    handleTouchEnd(event) {
      event.preventDefault();
      this.touches = Array.from(event.touches);
      
      if (this.touches.length === 0) {
        this.isDragging = false;
      }
    },

    zoomIn() {
      const newZoom = Math.min(3, this.zoomLevel * 1.2);
      this.zoomLevel = newZoom;
    },

    zoomOut() {
      const newZoom = Math.max(0.2, this.zoomLevel / 1.2);
      this.zoomLevel = newZoom;
    },

    resetZoom() {
      this.zoomLevel = 1;
      this.panX = 0;
      this.panY = 0;
    },

    toggleUnits() {
      this.animateChanges = true;
      this.isMetric = !this.isMetric;
      
      this.plates.forEach(plate => {
        if (this.isMetric) {
          plate.width = this.convertToMetric(plate.width);
          plate.height = this.convertToMetric(plate.height);
        } else {
          plate.width = this.convertToImperial(plate.width);
          plate.height = this.convertToImperial(plate.height);
        }
        
        plate.widthInput = this.formatNumber(plate.width);
        plate.heightInput = this.formatNumber(plate.height);
      });
      
      this.saveToMemory();
      this.renderCanvas();
      
      setTimeout(() => {
        this.animateChanges = false;
      }, 300);
    },

    convertToMetric(inches) {
      return inches * 2.54;
    },

    convertToImperial(cm) {
      return cm / 2.54;
    },

    handleImageUpload(event) {
      const file = event.target.files[0];
      if (!file) return;

      if (!file.type.startsWith('image/')) {
        this.showAlert('Please select a valid image file');
        return;
      }

      const reader = new FileReader();
      reader.onload = (e) => {
        const img = new Image();
        img.onload = () => {
          this.baseImage = img;
          this.renderCanvas();
          this.showAlert('Image uploaded successfully!');
        };
        img.onerror = () => {
          this.showAlert('Failed to load the uploaded image');
        };
        img.src = e.target.result;
      };
      reader.readAsDataURL(file);
    },

    downloadPreview() {
      const canvas = this.$refs.previewCanvas;
      if (!canvas) return;

      const exportCanvas = document.createElement('canvas');
      const exportCtx = exportCanvas.getContext('2d');
      
      exportCanvas.width = 1200;
      exportCanvas.height = 800;
      
      const originalWidth = this.canvasWidth;
      const originalHeight = this.canvasHeight;
      
      this.canvasWidth = exportCanvas.width;
      this.canvasHeight = exportCanvas.height;
      
      this.renderCanvasToContext(exportCtx, exportCanvas);
      
      this.canvasWidth = originalWidth;
      this.canvasHeight = originalHeight;
      this.renderCanvas();

      exportCanvas.toBlob((blob) => {
        const url = URL.createObjectURL(blob);
        const a = document.createElement('a');
        a.href = url;
        a.download = `garden-design-preview-${Date.now()}.png`;
        document.body.appendChild(a);
        a.click();
        document.body.removeChild(a);
        URL.revokeObjectURL(url);
      });
    },

    handleDragStart(index, event) {
      this.draggedIndex = index;
      event.dataTransfer.effectAllowed = 'move';
      event.dataTransfer.setData('text/html', index.toString());
    },

    handleDragOver(event) {
      event.preventDefault();
      event.dataTransfer.dropEffect = 'move';
    },

    handleDrop(targetIndex, event) {
      event.preventDefault();
      
      if (this.draggedIndex === null || this.draggedIndex === targetIndex) {
        return;
      }

      this.animateChanges = true;
      
      const draggedPlate = this.plates[this.draggedIndex];
      this.plates.splice(this.draggedIndex, 1);
      this.plates.splice(targetIndex, 0, draggedPlate);
      
      if (this.activePlateIndex === this.draggedIndex) {
        this.activePlateIndex = targetIndex;
      } else if (this.draggedIndex < this.activePlateIndex && targetIndex >= this.activePlateIndex) {
        this.activePlateIndex--;
      } else if (this.draggedIndex > this.activePlateIndex && targetIndex <= this.activePlateIndex) {
        this.activePlateIndex++;
      }
      
      this.saveToMemory();
      this.renderCanvas();
      
      setTimeout(() => {
        this.animateChanges = false;
      }, 300);
    },

    handleDragEnd() {
      this.draggedIndex = null;
    },

    loadFromMemory() {
      let savedData = null;
      
      try {
        const saved = localStorage.getItem('gardenDesignData');
        savedData = saved ? JSON.parse(saved) : null;
      } catch (error) {
        console.warn('Failed to load from localStorage:', error);
        
        savedData = this.persistentData.plates.length > 0 ? this.persistentData : null;
      }
      
      if (savedData && savedData.plates.length > 0) {
        this.isMetric = savedData.isMetric !== undefined ? savedData.isMetric : true;
        this.plates = savedData.plates.map(plate => ({
          ...plate,
          widthInput: this.formatNumber(plate.width),
          heightInput: this.formatNumber(plate.height),
          widthError: false,
          heightError: false
        }));
      } else {
        this.initializeDefaultPlate();
      }
    },

    clearSavedData() {
      try {
        localStorage.removeItem('gardenDesignData');
      } catch (error) {
        console.warn('Failed to clear localStorage:', error);
      }
      this.persistentData = { plates: [], isMetric: true };
    },

    initializeDefaultPlate() {
      const defaultWidth = this.isMetric ? 250 : 98;
      const defaultHeight = this.isMetric ? 128 : 50;
      
      this.plates = [{
        id: Date.now(),
        width: defaultWidth,
        height: defaultHeight,
        widthInput: this.formatNumber(defaultWidth),
        heightInput: this.formatNumber(defaultHeight),
        widthError: false,
        heightError: false
      }];
      this.saveToMemory();
    },

    saveToMemory() {
      const dataToSave = {
        plates: this.plates.map(plate => ({
          id: plate.id,
          width: plate.width,
          height: plate.height
        })),
        isMetric: this.isMetric,
        lastUpdated: Date.now()
      };
      
      try {
        localStorage.setItem('gardenDesignData', JSON.stringify(dataToSave));
      } catch (error) {
        console.warn('Failed to save to localStorage:', error);
        
        this.persistentData = dataToSave;
      }
    },

    formatNumber(num) {
      const rounded = Math.round(num * 10) / 10;
      if (this.locale === 'de') {
        return rounded.toString().replace('.', ',');
      }
      return rounded.toString();
    },

    parseNumber(str) {
      if (!str) return null;
      
      let cleanStr = str.replace(',', '.');
      const num = parseFloat(cleanStr);
      return isNaN(num) ? null : num;
    },

    validateAndUpdatePlate(index, dimension, value) {
      const plate = this.plates[index];
      const num = this.parseNumber(value);
      
      this.alertMessage = '';
      
      if (num === null && value !== '') {
        plate[`${dimension}Error`] = true;
        this.showAlert(`Invalid input for ${dimension === 'width' ? 'width' : 'height'}`);
        return;
      }

      if (num !== null) {
        let minWidth, maxWidth, minHeight, maxHeight;
        
        if (this.isMetric) {
          minWidth = 20; maxWidth = 300;
          minHeight = 30; maxHeight = 128;
        } else {
          minWidth = 8; maxWidth = 118;
          minHeight = 12; maxHeight = 50;
        }

        if (dimension === 'width' && (num < minWidth || num > maxWidth)) {
          plate[`${dimension}Error`] = true;
          this.showAlert(`Width must be between ${minWidth} and ${maxWidth} ${this.currentUnit}`);
          return;
        }
        
        if (dimension === 'height' && (num < minHeight || num > maxHeight)) {
          plate[`${dimension}Error`] = true;
          this.showAlert(`Height must be between ${minHeight} and ${maxHeight} ${this.currentUnit}`);
          return;
        }

        plate[`${dimension}Error`] = false;
        plate[dimension] = num;
        this.saveToMemory();
        this.renderCanvas();
      }
    },

    formatInput(index, dimension) {
      const plate = this.plates[index];
      if (!plate[`${dimension}Error`] && plate[dimension]) {
        plate[`${dimension}Input`] = this.formatNumber(plate[dimension]);
      }
    },

    showAlert(message) {
      this.alertMessage = message;
      setTimeout(() => {
        this.alertMessage = '';
      }, 3000);
    },

    addPlate() {
      if (this.plates.length >= 10) return;
      
      this.animateChanges = true;
      
      const defaultWidth = this.isMetric ? 30 : 12;
      const defaultHeight = this.isMetric ? 30 : 12;
      
      this.plates.push({
        id: Date.now(),
        width: defaultWidth,
        height: defaultHeight,
        widthInput: this.formatNumber(defaultWidth),
        heightInput: this.formatNumber(defaultHeight),
        widthError: false,
        heightError: false
      });
      
      this.activePlateIndex = this.plates.length - 1;
      this.saveToMemory();
      this.renderCanvas();
      
      setTimeout(() => {
        this.animateChanges = false;
      }, 300);
    },

    deletePlate(index) {
      if (this.plates.length <= 1) return;
      
      this.animateChanges = true;
      
      this.plates.splice(index, 1);
      if (this.activePlateIndex >= this.plates.length) {
        this.activePlateIndex = this.plates.length - 1;
      }
      this.saveToMemory();
      this.renderCanvas();
      
      setTimeout(() => {
        this.animateChanges = false;
      }, 300);
    },

    initializeCanvas() {
      this.handleResize();
    },

    handleResize() {
      const container = this.$refs.canvasWrapper;
      if (container) {
        const containerWidth = container.clientWidth - 10;
        const containerHeight = container.clientHeight - 140;
        
        this.canvasWidth = containerWidth;
        this.canvasHeight = containerHeight;
        
        this.$nextTick(() => {
          this.renderCanvas();
        });
      }
    },

    loadImage() {
      const img = new Image();
      img.crossOrigin = 'anonymous';
      img.onload = () => {
        this.baseImage = img;
        this.renderCanvas();
      };
      img.onerror = (error) => {
        console.error('Failed to load image:', error);
        this.createFallbackImage();
      };
      img.src = 'https://rueckwand24.com/cdn/shop/files/Kuechenrueckwand-Kuechenrueckwand-Gruene-frische-Kraeuter-KR-000018-HB.jpg?v=1695288356&width=1200';
    },

    createFallbackImage() {
      const canvas = document.createElement('canvas');
      canvas.width = 800;
      canvas.height = 400;
      const ctx = canvas.getContext('2d');
      
      const gradient = ctx.createLinearGradient(0, 0, 0, canvas.height);
      gradient.addColorStop(0, '#87CEEB');
      gradient.addColorStop(1, '#98FB98');
      ctx.fillStyle = gradient;
      ctx.fillRect(0, 0, canvas.width, canvas.height);
      
      ctx.fillStyle = '#228B22';
      for (let i = 0; i < 50; i++) {
        const x = Math.random() * canvas.width;
        const y = canvas.height * 0.6 + Math.random() * canvas.height * 0.4;
        const size = 5 + Math.random() * 15;
        ctx.beginPath();
        ctx.arc(x, y, size, 0, Math.PI * 2);
        ctx.fill();
      }
      
      this.baseImage = canvas;
      this.renderCanvas();
    },

    renderCanvas() {
      const canvas = this.$refs.previewCanvas;
      if (!canvas) return;
      this.renderCanvasToContext(canvas.getContext('2d'), canvas);
    },

    renderCanvasToContext(ctx, canvas) {
      if (!this.baseImage) return;
      
      ctx.clearRect(0, 0, canvas.width, canvas.height);
      
      const totalWidthCm = this.totalWidth;
      const maxHeightCm = Math.max(...this.plates.map(p => p.height));
      
      const totalWidthMetric = this.isMetric ? totalWidthCm : this.convertToMetric(totalWidthCm);
      const maxHeightMetric = this.isMetric ? maxHeightCm : this.convertToMetric(maxHeightCm);
      
      const spacing = this.plates.length > 1 ? (this.plates.length - 1) * 1 : 0;
      const scaleX = (canvas.width - spacing) / totalWidthMetric;
      const scaleY = canvas.height / maxHeightMetric;
      const scale = Math.min(scaleX, scaleY) * 0.9;
      
      const totalRenderedWidth = totalWidthMetric * scale + spacing;
      const maxRenderedHeight = maxHeightMetric * scale;
      
      const startX = (canvas.width - totalRenderedWidth) / 2;
      const startY = (canvas.height - maxRenderedHeight) / 2;
      
      let currentX = startX;
      
      this.plates.forEach((plate, index) => {
        const plateWidthMetric = this.isMetric ? plate.width : this.convertToMetric(plate.width);
        const plateHeightMetric = this.isMetric ? plate.height : this.convertToMetric(plate.height);
        
        const plateWidthPx = plateWidthMetric * scale;
        const plateHeightPx = plateHeightMetric * scale;
        
        const y = startY + (maxRenderedHeight - plateHeightPx) / 2;
        
        this.renderPlateSegment(ctx, currentX, y, plateWidthPx, plateHeightPx, plate, index, plateWidthMetric, plateHeightMetric);
        
        ctx.fillStyle = index === this.activePlateIndex ? '#4caf50' : '#666';
        ctx.font = '14px sans-serif';
        ctx.textAlign = 'center';
        ctx.fillText(`${index + 1}`, currentX + plateWidthPx/2, y - 8);
        
        currentX += plateWidthPx + (index < this.plates.length - 1 ? 1 : 0);
      });
    },

    renderPlateSegment(ctx, x, y, width, height, plate, plateIndex, plateWidthCm, plateHeightCm) {
      if (!this.baseImage) {
        ctx.fillStyle = '#f0f0f0';
        ctx.fillRect(x, y, width, height);
        ctx.fillStyle = '#999';
        ctx.font = '12px sans-serif';
        ctx.textAlign = 'center';
        ctx.fillText('Loading...', x + width/2, y + height/2);
        return;
      }
      
      const totalWidthCm = this.plates.reduce((sum, p) => {
        return sum + (this.isMetric ? p.width : this.convertToMetric(p.width));
      }, 0);
      
      const imageWidth = this.baseImage.width || 1200;
      const imageHeight = this.baseImage.height || 800;
      
      const baseImageWidthCm = 300;
      let sourceX = 0;
      
      for (let i = 0; i < plateIndex; i++) {
        sourceX += this.isMetric ? this.plates[i].width : this.convertToMetric(this.plates[i].width);
      }
      
      const sourceXPx = (sourceX / baseImageWidthCm) * imageWidth;
      const sourceWidthPx = (plateWidthCm / baseImageWidthCm) * imageWidth;
      
      const maxHeightCm = 128;
      const sourceHeightPx = (plateHeightCm / maxHeightCm) * imageHeight;
      const sourceYPx = imageHeight - sourceHeightPx;
      
      let actualSourceX = sourceXPx;
      let actualSourceWidth = sourceWidthPx;
      let actualSourceY = Math.max(0, sourceYPx);
      let actualSourceHeight = sourceHeightPx;
      let shouldMirror = false;
      
      if (totalWidthCm > 300) {
        const normalizedX = sourceX % (baseImageWidthCm * 2);
        
        if (normalizedX > baseImageWidthCm) {
          shouldMirror = true;
          actualSourceX = imageWidth - ((normalizedX - baseImageWidthCm) / baseImageWidthCm) * imageWidth;
        } else {
          actualSourceX = (normalizedX / baseImageWidthCm) * imageWidth;
        }
        actualSourceWidth = (plateWidthCm / baseImageWidthCm) * imageWidth;
      }
      
      actualSourceX = Math.max(0, Math.min(actualSourceX, imageWidth - 1));
      actualSourceWidth = Math.min(actualSourceWidth, imageWidth - actualSourceX);
      actualSourceHeight = Math.min(actualSourceHeight, imageHeight - actualSourceY);
      
      const canvas = ctx.canvas;
      const FORCE_BOTTOM_Y = canvas.height - 30;
      const correctedY = FORCE_BOTTOM_Y - height;
      
      ctx.save();
      
      try {
        if (shouldMirror) {
          ctx.scale(-1, 1);
          ctx.drawImage(
            this.baseImage,
            actualSourceX,
            actualSourceY,
            actualSourceWidth,
            actualSourceHeight,
            -x - width,
            correctedY,
            width,
            height
          );
        } else {
          ctx.drawImage(
            this.baseImage,
            actualSourceX,
            actualSourceY,
            actualSourceWidth,
            actualSourceHeight,
            x,
            correctedY,
            width,
            height
          );
        }
        
      } catch (error) {
        console.error('Error drawing image segment:', error);
        ctx.fillStyle = plateIndex === 0 ? '#e8f5e8' : '#ffe8e8';
        ctx.fillRect(x, correctedY, width, height);
        ctx.fillStyle = '#666';
        ctx.font = '10px sans-serif';
        ctx.textAlign = 'center';
        ctx.fillText(`${plateHeightCm.toFixed(0)}cm`, x + width/2, correctedY + height/2);
      }
      
      ctx.restore();
    }
  },

  watch: {
    plates: {
      handler() {
        this.$nextTick(() => {
          this.renderCanvas();
        });
      },
      deep: true
    }
  }
}
</script>

<style scoped>
.app-container {
  display: flex;
  height: 100vh;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  background-color: #f9f9f9;
  overflow: hidden;
}

.sidebar {
  width: 50%;
  background-color: #e8e8e8;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 2rem;
}

.canvas-container {
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 1rem;
}

.canvas-wrapper {
  flex: 1;
  width: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  overflow: hidden;
  position: relative;
  border-radius: 8px;
  background: #f0f0f0;
}

.preview-canvas {
  border-radius: 4px;
  box-shadow: 0 4px 8px rgba(0,0,0,0.1);
  transform-origin: center center;
  transition: transform 0.1s ease-out;
  user-select: none;
  touch-action: none;
}

.zoom-controls {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  background: white;
  border-radius: 8px;
  padding: 0.5rem;
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
}

.zoom-btn {
  width: 36px;
  height: 36px;
  border: 1px solid #dee2e6;
  background: white;
  border-radius: 6px;
  cursor: pointer;
  font-size: 1.2rem;
  font-weight: bold;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.2s ease;
}

.zoom-btn:hover:not(:disabled) {
  background: #f8f9fa;
  border-color: #adb5bd;
}

.zoom-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.zoom-level {
  font-size: 0.9rem;
  font-weight: 500;
  color: #495057;
  min-width: 50px;
  text-align: center;
}

.reset-btn {
  padding: 0.5rem 1rem;
  border: 1px solid #dee2e6;
  background: white;
  border-radius: 6px;
  cursor: pointer;
  font-size: 0.8rem;
  font-weight: 500;
  transition: all 0.2s ease;
}

.reset-btn:hover {
  background: #f8f9fa;
  border-color: #adb5bd;
}

.export-controls {
  display: flex;
  gap: 1rem;
  flex-wrap: wrap;
  justify-content: center;
}

.export-btn {
  background-color: #2196f3;
  color: white;
  border: none;
  padding: 0.75rem 1.5rem;
  border-radius: 6px;
  cursor: pointer;
  font-size: 0.9rem;
  transition: all 0.2s ease;
  touch-action: manipulation;
  min-height: 44px;
  white-space: nowrap;
}

.export-btn:hover {
  background-color: #1976d2;
  transform: translateY(-1px);
  box-shadow: 0 4px 8px rgba(33, 150, 243, 0.3);
}

.main-content {
  width: 50%;
  background-color: white;
  padding: 3rem 2rem;
  display: flex;
  flex-direction: column;
  overflow-y: auto;
  -webkit-overflow-scrolling: touch;
}

.header {
  margin-bottom: 1rem;
}

.title {
  font-size: 2rem;
  font-weight: 300;
  color: #333;
  margin: 0 0 1.5rem 0;
  text-align: left;
  letter-spacing: -1px;
}

.controls-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 1rem;
  margin-bottom: 1rem;
}

.unit-toggle-btn {
  background-color: #f8f9fa;
  color: #6c757d;
  border: 2px solid #e9ecef;
  padding: 0.75rem 1.25rem;
  border-radius: 8px;
  cursor: pointer;
  font-weight: 600;
  font-size: 1rem;
  transition: all 0.3s ease;
  min-width: 60px;
  min-height: 44px;
  touch-action: manipulation;
}

.unit-toggle-btn.active {
  background-color: #000000;
  color: white;
  border-color: #000000;
}

.unit-toggle-btn:hover:not(.active) {
  background-color: #e9ecef;
  border-color: #dee2e6;
}

.upload-section {
  display: flex;
  align-items: center;
}

.upload-image-btn {
  background-color: #f8f9fa;
  color: #495057;
  border: 2px solid #e9ecef;
  padding: 0.75rem 1.25rem;
  border-radius: 8px;
  cursor: pointer;
  font-size: 0.9rem;
  font-weight: 500;
  transition: all 0.3s ease;
  touch-action: manipulation;
  min-height: 44px;
  white-space: nowrap;
}

.upload-image-btn:hover {
  background-color: #e9ecef;
  border-color: #dee2e6;
  transform: translateY(-1px);
}

.alert {
  background-color: #ffebee;
  color: #c62828;
  padding: 0.75rem 1rem;
  border-radius: 4px;
  margin-bottom: 1rem;
  border-left: 4px solid #c62828;
  font-size: 0.9rem;
  animation: slideIn 0.3s ease;
}

@keyframes slideIn {
  from { opacity: 0; transform: translateY(-10px); }
  to { opacity: 1; transform: translateY(0); }
}

.measurement-section {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 2rem;
}

.measurement-box {
  background-color: #f8f9fa;
  border-radius: 12px;
  padding: 2rem;
  border: none;
  transition: all 0.3s ease;
  cursor: grab;
  position: relative;
}

.measurement-box:hover {
  box-shadow: 0 4px 16px rgba(0,0,0,0.08);
}

.measurement-box.dragging {
  opacity: 0.5;
  transform: rotate(2deg);
  cursor: grabbing;
}

.measurement-box.animated {
  animation: bounce 0.3s ease;
}

@keyframes bounce {
  0%, 100% { transform: scale(1); }
  50% { transform: scale(1.02); }
}

.debug-info {
  margin-top: 2rem;
  padding: 1rem;
  background-color: #f8f9fa;
  border-radius: 4px;
  font-size: 0.8rem;
  color: #666;
}

.debug-info p {
  margin: 0.25rem 0;
}

.plate-row {
  display: flex;
  align-items: center;
  gap: 1.5rem;
  margin-bottom: 1.5rem;
}

.plate-row.first-plate {
  margin-bottom: 0;
}

.plate-number {
  width: 48px;
  height: 48px;
  border-radius: 8px;
  background-color: #ffffff;
  border: 2px solid #e9ecef;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: 600;
  font-size: 1.2rem;
  color: #6c757d;
  flex-shrink: 0;
  transition: all 0.3s ease;
}

.plate-number.active {
  background-color: #000000;
  color: white;
  border-color: #000000;
}

.plate-inputs {
  display: flex;
  align-items: center;
  gap: 1rem;
  flex-wrap: wrap;
  flex: 1;
}

.dimension-input {
  width: 80px;
  height: 48px;
  padding: 0 1rem;
  border: 1px solid #dee2e6;
  border-radius: 8px;
  font-size: 1.2rem;
  font-weight: 500;
  text-align: center;
  background-color: white;
  transition: all 0.2s ease;
  flex-shrink: 0;
}

.dimension-input:focus {
  outline: none;
  border-color: #000000;
  box-shadow: 0 0 0 3px rgba(0, 0, 0, 0.1);
}

.dimension-input.error {
  border-color: #dc3545;
  background-color: #fff5f5;
  animation: shake 0.5s ease;
}

@keyframes shake {
  0%, 100% { transform: translateX(0); }
  25% { transform: translateX(-4px); }
  75% { transform: translateX(4px); }
}

.unit-text {
  color: #6c757d;
  font-size: 1rem;
  font-weight: 500;
  flex-shrink: 0;
}

.multiply-symbol {
  color: #6c757d;
  font-size: 1.5rem;
  font-weight: 300;
  flex-shrink: 0;
}

.remove-btn {
  width: 40px;
  height: 40px;
  border: none;
  background-color: #ffebee;
  color: #f8585a;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  font-size: 1.5rem;
  font-weight: 300;
  transition: all 0.2s ease;
  touch-action: manipulation;
  min-height: 44px;
  min-width: 44px;
  flex-shrink: 0;
}

.remove-btn:hover {
  background-color: #f8585a;
  color: white;
  transform: scale(1.05);
}

.dimension-labels {
  display: flex;
  gap: 8rem;
  margin-left: 5rem;
  flex-wrap: wrap;
}

.label-group {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.label-title {
  font-weight: 600;
  color: #495057;
  font-size: 1rem;
}

.label-range {
  color: #9ba3af;
  font-size: 0.875rem;
}

.conversion-info {
  display: flex;
  gap: 8.5rem;
  margin-left: 5rem;
  margin-top: 0.5rem;
  flex-wrap: wrap;
}

.conversion-value {
  color: #9ba3af;
  font-size: 0.875rem;
  font-weight: 500;
}

.add-plate-btn {
  background-color: transparent;
  color: #28a745;
  border: 2px solid #e9f7ef;
  padding: 10px 20px 10px 20px;
  border-radius: 12px;
  cursor: pointer;
  font-size: 1rem;
  font-weight: 500;
  align-self: end;
  transition: all 0.3s ease;
  touch-action: manipulation;
}

.add-plate-btn:hover {
  background-color: #e9f7ef;
  border-color: #28a745;
  transform: translateY(-2px);
}

.add-plate-btn.animated {
  animation: pulse 0.3s ease;
}

@keyframes pulse {
  0%, 100% { transform: scale(1); }
  50% { transform: scale(1.05); }
}

@media (max-width: 768px) {
  .app-container {
    flex-direction: column;
    height: 100vh;
    overflow: hidden;
  }
  
  .sidebar {
    width: 100%;
    height: 40vh;
    padding: 1rem;
    flex-shrink: 0;
  }

  .canvas-wrapper {
    flex: 1;
    min-height: 200px;
  }

  .zoom-controls {
    padding: 0.4rem;
    gap: 0.4rem;
  }

  .zoom-btn {
    width: 32px;
    height: 32px;
    font-size: 1rem;
  }

  .zoom-level {
    font-size: 0.8rem;
    min-width: 40px;
  }

  .reset-btn {
    padding: 0.4rem 0.8rem;
    font-size: 0.75rem;
  }
  
  .main-content {
    width: 100%;
    height: 60vh;
    padding: 1rem;
    background-color: #f5f5f5;
    overflow-y: auto;
    flex-shrink: 0;
    display: flex;
    flex-direction: column;
  }
  
  .header {
    margin-bottom: 0rem;
    flex-shrink: 0;
  }
  
  .title {
    display: none;
  }
  
  .controls-row {
    justify-content: center;
    gap: 1rem;
    margin-bottom: 1rem;
  }
  
  .unit-toggle-btn, .upload-image-btn {
    font-size: 0.9rem;
    padding: 0.6rem 1rem;
    border-radius: 8px;
  }
  
  .alert {
    margin-bottom: 1rem;
    flex-shrink: 0;
  }
  
  .measurement-section {
    gap: 1rem;
    width: 100%;
    padding: 0;
    flex: 1;
    display: flex;
    flex-direction: column;
    min-height: 0;
    overflow-y: auto;
    padding-bottom: 2rem;
  }
  
  .measurement-box {
    background-color: white;
    border-radius: 16px;
    padding: 1.5rem;
    box-shadow: 0 2px 8px rgba(0,0,0,0.1);
    margin-bottom: 1rem;
    width: 100%;
    box-sizing: border-box;
    flex-shrink: 0;
  }
  
  .plate-row {
    gap: 0.75rem;
    margin-bottom: 1rem;
    width: 100%;
    flex-wrap: nowrap;
    align-items: center;
  }
  
  .plate-row.first-plate {
    margin-bottom: 1rem;
  }
  
  .plate-number {
    width: 40px;
    height: 40px;
    border-radius: 8px;
    font-size: 1.1rem;
    font-weight: 600;
    background-color: #f8f9fa;
    border: 2px solid #e9ecef;
    flex-shrink: 0;
  }
  
  .plate-number.active {
    background-color: #000000;
    color: white;
    border-color: #000000;
  }
  
  .plate-inputs {
    gap: 0.5rem;
    justify-content: flex-start;
    flex-wrap: nowrap;
    flex: 1;
    align-items: center;
    min-width: 0;
  }
  
  .dimension-input {
    width: 60px;
    height: 40px;
    font-size: 1rem;
    font-weight: 600;
    border: 1px solid #dee2e6;
    border-radius: 8px;
    flex-shrink: 0;
  }
  
  .unit-text {
    font-size: 0.9rem;
    color: #6c757d;
    font-weight: 500;
    flex-shrink: 0;
  }
  
  .multiply-symbol {
    font-size: 1.1rem;
    color: #6c757d;
    flex-shrink: 0;
  }
  
  .remove-btn {
    width: 36px;
    height: 36px;
    background-color: #ffebee;
    color: #f8585a;
    border-radius: 50%;
    font-size: 1.1rem;
    min-height: 44px;
    min-width: 44px;
    flex-shrink: 0;
  }
  
  .dimension-labels {
    margin-left: 0;
    gap: 2.5rem;
    justify-content: space-between;
    margin-top: 0;
    width: 100%;
    display: flex;
  }
  
  .label-group {
    flex: 1;
    text-align: left;
  }
  
  .label-title {
    font-weight: 600;
    color: #495057;
    font-size: 1rem;
    display: block;
    margin-bottom: 0.25rem;
  }
  
  .label-range {
    color: #9ba3af;
    font-size: 0.875rem;
    display: block;
  }
  
  .conversion-info {
    margin-left: 0;
    gap: 2.5rem;
    justify-content: space-between;
    margin-top: 0.5rem;
    width: 100%;
    display: flex;
  }
  
  .conversion-value {
    color: #9ba3af;
    font-size: 0.875rem;
    font-weight: 500;
    flex: 1;
    text-align: left;
  }
  
  .add-plate-btn {
    width: 100%;
    background-color: white;
    color: #28a745;
    border: 2px solid #e9f7ef;
    border-radius: 16px;
    padding: 1.25rem;
    font-size: 1rem;
    font-weight: 600;
    margin-top: 1rem;
    margin-bottom: 1rem;
    box-sizing: border-box;
    flex-shrink: 0;
  }
  
  .add-plate-btn:hover {
    background-color: #e9f7ef;
    transform: none;
  }
}

@media (max-width: 480px) {
  .main-content {
    padding: 0.75rem;
  }
  
  .measurement-box {
    padding: 1rem;
  }
  
  .plate-row {
    gap: 0.5rem;
  }
  
  .dimension-input {
    width: 50px;
    font-size: 0.95rem;
  }
  
  .unit-text {
    font-size: 0.8rem;
  }
  
  .multiply-symbol {
    font-size: 1rem;
  }
  
  .dimension-labels {
    gap: 2rem;
  }
  
  .conversion-info {
    gap: 2rem;
  }

  .zoom-controls {
    padding: 0.3rem;
    gap: 0.3rem;
  }

  .zoom-btn {
    width: 28px;
    height: 28px;
    font-size: 0.9rem;
  }

  .zoom-level {
    font-size: 0.7rem;
    min-width: 35px;
  }

  .reset-btn {
    padding: 0.3rem 0.6rem;
    font-size: 0.7rem;
  }
}
</style>