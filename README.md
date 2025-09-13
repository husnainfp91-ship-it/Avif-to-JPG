<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>AVIF → JPG/PNG — Offline Converter</title>
  <style>
    :root{font-family:system-ui,-apple-system,Segoe UI,Roboto,'Helvetica Neue',Arial;--muted:#6b7280;--accent:#4f46e5}
    body{margin:0;background:#f8fafc;color:#0f172a;display:flex;min-height:100vh;align-items:center;justify-content:center;padding:24px}
    .card{width:100%;max-width:920px;background:#fff;border-radius:12px;box-shadow:0 6px 30px rgba(2,6,23,0.08);padding:20px}
    h1{margin:0 0 6px;font-size:20px}
    p.lead{margin:0 0 18px;color:var(--muted);font-size:13px}
    .grid{display:grid;grid-template-columns:1fr 1fr;gap:16px}
    .controls label{display:block;font-size:13px;margin-bottom:6px}
    input[type=file]{display:block}
    .drop{border:2px dashed #e6eef8;border-radius:10px;padding:18px;text-align:center;color:var(--muted);cursor:pointer}
    .preview{background:#f1f5f9;border-radius:8px;padding:8px;min-height:220px;display:flex;align-items:center;justify-content:center}
    .meta{margin-top:12px;font-size:13px;color:var(--muted)}
    .btn{display:inline-block;padding:8px 12px;border-radius:999px;background:var(--accent);color:#fff;text-decoration:none}
    .small{font-size:12px;color:var(--muted)}
    .row{display:flex;gap:8px;align-items:center}
    input[type=range]{width:100%}
    input[type=number]{width:100%}
    @media (max-width:760px){.grid{grid-template-columns:1fr} }
  </style>
</head>
<body>
  <main class="card">
    <header>
      <h1>AVIF / WebP → JPG or PNG (client-side)</h1>
      <p class="lead">All work happens in your browser — no uploads. Modern browsers can decode AVIF/WebP; if conversion fails try Chrome/Edge/Firefox latest.</p>
    </header>

    <section class="grid">
      <div class="controls">
        <label>Pick file (or drop below)</label>
        <input id="fileInput" type="file" accept="image/*" />

        <div id="drop" class="drop" tabindex="0">Drag & drop an image here</div>

        <div style="margin-top:12px">
          <label>Output format</label>
          <select id="formatSelect">
            <option value="image/jpeg">JPEG (.jpg)</option>
            <option value="image/png">PNG (.png)</option>
          </select>
        </div>

        <div style="margin-top:10px">
          <label>Quality (for JPEG)</label>
          <div class="row"><input id="qualityRange" type="range" min="0.1" max="1" step="0.01" value="0.92" /><div id="qLabel" class="small">92%</div></div>
        </div>

        <div style="margin-top:10px">
          <label>Max width (px) — 0 = original</label>
          <input id="maxWidth" type="number" min="0" value="0" />
        </div>

        <div style="margin-top:10px">
          <label>Max height (px) — 0 = original</label>
          <input id="maxHeight" type="number" min="0" value="0" />
        </div>

        <div style="margin-top:12px">
          <div class="meta">Status: <span id="status">idle</span></div>
          <div style="margin-top:8px" id="downloadWrap"></div>
        </div>
      </div>

      <div>
        <label>Preview</label>
        <div class="preview" id="previewArea">
          <div id="noPreview" class="small">No image yet — choose or drop a file.</div>
          <img id="previewImg" alt="preview" style="max-width:100%;max-height:420px;display:none;object-fit:contain;border-radius:6px" />
        </div>

        <div class="meta" style="margin-top:10px">Note: If your browser doesn't support AVIF decoding client-side, conversion will fail. Try a recent Chrome/Edge/Firefox or use a server-side tool.</div>
      </div>
    </section>

    <!-- hidden canvas used for conversion -->
    <canvas id="workerCanvas" style="display:none"></canvas>

    <footer style="margin-top:14px;text-align:right" class="small">Built client-side — no files leave your device.</footer>
  </main>

<script>
  const fileInput = document.getElementById('fileInput');
  const drop = document.getElementById('drop');
  const previewImg = document.getElementById('previewImg');
  const noPreview = document.getElementById('noPreview');
  const statusEl = document.getElementById('status');
  const downloadWrap = document.getElementById('downloadWrap');
  const formatSelect = document.getElementById('formatSelect');
  const qualityRange = document.getElementById('qualityRange');
  const qLabel = document.getElementById('qLabel');
  const maxWidthInput = document.getElementById('maxWidth');
  const maxHeightInput = document.getElementById('maxHeight');
  const canvas = document.getElementById('workerCanvas');

  function setStatus(s){ statusEl.textContent = s; }

  qualityRange.addEventListener('input', ()=>{ qLabel.textContent = Math.round(Number(qualityRange.value)*100) + '%'; });

  fileInput.addEventListener('change', e=>{ const f = e.target.files && e.target.files[0]; if(f) handleFile(f); });

  ['dragenter','dragover'].forEach(ev=> drop.addEventListener(ev, e=>{ e.preventDefault(); drop.style.background='#eef2ff'; }));
  ['dragleave','drop'].forEach(ev=> drop.addEventListener(ev, e=>{ e.preventDefault(); drop.style.background=''; }));
  drop.addEventListener('drop', e=>{ const f = e.dataTransfer.files && e.dataTransfer.files[0]; if(f) handleFile(f); });
  drop.addEventListener('click', ()=> fileInput.click());

  async function handleFile(file){
    setStatus('Preparing...');
    downloadWrap.innerHTML='';

    const objectUrl = URL.createObjectURL(file);
    previewImg.src = objectUrl;
    previewImg.style.display = '';
    noPreview.style.display = 'none';

    try{
      const outMime = formatSelect.value;
      const quality = Number(qualityRange.value);
      const maxW = Math.max(0, Number(maxWidthInput.value)||0);
      const maxH = Math.max(0, Number(maxHeightInput.value)||0);

      setStatus('Decoding image...');
      let imgBitmap;
      try{
        imgBitmap = await createImageBitmap(file);
      }catch(err){
        // Some browsers may not decode AVIF; provide a helpful error
        setStatus('Error: Cannot decode this image in your browser. Try a modern browser (Chrome/Edge/Firefox) or use a server tool.');
        console.error(err);
        return;
      }

      // calculate target size while preserving aspect
      let tw = imgBitmap.width, th = imgBitmap.height;
      if(maxW>0 && tw>maxW){ const r = maxW/tw; tw = Math.round(tw*r); th = Math.round(th*r); }
      if(maxH>0 && th>maxH){ const r = maxH/th; tw = Math.round(tw*r); th = Math.round(th*r); }

      canvas.width = tw; canvas.height = th;
      const ctx = canvas.getContext('2d');
      ctx.clearRect(0,0,tw,th);
      ctx.drawImage(imgBitmap,0,0,tw,th);

      setStatus('Encoding...');
      const blob = await new Promise((res)=> canvas.toBlob(res, outMime, quality));
      if(!blob){ setStatus('Error: conversion produced no data (format may not be supported).'); return; }

      const outUrl = URL.createObjectURL(blob);
      const suggestedName = (file.name || 'image').replace(/\.[^/.]+$/, '') + (outMime==='image/png'?'.png':'.jpg');

      const a = document.createElement('a');
      a.href = outUrl; a.download = suggestedName; a.textContent = 'Download ' + suggestedName; a.className='btn';
      downloadWrap.appendChild(a);
      setStatus('Done — ready to download.');

    }catch(e){
      console.error(e);
      setStatus('Error: ' + (e && e.message ? e.message : 'unknown'));
    }
  }
</script>
</body>
</html>
