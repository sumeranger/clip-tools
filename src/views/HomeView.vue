<script setup lang="ts">
import { ref } from "vue";
import VideoUpload from "../components/VideoUpload.vue";
import VideoPlayer from "../components/VideoPlayer.vue";
import VideoExport from "../components/VideoExport.vue";

const videoFile = ref<File | null>(null);
const startTime = ref(0);
const endTime = ref(0);

const handleFileSelected = (file: File) => {
  videoFile.value = file;
  startTime.value = 0;
  endTime.value = 0;
};

const handleDurationChange = (duration: number) => {
  endTime.value = duration;
};
</script>

<template>
  <div class="home-container">
    <header class="app-header">
      <h1 class="app-title">視頻剪輯工具</h1>
      <p class="app-subtitle">簡單、高效的在線視頻剪輯</p>
    </header>

    <main class="main-content">
      <div class="upload-section" v-if="!videoFile">
        <VideoUpload @file-selected="handleFileSelected" />
      </div>

      <div class="editor-section" v-if="videoFile">
        <div class="editor-header">
          <h2 class="section-title">視頻編輯</h2>
          <button class="change-video-button" @click="videoFile = null">更換視頻</button>
        </div>

        <VideoPlayer
          :videoFile="videoFile"
          v-model:startTime="startTime"
          v-model:endTime="endTime"
          @duration-change="handleDurationChange"
        />

        <div class="export-section">
          <VideoExport :videoFile="videoFile" :startTime="startTime" :endTime="endTime" />
        </div>
      </div>
    </main>

    <footer class="app-footer">
      <p>© {{ new Date().getFullYear() }} 視頻剪輯工具 - 輕鬆剪輯您的視頻</p>
    </footer>
  </div>
</template>

<style scoped>
.home-container {
  width: 100%;
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
  display: flex;
  flex-direction: column;
  min-height: 100vh;
}

.app-header {
  text-align: center;
  margin-bottom: 40px;
}

.app-title {
  font-size: 2.4rem;
  color: #42b883;
  margin-bottom: 8px;
  font-weight: 700;
}

.app-subtitle {
  font-size: 1.2rem;
  color: #666;
  margin: 0;
}

.main-content {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 30px;
}

.upload-section {
  max-width: 800px;
  margin: 0 auto;
  width: 100%;
}

.editor-section {
  display: flex;
  flex-direction: column;
  gap: 20px;
}

.editor-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 16px;
}

.section-title {
  font-size: 1.5rem;
  color: #333;
  margin: 0;
}

.change-video-button {
  padding: 8px 16px;
  background-color: #f0f0f0;
  border: none;
  border-radius: 4px;
  font-size: 14px;
  color: #333;
  cursor: pointer;
  transition: background-color 0.2s;
}

.change-video-button:hover {
  background-color: #e0e0e0;
}

.export-section {
  margin-top: 20px;
}

.app-footer {
  margin-top: 40px;
  text-align: center;
  padding: 16px 0;
  color: #666;
  font-size: 14px;
}
</style>
