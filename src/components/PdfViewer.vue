<template>
  <div>
    <div class="sidebar">
      <h3>PDF Viewer with Coordinate Tracking</h3>
      <div class="input-container">
        <input type="file" @change="handleFileInput" accept="application/pdf">
      </div>
      <hr class="separator">
      <div class="mode-container">
        <select v-model="currentMode" @change="updateModeInstructions">
          <option value="point">Point Mode</option>
          <option value="area">Area Mode</option>
        </select>
        <span id="modeInstructions">{{ modeInstructions }}</span>
      </div>
      <div class="display-mode">
        <select v-model="currentDisplayMode" @change="handleDisplayModeChange">
          <option value="single">First Page</option>
          <option value="all">All Pages</option>
        </select>
      </div>
      <div class="pdf-info-container">
  <div>PDF Info:</div>
  <div id="pdfInfo">{{ pdfInfo }}</div>
</div>
<h3>List selected</h3>
<div class="button-container">
    <div class="left-buttons">
    <button @click="copyAllAreas" v-if="currentMode === 'area'">
      {{ copyButtonStates.copyAllAreas ? 'Copied' : 'Copy All Areas' }}
    </button>
    <button @click="copyXPoints" v-if="currentMode === 'point'">
      {{ copyButtonStates.copyXPoints ? 'Copied' : 'Copy X Points' }}
    </button>
  </div>
  <div class="right-buttons">
  <button @click="clear" class="red-button">Clear</button>
  <button @click="clearAll" class="red-button">Clear All</button>
</div>
</div>
<ul id="coordinateList">
  <li v-for="(item, index) in filteredCoordinateList" :key="index">
    <template v-if="currentMode === 'point'">
        <div class="list-item">
        <span class="item-content">{{ item.id }}. x={{ Math.round(item.x) }}, y={{ Math.round(item.y) }}</span>
        <button class="deleteBtn" @click="deletePoint(item.id)">&#9747;</button>
      </div>
    </template>
    <template v-else>
      <div class="area-container">
        <span>Area {{ item.id }}:</span>
    <div class="button-container-area">
      <button class="copyBtn" @click="copyAreaToClipboard(item)" :disabled="item.justCopied">
        {{ item.justCopied ? 'Copied' : 'Copy' }}
      </button>
      <button class="deleteBtn" @click="deleteArea(item.id)">&#9747;</button>
    </div>
        <pre>{
  "x_top_left": {{ item.x_top_left }},
  "y_top_left": {{ item.y_top_left }},
  "x_bottom_right": {{ item.x_bottom_right }},
  "y_bottom_right": {{ item.y_bottom_right }}
}</pre>
      </div>
    </template>
  </li>
</ul>
    </div>
    <div class="main-content">
      <div id="fileDisplayArea" class="file-container" ref="fileDisplayArea"></div>
      <div id="pdfCoordinates">{{ pdfCoordinates }}</div>
    </div>
  </div>
</template>

<script>
import * as pdfjsLib from 'pdfjs-dist'
import 'pdfjs-dist/build/pdf.worker.entry'
import '../assets/style.css'
export default {
  name: 'PdfViewer',
  data() {
    return {
      currentPdf: null,
      currentMouseX: 0,
      currentMouseY: 0,
      currentMode: 'point',
      keypressStartPoint: null,
      clickModeData: [],
      keypressModeData: [],
      pointCounter: 1,
      areaCounter: 1,
      temporaryPoint: null,
      isCtrlKeyPressed: false,
      temporaryArea: null,
      currentDisplayMode: 'single',
      currentPage: 1,
      totalPages: 0,
      modeInstructions: '',
      pdfInfo: '',
      pdfCoordinates: '',
      pdfPageSize: '',
      copyButtonStates: {
      copyAllAreas: false,
      copyXPoints: false,
    }
    }
  },
computed: {
  filteredCoordinateList() {
    return this.currentMode === 'point' ? this.clickModeData : this.keypressModeData;
  }
},
  mounted() {
    this.updateModeInstructions()
    document.addEventListener('keydown', this.handleKeyDown)
    document.addEventListener('keyup', this.handleKeyUp)
  },
  beforeDestroy() {
    document.removeEventListener('keydown', this.handleKeyDown)
    document.removeEventListener('keyup', this.handleKeyUp)
  },
  methods: {
    handleFileInput(event) {
  const file = event.target.files[0];
  if (file && file.type === 'application/pdf') {
    const reader = new FileReader();
    reader.onload = (e) => {
      this.$refs.fileDisplayArea.innerHTML = '';
      pdfjsLib.getDocument({ data: e.target.result }).promise.then((pdf) => {
        this.currentPdf = pdf;
        this.totalPages = pdf.numPages;
        this.displayPdfInfo(pdf); 
        this.renderPages(pdf, 1);
      });
    };
    reader.readAsArrayBuffer(file);
  } else {
    this.$refs.fileDisplayArea.innerHTML = 'Please select a PDF file.';
  }
},
    
    updateModeInstructions() {
      this.modeInstructions = this.currentMode === 'point' 
        ? 'Press "Ctrl" to mark a point' 
        : 'Press and hold "Ctrl", then release to mark an area'
    },
    handleDisplayModeChange() {
      this.currentPage = 1
      this.renderPages(this.currentPdf, this.currentPage)
      this.clearAll()
    },
    renderPages(pdf, pageNumber) {
      if (this.currentDisplayMode === 'single') {
        this.$refs.fileDisplayArea.innerHTML = ''
        this.renderPage(pdf, pageNumber)
      } else {
        if (pageNumber === 1) {
          this.$refs.fileDisplayArea.innerHTML = ''
        }
        this.renderPage(pdf, pageNumber).then(() => {
          if (pageNumber < pdf.numPages) {
            this.renderPages(pdf, pageNumber + 1)
          }
        })
      }
    },
    renderPage(pdf, pageNumber) {
    return pdf.getPage(pageNumber).then((page) => {
    const viewport = page.getViewport({scale: 1});
    const canvas = document.createElement('canvas');
    const context = canvas.getContext('2d');
    const scale = this.$refs.fileDisplayArea.clientWidth / viewport.width;
    const scaledViewport = page.getViewport({scale: scale});
    this.pdfPageSize = `${Math.round(viewport.width)} x ${Math.round(viewport.height)}`

    canvas.height = scaledViewport.height
    canvas.width = scaledViewport.width

    const pdfOverlay = this.createPdfOverlay(scaledViewport, viewport, pageNumber)
    pdfOverlay.appendChild(canvas)

    return page.render({canvasContext: context, viewport: scaledViewport}).promise.then(() => {
      return page.getTextContent()
    }).then((textContent) => {
      const textLayerDiv = this.createTextLayerDiv(scaledViewport)
      pdfOverlay.appendChild(textLayerDiv)
      pdfjsLib.renderTextLayer({
        textContent: textContent,
        container: textLayerDiv,
        viewport: scaledViewport,
        textDivs: []
      })
      this.redrawPoints(pdfOverlay, pageNumber);
      this.redrawAreas(pdfOverlay, pageNumber);
    })
  })
},
redrawAreas(overlay, pageNumber) {
  this.keypressModeData.forEach(area => {
    if (area.page == pageNumber) {
      const markedArea = this.createMarkedArea(area, area.id);
      overlay.appendChild(markedArea);
    }
  });
},
redrawPoints(overlay, pageNumber) {
  this.clickModeData.forEach(point => {
    if (point.page == pageNumber) {
      const clickPoint = this.createClickPoint({
        clickX: (point.x / this.pdfPageSize.width) * overlay.offsetWidth,
        clickY: ((this.pdfPageSize.height - point.y) / this.pdfPageSize.height) * overlay.offsetHeight
      }, point.id);
      overlay.appendChild(clickPoint);
    }
  });
},
    createPdfOverlay(scaledViewport, viewport, pageNumber) {
      const pdfOverlay = document.createElement('div')
      pdfOverlay.className = 'pdfOverlay'
      pdfOverlay.style.width = `${scaledViewport.width}px`
      pdfOverlay.style.height = `${scaledViewport.height}px`
      pdfOverlay.setAttribute('data-original-width', viewport.width)
      pdfOverlay.setAttribute('data-original-height', viewport.height)
      pdfOverlay.setAttribute('data-page-number', pageNumber)
      pdfOverlay.addEventListener('mousemove', this.handleMouseMove)
      this.$refs.fileDisplayArea.appendChild(pdfOverlay)
      return pdfOverlay
    },
    createTextLayerDiv(viewport) {
      const textLayerDiv = document.createElement('div')
      textLayerDiv.className = 'textLayer'
      textLayerDiv.style.width = `${viewport.width}px`
      textLayerDiv.style.height = `${viewport.height}px`
      return textLayerDiv
    },
    handleMouseMove(e) {
      const coordinates = this.getCoordinates(e)
      this.currentMouseX = e.clientX
      this.currentMouseY = e.clientY
      this.pdfCoordinates = `x=${Math.round(coordinates.x)},  y=${Math.round(coordinates.y)}`

      if (this.isCtrlKeyPressed && this.currentMode === 'area' && this.keypressStartPoint) {
        this.updateTemporaryArea(this.keypressStartPoint, coordinates)
      }
    },
    getCoordinates(e) {
      const overlay = e.target.closest('.pdfOverlay')
      if (!overlay) return null

      const pageNumber = overlay.getAttribute('data-page-number')
      const originalWidth = parseFloat(overlay.getAttribute('data-original-width'))
      const originalHeight = parseFloat(overlay.getAttribute('data-original-height'))
      const bounds = overlay.getBoundingClientRect()

      const clickX = e.clientX - bounds.left
      const clickY = e.clientY - bounds.top

      const x = (clickX / bounds.width) * originalWidth
      const y = (clickY / bounds.height) * originalHeight

      return {
        x: x,
        y: originalHeight - y,
        clickX: clickX,
        clickY: clickY,
        page: pageNumber
      }
    },
    handleKeyDown(e) {
  if ((e.key === 'Control') && !this.isCtrlKeyPressed) {
    this.isCtrlKeyPressed = true;
    const overlay = document.elementFromPoint(this.currentMouseX, this.currentMouseY).closest('.pdfOverlay');
    if (overlay) {
      const coordinates = this.getCoordinates({ clientX: this.currentMouseX, clientY: this.currentMouseY, target: overlay });
      if (this.currentMode === 'area') {
        this.keypressStartPoint = coordinates;
        this.markTemporaryPoint(this.keypressStartPoint);
        this.updateTemporaryArea(this.keypressStartPoint, coordinates);
      } else if (this.currentMode === 'point') {
        this.markPoint(coordinates);
      }
    }
  }
},
handleKeyUp(e) {
  if (e.key === 'Control' && this.isCtrlKeyPressed) {
    this.isCtrlKeyPressed = false;
    if (this.currentMode === 'area' && this.keypressStartPoint) {
      const overlay = document.elementFromPoint(this.currentMouseX, this.currentMouseY).closest('.pdfOverlay');
      if (overlay) {
        const endPoint = this.getCoordinates({ clientX: this.currentMouseX, clientY: this.currentMouseY, target: overlay });
        this.markArea(this.keypressStartPoint, endPoint);
      }
    }
    this.keypressStartPoint = null;
    this.clearTemporaryMarkings();
  }
},
markPoint(coordinates) {
  if (!coordinates) return;
  const newPoint = {
    id: this.pointCounter++,
    x: coordinates.x,
    y: coordinates.y,
    page: coordinates.page
  };
  this.clickModeData.push(newPoint);
  const overlay = document.querySelector(`.pdfOverlay[data-page-number="${coordinates.page}"]`);
  if (overlay) {
    const clickPoint = this.createClickPoint(coordinates, newPoint.id);
    overlay.appendChild(clickPoint);
  }
},

markTemporaryPoint(coordinates) {
  this.clearTemporaryMarkings();
  this.temporaryPoint = this.createTemporaryMark(coordinates.clickX, coordinates.clickY);
},
updateTemporaryArea(start, end) {
  if (!start || !end) return;
  this.clearTemporaryArea();
  this.temporaryArea = this.createTemporaryArea(
    Math.min(start.clickX, end.clickX),
    Math.min(start.clickY, end.clickY),
    Math.abs(start.clickX - end.clickX),
    Math.abs(start.clickY - end.clickY)
  );
},
markArea(start, end) {
  if (!start || !end || start.page !== end.page) return;
  const area = {
    id: this.areaCounter++,
    page: start.page,
    x_top_left: Math.round(Math.min(start.x, end.x)),
    y_top_left: Math.round(Math.max(start.y, end.y)),
    x_bottom_right: Math.round(Math.max(start.x, end.x)),
    y_bottom_right: Math.round(Math.min(start.y, end.y))
  };
  this.keypressModeData.push(area);

  const overlay = document.querySelector(`.pdfOverlay[data-page-number="${area.page}"]`);
  if (overlay) {
    const markedArea = this.createMarkedArea(area, area.id);
    if (markedArea) {
      overlay.appendChild(markedArea);
    }
  }
},
    clearTemporaryMarkings() {
      this.clearTemporaryPoint()
      this.clearTemporaryArea()
    },
    clearTemporaryPoint() {
      if (this.temporaryPoint) {
        this.temporaryPoint.remove()
        this.temporaryPoint = null
      }
    },
    clearTemporaryArea() {
      if (this.temporaryArea) {
        this.temporaryArea.remove()
        this.temporaryArea = null
      }
    },
    createTemporaryMark(x, y) {
      const mark = document.createElement('div')
      mark.className = 'temporaryMark'
      mark.style.left = `${x}px`
      mark.style.top = `${y}px`
      this.$refs.fileDisplayArea.appendChild(mark)
      return mark
    },
    createClickPoint(coordinates, id) {
  const clickPoint = document.createElement('div');
  clickPoint.className = 'clickPoint';
  clickPoint.setAttribute('data-point-id', id);
  clickPoint.style.left = `${coordinates.clickX}px`;
  clickPoint.style.top = `${coordinates.clickY}px`;
  
  // Tạo dấu cộng
  const horizontalLine = document.createElement('div');
  horizontalLine.className = 'crosshair horizontal';
  const verticalLine = document.createElement('div');
  verticalLine.className = 'crosshair vertical';
  
  clickPoint.appendChild(horizontalLine);
  clickPoint.appendChild(verticalLine);

  // Thêm số ID
  const numberElement = document.createElement('span');
  numberElement.className = 'pointNumber';
  numberElement.textContent = id;
  clickPoint.appendChild(numberElement);

  return clickPoint;
},
createMarkedArea(coordinates, id) {
  const markedArea = document.createElement('div');
  markedArea.className = 'markedArea';
  markedArea.setAttribute('data-area-id', id);
  
  const overlay = document.querySelector(`.pdfOverlay[data-page-number="${coordinates.page}"]`);
  if (!overlay) return null;

  const originalWidth = parseFloat(overlay.getAttribute('data-original-width'));
  const originalHeight = parseFloat(overlay.getAttribute('data-original-height'));
  
  const scaleX = overlay.offsetWidth / originalWidth;
  const scaleY = overlay.offsetHeight / originalHeight;

  const left = coordinates.x_top_left * scaleX;
  const top = (originalHeight - coordinates.y_top_left) * scaleY;
  const width = (coordinates.x_bottom_right - coordinates.x_top_left) * scaleX;
  const height = (coordinates.y_top_left - coordinates.y_bottom_right) * scaleY;
  
  Object.assign(markedArea.style, {
    left: `${left}px`,
    top: `${top}px`,
    width: `${width}px`,
    height: `${height}px`,
    position: 'absolute',
    border: '2px solid red',
    backgroundColor: 'rgba(255, 0, 0, 0.1)',
  });

  const numberElement = document.createElement('div');
  numberElement.className = 'areaNumber';
  numberElement.textContent = id;
  markedArea.appendChild(numberElement);

  return markedArea;
},
    createTemporaryArea(x, y, width, height) {
      const area = document.createElement('div')
      area.className = 'temporaryArea'
      area.style.left = `${x}px`
      area.style.top = `${y}px`
      area.style.width = `${width}px`
      area.style.height = `${height}px`
      this.$refs.fileDisplayArea.appendChild(area)
      return area
    },
    displayPdfInfo(pdf) {
  pdf.getMetadata().then((data) => {
    pdf.getPage(1).then((page) => {
      const viewport = page.getViewport({scale: 1});
      this.pdfPageSize = `${Math.round(viewport.width)} x ${Math.round(viewport.height)}`;
      this.pdfInfo = `Title: ${data.info.Title || 'N/A'}, Size: ${this.pdfPageSize}`;
    });
  });
},
copyXPoints() {
    const text = this.clickModeData.map(point => `${Math.round(point.x)}`).join(',')
    this.copyToClipboard(text)
    this.updateCopyButtonState('copyXPoints')
  },
  copyAllAreas() {
    const areasJson = this.keypressModeData.map(area => ({
      x_top_left: Math.round(area.x_top_left),
      y_top_left: Math.round(area.y_top_left),
      x_bottom_right: Math.round(area.x_bottom_right),
      y_bottom_right: Math.round(area.y_bottom_right)
    }));
    
    const text = JSON.stringify(areasJson, null, 2);
    this.copyToClipboard(text);
    this.updateCopyButtonState('copyAllAreas')
  },
  copyAreaToClipboard(area) {
  const text = JSON.stringify({
    x_top_left: Math.round(area.x_top_left),
    y_top_left: Math.round(area.y_top_left),
    x_bottom_right: Math.round(area.x_bottom_right),
    y_bottom_right: Math.round(area.y_bottom_right)
  }, null, 2);
  this.copyToClipboard(text);
  
  // Sử dụng Vue.set để đảm bảo reactivity
  this.$set(area, 'justCopied', true);
  setTimeout(() => {
    this.$set(area, 'justCopied', false);
  }, 2000);
},
    copyToClipboard(text) {
      const textarea = document.createElement('textarea')
      textarea.value = text
      document.body.appendChild(textarea)
      textarea.select()
      document.execCommand('copy')
      document.body.removeChild(textarea)
    },
    updateCopyButtonState(buttonName) {
    this.copyButtonStates[buttonName] = true;
    setTimeout(() => {
      this.copyButtonStates[buttonName] = false;
    }, 2000);
  },
    deletePoint(id) {
  this.clickModeData = this.clickModeData.filter(item => item.id !== id);
  const clickPoint = document.querySelector(`.clickPoint[data-point-id="${id}"]`);
  if (clickPoint) {
    clickPoint.remove();
  }
  if (this.clickModeData.length === 0) {
    this.pointCounter = 1;
  }
},
deleteArea(id) {
  this.keypressModeData = this.keypressModeData.filter(area => area.id !== id);
  const markedArea = document.querySelector(`.markedArea[data-area-id="${id}"]`);
  if (markedArea) {
    markedArea.remove();
  }
  if (this.keypressModeData.length === 0) {
    this.areaCounter = 1;
  }
},
clear() {
  if (this.currentMode === 'point') {
    this.clickModeData = [];
    document.querySelectorAll('.clickPoint').forEach(point => point.remove());
    this.pointCounter = 1;
  } else {
    this.keypressModeData = [];
    document.querySelectorAll('.markedArea').forEach(area => area.remove());
    this.areaCounter = 1;
  }
},
    clearAll() {
      this.clickModeData = []
      this.keypressModeData = []
      document.querySelectorAll('.clickPoint').forEach(point => point.remove());
      this.pointCounter = 1;
      document.querySelectorAll('.markedArea').forEach(area => area.remove());
      this.areaCounter = 1;
    },
    formatAreaText(area) {
      return `(${Math.round(area.startX)}, ${Math.round(area.startY)}) to (${Math.round(area.endX)}, ${Math.round(area.endY)})`
    }
  }
}
</script>

