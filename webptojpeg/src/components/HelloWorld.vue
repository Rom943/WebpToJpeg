<template>
  <div class="converter">
    <h2>WebP to JPEG Converter</h2>
    
    <div class="upload-container">
      <label for="file-input" class="upload-button">
        Select WebP Images
      </label>
      <input 
        type="file" 
        id="file-input" 
        multiple 
        accept=".webp" 
        @change="handleFileSelect"
        class="file-input"
      />
      <div class="upload-info" v-if="selectedFiles.length > 0">
        {{ selectedFiles.length }} file(s) selected
      </div>
    </div>
    
    <button 
      @click="convertImages" 
      :disabled="selectedFiles.length === 0 || isConverting"
      class="convert-button"
    >
      {{ isConverting ? 'Converting...' : 'Convert to JPEG' }}
    </button>
    
    <div class="results" v-if="convertedImages.length > 0">
      <h3>Converted Images</h3>
      <div class="image-list">
        <div v-for="(image, index) in convertedImages" :key="index" class="image-item">
          <div class="image-preview">
            <img :src="image.url" :alt="image.name" />
          </div>
          <div class="image-name">{{ image.name }}</div>
          <a :href="image.url" :download="image.name" class="download-link">Download</a>
        </div>
      </div>
      <button @click="downloadAllImages" class="download-all-button">
        Download All Images (ZIP)
      </button>
    </div>
    
    <div v-if="error" class="error-message">
      {{ error }}
    </div>
  </div>
</template>

<script>
import JSZip from 'jszip';

export default {
  name: 'WebpToJpegConverter',
  data() {
    return {
      selectedFiles: [],
      convertedImages: [],
      isConverting: false,
      error: null
    };
  },
  methods: {
    handleFileSelect(event) {
      const files = event.target.files;
      
      // Filter to only include WebP files
      this.selectedFiles = Array.from(files).filter(file => 
        file.type === 'image/webp' || file.name.toLowerCase().endsWith('.webp')
      );
      
      // Reset previous results
      this.convertedImages = [];
      this.error = null;
    },
    
    async convertImages() {
      if (this.selectedFiles.length === 0) return;
      
      this.isConverting = true;
      this.error = null;
      this.convertedImages = [];
      
      try {
        // Loop through files with index to create sequential filenames
        for (let i = 0; i < this.selectedFiles.length; i++) {
          const file = this.selectedFiles[i];
          // Pass index + 1 to create sequential names (img-1, img-2, etc.)
          const convertedImage = await this.convertWebpToJpeg(file, i + 1);
          this.convertedImages.push(convertedImage);
        }
      } catch (err) {
        console.error('Conversion error:', err);
        this.error = `Error converting images: ${err.message}`;
      } finally {
        this.isConverting = false;
      }
    },
    
    async convertWebpToJpeg(file, index) {
      return new Promise((resolve, reject) => {
        const reader = new FileReader();
        
        reader.onload = (e) => {
          const img = new Image();
          
          img.onload = () => {
            // Create a canvas with the same dimensions as the image
            const canvas = document.createElement('canvas');
            canvas.width = img.width;
            canvas.height = img.height;
            
            // Draw the image onto the canvas
            const ctx = canvas.getContext('2d');
            ctx.drawImage(img, 0, 0);
            
            // Convert to JPEG format
            try {
              // Convert to JPEG with 0.9 quality (90%)
              const jpegUrl = canvas.toDataURL('image/jpeg', 0.9);
              
              // Generate sequential filename
              const fileName = `img-${index}.jpg`;
              
              resolve({
                name: fileName,
                url: jpegUrl,
                originalFile: file
              });
            } catch (err) {
              reject(err);
            }
          };
          
          img.onerror = () => {
            reject(new Error(`Failed to load image: ${file.name}`));
          };
          
          // Set the source of the image to the loaded file
          img.src = e.target.result;
        };
        
        reader.onerror = () => {
          reject(new Error(`Failed to read file: ${file.name}`));
        };
        
        // Read the file as a data URL
        reader.readAsDataURL(file);
      });
    },
    
    async downloadAllImages() {
      if (this.convertedImages.length === 0) return;
      
      try {
        const zip = new JSZip();
        
        // Add each converted image to the zip
        this.convertedImages.forEach(image => {
          // Extract the base64 data from the data URL
          const base64Data = image.url.split(',')[1];
          zip.file(image.name, base64Data, { base64: true });
        });
        
        // Generate the zip file
        const zipBlob = await zip.generateAsync({ type: 'blob' });
        
        // Create a download link for the zip file
        const zipUrl = URL.createObjectURL(zipBlob);
        const link = document.createElement('a');
        link.href = zipUrl;
        link.download = 'converted_images.zip';
        document.body.appendChild(link);
        link.click();
        document.body.removeChild(link);
        
        // Clean up
        URL.revokeObjectURL(zipUrl);
      } catch (err) {
        console.error('Error creating zip file:', err);
        this.error = `Error creating zip file: ${err.message}`;
      }
    }
  }
};
</script>

<style scoped>
.converter {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
  background-color: #f9f9f9;
  border-radius: 8px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
}

h2 {
  color: #42b983;
  margin-bottom: 20px;
}

.upload-container {
  margin-bottom: 20px;
}

.file-input {
  display: none;
}

.upload-button {
  display: inline-block;
  padding: 10px 20px;
  background-color: #42b983;
  color: white;
  border-radius: 4px;
  cursor: pointer;
  transition: background-color 0.3s;
}

.upload-button:hover {
  background-color: #3aa876;
}

.upload-info {
  margin-top: 10px;
  font-size: 0.9em;
  color: #666;
}

.convert-button, .download-all-button {
  padding: 10px 20px;
  background-color: #2c3e50;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  transition: background-color 0.3s;
  font-size: 16px;
}

.convert-button:hover, .download-all-button:hover {
  background-color: #1c2e40;
}

.convert-button:disabled {
  background-color: #cccccc;
  cursor: not-allowed;
}

.results {
  margin-top: 30px;
}

.image-list {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
  gap: 20px;
  margin-bottom: 20px;
}

.image-item {
  border: 1px solid #ddd;
  border-radius: 4px;
  padding: 10px;
  background-color: white;
}

.image-preview {
  height: 150px;
  display: flex;
  align-items: center;
  justify-content: center;
  overflow: hidden;
  margin-bottom: 10px;
}

.image-preview img {
  max-width: 100%;
  max-height: 100%;
  object-fit: contain;
}

.image-name {
  font-size: 0.9em;
  margin-bottom: 10px;
  word-break: break-all;
}

.download-link {
  display: block;
  text-align: center;
  padding: 5px;
  background-color: #42b983;
  color: white;
  text-decoration: none;
  border-radius: 4px;
}

.download-all-button {
  margin-top: 10px;
}

.error-message {
  margin-top: 20px;
  color: #e74c3c;
  padding: 10px;
  background-color: #fadbd8;
  border-radius: 4px;
}
</style>