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
  <div class="app-container">

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

      <div class="stats-grid">
        <div class="stat-card">
          <div class="stat-icon si-purple">📦</div>
          <div class="stat-body">
            <div class="stat-val" style="color:#7e22ce">{{ stats.total }}</div>
            <div class="stat-label">สินค้าทั้งหมด</div>
          </div>
        </div>
        <div class="stat-card">
          <div class="stat-icon si-red">⚠️</div>
          <div class="stat-body">
            <div class="stat-val" style="color:#e11d48">{{ stats.lowStock }}</div>
            <div class="stat-label">สต็อกใกล้หมด (<10)</div>
          </div>
        </div>
        <div class="stat-card">
          <div class="stat-icon si-indigo">📊</div>
          <div class="stat-body">
            <div class="stat-val" style="color:#4f46e5">{{ stats.totalItems.toLocaleString() }}</div>
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

      <div class="alert-low" v-if="stats.lowStock > 0">
        🚨 มีสินค้า <strong>{{ stats.lowStock }} รายการ</strong>
        ที่มีจำนวนสต็อกน้อยกว่า 10 ชิ้น — กรุณาตรวจสอบและเติมสต็อก
      </div>

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

      <div class="state-box" v-if="loading">
        <div class="state-icon">⏳</div>
        <div>กำลังโหลดข้อมูลสินค้า...</div>
      </div>

      <div class="state-box" v-else-if="filtered.length === 0">
        <div class="state-icon">📭</div>
        <div class="state-title">ไม่พบสินค้า</div>
        <div class="state-desc">
          {{ search || catFilter ? 'ลองเปลี่ยน keyword หรือ filter' : 'กดปุ่ม "+ เพิ่มสินค้า" เพื่อเริ่มต้น' }}
        </div>
      </div>

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

    <footer class="magical-bottom-bar">
      <span class="magical-text">✨ เลขที่ 046 | ปวส.2/3 | ปกฉัตร ลอยรัตน์ 🔮</span>
    </footer>

    <div class="overlay" v-if="showModal" @click.self="showModal = false">
      <div class="modal">
        <div class="modal-title">
          <span class="modal-title-icon">{{ editingId ? '✏️' : '📦' }}</span> 
          {{ editingId ? 'แก้ไขสินค้า' : 'เพิ่มสินค้าใหม่' }}
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

    <div class="overlay" v-if="confirmDelete" @click.self="confirmDelete = null">
      <div class="modal confirm">
        <div class="confirm-icon">🗑️</div>
        <div class="confirm-title">ยืนยันการลบสินค้า</div>
        <div class="confirm-desc">
          คุณต้องการลบ <strong>{{ confirmDelete?.name }}</strong> ออกจากระบบหรือไม่?<br>
          <span style="color:#e11d48">การกระทำนี้ไม่สามารถย้อนกลับได้</span>
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

.app-container {
  min-height: 100vh;
  position: relative;
  padding-bottom: 60px; /* เพิ่มพื้นที่ให้เว้นจาก footer เวทมนตร์ */
  background-color: #f8faff; 
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
}

.app-header {
  position: sticky; top: 0; z-index: 100;
  background: #ffffff; border-bottom: 1px solid #e0e7ff;
  height: 66px; padding: 0 1.5rem;
  display: flex; align-items: center; gap: .85rem;
  box-shadow: 0 2px 10px rgba(99, 102, 241, 0.08);
}
.logo { display: flex; align-items: center; gap: .6rem; }
.logo-icon { font-size: 1.6rem; }
.logo-name { 
  font-weight: 900; 
  font-size: 1.25rem; 
  background: linear-gradient(135deg, #7c3aed, #2563eb);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  line-height: 1; 
  letter-spacing: -0.5px;
}
.logo-sub  { font-size: .75rem; color: #64748b; margin-top: 2px;}

.btn-add {
  margin-left: auto;
  background: linear-gradient(135deg, #6366f1, #8b5cf6); 
  color: #ffffff;
  border: none; border-radius: 8px;
  padding: .6rem 1.3rem; font-size: .9rem; font-weight: 600;
  cursor: pointer; transition: all .3s ease;
  box-shadow: 0 4px 12px rgba(99, 102, 241, 0.25);
}
.btn-add:hover { 
  background: linear-gradient(135deg, #4f46e5, #7c3aed);
  transform: translateY(-1px);
  box-shadow: 0 6px 16px rgba(99, 102, 241, 0.35);
}

.main { max-width: 1280px; margin: 0 auto; padding: 2rem 1.5rem; }

.stats-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 1.2rem; margin-bottom: 2rem;
}
.stat-card {
  background: #fff; border: 1px solid #e0e7ff; border-radius: 14px;
  padding: 1.25rem; display: flex; align-items: center; gap: 1rem;
  box-shadow: 0 4px 6px rgba(99, 102, 241, 0.04);
}
.stat-icon {
  width: 48px; height: 48px; border-radius: 12px;
  display: flex; align-items: center; justify-content: center;
  font-size: 1.4rem; flex-shrink: 0;
}
.si-purple { background: #f3e8ff; }
.si-red    { background: #ffe4e6; }
.si-indigo { background: #e0e7ff; }
.si-blue   { background: #dbeafe; }

.stat-val { font-size: 1.7rem; font-weight: 800; line-height: 1; }
.stat-label { font-size: .85rem; color: #64748b; margin-top: .3rem; font-weight: 500;}

.alert-low {
  background: #fff1f2; border: 1px solid #fda4af;
  border-radius: 12px; padding: 1rem 1.25rem;
  font-size: .95rem; margin-bottom: 1.5rem; color: #e11d48;
  box-shadow: 0 2px 8px rgba(225, 29, 72, 0.06);
}

.toolbar { display: flex; gap: .75rem; margin-bottom: 1.5rem; flex-wrap: wrap; align-items: center; }
.input-search, .input-select {
  padding: .65rem 1rem; border: 1.5px solid #cbd5e1; border-radius: 10px;
  background: #fff; font-size: .95rem; outline: none; transition: all .2s ease;
  font-family: inherit;
}
.input-search { flex: 1; min-width: 200px; }
.input-search:focus, .input-select:focus { 
  border-color: #6366f1;
  box-shadow: 0 0 0 3px rgba(99, 102, 241, 0.15);
}
.input-select { cursor: pointer; }
.result-count { font-size: .85rem; color: #64748b; white-space: nowrap; font-weight: 500;}

.state-box { text-align: center; padding: 4rem 1rem; color: #64748b; }
.state-icon { font-size: 3rem; margin-bottom: 1rem; opacity: 0.8; }
.state-title { font-size: 1.15rem; font-weight: 700; color: #1e293b; margin-bottom: .4rem; }
.state-desc { font-size: .95rem; }

.product-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
  gap: 1.25rem;
}
.product-card {
  background: #fff; border: 1px solid #e2e8f0; border-radius: 16px;
  overflow: hidden; transition: all .3s cubic-bezier(0.4, 0, 0.2, 1);
}
.product-card:hover { 
  transform: translateY(-4px); 
  box-shadow: 0 12px 24px rgba(99, 102, 241, 0.12); 
  border-color: #c7d2fe;
}
.product-card.card-low  { border-color: #fca5a5; }
.product-card.card-out  { border-color: #cbd5e1; opacity: .75; }

.card-top { padding: 1.25rem; }
.cat-badge {
  display: inline-block; padding: .25rem .75rem;
  border-radius: 20px; font-size: .75rem; font-weight: 700; margin-bottom: .75rem;
}
.c-elec  { background: #dbeafe; color: #1e40af; }
.c-cloth { background: #fce7f3; color: #9d174d; }
.c-foot  { background: #f3e8ff; color: #6b21a8; }
.c-food  { background: #fef3c7; color: #92400e; }
.c-sport { background: #d1fae5; color: #065f46; }
.c-book  { background: #e0f2fe; color: #0369a1; }
.c-furn  { background: #fdf4ff; color: #7e22ce; }
.c-bag   { background: #fff7ed; color: #9a3412; }
.c-acc   { background: #f0fdf4; color: #15803d; }
.c-tool  { background: #f1f5f9; color: #475569; }
.c-other { background: #e0e7ff; color: #3730a3; } 

.product-name  { font-size: 1.05rem; font-weight: 700; color: #1e293b; margin-bottom: .35rem; line-height: 1.4; }
.product-desc  { font-size: .85rem; color: #64748b; line-height: 1.5; margin-bottom: 1rem; }
.product-price { font-size: 1.3rem; font-weight: 800; color: #4f46e5; } 

.stock-info { margin-top: 1rem; }
.stock-row  { display: flex; justify-content: space-between; font-size: .85rem; margin-bottom: .4rem; }
.stock-label { color: #64748b; font-weight: 500;}
.stock-num   { font-weight: 700; }
.stock-num.out  { color: #94a3b8; }
.stock-num.low  { color: #e11d48; }
.stock-num.mid  { color: #d97706; }
.stock-num.high { color: #059669; }
.stock-track { height: 8px; background: #f1f5f9; border-radius: 4px; overflow: hidden; }
.stock-fill  { height: 100%; border-radius: 4px; transition: width .5s ease-in-out; min-width: 4px; }
.stock-fill.out  { background: #cbd5e1; width: 2% !important; }
.stock-fill.low  { background: #e11d48; }
.stock-fill.mid  { background: #f59e0b; }
.stock-fill.high { background: #10b981; }

.card-footer {
  display: flex; gap: .5rem;
  padding: .85rem 1.25rem;
  border-top: 1px solid #f1f5f9; background: #fafcff;
}
.btn-edit, .btn-del {
  flex: 1; padding: .5rem; border-radius: 8px;
  font-size: .85rem; font-weight: 600; cursor: pointer; border: none; transition: all .2s;
}
.btn-edit { background: #e0e7ff; color: #4f46e5; } 
.btn-edit:hover { background: #c7d2fe; }
.btn-del  { background: #ffe4e6; color: #e11d48; }
.btn-del:hover  { background: #fecdd3; }

/* 🔮 CSS สำหรับ Magical Bottom Bar 🔮 */
.magical-bottom-bar {
  position: fixed;
  bottom: 0;
  left: 0;
  width: 100%;
  /* พื้นหลังไล่สีเข้มแนวเวทมนตร์ */
  background: linear-gradient(135deg, #0f172a, #312e81, #581c87, #312e81, #0f172a);
  background-size: 200% 200%;
  animation: magicBG 6s ease infinite; /* แอนิเมชันขยับพื้นหลัง */
  text-align: center;
  padding: 0.9rem;
  font-size: 1rem;
  font-weight: 800;
  letter-spacing: 0.5px;
  z-index: 100;
  box-shadow: 0 -4px 25px rgba(88, 28, 135, 0.4);
  border-top: 1px solid rgba(167, 139, 250, 0.3); /* ขอบเรืองแสงบนนิดๆ */
  display: flex;
  justify-content: center;
  align-items: center;
}

.magical-text {
  /* ตัวหนังสือไล่สีสะท้อนแสง */
  background: linear-gradient(to right, #c4b5fd, #fbcfe8, #a78bfa, #c4b5fd);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-size: 200% auto;
  animation: shineText 3s linear infinite; /* แอนิเมชันแสงวิ่งบนตัวหนังสือ */
  text-shadow: 0 0 10px rgba(167, 139, 250, 0.2);
}

@keyframes magicBG {
  0% { background-position: 0% 50%; }
  50% { background-position: 100% 50%; }
  100% { background-position: 0% 50%; }
}

@keyframes shineText {
  to { background-position: 200% center; }
}

/* Modal CSS */
.overlay {
  position: fixed; inset: 0;
  background: rgba(15, 23, 42, 0.6); 
  display: flex; align-items: center; justify-content: center;
  z-index: 500; padding: 1rem;
  backdrop-filter: blur(4px); 
}
.modal {
  background: #ffffff; border-radius: 20px;
  width: 100%; max-width: 500px;
  max-height: 90vh; overflow-y: auto; padding: 2.5rem;
  box-shadow: 0 20px 25px -5px rgba(0, 0, 0, 0.1), 0 8px 10px -6px rgba(0, 0, 0, 0.1);
}
.modal-title { 
  font-size: 1.3rem; font-weight: 800; margin-bottom: 1.75rem; color: #1e293b; 
  display: flex; align-items: center; gap: 0.5rem;
}
.modal-title-icon { font-size: 1.5rem; }

.form-group { margin-bottom: 1.25rem; }
.form-label { display: block; font-size: .87rem; font-weight: 600; margin-bottom: .4rem; color: #334155; }
.form-input {
  width: 100%; padding: .65rem .85rem;
  border: 1.5px solid #cbd5e1; border-radius: 10px;
  font-size: .95rem; outline: none; transition: all .2s;
  font-family: inherit;
}
.form-input:focus { 
  border-color: #6366f1; 
  box-shadow: 0 0 0 3px rgba(99, 102, 241, 0.15);
}
.form-row { display: grid; grid-template-columns: 1fr 1fr; gap: 1rem; }

.modal-footer {
  display: flex; gap: .75rem; justify-content: flex-end;
  padding-top: 1.25rem; border-top: 1px solid #e2e8f0; margin-top: 1.5rem;
}
.btn-cancel {
  background: #f1f5f9; color: #475569;
  border: none; border-radius: 10px; padding: .65rem 1.4rem;
  font-weight: 600; cursor: pointer; transition: all .2s;
}
.btn-cancel:hover { background: #e2e8f0; color: #1e293b; }
.btn-save {
  background: linear-gradient(135deg, #6366f1, #8b5cf6);
  color: #ffffff;
  border: none; border-radius: 10px; padding: .65rem 1.5rem;
  font-weight: 700; cursor: pointer; transition: all .3s;
  box-shadow: 0 4px 10px rgba(99, 102, 241, 0.2);
}
.btn-save:hover { 
  background: linear-gradient(135deg, #4f46e5, #7c3aed);
  transform: translateY(-1px);
  box-shadow: 0 6px 14px rgba(99, 102, 241, 0.3);
}

.confirm { text-align: center; max-width: 400px; padding: 2.5rem 2rem; }
.confirm-icon  { font-size: 3.5rem; margin-bottom: 1.25rem; }
.confirm-title { font-size: 1.25rem; font-weight: 800; margin-bottom: .75rem; color: #1e293b; }
.confirm-desc  { font-size: .95rem; color: #475569; margin-bottom: 1.75rem; line-height: 1.6; }
.confirm-actions { display: flex; gap: 1rem; justify-content: center; }
.btn-danger-confirm {
  background: #e11d48; color: #fff;
  border: none; border-radius: 10px; padding: .65rem 1.5rem;
  font-weight: 700; cursor: pointer; transition: all .2s;
  box-shadow: 0 4px 10px rgba(225, 29, 72, 0.2);
}
.btn-danger-confirm:hover { 
  background: #be123c; 
  transform: translateY(-1px);
  box-shadow: 0 6px 14px rgba(225, 29, 72, 0.3);
}

@media (max-width: 640px) {
  .form-row { grid-template-columns: 1fr; }
  .stats-grid { grid-template-columns: 1fr 1fr; }
  .main { padding: 1rem; }
}
</style>