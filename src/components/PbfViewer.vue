<template>
  <div class="content">
    <div class="canvas-container">
      <div v-if="state"> {{ state }}</div>
      <canvas ref="canvas"></canvas>
    </div>
    <div class="code-container"> {{ JSON.stringify(geojson, null, 2) }} </div>

  </div>
</template>

<script setup>
import * as vt from "@mapbox/vector-tile";
import Protobuf from "pbf";
let types = vt.VectorTileFeature.types;
import { nextTick, ref, useTemplateRef, watch } from "vue"


const props = defineProps({
  pbf: { type: Object }
})

const emit = defineEmits(["error"])

const canvas = useTemplateRef("canvas")
const canvasCtx = ref({})
const vectorTile = ref(null)
const state = ref("")
const geojson = ref({})

watch(() => props.pbf, async () => {
  if (!props.pbf) {
    state.value = "Please provide a pbf url or file"
    return;
  }
  await loadAndDraw();
}, { immediate: true })


async function loadAndDraw() {
  state.value = "Loading...";
  nextTick(() => {
    if (loadVectorTile()) {
      state.value = "Drawing...";
      redraw();
      state.value = "";
      nextTick(async () => {
        geojson.value = await toGeoJSON();
      })
    }
    else { state.value = "Error..." }

  })
}

function loadVectorTile() {
  let buffer = props.pbf;
  if (!buffer) return false;
  try {
    vectorTile.value = new vt.VectorTile(
      new Protobuf(new Uint8Array(buffer))
    );
  } catch (error) {
    console.error(error);
    emit(
      "error",
      "Unsupported pbf format. Loading resulted in: " + error
    );

    return false;
  }
  return true;
}

function redraw() {
  resetCanvas();
  drawVectorTile();
}

function resetCanvas() {
  let parent = canvas.value.parentElement;
  let width = Math.min(parent.offsetWidth, parent.offsetHeight);

  canvas.value.width = canvas.value.height = width - 4;
  canvasCtx.value = canvas.value.getContext("2d");
  canvasCtx.value.setTransform(1, 0, 0, 1, 0, 0);
  canvasCtx.value.clearRect(0, 0, canvas.value.width, canvas.value.height);
}

function setCanvasScale(tileExtent) {
  // translate canvas with 5% of the width
  let width = parseInt(canvas.value.width);
  canvasCtx.value.translate(width * 0.05, width * 0.05);

  // scale canvas to fit entire tileextent on 90% of the  current width
  let scale = (width / tileExtent) * 0.9;
  canvasCtx.value.scale(scale, scale);
}
function drawControlLine() {
  let style = canvasCtx.value.strokeStyle;
  canvasCtx.value.strokeStyle = "red";
  canvasCtx.value.beginPath();
  canvasCtx.value.moveTo(0, 0);
  canvasCtx.value.lineTo(0, 4096);
  canvasCtx.value.lineTo(4096, 4096);
  canvasCtx.value.lineTo(4096, 0);
  canvasCtx.value.lineTo(0, 0);

  canvasCtx.value.stroke();
  canvasCtx.value.strokeStyle = style;
}

function drawVectorTile() {
  if (!vectorTile.value) return;

  let keys = Object.keys(vectorTile.value.layers);
  if (keys.length == 0) {
    console.warn("No layers found. Nothing to plot");
    return;
  }

  let extent = vectorTile.value.layers[keys[0]].extent;
  setCanvasScale(extent);

  for (let key of keys) {
    let layer = vectorTile.value.layers[key];

    for (let i = 0; i < layer.length; i++) {
      let feature = layer.feature(i);

      let geometry = feature.loadGeometry();

      switch (types[feature.type]) {
        case "Polygon":
          drawPolygons(geometry);
          break;
        case "LineString":
          drawLines(geometry);
          break;
        case "Point":
          drawPoints(geometry);
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
  drawControlLine();
}

function drawPolygons(polygons) {
  polygons.forEach(polygon => {
    drawLine(polygon);
  });
}
function drawLines(lines) {
  lines.forEach(line => {
    drawLine(line);
  });
}
function drawPoints(points) {
  points.forEach(point => {
    drawPoint(point);
  });
}
function drawLine(line) {
  canvasCtx.value.beginPath();
  let point = line[0];
  canvasCtx.value.moveTo(point.x, point.y);

  line.slice(1).forEach(point => {
    canvasCtx.value.lineTo(point.x, point.y);
  });

  canvasCtx.value.stroke();
}
function drawPoint([point]) {
  // draw fill circle
  canvasCtx.value.beginPath();
  canvasCtx.value.arc(point.x, point.y, 10, 0, 2 * Math.PI);
  canvasCtx.value.stroke();
  canvasCtx.value.fill();
}
async function toGeoJSON() {
  return new Promise((resolve, reject) => {
    const features = {};
    if (!vectorTile.value) return features;
    let keys = Object.keys(vectorTile.value.layers);

    for (let key of keys) {
      let layer = vectorTile.value.layers[key];
      features[key] = {
        type: "FeatureCollection",
        features: []
      }
      for (let i = 0; i < layer.length; i++) {
        let feature = layer.feature(i);
        const geojson = feature.toGeoJSON();
        geojson.geometry = feature.loadGeometry();
        geojson.type = types[feature.type];
        features[key].features.push(geojson);
      }
    }
    resolve(features);
  })
}
</script>
