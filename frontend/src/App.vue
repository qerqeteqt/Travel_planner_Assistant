<script setup lang="ts">
import { ref } from "vue";

import type { Itinerary } from "./types";
import History from "./views/History.vue";
import Home from "./views/Home.vue";
import Result from "./views/Result.vue";

const currentView = ref<"home" | "result" | "history">("home");
const latestItinerary = ref<Itinerary | null>(null);

function handleGenerated(itinerary: Itinerary) {
  latestItinerary.value = itinerary;
  currentView.value = "result";
}

function openTrip(itinerary: Itinerary) {
  latestItinerary.value = itinerary;
  currentView.value = "result";
}

function updateCurrentItinerary(itinerary: Itinerary) {
  latestItinerary.value = itinerary;
  currentView.value = "result";
}
</script>

<template>
  <div class="app-shell">
    <div class="app-shell__glow app-shell__glow--left"></div>
    <div class="app-shell__glow app-shell__glow--right"></div>

    <header class="hero">
      <div class="hero__badge">Trip Planner Demo</div>
      <h1 class="hero__title">智能旅行助手</h1>

      <div class="hero__tabs">
        <button
          :class="['hero__tab', { 'hero__tab--active': currentView === 'home' }]"
          @click="currentView = 'home'"
        >
          规划页
        </button>
        <button
          :class="[
            'hero__tab',
            { 'hero__tab--active': currentView === 'result' },
            { 'hero__tab--disabled': !latestItinerary }
          ]"
          :disabled="!latestItinerary"
          @click="currentView = 'result'"
        >
          结果页
        </button>
        <button
          :class="['hero__tab', { 'hero__tab--active': currentView === 'history' }]"
          @click="currentView = 'history'"
        >
          历史列表
        </button>
      </div>
    </header>

    <main class="page-content">
      <Home
        v-if="currentView === 'home'"
        @generated="handleGenerated"
      />
      <Result
        v-else-if="currentView === 'result'"
        :itinerary="latestItinerary"
        @back-home="currentView = 'home'"
        @view-history="currentView = 'history'"
        @updated="updateCurrentItinerary"
      />
      <History
        v-else
        :active="currentView === 'history'"
        @open-trip="openTrip"
      />
    </main>
  </div>
</template>

<style scoped>
/* ====== Global Light Theme ====== */
:global(body) {
  margin: 0;
  min-width: 320px;
  font-family: "Microsoft YaHei", "PingFang SC", "Segoe UI", sans-serif;
  background: #f0f2f5;
  color: #262626;
  overflow-x: hidden;
}

:global(*) {
  box-sizing: border-box;
}

.app-shell {
  position: relative;
  min-height: 100vh;
  padding: 40px 24px 64px;
  overflow: hidden;
}

/* Subtle glow orbs */
.app-shell__glow {
  position: absolute;
  border-radius: 50%;
  opacity: 0.12;
  pointer-events: none;
}

.app-shell__glow--left {
  top: -120px;
  left: -120px;
  width: 400px;
  height: 400px;
  background: radial-gradient(circle, rgba(22, 119, 255, 0.06) 0%, transparent 70%);
}

.app-shell__glow--right {
  right: -100px;
  bottom: 80px;
  width: 350px;
  height: 350px;
  background: radial-gradient(circle, rgba(114, 46, 209, 0.05) 0%, transparent 70%);
}

/* ====== Hero ====== */
.hero {
  position: relative;
  z-index: 1;
  max-width: 1280px;
  margin: 0 auto 28px;
  text-align: center;
}

.hero::before {
  content: "";
  position: absolute;
  inset: -24px 0 auto;
  height: 220px;
  z-index: -1;
  border-radius: 36px;
  background: #ffffff;
  border: 1px solid #e8e8e8;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.06);
}

.hero__badge {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  padding: 6px 16px;
  border-radius: 999px;
  background: rgba(22, 119, 255, 0.08);
  border: 1px solid rgba(22, 119, 255, 0.2);
  color: #1677ff;
  font-size: 12px;
  font-weight: 600;
  letter-spacing: 0.08em;
  text-transform: uppercase;
}

.hero__badge::before {
  content: "";
  display: inline-block;
  width: 6px;
  height: 6px;
  border-radius: 50%;
  background: #1677ff;
  box-shadow: 0 0 6px rgba(22, 119, 255, 0.3);
  animation: dotPulse 2s ease-in-out infinite;
}

@keyframes dotPulse {
  0%, 100% { opacity: 1; box-shadow: 0 0 6px #1677ff; }
  50% { opacity: 0.3; box-shadow: 0 0 2px #1677ff; }
}

.hero__title {
  margin: 20px 0 0;
  font-size: 52px;
  line-height: 1.1;
  background: linear-gradient(135deg, #1677ff 0%, #722ed1 50%, #13c2c2 100%);
  -webkit-background-clip: text;
  background-clip: text;
  -webkit-text-fill-color: transparent;
}

.hero__tabs {
  display: inline-flex;
  gap: 8px;
  margin-top: 24px;
  padding: 6px;
  border-radius: 14px;
  background: #ffffff;
  border: 1px solid #e8e8e8;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.06);
}

.hero__tab {
  border: 1px solid transparent;
  border-radius: 10px;
  padding: 10px 20px;
  background: transparent;
  color: #8c8c8c;
  font-size: 13px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s ease;
}

.hero__tab:hover:not(:disabled) {
  color: #1677ff;
  background: rgba(22, 119, 255, 0.06);
}

.hero__tab--active {
  background: rgba(22, 119, 255, 0.1);
  border-color: rgba(22, 119, 255, 0.3);
  color: #1677ff;
  box-shadow: none;
}

.hero__tab--disabled {
  opacity: 0.3;
  cursor: not-allowed;
}

.page-content {
  position: relative;
  z-index: 1;
  max-width: 1280px;
  margin: 0 auto;
}

@media (max-width: 768px) {
  .app-shell {
    padding: 24px 16px 40px;
  }

  .hero__title {
    font-size: 34px;
  }

  .hero::before {
    inset: -20px 0 auto;
    height: 230px;
  }
}
</style>
