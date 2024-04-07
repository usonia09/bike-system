<script>
    import mapboxgl from "mapbox-gl";
    import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";
    import { onMount } from "svelte";


    mapboxgl.accessToken = import.meta.env.VITE_MY_TOKEN;
    const myVal = import.meta.env.VITE_MY_TOKEN;

    onMount(() => {
	let map = new mapboxgl.Map({
		/* options */
        container: "map",
        style: "mapbox://styles/mapbox/streets-v12",
        center: [-71.3679725, 42.40010974514135],
        zoom: 10,
	});
   })


</script>

<h1>Boston Bike System</h1>
<h1>{myVal}</h1>
<div id="map"/>

<style>
    @import url("$lib/global.css");
    #map {
	flex: 1;
    background-color: blue;
    }


</style>
