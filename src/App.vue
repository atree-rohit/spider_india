<template>
  <v-app
    id="inspire"
    dark
  >
    <v-app-bar
      app
      color="primary"
      dark
    >
      <div class="d-flex align-center">
        <span class="text-h5">Spider India</span>
      </div>

      <v-spacer></v-spacer>

    </v-app-bar>

    <v-main>
      <all-species />
    </v-main>

  </v-app>  
</template>

<script>
import AllSpecies from './components/AllSpecies.vue'

export default {
  name: 'App',
  components: {
    AllSpecies
  }
}
</script>

<style>

</style>
