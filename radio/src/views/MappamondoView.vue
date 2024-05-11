<template>
  <div id="container"></div>

  <video ref="videoPlayer" id="video-player" controls style="display: none;"></video>

  <!-- Footer -->
  <footer class="footer">
    <div class="container">
      <p style="font-family: Arial, Helvetica, sans-serif">Radio in riproduzione: {{ currentRadioName }}</p>
      <p style="font-family: Arial, Helvetica, sans-serif">Titolo della canzone: {{ currentSongTitle }}</p>
    </div>
  </footer>
</template>

<script>
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';

export default {
  mounted() {
    this.initWorld();
    this.animate();
    this.getRadios();
    window.addEventListener('resize', this.handleWindowResize);
    this.raycaster = new THREE.Raycaster();
    this.mouse = new THREE.Vector2();
    window.addEventListener('click', this.onDocumentMouseClick);

  },
  beforeUnmount() {
    window.removeEventListener('resize', this.handleWindowResize);
    window.removeEventListener('click', this.onDocumentMouseClick);

  },
  methods: {
    initWorld() {
      this.camera = new THREE.PerspectiveCamera(30, window.innerWidth / window.innerHeight, 0.1, 1000);
      this.camera.position.z = 5;

      this.renderer = new THREE.WebGLRenderer();
      this.renderer.setSize(window.innerWidth, window.innerHeight);
      document.getElementById('container').appendChild(this.renderer.domElement); // Utilizza document.getElementById

      // Inizializza i controlli della fotocamera (OrbitControls)
      this.controls = new OrbitControls(this.camera, this.renderer.domElement);
      this.controls.enableDamping = true;
      this.controls.minDistance = 3; // Imposta la distanza minima
      this.controls.maxDistance = 9;

      // Inizializza la scena e aggiungi la sfera (es. terra)
      const scene = new THREE.Scene();
      const geometry = new THREE.SphereGeometry(this.earthRadius, 64, 64);
      const texture = new THREE.TextureLoader().load('/texture.jpg');
      const material = new THREE.MeshPhongMaterial({ map: texture });
      const earth = new THREE.Mesh(geometry, material);
      scene.add(earth);

      // Aggiungi luci alla scena
      const ambientLight = new THREE.AmbientLight(0xffffff, 0.5);
      scene.add(ambientLight);

      const directionalLight = new THREE.DirectionalLight(0xffffff, 1);
      directionalLight.position.set(1, 1, 1).normalize();
      scene.add(directionalLight);

      this.scene = scene;

      // Inizializza raycaster e mouse
      this.raycaster = new THREE.Raycaster();
      this.mouse = new THREE.Vector2();

      // Aggiungi un listener per gli eventi del mouse sulla finestra
      window.addEventListener('click', this.onDocumentMouseClick, false);
    },
    async fetchRadios() {
      const response = await fetch('https://nl1.api.radio-browser.info/json/stations/search?limit=100&has_geo_info=true&hidebroken=true&order=clickcount&reverse=true');
      const data = await response.json();
      return data;
    },
    async getRadios() {
      try {
        this.radios = await this.fetchRadios();
        this.radios.forEach(radio => {
          this.addMarker(radio.geo_lat, radio.geo_long, radio.name);
          radio.playing = false;
          radio.audioPlayer = new Audio();
        });
        this.retrieveFavorites();
      } catch (error) {
        console.error('Error fetching radios:', error);
      }

      console.log('Radios:', this.radios);
      this.radios.forEach(radio => {
        console.log('Radio:', radio.name, 'Latitude:', radio.geo_lat, 'Longitude:', radio.geo_long);
      });
    },
    addMarker(longitude, latitude, radio) {
      if (longitude !== null && latitude !== null) {
        const phi = (90 - latitude) * (Math.PI / 180);
        const theta = (longitude + 180) * (Math.PI / 180);
        const x = -this.earthRadius * Math.sin(phi) * Math.cos(theta);
        const y = this.earthRadius * Math.cos(phi);
        const z = this.earthRadius * Math.sin(phi) * Math.sin(theta);

        const geometry = new THREE.SphereGeometry(0.01, 32, 32);
        const material = new THREE.MeshBasicMaterial({ color: 0xff0000 });
        const marker = new THREE.Mesh(geometry, material);
        marker.position.set(x, y, z);
        marker.userData = radio;
        marker.onClick = () => {
          console.log('Clicked on marker:', marker.userData);
        };
        this.scene.add(marker);

        const invisibleSphereGeometry = new THREE.SphereGeometry(0.015, 32, 32);
        const invisibleSphereMaterial = new THREE.MeshBasicMaterial({ visible: false });
        const invisibleSphere = new THREE.Mesh(invisibleSphereGeometry, invisibleSphereMaterial);
        invisibleSphere.position.set(x, y, z);
        invisibleSphere.userData = radio;
        invisibleSphere.onClick = marker.onClick;
        this.scene.add(invisibleSphere);
      } else {
        console.error('Longitude or latitude is null');
      }
    },
    onDocumentMouseClick(event) {
      event.preventDefault();
      this.mouse.x = (event.offsetX / this.renderer.domElement.clientWidth) * 2 - 1;
      this.mouse.y = -(event.offsetY / this.renderer.domElement.clientHeight) * 2 + 1;
      this.raycaster.setFromCamera(this.mouse, this.camera);
      const intersects = this.raycaster.intersectObjects(this.scene.children);

      if (intersects.length > 0) {
        console.log('Clicked on:', intersects[0].object.userData);
        this.handleMarkerClick(intersects[0]);
      }
    },
    handleMarkerClick(intersectedObject) {
      // Gestisci il click sull'oggetto intersecato
      if (intersectedObject.object.onClick) {
        intersectedObject.object.onClick();
        const previousRadio = this.selectedRadio;
        this.selectedRadio = intersectedObject.object.userData;
        this.$nextTick(() => {
          if (this.selectedRadio.url_resolved || this.selectedRadio.url) {
            this.lazyLoadAudio(this.selectedRadio);
            if (previousRadio && previousRadio !== this.selectedRadio) {
              this.pauseRadio(previousRadio);
            }
          }
        });
      }
    },
    lazyLoadAudio(radio) {
      console.log('Lazy loading audio for:', radio);
      if (!radio.audioPlayer) {
        console.log('Creating new audio player');
        radio.audioPlayer = new Audio(radio.url_resolved || radio.url);
        console.log('Audio player:', radio.audioPlayer);
        radio.audioPlayer.onloadeddata = () => {
          console.log('Audio loaded');
        };
        radio.audioPlayer.onerror = () => {
          console.error('Error loading audio');
        };
      }
      else {
        console.log('Audio player already exists');
      }
    },
    retrieveFavorites() {
      const favorites = JSON.parse(localStorage.getItem('favorites')) || [];
      this.radios.forEach(radio => {
        const fav = favorites.find(f => f.changeuuid === radio.changeuuid);
        radio.favorite = fav ? fav.favorite : false;
      });
    },
    animate() {
      requestAnimationFrame(this.animate);
      if (this.controls) {
        this.controls.update();
      }
      if (this.renderer && this.scene && this.camera) {
        this.renderer.render(this.scene, this.camera);
      }
    },
  }
};
</script>

<style>
#container {
  width: 100%;
  height: 100%;
}

.footer p {
  font-family: Helvetica, Arial, sans-serif;
}

.footer {
  position: fixed;
  bottom: 0;
  left: 0;
  width: 100%;
  background-color: black;
  /* Cambio colore di sfondo */
  color: #fff;
  /* Cambio colore del testo */
  text-align: center;
  padding: 20px 0;
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}
</style>