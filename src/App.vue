<template>
  <div>
    <!-- Navbar -->
    <Navbar />

    <div class="main-container">
      <!-- Sidebar -->
      <Sidebar :totalDistance="totalDistance" :highestSpeed="highestSpeed" :nearestStore="nearestStore" />

      <!-- Map View -->
      <div id="map"></div>
    </div>

    <!-- Footer -->
     <Footer ></Footer>
  </div>
</template>

<script>
import { ref, onMounted } from 'vue';
import Navbar from './components/Navbar.vue';
import Sidebar from './components/Sidebar.vue';
import Footer from './components/Footer.vue';
import { Loader } from '@googlemaps/js-api-loader';
import Papa from 'papaparse';

export default {
  components: {
    Navbar,
    Sidebar,
    Footer
  },
  setup() {
    const map = ref(null);
    const vehiclePath = ref([]);
    const stores = ref([]);
    const nearestStore = ref(null);
    const totalDistance = ref(0);
    const highestSpeed = ref(0);

    const loadData = async () => {
      await loadVehicleData();
      await loadStoreData();
      calculateTripDetails();
      loadMap();
    };

    // Load vehicle data
    const loadVehicleData = async () => {
      const response = await fetch('/PathTravelled.json');
      vehiclePath.value = await response.json();
    };

    // Load store data
    const loadStoreData = async () => {
      const response = await fetch('/storesCopy.csv');
      const csvText = await response.text();
      Papa.parse(csvText, {
        header: true,
        complete: (result) => {
          stores.value = result.data.map(store => ({
            name: store.name,
            latitude: parseFloat(store.latitude),
            longitude: parseFloat(store.longitude)
          }));
        }
      });
    };

    // Calculate trip details
    const calculateTripDetails = () => {
      let total = 0;
      let maxSpeed = 0;
      let closestStoreDist = Infinity;

      vehiclePath.value.forEach((point, index) => {
        if (index > 0) {
          const prevPoint = vehiclePath.value[index - 1];
          total += haversine(prevPoint.latitude, prevPoint.longitude, point.latitude, point.longitude);
        }
        maxSpeed = Math.max(maxSpeed, point.speed);

        stores.value.forEach(store => {
          const distance = haversine(point.latitude, point.longitude, store.latitude, store.longitude);
          if (distance < closestStoreDist) {
            closestStoreDist = distance;
            nearestStore.value = { ...store, timestamp: point.timestamp };
          }
        });
      });

      totalDistance.value = total.toFixed(2);
      highestSpeed.value = maxSpeed;
    };

    // Haversine formula to calculate the distance between two coordinates
    const haversine = (lat1, lon1, lat2, lon2) => {
      const toRad = (x) => (x * Math.PI) / 180;
      const R = 6371; // Radius of the Earth in km
      const dLat = toRad(lat2 - lat1);
      const dLon = toRad(lon2 - lon1);
      const a =
        Math.sin(dLat / 2) * Math.sin(dLat / 2) +
        Math.cos(toRad(lat1)) * Math.cos(toRad(lat2)) * Math.sin(dLon / 2) * Math.sin(dLon / 2);
      return R * (2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a)));
    };

    // Initialize Google Map
    const loadMap = async () => {
      const loader = new Loader({ apiKey: 'AIzaSyCuNL81CpRRy0GFH0fkK2hqdiJliuyKzZU' });
      const google = await loader.load();

      map.value = new google.maps.Map(document.getElementById('map'), {
        center: { lat: vehiclePath.value[0].latitude, lng: vehiclePath.value[0].longitude },
        zoom: 12
      });

      plotVehiclePath(google);
      plotStores(google);
    };

    // Plot vehicle path on the map
    const plotVehiclePath = (google) => {
      const pathCoordinates = vehiclePath.value.map((p) => ({ lat: p.latitude, lng: p.longitude }));
      new google.maps.Polyline({
        path: pathCoordinates,
        geodesic: true,
        strokeColor: '#FF0000',
        strokeOpacity: 1.0,
        strokeWeight: 2,
        map: map.value
      });
    };

    // Plot store markers on the map
    const plotStores = (google) => {
      stores.value.forEach((store) => {
        new google.maps.Marker({
          position: { lat: store.latitude, lng: store.longitude },
          map: map.value,
          title: store.name
        });
      });
    };

    onMounted(() => {
      loadData();
    });

    return {
      totalDistance,
      highestSpeed,
      nearestStore
    };
  }
};
</script>

<style scoped>
.main-container {
  display: flex;
  margin-top: 50px; /* Adjust for navbar height */
}

#map {
  flex: 1;
  height: calc(100vh - 50px - 60px); /* Full height minus navbar and footer */
}

Footer {
  margin-top: auto;
}
</style>
