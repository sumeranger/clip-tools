<script setup lang="ts">
import { ref, watch, onMounted, onUnmounted, computed } from "vue";

interface Props {
  videoFile: File | null;
  startTime: number;
  endTime: number;
}

const props = defineProps<Props>();
const emit = defineEmits<{
  "update:startTime": [time: number];
  "update:endTime": [time: number];
  "duration-change": [duration: number];
}>();

const videoElement = ref<HTMLVideoElement | null>(null);
const videoUrl = ref<string>("");
const isPlaying = ref(false);
const currentTime = ref(0);
const duration = ref(0);
const isDragging = ref(false);
const dragType = ref<"start" | "end" | "current" | null>(null);
const videoError = ref<string | null>(null);
const useVideoTag = ref(true); // 預設使用video標籤

// 格式化時間為 mm:ss.ms 格式
const formatTime = (time: number): string => {
  const minutes = Math.floor(time / 60);
  const seconds = Math.floor(time % 60);
  const ms = Math.floor((time % 1) * 10);
  return `${minutes.toString().padStart(2, "0")}:${seconds
    .toString()
    .padStart(2, "0")}.${ms}`;
};

const formattedCurrentTime = computed(() => formatTime(currentTime.value));
const formattedStartTime = computed(() => formatTime(props.startTime));
const formattedEndTime = computed(() => formatTime(props.endTime));
const formattedDuration = computed(() => formatTime(duration.value));

// 計算裁剪標記位置
const startPosition = computed(() => {
  return duration.value ? (props.startTime / duration.value) * 100 : 0;
});

const endPosition = computed(() => {
  return duration.value ? (props.endTime / duration.value) * 100 : 100;
});

const currentPosition = computed(() => {
  return duration.value ? (currentTime.value / duration.value) * 100 : 0;
});

// 釋放之前的視頻URL
const clearVideoUrl = () => {
  if (videoUrl.value) {
    URL.revokeObjectURL(videoUrl.value);
    videoUrl.value = "";
  }
};

// 嘗試使用MediaSource播放視頻
const tryMediaSourcePlayback = async (file: File) => {
  if (!window.MediaSource) {
    return false;
  }

  try {
    // MediaSource對象用於創建媒體流
    const mediaSource = new MediaSource();
    const url = URL.createObjectURL(mediaSource);
    videoUrl.value = url;

    // 創建一個FileReader來讀取文件
    const reader = new FileReader();
    let sourceBuffer: SourceBuffer;

    // 當MediaSource打開時
    const onMediaSourceOpen = () => {
      try {
        // 嘗試使用合適的MIME類型
        const mimeTypes = [
          'video/mp4; codecs="avc1.42E01E, mp4a.40.2"',
          'video/mp4; codecs="avc1.42E01E"',
          'video/mp4; codecs="avc1.4D401F"',
          'video/mp4; codecs="avc1.58A01E"',
          "video/mp4",
          'video/webm; codecs="vp8, vorbis"',
          'video/webm; codecs="vp9"',
          "video/webm",
        ];

        // 嘗試每一種MIME類型
        let mimeTypeSupported = false;
        for (const mimeType of mimeTypes) {
          if (MediaSource.isTypeSupported(mimeType)) {
            try {
              sourceBuffer = mediaSource.addSourceBuffer(mimeType);
              mimeTypeSupported = true;
              break;
            } catch (e) {
              console.warn("嘗試MIME類型失敗:", mimeType, e);
            }
          }
        }

        if (!mimeTypeSupported || !sourceBuffer) {
          throw new Error("未找到支持的MIME類型");
        }

        // 讀取完成時將數據添加到sourceBuffer
        reader.onload = () => {
          if (reader.result instanceof ArrayBuffer) {
            try {
              sourceBuffer.addEventListener("updateend", () => {
                if (!mediaSource.readyState || mediaSource.readyState === "open") {
                  try {
                    mediaSource.endOfStream();
                    if (videoElement.value) {
                      videoElement.value.play().catch((e) => {
                        console.warn("MediaSource播放失敗:", e);
                        videoError.value = "視頻播放失敗，請嘗試其他視頻";
                      });
                    }
                  } catch (e) {
                    console.warn("無法結束媒體流:", e);
                  }
                }
              });

              sourceBuffer.appendBuffer(reader.result);
            } catch (e) {
              console.warn("無法附加緩衝數據:", e);
              return false;
            }
          }
        };

        // 開始讀取文件
        reader.readAsArrayBuffer(file);
        return true;
      } catch (e) {
        console.warn("MediaSource設置失敗:", e);
        return false;
      }
    };

    // 設置MediaSource打開時的處理函數
    mediaSource.addEventListener("sourceopen", onMediaSourceOpen);

    // 給一些時間讓MediaSource初始化
    await new Promise((resolve) => setTimeout(resolve, 100));
    return true;
  } catch (e) {
    console.warn("MediaSource初始化失敗:", e);
    return false;
  }
};

// 處理視頻文件變更
watch(
  () => props.videoFile,
  async (newFile) => {
    // 重置錯誤狀態
    videoError.value = null;

    // 清理之前的URL
    clearVideoUrl();

    // 重置狀態
    currentTime.value = 0;
    duration.value = 0;
    isPlaying.value = false;
    useVideoTag.value = true; // 預設使用標準video標籤

    if (newFile) {
      try {
        // 嘗試創建對象URL（標準方法）
        videoUrl.value = URL.createObjectURL(newFile);

        // 確保視頻元素重新載入
        if (videoElement.value) {
          videoElement.value.load();

          // 給視頻元素一些時間來加載元數據
          await new Promise<void>((resolve) => {
            setTimeout(resolve, 300);
          });

          // 如果使用標準方法無法播放，嘗試MediaSource
          videoElement.value.addEventListener(
            "error",
            async () => {
              // 只嘗試一次MediaSource
              if (useVideoTag.value) {
                useVideoTag.value = false;
                console.log("標準視頻播放失敗，嘗試使用MediaSource");

                // 清理當前URL
                clearVideoUrl();

                // 嘗試使用MediaSource
                const success = await tryMediaSourcePlayback(newFile);
                if (!success) {
                  videoError.value = "視頻格式不支持或無法播放，請嘗試轉換格式";
                }
              }
            },
            { once: true }
          );
        }
      } catch (error) {
        console.error("創建視頻URL時出錯:", error);
        videoError.value = "無法載入視頻文件";
      }
    }
  },
  { immediate: true }
);

// 更新當前時間
const updateTime = () => {
  if (videoElement.value) {
    currentTime.value = videoElement.value.currentTime;

    // 當播放到結束標記時，跳回開始標記
    if (currentTime.value >= props.endTime) {
      videoElement.value.currentTime = props.startTime;
    }
  }
};

// 當視頻加載完成時
const handleVideoLoaded = () => {
  if (videoElement.value) {
    duration.value = videoElement.value.duration || 0;

    if (duration.value > 0) {
      emit("duration-change", duration.value);

      // 初始化結束時間為視頻總時長
      if (props.endTime === 0) {
        emit("update:endTime", duration.value);
      }
    } else {
      console.warn("視頻持續時間為0或未定義");
      videoError.value = "無法獲取視頻時長，請嘗試其他視頻";
    }
  }
};

// 處理視頻載入錯誤
const handleVideoError = (e: Event) => {
  console.error("視頻載入錯誤:", e);

  // 如果不是已經在嘗試MediaSource，則顯示錯誤
  if (useVideoTag.value) {
    // MediaSource錯誤處理會在上面的watch函數中處理
  } else {
    videoError.value = "視頻格式不支持或無法播放";
  }
};

// 播放/暫停按鈕點擊
const togglePlay = () => {
  if (!videoElement.value || videoError.value) return;

  if (isPlaying.value) {
    videoElement.value.pause();
    isPlaying.value = false;
  } else {
    // 如果在開始時間之前或結束時間之后，則從開始時間播放
    if (currentTime.value < props.startTime || currentTime.value >= props.endTime) {
      videoElement.value.currentTime = props.startTime;
    }

    // 使用 try-catch 捕獲播放時可能出現的錯誤
    try {
      const playPromise = videoElement.value.play();
      if (playPromise !== undefined) {
        playPromise
          .then(() => {
            isPlaying.value = true;
          })
          .catch((error) => {
            console.error("視頻播放錯誤:", error);
            videoError.value = "無法播放視頻，請嘗試其他瀏覽器";
            isPlaying.value = false;
          });
      }
    } catch (error) {
      console.error("視頻播放異常:", error);
      videoError.value = "播放視頻時出錯";
      isPlaying.value = false;
    }
  }
};

// 處理進度條點擊
const handleProgressClick = (e: MouseEvent) => {
  if (!videoElement.value || !duration.value || videoError.value) return;

  const progressBar = e.currentTarget as HTMLElement;
  const rect = progressBar.getBoundingClientRect();
  const offsetX = e.clientX - rect.left;
  const percentage = offsetX / rect.width;
  const newTime = percentage * duration.value;

  videoElement.value.currentTime = newTime;
  currentTime.value = newTime;
};

// 標記拖動事件
const handleMarkerMouseDown = (type: "start" | "end", e: MouseEvent) => {
  if (videoError.value) return;

  e.stopPropagation();
  isDragging.value = true;
  dragType.value = type;

  document.addEventListener("mousemove", handleMarkerMouseMove);
  document.addEventListener("mouseup", handleMarkerMouseUp);
};

const handleMarkerMouseMove = (e: MouseEvent) => {
  if (!isDragging.value || !dragType.value || !duration.value) return;

  const progressBar = document.querySelector(".video-progress") as HTMLElement;
  if (!progressBar) return;

  const rect = progressBar.getBoundingClientRect();
  const offsetX = Math.max(0, Math.min(e.clientX - rect.left, rect.width));
  const percentage = offsetX / rect.width;
  const newTime = Math.max(0, Math.min(percentage * duration.value, duration.value));

  if (dragType.value === "start") {
    // 確保開始時間不大於結束時間
    if (newTime < props.endTime) {
      emit("update:startTime", newTime);

      // 如果當前播放時間小於新的開始時間，則將播放頭移動到開始時間
      if (videoElement.value && currentTime.value < newTime) {
        videoElement.value.currentTime = newTime;
      }
    }
  } else if (dragType.value === "end") {
    // 確保結束時間不小於開始時間
    if (newTime > props.startTime) {
      emit("update:endTime", newTime);

      // 如果當前播放時間大於新的結束時間，則將播放頭移動到開始時間
      if (videoElement.value && currentTime.value > newTime) {
        videoElement.value.currentTime = props.startTime;
      }
    }
  }
};

const handleMarkerMouseUp = () => {
  isDragging.value = false;
  dragType.value = null;

  document.removeEventListener("mousemove", handleMarkerMouseMove);
  document.removeEventListener("mouseup", handleMarkerMouseUp);
};

// 設置開始/結束點按鈕
const setStartPoint = () => {
  if (!videoError.value) {
    emit("update:startTime", currentTime.value);
  }
};

const setEndPoint = () => {
  if (!videoError.value) {
    emit("update:endTime", currentTime.value);
  }
};

onMounted(() => {
  if (videoElement.value) {
    videoElement.value.addEventListener("timeupdate", updateTime);
    videoElement.value.addEventListener("loadedmetadata", handleVideoLoaded);
    videoElement.value.addEventListener("error", handleVideoError);
    videoElement.value.addEventListener("play", () => {
      isPlaying.value = true;
    });
    videoElement.value.addEventListener("pause", () => {
      isPlaying.value = false;
    });
  }
});

onUnmounted(() => {
  clearVideoUrl();
  if (videoElement.value) {
    videoElement.value.removeEventListener("timeupdate", updateTime);
    videoElement.value.removeEventListener("loadedmetadata", handleVideoLoaded);
    videoElement.value.removeEventListener("error", handleVideoError);
  }
});
</script>

<template>
  <div class="video-player-container">
    <div class="video-wrapper">
      <video
        ref="videoElement"
        class="video-element"
        :src="videoUrl"
        preload="metadata"
        @click="togglePlay"
        @loadedmetadata="handleVideoLoaded"
      ></video>

      <div v-if="videoError" class="video-error-overlay">
        <div class="video-error-message">
          <svg
            xmlns="http://www.w3.org/2000/svg"
            width="48"
            height="48"
            viewBox="0 0 24 24"
          >
            <path
              fill="currentColor"
              d="M12 2C6.48 2 2 6.48 2 12s4.48 10 10 10 10-4.48 10-10S17.52 2 12 2zm1 15h-2v-2h2v2zm0-4h-2V7h2v6z"
            />
          </svg>
          <p>{{ videoError }}</p>
          <p class="video-error-help">
            請嘗試其他格式的視頻文件，例如 MP4、WebM 或 OGG。某些編碼格式可能不被支持。
          </p>
          <div class="error-actions">
            <button class="error-button" @click="$emit('change-video')">更換視頻</button>
          </div>
        </div>
      </div>

      <div class="video-controls">
        <button
          class="control-button play-button"
          @click="togglePlay"
          :disabled="!!videoError"
        >
          <svg
            v-if="!isPlaying"
            xmlns="http://www.w3.org/2000/svg"
            width="24"
            height="24"
            viewBox="0 0 24 24"
          >
            <path fill="currentColor" d="M8 5v14l11-7z" />
          </svg>
          <svg
            v-else
            xmlns="http://www.w3.org/2000/svg"
            width="24"
            height="24"
            viewBox="0 0 24 24"
          >
            <path fill="currentColor" d="M6 19h4V5H6v14zm8-14v14h4V5h-4z" />
          </svg>
        </button>

        <div class="time-display">
          {{ formattedCurrentTime }} / {{ formattedDuration }}
        </div>

        <div class="progress-container">
          <div class="video-progress" @click="handleProgressClick">
            <div
              class="progress-bar-clip"
              :style="{
                left: startPosition + '%',
                width: endPosition - startPosition + '%',
              }"
            ></div>

            <div
              class="progress-bar-current"
              :style="{ width: currentPosition + '%' }"
            ></div>

            <div
              class="marker start-marker"
              :style="{ left: startPosition + '%' }"
              @mousedown="handleMarkerMouseDown('start', $event)"
            ></div>

            <div
              class="marker end-marker"
              :style="{ left: endPosition + '%' }"
              @mousedown="handleMarkerMouseDown('end', $event)"
            ></div>
          </div>
        </div>
      </div>
    </div>

    <div class="clip-controls" :class="{ disabled: videoError }">
      <div class="time-info">
        <div class="time-label">
          <span>開始時間:</span>
          <span class="time-value">{{ formattedStartTime }}</span>
        </div>
        <div class="time-label">
          <span>結束時間:</span>
          <span class="time-value">{{ formattedEndTime }}</span>
        </div>
        <div class="time-label">
          <span>剪輯長度:</span>
          <span class="time-value">{{
            formatTime(props.endTime - props.startTime)
          }}</span>
        </div>
      </div>

      <div class="marker-buttons">
        <button class="action-button" @click="setStartPoint" :disabled="!!videoError">
          設置開始點
        </button>
        <button class="action-button" @click="setEndPoint" :disabled="!!videoError">
          設置結束點
        </button>
      </div>
    </div>
  </div>
</template>

<style scoped>
.video-player-container {
  width: 100%;
  border-radius: 8px;
  overflow: hidden;
  background-color: #ffffff;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.1);
}

.video-wrapper {
  position: relative;
  width: 100%;
  background-color: #000;
  min-height: 240px;
}

.video-element {
  width: 100%;
  display: block;
}

.video-error-overlay {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(0, 0, 0, 0.7);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 5;
}

.video-error-message {
  text-align: center;
  color: white;
  padding: 20px;
  max-width: 80%;
}

.video-error-message svg {
  margin-bottom: 12px;
  color: #ff5252;
}

.video-error-help {
  margin-top: 12px;
  font-size: 14px;
  opacity: 0.8;
}

.video-controls {
  position: absolute;
  bottom: 0;
  left: 0;
  right: 0;
  background: linear-gradient(to top, rgba(0, 0, 0, 0.7), transparent);
  padding: 16px;
  display: flex;
  align-items: center;
  gap: 12px;
  z-index: 4;
}

.control-button {
  background: none;
  border: none;
  color: white;
  cursor: pointer;
  width: 40px;
  height: 40px;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 50%;
  background-color: rgba(255, 255, 255, 0.2);
  transition: background-color 0.2s;
}

.control-button:hover:not(:disabled) {
  background-color: rgba(255, 255, 255, 0.3);
}

.control-button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.time-display {
  color: white;
  font-size: 14px;
  min-width: 100px;
}

.progress-container {
  flex: 1;
  position: relative;
}

.video-progress {
  position: relative;
  height: 8px;
  background-color: rgba(255, 255, 255, 0.2);
  border-radius: 4px;
  cursor: pointer;
}

.progress-bar-current {
  position: absolute;
  top: 0;
  left: 0;
  height: 100%;
  background-color: #42b883;
  border-radius: 4px;
}

.progress-bar-clip {
  position: absolute;
  top: 0;
  height: 100%;
  background-color: rgba(66, 184, 131, 0.4);
  border-radius: 4px;
}

.marker {
  position: absolute;
  top: 50%;
  width: 12px;
  height: 12px;
  border-radius: 50%;
  transform: translate(-50%, -50%);
  cursor: pointer;
  z-index: 2;
}

.start-marker {
  background-color: #3498db;
}

.end-marker {
  background-color: #e74c3c;
}

.clip-controls {
  padding: 16px;
  display: flex;
  flex-wrap: wrap;
  justify-content: space-between;
  align-items: center;
  gap: 16px;
}

.clip-controls.disabled {
  opacity: 0.6;
}

.time-info {
  display: flex;
  flex-wrap: wrap;
  gap: 16px;
}

.time-label {
  display: flex;
  flex-direction: column;
  font-size: 14px;
}

.time-value {
  font-weight: 500;
  color: #333;
}

.marker-buttons {
  display: flex;
  gap: 8px;
}

.action-button {
  padding: 8px 16px;
  border-radius: 4px;
  border: none;
  background-color: #42b883;
  color: white;
  font-weight: 500;
  cursor: pointer;
  transition: background-color 0.2s;
}

.action-button:hover:not(:disabled) {
  background-color: #3a9d71;
}

.action-button:disabled {
  background-color: #cccccc;
  cursor: not-allowed;
}

/* 添加錯誤按鈕樣式 */
.error-actions {
  margin-top: 16px;
}

.error-button {
  background-color: white;
  color: #333;
  border: none;
  border-radius: 4px;
  padding: 8px 16px;
  font-weight: 500;
  cursor: pointer;
  transition: background-color 0.2s;
}

.error-button:hover {
  background-color: #f0f0f0;
}
</style>
