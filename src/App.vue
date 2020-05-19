<template>
  <div id="app">
    <div class="top">
      <input type="text" v-model="pbfUrl" />
    </div>
    <div class="main">
      <div v-if="error" class="error">{{error}}</div>
      <PbfViewer v-else :pbf="pbfUrl" @error="onError" />
    </div>
    <div class="bottom">
      <div>
        {{title}} - Find the source code on
        <a
          href="https://github.com/MapDev-OSM/vt-pbf-viewer"
        >GitHub</a>
        © 2020 dorinoltean@MapDev
      </div>
    </div>
  </div>
</template>

<script>
import PbfViewer from "./components/PbfViewer.vue";

let defaultPbfUrl = process.env.BASE_URL + "data/1444.pbf";

export default {
  name: "App",

  components: {
    PbfViewer
  },
  data: function() {
    return {
      pbfUrl: defaultPbfUrl,
      title: "Vector Tile PBF Viewer",
      error: ""
    };
  },
  created: function() {
    document.title = this.title + " - MapDev";
  },
  watch: {
    pbfUrl: function() {
      this.error = "";
    }
  },
  methods: {
    onError(error) {
      this.error = error;
    }
  }
};
</script>
