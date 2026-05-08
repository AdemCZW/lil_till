<script setup>
import { ref, computed, onMounted, onUnmounted, reactive } from 'vue'

// ===== 常數 =====
const STORAGE_KEY = 'salesTrackerData_v1'
const DEFAULT_ITEMS = [
  { name: '西瓜冰棒', price: 60 },
  { name: '哈密瓜冰棒', price: 60 },
  { name: '百香果冰棒', price: 60 },
  { name: '藍莓優格冰棒', price: 80 },
  { name: '草莓牛奶冰棒', price: 80 },
  { name: '葡萄冰棒', price: 80 },
  { name: '榴槤冰棒', price: 100 },
  { name: '客製化-野花園', price: 100 },
  { name: '客製化-花綻冰果', price: 100 },
  { name: '客製化-芒果芳香', price: 100 },
]

// ===== 工具 =====
function todayKey(d = new Date()) {
  return (
    d.getFullYear() +
    '-' +
    String(d.getMonth() + 1).padStart(2, '0') +
    '-' +
    String(d.getDate()).padStart(2, '0')
  )
}

function loadData() {
  try {
    const raw = localStorage.getItem(STORAGE_KEY)
    if (raw) return JSON.parse(raw)
  } catch (e) {
    /* ignore */
  }
  return { items: DEFAULT_ITEMS.slice(), salesByDate: {} }
}

// ===== 狀態 =====
const data = reactive(loadData())
const editMode = ref(false)
const sheetOpen = ref(false)
const newName = ref('')
const newPrice = ref('')
const now = ref(new Date())

let timerId = null
onMounted(() => {
  timerId = setInterval(() => (now.value = new Date()), 60000)
})
onUnmounted(() => clearInterval(timerId))

function saveData() {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(data))
}

// ===== 計算屬性 =====
const dateLabel = computed(() => {
  const d = now.value
  const week = ['日', '一', '二', '三', '四', '五', '六'][d.getDay()]
  return `${d.getFullYear()}/${String(d.getMonth() + 1).padStart(2, '0')}/${String(
    d.getDate()
  ).padStart(2, '0')} 週${week}`
})

const todaySales = computed(() => {
  const k = todayKey(now.value)
  if (!data.salesByDate[k]) data.salesByDate[k] = []
  return data.salesByDate[k]
})

const totalRevenue = computed(() =>
  todaySales.value.reduce((sum, s) => sum + Number(s.price), 0)
)

const stats = computed(() => {
  const map = {}
  todaySales.value.forEach((s) => {
    if (!map[s.name]) map[s.name] = { name: s.name, qty: 0, subtotal: 0 }
    map[s.name].qty += 1
    map[s.name].subtotal += Number(s.price)
  })
  return Object.values(map).sort((a, b) => b.subtotal - a.subtotal)
})

const reversedSales = computed(() => todaySales.value.slice().reverse())

// ===== 操作 =====
function sell(idx) {
  if (editMode.value) return
  const it = data.items[idx]
  if (!it) return
  const d = new Date()
  const time =
    String(d.getHours()).padStart(2, '0') +
    ':' +
    String(d.getMinutes()).padStart(2, '0') +
    ':' +
    String(d.getSeconds()).padStart(2, '0')
  todaySales.value.push({ name: it.name, price: Number(it.price), time })
  saveData()
  if (navigator.vibrate) navigator.vibrate(10)
}

function undoSale(reverseIdx) {
  // reverseIdx 是 reversedSales 陣列中的索引,需轉回原陣列索引
  const realIdx = todaySales.value.length - 1 - reverseIdx
  if (realIdx >= 0 && realIdx < todaySales.value.length) {
    todaySales.value.splice(realIdx, 1)
    saveData()
  }
}

function addItem() {
  const name = newName.value.trim()
  const price = Number(newPrice.value)
  if (!name) return alert('請輸入品項名稱')
  if (!(price >= 0)) return alert('請輸入有效單價')
  data.items.push({ name, price })
  saveData()
  newName.value = ''
  newPrice.value = ''
}

function removeItem(idx) {
  const it = data.items[idx]
  if (!it) return
  if (!confirm(`刪除「${it.name}」?\n(已記錄的銷售不會受影響)`)) return
  data.items.splice(idx, 1)
  saveData()
}

function toggleEdit() {
  editMode.value = !editMode.value
}

function resetToday() {
  if (!confirm('清除今日所有銷售紀錄?\n此動作無法復原')) return
  data.salesByDate[todayKey(now.value)] = []
  saveData()
}

function exportCSV() {
  const sales = todaySales.value
  if (sales.length === 0) return alert('今日尚無交易可匯出')
  let csv = '﻿'
  csv += `今日銷售統計,${todayKey(now.value)}\n\n`
  csv += '品項,數量,小計\n'
  stats.value.forEach((s) => {
    csv += `${s.name},${s.qty},${s.subtotal}\n`
  })
  csv += `\n總營收,,${totalRevenue.value}\n\n`
  csv += '交易明細\n時間,品項,金額\n'
  sales.forEach((s) => {
    csv += `${s.time},${s.name},${s.price}\n`
  })
  const blob = new Blob([csv], { type: 'text/csv;charset=utf-8' })
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  a.download = `sales-${todayKey(now.value)}.csv`
  a.click()
  URL.revokeObjectURL(url)
}

function openSheet() {
  sheetOpen.value = true
}
function closeSheet() {
  sheetOpen.value = false
}

function fmt(n) {
  return Number(n).toLocaleString()
}
</script>

<template>
  <div class="app" :class="{ 'edit-mode': editMode }">
    <!-- 頂部 -->
    <div class="topbar">
      <div class="topbar-row">
        <div>
          <div class="title">今日銷售</div>
          <div class="date">{{ dateLabel }}</div>
        </div>
        <button class="menu-btn" @click="openSheet">選單</button>
      </div>
    </div>

    <!-- 總營收 hero -->
    <div class="hero">
      <div class="label">TOTAL REVENUE</div>
      <div class="amount">
        <span class="currency">$</span>{{ fmt(totalRevenue) }}
      </div>
      <div class="meta">{{ todaySales.length }} 筆交易</div>
    </div>

    <div class="content">
      <!-- 品項 -->
      <div class="section">
        <div class="section-head">
          <h2>品項</h2>
          <button class="action" @click="toggleEdit">
            {{ editMode ? '完成' : '編輯' }}
          </button>
        </div>
        <div class="items-grid">
          <button
            v-for="(it, idx) in data.items"
            :key="idx"
            class="item-btn"
            @click="sell(idx)"
          >
            <button
              v-if="editMode"
              class="del"
              @click.stop="removeItem(idx)"
            >×</button>
            <div class="name">{{ it.name }}</div>
            <div class="price">${{ fmt(it.price) }}</div>
          </button>
          <div v-if="data.items.length === 0" class="empty-items">
            尚無品項,從下方新增
          </div>
        </div>
        <div class="add-form">
          <input
            v-model="newName"
            type="text"
            placeholder="品項名稱"
            autocomplete="off"
            @keyup.enter="addItem"
          />
          <input
            v-model="newPrice"
            type="number"
            inputmode="numeric"
            min="0"
            placeholder="單價"
            @keyup.enter="addItem"
          />
          <button class="add-btn" @click="addItem">新增</button>
        </div>
      </div>

      <!-- 各品項統計 -->
      <div class="section">
        <div class="section-head"><h2>各品項統計</h2></div>
        <div v-if="stats.length" class="stats">
          <div v-for="s in stats" :key="s.name" class="stat-row">
            <div class="name">{{ s.name }}</div>
            <div class="qty">×{{ s.qty }}</div>
            <div class="subtotal">${{ fmt(s.subtotal) }}</div>
          </div>
        </div>
        <div v-else class="empty-mini">尚無銷售紀錄</div>
      </div>

      <!-- 交易明細 -->
      <div class="section">
        <div class="section-head"><h2>交易明細</h2></div>
        <div v-if="reversedSales.length" class="timeline">
          <div v-for="(s, i) in reversedSales" :key="i" class="tx">
            <div class="info">
              <div class="n">{{ s.name }}</div>
              <div class="t">{{ s.time }}</div>
            </div>
            <div class="right">
              <span class="a">${{ fmt(s.price) }}</span>
              <button class="undo" @click="undoSale(i)">撤銷</button>
            </div>
          </div>
        </div>
        <div v-else class="empty-mini">尚無交易</div>
      </div>
    </div>

    <!-- 底部 sheet -->
    <transition name="fade">
      <div v-if="sheetOpen" class="sheet-mask" @click="closeSheet"></div>
    </transition>
    <transition name="slide">
      <div v-if="sheetOpen" class="sheet">
        <div class="grabber"></div>
        <button @click="exportCSV(); closeSheet()">匯出今日 CSV</button>
        <button class="danger" @click="resetToday(); closeSheet()">
          清除今日紀錄
        </button>
        <button class="cancel" @click="closeSheet">取消</button>
      </div>
    </transition>
  </div>
</template>

<style scoped>
* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
  -webkit-tap-highlight-color: transparent;
}
.app {
  --bg: #f5f5f5;
  --surface: #ffffff;
  --surface-2: #fafafa;
  --ink: #000000;
  --ink-soft: #1a1a1a;
  --muted: #8a8a8a;
  --line: #e5e5e5;
  --line-soft: #f0f0f0;

  font-family: -apple-system, BlinkMacSystemFont, 'PingFang TC',
    'Microsoft JhengHei', sans-serif;
  background: var(--bg);
  color: var(--ink);
  font-size: 15px;
  -webkit-font-smoothing: antialiased;
  max-width: 480px;
  margin: 0 auto;
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  padding-bottom: env(safe-area-inset-bottom);
}

/* 頂部 */
.topbar {
  background: var(--ink);
  color: white;
  padding: 14px 16px;
  padding-top: calc(14px + env(safe-area-inset-top));
  position: sticky;
  top: 0;
  z-index: 10;
}
.topbar-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
}
.title {
  font-size: 13px;
  opacity: 0.6;
  letter-spacing: 1px;
}
.date {
  font-size: 14px;
  margin-top: 2px;
  opacity: 0.85;
}
.menu-btn {
  background: transparent;
  border: 1px solid rgba(255, 255, 255, 0.25);
  color: white;
  padding: 8px 12px;
  border-radius: 8px;
  font-size: 13px;
  cursor: pointer;
}
.menu-btn:active {
  background: rgba(255, 255, 255, 0.1);
}

/* Hero */
.hero {
  background: var(--ink);
  color: white;
  padding: 24px 20px 32px;
  text-align: center;
}
.hero .label {
  font-size: 12px;
  letter-spacing: 2px;
  opacity: 0.55;
  text-transform: uppercase;
}
.hero .amount {
  font-size: 56px;
  font-weight: 700;
  margin-top: 8px;
  letter-spacing: -1px;
  font-variant-numeric: tabular-nums;
}
.hero .amount .currency {
  font-size: 28px;
  opacity: 0.5;
  margin-right: 4px;
  vertical-align: top;
  line-height: 56px;
}
.hero .meta {
  font-size: 12px;
  opacity: 0.5;
  margin-top: 6px;
}

/* 內容 */
.content {
  padding: 16px;
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 14px;
}
.section {
  background: var(--surface);
  border-radius: 14px;
  overflow: hidden;
}
.section-head {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 14px 16px 8px;
}
.section-head h2 {
  font-size: 12px;
  letter-spacing: 1.5px;
  color: var(--muted);
  text-transform: uppercase;
  font-weight: 600;
}
.section-head .action {
  background: transparent;
  border: none;
  color: var(--ink);
  font-size: 13px;
  cursor: pointer;
  padding: 4px 8px;
  margin: -4px -8px;
}
.section-head .action:active {
  opacity: 0.5;
}

/* 品項按鈕 */
.items-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 8px;
  padding: 4px 12px 12px;
}
.item-btn {
  background: var(--surface-2);
  border: 1px solid var(--line);
  color: var(--ink);
  padding: 18px 12px;
  border-radius: 12px;
  cursor: pointer;
  text-align: left;
  position: relative;
  transition: transform 0.08s, background 0.15s;
  min-height: 76px;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
}
.item-btn:active {
  transform: scale(0.97);
  background: var(--ink);
  color: white;
  border-color: var(--ink);
}
.item-btn .name {
  font-size: 16px;
  font-weight: 600;
  line-height: 1.2;
  word-break: break-all;
}
.item-btn .price {
  font-size: 13px;
  color: var(--muted);
  margin-top: 6px;
  font-variant-numeric: tabular-nums;
}
.item-btn:active .price {
  color: rgba(255, 255, 255, 0.7);
}
.item-btn .del {
  position: absolute;
  top: 8px;
  right: 8px;
  background: var(--ink);
  border: none;
  color: white;
  width: 22px;
  height: 22px;
  border-radius: 50%;
  cursor: pointer;
  font-size: 14px;
  line-height: 22px;
  text-align: center;
  padding: 0;
}
.app.edit-mode .item-btn {
  animation: shake 0.3s infinite alternate;
}
@keyframes shake {
  from {
    transform: rotate(-0.5deg);
  }
  to {
    transform: rotate(0.5deg);
  }
}
.empty-items {
  grid-column: 1 / -1;
  text-align: center;
  color: var(--muted);
  padding: 30px 12px;
  font-size: 13px;
}

/* 新增表單 */
.add-form {
  display: flex;
  gap: 6px;
  padding: 0 12px 14px;
}
.add-form input {
  flex: 1;
  min-width: 0;
  padding: 12px 14px;
  border: 1px solid var(--line);
  border-radius: 10px;
  font-size: 15px;
  background: var(--surface-2);
  color: var(--ink);
  -webkit-appearance: none;
}
.add-form input:focus {
  outline: none;
  border-color: var(--ink);
  background: white;
}
.add-form input[type='number'] {
  max-width: 90px;
}
.add-form .add-btn {
  background: var(--ink);
  color: white;
  border: none;
  padding: 0 16px;
  border-radius: 10px;
  font-size: 15px;
  font-weight: 500;
  cursor: pointer;
}
.add-form .add-btn:active {
  background: #333;
}

/* 統計 */
.stats {
  padding: 4px 16px 14px;
}
.stat-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 12px 0;
  border-bottom: 1px solid var(--line-soft);
}
.stat-row:last-child {
  border-bottom: none;
}
.stat-row .name {
  font-size: 15px;
  font-weight: 500;
  flex: 1;
  min-width: 0;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
.stat-row .qty {
  background: var(--line-soft);
  color: var(--ink-soft);
  padding: 3px 10px;
  border-radius: 100px;
  font-size: 12px;
  font-weight: 600;
  margin: 0 12px;
  font-variant-numeric: tabular-nums;
  min-width: 36px;
  text-align: center;
}
.stat-row .subtotal {
  font-size: 15px;
  font-weight: 600;
  font-variant-numeric: tabular-nums;
  min-width: 70px;
  text-align: right;
}

/* 交易明細 */
.timeline {
  padding: 0 16px 14px;
  max-height: 280px;
  overflow-y: auto;
  -webkit-overflow-scrolling: touch;
}
.tx {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 10px 0;
  border-bottom: 1px solid var(--line-soft);
}
.tx:last-child {
  border-bottom: none;
}
.tx .info {
  flex: 1;
  min-width: 0;
}
.tx .info .n {
  font-size: 14px;
  font-weight: 500;
}
.tx .info .t {
  font-size: 11px;
  color: var(--muted);
  margin-top: 2px;
  font-variant-numeric: tabular-nums;
}
.tx .right {
  display: flex;
  align-items: center;
  gap: 8px;
}
.tx .a {
  font-size: 14px;
  font-weight: 600;
  font-variant-numeric: tabular-nums;
}
.tx .undo {
  background: transparent;
  border: 1px solid var(--line);
  color: var(--muted);
  padding: 4px 10px;
  border-radius: 6px;
  cursor: pointer;
  font-size: 11px;
}
.tx .undo:active {
  background: var(--ink);
  color: white;
  border-color: var(--ink);
}

.empty-mini {
  padding: 20px;
  text-align: center;
  color: var(--muted);
  font-size: 13px;
}

/* Sheet */
.sheet-mask {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.45);
  z-index: 50;
  backdrop-filter: blur(2px);
}
.sheet {
  position: fixed;
  bottom: 0;
  left: 50%;
  transform: translateX(-50%);
  width: 100%;
  max-width: 480px;
  background: var(--surface);
  border-radius: 16px 16px 0 0;
  padding: 8px 12px calc(20px + env(safe-area-inset-bottom));
  z-index: 51;
}
.sheet .grabber {
  width: 36px;
  height: 4px;
  background: var(--line);
  border-radius: 2px;
  margin: 8px auto 14px;
}
.sheet button {
  width: 100%;
  background: transparent;
  border: none;
  padding: 16px;
  font-size: 16px;
  text-align: left;
  color: var(--ink);
  cursor: pointer;
  border-bottom: 1px solid var(--line-soft);
}
.sheet button:last-child {
  border-bottom: none;
}
.sheet button:active {
  background: var(--surface-2);
}
.sheet button.danger {
  color: #d70015;
}
.sheet button.cancel {
  margin-top: 8px;
  background: var(--surface-2);
  border-radius: 12px;
  text-align: center;
  font-weight: 500;
  border-bottom: none;
}

/* transition */
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.2s;
}
.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}
.slide-enter-active,
.slide-leave-active {
  transition: transform 0.25s ease-out;
}
.slide-enter-from,
.slide-leave-to {
  transform: translateX(-50%) translateY(100%);
}
</style>
