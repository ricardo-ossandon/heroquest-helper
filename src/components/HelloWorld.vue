<script setup>
import { onMounted } from 'vue';
import 'leaflet/dist/leaflet.css';
import L from 'leaflet';

const baseUrl = import.meta.env.BASE_URL;

onMounted(() => { 
  // --- DEFINICIÓN DE COORDENADAS ---
  // Leaflet trabaja con coordenadas [Y, X] (o [lat, lng]) por defecto.
  // Cuando usamos CRS.Simple, podemos pensar en [Y, X] o [fila, columna].
  // Definimos las dimensiones de tu imagen.
  const imgWidth = 7984;
  const imgHeight = 6403;

  // 1. --- INICIALIZAR EL MAPA ---
  // Ahora usamos 'L.map' con opciones especiales.
  const map = L.map('map', {
    crs: L.CRS.Simple, // ¡IMPORTANTE! Usa un sistema de coordenadas simple (X, Y)
    minZoom: -3,       // Ajusta el zoom mínimo (permite alejar más)
    maxZoom: 2,        // Ajusta el zoom máximo (permite acercar más)
    zoomSnap: 0.1,     // Permite valores de zoom intermedios
  });

  // 2. --- DEFINIR LÍMITES DE LA IMAGEN ---
  // Le decimos a Leaflet dónde "colocar" la imagen en el sistema de coordenadas.
  // Lo más fácil es decir que la esquina superior izquierda es [0, 0]
  // y la esquina inferior derecha es [ALTO, ANCHO].
  const bounds = [[0, 0], [imgHeight, imgWidth]];

  // 3. --- AÑADIR LA IMAGEN DEL TABLERO ---
  // Usamos 'L.imageOverlay' en lugar de 'L.tileLayer'.
  // La URL '/tablero_base.jpg' funciona porque está en la carpeta 'public'.
  const imageUrl = `${baseUrl}tablero_base.jpg`;
  L.imageOverlay(imageUrl, bounds).addTo(map);

  // 4. --- AJUSTAR LA VISTA INICIAL ---
  // Le decimos a Leaflet que ajuste el zoom para que la imagen
  // quepa perfectamente en el contenedor del mapa.
  map.fitBounds(bounds);

  // 5. --- (Opcional) BOTÓN PARA RESETEAR VISTA ---
  // Vamos a añadir un control personalizado en el mapa.
  // Esto cumple tu requisito de "volver a la vista global".
  const ResetViewControl = L.Control.extend({
    options: {
      position: 'topleft' // Dónde poner el botón
    },
    onAdd: function(map) {
      const container = L.DomUtil.create('div', 'leaflet-bar leaflet-control');
      const button = L.DomUtil.create('a', 'leaflet-control-button', container);
      button.innerHTML = '🏠'; // Ícono de casa (o usa una imagen)
      button.href = '#';
      button.role = 'button';
      button.title = 'Volver a la vista inicial';

      // Evita que el clic en el botón haga zoom en el mapa
      L.DomEvent.disableClickPropagation(button);

      // Define la acción del botón
      L.DomEvent.on(button, 'click', function(e) {
        e.preventDefault();
        map.fitBounds(bounds); // ¡La misma función de antes!
      });

      return container;
    }
  });

  // Añade el nuevo control al mapa
  map.addControl(new ResetViewControl());


  // 6. --- (Prueba) AÑADIR UN MARCADOR ---
  // Ahora las coordenadas son píxeles. [Y, X]
  // Pongamos un marcador en el centro.
  const center = [imgHeight / 2, imgWidth / 2];
  
  // (Leaflet a veces invierte los iconos por defecto, esto lo arregla)
  delete L.Icon.Default.prototype._getIconUrl;
  L.Icon.Default.mergeOptions({
    iconRetinaUrl: 'https://unpkg.com/leaflet@1.9.4/dist/images/marker-icon-2x.png',
    iconUrl: 'https://unpkg.com/leaflet@1.9.4/dist/images/marker-icon.png',
    shadowUrl: 'https://unpkg.com/leaflet@1.9.4/dist/images/marker-shadow.png',
  });

  L.marker(center).addTo(map)
    .bindPopup('El centro del tablero');

  // --- (Prueba) OBTENER COORDENADAS ---
  // Esto es MUY útil para ti. Escucha el evento 'click' en el mapa
  // y te dice en la consola las coordenadas [Y, X] donde hiciste clic.
  // Úsalo para "mapear" dónde están tus habitaciones.
  map.on('click', function(e) {
    console.log('Clic en coordenadas: ', e.latlng);
  });

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

<style>
/* Estilos globales (ahora en <style> normal, no 'scoped') */

/* Arregla un problema común de Leaflet + Vite */
.leaflet-control-button {
  font-size: 1.2rem;
  line-height: 26px; /* Ajusta a la altura de los botones de Leaflet */
  width: 26px;
  height: 26px;
  text-align: center;
  cursor: pointer;
  background-color: white; /* Fondo blanco */
  border-radius: 4px; /* Bordes redondeados */
  box-shadow: 0 1px 5px rgba(0,0,0,0.65); /* Sombra */
}

.leaflet-control-button:hover {
  background-color: #f4f4f4; /* Color al pasar el ratón */
}
</style>

<style scoped>
#map {
  /* Hacemos que ocupe casi toda la pantalla */
  height: 90vh;
  width: 100%;
  background-color: #333; /* Un fondo oscuro mientras carga la imagen */
}

header {
  text-align: center;
  padding: 0.5rem;
  background-color: #1a1a1a;
  color: white;
}
</style>