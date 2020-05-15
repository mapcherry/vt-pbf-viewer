<template>
  <div v-if="error" class="error">{{error}}</div>
  <canvas v-else ref="canvas"></canvas>
</template>

<script>
import vt from "@mapbox/vector-tile";
import Protobuf from "pbf";
let types = vt.VectorTileFeature.types;

export default {
  name: "PbfViewer",
  props: {
    pbf: String
  },
  data: function() {
    return { error: "", canvas: {}, canvasCtx: {} };
  },
  async mounted() {
    await this.redraw();
  },
  watch: {
    async pbf() {
      this.error = "";
      await this.$nextTick(); // wait for rerender of html
      await this.redraw();
    }
  },
  methods: {
    async redraw() {
      this.resetCanvas();
      let buffer = await this.fetchPbfBuffer();
      if (!buffer) return;

      this.drawPbf(buffer);
    },
    resetCanvas() {
      let parent = this.$refs["canvas"].parentElement;
      let width = Math.min(parent.offsetWidth, parent.offsetHeight);
      this.canvas = this.$refs["canvas"];
      this.canvas.width = this.canvas.height = width - 4;
      this.canvasCtx = this.canvas.getContext("2d");
      this.canvasCtx.setTransform(1, 0, 0, 1, 0, 0);
      this.canvasCtx.clearRect(0, 0, this.canvas.width, this.canvas.height);
    },

    setCanvasScale(tileExtent) {
      // translate canvas with 5% of the width
      let width = parseInt(this.canvas.width);
      this.canvasCtx.translate(width * 0.05, width * 0.05);

      // scale canvas to fit entire tileextent on 90% of the  current width
      let scale = (width / tileExtent) * 0.9;
      this.canvasCtx.scale(scale, scale);
    },
    drawControlLine() {
      let style = this.canvasCtx.strokeStyle;
      this.canvasCtx.strokeStyle = "red";
      this.canvasCtx.beginPath();
      this.canvasCtx.moveTo(0, 0);
      this.canvasCtx.lineTo(0, 4096);
      this.canvasCtx.lineTo(4096, 4096);
      this.canvasCtx.lineTo(4096, 0);
      this.canvasCtx.lineTo(0, 0);

      this.canvasCtx.stroke();
      this.canvasCtx.strokeStyle = style;
    },

    drawPbf(arrayBuffer) {
      try {
        var tile = new vt.VectorTile(new Protobuf(new Uint8Array(arrayBuffer)));
      } catch (error) {
        this.error = "Unsupported pbf format. Loading resulted in: " + error;
        return;
      }
      let keys = Object.keys(tile.layers);
      if (keys.length == 0) {
        console.warn("No layers found. Nothing to plot");
        return;
      }

      let extent = tile.layers[keys[0]].extent;
      this.setCanvasScale(extent);

      for (let key of keys) {
        let layer = tile.layers[key];

        for (let i = 0; i < layer.length; i++) {
          let feature = layer.feature(i);
          let geometry = feature.loadGeometry();

          switch (types[feature.type]) {
            case "Polygon":
              this.drawPolygons(geometry);
              break;
            case "LineString":
              this.drawLines(geometry);
              break;
            case "Point":
              this.drawPoints(geometry);
              break;
            default:
              console.error(
                "No support for drawing type: ",
                types[feature.type]
              );
              break;
          }
        }
      }
      this.drawControlLine();
    },

    async fetchPbfBuffer() {
      try {
        let response = await fetch(this.pbf);
        return await response.arrayBuffer();
      } catch (ex) {
        this.error = "Could not load pbf file: " + ex;
        return;
      }
    },
    drawPolygons(polygons) {
      polygons.forEach(polygon => {
        this.drawLine(polygon);
      });
    },
    drawLines(lines) {
      lines.forEach(line => {
        this.drawLine(line);
      });
    },
    drawPoints(points) {
      points.forEach(point => {
        this.drawPoint(point);
      });
    },
    drawLine(line) {
      this.canvasCtx.beginPath();
      let point = line[0];
      this.canvasCtx.moveTo(point.x, point.y);

      line.slice(1).forEach(point => {
        this.canvasCtx.lineTo(point.x, point.y);
      });

      this.canvasCtx.stroke();
    },
    drawPoint(/*point*/) {
      // skip drawing  points
    }
  }
};
</script>
