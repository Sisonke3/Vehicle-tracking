<template>
  <div>
    <!-- Navbar: Displays the app's title and navigation (if needed) -->
    <Navbar />

    <div class="main-container">
      <!-- Sidebar: Displays trip details like distance, highest speed, and nearest store -->
      <Sidebar :totalDistance="totalDistance" :highestSpeed="highestSpeed" :nearestStore="nearestStore" />

      <!-- Map View: Displays the map with vehicle path and stores -->
      <div id="map"></div>
    </div>

    <!-- Footer: Display the app's footer information -->
    <Footer />
  </div>
</template>

<script>
// Import necessary Vue components and libraries
import { ref, onMounted } from 'vue';
import Navbar from '../components/Navbar.vue';
import Sidebar from '../components/Sidebar.vue';
import Footer from '../components/Footer.vue';
import { Loader } from '@googlemaps/js-api-loader';
import Papa from 'papaparse';

export default {
  components: {
    Navbar,
    Sidebar,
    Footer
  },
  setup() {
    const map = ref(null);  // Google Map reference
    const vehiclePath = ref([]);  // Vehicle path data
    const stores = ref([]);  // Store data from CSV
    const nearestStore = ref(null);  // Nearest store to the path
    const totalDistance = ref(0);  // Total distance traveled
    const highestSpeed = ref(0);  // Highest speed recorded
    const vehicleTimestamp = ref('');  // Timestamp of the closest store encounter

    // Load data and initialize map
    const loadData = async () => {
      await loadVehicleData();
      await loadStoreData();
      calculateTripDetails();
      loadMap();
    };

    // Fetch vehicle path data from the JSON file
    const loadVehicleData = async () => {
      const response = await fetch('/PathTravelled.json');
      vehiclePath.value = await response.json();
    };

    // Fetch store data from the CSV file using PapaParse
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

    // Calculate total distance, highest speed, and closest store
    const calculateTripDetails = () => {
      let total = 0;
      let maxSpeed = 0;
      let closestStoreDist = Infinity;

      vehiclePath.value.forEach((point, index) => {
        // Calculate the distance traveled by the vehicle
        if (index > 0) {
          const prevPoint = vehiclePath.value[index - 1];
          total += haversine(prevPoint.latitude, prevPoint.longitude, point.latitude, point.longitude);
        }

        // Track the highest speed during the trip
        maxSpeed = Math.max(maxSpeed, point.speed);

        // Calculate the closest store to the vehicle's current position
        stores.value.forEach(store => {
          const distance = haversine(point.latitude, point.longitude, store.latitude, store.longitude);
          if (distance < closestStoreDist) {
            closestStoreDist = distance;
            nearestStore.value = { ...store, timestamp: point.timestamp };
            vehicleTimestamp.value = point.timestamp;  // Store timestamp of first proximity to closest store
          }
        });
      });

      // Store the calculated values in refs for use in UI
      totalDistance.value = total.toFixed(2);
      highestSpeed.value = maxSpeed;
    };

    // Haversine formula to calculate the distance between two geo-coordinates
    const haversine = (lat1, lon1, lat2, lon2) => {
      const toRad = (x) => (x * Math.PI) / 180;
      const R = 6371; // Radius of Earth in km
      const dLat = toRad(lat2 - lat1);
      const dLon = toRad(lon2 - lon1);
      const a =
        Math.sin(dLat / 2) * Math.sin(dLat / 2) +
        Math.cos(toRad(lat1)) * Math.cos(toRad(lat2)) * Math.sin(dLon / 2) * Math.sin(dLon / 2);
      return R * (2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a)));
    };

    // Initialize Google Maps and plot vehicle path and stores
    const loadMap = async () => {
      const loader = new Loader({ apiKey: 'AIzaSyCuNL81CpRRy0GFH0fkK2hqdiJliuyKzZU' });
      const google = await loader.load();

      // Initialize map centered around the vehicle's starting position
      map.value = new google.maps.Map(document.getElementById('map'), {
        center: { lat: vehiclePath.value[0].latitude, lng: vehiclePath.value[0].longitude },
        zoom: 12
      });

      // Plot vehicle path on the map
      plotVehiclePath(google);
      // Plot store markers on the map
      plotStores(google);
    };

    // Plot the vehicle's path as a polyline on the map
    const plotVehiclePath = (google) => {
      const pathCoordinates = vehiclePath.value.map(p => ({ lat: p.latitude, lng: p.longitude }));
      new google.maps.Polyline({
        path: pathCoordinates,
        geodesic: true,
        strokeColor: '#FF4500', // Orange color for vehicle path
        strokeOpacity: 1.0,
        strokeWeight: 3,
        map: map.value
      });
    };

    // Plot store markers on the map
    const plotStores = (google) => {
      stores.value.forEach(store => {
        const marker = new google.maps.Marker({
          position: { lat: store.latitude, lng: store.longitude },
          map: map.value,
          title: store.name
        });

        // Add event listener for store marker click
        marker.addEventListener('click', () => {
          alert(`Store: ${store.name}\nLatitude: ${store.latitude}\nLongitude: ${store.longitude}`);
        });
      });
    };

    // Add interactivity to the path points to display information on hover/click
    const addPathInteractivity = () => {
      const pathCoordinates = vehiclePath.value.map(p => ({ lat: p.latitude, lng: p.longitude }));
      pathCoordinates.forEach((coordinate, index) => {
        // Add an event listener for each path point (hover/click to show info)
        new google.maps.Marker({
          position: coordinate,
          map: map.value,
          title: `Point ${index + 1}`
        }).addEventListener('click', () => {
          const point = vehiclePath.value[index];
          alert(`Timestamp: ${point.timestamp}\nSpeed: ${point.speed} km/h\nHeading: ${point.heading}`);
        });
      });
    };

    // Call interactivity function once map and path are loaded
    onMounted(() => {
      loadData();
      addPathInteractivity();
    });

    return {
      totalDistance,
      highestSpeed,
      nearestStore,
      vehicleTimestamp
    };
  }
};
</script>

<style scoped>
/* Main container to hold map and sidebar */
.main-container {
  display: flex;
  margin-top: 50px;
  padding: 20px;
}

/* Map styling */
#map {
  flex: 1;
  height: calc(100vh - 50px - 60px); /* Full height minus navbar and footer */
}

/* Sidebar styling for displaying vehicle details */
.sidebar {
  width: 300px;
  background-color: #ffffff;
  padding: 20px;
  box-shadow: 2px 2px 5px rgba(0, 0, 0, 0.1);
}

.sidebar h3 {
  font-size: 1.5em;
  color: #FF4500; /* Orange color for headings */
}

/* Footer styling */
footer {
  text-align: center;
  padding: 10px;
  background-color: #FF4500; /* Orange color for footer */
  color: white;
}

/* Responsive design for smaller screens */
@media (max-width: 768px) {
  .main-container {
    flex-direction: column;
  }

  .sidebar {
    width: 100%;
    margin-top: 20px;
  }

  #map {
    height: 400px;
  }
}
</style>
