<script setup>
import { onMounted, ref } from "vue";
import "leaflet/dist/leaflet.css";
import L from "leaflet";
import mapTemplate from "../public/map_template.json";
import missions from "../public/missions.json";

// Solo necesitamos el estado de las áreas, no todo el JSON
const currentMission = ref(missions[0]);
const roomStates = ref([]);
import stairsImg from "../public/assets/staircase.png";
const TILE_SIZE = 300;
const DOOR_LENGTH = 300;
const DOOR_WIDTH = 100;

const baseUrl = import.meta.env.BASE_URL;
let mapInstance = null;

const discoveredDoors = ref({});

// --- FUNCIÓN PRINCIPAL DE INICIALIZACIÓN DE MAPA ---
const initializeMap = () => {
  const imgWidth = 7984;
  const imgHeight = 6403;
  const bounds = [
    [0, 0],
    [imgHeight, imgWidth],
  ];
  const imageUrl = `${baseUrl}tablero_base.jpg`; // CREAR MAPA

  mapInstance = L.map("map", {
    crs: L.CRS.Simple,
    minZoom: -4.5,
    maxZoom: 2,
    zoomSnap: 0.1,
  });

  mapInstance.createPane("fogPane");
  mapInstance.getPane("fogPane").style.zIndex = 410;

  mapInstance.createPane("doorPane");
  mapInstance.getPane("doorPane").style.zIndex = 420;

  // AÑADIR TABLERO BASE
  L.imageOverlay(imageUrl, bounds).addTo(mapInstance); // AJUSTAR VISTA INICIAL

  mapInstance.fitBounds(bounds, { maxZoom: -1.5 }); // Añadimos el fix de móvil que ya teníamos. // AÑADIR CONTROL DE RESETEO

  const ResetViewControl = L.Control.extend({
    options: { position: "topleft" },
    onAdd: function (map) {
      const container = L.DomUtil.create("div", "leaflet-bar leaflet-control");
      const button = L.DomUtil.create("a", "leaflet-control-button", container);
      button.innerHTML = "🏠";
      button.href = "#";
      button.role = "button";
      button.title = "Volver a la vista inicial"; // Evita que el clic en el botón haga zoom en el mapa
      L.DomEvent.disableClickPropagation(button);
      L.DomEvent.on(button, "click", function (e) {
        e.preventDefault();
        mapInstance.fitBounds(bounds, { maxZoom: -1.5 }); // Añadimos el fix de móvil aquí también.
      });
      return container;
    },
  });
  mapInstance.addControl(new ResetViewControl());

  const startCoords = currentMission.value.startLocation;

  // Agregar escaleras el inicio como entidad fija
  const entityRadius = TILE_SIZE;
  const entityBounds = [
    [startCoords[0] - entityRadius, startCoords[1] - entityRadius],
    [startCoords[0] + entityRadius, startCoords[1] + entityRadius],
  ];

  const staircaseOverlay = L.imageOverlay(stairsImg, entityBounds).addTo(
    mapInstance
  );

  staircaseOverlay.bindPopup("Inicio");

  // --- (Debug) OBTENER COORDENADAS ---
  mapInstance.on("click", function (e) {
    const yCoord = e.latlng.lat.toFixed(0);
    const xCoord = e.latlng.lng.toFixed(0);

    console.log(`Clic en coordenadas: [${yCoord}, ${xCoord}]`);
  });

  // ¡CARGAR LAS CAPAS DE LAS HABITACIONES!
  loadMissionAreas();
};

// --- GESTIONAR LAS HABITACIONES ---
function loadMissionAreas() {
  roomStates.value.forEach((room, index) => {
    // Solo creamos la capa si está inicialmente cerrada
    if (room.initialState === "CLOSED") {
      const areaBounds = room.mapArea; // L.rectangle es perfecto para cubrir áreas
      const fogColor = room.color || "black";

      const closedAreaOverlay = L.rectangle(areaBounds, {
        color: fogColor,
        weight: 1,
        fillColor: fogColor,
        fillOpacity: 0.9,
        pane: "fogPane",
      }).addTo(mapInstance);

      room.leafletOverlay = closedAreaOverlay; // Asignamos el evento de clic a la capa.

      closedAreaOverlay.on("click", (e) => {
        // Previene que el clic afecte a cosas debajo.
        L.DomEvent.stopPropagation(e); // ¡DESCUBRIR LA HABITACIÓN!

        discoverRoom(room, index);
      });
    } else {
      // Si la habitación está abierta, cargamos sus entidades desde el inicio
      loadRoomEntities(room);
      // Si está abierta al inicio, también cargamos las puertas asociadas
      currentMission.value.doors.forEach((door) => {
        if (door.linkedAreas.includes(room.id)) {
          addDoor(door);
        }
      });
    }
  });
}

// --- ENSAMBLAR EL MAPA ---
function assembleMapData() {
  const templateAreas = mapTemplate.areas || [];
  const missionAreas = currentMission.value.missionAreas || [];

  const combinedAreas = [];

  const missionMap = missionAreas.reduce((acc, area) => {
    acc[area.areaId] = acc[area.areaId] || [];
    acc[area.areaId].push(area);
    return acc;
  }, {});

  templateAreas.forEach((templateArea) => {
    const missionInfoList = missionMap[templateArea.id]; // Esto es un Arreglo [ {..} ]

    if (missionInfoList) {
      // Tomar el primer objeto de la lista para el estado
      const missionData = missionInfoList[0];

      const combined = {
        ...templateArea,
        ...missionData, // Desplegar el objeto, no el arreglo
        mapArea: templateArea.mapArea,
        color: templateArea.color,
        entities: missionData.entities || [], // Asegurar que entities sea un arreglo
      };
      combinedAreas.push(combined);
    } else {
      // Si la misión no la define, asumimos estado cerrado
      combinedAreas.push({
        ...templateArea,
        initialState: "CLOSED",
        entities: [],
      });
    }
  });

  roomStates.value = combinedAreas;
}

function loadRoomEntities(roomData) {
  const entityRadius = TILE_SIZE / 2;

  roomData.entities.forEach((entity) => {
    const imageUrl = new URL(
      `../public/assets/${entity.image}`,
      import.meta.url
    ).href;

    const y = entity.coords[0];
    const x = entity.coords[1];

    const entityBounds = [
      [y - entityRadius, x - entityRadius],
      [y + entityRadius, x + entityRadius],
    ];

    L.imageOverlay(imageUrl, entityBounds).addTo(mapInstance);
  });
}

// --- FUNCIÓN AL HACER CLIC EN UNA HABITACIÓN OCULTA ---
function discoverRoom(roomData, index) {
  console.log(`Descubriendo habitación: ${roomData.name} (ID: ${roomData.id})`);

  // Usamos el ID del área clicada.
  roomStates.value.forEach((roomPart, i) => {
    if (roomPart.id === roomData.id && roomPart.initialState === "CLOSED") {
      // Quitar la capa de "niebla de guerra" de esta parte
      if (roomPart.leafletOverlay) {
        roomPart.leafletOverlay.remove();
        delete roomPart.leafletOverlay;
      }

      // Actualizar el estado de Vue
      roomStates.value[i].initialState = "OPEN";

      // Cargar Monstruos, Trampas, etc.
      // NOTA: Solo cargamos las entidades y puertas UNA VEZ con el primer fragmento descubierto
      if (i === index) {
        loadRoomEntities(roomPart);

        currentMission.value.doors.forEach((door) => {
          if (door.linkedAreas.includes(roomPart.id)) {
            addDoor(door);
          }
        });
      }
    }
  });
}

function addDoor(door) {
  // 1. Evitar duplicados
  // Si esta puerta ya se ha renderizado, no hacer nada.
  if (discoveredDoors.value[door.id]) {
    return;
  }

  // 2. Marcarla como descubierta
  discoveredDoors.value[door.id] = true;

  // 3. Definir imagen y coordenadas
  const imageUrl = `${baseUrl}assets/door.png`;
  const [y, x] = door.coords;

  const halfLength = DOOR_LENGTH / 2;
  const halfWidth = DOOR_WIDTH / 2;

  let entityBounds;

  // 4. Calcular los 'bounds' según la orientación
  if (door.orientation === "V") {
    // Para una puerta vertical, el 'largo' (length) va en el eje Y
    // y el 'ancho' (width) en el eje X.
    entityBounds = [
      [y - halfLength, x - halfWidth], // [lat_min, lng_min]
      [y + halfLength, x + halfWidth], // [lat_max, lng_max]
    ];
  } else {
    // Para una puerta horizontal, el 'largo' va en el eje X
    // y el 'ancho' en el eje Y.
    entityBounds = [
      [y - halfWidth, x - halfLength], // [lat_min, lng_min]
      [y + halfWidth, x + halfLength], // [lat_max, lng_max]
    ];
  }

  // 5. Añadir la puerta al mapa como un imageOverlay
  const doorOverlay = L.imageOverlay(imageUrl, entityBounds, {
    pane: "doorPane",
  }).addTo(mapInstance);
}

// --- INICIO DE LA APLICACIÓN ---
onMounted(() => {
  assembleMapData();
  initializeMap();

  // Arreglo para que los marcadores de Leaflet se vean bien con Vite
  delete L.Icon.Default.prototype._getIconUrl;
  L.Icon.Default.mergeOptions({
    iconRetinaUrl:
      "https://unpkg.com/leaflet@1.9.4/dist/images/marker-icon-2x.png",
    iconUrl: "https://unpkg.com/leaflet@1.9.4/dist/images/marker-icon.png",
    shadowUrl: "https://unpkg.com/leaflet@1.9.4/dist/images/marker-shadow.png",
  });
});
</script>

<template>
   
  <header>
       
    <h1>{{ currentMission.title }}</h1>
       
    <p>{{ currentMission.briefing }}</p>
     
  </header>

   
  <main>
       
    <div id="map"></div>
     
  </main>
</template>

<style>
/* Arregla un problema común de Leaflet + Vite */
.leaflet-control-button {
  font-size: 1.2rem;
  line-height: 26px; /* Ajusta a la altura de los botones de Leaflet */
  width: 26px;
  height: 26px;
  text-align: center;
  cursor: pointer;
  background-color: white; /* Fondo blanco */
  border-radius: 4px;
  box-shadow: 0 1px 5px rgba(0, 0, 0, 0.65);
}

.leaflet-control-button:hover {
  background-color: #f4f4f4; /* Color al pasar el ratón */
}
</style>
<style scoped>
#map {
  height: 90vh;
  width: 100%;
  background-color: #333;
}
</style>
