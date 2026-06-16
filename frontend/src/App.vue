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
  ...new Set(products.value.map(p => p.category))].sort()
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

// คืน CSS class ตาม stock level
function stockClass(s) {
  if (s <= 0) return 'out'    // เสื่อมสลาย / หมด
  if (s < 10) return 'low'    // ใกล้หมด
  if (s < 30) return 'mid'    // ปานกลาง
  return 'high'               // พลังเต็มเปี่ยม
}

// คืน CSS class ตาม category (ปรับปรุงเฉดสีเวทมนตร์)
function catClass(cat) {
  const m = {
    Electronics: 'c-elec',  Clothing: 'c-cloth',
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
  <div class="witch-theme">

    <header class="app-header">
      <div class="logo">
        <span class="logo-icon">🔮</span>
        <div>
          <div class="logo-name">PotionCraft</div>
          <div class="logo-sub">ระบบบริหารคลังวัตถุดิบและไอเทมเวทมนตร์</div>
        </div>
      </div>
      <button class="btn-add" @click="openAdd">✨ ปรุงไอเทมใหม่</button>
    </header>

    <main class="main">

      <div class="stats-grid">
        <div class="stat-card">
          <div class="stat-icon si-purple">🧪</div>
          <div class="stat-body">
            <div class="stat-val" style="color:#7c3aed">{{ stats.total }}</div>
            <div class="stat-label">มนตราทั้งหมด</div>
          </div>
        </div>
        <div class="stat-card">
          <div class="stat-icon si-magenta">💥</div>
          <div class="stat-body">
            <div class="stat-val" style="color:#db2777">{{ stats.lowStock }}</div>
            <div class="stat-label">พลังอ่อนแรง (<10)</div>
          </div>
        </div>
        <div class="stat-card">
          <div class="stat-icon si-gold">✨</div>
          <div class="stat-body">
            <div class="stat-val" style="color:#d97706">{{ stats.totalItems.toLocaleString() }}</div>
            <div class="stat-label">ชิ้นส่วนเวทมนตร์รวม</div>
          </div>
        </div>
        <div class="stat-card">
          <div class="stat-icon si-indigo">🪙</div>
          <div class="stat-body">
            <div class="stat-val" style="color:#4f46e5; font-size:1.3rem">
              ฿{{ Math.round(stats.totalValue).toLocaleString() }}
            </div>
            <div class="stat-label">มูลค่าคลังเวทมนตร์</div>
          </div>
        </div>
      </div>

      <div class="alert-low" v-if="stats.lowStock > 0">
        🔮 มีวัตถุดิบเวทมนตร์ <strong>{{ stats.lowStock }} รายการ</strong> 
        ที่พลังกำลังจะสูญสิ้น (น้อยกว่า 10 ชิ้น) — พ่อมดแม่มดโปรดร่ายคาถาเติมด่วน!
      </div>

      <div class="toolbar">
        <input
          v-model="search"
          @input="debounceFetch"
          class="input-search"
          placeholder="📜 ค้นหาชื่อไอเทม หรือคัมภีร์คาถา..."
        />
        <select v-model="catFilter" @change="fetchProducts" class="input-select">
          <option value="">ทุกแขนงเวทมนตร์</option>
          <option v-for="c in categories" :key="c" :value="c">{{ c }}</option>
        </select>
        <span class="result-count" v-if="!loading">
          ค้นพบไอเทม {{ filtered.length }} / {{ products.length }} ตำรับ
        </span>
      </div>

      <div class="state-box" v-if="loading">
        <div class="state-icon loading-spin">🌀</div>
        <div>กำลังสื่อสารกับมิติเวทมนตร์...</div>
      </div>

      <div class="state-box" v-else-if="filtered.length === 0">
        <div class="state-icon">👁️‍🗨️</div>
        <div class="state-title font-magic">ความว่างเปล่า</div>
        <div class="state-desc">
          {{ search || catFilter ? 'ไม่พบร่องรอยของเวทมนตร์นี้ ลองเปลี่ยนคำร่ายคาถาดู' : 'กดปุ่ม "✨ ปรุงไอเทมใหม่" เพื่อเริ่มสร้างประวัติศาสตร์' }}
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
            <span class="cat-badge" :class="catClass(p.category)">🌟 {{ p.category }}</span>
            <div class="product-name font-magic">{{ p.name }}</div>
            <div class="product-desc" v-if="p.description">{{ p.description }}</div>
            <div class="product-price">
              ฿{{ parseFloat(p.price).toLocaleString('th-TH', { minimumFractionDigits: 2 }) }}
            </div>
            <div class="stock-info">
              <div class="stock-row">
                <span class="stock-label">ระดับพลังงานคลัง</span>
                <span class="stock-num" :class="stockClass(p.stock)">
                  {{ p.stock.toLocaleString() }} ชิ้น
                  <span v-if="p.stock <= 0"> — สูญสลายแล้ว!</span>
                  <span v-else-if="p.stock < 10"> — พลังงานต่ำ!</span>
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
            <button class="btn-edit" @click="openEdit(p)">📝 ปรับสูตร</button>
            <button class="btn-del" @click="confirmDelete = p">🧹 สลายมนต์</button>
          </div>
        </div>
      </div>

    </main>

    <div class="overlay" v-if="showModal" @click.self="showModal = false">
      <div class="modal">
        <div class="modal-title font-magic">
          {{ editingId ? '🔮 ปรับแต่งสูตรเวทมนตร์' : '🧪 ปรุงไอเทมใหม่' }}
        </div>
        <form @submit.prevent="saveProduct">
          <div class="form-group">
            <label class="form-label">ชื่อไอเทมเวทมนตร์ *</label>
            <input class="form-input" v-model="form.name" placeholder="เช่น น้ำยาโชคดีฟิลิกซ์ ฟิลิซิส" required />
          </div>
          <div class="form-row">
            <div class="form-group">
              <label class="form-label">สายเวทมนตร์ *</label>
              <input class="form-input" v-model="form.category" list="cat-list" placeholder="เช่น Potions" required />
              <datalist id="cat-list">
                <option v-for="c in categories" :value="c" :key="c" />
              </datalist>
            </div>
            <div class="form-group">
              <label class="form-label">ราคาแลกเปลี่ยน (บาท) *</label>
              <input class="form-input" v-model="form.price" type="number" min="0" step="0.01" placeholder="0.00" required />
            </div>
          </div>
          <div class="form-group">
            <label class="form-label">จำนวนโหลคลังสินค้า *</label>
            <input class="form-input" v-model="form.stock" type="number" min="0" placeholder="0" required />
          </div>
          <div class="form-group">
            <label class="form-label">บันทึกคัมภีร์ (คำอธิบาย)</label>
            <textarea class="form-input" v-model="form.description" rows="3" placeholder="รายละเอียดส่วนผสมเวทมนตร์..." style="resize:vertical"></textarea>
          </div>
          <div class="modal-footer">
            <button type="button" class="btn-cancel" @click="showModal = false">ปิดตำรา</button>
            <button type="submit" class="btn-save">
              {{ editingId ? 'บันทึกการร่ายมนต์' : 'หลอมรวมไอเทม' }}
            </button>
          </div>
        </form>
      </div>
    </div>

    <div class="overlay" v-if="confirmDelete" @click.self="confirmDelete = null">
      <div class="modal confirm">
        <div class="confirm-icon">🔥</div>
        <div class="confirm-title font-magic">ยืนยันการทำลายมนตรา</div>
        <div class="confirm-desc">
          คุณต้องการลบคาถาและไอเทม <strong>{{ confirmDelete?.name }}</strong> ออกจากมิตินี้ใช่หรือไม่?<br>
          <span style="color:#ec4899">เมื่อทำลายแล้ว จะไม่สามารถกู้คืนจิตวิญญาณกลับมาได้</span>
        </div>
        <div class="confirm-actions">
          <button class="btn-cancel" @click="confirmDelete = null">ยับยั้งไว้</button>
          <button class="btn-danger-confirm" @click="deleteProduct(confirmDelete.id)">เผาทำลาย</button>
        </div>
      </div>
    </div>

  </div>
</template>

<style scoped>
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

/* โทนสีหลัก: พื้นหลังสว่างโทนเนียนตาแบบกระดาษเวทมนตร์ ตัดกับสีม่วงเข้ม ลึกลับ */
.witch-theme {
  background-color: #f5f3ff; 
  min-height: 100vh;
  color: #352660;
  font-family: system-ui, -apple-system, sans-serif;
}

.font-magic {
  letter-spacing: 0.5px;
  color: #4c1d95;
}

.app-header {
  position: sticky; top: 0; z-index: 100;
  background: rgba(255, 255, 255, 0.92);
  backdrop-filter: blur(8px);
  border-bottom: 2px solid #ddd6fe;
  height: 65px; padding: 0 1.5rem;
  display: flex; align-items: center; gap: .85rem;
  box-shadow: 0 4px 20px rgba(124, 58, 237, 0.08);
}
.logo { display: flex; align-items: center; gap: .6rem; }
.logo-icon { font-size: 1.8rem; filter: drop-shadow(0 2px 4px rgba(124,58,237,0.3)); }
.logo-name { font-weight: 800; font-size: 1.3rem; color: #5b21b6; line-height: 1; }
.logo-sub  { font-size: .75rem; color: #7c3aed; opacity: 0.8; }

.btn-add {
  margin-left: auto;
  background: linear-gradient(135deg, #7c3aed 0%, #6d28d9 100%);
  color: #fff;
  border: none; border-radius: 10px;
  padding: .6rem 1.4rem; font-size: .9rem; font-weight: 700;
  cursor: pointer; transition: all .2s;
  box-shadow: 0 4px 12px rgba(109, 40, 217, 0.3);
}
.btn-add:hover { 
  transform: translateY(-1px);
  box-shadow: 0 6px 16px rgba(109, 40, 217, 0.45);
  background: linear-gradient(135deg, #8b5cf6 0%, #7c3aed 100%);
}

.main { max-width: 1280px; margin: 0 auto; padding: 1.75rem 1.5rem; }

/* Dashboard บัตรสถิติกึ่งโปร่งแสง สไตล์เวทมนตร์ */
.stats-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(220px, 1fr));
  gap: 1.2rem; margin-bottom: 1.5rem;
}
.stat-card {
  background: #ffffff; 
  border: 1px solid #e9d5ff; 
  border-radius: 16px;
  padding: 1.2rem; display: flex; align-items: center; gap: .9rem;
  box-shadow: 0 4px 15px rgba(124, 58, 237, 0.04);
  transition: transform .2s;
}
.stat-card:hover { transform: translateY(-2px); }

.stat-icon {
  width: 48px; height: 48px; border-radius: 12px;
  display: flex; align-items: center; justify-content: center;
  font-size: 1.4rem; flex-shrink: 0;
}
.si-purple  { background: #ede9fe; }
.si-magenta { background: #fce7f3; }
.si-gold    { background: #fef3c7; }
.si-indigo  { background: #e0e7ff; }

.stat-val { font-size: 1.8rem; font-weight: 800; line-height: 1; }
.stat-label { font-size: .82rem; color: #6d28d9; opacity: 0.8; margin-top: .25rem; font-weight: 500; }

.alert-low {
  background: #fdf2f8; border: 1px solid #fbcfe8;
  border-radius: 12px; padding: .9rem 1.3rem;
  font-size: .92rem; margin-bottom: 1.5rem; color: #9d174d;
  box-shadow: 0 4px 12px rgba(219, 39, 119, 0.05);
}

/* Toolbar และฟิลเตอร์ค้นหา */
.toolbar { display: flex; gap: .75rem; margin-bottom: 1.5rem; flex-wrap: wrap; align-items: center; }
.input-search {
  flex: 1; min-width: 250px;
  padding: .65rem 1.1rem; border: 2px solid #ddd6fe; border-radius: 10px;
  font-size: .95rem; outline: none; transition: all .2s;
  background-color: #fff;
}
.input-search:focus { border-color: #7c3aed; box-shadow: 0 0 0 3px rgba(124, 58, 237, 0.15); }
.input-select {
  padding: .65rem 1rem; border: 2px solid #ddd6fe; border-radius: 10px;
  background: #fff; font-size: .9rem; outline: none; cursor: pointer; color: #4c1d95;
  font-weight: 600;
}
.input-select:focus { border-color: #7c3aed; }
.result-count { font-size: .85rem; color: #6d28d9; opacity: 0.8; white-space: nowrap; font-weight: 500; }

/* Loading & Empty state */
.state-box { text-align: center; padding: 5rem 1rem; color: #6d28d9; }
.state-icon { font-size: 3.5rem; margin-bottom: .75rem; display: inline-block; }
.loading-spin { animation: spin 2s linear infinite; }
@keyframes spin { 100% { transform: rotate(360deg); } }
.state-title { font-size: 1.25rem; font-weight: 700; margin-bottom: .4rem; }
.state-desc { font-size: .92rem; opacity: 0.8; }

/* การ์ดสินค้าเวทมนตร์ */
.product-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(285px, 1fr));
  gap: 1.5rem;
}
.product-card {
  background: #fff; border: 1px solid #e9d5ff; border-radius: 16px;
  overflow: hidden; transition: all .3s cubic-bezier(0.4, 0, 0.2, 1);
  box-shadow: 0 4px 15px rgba(124, 58, 237, 0.03);
}
.product-card:hover { 
  transform: translateY(-5px); 
  box-shadow: 0 12px 28px rgba(109, 40, 217, 0.12);
  border-color: #c084fc;
}
.product-card.card-low  { border-color: #fbcfe8; background: #fffdfebd; }
.product-card.card-out  { border-color: #e4e4e7; background: #fafafa; opacity: .65; }

.card-top { padding: 1.25rem; }
.cat-badge {
  display: inline-block; padding: .25rem .75rem;
  border-radius: 20px; font-size: .72rem; font-weight: 700; margin-bottom: .75rem;
  box-shadow: 0 2px 6px rgba(0,0,0,0.03);
}

/* พาเลทสี Badge หมวดหมู่สไตล์เวทมนตร์ */
.c-elec  { background: #e0e7ff; color: #3730a3; } /* สายอสนีบาต/ไฟฟ้า */
.c-cloth { background: #fce7f3; color: #9d174d; } /* ผ้าคลุมแม่มด */
.c-foot  { background: #f3e8ff; color: #6b21a8; } /* รองเท้าเวทมนตร์ */
.c-food  { background: #fef3c7; color: #92400e; } /* เสบียงเอลฟ์ */
.c-sport { background: #ccfbf1; color: #115e59; } /* อุปกรณ์ควิดดิช */
.c-book  { background: #e0f2fe; color: #0369a1; } /* คัมภีร์เวท */
.c-furn  { background: #fae8ff; color: #86198f; } /* เฟอร์นิเจอร์ลอยได้ */
.c-bag   { background: #ffedd5; color: #9a3412; } /* กระเป๋ามิติมืด */
.c-acc   { background: #dcfce7; color: #166534; } /* เครื่องรางพิทักษ์ */
.c-tool  { background: #f1f5f9; color: #475569; } /* ไม้กายสิทธิ์ */
.c-other { background: #f3e8ff; color: #5b21b6; }

.product-name  { font-size: 1.1rem; font-weight: 800; color: #2e1065; margin-bottom: .4rem; line-height: 1.4; }
.product-desc  { font-size: .85rem; color: #6d28d9; opacity: 0.75; line-height: 1.5; margin-bottom: .85rem; }
.product-price { font-size: 1.35rem; font-weight: 900; color: #7c3aed; }

.stock-info { margin-top: 1rem; }
.stock-row  { display: flex; justify-content: space-between; font-size: .82rem; margin-bottom: .35rem; }
.stock-label { color: #6d28d9; opacity: 0.7; }
.stock-num   { font-weight: 700; }
.stock-num.out  { color: #9ca3af; }
.stock-num.low  { color: #db2777; }
.stock-num.mid  { color: #d97706; }
.stock-num.high { color: #2563eb; }

/* หลอดพลังงานมานาคลังสินค้า */
.stock-track { height: 7px; background: #f3e8ff; border-radius: 4px; overflow: hidden; }
.stock-fill  { height: 100%; border-radius: 4px; transition: width .5s cubic-bezier(0.4, 0, 0.2, 1); min-width: 5px; }
.stock-fill.out  { background: #d1d5db; width: 2% !important; }
.stock-fill.low  { background: linear-gradient(90deg, #ec4899, #db2777); }
.stock-fill.mid  { background: linear-gradient(90deg, #f59e0b, #d97706); }
.stock-fill.high { background: linear-gradient(90deg, #a78bfa, #7c3aed); }

.card-footer {
  display: flex; gap: .5rem;
  padding: .85rem 1.25rem;
  border-top: 1px solid #f3e8ff; background: #faf9ff;
}
.btn-edit, .btn-del {
  flex: 1; padding: .5rem; border-radius: 8px;
  font-size: .85rem; font-weight: 600; cursor: pointer; border: none; transition: all .2s;
}
.btn-edit { background: #ede9fe; color: #5b21b6; }
.btn-edit:hover { background: #ddd6fe; }
.btn-del  { background: #fce7f3; color: #db2777; }
.btn-del:hover  { background: #fbcfe8; }

/* หน้าต่างเวทมนตร์ (Modal) popup */
.overlay {
  position: fixed; inset: 0;
  background: rgba(46, 16, 101, 0.4);
  backdrop-filter: blur(4px);
  display: flex; align-items: center; justify-content: center;
  z-index: 500; padding: 1rem;
}
.modal {
  background: #fff; border-radius: 20px;
  width: 100%; max-width: 500px;
  max-height: 90vh; overflow-y: auto; padding: 2rem;
  border: 1px solid #ddd6fe;
  box-shadow: 0 20px 50px rgba(76, 29, 149, 0.15);
}
.modal-title { font-size: 1.3rem; font-weight: 800; margin-bottom: 1.5rem; color: #2e1065; }

.form-group { margin-bottom: 1.1rem; }
.form-label { display: block; font-size: .87rem; font-weight: 700; margin-bottom: .4rem; color: #4c1d95; }
.form-input {
  width: 100%; padding: .65rem .9rem;
  border: 2px solid #ddd6fe; border-radius: 10px;
  font-size: .95rem; outline: none; transition: all .2s;
}
.form-input:focus { border-color: #7c3aed; box-shadow: 0 0 0 3px rgba(124, 58, 237, 0.1); }
.form-row { display: grid; grid-template-columns: 1fr 1fr; gap: .75rem; }

.modal-footer {
  display: flex; gap: .75rem; justify-content: flex-end;
  padding-top: 1.1rem; border-top: 1px solid #f3e8ff; margin-top: 1.2rem;
}
.btn-cancel {
  background: #f3e8ff; color: #6d28d9;
  border: none; border-radius: 9px; padding: .6rem 1.4rem;
  font-weight: 600; cursor: pointer;
}
.btn-cancel:hover { background: #e9d5ff; }
.btn-save {
  background: linear-gradient(135deg, #7c3aed 0%, #6d28d9 100%); color: #fff;
  border: none; border-radius: 9px; padding: .6rem 1.4rem;
  font-weight: 700; cursor: pointer;
}
.btn-save:hover { background: linear-gradient(135deg, #8b5cf6 0%, #7c3aed 100%); }

.confirm { text-align: center; max-width: 400px; border-color: #fbcfe8; }
.confirm-icon  { font-size: 3.5rem; margin-bottom: 0.5rem; }
.confirm-title { font-size: 1.2rem; font-weight: 800; margin-bottom: .5rem; color: #9d174d; }
.confirm-desc  { font-size: .92rem; color: #6d28d9; margin-bottom: 1.5rem; line-height: 1.6; }
.btn-danger-confirm {
  background: linear-gradient(135deg, #ec4899 0%, #db2777 100%); color: #fff;
  border: none; border-radius: 9px; padding: .6rem 1.5rem;
  font-weight: 700; cursor: pointer;
  box-shadow: 0 4px 12px rgba(219, 39, 119, 0.2);
}
.btn-danger-confirm:hover { background: linear-gradient(135deg, #f472b6 0%, #ec4899 100%); }

@media (max-width: 640px) {
  .form-row { grid-template-columns: 1fr; }
  .stats-grid { grid-template-columns: 1fr 1fr; }
}
</style>