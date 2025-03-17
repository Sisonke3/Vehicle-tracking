<template>
    <div class="sidebar">
      <h3>Trip Details</h3>
      <p><strong>Total Distance:</strong> {{ totalDistance }} km</p>
      <p><strong>Highest Speed:</strong> {{ highestSpeed }} km/h</p>
      <p v-if="nearestStore"><strong>Nearest Store:</strong> {{ nearestStore.name }}</p>
      <p v-if="nearestStore">Timestamp: {{ nearestStore.timestamp }}</p>
    </div>
  </template>
  
  <script>
export default {
  props: {
    totalDistance: {
      type: String,
      required: true
    },
    highestSpeed: {
      type: [String, Number],  // Allow both String and Number types
      required: true
    },
    nearestStore: {
      type: Object,
      required: false
    }
  }
};
</script>

  
  <style scoped>
  
  .sidebar {
  background-color: #ffffff;
  padding: 20px;
  box-shadow: 2px 2px 5px rgba(0, 0, 0, 0.1);
}

h3 {
  color: #FF4500;
}
  h3 {
    margin-top: 0;
  }
  </style>
  