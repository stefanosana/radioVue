<template>
  <div ref="container"></div>
  <div class="navbar" v-if="selectedRadio">
    <img :src="selectedRadio.favicon || defaultImage" class="radio-logo" alt="Radio logo">
    <h4>{{ selectedRadio.name }}</h4>
    <h2>{{ selectedRadio.country }}</h2>
    <button @click="togglePlayPause(selectedRadio)">{{ selectedRadio.playing ? 'Pause' : 'Play' }}</button>
    <div v-if="selectedRadio.playing" class="sound-wave">
      <div class="bar"></div>
      <div class="bar"></div>
      <div class="bar"></div>
      <div class="bar"></div>
    </div>
  </div>
</template>

<script>
// Importa le librerie Three.js e Hls.js
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';
import Hls from 'hls.js';

export default {
  name: 'ThreeJsScene', // Nome del componente Vue.js
  data() {
    return {
      // Variabili di stato per la scena Three.js
      camera: null,
      renderer: null,
      controls: null,
      earthRadius: 1,
      radios: [],
      selectedRadio: null,
    };
  },
  mounted() {
    // Funzioni eseguite quando il componente viene montato
    this.init(); // Inizializza la scena Three.js
    this.animate(); // Avvia l'animazione della scena
    this.getRadios(); // Ottiene le stazioni radio disponibili
    window.addEventListener('resize', this.handleWindowResize); // Aggiunge un listener per il ridimensionamento della finestra
    this.raycaster = new THREE.Raycaster(); // Inizializza il raycaster per il rilevamento degli oggetti intersecati dal mouse
    this.mouse = new THREE.Vector2(); // Inizializza il vettore del mouse per le coordinate
    window.addEventListener('click', this.onDocumentMouseClick); // Aggiunge un listener per il click del mouse
  },
  beforeUnmount() {
    // Funzioni eseguite prima che il componente venga smontato
    window.removeEventListener('resize', this.handleWindowResize); // Rimuove il listener per il ridimensionamento della finestra
    window.removeEventListener('click', this.onDocumentMouseClick); // Rimuove il listener per il click del mouse
  },
  methods: {
    init() {
      // Inizializza la scena Three.js
      this.camera = new THREE.PerspectiveCamera(30, window.innerWidth / window.innerHeight, 0.1, 1000); // Crea la fotocamera prospettica
      this.camera.position.z = 5; // Imposta la posizione della fotocamera

      this.renderer = new THREE.WebGLRenderer(); // Crea il renderer WebGL
      this.renderer.setSize(window.innerWidth, window.innerHeight); // Imposta le dimensioni del renderer
      this.$refs.container.appendChild(this.renderer.domElement); // Aggiunge il renderer al container HTML

      // Inizializza i controlli della fotocamera (OrbitControls)
      this.controls = new OrbitControls(this.camera, this.renderer.domElement); // Crea i controlli della fotocamera OrbitControls
      this.controls.enableDamping = true; // Abilita il damping per i controlli della fotocamera
      this.controls.minDistance = 3; // Imposta la distanza minima della fotocamera
      this.controls.maxDistance = 9; // Imposta la distanza massima della fotocamera

      // Inizializza la scena e aggiungi la sfera (es. terra)
      const scene = new THREE.Scene(); // Crea una nuova scena Three.js
      const geometry = new THREE.SphereGeometry(this.earthRadius, 64, 64); // Crea la geometria per la sfera
      const texture = new THREE.TextureLoader().load("texture.jpg"); // Carica la texture per la sfera
      const material = new THREE.MeshPhongMaterial({ map: texture }); // Crea il materiale per la sfera
      const earth = new THREE.Mesh(geometry, material); // Crea la mesh per la sfera
      scene.add(earth); // Aggiunge la sfera alla scena

      // Aggiungi luci alla scena
      const ambientLight = new THREE.AmbientLight(0xffffff, 0.5); // Crea luce ambientale
      scene.add(ambientLight); // Aggiunge la luce ambientale alla scena

      const directionalLight = new THREE.DirectionalLight(0xffffff, 1); // Crea luce direzionale
      directionalLight.position.set(1, 1, 1).normalize(); // Imposta la posizione della luce direzionale
      scene.add(directionalLight); // Aggiunge la luce direzionale alla scena

      this.scene = scene; // Imposta la scena

      // Inizializza raycaster e mouse
      this.raycaster = new THREE.Raycaster(); // Inizializza il raycaster per il rilevamento degli oggetti intersecati dal mouse
      this.mouse = new THREE.Vector2(); // Inizializza il vettore del mouse per le coordinate

      // Aggiungi un listener per gli eventi del mouse sulla finestra
      window.addEventListener('click', this.onDocumentMouseClick, false); // Aggiunge un listener per il click del mouse
    },
    onDocumentMouseClick(event) {
      // Gestisce il click del mouse sulla finestra
      event.preventDefault(); // Impedisce l'azione predefinita del browser
      this.mouse.x = (event.offsetX / this.renderer.domElement.clientWidth) * 2 - 1; // Calcola la coordinata x del mouse
      this.mouse.y = -(event.offsetY / this.renderer.domElement.clientHeight) * 2 + 1; // Calcola la coordinata y del mouse
      this.raycaster.setFromCamera(this.mouse, this.camera); // Imposta il raycaster in base alla posizione del mouse e della fotocamera
      const intersects = this.raycaster.intersectObjects(this.scene.children); // Trova gli oggetti intersecati dal raycaster nella scena

      if (intersects.length > 0) {
        // Se ci sono oggetti intersecati
        console.log('Clicked on:', intersects[0].object.userData); // Stampa le informazioni sull'oggetto intersecato
        this.handleMarkerClick(intersects[0]); // Gestisce il click sull'oggetto intersecato
      }
    },
    handleMarkerClick(intersectedObject) {
      // Gestisce il click sull'oggetto intersecato
      if (intersectedObject.object.onClick) {
        // Se l'oggetto ha una funzione onClick definita
        intersectedObject.object.onClick(); // Esegue la funzione onClick dell'oggetto
        const previousRadio = this.selectedRadio; // Salva la stazione radio precedente
        this.selectedRadio = intersectedObject.object.userData; // Imposta la stazione radio selezionata
        this.$nextTick(() => {
          // Aspetta il prossimo tick del Vue
          if (this.selectedRadio.url_resolved || this.selectedRadio.url) {
            // Se l'URL della stazione radio è risolto o definito
            this.lazyLoadAudio(this.selectedRadio); // Carica in modo pigro l'audio per la stazione radio selezionata
            if (previousRadio && previousRadio !== this.selectedRadio) {
              // Se c'è una stazione radio precedente e non è uguale alla stazione radio selezionata
              this.pauseRadio(previousRadio); // Metti in pausa la stazione radio precedente
            }
          }
        });
      }
    },
    lazyLoadAudio(radio) {
      // Carica in modo pigro l'audio per la stazione radio
      console.log('Lazy loading audio for:', radio); // Stampa un messaggio di debug
      if (!radio.audioPlayer) {
        // Se il lettore audio per la stazione radio non esiste
        console.log('Creating new audio player'); // Stampa un messaggio di debug
        radio.audioPlayer = new Audio(radio.url_resolved || radio.url); // Crea un nuovo lettore audio con l'URL della stazione radio
        console.log('Audio player:', radio.audioPlayer); // Stampa il lettore audio
        radio.audioPlayer.onloadeddata = () => {
          // Aggiunge un gestore per l'evento di caricamento dei dati dell'audio
          console.log('Audio loaded'); // Stampa un messaggio di debug
        };
        radio.audioPlayer.onerror = () => {
          // Aggiunge un gestore per l'evento di errore del caricamento dell'audio
          console.error('Error loading audio'); // Stampa un messaggio di errore
        };
      }
      else {
        // Se il lettore audio per la stazione radio esiste già
        console.log('Audio player already exists'); // Stampa un messaggio di debug
      }
    },
    addMarker(longitude, latitude, radio) {
      // Aggiunge un marker per la stazione radio sulla sfera (terra)
      if (longitude !== null && latitude !== null) {
        // Se le coordinate di longitudine e latitudine sono definite
        const phi = (90 - latitude) * (Math.PI / 180); // Calcola la latitudine in radianti
        const theta = (longitude + 180) * (Math.PI / 180); // Calcola la longitudine in radianti
        const x = -this.earthRadius * Math.sin(phi) * Math.cos(theta); // Calcola la coordinata x sulla sfera
        const y = this.earthRadius * Math.cos(phi); // Calcola la coordinata y sulla sfera
        const z = this.earthRadius * Math.sin(phi) * Math.sin(theta); // Calcola la coordinata z sulla sfera

        const geometry = new THREE.SphereGeometry(0.01, 32, 32); // Crea la geometria per il marker
        const material = new THREE.MeshBasicMaterial({ color: 0xff0000 }); // Crea il materiale per il marker
        const marker = new THREE.Mesh(geometry, material); // Crea il mesh per il marker
        marker.position.set(x, y, z); // Imposta la posizione del marker
        marker.userData = radio; // Aggiunge i dati della stazione radio al marker
        marker.onClick = () => {
          console.log('Clicked on marker:', marker.userData); // Stampa un messaggio di debug quando viene cliccato il marker
        };
        this.scene.add(marker); // Aggiunge il marker alla scena

        const invisibleSphereGeometry = new THREE.SphereGeometry(0.015, 32, 32); // Crea la geometria per la sfera invisibile
        const invisibleSphereMaterial = new THREE.MeshBasicMaterial({ visible: false }); // Crea il materiale per la sfera invisibile
        const invisibleSphere = new THREE.Mesh(invisibleSphereGeometry, invisibleSphereMaterial); // Crea il mesh per la sfera invisibile
        invisibleSphere.position.set(x, y, z); // Imposta la posizione della sfera invisibile
        invisibleSphere.userData = radio; // Aggiunge i dati della stazione radio alla sfera invisibile
        invisibleSphere.onClick = marker.onClick; // Assegna la stessa funzione onClick del marker alla sfera invisibile
        this.scene.add(invisibleSphere); // Aggiunge la sfera invisibile alla scena
      } else {
        // Se le coordinate di longitudine o latitudine sono null
        console.error('Longitude or latitude is null'); // Stampa un messaggio di errore
      }
    },
    animate() {
      // Funzione per l'animazione della scena
      requestAnimationFrame(this.animate); // Richiede l'animazione per il prossimo frame
      if (this.controls) {
        // Se i controlli della fotocamera sono definiti
        this.controls.update(); // Aggiorna i controlli della fotocamera
      }
      if (this.renderer && this.scene && this.camera) {
        // Se il renderer, la scena e la fotocamera sono definiti
        this.renderer.render(this.scene, this.camera); // Rende la scena con la fotocamera
      }
    },
    handleWindowResize() {
      // Funzione per gestire il ridimensionamento della finestra
      this.camera.aspect = window.innerWidth / window.innerHeight; // Aggiorna l'aspect ratio della fotocamera
      this.camera.updateProjectionMatrix(); // Aggiorna la matrice di proiezione della fotocamera
      this.renderer.setSize(window.innerWidth, window.innerHeight); // Imposta le nuove dimensioni del renderer
    },
    async fetchRadios() {
      // Funzione per ottenere le stazioni radio disponibili
      const response = await fetch('https://nl1.api.radio-browser.info/json/stations/search?limit=100&has_geo_info=true&hidebroken=true&order=clickcount&reverse=true'); // Esegue una richiesta GET per ottenere i dati delle stazioni radio
      const data = await response.json(); // Ottiene i dati delle stazioni radio in formato JSON
      return data; // Restituisce i dati delle stazioni radio
    },
    async getRadios() {
      // Funzione per ottenere e aggiungere le stazioni radio alla scena
      try {
        this.radios = await this.fetchRadios(); // Ottiene i dati delle stazioni radio
        this.radios.forEach(radio => {
          // Per ogni stazione radio
          this.addMarker(radio.geo_long, radio.geo_lat, radio); // Aggiunge un marker sulla sfera per la stazione radio
          radio.playing = false; // Imposta lo stato di riproduzione della stazione radio su falso
          radio.audioPlayer = new Audio(); // Crea un nuovo lettore audio per la stazione radio
        });
      } catch (error) {
        console.error('Error fetching radios:', error); // Stampa un messaggio di errore se si verifica un errore durante il recupero delle stazioni radio
      }
    },
    togglePlayPause(radio) {
      // Funzione per mettere in pausa o riprodurre una stazione radio
      if (radio.playing) {
        // Se la stazione radio è in riproduzione
        this.pauseRadio(radio); // Metti in pausa la stazione radio
      } else {
        // Se la stazione radio non è in riproduzione
        this.pauseAllRadios(); // Metti in pausa tutte le altre stazioni radio
        this.playRadio(radio); // Riproduci la stazione radio
      }
    },
    playRadio(radio) {
      // Funzione per riprodurre una stazione radio
      const audioUrl = radio.url_resolved || radio.url; // Ottiene l'URL dell'audio della stazione radio
      if (audioUrl.includes('m3u8')) {
        // Se l'URL dell'audio è un file .m3u8 (HLS)
        if (Hls.isSupported()) {
          // Se HLS è supportato nel browser
          const hls = new Hls(); // Crea un nuovo oggetto Hls
          hls.loadSource(audioUrl); // Carica la sorgente HLS
          hls.attachMedia(radio.audioPlayer); // Collega l'oggetto Hls al lettore audio
        } else {
          console.error('HLS is not supported in this browser.'); // Stampa un messaggio di errore se HLS non è supportato nel browser
        }
      } else {
        // Se l'URL dell'audio non è un file .m3u8
        radio.audioPlayer.src = audioUrl; // Imposta l'URL dell'audio nel lettore audio
      }
      radio.audioPlayer.play() // Avvia la riproduzione dell'audio
        .catch(error => {
          console.error('Error playing audio:', error); // Stampa un messaggio di errore se si verifica un errore durante la riproduzione dell'audio
          if (error.name === 'NotAllowedError') {
            console.error('Please ensure that the audio playback is allowed in your browser settings.'); // Stampa un messaggio se la riproduzione audio non è consentita nelle impostazioni del browser
          }
        });
      radio.playing = true; // Imposta lo stato di riproduzione della stazione radio su vero
    },
    pauseRadio(radio) {
      // Funzione per mettere in pausa una stazione radio
      radio.audioPlayer.pause(); // Metti in pausa la riproduzione dell'audio
      radio.playing = false; // Imposta lo stato di riproduzione della stazione radio su falso
    },
    pauseAllRadios() {
      // Funzione per mettere in pausa tutte le stazioni radio
      this.radios.forEach(radio => {
        // Per ogni stazione radio
        if (radio.playing) {
          // Se la stazione radio è in riproduzione
          this.pauseRadio(radio); // Metti in pausa la stazione radio
        }
      });
    },
    created() {
      // Funzione eseguita al momento della creazione del componente
      this.getRadios(); // Ottiene le stazioni radio disponibili
    }
  }
};
</script>


<style scoped>
.navbar {
  color: white;
  position: fixed;
  bottom: 0;
  left: 0;
  width: 100%;
  background-color: red;
  box-shadow: 0 -2px 5px rgba(0, 0, 0, 0.3);
  display: flex;
  align-items: center;
  justify-content: space-around;
  padding: 10px;
}

.radio-logo {
  width: 50px;
  height: 50px;
  border-radius: 25px;
}

.sound-wave {
  display: flex;
  align-items: center;
  height: 20px;
  margin-left: 10px;
  margin-top: 2px;
}

.bar {
  width: 4px;
  height: 100%;
  margin: 0 2px;
  background-color: #333;
  animation: pulse 0.8s infinite ease-in-out alternate;
}

.bar:nth-child(1) {
  animation-delay: 0s;
}

.bar:nth-child(2) {
  animation-delay: 0.1s;
}

.bar:nth-child(3) {
  animation-delay: 0.2s;
}

.bar:nth-child(4) {
  animation-delay: 0.3s;
}

@keyframes pulse {
  0% {
    transform: scaleY(1);
  }

  100% {
    transform: scaleY(1.5);
  }
}
</style>