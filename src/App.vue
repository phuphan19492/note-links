<script setup>
import { ref, computed, onMounted } from 'vue';
import { createClient } from '@supabase/supabase-js';

const supabaseUrl = import.meta.env.VITE_SUPABASE_URL;
const supabaseKey = import.meta.env.VITE_SUPABASE_ANON_KEY;
const supabase = createClient(supabaseUrl, supabaseKey);

// --- STATE ---
const isAuthenticated = ref(false);
const inputPassword = ref('');
const SECRET_PASSWORD = import.meta.env.VITE_APP_PASSWORD;

const links = ref([]);
const statusList = ref([]);
const newBaseUrl = ref('');
const newAffUrl = ref('');
const newNote = ref('');
const isAdding = ref(false);

// Search & Sort
const searchQuery = ref('');
const sortOrder = ref('desc'); // 'desc' = mới nhất, 'asc' = cũ nhất

// Edit state
const editingId = ref(null);
const editForm = ref({ url: '', aff_url: '', note: '', status_id: null });

// Loading states
const isLoading = ref(false);
const isSaving = ref(false);

// --- XÁC THỰC ---
const login = async () => {
  if (inputPassword.value === SECRET_PASSWORD) {
    isAuthenticated.value = true;
    localStorage.setItem('family_secret', SECRET_PASSWORD);
    await fetchStatuses();
    fetchLinks();
  } else {
    shakeLogin();
  }
};

const shakeLogin = () => {
  const el = document.getElementById('login-box');
  if (el) { el.classList.add('shake'); setTimeout(() => el.classList.remove('shake'), 500); }
};

const logout = () => {
  isAuthenticated.value = false;
  localStorage.removeItem('family_secret');
  inputPassword.value = '';
  links.value = [];
  statusList.value = [];
};

// --- SUPABASE ---
const fetchStatuses = async () => {
  const { data, error } = await supabase.from('statuses').select('*').order('id', { ascending: true });
  if (!error) statusList.value = data;
};

const fetchLinks = async () => {
  isLoading.value = true;
  const { data, error } = await supabase
    .from('links')
    .select('*, statuses(*)')
    .order('created_at', { ascending: false });
  isLoading.value = false;
  if (error) console.error('Lỗi tải data:', error);
  else links.value = data;
};

// Search + Sort computed
const filteredLinks = computed(() => {
  let result = [...links.value];
  if (searchQuery.value.trim()) {
    const q = searchQuery.value.toLowerCase();
    result = result.filter(l =>
      (l.note && l.note.toLowerCase().includes(q)) ||
      (l.url && l.url.toLowerCase().includes(q)) ||
      (l.aff_url && l.aff_url.toLowerCase().includes(q))
    );
  }
  result.sort((a, b) => {
    const ta = new Date(a.created_at).getTime();
    const tb = new Date(b.created_at).getTime();
    return sortOrder.value === 'desc' ? tb - ta : ta - tb;
  });
  return result;
});

const checkLinkStatus = computed(() => {
  if (!newBaseUrl.value) return null;
  const existing = links.value.find(l => l.url === newBaseUrl.value);
  if (existing) return existing.aff_url ? 'has_aff' : 'no_aff';
  return 'new';
});

const checkLinkLabel = computed(() => {
  const map = { has_aff: 'Đã có link AFF ✓', no_aff: 'Chưa có link AFF ⚠', new: 'Link mới ✦' };
  return checkLinkStatus.value ? map[checkLinkStatus.value] : 'Chưa nhập';
});

const addLink = async () => {
  if (!newBaseUrl.value) return;
  isSaving.value = true;
  const existing = links.value.find(l => l.url === newBaseUrl.value);
  if (existing) {
    const { error } = await supabase.from('links')
      .update({ aff_url: newAffUrl.value || existing.aff_url, note: newNote.value || existing.note })
      .eq('id', existing.id);
    if (error) console.error(error);
  } else {
    const waitingStatus = statusList.value.find(s => s.code === 'waiting');
    const defaultStatusId = waitingStatus ? waitingStatus.id : 1;
    const { error } = await supabase.from('links')
      .insert([{ url: newBaseUrl.value, aff_url: newAffUrl.value, note: newNote.value, status_id: defaultStatusId }]);
    if (error) console.error(error);
  }
  newBaseUrl.value = '';
  newAffUrl.value = '';
  newNote.value = '';
  isAdding.value = false;
  isSaving.value = false;
  fetchLinks();
};

// Edit inline
const startEdit = (link) => {
  editingId.value = link.id;
  editForm.value = { url: link.url, aff_url: link.aff_url || '', note: link.note || '', status_id: link.status_id };
};

const cancelEdit = () => { editingId.value = null; };

const saveEdit = async (id) => {
  isSaving.value = true;
  const { error } = await supabase.from('links')
    .update({ url: editForm.value.url, aff_url: editForm.value.aff_url, note: editForm.value.note, status_id: editForm.value.status_id })
    .eq('id', id);
  isSaving.value = false;
  if (!error) { editingId.value = null; fetchLinks(); }
};

const updateStatus = async (link) => {
  if (statusList.value.length === 0) return;
  const currentIndex = statusList.value.findIndex(s => s.id === link.status_id);
  const nextIndex = (currentIndex + 1) % statusList.value.length;
  const nextStatusId = statusList.value[nextIndex].id;
  const { error } = await supabase.from('links').update({ status_id: nextStatusId }).eq('id', link.id);
  if (!error) fetchLinks();
};

const deleteLink = async (id) => {
  if (!confirm('Bạn có chắc muốn xóa link này không?')) return;
  const { error } = await supabase.from('links').delete().eq('id', id);
  if (!error) fetchLinks();
};

const formatDate = (dateStr) => {
  if (!dateStr) return '';
  const d = new Date(dateStr);
  return d.toLocaleDateString('vi-VN', { day: '2-digit', month: '2-digit', year: 'numeric', hour: '2-digit', minute: '2-digit' });
};

const truncate = (str, max = 50) => str && str.length > max ? str.slice(0, max) + '…' : str;

onMounted(async () => {
  const savedPass = localStorage.getItem('family_secret');
  if (savedPass === SECRET_PASSWORD) {
    isAuthenticated.value = true;
    await fetchStatuses();
    fetchLinks();
  }
});
</script>

<template>
  <div class="app-root">
    <!-- BG Gradient Blobs -->
    <div class="bg-blob blob-1"></div>
    <div class="bg-blob blob-2"></div>
    <div class="bg-blob blob-3"></div>

    <!-- LOGIN SCREEN -->
    <div v-if="!isAuthenticated" class="login-wrap">
      <div id="login-box" class="login-box glass">
        <div class="login-icon">🔐</div>
        <h1 class="login-title">Kho Affiliate</h1>
        <p class="login-sub">Nội bộ gia đình · Nhập mật khẩu để tiếp tục</p>
        <input
          id="login-password"
          type="password"
          v-model="inputPassword"
          @keyup.enter="login"
          placeholder="Mật khẩu..."
          class="login-input"
          autocomplete="current-password"
        />
        <button @click="login" class="btn-primary login-btn">
          <span>Mở Khóa</span>
          <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M5 12h14M12 5l7 7-7 7"/></svg>
        </button>
      </div>
    </div>

    <!-- MAIN APP -->
    <div v-else class="main-wrap">
      <!-- HEADER -->
      <header class="app-header glass">
        <div class="header-left">
          <span class="header-logo">🛍️</span>
          <div>
            <h1 class="header-title">Kho Link Affiliate</h1>
            <p class="header-sub">{{ links.length }} links đang lưu trữ</p>
          </div>
        </div>
        <button @click="logout" class="btn-ghost">
          <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M9 21H5a2 2 0 0 1-2-2V5a2 2 0 0 1 2-2h4M16 17l5-5-5-5M21 12H9"/></svg>
          Đăng xuất
        </button>
      </header>

      <!-- ADD LINK SECTION -->
      <div class="add-section glass">
        <button @click="isAdding = !isAdding" class="add-toggle-btn" :class="{ active: isAdding }">
          <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5">
            <line x1="12" y1="5" x2="12" y2="19"/><line x1="5" y1="12" x2="19" y2="12"/>
          </svg>
          {{ isAdding ? 'Thu gọn' : 'Thêm link mới' }}
        </button>

        <transition name="slide-down">
          <div v-if="isAdding" class="add-form">
            <div class="status-badge-row">
              <span class="status-check" :class="checkLinkStatus">{{ checkLinkLabel }}</span>
            </div>
            <div class="form-grid">
              <div class="form-group">
                <label>🔗 Link Base (Link gốc)</label>
                <input v-model="newBaseUrl" placeholder="https://shopee.vn/..." class="form-input" />
              </div>
              <div class="form-group">
                <label>💎 Link AFF (Affiliate)</label>
                <input v-model="newAffUrl" placeholder="https://shope.ee/..." class="form-input" />
              </div>
              <div class="form-group full">
                <label>📝 Ghi chú</label>
                <input v-model="newNote" placeholder="Tên sản phẩm, quà cho vợ..." class="form-input" />
              </div>
            </div>
            <button @click="addLink" class="btn-primary save-btn" :disabled="isSaving || !newBaseUrl">
              <span v-if="isSaving">Đang lưu...</span>
              <span v-else>💾 Lưu / Cập nhật Link</span>
            </button>
          </div>
        </transition>
      </div>

      <!-- TOOLBAR: SEARCH + SORT -->
      <div class="toolbar glass">
        <div class="search-wrap">
          <svg class="search-icon" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><circle cx="11" cy="11" r="8"/><line x1="21" y1="21" x2="16.65" y2="16.65"/></svg>
          <input v-model="searchQuery" placeholder="Tìm kiếm sản phẩm, link..." class="search-input" id="search-input" />
          <button v-if="searchQuery" @click="searchQuery=''" class="clear-search">✕</button>
        </div>
        <div class="sort-wrap">
          <span class="sort-label">Sắp xếp:</span>
          <button @click="sortOrder = 'desc'" class="sort-btn" :class="{ active: sortOrder === 'desc' }">
            <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><polyline points="18 15 12 9 6 15"/></svg>
            Mới nhất
          </button>
          <button @click="sortOrder = 'asc'" class="sort-btn" :class="{ active: sortOrder === 'asc' }">
            <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><polyline points="6 9 12 15 18 9"/></svg>
            Cũ nhất
          </button>
        </div>
      </div>

      <!-- RESULTS COUNT -->
      <div class="results-info" v-if="searchQuery">
        <span>Tìm thấy <strong>{{ filteredLinks.length }}</strong> kết quả cho "<em>{{ searchQuery }}</em>"</span>
      </div>

      <!-- LOADING -->
      <div v-if="isLoading" class="loading-wrap">
        <div class="spinner"></div>
        <span>Đang tải dữ liệu...</span>
      </div>

      <!-- EMPTY STATE -->
      <div v-else-if="filteredLinks.length === 0 && !isLoading" class="empty-state glass">
        <div class="empty-icon">{{ searchQuery ? '🔍' : '📭' }}</div>
        <p>{{ searchQuery ? 'Không tìm thấy kết quả nào.' : 'Chưa có link nào. Hãy thêm link đầu tiên!' }}</p>
      </div>

      <!-- LINKS LIST -->
      <transition-group name="list" tag="div" class="links-list">
        <div v-for="link in filteredLinks" :key="link.id" class="link-card glass" :class="{ editing: editingId === link.id }">

          <!-- VIEW MODE -->
          <template v-if="editingId !== link.id">
            <div class="card-top">
              <div class="card-meta">
                <h3 class="card-note">{{ link.note || 'Không có ghi chú' }}</h3>
                <span class="card-date">{{ formatDate(link.created_at) }}</span>
              </div>
              <div class="card-actions">
                <span
                  v-if="link.statuses"
                  @click="updateStatus(link)"
                  class="status-badge clickable"
                  :style="{ background: link.statuses.bg_color + '22', color: link.statuses.bg_color, border: '1px solid ' + link.statuses.bg_color + '55' }"
                  title="Bấm để đổi trạng thái"
                >
                  <span class="status-dot" :style="{ background: link.statuses.bg_color }"></span>
                  {{ link.statuses.label }}
                </span>
                <button @click="startEdit(link)" class="icon-btn edit-btn" title="Chỉnh sửa">
                  <svg width="15" height="15" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M11 4H4a2 2 0 0 0-2 2v14a2 2 0 0 0 2 2h14a2 2 0 0 0 2-2v-7"/><path d="M18.5 2.5a2.121 2.121 0 0 1 3 3L12 15l-4 1 1-4 9.5-9.5z"/></svg>
                </button>
                <button @click="deleteLink(link.id)" class="icon-btn delete-btn" title="Xóa">
                  <svg width="15" height="15" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><polyline points="3 6 5 6 21 6"/><path d="M19 6l-1 14H6L5 6"/><path d="M10 11v6M14 11v6"/><path d="M9 6V4h6v2"/></svg>
                </button>
              </div>
            </div>

            <div class="card-links">
              <div class="link-row">
                <span class="link-label base">BASE</span>
                <a :href="link.url" target="_blank" class="link-url base-url" :title="link.url">{{ link.url }}</a>
                <button @click="navigator.clipboard.writeText(link.url)" class="copy-btn" title="Copy">
                  <svg width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><rect x="9" y="9" width="13" height="13" rx="2"/><path d="M5 15H4a2 2 0 0 1-2-2V4a2 2 0 0 1 2-2h9a2 2 0 0 1 2 2v1"/></svg>
                </button>
              </div>
              <div class="link-row">
                <span class="link-label aff">AFF</span>
                <a v-if="link.aff_url" :href="link.aff_url" target="_blank" class="link-url aff-url" :title="link.aff_url">{{ link.aff_url }}</a>
                <span v-else class="no-aff">Chưa có link AFF</span>
                <button v-if="link.aff_url" @click="navigator.clipboard.writeText(link.aff_url)" class="copy-btn" title="Copy AFF">
                  <svg width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><rect x="9" y="9" width="13" height="13" rx="2"/><path d="M5 15H4a2 2 0 0 1-2-2V4a2 2 0 0 1 2-2h9a2 2 0 0 1 2 2v1"/></svg>
                </button>
              </div>
            </div>
          </template>

          <!-- EDIT MODE -->
          <template v-else>
            <div class="edit-header">
              <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="#a78bfa" stroke-width="2"><path d="M11 4H4a2 2 0 0 0-2 2v14a2 2 0 0 0 2 2h14a2 2 0 0 0 2-2v-7"/><path d="M18.5 2.5a2.121 2.121 0 0 1 3 3L12 15l-4 1 1-4 9.5-9.5z"/></svg>
              <span>Chỉnh sửa link</span>
            </div>
            <div class="edit-form">
              <div class="form-group">
                <label>📝 Ghi chú</label>
                <input v-model="editForm.note" class="form-input" placeholder="Tên sản phẩm..." />
              </div>
              <div class="form-group">
                <label>🔗 Link Base</label>
                <input v-model="editForm.url" class="form-input" placeholder="https://..." />
              </div>
              <div class="form-group">
                <label>💎 Link AFF</label>
                <input v-model="editForm.aff_url" class="form-input" placeholder="https://..." />
              </div>
              <div class="form-group">
                <label>📊 Trạng thái</label>
                <select v-model="editForm.status_id" class="form-input form-select">
                  <option v-for="s in statusList" :key="s.id" :value="s.id">{{ s.label }}</option>
                </select>
              </div>
            </div>
            <div class="edit-actions">
              <button @click="cancelEdit" class="btn-ghost small">Huỷ</button>
              <button @click="saveEdit(link.id)" class="btn-primary small" :disabled="isSaving">
                {{ isSaving ? 'Đang lưu...' : '✓ Lưu thay đổi' }}
              </button>
            </div>
          </template>

        </div>
      </transition-group>
    </div>
  </div>
</template>

<style>
@import url('https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600;700&display=swap');

*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

:root {
  --bg: #0d0e14;
  --surface: rgba(255,255,255,0.04);
  --surface-hover: rgba(255,255,255,0.07);
  --border: rgba(255,255,255,0.08);
  --border-hover: rgba(255,255,255,0.15);
  --text: #f0f0f5;
  --text-muted: #888;
  --text-dim: #555;
  --accent: #7c3aed;
  --accent-glow: rgba(124,58,237,0.35);
  --green: #10b981;
  --amber: #f59e0b;
  --red: #ef4444;
  --blue: #3b82f6;
  --radius: 14px;
  --radius-sm: 8px;
}

html, body {
  background: var(--bg);
  color: var(--text);
  font-family: 'Inter', sans-serif;
  min-height: 100vh;
  overflow-x: hidden;
}

/* BG BLOBS */
.bg-blob {
  position: fixed;
  border-radius: 50%;
  filter: blur(80px);
  pointer-events: none;
  z-index: 0;
  opacity: 0.5;
}
.blob-1 { width: 500px; height: 500px; background: radial-gradient(circle, #7c3aed44, transparent); top: -150px; left: -100px; }
.blob-2 { width: 400px; height: 400px; background: radial-gradient(circle, #3b82f633, transparent); bottom: -100px; right: -100px; }
.blob-3 { width: 300px; height: 300px; background: radial-gradient(circle, #10b98122, transparent); top: 50%; left: 50%; transform: translate(-50%,-50%); }

/* GLASS */
.glass {
  background: var(--surface);
  border: 1px solid var(--border);
  backdrop-filter: blur(20px);
  -webkit-backdrop-filter: blur(20px);
}

.app-root { position: relative; z-index: 1; }

/* ===== LOGIN ===== */
.login-wrap {
  min-height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 20px;
}
.login-box {
  width: 100%;
  max-width: 400px;
  border-radius: 24px;
  padding: 48px 40px;
  text-align: center;
  animation: fadeUp 0.5s ease;
}
.login-icon { font-size: 52px; margin-bottom: 16px; display: block; animation: float 3s ease-in-out infinite; }
.login-title { font-size: 26px; font-weight: 700; margin-bottom: 6px; background: linear-gradient(135deg, #a78bfa, #60a5fa); -webkit-background-clip: text; -webkit-text-fill-color: transparent; background-clip: text; }
.login-sub { color: var(--text-muted); font-size: 13px; margin-bottom: 28px; }
.login-input {
  width: 100%; padding: 14px 18px;
  background: rgba(255,255,255,0.06); border: 1px solid var(--border);
  border-radius: var(--radius-sm); color: var(--text); font-size: 15px; font-family: inherit;
  margin-bottom: 14px; transition: border 0.2s, box-shadow 0.2s;
  outline: none;
}
.login-input:focus { border-color: var(--accent); box-shadow: 0 0 0 3px var(--accent-glow); }
.login-btn { width: 100%; gap: 8px; font-size: 15px; padding: 14px; }

/* ===== MAIN ===== */
.main-wrap {
  max-width: 700px;
  margin: 0 auto;
  padding: 24px 16px 60px;
  display: flex;
  flex-direction: column;
  gap: 14px;
}

/* HEADER */
.app-header {
  border-radius: var(--radius);
  padding: 16px 20px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  position: sticky; top: 16px; z-index: 100;
}
.header-left { display: flex; align-items: center; gap: 12px; }
.header-logo { font-size: 28px; }
.header-title { font-size: 18px; font-weight: 700; color: var(--text); }
.header-sub { font-size: 12px; color: var(--text-muted); margin-top: 1px; }

/* BUTTONS */
.btn-primary {
  display: inline-flex; align-items: center; justify-content: center;
  gap: 8px; padding: 10px 20px;
  background: linear-gradient(135deg, #7c3aed, #5b21b6);
  color: white; border: none; border-radius: var(--radius-sm);
  font-family: inherit; font-size: 14px; font-weight: 600;
  cursor: pointer; transition: transform 0.15s, box-shadow 0.15s, opacity 0.15s;
  box-shadow: 0 4px 20px var(--accent-glow);
}
.btn-primary:hover { transform: translateY(-1px); box-shadow: 0 6px 28px var(--accent-glow); }
.btn-primary:active { transform: translateY(0); }
.btn-primary:disabled { opacity: 0.5; cursor: not-allowed; transform: none; }

.btn-ghost {
  display: inline-flex; align-items: center; gap: 6px;
  padding: 8px 14px;
  background: transparent; border: 1px solid var(--border);
  border-radius: var(--radius-sm); color: var(--text-muted);
  font-family: inherit; font-size: 13px; font-weight: 500;
  cursor: pointer; transition: border 0.2s, color 0.2s, background 0.2s;
}
.btn-ghost:hover { border-color: var(--border-hover); color: var(--text); background: var(--surface-hover); }
.btn-ghost.small { padding: 7px 14px; font-size: 13px; }

/* ADD SECTION */
.add-section { border-radius: var(--radius); padding: 16px 20px; }
.add-toggle-btn {
  display: inline-flex; align-items: center; gap: 8px;
  background: linear-gradient(135deg, rgba(124,58,237,0.2), rgba(91,33,182,0.2));
  border: 1px solid rgba(124,58,237,0.4); border-radius: var(--radius-sm);
  color: #a78bfa; font-family: inherit; font-size: 14px; font-weight: 600;
  padding: 10px 18px; cursor: pointer; transition: all 0.2s; width: 100%;
}
.add-toggle-btn:hover { background: linear-gradient(135deg, rgba(124,58,237,0.3), rgba(91,33,182,0.3)); }
.add-toggle-btn.active { background: linear-gradient(135deg, rgba(124,58,237,0.35), rgba(91,33,182,0.35)); }

.add-form { margin-top: 16px; }
.status-badge-row { margin-bottom: 14px; }
.status-check {
  display: inline-block; padding: 4px 12px;
  border-radius: 20px; font-size: 12px; font-weight: 600;
}
.status-check.has_aff { background: rgba(16,185,129,0.15); color: var(--green); border: 1px solid rgba(16,185,129,0.3); }
.status-check.no_aff { background: rgba(245,158,11,0.15); color: var(--amber); border: 1px solid rgba(245,158,11,0.3); }
.status-check.new { background: rgba(59,130,246,0.15); color: var(--blue); border: 1px solid rgba(59,130,246,0.3); }

.form-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 10px; margin-bottom: 14px; }
.form-group { display: flex; flex-direction: column; gap: 6px; }
.form-group.full { grid-column: 1 / -1; }
.form-group label { font-size: 12px; font-weight: 500; color: var(--text-muted); text-transform: uppercase; letter-spacing: 0.05em; }
.form-input {
  padding: 11px 14px;
  background: rgba(255,255,255,0.05); border: 1px solid var(--border);
  border-radius: var(--radius-sm); color: var(--text);
  font-family: inherit; font-size: 14px; outline: none;
  transition: border 0.2s, box-shadow 0.2s; width: 100%;
}
.form-input:focus { border-color: var(--accent); box-shadow: 0 0 0 3px var(--accent-glow); }
.form-input::placeholder { color: var(--text-dim); }
.form-select { cursor: pointer; }
.form-select option { background: #1a1b23; color: var(--text); }
.save-btn { width: 100%; padding: 13px; font-size: 15px; }

/* TOOLBAR */
.toolbar {
  border-radius: var(--radius);
  padding: 12px 16px;
  display: flex; align-items: center; gap: 12px;
  flex-wrap: wrap;
}
.search-wrap {
  flex: 1; min-width: 160px;
  position: relative; display: flex; align-items: center;
}
.search-icon { position: absolute; left: 12px; color: var(--text-muted); pointer-events: none; }
.search-input {
  width: 100%; padding: 10px 36px;
  background: rgba(255,255,255,0.05); border: 1px solid var(--border);
  border-radius: var(--radius-sm); color: var(--text);
  font-family: inherit; font-size: 14px; outline: none;
  transition: border 0.2s, box-shadow 0.2s;
}
.search-input:focus { border-color: var(--accent); box-shadow: 0 0 0 3px var(--accent-glow); }
.search-input::placeholder { color: var(--text-dim); }
.clear-search {
  position: absolute; right: 10px; background: none; border: none;
  color: var(--text-muted); cursor: pointer; font-size: 14px; padding: 2px 4px;
  border-radius: 4px; transition: color 0.2s;
}
.clear-search:hover { color: var(--text); }

.sort-wrap { display: flex; align-items: center; gap: 6px; }
.sort-label { font-size: 12px; color: var(--text-muted); white-space: nowrap; }
.sort-btn {
  display: inline-flex; align-items: center; gap: 4px;
  padding: 7px 12px; background: transparent; border: 1px solid var(--border);
  border-radius: var(--radius-sm); color: var(--text-muted);
  font-family: inherit; font-size: 12px; font-weight: 500;
  cursor: pointer; transition: all 0.2s; white-space: nowrap;
}
.sort-btn:hover { border-color: var(--border-hover); color: var(--text); }
.sort-btn.active { background: rgba(124,58,237,0.2); border-color: rgba(124,58,237,0.5); color: #a78bfa; }

/* RESULTS INFO */
.results-info { font-size: 13px; color: var(--text-muted); padding: 0 4px; }
.results-info strong { color: var(--text); }
.results-info em { color: #a78bfa; font-style: normal; }

/* LOADING */
.loading-wrap {
  display: flex; align-items: center; justify-content: center;
  gap: 12px; padding: 40px; color: var(--text-muted); font-size: 14px;
}
.spinner {
  width: 24px; height: 24px;
  border: 2px solid var(--border);
  border-top-color: var(--accent);
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
}

/* EMPTY STATE */
.empty-state {
  border-radius: var(--radius);
  padding: 60px 20px;
  text-align: center; color: var(--text-muted);
}
.empty-icon { font-size: 40px; margin-bottom: 12px; }

/* LINK CARDS */
.links-list { display: flex; flex-direction: column; gap: 10px; }
.link-card {
  border-radius: var(--radius);
  padding: 18px 20px;
  transition: border-color 0.2s, box-shadow 0.2s, transform 0.2s;
}
.link-card:hover { border-color: var(--border-hover); box-shadow: 0 4px 24px rgba(0,0,0,0.3); transform: translateY(-1px); }
.link-card.editing { border-color: rgba(124,58,237,0.5); box-shadow: 0 0 0 1px rgba(124,58,237,0.2); }

.card-top {
  display: flex; align-items: flex-start;
  justify-content: space-between; gap: 10px; margin-bottom: 14px;
}
.card-meta { flex: 1; overflow: hidden; }
.card-note { font-size: 15px; font-weight: 600; color: var(--text); margin-bottom: 4px; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.card-date { font-size: 11px; color: var(--text-dim); }

.card-actions { display: flex; align-items: center; gap: 6px; flex-shrink: 0; }

.status-badge {
  display: inline-flex; align-items: center; gap: 5px;
  padding: 4px 10px; border-radius: 20px;
  font-size: 11px; font-weight: 700; letter-spacing: 0.04em;
  white-space: nowrap; transition: opacity 0.2s;
}
.status-badge.clickable { cursor: pointer; }
.status-badge.clickable:hover { opacity: 0.75; }
.status-dot { width: 6px; height: 6px; border-radius: 50%; flex-shrink: 0; }

.icon-btn {
  width: 32px; height: 32px;
  display: flex; align-items: center; justify-content: center;
  background: transparent; border: 1px solid var(--border);
  border-radius: var(--radius-sm); cursor: pointer; transition: all 0.2s;
  color: var(--text-muted);
}
.icon-btn:hover { border-color: var(--border-hover); color: var(--text); background: var(--surface-hover); }
.edit-btn:hover { border-color: rgba(124,58,237,0.5); color: #a78bfa; }
.delete-btn:hover { border-color: rgba(239,68,68,0.5); color: var(--red); background: rgba(239,68,68,0.08); }

.card-links { display: flex; flex-direction: column; gap: 8px; }
.link-row {
  display: flex; align-items: center; gap: 8px;
  background: rgba(255,255,255,0.02); border: 1px solid rgba(255,255,255,0.05);
  border-radius: var(--radius-sm); padding: 8px 10px;
}
.link-label {
  font-size: 10px; font-weight: 800; letter-spacing: 0.08em;
  padding: 2px 7px; border-radius: 4px; flex-shrink: 0;
}
.link-label.base { background: rgba(59,130,246,0.15); color: #60a5fa; }
.link-label.aff { background: rgba(16,185,129,0.15); color: #34d399; }
.link-url {
  flex: 1; font-size: 12px; color: var(--text-muted);
  text-decoration: none; white-space: nowrap; overflow: hidden; text-overflow: ellipsis;
  transition: color 0.2s;
}
.link-url:hover { color: var(--text); }
.aff-url { color: #60a5fa !important; font-weight: 500; }
.no-aff { flex: 1; font-size: 12px; color: var(--amber); font-style: italic; }

.copy-btn {
  width: 26px; height: 26px; flex-shrink: 0;
  display: flex; align-items: center; justify-content: center;
  background: transparent; border: 1px solid var(--border);
  border-radius: 5px; cursor: pointer; color: var(--text-dim);
  transition: all 0.2s;
}
.copy-btn:hover { border-color: var(--border-hover); color: var(--text); }

/* EDIT MODE */
.edit-header {
  display: flex; align-items: center; gap: 8px;
  font-size: 14px; font-weight: 600; color: #a78bfa;
  margin-bottom: 14px; padding-bottom: 12px;
  border-bottom: 1px solid var(--border);
}
.edit-form { display: flex; flex-direction: column; gap: 10px; margin-bottom: 14px; }
.edit-actions {
  display: flex; justify-content: flex-end; gap: 8px;
  padding-top: 12px; border-top: 1px solid var(--border);
}

/* ANIMATIONS */
@keyframes fadeUp { from { opacity: 0; transform: translateY(20px); } to { opacity: 1; transform: translateY(0); } }
@keyframes float { 0%, 100% { transform: translateY(0); } 50% { transform: translateY(-8px); } }
@keyframes spin { to { transform: rotate(360deg); } }
@keyframes shake {
  0%, 100% { transform: translateX(0); }
  20% { transform: translateX(-10px); }
  40% { transform: translateX(10px); }
  60% { transform: translateX(-6px); }
  80% { transform: translateX(6px); }
}
.shake { animation: shake 0.4s ease; }

.slide-down-enter-active { animation: slideDown 0.25s ease; }
.slide-down-leave-active { animation: slideDown 0.2s ease reverse; }
@keyframes slideDown { from { opacity: 0; transform: translateY(-10px); } to { opacity: 1; transform: translateY(0); } }

.list-enter-active { animation: fadeUp 0.3s ease; }
.list-leave-active { animation: fadeUp 0.2s ease reverse; }
.list-move { transition: transform 0.3s ease; }

/* RESPONSIVE */
@media (max-width: 500px) {
  .form-grid { grid-template-columns: 1fr; }
  .form-group.full { grid-column: 1; }
  .toolbar { flex-direction: column; align-items: stretch; }
  .sort-wrap { justify-content: center; }
  .main-wrap { padding: 12px 10px 60px; }
  .app-header { top: 8px; }
  .card-top { flex-wrap: wrap; }
}
</style>