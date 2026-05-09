<script>
  import mapboxgl from "mapbox-gl";
  import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";
  import { onMount } from "svelte";

  mapboxgl.accessToken = "pk.eyJ1Ijoicm9kcmlnb2FnYWxsYXJkbyIsImEiOiJjbW94bmZrenkwMWlhMnFxNGQ2M2QyaTJnIn0.GvNK5oaTMDx7p0LnrzVYPg";

  let map;

  async function initMap() {
    map = new mapboxgl.Map({
      container: "map",
      style: "mapbox://styles/mapbox/streets-v12",
      center: [-71.09415, 42.36027],
      zoom: 12,
    });
    await new Promise(resolve => map.on("load", resolve));
  }

  onMount(() => {
    initMap();
  });
</script>

<h1>🚲 BikeWatch</h1>

<div id="map"></div>

<style>
  @import url("$lib/global.css");

  #map {
    flex: 1;
    min-height: 600px;
  }
</style>
