<script lang="ts">
	import AQIChart from '$lib/AQIChart.svelte';
	import * as d3 from 'd3';
	import Line from '$lib/line.svelte';

	const datasets = {
		avalon: 'https://dig.cmu.edu/datavis-fall-2025/assignments/data/%5BAvalon%5D_daily-avg.csv',
		glassport_high_street:
			'https://dig.cmu.edu/datavis-fall-2025/assignments/data/%5BGlassport%20High%20Street%5D_daily-avg.csv',
		lawrenceville:
			'https://dig.cmu.edu/datavis-fall-2025/assignments/data/%5BLawrenceville%5D_daily-avg.csv',
		liberty_sahs:
			'https://dig.cmu.edu/datavis-fall-2025/assignments/data/%5BLiberty%20(SAHS)%5D_daily-avg.csv',
		manchester:
			'https://dig.cmu.edu/datavis-fall-2025/assignments/data/%5BManchester%5D_daily-avg.csv',
		north_braddock:
			'https://dig.cmu.edu/datavis-fall-2025/assignments/data/%5BNorth%20Braddock%5D_daily-avg.csv',
		parkway_east_near_road:
			'https://dig.cmu.edu/datavis-fall-2025/assignments/data/%5BParkway%20East%20(Near%20Road)%5D_daily-avg.csv',
		usa_pennsylvania_pittsburgh:
			'https://dig.cmu.edu/datavis-fall-2025/assignments/data/%5BUSA-Pennsylvania-Pittsburgh%5D_daily-avg.csv'
	};


	let localStorage = globalThis.localStorage ?? {};
	let selectedDataset: keyof typeof datasets = $state(localStorage.selectedDataset ?? "avalon");

	$effect(() => {localStorage.selectedDataset = selectedDataset;});					

	const data = $derived.by(() =>
		d3.csv(datasets[selectedDataset], (d: any) => ({
			city: d.City,
			country: d.Country,
			mainPollutant: d['Main pollutant'],
			pm25: +d['PM2.5'],
			state: d.State,
			stationName: d['Station name'],
			timestamp: new Date(d['Timestamp(UTC)']),
			usAqi: +d['US AQI']
		}))
	);

	type requiredColumns = { time:Date; aqi:number};

	async function clean(datasetKey: keyof typeof datasets): Promise<requiredColumns[]>{
		const url=datasets[datasetKey];
		const columns = await d3.csv(url,(d: any) =>({
			time: new Date(d["Timestamp(UTC)"]),
			aqi: +d["US AQI"]
		}));

		return columns
			.filter(r =>r.time instanceof Date && ! isNaN(+r.time) && Number.isFinite(r.aqi))
			.sort((a,b) => +a.time - +b.time)
	}

	const cleanedDataset = $derived.by(() => clean(selectedDataset));

	let query = $state("");




</script>

{#await cleanedDataset}
  <p>loading…</p>
{:then cleaned}
  <Line bind:selectedDataset data={cleaned} />
{:catch err}
  <p>failed: {err.message}</p>
{/await}



{#await data}
	<!-- promise is pending -->
	<p>loading data...</p>
{:then data}
	<!-- promise was fulfilled or not a Promise -->
	<h2>AQI Chart</h2>
	<AQIChart {data} />
{:catch error}
	<!-- promise was rejected -->
	<p>Something went wrong: {error.message}</p>
{/await}

<style>
	* {
		font-family: sans-serif;
	}


</style>
