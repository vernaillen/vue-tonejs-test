<template>
  <v-app>
    <v-app-bar
            app
            color="primary"
            dark
    >
      <div class="d-flex align-center">
        <h2>Vue + Tone.js playground</h2>
      </div>

      <v-spacer></v-spacer>

      <v-btn
              href="https://github.com/vernaillen/vue-tonejs-test/"
              target="_blank"
              text
      >
        <span class="mr-2">Source code on GitHub</span>
        <v-icon>mdi-open-in-new</v-icon>
      </v-btn>
    </v-app-bar>

    <v-content >
      <piano></piano>
    </v-content>
  </v-app>
</template>

<script>
import Piano from './components/Piano'

export default {
  name: 'App',
  components: {
    Piano
  }
}
</script>
