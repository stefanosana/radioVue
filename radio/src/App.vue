<template>
  <div id="app">
    <nav>
      <router-link to="/">HOME</router-link> |
      <router-link to="/preferiti">PREFERITI</router-link> |
      <router-link to="/mappamondo">GEOMAP</router-link> 
    </nav>
    <router-view :current-radio-name="currentRadioName"/>
    <v-layout>
    </v-layout>
  </div>
</template>

<script>
export default {
  data() {
    return {
      icons: [
        'mdi-facebook',
        'mdi-twitter',
        'mdi-linkedin',
        'mdi-instagram',
      ],
    }
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
}

nav {
  padding: 30px;
}

nav a {
  font-weight: bold;
  color: #2c3e50;
}

nav a.router-link-exact-active {
  color: red;
}
</style>
