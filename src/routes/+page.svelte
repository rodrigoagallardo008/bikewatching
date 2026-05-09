<script>
  import mapboxgl from "mapbox-gl";
  import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";
  import { onMount } from "svelte";
  import * as d3 from 'd3';

  mapboxgl.accessToken = "pk.eyJ1Ijoicm9kcmlnb2FnYWxsYXJkbyIsImEiOiJjbW94bmZrenkwMWlhMnFxNGQ2M2QyaTJnIn0.GvNK5oaTMDx7p0LnrzVYPg";

  let map;
  let stations = $state([]);
  let mapViewChanged = $state(0);

  async function initMap() {
    map = new mapboxgl.Map({
      container: "map",
      style: "mapbox://styles/mapbox/streets-v12",
      center: [-71.09415, 42.36027],
      zoom: 12,
    });
    await new Promise(resolve => map.on("load", resolve));
    map.on("move", () => mapViewChanged++);

    // Boston bike lanes
    map.addSource("boston_route", {
      type: "geojson",
      data: "https://bostonopendata-boston.opendata.arcgis.com/datasets/boston::existing-bike-network-2022.geojson?outSR=%7B%22latestWkid%22%3A3857%2C%22wkid%22%3A102100%7D",
    });
    map.addLayer({
      id: "boston-bike-lanes",
      type: "line",
      source: "boston_route",
      paint: {
        "line-color": "green",
        "line-width": 3,
        "line-opacity": 0.4,
      },
    });

    // Cambridge bike lanes
    map.addSource("cambridge_route", {
      type: "geojson",
      data: "https://raw.githubusercontent.com/cambridgegis/cambridgegis_data/main/Recreation/Bike_Facilities/RECREATION_BikeFacilities.geojson",
    });
    map.addLayer({
      id: "cambridge-bike-lanes",
      type: "line",
      source: "cambridge_route",
      paint: {
        "line-color": "green",
        "line-width": 3,
        "line-opacity": 0.4,
      },
    });
  }

  async function loadStations() {
    stations = await d3.csv("https://vis-society.github.io/labs/9/data/bluebikes-stations.csv");
  }

  onMount(() => {
    initMap();
    loadStations();
  });

  function getCoords(station) {
    let point = new mapboxgl.LngLat(+station.Long, +station.Lat);
    let {x, y} = map.project(point);
    return {cx: x, cy: y};
  }
</script>

<h1>🚲 BikeWatch</h1>

<div id="map">
  <svg>
    {#key mapViewChanged}
      {#each stations as station}
        <circle {...getCoords(station)} r="5" fill="steelblue" fill-opacity="0.6" stroke="white" />
      {/each}
    {/key}
  </svg>
</div>

<style>
  @import url("$lib/global.css");

  #map {
    flex: 1;
    min-height: 600px;
  }

  #map svg {
    position: absolute;
    z-index: 1;
    width: 100%;
    height: 100%;
    pointer-events: none;
  }

  #map svg circle {
    pointer-events: auto;
  }
</style>
