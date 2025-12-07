<script setup lang="ts">
import { ref, computed, onMounted, nextTick } from 'vue'
import { useRoute } from 'vue-router'
import { getLeaderboard } from '../api/gasClient'
import type { LeaderboardMember } from '../types'
import MemberCard from '../components/MemberCard.vue'
import PullToRefresh from '../components/PullToRefresh.vue'
import EmptyState from '../components/EmptyState.vue'
import ErrorState from '../components/ErrorState.vue'
import Icon from '../components/Icon.vue'

const route = useRoute()

const members = ref<LeaderboardMember[]>([])
const loading = ref(true)
const error = ref<string | null>(null)
const searchQuery = ref('')
const sortBy = ref<'score' | 'trophies' | 'name'>('score')
const showSortMenu = ref(false)
const isScrolled = ref(false)

// Expansion & Selection
const expandedIds = ref<Set<string>>(new Set())
const selectedIds = ref<Set<string>>(new Set())
const selectionMode = ref(false)

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

function getRank(id: string): number {
  return members.value.findIndex(m => m.id === id) + 1
}

// Filtered and sorted members
const filteredMembers = computed(() => {
  let result = [...members.value]
  
  // Filter by search
  if (searchQuery.value) {
    const query = searchQuery.value.toLowerCase()
    result = result.filter(m => 
      m.n.toLowerCase().includes(query) || 
      m.id.toLowerCase().includes(query)
    )
  }
  
  // Sort
  result.sort((a, b) => {
    switch (sortBy.value) {
      case 'score': return b.s - a.s
      case 'trophies': return b.t - a.t
      case 'name': return a.n.localeCompare(b.n)
      default: return 0
    }
  })
  
  return result
})

// Stats
const stats = computed(() => ({
  total: members.value.length,
  avgScore: Math.round(members.value.reduce((sum, m) => sum + m.s, 0) / (members.value.length || 1)),
  avgTrophies: Math.round(members.value.reduce((sum, m) => sum + m.t, 0) / (members.value.length || 1))
}))

async function loadData() {
  loading.value = true
  error.value = null
  
  try {
    const response = await getLeaderboard()
    if (response.success && response.data) {
      members.value = response.data.lb || []
      
      // Handle Pin/Scroll-to
      const pinId = route.query.pin as string
      if (pinId && members.value.some(m => m.id === pinId)) {
        expandedIds.value.add(pinId)
        nextTick(() => {
          const el = document.getElementById(`member-${pinId}`)
          if (el) {
            el.scrollIntoView({ behavior: 'smooth', block: 'center' })
            el.classList.add('flash-highlight')
          }
        })
      }
    } else {
      error.value = response.error?.message || 'Failed to load data'
    }
  } catch (e) {
    error.value = e instanceof Error ? e.message : 'Network error'
  } finally {
    loading.value = false
  }
}

onMounted(loadData)
</script>

<template>
  <div class="leaderboard-view">
    <PullToRefresh @refresh="loadData" />
    
    <!-- Functional Top App Bar -->
    <header class="top-app-bar" :class="{ 'scrolled': isScrolled }">
      <div class="toolbar-content">
        <h1 class="page-title" v-if="!selectionMode">Leaderboard</h1>
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
            class="icon-btn"
            @click="loadData"
            :disabled="loading"
            v-tooltip="'Refresh'"
          >
            <Icon name="refresh" :class="{ 'spin': loading }" />
          </button>
        </div>
      </div>
      
      <!-- Stats Summary (Collapsed in bar or just below) -->
      <div class="stats-ticker" v-if="!selectionMode">
        <span>{{ stats.total }} Members</span>
        <span class="dot">•</span>
        <span>{{ stats.avgScore.toLocaleString() }} Avg Score</span>
      </div>
    </header>

    <!-- Floating Search & Filter Bar -->
    <div class="search-container animate-fade-in">
      <div class="search-bar surface-container-high">
        <Icon name="search" class="search-leading-icon" />
        <input 
          v-model="searchQuery"
          type="text" 
          placeholder="Search members..."
          class="search-input"
        />
        <button 
          class="icon-btn filter-btn"
          @click="showSortMenu = !showSortMenu"
          v-tooltip="'Sort'"
        >
          <Icon name="filter" />
        </button>
      </div>
      
      <!-- Expandable Sort Options -->
      <div class="sort-options" :class="{ 'open': showSortMenu }">
        <button 
          v-for="option in [
            { key: 'score', label: 'Score' },
            { key: 'trophies', label: 'Trophies' },
            { key: 'name', label: 'Name' }
          ]"
          :key="option.key"
          class="chip"
          :class="{ 'chip-active': sortBy === option.key }"
          @click="sortBy = option.key as typeof sortBy"
        >
          {{ option.label }}
        </button>
      </div>
    </div>
    
    <!-- Error State -->
    <ErrorState 
      v-if="error" 
      :message="error" 
      @retry="loadData" 
    />
    
    <!-- Loading State -->
    <div v-else-if="loading" class="member-list">
      <div v-for="i in 5" :key="i" class="skeleton-card">
        <div class="skeleton" style="width: 40px; height: 40px; border-radius: 50%;"></div>
        <div class="skeleton" style="width: 120px; height: 20px;"></div>
        <div class="skeleton" style="width: 60px; height: 20px; margin-left: auto;"></div>
      </div>
    </div>
    
    <!-- Empty State -->
    <EmptyState 
      v-else-if="filteredMembers.length === 0"
      icon="🔍"
      :message="searchQuery ? `No members match '${searchQuery}'` : 'No member data available'"
      hint="Try adjusting your filters or search query"
    />
    
    <!-- Member List -->
    <div v-else class="member-list stagger-children">
      <MemberCard
        v-for="member in filteredMembers"
        :key="member.id"
        :id="`member-${member.id}`"
        :member="member"
        :rank="getRank(member.id)"
        :expanded="expandedIds.has(member.id)"
        :selected="selectedIds.has(member.id)"
        :selection-mode="selectionMode"
        @toggle="toggleExpand(member.id)"
        @toggle-select="toggleSelect(member.id)"
      />
    </div>
  </div>
</template>

<style scoped>
.leaderboard-view {
  display: flex;
  flex-direction: column;
  gap: 1rem;
  min-height: 100%;
}

/* Actions */
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

.spin {
  animation: spin 1s linear infinite;
}

@keyframes spin {
  100% { transform: rotate(360deg); }
}

/* Top App Bar */
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

.stats-ticker {
  font-size: 0.75rem;
  color: var(--md-sys-color-on-surface-variant);
  margin-top: 0.25rem;
  display: flex;
  gap: 0.5rem;
  align-items: center;
}

.dot { font-weight: bold; }

/* Floating Search Bar */
.search-container {
  padding: 1rem;
  background: transparent;
}

.search-bar {
  display: flex;
  align-items: center;
  padding: 0.5rem 1rem;
  background: var(--md-sys-color-surface-container-high);
  border-radius: 2rem; /* Fully rounded */
  gap: 0.75rem;
  box-shadow: var(--md-sys-elevation-1);
}

.search-leading-icon {
  color: var(--md-sys-color-on-surface-variant);
}

.search-input {
  flex: 1;
  background: transparent;
  border: none;
  font-size: 1rem;
  color: var(--md-sys-color-on-surface);
  outline: none;
}

.search-input::placeholder {
  color: var(--md-sys-color-on-surface-variant);
}

/* Sort Options */
.sort-options {
  max-height: 0;
  overflow: hidden;
  transition: all 0.3s var(--md-sys-motion-easing-emphasized);
  display: flex;
  gap: 0.5rem;
  margin-top: 0;
  padding: 0 0.5rem;
}

.sort-options.open {
  max-height: 100px;
  margin-top: 0.75rem;
}

.chip {
  padding: 0.375rem 1rem;
  border-radius: 0.5rem;
  border: 1px solid var(--md-sys-color-outline);
  background: transparent;
  color: var(--md-sys-color-on-surface-variant);
  font-size: 0.875rem;
  font-weight: 500;
  cursor: pointer;
}

.chip-active {
  background: var(--md-sys-color-secondary-container);
  color: var(--md-sys-color-on-secondary-container);
  border-color: transparent;
}

.member-list {
  padding: 0 1rem 120px;
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

/* Skeletons */
.skeleton-card {
  display: flex;
  align-items: center;
  gap: 1rem;
  padding: 1rem;
  background: var(--md-sys-color-surface-container);
  border-radius: var(--md-sys-shape-corner-large);
  animation: pulse 1.5s infinite;
}

.skeleton {
  background: var(--md-sys-color-surface-variant);
  border-radius: 4px;
}

/* Error/Empty States */
.error-state,
.empty-state {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 1rem;
  padding: 2rem;
  text-align: center;
}

.error-icon,
.empty-icon {
  font-size: 3rem;
}

.error-state p,
.empty-state p {
  color: var(--cr-text-secondary);
  margin: 0;
}
</style>
