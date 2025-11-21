<script setup>
import UMAPViewer from './components/UMAPViewer.vue'
</script>

<template>
  <div class="app">
    <header>
      <h1>UMAP Viewer</h1>
      <p>Interactive visualization of UMAP dimensionality reduction</p>
    </header>
    <main>
      <UMAPViewer :width="1000" :height="700" />
    </main>
  </div>
</template>

<style scoped>
.app {
  min-height: 100vh;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  padding: 20px;
}

header {
  text-align: center;
  color: white;
  margin-bottom: 30px;
}

header h1 {
  margin: 0 0 10px 0;
  font-size: 2.5em;
  font-weight: 700;
  text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.2);
}

header p {
  margin: 0;
  font-size: 1.1em;
  opacity: 0.9;
}

main {
  display: flex;
  justify-content: center;
  align-items: flex-start;
}
</style>

<style>
* {
  box-sizing: border-box;
}

body {
  margin: 0;
  padding: 0;
}
</style>
