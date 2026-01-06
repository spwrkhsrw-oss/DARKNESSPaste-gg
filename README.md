<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>DarKNessPastie</title>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; font-family: Consolas, monospace; }
    html,body { height: 100%; }
    body { background: #0b0e14; color: #c9d1d9; display:flex; flex-direction:column; min-height:100vh; }

    header {
      background: linear-gradient(90deg,#02040a,#071026);
      padding: 1rem 1.2rem;
      display:flex;
      justify-content:space-between;
      align-items:center;
      border-bottom:1px solid #1f2937;
    }
    header h1 { color:#38bdf8; font-size:1.25rem; letter-spacing:0.6px; }
    header span { color:#64748b; font-size:0.85rem; }

    main { max-width:1100px; margin:1.2rem auto; padding:0 1rem; width:100%; flex:1 0 auto; }

    .toolbar { display:flex; gap:0.6rem; align-items:center; margin-bottom:0.8rem; }
    .toolbar select, .toolbar button, .inline-btn {
      background:#020617; color:#e5e7eb; border:1px solid #1f2937;
      padding:0.5rem 0.8rem; border-radius:8px; cursor:pointer;
      font-family:inherit;
    }
    .toolbar button { background:#38bdf8; color:#020617; font-weight:700; border:none; }
    textarea {
      width:100%; min-height:380px; background:#020617; color:#e5e7eb;
      border:1px solid #1f2937; border-radius:10px; padding:1rem; resize:vertical;
      font-size:0.95rem; line-height:1.6; white-space:pre-wrap; overflow-wrap:break-word;
    }
    .info { margin-top:0.8rem; display:flex; justify-content:space-between; color:#94a3b8; font-size:0.85rem; }
    .panel { margin-top:0.8rem; padding:0.8rem; border-radius:10px; background:linear-gradient(#020617,#060718); border:1px solid #111827; color:#c9d1d9; }
    .links { display:flex; gap:0.5rem; flex-wrap:wrap; align-items:center; margin-top:0.5rem; }

    .small { font-size:0.85rem; color:#94a3b8; }
    .meta { color:#94a3b8; font-size:0.82rem; margin-top:0.4rem; }

    footer { margin-top:1.6rem; padding:1rem; text-align:center; color:#475569; border-top:1px solid #1f2937; font-size:0.85rem; }
    .btn-ghost { background:transparent; border:1px solid #334155; color:#cbd5e1; padding:0.35rem 0.6rem; border-radius:6px; }
    .actions { display:flex; gap:0.5rem; margin-left:auto; }
    .row { display:flex; gap:0.5rem; align-items:center; }
    .pill { background:#0b1220; padding:0.25rem 0.5rem; border-radius:999px; border:1px solid #172031; color:#9fb4c9; font-size:0.8rem; }
    .notice { color:#60a5fa; }
    a.linkish { color:#38bdf8; text-decoration:none; font-weight:600; }
    @media (max-width:720px) {
      header { flex-direction:column; gap:8px; align-items:flex-start;}
      .toolbar { flex-direction:column; align-items:stretch; }
      .actions { margin-left:0; }
    }
  </style>
</head>
<body>

<header>
  <div>
    <h1>DarKNessPastie</h1>
    <div style="font-size:0.8rem;color:#64748b">Dark Paste • Secure • Fast</div>
  </div>
  <div class="row">
    <div class="pill" id="status-pill">Ready</div>
  </div>
</header>

<main id="app">
  <div class="toolbar">
    <select id="lang">
      <option value="text">Plain Text</option>
      <option value="html">HTML</option>
      <option value="javascript">JavaScript</option>
      <option value="python">Python</option>
      <option value="css">CSS</option>
      <option value="json">JSON</option>
      <option value="bash">Bash</option>
    </select>

    <div style="display:flex; gap:0.5rem; align-items:center;">
      <button id="createBtn">Create Paste</button>
      <button id="updateBtn" class="btn-ghost" title="Update existing paste (if viewing one)">Update</button>
      <button id="deleteBtn" class="btn-ghost" title="Delete paste from local storage">Delete</button>
    </div>

    <div class="actions" style="margin-left:auto;">
      <button id="copyBtn" class="btn-ghost">Copy</button>
      <button id="pasteBtn" class="btn-ghost" title="Replace editor with clipboard content">Paste ⌘V</button>
    </div>
  </div>

  <textarea id="paste" placeholder="Paste your code or text here..."></textarea>

  <div class="panel" id="metaPanel" style="display:none;">
    <div><strong id="pasteIdLabel"></strong> <span class="small" id="langLabel"></span></div>
    <div class="links" id="linksArea">
      <!-- Generated links and buttons will appear here -->
    </div>
    <div class="meta" id="createdAtLabel"></div>
  </div>

  <div id="listPanel" class="panel" style="margin-top:0.8rem;">
    <div style="display:flex; justify-content:space-between; align-items:center;">
      <div><strong>Saved local pastes</strong> <span class="small"> (stored in your browser)</span></div>
      <div><button id="clearAll" class="btn-ghost">Clear all</button></div>
    </div>
    <div id="list" style="margin-top:0.6rem; display:flex; gap:0.6rem; flex-wrap:wrap;"></div>
  </div>

  <div class="info">
    <span id="info-status">Ready</span>
    <span>Public • No server required • Raw view: add <code>/raw/&lt;id&gt;</code> to URL</span>
  </div>
</main>

<footer>
  <p>© 2026 DarKNessPastie — Dark themed paste service</p>
</footer>

<script>
/*
  Single-file paste app that:
   - saves pastes to localStorage as JSON at key 'paste_<id>'
   - generates normal view link: <base>#<id>
   - generates raw view link: <base>/raw/<id>
   - supports visiting raw URL and showing raw content
   - copy / paste / load into editor / update / delete
*/

(function () {
  // Utilities
  const $ = (id) => document.getElementById(id);
  function nowISO() { return (new Date()).toISOString(); }

  function generateId() {
    // time-based + random to avoid collisions
    const t = Date.now().toString(36);
    const r = Math.random().toString(36).slice(2,9);
    return (t + r).slice(0, 12);
  }

  function savePasteObj(obj) {
    try {
      localStorage.setItem('paste_' + obj.id, JSON.stringify(obj));
      return true;
    } catch (e) {
      console.error('save error', e);
      alert('Failed to save paste (localStorage may be full or blocked).');
      return false;
    }
  }

  function getPasteObj(id) {
    try {
      const raw = localStorage.getItem('paste_' + id);
      return raw ? JSON.parse(raw) : null;
    } catch (e) {
      return null;
    }
  }

  function removePaste(id) {
    localStorage.removeItem('paste_' + id);
  }

  // build base URLs
  function getBaseNoHashNoSearch() {
    return location.href.split('#')[0].split('?')[0];
  }
  function getBaseForRaw() {
    // ensure path ends with /
    const p = location.pathname;
    const path = p.endsWith('/') ? p : p + '/';
    return location.origin + path;
  }

  // detect id from current location (pathname or hash)
  function detectIdFromLocation() {
    // 1) raw path like /raw/<id> or .../raw/<id>/ => find "raw" segment
    const segs = location.pathname.split('/').filter(Boolean); // remove empty
    const rawIndex = segs.indexOf('raw');
    if (rawIndex !== -1 && segs.length > rawIndex + 1) {
      return { id: segs[rawIndex + 1], raw: true };
    }
    // 2) maybe path ends with id and page was hosted at root/raw not used; fallback: check last segment if not index.html
    // Example: /<id>
    if (segs.length >= 1) {
      const last = segs[segs.length - 1];
      // don't treat common filenames as id
      if (!last.toLowerCase().includes('index.html') && last.length >= 6 && last.length <= 20) {
        // prefer hash first, but if hash absent, allow last segment
        // if it's 'raw' itself, ignore
        if (last !== 'raw') return { id: last, raw: false };
      }
    }
    // 3) hash like #<id>
    const h = (location.hash || '').replace('#','');
    if (h && h.length >= 3) return { id: h, raw: false };
    return null;
  }

  // UI
  const pasteEl = $('paste');
  const createBtn = $('createBtn');
  const updateBtn = $('updateBtn');
  const deleteBtn = $('deleteBtn');
  const copyBtn = $('copyBtn');
  const pasteBtn = $('pasteBtn');
  const langSel = $('lang');
  const metaPanel = $('metaPanel');
  const pasteIdLabel = $('pasteIdLabel');
  const langLabel = $('langLabel');
  const linksArea = $('linksArea');
  const createdAtLabel = $('createdAtLabel');
  const statusPill = $('status-pill');
  const infoStatus = $('info-status');
  const list = $('list');
  const clearAllBtn = $('clearAll');

  function setStatus(text) {
    statusPill.textContent = text;
    infoStatus.textContent = text;
  }

  function refreshSavedList() {
    list.innerHTML = '';
    const keys = Object.keys(localStorage).filter(k => k.startsWith('paste_')).sort().reverse();
    if (!keys.length) {
      list.innerHTML = '<div class="small">No saved pastes in this browser.</div>';
      return;
    }
    for (const k of keys) {
      try {
        const obj = JSON.parse(localStorage.getItem(k));
        const el = document.createElement('button');
        el.className = 'inline-btn';
        el.style.border = '1px solid #111827';
        el.style.background = '#020617';
        el.style.padding = '0.4rem 0.6rem';
        el.style.borderRadius = '8px';
        el.style.cursor = 'pointer';
        el.title = (obj && obj.content && obj.content.slice(0,200)) || '';
        el.textContent = `${obj.id} · ${obj.lang || 'text'} · ${new Date(obj.createdAt).toLocaleString() || ''}`;
        el.onclick = () => {
          // load into editor and set hash
          pasteEl.value = obj.content;
          langSel.value = obj.lang || 'text';
          window.location.hash = obj.id;
          setStatus('Loaded paste: ' + obj.id);
          renderMeta(obj.id);
        };
        list.appendChild(el);
      } catch (e) { console.warn('malformed paste', k); }
    }
  }

  function renderMeta(id) {
    const obj = getPasteObj(id);
    if (!obj) {
      metaPanel.style.display = 'none';
      setStatus('Ready');
      return;
    }
    metaPanel.style.display = 'block';
    pasteIdLabel.textContent = 'Paste: ' + obj.id;
    langLabel.textContent = '• ' + (obj.lang || 'text');
    createdAtLabel.textContent = 'Created: ' + new Date(obj.createdAt).toLocaleString();
    linksArea.innerHTML = '';

    // view link (hash)
    const viewLink = getBaseNoHashNoSearch() + '#' + obj.id;
    const a = document.createElement('a');
    a.href = viewLink;
    a.textContent = 'Open View';
    a.className = 'linkish';
    a.target = '_blank';
    linksArea.appendChild(a);

    // raw link
    const rawBase = getBaseForRaw();
    const rawLink = rawBase + 'raw/' + obj.id;
    const ar = document.createElement('a');
    ar.href = rawLink;
    ar.textContent = 'Open Raw';
    ar.className = 'linkish';
    ar.style.marginLeft = '0.6rem';
    ar.target = '_blank';
    linksArea.appendChild(ar);

    // copy buttons
    const copyLinkBtn = document.createElement('button');
    copyLinkBtn.className = 'btn-ghost';
    copyLinkBtn.textContent = 'Copy Link';
    copyLinkBtn.onclick = async () => {
      try {
        await navigator.clipboard.writeText(viewLink);
        setStatus('Link copied to clipboard');
      } catch (e) {
        prompt('Copy link (fallback):', viewLink);
      }
    };
    linksArea.appendChild(copyLinkBtn);

    const copyRawBtn = document.createElement('button');
    copyRawBtn.className = 'btn-ghost';
    copyRawBtn.textContent = 'Copy Raw Link';
    copyRawBtn.onclick = async () => {
      try {
        await navigator.clipboard.writeText(rawLink);
        setStatus('Raw link copied to clipboard');
      } catch (e) {
        prompt('Copy raw link (fallback):', rawLink);
      }
    };
    linksArea.appendChild(copyRawBtn);

    // load into editor button
    const loadBtn = document.createElement('button');
    loadBtn.className = 'btn-ghost';
    loadBtn.textContent = 'Load into editor';
    loadBtn.onclick = () => {
      pasteEl.value = obj.content;
      langSel.value = obj.lang || 'text';
      setStatus('Loaded paste into editor');
      window.location.hash = obj.id;
    };
    linksArea.appendChild(loadBtn);

    // copy content button
    const copyContentBtn = document.createElement('button');
    copyContentBtn.className = 'btn-ghost';
    copyContentBtn.textContent = 'Copy Content';
    copyContentBtn.onclick = async () => {
      try {
        await navigator.clipboard.writeText(obj.content);
        setStatus('Paste content copied');
      } catch (e) {
        prompt('Copy content (fallback):', obj.content);
      }
    };
    linksArea.appendChild(copyContentBtn);

    // delete button in meta
    const delBtn = document.createElement('button');
    delBtn.className = 'btn-ghost';
    delBtn.textContent = 'Delete';
    delBtn.onclick = () => {
      if (!confirm('Delete paste ' + obj.id + ' from local storage?')) return;
      removePaste(obj.id);
      setStatus('Deleted ' + obj.id);
      metaPanel.style.display = 'none';
      refreshSavedList();
      // if hash pointed to it, clear
      if (location.hash.replace('#','') === obj.id) {
        history.replaceState(null,'', getBaseNoHashNoSearch());
      }
    };
    linksArea.appendChild(delBtn);
  }

  // create new paste
  createBtn.onclick = () => {
    const content = pasteEl.value || '';
    if (!content.trim()) return alert('Paste cannot be empty');
    const id = generateId();
    const obj = { id, content, lang: langSel.value, createdAt: nowISO() };
    if (savePasteObj(obj)) {
      window.location.hash = id;
      setStatus('Saved paste: ' + id);
      renderMeta(id);
      refreshSavedList();
    }
  };

  // update existing paste if viewing one
  updateBtn.onclick = () => {
    const id = (location.hash || '').replace('#','');
    if (!id) return alert('No paste id in URL hash to update. Open a paste first.');
    const existing = getPasteObj(id);
    if (!existing) return alert('Paste not found in localStorage for id: ' + id);
    existing.content = pasteEl.value;
    existing.lang = langSel.value;
    existing.updatedAt = nowISO();
    if (savePasteObj(existing)) {
      setStatus('Updated paste: ' + id);
      renderMeta(id);
      refreshSavedList();
    }
  };

  deleteBtn.onclick = () => {
    const id = (location.hash || '').replace('#','');
    if (!id) return alert('No paste id in URL hash to delete.');
    if (!confirm('Delete paste ' + id + ' from local storage?')) return;
    removePaste(id);
    setStatus('Deleted ' + id);
    metaPanel.style.display = 'none';
    refreshSavedList();
    history.replaceState(null,'', getBaseNoHashNoSearch());
  };

  // copy editor content to clipboard
  copyBtn.onclick = async () => {
    try {
      await navigator.clipboard.writeText(pasteEl.value);
      setStatus('Editor content copied');
    } catch (e) {
      prompt('Copy (fallback):', pasteEl.value);
    }
  };

  // paste from clipboard into editor (requires secure origin)
  pasteBtn.onclick = async () => {
    try {
      const text = await navigator.clipboard.readText();
      pasteEl.value = text;
      setStatus('Pasted from clipboard into editor');
    } catch (e) {
      alert('Unable to read from clipboard. Your browser may block this or you need to use keyboard paste.');
    }
  };

  // clear storage
  clearAllBtn.onclick = () => {
    if (!confirm('Clear ALL pastes saved in this browser? This cannot be undone.')) return;
    const keys = Object.keys(localStorage).filter(k => k.startsWith('paste_'));
    for (const k of keys) localStorage.removeItem(k);
    refreshSavedList();
    setStatus('All pastes cleared');
    metaPanel.style.display = 'none';
    history.replaceState(null,'', getBaseNoHashNoSearch());
  };

  // initial rendering and location handling
  function boot() {
    refreshSavedList();
    const detected = detectIdFromLocation();
    if (detected && detected.id) {
      const obj = getPasteObj(detected.id);
      if (detected.raw) {
        // raw mode: output only plain text
        if (!obj) {
          document.body.innerHTML = '<pre style="color:#f87171;background:#0b0e14;padding:1rem;margin:1rem;border-radius:6px;">Paste not found: ' + detected.id + '</pre>';
          return;
        }
        // Replace document content with raw content (plain text)
        document.documentElement.style.height = '100%';
        document.body.style.margin = '0';
        document.body.style.background = '#030409';
        document.body.innerHTML = ''; // remove app UI
        const pre = document.createElement('pre');
        pre.style.whiteSpace = 'pre-wrap';
        pre.style.wordBreak = 'break-word';
        pre.style.padding = '1rem';
        pre.style.margin = '0';
        pre.style.fontFamily = 'monospace';
        pre.style.fontSize = '14px';
        pre.style.color = '#e6eef6';
        pre.style.background = '#030409';
        pre.textContent = obj.content;
        document.body.appendChild(pre);

        // try to copy to clipboard a convenience button in raw if allowed
        const tools = document.createElement('div');
        tools.style.position = 'fixed';
        tools.style.right = '12px';
        tools.style.top = '12px';
        tools.style.display = 'flex';
        tools.style.gap = '8px';
        const copyBtn = document.createElement('button');
        copyBtn.textContent = 'Copy';
        copyBtn.onclick = async () => {
          try {
            await navigator.clipboard.writeText(obj.content);
            alert('Copied');
          } catch (e) { prompt('Copy:', obj.content); }
        };
        copyBtn.style.padding = '6px 8px';
        copyBtn.style.borderRadius = '6px';
        copyBtn.style.border = 'none';
        copyBtn.style.background = '#0ea5e9';
        copyBtn.style.color = '#042028';
        tools.appendChild(copyBtn);
        document.body.appendChild(tools);

        // set appropriate document title
        document.title = obj.id + ' — raw';
        return;
      } else {
        // normal view: load into editor and show meta
        pasteEl.value = obj ? obj.content : '';
        langSel.value = obj ? (obj.lang || 'text') : 'text';
        window.location.hash = detected.id; // normalize
        setStatus(obj ? ('Loaded paste: ' + obj.id) : 'Paste not found');
        if (obj) renderMeta(obj.id);
      }
    }
  }

  // when hash changes, re-render meta if needed
  window.addEventListener('hashchange', () => {
    const id = (location.hash || '').replace('#','');
    if (id) renderMeta(id);
  });

  // keyboard shortcuts: Ctrl/Cmd+Enter to create, Ctrl/Cmd+S to update
  window.addEventListener('keydown', (ev) => {
    if ((ev.ctrlKey || ev.metaKey) && ev.key === 'Enter') {
      ev.preventDefault();
      createBtn.click();
    } else if ((ev.ctrlKey || ev.metaKey) && (ev.key === 's' || ev.key === 'S')) {
      ev.preventDefault();
      updateBtn.click();
    }
  });

  // initialization
  setStatus('Ready');
  boot();
})();
</script>

</body>
</html>
