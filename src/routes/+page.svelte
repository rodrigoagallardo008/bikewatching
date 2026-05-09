<script>
  import mapboxgl from "mapbox-gl";
  import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";
  import { onMount } from "svelte";
  import * as d3 from 'd3';

  mapboxgl.accessToken = "pk.eyJ1Ijoicm9kcmlnb2FnYWxsYXJkbyIsImEiOiJjbW94bmZrenkwMWlhMnFxNGQ2M2QyaTJnIn0.GvNK5oaTMDx7p0LnrzVYPg";

  let map;
  let stations = $state([]);
  let mapViewChanged = $state(0);
  let trips = [];
  let departures = $state(new Map());
  let arrivals = $state(new Map());
  const departuresByMinute = Array.from({length: 1440}, () => []);
  const arrivalsByMinute = Array.from({length: 1440}, () => []);
  let timeFilter = $state(-1);
  let timeFilterLabel = $derived(
    new Date(0, 0, 0, 0, timeFilter).toLocaleString("en", {timeStyle: "short"})
  );

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

  function minutesSinceMidnight(date) {
    return date.getHours() * 60 + date.getMinutes();
  }

  function filterByMinute(tripsByMinute, minute) {
    let minMinute = (minute - 60 + 1440) % 1440;
    let maxMinute = (minute + 60) % 1440;
    if (minMinute > maxMinute) {
      return [...tripsByMinute.slice(minMinute), ...tripsByMinute.slice(0, maxMinute)].flat();
    } else {
      return tripsByMinute.slice(minMinute, maxMinute).flat();
    }
  }

  async function loadStations() {
    stations = await d3.csv("https://vis-society.github.io/labs/9/data/bluebikes-stations.csv");
  }

  async function loadTrips() {
    trips = await d3.csv("https://vis-society.github.io/labs/9/data/bluebikes-traffic-2024-03.csv").then(trips => {
      for (let trip of trips) {
        trip.started_at = new Date(trip.started_at);
        trip.ended_at = new Date(trip.ended_at);
        departuresByMinute[minutesSinceMidnight(trip.started_at)].push(trip);
        arrivalsByMinute[minutesSinceMidnight(trip.ended_at)].push(trip);
      }
      return trips;
    });

    departures = d3.rollup(trips, v => v.length, d => d.start_station_id);
    arrivals = d3.rollup(trips, v => v.length, d => d.end_station_id);

    stations = stations.map(station => {
      let id = station.Number;
      station.arrivals = arrivals.get(id) ?? 0;
      station.departures = departures.get(id) ?? 0;
      station.totalTraffic = station.arrivals + station.departures;
      return station;
    });
  }

  onMount(() => {
    initMap();
    loadStations().then(loadTrips);
  });

  let filteredDepartures = $derived(
    timeFilter === -1
      ? departures
      : d3.rollup(filterByMinute(departuresByMinute, timeFilter), v => v.length, d => d.start_station_id)
  );

  let filteredArrivals = $derived(
    timeFilter === -1
      ? arrivals
      : d3.rollup(filterByMinute(arrivalsByMinute, timeFilter), v => v.length, d => d.end_station_id)
  );

  let filteredStations = $derived(
    stations.map(station => {
      const id = station.Number;
      const arr = selectedStation ? (arrivals.get(id) ?? 0) : (filteredArrivals.get(id) ?? 0);
      const dep = selectedStation ? (departures.get(id) ?? 0) : (filteredDepartures.get(id) ?? 0);
      return { ...station, arrivals: arr, departures: dep, totalTraffic: arr + dep };
    })
  );

  let radiusScale = $derived(
    d3.scaleSqrt()
      .domain([0, d3.max(filteredStations, d => d.totalTraffic) || 0])
      .range(selectedStation ? [0, 25] : timeFilter === -1 ? [0, 25] : [3, 30])
  );

  let stationFlow = d3.scaleQuantize().domain([0, 1]).range([0, 0.5, 1]);

  let selectedStation = $state(null);
  let isochrone = $state(null);
  const urlBase = 'https://api.mapbox.com/isochrone/v1/mapbox/';
  const profile = 'cycling';
  const minutes = [5, 10, 15, 20];
  const contourColors = ['03045e', '0077b6', '00b4d8', '90e0ef'];

  async function getIso(lon, lat) {
    const base = `${urlBase}${profile}/${lon},${lat}`;
    const params = new URLSearchParams({
      contours_minutes: minutes.join(','),
      contours_colors: contourColors.join(','),
      polygons: 'true',
      access_token: mapboxgl.accessToken
    });
    const url = `${base}?${params.toString()}`;
    const query = await fetch(url, { method: 'GET' });
    isochrone = await query.json();
  }

  $effect(() => {
    if (selectedStation) {
      getIso(+selectedStation.Long, +selectedStation.Lat);
    } else {
      isochrone = null;
    }
  });

  function geoJSONPolygonToPath(feature) {
    const path = d3.path();
    const rings = feature.geometry.coordinates;
    for (const ring of rings) {
      for (let i = 0; i < ring.length; i++) {
        const [lng, lat] = ring[i];
        const { x, y } = map.project([lng, lat]);
        if (i === 0) path.moveTo(x, y);
        else path.lineTo(x, y);
      }
      path.closePath();
    }
    return path.toString();
  }

  function getCoords(station) {
    let point = new mapboxgl.LngLat(+station.Long, +station.Lat);
    let {x, y} = map.project(point);
    return {cx: x, cy: y};
  }
</script>

<header>
  <h1>🚲 BikeWatch</h1>
  <label>
    Filter by time:
    <input type="range" min="-1" max="1440" bind:value={timeFilter} />
    {#if timeFilter !== -1}
      <time style="display: block">{timeFilterLabel}</time>
    {:else}
      <em style="display: block">(any time)</em>
    {/if}
  </label>
</header>

<div id="map">
  <svg>
    {#key mapViewChanged}
      {#if isochrone}
        {#each isochrone.features as feature}
          <path
            d={geoJSONPolygonToPath(feature)}
            fill={feature.properties.fillColor}
            fill-opacity="0.2"
            stroke="#000000"
            stroke-opacity="0.5"
            stroke-width="1"
          >
            <title>{feature.properties.contour} minutes of biking</title>
          </path>
        {/each}
      {/if}
      {#each filteredStations as station}
        <circle
          {...getCoords(station)}
          r={radiusScale(station.totalTraffic)}
          fill-opacity="0.6"
          stroke="white"
          style="--departure-ratio: {stationFlow(station.departures / station.totalTraffic)}"
          class={station.Number === selectedStation?.Number ? "selected" : ""}
          onmousedown={() => selectedStation = selectedStation?.Number !== station.Number ? station : null}
        >
          <title>{station.totalTraffic} trips ({station.departures} departures, {station.arrivals} arrivals)</title>
        </circle>
      {/each}
    {/key}
  </svg>
</div>

<div class="legend">
  <div style="--departure-ratio: 1">More departures</div>
  <div style="--departure-ratio: 0.5">Balanced</div>
  <div style="--departure-ratio: 0">More arrivals</div>
</div>

{#if selectedStation}
<div class="isochrone-legend">
  <strong>Cycling distance from selected station:</strong>
  {#each minutes as min, i}
    <span class="iso-swatch" style="background: #{contourColors[i]}">
      {min} min
    </span>
  {/each}
</div>
{/if}

<style>
  @import url("$lib/global.css");

  header {
    display: flex;
    gap: 1em;
    align-items: baseline;
  }
  header label {
    margin-left: auto;
  }
  em {
    color: #888;
    font-style: italic;
  }

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

    circle {
      transition: opacity 0.2s ease;
    }
    &:has(circle.selected) circle:not(.selected) {
      opacity: 0.3;
    }
    path {
      pointer-events: auto;
    }
  }

  :root {
    --color-departures: steelblue;
    --color-arrivals: darkorange;
  }

  #map svg circle, .legend > div {
    --color: color-mix(
      in oklch,
      var(--color-departures) calc(100% * var(--departure-ratio)),
      var(--color-arrivals)
    );
    fill: var(--color);
    pointer-events: auto;
  }

  .legend {
    display: flex;
    gap: 1px;
    margin-block: 1em;
  }

  .legend > div {
    flex: 1;
    padding: 0.4em 1em;
    text-align: center;
    font-size: 0.85em;
    color: white;
    background: var(--color);
  }

  .isochrone-legend {
    display: flex;
    align-items: center;
    gap: 0.5em;
    margin-block: 0.5em;
    font-size: 0.85em;
  }
  .iso-swatch {
    padding: 0.2em 0.6em;
    border-radius: 4px;
    color: white;
    opacity: 0.8;
  }
</style>
