
<!DOCTYPE html>
<html lang="de">
<head>
  <meta charset="UTF-8">
  <title>Transkribus Viewer – Global Search</title>
  <style>
    body { margin: 0; font-family: sans-serif; display: flex; height: 100vh; }
    #img-container { width: 70%; height: 100vh; overflow: auto; position: relative; background: #eee; }
    #textlines { width: 30%; overflow-y: scroll; padding: 10px; box-sizing: border-box; }
    .line { padding: 8px; margin-bottom: 4px; border-bottom: 1px solid #ccc; cursor: pointer; }
    .line:hover { background-color: #f0f0f0; }
    .line.active { background-color: #cce5ff; }
    #zoom-img { display: block; width: 100%; height: auto; max-width: none; }
    .overlay-box {
      position: absolute;
      border: 2px solid red;
      background-color: rgba(255, 0, 0, 0.1);
      pointer-events: none;
      z-index: 10;
    }
    #header {
      position: fixed;
      top: 0; left: 0;
      width: 100%;
      background: #ddd;
      padding: 8px;
      display: flex;
      gap: 10px;
      align-items: center;
      z-index: 99;
    }
    select, button {
      padding: 5px 10px;
    }
    body.loaded #img-container, body.loaded #textlines {
      padding-top: 48px;
    }
    #search-results {
      position: absolute;
      right: 10px;
      top: 50px;
      background: #fff;
      border: 1px solid #ccc;
      max-height: 60vh;
      overflow: auto;
      display: none;
      z-index: 1000;
      padding: 10px;
      box-shadow: 0 0 8px rgba(0,0,0,0.1);
    }
    #results-list {
      list-style: none;
      padding-left: 0;
      margin: 0;
    }
    #results-list li {
      margin: 5px 0;
    }
  </style>
</head>
<body>
  <div id="header">
    <button onclick="prevPage()">← Zurück</button>
    <select id="page-select" onchange="loadPage(this.value)"></select>
    <button onclick="nextPage()">Weiter →</button>
    <div style="margin-left:auto; display:flex; align-items:center; gap:10px;">
      <input type="text" id="searchBox" placeholder="🔍 Textsuche..." oninput="globalSearch(this.value)">
      <img src="logo.png" alt="Logo" style="height:40px;">
    </div>
  </div>

  <div id="img-container">
    <img id="zoom-img" src="" />
    <div id="overlay" class="overlay-box" style="display: none;"></div>
  </div>

  <div id="textlines"></div>

  <div id="search-results">
    <b>Trefferliste:</b>
    <ul id="results-list"></ul>
  </div>

  <script>
    let pages = [];
    let currentIndex = 0;
    let globalIndex = [];

    async function fetchPages() {
      const res = await fetch('pages.json');
      pages = await res.json();
      const select = document.getElementById('page-select');
      pages.forEach((p, idx) => {
        const opt = document.createElement('option');
        opt.value = idx;
        opt.textContent = (idx + 1) + '. ' + p.label;
        select.appendChild(opt);
      });
      buildIndex();
      loadPage(0);
      document.body.classList.add('loaded');
    }

    function loadPage(index) {
      currentIndex = parseInt(index);
      const p = pages[currentIndex];
      document.getElementById('zoom-img').src = p.image;
      document.getElementById('overlay').style.display = 'none';

      fetch(p.xml).then(res => res.text()).then(str => {
        const parser = new DOMParser();
        const xml = parser.parseFromString(str, "application/xml");
        const lines = [...xml.getElementsByTagName('*')].filter(n => n.localName === "TextLine");
        const out = lines.map((line, i) => {
          const text = [...line.getElementsByTagName('*')].find(n => n.localName === 'Unicode')?.textContent || '';
          const points = [...line.getElementsByTagName('*')].find(n => n.localName === 'Coords')?.getAttribute('points') || '';
          return `<div class="line" data-points="${points}"><b>${i+1}.</b> ${text}</div>`;
        }).join('');
        document.getElementById('textlines').innerHTML = out;

        document.querySelectorAll('.line').forEach(el => {
          el.addEventListener('click', () => zoomToPolygon(el.dataset.points, el));
        });

        document.getElementById('page-select').value = currentIndex;
      });
    }

    function zoomToPolygon(pointsStr, element) {
      const img = document.getElementById("zoom-img");
      const container = document.getElementById("img-container");
      const overlay = document.getElementById("overlay");
      const scaleX = img.clientWidth / img.naturalWidth;
      const scaleY = img.clientHeight / img.naturalHeight;

      const points = pointsStr.split(" ").map(p => {
        const [x, y] = p.split(",").map(Number);
        return { x: x * scaleX, y: y * scaleY };
      });

      const minX = Math.min(...points.map(p => p.x));
      const minY = Math.min(...points.map(p => p.y));
      const maxX = Math.max(...points.map(p => p.x));
      const maxY = Math.max(...points.map(p => p.y));

      const offsetY = 45;
      overlay.style.left = minX + "px";
      overlay.style.top = (minY + offsetY) + "px";
      overlay.style.width = (maxX - minX) + "px";
      overlay.style.height = (maxY - minY) + "px";
      overlay.style.display = "block";

      container.scrollTo({
        left: minX - 40,
        top: minY - 40,
        behavior: "smooth"
      });

      document.querySelectorAll('.line').forEach(l => l.classList.remove('active'));
      element.classList.add('active');
    }

    function prevPage() {
      if (currentIndex > 0) loadPage(currentIndex - 1);
    }

    function nextPage() {
      if (currentIndex < pages.length - 1) loadPage(currentIndex + 1);
    }

    function buildIndex() {
      pages.forEach((p, pageIndex) => {
        fetch(p.xml).then(r => r.text()).then(str => {
          const parser = new DOMParser();
          const xml = parser.parseFromString(str, "application/xml");
          const lines = [...xml.getElementsByTagName('*')].filter(n => n.localName === "TextLine");
          lines.forEach((line, i) => {
            const text = [...line.getElementsByTagName('*')].find(n => n.localName === 'Unicode')?.textContent || "";
            const points = [...line.getElementsByTagName('*')].find(n => n.localName === 'Coords')?.getAttribute('points') || "";
            globalIndex.push({
              page: pageIndex,
              lineIndex: i,
              text,
              points
            });
          });
        });
      });
    }

    function globalSearch(query) {
      const results = document.getElementById("search-results");
      const list = document.getElementById("results-list");
      const q = query.toLowerCase();
      list.innerHTML = "";

      if (!q || q.length < 2) {
        results.style.display = "none";
        return;
      }

      const hits = globalIndex.filter(item => item.text.toLowerCase().includes(q));
      if (hits.length === 0) {
        list.innerHTML = "<li>Keine Treffer gefunden.</li>";
      } else {
        hits.forEach(hit => {
          const li = document.createElement("li");
          li.innerHTML = `<a href="#" onclick="jumpToHit(${hit.page}, ${hit.lineIndex});return false;">Seite ${hit.page+1}: ${hit.text.replace(new RegExp('('+q+')', 'gi'), '<mark>$1</mark>')}</a>`;
          list.appendChild(li);
        });
      }
      results.style.display = "block";
    }

    function jumpToHit(pageIndex, lineIndex) {
      loadPage(pageIndex);
      setTimeout(() => {
        const line = document.querySelectorAll('.line')[lineIndex];
        if (line) {
          line.scrollIntoView({ behavior: 'smooth', block: 'center' });
          line.classList.add('active');
          zoomToPolygon(line.dataset.points, line);
        }
      }, 500);
    }

    fetchPages();
  </script>
</body>
</html>
