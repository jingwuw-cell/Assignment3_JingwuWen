
<script lang="ts">
  import * as d3 from 'd3'
  import { line as d3line, curveLinear } from 'd3-shape';
  let { selectedDataset = $bindable('avalon'), data = [], stations = [] } = $props();


  const W = 900, H = Math.round(W * 9/16); //make H an integer
  const M = { top: 20, right: 30, bottom: 30, left: 40 };
  const y_scale=1.1;
  const x = d3.scaleUtc().range([M.left, W - M.right]);
  const y = d3.scaleLinear().range([H - M.bottom, M.top]);


  // find the monthly mean
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

  //find the p10-p90 band
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

  //aqi level band
  const BANDS = [
    { name: "Good", min: 0,   max: 50,  color: "#9cd84e" },
    { name: "Moderate", min: 50,  max: 100, color: "#facf39" },
    { name: "USG", min: 100, max: 150, color: "#f99049" },
    { name: "Unhealthy", min: 150, max: 200, color: "#f65e5f" },
    { name: "Very Unhealthy", min: 200, max: 300, color: "#a070b6" },
    { name: "Hazardous", min: 300, max: Infinity, color: "#a06a7b" }
  ];

  type BandRect = { id:string; x:number; y:number; w:number; h:number; color:string }
  let bandRects = $state<BandRect[]>([]);

  let pathD = $state("");

  // store the row data checkbox in local storage
  let RawBottonStorage = globalThis.localStorage ?? {};
  let showRaw = $state((RawBottonStorage.showRaw ?? 'false') === 'true');
  $effect(() => { RawBottonStorage.showRaw = String(showRaw); });


  let rawPts = $state<Array<{cx:number; cy:number}>>([]);  //store the raw data points

  $effect(() => {
    if (!mean_data.length) { pathD = ""; return; }

    const [xStart, xEnd] = d3.extent(mean_data, d => d.time) as [Date, Date];
    const xMin = d3.utcDay.offset(xStart, -15);
    const xMax = d3.utcDay.offset(xEnd,   +15);
    x.domain([xMin, xMax]);
    const ymaxRaw = (d3.max(data as Array<{aqi:number}>, d => d.aqi) ?? 0) * y_scale || 1;
    y.domain([0, ymaxRaw]);

    //draw the line
    pathD = d3.line<{ time: Date; aqi: number }>()
        .x(d => x(d.time))
        .y(d => y(d.aqi))(mean_data) ?? "";

    const [yMin, yMax] = y.domain();
    const clamp = (v:number) => Math.max(yMin, Math.min(yMax, v));
    const plotX = M.left;
    const plotW = W - M.left - M.right;

    //draw the color bands for us aqi
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
        .y1(d => y(d.p90))(band_data) ?? "" : "";

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
    const xAxis = d3.axisBottom(x).ticks(W / 80).tickSizeOuter(0);
    const g = d3.select(gx);
    g.transition().duration(T).call(xAxis);
    g.selectAll<SVGTextElement, unknown>("text.x-label")
      .data([null])
      .join("text")
        .attr("class", "x-label")
        .attr("text-anchor", "middle")
        .attr("fill", "currentColor")
        .attr("x", (W - M.left - M.right) / 2 + M.left)
        .attr("y", M.bottom) 
        .text("Time");
  });

  $effect(() => {
    if (!gy || !mean_data.length) return;
    const yAxis = d3.axisLeft(y).ticks(H / 40);
    const g = d3.select(gy);

    g.transition().duration(T).call(yAxis);

    g.selectAll<SVGTextElement, unknown>("text.y-label")
      .data([null])
      .join("text")
        .attr("class", "y-label")
        .attr("text-anchor", "middle")
        .attr("fill", "currentColor")
        .attr("x", -6)
        .attr("y", M.top - 8)
        .text("US AQI");
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

  //Transition for line and edges

  let lineG: SVGGElement | null = null;
  let edgesG: SVGGElement | null = null;
  let prevEdgeXs: number[] = []; // edges for previous data
  let prevPathD = ""; // path for previous data
  let prevXDomain: [Date, Date] | null = null;
  let prevYDomain: [number, number] | null = null;

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
          const curX = x.domain();
          const curY = y.domain();
          const sameX = !!prevXDomain && (+prevXDomain[0] === +curX[0]) && (+prevXDomain[1] === +curX[1]);
          const sameY = !!prevYDomain && (prevYDomain[0] === curY[0]) && (prevYDomain[1] === curY[1]);
          const domainsSame = sameX && sameY;

          if(domainsSame){
            const from = this.getAttribute("d") || prevPathD || d;
            const interp = d3.interpolateString(from, d);
            return (t: number) => interp(t);
          }

          const oldX = prevXDomain ?? x.domain();
          const oldY = prevYDomain ?? y.domain();

          const ix0 = d3.interpolateNumber(+oldX[0], +curX[0]);
          const ix1 = d3.interpolateNumber(+oldX[1], +curX[1]);
          const iy0 = d3.interpolateNumber(oldY[0],  curY[0]);
          const iy1 = d3.interpolateNumber(oldY[1],  curY[1]);

          const xr = x.range();
          const yr = y.range();

          return (t: number) => {
            const xt = d3.scaleUtc().domain([new Date(ix0(t)), new Date(ix1(t))]).range(xr);
            const yt = d3.scaleLinear().domain([iy0(t), iy1(t)]).range(yr);

            return d3.line<{ time: Date; aqi: number }>()
              .x(p => xt(p.time))
              .y(p => yt(p.aqi))(mean_data) ?? "";
          };
        }),
      update => update
        .transition()
        .duration(T)
        .attrTween("d", function (d) {
          const curX = x.domain();
          const curY = y.domain();
          const sameX = !!prevXDomain && (+prevXDomain[0] === +curX[0]) && (+prevXDomain[1] === +curX[1]);
          const sameY = !!prevYDomain && (prevYDomain[0] === curY[0]) && (prevYDomain[1] === curY[1]);
          const domainsSame = sameX && sameY;

          if(domainsSame){
            const from = this.getAttribute("d") || prevPathD || d;
            const interp = d3.interpolateString(from, d);
            return (t: number) => interp(t);
          }
          

          const ix0 = d3.interpolateNumber(+oldX[0], +curX[0]);
          const ix1 = d3.interpolateNumber(+oldX[1], +curX[1]);
          const iy0 = d3.interpolateNumber(oldY[0],  curY[0]);
          const iy1 = d3.interpolateNumber(oldY[1],  curY[1]);

          const xr = x.range();
          const yr = y.range();

          return (t: number) => {
            const xt = d3.scaleUtc().domain([new Date(ix0(t)), new Date(ix1(t))]).range(xr);
            const yt = d3.scaleLinear().domain([iy0(t), iy1(t)]).range(yr);

            return d3.line<{ time: Date; aqi: number }>()
              .x(p => xt(p.time))
              .y(p => yt(p.aqi))(mean_data) ?? "";
          };
        }),
      exit => exit
        .transition()
        .duration(T)
        .style("opacity", 0)
        .remove()
      );

    if (pathD && pathD.length) prevPathD = pathD;
    prevXDomain = x.domain();
    prevYDomain = y.domain();

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
      enter => enter
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
      update => update
        .transition()
        .duration(T)
        .attr("x1", d => d.x)
        .attr("x2", d => d.x)
        .attr("y1", M.top)
        .attr("y2", H - M.bottom),
      exit => exit
        .transition()
        .duration(T)
        .attr("y2", M.top)
        .style("opacity", 0)
        .remove()
    );

    prevEdgeXs = edgeXs.slice();
  });

  //tooltip for month and its mean aqi
  let svgEl: SVGSVGElement | null = null;

  function installTooltip(svgNode: SVGSVGElement) {
    const svg = d3.select(svgNode)
      .on("pointerenter pointermove", pointermoved)
      .on("pointerleave", pointerleft)
      .on("touchstart", (event: any) => event.preventDefault());

    const tooltip = svg.append("g");
    let placeAbove = true;

    function formatValue(aqi: number){
      return Number(aqi).toLocaleString("en-US", { maximumFractionDigits: 0 });
    }

    function formatDate(time: Date){
      return time.toLocaleString("en-US",{
        month:"short",
        year:"numeric",
       timeZone:"UTC"
      });
    } 

    const bisect = d3.bisector((d: any) => d.time).center;

    function pointermoved(event: PointerEvent){
      const i = bisect(mean_data, x.invert(d3.pointer(event)[0]));
      const d = mean_data[i];
      if (!d) return;

      const L = mean_data[i - 1], R = mean_data[i + 1];
      const sL = L ? (d.aqi - L.aqi) / (+d.time - +L.time) : 0;
      const sR = R ? (R.aqi - d.aqi) / (+R.time - +d.time) : 0;
      placeAbove = sL > sR;

      tooltip.style("display", null);
      tooltip.attr("transform", `translate(${x(d.time)},${y(d.aqi)})`);

      const path = tooltip.selectAll("path")
        .data([,])
        .join("path")
        .attr("fill","white")
        .attr("stroke","black")
        .attr("opacity",0.9);

      const text = tooltip.selectAll("text")
        .data([,])
        .join("text")
        .call(text => text
          .selectAll("tspan")
          .data([formatDate(d.time), formatValue(d.aqi)])
          .join("tspan")
            .attr("x",0)
            .attr("y", (_: any, i: number) => `${i * 1.1}em`)
            .attr("font-weight", (_: any, i: number) => i ? null : "bold")
            .text((s: string) => s)
            .text((s: string, i: number) => i ? `AQI: ${s}` : s)
        );

      size(text as any, path as any, placeAbove);
    }

    function pointerleft(){
      tooltip.style("display","none");
    }

    function size(
        text: d3.Selection<SVGTextElement, unknown, null, undefined>, 
        path: d3.Selection<SVGPathElement, unknown, null, undefined>,
        above: boolean) {
      const bbox = (text.node() as SVGGraphicsElement).getBBox();
      const w = bbox.width, h = bbox.height, y0 = bbox.y;
      if (!above) {
        text.attr("transform", `translate(${-w / 2},${15 - y0})`);
        path.attr("d", `M${-w / 2 - 10},5H-5l5,-5l5,5H${w / 2 + 10}v${h + 20}h-${w + 20}z`);
      } 
      else {
        text.attr("transform", `translate(${-w / 2},${-(h + 15) - y0})`);
        path.attr("d", `M${-w / 2 - 10},-5H-5l5,5l5,-5H${w / 2 + 10}v-${h + 20}h-${w + 20}z`);
      }
    }
  }

  let tooltipReady = $state(false);

  $effect(() => {
    if (!svgEl || !mean_data.length || tooltipReady === true) return;
    installTooltip(svgEl);
    tooltipReady = true;
  });

  //transition for p10-p90 band

  let p10p90bandG: SVGGElement | null = null;
  let prevBandPath = "";

  $effect(() => {
    if (!p10p90bandG) return;

    const hasBand = !!bandPath;
    const sel = d3
      .select(p10p90bandG)
      .selectAll<SVGPathElement, string>("path")
      .data(hasBand ? [bandPath] : [], () => "p10p90-band");

    sel.join(
      enter => enter
        .append("path")
        .attr("fill", "black")
        .attr("opacity", 0.20)
        .attr("d", d => (prevBandPath && prevBandPath.length ? prevBandPath : d))
        .transition().duration(T)
        .attrTween("d", function (d) {
          const from = this.getAttribute("d") || d;
          const interp = d3.interpolateString(from, d);
          return t => interp(t);
        }),

      update => update
        .transition().duration(T)
        .attrTween("d", function (d) {
          const from = this.getAttribute("d") || d;
          const interp = d3.interpolateString(from, d);
          return t => interp(t);
        }),

      exit => exit
        .transition().duration(T)
        .style("opacity", 0)
        .remove()
    );

    if (bandPath && bandPath.length) prevBandPath = bandPath;
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


<svg bind:this={svgEl} 
  width={W} height={H} viewBox={`0 0 ${W} ${H}`} 
  style="--t:${T}ms; max-width:100%; height:auto; overflow:visible">

    <g class="bands" opacity="0.8" bind:this={bandsG}></g>

    <g class="p1090" bind:this={p10p90bandG}></g>

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
        gap: .1rem;
        align-items: stretch;
        font-size: 11px;
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

  .x-axis .x-label, .y-axis .y-label {
    font-size: 12px;
    font-weight: 600;
    fill: black;
  }

</style>