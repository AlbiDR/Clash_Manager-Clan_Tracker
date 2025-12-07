<script setup lang="ts">
import type { LeaderboardMember } from '../types'
import Icon from './Icon.vue'
import WarHistoryChart from './WarHistoryChart.vue'

const props = defineProps<{
  member: LeaderboardMember
  rank: number
  expanded: boolean
  selected?: boolean
  selectionMode?: boolean
}>()

const emit = defineEmits<{
  toggle: []
  toggleSelect: []
}>()

function getRoleBadgeClass(role: string): string {
  const r = role.toLowerCase()
  if (r === 'leader') return 'badge-leader'
  if (r === 'co-leader' || r === 'coleader') return 'badge-co-leader'
  if (r === 'elder') return 'badge-elder'
  return 'badge-member'
}

function formatRole(role: string): string {
  const r = role.toLowerCase()
  if (r === 'coleader') return 'Co-Leader'
  return role.charAt(0).toUpperCase() + role.slice(1).toLowerCase()
}

// Tonal color class based on score
function getScoreTone(score: number): string {
  if (score >= 80) return 'tone-high'
  if (score >= 50) return 'tone-mid'
  return 'tone-low'
}

function handleCardClick() {
  if (props.selectionMode) {
    emit('toggleSelect')
  } else {
    emit('toggle')
  }
}

// Long press for selection
let longPressTimer: ReturnType<typeof setTimeout> | null = null

function startLongPress() {
  if (props.selectionMode) return
  longPressTimer = setTimeout(() => {
    emit('toggleSelect')
  }, 500)
}

function cancelLongPress() {
  if (longPressTimer) {
    clearTimeout(longPressTimer)
    longPressTimer = null
  }
}
</script>

<template>
  <div 
    class="member-card"
    :class="{ 
      'member-card-expanded': expanded,
      'member-card-selected': selected 
    }"
    @click="handleCardClick"
    @mousedown="startLongPress"
    @touchstart="startLongPress"
    @mouseup="cancelLongPress"
    @touchend="cancelLongPress"
    @mouseleave="cancelLongPress"
  >
    <!-- Main Row -->
    <div class="card-header">
      <!-- LEADING ELEMENT: RANK or CHECKBOX (Avatar Swap) -->
      <div class="leading-container">
        <transition name="swap-rotate" mode="out-in">
          <div 
            v-if="selected || selectionMode" 
            class="avatar-checkbox" 
            :class="{ 'checked': selected }"
            key="checkbox"
          >
            <Icon name="check" size="16" class="check-icon" />
          </div>
          <div 
            v-else 
            class="rank-avatar" 
            :class="{ 'rank-top': rank <= 3 }"
            key="rank"
          >
            <span v-if="rank === 1">🥇</span>
            <span v-else-if="rank === 2">🥈</span>
            <span v-else-if="rank === 3">🥉</span>
            <span v-else>{{ rank }}</span>
          </div>
        </transition>
      </div>
      
      <!-- Member Info -->
      <div class="member-info">
        <div class="member-header">
          <span class="member-name">{{ member.n }}</span>
          <span 
            v-if="member.d.role && member.d.role.toLowerCase() !== 'member'"
            class="badge" 
            :class="getRoleBadgeClass(member.d.role)"
          >
            {{ formatRole(member.d.role) }}
          </span>
        </div>
        
        <div class="member-stats">
          <span class="stat-item">
            <Icon name="trophy" size="14" class="stat-icon" />
            {{ member.t.toLocaleString() }}
          </span>
          <span class="stat-item">
            <Icon name="warlog" size="14" class="stat-icon" />
            {{ member.d.rate }}
          </span>
          <span class="stat-item" v-if="member.d.days > 0">
            <Icon name="donation" size="14" class="stat-icon" />
            {{ member.d.days }}d
          </span>
        </div>
      </div>
      
      <!-- Score Pod + Chevron -->
      <div class="action-area">
        <div class="stat-pod" :class="getScoreTone(member.s)">
          <span class="stat-score">{{ Math.round(member.s) }}</span>
          <span class="stat-sub">SCORE</span>
        </div>
      </div>
    </div>
    
    <!-- Expanded Details -->
    <div class="card-body" :class="{ 'card-body-open': expanded }">
      <div class="expanded-content">
        <WarHistoryChart :history="member.d.hist" />
        
        <div class="detail-row">
           <div class="detail-item">
             <span class="label">Donations</span>
             <span class="value">{{ member.d.avg }} avg</span>
           </div>
           <div class="detail-item">
             <span class="label">Last Seen</span>
             <span class="value">{{ member.d.seen }}</span>
           </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.member-card {
  background: var(--md-sys-color-surface-container);
  border-radius: var(--md-sys-shape-corner-large);
  padding: 0.75rem 1rem;
  position: relative;
  transition: all var(--md-sys-motion-duration-medium2) var(--md-sys-motion-easing-standard);
  border: 1px solid transparent;
  width: 100%;
  user-select: none;
  -webkit-tap-highlight-color: transparent;
}

.member-card:active {
  background: var(--md-sys-color-surface-container-high);
}

.member-card-selected {
  background: var(--md-sys-color-secondary-container);
  border: 1px solid var(--md-sys-color-primary);
}

/* Leading Container (Avatar Swap) */
.leading-container {
  width: 40px;
  height: 40px;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}

.rank-avatar {
  width: 32px;
  height: 32px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: 700;
  color: var(--md-sys-color-on-surface-variant);
  background: var(--md-sys-color-surface-container-high);
  border-radius: 50%;
  font-size: 0.875rem;
}

.rank-top {
  background: transparent;
  font-size: 1.5rem;
}

.avatar-checkbox {
  width: 24px;
  height: 24px;
  border-radius: 2px;
  border: 2px solid var(--md-sys-color-on-surface-variant);
  display: flex;
  align-items: center;
  justify-content: center;
}

.avatar-checkbox.checked {
  background: var(--md-sys-color-primary);
  border-color: var(--md-sys-color-primary);
}

.check-icon {
  color: var(--md-sys-color-on-primary);
}

/* Swap Transition */
.swap-rotate-enter-active,
.swap-rotate-leave-active {
  transition: all 0.2s var(--md-sys-motion-easing-standard);
}

.swap-rotate-enter-from {
  opacity: 0;
  transform: rotate(-90deg) scale(0.5);
}

.swap-rotate-leave-to {
  opacity: 0;
  transform: rotate(90deg) scale(0.5);
}

/* Layout */
.card-header {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.member-info {
  flex: 1;
  min-width: 0;
  display: flex;
  flex-direction: column;
  gap: 0.125rem;
}

.member-header {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.member-name {
  font-weight: 500;
  font-size: 1rem;
  color: var(--md-sys-color-on-surface);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.badge {
  font-size: 0.625rem;
  padding: 0.125rem 0.375rem;
  border-radius: 4px;
  font-weight: 700;
  text-transform: uppercase;
  background: var(--md-sys-color-surface-variant);
  color: var(--md-sys-color-on-surface-variant);
}

.badge-leader { background: #FFD700; color: #000; }
.badge-co-leader { background: #C0C0C0; color: #000; }
.badge-elder { background: #cd7f32; color: #fff; }

.member-stats {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  color: var(--md-sys-color-outline);
  font-size: 0.75rem;
}

.stat-item {
  display: flex;
  align-items: center;
  gap: 0.25rem;
}

.stat-icon {
  color: var(--md-sys-color-outline);
}

/* Score Pod */
.stat-pod {
  width: 3rem;
  height: 3rem;
  border-radius: 12px;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
}

.tone-high {
  background: var(--md-sys-color-primary-container);
  color: var(--md-sys-color-on-primary-container);
}
.tone-mid {
  background: var(--md-sys-color-secondary-container);
  color: var(--md-sys-color-on-secondary-container);
}
.tone-low {
  background: var(--md-sys-color-surface-variant);
  color: var(--md-sys-color-on-surface-variant);
}

.stat-score {
  font-size: 1rem;
  font-weight: 700;
  line-height: 1;
}

.stat-sub {
  font-size: 0.45rem;
  font-weight: 700;
  opacity: 0.8;
  margin-top: 1px;
}

/* Expanded */
.card-body {
  display: grid;
  grid-template-rows: 0fr;
  transition: grid-template-rows 0.3s var(--md-sys-motion-easing-emphasized);
}

.card-body-open {
  grid-template-rows: 1fr;
  margin-top: 0.5rem;
  padding-top: 0.5rem;
  border-top: 1px solid var(--md-sys-color-outline-variant);
}

.expanded-content {
  overflow: hidden;
}

.detail-row {
  display: flex;
  gap: 1rem;
  margin-top: 0.5rem;
}

.detail-item {
  display: flex;
  flex-direction: column;
}

.label { font-size: 0.75rem; color: var(--md-sys-color-outline); }
.value { font-size: 0.875rem; color: var(--md-sys-color-on-surface); font-weight: 500; }
</style>
