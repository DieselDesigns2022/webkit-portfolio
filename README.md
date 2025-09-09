<!-- File: index.html (Gallery for Customers) -->

<!doctype html>

<html lang="en">
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <title>Portfolio Gallery</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
      .no-select { user-select: none; -webkit-user-select: none; }
    </style>
  </head>
  <body class="min-h-screen bg-zinc-50 text-zinc-900">
    <header class="sticky top-0 z-20 bg-white/90 backdrop-blur border-b">
      <div class="max-w-6xl mx-auto px-4 py-3 flex items-center gap-3">
        <div class="w-8 h-8 rounded-xl bg-black text-white grid place-items-center font-bold">PG</div>
        <h1 class="text-lg font-semibold tracking-tight">Portfolio Gallery</h1>
        <span id="status" class="text-xs text-zinc-500"></span>
        <div class="ml-auto flex items-center gap-2">
          <div class="inline-flex rounded-xl border overflow-hidden">
            <button id="gridBtn" class="px-3 py-2 text-sm bg-zinc-100">Grid</button>
            <button id="flipBtn" class="px-3 py-2 text-sm">Flipbook</button>
          </div>
        </div>
      </div>
    </header>

```
<main class="max-w-6xl mx-auto px-4 py-4">
  <div id="banner" class="hidden mb-3 p-3 rounded-xl border bg-amber-50 text-amber-900 text-sm"></div>

  <!-- Category tabs -->
  <div id="catTabs" class="flex flex-wrap gap-2 mb-4"></div>

  <!-- Grid container (grouped by category when All is selected) -->
  <section id="gridView" class="grid gap-6"></section>

  <!-- Flipbook view -->
  <section id="flipView" class="hidden">
    <div class="relative rounded-2xl border bg-white overflow-hidden">
      <div id="flipViewport" class="w-full aspect-[16/10] md:aspect-[16/9] grid place-items-center bg-zinc-50 no-select"></div>
      <button id="prevBtn" class="absolute left-2 top-1/2 -translate-y-1/2 bg-white/90 hover:bg-white px-3 py-2 rounded-xl border shadow">‹</button>
      <button id="nextBtn" class="absolute right-2 top-1/2 -translate-y-1/2 bg-white/90 hover:bg-white px-3 py-2 rounded-xl border shadow">›</button>
      <div class="absolute left-2 bottom-2 bg-white/90 px-2 py-1 rounded-lg text-xs" id="flipMeta"></div>
    </div>
  </section>
</main>

<footer class="max-w-6xl mx-auto px-4 py-8 text-center text-xs text-zinc-500">
  © <span id="year"></span> Your Name — Built for client viewing. 
</footer>

<script>
  // ---------------- Config (why: point the app to your data) ----------------
  // Put images under /assets/<category>/image.jpg in your repo.
  // Create /manifest.json (see instructions below). The site loads ALL images from it.
  const MANIFEST_URL = './manifest.json';

  // ---------------- State ----------------
  const App = {
    data: null,             // loaded manifest
    view: 'grid',           // 'grid' | 'flip'
    filter: 'all',          // category id or 'all'
    flatList: [],           // flattened items for flipbook
  };

  const el = (sel, root=document) => root.querySelector(sel);
  const els = (sel, root=document) => [...root.querySelectorAll(sel)];
  const status = el('#status');

  // ---------------- Boot ----------------
  (async function init(){
    el('#year').textContent = new Date().getFullYear();
    const params = new URLSearchParams(location.search);
    const v = params.get('view');
    const c = params.get('cat');
    if (v === 'flip') App.view = 'flip';
    if (c) App.filter = c;

    try {
      const res = await fetch(MANIFEST_URL, { cache: 'no-store' });
      if (!res.ok) throw new Error('Manifest not found');
      const manifest = await res.json();
      App.data = normalizeManifest(manifest);
      setBanner('');
    } catch (err) {
      // Fallback demo data to help first-time setup
      App.data = demoManifest();
      setBanner('Using demo data. Create a <code>manifest.json</code> and an <code>assets/</code> folder in your repo to replace this.');
    }

    attachUI();
    renderAll();
  })();

  function attachUI(){
    el('#gridBtn').onclick = ()=>{ App.view='grid'; syncURL(); renderAll(); };
    el('#flipBtn').onclick = ()=>{ App.view='flip'; syncURL(); renderAll(); };
    window.addEventListener('keydown', (e)=>{
      if (App.view!== 'flip') return;
      if (e.key==='ArrowLeft') step(-1);
      if (e.key==='ArrowRight') step(1);
    });
  }

  function setBanner(html){
    const b = el('#banner');
    if (!html) { b.classList.add('hidden'); b.innerHTML=''; return; }
    b.classList.remove('hidden');
    b.innerHTML = html;
  }

  function syncURL(){
    const u = new URL(location);
    u.searchParams.set('view', App.view==='flip'?'flip':'grid');
    if (App.filter && App.filter!=='all') u.searchParams.set('cat', App.filter); else u.searchParams.delete('cat');
    history.replaceState({}, '', u);
  }

  // ---------------- Manifest helpers ----------------
  function normalizeManifest(m){
    // Expected shape:
    // { title?: string, categories: [{ id, name, items: [{ src, title? }] }] }
    const cats = (m.categories||[]).map(cat=>({
      id: cat.id || slugify(cat.name||'category'),
      name: cat.name || 'Category',
      items: (cat.items||[]).map(it=>({ src: it.src, title: it.title || basename(it.src) }))
    }));
    return { title: m.title || 'Portfolio', categories: cats };
  }
  function slugify(s){ return String(s).toLowerCase().replace(/[^a-z0-9]+/g,'-').replace(/(^-|-$)/g,''); }
  function basename(p){ return String(p).split('/').pop().replace(/[-_]+/g,' ').replace(/\.[^.]+$/, ''); }

  function flatten(){
    const out = [];
    for (const cat of App.data.categories) {
      for (const it of cat.items) out.push({ ...it, catId: cat.id, catName: cat.name });
    }
    return out;
  }

  // ---------------- Rendering ----------------
  function renderAll(){
    // Tabs
    renderTabs();
    // Toggle buttons
    el('#gridBtn').classList.toggle('bg-zinc-100', App.view==='grid');
    el('#flipBtn').classList.toggle('bg-zinc-100', App.view==='flip');
    // Views
    if (App.view==='grid') { el('#gridView').classList.remove('hidden'); el('#flipView').classList.add('hidden'); renderGrid(); }
    else { el('#gridView').classList.add('hidden'); el('#flipView').classList.remove('hidden'); renderFlip(); }
  }

  function renderTabs(){
    const ct = el('#catTabs');
    ct.innerHTML = '';
    const mk = (id, label)=>{
      const b = document.createElement('button');
      b.className = 'px-3 py-1.5 text-sm rounded-xl border ' + (App.filter===id? 'bg-zinc-900 text-white border-zinc-900' : 'bg-white hover:bg-zinc-50');
      b.textContent = label; b.onclick = ()=>{ App.filter=id; syncURL(); renderAll(); };
      ct.appendChild(b);
    };
    mk('all','All');
    for (const cat of App.data.categories) mk(cat.id, cat.name);
  }

  function filteredCats(){
    if (App.filter==='all') return App.data.categories;
    const cat = App.data.categories.find(c=>c.id===App.filter);
    return cat ? [cat] : [];
  }

  function renderGrid(){
    const host = el('#gridView');
    host.innerHTML='';
    const cats = filteredCats();
    if (!cats.length) { host.innerHTML = '<div class="text-sm text-zinc-500">No items.</div>'; return; }

    if (App.filter==='all') {
      // Grouped by category
      for (const cat of cats) {
        const section = document.createElement('section');
        section.innerHTML = `<h2 class="text-base font-semibold mb-2">${escapeHtml(cat.name)}</h2>`;
        const grid = document.createElement('div');
        grid.className = 'grid gap-3 sm:grid-cols-2 md:grid-cols-3 lg:grid-cols-4';
        for (const it of cat.items) grid.appendChild(gridCard(it, cat));
        section.appendChild(grid);
        host.appendChild(section);
      }
    } else {
      // Flat grid for the chosen category
      const grid = document.createElement('div');
      grid.className = 'grid gap-3 sm:grid-cols-2 md:grid-cols-3 lg:grid-cols-4';
      for (const it of cats[0].items) grid.appendChild(gridCard(it, cats[0]));
      host.appendChild(grid);
    }
  }

  function gridCard(it, cat){
    const card = document.createElement('figure');
    card.className = 'rounded-2xl overflow-hidden border bg-white hover:shadow-sm transition';
    const img = document.createElement('img');
    img.src = it.src; img.alt = it.title || ''; img.loading = 'lazy'; img.className = 'w-full h-48 object-cover bg-zinc-100';
    const cap = document.createElement('figcaption');
    cap.className = 'p-2 text-xs text-zinc-600 flex items-center justify-between';
    cap.innerHTML = `<span class="truncate" title="${escapeHtml(it.title)}">${escapeHtml(it.title)}</span><span class="text-zinc-400">${escapeHtml(cat.name)}</span>`;
    card.appendChild(img); card.appendChild(cap);
    card.addEventListener('click', ()=>{ App.view='flip'; App.filter = 'all'; syncURL(); renderAll();
      // Jump to this image in flipbook
      App.flatList = flatten();
      const idx = App.flatList.findIndex(x=>x.src===it.src);
      if (idx>=0) currentIndex = idx; showSlide();
    });
    return card;
  }

  // --------------- Flipbook ---------------
  let currentIndex = 0;
  function renderFlip(){
    App.flatList = App.filter==='all' ? flatten() : filteredCats()[0]?.items.map(it=>({ ...it, catName: filteredCats()[0].name, catId: filteredCats()[0].id })) || [];
    if (!App.flatList.length) {
      el('#flipViewport').innerHTML = '<div class="text-sm text-zinc-500">No items.</div>';
      el('#flipMeta').textContent = '';
      return;
    }
    // Clamp index
    if (currentIndex >= App.flatList.length) currentIndex = 0;
    showSlide();
    el('#prevBtn').onclick = ()=>step(-1);
    el('#nextBtn').onclick = ()=>step(1);
    // touch
    const vp = el('#flipViewport');
    let sx=0, sy=0, tracking=false;
    vp.onpointerdown = (e)=>{ tracking=true; sx=e.clientX; sy=e.clientY; };
    vp.onpointerup = (e)=>{ if (!tracking) return; tracking=false; const dx=e.clientX-sx, dy=e.clientY-sy; if (Math.abs(dx)>40 && Math.abs(dx)>Math.abs(dy)) step(dx<0?1:-1); };
  }

  function step(d){ currentIndex = (currentIndex + d + App.flatList.length) % App.flatList.length; showSlide(); }

  function showSlide(){
    const vp = el('#flipViewport');
    const it = App.flatList[currentIndex];
    vp.innerHTML = '';
    const img = document.createElement('img');
    img.src = it.src; img.alt = it.title || ''; img.className = 'max-w-full max-h-full rounded-lg shadow';
    vp.appendChild(img);
    el('#flipMeta').textContent = `${currentIndex+1} / ${App.flatList.length} • ${it.title} • ${it.catName||''}`;
  }

  // ---------------- Utilities ----------------
  function escapeHtml(s){ return String(s).replace(/[&<>"']/g, c => ({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;','\'':'&#39;'}[c])); }

  function demoManifest(){
    return {
      title: 'Demo Portfolio',
      categories: [
        { id: 'web', name: 'Web', items: [
          { src: 'https://images.unsplash.com/photo-1522071820081-009f0129c71c?q=80&w=1200&auto=format&fit=crop', title: 'SaaS Dashboard' },
          { src: 'https://images.unsplash.com/photo-1522199710521-72d69614c702?q=80&w=1200&auto=format&fit=crop', title: 'Landing Page' }
        ]},
        { id: 'mobile', name: 'Mobile', items: [
          { src: 'https://images.unsplash.com/photo-1547658719-da2b51169166?q=80&w=1200&auto=format&fit=crop', title: 'iOS Flow' },
          { src: 'https://images.unsplash.com/photo-1516259762381-22954d7d3ad2?q=80&w=1200&auto=format&fit=crop', title: 'Android Screens' }
        ]}
      ]
    };
  }
</script>
```

  </body>
</html>
