<script setup lang="ts">
// 移除不存在的組件引用
</script>

<template>
  <div class="app">
    <nav class="navbar" v-if="$route.path !== '/'">
      <router-link to="/" class="nav-link home-link">
        <span class="nav-logo">剪輯工具</span>
      </router-link>

      <div class="nav-links">
        <router-link to="/about" class="nav-link">關於</router-link>
      </div>
    </nav>

    <div class="navbar-spacer" v-if="$route.path !== '/'"></div>

    <RouterView />
  </div>
</template>

<style>
/* 全局樣式 */
:root {
  --primary-color: #42b883;
  --primary-dark: #3a9d71;
  --text-color: #333;
  --light-gray: #f5f5f5;
  --medium-gray: #e0e0e0;
  --dark-gray: #666;
}

* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

body {
  font-family: "Inter", "Helvetica Neue", Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  color: var(--text-color);
  background-color: var(--light-gray);
  line-height: 1.6;
}

/* 導航欄樣式 */
.navbar {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  background-color: white;
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0 24px;
  height: 60px;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
  z-index: 1000;
}

.navbar-spacer {
  height: 60px;
}

.nav-logo {
  color: var(--primary-color);
  font-weight: 700;
  font-size: 1.4rem;
}

.nav-links {
  display: flex;
  gap: 16px;
}

.nav-link {
  color: var(--text-color);
  text-decoration: none;
  font-weight: 500;
  padding: 8px 12px;
  border-radius: 4px;
  transition: background-color 0.2s;
}

.nav-link:hover {
  background-color: rgba(66, 184, 131, 0.1);
}

.home-link {
  padding: 0;
}

.home-link:hover {
  background-color: transparent;
}

/* 響應式調整 */
@media (max-width: 768px) {
  .navbar {
    padding: 0 16px;
  }
}
</style>
