---
theme: dashboard
title: Observatorio de Paz y Seguridad – Seguridad en Colombia a través de los años
toc: false
---

```js
// ── Años disponibles por variable (basado en análisis del CSV real) ──────────
const YEARS_ALL     = [2004,2005,2006,2007,2008,2009,2010,2011,2012,2013,2014,2016,2018,2020,2021,2023,2025];
const YEARS_EXC_BIN = [2004,2005,2006,2007,2008,2009,2010,2011,2012,2013,2014,2016,2018];
const YEARS_EXC18   = [2008,2009,2010,2011,2012,2013,2014,2016,2018,2020,2021];
const YEARS_EXC20   = [2012,2013,2014,2016];
const YEARS_WC      = [2004,2005,2006,2007,2008,2009,2010,2011,2012,2013,2014,2016,2018];
const YEARS_EFF     = [2009,2010,2011,2012,2013,2014,2016,2018,2020,2023,2025];
const YEARS_CP      = [2004,2005,2006,2007,2008,2009,2010,2011,2012,2013,2014,2016,2018,2020,2023,2025];
const YEARS_PROT3   = [2010,2011,2012,2013,2014,2016,2018,2020,2021];
const YEARS_COLCP8A = [2004,2005,2006,2007,2008,2014,2016,2018,2020];
```

```js
// ── Carga y normalización de datos ──────────────────────────────────────────
// exc13-exc20, wc1-wc3: "Sí"/"No"
// prot3: "Sí ha participado" / "No ha participado"
// eff1, eff2: escala 1-7 con etiquetas ("Muy en desacuerdo"=1 … "Muy de acuerdo"=7)
// exc07: escala categórica de percepción de corrupción entre políticos
// pn4: "Muy satisfecho(a)" / "Satisfecho(a)" / "Insatisfecho(a)" / "Muy insatisfecho(a)"

const raw = await FileAttachment("data/observatorio_datos_years.csv").csv({ typed: true });

const siNo = v => {
  if (v == null || v === "") return null;
  const s = String(v).trim().toLowerCase().normalize("NFD").replace(/[\u0300-\u036f]/g,"");
  if (s === "si" || s === "1") return 1;
  if (s === "no" || s === "0") return 0;
  return null;
};

const siNoPartic = v => {
  if (v == null || v === "") return null;
  const s = String(v).trim().toLowerCase().normalize("NFD").replace(/[\u0300-\u036f]/g,"");
  if (s.startsWith("si")) return 1;
  if (s.startsWith("no")) return 0;
  return null;
};

const effScale = v => {
  if (v == null || v === "") return null;
  const s = String(v).trim().toLowerCase().normalize("NFD").replace(/[\u0300-\u036f]/g,"");
  if (s === "muy en desacuerdo") return 1;
  if (s === "muy de acuerdo")    return 7;
  const n = +v;
  return isNaN(n) ? null : n;
};

const data = raw.map(d => {
  const exc13 = siNo(d.exc13); const exc14 = siNo(d.exc14);
  const exc15 = siNo(d.exc15); const exc16 = siNo(d.exc16);
  const exc18 = siNo(d.exc18); const exc20 = siNo(d.exc20);
  const excFlags = [exc13,exc14,exc15,exc16,exc18,exc20].filter(x => x !== null);
  const victima_total = excFlags.length > 0 ? (excFlags.some(x => x===1) ? 1 : 0) : null;

  return {
    ...d,
    year: +d.year,
    exc13, exc14, exc15, exc16, exc18, exc20,
    wc1:  siNo(d.wc1), wc2: siNo(d.wc2), wc3: siNo(d.wc3),
    prot3: siNoPartic(d.prot3),
    eff1:  effScale(d.eff1),
    eff2:  effScale(d.eff2),
    victima_total
  };
});

const YEARS_DATA = [...new Set(data.map(d => d.year))].sort((a,b) => a-b);
```

```js
// ── Utilidades ───────────────────────────────────────────────────────────────

function pctSi(arr, variable) {
  const valid = arr.filter(d => d[variable] === 1 || d[variable] === 0);
  if (!valid.length) return null;
  return (valid.filter(d => d[variable] === 1).length / valid.length) * 100;
}

function pctVictima(arr) {
  const valid = arr.filter(d => d.victima_total === 1 || d.victima_total === 0);
  if (!valid.length) return null;
  return (valid.filter(d => d.victima_total === 1).length / valid.length) * 100;
}

function timeSeries(variable, availYears, filterFn = () => true) {
  return YEARS_DATA.filter(y => availYears.includes(y)).map(year => {
    const pct = pctSi(data.filter(d => d.year === year && filterFn(d)), variable);
    return { year, pct };
  }).filter(d => d.pct !== null);
}

function timeSeriesVictima(filterFn = () => true) {
  return YEARS_DATA.map(year => {
    const pct = pctVictima(data.filter(d => d.year === year && filterFn(d)));
    return { year, pct };
  }).filter(d => d.pct !== null);
}

function timeSeriesMedia(variable, availYears, filterFn = () => true) {
  return YEARS_DATA.filter(y => availYears.includes(y)).map(year => {
    const sub = data.filter(d => d.year === year && filterFn(d));
    const valid = sub.filter(d => d[variable] !== null && !isNaN(d[variable]));
    if (!valid.length) return { year, media: null };
    return { year, media: valid.reduce((s,d)=>s+d[variable],0) / valid.length };
  }).filter(d => d.media !== null);
}

function unique(col) {
  return [...new Set(data.map(d => d[col]).filter(v => v != null && v !== ""))].sort();
}
```

```js
// ── Controles globales ───────────────────────────────────────────────────────
const region = view(Inputs.select(["Todas", ...unique("estratopri")], {
  label: "Región",
  value: "Todas"
}));

const ur = view(Inputs.select(["Todas", ...unique("ur")], {
  label: "Zona",
  value: "Todas"
}));

const sexo = view(Inputs.select(["Todos", ...unique("sexo_std")], {
  label: "Género",
  value: "Todos"
}));

const etid = view(Inputs.select(["Todas", ...unique("etid")], {
  label: "Etnicidad",
  value: "Todas"
}));

function filtroGlobal(d) {
  if (region !== "Todas" && d.estratopri !== region) return false;
  if (ur     !== "Todas" && d.ur         !== ur)     return false;
  if (sexo   !== "Todos" && d.sexo_std   !== sexo)   return false;
  if (etid   !== "Todas" && d.etid       !== etid)   return false;
  return true;
}
```

```js
// ── KPIs sección 1 ────────────────────────────────────────────────────────────
const lastYear  = Math.max(...YEARS_DATA);
const prevYear  = YEARS_DATA[YEARS_DATA.indexOf(lastYear) - 1];
const subLast   = data.filter(d => d.year === lastYear && filtroGlobal(d));
const subPrev   = data.filter(d => d.year === prevYear && filtroGlobal(d));
const kpi_last  = pctVictima(subLast);
const kpi_prev  = pctVictima(subPrev);
const kpi_delta = (kpi_last !== null && kpi_prev !== null) ? (kpi_last - kpi_prev).toFixed(1) : null;
const deltaSign  = kpi_delta === null ? "" : (+kpi_delta > 0 ? "▲ +" : "▼ ");
const deltaClass = kpi_delta === null ? "" : (+kpi_delta > 0 ? "kpi-delta-pos" : "kpi-delta-neg");
```

<style>
.container { max-width: 1100px; margin: 0 auto; padding: 0 20px; }
.container :is(p, h1, h2, h3, h4, ul, ol, blockquote) { max-width: none !important; }
.container p { text-align: justify; line-height: 1.6; }
.container > * { width: 100%; }
h3 { margin-bottom: 16px !important; }
.filtros-bar {
  display: flex; flex-wrap: wrap; gap: 12px; align-items: flex-end;
  padding: 14px 18px; background: var(--theme-background-alt);
  border: 1px solid var(--theme-foreground-faint);
  border-radius: 8px; margin-bottom: 24px;
}
.kpi-grid {
  display: grid; grid-template-columns: repeat(auto-fit, minmax(180px,1fr));
  gap: 12px; margin-bottom: 24px;
}
.kpi-card {
  background: var(--theme-background-alt);
  border: 1px solid var(--theme-foreground-faint);
  border-radius: 8px; padding: 16px 18px;
}
.kpi-card .kpi-label { font-size:11px; color:var(--theme-foreground-muted); margin-bottom:6px; text-transform:uppercase; letter-spacing:.06em; }
.kpi-card .kpi-value { font-size:28px; font-weight:600; line-height:1; }
.kpi-card .kpi-sub   { font-size:12px; color:var(--theme-foreground-muted); margin-top:4px; }
.kpi-delta-pos { color:#e74c3c; }
.kpi-delta-neg { color:#27ae60; }
.sec-header {
  display:flex; align-items:center; gap:10px;
  margin:36px 0 18px; padding-bottom:10px;
  border-bottom:2px solid var(--theme-foreground-faint);
}
.sec-num {
  width:26px; height:26px; border-radius:50%;
  display:flex; align-items:center; justify-content:center;
  font-size:12px; font-weight:700; flex-shrink:0;
}
.sec-header h2 { margin:0 !important; font-size:18px; font-weight:600; }
.sec-header .sec-years { font-size:11px; color:var(--theme-foreground-muted); margin-left:auto; }
.aviso-nd {
  font-size:11px; color:var(--theme-foreground-muted);
  background:var(--theme-background-alt);
  border-left:3px solid var(--theme-foreground-faint);
  padding:6px 10px; border-radius:0 4px 4px 0; margin-bottom:12px;
}
.chart-wrap { width:100%; margin-bottom:24px; }
.chart-2col { display:grid; grid-template-columns:1fr 1fr; gap:20px; margin-bottom:24px; }
@media (max-width:700px) { .chart-2col { grid-template-columns:1fr; } }
</style>

<div class="container">

<div class="filtros-bar">
  ${region} ${ur} ${sexo} ${etid}
</div>

<!-- ══════════════════════════════════════════════════════════
     SECCIÓN 1 — PANORAMA GENERAL
══════════════════════════════════════════════════════════ -->
<div class="sec-header">
  <div class="sec-num" style="background:#dbeafe;color:#1d4ed8;">1</div>
  <h2>Panorama general de seguridad</h2>
  <span class="sec-years">2004 – ${lastYear}</span>
</div>

<p>
  El índice de victimización total indica si la persona reportó haber sido víctima de al menos
  uno de los siguientes delitos en los últimos 12 meses: extorsión, amenazas, agresión física,
  intento de robo a vivienda, fraude u otro delito. La variable exc07 corresponde a la
  percepción sobre qué tan generalizada está la corrupción entre los políticos, y se analiza
  por separado en la sección 2.
</p>

<div class="kpi-grid">
  <div class="kpi-card">
    <div class="kpi-label">% víctimas (${lastYear})</div>
    <div class="kpi-value">${kpi_last !== null ? kpi_last.toFixed(1)+"%" : "N/D"}</div>
    <div class="kpi-sub">Al menos un delito reportado</div>
  </div>
  <div class="kpi-card">
    <div class="kpi-label">Cambio vs ${prevYear}</div>
    <div class="kpi-value ${deltaClass}">${kpi_delta !== null ? deltaSign+kpi_delta+" pp" : "N/D"}</div>
    <div class="kpi-sub">Puntos porcentuales</div>
  </div>
  <div class="kpi-card">
    <div class="kpi-label">% extorsión (${lastYear})</div>
    <div class="kpi-value">${(() => { const p = pctSi(subLast,"exc13"); return p !== null ? p.toFixed(1)+"%" : "N/D"; })()}</div>
    <div class="kpi-sub">exc13 — tipo más grave registrado</div>
  </div>
  <div class="kpi-card">
    <div class="kpi-label">Encuestados (${lastYear})</div>
    <div class="kpi-value">${subLast.length.toLocaleString("es-CO")}</div>
    <div class="kpi-sub">Con filtros aplicados</div>
  </div>
</div>

```js
const serieVictima = timeSeriesVictima(filtroGlobal);
```

<div class="chart-wrap">

```js
Plot.plot({
  title: "Evolución del índice de victimización total (% víctimas)",
  width, height: 320,
  x: { label: "Año", tickFormat: d => String(d), grid: true },
  y: { label: "% víctimas", domain: [0, 50], grid: true },
  marks: [
    Plot.ruleY([0]),
    Plot.areaY(serieVictima, { x:"year", y:"pct", fill:"#1d4ed8", fillOpacity:0.08 }),
    Plot.lineY(serieVictima, { x:"year", y:"pct", stroke:"#1d4ed8", strokeWidth:2.5, tip:true }),
    Plot.dotY(serieVictima,  { x:"year", y:"pct", fill:"#1d4ed8", r:4 }),
    Plot.text(serieVictima.filter((_,i,a)=>i===a.length-1), {
      x:"year", y: d => d.pct+2.5,
      text: d => `${d.pct.toFixed(1)}%`, fontSize:11, fill:"#1d4ed8"
    })
  ]
})
```

</div>

```js
const barrasRegion = unique("estratopri").map(reg => {
  const pct = pctVictima(data.filter(d => d.year===lastYear && d.estratopri===reg && filtroGlobal(d)));
  return { region: reg, pct };
}).filter(d => d.pct !== null).sort((a,b) => b.pct-a.pct);

const barrasZona = ["Urbano","Rural"].map(z => {
  const pct = pctVictima(data.filter(d => d.year===lastYear && d.ur===z && filtroGlobal(d)));
  return { zona: z, pct };
}).filter(d => d.pct !== null);
```

<div class="chart-2col">

```js
Plot.plot({
  title: `% víctimas por región (${lastYear})`,
  width: width/2-12, height:240, marginLeft:140,
  x: { label:"%", domain:[0,50], grid:true }, y: { label:null },
  marks: [
    Plot.barX(barrasRegion, { x:"pct", y:"region", fill:"#1d4ed8", fillOpacity:0.8, tip:true }),
    Plot.text(barrasRegion, { x:"pct", y:"region", text:d=>`${d.pct.toFixed(1)}%`, dx:5, fontSize:11, textAnchor:"start" })
  ]
})
```

```js
Plot.plot({
  title: `% víctimas urbano vs rural (${lastYear})`,
  width: width/2-12, height:240, marginLeft:80,
  x: { label:"%", domain:[0,50], grid:true }, y: { label:null },
  marks: [
    Plot.barX(barrasZona, {
      x:"pct", y:"zona",
      fill: d => d.zona==="Urbano" ? "#1d4ed8" : "#7c3aed",
      tip:true
    }),
    Plot.text(barrasZona, { x:"pct", y:"zona", text:d=>`${d.pct.toFixed(1)}%`, dx:5, fontSize:12, textAnchor:"start" })
  ]
})
```

</div>


<!-- ══════════════════════════════════════════════════════════
     SECCIÓN 2 — TIPOLOGÍA DE LA INSEGURIDAD Y PERCEPCIÓN DE CORRUPCIÓN
══════════════════════════════════════════════════════════ -->
<div class="sec-header">
  <div class="sec-num" style="background:#fee2e2;color:#991b1b;">2</div>
  <h2>Tipología de la inseguridad</h2>
  <span class="sec-years">cobertura variable por delito</span>
</div>

<p>
  Comparación de la prevalencia de cada tipo de delito a lo largo del tiempo.
  Extorsión, amenazas, agresión y robo a vivienda están disponibles hasta 2018;
  fraude desde 2008; otro delito solo en 2012–2016. La variable exc07 mide la percepción
  ciudadana sobre qué tan generalizada está la corrupción entre los políticos.
</p>

```js
const delitosDef = [
  { var:"exc13", label:"Extorsión",             years:YEARS_EXC_BIN, color:"#dc2626" },
  { var:"exc14", label:"Amenazas",              years:YEARS_EXC_BIN, color:"#d97706" },
  { var:"exc15", label:"Agresión física",       years:YEARS_EXC_BIN, color:"#7c3aed" },
  { var:"exc16", label:"Intento robo vivienda", years:YEARS_EXC_BIN, color:"#059669" },
  { var:"exc18", label:"Fraude / estafa",       years:YEARS_EXC18,   color:"#db2777" },
  { var:"exc20", label:"Otro delito",           years:YEARS_EXC20,   color:"#92400e" },
];

const seriesDelitos = delitosDef.flatMap(def =>
  timeSeries(def.var, def.years, filtroGlobal).map(d => ({...d, tipo:def.label, color:def.color}))
);
```

<div class="aviso-nd">
  exc13–exc16: disponibles hasta 2018. exc18: desde 2008, sin 2023 ni 2025. exc20: solo 2012–2016.
</div>

<div class="chart-wrap">

```js
Plot.plot({
  title: "Evolución por tipo de delito (% víctimas)",
  width, height:360,
  x: { label:"Año", tickFormat:d=>String(d), grid:true },
  y: { label:"% víctimas", domain:[0,25], grid:true },
  color: { domain:delitosDef.map(d=>d.label), range:delitosDef.map(d=>d.color), legend:true },
  marks: [
    Plot.ruleY([0]),
    ...delitosDef.map(def => {
      const serie = seriesDelitos.filter(d => d.tipo===def.label);
      return [
        Plot.lineY(serie, { x:"year", y:"pct", stroke:def.color, strokeWidth:2, tip:true }),
        Plot.dotY(serie,  { x:"year", y:"pct", fill:def.color, r:3 })
      ];
    }).flat()
  ]
})
```

</div>

```js
const yearS2 = view(Inputs.select(YEARS_DATA.map(String), {
  label: "Año para comparar",
  value: String(2018)
}));
```

${yearS2}

```js
const barrasDelitos = delitosDef.map(def => {
  if (!def.years.includes(+yearS2)) return null;
  const pct = pctSi(data.filter(d => d.year===+yearS2 && filtroGlobal(d)), def.var);
  return pct !== null ? { tipo:def.label, pct, color:def.color } : null;
}).filter(Boolean).sort((a,b) => b.pct-a.pct);
```

<div class="chart-wrap">

```js
barrasDelitos.length
? Plot.plot({
    title: `Prevalencia por tipo de delito — ${yearS2}`,
    width, height: Math.max(180, barrasDelitos.length*44+60), marginLeft:170,
    x: { label:"%", domain:[0,25], grid:true }, y: { label:null },
    marks: [
      Plot.barX(barrasDelitos, { x:"pct", y:"tipo", fill:d=>d.color, fillOpacity:0.85, tip:true }),
      Plot.text(barrasDelitos, { x:"pct", y:"tipo", text:d=>`${d.pct.toFixed(1)}%`, dx:5, fontSize:11, textAnchor:"start" })
    ]
  })
: html`<div class="aviso-nd">No hay variables de victimización disponibles para ${yearS2}</div>`
```

</div>

<!-- Percepción de corrupción exc07 -->
```js
const exc07Orden  = ["Nada generalizada","Poco generalizada","Menos de la mitad","La mitad de los políticos","Más de la mitad","Algo generalizada","Muy generalizada","Todos"];
const exc07Colors = ["#bbf7d0","#86efac","#4ade80","#22c55e","#f59e0b","#f97316","#ef4444","#991b1b"];

const stackExc07 = YEARS_DATA.flatMap(year => {
  const valid = data.filter(d => d.year===year && filtroGlobal(d) && exc07Orden.includes(d.exc07));
  if (!valid.length) return [];
  return exc07Orden.map(cat => ({
    year, cat,
    pct: (valid.filter(d=>d.exc07===cat).length / valid.length) * 100
  }));
});
```

<div class="chart-wrap">

```js
Plot.plot({
  title: "Percepción de corrupción entre políticos — distribución por año (exc07)",
  width, height:320,
  x: { label:"Año", tickFormat:d=>String(d) },
  y: { label:"%", domain:[0,100], grid:true },
  color: { domain:exc07Orden, range:exc07Colors, legend:true, columns:4 },
  marks: [
    Plot.barY(stackExc07, Plot.stackY({ x:"year", y:"pct", fill:"cat", tip:true })),
    Plot.ruleY([0])
  ]
})
```

</div>


<!-- ══════════════════════════════════════════════════════════
     SECCIÓN 3 — CONTEXTO DEL CONFLICTO ARMADO
══════════════════════════════════════════════════════════ -->
<div class="sec-header">
  <div class="sec-num" style="background:#d1fae5;color:#065f46;">3</div>
  <h2>Contexto del conflicto armado</h2>
  <span class="sec-years">2004 – 2018</span>
</div>

<p>
  Estas variables capturan el impacto del conflicto armado en los hogares colombianos:
  si algún familiar fue víctima de muerte o desaparición (wc1), desplazamiento forzado (wc2)
  o tuvo que salir del país (wc3). No se midieron en 2020, 2021, 2023 ni 2025.
</p>

<div class="aviso-nd">
  wc1, wc2, wc3 no disponibles en 2020, 2021, 2023 ni 2025. Análisis cubre 2004–2018.
</div>

```js
const serieWC1 = timeSeries("wc1", YEARS_WC, filtroGlobal);
const serieWC2 = timeSeries("wc2", YEARS_WC, filtroGlobal);
const serieWC3 = timeSeries("wc3", YEARS_WC, filtroGlobal);
```

<div class="chart-wrap">

```js
Plot.plot({
  title: "Impacto del conflicto armado en hogares (% afectados)",
  width, height:320,
  x: { label:"Año", tickFormat:d=>String(d), grid:true },
  y: { label:"% hogares afectados", domain:[0,60], grid:true },
  color: {
    domain: ["Familiar fallecido/desaparecido (wc1)","Familiar desplazado (wc2)","Familiar salió del país (wc3)"],
    range:  ["#065f46","#0891b2","#7c3aed"],
    legend: true
  },
  marks: [
    Plot.ruleY([0]),
    Plot.lineY(serieWC1, { x:"year", y:"pct", stroke:"#065f46", strokeWidth:2.5, tip:true }),
    Plot.dotY(serieWC1,  { x:"year", y:"pct", fill:"#065f46", r:4 }),
    Plot.lineY(serieWC2, { x:"year", y:"pct", stroke:"#0891b2", strokeWidth:2, tip:true }),
    Plot.dotY(serieWC2,  { x:"year", y:"pct", fill:"#0891b2", r:4 }),
    Plot.lineY(serieWC3, { x:"year", y:"pct", stroke:"#7c3aed", strokeWidth:2, strokeDasharray:"4,3", tip:true }),
    Plot.dotY(serieWC3,  { x:"year", y:"pct", fill:"#7c3aed", r:4 })
  ]
})
```

</div>

```js
const kpiWC = [
  { label:"Familiar fallecido/desaparecido", var:"wc1" },
  { label:"Familiar desplazado",             var:"wc2" },
  { label:"Familiar salió del país",         var:"wc3" },
].map(k => ({
  ...k,
  pct: pctSi(data.filter(d => d.year===2018 && filtroGlobal(d)), k.var)
}));
```

<div class="kpi-grid">
  ${kpiWC.map(k => `
  <div class="kpi-card">
    <div class="kpi-label">${k.label}</div>
    <div class="kpi-value">${k.pct !== null ? k.pct.toFixed(1)+"%" : "N/D"}</div>
    <div class="kpi-sub">Último año disponible: 2018</div>
  </div>`).join("")}
</div>


<!-- ══════════════════════════════════════════════════════════
     SECCIÓN 4 — PERCEPCIÓN INSTITUCIONAL
══════════════════════════════════════════════════════════ -->
<div class="sec-header">
  <div class="sec-num" style="background:#ede9fe;color:#5b21b6;">4</div>
  <h2>Percepción institucional</h2>
  <span class="sec-years">2004 – ${lastYear}</span>
</div>

<p>
  Analiza la satisfacción con la democracia (pn4) y la eficacia política interna, medida
  por el grado de acuerdo con que los gobernantes se interesan por la gente (eff1) y que
  el encuestado entiende los asuntos políticos (eff2). eff1 y eff2 usan escala
  1 (Muy en desacuerdo) a 7 (Muy de acuerdo).
</p>

```js
const pn4Cats   = ["Muy satisfecho(a)","Satisfecho(a)","Insatisfecho(a)","Muy insatisfecho(a)"];
const pn4Colors = ["#15803d","#86efac","#fca5a5","#dc2626"];

const stackPn4 = YEARS_DATA.flatMap(year => {
  const valid = data.filter(d => d.year===year && filtroGlobal(d) && pn4Cats.includes(d.pn4));
  if (!valid.length) return [];
  return pn4Cats.map(cat => ({
    year, cat,
    pct: (valid.filter(d=>d.pn4===cat).length / valid.length) * 100
  }));
});
```

<div class="chart-wrap">

```js
Plot.plot({
  title: "Satisfacción con la democracia — distribución por año (pn4)",
  width, height:320,
  x: { label:"Año", tickFormat:d=>String(d) },
  y: { label:"%", domain:[0,100], grid:true },
  color: { domain:pn4Cats, range:pn4Colors, legend:true },
  marks: [
    Plot.barY(stackPn4, Plot.stackY({ x:"year", y:"pct", fill:"cat", tip:true })),
    Plot.ruleY([0])
  ]
})
```

</div>

```js
const serieEff1 = timeSeriesMedia("eff1", YEARS_EFF, filtroGlobal);
const serieEff2 = timeSeriesMedia("eff2", YEARS_EFF, filtroGlobal);
```

<div class="aviso-nd">
  eff1 y eff2 no disponibles en 2004–2008 ni en 2021. Línea punteada marca el punto medio de la escala (4).
</div>

<div class="chart-2col">

```js
Plot.plot({
  title: "Los gobernantes se interesan por la gente (eff1) — media escala 1–7",
  width: width/2-12, height:260,
  x: { label:"Año", tickFormat:d=>String(d), grid:true },
  y: { label:"Media", domain:[1,7], grid:true },
  marks: [
    Plot.ruleY([4], { stroke:"#aaa", strokeDasharray:"4,3" }),
    Plot.lineY(serieEff1, { x:"year", y:"media", stroke:"#5b21b6", strokeWidth:2.5, tip:true }),
    Plot.dotY(serieEff1,  { x:"year", y:"media", fill:"#5b21b6", r:4 })
  ]
})
```

```js
Plot.plot({
  title: "Comprensión de asuntos políticos (eff2) — media escala 1–7",
  width: width/2-12, height:260,
  x: { label:"Año", tickFormat:d=>String(d), grid:true },
  y: { label:"Media", domain:[1,7], grid:true },
  marks: [
    Plot.ruleY([4], { stroke:"#aaa", strokeDasharray:"4,3" }),
    Plot.lineY(serieEff2, { x:"year", y:"media", stroke:"#0891b2", strokeWidth:2.5, tip:true }),
    Plot.dotY(serieEff2,  { x:"year", y:"media", fill:"#0891b2", r:4 })
  ]
})
```

</div>


<!-- ══════════════════════════════════════════════════════════
     SECCIÓN 5 — PARTICIPACIÓN SOCIAL
══════════════════════════════════════════════════════════ -->
<div class="sec-header">
  <div class="sec-num" style="background:#fce7f3;color:#9d174d;">5</div>
  <h2>Participación social</h2>
  <span class="sec-years">cobertura irregular por variable</span>
</div>

<p>
  Mide la frecuencia de participación en espacios comunitarios y cívicos: organizaciones
  religiosas (cp6), partidos políticos (cp13), Juntas de Acción Comunal (colcp8a) y
  manifestaciones públicas (prot3). Las preguntas de frecuencia usan la escala:
  Nunca / Una o dos veces al año / Una o dos veces al mes / Una vez a la semana.
</p>

```js
const yearS5 = view(Inputs.select(YEARS_DATA.map(String), {
  label: "Año",
  value: String(2018)
}));
```

${yearS5}

```js
const freqOrden  = ["Una vez a la semana","Una o dos veces al mes","Una o dos veces al año","Nunca"];
const freqColors = ["#1d4ed8","#60a5fa","#fcd34d","#e5e7eb"];

function stackFreq(variable, availYears, yearVal) {
  if (!availYears.includes(+yearVal)) return [];
  const valid = data.filter(d => d.year===+yearVal && filtroGlobal(d) && freqOrden.includes(d[variable]));
  if (!valid.length) return [];
  return freqOrden.map(cat => ({
    cat, pct: (valid.filter(d=>d[variable]===cat).length / valid.length) * 100
  }));
}

const freqCp6    = stackFreq("cp6",     YEARS_CP,      yearS5);
const freqCp13   = stackFreq("cp13",    YEARS_CP,      yearS5);
const freqColcp8 = stackFreq("colcp8a", YEARS_COLCP8A, yearS5);
```

<div class="chart-2col">

```js
freqCp6.length
? Plot.plot({
    title: `Asistencia a org. religiosa — ${yearS5} (cp6)`,
    width:width/2-12, height:210, marginLeft:170,
    x:{label:"%",domain:[0,100],grid:true}, y:{label:null},
    color:{domain:freqOrden,range:freqColors,legend:false},
    marks:[
      Plot.barX(freqCp6,{x:"pct",y:"cat",fill:"cat",tip:true}),
      Plot.text(freqCp6,{x:"pct",y:"cat",text:d=>`${d.pct.toFixed(1)}%`,dx:5,fontSize:11,textAnchor:"start"})
    ]
  })
: html`<div class="aviso-nd">cp6 no disponible en ${yearS5}</div>`
```

```js
freqCp13.length
? Plot.plot({
    title: `Asistencia a partido político — ${yearS5} (cp13)`,
    width:width/2-12, height:210, marginLeft:170,
    x:{label:"%",domain:[0,100],grid:true}, y:{label:null},
    color:{domain:freqOrden,range:freqColors,legend:false},
    marks:[
      Plot.barX(freqCp13,{x:"pct",y:"cat",fill:"cat",tip:true}),
      Plot.text(freqCp13,{x:"pct",y:"cat",text:d=>`${d.pct.toFixed(1)}%`,dx:5,fontSize:11,textAnchor:"start"})
    ]
  })
: html`<div class="aviso-nd">cp13 no disponible en ${yearS5}</div>`
```

</div>

```js
freqColcp8.length
? Plot.plot({
    title: `Asistencia a Junta de Acción Comunal — ${yearS5} (colcp8a)`,
    width, height:210, marginLeft:170,
    x:{label:"%",domain:[0,100],grid:true}, y:{label:null},
    color:{domain:freqOrden,range:freqColors,legend:false},
    marks:[
      Plot.barX(freqColcp8,{x:"pct",y:"cat",fill:"cat",tip:true}),
      Plot.text(freqColcp8,{x:"pct",y:"cat",text:d=>`${d.pct.toFixed(1)}%`,dx:5,fontSize:11,textAnchor:"start"})
    ]
  })
: html`<div class="aviso-nd">colcp8a no disponible en ${yearS5} — disponible en: ${YEARS_COLCP8A.join(", ")}</div>`
```

```js
const serieProt3 = timeSeries("prot3", YEARS_PROT3, filtroGlobal);
```

<div class="aviso-nd">
  prot3 disponible en 2010–2021. No incluida en 2004–2008, 2023 ni 2025.
</div>

<div class="chart-wrap">

```js
Plot.plot({
  title: "Participación en manifestaciones o protestas públicas — % (prot3)",
  width, height:260,
  x:{label:"Año",tickFormat:d=>String(d),grid:true},
  y:{label:"% participó",domain:[0,20],grid:true},
  marks:[
    Plot.ruleY([0]),
    Plot.areaY(serieProt3,{x:"year",y:"pct",fill:"#9d174d",fillOpacity:0.12}),
    Plot.lineY(serieProt3,{x:"year",y:"pct",stroke:"#9d174d",strokeWidth:2.5,tip:true}),
    Plot.dotY(serieProt3, {x:"year",y:"pct",fill:"#9d174d",r:4})
  ]
})
```

</div>

---

<p style="font-size:12px;color:var(--theme-foreground-muted);text-align:center;margin-top:32px;">
  Fuente: Barómetro de las Américas – LAPOP Colombia · 2004–2025 · Observatorio de Paz y Seguridad
</p>

</div>
