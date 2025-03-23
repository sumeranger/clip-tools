<script setup lang="ts">
import { ref } from "vue";

const emit = defineEmits<{
  "file-selected": [file: File];
}>();

const fileInput = ref<HTMLInputElement | null>(null);
const dragActive = ref(false);
const fileName = ref("");

const handleFileChange = (event: Event) => {
  const target = event.target as HTMLInputElement;
  if (target.files && target.files.length > 0) {
    const file = target.files[0];
    fileName.value = file.name;
    emit("file-selected", file);
  }
};

const handleDragOver = (e: DragEvent) => {
  e.preventDefault();
  dragActive.value = true;
};

const handleDragLeave = () => {
  dragActive.value = false;
};

const handleDrop = (e: DragEvent) => {
  e.preventDefault();
  dragActive.value = false;

  if (e.dataTransfer?.files && e.dataTransfer.files.length > 0) {
    const file = e.dataTransfer.files[0];
    if (file.type.startsWith("video/")) {
      fileName.value = file.name;
      emit("file-selected", file);
    }
  }
};

const triggerFileInput = () => {
  fileInput.value?.click();
};
</script>

<template>
  <div class="upload-container">
    <div
      class="upload-area"
      :class="{ 'drag-active': dragActive }"
      @dragover="handleDragOver"
      @dragleave="handleDragLeave"
      @drop="handleDrop"
      @click="triggerFileInput"
    >
      <div class="upload-icon">
        <svg
          xmlns="http://www.w3.org/2000/svg"
          width="48"
          height="48"
          viewBox="0 0 24 24"
        >
          <path
            fill="currentColor"
            d="M19.35 10.04A7.49 7.49 0 0 0 12 4C9.11 4 6.6 5.64 5.35 8.04A5.994 5.994 0 0 0 0 14c0 3.31 2.69 6 6 6h13c2.76 0 5-2.24 5-5c0-2.64-2.05-4.78-4.65-4.96zM14 13v4h-4v-4H7l5-5l5 5h-3z"
          />
        </svg>
      </div>
      <p class="upload-text">將視頻文件拖放到此處或點擊上傳</p>
      <p v-if="fileName" class="file-name">已選擇: {{ fileName }}</p>
      <input
        ref="fileInput"
        type="file"
        accept="video/*"
        class="file-input"
        @change="handleFileChange"
      />
    </div>
  </div>
</template>

<style scoped>
.upload-container {
  width: 100%;
  margin: 0 auto;
}

.upload-area {
  border: 2px dashed #ccc;
  border-radius: 8px;
  padding: 40px 20px;
  text-align: center;
  cursor: pointer;
  transition: all 0.3s ease;
  background-color: #f9f9f9;
}

.upload-area:hover {
  border-color: #42b883;
  background-color: #f0f0f0;
}

.drag-active {
  border-color: #42b883;
  background-color: rgba(66, 184, 131, 0.1);
}

.upload-icon {
  color: #888;
  margin-bottom: 16px;
}

.upload-text {
  font-size: 16px;
  color: #555;
  margin-bottom: 8px;
}

.file-name {
  font-size: 14px;
  color: #42b883;
  margin-top: 16px;
  font-weight: 500;
}

.file-input {
  display: none;
}
</style>
