<template>
  <div id="app">
    <h1>Three.js Room</h1>
    <!-- Use the ThreeScene component here -->
    <ThreeScene />
  </div>
</template>

<script>
// Import the ThreeScene component
import ThreeScene from './components/ThreeScene.vue';
export default {
  name: 'App',
  components: {
    ThreeScene,
  }
}
</script>

<style>
#app {
  text-align: center;
}
</style>
