<script setup>
// Importa las funciones necesarias de Vue y Leaflet
import { onMounted } from 'vue';
import 'leaflet/dist/leaflet.css'; // Importante: importa los estilos de Leaflet
import L from 'leaflet'; // Importa la biblioteca de Leaflet

// 'onMounted' es un "hook" de Vue que se ejecuta
// cuando el componente está listo y montado en el DOM.
// Es el lugar perfecto para inicializar Leaflet,
// ya que necesita que el <div id="map"> exista.
onMounted(() => {
  // 1. Inicializa el mapa y lo centra en un punto (ej. coordenadas [0, 0])
  //    con un nivel de zoom inicial (ej. 1).
  const map = L.map('map').setView([0, 0], 1);

  // 2. Añade una capa de "azulejos" (tiles) al mapa.
  //    Esto es lo que ves (calles, etc.). Usaremos OpenStreetMap.
  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors'
  }).addTo(map);

  // 3. (Opcional) Añade un marcador de ejemplo
  L.marker([0, 0]).addTo(map)
    .bindPopup('¡Aquí comienza la aventura!')
    .openPopup();
});
</script>

<template>
  <header>
    <h1>HeroQuest Helper</h1>
  </header>

  <main>
    <div id="map"></div>
  </main>
</template>

<style scoped>
/* Es VITAL que el div del mapa tenga un alto definido */
#map {
  height: 70vh; /* 70% de la altura de la pantalla */
  width: 100%;
}

/* Estilos generales (puedes borrarlos) */
header {
  text-align: center;
  padding: 1rem;
}
</style>