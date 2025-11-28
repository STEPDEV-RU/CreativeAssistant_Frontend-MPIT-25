<template>
  <div :class="['app-root', { dark: isDark }]">
    <!-- Theme toggle fixed to top-right -->
    <div class="topbar">
      <label class="theme-toggle" title="Переключить тему">
        <input type="checkbox" v-model="isDark" />
        <span class="toggle-ui" aria-hidden>
          <svg v-if="isDark" width="16" height="16" viewBox="0 0 24 24" fill="none">
            <path d="M21 12.79A9 9 0 1111.21 3 7 7 0 0021 12.79z" fill="currentColor"/>
          </svg>
          <svg v-else width="16" height="16" viewBox="0 0 24 24" fill="none">
            <path d="M6.76 4.84l-1.8-1.79L3.17 4.83l1.79 1.8 1.8-1.79zM1 13h3v-2H1v2zm10 9h2v-3h-2v3zm7.03-2.44l1.8 1.79 1.79-1.79-1.8-1.8-1.79 1.8zM17.24 4.84l1.79-1.79L17.24 1.2l-1.79 1.79 1.79 1.85zM21 11v2h3v-2h-3zM4.22 19.78l1.79-1.79-1.79-1.8L2.42 17.98l1.8 1.8zM12 5a7 7 0 100 14 7 7 0 000-14z" fill="currentColor"/>
          </svg>
        </span>
      </label>
    </div>

    <!-- Centered content wrapper (form + image) -->
    <div class="center-wrapper">
      <main class="main-layout">
        <!-- LEFT: форма -->
        <section class="panel form-panel">
          <h2 class="panel-title">Запрос</h2>

          <label class="field-label">Промпт</label>
          <textarea v-model="prompt" placeholder="Введите ваш промпт..." rows="6"></textarea>

          <label class="field-label">Негативный промпт (опционально)</label>
          <input v-model="negativePrompt" type="text" placeholder="Например: low quality, blurry" />

          <div class="controls-row">
            <button class="btn primary" @click="submitPrompt" :disabled="loading || !prompt.trim()">
              <span v-if="!loading">Сгенерировать</span>
              <span v-else>Генерация...</span>
            </button>

            <div class="timer" v-if="loading || elapsed !== 0">
              <svg class="timer-icon" viewBox="0 0 24 24" width="16" height="16"><path d="M12 8v5l4 2" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/></svg>
              <span>{{ formattedElapsed }}</span>
            </div>
          </div>

          <!-- loader + текст -->
          <div class="loading-area" v-if="loading">
            <div class="loader"></div>
            <div class="loading-text">
              Генерация изображения...<br />
              <small>Это может занять ~1–2 минуты</small>
            </div>
          </div>

          <!-- сообщение об ошибке -->
          <div class="error" v-if="error">{{ error }}</div>
        </section>

        <!-- RIGHT: изображение -->
        <aside class="panel image-panel">
          <h2 class="panel-title">Результат</h2>

          <div v-if="!imageUrl" class="placeholder">
            <svg width="96" height="96" viewBox="0 0 24 24" fill="none"><path d="M3 3h18v18H3z" stroke="currentColor" stroke-width="1.2" stroke-linecap="round" stroke-linejoin="round"/></svg>
            <p class="placeholder-text">Здесь появится сгенерированное изображение</p>
          </div>

          <div v-else class="result-wrap">
            <img :src="imageUrl" alt="Generated" class="result-img" />

            <div class="result-controls">
              <button class="btn secondary" @click="downloadImage" :disabled="!imageUrl">
                <!-- download icon -->
                <svg width="16" height="16" viewBox="0 0 24 24" fill="none"><path d="M12 3v12" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/><path d="M8 11l4 4 4-4" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/><path d="M5 21h14" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/></svg>
                Скачать
              </button>

              <a v-if="imageUrl" :href="imageUrl" :download="downloadFilename" class="download-link-hidden" ref="hiddenLink"></a>
            </div>

            <div class="meta">
              <div>Время генерации: <strong>{{ formattedElapsed }}</strong></div>
              <div v-if="prompt">Промпт: <em>{{ prompt }}</em></div>
              <div v-if="negativePrompt">Негативный промпт: <em>{{ negativePrompt }}</em></div>
            </div>
          </div>
        </aside>
      </main>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onUnmounted } from "vue";

const prompt = ref("");
const negativePrompt = ref("");
const loading = ref(false);
const imageUrl = ref(null); // object URL
const imageBlob = ref(null); // Blob
const error = ref("");
const isDark = ref(false);

let startTime = null;
const elapsed = ref(0); // seconds
let timerInterval = null;

const downloadFilename = "result.png";

const formattedElapsed = computed(() => {
  const total = elapsed.value;
  const hours = Math.floor(total / 3600).toString().padStart(2, "0");
  const minutes = Math.floor((total % 3600) / 60).toString().padStart(2, "0");
  const seconds = Math.floor(total % 60).toString().padStart(2, "0");
  return `${hours}:${minutes}:${seconds}`;
});

function startTimer() {
  elapsed.value = 0;
  startTime = Date.now();
  if (timerInterval) clearInterval(timerInterval);
  timerInterval = setInterval(() => {
    elapsed.value = Math.floor((Date.now() - startTime) / 1000);
  }, 250);
}

function stopTimer() {
  if (timerInterval) {
    clearInterval(timerInterval);
    timerInterval = null;
  }
  if (startTime) {
    startTime = null;
  }
}

// helper: revoke previous object URL
function revokeImageUrl() {
  if (imageUrl.value) {
    try { URL.revokeObjectURL(imageUrl.value); } catch (e) {}
    imageUrl.value = null;
  }
  imageBlob.value = null;
}

// Конструируем тело запроса: include negative_prompt только если не пусто
function buildRequestBody() {
  const body = { prompt: prompt.value };
  const neg = (negativePrompt.value || "").trim();
  if (neg) body.negative_prompt = neg;
  return body;
}

async function submitPrompt() {
  if (!prompt.value.trim()) return;
  loading.value = true;
  error.value = "";
  revokeImageUrl();

  // старт таймера
  startTimer();

  try {
    const resp = await fetch("http://localhost:5000/generate", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(buildRequestBody()),
    });

    if (!resp.ok) {
      let msg = `Ошибка сервера: ${resp.status}`;
      try {
        const j = await resp.json();
        if (j?.error) msg = j.error;
      } catch (e) {}
      throw new Error(msg);
    }

    // получаем blob изображения
    const blob = await resp.blob();
    imageBlob.value = blob;
    imageUrl.value = URL.createObjectURL(blob);

  } catch (err) {
    console.error(err);
    error.value = err?.message || "Ошибка при подключении к API";
  } finally {
    loading.value = false;
    stopTimer();
  }
}

function downloadImage() {
  if (!imageBlob.value && !imageUrl.value) return;

  const url = imageUrl.value || URL.createObjectURL(imageBlob.value);
  const a = document.createElement("a");
  a.href = url;
  a.download = downloadFilename;
  document.body.appendChild(a);
  a.click();
  a.remove();

  if (!imageUrl.value) {
    setTimeout(() => {
      try { URL.revokeObjectURL(url); } catch (e) {}
    }, 1000);
  }
}

// очистка при размонтировании
onUnmounted(() => {
  if (timerInterval) clearInterval(timerInterval);
  revokeImageUrl();
});
</script>
