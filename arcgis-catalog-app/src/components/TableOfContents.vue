<template>
  <div class="toc-container">
    <h4>Table of Contents</h4>
    <div v-if="!catalogLayer || !datasets.length" class="toc-empty">
      Load a catalog to see layers.
    </div>
    <ul v-else class="dataset-list">
      <li v-for="(dataset, index) in datasets" :key="dataset.id || dataset.name || index" class="dataset-item">
        <div class="dataset-header">
          <span class="dataset-name">{{ dataset.name || `Dataset ${index + 1}` }}</span>
          <button @click="zoomToDataset(dataset)" class="toc-button" title="Zoom to extent">Zoom To</button>
        </div>
        <div class="dataset-controls">
          <div class="control-group">
            <label :for="'visible-' + index">Visible:</label>
            <input type="checkbox" :id="'visible-' + index" v-model="dataset.visible" @change="toggleDatasetVisibility(dataset)">
          </div>
          <div class="control-group">
            <label :for="'opacity-' + index">Opacity:</label>
            <input type="range" :id="'opacity-' + index" min="0" max="1" step="0.1" v-model.number="dataset.opacity" @input="setDatasetOpacity(dataset)" :disabled="!dataset.visible">
          </div>
        </div>
        <!-- Layer reordering is complex with CatalogLayer's structure. -->
        <!-- Typically, CatalogLayer manages its own sublayers. -->
        <!-- We might be able to reorder the main CatalogLayer among other map layers, -->
        <!-- but not its internal datasets against each other easily unless API supports it. -->
        <!-- For now, reordering within TOC for CatalogLayer datasets is omitted. -->
      </li>
    </ul>
  </div>
</template>

<script>
export default {
  name: "TableOfContents",
  props: {
    catalogLayer: Object // The loaded CatalogLayer instance
  },
  data() {
    return {
      datasets: [] // Will hold { name, id, visible, opacity, sourceDataset }
    };
  },
  watch: {
    catalogLayer: {
      immediate: true,
      handler(newCatalogLayer) {
        this.datasets = []; // Clear previous datasets
        if (newCatalogLayer && newCatalogLayer.serviceDescription && newCatalogLayer.serviceDescription.datasets) {
          // Initialize internal state for each dataset for TOC controls
          this.datasets = newCatalogLayer.serviceDescription.datasets.map(ds => ({
            name: ds.name || ds.id, // Use name, fallback to id
            id: ds.id, // Assuming each dataset has a unique ID in the JSON
            sourceDataset: ds, // Keep original dataset info
            visible: newCatalogLayer.isDatasetVisible(ds.id), // Check initial visibility
            opacity: newCatalogLayer.getDatasetOpacity(ds.id) // Check initial opacity
          }));
        }
      }
    }
  },
  methods: {
    toggleDatasetVisibility(dataset) {
      if (!this.catalogLayer) return;
      // dataset.visible is already updated by v-model
      this.catalogLayer.setDatasetVisibility(dataset.id, dataset.visible);
      // If turning visible, ensure opacity is respected (it might be 0)
      // If turning off, opacity slider might be disabled, which is fine.
      // If turning on, opacity slider becomes enabled.
      // We may need to refresh the opacity value if it was changed while invisible.
      if(dataset.visible) {
        this.setDatasetOpacity(dataset); // re-apply current opacity
      }
    },
    setDatasetOpacity(dataset) {
      if (!this.catalogLayer || !dataset.visible) return;
      // dataset.opacity is already updated by v-model
      this.catalogLayer.setDatasetOpacity(dataset.id, dataset.opacity);
    },
    async zoomToDataset(dataset) {
      if (!this.catalogLayer || !this.catalogLayer.map) return;
      try {
        const extent = await this.catalogLayer.getDatasetExtent(dataset.id);
        if (extent) {
          this.catalogLayer.map.goTo(extent).catch(error => {
            console.warn("Error zooming to dataset extent:", error);
            // Fallback or notification if zoom fails
          });
        } else {
          console.warn("Extent not available for dataset:", dataset.name);
          // Notify user: Extent not available
        }
      } catch (error) {
        console.error("Error getting dataset extent:", error);
      }
    },
    // Method to refresh TOC when external changes happen (e.g., layer loaded, visibility changed by other means)
    // This could be called by the parent component if needed.
    refresh() {
        if (this.catalogLayer && this.catalogLayer.serviceDescription && this.catalogLayer.serviceDescription.datasets) {
            this.datasets = this.catalogLayer.serviceDescription.datasets.map(ds => ({
                ...ds, // spread original dataset properties
                name: ds.name || ds.id,
                id: ds.id,
                sourceDataset: ds,
                visible: this.catalogLayer.isDatasetVisible(ds.id),
                opacity: this.catalogLayer.getDatasetOpacity(ds.id)
            }));
        } else {
            this.datasets = [];
        }
    }
  },
  mounted() {
    // Initial population is handled by the watcher.
    // If the catalogLayer prop could be ready before mount in some edge cases (unlikely for async ops),
    // an explicit call to refresh might be needed here.
  }
};
</script>

<style scoped>
.toc-container {
  padding: 10px;
}
.toc-empty {
  color: #666;
  font-style: italic;
}
.dataset-list {
  list-style: none;
  padding: 0;
  margin: 0;
}
.dataset-item {
  background-color: #fff;
  border: 1px solid #ddd;
  padding: 10px;
  margin-bottom: 8px;
  border-radius: 4px;
  transition: background-color 0.2s ease-in-out;
}
.dataset-item:hover {
  background-color: #f5f5f5;
}
.dataset-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 8px;
}
.dataset-name {
  font-weight: bold;
  font-size: 1.1em;
}
.dataset-controls {
  display: flex;
  flex-direction: column; /* Stack controls vertically */
  gap: 8px; /* Space between control groups */
  margin-top: 8px;
}
.control-group {
  display: flex;
  align-items: center;
  gap: 5px; /* Space between label and input */
}
.dataset-controls label {
  font-size: 0.9em;
  flex-shrink: 0; /* Prevent label from shrinking */
}
.dataset-controls input[type="checkbox"] {
  margin: 0; /* Reset default margins */
  vertical-align: middle; /* Still useful for inline-block behavior if flex wasn't perfect */
}
.dataset-controls input[type="range"] {
  width: 120px; /* Slightly wider */
  vertical-align: middle;
}
.toc-button {
  padding: 5px 8px;
  background-color: #e7e7e7;
  border: 1px solid #ccc;
  border-radius: 3px;
  cursor: pointer;
  font-size: 0.9em;
}
.toc-button:hover {
  background-color: #d4d4d4;
}
</style>
