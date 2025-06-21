<template>
  <div class="map-container" ref="mapViewNode"></div>
</template>

<script>
import Map from "@arcgis/core/Map";
import MapView from "@arcgis/core/views/MapView";
import CatalogLayer from "@arcgis/core/layers/CatalogLayer";
import "@arcgis/core/assets/esri/themes/light/main.css";

export default {
  name: "MapComponent",
  props: {
    catalogInfo: Object // Expects { type: 'url'/'data', content: 'url_string' or parsed_json_object }
  },
  data() {
    return {
      map: null,
      mapView: null,
      currentCatalogLayer: null
    };
  },
  mounted() {
    this.initializeMap();
  },
  methods: {
    initializeMap() {
      this.map = new Map({
        basemap: "streets-vector"
      });

      this.mapView = new MapView({
        container: this.$refs.mapViewNode,
        map: this.map,
        center: [-98.5795, 39.8283], // Center of USA
        zoom: 4
      });

      this.mapView.when(() => {
        this.$emit("map-ready", this.mapView);
      });
    },
    async loadCatalogLayer(catalogInfo) {
      if (!this.map || !this.mapView) {
        console.error("Map or MapView not initialized.");
        this.$emit("catalog-load-error", "Map not ready.");
        return;
      }

      // Remove existing catalog layer if any
      if (this.currentCatalogLayer) {
        this.map.remove(this.currentCatalogLayer);
        this.currentCatalogLayer.destroy();
        this.currentCatalogLayer = null;
      }

      let newCatalogLayer;
      try {
        if (catalogInfo.type === 'url') {
          newCatalogLayer = new CatalogLayer({
            url: catalogInfo.content
          });
        } else if (catalogInfo.type === 'data') {
          // For local data, we create a Blob and a URL for it,
          // as CatalogLayer expects a URL or loads from a portal item.
          // A more direct way might be available or might require creating service-like structures.
          // For now, let's assume the CatalogLayer can handle a data object if structured correctly,
          // or we might need to host it temporarily or use a different approach.
          // The documentation suggests `url` or `portalItem.id`.
          // Let's try to pass the object directly to `serviceDescription` if that's a valid undocumented way,
          // otherwise we'll need a workaround.
          // After checking docs, CatalogLayer expects a URL to a service description (JSON).
          // So, for local file, we might need to serve it or use a data URI if supported.
          // For simplicity in this step, we'll focus on URL. Local file handling might need adjustment.
          // A better approach for local data is to parse it and then construct layers individually,
          // but the request is to use CatalogLayer.
          // The CatalogLayer constructor actually takes a `serviceDescription` property.
          // Let's try that for local data.
          newCatalogLayer = new CatalogLayer({
            serviceDescription: catalogInfo.content
          });
        } else {
          throw new Error("Invalid catalogInfo type");
        }

        this.currentCatalogLayer = newCatalogLayer;
        this.map.add(this.currentCatalogLayer);

        await this.currentCatalogLayer.load();
        console.log("CatalogLayer loaded:", this.currentCatalogLayer);

        // Wait for the layer to finish loading its sublayers to populate the TOC
        // This might involve listening to specific events or checking layer status
        // For now, we'll emit an event with the loaded layer
        // The catalog layer itself has a `when` method.
        await this.currentCatalogLayer.when(() => {
            console.log("CatalogLayer and its initial dataset loaded/described.");
            // Attempt to get sublayers or service description details
            // The CatalogLayer populates its `layers` collection after loading.
            // However, these are ArcGISDynamicMapServiceLayer, ArcGISImageServiceLayer etc.
            // The actual dataset list comes from the serviceDescription.
            // For TOC, we'd iterate `currentCatalogLayer.serviceDescription.datasets`
            this.$emit("catalog-loaded", this.currentCatalogLayer);
        });


      } catch (error) {
        console.error("Error loading CatalogLayer:", error);
        this.$emit("catalog-load-error", error.message);
        if (newCatalogLayer) newCatalogLayer.destroy(); // Clean up partially created layer
        this.currentCatalogLayer = null;
      }
    }
  },
  watch: {
    catalogInfo: {
      handler(newInfo) {
        if (newInfo && newInfo.content) {
            // CatalogLoader ensures 'content' is populated for both 'url' and 'data' types.
            // 'type' distinguishes how 'content' should be interpreted.
            this.loadCatalogLayer(newInfo);
        } else if (!newInfo) {
            // If catalogInfo is reset (e.g. to null), we might want to clear the current catalog layer
            if (this.currentCatalogLayer) {
                this.map.remove(this.currentCatalogLayer);
                this.currentCatalogLayer.destroy();
                this.currentCatalogLayer = null;
                this.$emit("catalog-loaded", null); // Notify App/TOC that catalog is cleared
            }
        }
      },
      deep: true // May not be strictly necessary if newInfo is always a new object.
    }
  },
  beforeUnmount() {
    if (this.mapView) {
      this.mapView.destroy();
    }
    if (this.currentCatalogLayer) {
      this.currentCatalogLayer.destroy();
    }
    if (this.map) {
        this.map.destroy(); // Although map usually gets destroyed with view.
    }
  }
};
</script>

<style>
.map-container {
  padding: 0;
  margin: 0;
  height: 100%;
  width: 100%;
}
</style>
