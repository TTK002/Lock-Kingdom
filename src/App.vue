<template>
  <div id="app">
    <TeamAnalyzer />
  </div>
</template>

<script setup lang="ts">
import TeamAnalyzer from './components/TeamAnalyzer.vue'
</script>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  background: linear-gradient(135deg, #c3ecff 0%, #ffe6c7 100%);
  min-height: 100vh;
  padding: 20px;
}
</style>