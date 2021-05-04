<template>
  <div id="app">
    <img alt="Vue logo" src="./assets/logo.png">
    <HelloThere :msg="msg"/>
  </div>
</template>

<script>
import axios from 'axios'
import HelloThere from './components/HelloThere.vue'

export default {
  name: 'App',
  components: {
    HelloThere
  },
  data () {
    return {
      msg: null
    }
  },
  mounted() {
    axios
      .get('http://backend.test/api/hello-there')
      .then(response => (this.msg = response.data))
  }
}
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
