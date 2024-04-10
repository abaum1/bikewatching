<script>
	import mapboxgl from "mapbox-gl";
	import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";
	import * as d3 from "d3";
	mapboxgl.accessToken =
		"pk.eyJ1IjoiYW1lbGlhLWIiLCJhIjoiY2x1cm9mZzB1MDNsczJqb2lldXBkd3Z0dyJ9.UFyVxsKyA6Z4EvtOWDCkng";

	import { onMount } from "svelte";

	let map;
	let stations = [];
	let bbTrips = [];
	let mapViewChanged = 0;
	let departures = [];
	let arrivals = [];
	let radiusScale;
	let timeFilter = -1;
	let departuresByMinute = Array.from({ length: 1440 }, () => []);
	let arrivalsByMinute = Array.from({ length: 1440 }, () => []);

	const TRIP_DATA_URL =
		"https://vis-society.github.io/labs/8/data/bluebikes-traffic-2024-03.csv";

	onMount(async () => {
		// Wrap the map creation in a try-catch block to handle any potential errors
		try {
			map = new mapboxgl.Map({
				container: "map", // Assuming you have a div element with id="map-container" in your HTML
				zoom: 12,
				style: "mapbox://styles/mapbox/light-v10",
				center: [-71.08943706294963, 42.33920769686098],
			});
			await new Promise((resolve) => map.on("load", resolve));
			map.addSource("boston_route", {
				type: "geojson",
				data: "https://bostonopendata-boston.opendata.arcgis.com/datasets/boston::existing-bike-network-2022.geojson?outSR=%7B%22latestWkid%22%3A3857%2C%22wkid%22%3A102100%7D",
			});
			map.addLayer({
				id: "boston-bike-lanes", // A name for our layer (up to you)
				type: "line", // one of the supported layer types, e.g. line, circle, etc.
				source: "boston_route", // The id we specified in `addSource()`
				paint: {
					// paint params, e.g. colors, thickness, etc.
					"line-color": "green",
				},
			});
			// cambridge bike lanes
			map.addSource("cambridge_route", {
				type: "geojson",
				data: "https://raw.githubusercontent.com/cambridgegis/cambridgegis_data/main/Recreation/Bike_Facilities/RECREATION_BikeFacilities.geojson",
			});
			map.addLayer({
				id: "cambridge-bike-lanes", // A name for our layer (up to you)
				type: "line", // one of the supported layer types, e.g. line, circle, etc.
				source: "cambridge_route", // The id we specified in `addSource()`
				paint: {
					// paint params, e.g. colors, thickness, etc.
					"line-color": "green",
				},
			});
			stations = await d3.csv(
				"https://vis-society.github.io/labs/8/data/bluebikes-stations.csv",
			);

			bbTrips = await d3.csv(TRIP_DATA_URL).then((trips) => {
				for (let trip of trips) {
					trip.started_at = new Date(trip.started_at);
					trip.ended_at = new Date(trip.ended_at);

					let startMinutesIntoDay = minutesSinceMidnight(
						trip.started_at,
					);
					let endMinutesIntoDay = minutesSinceMidnight(trip.ended_at);

					departuresByMinute[startMinutesIntoDay].push(trip);
					arrivalsByMinute[endMinutesIntoDay].push(trip);
				}
				return trips;
			});

			departures = d3.rollup(
				bbTrips,
				(v) => v.length,
				(d) => d.start_station_id,
			);
			arrivals = d3.rollup(
				bbTrips,
				(v) => v.length,
				(d) => d.end_station_id,
			);
			stations = stations.map((station) => {
				let id = station.Number;
				station.arrivals = arrivals.get(id) ?? 0;
				station.departures = departures.get(id) ?? 0;
				station.totalTraffic = station.arrivals + station.departures;
				return station;
			});

			console.log("stations", stations);
		} catch (error) {
			console.error("Error loading map:", error);
		}
	});

	function getCoords(station) {
		let point = new mapboxgl.LngLat(+station.Long, +station.Lat);
		let { x, y } = map.project(point);
		return { cx: x, cy: y };
	}

	$: radiusScale = d3
		.scaleSqrt()
		.domain([0, d3.max(stations, (d) => d.totalTraffic)])
		.range([0, 25]);

	$: map?.on("move", (evt) => mapViewChanged++);
	$: timeFilterLabel =
		timeFilter === -1
			? "(any time)"
			: new Date(0, 0, 1, 0, timeFilter).toLocaleString("en", {
					timeStyle: "short",
				});

	function minutesSinceMidnight(date) {
		return date.getHours() * 60 + date.getMinutes();
	}

	function filterByMinute(tripsByMinute, minute) {
		// Normalize both to the [0, 1439] range
		// % is the remainder operator: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Remainder
		let minMinute = (minute - 60 + 1440) % 1440;
		let maxMinute = minute + (60 % 1440);

		if (minMinute > maxMinute) {
			let beforeMidnight = tripsByMinute.slice(minMinute);
			let afterMidnight = tripsByMinute.slice(0, maxMinute);
			return beforeMidnight.concat(afterMidnight).flat();
		} else {
			return tripsByMinute.slice(minMinute, maxMinute).flat();
		}
	}
	$: filteredDepartures = filterByMinute(departuresByMinute, timeFilter);
	$: filteredArrivals = filterByMinute(arrivalsByMinute, timeFilter);


	// Need to convert the array to an object first to call .get()
	$: filteredDeparturesAggregate = d3.rollup(
		filteredDepartures,
		v => v.length,
		d => d.start_station_id,
	);
	$: filteredArrivalsAggregate = d3.rollup(
		filteredArrivals,
		v => v.length,
		d => d.end_station_id,
	);


	$: stations =
		timeFilter === -1
			? stations
			: stations.map((station) => {
					station = { ...station }; // Clone to avoid mutating the original
					let id = station.Number;
					station.departures = filteredDeparturesAggregate.get(id) ?? 0;
					station.arrivals = filteredArrivalsAggregate.get(id) ?? 0;
					station.totalTraffic =
						station.departures + station.arrivals;
					return station;
				});

	// console.log("timeFilter", timeFilter);
</script>

<svelte:head>
	<link rel="icon" type="image/png" href="images/bike.svg" />
	<title>Bikewatching</title>
</svelte:head>

<h1>Bikewatching</h1>
<header class="timeselector">
    Filter by time:
    <input type="range" min="-1" max="1440" bind:value={timeFilter}>
    <time> 
		<!-- the fact that the timeFilterLabel label is abstracted into a variable means that it is just -->
		<!-- litening to the timeFilter but not blocking timeFilter execution. When I had the reactive component here
		it was reevaluating each time the input was revalutaing the ternary and causing jittery.-->
        { timeFilterLabel }
    </time>
</header>
<p>
	This is an immersive visualization map of bike traffic in the Boston area.
</p>
<div id="map">
	{#key mapViewChanged}
		<svg>
			{#each stations as station}
				<circle
					{...getCoords(station)}
					r={radiusScale(station.totalTraffic)}
					fill="steelblue"
				>
					<title
						>{station.totalTraffic} trips ({station.departures} departures,
						{station.arrivals} arrivals)</title
					>
				</circle>
			{/each}
		</svg>
	{/key}
</div>

<style>
	@import url("$lib/global.css");
	#map {
		flex: 1;
		background-color: yellowgreen;
	}
	#map svg {
		width: 100%;
		height: 100%;
		position: absolute;
		z-index: 1;
		pointer-events: none;

		circle {
			pointer-events: auto;
			fill-opacity: 60%;
			stroke: white;
		}
	}
	.timeselector {
		display: flex;
		gap: 1em;
		align-items: center;
		label {
			margin-left: auto;
		}
	}

	em {
		color: #888; /* Lighter color */
		font-style: italic; /* Italic font */
		display: block;
	}
	time {
		display: block;
	}
</style>
