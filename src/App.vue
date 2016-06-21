<template>
  <div id="app">
    <header>
      <div class="container">
        <h1>Spotify track sorter</h1>
      </div>
    </header>
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
  background: #232323;
  color:#fff;
}

#app {
  font-family: Source Sans Pro, Helvetica, sans-serif;
  text-align: center;
}

.row {
  margin-top: 50px;
}
article {
  background: #fafafa;
  border:1px solid #d6d6d6;
  border-radius: 10px;
  margin-bottom: 10px;
}
.col-md-3 {
  height: 160px;
}
h2 {
    color: #4B917D;
}

.deleted-section {
  padding: 20px 0;
  background: #fff;
  color:#000;
}
</style>
