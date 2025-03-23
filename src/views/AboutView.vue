<template>
  <div class="about-container">
    <h1>關於視頻剪輯工具</h1>

    <section class="about-section">
      <h2>項目介紹</h2>
      <p>
        視頻剪輯工具是一款簡單易用的在線視頻編輯應用，幫助用戶快速剪輯視頻片段。無需下載安裝專業的視頻編輯軟件，您可以直接在瀏覽器中完成視頻剪輯操作。
      </p>
    </section>

    <section class="about-section">
      <h2>主要功能</h2>
      <ul class="feature-list">
        <li>上傳本地視頻文件</li>
        <li>實時預覽視頻內容</li>
        <li>設置精確的剪輯起止點</li>
        <li>導出剪輯後的視頻並下載保存</li>
      </ul>
    </section>

    <section class="about-section">
      <h2>技術實現</h2>
      <p>
        本項目使用 Vue 3 + TypeScript
        構建，採用現代化的前端技術框架，確保操作流暢且界面友好。視頻處理過程完全在瀏覽器中進行，保證了您的視頻隱私安全。
      </p>
    </section>

    <div class="back-link">
      <router-link to="/">返回首頁</router-link>
    </div>
  </div>
</template>

<style scoped>
.about-container {
  max-width: 800px;
  margin: 0 auto;
  padding: 40px 20px;
}

h1 {
  color: #42b883;
  margin-bottom: 32px;
  text-align: center;
}

.about-section {
  margin-bottom: 40px;
}

h2 {
  color: #333;
  margin-bottom: 16px;
  font-size: 1.5rem;
}

p {
  line-height: 1.6;
  color: #555;
}

.feature-list {
  padding-left: 20px;
}

.feature-list li {
  margin-bottom: 10px;
  line-height: 1.5;
  color: #555;
}

.back-link {
  margin-top: 40px;
  text-align: center;
}

.back-link a {
  color: #42b883;
  text-decoration: none;
  font-weight: 500;
  padding: 8px 16px;
  border: 1px solid #42b883;
  border-radius: 4px;
  transition: all 0.3s ease;
}

.back-link a:hover {
  background-color: #42b883;
  color: white;
}
</style>
