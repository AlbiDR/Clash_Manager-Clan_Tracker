<script setup lang="ts">
import { ref, onMounted, onUnmounted, nextTick } from 'vue'
import { useRoute } from 'vue-router'
import { getLeaderboard, dismissRecruits } from '../api/gasClient'
import type { Recruit } from '../types'
import RecruitCard from '../components/RecruitCard.vue'
import PullToRefresh from '../components/PullToRefresh.vue'
import EmptyState from '../components/EmptyState.vue'
import ErrorState from '../components/ErrorState.vue'
import Icon from '../components/Icon.vue'

const route = useRoute()

const recruits = ref<Recruit[]>([])
const loading = ref(true)
const error = ref<string | null>(null)
const selectedIds = ref<Set<string>>(new Set())
const expandedIds = ref<Set<string>>(new Set())
const dismissing = ref(false)
const selectionMode = ref(false)
const isScrolled = ref(false)

function handleScroll() {
  isScrolled.value = window.scrollY > 10
}

async function loadData() {
  loading.value = true
  error.value = null
  
  try {
    const response = await getLeaderboard()
    // Check if response exists and has success status
    // The API might return LegacyApiResponse or ApiResponse, gasClient normalizes it usually.
    // Based on previous files, we check .success (boolean) or .status === 'success'
    // Let's cover both or checks usage in gasClient. 
    // gasClient getLeaderboard returns Promise<ApiResponse<WebAppData>>
    // where ApiResponse has { success: boolean, data?: T, error?: ... }
    if (response && response.success && response.data) {
      recruits.value = response.data.hh || []
      
      // Handle Pin/Scroll-to
      const pinId = route.query.pin as string
      if (pinId && recruits.value.some(r => r.id === pinId)) {
        expandedIds.value.add(pinId)
        nextTick(() => {
          const el = document.getElementById(`recruit-${pinId}`)
          if (el) {
            el.scrollIntoView({ behavior: 'smooth', block: 'center' })
          }
        })
      }
    } else {
      error.value = response?.error?.message || 'Failed to load recruits'
    }
  } catch (e) {
    error.value = e instanceof Error ? e.message : 'Network error'
  } finally {
    loading.value = false
  }
}

function toggleExpand(id: string) {
  if (expandedIds.value.has(id)) {
    expandedIds.value.delete(id)
  } else {
    expandedIds.value.add(id)
  }
  // Force reactivity
  expandedIds.value = new Set(expandedIds.value)
}

function toggleSelect(id: string) {
  if (selectedIds.value.has(id)) {
    selectedIds.value.delete(id)
  } else {
    selectedIds.value.add(id)
    if (!selectionMode.value) selectionMode.value = true
  }
  selectedIds.value = new Set(selectedIds.value)
  
  if (selectedIds.value.size === 0) {
    selectionMode.value = false
  }
}

async function dismissRecruitsFn() {
  if (selectedIds.value.size === 0) return
  
  dismissing.value = true
  try {
    const idsToDismiss = Array.from(selectedIds.value)
    const response = await dismissRecruits(idsToDismiss)
    
    if (response.status === 'success') {
      // Remove from local list
      recruits.value = recruits.value.filter(r => !selectedIds.value.has(r.id))
      selectedIds.value.clear()
      selectionMode.value = false
    } else {
      alert('Failed to dismiss: ' + (response.error?.message || 'Unknown error'))
    }
  } catch (e) {
    alert('Error dismissing recruits')
  } finally {
    dismissing.value = false
  }
}

onMounted(() => {
  loadData()
  window.addEventListener('scroll', handleScroll)
})

onUnmounted(() => {
  window.removeEventListener('scroll', handleScroll)
})
</script>

<template>
  <div class="recruiter-view">
    <PullToRefresh @refresh="loadData" />
    
    <!-- Functional Top App Bar -->
    <header class="top-app-bar" :class="{ 'scrolled': isScrolled }">
      <div class="toolbar-content">
        <h1 class="page-title" v-if="!selectionMode">Recruiter</h1>
        <div class="selection-header" v-else>
          <button class="icon-btn" @click="selectionMode = false">
            <Icon name="close" />
          </button>
          <span class="selection-count">{{ selectedIds.size }} selected</span>
        </div>
        
        <div class="actions">
          <button 
            v-if="!selectionMode"
            class="icon-btn" 
            @click="selectionMode = true"
            v-tooltip="'Select'"
          >
            <Icon name="check" />
          </button>
          
          <button 
            v-if="selectionMode"
            class="icon-btn"
            @click="dismissRecruitsFn"
            :disabled="dismissing || selectedIds.size === 0"
            v-tooltip="'Dismiss Selected'"
          >
             <Icon name="close" />
          </button>
          
          <button 
            v-if="!selectionMode"
            class="icon-btn"
            @click="loadData"
            :disabled="loading"
            v-tooltip="'Refresh'"
          >
            <Icon name="refresh" :class="{ 'spin': loading }" />
          </button>
        </div>
      </div>
    </header>

    <!-- Error State -->
    <ErrorState 
      v-if="error" 
      :message="error" 
      @retry="loadData" 
    />
    
    <!-- Loading State -->
    <div v-else-if="loading" class="recruit-list">
      <div v-for="i in 5" :key="i" class="skeleton-card"></div>
    </div>
    
    <!-- Empty State -->
    <EmptyState 
      v-else-if="recruits.length === 0"
      icon="scope"
      message="No recruits found"
      hint="Check back later for new prospects"
    />
    
    <!-- Recruit List -->
    <div v-else class="recruit-list stagger-children">
      <RecruitCard
        v-for="recruit in recruits"
        :key="recruit.id"
        :id="`recruit-${recruit.id}`"
        :recruit="recruit"
        :selected="selectedIds.has(recruit.id)"
        :expanded="expandedIds.has(recruit.id)"
        :selection-mode="selectionMode"
        @toggle-select="toggleSelect(recruit.id)"
        @toggle-expand="toggleExpand(recruit.id)"
      />
    </div>
  </div>
</template>

<style scoped>
.recruiter-view {
  min-height: 100%;
}

/* App Bar */
.top-app-bar {
  position: sticky;
  top: 0;
  z-index: 50;
  background: var(--md-sys-color-surface);
  transition: all 0.2s ease;
  padding: 0.5rem 1rem;
}

.top-app-bar.scrolled {
  background: var(--md-sys-color-surface-container);
  border-bottom: 1px solid var(--md-sys-color-outline-variant);
}

.toolbar-content {
  display: flex;
  align-items: center;
  justify-content: space-between;
  height: 48px;
}

.page-title {
  font-size: 1.375rem;
  font-weight: 500;
  margin: 0;
}

.selection-header {
  display: flex;
  align-items: center;
  gap: 1rem;
  flex: 1;
}

.selection-count {
  font-size: 1.125rem;
  font-weight: 500;
}

.actions {
  display: flex;
  gap: 0.5rem;
}

.icon-btn {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  border: none;
  background: transparent;
  color: var(--md-sys-color-on-surface);
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  transition: background-color 0.2s;
}

.icon-btn:hover {
  background-color: var(--md-sys-color-surface-variant);
}

.icon-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.spin { animation: spin 1s linear infinite; }
@keyframes spin { 100% { transform: rotate(360deg); } }

.recruit-list {
  padding: 0 1rem 120px;
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

/* Skeletons */
.skeleton-card {
  height: 80px;
  background: var(--md-sys-color-surface-container);
  border-radius: var(--md-sys-shape-corner-large);
  animation: pulse 1.5s infinite;
}
</style>
