<template>
  <div class="app">
    <h1 class="flex items-center justify-center gap-2">
      <Activity class="w-6 h-6 text-blue-400" />
      RandomTrust — прозрачный ГСЧ
    </h1>

    <div class="controls">
      <div class="grid">
        <label>Минимум</label>
        <input v-model.number="min" type="number" />
        <label>Максимум</label>
        <input v-model.number="max" type="number" />
        <label>Кол-во чисел</label>
        <input v-model.number="count" type="number" />
        <label>Источник энтропии</label>
        <select v-model="source">
          <option value="weather">Open-Meteo</option>
          <option value="donki">NASA DONKI</option>
        </select>
        <label>Скорость визуализации</label>
        <input v-model.number="speed" type="range" min="100" max="1200" step="50" />
        <span class="speed">{{ Math.round(speed) }} мс/шаг</span>
      </div>

      <div class="entropy" @mousemove="captureEntropy" @click="captureEntropyClick">
        <MousePointer class="inline w-4 h-4 text-blue-300 mr-1" />
        Двигай мышкой и кликай — добавляешь личную энтропию.
        <small>Событий: {{ entropySamples.length }}</small>
      </div>

      <div class="buttons">
        <button :disabled="loading" @click="startFlow">
          <Rocket class="inline w-4 h-4 mr-1" /> Начать
        </button>
        <button v-if="hasBits" @click="downloadBits">
          <Download class="inline w-4 h-4 mr-1" /> Скачать random_bits.bin
        </button>
        <button v-if="hasBits" @click="analyzeOnly">
          <Search class="inline w-4 h-4 mr-1" /> Перепроверить
        </button>
        <button v-if="step>0" class="ghost" @click="reset">
          <RotateCcw class="inline w-4 h-4 mr-1" /> Сброс
        </button>
      </div>
    </div>

    <div class="progress-wrap" v-if="step>0">
      <div class="progress" :style="{ width: progress + '%' }"></div>
    </div>

    <transition name="fade">
      <div v-if="step===1" class="stage">
        <h2 class="flex items-center gap-2">
          <Signal class="w-5 h-5 text-blue-400" /> Сбор энтропии
        </h2>
        <p class="muted">Внешняя ({{ sourceLabel }}), локальная (тайминги/CPU) и пользовательская (курсор/клики).</p>
        <div class="console"><div v-for="(l,i) in logs" :key="i" class="line">{{ l }}</div></div>
      </div>
    </transition>

    <transition name="fade">
      <div v-if="step===2" class="stage">
        <h2 class="flex items-center gap-2">
          <Shuffle class="w-5 h-5 text-blue-400" /> Генерация комбинации
        </h2>
        <div class="numbers">
          <span v-for="(n,i) in revealedNumbers" :key="i" class="chip">{{ n }}</span>
          <span v-if="revealedNumbers.length < (result?.numbers.length||0)" class="cursor">▌</span>
        </div>
      </div>
    </transition>

    <transition name="fade">
      <div v-if="step===3" class="stage">
        <h2 class="flex items-center gap-2">
          <FileText class="w-5 h-5 text-blue-400" /> Формирование бинарного файла
        </h2>
        <p class="muted">1 000 000 бит для аудита NIST/Dieharder.</p>
        <pre class="bits">{{ bitsPreview }}</pre>
        <small class="muted">Показаны первые {{ bitsPreview.length }} бит</small>
      </div>
    </transition>

    <transition name="fade">
      <div v-if="step===4" class="stage">
        <h2 class="flex items-center gap-2">
          <Activity class="w-5 h-5 text-blue-400" /> Аудит последовательности
        </h2>
        <div class="kpis">
          <div class="kpi"><div class="kpi-title">Доля 1</div><div class="kpi-val">{{ fmt(analysis?.ones_fraction) }}</div></div>
          <div class="kpi"><div class="kpi-title">Прогонов</div><div class="kpi-val">{{ analysis?.runs_count }}</div></div>
          <div class="kpi"><div class="kpi-title">Сред. длина</div><div class="kpi-val">{{ fmt(analysis?.avg_run_length) }}</div></div>
        </div>
        <div class="hash"><b>Хэш файла:</b> <code>{{ result?.hash }}</code></div>
      </div>
    </transition>

    <transition name="fade">
      <div v-if="step===5" class="stage final">
        <h2 class="flex items-center gap-2">
          <CheckCircle class="w-5 h-5 text-green-400" /> Результат ГСЧ
        </h2>
        <div class="numbers final-numbers">
          <span v-for="(n,i) in result?.numbers || []" :key="i" class="chip">{{ n }}</span>
        </div>
        <div class="hash"><b>Хэш:</b> <code>{{ result?.hash }}</code></div>
      </div>
    </transition>
  </div>
</template>

<script setup>
import { ref, computed } from "vue";
import {
  Activity, MousePointer, Rocket, Download, Search, RotateCcw,
  Signal, Shuffle, FileText, CheckCircle
} from "lucide-vue-next";

const min = ref(1);
const max = ref(24);
const count = ref(7);
const source = ref("weather");
const speed = ref(500);
const step = ref(0);
const progress = ref(0);
const loading = ref(false);
const entropySamples = ref([]);
const logs = ref([]);
const result = ref(null);
const revealedNumbers = ref([]);
const bitsPreview = ref("");
const analysis = ref(null);
const hasBits = ref(false);
const sourceLabel = computed(() => source.value === "donki" ? "NASA DONKI" : "Open-Meteo");

function captureEntropy(e) {
  entropySamples.value.push(`${Date.now()}_${e.clientX}_${e.clientY}`);
  if (entropySamples.value.length > 300) entropySamples.value.shift();
}
function captureEntropyClick(e) {
  entropySamples.value.push(`${Date.now()}_CLICK_${e.clientX}_${e.clientY}`);
}
const sleep = (ms) => new Promise((r) => setTimeout(r, ms));
const fmt = (v) => (v == null ? "—" : typeof v === "number" ? (v % 1 ? v.toFixed(4) : v) : v);

async function startFlow() {
  reset();
  loading.value = true;
  step.value = 1; progress.value = 8;
  logs.value.push("Сбор внешней энтропии...");
  await sleep(speed.value);
  logs.value.push("Локальная энтропия...");
  await sleep(speed.value);
  logs.value.push(`Пользовательская энтропия: ${entropySamples.value.length} событий`);
  await sleep(speed.value);
  progress.value = 20;
  const payload = {
    range_min: min.value,
    range_max: max.value,
    count: count.value,
    source: source.value,
    client_entropy: entropySamples.value.join(";"),
  };
  try {
    const res = await fetch("https://test-backend-iqyp.onrender.com/api/generate", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(payload),
    });
    const json = await res.json();
    result.value = json;
    hasBits.value = true;
    step.value = 2; progress.value = 45;
    revealedNumbers.value = [];
    for (const n of json.numbers) {
      revealedNumbers.value.push(n);
      await sleep(speed.value);
    }
    step.value = 3; progress.value = 65;
    bitsPreview.value = "";
    const bitsRes = await fetch("https://test-backend-iqyp.onrender.com/api/bits");
    const buf = await bitsRes.arrayBuffer();
    const bytes = new Uint8Array(buf);
    let bitsText = "";
    for (let b of bytes) bitsText += b.toString(2).padStart(8, "0");
    const preview = bitsText.slice(0, 128);
    for (const ch of preview) {
      bitsPreview.value += ch;
      await sleep(Math.max(30, speed.value / 10));
    }
    step.value = 4; progress.value = 82;
    const ares = await fetch("https://test-backend-iqyp.onrender.com/api/analyze");
    analysis.value = await ares.json();
    await sleep(speed.value);
    step.value = 5; progress.value = 100;
    loading.value = false;
  } catch (e) {
    loading.value = false;
    logs.value.push("Ошибка: " + e);
  }
}

async function analyzeOnly() {
  step.value = 4;
  progress.value = 82;
  logs.value.push("Повторная проверка случайности...");
  try {
    const res = await fetch("http://127.0.0.1:3000/api/analyze");
    const data = await res.json();
    analysis.value = data;
    logs.value.push("Анализ завершен");
    logs.value.push(`Доля 1: ${data.ones_fraction.toFixed(4)}`);
    logs.value.push(`Прогонов: ${data.runs_count}`);
    logs.value.push(`Средняя длина прогона: ${data.avg_run_length.toFixed(2)}`);
    progress.value = 100;
  } catch (err) {
    logs.value.push("Ошибка при проверке: " + err);
  }
}

function downloadBits() { window.open("https://test-backend-iqyp.onrender.com/api/bits", "_blank"); }
function reset() {
  step.value = 0; progress.value = 0; loading.value = false;
  logs.value = []; result.value = null; revealedNumbers.value = [];
  bitsPreview.value = ""; analysis.value = null;
}
</script>

<style>
:root { --bg:#020617; --panel:rgba(255,255,255,.06); --muted:#94a3b8; --brand:#3b82f6; }
* { box-sizing: border-box; }
body { margin:0; background: radial-gradient(circle at 50% -20%, #0b1027, #000); color:#f1f5f9; font-family: Inter, system-ui, sans-serif; min-height: 100vh;}
.app { padding: 24px 16px 60px; max-width: 1100px; margin: 0 auto; }
h1 { text-align:center; margin: 0 0 18px; font-weight: 700; }
.controls { background: var(--panel); border-radius: 16px; padding: 16px; }
.grid { display:grid; grid-template-columns: 140px 1fr 140px 1fr 140px 1fr; gap:10px 12px; align-items:center; }
.grid label { color:#cbd5e1; text-align:right; }
.grid input, .grid select { padding:8px 10px; border-radius:8px; border:1px solid #334155; background:#0b1220; color:#e5e7eb; }
.speed { color:#cbd5e1; }
.entropy { margin-top:12px; border:1px dashed #334155; border-radius:12px; padding:12px; background:rgba(255,255,255,.04); display: flex; justify-content: center; align-items: center; gap: 0.3em;}
.entropy small { color: var(--muted); margin-left:8px; }
.buttons { display:flex; gap:10px; flex-wrap:wrap; margin-top:12px; }
button { background: var(--brand); border:none; color:white; padding:10px 18px; border-radius:10px; font-weight:600; cursor:pointer; transition:.2s; display: flex; justify-content: center; align-items: center; gap: 0.3em; }
button:hover { filter: brightness(1.1); transform: translateY(-1px); }
button.ghost { background: transparent; border:1px solid #334155; color:#cbd5e1; }
.progress-wrap { height:10px; background:#0b1220; border-radius:999px; margin:18px 0; overflow:hidden; }
.progress { height:100%; background: linear-gradient(90deg, #3b82f6, #60a5fa); width:0%; transition: width .4s ease; }
.stage { background: var(--panel); border-radius:16px; padding:18px; margin: 16px 0; }
.muted { color: var(--muted); }
.console { font-family: monospace; background:#0b1220; border:1px solid #334155; border-radius:10px; padding:10px; max-height:180px; overflow:auto; }
.console .line { padding:2px 0; }
.numbers { display:flex; gap:10px; flex-wrap:wrap; justify-content:center; margin-top:8px; }
.chip { background: rgba(59,130,246,.15); border:1px solid #1d4ed8; padding:10px 12px; border-radius:12px; font-size:22px; }
.cursor { opacity:.6; animation: blink 1s infinite; }
.bits { background:#0b1220; border:1px solid #334155; border-radius:10px; padding:12px; word-break:break-word; min-height:70px; }
.kpis { display:flex; gap:12px; justify-content:center; flex-wrap:wrap; }
.kpi { background:#0b1220; border:1px solid #334155; border-radius:12px; padding:12px 16px; min-width:140px; }
.kpi-title { color:#cbd5e1; font-size:12px; }
.kpi-val { font-size:20px; font-weight:700; }
.hash { margin-top:10px; word-break: break-all; }
.final-numbers .chip { background: rgba(16,185,129,.14); border-color:#10b981; }
.fade-enter-active, .fade-leave-active { transition: opacity .35s ease, transform .35s ease; }
.fade-enter-from, .fade-leave-to { opacity:0; transform: translateY(6px); }
@keyframes blink { 0%,100%{opacity:.2} 50%{opacity:1} }
</style>
