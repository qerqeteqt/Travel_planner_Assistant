<script setup lang="ts">
import { message } from "ant-design-vue";
import { onMounted, ref, watch } from "vue";

import { deleteTrip, getTripDetail, listTrips } from "../services/api";
import type { Itinerary, TripSummaryItem } from "../types";

const props = defineProps<{
  active: boolean;
}>();

const emit = defineEmits<{
  openTrip: [itinerary: Itinerary];
}>();

const loading = ref(false);
const items = ref<TripSummaryItem[]>([]);
const deletingTripId = ref("");

async function loadTrips() {
  loading.value = true;
  try {
    const response = await listTrips();
    items.value = response.items;
  } catch (error) {
    console.error(error);
    message.error("历史列表加载失败。");
  } finally {
    loading.value = false;
  }
}

async function openTrip(tripId: string) {
  try {
    const response = await getTripDetail(tripId);
    emit("openTrip", response.itinerary);
    message.success("已加载已保存行程。");
  } catch (error) {
    console.error(error);
    message.error("读取行程详情失败。");
  }
}

async function removeTrip(tripId: string) {
  const confirmed = window.confirm("确定要删除这条已保存行程吗？删除后无法恢复。");
  if (!confirmed) {
    return;
  }

  deletingTripId.value = tripId;
  try {
    await deleteTrip(tripId);
    items.value = items.value.filter((item) => item.trip_id !== tripId);
    message.success("行程已删除。");
  } catch (error) {
    console.error(error);
    message.error("删除行程失败。");
  } finally {
    deletingTripId.value = "";
  }
}

onMounted(() => {
  if (props.active) {
    void loadTrips();
  }
});

watch(
  () => props.active,
  (active) => {
    if (active) {
      void loadTrips();
    }
  }
);
</script>

<template>
  <section class="history-page">
    <div class="history-header">
      <div>
        <h2>历史行程</h2>
        <p>这里会展示已经保存到后端数据库里的 itinerary 摘要。</p>
      </div>
      <button class="refresh-button" @click="loadTrips">刷新列表</button>
    </div>

    <div v-if="loading" class="history-state">正在加载历史列表...</div>
    <div v-else-if="items.length === 0" class="history-state">还没有已保存的行程。</div>

    <div v-else class="history-grid">
      <article
        v-for="item in items"
        :key="item.trip_id"
        class="history-card"
      >
        <div class="history-card__destination">{{ item.destination }}</div>
        <div class="history-card__trip-id">{{ item.trip_id }}</div>
        <p class="history-card__summary">{{ item.summary }}</p>
        <div class="history-card__time">
          更新时间：{{ item.updated_at || "未记录" }}
        </div>
        <div class="history-card__actions">
          <button class="history-card__button" @click="openTrip(item.trip_id)">
            查看详情
          </button>
          <button
            class="history-card__button history-card__button--danger"
            :disabled="deletingTripId === item.trip_id"
            @click="removeTrip(item.trip_id)"
          >
            {{ deletingTripId === item.trip_id ? "删除中..." : "删除行程" }}
          </button>
        </div>
      </article>
    </div>
  </section>
</template>

<style scoped>
.history-page {
  display: grid;
  gap: 18px;
}

/* ====== Header ====== */
.history-header {
  display: flex;
  justify-content: space-between;
  align-items: end;
  gap: 16px;
  padding: 24px;
  border-radius: 14px;
  background: #ffffff;
  border: 1px solid #e8e8e8;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.06);
}

.history-header h2 {
  margin: 0 0 8px;
  font-size: 28px;
  color: #1677ff;
  letter-spacing: 0.04em;
}

.history-header p {
  margin: 0;
  color: #bfbfbf;
}

.history-header .refresh-button {
  background: rgba(22, 119, 255, 0.08);
  border: 1px solid rgba(22, 119, 255, 0.2);
  color: #1677ff;
}

/* ====== Buttons ====== */
.refresh-button,
.history-card__button {
  border: 1px solid #d9d9d9;
  border-radius: 8px;
  padding: 10px 16px;
  background: #ffffff;
  color: #595959;
  font-weight: 700;
  font-size: 12px;
  letter-spacing: 0.04em;
  text-transform: uppercase;
  cursor: pointer;
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
}

.refresh-button:hover,
.history-card__button:hover:not(:disabled) {
  border-color: #1677ff;
  color: #1677ff;
  background: rgba(22, 119, 255, 0.04);
  transform: translateY(-1px);
}

.history-card__actions {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 10px;
}

.history-card__button--danger {
  background: #ffffff;
  border-color: #d9d9d9;
  color: #ff4d4f;
}

.history-card__button--danger:hover:not(:disabled) {
  border-color: #ff4d4f;
  color: #ff4d4f;
  background: rgba(255, 77, 79, 0.04);
}

.history-card__button:disabled {
  opacity: 0.4;
  cursor: wait;
  transform: none;
}

/* ====== State ====== */
.history-state {
  padding: 40px 28px;
  border-radius: 14px;
  background: #ffffff;
  border: 1px solid #e8e8e8;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.06);
  color: #bfbfbf;
  text-align: center;
  font-size: 14px;
}

/* ====== Grid ====== */
.history-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
  gap: 16px;
}

.history-card {
  display: grid;
  gap: 12px;
  padding: 22px;
  border-radius: 14px;
  background: #ffffff;
  border: 1px solid #e8e8e8;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.06);
  transition: all 0.3s ease;
}

.history-card:hover {
  border-color: #d0d0d0;
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.08);
  transform: translateY(-2px);
}

.history-card__destination {
  font-size: 28px;
  font-weight: 800;
  color: #1677ff;
  letter-spacing: 0.02em;
}

.history-card__trip-id {
  color: #bfbfbf;
  font-size: 11px;
  word-break: break-all;
  font-family: "JetBrains Mono", "Fira Code", monospace;
  letter-spacing: 0.03em;
}

.history-card__summary {
  margin: 0;
  color: #595959;
  line-height: 1.7;
  font-size: 13px;
}

.history-card__time {
  color: #bfbfbf;
  font-size: 12px;
}
</style>
