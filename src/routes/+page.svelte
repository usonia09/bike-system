<script>
    import mapboxgl from "mapbox-gl";
    import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";
    import { onMount } from "svelte";
    import * as d3 from "d3";



    mapboxgl.accessToken = import.meta.env.VITE_MY_TOKEN;
    const myVal = "pk.eyJ1IjoidXNvbmlhIiwiYSI6ImNsdW9xMnZlYjBpZmkya3BiODN4aHJmaDEifQ.7_EpwmnX-wcM1QNdRvujJg";

    let stations = [];
    let filteredTrips = [];
    let map, trips, departures, arrivals, filteredArrivals, filteredDepartures, filteredStations;
    let minRadius = 0;
    let maxRadius = 25;
    let mapViewChanged = 0;
    let timeFilter = -1;
    let departuresByMinute = Array.from({length: 1440}, () => []);
    let arrivalsByMinute = Array.from({length: 1440}, () => []);


    $: radiusScale = d3.scaleSqrt()
	.domain([0, d3.max(stations, d => d.totalTraffic)])
	.range([0, 25]);


    let stationFlow = d3.scaleQuantize()
	.domain([0, 1])
	.range([0, 0.5, 1]);

    $: timeFilterLabel = new Date(0, 0, 0, 0, timeFilter)
                     .toLocaleString("en", {timeStyle: "short"});



    function getCoords (station) {
        let point = new mapboxgl.LngLat(+station.Long, +station.Lat);
        let {x, y} = map.project(point);
        return {cx: x, cy: y};
    }

    $: map?.on("move", evt => mapViewChanged++);

    function minutesSinceMidnight (date) {
        return date.getHours() * 60 + date.getMinutes();
    }

    function filterByMinute (tripsByMinute, minute) {
	// Normalize both to the [0, 1439] range
	// % is the remainder operator: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Remainder
	let minMinute = (minute - 60 + 1440) % 1440;
	let maxMinute = minute + 60 % 1440;

	if (minMinute > maxMinute) {
		let beforeMidnight = tripsByMinute.slice(minMinute);
		let afterMidnight = tripsByMinute.slice(0, maxMinute);
		return beforeMidnight.concat(afterMidnight).flat();
	}
	else {
		return tripsByMinute.slice(minMinute, maxMinute).flat();
	}
}


    // $: filteredTrips = timeFilter === -1? trips : trips.filter(trip => {
    //     let startedMinutes = minutesSinceMidnight(trip.started_at);
    //     let endedMinutes = minutesSinceMidnight(trip.ended_at);
    //     return Math.abs(startedMinutes - timeFilter) <= 60
    //         || Math.abs(endedMinutes - timeFilter) <= 60;
    // });

    $: filteredArrivals = timeFilter === -1? arrivals :d3.rollup(filterByMinute(arrivalsByMinute, timeFilter), v => v.length, d => d.start_station_id);
    $: filteredDepartures = timeFilter === -1? departures :d3.rollup(filterByMinute(departuresByMinute, timeFilter), v => v.length, d => d.end_station_id);

    $: filteredStations = timeFilter === -1? stations : stations.map (station => {
        station = {...station};
        let id = station.Number;
        station.arrivals = filteredArrivals.get(id) ?? 0;
        station.departures = filteredDepartures.get(id) ?? 0;
        station.totalTraffic = station.arrivals + station.departures
        return station;
    
    })

    onMount (async () => {
        stations = await d3.csv("https://vis-society.github.io/labs/8/data/bluebikes-stations.csv", station => ({
          ...station
        }));

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


        map.addSource("cambridge_route", {
            type: "geojson",
            data: "https://raw.githubusercontent.com/cambridgegis/cambridgegis_data/main/Recreation/Bike_Facilities/RECREATION_BikeFacilities.geojson",
        });

        map.addLayer({
        id: "layer1", // A name for our layer (up to you)
        type: "line", // one of the supported layer types, e.g. line, circle, etc.
        source: "boston_route", // The id we specified in `addSource()`
        paint: {
            // paint params, e.g. colors, thickness, etc.
            "line-color": "#DE75F3",
            "line-width": 3,
            "line-opacity":0.4
        },
        });

        map.addLayer({
        id: "layer2", // A name for our layer (up to you)
        type: "line", // one of the supported layer types, e.g. line, circle, etc.
        source: "cambridge_route", // The id we specified in `addSource()`
        paint: {
            // paint params, e.g. colors, thickness, etc.
            "line-color": "#DE75F3",
            "line-width": 3,
            "line-opacity":0.4
        },
        });

        trips = await d3.csv("https://vis-society.github.io/labs/8/data/bluebikes-traffic-2024-03.csv").then(trips => {
        for (let trip of trips) {
            trip.started_at = new Date(trip.started_at);
            trip.ended_at = new Date(trip.ended_at);
            let startedMinutes = minutesSinceMidnight(trip.started_at);
            departuresByMinute[startedMinutes].push(trip);
            let endedMinutes = minutesSinceMidnight(trip.ended_at);
            arrivalsByMinute[endedMinutes].push(trip)
        }
            return trips;
        });

        departures = d3.rollup(trips, v => v.length, d => d.start_station_id);
        arrivals = d3.rollup(trips, v => v.length, d => d.end_station_id);

        stations = stations.map(station => {
            let id = station.Number;
            station.arrivals = arrivals.get(id) ?? 0;
            station.departures = departures.get(id) ?? 0;
            station.totalTraffic = station.arrivals + station.departures
            return station;
        });

        trips = trips.map(trip => {
            let id = trip.Number;
            trip.arrivals = arrivals.get(id) ?? 0;
            trip.departures = departures.get(id) ?? 0;
            trip.totalTraffic = trip.arrivals + trip.departures
            return trip;
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
            {#each filteredStations as station }
            <circle { ...getCoords(station) } r="{radiusScale(station.totalTraffic)}" fill-opacity="50%" stroke="white" pointer-events="auto" style="--departure-ratio: { stationFlow(station.departures / station.totalTraffic) }">
                 <title>{station.totalTraffic} trips ({station.departures} departures, { station.arrivals} arrivals)</title>
            </circle>
                
            {/each}
        {/key}
    </svg>
</div>
<div class="legend">
	<div style="--departure-ratio: 1; text-align: start">More departures</div>
	<div style="--departure-ratio: 0.5; text-align: center">Balanced</div>
	<div style="--departure-ratio: 0; text-align: end">More arrivals</div>
</div>



<style>
    @import url("$lib/global.css");
    #map {
	flex: 1;
    }

    div {
        --color-departures: steelblue;
        --color-arrivals: darkorange;
        --color: color-mix(
            in oklch,
            var(--color-departures) calc(100% * var(--departure-ratio)),
            var(--color-arrivals)
        );
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

    #map svg circle {
        --color-departures: steelblue;
        --color-arrivals: darkorange;
        --color: color-mix(
            in oklch,
            var(--color-departures) calc(100% * var(--departure-ratio)),
            var(--color-arrivals)
        );
        fill: var(--color);

    }

    #time{
        display: block;

    }
    
    .legend {
        display: flex;
        gap: 1px;
        margin-block: 1em;
    }

    .legend div {
        flex: 1;
        background-color: var(--color);
        gap: 1px;
        font-weight: bold;
        color: white;
        padding: 10px;
    }


</style>
