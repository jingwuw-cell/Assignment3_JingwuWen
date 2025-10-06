
<script lang="ts">
    import * as d3 from 'd3'
    import { line as d3line, curveLinear } from 'd3-shape';
    let { selectedDataset = $bindable('avalon'), data = [], stations = [] } = $props();


const W = 928, H = Math.round(W * 0.618);
const M = { top: 20, right: 30, bottom: 30, left: 40 };
const y_scale=1.1;
const x = d3.scaleUtc().range([M.left, W - M.right]);
const y = d3.scaleLinear().range([H - M.bottom, M.top]);

function toMonthlyMean(rows: Array<{ time: any; aqi: number }>) {
    const map = new Map<string, { y: number; m: number; sum: number; n: number }>();
    for (const r of rows) {
      const t = new Date(r.time);
      const a = +r.aqi;
      if (Number.isNaN(+t) || !Number.isFinite(a)) continue;
      const y = t.getUTCFullYear(), m = t.getUTCMonth();
      const k = `${y}-${m}`;
      const acc = map.get(k) ?? { y, m, sum: 0, n: 0 };
      acc.sum += a; acc.n += 1;
      map.set(k, acc);
    }
    return Array.from(map.values())
        .map(d => ({ time: new Date(Date.UTC(d.y, d.m, 15)), aqi: d.sum / d.n }))
        .sort((a, b) => +a.time - +b.time);
}

function toMonthlyP1090(rows: Array<{ time: any; aqi: number }>) {
  const g = d3.group(
    rows,
    (r) => new Date(r.time).getUTCFullYear(),
    (r) => new Date(r.time).getUTCMonth()
  );

  const out: Array<{ time: Date; p10: number; p90: number }> = [];
  for (const [yy, mMap] of g) {
    for (const [mm, arr] of mMap) {
      const vals = arr
        .map((d: any) => +d.aqi)
        .filter((v) => Number.isFinite(v))
        .sort(d3.ascending);
      if (!vals.length) continue;
      out.push({
        time: new Date(Date.UTC(+yy, +mm, 15)),
        p10: d3.quantile(vals, 0.10)!,
        p90: d3.quantile(vals, 0.90)!,
      });
    }
  }
  return out.sort((a, b) => +a.time - +b.time);
}


const mean_data = $derived(toMonthlyMean(data));

const band_data = $derived(toMonthlyP1090(data));
let bandPath = $state("");


const BANDS = [
  { name: "Good", min: 0,   max: 50,  color: "#9cd84e" },
  { name: "Moderate", min: 50,  max: 100, color: "#facf39" },
  { name: "USG", min: 100, max: 150, color: "#f99049" },
  { name: "Unhealthy", min: 150, max: 200, color: "#f65e5f" },
  { name: "Very Unhealthy", min: 200, max: 300, color: "#a070b6" },
  { name: "Hazardous", min: 300, max: Infinity, color: "#a06a7b" }
];

type BandRect = { id:string; x:number; y:number; w:number; h:number; color:string };

let bandRects = $state<BandRect[]>([]);

let pathD = $state("");


let RawBottonStorage = globalThis.localStorage ?? {};
let showRaw = $state((RawBottonStorage.showRaw ?? 'false') === 'true');
$effect(() => { RawBottonStorage.showRaw = String(showRaw); });


let rawPts = $state<Array<{cx:number; cy:number}>>([]);

$effect(() => {
    if (!mean_data.length) { pathD = ""; return; }

    const [xStart, xEnd] = d3.extent(mean_data, d => d.time) as [Date, Date];
    const xMin = d3.utcDay.offset(xStart, -15);
    const xMax = d3.utcDay.offset(xEnd,   +15);
    x.domain([xMin, xMax]);
    const ymaxRaw = (d3.max(data as Array<{aqi:number}>, d => d.aqi) ?? 0) * y_scale || 1;
    y.domain([0, ymaxRaw]);

    pathD = d3.line<{ time: Date; aqi: number }>()
        .x(d => x(d.time))
        .y(d => y(d.aqi))(mean_data) ?? "";

  const [yMin, yMax] = y.domain();
  const clamp = (v:number) => Math.max(yMin, Math.min(yMax, v));
  const plotX = M.left;
  const plotW = W - M.left - M.right;

bandRects = BANDS.map(b => {
  const lo = clamp(b.min), hi = clamp(b.max);
  const ok = hi > yMin && lo < yMax && lo < hi;
  if (!ok) return null;
  const yTop = y(hi);
  const yBot = y(lo);
  return { id: b.name, x: plotX, y: yTop, w: plotW, h: yBot - yTop, color: b.color };
}).filter(Boolean) as BandRect[];


    bandPath = band_data.length
    ? d3.area<{ time: Date; p10: number; p90: number }>()
        .x(d => x(d.time))
        .y0(d => y(d.p10))
        .y1(d => y(d.p90))(band_data) ?? ""
    : "";

    rawPts = (data as Array<{time: Date; aqi: number}>)
        .map(d => ({ cx: x(d.time), cy: y(d.aqi) }));

});



let gx: SVGGElement | null = null, gy: SVGGElement | null = null;


let edgeXs = $state<number[]>([]);

$effect(() => {
  if (!mean_data.length) { pathD = ""; edgeXs = []; return; }

  const firstX = x(mean_data[0].time);
  const lastX  = x(mean_data[mean_data.length - 1].time);
  edgeXs = firstX === lastX ? [firstX] : [firstX, lastX];
});

const T = 800;
let lineEl: SVGPathElement | null = null;

$effect(() => {
  if (!gx || !mean_data.length) return;
  d3.select(gx).transition().duration(T).call(d3.axisBottom(x).ticks(W / 80).tickSizeOuter(0));
});
$effect(() => {
  if (!gy || !mean_data.length) return;
  d3.select(gy).transition().duration(T).call(d3.axisLeft(y).ticks(H / 40));
});

//Transition for band

let bandsG: SVGGElement | null = null;
let hasRenderedBands = $state(false);

$effect(() => {
  if (!bandsG) return;

  const rects = d3
    .select(bandsG)
    .selectAll<SVGRectElement, BandRect>("rect")
    .data(bandRects, (d: any) => d.id);

    const yZero = y(0);
    const yTop  = y(y.domain()[1]); 

  rects.join(
    enter => enter
      .append("rect")
      .attr("fill", d => d.color)
      .attr("x", d => d.x)
      .attr("width", d => d.w)
      .attr("y", hasRenderedBands ? yTop : yZero)
      .attr("height", 0)
      .transition().duration(T)
      .attr("y", d => d.y)
      .attr("height", d => d.h),

    update => update
      .transition().duration(T)
      .attr("x", d => d.x)
      .attr("width", d => d.w)
      .attr("y", d => d.y)
      .attr("height", d => d.h)
      .attr("fill", d => d.color),

    exit => exit
        .transition().duration(T)
        .attr("y", _ => y(y.domain()[1]))
        .attr("height", 0)
        .remove()
  );

  hasRenderedBands = true;
});

//Transition for line

let lineG: SVGGElement | null = null;
let edgesG: SVGGElement | null = null;
let prevEdgeXs: number[] = [];
let prevPathD = "";

$effect(() => {
  if (!lineG) return;

  const hasLine = !!pathD;
  const sel = d3
    .select(lineG)
    .selectAll<SVGPathElement, string>("path")
    .data(hasLine ? [pathD] : [], () => "mean-line");

  sel.join(
    enter => enter
        .append("path")
        .attr("fill", "none")
        .attr("stroke", "black")
        .attr("stroke-width", 1.5)
        .attr("d", d => (prevPathD && prevPathD.length ? prevPathD : d))
        .transition()
        .duration(T)
        .attrTween("d", function (d) {
          const from = this.getAttribute("d") || d;
          if (!from || from.length === 0) return () => d;
          const interp = d3.interpolateString(from, d);
          return t => interp(t);
        }),
    update => update
        .transition()
        .duration(T)
        .attrTween("d", function (d) {
          const from = this.getAttribute("d") || d;
          if (!from || from.length === 0) return () => d;
          const interp = d3.interpolateString(from, d);
          return t => interp(t);
        }),
    exit => exit
        .transition()
        .duration(T)
        .style("opacity", 0)
        .remove()
  );

  if (pathD && pathD.length) prevPathD = pathD;
});


$effect(() => {
  if (!edgesG) return;

  const data = edgeXs.map((ex, i) => ({ id: i ? "end" : "start", x: ex }));
  const prevMap = new Map(
    (prevEdgeXs || []).map((px, i) => [i ? "end" : "start", px])
  );

  const sel = d3
    .select(edgesG)
    .selectAll<SVGLineElement, { id: string; x: number }>("line")
    .data(data, d => d.id);

  sel.join(
    enter =>
      enter
        .append("line")
        .attr("stroke", "black")
        .attr("stroke-dasharray", "4,4")
        .attr("opacity", 0.6)
        .attr("x1", d => prevMap.get(d.id) ?? d.x)
        .attr("x2", d => prevMap.get(d.id) ?? d.x)
        .attr("y1", M.top)
        .attr("y2", H - M.bottom)
        .transition()
        .duration(T)
        .attr("x1", d => d.x)
        .attr("x2", d => d.x),
    update =>
      update
        .transition()
        .duration(T)
        .attr("x1", d => d.x)
        .attr("x2", d => d.x)
        .attr("y1", M.top)
        .attr("y2", H - M.bottom),
    exit =>
      exit
        .transition()
        .duration(T)
        .attr("y2", M.top)
        .style("opacity", 0)
        .remove()
  );

  prevEdgeXs = edgeXs.slice();
});



</script>


<div class="container">

    <select bind:value={selectedDataset} class="areaSelect">
      {#each stations as s}
          <option value={s.key}>{s.name}{s.count ? ` (${s.count})` : ''}</option>
      {/each}
    </select>


    <label class="showRaw">
        <span>show raw data</span>
        <input type="checkbox" bind:checked={showRaw}/>
    </label>
    <p class="dataRecords">Number of records: {data.length}</p>


<svg width={W} height={H} viewBox={`0 0 ${W} ${H}`} style="--t:${T}ms; max-width:100%; height:auto; overflow:visible">

    <g class="bands" opacity="0.8" bind:this={bandsG}></g>

    <g class="p1090">
        {#if bandPath}
            <path d={bandPath} fill="black" opacity="0.20" />
    {/if}</g>

    {#if showRaw}
        <g class="raw" fill="black" opacity="0.9">
            {#each rawPts as p}
                <circle cx={p.cx} cy={p.cy} r="1.8" />
    {/each}</g>{/if}


    <g class="line" bind:this={lineG}></g>

    <g class="edges" bind:this={edgesG}></g>


    <g class="x-axis" bind:this={gx} transform={`translate(0,${H - M.bottom})`}></g>
    <g class="y-axis" bind:this={gy} transform={`translate(${M.left},0)`}></g>


</svg>

</div>

<style>
    svg {
	    max-width: 100%;
        height: auto;
    }

    .container{
        display: flex;
        flex-direction: column;
        --h:18px;
        grid-template-columns: auto auto 1fr;
        gap: .5rem;
        align-items: stretch;
        font-size: 12px;
    }

    .areaSelect{
        margin: 0;
        width: 220px;
        grid-column: 1 / -1;
        height:var(--h);
        font: inherit;
        line-height: var(--h);
    }
    .showRaw{
        display:flex;
        align-items:center;
        gap:.5rem;
        grid-column: 1 / -1;
        height:var(--h);
        font: inherit;
        width:100%;
        margin:0;
    }
    .showRaw input{
        width: 1rem;
        height: 1rem;
    }
    .dataRecords{
        margin: 0;
        grid-column: 1 / -1;
        height:var(--h);
        font: inherit;
    }

    .bands rect,.p1090 path,.raw circle,.edges line {
        transition:
        x var(--t) ease, y var(--t) ease,
        width var(--t) ease, height var(--t) ease,
        cx var(--t) ease, cy var(--t) ease, r var(--t) ease,
        opacity var(--t) ease, transform var(--t) ease;
    }
</style>