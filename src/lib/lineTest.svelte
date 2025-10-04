
<script lang="ts">
    import * as d3 from 'd3'
    import { line as d3line, curveLinear } from 'd3-shape';

    let { selectedDataset = $bindable('avalon'), data=[]}= $props();


function toMonthly(rows: { time: Date; aqi: number }[]) {
  const map = new Map<string, { y: number; m: number; sum: number; n: number }>();
  for (const r of rows) {
    if (!(r.time instanceof Date) || Number.isNaN(+r.time) || !Number.isFinite(r.aqi)) continue;
    const y = r.time.getUTCFullYear(), m = r.time.getUTCMonth();
    const k = `${y}-${m}`;
    const a = map.get(k) || { y, m, sum: 0, n: 0 };
    a.sum += r.aqi; a.n += 1;
    map.set(k, a);
  }
  return [...map.values()]
    .map(a => ({ time: new Date(Date.UTC(a.y, a.m, 15)), aqi: a.sum / a.n }))
    .sort((a, b) => +a.time - +b.time);
}

function toMonthlyStats(rows: { time: Date; aqi: number }[]) {
    const g = d3.group(rows, d => d.time.getUTCFullYear(), d => d.time.getUTCMonth());
    const out: Array<{ time: Date; aqi: number; p10: number; p90: number }> = [];
    for (const [y, mMap] of g) {
        for (const [m, arr] of mMap) {
            const vals = arr.map(d => d.aqi).sort(d3.ascending);
            out.push({
                time: new Date(Date.UTC(+y, +m, 15)),
                aqi: d3.mean(vals)!,
                p10: d3.quantile(vals, 0.10)!,
                p90: d3.quantile(vals, 0.90)!
            });
        }
    }
    return out.sort((a, b) => +a.time - +b.time);
}

function chart(data, raw, opts = {}){

    const width = 928;
    const height = width*0.618;
    const marginTop = 20;
    const marginRight = 30;
    const marginBottom = 30;
    const marginLeft = 40;

    const y_scale = 1.1;

    const ext = d3.extent(raw, d => d.time) as [Date, Date];
    const xMin = d3.utcDay.offset(d3.utcMonth.floor(ext[0]));
    const xMax = d3.utcMonth.offset(d3.utcMonth.floor(ext[1]));
    const x = d3.scaleUtc([xMin, xMax], [marginLeft, width - marginRight]);

    //const x = d3.scaleUtc(d3.extent(raw, d => d.time)  as [Date, Date], [marginLeft, width - marginRight]);
    const y = d3.scaleLinear([0, y_scale*d3.max(raw, d => d.aqi)], [height - marginBottom, marginTop]);

    const line = d3.line()
        .x(d => x(d.time))
        .y(d => y(d.aqi));

    const svg = d3.create("svg")
      .attr("width", width)
      .attr("height", height)
      .attr("viewBox", [0, 0, width, height])
      .attr("style", "max-width: 100%; height: auto; height: intrinsic; font: 10px sans-serif;")
      .style("-webkit-tap-highlight-color", "transparent")
      .style("overflow", "visible")
      .on("pointerenter pointermove", pointermoved)
      .on("pointerleave", pointerleft)
      .on("touchstart", event => event.preventDefault());

    const BANDS = [
        {name: "Good", min: 0, max: 50, color: "#9cd84e"},
        {name: "Moderate", min: 50, max: 100, color: "#facf39"},
        {name: "Unhealthy for Sensitive Groups", min: 100, max: 150, color: "#f99049"},
        {name: "Unhealthy", min: 150, max: 200, color: "#f65e5f" },
        {name: "Very Unhealthy", min: 200, max: 300, color: "#a070b6"},
        {name: "Hazardous", min: 300, max: Infinity, color: "#a06a7b"}
    ];

    const plotX = marginLeft;
    const plotW = width - marginLeft - marginRight;

    const [yMinDomain, yMaxDomain] = y.domain();
    const clamp = (v: number, lo: number, hi: number) => Math.max(lo, Math.min(hi, v));

    for (const b of BANDS) {
        const lo = clamp(b.min, yMinDomain, yMaxDomain);
        const hi = clamp(b.max, yMinDomain, yMaxDomain);
        if (hi <= yMinDomain || lo >= yMaxDomain || lo >= hi) continue;

    const yTop = y(hi);
    const yBot = y(lo);

    svg.append("rect")
        .attr("x", plotX)
        .attr("y", yTop)
        .attr("width", plotW)
        .attr("height", yBot - yTop)
        .attr("fill", b.color)
        .attr("opacity", 0.80);
    }

const xFirst = x(data[0].time);
const xLast  = x(data[data.length - 1].time);
const yTop   = marginTop;
const yBot   = height - marginBottom;

svg.append("line")
  .attr("x1", xFirst).attr("x2", xFirst)
  .attr("y1", yTop).attr("y2", yBot)
  .attr("stroke", "black")
  .attr("stroke-dasharray", "4,4")
  .attr("opacity", 0.6);

svg.append("line")
  .attr("x1", xLast).attr("x2", xLast)
  .attr("y1", yTop).attr("y2", yBot)
  .attr("stroke", "black")
  .attr("stroke-dasharray", "4,4")
  .attr("opacity", 0.6);

if (opts.band && opts.band.length) {
  const area = d3.area()
    .x((d: any) => x(d.time))
    .y0((d: any) => y(d.p10))
    .y1((d: any) => y(d.p90));

  svg.append("path")
    .attr("fill", "black")
    .attr("opacity", 0.2)
    .attr("d", area(opts.band));
}

    svg.append("g")
      .attr("transform", `translate(0,${height - marginBottom})`)
      .call(d3.axisBottom(x).ticks(width / 80).tickSizeOuter(0));

    svg.append("g")
      .attr("transform", `translate(${marginLeft},0)`)
      .call(d3.axisLeft(y).ticks(height / 40))
      .call(g => g.select(".domain").remove())
      .call(g => g.selectAll(".tick line").clone()
          .attr("x2", width - marginLeft - marginRight)
          .attr("stroke-opacity", 0.1))
      .call(g => g.append("text")
          .attr("x", -marginLeft)
          .attr("y", 10)
          .attr("fill", "currentColor")
          .attr("text-anchor", "start")
          .text("↑ AQI"));

    if (opts.raw && opts.raw.length) {
        svg.append("g")
            .attr("fill", "black")
            .attr("opacity", 0.8)
            .selectAll("circle")
            .data(opts.raw)
            .join("circle")
            .attr("cx", d => x(d.time))
            .attr("cy", d => y(d.aqi))
            .attr("r", 1.5);
    }

    svg.append("path")
      .attr("fill", "none")
      .attr("stroke", "black")
      .attr("stroke-width", 1.5)
      .attr("d", line(data));

      

    const tooltip = svg.append("g");

    function formatValue(aqi){
        return Number(aqi).toLocaleString("en-US", { maximumFractionDigits: 0 });
    }

    function formatDate(time){
        return time.toLocaleString("en-US",{
            month:"short",
            year:"numeric",
            timeZone:"UTC"});
    }

    const bisect=d3.bisector(d => d.time).center;

    function pointermoved(event){
        const i=bisect(data, x.invert(d3.pointer(event)[0]));
        tooltip.style("display", null);
        tooltip.attr("transform", `translate(${x(data[i].time)},${y(data[i].aqi)})`);
        
        const path = tooltip.selectAll("path")
            .data([,])
            .join("path")
                .attr("fill","white")
                .attr("stroke","black");

        const text = tooltip.selectAll("text")
            .data([,])
            .join("text")
            .call(text => text
                .selectAll("tspan")
                .data([formatDate(data[i].time),formatValue(data[i].aqi)])
                .join("tspan")
                    .attr("x",0)
                    .attr("y", (_,i) => `${i *1.1}em`)
                    .attr("font-weight",(_,i) => i ? null : "bold")
                    .text(d =>d));
        size(text, path);
    }

    function pointerleft(){
        tooltip.style("display","none");
    }

    function size(text, path) {
    const {x, y, width: w, height: h} = text.node().getBBox();
    text.attr("transform", `translate(${-w / 2},${15 - y})`);
    path.attr("d", `M${-w / 2 - 10},5H-5l5,-5l5,5H${w / 2 + 10}v${h + 20}h-${w + 20}z`);
  }

    return svg.node();
}

let mount: HTMLDivElement | null = null;
let lineStorage = globalThis.localStorage ?? {};
let showRaw = $state(lineStorage.showRaw ?? false)
$effect(() => {lineStorage.showRaw = showRaw;});	

$effect(() => {
  if (!mount) return;
  const monthly = toMonthly(data);
  const band = toMonthlyStats(data);
  mount.replaceChildren();
  if (!monthly.length) return;
  mount.appendChild(chart(monthly, data, { raw: showRaw ? data : null, band  }));
});


</script>

<div class="container">
    <select bind:value={selectedDataset} class="areaSelect">
        <option value="avalon">avalon</option>
        <option value="glassport_high_street">glassport_high_street</option>
        <option value="lawrenceville">lawrenceville</option>
        <option value="liberty_sahs">liberty_sahs</option>
        <option value="manchester">manchester</option>
        <option value="north_braddock">north_braddock</option>
        <option value="parkway_east_near_road">parkway_east_near_road</option>
        <option value="usa_pennsylvania_pittsburgh">usa_pennsylvania_pittsburgh</option>
    </select>
    <label class="showRaw">
        <span>show raw data</span>
        <input type="checkbox" bind:checked={showRaw} />
    </label>
    <p class="dataRecords">Number of records: {data.length}</p>
    <div id="mount" bind:this={mount}></div>
</div>

<style>
svg {
	max-width: 100%;
    height: auto;
    margin-block: 1rem 2rem;
    overflow: visible;
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

</style>