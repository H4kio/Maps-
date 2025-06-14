<!DOCTYPE html>
<html lang="fr">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1" />
<title>Carte Interactive SWAT - AsylumRP</title>
<style>
  body, html {
    height: 100%; margin: 0; font-family: Arial, sans-serif; background: #111;
    color: #eee;
  }
  #map {
    position: relative;
    width: 100%;
    height: 100vh;
    background: url('https://wiki.21jumpclick.fr/images/7/76/Map_gta_V_with_ZIP_code.png') no-repeat center/cover;
  }
  .marker {
    position: absolute;
    width: 28px;
    height: 28px;
    font-size: 24px;
    line-height: 28px;
    text-align: center;
    cursor: pointer;
    user-select: none;
    transform: translate(-50%, -50%);
    transition: transform 0.2s;
  }
  .marker:hover {
    transform: translate(-50%, -50%) scale(1.3);
  }
  /* Modal styles */
  #modal {
    display: none;
    position: fixed;
    top:0; left:0; right:0; bottom:0;
    background: rgba(0,0,0,0.85);
    z-index: 9999;
    justify-content: center;
    align-items: center;
  }
  #modalContent {
    background: #222;
    border-radius: 8px;
    max-width: 800px;
    max-height: 90vh;
    width: 90vw;
    padding: 10px;
    position: relative;
    display: flex;
    flex-direction: column;
    color: #eee;
  }
  #modalHeader {
    font-size: 1.3em;
    margin-bottom: 8px;
  }
  #closeBtn {
    position: absolute;
    right: 10px;
    top: 10px;
    background: transparent;
    border: none;
    color: #eee;
    font-size: 1.5em;
    cursor: pointer;
  }
  #planContainer {
    position: relative;
    flex-grow: 1;
    background: #111;
    border: 1px solid #555;
    overflow: hidden;
    border-radius: 5px;
  }
  #planImg {
    max-width: 100%;
    max-height: 80vh;
    display: block;
  }
  /* Points */
  .point {
    position: absolute;
    width: 26px;
    height: 26px;
    border-radius: 50%;
    color: white;
    text-align: center;
    line-height: 26px;
    font-weight: bold;
    cursor: move;
    user-select: none;
    border: 2px solid white;
    transition: transform 0.2s;
  }
  .point:hover {
    transform: scale(1.3);
    z-index: 10;
  }
  /* Point types colors */
  .entry { background: #2196F3; border-color: #0b61a4; }    /* bleu */
  .exit { background: #f44336; border-color: #a12f28; }     /* rouge */
  .attention { background: #ff9800; border-color: #a66b00; } /* orange */
  .camera { background: #4caf50; border-color: #2d6b22; }    /* vert */
  .door { background: #9c27b0; border-color: #5a146f; }      /* violet */

  /* Controls */
  #controls {
    margin-top: 10px;
    display: flex;
    gap: 10px;
    flex-wrap: wrap;
  }
  #controls button {
    background: #333;
    border: 1px solid #666;
    color: #eee;
    padding: 6px 10px;
    border-radius: 4px;
    cursor: pointer;
    user-select: none;
  }
  #controls button:hover {
    background: #555;
  }

  #jsonOutput {
    margin-top: 10px;
    background: #222;
    color: #ddd;
    max-height: 150px;
    overflow-y: auto;
    font-family: monospace;
    padding: 10px;
    border-radius: 5px;
  }
</style>
</head>
<body>

<div id="map">
  <!-- Exemple marqueurs bâtiments -->
  <div class="marker" style="top: 43.6%; left: 27.6%;" data-name="Hôpital Asylum" data-plan="https://i.imgur.com/fXHb8de.png">🏥</div>
  <div class="marker" style="top: 45.0%; left: 28.8%;" data-name="Gouvernement" data-plan="https://i.imgur.com/2GkfNw9.png">🏛️</div>
  <div class="marker" style="top: 47.0%; left: 30.2%;" data-name="Commissariat Central" data-plan="https://i.imgur.com/9rYJ5qt.png">🚓</div>
</div>

<!-- Modal pour plan + points -->
<div id="modal">
  <div id="modalContent">
    <button id="closeBtn" title="Fermer">×</button>
    <div id="modalHeader"></div>
    <div id="planContainer">
      <img id="planImg" src="" alt="Plan du bâtiment" />
    </div>
    <div id="controls">
      <button data-type="entry" style="background:#2196F3;">🔵 Entrée</button>
      <button data-type="exit" style="background:#f44336;">🔴 Sortie</button>
      <button data-type="attention" style="background:#ff9800;">⚠️ Attention</button>
      <button data-type="camera" style="background:#4caf50;">📷 Caméra</button>
      <button data-type="door" style="background:#9c27b0;">🚪 Porte</button>
      <button id="clearPoints">🗑️ Supprimer tous</button>
      <button id="exportPoints">💾 Exporter JSON</button>
    </div>
    <pre id="jsonOutput"></pre>
  </div>
</div>

<script>
  const map = document.getElementById('map');
  const modal = document.getElementById('modal');
  const modalHeader = document.getElementById('modalHeader');
  const planImg = document.getElementById('planImg');
  const closeBtn = document.getElementById('closeBtn');
  const controls = document.getElementById('controls');
  const jsonOutput = document.getElementById('jsonOutput');
  const planContainer = document.getElementById('planContainer');

  let currentBuilding = null;
  let points = [];

  // Ouvre modal et charge plan
  map.querySelectorAll('.marker').forEach(marker => {
    marker.addEventListener('click', () => {
      currentBuilding = {
        name: marker.dataset.name,
        plan: marker.dataset.plan
      };
      openModal(currentBuilding);
    });
  });

  function openModal(building) {
    modal.style.display = 'flex';
    modalHeader.textContent = building.name;
    planImg.src = building.plan;
    clearAllPoints();
    jsonOutput.textContent = '';
  }

  closeBtn.addEventListener('click', () => {
    modal.style.display = 'none';
    clearAllPoints();
  });

  // Ajouter point au clic sur plan
  planContainer.addEventListener('click', (e) => {
    if (e.target !== planImg) return; // clique sur l'image seulement
    if (!selectedType) {
      alert("Sélectionne un type de point avant de cliquer sur le plan.");
      return;
    }
    const rect = planImg.getBoundingClientRect();
    const xPercent = ((e.clientX - rect.left) / rect.width) * 100;
    const yPercent = ((e.clientY - rect.top) / rect.height) * 100;

    addPoint(xPercent, yPercent, selectedType);
    updateJsonOutput();
  });

  // Ajouter points dynamique et draggable
  function addPoint(x, y, type) {
    const point = document.createElement('div');
    point.classList.add('point', type);
    point.style.left = x + '%';
    point.style.top = y + '%';
    point.title = type.charAt(0).toUpperCase() + type.slice(1);
    point.textContent = {
      entry: 'E',
      exit: 'S',
      attention: '!',
      camera: 'C',
      door: 'D'
    }[type] || '?';

    // Drag & drop
    point.draggable = true;
    point.addEventListener('dragstart', e => {
      e.dataTransfer.setData('text/plain', null);
      point.dataset.dragging = 'true';
    });
    point.addEventListener('dragend', e => {
      const rect = planImg.getBoundingClientRect();
      const leftPercent = ((e.pageX - rect.left) / rect.width) * 100;
      const topPercent = ((e.pageY - rect.top) / rect.height) * 100;
      point.style.left = leftPercent.toFixed(1) + '%';
      point.style.top = topPercent.toFixed(1) + '%';
      delete point.dataset.dragging;
      updateJsonOutput();
    });

    // Suppression au clic droit
    point.addEventListener('contextmenu', e => {
      e.preventDefault();
      planContainer.removeChild(point);
      points = points.filter(p => p.el !== point);
      updateJsonOutput();
    });

    planContainer.appendChild(point);
    points.push({el: point, type, x, y});
  }

  // Gestion sélection type point
  let selectedType = null;
  controls.querySelectorAll('button[data-type]').forEach(btn => {
    btn.addEventListener('click', () => {
      selectedType = btn.dataset.type;
      // Marquer sélection
      controls.querySelectorAll('button[data-type]').forEach(b => b.style.outline = '');
      btn.style.outline = '2px solid #fff';
    });
  });

  // Supprimer tous les points
  document.getElementById('clearPoints').addEventListener('click', () => {
    clearAllPoints();
    updateJsonOutput();
  });

  function clearAllPoints() {
    points.forEach(p => planContainer.removeChild(p.el));
    points = [];
  }

  // Export JSON
  document.getElementById('exportPoints').addEventListener('click', () => {
    if(points.length === 0) {
      alert("Aucun point à exporter.");
      return;
    }
    const exportData = points.map(p => ({
      type: p.type,
      x: parseFloat(p.el.style.left),
      y: parseFloat(p.el.style.top)
    }));
    jsonOutput.textContent = JSON.stringify(exportData, null, 2);
    alert("Positions exportées en JSON affiché en bas.");
  });

  // Mise à jour JSON automatiquement (optionnel)
  function updateJsonOutput() {
    const exportData = points.map(p => ({
      type: p.type,
      x: parseFloat(p.el.style.left),
      y: parseFloat(p.el.style.top)
    }));
    jsonOutput.textContent = JSON.stringify(exportData, null, 2);
  }
</script>

</body>
</html>
