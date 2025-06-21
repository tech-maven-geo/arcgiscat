<template>
  <div id="app">
    <div class="sidebar">
      <CatalogLoader @load-catalog="handleLoadCatalog" />
      <TableOfContents ref="toc" :catalogLayer="loadedCatalogLayer" />
      <div v-if="catalogLoadError" class="error-message">
        Error loading catalog: {{ catalogLoadError }}
      </div>
    </div>
    <MapComponent ref="mapComponent" :catalogInfo="catalogToLoad" @map-ready="onMapReady" @catalog-loaded="onCatalogLoaded" @catalog-load-error="onCatalogLoadError" class="map-view-area" />
  </div>
</template>

<script>
import MapComponent from './components/MapComponent.vue';
import CatalogLoader from './components/CatalogLoader.vue';
import TableOfContents from './components/TableOfContents.vue';

export default {
  name: 'App',
  components: {
    MapComponent,
    CatalogLoader,
    TableOfContents
  },
  data() {
    return {
      catalogToLoad: null,
      mapView: null,
      loadedCatalogLayer: null,
      catalogLoadError: null
    };
  },
  methods: {
    handleLoadCatalog(catalogInfo) {
      console.log("App: Received catalog info", catalogInfo);
      this.catalogLoadError = null;
      this.loadedCatalogLayer = null; // Clear previous layer from TOC
      if (this.$refs.toc) { // Ensure TOC is rendered
        this.$refs.toc.datasets = []; // Clear TOC datasets directly or via a method
      }
      this.catalogToLoad = catalogInfo;
    },
    onMapReady(mapView) {
      this.mapView = mapView;
      console.log("App: Map is ready.");
    },
    onCatalogLoaded(catalogLayer) {
      this.loadedCatalogLayer = catalogLayer; // This will make it available to TOC via prop
      this.catalogLoadError = null;
      console.log("App: Catalog layer loaded successfully.", catalogLayer);
      // TOC will update via its watcher on `loadedCatalogLayer`
      // If explicit refresh is needed: this.$refs.toc.refresh();
    },
    onCatalogLoadError(errorMessage) {
      this.catalogLoadError = errorMessage;
      console.error("App: Catalog load error:", errorMessage);
      this.loadedCatalogLayer = null; // Clear any previously loaded/failed layer
       if (this.$refs.toc) {
        this.$refs.toc.datasets = []; // Clear TOC
      }
    }
  }
};
</script>

<style>
html, body, #app {
  padding: 0;
  margin: 0;
  height: 100vh;
  width: 100vw;
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  display: flex; /* Use flexbox for layout */
  overflow: hidden; /* Prevent scrollbars on #app */
}

.sidebar {
  width: 350px; /* Adjust width as needed */
  height: 100%;
  /* padding: 10px; Let child components manage their padding */
  background-color: #f0f0f0;
  box-shadow: 2px 0 5px rgba(0,0,0,0.1);
  overflow-y: auto; /* Allow scrolling in sidebar if content exceeds height */
  z-index: 10; /* Ensure sidebar is above map if overlapping occurs */
  display: flex;
  flex-direction: column;
}

.sidebar > * { /* Ensure child components like CatalogLoader and TOC take full width of sidebar */
    width: 100%;
    box-sizing: border-box; /* Include padding and border in the element's total width and height */
}


.map-view-area {
  flex-grow: 1; /* Map takes remaining space */
  height: 100%;
}

.error-message {
  color: red;
  margin-top: 10px;
  padding: 10px; /* Increased padding for better visibility */
  border: 1px solid red;
  background-color: #ffe0e0;
  text-align: center;
}
</style>
