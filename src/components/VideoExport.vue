<script setup lang="ts">
import { ref, computed, onUnmounted } from "vue";

interface Props {
  videoFile: File | null;
  startTime: number;
  endTime: number;
}

const props = defineProps<Props>();

const isExporting = ref(false);
const exportProgress = ref(0);
const exportedVideoUrl = ref<string | null>(null);
const errorMessage = ref<string | null>(null);
const supportedFormat = ref(true);

// 檢查瀏覽器支持的視頻格式
const checkVideoSupport = () => {
  const video = document.createElement("video");
  return {
    mp4: video.canPlayType("video/mp4"),
    webm: video.canPlayType("video/webm"),
    ogg: video.canPlayType("video/ogg"),
    h264: video.canPlayType('video/mp4; codecs="avc1.42E01E"'), // H.264 基本檔次
    h265: video.canPlayType('video/mp4; codecs="hev1"'), // H.265/HEVC
    vp8: video.canPlayType('video/webm; codecs="vp8"'), // VP8
    vp9: video.canPlayType('video/webm; codecs="vp9"'), // VP9
  };
};

const videoSupport = checkVideoSupport();

const clipDuration = computed(() => {
  return props.endTime - props.startTime;
});

// 檢查文件類型是否被支持
const checkFileTypeSupport = (file: File) => {
  const fileType = file.type.toLowerCase();

  // 檢查是否為視頻文件
  if (!fileType.startsWith("video/")) {
    console.warn("非視頻文件類型:", fileType);
    return false;
  }

  // 對於可能的無MIME類型情況，根據副檔名判斷
  if (!fileType && file.name) {
    const extension = file.name.split(".").pop()?.toLowerCase();
    if (extension === "mp4" && videoSupport.mp4) return true;
    if (extension === "webm" && videoSupport.webm) return true;
    if (extension === "ogg" && videoSupport.ogg) return true;

    console.warn("未知文件類型，但副檔名為:", extension);
    return false;
  }

  // 標準MIME類型檢查
  if (fileType.includes("mp4") && !videoSupport.mp4) {
    console.warn("MP4格式不支持");
    return false;
  }
  if (fileType.includes("webm") && !videoSupport.webm) {
    console.warn("WebM格式不支持");
    return false;
  }
  if (fileType.includes("ogg") && !videoSupport.ogg) {
    console.warn("OGG格式不支持");
    return false;
  }

  return true;
};

// 當組件屬性變化時檢查文件支持情況
const checkCurrentFileSupport = () => {
  if (!props.videoFile) {
    supportedFormat.value = true;
    return;
  }

  supportedFormat.value = checkFileTypeSupport(props.videoFile);
};

// 監視視頻文件變化
const watchVideoFile = () => {
  checkCurrentFileSupport();
};

// 初始檢查
watchVideoFile();

const exportVideo = async () => {
  if (!props.videoFile || !clipDuration.value) {
    errorMessage.value = "請先上傳視頻並設置剪輯起止點";
    return;
  }

  if (!supportedFormat.value) {
    errorMessage.value = "當前瀏覽器不支持此視頻格式，請嘗試轉換為 MP4、WebM 或 OGG 格式";
    return;
  }

  try {
    isExporting.value = true;
    errorMessage.value = null;
    exportProgress.value = 0;

    // 首先，創建一個視頻元素來播放原始視頻
    const videoElement = document.createElement("video");
    videoElement.src = URL.createObjectURL(props.videoFile);
    videoElement.muted = false; // 取消靜音以捕獲音頻

    // 加載元數據時特別處理
    let metadataLoaded = false;

    // 等待視頻元數據加載
    await new Promise<void>((resolve, reject) => {
      videoElement.onloadedmetadata = () => {
        metadataLoaded = true;
        console.log(
          "視頻元數據載入完成:",
          videoElement.videoWidth,
          "x",
          videoElement.videoHeight
        );
        resolve();
      };
      videoElement.onerror = (e) => {
        console.error("視頻載入失敗:", videoElement.error);
        reject(new Error(`視頻加載失敗: ${videoElement.error?.message || "未知錯誤"}`));
      };
      // 設置超時處理
      setTimeout(() => {
        if (!metadataLoaded) {
          reject(new Error("視頻元數據加載超時"));
        }
      }, 10000); // 10秒超時

      videoElement.load();
    });

    if (!metadataLoaded) {
      throw new Error("視頻元數據無法載入");
    }

    // 創建一個畫布
    const canvas = document.createElement("canvas");
    const ctx = canvas.getContext("2d");
    if (!ctx) {
      throw new Error("無法創建畫布上下文");
    }

    // 設置畫布尺寸與視頻尺寸相同
    canvas.width = videoElement.videoWidth || 640; // 預設寬度
    canvas.height = videoElement.videoHeight || 360; // 預設高度

    // 創建音頻上下文
    const audioContext = new AudioContext();
    const audioSource = audioContext.createMediaElementSource(videoElement);
    const audioDestination = audioContext.createMediaStreamDestination();
    audioSource.connect(audioDestination);

    // 檢查MediaRecorder支持
    if (!window.MediaRecorder) {
      throw new Error("您的瀏覽器不支持MediaRecorder API，無法導出視頻");
    }

    // 檢查支持的MIME類型并選擇最佳格式
    const checkMimeTypeSupport = (mimeType: string) => {
      try {
        return MediaRecorder.isTypeSupported(mimeType);
      } catch (e) {
        console.warn(`檢查MIME類型支持失敗: ${mimeType}`, e);
        return false;
      }
    };

    // 按優先順序嘗試不同的MIME類型
    const mimeTypes = [
      "video/webm;codecs=vp9,opus",
      "video/webm;codecs=vp8,opus",
      "video/webm;codecs=vp9",
      "video/webm;codecs=vp8",
      "video/webm",
      "video/mp4;codecs=h264,aac",
      "video/mp4",
    ];

    let selectedMimeType = "";
    for (const mimeType of mimeTypes) {
      if (checkMimeTypeSupport(mimeType)) {
        selectedMimeType = mimeType;
        console.log("選擇的MIME類型:", selectedMimeType);
        break;
      }
    }

    if (!selectedMimeType) {
      console.warn("未找到支持的MIME類型，使用默認");
    }

    // 配置MediaRecorder選項
    const recorderOptions: MediaRecorderOptions = {
      videoBitsPerSecond: 5000000, // 5Mbps
    };

    if (selectedMimeType) {
      recorderOptions.mimeType = selectedMimeType;
    }

    // 合併視頻和音頻流
    const videoStream = canvas.captureStream(30); // 30fps
    const combinedStream = new MediaStream([
      ...videoStream.getVideoTracks(),
      ...audioDestination.stream.getAudioTracks(),
    ]);

    let mediaRecorder: MediaRecorder;
    try {
      mediaRecorder = new MediaRecorder(combinedStream, recorderOptions);
    } catch (e) {
      console.error("創建MediaRecorder失敗:", e);

      // 嘗試創建沒有指定MIME類型的MediaRecorder
      try {
        mediaRecorder = new MediaRecorder(combinedStream);
      } catch (fallbackError) {
        throw new Error("瀏覽器不支持視頻錄製功能，請嘗試其他瀏覽器");
      }
    }

    const chunks: Blob[] = [];
    mediaRecorder.ondataavailable = (e) => {
      if (e.data.size > 0) {
        chunks.push(e.data);
      }
    };

    mediaRecorder.onstop = () => {
      try {
        // 合併所有錄製的塊
        const mimeType = selectedMimeType || "video/webm";
        const blob = new Blob(chunks, { type: mimeType });

        // 創建導出 URL
        if (exportedVideoUrl.value) {
          URL.revokeObjectURL(exportedVideoUrl.value);
        }
        exportedVideoUrl.value = URL.createObjectURL(blob);
        isExporting.value = false;
        exportProgress.value = 100;
      } catch (error) {
        console.error("創建輸出視頻文件失敗:", error);
        errorMessage.value = "創建輸出視頻文件失敗";
        isExporting.value = false;
      }
    };

    // 開始錄製
    mediaRecorder.start(1000); // 每秒收集一次數據

    // 將視頻定位到起始點
    videoElement.currentTime = props.startTime;

    // 一旦視頻到達指定位置，我們開始繪製幀
    videoElement.onseeked = async () => {
      // 繪製當前視頻幀到畫布
      const drawFrame = () => {
        if (videoElement.readyState >= 2) {
          // 確保視頻已加載
          ctx.drawImage(videoElement, 0, 0, canvas.width, canvas.height);
        }

        // 更新進度
        const currentProgress =
          ((videoElement.currentTime - props.startTime) / clipDuration.value) * 100;
        exportProgress.value = Math.min(99, currentProgress); // 保持在99%以下，直到完成
      };

      // 開始播放
      try {
        await videoElement.play();
      } catch (error) {
        console.error("視頻播放失敗:", error);
        throw new Error("視頻播放失敗，請確保視頻格式受支持");
      }

      // 每幀繪製
      const drawInterval = setInterval(() => {
        // 如果視頻已經播放到結束位置，停止繪製
        if (videoElement.currentTime >= props.endTime) {
          clearInterval(drawInterval);
          videoElement.pause();

          // 至少延遲100毫秒停止錄製，確保所有幀被捕獲
          setTimeout(() => {
            try {
              mediaRecorder.stop();
            } catch (e) {
              console.error("停止MediaRecorder時出錯:", e);
              errorMessage.value = "導出過程中發生錯誤";
              isExporting.value = false;
            }
          }, 100);
        } else {
          drawFrame();
        }
      }, 1000 / 30); // 30fps
    };
  } catch (error) {
    console.error("導出視頻時出錯:", error);
    errorMessage.value = `導出失敗: ${
      error instanceof Error ? error.message : "未知錯誤"
    }`;
    isExporting.value = false;
  }
};

const downloadVideo = () => {
  if (!exportedVideoUrl.value) return;

  const a = document.createElement("a");
  a.href = exportedVideoUrl.value;
  a.download = `clip_${new Date().getTime()}.webm`;
  document.body.appendChild(a);
  a.click();
  document.body.removeChild(a);
};

// 當組件卸載時清理URL
const cleanupUrls = () => {
  if (exportedVideoUrl.value) {
    URL.revokeObjectURL(exportedVideoUrl.value);
  }
};

onUnmounted(cleanupUrls);
</script>

<template>
  <div class="video-export-container">
    <div class="export-header">
      <h3>導出剪輯視頻</h3>
      <p class="clip-info">
        剪輯長度: {{ Math.floor(clipDuration / 60) }}:{{
          Math.floor(clipDuration % 60)
            .toString()
            .padStart(2, "0")
        }}
      </p>
    </div>

    <div v-if="errorMessage" class="error-message">
      {{ errorMessage }}
    </div>

    <div v-if="!supportedFormat && props.videoFile" class="warning-message">
      <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24">
        <path
          fill="currentColor"
          d="M12 2C6.48 2 2 6.48 2 12s4.48 10 10 10 10-4.48 10-10S17.52 2 12 2zm1 15h-2v-2h2v2zm0-4h-2V7h2v6z"
        />
      </svg>
      <span>您的瀏覽器可能不支持當前視頻格式，建議轉換為MP4、WebM或OGG格式</span>
    </div>

    <div class="export-actions">
      <button
        class="export-button"
        @click="exportVideo"
        :disabled="isExporting || !videoFile || !clipDuration"
      >
        <svg
          v-if="!isExporting"
          xmlns="http://www.w3.org/2000/svg"
          width="20"
          height="20"
          viewBox="0 0 24 24"
        >
          <path fill="currentColor" d="M5 20h14v-2H5v2zm0-10h4v6h6v-6h4l-7-7-7 7z" />
        </svg>
        <span v-if="!isExporting">導出視頻</span>
        <span v-else>導出中... {{ Math.floor(exportProgress) }}%</span>
      </button>

      <button v-if="exportedVideoUrl" class="download-button" @click="downloadVideo">
        <svg
          xmlns="http://www.w3.org/2000/svg"
          width="20"
          height="20"
          viewBox="0 0 24 24"
        >
          <path fill="currentColor" d="M19 9h-4V3H9v6H5l7 7 7-7zM5 18v2h14v-2H5z" />
        </svg>
        下載視頻
      </button>
    </div>

    <div v-if="isExporting" class="progress-bar-container">
      <div class="progress-bar" :style="{ width: exportProgress + '%' }"></div>
    </div>

    <div v-if="exportedVideoUrl" class="preview-container">
      <h4>預覽</h4>
      <video class="preview-video" :src="exportedVideoUrl" controls></video>
    </div>

    <div class="format-info">
      <p>支持的視頻格式:</p>
      <ul class="format-list">
        <li :class="{ supported: videoSupport.mp4 }">
          MP4 {{ videoSupport.mp4 ? "✓" : "✗" }}
        </li>
        <li :class="{ supported: videoSupport.webm }">
          WebM {{ videoSupport.webm ? "✓" : "✗" }}
        </li>
        <li :class="{ supported: videoSupport.ogg }">
          OGG {{ videoSupport.ogg ? "✓" : "✗" }}
        </li>
      </ul>
    </div>
  </div>
</template>

<style scoped>
.video-export-container {
  width: 100%;
  background-color: #ffffff;
  border-radius: 8px;
  padding: 16px;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.1);
}

.export-header {
  margin-bottom: 16px;
}

.export-header h3 {
  margin: 0 0 8px 0;
  color: #333;
  font-size: 18px;
}

.clip-info {
  font-size: 14px;
  color: #666;
  margin: 0;
}

.error-message {
  padding: 12px;
  background-color: #ffebee;
  color: #d32f2f;
  border-radius: 4px;
  margin-bottom: 16px;
  font-size: 14px;
}

.warning-message {
  padding: 12px;
  background-color: #fff8e1;
  color: #f57c00;
  border-radius: 4px;
  margin-bottom: 16px;
  font-size: 14px;
  display: flex;
  align-items: center;
  gap: 8px;
}

.export-actions {
  display: flex;
  gap: 12px;
  margin-bottom: 16px;
}

.export-button,
.download-button {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 10px 16px;
  border-radius: 4px;
  border: none;
  font-weight: 500;
  cursor: pointer;
  transition: background-color 0.2s;
}

.export-button {
  background-color: #42b883;
  color: white;
}

.export-button:hover:not(:disabled) {
  background-color: #3a9d71;
}

.export-button:disabled {
  background-color: #cccccc;
  cursor: not-allowed;
}

.download-button {
  background-color: #3498db;
  color: white;
}

.download-button:hover {
  background-color: #2980b9;
}

.progress-bar-container {
  height: 6px;
  background-color: #e0e0e0;
  border-radius: 3px;
  overflow: hidden;
  margin-bottom: 16px;
}

.progress-bar {
  height: 100%;
  background-color: #42b883;
  transition: width 0.2s;
}

.preview-container {
  margin-top: 24px;
}

.preview-container h4 {
  margin: 0 0 12px 0;
  font-size: 16px;
  color: #333;
}

.preview-video {
  width: 100%;
  border-radius: 4px;
  background-color: #000;
}

.format-info {
  margin-top: 24px;
  font-size: 14px;
  color: #666;
}

.format-list {
  display: flex;
  gap: 16px;
  list-style: none;
  padding: 0;
  margin: 8px 0 0 0;
}

.format-list li {
  display: flex;
  align-items: center;
  gap: 4px;
}

.format-list li.supported {
  color: #42b883;
  font-weight: 500;
}
</style>
