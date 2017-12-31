<template>
  <div id="app">
    <img src="./assets/logo.png">
    <bdo-timers/>
  </div>
</template>

<script>
import BdoTimers from "./components/BdoTimers";

export default {
  name: "app",
  components: {
    BdoTimers
  }
};
</script>

<style>
#app {
  font-family: "Avenir", Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
