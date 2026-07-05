<script setup lang="ts">
import { computed, ref, watch } from "vue";
import { message } from "ant-design-vue";

import AmapTripMap from "../components/AmapTripMap.vue";
import {
  editTrip,
  fetchWeatherForecast,
  getMarkdownExportUrl,
  getPdfExportUrl,
  saveTrip,
} from "../services/api";
import type { Itinerary, WeatherForecastResponse } from "../types";

const props = defineProps<{
  itinerary: Itinerary | null;
}>();

const emit = defineEmits<{
  backHome: [];
  viewHistory: [];
  updated: [itinerary: Itinerary];
}>();

const saving = ref(false);
const exportingPdf = ref(false);
const exportingMarkdown = ref(false);
const editing = ref(false);
const editScope = ref("day_1");
const editInstruction = ref("这一天节奏更轻松一点，减少固定安排。");
const weatherLoading = ref(false);
const weatherError = ref("");
const weather = ref<WeatherForecastResponse | null>(null);

function formatShortDate(dateText?: string | null): string {
  if (!dateText) {
    return "待定";
  }

  const parts = dateText.split("-");
  if (parts.length !== 3) {
    return dateText;
  }

  return `${parts[1]}-${parts[2]}`;
}

function formatWeatherDate(dateText?: string | null, week?: string | null): string {
  const weekdayMap: Record<string, string> = {
    "1": "周一",
    "2": "周二",
    "3": "周三",
    "4": "周四",
    "5": "周五",
    "6": "周六",
    "7": "周日",
  };
  const weekday = week ? weekdayMap[week] || `周${week}` : "";
  return [formatShortDate(dateText), weekday].filter(Boolean).join(" ");
}

const budgetItems = computed(() => {
  if (!props.itinerary) {
    return [];
  }

  const budget = props.itinerary.budget_breakdown;
  return [
    { label: "景点门票", value: `¥${budget.tickets.toFixed(0)}` },
    { label: "酒店住宿", value: `¥${budget.hotel.toFixed(0)}` },
    { label: "餐饮费用", value: `¥${budget.meals.toFixed(0)}` },
    { label: "交通费用", value: `¥${budget.transport.toFixed(0)}` },
  ];
});

const dayBudgetItems = computed(() => {
  if (!props.itinerary) {
    return [];
  }

  return props.itinerary.days.map((day) => {
    const tickets = day.spots.reduce((sum, spot) => sum + (spot.estimated_cost ?? 0), 0);
    const meals = day.meals.reduce((sum, meal) => sum + (meal.estimated_cost ?? 0), 0);
    const transport = day.transport.reduce((sum, item) => sum + (item.estimated_cost ?? 0), 0);
    const hotel = day.hotel?.estimated_cost ?? 0;
    const total = tickets + meals + transport + hotel;

    return {
      key: day.day_index,
      title: `第${day.day_index}天`,
      subtitle: day.theme || "未命名主题",
      tickets,
      meals,
      transport,
      hotel,
      total,
    };
  });
});

const mapPoints = computed(() => {
  if (!props.itinerary) {
    return [];
  }

  return props.itinerary.days.flatMap((day) =>
    day.spots.map((spot) => ({
      key: `${day.day_index}-${spot.name}`,
      dayIndex: day.day_index,
      date: day.date || "待定",
      theme: day.theme || "未命名主题",
      name: spot.name,
      address: spot.address || spot.location || "待补充",
      latitude: spot.latitude,
      longitude: spot.longitude,
      poiId: spot.poi_id,
      imageUrl: spot.image_url,
      description: spot.description || "暂无说明",
    }))
  );
});

const technicalTipKeywords = [
  "LLM",
  "RAG",
  "LangChain",
  "Chroma",
  "演示",
  "测试",
  "规则",
  "模型",
  "源码",
];

const rainWeatherKeywords = ["雨", "阵雨", "雷阵雨", "小雨", "中雨", "大雨"];
const sunnyTipKeywords = ["防晒", "太阳", "日照", "晒"];

const weatherText = computed(() => {
  if (!weather.value) {
    return "";
  }

  return weather.value.days
    .map((day) => `${day.day_weather || ""}${day.night_weather || ""}`)
    .join(" ");
});

const hasRainyWeather = computed(() => {
  return rainWeatherKeywords.some((keyword) => weatherText.value.includes(keyword));
});

const displayTips = computed(() => {
  if (!props.itinerary) {
    return [];
  }

  const tips = props.itinerary.tips
    .map((tip) => tip.trim())
    .filter(Boolean)
    .filter((tip) => !technicalTipKeywords.some((keyword) => tip.includes(keyword)));

  const weatherAwareTips = hasRainyWeather.value
    ? tips.filter((tip) => !sunnyTipKeywords.some((keyword) => tip.includes(keyword)))
    : tips;

  if (hasRainyWeather.value) {
    weatherAwareTips.push("天气可能有雨，建议随身带伞或轻便雨衣。");
    weatherAwareTips.push("阴雨天路面湿滑，洱海边和古镇石板路建议穿防滑鞋。");
  }

  const uniqueTips = Array.from(new Set(weatherAwareTips));
  if (uniqueTips.length) {
    return uniqueTips;
  }

  return [
    `建议根据${props.itinerary.destination}当天实时天气准备雨具或薄外套。`,
    "古镇、生态廊道和石板路更适合慢慢走，鞋子尽量选择舒适防滑的款式。",
  ];
});

function buildVisibleItinerary(): Itinerary | null {
  if (!props.itinerary) {
    return null;
  }

  return {
    ...props.itinerary,
    tips: displayTips.value,
  };
}

async function loadWeather() {
  if (!props.itinerary?.destination) {
    weather.value = null;
    return;
  }

  weatherLoading.value = true;
  weatherError.value = "";
  try {
    weather.value = await fetchWeatherForecast(props.itinerary.destination);
  } catch (error) {
    console.error(error);
    weather.value = null;
    weatherError.value = "天气信息加载失败。";
  } finally {
    weatherLoading.value = false;
  }
}

watch(
  () => props.itinerary?.destination,
  () => {
    void loadWeather();
  },
  { immediate: true }
);

watch(
  () => props.itinerary?.trip_id,
  () => {
    const firstDay = props.itinerary?.days[0];
    editScope.value = firstDay ? `day_${firstDay.day_index}` : "day_1";
  },
  { immediate: true }
);

async function openPdfExport() {
  const itineraryToExport = buildVisibleItinerary();
  if (!itineraryToExport) {
    return;
  }

  const exportWindow = window.open("about:blank", "_blank");
  exportingPdf.value = true;
  try {
    await saveTrip(itineraryToExport);
    const exportUrl = getPdfExportUrl(itineraryToExport.trip_id);
    if (exportWindow) {
      exportWindow.location.href = exportUrl;
    } else {
      window.location.href = exportUrl;
    }
  } catch (error) {
    console.error(error);
    exportWindow?.close();
    message.error("导出 PDF 前同步当前行程失败。");
  } finally {
    exportingPdf.value = false;
  }
}

async function openMarkdownExport() {
  const itineraryToExport = buildVisibleItinerary();
  if (!itineraryToExport) {
    return;
  }

  const exportWindow = window.open("about:blank", "_blank");
  exportingMarkdown.value = true;
  try {
    await saveTrip(itineraryToExport);
    const exportUrl = getMarkdownExportUrl(itineraryToExport.trip_id);
    if (exportWindow) {
      exportWindow.location.href = exportUrl;
    } else {
      window.location.href = exportUrl;
    }
  } catch (error) {
    console.error(error);
    exportWindow?.close();
    message.error("导出 Markdown 前同步当前行程失败。");
  } finally {
    exportingMarkdown.value = false;
  }
}

async function handleSave() {
  const itineraryToSave = buildVisibleItinerary();
  if (!itineraryToSave) {
    return;
  }

  saving.value = true;
  try {
    await saveTrip(itineraryToSave);
    message.success("行程已保存，可以去历史列表查看。");
  } catch (error) {
    console.error(error);
    message.error("保存行程失败。");
  } finally {
    saving.value = false;
  }
}

async function handleEdit() {
  if (!props.itinerary) {
    return;
  }

  const instruction = editInstruction.value.trim();
  if (!instruction) {
    message.warning("请先输入想如何调整行程。");
    return;
  }

  editing.value = true;
  try {
    const updatedItinerary = await editTrip({
      trip_id: props.itinerary.trip_id,
      current_itinerary: props.itinerary,
      user_instruction: instruction,
      edit_scope: editScope.value,
      preserve_constraints: ["保留预算结构", "保留目的地和旅行日期"],
    });
    emit("updated", updatedItinerary);
    message.success("行程已智能调整。");
  } catch (error) {
    console.error(error);
    message.error("智能调整失败，请稍后再试。");
  } finally {
    editing.value = false;
  }
}
</script>

<template>
  <section v-if="itinerary" class="result-page">
    <aside class="sidebar-card">
      <div class="sidebar-card__title">行程导航</div>
      <ul class="sidebar-list">
        <li>行程概览</li>
        <li>预算明细</li>
        <li>按天花费</li>
        <li>智能调整</li>
        <li>景点地图</li>
        <li>天气信息</li>
        <li>点位明细</li>
        <li>每日行程</li>
      </ul>

      <div class="sidebar-actions">
        <button class="back-button" @click="$emit('backHome')">返回规划页</button>
        <button class="save-button" :disabled="saving" @click="handleSave">
          {{ saving ? "保存中..." : "保存行程" }}
        </button>
        <button class="history-button" @click="$emit('viewHistory')">历史列表</button>
        <button class="export-button" :disabled="exportingPdf" @click="openPdfExport">
          {{ exportingPdf ? "准备 PDF..." : "导出 PDF" }}
        </button>
        <button
          class="export-button export-button--light"
          :disabled="exportingMarkdown"
          @click="openMarkdownExport"
        >
          {{ exportingMarkdown ? "准备中..." : "导出 Markdown" }}
        </button>
      </div>
    </aside>

    <div class="result-grid">
      <section class="result-card">
        <div class="result-card__title">{{ itinerary.destination }}旅行计划</div>
        <div class="info-row"><strong>行程 ID：</strong>{{ itinerary.trip_id }}</div>
        <div class="info-row">
          <strong>日期：</strong>
          {{ itinerary.days[0]?.date || "待定" }} 至
          {{ itinerary.days[itinerary.days.length - 1]?.date || "待定" }}
        </div>
        <div class="info-row"><strong>地点：</strong>{{ itinerary.destination }}</div>
        <div class="info-row summary-text">{{ itinerary.summary }}</div>
        <div v-if="displayTips.length" class="overview-tips">
          <div class="overview-tips__title">旅行提示</div>
          <ul>
            <li v-for="tip in displayTips" :key="tip">{{ tip }}</li>
          </ul>
        </div>
      </section>

      <section class="result-card">
        <div class="result-card__title">预算明细</div>
        <div class="budget-grid">
          <div v-for="item in budgetItems" :key="item.label" class="budget-box">
            <div class="budget-box__label">{{ item.label }}</div>
            <div class="budget-box__value">{{ item.value }}</div>
          </div>
        </div>
        <div class="budget-total">
          <span>预估总费用</span>
          <strong>¥{{ itinerary.estimated_budget.toFixed(0) }}</strong>
        </div>
      </section>

      <section class="result-card result-card--map">
        <div class="result-card__title">景点地图</div>
        <AmapTripMap :points="mapPoints" />
      </section>

      <section class="result-card result-card--weather">
        <div class="result-card__title">天气信息</div>

        <div v-if="weatherLoading" class="weather-state">正在加载天气信息...</div>
        <div v-else-if="weatherError" class="weather-state">{{ weatherError }}</div>
        <div v-else-if="weather" class="weather-grid">
          <article
            v-for="day in weather.days"
            :key="`${day.date}-${day.week}`"
            class="weather-card"
          >
            <div class="weather-card__date">
              {{ formatWeatherDate(day.date, day.week) }}
            </div>
            <div class="weather-card__temp">
              {{ day.day_temp || "-" }}° / {{ day.night_temp || "-" }}°
            </div>
            <div class="weather-card__desc">白天：{{ day.day_weather || "未知" }}</div>
            <div class="weather-card__desc">夜间：{{ day.night_weather || "未知" }}</div>
          </article>
        </div>
        <div v-else class="weather-state">暂无天气信息。</div>
      </section>

      <section class="result-card result-card--full">
        <div class="result-card__title">智能调整行程</div>
        <div class="edit-panel">
          <div class="edit-panel__controls">
            <label class="edit-field">
              <span>调整范围</span>
              <select v-model="editScope">
                <option
                  v-for="day in itinerary.days"
                  :key="day.day_index"
                  :value="`day_${day.day_index}`"
                >
                  第{{ day.day_index }}天 · {{ day.theme || "未命名主题" }}
                </option>
              </select>
            </label>
            <button
              class="edit-submit-button"
              :disabled="editing"
              @click="handleEdit"
            >
              {{ editing ? "调整中..." : "智能调整" }}
            </button>
          </div>
          <textarea
            v-model="editInstruction"
            class="edit-textarea"
            rows="3"
            placeholder="例如：第二天轻松一点，不要安排太满；第三天想换成适合看日落的地点。"
          ></textarea>
        </div>
      </section>

      <section class="result-card result-card--full">
        <div class="result-card__title">按天花费</div>
        <div class="day-budget-grid">
          <article
            v-for="item in dayBudgetItems"
            :key="item.key"
            class="day-budget-card"
          >
            <div class="day-budget-card__header">
              <span>{{ item.title }}</span>
              <span>{{ item.subtitle }}</span>
            </div>
            <div class="day-budget-card__body">
              <div class="day-budget-row">
                <span>门票</span>
                <strong>¥{{ item.tickets.toFixed(0) }}</strong>
              </div>
              <div class="day-budget-row">
                <span>餐饮</span>
                <strong>¥{{ item.meals.toFixed(0) }}</strong>
              </div>
              <div class="day-budget-row">
                <span>交通</span>
                <strong>¥{{ item.transport.toFixed(0) }}</strong>
              </div>
              <div class="day-budget-row">
                <span>住宿</span>
                <strong>¥{{ item.hotel.toFixed(0) }}</strong>
              </div>
              <div class="day-budget-row day-budget-row--total">
                <span>当日合计</span>
                <strong>¥{{ item.total.toFixed(0) }}</strong>
              </div>
            </div>
          </article>
        </div>
      </section>

      <section class="result-card result-card--full">
        <div class="result-card__title">地图点位明细</div>
        <div class="point-grid">
          <article v-for="point in mapPoints" :key="point.key" class="point-card">
            <div class="point-card__header">
              <span>第{{ point.dayIndex }}天 · {{ point.name }}</span>
              <span>{{ formatShortDate(point.date) }}</span>
            </div>

            <div class="point-card__body">
              <div
                v-if="point.imageUrl"
                class="point-card__image"
                :style="{ backgroundImage: `url(${point.imageUrl})` }"
              ></div>
              <div v-else class="point-card__image point-card__image--empty">
                暂无景点图片
              </div>
              <div class="point-card__line">
                <strong>主题：</strong>
                <span>{{ point.theme }}</span>
              </div>
              <div class="point-card__line">
                <strong>地址：</strong>
                <span>{{ point.address }}</span>
              </div>
              <div class="point-card__desc">{{ point.description }}</div>
            </div>
          </article>
        </div>
      </section>

      <section class="result-card result-card--full">
        <div class="result-card__title">每日行程</div>
        <div class="day-list">
          <details
            v-for="day in itinerary.days"
            :key="day.day_index"
            class="day-card"
            :open="day.day_index === 1"
          >
            <summary class="day-card__header">
              <span>第{{ day.day_index }}天 · {{ day.theme || "未命名主题" }}</span>
              <span class="day-card__meta">{{ formatShortDate(day.date) }}</span>
            </summary>

            <div class="day-card__body">
              <div class="day-card__section">
                <strong>主要景点：</strong>
                <span>{{ day.spots[0]?.name || "未安排" }}</span>
              </div>
              <div class="day-card__section">
                <strong>景点地址：</strong>
                <span>{{ day.spots[0]?.address || day.spots[0]?.location || "待补充" }}</span>
              </div>
              <div class="day-card__section">
                <strong>餐饮建议：</strong>
                <span>{{ day.meals[0]?.name || "未安排" }}</span>
              </div>
              <div class="day-card__section">
                <strong>住宿安排：</strong>
                <span>{{ day.hotel?.name || "未安排" }}</span>
              </div>
              <div class="day-card__section">
                <strong>交通信息：</strong>
                <span>
                  {{
                    day.transport[0]?.distance_km != null
                      ? `${day.transport[0].distance_km.toFixed(2)} km / ${day.transport[0].estimated_minutes ?? 0} 分钟`
                      : day.transport[0]?.duration || "待补充"
                  }}
                </span>
              </div>
              <div class="day-card__section">
                <strong>备注：</strong>
                <span>{{ day.notes[day.notes.length - 1] || "无" }}</span>
              </div>
            </div>
          </details>
        </div>
      </section>
    </div>
  </section>

  <section v-else class="empty-state">
    <div class="empty-state__card">
      <h2>还没有生成结果</h2>
      <p>先回到规划页生成一条 itinerary，结果页就会开始展示真实数据。</p>
      <button class="back-button" @click="$emit('backHome')">返回规划页</button>
    </div>
  </section>
</template>

<style scoped>
.result-page {
  display: grid;
  grid-template-columns: 200px 1fr;
  gap: 22px;
}

/* ====== Card Base ====== */
.sidebar-card,
.result-card,
.empty-state__card {
  border-radius: 14px;
  background: #ffffff;
  border: 1px solid #e8e8e8;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.06);
  transition: all 0.3s ease;
}

.result-card:hover {
  border-color: #d0d0d0;
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.08);
}

.sidebar-card {
  align-self: start;
  padding: 18px;
}

.sidebar-card__title,
.result-card__title {
  margin-bottom: 14px;
  padding: 10px 14px;
  border-radius: 8px;
  background: rgba(22, 119, 255, 0.06);
  border-left: 3px solid #1677ff;
  color: #1677ff;
  font-size: 14px;
  font-weight: 700;
  letter-spacing: 0.05em;
  text-transform: uppercase;
}

.sidebar-list {
  display: grid;
  gap: 10px;
  padding: 0;
  margin: 0 0 18px;
  list-style: none;
  color: #8c8c8c;
  font-size: 13px;
}

.sidebar-list li {
  padding: 6px 10px;
  border-radius: 6px;
  transition: all 0.2s ease;
  cursor: default;
}

.sidebar-list li:hover {
  background: rgba(22, 119, 255, 0.06);
  color: #1677ff;
}

.sidebar-actions {
  display: grid;
  gap: 8px;
}

/* ====== Sidebar Buttons ====== */
.back-button,
.save-button,
.history-button,
.export-button {
  width: 100%;
  border: 1px solid #d9d9d9;
  border-radius: 8px;
  padding: 10px 14px;
  font-size: 12px;
  font-weight: 700;
  letter-spacing: 0.04em;
  text-transform: uppercase;
  cursor: pointer;
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  background: #ffffff;
  color: #595959;
}

.back-button:hover,
.history-button:hover,
.export-button:hover {
  border-color: #1677ff;
  color: #1677ff;
  background: rgba(22, 119, 255, 0.04);
  transform: translateY(-1px);
}

.save-button {
  background: #1677ff;
  border-color: #1677ff;
  color: #ffffff;
}

.save-button:hover:not(:disabled) {
  background: #4096ff;
  border-color: #4096ff;
  box-shadow: 0 2px 8px rgba(22, 119, 255, 0.3);
  transform: translateY(-1px);
  color: #ffffff;
}

.save-button:disabled,
.export-button:disabled {
  opacity: 0.4;
  cursor: wait;
  transform: none;
}

.export-button--light {
  border-color: #d9d9d9;
  color: #52c41a;
}

.export-button--light:hover {
  border-color: #52c41a;
  color: #52c41a;
  background: rgba(82, 196, 26, 0.04);
}

.result-grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 16px;
}

.result-card {
  padding: 18px;
}

.result-card--map,
.result-card--weather {
  min-height: 330px;
}

.result-card--full {
  grid-column: 1 / -1;
}

/* ====== Info Rows ====== */
.info-row {
  margin-bottom: 10px;
  color: #595959;
  line-height: 1.7;
}

.info-row strong {
  color: #1677ff;
}

.summary-text {
  margin-top: 14px;
}

/* ====== Tips ====== */
.overview-tips {
  margin-top: 18px;
  padding: 14px 16px;
  border-radius: 10px;
  background: #fafafa;
  border: 1px solid #f0f0f0;
}

.overview-tips__title {
  margin-bottom: 8px;
  color: #1677ff;
  font-weight: 800;
  font-size: 13px;
}

.overview-tips ul {
  display: grid;
  gap: 8px;
  margin: 0;
  padding-left: 18px;
  color: #8c8c8c;
  line-height: 1.7;
}

/* ====== Budget ====== */
.budget-grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 12px;
}

.budget-box {
  padding: 16px;
  border-radius: 10px;
  background: #fafafa;
  border: 1px solid #f0f0f0;
  transition: all 0.3s ease;
}

.budget-box:hover {
  border-color: #d0d0d0;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.06);
  transform: translateY(-2px);
}

.budget-box__label {
  color: #bfbfbf;
  font-size: 12px;
  letter-spacing: 0.04em;
  text-transform: uppercase;
}

.budget-box__value {
  margin-top: 10px;
  color: #1677ff;
  font-size: 24px;
  font-weight: 700;
}

.budget-total {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-top: 14px;
  padding: 16px 18px;
  border-radius: 12px;
  background: linear-gradient(135deg, rgba(22, 119, 255, 0.08), rgba(114, 46, 209, 0.06));
  border: 1px solid rgba(22, 119, 255, 0.15);
  color: #262626;
}

.budget-total strong {
  font-size: 28px;
  color: #1677ff;
}

/* ====== Weather ====== */
.weather-state {
  color: #bfbfbf;
  line-height: 1.8;
}

.weather-grid {
  display: grid;
  gap: 12px;
}

.weather-card {
  padding: 14px;
  border-radius: 10px;
  background: #fafafa;
  border: 1px solid #f0f0f0;
  transition: all 0.3s ease;
}

.weather-card:hover {
  border-color: #d0d0d0;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.06);
}

.weather-card__date {
  color: #1677ff;
  font-weight: 700;
  font-size: 13px;
}

.weather-card__temp {
  margin: 8px 0;
  color: #13c2c2;
  font-size: 26px;
  font-weight: 800;
}

.weather-card__desc {
  color: #8c8c8c;
  line-height: 1.7;
  font-size: 13px;
}

/* ====== Edit Panel ====== */
.edit-panel {
  display: grid;
  gap: 14px;
}

.edit-panel__controls {
  display: grid;
  grid-template-columns: minmax(0, 1fr) 150px;
  gap: 12px;
  align-items: end;
}

.edit-field {
  display: grid;
  gap: 8px;
  color: #595959;
  font-weight: 700;
  font-size: 13px;
}

.edit-field select,
.edit-textarea {
  width: 100%;
  border: 1px solid #d9d9d9;
  border-radius: 8px;
  background: #ffffff;
  color: #262626;
  font: inherit;
  outline: none;
  transition: all 0.3s ease;
}

.edit-field select {
  min-height: 44px;
  padding: 0 14px;
}

.edit-textarea {
  resize: vertical;
  min-height: 92px;
  padding: 12px 14px;
  line-height: 1.7;
}

.edit-field select:focus,
.edit-textarea:focus {
  border-color: #1677ff;
  box-shadow: 0 0 0 3px rgba(22, 119, 255, 0.1);
}

.edit-submit-button {
  min-height: 44px;
  border: 1px solid #1677ff;
  border-radius: 8px;
  background: #1677ff;
  color: #ffffff;
  font-weight: 800;
  cursor: pointer;
  transition: all 0.3s ease;
  letter-spacing: 0.04em;
  text-transform: uppercase;
}

.edit-submit-button:hover:not(:disabled) {
  background: #4096ff;
  border-color: #4096ff;
  box-shadow: 0 2px 8px rgba(22, 119, 255, 0.3);
  transform: translateY(-1px);
}

.edit-submit-button:disabled {
  opacity: 0.4;
  cursor: wait;
}

/* ====== Day Budget Cards ====== */
.day-budget-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(220px, 1fr));
  gap: 14px;
}

.day-budget-card {
  border-radius: 10px;
  overflow: hidden;
  border: 1px solid #f0f0f0;
  background: #fafafa;
  transition: all 0.3s ease;
}

.day-budget-card:hover {
  border-color: #d0d0d0;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.06);
  transform: translateY(-2px);
}

.day-budget-card__header {
  display: flex;
  justify-content: space-between;
  gap: 12px;
  padding: 14px 16px;
  background: rgba(22, 119, 255, 0.06);
  color: #1677ff;
  font-weight: 700;
  font-size: 13px;
}

.day-budget-card__body {
  display: grid;
  gap: 10px;
  padding: 16px;
}

.day-budget-row {
  display: flex;
  justify-content: space-between;
  gap: 12px;
  color: #595959;
  font-size: 13px;
}

.day-budget-row--total {
  padding-top: 10px;
  border-top: 1px solid #e8e8e8;
  color: #1677ff;
  font-weight: 700;
}

/* ====== Point Cards ====== */
.point-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
  gap: 14px;
}

.point-card {
  border-radius: 10px;
  overflow: hidden;
  border: 1px solid #f0f0f0;
  background: #fafafa;
  transition: all 0.3s ease;
}

.point-card:hover {
  border-color: #d0d0d0;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.06);
  transform: translateY(-2px);
}

.point-card__header {
  display: flex;
  justify-content: space-between;
  gap: 12px;
  padding: 14px 16px;
  background: rgba(22, 119, 255, 0.06);
  color: #1677ff;
  font-weight: 700;
  font-size: 13px;
}

.point-card__body {
  display: grid;
  gap: 10px;
  padding: 16px;
}

.point-card__image {
  min-height: 150px;
  border-radius: 8px;
  background-position: center;
  background-size: cover;
  background-color: #f5f5f5;
  border: 1px solid #f0f0f0;
}

.point-card__image--empty {
  display: grid;
  place-items: center;
  color: #bfbfbf;
  font-size: 13px;
  font-weight: 600;
  background:
    repeating-linear-gradient(45deg, rgba(0, 0, 0, 0.02) 0px, rgba(0, 0, 0, 0.02) 4px, transparent 4px, transparent 12px),
    #f5f5f5;
}

.point-card__line {
  color: #595959;
  line-height: 1.7;
  font-size: 13px;
}

.point-card__desc {
  padding-top: 10px;
  border-top: 1px solid #f0f0f0;
  color: #8c8c8c;
  line-height: 1.7;
  font-size: 13px;
}

/* ====== Day Cards (Details) ====== */
.day-list {
  display: grid;
  gap: 12px;
}

.day-card {
  border-radius: 10px;
  border: 1px solid #f0f0f0;
  background: #fafafa;
  overflow: hidden;
  transition: all 0.3s ease;
}

.day-card:hover {
  border-color: #d0d0d0;
}

.day-card__header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 12px;
  padding: 14px 16px;
  background: rgba(22, 119, 255, 0.04);
  color: #1677ff;
  font-weight: 700;
  font-size: 13px;
  cursor: pointer;
  list-style: none;
  transition: background 0.2s ease;
}

.day-card__header:hover {
  background: rgba(22, 119, 255, 0.08);
}

.day-card__header::-webkit-details-marker {
  display: none;
}

.day-card__header::after {
  content: "EXPAND";
  flex: 0 0 auto;
  padding: 4px 10px;
  border-radius: 4px;
  background: rgba(22, 119, 255, 0.08);
  border: 1px solid rgba(22, 119, 255, 0.2);
  color: #1677ff;
  font-size: 10px;
  letter-spacing: 0.06em;
}

.day-card[open] .day-card__header::after {
  content: "COLLAPSE";
}

.day-card__meta {
  margin-left: auto;
  color: #bfbfbf;
  font-size: 12px;
}

.day-card__body {
  display: grid;
  gap: 10px;
  padding: 16px;
  background: #ffffff;
}

.day-card__section {
  color: #595959;
  line-height: 1.7;
  font-size: 13px;
}

.day-card__section strong {
  color: #1677ff;
}

/* ====== Empty State ====== */
.empty-state {
  display: grid;
  place-items: center;
  min-height: 360px;
}

.empty-state__card {
  max-width: 560px;
  padding: 36px;
  text-align: center;
}

.empty-state__card h2 {
  margin: 0 0 12px;
  color: #1677ff;
}

.empty-state__card p {
  margin: 0 0 18px;
  color: #8c8c8c;
  line-height: 1.7;
}

.empty-state__card .back-button {
  width: auto;
  display: inline-block;
}

@media (max-width: 960px) {
  .result-page {
    grid-template-columns: 1fr;
  }

  .result-grid {
    grid-template-columns: 1fr;
  }

  .edit-panel__controls {
    grid-template-columns: 1fr;
  }
}
</style>
