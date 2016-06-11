<template>
  <div id="app">
    <img class="logo" src="./assets/logo.png">
    <hello></hello>
  </div>
</template>

<script>
import Hello from './components/Hello'

export default {
  components: {
    Hello
  }
}
</script>

<style>
html {
  height: 100%;
}

body {
  display: flex;
  align-items: center;
  justify-content: center;
}

#app {
  color: #2c3e50;
  margin-top: -100px;
  font-family: Source Sans Pro, Helvetica, sans-serif;
  text-align: center;
}

#app a {
  color: #42b983;
  text-decoration: none;
}
.row {
  margin-top: 50px;
}
article {
  background: #fafafa;
  border:1px solid #d6d6d6;
  border-radius: 4%;
  margin-bottom: 10px;
}
.col-md-3 {
  height: 160px;
}
h2 {
    color: #42b983;
}
</style>
