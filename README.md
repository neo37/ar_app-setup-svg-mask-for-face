# setup svg Face Mask online while it detecting. face tracking, mediapipe, js and svg with Neo Brutalism Design
## Demo [pizda.online](https://pizda.online)

This document contains two versions of an HTML page that implements a face mask effect using MediaPipe Face Mesh and a neo‑brutalism design style. The page allows users to choose the image source – either live camera video (default) or an uploaded photo. Facial landmarks are detected with MediaPipe Face Mesh, then a mask built from polygons (obtained via clustering and computing a convex hull) “sticks” to the face. When the freeze mode is enabled, the polygon shapes are computed once and then only transformed on each update to remain attached to the face.

Additionally, a top menu with three link buttons is placed at the top of the page, and in the global settings area (top left) there are two input fields and an **Apply Global Settings** button. When clicked, for each polygon the following occurs:
  
1. **Computation of the average color** from the area of the video (or photo) covered by that polygon.  
2. **Application of this color** as the fill and stroke, with the fill opacity and stroke width set according to the entered values.

Save the code in a single HTML file, run it over HTTPS (or on localhost), and enjoy the neo‑brutalist design!

---

## Russian Version

```html
<!DOCTYPE html>
<html lang="ru">
<head>
  <meta charset="UTF-8">
  <title>Neo Brutalism Face Mask</title>
  <style>
    /* Основной стиль в духе neo brutalism: жёсткие линии, контраст, моноширинный шрифт */
    * { box-sizing: border-box; }
    body, html {
      margin: 0;
      padding: 0;
      font-family: "Arial Black", Gadget, sans-serif;
      background: #f5f5f5;
      color: #111;
      overflow: hidden;
    }
    /* Верхнее меню */
    #topMenu {
      position: fixed;
      top: 0;
      left: 0;
      width: 100%;
      background: #000;
      border-bottom: 4px solid #ff0033;
      padding: 10px 0;
      z-index: 30;
      text-align: center;
    }
    #topMenu a {
      color: #ff0033;
      text-decoration: none;
      margin: 0 15px;
      padding: 8px 16px;
      border: 2px solid #ff0033;
      transition: background 0.3s;
    }
    #topMenu a:hover {
      background: #ff0033;
      color: #fff;
    }
    /* Глобальные настройки */
    #globalControls {
      position: fixed;
      top: 60px;
      left: 10px;
      background: #000;
      color: #ff0033;
      padding: 10px;
      border: 4px solid #ff0033;
      border-radius: 4px;
      z-index: 30;
      font-size: 14px;
    }
    #globalControls label {
      display: block;
      margin-bottom: 8px;
    }
    #globalControls input {
      width: 80px;
      padding: 2px 4px;
      font-family: inherit;
      font-size: 14px;
      color: #111;
    }
    #globalControls button {
      margin-top: 5px;
      padding: 6px 12px;
      border: 2px solid #ff0033;
      background: #000;
      color: #ff0033;
      cursor: pointer;
      transition: background 0.3s;
    }
    #globalControls button:hover {
      background: #ff0033;
      color: #fff;
    }
    /* Контейнер для видео/фото и маски */
    #videoContainer {
      position: relative;
      width: 100vw;
      height: calc(100vh - 60px);
      margin-top: 60px; /* место под меню */
      background: #000;
    }
    video, #uploadedImage {
      width: 100%;
      height: 100%;
      object-fit: cover;
      display: block;
    }
    /* SVG-оверлей для маски */
    #mask {
      position: absolute;
      top: 0;
      left: 0;
      width: 100%;
      height: 100%;
      pointer-events: none;
    }
    /* Панель dat.GUI – оставляем базовое позиционирование */
    #guiContainer {
      position: fixed;
      top: 10px;
      right: 10px;
      z-index: 25;
    }
  </style>
</head>
<body>
  <!-- Верхнее меню -->
  <div id="topMenu">
    <a href="dev/i.php" target="_blank">Dev i.php</a>
    <a href="dev/i2.php" target="_blank">Dev i2.php</a>
    <a href="dev/i3.php" target="_blank">Dev i3.php</a>
  </div>
  
  <!-- Глобальные настройки -->
  <div id="globalControls">
    <label>
      Прозрачность всех (0–1): 
      <input type="number" id="globalFillOpacity" min="0" max="1" step="0.01" value="0.3">
    </label>
    <label>
      Толщина линий всех: 
      <input type="number" id="globalStrokeWidth" min="1" max="10" step="0.5" value="2">
    </label>
    <label>
      Загрузить свою фотографию:
      <input type="file" id="photoUpload" accept="image/*">
    </label>
    <button id="applyGlobal">Применить глобальные настройки</button>
  </div>

  <div id="videoContainer">
    <!-- Видео по умолчанию; если пользователь загрузит фото, оно заменит видео -->
    <video id="video" autoplay muted playsinline></video>
    <!-- Элемент для загруженного фото (скрыт, пока не выбрано фото) -->
    <img id="uploadedImage" style="display:none;" alt="Загруженная фотография">
    <!-- SVG-оверлей с viewBox 0 0 640 480 -->
    <svg id="mask" viewBox="0 0 640 480" preserveAspectRatio="none">
      <g id="facePolygons"></g>
    </svg>
    <div id="guiContainer"></div>
  </div>

  <!-- Скрытый canvas для выборки цвета -->
  <canvas id="hiddenCanvas" width="640" height="480" style="display:none;"></canvas>

  <!-- Подключаем MediaPipe Face Mesh и утилиты -->
  <script src="https://cdn.jsdelivr.net/npm/@mediapipe/face_mesh/face_mesh.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/@mediapipe/drawing_utils/drawing_utils.js"></script>
  <!-- Подключаем dat.GUI -->
  <script src="https://cdnjs.cloudflare.com/ajax/libs/dat-gui/0.7.7/dat.gui.min.js"></script>
  
  <script>
    "use strict";
    
    /**********************************
     * Глобальные переменные и настройки
     **********************************/
    const videoWidth = 640, videoHeight = 480;
    let usePhoto = false;  // Флаг: используется ли загруженная фотография
    let uploadedPhoto = null;
    
    const videoElement = document.getElementById('video');
    const uploadedImageElement = document.getElementById('uploadedImage');
    
    // Параметры маски
    const maskParams = {
      polygonCount: 36,      // по умолчанию 36 многоугольников, максимум 48
      freezePolygons: true   // режим фиксации: локальная форма вычисляется один раз
    };
    
    // Массив индивидуальных стилей для многоугольников
    let polygonStyles = [];
    // Сохранённая локальная форма многоугольников (координаты в системе лица)
    let frozenPolygonsLocal = null;
    // Трансформация лица в момент фиксации
    let frozenFaceTransform = null;
    // Последние landmark от Face Mesh
    let currentLandmarks = null;
    // Текущие (вычисленные) многоугольники в пиксельных координатах
    let currentPolygons = [];
    
    /**********************************
     * Вычисление трансформации лица
     **********************************/
    function computeFaceTransform(landmarks) {
      const lmRight = landmarks[33], lmLeft = landmarks[263];
      const re = [lmRight.x * videoWidth, lmRight.y * videoHeight],
            le = [lmLeft.x * videoWidth, lmLeft.y * videoHeight];
      const dx = le[0] - re[0], dy = le[1] - re[1];
      const angle = Math.atan2(dy, dx) * 180 / Math.PI;
      const scale = Math.hypot(dx, dy) * 2.5;
      let sumX = 0, sumY = 0;
      landmarks.forEach(lm => { sumX += lm.x; sumY += lm.y; });
      const center = [ (sumX/landmarks.length)*videoWidth, (sumY/landmarks.length)*videoHeight ];
      return { center, scale, angle };
    }
    
    function computeLocalCoordinates(p, faceTransform) {
      const vx = p[0] - faceTransform.center[0],
            vy = p[1] - faceTransform.center[1];
      const rad = -faceTransform.angle * Math.PI/180;
      const rx = vx * Math.cos(rad) - vy * Math.sin(rad),
            ry = vx * Math.sin(rad) + vy * Math.cos(rad);
      return [ rx / faceTransform.scale, ry / faceTransform.scale ];
    }
    
    function applyTransform(localPoint, faceTransform) {
      const rad = faceTransform.angle * Math.PI/180;
      const rx = localPoint[0] * faceTransform.scale,
            ry = localPoint[1] * faceTransform.scale;
      const xRot = rx * Math.cos(rad) - ry * Math.sin(rad),
            yRot = rx * Math.sin(rad) + ry * Math.cos(rad);
      return [ faceTransform.center[0] + xRot, faceTransform.center[1] + yRot ];
    }
    
    /**********************************
     * Функции кластеризации и выпуклой оболочки
     **********************************/
    function dist2(a, b) {
      const dx = a[0]-b[0], dy = a[1]-b[1];
      return dx*dx+dy*dy;
    }
    
    function kMeans(points, k, iterations = 10) {
      let centroids = [];
      for (let i = 0; i < k; i++) {
        centroids.push(points[Math.floor(Math.random() * points.length)]);
      }
      let clusters = [];
      for (let iter = 0; iter < iterations; iter++) {
        clusters = Array.from({length: k}, () => []);
        for (const p of points) {
          let minD = Infinity, clusterIndex = 0;
          for (let i = 0; i < k; i++) {
            const d = dist2(p, centroids[i]);
            if (d < minD) { minD = d; clusterIndex = i; }
          }
          clusters[clusterIndex].push(p);
        }
        for (let i = 0; i < k; i++) {
          if (clusters[i].length === 0) {
            centroids[i] = points[Math.floor(Math.random() * points.length)];
          } else {
            let sumX = 0, sumY = 0;
            clusters[i].forEach(p => { sumX += p[0]; sumY += p[1]; });
            centroids[i] = [ sumX/clusters[i].length, sumY/clusters[i].length ];
          }
        }
      }
      return clusters;
    }
    
    function convexHull(points) {
      if (points.length <= 3) return points.slice();
      const pts = points.slice().sort((a,b)=> a[0]-b[0] || a[1]-b[1]);
      const lower = [];
      pts.forEach(p=>{
        while(lower.length>=2 && cross(lower[lower.length-2], lower[lower.length-1], p) <= 0){
          lower.pop();
        }
        lower.push(p);
      });
      const upper = [];
      for(let i = pts.length-1; i>=0; i--){
        const p = pts[i];
        while(upper.length>=2 && cross(upper[upper.length-2], upper[upper.length-1], p) <= 0){
          upper.pop();
        }
        upper.push(p);
      }
      lower.pop();
      upper.pop();
      return lower.concat(upper);
      function cross(A,B,C){
        return (B[0]-A[0])*(C[1]-A[1]) - (B[1]-A[1])*(C[0]-A[0]);
      }
    }
    
    /**********************************
     * Обновление и отрисовка маски
     **********************************/
    // Если фиксирование включено, локальная форма вычисляется один раз и затем трансформируется
    function updatePolygons() {
      const group = document.getElementById('facePolygons');
      // Если не фиксировано, сбрасываем сохранённое
      if (!maskParams.freezePolygons) { frozenPolygonsLocal = null; }
      if (!currentLandmarks) return;
      
      // Преобразуем landmark в пиксельные координаты
      const points = currentLandmarks.map(lm => [lm.x*videoWidth, lm.y*videoHeight]);
      const k = Math.max(1, Math.floor(maskParams.polygonCount));
      
      let newPolygons;
      if (maskParams.freezePolygons) {
        if (!frozenPolygonsLocal) {
          const clusters = kMeans(points, k, 10);
          const frozenPolys = clusters.map(cluster => {
            return (cluster.length < 3) ? null : convexHull(cluster);
          });
          frozenFaceTransform = computeFaceTransform(currentLandmarks);
          frozenPolygonsLocal = frozenPolys.map(poly => {
            if (!poly) return null;
            return poly.map(p => computeLocalCoordinates(p, frozenFaceTransform));
          });
        }
        const currentFaceTransform = computeFaceTransform(currentLandmarks);
        newPolygons = frozenPolygonsLocal.map(polyLocal => {
          if (!polyLocal) return null;
          return polyLocal.map(ptLocal => applyTransform(ptLocal, currentFaceTransform));
        });
      } else {
        const clusters = kMeans(points, k, 10);
        newPolygons = clusters.map(cluster => {
          return (cluster.length < 3) ? null : convexHull(cluster);
        });
      }
      currentPolygons = newPolygons;
      drawPolygons(newPolygons);
    }
    
    function drawPolygons(newPolygons) {
      const group = document.getElementById('facePolygons');
      group.innerHTML = '';
      if (!newPolygons) return;
      newPolygons.forEach((poly, idx) => {
        if (!poly) return;
        const pointsStr = poly.map(p => p[0].toFixed(0)+','+p[1].toFixed(0)).join(' ');
        const polygon = document.createElementNS("http://www.w3.org/2000/svg", "polygon");
        polygon.setAttribute('points', pointsStr);
        if (!polygonStyles[idx]) {
          polygonStyles[idx] = {
            color: "#" + Math.floor(Math.random()*16777215).toString(16).padStart(6,'0'),
            fillOpacity: 0.3,
            strokeOpacity: 1,
            strokeWidth: 2
          };
          updatePolygonStylesGUI();
        }
        const style = polygonStyles[idx];
        polygon.setAttribute('fill', style.color);
        polygon.setAttribute('fill-opacity', style.fillOpacity);
        polygon.setAttribute('stroke', style.color);
        polygon.setAttribute('stroke-opacity', style.strokeOpacity);
        polygon.setAttribute('stroke-width', style.strokeWidth);
        group.appendChild(polygon);
      });
    }
    
    /**********************************
     * Функция для вычисления среднего цвета внутри многоугольника
     **********************************/
    function pointInPolygon(x, y, poly) {
      let inside = false;
      for (let i = 0, j = poly.length - 1; i < poly.length; j = i++) {
        const xi = poly[i][0], yi = poly[i][1],
              xj = poly[j][0], yj = poly[j][1];
        const intersect = ((yi > y) !== (yj > y)) &&
                          (x < (xj - xi) * (y - yi) / (yj - yi + 0.00001) + xi);
        if (intersect) inside = !inside;
      }
      return inside;
    }
    
    function averageColorForPolygon(poly, ctx) {
      let minX = Infinity, minY = Infinity, maxX = -Infinity, maxY = -Infinity;
      poly.forEach(([x,y]) => {
        if (x < minX) minX = x;
        if (y < minY) minY = y;
        if (x > maxX) maxX = x;
        if (y > maxY) maxY = y;
      });
      minX = Math.floor(minX); minY = Math.floor(minY);
      maxX = Math.ceil(maxX); maxY = Math.ceil(maxY);
      const width = maxX - minX, height = maxY - minY;
      if (width <= 0 || height <= 0) return "#000000";
      const imageData = ctx.getImageData(minX, minY, width, height);
      const data = imageData.data;
      let rSum = 0, gSum = 0, bSum = 0, count = 0;
      for (let j = 0; j < height; j++) {
        for (let i = 0; i < width; i++) {
          const x = i + minX, y = j + minY;
          if (pointInPolygon(x, y, poly)) {
            const index = (j * width + i) * 4;
            rSum += data[index];
            gSum += data[index+1];
            bSum += data[index+2];
            count++;
          }
        }
      }
      if (count === 0) return "#000000";
      const rAvg = Math.round(rSum/count),
            gAvg = Math.round(gSum/count),
            bAvg = Math.round(bSum/count);
      return "#" + [rAvg, gAvg, bAvg].map(c=>c.toString(16).padStart(2,'0')).join('');
    }
    
    /**********************************
     * Обновление панели GUI для стилей многоугольников
     **********************************/
    let polygonGUIFolder = null;
    function updatePolygonStylesGUI() {
      if (polygonGUIFolder) { gui.removeFolder(polygonGUIFolder); }
      polygonGUIFolder = gui.addFolder('Стили многоугольников');
      polygonStyles.forEach((style, idx) => {
        const folder = polygonGUIFolder.addFolder('Многоугольник ' + (idx+1));
        folder.addColor(style, 'color')
          .name('Цвет')
          .onChange(val => { style.color = val; updatePolygons(); });
        folder.add(style, 'fillOpacity', 0, 1)
          .name('Прозрачность заливки')
          .onChange(val => { style.fillOpacity = val; updatePolygons(); });
        folder.add(style, 'strokeOpacity', 0, 1)
          .name('Прозрачность обводки')
          .onChange(val => { style.strokeOpacity = val; updatePolygons(); });
        folder.add(style, 'strokeWidth', 1, 10)
          .name('Толщина линии')
          .onChange(val => { style.strokeWidth = val; updatePolygons(); });
        folder.open();
      });
      polygonGUIFolder.open();
    }
    
    /**********************************
     * Инициализация панели dat.GUI
     **********************************/
    const gui = new dat.GUI({ autoPlace: false });
    document.getElementById('guiContainer').appendChild(gui.domElement);
    gui.add(maskParams, 'polygonCount', 1, 48).step(1)
       .name('Кол-во многоугольников')
       .onChange(val => {
         frozenPolygonsLocal = null;
         polygonStyles = [];
         updatePolygons();
         updatePolygonStylesGUI();
       });
    gui.add(maskParams, 'freezePolygons')
       .name('Фиксировать многоугольники')
       .onChange(val => { if (!val) { frozenPolygonsLocal = null; } });
    
    /**********************************
     * Обработка глобальных настроек
     **********************************/
    document.getElementById('applyGlobal').addEventListener('click', function() {
      const globalFillOpacity = parseFloat(document.getElementById('globalFillOpacity').value);
      const globalStrokeWidth = parseFloat(document.getElementById('globalStrokeWidth').value);
      
      const hiddenCanvas = document.getElementById('hiddenCanvas');
      const ctx = hiddenCanvas.getContext('2d');
      // Рисуем текущий источник (видео или фотографию) на canvas
      if (usePhoto && uploadedPhoto) {
        ctx.drawImage(uploadedPhoto, 0, 0, videoWidth, videoHeight);
      } else {
        ctx.drawImage(videoElement, 0, 0, videoWidth, videoHeight);
      }
      
      currentPolygons.forEach((poly, idx) => {
        if (!poly) return;
        const avgColor = averageColorForPolygon(poly, ctx);
        polygonStyles[idx].color = avgColor;
        polygonStyles[idx].fillOpacity = globalFillOpacity;
        polygonStyles[idx].strokeWidth = globalStrokeWidth;
        polygonStyles[idx].strokeOpacity = 1;
      });
      drawPolygons(currentPolygons);
      updatePolygonStylesGUI();
    });
    
    /**********************************
     * Обработка загрузки фото
     **********************************/
    document.getElementById('photoUpload').addEventListener('change', function(e) {
      const file = e.target.files[0];
      if (!file) return;
      const reader = new FileReader();
      reader.onload = function(event) {
        // Создаём элемент изображения
        uploadedPhoto = new Image();
        uploadedPhoto.onload = function() {
          usePhoto = true;
          // Скрываем видео и показываем фото
          videoElement.style.display = "none";
          uploadedImageElement.src = uploadedPhoto.src;
          uploadedImageElement.style.display = "block";
          // Если камера уже запущена, остановим её (если возможно)
          if (camera && typeof camera.stop === "function") {
            camera.stop();
          }
          // Запускаем Face Mesh один раз для фото (так как фото статичное)
          faceMesh.send({ image: uploadedPhoto });
        };
        uploadedPhoto.src = event.target.result;
      };
      reader.readAsDataURL(file);
    });
    
    /**********************************
     * Настройка MediaPipe Face Mesh
     **********************************/
    const faceMesh = new FaceMesh({
      locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/face_mesh/${file}`
    });
    faceMesh.setOptions({
      maxNumFaces: 1,
      refineLandmarks: true,
      minDetectionConfidence: 0.5,
      minTrackingConfidence: 0.5
    });
    faceMesh.onResults(onResults);
    
    // Если используется видео – запускаем камеру; если фото, камера не запускается.
    let camera = null;
    if (!usePhoto) {
      camera = new Camera(videoElement, {
        onFrame: async () => { await faceMesh.send({ image: videoElement }); },
        width: videoWidth,
        height: videoHeight
      });
      camera.start();
    }
    
    /**********************************
     * Callback от Face Mesh
     **********************************/
    function onResults(results) {
      if (results.multiFaceLandmarks && results.multiFaceLandmarks.length > 0) {
        currentLandmarks = results.multiFaceLandmarks[0];
        // Маска всегда обновляется, чтобы оставаться на лице
        updatePolygons();
      }
    }
  </script>
</body>
</html>
```

---

## English Version

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Neo Brutalism Face Mask</title>
  <style>
    /* Basic style in the spirit of neo brutalism: harsh lines, high contrast, monospaced font */
    * { box-sizing: border-box; }
    body, html {
      margin: 0;
      padding: 0;
      font-family: "Arial Black", Gadget, sans-serif;
      background: #f5f5f5;
      color: #111;
      overflow: hidden;
    }
    /* Top menu */
    #topMenu {
      position: fixed;
      top: 0;
      left: 0;
      width: 100%;
      background: #000;
      border-bottom: 4px solid #ff0033;
      padding: 10px 0;
      z-index: 30;
      text-align: center;
    }
    #topMenu a {
      color: #ff0033;
      text-decoration: none;
      margin: 0 15px;
      padding: 8px 16px;
      border: 2px solid #ff0033;
      transition: background 0.3s;
    }
    #topMenu a:hover {
      background: #ff0033;
      color: #fff;
    }
    /* Global controls */
    #globalControls {
      position: fixed;
      top: 60px;
      left: 10px;
      background: #000;
      color: #ff0033;
      padding: 10px;
      border: 4px solid #ff0033;
      border-radius: 4px;
      z-index: 30;
      font-size: 14px;
    }
    #globalControls label {
      display: block;
      margin-bottom: 8px;
    }
    #globalControls input {
      width: 80px;
      padding: 2px 4px;
      font-family: inherit;
      font-size: 14px;
      color: #111;
    }
    #globalControls button {
      margin-top: 5px;
      padding: 6px 12px;
      border: 2px solid #ff0033;
      background: #000;
      color: #ff0033;
      cursor: pointer;
      transition: background 0.3s;
    }
    #globalControls button:hover {
      background: #ff0033;
      color: #fff;
    }
    /* Container for video/photo and mask */
    #videoContainer {
      position: relative;
      width: 100vw;
      height: calc(100vh - 60px);
      margin-top: 60px; /* space for the menu */
      background: #000;
    }
    video, #uploadedImage {
      width: 100%;
      height: 100%;
      object-fit: cover;
      display: block;
    }
    /* SVG overlay for the mask */
    #mask {
      position: absolute;
      top: 0;
      left: 0;
      width: 100%;
      height: 100%;
      pointer-events: none;
    }
    /* dat.GUI panel – basic positioning */
    #guiContainer {
      position: fixed;
      top: 10px;
      right: 10px;
      z-index: 25;
    }
  </style>
</head>
<body>
  <!-- Top menu -->
  <div id="topMenu">
    <a href="dev/i.php" target="_blank">Dev i.php</a>
    <a href="dev/i2.php" target="_blank">Dev i2.php</a>
    <a href="dev/i3.php" target="_blank">Dev i3.php</a>
  </div>
  
  <!-- Global controls -->
  <div id="globalControls">
    <label>
      Global Fill Opacity (0–1): 
      <input type="number" id="globalFillOpacity" min="0" max="1" step="0.01" value="0.3">
    </label>
    <label>
      Global Stroke Width: 
      <input type="number" id="globalStrokeWidth" min="1" max="10" step="0.5" value="2">
    </label>
    <label>
      Upload your photo:
      <input type="file" id="photoUpload" accept="image/*">
    </label>
    <button id="applyGlobal">Apply Global Settings</button>
  </div>

  <div id="videoContainer">
    <!-- Video is the default; if the user uploads a photo, it replaces the video -->
    <video id="video" autoplay muted playsinline></video>
    <!-- Element for the uploaded photo (hidden until a photo is chosen) -->
    <img id="uploadedImage" style="display:none;" alt="Uploaded photo">
    <!-- SVG overlay with viewBox 0 0 640 480 -->
    <svg id="mask" viewBox="0 0 640 480" preserveAspectRatio="none">
      <g id="facePolygons"></g>
    </svg>
    <div id="guiContainer"></div>
  </div>

  <!-- Hidden canvas for color sampling -->
  <canvas id="hiddenCanvas" width="640" height="480" style="display:none;"></canvas>

  <!-- Include MediaPipe Face Mesh and utilities -->
  <script src="https://cdn.jsdelivr.net/npm/@mediapipe/face_mesh/face_mesh.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/@mediapipe/drawing_utils/drawing_utils.js"></script>
  <!-- Include dat.GUI -->
  <script src="https://cdnjs.cloudflare.com/ajax/libs/dat-gui/0.7.7/dat.gui.min.js"></script>
  
  <script>
    "use strict";
    
    /**********************************
     * Global variables and settings
     **********************************/
    const videoWidth = 640, videoHeight = 480;
    let usePhoto = false;  // Flag: whether an uploaded photo is used
    let uploadedPhoto = null;
    
    const videoElement = document.getElementById('video');
    const uploadedImageElement = document.getElementById('uploadedImage');
    
    // Mask parameters
    const maskParams = {
      polygonCount: 36,      // Default 36 polygons, maximum 48
      freezePolygons: true   // Freeze mode: compute local shape only once
    };
    
    // Array of individual styles for the polygons
    let polygonStyles = [];
    // Saved local polygon shapes (coordinates in the face coordinate system)
    let frozenPolygonsLocal = null;
    // Face transformation at the time of freezing
    let frozenFaceTransform = null;
    // Latest landmarks from Face Mesh
    let currentLandmarks = null;
    // Current (computed) polygons in pixel coordinates
    let currentPolygons = [];
    
    /**********************************
     * Compute face transformation
     **********************************/
    function computeFaceTransform(landmarks) {
      const lmRight = landmarks[33], lmLeft = landmarks[263];
      const re = [lmRight.x * videoWidth, lmRight.y * videoHeight],
            le = [lmLeft.x * videoWidth, lmLeft.y * videoHeight];
      const dx = le[0] - re[0], dy = le[1] - re[1];
      const angle = Math.atan2(dy, dx) * 180 / Math.PI;
      const scale = Math.hypot(dx, dy) * 2.5;
      let sumX = 0, sumY = 0;
      landmarks.forEach(lm => { sumX += lm.x; sumY += lm.y; });
      const center = [ (sumX/landmarks.length)*videoWidth, (sumY/landmarks.length)*videoHeight ];
      return { center, scale, angle };
    }
    
    function computeLocalCoordinates(p, faceTransform) {
      const vx = p[0] - faceTransform.center[0],
            vy = p[1] - faceTransform.center[1];
      const rad = -faceTransform.angle * Math.PI/180;
      const rx = vx * Math.cos(rad) - vy * Math.sin(rad),
            ry = vx * Math.sin(rad) + vy * Math.cos(rad);
      return [ rx / faceTransform.scale, ry / faceTransform.scale ];
    }
    
    function applyTransform(localPoint, faceTransform) {
      const rad = faceTransform.angle * Math.PI/180;
      const rx = localPoint[0] * faceTransform.scale,
            ry = localPoint[1] * faceTransform.scale;
      const xRot = rx * Math.cos(rad) - ry * Math.sin(rad),
            yRot = rx * Math.sin(rad) + ry * Math.cos(rad);
      return [ faceTransform.center[0] + xRot, faceTransform.center[1] + yRot ];
    }
    
    /**********************************
     * Clustering and convex hull functions
     **********************************/
    function dist2(a, b) {
      const dx = a[0]-b[0], dy = a[1]-b[1];
      return dx*dx+dy*dy;
    }
    
    function kMeans(points, k, iterations = 10) {
      let centroids = [];
      for (let i = 0; i < k; i++) {
        centroids.push(points[Math.floor(Math.random() * points.length)]);
      }
      let clusters = [];
      for (let iter = 0; iter < iterations; iter++) {
        clusters = Array.from({length: k}, () => []);
        for (const p of points) {
          let minD = Infinity, clusterIndex = 0;
          for (let i = 0; i < k; i++) {
            const d = dist2(p, centroids[i]);
            if (d < minD) { minD = d; clusterIndex = i; }
          }
          clusters[clusterIndex].push(p);
        }
        for (let i = 0; i < k; i++) {
          if (clusters[i].length === 0) {
            centroids[i] = points[Math.floor(Math.random() * points.length)];
          } else {
            let sumX = 0, sumY = 0;
            clusters[i].forEach(p => { sumX += p[0]; sumY += p[1]; });
            centroids[i] = [ sumX/clusters[i].length, sumY/clusters[i].length ];
          }
        }
      }
      return clusters;
    }
    
    function convexHull(points) {
      if (points.length <= 3) return points.slice();
      const pts = points.slice().sort((a,b)=> a[0]-b[0] || a[1]-b[1]);
      const lower = [];
      pts.forEach(p=>{
        while(lower.length>=2 && cross(lower[lower.length-2], lower[lower.length-1], p) <= 0){
          lower.pop();
        }
        lower.push(p);
      });
      const upper = [];
      for(let i = pts.length-1; i>=0; i--){
        const p = pts[i];
        while(upper.length>=2 && cross(upper[upper.length-2], upper[upper.length-1], p) <= 0){
          upper.pop();
        }
        upper.push(p);
      }
      lower.pop();
      upper.pop();
      return lower.concat(upper);
      function cross(A,B,C){
        return (B[0]-A[0])*(C[1]-A[1]) - (B[1]-A[1])*(C[0]-A[0]);
      }
    }
    
    /**********************************
     * Update and render the mask
     **********************************/
    // If freeze mode is enabled, the local shape is computed once and then transformed
    function updatePolygons() {
      const group = document.getElementById('facePolygons');
      // If not freezing, reset the saved shape
      if (!maskParams.freezePolygons) { frozenPolygonsLocal = null; }
      if (!currentLandmarks) return;
      
      // Convert landmarks to pixel coordinates
      const points = currentLandmarks.map(lm => [lm.x*videoWidth, lm.y*videoHeight]);
      const k = Math.max(1, Math.floor(maskParams.polygonCount));
      
      let newPolygons;
      if (maskParams.freezePolygons) {
        if (!frozenPolygonsLocal) {
          const clusters = kMeans(points, k, 10);
          const frozenPolys = clusters.map(cluster => {
            return (cluster.length < 3) ? null : convexHull(cluster);
          });
          frozenFaceTransform = computeFaceTransform(currentLandmarks);
          frozenPolygonsLocal = frozenPolys.map(poly => {
            if (!poly) return null;
            return poly.map(p => computeLocalCoordinates(p, frozenFaceTransform));
          });
        }
        const currentFaceTransform = computeFaceTransform(currentLandmarks);
        newPolygons = frozenPolygonsLocal.map(polyLocal => {
          if (!polyLocal) return null;
          return polyLocal.map(ptLocal => applyTransform(ptLocal, currentFaceTransform));
        });
      } else {
        const clusters = kMeans(points, k, 10);
        newPolygons = clusters.map(cluster => {
          return (cluster.length < 3) ? null : convexHull(cluster);
        });
      }
      currentPolygons = newPolygons;
      drawPolygons(newPolygons);
    }
    
    function drawPolygons(newPolygons) {
      const group = document.getElementById('facePolygons');
      group.innerHTML = '';
      if (!newPolygons) return;
      newPolygons.forEach((poly, idx) => {
        if (!poly) return;
        const pointsStr = poly.map(p => p[0].toFixed(0)+','+p[1].toFixed(0)).join(' ');
        const polygon = document.createElementNS("http://www.w3.org/2000/svg", "polygon");
        polygon.setAttribute('points', pointsStr);
        if (!polygonStyles[idx]) {
          polygonStyles[idx] = {
            color: "#" + Math.floor(Math.random()*16777215).toString(16).padStart(6,'0'),
            fillOpacity: 0.3,
            strokeOpacity: 1,
            strokeWidth: 2
          };
          updatePolygonStylesGUI();
        }
        const style = polygonStyles[idx];
        polygon.setAttribute('fill', style.color);
        polygon.setAttribute('fill-opacity', style.fillOpacity);
        polygon.setAttribute('stroke', style.color);
        polygon.setAttribute('stroke-opacity', style.strokeOpacity);
        polygon.setAttribute('stroke-width', style.strokeWidth);
        group.appendChild(polygon);
      });
    }
    
    /**********************************
     * Function to compute the average color inside a polygon
     **********************************/
    function pointInPolygon(x, y, poly) {
      let inside = false;
      for (let i = 0, j = poly.length - 1; i < poly.length; j = i++) {
        const xi = poly[i][0], yi = poly[i][1],
              xj = poly[j][0], yj = poly[j][1];
        const intersect = ((yi > y) !== (yj > y)) &&
                          (x < (xj - xi) * (y - yi) / (yj - yi + 0.00001) + xi);
        if (intersect) inside = !inside;
      }
      return inside;
    }
    
    function averageColorForPolygon(poly, ctx) {
      let minX = Infinity, minY = Infinity, maxX = -Infinity, maxY = -Infinity;
      poly.forEach(([x,y]) => {
        if (x < minX) minX = x;
        if (y < minY) minY = y;
        if (x > maxX) maxX = x;
        if (y > maxY) maxY = y;
      });
      minX = Math.floor(minX); minY = Math.floor(minY);
      maxX = Math.ceil(maxX); maxY = Math.ceil(maxY);
      const width = maxX - minX, height = maxY - minY;
      if (width <= 0 || height <= 0) return "#000000";
      const imageData = ctx.getImageData(minX, minY, width, height);
      const data = imageData.data;
      let rSum = 0, gSum = 0, bSum = 0, count = 0;
      for (let j = 0; j < height; j++) {
        for (let i = 0; i < width; i++) {
          const x = i + minX, y = j + minY;
          if (pointInPolygon(x, y, poly)) {
            const index = (j * width + i) * 4;
            rSum += data[index];
            gSum += data[index+1];
            bSum += data[index+2];
            count++;
          }
        }
      }
      if (count === 0) return "#000000";
      const rAvg = Math.round(rSum/count),
            gAvg = Math.round(gSum/count),
            bAvg = Math.round(bSum/count);
      return "#" + [rAvg, gAvg, bAvg].map(c=>c.toString(16).padStart(2,'0')).join('');
    }
    
    /**********************************
     * Update the dat.GUI panel for polygon styles
     **********************************/
    let polygonGUIFolder = null;
    function updatePolygonStylesGUI() {
      if (polygonGUIFolder) { gui.removeFolder(polygonGUIFolder); }
      polygonGUIFolder = gui.addFolder('Polygon Styles');
      polygonStyles.forEach((style, idx) => {
        const folder = polygonGUIFolder.addFolder('Polygon ' + (idx+1));
        folder.addColor(style, 'color')
          .name('Color')
          .onChange(val => { style.color = val; updatePolygons(); });
        folder.add(style, 'fillOpacity', 0, 1)
          .name('Fill Opacity')
          .onChange(val => { style.fillOpacity = val; updatePolygons(); });
        folder.add(style, 'strokeOpacity', 0, 1)
          .name('Stroke Opacity')
          .onChange(val => { style.strokeOpacity = val; updatePolygons(); });
        folder.add(style, 'strokeWidth', 1, 10)
          .name('Stroke Width')
          .onChange(val => { style.strokeWidth = val; updatePolygons(); });
        folder.open();
      });
      polygonGUIFolder.open();
    }
    
    /**********************************
     * Initialize dat.GUI panel
     **********************************/
    const gui = new dat.GUI({ autoPlace: false });
    document.getElementById('guiContainer').appendChild(gui.domElement);
    gui.add(maskParams, 'polygonCount', 1, 48).step(1)
       .name('Polygon Count')
       .onChange(val => {
         frozenPolygonsLocal = null;
         polygonStyles = [];
         updatePolygons();
         updatePolygonStylesGUI();
       });
    gui.add(maskParams, 'freezePolygons')
       .name('Freeze Polygons')
       .onChange(val => { if (!val) { frozenPolygonsLocal = null; } });
    
    /**********************************
     * Global settings handling
     **********************************/
    document.getElementById('applyGlobal').addEventListener('click', function() {
      const globalFillOpacity = parseFloat(document.getElementById('globalFillOpacity').value);
      const globalStrokeWidth = parseFloat(document.getElementById('globalStrokeWidth').value);
      
      const hiddenCanvas = document.getElementById('hiddenCanvas');
      const ctx = hiddenCanvas.getContext('2d');
      // Draw the current source (video or photo) onto the canvas
      if (usePhoto && uploadedPhoto) {
        ctx.drawImage(uploadedPhoto, 0, 0, videoWidth, videoHeight);
      } else {
        ctx.drawImage(videoElement, 0, 0, videoWidth, videoHeight);
      }
      
      currentPolygons.forEach((poly, idx) => {
        if (!poly) return;
        const avgColor = averageColorForPolygon(poly, ctx);
        polygonStyles[idx].color = avgColor;
        polygonStyles[idx].fillOpacity = globalFillOpacity;
        polygonStyles[idx].strokeWidth = globalStrokeWidth;
        polygonStyles[idx].strokeOpacity = 1;
      });
      drawPolygons(currentPolygons);
      updatePolygonStylesGUI();
    });
    
    /**********************************
     * Photo upload handling
     **********************************/
    document.getElementById('photoUpload').addEventListener('change', function(e) {
      const file = e.target.files[0];
      if (!file) return;
      const reader = new FileReader();
      reader.onload = function(event) {
        // Create an image element
        uploadedPhoto = new Image();
        uploadedPhoto.onload = function() {
          usePhoto = true;
          // Hide the video and show the photo
          videoElement.style.display = "none";
          uploadedImageElement.src = uploadedPhoto.src;
          uploadedImageElement.style.display = "block";
          // If the camera is running, stop it (if possible)
          if (camera && typeof camera.stop === "function") {
            camera.stop();
          }
          // Run Face Mesh once for the photo (since it's static)
          faceMesh.send({ image: uploadedPhoto });
        };
        uploadedPhoto.src = event.target.result;
      };
      reader.readAsDataURL(file);
    });
    
    /**********************************
     * Initialize MediaPipe Face Mesh
     **********************************/
    const faceMesh = new FaceMesh({
      locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/face_mesh/${file}`
    });
    faceMesh.setOptions({
      maxNumFaces: 1,
      refineLandmarks: true,
      minDetectionConfidence: 0.5,
      minTrackingConfidence: 0.5
    });
    faceMesh.onResults(onResults);
    
    // If using video – start the camera; if a photo is used, the camera is not started.
    let camera = null;
    if (!usePhoto) {
      camera = new Camera(videoElement, {
        onFrame: async () => { await faceMesh.send({ image: videoElement }); },
        width: videoWidth,
        height: videoHeight
      });
      camera.start();
    }
    
    /**********************************
     * Face Mesh callback
     **********************************/
    function onResults(results) {
      if (results.multiFaceLandmarks && results.multiFaceLandmarks.length > 0) {
        currentLandmarks = results.multiFaceLandmarks[0];
        // The mask is continuously updated to stick to the face
        updatePolygons();
      }
    }
  </script>
</body>
</html>
```
