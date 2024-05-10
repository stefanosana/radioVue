<template>
  <div id="container"></div>
</template>

<script>
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';

export default {
  mounted() {
    this.initWorld();
    this.getRadios();
    this.animate();
  },
  methods: {
    initWorld() {
      this.scene = new THREE.Scene();
      this.camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
      this.renderer = new THREE.WebGLRenderer();
      this.renderer.setSize(window.innerWidth, window.innerHeight);
      document.getElementById('container').appendChild(this.renderer.domElement);

      this.controls = new OrbitControls(this.camera, this.renderer.domElement);
      this.controls.autoRotate = false;

      // Aggiungi luci per illuminare la scena (opzionale)
      const ambientLight = new THREE.AmbientLight(0xffffff, 0.5);
      this.scene.add(ambientLight);
      const directionalLight = new THREE.DirectionalLight(0xffffff, 0.5);
      directionalLight.position.set(0, 1, 1);
      this.scene.add(directionalLight);

      // Aggiungi un mappamondo
      const geometry = new THREE.SphereGeometry(5, 32, 32);
      const texture = new THREE.TextureLoader().load('/texture.jpg');
      const material = new THREE.MeshBasicMaterial({ map: texture });
      const globe = new THREE.Mesh(geometry, material);
      this.scene.add(globe);

      this.camera.position.z = 15;
    },
    getRadios() {
  fetch('https://nl1.api.radio-browser.info/json/stations/search?limit=100&countrycode=IT&hidebroken=true&order=clickcount&reverse=true')
    .then(response => response.json())
    .then(data => {
      data.forEach(station => {
        if (station.url_resolved) {
          const matchLatitude = station.url_resolved.match(/(?<=@)\d+\.\d+/);
          const matchLongitude = station.url_resolved.match(/(?<=\/)\d+\.\d+/);
          if (matchLatitude && matchLongitude) {
            const latitude = parseFloat(matchLatitude[0]);
            const longitude = parseFloat(matchLongitude[0]);
            this.addMarker(latitude, longitude, station.name);
          }
        }
      });
    })
    .catch(error => {
      console.error('Error fetching radios:', error);
    });
},

addMarker(latitude, longitude, name) {
  const markerGeometry = new THREE.SphereGeometry(0.1, 8, 8);
  const markerMaterial = new THREE.MeshBasicMaterial({ color: 0xff0000 });
  const marker = new THREE.Mesh(markerGeometry, markerMaterial);

  // Converti le coordinate latitudine e longitudine in coordinate cartesiane
  const phi = (90 - latitude) * (Math.PI / 180);
  const theta = (180 - longitude) * (Math.PI / 180);
  marker.position.setFromSphericalCoords(5, phi, theta);

  // Aggiungi un'etichetta al marker
  const labelDiv = document.createElement('div');
  labelDiv.textContent = name;
  labelDiv.style.position = 'absolute';
  labelDiv.style.color = '#ffffff';
  labelDiv.style.top = '-20px';
  labelDiv.style.left = '-20px';
  document.body.appendChild(labelDiv);

  const label = new THREE.CSS2DObject(labelDiv);
  marker.add(label);

  this.scene.add(marker);
},

    addMarker(latitude, longitude, name) {
      const markerGeometry = new THREE.SphereGeometry(0.1, 8, 8);
      const markerMaterial = new THREE.MeshBasicMaterial({ color: 0xff0000 });
      const marker = new THREE.Mesh(markerGeometry, markerMaterial);

      // Converti le coordinate latitudine e longitudine in coordinate cartesiane
      const phi = (90 - latitude) * (Math.PI / 180);
      const theta = (180 - longitude) * (Math.PI / 180);
      marker.position.setFromSphericalCoords(5, phi, theta);

      // Aggiungi un'etichetta al marker
      const labelDiv = document.createElement('div');
      labelDiv.textContent = name;
      labelDiv.style.position = 'absolute';
      labelDiv.style.color = '#ffffff';
      labelDiv.style.top = '-20px';
      labelDiv.style.left = '-20px';
      document.body.appendChild(labelDiv);

      const label = new THREE.CSS2DObject(labelDiv);
      marker.add(label);

      this.scene.add(marker);
    },
    animate() {
      requestAnimationFrame(this.animate);
      this.controls.update();
      this.renderer.render(this.scene, this.camera);
    }
  }
};
</script>

<style>
#container {
  width: 100%;
  height: 100%;
}
</style>