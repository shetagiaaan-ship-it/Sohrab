<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Mobile Repair Manager</title>

  <!-- PWA manifest -->
  <link rel="manifest" href="manifest.json" />
  <!-- Theme / status bar color -->
  <meta name="theme-color" content="#0f172a"/>

  <!-- App icons for devices that read these meta tags -->
  <link rel="icon" href="assets/icon-192.svg" />
  <link rel="apple-touch-icon" href="assets/icon-192.svg" />

  <link rel="stylesheet" href="style.css" />
</head>
<body>
  <div class="app">

    <header class="app-header">
      <div class="brand">
        <img src="assets/icon-192.svg" alt="logo" class="logo" />
        <div>
          <h1>FixIt Mobile</h1>
          <p class="tag">Repair shop management</p>
        </div>
      </div>
      <button id="menuToggle" class="hamburger" aria-label="Open menu">☰</button>
    </header>

    <nav id="mainNav" class="nav">
      <a href="#/" class="nav-link" data-route>Home</a>
      <a href="#/register" class="nav-link" data-route>Register Customer</a>
      <a href="#/tracker" class="nav-link" data-route>Repair Tracker</a>
      <a href="#/history" class="nav-link" data-route>History</a>
      <a href="#/invoice" class="nav-link" data-route>Invoice Preview</a>
      <div class="nav-footer">v1.0 — Offline ready</div>
    </nav>

    <main id="main" class="main-content">

      <!-- HOME -->
      <section id="page-home" class="page active">
        <div class="hero">
          <h2>Welcome to FixIt Mobile</h2>
          <p>Manage customers, track repairs, create invoices — all offline and installable.</p>
          <div class="quick-actions">
            <a href="#/register" class="btn">New Customer</a>
            <a href="#/tracker" class="btn outline">View Repairs</a>
          </div>
        </div>

        <div class="stats">
          <div class="card">
            <h3 id="stat-total">0</h3>
            <p>Total Orders</p>
          </div>
          <div class="card">
            <h3 id="stat-inprogress">0</h3>
            <p>In Progress</p>
          </div>
          <div class="card">
            <h3 id="stat-completed">0</h3>
            <p>Completed</p>
          </div>
        </div>
      </section>

      <!-- REGISTER -->
      <section id="page-register" class="page">
        <h2>Register Customer / New Repair</h2>
        <form id="form-register" class="form">
          <div class="row">
            <label>Name<input type="text" id="cust-name" required /></label>
            <label>Phone<input type="tel" id="cust-phone" required /></label>
          </div>
          <label>Address<textarea id="cust-address"></textarea></label>
          <div class="row">
            <label>Device Model<input type="text" id="device-model" required /></label>
            <label>Problem / Notes<input type="text" id="device-notes" /></label>
          </div>

          <div class="row items-center">
            <label class="switch">
              <input type="checkbox" id="pickupDelivery" />
              <span class="slider"></span>
            </label>
            <label class="muted">Pickup & Delivery</label>
          </div>

          <div class="row">
            <button type="submit" class="btn">Create Order</button>
            <button type="button" id="form-reset" class="btn outline">Reset</button>
          </div>
        </form>
      </section>

      <!-- TRACKER -->
      <section id="page-tracker" class="page">
        <div class="toolbar">
          <div class="search">
            <input id="searchInput" placeholder="Search name or device model..." />
            <button id="clearSearch" title="clear">✕</button>
          </div>

          <div class="status-filter">
            <select id="statusFilter">
              <option value="all">All statuses</option>
              <option value="Received">Received</option>
              <option value="In Progress">In Progress</option>
              <option value="Completed">Completed</option>
              <option value="Delivered">Delivered</option>
            </select>
            <button id="btn-new-sample" class="btn mini">Add sample</button>
          </div>
        </div>

        <div id="ordersList" class="list"></div>
      </section>

      <!-- HISTORY -->
      <section id="page-history" class="page">
        <h2>Repair History</h2>
        <p class="muted">Completed & Delivered orders</p>
        <div id="historyList" class="list"></div>
      </section>

      <!-- INVOICE -->
      <section id="page-invoice" class="page">
        <h2>Invoice Preview</h2>
        <div class="invoice-select">
          <label>Choose order:
            <select id="invoiceSelect">
              <option value="">-- pick an order --</option>
            </select>
          </label>
        </div>

        <div id="invoicePreview" class="invoice card hidden">
          <!-- Invoice content filled by JS -->
        </div>
      </section>

    </main>

    <footer class="app-footer">
      <div>© <span id="year"></span> FixIt Mobile</div>
      <div class="muted">Works offline — try turning off network after first load.</div>
    </footer>

  </div>

  <script src="script.js" defer></script>
  <script>
    // Register service worker
    if ('serviceWorker' in navigator) {
      navigator.serviceWorker.register('service-worker.js')
        .then(reg => console.log('SW registered', reg.scope))
        .catch(err => console.warn('SW reg failed', err));
    }
  </script>
</body>
</html>

{
  "name": "FixIt Mobile - Repair Shop Manager",
  "short_name": "FixIt",
  "description": "Mobile phone repair shop manager (offline-capable PWA).",
  "icons": [
    {
      "src": "assets/icon-192.svg",
      "sizes": "192x192",
      "type": "image/svg+xml",
      "purpose": "any maskable"
    },
    {
      "src": "assets/icon-512.svg",
      "sizes": "512x512",
      "type": "image/svg+xml",
      "purpose": "any maskable"
    }
  ],
  "start_url": "./index.html",
  "scope": "./",
  "display": "standalone",
  "background_color": "#0f172a",
  "theme_color": "#0ea5a4",
  "orientation": "portrait-primary"
}

/* script.js
   - Simple SPA routing (hash)
   - IndexedDB wrapper for storing orders
   - Render functions for pages
   - Search, filtering, sample data, invoice preview
*/

/* ---------------------------
   IndexedDB helper (small)
   --------------------------- */
const DB_NAME = 'fixit-db';
const DB_VERSION = 1;
const STORE_NAME = 'orders';

function openDB(){
  return new Promise((resolve, reject) => {
    const req = indexedDB.open(DB_NAME, DB_VERSION);
    req.onupgradeneeded = e => {
      const db = e.target.result;
      if (!db.objectStoreNames.contains(STORE_NAME)) {
        const store = db.createObjectStore(STORE_NAME, { keyPath: 'id' });
        store.createIndex('customer', 'customer.name', { unique: false });
        store.createIndex('device', 'device.model', { unique: false });
        store.createIndex('status', 'status', { unique: false });
      }
    };
    req.onsuccess = e => resolve(e.target.result);
    req.onerror = e => reject(e.target.error);
  });
}

async function idbAdd(order){
  const db = await openDB();
  return new Promise((resolve, reject) => {
    const tx = db.transaction(STORE_NAME, 'readwrite');
    const store = tx.objectStore(STORE_NAME);
    const req = store.add(order);
    req.onsuccess = () => resolve(order);
    req.onerror = e => reject(e.target.error);
  });
}

async function idbPut(order){
  const db = await openDB();
  return new Promise((resolve, reject) => {
    const tx = db.transaction(STORE_NAME, 'readwrite');
    const store = tx.objectStore(STORE_NAME);
    const req = store.put(order);
    req.onsuccess = () => resolve(order);
    req.onerror = e => reject(e.target.error);
  });
}

async function idbGetAll(){
  const db = await openDB();
  return new Promise((resolve, reject) => {
    const tx = db.transaction(STORE_NAME, 'readonly');
    const store = tx.objectStore(STORE_NAME);
    const req = store.getAll();
    req.onsuccess = () => resolve(req.result);
    req.onerror = e => reject(e.target.error);
  });
}

async function idbGetById(id){
  const db = await openDB();
  return new Promise((resolve, reject) => {
    const tx = db.transaction(STORE_NAME, 'readonly');
    const store = tx.objectStore(STORE_NAME);
    const req = store.get(id);
    req.onsuccess = () => resolve(req.result);
    req.onerror = e => reject(e.target.error);
  });
}

async function idbDelete(id){
  const db = await openDB();
  return new Promise((resolve, reject) => {
    const tx = db.transaction(STORE_NAME, 'readwrite');
    const store = tx.objectStore(STORE_NAME);
    const req = store.delete(id);
    req.onsuccess = () => resolve();
    req.onerror = e => reject(e.target.error);
  });
}

/* ---------------------------
   Utilities
   --------------------------- */
function uid(){
  // timestamp + random 4-digit -> unique enough for local usage
  return 'R' + Date.now().toString(36) + Math.floor(Math.random()*9000 + 1000);
}

function qs(sel, root=document){ return root.querySelector(sel) }
function qsa(sel, root=document){ return [...root.querySelectorAll(sel)] }

/* ---------------------------
   App state and navigation
   --------------------------- */
const routes = {
  '/': 'page-home',
  '/register': 'page-register',
  '/tracker': 'page-tracker',
  '/history': 'page-history',
  '/invoice': 'page-invoice'
};

function navigate(){
  const hash = location.hash.replace('#','') || '/';
  const route = routes[hash] ? hash : '/';
  // set active page
  qsa('.page').forEach(p => p.classList.remove('active'));
  qs('#' + routes[route]).classList.add('active');

  // highlight nav
  qsa('.nav-link').forEach(a => a.classList.remove('active'));
  qsa(`.nav-link[href="#${route}"]`).forEach(a => a.classList.add('active'));

  // do route-specific actions
  if (route === '/') updateStats();
  if (route === '/tracker') renderOrders();
  if (route === '/history') renderHistory();
  if (route === '/invoice') populateInvoiceSelect();
}

/* ---------------------------
   DOM bindings & handlers
   --------------------------- */
document.addEventListener('DOMContentLoaded', () => {
  // Setup UI details
  qs('#year').textContent = new Date().getFullYear();

  // mobile nav toggle
  qs('#menuToggle').addEventListener('click', () => {
    qs('#mainNav').classList.toggle('open');
  });
  // close nav when clicking link on mobile
  qsa('.nav-link').forEach(a => a.addEventListener('click', () => qs('#mainNav').classList.remove('open')));

  // Routing
  window.addEventListener('hashchange', navigate);
  navigate(); // initial

  // Form: register
  qs('#form-register').addEventListener('submit', async (e) => {
    e.preventDefault();
    const order = {
      id: uid(),
      createdAt: new Date().toISOString(),
      customer: {
        name: qs('#cust-name').value.trim(),
        phone: qs('#cust-phone').value.trim(),
        address: qs('#cust-address').value.trim()
      },
      device: {
        model: qs('#device-model').value.trim(),
        notes: qs('#device-notes').value.trim()
      },
      pickup: qs('#pickupDelivery').checked,
      status: 'Received', // default status
      history: [
        { status: 'Received', at: new Date().toISOString() }
      ],
      price: 0 // can be edited later (simple)
    };

    // add to IDB
    try {
      await idbAdd(order);
      alert('Order created: ' + order.id);
      qs('#form-register').reset();
      // after create, navigate to tracker to see it
      location.hash = '#/tracker';
      renderOrders();
    } catch (err) {
      console.error(err);
      alert('Failed to create order. See console.');
    }
  });

  // reset
  qs('#form-reset').addEventListener('click', () => qs('#form-register').reset());

  // search
  qs('#searchInput').addEventListener('input', () => renderOrders());
  qs('#clearSearch').addEventListener('click', () => { qs('#searchInput').value=''; renderOrders(); });

  // status filter
  qs('#statusFilter').addEventListener('change', () => renderOrders());

  // add sample
  qs('#btn-new-sample').addEventListener('click', async () => {
    const sample = {
      id: uid(),
      createdAt: new Date().toISOString(),
      customer: { name: 'Sample User', phone: '555-1234', address: '123 Sample St.' },
      device: { model: 'Phone X', notes: 'Screen cracked' },
      pickup: Math.random() > 0.5,
      status: ['Received','In Progress','Completed'][Math.floor(Math.random()*3)],
      history: [],
      price: Math.floor(Math.random()*200) + 20
    };
    sample.history.push({status: sample.status, at: new Date().toISOString()});
    await idbAdd(sample);
    renderOrders();
    updateStats();
    alert('Sample order added: ' + sample.id);
  });

  // invoice selection
  qs('#invoiceSelect').addEventListener('change', async (e) => {
    const id = e.target.value;
    if (!id) { qs('#invoicePreview').classList.add('hidden'); return; }
    const order = await idbGetById(id);
    showInvoice(order);
  });

  // Populate lists
  renderOrders();
  populateInvoiceSelect();
  updateStats();
});

/* ---------------------------
   Render functions
   --------------------------- */

async function getAllOrders(){
  const all = await idbGetAll();
  // sort newest first
  return all.sort((a,b) => new Date(b.createdAt) - new Date(a.createdAt));
}

async function renderOrders(){
  const listEl = qs('#ordersList');
  listEl.innerHTML = '';
  const all = await getAllOrders();
  const q = qs('#searchInput').value.trim().toLowerCase();
  const filter = qs('#statusFilter').value;

  const filtered = all.filter(o => {
    if (filter !== 'all' && o.status !== filter) return false;
    if (!q) return true;
    // search in customer name and device model
    return (o.customer.name && o.customer.name.toLowerCase().includes(q)) ||
           (o.device.model && o.device.model.toLowerCase().includes(q));
  });

  if (filtered.length === 0) {
    listEl.innerHTML = `<div class="card muted">No orders found.</div>`;
    return;
  }

  for (const o of filtered) {
    const div = document.createElement('div');
    div.className = 'order-card';

    // header row
    const top = document.createElement('div'); top.className = 'order-top';
    top.innerHTML = `
      <div>
        <strong>${o.customer.name || '—'}</strong>
        <div class="order-meta">${o.device.model || ''} • ${o.customer.phone || ''}</div>
      </div>
      <div class="items">
        <div class="badge ${cssSafe(o.status)}">${o.status}</div>
        <div class="order-meta">#${o.id}</div>
      </div>
    `;
    div.appendChild(top);

    // details
    const details = document.createElement('div');
    details.innerHTML = `
      <p class="muted">${o.device.notes || ''}</p>
      <div class="order-meta">Created: ${new Date(o.createdAt).toLocaleString()}</div>
    `;
    div.appendChild(details);

    // actions
    const actions = document.createElement('div');
    actions.className = 'order-actions';
    // status select
    const sel = document.createElement('select');
    ['Received','In Progress','Completed','Delivered'].forEach(s => {
      const opt = document.createElement('option'); opt.value = s; opt.textContent = s;
      if (s === o.status) opt.selected = true;
      sel.appendChild(opt);
    });
    sel.addEventListener('change', async (e) => {
      o.status = e.target.value;
      o.history = o.history || [];
      o.history.push({ status: o.status, at: new Date().toISOString() });
      await idbPut(o);
      renderOrders();
      updateStats();
    });

    // edit price button
    const btnPrice = document.createElement('button'); btnPrice.className = 'btn mini outline'; btnPrice.textContent='Set Price';
    btnPrice.addEventListener('click', async () => {
      const val = prompt('Set price for order ' + o.id, o.price || '0');
      if (val !== null) {
        o.price = parseFloat(val) || 0;
        await idbPut(o);
        renderOrders();
      }
    });

    // invoice preview quick
    const btnInv = document.createElement('button'); btnInv.className='btn mini'; btnInv.textContent='Invoice';
    btnInv.addEventListener('click', () => {
      location.hash = '#/invoice';
      setTimeout(() => {
        qs('#invoiceSelect').value = o.id;
        qs('#invoiceSelect').dispatchEvent(new Event('change'));
      }, 80);
    });

    // delete
    const btnDel = document.createElement('button'); btnDel.className='btn mini outline'; btnDel.textContent='Delete';
    btnDel.addEventListener('click', async () => {
      if (confirm('Delete order ' + o.id + '?')) {
        await idbDelete(o.id);
        renderOrders();
        updateStats();
      }
    });

    actions.appendChild(sel);
    actions.appendChild(btnPrice);
    actions.appendChild(btnInv);
    actions.appendChild(btnDel);
    div.appendChild(actions);

    listEl.appendChild(div);
  }
}

function cssSafe(str){
  return String(str).replace(/\s/g, '-');
}

async function renderHistory(){
  const el = qs('#historyList');
  el.innerHTML = '';
  const all = await getAllOrders();
  const history = all.filter(o => o.status === 'Completed' || o.status === 'Delivered');
  if (!history.length) { el.innerHTML='<div class="card muted">No history yet.</div>'; return; }
  history.forEach(o => {
    const d = document.createElement('div'); d.className='order-card';
    d.innerHTML = `<div class="order-top"><div><strong>${o.customer.name}</strong><div class="order-meta">${o.device.model}</div></div><div class="order-meta">${new Date(o.createdAt).toLocaleDateString()}</div></div><div class="muted">${o.device.notes || ''}</div>`;
    el.appendChild(d);
  });
}

async function populateInvoiceSelect(){
  const sel = qs('#invoiceSelect');
  sel.innerHTML = `<option value="">-- pick an order --</option>`;
  const all = await getAllOrders();
  all.forEach(o => {
    const opt = document.createElement('option');
    opt.value = o.id;
    opt.textContent = `#${o.id} — ${o.customer.name} — ${o.device.model}`;
    sel.appendChild(opt);
  });
}

function showInvoice(o){
  const inv = qs('#invoicePreview');
  if (!o) { inv.classList.add('hidden'); return; }
  inv.classList.remove('hidden');
  inv.innerHTML = `
    <div style="display:flex;justify-content:space-between;align-items:center">
      <div>
        <h3>FixIt Mobile</h3>
        <div class="muted">Invoice</div>
      </div>
      <div class="text-muted">#${o.id}<br>${new Date(o.createdAt).toLocaleDateString()}</div>
    </div>

    <hr style="margin:10px 0;border:none;border-top:1px solid rgba(255,255,255,0.04)" />

    <div>
      <strong>Customer</strong>
      <div>${o.customer.name} • ${o.customer.phone}</div>
      <div class="muted">${o.customer.address || ''}</div>
    </div>

    <div style="margin-top:10px">
      <strong>Device</strong>
      <div>${o.device.model}</div>
      <div class="muted">${o.device.notes || ''}</div>
    </div>

    <div style="display:flex;justify-content:space-between;margin-top:12px">
      <div class="muted">Pickup & Delivery</div>
      <div>${o.pickup ? 'Yes' : 'No'}</div>
    </div>

    <div style="display:flex;justify-content:space-between;margin-top:12px">
      <div class="muted">Status</div>
      <div>${o.status}</div>
    </div>

    <div style="display:flex;justify-content:space-between;margin-top:18px;font-weight:700">
      <div>Total</div>
      <div>${(o.price || 0).toFixed(2)} USD</div>
    </div>

    <div style="margin-top:18px;display:flex;gap:8px">
      <button id="printInvoice" class="btn">Print</button>
      <button id="closeInvoice" class="btn outline">Close</button>
    </div>
  `;

  qs('#printInvoice').addEventListener('click', () => {
    window.print();
  });
  qs('#closeInvoice').addEventListener('click', () => {
    qs('#invoicePreview').classList.add('hidden');
  });
}

/* ---------------------------
   Dashboard stats
   --------------------------- */
async function updateStats(){
  const all = await getAllOrders();
  qs('#stat-total').textContent = all.length;
  qs('#stat-inprogress').textContent = all.filter(o => o.status === 'In Progress').length;
  qs('#stat-completed').textContent = all.filter(o => o.status === 'Completed' || o.status === 'Delivered').length;
}

// service-worker.js
// Caches core app shell and assets for offline use.
// Basic cache-first strategy with network fallback for navigation requests.

const CACHE_NAME = 'fixit-shell-v1';
const ASSETS = [
  '/',
  '/index.html',
  '/style.css',
  '/script.js',
  '/manifest.json',
  '/assets/icon-192.svg',
  '/assets/icon-512.svg'
];

// Install - cache app shell
self.addEventListener('install', evt => {
  self.skipWaiting();
  evt.waitUntil(
    caches.open(CACHE_NAME)
      .then(cache => cache.addAll(ASSETS))
      .catch(err => console.error('SW cache failed', err))
  );
});

// Activate - cleanup old caches
self.addEventListener('activate', evt => {
  evt.waitUntil(
    caches.keys().then(keys => Promise.all(
      keys.map(key => {
        if (key !== CACHE_NAME) return caches.delete(key);
      })
    ))
  );
  self.clients.claim();
});

// Fetch - respond with cache, fallback to network, fallback to offline page if needed
self.addEventListener('fetch', evt => {
  const req = evt.request;

  // For navigation requests, prefer network but fall back to cache (so latest content when online)
  if (req.mode === 'navigate') {
    evt.respondWith(
      fetch(req)
        .then(res => {
          // update cache in background
          const copy = res.clone();
          caches.open(CACHE_NAME).then(cache => cache.put(req, copy));
          return res;
        })
        .catch(() => caches.match('/index.html'))
    );
    return;
  }

  // For other requests, try cache first, then network
  evt.respondWith(
    caches.match(req).then(match => match || fetch(req).then(res => {
      // optionally cache fetched response
      if (req.method === 'GET' && res && res.status === 200) {
        const copy = res.clone();
        caches.open(CACHE_NAME).then(cache => cache.put(req, copy));
      }
      return res;
    }).catch(() => {
      // final fallback: if image request, return icon
      if (req.destination === 'image') {
        return caches.match('/assets/icon-192.svg');
      }
    }))
  );
});

/* style.css - clean, modern, responsive layout with subtle animations */

/* Basic variables */
:root{
  --bg:#0f172a;
  --card:#0b1220;
  --accent:#0ea5a4;
  --muted:#9aa6b2;
  --glass: rgba(255,255,255,0.03);
  --radius:14px;
  --gap:14px;
  --text:#e6eef6;
}

/* Reset */
*{box-sizing:border-box}
html,body,#main{height:100%}
body{
  font-family: Inter, ui-sans-serif, system-ui, -apple-system, "Segoe UI", Roboto, "Helvetica Neue", Arial;
  margin:0;
  background: linear-gradient(180deg,#071026 0%, #071022 60%), var(--bg);
  color:var(--text);
  -webkit-font-smoothing:antialiased;
  -moz-osx-font-smoothing:grayscale;
  line-height:1.3;
}

/* App layout */
.app{max-width:1100px;margin:0 auto;display:grid;grid-template-columns:260px 1fr;gap:20px;padding:18px}
@media(max-width:900px){ .app{grid-template-columns:1fr;padding:10px} .nav{position:fixed;left:-320px;top:0;bottom:0;z-index:90;transition:left .25s ease} .nav.open{left:0;} .app-header{z-index:100;} }

.app-header{grid-column:1/-1;display:flex;justify-content:space-between;align-items:center;padding:10px;border-radius:12px;background:linear-gradient(180deg, rgba(255,255,255,0.02), transparent);backdrop-filter: blur(6px);}
.brand{display:flex;gap:12px;align-items:center}
.logo{width:46px;height:46px;border-radius:8px;background:var(--glass);padding:6px}
.brand h1{margin:0;font-size:1.1rem}
.brand p{margin:0;font-size:.8rem;color:var(--muted)}

.hamburger{display:none;border:0;background:none;color:var(--text);font-size:20px}
@media(max-width:900px){ .hamburger{display:block} }

/* Nav */
.nav{background:linear-gradient(180deg, rgba(255,255,255,0.02), transparent);padding:18px;border-radius:12px;height:calc(100vh - 64px);overflow:auto}
.nav a{display:block;padding:10px;border-radius:8px;color:var(--text);text-decoration:none;margin-bottom:6px;transition:background .15s}
.nav a:hover, .nav a.active{background:rgba(255,255,255,0.03)}
.nav-footer{margin-top:20px;color:var(--muted);font-size:.85rem}

/* Main content */
.main-content{padding:12px;border-radius:12px;background:transparent;overflow:auto}
.page{display:none;animation:fadeIn .28s ease}
.page.active{display:block}
@keyframes fadeIn{from{opacity:0;transform:translateY(6px)}to{opacity:1;transform:none}}

/* Hero / quick actions */
.hero{padding:18px;border-radius:12px;background:linear-gradient(90deg, rgba(255,255,255,0.02), rgba(255,255,255,0.01));margin-bottom:16px}
.quick-actions{display:flex;gap:10px;margin-top:12px}
.btn{background:var(--accent);color:#012;border:none;padding:10px 14px;border-radius:10px;cursor:pointer;text-decoration:none;display:inline-block;box-shadow:0 6px 18px rgba(12,18,28,0.6)}
.btn.outline{background:transparent;border:1px solid rgba(255,255,255,0.05);color:var(--text)}
.btn.mini{padding:6px 8px;font-size:.9rem}
.muted{color:var(--muted)}

/* Stats */
.stats{display:flex;gap:12px;margin-top:8px}
@media(max-width:700px){ .stats{flex-direction:column} }
.card{flex:1;padding:16px;border-radius:12px;background:var(--card);box-shadow:0 4px 10px rgba(0,0,0,0.4);text-align:center}
.card h3{margin:0;font-size:1.7rem}

/* Forms */
.form{display:block;gap:12px;max-width:860px}
.form label{display:block;margin-bottom:8px}
.form input[type="text"], .form input[type="tel"], .form textarea, .form select{
  display:block;width:100%;padding:10px;border-radius:10px;border:1px solid rgba(255,255,255,0.04);background:transparent;color:var(--text)
}
.form textarea{min-height:76px;resize:vertical}

/* Rows */
.row{display:flex;gap:12px}
.row.items-center{align-items:center}
@media(max-width:700px){ .row{flex-direction:column} }

/* Switch */
.switch{position:relative;display:inline-block;width:48px;height:28px}
.switch input{opacity:0;width:0;height:0}
.slider{position:absolute;cursor:pointer;top:0;left:0;right:0;bottom:0;background:rgba(255,255,255,0.06);border-radius:28px;transition:.2s}
.slider:before{content:"";position:absolute;height:20px;width:20px;left:4px;top:4px;background:white;border-radius:50%;transition:.2s}
.switch input:checked + .slider{background:var(--accent)}
.switch input:checked + .slider:before{transform:translateX(20px)}

/* List of orders */
.list{display:flex;flex-direction:column;gap:10px;margin-top:12px}
.order-card{display:flex;flex-direction:column;padding:12px;border-radius:12px;background:var(--card);transition:transform .12s, box-shadow .12s}
.order-card:hover{transform:translateY(-4px)}
.order-top{display:flex;justify-content:space-between;align-items:center}
.order-meta{font-size:.9rem;color:var(--muted)}
.order-actions{display:flex;gap:8px}
.badge{padding:6px 8px;border-radius:8px;font-weight:600;font-size:.85rem}

/* badges per status */
.badge.Received{background:rgba(255,255,255,0.03)}
.badge["In Progress"], .badge.In\ Progress{background:rgba(255,255,255,0.03)}
.badge['In Progress']{ }
.badge['In Progress']{}

.badge.In\ Progress,.badge["In Progress"], .badge["In Progress"]{ background:linear-gradient(90deg,#ffb34733,#ffcc3330); color: #072018}
.badge.Received{background:linear-gradient(90deg,#9bd3ff33,#c0e6ff2b); color:#001426}
.badge.Completed{background:linear-gradient(90deg,#bde4c333,#dbf6d430); color:#06210a}
.badge.Delivered{background:linear-gradient(90deg,#f3d6ff33,#fde0ff2b); color:#21021e}

/* invoice */
.invoice{padding:16px}
.invoice .row{justify-content:space-between}

/* small helpers */
.hidden{display:none}
.items{display:flex;gap:8px;flex-wrap:wrap}
.controls{display:flex;gap:8px}

/* Footer */
.app-footer{grid-column:1/-1;padding:12px;margin-top:10px;border-radius:12px;background:linear-gradient(180deg, rgba(255,255,255,0.01), transparent);display:flex;justify-content:space-between;align-items:center;color:var(--muted);font-size:.9rem;}

/* tiny helpers */
.text-muted{color:var(--muted);font-size:.9rem}
.center{display:flex;align-items:center;justify-content:center}
