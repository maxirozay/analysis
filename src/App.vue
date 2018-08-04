<template>
  <div id="app">
    <TheHome/>
  </div>
</template>

<script>
import TheHome from './components/TheHome.vue'

export default {
  name: 'app',
  components: {
    TheHome
  }
}
</script>

<style>
body {
  margin: 0
}
#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}
</style>
