<script lang="ts">
	import * as d3 from 'd3';

	interface Item {
		city: string;
		country: string;
		mainPollutant: string;
		pm25: number;
		state: string;
		stationName: string;
		timestamp: Date;
		usAqi: number;
	}

	// properties this component accepts
	const { data }: { data: Item[] } = $props();

	// just for debugging; can be removed
	$inspect(data);
</script>

<pre>
{JSON.stringify(data[0], null, 2)}
</pre>

<style>
	svg {
		font-family: sans-serif;
	}
</style>
