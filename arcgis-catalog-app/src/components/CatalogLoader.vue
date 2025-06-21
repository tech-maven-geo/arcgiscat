<template>
  <div class="catalog-loader">
    <h4>Load Catalog Layer</h4>
    <div class="input-group">
      <label for="catalog-url">Catalog URL:</label>
      <input type="text" id="catalog-url" v-model="catalogUrl" placeholder="Enter catalog JSON URL">
    </div>
    <div class="input-group">
      <label for="catalog-file">Or Local File:</label>
      <input type="file" id="catalog-file" @change="handleFileUpload" accept=".json">
    </div>
    <button @click="loadCatalog">Load Catalog</button>
    <div v-if="errorMessage" class="error-message">{{ errorMessage }}</div>
  </div>
</template>

<script>
export default {
  name: "CatalogLoader",
  data() {
    return {
      catalogUrl: "",
      selectedFile: null,
      errorMessage: ""
    };
  },
  methods: {
    handleFileUpload(event) {
      this.selectedFile = event.target.files[0];
      this.catalogUrl = ""; // Clear URL if a file is selected
      this.errorMessage = "";
    },
    loadCatalog() {
      this.errorMessage = "";
      if (this.selectedFile) {
        const reader = new FileReader();
        reader.onload = (e) => {
          try {
            const catalogData = JSON.parse(e.target.result);
            this.$emit("load-catalog", { type: 'data', content: catalogData });
          } catch (error) {
            this.errorMessage = "Error parsing JSON file: " + error.message;
            console.error("Error parsing JSON file:", error);
          }
        };
        reader.onerror = (e) => {
          this.errorMessage = "Error reading file: " + e.target.error.name;
          console.error("Error reading file:", e.target.error);
        };
        reader.readAsText(this.selectedFile);
      } else if (this.catalogUrl.trim() !== "") {
        this.$emit("load-catalog", { type: 'url', content: this.catalogUrl.trim() });
      } else {
        this.errorMessage = "Please provide a URL or select a local JSON file.";
      }
    }
  }
};
</script>

<style scoped>
.catalog-loader {
  padding: 10px;
  background-color: #f9f9f9;
  border-radius: 5px;
  margin-bottom: 10px;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
}
.input-group {
  margin-bottom: 10px;
}
.input-group label {
  display: block;
  margin-bottom: 5px;
  font-weight: bold;
}
.input-group input[type="text"],
.input-group input[type="file"] {
  width: calc(100% - 22px); /* Adjust for padding/border */
  padding: 8px;
  border: 1px solid #ccc;
  border-radius: 4px;
}
button {
  padding: 10px 15px;
  background-color: #007bff;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 16px;
}
button:hover {
  background-color: #0056b3;
}
.error-message {
  color: #D8000C; /* Darker red for text */
  background-color: #FFD2D2; /* Light red background */
  border: 1px solid #D8000C;
  padding: 10px;
  margin-top: 10px;
  border-radius: 4px;
  font-size: 0.9em;
}
</style>
