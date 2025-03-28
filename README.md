# 視頻剪輯工具

一個簡單、高效的前端視頻剪輯工具，使用 Vue 3 + TypeScript 構建。

## 功能特點

- 🎬 上傳本地視頻文件
- 👁️ 即時預覽視頻內容
- ✂️ 精確設置剪輯起止點
- 💾 導出並下載剪輯後的視頻

## 技術棧

- Vue 3
- TypeScript
- Vite
- Vue Router
- Pinia

## 安裝與使用

### 前提條件

- Node.js (v16+)
- Yarn 包管理器

### 安裝依賴

```bash
# 進入項目目錄
cd clip-tools

# 安裝依賴
yarn
```

### 開發環境運行

```bash
yarn dev
```

### 生產環境構建

```bash
yarn build
```

## 使用指南

1. 上傳視頻：點擊上傳區域或將視頻文件拖放到上傳區域
2. 預覽視頻：上傳成功後自動顯示視頻預覽
3. 設置剪輯點：
   - 使用進度條上的藍色和紅色標記調整開始和結束位置
   - 或使用"設置開始點"和"設置結束點"按鈕在當前播放位置設置剪輯點
4. 導出視頻：點擊"導出視頻"按鈕處理剪輯
5. 下載視頻：處理完成後點擊"下載視頻"保存到本地

## 瀏覽器兼容性

該工具使用現代 Web API 進行視頻處理，推薦使用以下瀏覽器的最新版本：

- Google Chrome
- Mozilla Firefox
- Microsoft Edge
- Safari

## 注意事項

- 視頻處理完全在瀏覽器中進行，不會上傳到任何伺服器
- 較大的視頻文件可能需要更多的處理時間和瀏覽器資源

## 許可證

MIT
