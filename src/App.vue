<script setup>
import ColorGenerator from './components/ColorGenerator.vue'
import { ref } from 'vue';

const image = ref(null)
</script>

<template>
  <main style="text-align: center">
    <h1>Image Color Generator</h1>
    <ColorGenerator @input-file="(e) => image = e" />
  </main>
</template>

