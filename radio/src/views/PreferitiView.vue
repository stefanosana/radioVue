<template>
    <div>
        <!-- Sezione delle radio preferite -->
        <v-container>
            <h1>RADIO PREFERITE</h1> <br> <br> <br>
            <v-row>
                <v-col v-for="favRadio in favoriteRadios" :key="favRadio.id" cols="12" sm="6" md="4">
                    <v-card class="mb-3 custom-card">
                        <v-img v-if="radio.favicon != null" :src="radio.favicon" height="200"  style="border-radius: 10px 10px 0 0;"></v-img>
                        <v-img v-else :src="defaultImage" height="200" style="border-radius: 10px 10px 0 0;"></v-img>

                        <!-- Bottone rimuovi dai preferiti -->
                        <v-btn icon @click="toggleFavorite(favRadio)" color="red" style="margin-right: 5px;">
                            <v-icon>mdi-heart</v-icon>
                        </v-btn>

                        <v-card-title style="font-family: Arial, Helvetica, sans-serif">{{ favRadio.name
                            }}</v-card-title>
                    </v-card>
                </v-col>
            </v-row>
        </v-container>

    </div>
</template>

<script>
import Hls from 'hls.js';
import defaultImage from '/public/radio.jpg';

export default {
    name: 'HomeView',
    data() {
        return {
            radios: [],
            favoriteRadios: [],
            // Altri dati rimangono invariati
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
        },
        toggleFavorite(radio) {
            const index = this.favoriteRadios.findIndex(favRadio => favRadio.id === radio.id);
            if (index === -1) {
                this.favoriteRadios.push(radio);
            } else {
                this.favoriteRadios.splice(index, 1);
            }
            localStorage.setItem('favorites', JSON.stringify(this.favoriteRadios));
        },
        // Metodo per ottenere le radio preferite
        getFavoriteRadios() {
            const favoriteRadios = JSON.parse(localStorage.getItem('favorites')) || [];
            this.favoriteRadios = favoriteRadios;
        },
    },
    created() {
        this.getRadios();
        this.getFavoriteRadios(); // Chiamiamo il metodo per ottenere le radio preferite
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