<script setup>
import { RouterLink, RouterView } from 'vue-router'
</script>

<template>
  <header>
    File
  </header>

  <RouterView />
</template>

<style scoped>

header {
  width: 100%;
  height: 32px;
  /* padding: 0 10px; */
  display:flex;
  flex-direction: column;
  justify-content: center;
  background-color: black;
  color:white;
}

</style>
