<template>
  <div>
    <v-container>
      <h1>RADIO ITALIANE</h1> <br> <br> <br>
      <v-row>
        <v-col v-for="radio in radios" :key="radio.id" cols="12" sm="6" md="4">
          <v-card class="mb-3 custom-card" @click="togglePlay(radio)">
            <v-img v-if="radio.favicon" :src="radio.favicon" height="200" contain></v-img>
            <v-img v-else src="radio.src/views/img/radio.jpg" height="200" contain></v-img>
            <v-card-title style="font-family: Arial, Helvetica, sans-serif">{{ radio.name }}</v-card-title>
          </v-card>
        </v-col>
      </v-row>
    </v-container>

    <!-- Video Player HLS -->
    <video ref="videoPlayer" id="video-player" controls style="display: none;"></video>

    <!-- Footer -->
    <footer class="footer">
      <div class="container">
        <p style="font-family: Arial, Helvetica, sans-serif">Radio in riproduzione: {{ currentRadioName }}</p>
        <p style="font-family: Arial, Helvetica, sans-serif">Titolo della canzone: {{ currentSongTitle }}</p>
      </div>
    </footer>
  </div>
</template>

<script>
import Hls from 'hls.js';

export default {
  name: 'HomeView',
  data() {
    return {
      radios: [],
      currentRadioName: '',
      currentSongTitle: '',
      currentRadioUrl: null,
      currentHlsInstance: null,
      isPlaying: false
    }
  },
  methods: {
    getRadios() {
      fetch('https://nl1.api.radio-browser.info/json/stations/search?limit=100&countrycode=IT&hidebroken=true&order=clickcount&reverse=true')
        .then(response => response.json())
        .then(data => {
          this.radios = data.map(station => ({
            ...station,
            favicon: station.favicon || 'radio/img/radio.jpg' // Aggiungi un'immagine predefinita per le stazioni senza favicon
          }));
        })
        .catch(error => {
          console.error('Error fetching radios:', error);
        });
    },
    togglePlay(radio) {
      if (this.currentRadioUrl === radio.url) {
        this.stopRadio();
      } else {
        this.playRadio(radio);
      }
    },
    playRadio(radio) {
      if (this.currentHlsInstance) {
        this.currentHlsInstance.destroy();
        this.currentHlsInstance = null;
      }

      const video = this.$refs.videoPlayer;
      if (radio.url.endsWith('.m3u8')) {
        if (Hls.isSupported()) {
          const hls = new Hls();
          hls.loadSource(radio.url);
          hls.attachMedia(video);
          hls.on(Hls.Events.MANIFEST_PARSED, () => {
            video.play();
          });
          this.currentHlsInstance = hls;
        } else if (video.canPlayType('application/vnd.apple.mpegurl')) {
          video.src = radio.url;
          video.play();
        }
      } else {
        video.src = radio.url;
        video.play();
      }

      this.currentRadioUrl = radio.url;
      this.currentRadioName = radio.name;
      this.isPlaying = true;
    },
    stopRadio() {
      const video = this.$refs.videoPlayer;
      video.pause();
      video.src = '';
      if (this.currentHlsInstance) {
        this.currentHlsInstance.destroy();
        this.currentHlsInstance = null;
      }
      this.currentRadioUrl = null;
      this.currentRadioName = '';
      this.isPlaying = false;
    }
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
  background-color: red;
  /* Cambio colore di sfondo */
  color: #fff;
  /* Cambio colore del testo */
  text-align: center;
  padding: 20px 0;
}

.footer p {
  font-family: Helvetica, Arial, sans-serif;
}

.footer {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

/* Aggiungiamo uno stile personalizzato alle card */
.custom-card {
  background-color: #fff;
  /* Sfondo delle card */
  border-radius: 10px;
  /* Bordi arrotondati */
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
  /* Ombra */
  cursor: pointer;
  /* Cambio del cursore al passaggio sopra la card */
}

.custom-card:hover {
  transform: translateY(-2px);
  /* Effetto di sollevamento al passaggio del mouse */
}
</style>
