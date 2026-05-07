<template>
  <div id="app">
    <WhackAMole />
  </div>
</template>

<script>
import WhackAMole from './components/WhackAMole.vue'

export default {
  name: 'App',
  components: { WhackAMole }
}
</script>

<style>
* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

body {
  background: linear-gradient(160deg, #0d0d2b 0%, #1a0a2e 40%, #0f1a2e 100%);
}
</style>