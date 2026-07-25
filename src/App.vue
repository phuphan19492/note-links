<script setup>
import { ref, computed, onMounted } from 'vue';
import { createClient } from '@supabase/supabase-js';

const supabaseUrl = import.meta.env.VITE_SUPABASE_URL;
const supabaseKey = import.meta.env.VITE_SUPABASE_ANON_KEY;

const isConfigured = ref(!!supabaseUrl && !!supabaseKey && supabaseUrl !== 'https://your-project.supabase.co');

let supabase = null;
if (isConfigured.value) {
  try {
    supabase = createClient(supabaseUrl, supabaseKey);
  } catch (error) {
    console.error('Lỗi khởi tạo Supabase:', error);
    isConfigured.value = false;
  }
}

if (!isConfigured.value) {
  // Mock client to prevent runtime exceptions prior to setup completion
  supabase = {
    from: () => ({
      select: () => ({ order: () => Promise.resolve({ data: [], error: null }) }),
      insert: () => Promise.resolve({ error: null }),
      update: () => ({ eq: () => Promise.resolve({ error: null }) }),
      delete: () => ({ eq: () => Promise.resolve({ error: null }) })
    })
  };
}

// --- STATE ---
const isAuthenticated = ref(false);
const inputPassword = ref('');
const SECRET_PASSWORD = import.meta.env.VITE_APP_PASSWORD;

const canInputAff = computed(() => {
  return inputPassword.value === 'Godislove19492!!' || (SECRET_PASSWORD && inputPassword.value === SECRET_PASSWORD);
});

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

// Copy & Delete feedback state
const copiedKey = ref('');
const deletingId = ref(null);

// Loading states
const isLoading = ref(false);
const isSaving = ref(false);
const isFetchingTitle = ref(false);
const autoTitleEnabled = ref(true);

// --- AUTO TITLE FETCH ---
const fetchPageTitle = async (url) => {
  if (!url || !autoTitleEnabled.value) return;
  let parsedUrl;
  try {
    parsedUrl = new URL(url);
  } catch {
    return; // not a valid URL yet
  }
  if (!['http:', 'https:'].includes(parsedUrl.protocol)) return;

  isFetchingTitle.value = true;
  try {
    const proxyUrl = `https://api.allorigins.win/get?url=${encodeURIComponent(url)}`;
    const res = await fetch(proxyUrl, { signal: AbortSignal.timeout(8000) });
    if (!res.ok) throw new Error('proxy error');
    const json = await res.json();
    const match = json.contents?.match(/<title[^>]*>([^<]+)<\/title>/i);
    if (match && match[1]) {
      const title = match[1].trim().replace(/&amp;/g, '&').replace(/&lt;/g, '<').replace(/&gt;/g, '>').replace(/&#039;/g, "'").replace(/&quot;/g, '"');
      if (!newNote.value || newNote.value === lastAutoTitle.value) {
        newNote.value = title;
        lastAutoTitle.value = title;
      }
    }
  } catch (e) {
    console.warn('Không thể tự động lấy tiêu đề:', e.message);
  } finally {
    isFetchingTitle.value = false;
  }
};

const lastAutoTitle = ref('');

const onBaseUrlInput = (e) => {
  const url = e.target.value.trim();
  newBaseUrl.value = url;
  fetchPageTitle(url);
};

// --- XÁC THỰC ---
const login = async () => {
  const pass = inputPassword.value;
  if (pass === '04082024' || pass === 'Godislove19492!!' || (SECRET_PASSWORD && pass === SECRET_PASSWORD)) {
    isAuthenticated.value = true;
    localStorage.setItem('family_secret', pass);
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
    const updateData = { note: newNote.value || existing.note };
    if (canInputAff.value) {
      updateData.aff_url = newAffUrl.value || existing.aff_url;
    }
    const { error } = await supabase.from('links')
      .update(updateData)
      .eq('id', existing.id);
    if (error) console.error(error);
  } else {
    const waitingStatus = statusList.value.find(s => s.code === 'waiting');
    const defaultStatusId = waitingStatus ? waitingStatus.id : 1;
    const insertData = { url: newBaseUrl.value, note: newNote.value, status_id: defaultStatusId };
    if (canInputAff.value) {
      insertData.aff_url = newAffUrl.value;
    }
    const { error } = await supabase.from('links')
      .insert([insertData]);
    if (error) console.error(error);
  }
  newBaseUrl.value = '';
  newAffUrl.value = '';
  newNote.value = '';
  lastAutoTitle.value = '';
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
  const updateData = { url: editForm.value.url, note: editForm.value.note, status_id: editForm.value.status_id };
  if (canInputAff.value) {
    updateData.aff_url = editForm.value.aff_url;
  }
  const { error } = await supabase.from('links')
    .update(updateData)
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

const confirmDelete = async (id) => {
  const { error } = await supabase.from('links').delete().eq('id', id);
  if (!error) {
    deletingId.value = null;
    fetchLinks();
  }
};

const copyToClipboard = async (text, key) => {
  try {
    if (navigator && navigator.clipboard) {
      await navigator.clipboard.writeText(text);
      copiedKey.value = key;
      setTimeout(() => {
        if (copiedKey.value === key) {
          copiedKey.value = '';
        }
      }, 1500);
    }
  } catch (err) {
    console.error('Failed to copy: ', err);
  }
};

const formatDate = (dateStr) => {
  if (!dateStr) return '';
  const d = new Date(dateStr);
  return d.toLocaleDateString('vi-VN', { day: '2-digit', month: '2-digit', year: 'numeric', hour: '2-digit', minute: '2-digit' });
};

const truncate = (str, max = 50) => str && str.length > max ? str.slice(0, max) + '…' : str;

onMounted(async () => {
  if (!isConfigured.value) return;
  const savedPass = localStorage.getItem('family_secret');
  if (savedPass === '04082024' || savedPass === 'Godislove19492!!' || (SECRET_PASSWORD && savedPass === SECRET_PASSWORD)) {
    isAuthenticated.value = true;
    inputPassword.value = savedPass;
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

    <!-- CONFIGURATION SETUP REQUIRED SCREEN -->
    <div v-if="!isConfigured" class="login-wrap">
      <div class="login-box glass setup-box">
        <div class="login-icon">
          <svg width="48" height="48" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"><circle cx="12" cy="12" r="3"/><path d="M19.4 15a1.65 1.65 0 0 0 .33 1.82l.06.06a2 2 0 0 1 0 2.83 2 2 0 0 1-2.83 0l-.06-.06a1.65 1.65 0 0 0-1.82-.33 1.65 1.65 0 0 0-1 1.51V21a2 2 0 0 1-2 2 2 2 0 0 1-2-2v-.09A1.65 1.65 0 0 0 9 19.4a1.65 1.65 0 0 0-1.82.33l-.06.06a2 2 0 0 1-2.83 0 2 2 0 0 1 0-2.83l.06-.06a1.65 1.65 0 0 0 .33-1.82 1.65 1.65 0 0 0-1.51-1H3a2 2 0 0 1-2-2 2 2 0 0 1 2-2h.09A1.65 1.65 0 0 0 4.6 9a1.65 1.65 0 0 0-.33-1.82l-.06-.06a2 2 0 0 1 0-2.83 2 2 0 0 1 2.83 0l.06.06a1.65 1.65 0 0 0 1.82.33H9a1.65 1.65 0 0 0 1-1.51V3a2 2 0 0 1 2-2 2 2 0 0 1 2 2v.09a1.65 1.65 0 0 0 1 1.51 1.65 1.65 0 0 0 1.82-.33l.06-.06a2 2 0 0 1 2.83 0 2 2 0 0 1 0 2.83l-.06.06a1.65 1.65 0 0 0-.33 1.82V9a1.65 1.65 0 0 0 1.51 1H21a2 2 0 0 1 2 2 2 2 0 0 1-2 2h-.09a1.65 1.65 0 0 0-1.51 1z"/></svg>
        </div>
        <h1 class="login-title">Cấu hình ứng dụng</h1>
        <p class="login-sub">Chưa tìm thấy cấu hình Supabase hoặc file <code>.env</code></p>
        
        <div class="setup-instructions">
          <p class="setup-text">Vui lòng tạo file <code>.env</code> ở thư mục gốc của dự án và cấu hình các biến môi trường sau:</p>
          <pre class="env-preview"><code>VITE_SUPABASE_URL=https://your-project.supabase.co
VITE_SUPABASE_ANON_KEY=your-anon-key-here
VITE_APP_PASSWORD=your-secret-password-here</code></pre>
          <p class="setup-note">Sau khi lưu file, bạn cần khởi động lại server dev (chạy lại lệnh <code>npm run dev</code>) để tải cấu hình mới.</p>
        </div>
      </div>
    </div>

    <!-- LOGIN SCREEN -->
    <div v-else-if="!isAuthenticated" class="login-wrap">
      <div id="login-box" class="login-box glass">
        <div class="login-icon">
          <svg width="48" height="48" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"><rect x="3" y="11" width="18" height="11" rx="2" ry="2"/><path d="M7 11V7a5 5 0 0 1 10 0v4"/></svg>
        </div>
        <h1 class="login-title">Note Links</h1>
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
          <span class="header-logo">
            <svg width="28" height="28" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><path d="M10 13a5 5 0 0 0 7.54.54l3-3a5 5 0 0 0-7.07-7.07l-1.72 1.71"/><path d="M14 11a5 5 0 0 0-7.54-.54l-3 3a5 5 0 0 0 7.07 7.07l1.71-1.71"/></svg>
          </span>
          <div>
            <h1 class="header-title">Note Links</h1>
            <p class="header-sub">
              <span class="status-pulse-dot"></span>
              {{ links.length }} links đang lưu trữ
            </p>
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
                <input
                  :value="newBaseUrl"
                  @input="onBaseUrlInput"
                  placeholder="https://shopee.vn/..."
                  class="form-input"
                />
              </div>
              <div class="form-group">
                <label>💎 Link AFF (Affiliate)</label>
                <input
                  v-model="newAffUrl"
                  :disabled="!canInputAff"
                  :placeholder="canInputAff ? 'https://shope.ee/...' : 'Không có quyền nhập Link AFF'"
                  class="form-input"
                />
              </div>
              <div class="form-group full">
                <div class="note-label-row">
                  <label>📝 Ghi chú</label>
                  <button
                    type="button"
                    class="auto-toggle-btn"
                    :class="{ active: autoTitleEnabled }"
                    @click="autoTitleEnabled = !autoTitleEnabled"
                    :title="autoTitleEnabled ? 'Tắt tự động lấy tiêu đề' : 'Bật tự động lấy tiêu đề'"
                  >
                    <svg width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5"><path d="M12 2a10 10 0 1 0 10 10"/><path d="M12 6v6l4 2"/></svg>
                    {{ autoTitleEnabled ? 'Auto ON' : 'Auto OFF' }}
                  </button>
                </div>
                <div class="note-input-wrap">
                  <input
                    v-model="newNote"
                    placeholder="Tên sản phẩm, quà cho vợ..."
                    class="form-input note-input"
                    :class="{ loading: isFetchingTitle }"
                  />
                  <div v-if="isFetchingTitle" class="note-spinner-wrap">
                    <div class="note-spinner"></div>
                  </div>
                </div>
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
          <div class="sort-switch">
            <button @click="sortOrder = 'desc'" class="sort-switch-btn" :class="{ active: sortOrder === 'desc' }">
              Mới nhất
            </button>
            <button @click="sortOrder = 'asc'" class="sort-switch-btn" :class="{ active: sortOrder === 'asc' }">
              Cũ nhất
            </button>
          </div>
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
        <div class="empty-icon">
          <svg v-if="searchQuery" width="48" height="48" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"><circle cx="11" cy="11" r="8"/><line x1="21" y1="21" x2="16.65" y2="16.65"/></svg>
          <svg v-else width="48" height="48" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"><polyline points="22 12 16 12 14 15 10 15 8 12 2 12"/><path d="M5.45 5.11L2 12v6a2 2 0 0 0 2 2h16a2 2 0 0 0 2-2v-6l-3.45-6.89A2 2 0 0 0 16.76 4H7.24a2 2 0 0 0-1.79 1.11z"/></svg>
        </div>
        <p>{{ searchQuery ? 'Không tìm thấy kết quả nào.' : 'Chưa có link nào. Hãy thêm link đầu tiên!' }}</p>
      </div>

      <!-- LINKS LIST -->
      <transition-group name="list" tag="div" class="links-list">
        <div v-for="link in filteredLinks" :key="link.id" class="link-card glass" :class="{ editing: editingId === link.id }">

          <!-- INLINE DELETE CONFIRM OVERLAY -->
          <div v-if="deletingId === link.id" class="delete-confirm-overlay">
            <div class="delete-confirm-content">
              <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="var(--red)" stroke-width="2" class="delete-confirm-icon"><path d="M10.29 3.86L1.82 18a2 2 0 0 0 1.71 3h16.94a2 2 0 0 0 1.71-3L13.71 3.86a2 2 0 0 0-3.42 0z"/><line x1="12" y1="9" x2="12" y2="13"/><line x1="12" y1="17" x2="12.01" y2="17"/></svg>
              <div class="delete-confirm-text-wrap">
                <span class="delete-title">Xóa liên kết này?</span>
                <span class="delete-subtitle">Thao tác này không thể hoàn tác.</span>
              </div>
            </div>
            <div class="delete-confirm-actions">
              <button @click="deletingId = null" class="btn-ghost small">Hủy</button>
              <button @click="confirmDelete(link.id)" class="btn-danger small">Xóa</button>
            </div>
          </div>

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
                  :style="{ background: link.statuses.bg_color + '15', color: link.statuses.bg_color, border: '1px solid ' + link.statuses.bg_color + '30' }"
                  title="Bấm để đổi trạng thái"
                >
                  <span class="status-dot" :style="{ background: link.statuses.bg_color }"></span>
                  {{ link.statuses.label }}
                </span>
                <button @click="startEdit(link)" class="icon-btn edit-btn" title="Chỉnh sửa">
                  <svg width="15" height="15" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M11 4H4a2 2 0 0 0-2 2v14a2 2 0 0 0 2 2h14a2 2 0 0 0 2-2v-7"/><path d="M18.5 2.5a2.121 2.121 0 0 1 3 3L12 15l-4 1 1-4 9.5-9.5z"/></svg>
                </button>
                <button @click="deletingId = link.id" class="icon-btn delete-btn" title="Xóa">
                  <svg width="15" height="15" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><polyline points="3 6 5 6 21 6"/><path d="M19 6l-1 14H6L5 6"/><path d="M10 11v6M14 11v6"/><path d="M9 6V4h6v2"/></svg>
                </button>
              </div>
            </div>

            <div class="card-links">
              <div class="link-row">
                <span class="link-label base">BASE</span>
                <a :href="link.url" target="_blank" class="link-url base-url" :title="link.url">{{ link.url }}</a>
                <button @click="copyToClipboard(link.url, `${link.id}-base`)" class="copy-btn" :class="{ copied: copiedKey === `${link.id}-base` }" title="Copy">
                  <svg v-if="copiedKey !== `${link.id}-base`" width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><rect x="9" y="9" width="13" height="13" rx="2"/><path d="M5 15H4a2 2 0 0 1-2-2V4a2 2 0 0 1 2-2h9a2 2 0 0 1 2 2v1"/></svg>
                  <svg v-else width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="var(--green)" stroke-width="2.5"><polyline points="20 6 9 17 4 12"/></svg>
                  <span v-if="copiedKey === `${link.id}-base`" class="tooltip-text">Đã copy!</span>
                </button>
              </div>
              <div class="link-row">
                <span class="link-label aff">AFF</span>
                <a v-if="link.aff_url" :href="link.aff_url" target="_blank" class="link-url aff-url" :title="link.aff_url">{{ link.aff_url }}</a>
                <span v-else class="no-aff">Chưa có link AFF</span>
                <button v-if="link.aff_url" @click="copyToClipboard(link.aff_url, `${link.id}-aff`)" class="copy-btn" :class="{ copied: copiedKey === `${link.id}-aff` }" title="Copy AFF">
                  <svg v-if="copiedKey !== `${link.id}-aff`" width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><rect x="9" y="9" width="13" height="13" rx="2"/><path d="M5 15H4a2 2 0 0 1-2-2V4a2 2 0 0 1 2-2h9a2 2 0 0 1 2 2v1"/></svg>
                  <svg v-else width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="var(--green)" stroke-width="2.5"><polyline points="20 6 9 17 4 12"/></svg>
                  <span v-if="copiedKey === `${link.id}-aff`" class="tooltip-text">Đã copy!</span>
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
                <input
                  v-model="editForm.aff_url"
                  :disabled="!canInputAff"
                  :placeholder="canInputAff ? 'https://...' : 'Không có quyền nhập Link AFF'"
                  class="form-input"
                />
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
@import url('https://fonts.googleapis.com/css2?family=Outfit:wght@300;400;500;600;700;800&display=swap');

*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

:root {
  --bg: #f8fafc;
  --surface: rgba(255, 255, 255, 0.75);
  --surface-hover: rgba(255, 255, 255, 0.9);
  --surface-overlay: rgba(255, 255, 255, 0.85);
  --border: rgba(148, 163, 184, 0.15);
  --border-hover: rgba(148, 163, 184, 0.25);
  --text: #1e293b;
  --text-muted: #64748b;
  --text-dim: #94a3b8;
  --accent: #f97316; /* FPT Orange */
  --accent-glow: rgba(249, 115, 22, 0.15);
  --accent-hover: #ea580c;
  --green: #10b981;
  --amber: #f59e0b;
  --red: #ef4444;
  --blue: #3b82f6;
  --radius: 16px;
  --radius-sm: 10px;
  
  color-scheme: light dark;
}

@media (prefers-color-scheme: dark) {
  :root {
    --bg: #0b0f19;
    --surface: rgba(17, 24, 39, 0.7);
    --surface-hover: rgba(17, 24, 39, 0.85);
    --surface-overlay: rgba(17, 24, 39, 0.9);
    --border: rgba(255, 255, 255, 0.08);
    --border-hover: rgba(255, 255, 255, 0.15);
    --text: #e2e8f0;
    --text-muted: #94a3b8;
    --text-dim: #64748b;
    --accent: #f97316;
    --accent-glow: rgba(249, 115, 22, 0.25);
    --accent-hover: #ff8533;
  }
}

html, body {
  background: var(--bg);
  color: var(--text);
  font-family: 'Outfit', sans-serif;
  min-height: 100vh;
  overflow-x: hidden;
  transition: background-color 0.3s ease, color 0.3s ease;
}

/* BG BLOBS */
.bg-blob {
  position: fixed;
  border-radius: 50%;
  filter: blur(120px);
  pointer-events: none;
  z-index: 0;
  opacity: 0.65;
  transition: all 0.5s ease;
}
.blob-1 { width: 500px; height: 500px; background: radial-gradient(circle, rgba(249, 115, 22, 0.12), transparent); top: -150px; left: -100px; }
.blob-2 { width: 400px; height: 400px; background: radial-gradient(circle, rgba(59, 130, 246, 0.08), transparent); bottom: -100px; right: -100px; }
.blob-3 { width: 300px; height: 300px; background: radial-gradient(circle, rgba(245, 158, 11, 0.08), transparent); top: 50%; left: 50%; transform: translate(-50%,-50%); }

/* GLASSMORPHISM */
.glass {
  background: var(--surface);
  border: 1px solid var(--border);
  backdrop-filter: blur(20px);
  -webkit-backdrop-filter: blur(20px);
  box-shadow: 0 8px 32px 0 rgba(0, 0, 0, 0.03);
  transition: background-color 0.3s, border-color 0.3s, box-shadow 0.3s;
}

.app-root { position: relative; z-index: 1; }

/* ===== LOGIN & SETUP ===== */
.login-wrap {
  min-height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 20px;
}
.login-box {
  width: 100%;
  max-width: 420px;
  border-radius: 24px;
  padding: 48px 40px;
  text-align: center;
  box-shadow: 0 12px 40px 0 rgba(0, 0, 0, 0.05);
  animation: fadeUp 0.6s cubic-bezier(0.16, 1, 0.3, 1) forwards;
}
.login-icon { 
  color: var(--accent); 
  margin-bottom: 20px; 
  display: flex; 
  justify-content: center; 
  animation: float 3s ease-in-out infinite; 
}
.login-title { 
  font-size: 28px; 
  font-weight: 800; 
  margin-bottom: 8px; 
  letter-spacing: -0.02em;
  background: linear-gradient(135deg, var(--accent), #ff9f43); 
  -webkit-background-clip: text; 
  -webkit-text-fill-color: transparent; 
  background-clip: text; 
}
.login-sub { color: var(--text-muted); font-size: 14px; margin-bottom: 32px; }
.login-input {
  width: 100%; padding: 14px 18px;
  background: rgba(0, 0, 0, 0.02); 
  border: 1px solid var(--border);
  border-radius: var(--radius-sm); 
  color: var(--text); 
  font-size: 15px; 
  font-family: inherit;
  margin-bottom: 16px; 
  transition: all 0.2s ease;
  outline: none;
}
@media (prefers-color-scheme: dark) {
  .login-input {
    background: rgba(255, 255, 255, 0.03);
  }
}
.login-input:focus { 
  border-color: var(--accent); 
  background: transparent;
  box-shadow: 0 0 0 4px var(--accent-glow); 
}
.login-btn { width: 100%; gap: 8px; font-size: 15px; padding: 14px; }

/* ===== SETUP SCREEN ===== */
.setup-box {
  max-width: 520px !important;
  text-align: left !important;
}
.setup-instructions {
  margin-top: 24px;
  background: rgba(0, 0, 0, 0.02);
  border-radius: var(--radius-sm);
  padding: 20px;
  border: 1px solid var(--border);
}
@media (prefers-color-scheme: dark) {
  .setup-instructions {
    background: rgba(255, 255, 255, 0.02);
  }
}
.setup-text {
  font-size: 14px;
  color: var(--text);
  margin-bottom: 14px;
  line-height: 1.6;
}
.env-preview {
  background: rgba(0, 0, 0, 0.03);
  border: 1px solid var(--border);
  border-radius: var(--radius-sm);
  padding: 14px;
  font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, monospace;
  font-size: 13px;
  color: var(--accent);
  overflow-x: auto;
  margin-bottom: 14px;
  line-height: 1.6;
}
.setup-note {
  font-size: 13px;
  color: var(--amber);
  line-height: 1.6;
}

/* ===== MAIN APP ===== */
.main-wrap {
  max-width: 1280px;
  margin: 0 auto;
  padding: 32px 24px 80px;
  display: flex;
  flex-direction: column;
  gap: 16px;
  animation: fadeUp 0.6s cubic-bezier(0.16, 1, 0.3, 1) forwards;
}

/* HEADER */
.app-header {
  border-radius: var(--radius);
  padding: 18px 24px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  position: sticky; top: 16px; z-index: 100;
  box-shadow: 0 4px 30px rgba(0,0,0,0.02);
}
.header-left { display: flex; align-items: center; gap: 14px; }
.header-logo { 
  color: var(--accent); 
  display: flex; 
  align-items: center; 
  transition: transform 0.3s ease;
}
.app-header:hover .header-logo {
  transform: rotate(15deg) scale(1.05);
}
.header-title { font-size: 20px; font-weight: 800; color: var(--text); letter-spacing: -0.01em; }
.header-sub { 
  font-size: 13px; 
  color: var(--text-muted); 
  margin-top: 2px; 
  display: flex; 
  align-items: center; 
}

/* Pulse indicator dot */
.status-pulse-dot {
  display: inline-block;
  width: 8px;
  height: 8px;
  background-color: var(--green);
  border-radius: 50%;
  margin-right: 8px;
  position: relative;
}
.status-pulse-dot::after {
  content: '';
  position: absolute;
  top: 0; left: 0;
  width: 100%; height: 100%;
  border-radius: 50%;
  background-color: var(--green);
  animation: pulse 1.8s infinite ease-in-out;
}

/* BUTTONS */
.btn-primary {
  display: inline-flex; align-items: center; justify-content: center;
  gap: 8px; padding: 10px 22px;
  background: linear-gradient(135deg, var(--accent), var(--accent-hover));
  color: white; border: none; border-radius: var(--radius-sm);
  font-family: inherit; font-size: 14px; font-weight: 600;
  cursor: pointer; transition: all 0.2s cubic-bezier(0.16, 1, 0.3, 1);
  box-shadow: 0 4px 16px var(--accent-glow);
}
.btn-primary:hover { 
  transform: translateY(-2px); 
  box-shadow: 0 6px 22px var(--accent-glow); 
  opacity: 0.95;
}
.btn-primary:active { transform: translateY(0); }
.btn-primary:disabled { opacity: 0.5; cursor: not-allowed; transform: none; box-shadow: none; }

.btn-ghost {
  display: inline-flex; align-items: center; gap: 8px;
  padding: 10px 18px;
  background: transparent; border: 1px solid var(--border);
  border-radius: var(--radius-sm); color: var(--text-muted);
  font-family: inherit; font-size: 14px; font-weight: 500;
  cursor: pointer; transition: all 0.2s ease;
}
.btn-ghost:hover { 
  border-color: var(--border-hover); 
  color: var(--text); 
  background: var(--surface-hover); 
}
.btn-ghost.small { padding: 8px 14px; font-size: 13px; }

.btn-danger {
  display: inline-flex; align-items: center; gap: 6px;
  padding: 8px 16px;
  background: var(--red);
  color: white; border: none; border-radius: var(--radius-sm);
  font-family: inherit; font-size: 13px; font-weight: 600;
  cursor: pointer; transition: all 0.2s ease;
  box-shadow: 0 4px 12px rgba(239, 68, 68, 0.2);
}
.btn-danger:hover {
  background: #dc2626;
  transform: translateY(-1px);
  box-shadow: 0 6px 16px rgba(239, 68, 68, 0.3);
}

/* ADD SECTION */
.add-section { border-radius: var(--radius); padding: 18px 24px; }
.add-toggle-btn {
  display: inline-flex; align-items: center; justify-content: center; gap: 8px;
  background: linear-gradient(135deg, rgba(249, 115, 22, 0.08), rgba(234, 88, 12, 0.08));
  border: 1px solid rgba(249, 115, 22, 0.25); border-radius: var(--radius-sm);
  color: var(--accent); font-family: inherit; font-size: 14px; font-weight: 600;
  padding: 12px 18px; cursor: pointer; transition: all 0.2s ease; width: 100%;
}
.add-toggle-btn:hover { 
  background: linear-gradient(135deg, rgba(249, 115, 22, 0.12), rgba(234, 88, 12, 0.12));
  border-color: rgba(249, 115, 22, 0.4);
}
.add-toggle-btn.active { 
  background: linear-gradient(135deg, rgba(249, 115, 22, 0.15), rgba(234, 88, 12, 0.15));
  border-color: var(--accent);
}

.add-form { margin-top: 20px; }
.status-badge-row { margin-bottom: 16px; }
.status-check {
  display: inline-block; padding: 4px 14px;
  border-radius: 20px; font-size: 12px; font-weight: 600;
}
.status-check.has_aff { background: rgba(16,185,129,0.1); color: var(--green); border: 1px solid rgba(16,185,129,0.25); }
.status-check.no_aff { background: rgba(245,158,11,0.1); color: var(--amber); border: 1px solid rgba(245,158,11,0.25); }
.status-check.new { background: rgba(59,130,246,0.1); color: var(--blue); border: 1px solid rgba(59,130,246,0.25); }

.form-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 14px; margin-bottom: 18px; }
.form-group { display: flex; flex-direction: column; gap: 6px; }
.form-group.full { grid-column: 1 / -1; }
.form-group label { font-size: 12px; font-weight: 600; color: var(--text-muted); letter-spacing: 0.02em; }

.note-label-row { display: flex; align-items: center; justify-content: space-between; margin-bottom: 2px; }
.auto-toggle-btn {
  display: inline-flex; align-items: center; gap: 4px;
  padding: 4px 10px; border-radius: 20px;
  font-size: 11px; font-weight: 600; font-family: inherit;
  border: 1px solid var(--border); cursor: pointer;
  background: transparent; color: var(--text-dim);
  transition: all 0.2s ease;
}
.auto-toggle-btn.active { background: rgba(16,185,129,0.12); border-color: rgba(16,185,129,0.3); color: var(--green); }
.auto-toggle-btn:not(.active) { background: rgba(239,68,68,0.06); border-color: rgba(239,68,68,0.2); color: var(--red); }

.note-input-wrap { position: relative; }
.note-input { padding-right: 42px !important; }
.note-input.loading { border-color: rgba(249,115,22,0.4); box-shadow: 0 0 0 3px var(--accent-glow); }
.note-spinner-wrap {
  position: absolute; right: 14px; top: 50%; transform: translateY(-50%);
  display: flex; align-items: center;
}
.note-spinner {
  width: 18px; height: 18px;
  border: 2px solid var(--border);
  border-top-color: var(--accent);
  border-radius: 50%;
  animation: spin 0.7s linear infinite;
}

.form-input {
  padding: 12px 16px;
  background: rgba(0, 0, 0, 0.02); border: 1px solid var(--border);
  border-radius: var(--radius-sm); color: var(--text);
  font-family: inherit; font-size: 14.5px; outline: none;
  transition: all 0.2s ease; width: 100%;
}
@media (prefers-color-scheme: dark) {
  .form-input {
    background: rgba(255, 255, 255, 0.03);
  }
}
.form-input:focus { border-color: var(--accent); background: transparent; box-shadow: 0 0 0 3px var(--accent-glow); }
.form-input::placeholder { color: var(--text-dim); }
.form-input:disabled {
  background: rgba(0, 0, 0, 0.06);
  color: var(--text-dim);
  cursor: not-allowed;
  border-color: var(--border);
}
@media (prefers-color-scheme: dark) {
  .form-input:disabled {
    background: rgba(255, 255, 255, 0.04);
  }
}
.form-select { cursor: pointer; }
.form-select option { background: var(--bg); color: var(--text); }
.save-btn { width: 100%; padding: 14px; font-size: 15px; }

/* TOOLBAR */
.toolbar {
  border-radius: var(--radius);
  padding: 14px 20px;
  display: flex; align-items: center; justify-content: space-between; gap: 16px;
}
.search-wrap {
  flex: 1; max-width: 600px;
  position: relative; display: flex; align-items: center;
}
.search-icon { position: absolute; left: 14px; color: var(--text-muted); pointer-events: none; }
.search-input {
  width: 100%; padding: 11px 16px 11px 40px;
  background: rgba(0, 0, 0, 0.02); border: 1px solid var(--border);
  border-radius: var(--radius-sm); color: var(--text);
  font-family: inherit; font-size: 14.5px; outline: none;
  transition: all 0.2s ease;
}
@media (prefers-color-scheme: dark) {
  .search-input {
    background: rgba(255, 255, 255, 0.03);
  }
}
.search-input:focus { border-color: var(--accent); background: transparent; box-shadow: 0 0 0 3px var(--accent-glow); }
.search-input::placeholder { color: var(--text-dim); }
.clear-search {
  position: absolute; right: 14px; background: none; border: none;
  color: var(--text-dim); cursor: pointer; font-size: 14px; padding: 4px;
  border-radius: 50%; display: flex; align-items: center; justify-content: center;
  transition: all 0.2s;
}
.clear-search:hover { color: var(--text); background: rgba(0,0,0,0.05); }
@media (prefers-color-scheme: dark) {
  .clear-search:hover { background: rgba(255,255,255,0.05); }
}

.sort-wrap { display: flex; align-items: center; gap: 10px; }
.sort-label { font-size: 13px; font-weight: 500; color: var(--text-muted); white-space: nowrap; }

/* Segment switch styling */
.sort-switch {
  display: flex;
  background: rgba(0, 0, 0, 0.03);
  padding: 3px;
  border-radius: 9px;
  border: 1px solid var(--border);
}
@media (prefers-color-scheme: dark) {
  .sort-switch {
    background: rgba(255, 255, 255, 0.03);
  }
}
.sort-switch-btn {
  padding: 6px 14px;
  border: none;
  background: transparent;
  color: var(--text-muted);
  font-family: inherit;
  font-size: 12.5px;
  font-weight: 600;
  border-radius: 6px;
  cursor: pointer;
  transition: all 0.2s cubic-bezier(0.16, 1, 0.3, 1);
}
.sort-switch-btn:hover {
  color: var(--text);
}
.sort-switch-btn.active {
  background: var(--surface);
  color: var(--accent);
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.04);
}

/* RESULTS INFO */
.results-info { font-size: 13.5px; color: var(--text-muted); padding: 0 6px; }
.results-info strong { color: var(--text); }
.results-info em { color: var(--accent); font-style: normal; font-weight: 600; }

/* LOADING */
.loading-wrap {
  display: flex; align-items: center; justify-content: center;
  gap: 14px; padding: 50px; color: var(--text-muted); font-size: 14.5px;
}
.spinner {
  width: 26px; height: 26px;
  border: 2.5px solid var(--border);
  border-top-color: var(--accent);
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
}

/* EMPTY STATE */
.empty-state {
  border-radius: var(--radius);
  padding: 64px 24px;
  text-align: center; color: var(--text-muted);
}
.empty-icon { color: var(--text-dim); margin-bottom: 20px; display: flex; justify-content: center; }

/* LINK CARDS */
.links-list {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(340px, 1fr));
  gap: 14px;
  align-items: start;
}
.link-card {
  position: relative;
  border-radius: var(--radius);
  padding: 20px 24px;
  transition: all 0.25s cubic-bezier(0.16, 1, 0.3, 1);
  overflow: hidden;
}
.link-card:hover { 
  border-color: var(--border-hover); 
  box-shadow: 0 10px 30px rgba(0,0,0,0.05); 
  transform: translateY(-2px); 
}
.link-card.editing { border-color: rgba(249, 115, 22, 0.4); box-shadow: 0 0 0 1px rgba(249, 115, 22, 0.1); }

.card-top {
  display: flex; align-items: flex-start;
  justify-content: space-between; gap: 14px; margin-bottom: 18px;
}
.card-meta { flex: 1; overflow: hidden; }
.card-note { 
  font-size: 16.5px; 
  font-weight: 700; 
  color: var(--text); 
  margin-bottom: 5px; 
  white-space: nowrap; 
  overflow: hidden; 
  text-overflow: ellipsis; 
  letter-spacing: -0.01em;
}
.card-date { font-size: 12px; color: var(--text-dim); }

.card-actions { display: flex; align-items: center; gap: 8px; flex-shrink: 0; }

.status-badge {
  display: inline-flex; align-items: center; gap: 6px;
  padding: 4px 12px; border-radius: 20px;
  font-size: 11.5px; font-weight: 700; letter-spacing: 0.02em;
  white-space: nowrap; transition: all 0.2s;
}
.status-badge.clickable { cursor: pointer; }
.status-badge.clickable:hover { filter: brightness(0.95); transform: translateY(-0.5px); }
.status-dot { width: 6px; height: 6px; border-radius: 50%; flex-shrink: 0; }

.icon-btn {
  width: 34px; height: 34px;
  display: flex; align-items: center; justify-content: center;
  background: transparent; border: 1px solid var(--border);
  border-radius: var(--radius-sm); cursor: pointer; transition: all 0.2s ease;
  color: var(--text-muted);
}
.icon-btn:hover { border-color: var(--border-hover); color: var(--text); background: var(--surface-hover); }
.edit-btn:hover { border-color: rgba(249, 115, 22, 0.4); color: var(--accent); }
.delete-btn:hover { border-color: rgba(239, 68, 68, 0.4); color: var(--red); background: rgba(239, 68, 68, 0.08); }

.card-links { display: flex; flex-direction: column; gap: 10px; }
.link-row {
  display: flex; align-items: center; gap: 10px;
  background: rgba(0, 0, 0, 0.015); border: 1px solid rgba(0, 0, 0, 0.03);
  border-radius: var(--radius-sm); padding: 10px 14px;
  transition: background-color 0.2s;
}
@media (prefers-color-scheme: dark) {
  .link-row {
    background: rgba(255, 255, 255, 0.015);
    border-color: rgba(255, 255, 255, 0.03);
  }
}
.link-row:hover {
  background: rgba(0, 0, 0, 0.03);
}
@media (prefers-color-scheme: dark) {
  .link-row:hover {
    background: rgba(255, 255, 255, 0.03);
  }
}
.link-label {
  font-size: 10px; font-weight: 800; letter-spacing: 0.06em;
  padding: 3px 8px; border-radius: 5px; flex-shrink: 0;
}
.link-label.base { background: rgba(59, 130, 246, 0.12); color: #3b82f6; }
.link-label.aff { background: rgba(16, 185, 129, 0.12); color: #10b981; }

.link-url {
  flex: 1; font-size: 13px; color: var(--text-muted);
  text-decoration: none; white-space: nowrap; overflow: hidden; text-overflow: ellipsis;
  transition: color 0.2s;
}
.link-url:hover { color: var(--text); text-decoration: underline; }
.aff-url { color: #3b82f6 !important; font-weight: 600; }
.no-aff { flex: 1; font-size: 13px; color: var(--amber); font-style: italic; }

/* Copy buttons styling & tooltip */
.copy-btn {
  width: 28px; height: 28px; flex-shrink: 0;
  display: flex; align-items: center; justify-content: center;
  background: transparent; border: 1px solid var(--border);
  border-radius: 6px; cursor: pointer; color: var(--text-dim);
  transition: all 0.2s ease;
  position: relative;
}
.copy-btn:hover { border-color: var(--border-hover); color: var(--text); }
.copy-btn.copied {
  border-color: rgba(16, 185, 129, 0.3);
  background: rgba(16, 185, 129, 0.05);
}
.tooltip-text {
  position: absolute;
  bottom: 125%;
  left: 50%;
  transform: translateX(-50%) translateY(-4px);
  background: #10b981;
  color: white;
  font-size: 10px;
  font-weight: 700;
  padding: 4px 8px;
  border-radius: 5px;
  white-space: nowrap;
  box-shadow: 0 4px 12px rgba(16, 185, 129, 0.25);
  pointer-events: none;
  animation: tooltipFade 0.2s cubic-bezier(0.16, 1, 0.3, 1) forwards;
}
.tooltip-text::after {
  content: '';
  position: absolute;
  top: 100%;
  left: 50%;
  transform: translateX(-50%);
  border: 4px solid transparent;
  border-top-color: #10b981;
}

/* DELETE OVERLAY */
.delete-confirm-overlay {
  position: absolute;
  inset: 0;
  background: var(--surface-overlay);
  backdrop-filter: blur(16px);
  -webkit-backdrop-filter: blur(16px);
  border-radius: var(--radius);
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  gap: 16px;
  padding: 24px;
  z-index: 10;
  animation: overlayIn 0.3s cubic-bezier(0.16, 1, 0.3, 1) forwards;
}
.delete-confirm-content {
  display: flex;
  align-items: center;
  gap: 14px;
  text-align: left;
}
.delete-confirm-icon {
  animation: float 2.5s ease-in-out infinite;
}
.delete-confirm-text-wrap {
  display: flex;
  flex-direction: column;
}
.delete-title {
  font-size: 16px;
  font-weight: 700;
  color: var(--text);
}
.delete-subtitle {
  font-size: 12.5px;
  color: var(--text-muted);
  margin-top: 3px;
}
.delete-confirm-actions {
  display: flex;
  gap: 10px;
}

/* EDIT MODE IN CARD */
.edit-header {
  display: flex; align-items: center; gap: 10px;
  font-size: 15px; font-weight: 700; color: var(--accent);
  margin-bottom: 16px; padding-bottom: 12px;
  border-bottom: 1px solid var(--border);
}
.edit-form { display: flex; flex-direction: column; gap: 12px; margin-bottom: 18px; }
.edit-actions {
  display: flex; justify-content: flex-end; gap: 10px;
  padding-top: 14px; border-top: 1px solid var(--border);
}

/* ANIMATIONS */
@keyframes fadeUp { from { opacity: 0; transform: translateY(16px); } to { opacity: 1; transform: translateY(0); } }
@keyframes float { 0%, 100% { transform: translateY(0); } 50% { transform: translateY(-6px); } }
@keyframes spin { to { transform: rotate(360deg); } }
@keyframes shake {
  0%, 100% { transform: translateX(0); }
  20% { transform: translateX(-8px); }
  40% { transform: translateX(8px); }
  60% { transform: translateX(-5px); }
  80% { transform: translateX(5px); }
}
.shake { animation: shake 0.4s ease; }

@keyframes tooltipFade {
  from { opacity: 0; transform: translateX(-50%) translateY(4px); }
  to { opacity: 1; transform: translateX(-50%) translateY(0); }
}

@keyframes overlayIn {
  from { opacity: 0; backdrop-filter: blur(0px); -webkit-backdrop-filter: blur(0px); }
  to { opacity: 1; backdrop-filter: blur(16px); -webkit-backdrop-filter: blur(16px); }
}

.slide-down-enter-active { animation: slideDown 0.25s ease; }
.slide-down-leave-active { animation: slideDown 0.2s ease reverse; }
@keyframes slideDown { from { opacity: 0; transform: translateY(-8px); } to { opacity: 1; transform: translateY(0); } }

.list-enter-active { animation: fadeUp 0.3s ease; }
.list-leave-active { animation: fadeUp 0.2s ease reverse; }
.list-move { transition: transform 0.3s ease; }

@keyframes pulse {
  0% { transform: scale(1); opacity: 0.8; }
  100% { transform: scale(2.6); opacity: 0; }
}

/* RESPONSIVE */
@media (max-width: 768px) {
  .links-list {
    grid-template-columns: 1fr;
  }
  .search-wrap {
    max-width: 100%;
  }
}

@media (max-width: 500px) {
  .form-grid { grid-template-columns: 1fr; }
  .form-group.full { grid-column: 1; }
  .toolbar { flex-direction: column; align-items: stretch; gap: 12px; }
  .sort-wrap { justify-content: space-between; }
  .main-wrap { padding: 16px 12px 60px; }
  .app-header { top: 8px; padding: 14px 18px; }
  .card-top { flex-wrap: wrap; }
}
</style>