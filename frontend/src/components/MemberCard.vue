<script setup lang="ts">
import type { LeaderboardMember } from '../types'
import WarHistoryChart from './WarHistoryChart.vue'
import Icon from './Icon.vue'
import { computed } from 'vue'

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

function openInGame() {
  window.open(`clashroyale://playerInfo?id=${props.member.id}`, '_blank')
}

function openRoyaleAPI() {
  window.open(`https://royaleapi.com/player/${props.member.id}`, '_blank')
}

const checkboxClasses = computed(() => 
  props.selected ? 'selection-checkbox selected' : 'selection-checkbox'
)

function handleCardClick() {
  if (props.selectionMode) {
    emit('toggleSelect')
  } else {
    emit('toggle')
  }
}

function handleSelectClick(e: Event) {
  e.stopPropagation()
  emit('toggleSelect')
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
  >
    <!-- Explicit Selection Indicator/Checkbox -->
    <div 
      :class="checkboxClasses" 
      @click="handleSelectClick"
    >
      <transition name="scale">
        <Icon v-if="selected" name="check" size="14" class="check-mark" />
      </transition>
    </div>
    <!-- Main Row -->
    <div class="card-header">
      <!-- Rank Badge -->
      <div class="rank" :class="{ 'rank-top': rank <= 3 }">
        <span v-if="rank === 1">🥇</span>
        <span v-else-if="rank === 2">🥈</span>
        <span v-else-if="rank === 3">🥉</span>
        <span v-else>{{ rank }}</span>
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
            <Icon name="trophy" size="12" class="stat-icon" />
            {{ member.t.toLocaleString() }}
          </span>
          <span class="stat-item">
            <Icon name="warlog" size="12" class="stat-icon" />
            {{ member.d.rate }}
          </span>
          <span class="stat-item" v-if="member.d.days > 0">
            <span class="stat-icon">📅</span> <!-- Calendar icon not in set, keep emoji or add? 'donation' is similar visual weight -->
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
        <div class="chevron-btn" :class="{ 'chevron-open': expanded }">
          <Icon name="expand" size="20" />
        </div>
      </div>
    </div>
    
    <!-- Expanded Details -->
    <div class="card-body" :class="{ 'card-body-open': expanded }">
      <!-- Grid Stats -->
      <div class="grid-stats">
        <div class="stat-box">
          <span class="stat-box-label">Avg/Day</span>
          <span class="stat-box-value">{{ member.d.avg }}</span>
        </div>
        <div class="stat-box">
          <span class="stat-box-label">War Rate</span>
          <span class="stat-box-value">{{ member.d.rate }}</span>
        </div>
        <div class="stat-box">
          <span class="stat-box-label">Last Seen</span>
          <span class="stat-box-value">{{ member.d.seen }}</span>
        </div>
      </div>
      
      <!-- War History Chart -->
      <WarHistoryChart :history="member.d.hist" />
      
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
</template>

<style scoped>
.member-card {
  background: var(--md-sys-color-surface-container, #f3f3f3);
  border-radius: var(--md-sys-shape-corner-large);
  padding: 1rem;
  padding-left: 3.5rem; /* Space for checkbox */
  position: relative;
  transition: all var(--md-sys-motion-duration-medium2) var(--md-sys-motion-easing-standard);
  border: 1px solid transparent; /* Prevent jump */
  cursor: pointer;
}

.member-card:active {
  transform: scale(0.98);
}

.member-card-selected {
  background: var(--md-sys-color-secondary-container);
  border-color: var(--md-sys-color-primary);
}

.member-card-expanded {
  background: var(--md-sys-color-surface-container-high);
  box-shadow: var(--md-sys-elevation-2);
}

/* Duplicated selection styles for parity with RecruitCard */
.selection-checkbox {
  position: absolute;
  left: 1rem;
  top: 50%;
  transform: translateY(-50%);
  width: 24px;
  height: 24px;
  border-radius: 50%;
  border: 2px solid var(--md-sys-color-outline);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 2;
  transition: all 0.2s ease;
  background: transparent;
}

.selection-checkbox.selected {
  background: var(--md-sys-color-primary);
  border-color: var(--md-sys-color-primary);
}

.check-mark {
  color: var(--md-sys-color-on-primary);
}

.scale-enter-active,
.scale-leave-active {
  transition: transform 0.2s ease;
}

.scale-enter-from,
.scale-leave-to {
  transform: scale(0);
}

/* Header Layout */
.card-header {
  display: flex;
  align-items: center;
  gap: 0.75rem;
}

/* Rank */
.rank {
  width: 2.5rem;
  height: 2.5rem;
  display: flex;
  align-items: center;
  justify-content: center;
  background: var(--md-sys-color-surface-variant, #e0e0e0);
  border-radius: 50%;
  font-weight: 700;
  font-size: 0.875rem;
  color: var(--md-sys-color-on-surface-variant, #49454f);
  flex-shrink: 0;
}

.rank-top {
  background: transparent;
  font-size: 1.5rem;
}

/* Member Info */
.member-info {
  flex: 1;
  min-width: 0;
}

.member-header {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  margin-bottom: 0.25rem;
}

.member-name {
  font-weight: 600;
  font-size: 1rem;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  color: var(--md-sys-color-on-surface, #1c1b1f);
}

/* Role Badges */
.badge {
  font-size: 0.625rem;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.03em;
  padding: 0.25rem 0.5rem;
  border-radius: 0.5rem;
}

.badge-leader {
  background: var(--md-sys-color-primary-container, #eaddff);
  color: var(--md-sys-color-on-primary-container, #21005e);
}

.badge-co-leader {
  background: var(--md-sys-color-secondary-container, #e8def8);
  color: var(--md-sys-color-on-secondary-container, #1e192b);
}

.badge-elder {
  background: var(--md-sys-color-tertiary-container, #ffd8e4);
  color: var(--md-sys-color-on-tertiary-container, #31111d);
}

.badge-member {
  display: none;
}

/* Stats Row */
.member-stats {
  display: flex;
  gap: 0.75rem;
  flex-wrap: wrap;
}

.stat-item {
  display: flex;
  align-items: center;
  gap: 0.25rem;
  font-size: 0.75rem;
  color: var(--md-sys-color-outline, #79747e);
}

.stat-icon {
  font-size: 0.625rem;
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
  width: 3.5rem;
  height: 3.5rem;
  border-radius: 1rem;
  flex-shrink: 0;
}

.tone-high {
  background: var(--md-sys-color-primary-container, #eaddff);
  color: var(--md-sys-color-on-primary-container, #21005e);
}

.tone-mid {
  background: var(--md-sys-color-secondary-container, #e8def8);
  color: var(--md-sys-color-on-secondary-container, #1e192b);
}

.tone-low {
  background: var(--md-sys-color-surface-variant, #e7e0ec);
  color: var(--md-sys-color-on-surface-variant, #49454f);
}

.stat-score {
  font-size: 1.125rem;
  font-weight: 700;
  line-height: 1;
  font-family: system-ui, sans-serif;
}

.stat-sub {
  font-size: 0.5rem;
  font-weight: 700;
  opacity: 0.7;
  margin-top: 2px;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

/* Chevron */
.chevron-btn {
  width: 2rem;
  height: 2rem;
  display: flex;
  align-items: center;
  justify-content: center;
  color: var(--md-sys-color-on-surface-variant, #49454f);
  opacity: 0.5;
  transition: transform 0.3s cubic-bezier(0.2, 0, 0, 1);
}

.chevron-btn svg {
  width: 1.25rem;
  height: 1.25rem;
  fill: none;
  stroke: currentColor;
  stroke-width: 2;
  stroke-linecap: round;
  stroke-linejoin: round;
}

.chevron-open {
  transform: rotate(180deg);
  opacity: 1;
}

/* Expanded Body */
.card-body {
  max-height: 0;
  opacity: 0;
  overflow: hidden;
  transition: all 0.3s cubic-bezier(0.2, 0, 0, 1);
}

.card-body-open {
  max-height: 400px;
  opacity: 1;
  margin-top: 1rem;
  padding-top: 1rem;
  border-top: 1px solid var(--md-sys-color-outline-variant, #cac4d0);
}

/* Grid Stats */
.grid-stats {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 0.5rem;
  margin-bottom: 1rem;
}

.stat-box {
  background: var(--md-sys-color-surface-container, #f3f3f3);
  padding: 0.75rem 0.5rem;
  border-radius: 0.75rem;
  text-align: center;
}

.stat-box-label {
  display: block;
  font-size: 0.625rem;
  font-weight: 600;
  text-transform: uppercase;
  color: var(--md-sys-color-outline, #79747e);
  margin-bottom: 0.25rem;
}

.stat-box-value {
  font-size: 0.9375rem;
  font-weight: 600;
  color: var(--md-sys-color-on-surface, #1c1b1f);
}

/* Action Buttons */
.btn-row {
  display: flex;
  gap: 0.75rem;
  margin-top: 1rem;
}

.btn-action {
  flex: 1;
  padding: 0.875rem 1rem;
  border: none;
  border-radius: 1.5rem;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  transition: transform 0.15s ease, opacity 0.15s ease;
}

.btn-action:active {
  transform: scale(0.97);
}

.btn-primary {
  background: var(--md-sys-color-primary, #6750a4);
  color: var(--md-sys-color-on-primary, #ffffff);
}

.btn-secondary {
  background: var(--md-sys-color-secondary-container, #e8def8);
  color: var(--md-sys-color-on-secondary-container, #1e192b);
}
</style>
