<script setup>
import FileUploader from './components/FileUploader.vue'
</script>

<template>
  <div class="app-container">
    <header class="app-header">
      <h1>账单分析工具</h1>
      <p>支持上传Excel和CSV文件进行数据解析</p>
    </header>

    <main>
      <FileUploader />
    </main>
  </div>
</template>

<style scoped>
.app-container {
  min-height: 100vh;
  width: 100vw;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  box-sizing: border-box;
}

.app-header {
  text-align: center;
  padding: 40px 20px;
  color: white;
}

.app-header h1 {
  font-size: 2.5rem;
  margin-bottom: 10px;
  font-weight: 600;
}

.app-header p {
  font-size: 1.2rem;
  opacity: 0.9;
  margin: 0;
}

main {
  padding: 0 20px 40px;
  width: 100%;
  box-sizing: border-box;
}
</style>
