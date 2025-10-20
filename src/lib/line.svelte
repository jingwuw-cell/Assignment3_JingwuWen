<script lang="ts">
  import * as d3 from 'd3';
  let { selectedDataset = $bindable('avalon'), data = [], stations = [] } = $props();

  const W = 900, H = Math.round(W * 9/16);
  const M = { top: 20, right: 30, bottom: 30, left: 40 };
  const y_scale = 1.1;
  const x = d3.scaleUtc().range([M.left, W - M.right]);
  const y = d3.scaleLinear().range([H - M.bottom, M.top]);
  let xDomainRange = $state<[Date, Date] | null>(null);
  const yearStart = (y: number) => new Date(Date.UTC(y, 0, 1));

  function toMonthlyMean(rows: Array<{ time: any; aqi: number }>) {
    const map = new Map<string, { y: number; m: number; sum: number; n: number }>();
    for (const r of rows) {
      const t = new Date(r.time);
      const a = +r.aqi;
      if (Number.isNaN(+t) || !Number.isFinite(a)) continue;
      const y0 = t.getUTCFullYear(), m0 = t.getUTCMonth();
      const k = `${y0}-${m0}`;
      const acc = map.get(k) ?? { y: y0, m: m0, sum: 0, n: 0 };
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
        const vals = arr.map((d: any) => +d.aqi).filter((v) => Number.isFinite(v)).sort(d3.ascending);
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

  function toQuarterMeans(rows: Array<{ time: any; aqi: number }>) {
    const map = new Map<string, { y: number; q: number; sum: number; n: number }>();
    for (const r of rows) {
      const t = new Date(r.time);
      const a = +r.aqi;
      if (Number.isNaN(+t) || !Number.isFinite(a)) continue;
      const y0 = t.getUTCFullYear();
      const q0 = Math.floor(t.getUTCMonth() / 3) + 1;
      const k = `${y0}-Q${q0}`;
      const acc = map.get(k) ?? { y: y0, q: q0, sum: 0, n: 0 };
      acc.sum += a; acc.n += 1;
      map.set(k, acc);
    }
    const midMonth = new Map([[1,1],[2,4],[3,7],[4,10]]);
    return Array.from(map.values())
      .map(d => ({
        year: d.y,
        q: d.q as 1|2|3|4,
        time: new Date(Date.UTC(d.y, midMonth.get(d.q)!, 15)),
        aqi: d.sum / d.n
      }))
      .sort((a, b) => +a.time - +b.time);
  }

  const mean_data = $derived(toMonthlyMean(data));
  const band_data = $derived(toMonthlyP1090(data));
  const quarter_data = $derived(toQuarterMeans(data));

  let bandPath = $state("");

  const BANDS = [
    { name: "Good", min: 0,   max: 50,  color: "#9cd84e" },
    { name: "Moderate", min: 50,  max: 100, color: "#facf39" },
    { name: "USG", min: 100, max: 150, color: "#f99049" },
    { name: "Unhealthy", min: 150, max: 200, color: "#f65e5f" },
    { name: "Very Unhealthy", min: 200, max: 300, color: "#a070b6" },
    { name: "Hazardous", min: 300, max: Infinity, color: "#a06a7b" }
  ];

  type BandRect = { id: string; x: number; y: number; w: number; h: number; color: string };
  let bandRects = $state<BandRect[]>([]);
  let pathD = $state("");

  let RawBottonStorage = globalThis.localStorage ?? {};
  let showRaw = $state((RawBottonStorage.showRaw ?? 'false') === 'true');
  $effect(() => { RawBottonStorage.showRaw = String(showRaw); });

  let rawPts = $state<Array<{ cx: number; cy: number }>>([]);

  type ChartMode = 'general' | 'annual' | 'quarter';
  let chartMode = $state<ChartMode>('general');
  let annualYear = $state<number | null>(null);
  let selectedQuarter = $state<1|2|3|4|null>(null);

  const modeLabel = $derived(
    chartMode === 'general' ? 'General'
    : chartMode === 'annual' ? `Annual (${annualYear ?? '-'})`
    : `Quarter`
  );

  function yearOf(d: Date) { return d.getUTCFullYear(); }

  $effect(() => {
    if (!mean_data.length) { pathD = ""; bandPath = ""; rawPts = []; return; }

    let xMin: Date, xMax: Date;

    if (chartMode === 'annual' && annualYear !== null) {
      const yearStart = new Date(Date.UTC(annualYear, 0, 1));
      const yearEnd   = new Date(Date.UTC(annualYear, 11, 31, 23, 59, 59, 999));
      xMin = d3.utcDay.offset(yearStart, -15);
      xMax = d3.utcDay.offset(yearEnd, +15);
    } else if (chartMode === 'quarter') {
      if (quarter_data.length) {
        const firstYear = quarter_data[0].year;
        const lastYear  = quarter_data[quarter_data.length - 1].year;
        const t0 = yearStart(firstYear);
        const t1 = yearStart(lastYear);
        xMin = d3.utcDay.offset(t0, -15);
        xMax = d3.utcDay.offset(t1, +15);
      } else {
        const [xStart, xEnd] = d3.extent(mean_data, d => d.time) as [Date, Date];
        xMin = d3.utcDay.offset(xStart, -15);
        xMax = d3.utcDay.offset(xEnd, +15);
      }
    } else {
      const [xStart, xEnd] = d3.extent(mean_data, d => d.time) as [Date, Date];
      xMin = d3.utcDay.offset(xStart, -15);
      xMax = d3.utcDay.offset(xEnd, +15);
    }

    x.domain([xMin, xMax]);
    xDomainRange = [xMin, xMax];

    const ymaxRaw = (d3.max(data as Array<{ aqi: number }>, d => d.aqi) ?? 0) * y_scale || 1;
    const ymaxQuarter = ((d3.max(quarter_data, d => d.aqi) ?? 0) * 1.3) || 1;
    const yMax1 = chartMode === 'quarter' ? ymaxQuarter : ymaxRaw;
    y.domain([0, yMax1]);

    if (chartMode === 'quarter') {
      pathD = "";
    } else {
      const md = (chartMode === 'annual' && annualYear !== null)
        ? mean_data.filter(d => yearOf(d.time) === annualYear)
        : mean_data;

      pathD = (md.length
        ? d3.line<{ time: Date; aqi: number }>().x(d => x(d.time)).y(d => y(d.aqi))(md)
        : "") ?? "";
    }

    const [yMin, yMax] = y.domain();
    const clamp = (v: number) => Math.max(yMin, Math.min(yMax, v));
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

    if (chartMode === 'quarter') {
      bandPath = "";
    } else {
      const bd = (chartMode === 'annual' && annualYear !== null)
        ? band_data.filter(d => yearOf(d.time) === annualYear)
        : band_data;
      bandPath = (bd.length
        ? d3.area<{ time: Date; p10: number; p90: number }>().x(d => x(d.time)).y0(d => y(d.p10)).y1(d => y(d.p90))(bd)
        : "") ?? "";
    }

    if (chartMode === 'quarter') {
      rawPts = [];
    } else {
      const rawV = (chartMode === 'annual' && annualYear !== null)
        ? (data as Array<{ time: Date; aqi: number }>).filter(d => new Date(d.time).getUTCFullYear() === annualYear)
        : (data as Array<{ time: Date; aqi: number }>);
      rawPts = rawV.map(d => ({ cx: x(new Date(d.time)), cy: y(d.aqi) }));
    }
  });

  let gx: SVGGElement | null = null, gy: SVGGElement | null = null;
  let edgeXs = $state<number[]>([]);

  $effect(() => {
    if (chartMode === 'quarter') { edgeXs = []; return; }
    const md = (chartMode === 'annual' && annualYear !== null)
      ? mean_data.filter(d => yearOf(d.time) === annualYear)
      : mean_data;
    if (!md.length) { edgeXs = []; return; }
    const firstX = x(md[0].time);
    const lastX = x(md[md.length - 1].time);
    edgeXs = firstX === lastX ? [firstX] : [firstX, lastX];
  });

  const T = 800;

  $effect(() => {
    if (!gx || !xDomainRange) return;
    const xAxis =
      chartMode === 'annual'
        ? d3.axisBottom(x).ticks(d3.utcMonth.every(1)).tickFormat(d3.utcFormat('%b')).tickSizeOuter(0)
        : chartMode === 'quarter'
          ? d3.axisBottom(x).ticks(d3.utcYear.every(1)).tickFormat(d3.utcFormat('%Y')).tickSizeOuter(0)
          : d3.axisBottom(x).ticks(W / 80).tickSizeOuter(0);

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
    if (!gy || !xDomainRange) return;
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

  let bandsG: SVGGElement | null = null;
  let hasRenderedBands = $state(false);

  $effect(() => {
    if (!bandsG) return;
    const rects = d3.select(bandsG)
      .selectAll<SVGRectElement, BandRect>("rect")
      .data(bandRects, (d: any) => d.id);
    const yZero = y(0);
    const yTop = y(y.domain()[1]);
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

  let lineG: SVGGElement | null = null;
  $effect(() => {
    if (!lineG) return;
    if (chartMode === 'quarter') {
      d3.select(lineG).selectAll("*").remove();
      return;
    }
    const hasLine = !!pathD;
    const sel = d3.select(lineG)
      .selectAll<SVGPathElement, {id:string; d:string; stroke:string; opacity:number}>("path.mean")
      .data(hasLine ? [{id:"mean", d: pathD, stroke: "black", opacity: 1}] : [], d => d.id);

    sel.join(
      enter => enter
        .append("path")
        .attr("class", "mean")
        .attr("fill", "none")
        .attr("stroke", d => d.stroke)
        .attr("stroke-width", 1.5)
        .attr("opacity", d => d.opacity)
        .attr("d", d => d.d),
      update => update
        .attr("stroke", d => d.stroke)
        .attr("opacity", d => d.opacity)
        .attr("d", d => d.d),
      exit => exit.remove()
    );
  });

  const QCOLORS: Record<1|2|3|4,string> = {
    1: '#93C5FD',
    2: '#3B82F6',
    3: '#1D4ED8',
    4: '#172554',
  };

  let quarterLineG: SVGGElement | null = null;
  $effect(() => {
    if (!quarterLineG) return;
    if (chartMode !== 'quarter') {
      d3.select(quarterLineG).selectAll("*").remove();
      return;
    }
    const grouped = d3.group(quarter_data, d => d.q);
    const qIds: Array<1|2|3|4> = [1,2,3,4];
    const lines = qIds.map(q => {
      const arr = (grouped.get(q) ?? []).slice().sort((a,b) => a.year - b.year);
      const dstr = arr.length
        ? d3.line<{time:Date; aqi:number}>()
            .x(d => x(d.time))
            .y(d => y(d.aqi))(arr.map(d => ({time:yearStart(d.year), aqi:d.aqi}))) ?? ""
        : "";
      return {
        id: `Q${q}`,
        d: dstr,
        stroke: QCOLORS[q],
        opacity: (selectedQuarter === q ? 1 : 0.5)
      };
    });

    const sel = d3.select(quarterLineG)
      .selectAll<SVGPathElement, any>("path.qline")
      .data(lines.filter(l => l.d), d => d.id);

    sel.join(
      enter => enter
        .append("path")
        .attr("class", "qline")
        .attr("fill", "none")
        .attr("stroke-width", 1.5)
        .attr("stroke", d => d.stroke)
        .attr("opacity", d => d.opacity)
        .attr("d", d => d.d),
      update => update
        .attr("stroke", d => d.stroke)
        .attr("opacity", d => d.opacity)
        .attr("d", d => d.d),
      exit => exit.remove()
    );
  });

  let starG: SVGGElement | null = null;
  $effect(() => {
    if (!starG) return;
    if (chartMode !== 'quarter') {
      d3.select(starG).selectAll("*").remove();
      return;
    }
    const stars = quarter_data.map(d => ({
      id: `${d.year}-Q${d.q}`,
      x: x(yearStart(d.year)),
      y: y(d.aqi),
      q: d.q
    }));

    const sym = d3.symbol().type(d3.symbolStar).size(20);

    const sel = d3.select(starG)
      .selectAll<SVGPathElement, any>("path.star")
      .data(stars, d => d.id);

    sel.join(
      enter => enter
        .append("path")
        .attr("class", "star")
        .attr("transform", d => `translate(${d.x},${d.y})`)
        .attr("d", sym as any)
        .attr("fill", d => QCOLORS[d.q])
        .attr("opacity", d => (selectedQuarter && selectedQuarter !== d.q ? 0.35 : 1)),
      update => update
        .attr("transform", d => `translate(${d.x},${d.y})`)
        .attr("fill", d => QCOLORS[d.q])
        .attr("opacity", d => (selectedQuarter && selectedQuarter !== d.q ? 0.35 : 1)),
      exit => exit.remove()
    );
  });

  let edgesG: SVGGElement | null = null;
  $effect (() => {
    if (!edgesG) return;
    const sel = d3.select(edgesG)
      .selectAll<SVGLineElement, { id: string; x: number }>("line")
      .data(chartMode === 'quarter' ? [] : edgeXs.map((ex, i) => ({ id: i ? "end" : "start", x: ex })), d => d.id);

    sel.join(
      enter => enter
        .append("line")
        .attr("stroke", "black")
        .attr("stroke-dasharray", "4,4")
        .attr("opacity", 0.6)
        .attr("x1", d => d.x)
        .attr("x2", d => d.x)
        .attr("y1", M.top)
        .attr("y2", H - M.bottom),
      update => update
        .attr("x1", d => d.x)
        .attr("x2", d => d.x)
        .attr("y1", M.top)
        .attr("y2", H - M.bottom),
      exit => exit.remove()
    );
  });

  let svgEl: SVGSVGElement | null = null;

  let quartersG: SVGGElement | null = null;
  type QRect = { id: string; q: number; x: number; w: number; y: number; h: number; year: number };
  let qRects = $state<QRect[]>([]);
  let hoveredKey = $state<string | null>(null);
  let quarterInfo = $state("Quarter: -");

  $effect(() => {
    if (!mean_data.length || !xDomainRange) { qRects = []; return; }
    const [d0, d1] = xDomainRange;
    const start = new Date(Date.UTC(d0.getUTCFullYear(), Math.floor(d0.getUTCMonth() / 3) * 3, 1));
    const end = new Date(Date.UTC(d1.getUTCFullYear(), Math.floor(d1.getUTCMonth() / 3) * 3 + 3, 1));
    const plotL = M.left, plotR = W - M.right;
    const out: QRect[] = [];
    for (let t = new Date(start); +t < +end; t = new Date(Date.UTC(t.getUTCFullYear(), t.getUTCMonth() + 3, 1))) {
      const yyy = t.getUTCFullYear();
      const mm = t.getUTCMonth();
      const q = Math.floor(mm / 3) + 1;
      const t0 = new Date(Date.UTC(yyy, mm, 1));
      const t1 = new Date(Date.UTC(yyy, mm + 3, 0, 23, 59, 59, 999));
      const x0 = x(t0), x1 = x(t1);
      const xa = Math.min(x0, x1), xb = Math.max(x0, x1);
      const x0c = Math.max(plotL, Math.min(plotR, xa));
      const x1c = Math.max(plotL, Math.min(plotR, xb));
      if (x1c <= x0c) continue;
      out.push({ id: `${yyy}-Q${q}`, q, year: yyy, x: x0c, w:  x1c - x0c, y: M.top, h: H - M.top - M.bottom });
    }
    qRects = (chartMode === 'annual' && annualYear !== null) ? out.filter(r => r.year === annualYear)
         : (chartMode === 'quarter' ? [] : out);
  });

  let tooltip: d3.Selection<SVGGElement, unknown, null, undefined> | null = null;

  function installTooltip(svgNode: SVGSVGElement) {
    const svg = d3.select(svgNode)
      .on("pointerenter pointermove", pointermoved)
      .on("pointerleave", pointerleft)
      .on("touchstart", (event: any) => event.preventDefault())
      .on("wheel", (event: WheelEvent) => {
        event.preventDefault();
        if (!mean_data.length) return;
        if (chartMode === 'quarter') {
          if (event.deltaY > 0) { chartMode = 'general'; selectedQuarter = null; annualYear = null; }
          return;
        }
        if (event.deltaY < 0) {
          const px = d3.pointer(event)[0];
          const dt = x.invert(px);
          const ySel = dt.getUTCFullYear();
          chartMode = 'annual';
          annualYear = ySel;
        } else if (event.deltaY > 0) {
          chartMode = 'general';
          annualYear = null;
        }
      });

    tooltip = svg.append("g");
    let placeAbove = true;

    function formatValue(aqi: number) {
      return Number(aqi).toLocaleString("en-US", { maximumFractionDigits: 0 });
    }
    function formatDate(time: Date) {
      return time.toLocaleString("en-US", { month: "short", year: "numeric", timeZone: "UTC" });
    }
    const bisect = d3.bisector((d: any) => d.time).center;

    function pointermoved(event: PointerEvent) {
      if (chartMode === 'quarter') {
        tooltip?.style("display", "none");
        return;
      }
      const px = d3.pointer(event)[0];
      const dt = x.invert(px);
      const q = Math.floor(dt.getUTCMonth() / 3) + 1;
      const yyy = dt.getUTCFullYear();
      hoveredKey = `${yyy}-Q${q}`;
      quarterInfo = `Quarter: ${yyy} Q${q}`;

      const md = (chartMode === 'annual' && annualYear !== null)
        ? mean_data.filter(d => yearOf(d.time) === annualYear)
        : mean_data;
      if (!md.length) return;

      const i = bisect(md, x.invert(px));
      const d = md[i];
      if (!d) return;

      const L = md[i - 1], R = md[i + 1];
      const sL = L ? (d.aqi - L.aqi) / (+d.time - +L.time) : 0;
      const sR = R ? (R.aqi - d.aqi) / (+R.time - +d.time) : 0;
      placeAbove = sL > sR;

      tooltip.style("display", null);
      tooltip.attr("transform", `translate(${x(d.time)},${y(d.aqi)})`);

      const path = tooltip.selectAll("path.tip")
        .data([,])
        .join("path")
        .attr("class", "tip")
        .attr("fill", "white")
        .attr("stroke", "black")
        .attr("opacity", 0.7);

      const text = tooltip.selectAll("text")
        .data([,])
        .join("text")
        .attr("font-size", 8)
        .call(text => text
          .selectAll("tspan")
          .data([formatDate(d.time), formatValue(d.aqi)])
          .join("tspan")
          .attr("x", 0)
          .attr("y", (_: any, i: number) => `${i * 1.1}em`)
          .attr("font-weight", (_: any, i: number) => i ? null : "bold")
          .text((s: string) => s)
          .text((s: string, i: number) => i ? `AQI: ${s}` : s)
        );

      size(text as any, path as any, placeAbove);
      
    }

    function pointerleft() {
      if (chartMode === 'quarter') return;
      hoveredKey = null;
      quarterInfo = "Quarter: -";
      tooltip.style("display", "none");
      prevHoveredKey = null;
      
    }

    function size(
      text: d3.Selection<SVGTextElement, unknown, null, undefined>,
      path: d3.Selection<SVGPathElement, unknown, null, undefined>,
      above: boolean
      ) {
      const bbox = (text.node() as SVGGraphicsElement).getBBox();
      const w = bbox.width, h = bbox.height, y0 = bbox.y;
      if (!above) {
        text.attr("transform", `translate(${-w / 2},${6 - y0})`);
        path.attr("d", `M${-w / 2 - 1},5H-5l5,-5l5,5H${w / 2 + 1}v${h + 2}h-${w + 2}z`);
      } else {
        text.attr("transform", `translate(${-w / 2},${-(h + 6) - y0})`);
        path.attr("d", `M${-w / 2 - 1},-5H-5l5,5l5,-5H${w / 2 + 1}v-${h + 2}h-${w + 2}z`);
      }
    }
  }
  
  $effect(() => {
    if (chartMode === 'quarter') {
      tooltip?.style("display", "none");
    }
  });

  let tooltipReady = $state(false);
  $effect(() => {
    if (!svgEl || tooltipReady === true) return;
    installTooltip(svgEl);
    tooltipReady = true;
  });

  let p10p90bandG: SVGGElement | null = null;
  $effect(() => {
    if (!p10p90bandG) return;
    const hasBand = !!bandPath;
    const sel = d3.select(p10p90bandG)
      .selectAll<SVGPathElement, string>("path")
      .data(hasBand ? [bandPath] : [], () => "p10p90-band");
    sel.join(
      enter => enter
        .append("path")
        .attr("fill", "black")
        .attr("opacity", 0.20)
        .attr("d", d => d),
      update => update.attr("d", d => d),
      exit => exit.remove()
    );
  });


  $effect(() => {
    if (!quartersG) return;

    const sel = d3.select(quartersG)
      .selectAll<SVGRectElement, QRect>("rect")
      .data(qRects, d => d.id);

    let hoverYear: number | null = null;
    let hoverQuarter: number | null = null;
    if (hoveredKey) {
      const m = /^(\d{4})-Q([1-4])$/.exec(hoveredKey);
      if (m) {
        hoverYear = +m[1];
        hoverQuarter = +m[2];
      }
    }

    function rectOpacity(r: QRect) {
      if (chartMode === 'quarter') return 0;
      if (hoverYear == null || hoverQuarter == null) return 0;
      if (r.year === hoverYear && r.id === hoveredKey) return 0.0;
      if (r.year === hoverYear) return 0.2;
      return 0.4;
    }

    const t = d3.transition("quarterOpacity").duration(300).ease(d3.easeCubicOut);

    sel.join(
      enter => enter.append("rect")
        .attr("fill","black")
        .attr("x", d => d.x)
        .attr("y", d => d.y)
        .attr("width", d => d.w)
        .attr("height", d => d.h)
        .style("opacity", 0) 
        .on("click", (_e, d) => {
          if (chartMode !== 'general') return;
          chartMode = 'quarter';
          selectedQuarter = d.q as 1|2|3|4;
        })
        .transition(t).style("opacity", d => rectOpacity(d)),
      update => update
          .on("click", (_e, d) => {
          if (chartMode !== 'general') return;
          chartMode = 'quarter';
          selectedQuarter = d.q as 1|2|3|4;
        })
        .transition(t)
        .attr("x", d => d.x)
        .attr("y", d => d.y)
        .attr("width", d => d.w)
        .attr("height", d => d.h)
        .style("opacity", d => rectOpacity(d)),
      exit => exit.transition(t).style("opacity", 0).remove()
    );
  });

  let legendQuarterG: SVGGElement | null = null;

  $effect(() => {
    if (!legendQuarterG) return;
    const g = d3.select(legendQuarterG);
    g.selectAll("*").remove();
    if (chartMode !== 'quarter') return;

    const boxW = 150;
    const boxH = Math.round(boxW * 0.618);
    const pad  = 10;
    const rowH = 18;

    const x0 = M.left + 8;
    const y0 = H - M.bottom - boxH - 8;

    g.append("rect")
      .attr("x", x0)
      .attr("y", y0)
      .attr("rx", 6).attr("ry", 6)
      .attr("width", boxW)
      .attr("height", boxH)
      .attr("fill", "white")
      .attr("opacity", 0.5)
      .attr("stroke", "rgba(0,0,0,.25)");


    g.append("text")
      .attr("x", x0 + boxW/2 - 38)
      .attr("y", y0 + pad + 6)
      .attr("font-size", 11)
      .attr("font-weight", 600)
      .attr("fill", "#111")
      .text("Quarter Legend");

    const yStart = y0 + pad + 18;

    const items = ([
      { q: 1, label: "Quarter 1" },
      { q: 2, label: "Quarter 2" },
      { q: 3, label: "Quarter 3" },
      { q: 4, label: "Quarter 4" },
    ] as const).map((it, i) => ({
      ...it,
      cx: x0 + pad + 40,
      x1: x0 + pad,
      x2: x0 + pad + 80,
      cy: yStart + i * rowH,
      tx: x0 + pad + 90,
      ty: yStart + i * rowH + 3,
    }));

    g.selectAll("line.legend-line-outline")
      .data(items)
      .enter()
      .append("line")
      .attr("class", "legend-line-outline")
      .attr("x1", d => d.x1).attr("x2", d => d.x2)
      .attr("y1", d => d.cy).attr("y2", d => d.cy)
      .attr("stroke", "white")
      .attr("stroke-width", 4)
      .attr("opacity", 0.8);

    g.selectAll("line.legend-line")
      .data(items)
      .enter()
      .append("line")
      .attr("class", "legend-line")
      .attr("x1", d => d.x1).attr("x2", d => d.x2)
      .attr("y1", d => d.cy).attr("y2", d => d.cy)
      .attr("stroke", d => QCOLORS[d.q])
      .attr("stroke-width", 2)
      .attr("stroke-linecap", "round");

    const sym = d3.symbol().type(d3.symbolStar).size(35);
    g.selectAll("path.legend-star-outline")
      .data(items)
      .enter()
      .append("path")
      .attr("class", "legend-star-outline")
      .attr("transform", d => `translate(${d.cx},${d.cy})`)
      .attr("d", sym as any)
      .attr("fill", "white")
      .attr("opacity", 0.9);

    g.selectAll("path.legend-star")
      .data(items)
      .enter()
      .append("path")
      .attr("class", "legend-star")
      .attr("transform", d => `translate(${d.cx},${d.cy})`)
      .attr("d", sym as any)
      .attr("fill", d => QCOLORS[d.q]);

    g.selectAll("text.legend-label")
      .data(items)
      .enter()
      .append("text")
      .attr("class", "legend-label")
      .attr("x", d => d.tx)
      .attr("y", d => d.ty)
      .attr("font-size", 11)
      .attr("fill", "#111")
      .text(d => d.label);
  });

</script>

<div class="container" data-mode={chartMode}>
  <select bind:value={selectedDataset} 
    class="areaSelect" 
    disabled={chartMode === 'annual' || chartMode === 'quarter'}
  >
    {#each stations as s}
      <option value={s.key}>{s.name}{s.count ? ` (${s.count})` : ''}</option>
    {/each}
  </select>

  <label class="showRaw">
    <span>show raw data</span>
    <input type="checkbox" bind:checked={showRaw} disabled={chartMode==='quarter'}/>
  </label>
  <p class="dataRecords">Number of records: {data.length}</p>
  <p class="modeLabel">Line chart: {modeLabel}</p>
  <p class="quarterInfo">{quarterInfo}</p>
  

  <svg bind:this={svgEl}
    width={W} height={H} viewBox={`0 0 ${W} ${H}`}
    style="--t:${T}ms; max-width:100%; height:auto; overflow:visible">

    <g class="bands" opacity="0.8" bind:this={bandsG}></g>
    <g class="p1090" bind:this={p10p90bandG}></g>
    <g class="quarters" bind:this={quartersG}></g>

    {#if showRaw && chartMode!=='quarter'}
      <g class="raw" fill="black" opacity="0.7">
        {#each rawPts as p}
          <circle cx={p.cx} cy={p.cy} r="1" />
        {/each}
      </g>
    {/if}

    <g class="line" bind:this={lineG}></g>
    <g class="quarter-lines" bind:this={quarterLineG}></g>
    <g class="stars" bind:this={starG}></g>
    <g class="edges" bind:this={edgesG}></g>

    <g class="x-axis" bind:this={gx} transform={`translate(0,${H - M.bottom})`}></g>
    <g class="y-axis" bind:this={gy} transform={`translate(${M.left},0)`}></g>

    <g class="quarter-legend" bind:this={legendQuarterG} style="pointer-events:none;"></g>
  </svg>

  <div class="legend" aria-label="US AQI legend">
    <div class="legend-label">US AQI</div>
    <div class="legend-pills">
      {#each BANDS as b}
        <div
          class="pill"
          style={`--c:${b.color}`}
          aria-label={`${b.name} ${Number.isFinite(b.max) ? `${b.min}-${b.max}` : `${b.min}+`}`}
        >
          <div class="pill-title">{b.name}</div>
          <div class="pill-range">
            {#if Number.isFinite(b.max)}
              {#if +b.min === 0}
                {b.min}-{b.max}
              {:else}
                {b.min+1}-{b.max}
              {/if}
            {:else}
              {b.min}+
            {/if}
          </div>
        </div>
      {/each}
    </div>
  </div>

  
  <h3>Actions:</h3>
  <div class="mode-hints" aria-hidden="true">
    <div class="hint hint--1">Click the blank for the Quarter View</div>
    <div class="hint hint--2">Scroll down back to the General Chart</div>
    <div class="hint hint--3">Scroll up to the Annual Chart</div>
  </div>
</div>

<style>
  svg { max-width: 100%; height: auto; }
  .container{
    display: flex;
    flex-direction: column;
    --h:18px;
    gap: .1rem;
    align-items: stretch;
    font-size: 11px;
  }

  .areaSelect:hover {
    transition:300ms;
    background: rgba(0, 0, 0, 0.1);
  }

  .areaSelect:disabled {
    transition:800ms;
    pointer-events: none;
    opacity: .6;
    cursor: not-allowed;
    background: #111111ff; 
  }

  .areaSelect{
    transition:800ms;
    margin: 0;
    width: 220px;
    grid-column: 1 / -1;
    height: var(--h);
    font: inherit;
    line-height: var(--h);
  }
  .modeLabel{
    margin: .1rem 0 .1rem;
    font-size: 12px;
    font-weight: 600;
    grid-column: 1 / -1;
  }
  .showRaw{
    display: flex;
    align-items: center;
    gap: .5rem;
    grid-column: 1 / -1;
    height: var(--h);
    font: inherit;
    width: 100%;
    margin: 0;
  }
  .showRaw input{ width: 1rem; height: 1rem; }
  .dataRecords{
    margin: 0;
    grid-column: 1 / -1;
    height: var(--h);
    font: inherit;
  }
  .bands rect, .p1090 path, .raw circle, .edges line {
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
  .legend{
    display: flex;
    align-items: center;
    gap: .75rem;
    margin: .25rem 0 .5rem;
    font-size: 12px;
    line-height: 1;
  }
  .legend-label{
    color: #333;
    font-weight: 600;
    white-space: nowrap;
    margin-right: .25rem;
  }
  .legend-pills{
    display: flex;
    flex-wrap: wrap;
    gap: .5rem .6rem;
  }
  .pill{
    display: flex; align-items: center; gap: .6rem;
    background: var(--c);
    color: #222;
    border-radius: 9999px;
    padding: .35rem .6rem;
    box-shadow: 0 1px 0 rgba(0,0,0,.06) inset, 0 .5px 1px rgba(0,0,0,.07);
    min-height: 24px;
  }
  .pill-title{ font-weight: 600; }
  .pill-range{ opacity: .9; font-weight: 600; letter-spacing: .2px; }
  .quarterInfo{ margin: .25rem 0 0; font-size: 12px; font-weight: 600; }

  .mode-hints{
    display: grid;
    grid-template-columns: repeat(4, minmax(0, 1fr));
    gap: 3px;
    margin: .1rem 0;
    pointer-events: none;
  }

  .hint{
    border-radius: 16px;
    padding: 3px 6px;
    height: 28px;
    width: 220px;
    display: flex;
    align-items: center;
    justify-content: center;
    text-align: center;
    white-space: nowrap;
    font-size: 14px;
    font-weight: 500;
    background: rgba(0,0,0,.06);
    opacity: .9;
    
  }

  .container[data-mode="quarter"] .hint--1,
  .container[data-mode="quarter"] .hint--3,
  .container[data-mode="general"] .hint--2,
  .container[data-mode="annual"] .hint--1,
  .container[data-mode="annual"] .hint--3  { 
    transition: 800ms;
    color: #363636ff;
    background: #111827; 
  }
  .container[data-mode="quarter"] .hint--2,
  .container[data-mode="general"] .hint--1, 
  .container[data-mode="general"] .hint--3,
  .container[data-mode="annual"] .hint--2 {
    transition: 800ms;
    color: #131414ff;
    background: #27ff4bff; 
  }


</style>
