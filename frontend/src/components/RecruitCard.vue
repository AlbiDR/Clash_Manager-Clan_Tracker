<script setup lang="ts">
import Icon from './Icon.vue'
import { computed } from 'vue'
import type { Recruit } from '../types'

const props = defineProps<{
  recruit: Recruit
  selected: boolean
  expanded: boolean
  selectionMode: boolean
}>()

const emit = defineEmits<{
  toggleSelect: []
  toggleExpand: []
}>()

// Calculate how long ago the recruit was found
const timeAgo = computed(() => {
  if (!props.recruit.d.ago) return 'Unknown'
  
  const found = new Date(props.recruit.d.ago)
  const now = new Date()
  const diffMs = now.getTime() - found.getTime()
  const diffDays = Math.floor(diffMs / (1000 * 60 * 60 * 24))
  
  if (diffDays === 0) return 'Today'
  if (diffDays === 1) return 'Yesterday'
  if (diffDays < 7) return `${diffDays}d ago`
  return `${Math.floor(diffDays / 7)}w ago`
})

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
    emit('toggleExpand')
  }
}

// Long press logic
let longPressTimer: number | null = null

function startLongPress() {
  if (props.selectionMode) return
  longPressTimer = setTimeout(() => {
    emit('toggleSelect')
  }, 500) as unknown as number
}

function cancelLongPress() {
  if (longPressTimer) {
    clearTimeout(longPressTimer)
    longPressTimer = null
  }
}

function openInGame() {
  window.open(`clashroyale://playerInfo?id=${props.recruit.id}`, '_blank')
}

function openRoyaleAPI() {
  window.open(`https://royaleapi.com/player/${props.recruit.id}`, '_blank')
}
</script>

<template>
  <div 
    class="recruit-card"
    :class="{ 
      'recruit-card-expanded': expanded,
      'recruit-card-selected': selected 
    }"
    @click="handleCardClick"
    @mousedown="startLongPress"
    @touchstart="startLongPress"
    @mouseup="cancelLongPress"
    @touchend="cancelLongPress"
    @mouseleave="cancelLongPress"
  >
    
    <!-- Header -->
    <div class="card-header">
       <!-- LEADING ELEMENT: RECRUIT ICON SWAP -->
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
            class="recruit-avatar" 
            key="avatar"
          >
            <Icon name="telescope" size="18" />
          </div>
        </transition>
      </div>

      <!-- Info -->
      <div class="recruit-info">
        <div class="recruit-header">
          <span class="recruit-name">{{ recruit.n }}</span>
        </div>
        
        <div class="recruit-stats">
          <span class="stat-item">
            <Icon name="trophy" size="14" class="stat-icon" />
            {{ recruit.t.toLocaleString() }}
          </span>
          <span class="stat-item">
            <Icon name="donation" size="14" class="stat-icon" />
            {{ recruit.d.don.toLocaleString() }}
          </span>
          <span class="stat-item">
            <Icon name="warlog" size="14" class="stat-icon" />
            {{ recruit.d.war }}
          </span>
        </div>
      </div>
      
      <!-- Score Pod -->
      <div class="action-area">
        <div class="stat-pod" :class="getScoreTone(recruit.s)">
          <span class="stat-score">{{ Math.round(recruit.s) }}</span>
          <span class="stat-sub">SCORE</span>
        </div>
      </div>
    </div>
    
    <!-- Expanded Body -->
    <div class="card-body" :class="{ 'card-body-open': expanded }">
      <div class="expanded-content">
        <!-- Grid Stats -->
        <div class="grid-stats">
          <div class="stat-box">
            <span class="stat-box-label">Found</span>
            <span class="stat-box-value">{{ timeAgo }}</span>
          </div>
          <div class="stat-box">
            <span class="stat-box-label">Cards Won</span>
            <span class="stat-box-value">{{ recruit.d.cards || '-' }}</span>
          </div>
          <div class="stat-box">
            <span class="stat-box-label">War Wins</span>
            <span class="stat-box-value">{{ recruit.d.war }}</span>
          </div>
        </div>
        
        <!-- Action Buttons -->
        <div class="btn-row">
          <button class="btn-action btn-secondary" @click.stop="openRoyaleAPI">
            RoyaleAPI
          </button>
          <button class="btn-action btn-primary" @click.stop="openInGame">
            Open in Game
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.recruit-card {
  background: var(--md-sys-color-surface-container);
  border-radius: var(--md-sys-shape-corner-large);
  padding: 0.75rem 1rem;
  position: relative;
  transition: all 0.2s cubic-bezier(0.2, 0, 0, 1);
  border: 1px solid transparent; /* Prevent jump on selection */
  user-select: none;
}

.recruit-card:active {
  background: var(--md-sys-color-surface-container-high);
}

.recruit-card-selected {
  background: var(--md-sys-color-secondary-container);
  border-color: var(--md-sys-color-primary);
}

.recruit-card-expanded {
  background: var(--md-sys-color-surface-container-high);
  box-shadow: var(--md-sys-elevation-2);
}

/* Leading */
.leading-container {
  width: 40px;
  height: 40px;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}

.recruit-avatar {
  width: 32px;
  height: 32px;
  border-radius: 50%;
  background: var(--md-sys-color-surface-variant);
  color: var(--md-sys-color-on-surface-variant);
  display: flex;
  align-items: center;
  justify-content: center;
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


/* Header Layout */
.card-header {
  display: flex;
  align-items: center;
  gap: 1rem;
}

/* Recruit Info */
.recruit-info {
  flex: 1;
  min-width: 0;
  display: flex;
  flex-direction: column;
  gap: 0.125rem;
}

.recruit-name {
  font-weight: 500;
  font-size: 1rem;
  color: var(--md-sys-color-on-surface);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

/* Stats Row */
.recruit-stats {
  display: flex;
  gap: 0.75rem;
  flex-wrap: wrap;
}

.stat-item {
  display: flex;
  align-items: center;
  gap: 0.25rem;
  font-size: 0.75rem;
  color: var(--md-sys-color-outline);
}

.stat-icon {
  color: var(--md-sys-color-outline);
}

/* Action Area */
.action-area {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

/* Score Pod */
.stat-pod {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  width: 3rem;
  height: 3rem;
  border-radius: 12px;
  flex-shrink: 0;
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
  opacity: 0.7;
  margin-top: 2px;
  text-transform: uppercase;
}

/* Expanded Body */
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


/* Grid Stats */
.grid-stats {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 0.5rem;
  margin-bottom: 1rem;
}

.stat-box {
  background: var(--md-sys-color-surface-container-high);
  padding: 0.75rem 0.5rem;
  border-radius: 0.75rem;
  text-align: center;
}

.stat-box-label {
  display: block;
  font-size: 0.625rem;
  font-weight: 600;
  text-transform: uppercase;
  color: var(--md-sys-color-outline);
  margin-bottom: 0.25rem;
}

.stat-box-value {
  font-size: 0.9375rem;
  font-weight: 600;
  color: var(--md-sys-color-on-surface);
}

/* Action Buttons */
.btn-row {
  display: flex;
  gap: 0.75rem;
}

.btn-action {
  flex: 1;
  padding: 0.75rem 1rem;
  border: none;
  border-radius: 2rem;
  font-size: 0.875rem;
  font-weight: 500;
  cursor: pointer;
  transition: transform 0.1s ease;
}

.btn-action:active {
  transform: scale(0.98);
}

.btn-primary {
  background: var(--md-sys-color-primary);
  color: var(--md-sys-color-on-primary);
}

.btn-secondary {
  background: var(--md-sys-color-secondary-container);
  color: var(--md-sys-color-on-secondary-container);
}
</style>
