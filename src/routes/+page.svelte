<script>
    import mapboxgl from "mapbox-gl";
    import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";
    import { onMount } from "svelte";
    import * as d3 from "d3";



    mapboxgl.accessToken = import.meta.env.VITE_MY_TOKEN;
    const myVal = import.meta.env.VITE_MY_TOKEN;

    let stations = [];
    let map;
    let mapViewChanged = 0;
    let timeFilter = -1;

    $: timeFilterLabel = new Date(0, 0, 0, 0, timeFilter)
                     .toLocaleString("en", {timeStyle: "short"});



    function getCoords (station) {
        let point = new mapboxgl.LngLat(+station.Long, +station.Lat);
        let {x, y} = map.project(point);
        return {cx: x, cy: y};
    }

    $: map?.on("move", evt => mapViewChanged++);


    onMount (async () => {
        stations = await d3.csv("https://vis-society.github.io/labs/8/data/bluebikes-stations.csv", station => ({
          ...station
        }));
        console.log(stations)

        map = new mapboxgl.Map({
            /* options */
            container: "map",
            style: "mapbox://styles/mapbox/streets-v12",
            center: [-71.3679725, 42.40010974514135],
            zoom: 10,
        });
        await new Promise(resolve => map.on("load", resolve));
        map.addSource("boston_route", {
            type: "geojson",
            data: "https://bostonopendata-boston.opendata.arcgis.com/datasets/boston::existing-bike-network-2022.geojson?outSR=%7B%22latestWkid%22%3A3857%2C%22wkid%22%3A102100%7D",
        });

        map.addLayer({
        id: "layer1", // A name for our layer (up to you)
        type: "line", // one of the supported layer types, e.g. line, circle, etc.
        source: "boston_route", // The id we specified in `addSource()`
        paint: {
            // paint params, e.g. colors, thickness, etc.
        },
        });


   })



</script>

<h1>Boston Bike System</h1>

<header>
    <label for="slider">Filter by time: </label>
    <input type="range" id="slider" bind:value={timeFilter} min="-1" max="1440">

    {#if timeFilter==-1}
        <em  id="time">(any time)</em>
    {:else}
        <time id="time">{timeFilterLabel}</time>
    {/if}
</header>

<div id="map">
	<svg>
        {#key mapViewChanged}
        <!-- render stations here -->
            {#each stations as station }
            <circle { ...getCoords(station) } r="5" fill="steelblue" />
            {/each}
        {/key}
    </svg>
</div>


<style>
    @import url("$lib/global.css");
    #map {
	flex: 1;
    }

    #map svg {
    position: absolute;
    z-index: 1;
    width: 100%;
    height: 100%;
    pointer-events: none;
    }

    header {
        display: flex;
        gap: 1em;
        align-items: baseline;
        margin-left: auto;
        display: block;
    }

    #time{
        display: block;

    }



</style>
