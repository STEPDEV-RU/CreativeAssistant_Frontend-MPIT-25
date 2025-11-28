<template>
  <div class="min-h-screen flex items-center justify-center bg-gray-100 p-6">
    <div class="prompt-container">

      <!-- Если не грузится — показываем форму -->
      <div v-if="!loading">
        <h1>Creative Assistant</h1>
        <textarea v-model="prompt" placeholder="Введите ваш промпт..."></textarea>
        <button @click="submitPrompt" :disabled="!prompt.trim()">
          Отправить
        </button>
      </div>

      <!-- Если грузится — показываем спиннер -->
      <div v-else class="loading-wrapper">
        <div class="loader"></div>
        <p class="loading-text">
          Генерация изображения...<br />
          <span class="loading-sub">Это может занять 1–2 минуты</span>
        </p>
      </div>

      <!-- Показываем изображение после генерации -->
      <img v-if="imageUrl" :src="imageUrl" class="result-image"  alt=""/>
    </div>

  </div>
</template>

<script setup>
import { ref } from "vue";

const prompt = ref("");
const loading = ref(false);
const imageUrl = ref(null);

const submitPrompt = async () => {
  if (!prompt.value.trim()) return;

  loading.value = true;
  imageUrl.value = null;

  try {
    const response = await fetch("http://localhost:5000/generate", {
    //const response = await fetch("http://192.168.137.1:5000/generate", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ prompt: prompt.value }),
    });

    if (!response.ok) {
      const err = await response.json();
      alert("Ошибка: " + (err.error || "неизвестная ошибка"));
      return;
    }

    // Получаем файл как Blob
    const blob = await response.blob();

    // Создаём ссылку для отображения на странице
    imageUrl.value = URL.createObjectURL(blob);

    // Если хочешь сразу скачать файл, можно так:
    const link = document.createElement("a");
    link.href = URL.createObjectURL(blob);
    link.download = "result.png";
    link.click();

  } catch (err) {
    console.error(err);
    alert("Ошибка при подключении к API");
  } finally {
    loading.value = false;
  }
};
</script>

<style>
/* Общие настройки страницы */
body {
  margin: 0;
  font-family: "Inter", sans-serif;
  background: #f3f4f6; /* светлый фон всей страницы */
  min-height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 24px;
}

/* Карточка с интерфейсом */
.prompt-container {
  width: 100%;
  max-width: 480px;
  background: #ffffff;
  border-radius: 24px;
  padding: 32px;
  box-shadow: 0 10px 30px rgba(0,0,0,0.07);
  border: 1px solid #e5e7eb;
}

/* Заголовок */
.prompt-container h1 {
  font-size: 24px;
  font-weight: 600;
  color: #1f2937;
  text-align: center;
  margin-bottom: 24px;
}

/* Текстовое поле */
.prompt-container textarea {
  width: 100%;
  height: 160px;
  padding: 14px;
  border-radius: 16px;
  border: 1px solid #d1d5db;
  outline: none;
  font-size: 15px;
  line-height: 1.5;
  color: #111827;
  background: #fafafa;
  resize: vertical;
  transition: border-color 0.25s ease, box-shadow 0.25s ease;
}

.prompt-container textarea:focus {
  border-color: #6366f1;
  box-shadow: 0 0 0 3px rgba(99,102,241,0.15);
  background: #ffffff;
}

/* Кнопка */
.prompt-container button {
  width: 100%;
  margin-top: 20px;
  padding: 12px 0;
  border: none;
  border-radius: 16px;
  font-size: 16px;
  font-weight: 600;
  color: #ffffff;
  cursor: pointer;
  background-color: #6366f1;
  transition: background-color 0.25s ease, opacity 0.25s ease;
}

.prompt-container button:hover {
  background-color: #4f46e5;
}

.prompt-container button:disabled {
  opacity: 0.55;
  cursor: not-allowed;
}

/* Анимация загрузки */
.loading-wrapper {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 24px 0;
}

.loader {
  width: 48px;
  height: 48px;
  border: 5px solid #e5e7eb;
  border-top-color: #6366f1;
  border-radius: 50%;
  animation: spin 1s linear infinite;
  margin-bottom: 16px;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

/* Текст во время загрузки */
.loading-text {
  text-align: center;
  font-size: 16px;
  color: #374151;
  line-height: 1.4;
}

.loading-sub {
  font-size: 14px;
  color: #6b7280;
}

/* Стиль результата */
.result-image {
  margin-top: 24px;
  width: 100%;
  border-radius: 16px;
  box-shadow: 0 10px 30px rgba(0,0,0,0.08);
  border: 1px solid #e5e7eb;
}
</style>

