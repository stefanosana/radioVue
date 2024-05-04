
<script setup>
  import { VideoPlayer } from 'vue-hls-video-player';

  function processPause(progress) {
    console.log(progress)
  }
</script>

<template>
  <div>
    <v-container>
      <h1>Radio ITALIANE</h1> <br> <br> <br>
      <v-row>
        <v-col v-for="radio in radios" :key="radio.id" cols="12" sm="6" md="4">
          <v-card class="mb-3" @click="playRadio(radio)">
            <v-img v-if="radio.favicon" :src="radio.favicon" height="200" contain></v-img>
            <v-img v-else src="radio/src/views/img/radio.jpg" height="200" contain></v-img>
            <v-card-title>{{ radio.name }}</v-card-title>
          </v-card>
        </v-col>
      </v-row>
      <audio ref="audioPlayer" :src="currentRadioUrl" controls></audio>
    </v-container>

    <!-- Footer -->
    <footer class="footer">
      <div class="container">
        <p>Radio in riproduzione: {{ currentRadioName }}</p>
        <p>Titolo della canzone: {{ currentSongTitle }}</p>
      </div>
    </footer>
  </div>

  <VideoPlayer
      type="default"
      @pause="processPause"
      previewImageLink="poster.webp"
      link="videoLink.m3u8"
      :progress="30"
      :isMuted="false"
      :isControls="true"
      class="customClassName"
  />
  
</template>


<script>
export default {
  name: 'HomeView',
  data() {
    return {
      radios: [],
      currentRadioUrl: null,
      currentRadioName: '',
      currentSongTitle: '',
      audio: null,
      isPlaying: false
    }
  },
  methods: {
    playRadio(radio) {
      if (this.audio) {
        this.audio.pause(); // Interrompe la riproduzione dell'audio corrente
      }
      this.currentRadioName = radio.name;
      this.currentRadioUrl = radio.url;
      this.audio = this.$refs.audioPlayer;
      this.audio.oncanplaythrough = () => {
        this.audio.play(); // Avvia la riproduzione dell'audio della nuova stazione radio
        this.isPlaying = true;
      }
    },
    getRadios() {
      fetch('https://nl1.api.radio-browser.info/json/stations/search?limit=100&countrycode=IT&hidebroken=true&order=clickcount&reverse=true')
        .then(response => response.json())
        .then(data => {
          this.radios = data;
          console.log(data);
        });
    },
  },
  created() {
    this.getRadios();
  },
}
</script>

<style scoped>
.footer {
  position: fixed;
  bottom: 0;
  left: 0;
  width: 100%;
  background-color: #f5f5f5;
  text-align: center;
  padding: 20px 0;
}
</style>
