<script setup lang="ts">
import axios from "axios";
import { computed, reactive, ref } from "vue";
import { message } from "ant-design-vue";

import { generateTrip } from "../services/api";
import type { Itinerary, TripRequestPayload } from "../types";

const emit = defineEmits<{
  generated: [itinerary: Itinerary];
}>();

const preferenceOptions = [
  "自然风景",
  "拍照",
  "美食",
  "古镇",
  "休闲",
];

const dietaryOptions = [
  "少辣",
  "不吃香菜",
  "不吃葱",
];

function formatDate(date: Date): string {
  const y = date.getFullYear();
  const m = String(date.getMonth() + 1).padStart(2, "0");
  const d = String(date.getDate()).padStart(2, "0");
  return `${y}-${m}-${d}`;
}

const today = new Date();
const todayPlus2 = new Date(today);
todayPlus2.setDate(todayPlus2.getDate() + 2);

const formState = reactive({
  destination: "大理",
  startDate: formatDate(today),
  endDate: formatDate(todayPlus2),
  travelers: 2,
  budget: 3200,
  hotelLevel: "舒适型",
  pace: "轻松",
  preferences: ["自然风景", "拍照", "美食"],
  dietaryPreferences: ["少辣"],
  notes: "不想太早起床，希望安排一个适合看日落的地点。",
});

const isSubmitting = ref(false);

const dayCount = computed(() => {
  const start = new Date(formState.startDate);
  const end = new Date(formState.endDate);
  const diff = end.getTime() - start.getTime();
  return Number.isNaN(diff) ? 0 : Math.max(Math.floor(diff / 86400000) + 1, 0);
});

async function handleSubmit() {
  const payload: TripRequestPayload = {
    destination: formState.destination,
    start_date: formState.startDate,
    end_date: formState.endDate,
    travelers: formState.travelers,
    budget: formState.budget,
    preferences: formState.preferences,
    pace: formState.pace,
    dietary_preferences: formState.dietaryPreferences,
    hotel_level: formState.hotelLevel,
    special_notes: formState.notes,
  };

  isSubmitting.value = true;
  try {
    const itinerary = await generateTrip(payload);
    message.success("行程生成成功，已切换到结果页。");
    emit("generated", itinerary);
  } catch (error) {
    console.error(error);
    if (axios.isAxiosError(error)) {
      if (error.code === "ECONNABORTED") {
        message.error("行程生成超时，模型返回较慢，请稍后再试。");
      } else if (error.response) {
        message.error(`行程生成失败：后端返回 ${error.response.status}。`);
      } else {
        message.error("行程生成失败，请检查前端到后端的连接。");
      }
    } else {
      message.error("行程生成失败，请检查后端地址或服务状态。");
    }
  } finally {
    isSubmitting.value = false;
  }
}
</script>

<template>
  <section class="home-page">
    <div class="planner-card">
      <div class="section-title">
        <span class="section-title__icon">📍</span>
        <span>目的地与日期</span>
      </div>

      <a-row :gutter="[16, 16]">
        <a-col :xs="24" :md="8">
          <label class="field-label">目的地城市</label>
          <a-input v-model:value="formState.destination" placeholder="请输入目的地" />
        </a-col>
        <a-col :xs="24" :md="5">
          <label class="field-label">开始日期</label>
          <a-input v-model:value="formState.startDate" />
        </a-col>
        <a-col :xs="24" :md="5">
          <label class="field-label">结束日期</label>
          <a-input v-model:value="formState.endDate" />
        </a-col>
        <a-col :xs="12" :md="3">
          <label class="field-label">人数</label>
          <a-input-number v-model:value="formState.travelers" :min="1" style="width: 100%" />
        </a-col>
        <a-col :xs="12" :md="3">
          <label class="field-label">旅行天数</label>
          <div class="pill-box">{{ dayCount }} 天</div>
        </a-col>
      </a-row>
    </div>

    <div class="planner-card">
      <div class="section-title">
        <span class="section-title__icon">⚙️</span>
        <span>偏好设置</span>
      </div>

      <a-row :gutter="[16, 16]">
        <a-col :xs="24" :md="8">
          <label class="field-label">节奏偏好</label>
          <a-select
            v-model:value="formState.pace"
            :options="[
              { label: '轻松', value: '轻松' },
              { label: '适中', value: '适中' },
              { label: '紧凑', value: '紧凑' }
            ]"
          />
        </a-col>
        <a-col :xs="24" :md="8">
          <label class="field-label">住宿偏好</label>
          <a-select
            v-model:value="formState.hotelLevel"
            :options="[
              { label: '舒适型', value: '舒适型' },
              { label: '高档型', value: '高档型' },
              { label: '经济型', value: '经济型' }
            ]"
          />
        </a-col>
        <a-col :xs="24" :md="8">
          <label class="field-label">预算</label>
          <a-input-number v-model:value="formState.budget" :min="0" style="width: 100%" />
        </a-col>
      </a-row>

      <div class="checkbox-area">
        <label class="field-label">旅行偏好</label>
        <a-checkbox-group v-model:value="formState.preferences" :options="preferenceOptions" />
      </div>

      <div class="checkbox-area">
        <label class="field-label">饮食偏好</label>
        <a-checkbox-group
          v-model:value="formState.dietaryPreferences"
          :options="dietaryOptions"
        />
      </div>
    </div>

    <div class="planner-card">
      <div class="section-title">
        <span class="section-title__icon">💬</span>
        <span>额外要求</span>
      </div>
      <a-textarea
        v-model:value="formState.notes"
        :rows="4"
        placeholder="输入想要保留的偏好、节奏和备注"
      />
    </div>

    <div class="submit-panel">
      <button
        class="submit-panel__button"
        :disabled="isSubmitting"
        @click="handleSubmit"
      >
        {{ isSubmitting ? "正在生成中..." : "开始规划" }}
      </button>
      <div class="submit-panel__status">
        当前已接上 `/trip/generate`，生成成功后会直接展示真实 itinerary。
      </div>
    </div>
  </section>
</template>

<style scoped>
.home-page {
  display: grid;
  gap: 18px;
}

/* ====== Card Base ====== */
.planner-card {
  padding: 24px;
  border-radius: 16px;
  background: #ffffff;
  border: 1px solid #e8e8e8;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.06);
  transition: border-color 0.3s ease, box-shadow 0.3s ease;
}

.planner-card:hover {
  border-color: #d0d0d0;
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.08);
}

/* ====== Section Title ====== */
.section-title {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-bottom: 18px;
  padding-bottom: 14px;
  border-bottom: 1px solid #f0f0f0;
  color: #1677ff;
  font-size: 15px;
  font-weight: 700;
  letter-spacing: 0.04em;
  text-transform: uppercase;
}

.section-title__icon {
  font-size: 18px;
}

/* ====== Form Fields ====== */
.field-label {
  display: block;
  margin-bottom: 8px;
  color: #8c8c8c;
  font-size: 12px;
  font-weight: 600;
  letter-spacing: 0.04em;
  text-transform: uppercase;
}

/* ====== Day Count Pill ====== */
.pill-box {
  display: flex;
  align-items: center;
  justify-content: center;
  min-height: 32px;
  border-radius: 8px;
  background: rgba(22, 119, 255, 0.08);
  border: 1px solid rgba(22, 119, 255, 0.2);
  color: #1677ff;
  font-weight: 700;
  font-size: 14px;
}

.checkbox-area {
  margin-top: 18px;
}

/* ====== Submit Panel ====== */
.submit-panel {
  padding: 12px 8px 0;
  text-align: center;
}

.submit-panel__button {
  min-width: 240px;
  border: 1px solid #1677ff;
  border-radius: 8px;
  padding: 16px 32px;
  background: #1677ff;
  color: #ffffff;
  font-size: 15px;
  font-weight: 700;
  letter-spacing: 0.06em;
  text-transform: uppercase;
  cursor: pointer;
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
}

.submit-panel__button:hover:not(:disabled) {
  transform: translateY(-2px);
  background: #4096ff;
  border-color: #4096ff;
  box-shadow: 0 4px 16px rgba(22, 119, 255, 0.3);
}

.submit-panel__button:active:not(:disabled) {
  transform: translateY(0) scale(0.98);
  transition: all 0.1s ease;
}

.submit-panel__button:disabled {
  opacity: 0.4;
  cursor: wait;
}

.submit-panel__status {
  margin-top: 12px;
  color: #bfbfbf;
  font-size: 12px;
  letter-spacing: 0.03em;
}
</style>
