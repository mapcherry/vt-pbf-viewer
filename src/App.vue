<template>
  <div id="app">
    <div
      class="p-6 flex flex-col items-center justify-center gap-6 bg-white/50 backdrop-blur-sm rounded-xl shadow-sm border border-gray-100 max-w-2xl mx-auto mt-8">
      <div class="flex flex-row items-center justify-center gap-6">
        <label class="flex items-center gap-2 cursor-pointer group">
          <input type="radio" class="w-5 h-5 accent-blue-600 cursor-pointer transition-transform group-hover:scale-110"
            v-model="mode" value="url" />
          <span class="text-gray-700 font-medium group-hover:text-blue-600 transition-colors">URL</span>
        </label>
        <label class="flex items-center gap-2 cursor-pointer group">
          <input type="radio" class="w-5 h-5 accent-blue-600 cursor-pointer transition-transform group-hover:scale-110"
            v-model="mode" value="file" />
          <span class="text-gray-700 font-medium group-hover:text-blue-600 transition-colors">File</span>
        </label>
      </div>

      <div class="w-full max-w-md">
        <input v-if="mode === 'url'" type="text" v-model="pbfUrl" placeholder="Enter PBF URL..."
          class="w-full px-4 py-2.5 bg-white border border-gray-300 text-gray-900 text-sm rounded-lg focus:ring-2 focus:ring-blue-500/40 focus:border-blue-500 block shadow-sm transition-all duration-200 outline-none" />

        <div v-if="mode === 'file'" class="w-full">
          <input type="file" @change="handleFileChange" accept=".pbf"
            class="block w-full text-sm text-gray-500 file:mr-4 file:py-2.5 file:px-4 file:rounded-full file:border-0 file:text-sm file:font-semibold file:bg-blue-50 file:text-blue-600 hover:file:bg-blue-100 transition-all cursor-pointer border border-gray-200 rounded-lg bg-white" />
        </div>
      </div>
    </div>
    <div class="main">
      <div v-if="error" class="error">{{ error }}</div>
      <PbfViewer v-else :pbf="pbfBuffer" @error="onError" />
    </div>
    <div class="bottom">
      <div>
        Vector Tiles PBF Viewer - Find the source code on
        <a href="https://github.com/mapcherry/vt-pbf-viewer">GitHub</a>
        © 2020 Mapcherry
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, watch } from "vue";
import PbfViewer from "./components/PbfViewer.vue";
import { useLocalStorage } from '@vueuse/core'

let defaultPbfUrl = "data/1444.pbf";
//let defaultPbfUrl = "https://api.mapcherry.io/tiles/mapcherry/grafitti-local/12/2316/1444.pbf?key=5b3e082044032a851e5b13b7d4915748";

const mode = useLocalStorage("mode", "url");
const pbfUrl = useLocalStorage("pbfUrl", defaultPbfUrl);
const selectedFile = ref(null);

const pbfBuffer = ref(null);
const error = ref("");


function onError(error) {
  error.value = error;
}

async function fetchPbfBuffer() {
  try {
    let response = await fetch(pbfUrl.value);
    if (response.status < 200 || response.status >= 300) {
      error.value = "Could not load pbf file: " + response.status + " " + await response.text();
      return;
    }
    return await response.arrayBuffer();
  } catch (ex) {
    error.value = "Could not load pbf file: " + ex;
    return;
  }
}

function handleFileChange(event) {
  selectedFile.value = event.target.files[0];
  loadFromFile();
}

watch(pbfUrl, async () => {
  if (mode.value === "url") {
    await loadFromUrl();
  }
})

watch(mode, async () => {
  pbfBuffer.value = null;
  if (mode.value === "url") {
    await loadFromUrl();
  }
  else {
    loadFromFile();
  }
}, {
  immediate: true
})

function loadFromFile() {
  if (!selectedFile.value) {
    error.value = "Please select a file";
    return;
  }
  error.value = "Loading...";
  const reader = new FileReader();
  reader.onload = (e) => {
    pbfBuffer.value = e.target.result;
    error.value = "";
  };
  reader.onerror = (e) => {
    error.value = "Could not load pbf file: " + e.target.error;
  };
  reader.readAsArrayBuffer(selectedFile.value);
}

async function loadFromUrl() {
  if (!pbfUrl.value) {
    error.value = "Please enter a URL";
    return;
  }
  error.value = "Loading...";
  pbfBuffer.value = await fetchPbfBuffer();
  if (pbfBuffer.value) {
    error.value = "";
  }
}

</script>
