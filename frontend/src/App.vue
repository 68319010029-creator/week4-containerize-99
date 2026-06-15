<script setup>
import { ref, computed, onMounted } from 'vue'

// ── State ────────────────────────────────────────────────────────
const products      = ref([])       // รายการสินค้าทั้งหมดจาก API
const loading       = ref(true)     // แสดง loading spinner
const search        = ref('')       // ข้อความค้นหา
const catFilter     = ref('')       // category ที่กำลัง filter
const showModal     = ref(false)    // แสดง/ซ่อน modal เพิ่ม/แก้ไข
const editingId     = ref(null)     // null = กำลัง add, มีค่า = กำลัง edit
const confirmDelete = ref(null)     // เก็บ product ที่จะลบ (ใช้ confirm dialog)

// ข้อมูลในฟอร์ม modal
const form = ref({ name: '', category: '', price: '', stock: 0, description: '' })

let debounceTimer = null            // สำหรับ debounce search input
// ── Computed ─────────────────────────────────────────────────────

// กรองสินค้าตาม search + catFilter พร้อมกัน
const filtered = computed(() => {
  const s = search.value.toLowerCase()
  return products.value.filter(p => {
    // ตรงกับ search text? (เช็คทั้ง name และ description)
    const matchS = !s ||
      p.name.toLowerCase().includes(s) ||
      (p.description || '').toLowerCase().includes(s)
    // ตรงกับ category filter?
    const matchC = !catFilter.value || p.category === catFilter.value
    return matchS && matchC
  })
})

// สร้าง list ของ categories จาก products ที่มีอยู่ (unique + sort)
const categories = computed(() =>
  [...new Set(products.value.map(p => p.category))].sort()
)

// คำนวณ 4 ตัวเลข dashboard stats
const stats = computed(() => ({
  total:      products.value.length,
  lowStock:   products.value.filter(p => p.stock < 10).length,
  totalItems: products.value.reduce((s, p) => s + p.stock, 0),
  totalValue: products.value.reduce((s, p) => s + parseFloat(p.price) * p.stock, 0)
}))

// ── Methods ───────────────────────────────────────────────────────

// debounce: รอ 280ms หลัง user หยุดพิมพ์แล้วค่อย fetch
// (ป้องกัน API ถูกเรียกทุก keystroke)
function debounceFetch() {
  clearTimeout(debounceTimer)
  debounceTimer = setTimeout(fetchProducts, 280)
}

// ดึงสินค้าจาก API (ส่ง search + catFilter เป็น query params)
async function fetchProducts() {
  loading.value = true
  const q = new URLSearchParams()
  if (search.value)    q.set('search',   search.value)
  if (catFilter.value) q.set('category', catFilter.value)
  try {
    const res = await fetch(`/api/products?${q}`)
    products.value = await res.json()
  } catch {
    products.value = []
  } finally {
    loading.value = false
  }
}

// เปิด modal แบบ Add (clear form)
function openAdd() {
  editingId.value = null
  form.value = { name: '', category: '', price: '', stock: 0, description: '' }
  showModal.value = true
}

// เปิด modal แบบ Edit (copy ข้อมูลจาก product เข้า form)
function openEdit(p) {
  editingId.value = p.id
  form.value = {
    name: p.name, category: p.category,
    price: p.price, stock: p.stock,
    description: p.description || ''
  }
  showModal.value = true
}

// บันทึก (ถ้า editingId มีค่า = PUT, ถ้าเป็น null = POST)
async function saveProduct() {
  const body = {
    name:        form.value.name,
    category:    form.value.category,
    price:       parseFloat(form.value.price),
    stock:       parseInt(form.value.stock),
    description: form.value.description
  }
  const url    = editingId.value ? `/api/products/${editingId.value}` : '/api/products'
  const method = editingId.value ? 'PUT' : 'POST'
  await fetch(url, { method, headers: { 'Content-Type': 'application/json' }, body: JSON.stringify(body) })
  showModal.value = false
  fetchProducts()   // reload สินค้าใหม่
}

// ลบสินค้า
async function deleteProduct(id) {
  await fetch(`/api/products/${id}`, { method: 'DELETE' })
  confirmDelete.value = null
  fetchProducts()
}

// คืน CSS class ตาม stock level (ใช้กับ stock bar และตัวเลข)
function stockClass(s) {
  if (s <= 0) return 'out'    // หมด
  if (s < 10) return 'low'    // ใกล้หมด
  if (s < 30) return 'mid'    // ปานกลาง
  return 'high'               // เพียงพอ
}

// คืน CSS class ตาม category (ใช้กับ badge สี)
function catClass(cat) {
  const m = {
    Electronics: 'c-elec', Clothing: 'c-cloth',
    Footwear:    'c-foot',  Food:     'c-food',
    Sports:      'c-sport', Books:    'c-book',
    Furniture:   'c-furn',  Bags:     'c-bag',
    Accessories: 'c-acc',   Tools:    'c-tool'
  }
  return m[cat] || 'c-other'
}

// โหลดสินค้าทันทีเมื่อ component mount ครั้งแรก
onMounted(fetchProducts)
</script>

<template>
  <div class="app-shell">

    <!-- HEADER -->
    <header class="app-header">
      <div class="logo">
        <span class="logo-icon">📦</span>
        <div>
          <div class="logo-name">StockPro</div>
          <div class="logo-sub">ระบบจัดการสินค้าคงคลัง</div>
        </div>
      </div>
      <button class="btn-add" @click="openAdd">+ เพิ่มสินค้า</button>
    </header>

    <main class="main">

      <section class="hero-panel">
        <div class="hero-copy">
          <p class="hero-subtitle">ระบบจัดการสต็อกสไตล์ใหม่</p>
          <h1>StockPro Pulse</h1>
          <p class="hero-text">ดูภาพรวมสินค้าทั้งหมดได้ในหน้าเดียว พร้อมแจ้งเตือนสต็อกต่ำและจัดการสินค้าอย่างรวดเร็ว</p>
        </div>
        <div class="hero-actions">
          <div class="hero-value">
            <span>สินค้ารวม</span>
            <strong>{{ stats.total }}</strong>
          </div>
          <div class="hero-value hero-warning">
            <span>สต็อกต่ำ</span>
            <strong>{{ stats.lowStock }}</strong>
          </div>
        </div>
      </section>

      <!-- STATS DASHBOARD -->
      <div class="stats-grid">
        <div class="stat-card">
          <div class="stat-icon si-green">📦</div>
          <div class="stat-body">
            <div class="stat-val" style="color:#059669">{{ stats.total }}</div>
            <div class="stat-label">สินค้าทั้งหมด</div>
          </div>
        </div>
        <div class="stat-card">
          <div class="stat-icon si-red">⚠️</div>
          <div class="stat-body">
            <div class="stat-val" style="color:#dc2626">{{ stats.lowStock }}</div>
            <div class="stat-label">สต็อกใกล้หมด (<10)</div>
          </div>
        </div>
        <div class="stat-card">
          <div class="stat-icon si-amber">📊</div>
          <div class="stat-body">
            <div class="stat-val" style="color:#d97706">{{ stats.totalItems.toLocaleString() }}</div>
            <div class="stat-label">จำนวนสต็อกรวม</div>
          </div>
        </div>
        <div class="stat-card">
          <div class="stat-icon si-blue">💰</div>
          <div class="stat-body">
            <div class="stat-val" style="color:#2563eb;font-size:1.3rem">
              ฿{{ Math.round(stats.totalValue).toLocaleString() }}
            </div>
            <div class="stat-label">มูลค่าสต็อกรวม</div>
          </div>
        </div>
      </div>

      <!-- LOW STOCK ALERT -->
      <div class="alert-low" v-if="stats.lowStock > 0">
        🚨 มีสินค้า <strong>{{ stats.lowStock }} รายการ</strong>
        ที่มีจำนวนสต็อกน้อยกว่า 10 ชิ้น — กรุณาตรวจสอบและเติมสต็อก
      </div>

      <!-- TOOLBAR -->
      <div class="toolbar">
        <input
          v-model="search"
          @input="debounceFetch"
          class="input-search"
          placeholder="🔍 ค้นหาชื่อสินค้า หรือคำอธิบาย..."
        />
        <select v-model="catFilter" @change="fetchProducts" class="input-select">
          <option value="">ทุกหมวดหมู่</option>
          <option v-for="c in categories" :key="c" :value="c">{{ c }}</option>
        </select>
        <span class="result-count" v-if="!loading">
          แสดง {{ filtered.length }} / {{ products.length }} รายการ
        </span>
      </div>
            <!-- LOADING -->
      <div class="state-box" v-if="loading">
        <div class="state-icon">⏳</div>
        <div>กำลังโหลดข้อมูลสินค้า...</div>
      </div>

      <!-- EMPTY -->
      <div class="state-box" v-else-if="filtered.length === 0">
        <div class="state-icon">📭</div>
        <div class="state-title">ไม่พบสินค้า</div>
        <div class="state-desc">
          {{ search || catFilter ? 'ลองเปลี่ยน keyword หรือ filter' : 'กดปุ่ม "+ เพิ่มสินค้า" เพื่อเริ่มต้น' }}
        </div>
      </div>

      <!-- PRODUCT GRID -->
      <div class="product-grid" v-else>
        <div
          v-for="p in filtered"
          :key="p.id"
          class="product-card"
          :class="{ 'card-low': p.stock > 0 && p.stock < 10, 'card-out': p.stock <= 0 }"
        >
          <div class="card-top">
            <span class="cat-badge" :class="catClass(p.category)">{{ p.category }}</span>
            <div class="product-name">{{ p.name }}</div>
            <div class="product-desc" v-if="p.description">{{ p.description }}</div>
            <div class="product-price">
              ฿{{ parseFloat(p.price).toLocaleString('th-TH', { minimumFractionDigits: 2 }) }}
            </div>
            <div class="stock-info">
              <div class="stock-row">
                <span class="stock-label">สต็อก</span>
                <span class="stock-num" :class="stockClass(p.stock)">
                  {{ p.stock.toLocaleString() }} ชิ้น
                  <span v-if="p.stock <= 0"> — หมดแล้ว!</span>
                  <span v-else-if="p.stock < 10"> — ใกล้หมด!</span>
                </span>
              </div>
              <div class="stock-track">
                <div
                  class="stock-fill"
                  :class="stockClass(p.stock)"
                  :style="{ width: Math.min((p.stock / 80) * 100, 100) + '%' }"
                ></div>
              </div>
            </div>
          </div>
          <div class="card-footer">
            <button class="btn-edit" @click="openEdit(p)">✏️ แก้ไข</button>
            <button class="btn-del" @click="confirmDelete = p">🗑️ ลบ</button>
          </div>
        </div>
      </div>

    </main>
        <!-- ADD/EDIT MODAL -->
    <div class="overlay" v-if="showModal" @click.self="showModal = false">
      <div class="modal">
        <div class="modal-title">
          {{ editingId ? '✏️ แก้ไขสินค้า' : '📦 เพิ่มสินค้าใหม่' }}
        </div>
        <form @submit.prevent="saveProduct">
          <div class="form-group">
            <label class="form-label">ชื่อสินค้า *</label>
            <input class="form-input" v-model="form.name" placeholder="เช่น MacBook Air M3" required />
          </div>
          <div class="form-row">
            <div class="form-group">
              <label class="form-label">หมวดหมู่ *</label>
              <input class="form-input" v-model="form.category" list="cat-list" placeholder="เช่น Electronics" required />
              <datalist id="cat-list">
                <option v-for="c in categories" :value="c" :key="c" />
              </datalist>
            </div>
            <div class="form-group">
              <label class="form-label">ราคา (บาท) *</label>
              <input class="form-input" v-model="form.price" type="number" min="0" step="0.01" placeholder="0.00" required />
            </div>
          </div>
          <div class="form-group">
            <label class="form-label">จำนวนสต็อก (ชิ้น) *</label>
            <input class="form-input" v-model="form.stock" type="number" min="0" placeholder="0" required />
          </div>
          <div class="form-group">
            <label class="form-label">คำอธิบายสินค้า</label>
            <textarea class="form-input" v-model="form.description" rows="3" placeholder="รายละเอียดเพิ่มเติม (ถ้ามี)" style="resize:vertical"></textarea>
          </div>
          <div class="modal-footer">
            <button type="button" class="btn-cancel" @click="showModal = false">ยกเลิก</button>
            <button type="submit" class="btn-save">
              {{ editingId ? 'บันทึกการเปลี่ยนแปลง' : 'เพิ่มสินค้า' }}
            </button>
          </div>
        </form>
      </div>
    </div>

    <!-- DELETE CONFIRM -->
    <div class="overlay" v-if="confirmDelete" @click.self="confirmDelete = null">
      <div class="modal confirm">
        <div class="confirm-icon">🗑️</div>
        <div class="confirm-title">ยืนยันการลบสินค้า</div>
        <div class="confirm-desc">
          คุณต้องการลบ <strong>{{ confirmDelete?.name }}</strong> ออกจากระบบหรือไม่?<br>
          <span style="color:#dc2626">การกระทำนี้ไม่สามารถย้อนกลับได้</span>
        </div>
        <div class="confirm-actions">
          <button class="btn-cancel" @click="confirmDelete = null">ยกเลิก</button>
          <button class="btn-danger-confirm" @click="deleteProduct(confirmDelete.id)">ลบเลย</button>
        </div>
      </div>
    </div>

  </div>
</template>

<style scoped>
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

.app-shell {
  min-height: 100vh;
  background: radial-gradient(circle at top left, rgba(59,130,246,0.3), transparent 24%), radial-gradient(circle at bottom right, rgba(79,70,229,0.22), transparent 24%), linear-gradient(180deg, #090b14 0%, #0f172a 32%, #111827 100%);
  color: #e2e8f0;
  font-family: Inter, system-ui, -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
}

.app-header {
  position: sticky; top: 0; z-index: 100;
  background: rgba(15,23,42,0.92); border-bottom: 1px solid rgba(148,163,184,0.16);
  height: 74px; padding: 0 1.6rem;
  display: flex; align-items: center; gap: 1rem;
  box-shadow: 0 20px 40px rgba(0,0,0,0.35);
}
.logo { display: flex; align-items: center; gap: 0.75rem; }
.logo-icon { font-size: 1.7rem; }
.logo-name { font-weight: 800; font-size: 1.2rem; color: #0f766e; line-height: 1; }
.logo-sub { font-size: 0.75rem; color: #475569; letter-spacing: 0.02em; }
.btn-add {
  margin-left: auto;
  background: linear-gradient(135deg, #38bdf8, #6366f1); color: #fff;
  border: none; border-radius: 999px;
  padding: 0.75rem 1.45rem; font-size: 0.95rem; font-weight: 800;
  cursor: pointer; transition: transform 0.2s, box-shadow 0.2s, background 0.2s;
}
.btn-add:hover { transform: translateY(-1px); box-shadow: 0 18px 28px rgba(59,130,246,0.32); background: linear-gradient(135deg, #22d3ee, #4f46e5); }

.main { max-width: 1280px; margin: 0 auto; padding: 2rem 1.5rem 2.5rem; }

.hero-panel {
  display: grid; grid-template-columns: 1.5fr 1fr; gap: 1.25rem;
  background: rgba(15,23,42,0.72); border: 1px solid rgba(148,163,184,0.18);
  backdrop-filter: blur(18px); border-radius: 24px; padding: 1.8rem;
  margin-bottom: 1.5rem; box-shadow: inset 0 0 0 1px rgba(255,255,255,0.04);
}
.hero-copy h1 { font-size: 2.15rem; margin-bottom: 0.75rem; color: #f8fafc; }
.hero-copy .hero-subtitle { text-transform: uppercase; letter-spacing: 0.18em; color: #60a5fa; font-size: 0.8rem; font-weight: 700; margin-bottom: 0.8rem; }
.hero-copy .hero-text { max-width: 54ch; color: #cbd5e1; line-height: 1.8; }
.hero-actions { display: grid; grid-template-columns: 1fr 1fr; gap: 0.9rem; }
.hero-value { background: rgba(255,255,255,0.06); border: 1px solid rgba(148,163,184,0.14); border-radius: 18px; padding: 1rem 1.2rem; display: flex; flex-direction: column; justify-content: center; }
.hero-value span { color: #94a3b8; font-size: 0.85rem; margin-bottom: 0.35rem; }
.hero-value strong { font-size: 2rem; color: #f8fafc; }
.hero-warning { border-color: rgba(239,68,68,0.25); background: rgba(248,113,113,0.08); }

.stats-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(220px, 1fr));
  gap: 1rem; margin-bottom: 1.5rem;
}
.stat-card {
  background: rgba(255,255,255,0.08); border: 1px solid rgba(148,163,184,0.14); border-radius: 18px;
  padding: 1.25rem 1.3rem; display: flex; align-items: center; gap: 1rem;
  transition: transform 0.2s, box-shadow 0.2s, border-color 0.2s;
}
.stat-card:hover { transform: translateY(-2px); box-shadow: 0 26px 45px rgba(15,23,42,0.18); border-color: rgba(96,165,250,0.3); }
.stat-icon {
  width: 54px; height: 54px; border-radius: 16px;
  display: flex; align-items: center; justify-content: center;
  font-size: 1.35rem; flex-shrink: 0;
}
.si-green { background: rgba(16,185,129,0.12); color: #22c55e; }
.si-red { background: rgba(248,113,113,0.14); color: #ef4444; }
.si-amber { background: rgba(251,191,36,0.13); color: #f59e0b; }
.si-blue { background: rgba(59,130,246,0.12); color: #60a5fa; }
.stat-val { font-size: 1.8rem; font-weight: 900; line-height: 1; color: #f8fafc; }
.stat-label { font-size: 0.84rem; color: #94a3b8; margin-top: 0.35rem; }

.alert-low {
  background: rgba(248,113,113,0.12); border: 1px solid rgba(248,113,113,0.3);
  border-radius: 18px; padding: 1rem 1.2rem;
  font-size: 0.96rem; margin-bottom: 1.5rem; color: #fee2e2;
  line-height: 1.5; box-shadow: inset 0 0 0 1px rgba(248,113,113,0.1);
}

.toolbar { display: flex; gap: 0.85rem; margin-bottom: 1.5rem; flex-wrap: wrap; align-items: center; justify-content: space-between; }
.input-search {
  flex: 1; min-width: 240px;
  padding: 0.85rem 1rem; border: 1px solid rgba(148,163,184,0.25); border-radius: 999px;
  background: rgba(255,255,255,0.06); font-size: 0.95rem; color: #e2e8f0; outline: none;
  transition: border-color 0.2s, box-shadow 0.2s, background 0.2s;
}
.input-search::placeholder { color: #94a3b8; }
.input-search:focus { border-color: #60a5fa; background: rgba(255,255,255,0.12); box-shadow: 0 0 0 4px rgba(96,165,250,0.1); }
.input-select {
  padding: 0.85rem 1rem; border: 1px solid rgba(148,163,184,0.25); border-radius: 999px;
  background: rgba(255,255,255,0.06); color: #e2e8f0; font-size: 0.95rem; outline: none; cursor: pointer;
  transition: border-color 0.2s, box-shadow 0.2s, background 0.2s;
}
.input-select:focus { border-color: #60a5fa; background: rgba(255,255,255,0.12); box-shadow: 0 0 0 4px rgba(96,165,250,0.1); }
.result-count { font-size: 0.86rem; color: #94a3b8; white-space: nowrap; }

.state-box { text-align: center; padding: 4rem 1rem; color: #cbd5e1; background: rgba(15,23,42,0.82); border: 1px solid rgba(148,163,184,0.16); border-radius: 22px; }
.state-icon { font-size: 3rem; margin-bottom: 1rem; }
.state-title { font-size: 1.1rem; font-weight: 700; color: #f8fafc; margin-bottom: 0.35rem; }
.state-desc { font-size: 0.95rem; color: #94a3b8; }

.product-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
  gap: 1.25rem;
}
.product-card {
  position: relative;
  background: rgba(15,23,42,0.92); border: 1px solid rgba(96,165,250,0.16); border-radius: 28px;
  overflow: hidden; transition: transform 0.25s ease, box-shadow 0.25s ease, border-color 0.2s ease;
  box-shadow: 0 24px 70px rgba(0,0,0,0.24);
}
.product-card::before {
  content: '';
  position: absolute; inset: 0 0 auto;
  height: 5px; left: 0; right: 0;
  background: linear-gradient(90deg, rgba(56,189,248,0.9), rgba(99,102,241,0.9));
}
.product-card:hover { transform: translateY(-6px); box-shadow: 0 32px 80px rgba(0,0,0,0.28); }
.product-card.card-low { border-color: rgba(248,113,113,0.5); }
.product-card.card-out { border-color: rgba(148,163,184,0.28); opacity: 0.95; }

.card-top { padding: 1.4rem 1.4rem 1rem; }
.cat-badge {
  display: inline-flex; align-items: center; padding: 0.4rem 0.9rem;
  border-radius: 999px; font-size: 0.78rem; font-weight: 700; margin-bottom: 0.85rem;
  letter-spacing: 0.01em; box-shadow: inset 0 0 0 1px rgba(255,255,255,0.08);
}
.c-elec { background: rgba(59,130,246,0.14); color: #bfdbfe; }
.c-cloth { background: rgba(236,72,153,0.15); color: #fbcfe8; }
.c-foot { background: rgba(168,85,247,0.14); color: #ddd6fe; }
.c-food { background: rgba(245,158,11,0.16); color: #fde68a; }
.c-sport { background: rgba(16,185,129,0.12); color: #bbf7d0; }
.c-book { background: rgba(59,130,246,0.13); color: #bfdbfe; }
.c-furn { background: rgba(124,58,237,0.14); color: #ddd6fe; }
.c-bag { background: rgba(249,115,22,0.14); color: #fed7aa; }
.c-acc { background: rgba(16,185,129,0.12); color: #a7f3d0; }
.c-tool { background: rgba(148,163,184,0.14); color: #cbd5e1; }
.c-other { background: rgba(167,139,250,0.14); color: #ddd6fe; }

.product-name { font-size: 1.12rem; font-weight: 900; color: #f8fafc; margin-bottom: 0.4rem; line-height: 1.35; }
.product-desc { font-size: 0.9rem; color: #cbd5e1; margin-bottom: 0.9rem; line-height: 1.7; min-height: 2.8rem; }
.product-price { font-size: 1.35rem; font-weight: 900; color: #38bdf8; }

.stock-info { margin-top: 1rem; }
.stock-row { display: flex; justify-content: space-between; gap: 0.5rem; align-items: center; font-size: 0.85rem; margin-bottom: 0.4rem; }
.stock-label { color: #64748b; font-weight: 700; text-transform: uppercase; letter-spacing: 0.04em; font-size: 0.74rem; }
.stock-num { font-weight: 700; }
.stock-num.out { color: #64748b; }
.stock-num.low { color: #dc2626; }
.stock-num.mid { color: #d97706; }
.stock-num.high { color: #059669; }
.stock-track { height: 8px; background: rgba(148,163,184,0.18); border-radius: 999px; overflow: hidden; }
.stock-fill { height: 100%; border-radius: 999px; transition: width 0.4s ease; min-width: 4px; }
.stock-fill.out { background: rgba(148,163,184,0.65); width: 2% !important; }
.stock-fill.low { background: linear-gradient(90deg, #f87171, #fb7185); }
.stock-fill.mid { background: linear-gradient(90deg, #fbbf24, #f59e0b); }
.stock-fill.high { background: linear-gradient(90deg, #38bdf8, #0ea5e9); }

.card-footer {
  display: flex; gap: 0.8rem;
  padding: 1rem 1.35rem 1.1rem;
  border-top: 1px solid rgba(148,163,184,0.18); background: rgba(15,23,42,0.88);
}
.btn-edit, .btn-del {
  flex: 1; padding: 0.8rem 0.9rem; border-radius: 999px;
  font-size: 0.88rem; font-weight: 800; cursor: pointer; border: none;
  transition: transform 0.2s, background 0.2s, color 0.2s;
}
.btn-edit { background: rgba(56,189,248,0.14); color: #93c5fd; }
.btn-edit:hover { transform: translateY(-1px); background: rgba(56,189,248,0.24); }
.btn-del { background: rgba(248,113,113,0.16); color: #fecaca; }
.btn-del:hover { transform: translateY(-1px); background: rgba(248,113,113,0.28); }

.overlay {
  position: fixed; inset: 0;
  background: rgba(15,23,42,0.85);
  display: flex; align-items: center; justify-content: center;
  z-index: 500; padding: 1rem;
}
.modal {
  background: rgba(15,23,42,0.98); border-radius: 28px;
  width: 100%; max-width: 540px;
  max-height: 90vh; overflow-y: auto; padding: 2rem;
  box-shadow: 0 40px 100px rgba(0,0,0,0.35);
  border: 1px solid rgba(96,165,250,0.18);
}
.modal-title { font-size: 1.45rem; font-weight: 900; margin-bottom: 1.35rem; color: #f8fafc; }

.form-group { margin-bottom: 1.2rem; }
.form-label { display: block; font-size: 0.92rem; font-weight: 700; margin-bottom: 0.45rem; color: #cbd5e1; }
.form-input {
  width: 100%; padding: 0.85rem 1rem;
  border: 1.5px solid rgba(148,163,184,0.24); border-radius: 20px;
  background: rgba(15,23,42,0.82); color: #e2e8f0; font-size: 0.96rem; outline: none;
  transition: border-color 0.2s, box-shadow 0.2s, background 0.2s;
}
.form-input:focus { border-color: #60a5fa; box-shadow: 0 0 0 4px rgba(96,165,250,0.14); background: rgba(255,255,255,0.08); }
.form-row { display: grid; grid-template-columns: repeat(2, minmax(0, 1fr)); gap: 0.9rem; }

.modal-footer {
  display: flex; gap: 0.85rem; justify-content: flex-end;
  padding-top: 1.1rem; border-top: 1px solid rgba(148,163,184,0.18); margin-top: 1rem;
}
.btn-cancel {
  background: rgba(148,163,184,0.14); color: #e2e8f0;
  border: 1px solid rgba(148,163,184,0.28); border-radius: 20px; padding: 0.85rem 1.25rem;
  font-weight: 700; cursor: pointer; transition: background 0.2s, transform 0.2s;
}
.btn-cancel:hover { background: rgba(96,165,250,0.16); transform: translateY(-1px); }
.btn-save {
  background: linear-gradient(135deg, #38bdf8, #2563eb);
  color: #fff; border: none; border-radius: 20px; padding: 0.85rem 1.4rem;
  font-weight: 800; cursor: pointer; transition: background 0.2s, transform 0.2s;
}
.btn-save:hover { background: linear-gradient(135deg, #22d3ee, #1d4ed8); transform: translateY(-1px); }

.confirm { text-align: center; max-width: 400px; }
.confirm-icon { font-size: 3rem; margin-bottom: 1rem; }
.confirm-title { font-size: 1.15rem; font-weight: 800; margin-bottom: 0.6rem; color: #f8fafc; }
.confirm-desc { font-size: 0.96rem; color: #cbd5e1; margin-bottom: 1.4rem; line-height: 1.7; }
.confirm-actions { display: flex; gap: 0.9rem; justify-content: center; flex-wrap: wrap; }
.btn-danger-confirm {
  background: rgba(248,113,113,0.24); color: #fee2e2;
  border: 1px solid rgba(248,113,113,0.3); border-radius: 20px; padding: 0.75rem 1.4rem;
  font-weight: 800; cursor: pointer; transition: background 0.2s, transform 0.2s;
}
.btn-danger-confirm:hover { background: rgba(248,113,113,0.38); transform: translateY(-1px); }

@media (max-width: 880px) {
  .hero-panel { grid-template-columns: 1fr; }
  .toolbar { flex-direction: column; align-items: stretch; }
  .result-count { margin-top: 0.35rem; }
  .stats-grid { grid-template-columns: 1fr; }
}
@media (max-width: 640px) {
  .hero-panel { padding: 1.2rem; }
  .hero-copy h1 { font-size: 1.9rem; }
  .form-row { grid-template-columns: 1fr; }
  .product-grid { grid-template-columns: 1fr; }
  .main { padding: 1.5rem 1rem 2rem; }
  .app-header { padding: 0 1rem; height: auto; min-height: 68px; }
}
</style>