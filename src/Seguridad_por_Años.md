---
theme: dashboard
title: Observatorio de Paz y Seguridad – Seguridad en Colombia a través de los años
toc: false
---
<style>
.container {
  max-width: 1100px;
  margin: 0 auto;
  padding: 0 20px;
}

.container :is(p, h1, h2, h3, h4, ul, ol, blockquote) {
  max-width: none !important;
}

.container p {
  text-align: justify;
  line-height: 1.6;
}

.container > * {
  width: 100%;
}
</style>
<div class="container">

<style>
h3 {
  margin-bottom: 30px !important;
}
</style>

```js
//Load data
import * as d3 from "npm:d3";
import * as Plot from "npm:@observablehq/plot";

// Load + normalize in one step
const raw = (await FileAttachment("data/observatorio_datos_years.csv").csv({ typed: true }))
  .map(d =>
    Object.fromEntries(
      Object.entries(d).map(([k, v]) => [String(k).trim().toLowerCase(), v])
    )
  );

// Clean data
const clean = raw.map(d => ({
  ...d,
  year: Number(d.year),
  estratopri: d.estratopri == null ? null : String(d.estratopri).trim(),
  ur: d.ur == null ? null : String(d.ur).trim(),
  sexo_std: d.sexo_std == null ? null : String(d.sexo_std).trim(),
  etid: d.etid == null ? null : String(d.etid).trim(),
  wealth: d.wealth == null ? null : String(d.wealth).trim()
}));

// Helpers
const siNo = v => {
  if (v == null || v === "") return null;
  const s = String(v).trim().toLowerCase().normalize("NFD").replace(/[\u0300-\u036f]/g, "");
  if (s === "si" || s === "1" || s === "true") return 1;
  if (s === "no" || s === "0" || s === "false") return 0;
  return null;
};

const siNoPartic = v => {
  if (v == null || v === "") return null;
  const s = String(v).trim().toLowerCase().normalize("NFD").replace(/[\u0300-\u036f]/g, "");
  if (s.startsWith("si")) return 1;
  if (s.startsWith("no")) return 0;
  return null;
};

const effScale = v => {
  if (v == null || v === "") return null;
  const s = String(v).trim().toLowerCase().normalize("NFD").replace(/[\u0300-\u036f]/g, "");
  if (s === "muy en desacuerdo") return 1;
  if (s === "muy de acuerdo") return 7;
  const n = Number(String(v).trim());
  return Number.isNaN(n) ? null : n;
};

// Valid years (exc13–exc16, 2004-2018 without 2015 and 2017)
const YEARS_VICTIMA = [2004, 2005, 2006, 2007, 2008, 2009, 2010, 2011, 2012, 2013, 2014, 2016, 2018];

// Build final data array
// FIX CLAVE: exc07 se guarda como string (texto), NO con siNo() que lo destruiría
const data = clean.map(d => {
  const exc13 = siNo(d.exc13);
  const exc14 = siNo(d.exc14);
  const exc15 = siNo(d.exc15);
  const exc16 = siNo(d.exc16);
  const exc18 = siNo(d.exc18);
  const exc20 = siNo(d.exc20);
  // exc07: se conserva el valor original sin unificar (dos escalas históricas distintas)
  const exc07 = (d.exc07 == null || d.exc07 === "") ? null : String(d.exc07).trim();

  const coreFlags = [exc13, exc14, exc15, exc16].filter(x => x !== null);

  const victima_total =
    coreFlags.length > 0
      ? (coreFlags.some(x => x === 1) ? 1 : 0)
      : null;

  return {
    ...d,
    exc13, exc14, exc15, exc16, exc18, exc20, exc07,
    wc1: siNo(d.wc1),
    wc2: siNo(d.wc2),
    wc3: siNo(d.wc3),
    prot3: siNoPartic(d.prot3),
    eff1: effScale(d.eff1),
    eff2: effScale(d.eff2),
    victima_total
  };
});

// Subset restricted to 2004–2018
const dataCore = data.filter(d => YEARS_VICTIMA.includes(d.year));
```

## Observatorio de Paz y Seguridad – Seguridad en Colombia a través de los años
---
## Sección 1: Panorama General de Seguridad en Colombia según la percepción del ciudadano

---

<p>El índice de victimización comparable (exc13–exc16) permite analizar la evolución de la inseguridad en Colombia utilizando únicamente los delitos medidos de forma consistente en el tiempo. Esto garantiza comparabilidad longitudinal y evita sesgos derivados de cambios en el cuestionario. La métrica refleja el porcentaje de personas que reportaron haber sido víctimas de al menos uno de estos delitos en los últimos 12 meses.</p>

```js
// Utility functions
function pctSi(arr, variable) {
  const valid = arr.filter(d => d[variable] === 1 || d[variable] === 0);
  if (!valid.length) return null;
  return (valid.filter(d => d[variable] === 1).length / valid.length) * 100;
}

function pctVictima(arr) {
  if (!arr.length) return null;
  const victimas = arr.filter(d => d.victima_total === 1).length;
  return (victimas / arr.length) * 100;
}

function unique(col) {
  return [...new Set(dataCore.map(d => d[col]).filter(v => v != null && v !== ""))].sort();
}

function timeSeriesVictima(filterFn = () => true) {
  return YEARS_VICTIMA.map(year => {
    const sub = dataCore.filter(d => d.year === year && filterFn(d));
    const pct = pctVictima(sub);
    return { year, pct };
  }).filter(d => d.pct !== null);
}
```

```js
// Global filter inputs
const selRegion = Inputs.select(["Todas", ...unique("estratopri")], {
  label: "Región",
  value: "Todas"
});

const selUr = Inputs.select(["Todas", ...unique("ur")], {
  label: "Zona",
  value: "Todas"
});

const selSexo = Inputs.select(["Todos", ...unique("sexo_std")], {
  label: "Género",
  value: "Todos"
});

const selEtid = Inputs.select(["Todas", ...unique("etid")], {
  label: "Etnicidad",
  value: "Todas"
});

const selYear = Inputs.select(YEARS_VICTIMA, {
  label: "Año (KPIs)",
  value: 2018
});
```

```js
// Reactive values
const region = Generators.input(selRegion);
const zona   = Generators.input(selUr);
const sexo   = Generators.input(selSexo);
const etid   = Generators.input(selEtid);
```

```js
const yearKpi = Generators.input(selYear);
```

```js
// filtroGlobal — celda reactiva propia
const filtroGlobal = (d) => {
  if (region !== "Todas" && d.estratopri !== region) return false;
  if (zona   !== "Todas" && d.ur         !== zona)   return false;
  if (sexo   !== "Todos" && d.sexo_std   !== sexo)   return false;
  if (etid   !== "Todas" && d.etid       !== etid)   return false;
  return true;
};
```

```js
// Serie reactiva para la gráfica principal
const serieVictima = timeSeriesVictima(filtroGlobal);
```

```js
// KPIs reactivos según el año elegido
const yearPrev    = YEARS_VICTIMA[YEARS_VICTIMA.indexOf(yearKpi) - 1] ?? null;

const subKpi      = dataCore.filter(d => d.year === yearKpi  && filtroGlobal(d));
const subKpiPrev  = yearPrev !== null
  ? dataCore.filter(d => d.year === yearPrev && filtroGlobal(d))
  : [];

const kpi_n        = subKpi.length;
const kpi_pct      = pctVictima(subKpi);
const kpi_pct_prev = pctVictima(subKpiPrev);

const kpi_delta =
  kpi_pct !== null && kpi_pct_prev !== null
    ? kpi_pct - kpi_pct_prev
    : null;

const deltaSign =
  kpi_delta === null ? "" : kpi_delta > 0 ? "▲ +" : kpi_delta < 0 ? "▼ " : "= ";

const deltaClass =
  kpi_delta === null ? "" : kpi_delta > 0 ? "kpi-delta-pos" : kpi_delta < 0 ? "kpi-delta-neg" : "";
```

<div class="filtros-bar">
  ${selRegion} ${selUr} ${selSexo} ${selEtid}
</div>

<p>
  <strong>Resumen de filtros:</strong>
  ${region} | ${zona} | ${sexo} | ${etid}
</p>

<!--KPIs Styles-->
<style>
.kpi-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 12px;
  margin: 18px 0 24px 0;
}

.kpi-card {
  background: var(--theme-background-alt, rgba(255,255,255,0.03));
  border: 1px solid var(--theme-foreground-faint, rgba(255,255,255,0.12));
  border-radius: 8px;
  padding: 16px 18px;
}

.kpi-label {
  font-size: 11px;
  color: var(--theme-foreground-muted);
  margin-bottom: 6px;
  text-transform: uppercase;
  letter-spacing: .06em;
}

.kpi-value {
  font-size: 28px;
  font-weight: 600;
  line-height: 1;
}

.kpi-sub {
  font-size: 12px;
  color: var(--theme-foreground-muted);
  margin-top: 6px;
}

.kpi-delta-pos { color: #e74c3c; }
.kpi-delta-neg { color: #27ae60; }
</style>

<div style="margin-bottom: 12px;">
  ${selYear}
</div>

<div class="kpi-grid">
  <div class="kpi-card">
    <div class="kpi-label">Encuestados (${String(yearKpi)})</div>
    <div class="kpi-value">${kpi_n.toLocaleString("es-CO")}</div>
    <div class="kpi-sub">Con filtros aplicados</div>
  </div>

  <div class="kpi-card">
    <div class="kpi-label">% víctimas (${String(yearKpi)})</div>
    <div class="kpi-value">
      ${kpi_pct !== null ? kpi_pct.toFixed(1) + "%" : "N/D"}
    </div>
    <div class="kpi-sub">Al menos 1 de los 4 delitos comparables</div>
  </div>

  <div class="kpi-card">
    <div class="kpi-label">Cambio vs ${yearPrev ?? "—"}</div>
    <div class="kpi-value ${deltaClass}">
      ${kpi_delta !== null ? deltaSign + Math.abs(kpi_delta).toFixed(1) + " pp" : "N/D"}
    </div>
    <div class="kpi-sub">
      ${kpi_pct_prev !== null
        ? "Año anterior: " + kpi_pct_prev.toFixed(1) + "%"
        : "Sin año anterior disponible"}
    </div>
  </div>
</div>

```js
Plot.plot({
  title: "Victimización comparable 2004–2018 (exc13, exc14, exc15, exc16)",
  subtitle: "% de personas que reportaron al menos un delito de los 4 comparables",
  width,
  height: 340,
  x: {
    label: "Año",
    tickFormat: d => String(d),
    tickValues: YEARS_VICTIMA,
    grid: true
  },
  y: {
    label: "% víctimas",
    domain: [0, 20],
    grid: true
  },
  marks: [
    Plot.ruleY([0]),
    Plot.areaY(serieVictima, {
      x: "year",
      y: "pct",
      fill: "#1d4ed8",
      fillOpacity: 0.08
    }),
    Plot.lineY(serieVictima, {
      x: "year",
      y: "pct",
      stroke: "#1d4ed8",
      strokeWidth: 2.5,
      tip: true
    }),
    Plot.dotY(serieVictima, {
      x: "year",
      y: "pct",
      fill: "white",
      stroke: "#1d4ed8",
      strokeWidth: 2,
      r: 5
    }),
    Plot.text(serieVictima, {
      x: "year",
      y: "pct",
      text: d => d.pct.toFixed(1) + "%",
      dy: -12,
      fontSize: 10,
      fill: "currentColor"
    }),
    Plot.ruleX([yearKpi], {
      stroke: "#f59e0b",
      strokeWidth: 1.5,
      strokeDasharray: "4,3"
    })
  ]
})
```


<p>Entre 2004 y 2018, la victimización muestra una tendencia general a la baja con fluctuaciones intermedias. Se observa una reducción importante entre 2004 (8.5%) y 2009 (2.9%), seguida de una estabilización con leves repuntes alrededor de 2012 (5.6%) y 2014 (5.0%). Posteriormente, la tendencia vuelve a moderarse, alcanzando 3.5% en 2018. Estos resultados sugieren que, si bien hubo mejoras estructurales en los niveles de seguridad durante la década analizada, la victimización no desaparece sino que presenta ciclos de repunte asociados posiblemente a dinámicas locales o cambios en las modalidades delictivas.</p>

---

### Análisis territorial — % victimización por región

```js
// GeoJSON Colombia (departamentos)
const colombiaGeo = await fetch(
  "https://gist.githubusercontent.com/john-guerra/43c7656821069d00dcbc/raw/be6a6e239cd5b5b803c6e7c2ec405b793a9064dd/Colombia.geo.json"
).then(r => r.json());
```

```js
// Mapeo departamento → región (estratopri)
const REGION_MAP_VICTIMA = {
  "ANTIOQUIA": "Andina", "BOYACA": "Andina", "CALDAS": "Andina",
  "CUNDINAMARCA": "Andina", "HUILA": "Andina", "NORTE DE SANTANDER": "Andina",
  "QUINDIO": "Andina", "RISARALDA": "Andina", "SANTANDER": "Andina", "TOLIMA": "Andina",
  "ATLANTICO": "Caribe", "BOLIVAR": "Caribe", "CESAR": "Caribe", "CORDOBA": "Caribe",
  "LA GUAJIRA": "Caribe", "MAGDALENA": "Caribe", "SUCRE": "Caribe", "SAN ANDRES": "Caribe",
  "CAUCA": "Pacífica", "CHOCO": "Pacífica", "NARIÑO": "Pacífica", "VALLE DEL CAUCA": "Pacífica",
  "AMAZONAS": "Orinoquía-Amazonía", "ARAUCA": "Orinoquía-Amazonía", "CAQUETA": "Orinoquía-Amazonía",
  "CASANARE": "Orinoquía-Amazonía", "GUAINIA": "Orinoquía-Amazonía", "GUAVIARE": "Orinoquía-Amazonía",
  "META": "Orinoquía-Amazonía", "PUTUMAYO": "Orinoquía-Amazonía", "VAUPES": "Orinoquía-Amazonía",
  "VICHADA": "Orinoquía-Amazonía",
  "BOGOTA": "Bogotá D.C.", "BOGOTÁ": "Bogotá D.C.", "BOGOTA D.C.": "Bogotá D.C."
};

// Pesos poblacionales DANE 2025 para cálculo ponderado
const REGION_PESOS_VICTIMA = {
  "Andina": 22800000, "Bogotá D.C.": 7942867,
  "Caribe": 11500000, "Pacífica": 5800000, "Orinoquía-Amazonía": 2014000
};
const TOTAL_POB_VICTIMA = d3.sum(Object.values(REGION_PESOS_VICTIMA));

// Centroides aproximados por región para etiquetas
const REGION_CENTROIDS_VICTIMA = [
  { region: "Andina",             x: -74.5,  y:  5.5  },
  { region: "Caribe",             x: -75.0,  y: 10.2  },
  { region: "Pacífica",           x: -77.2,  y:  4.5  },
  { region: "Orinoquía-Amazonía", x: -71.5,  y:  2.0  },
  { region: "Bogotá D.C.",        x: -74.08, y:  4.71 }
];
```

```js
// Selector de año para el mapa — usa YEARS_VICTIMA
const selYearMapa = Inputs.select(YEARS_VICTIMA, {
  label: "Año (mapa)",
  value: 2018
});
```

```js
const yearMapa = Generators.input(selYearMapa);
```

```js
// % victimización por región para el año seleccionado
const victByRegionMapa = (() => {
  const sub = dataCore.filter(d => d.year === yearMapa && d.estratopri != null);
  const rollup = d3.rollups(sub, v => pctVictima(v), d => d.estratopri);
  return new Map(rollup);
})();

// % victimización ponderado por población para el año
const pctPonderadoMapa = (() => {
  let sum = 0, totalPeso = 0;
  for (const [region, pesos] of Object.entries(REGION_PESOS_VICTIMA)) {
    const pct = victByRegionMapa.get(region);
    if (pct != null) { sum += pct * pesos; totalPeso += pesos; }
  }
  return totalPeso > 0 ? sum / totalPeso : null;
})();

// Rango para la escala de color (sobre todos los años para escala fija)
const victExtent = (() => {
  const todos = YEARS_VICTIMA.flatMap(year => {
    const sub = dataCore.filter(d => d.year === year && d.estratopri != null);
    return d3.rollups(sub, v => pctVictima(v), d => d.estratopri)
      .map(([, pct]) => pct).filter(v => v != null);
  });
  return d3.extent(todos);
})();

// Añadir % victimización a features del GeoJSON
const geoVictima = {
  ...colombiaGeo,
  features: colombiaGeo.features.map(f => {
    const dpto = f.properties.NOMBRE_DPT?.toUpperCase().trim();
    const region = REGION_MAP_VICTIMA[dpto] ?? null;
    const pct = region ? victByRegionMapa.get(region) : null;
    return { ...f, properties: { ...f.properties, region, pct } };
  })
};
```

<div style="margin-bottom: 12px;">${selYearMapa}</div>

```js
{
  // Escala de color fija (rojo=baja, verde=alta victimización invertido: rojo=alta, azul=baja)
  const colorScaleMapa = d3.scaleSequential()
    .domain(victExtent)
    .interpolator(d3.interpolateYlOrRd);  // amarillo→rojo = baja→alta victimización

  function mapaVictimizacion({width} = {}) {
    const height = width * 1.15;
    const projection = d3.geoMercator().fitSize([width, height], geoVictima);
    const path = d3.geoPath(projection);

    const labelData = REGION_CENTROIDS_VICTIMA.map(c => {
      const pct = victByRegionMapa.get(c.region);
      const [px, py] = projection([c.x, c.y]);
      return { ...c, px, py, pct };
    }).filter(d => d.pct != null);

    const svg = d3.create("svg")
      .attr("width", width)
      .attr("height", height)
      .style("font-family", "sans-serif");

    // Departamentos coloreados
    svg.selectAll("path")
      .data(geoVictima.features)
      .join("path")
      .attr("d", path)
      .attr("fill", d => d.properties.pct != null ? colorScaleMapa(d.properties.pct) : "#444")
      .attr("stroke", "white")
      .attr("stroke-width", 0.5)
      .append("title")
      .text(d => {
        const r = d.properties.region ?? "Sin datos";
        const p = d.properties.pct != null ? d.properties.pct.toFixed(1) + "%" : "N/D";
        return `${d.properties.NOMBRE_DPT}\nRegión: ${r}\n% víctimas (${yearMapa}): ${p}`;
      });

    // Etiquetas por región
    labelData.forEach(d => {
      const g = svg.append("g").attr("transform", `translate(${d.px},${d.py})`);
      g.append("rect")
        .attr("x", -54).attr("y", -28).attr("width", 0).attr("height", 0)
        .attr("rx", 6).attr("fill", "rgba(0,0,0,0.70)");
      g.append("text")
        .attr("text-anchor", "middle").attr("y", -12).attr("fill", "white")
        .attr("font-size", d.region === "Bogotá D.C." ? "8px" : "9px")
        .attr("font-weight", "bold").text(d.region);
      g.append("text")
        .attr("text-anchor", "middle").attr("y", 5)
        .attr("fill", "white")
        .attr("font-size", "12px").attr("font-weight", "bold")
        .text(d.pct.toFixed(1) + "%");
    });

    // Leyenda
    const lw = Math.min(180, width * 0.38), lh = 11;
    const lx = width - lw - 14, ly = height - 48;
    const defs = svg.append("defs");
    const grad = defs.append("linearGradient").attr("id", "vict-grad");
    d3.range(0, 1.01, 0.1).forEach(t => {
      grad.append("stop").attr("offset", `${t*100}%`)
        .attr("stop-color", colorScaleMapa(victExtent[0] + t*(victExtent[1]-victExtent[0])));
    });
    const lg = svg.append("g").attr("transform", `translate(${lx},${ly})`);
    lg.append("rect").attr("width", lw).attr("height", lh).attr("rx", 3).attr("fill", "url(#vict-grad)");
    lg.append("text").attr("y", lh+13).attr("fill", "currentColor").attr("font-size", "9px")
      .text(victExtent[0].toFixed(1) + "% (baja)");
    lg.append("text").attr("x", lw).attr("y", lh+13).attr("text-anchor", "end")
      .attr("fill", "currentColor").attr("font-size", "9px").text(victExtent[1].toFixed(1) + "% (alta)");
    lg.append("text").attr("x", lw/2).attr("y", -5).attr("text-anchor", "middle")
      .attr("fill", "currentColor").attr("font-size", "9px").text("% victimización por región");

    return svg.node();
  }

  // KPI ponderado junto al mapa
  const kpiDiv = document.createElement("div");
  kpiDiv.style.cssText = "font-size:13px; margin-bottom:10px; color:var(--theme-foreground-muted)";
  kpiDiv.innerHTML = pctPonderadoMapa != null
    ? `<strong style="color:var(--theme-foreground)">% victimización ponderado por población (${yearMapa}):</strong> ${pctPonderadoMapa.toFixed(1)}% <span style="font-size:11px">(ajustado por peso regional DANE 2025)</span>`
    : "Sin datos para el año seleccionado";

  const wrapper = document.createElement("div");
  wrapper.appendChild(kpiDiv);

  const mapContainer = document.createElement("div");
  mapContainer.style.cssText = "max-width:520px; margin:0 auto;";

  // Función resize
  const ro = new ResizeObserver(([entry]) => {
    mapContainer.innerHTML = "";
    mapContainer.appendChild(mapaVictimizacion({ width: Math.min(entry.contentRect.width, 520) }));
  });
  ro.observe(wrapper);
  mapContainer.appendChild(mapaVictimizacion({ width: 480 }));
  wrapper.appendChild(mapContainer);
  display(wrapper);
}
```

---

  <p>El análisis territorial de la victimización entre 2004 y 2018 revela una reducción sostenida en todas las regiones del país, aunque con trayectorias y velocidades diferenciadas. En 2004, los niveles eran generalizadamente altos: Bogotá D.C. registraba el 10.4%, el Caribe el 10.3%, la Pacífica el 8.8% y la Andina el 7.3%, mientras que la Orinoquía-Amazonía presentaba el valor más bajo con 6.4%. Para 2018, todos los territorios habían reducido sustancialmente sus cifras, con la Andina descendiendo al 2.1% y la Orinoquía-Amazonía al 2.8%, consolidándose como las regiones con menor victimización al final del período. Sin embargo, la Pacífica y el Caribe mantuvieron los índices más elevados en 2018, con 5.6% y 4.3% respectivamente, lo que sugiere que la mejora agregada no ha sido homogénea y que persisten brechas territoriales estructurales. Un patrón notable es la alta volatilidad interanual en varias regiones: la Pacífica, por ejemplo, oscila entre 1.1% en 2010 y 11.0% en 2005, y Bogotá D.C. entre 2.2% en 2006 y 10.4% en 2004, lo cual indica que los factores que determinan la victimización en estos territorios son sensibles a dinámicas locales y no responden de manera lineal a tendencias nacionales. En términos de tamaño muestral, las regiones Andina y Orinoquía-Amazonía concentran consistentemente el mayor número de encuestados, lo que otorga mayor estabilidad estadística a sus estimaciones, mientras que Pacífica y Bogotá D.C. presentan muestras más reducidas, lo que podría amplificar la varianza en sus porcentajes. En conjunto, los datos evidencian que Colombia experimentó una mejora real y sostenida en materia de seguridad ciudadana durante el período analizado, pero que esta mejora ha sido geográficamente desigual, con la región Pacífica y el Caribe como territorios que requieren atención prioritaria desde una perspectiva de política pública territorial.
  </p>

---

```js
// Checkbox multi-selección de región
const regionesDisponibles = [...new Set(dataCore.map(d => d.estratopri).filter(v => v != null && v !== ""))].sort();
const selRegionChart = Inputs.checkbox(regionesDisponibles, {
  label: "Filtrar región",
  value: regionesDisponibles  // todas seleccionadas por defecto
});
```

```js
const regionesSeleccionadas = Generators.input(selRegionChart);
```

```js
// Series para gráficas de líneas múltiples
const serieRegion = YEARS_VICTIMA.flatMap(year => {
  const sub = dataCore.filter(d => d.year === year && filtroGlobal(d) && d.estratopri != null);
  const byRegion = d3.rollups(sub, v => pctVictima(v), d => d.estratopri);
  return byRegion
    .filter(([, pct]) => pct !== null)
    .map(([region, pct]) => ({ year, category: region, pct }));
});

const serieZona = YEARS_VICTIMA.flatMap(year => {
  const sub = dataCore.filter(d => d.year === year && filtroGlobal(d) && d.ur != null);
  const byZona = d3.rollups(sub, v => pctVictima(v), d => d.ur);
  return byZona
    .filter(([, pct]) => pct !== null)
    .map(([zona, pct]) => ({ year, category: zona, pct }));
});

// Filtrar por regiones seleccionadas con checkbox
const serieRegionFiltrada = regionesSeleccionadas.length === 0
  ? serieRegion
  : serieRegion.filter(d => regionesSeleccionadas.includes(d.category));
```

  ${selRegionChart}

```js
{
  const ultimoAnoRegion = d3.max(serieRegionFiltrada, d => d.year);

  const chartRegion = Plot.plot({
    title: "% víctimas por región",
    width: Math.floor(width / 2) - 12,
    height: 340,
    x: {
      label: "Año",
      tickFormat: d => String(d),
      tickValues: YEARS_VICTIMA,
      grid: true
    },
    y: { label: "% víctimas", domain: [0, 12], grid: true },
    color: { legend: true },
    marks: [
      Plot.ruleY([0]),
      Plot.lineY(serieRegionFiltrada, {
        x: "year", y: "pct", z: "category",
        stroke: "category", strokeWidth: 2, tip: true
      }),
      Plot.dotY(serieRegionFiltrada, {
        x: "year", y: "pct", fill: "category", r: 4
      }),
      Plot.text(serieRegionFiltrada, {
        x: "year",
        y: "pct",
        z: "category",
        text: d => d.pct.toFixed(1) + "%",
        dy: -10,
        fontSize: 10,
        fill: "white"
      }
    )
  ]
  });

  const chartZona = Plot.plot({
    title: "% víctimas urbano vs rural",
    width: Math.floor(width / 2) - 12,
    height: 340,
    x: {
      label: "Año",
      tickFormat: d => String(d),
      tickValues: YEARS_VICTIMA,
      grid: true
    },
    y: { label: "% víctimas", domain: [0, 10], grid: true },
    color: { legend: true },
    marks: [
    Plot.ruleY([0]),

    Plot.lineY(serieZona, {
      x: "year", y: "pct", z: "category",
      stroke: "category", strokeWidth: 2, tip: true
    }),

    Plot.dotY(serieZona, {
      x: "year", y: "pct", fill: "category", r: 4
    }),

    // 👇 AGREGA ESTO
    Plot.text(serieZona, {
      x: "year",
      y: "pct",
      z: "category",
      text: d => d.pct.toFixed(1) + "%",
      dy: -10,
      fontSize: 10,
      fill: "white"
    })
  ]
  });

  const div = document.createElement("div");
  div.style.cssText = "display:flex; gap:24px; align-items:flex-start;";
  div.appendChild(chartRegion);
  div.appendChild(chartZona);
  display(div);
}
```
<p>Por otro lado, la comparación entre zonas urbanas y rurales muestra diferencias consistentes, ay que la victimización urbana tiende a ser más alta en varios años iniciales, reflejando mayor exposición a delitos como robo o extorsión. Sin embargo, la brecha no es constante porque en algunos años las diferencias se reducen o incluso se invierten. Se puede ver que ambas series muestran una tendencia general descendente, lo que refuerza la mejora agregada observada en el indicador general; esto sugiere que las dinámicas de inseguridad están influenciadas tanto por el entorno urbano como por factores específicos del territorio rural.</p>

---
## Sección 2: Tipología de la inseguridad
---

<p>Esta sección analiza la evolución de los distintos tipos de victimización reportados por los ciudadanos en los últimos 12 meses, desagregando el fenómeno de la inseguridad en categorías específicas: extorsión, amenazas, agresión física, intento de robo a vivienda, fraude/estafa y otros delitos. Este enfoque permite identificar no solo la magnitud del problema, sino también los cambios en su composición a lo largo del tiempo. Adicionalmente, se incorpora la percepción de corrupción entre los políticos (exc07), como una dimensión complementaria que refleja el entorno institucional en el que se desarrolla la seguridad ciudadana</p>


```js
// Sección 2: configuración
const DELITOS_DEF = [
  { var: "exc13", label: "Extorsión" },
  { var: "exc14", label: "Amenazas" },
  { var: "exc15", label: "Agresión física" },
  { var: "exc16", label: "Intento robo vivienda" },
  { var: "exc18", label: "Fraude / estafa" },
  { var: "exc20", label: "Otro delito" }
];

const YEARS_DELITOS = [2004,2005,2006,2007,2008,2009,2010,2011,2012,2013,2014,2016,2018,2020,2021];
const YEARS_EXC07   = [2004,2005,2006,2007,2008,2009,2010,2011,2012,2013,2014,2016,2018,2020,2021,2023];

// Años disponibles por variable (para evitar nulls silenciosos)
const YEARS_POR_VAR = {
  exc13: [2004,2005,2006,2007,2008,2009,2010,2011,2012,2013,2014,2016,2018],
  exc14: [2004,2005,2006,2007,2008,2009,2010,2011,2012,2013,2014,2016,2018],
  exc15: [2004,2005,2006,2007,2008,2009,2010,2011,2012,2013,2014,2016,2018],
  exc16: [2004,2005,2006,2007,2008,2009,2010,2011,2012,2013,2014,2016,2018],
  exc18: [2008,2009,2010,2011,2012,2013,2014,2016,2018,2020,2021],
  exc20: [2012,2013,2014,2016]
};

// Dos escalas históricas diferentes — NO se unifican
// Periodo A (2004–2014): escala de intensidad
const EXC07_CATS_A = [
  "Nada generalizada",
  "Poco generalizada",
  "Algo generalizada",
  "Muy generalizada"
];
// Periodo B (2016–2023): escala de proporción de políticos
const EXC07_CATS_B = [
  "Ninguno",
  "Menos de la mitad",
  "La mitad de los políticos",
  "Más de la mitad",
  "Todos"
];
const YEARS_EXC07_A = [2004,2005,2006,2007,2008,2009,2010,2011,2012,2013,2014];
const YEARS_EXC07_B = [2016,2018,2020,2021,2023];
```

```js
// Checkbox multi-selección de delitos
const selDelitoLinea = Inputs.checkbox(
  DELITOS_DEF.map(d => d.label),
  { label: "Filtrar delito", value: DELITOS_DEF.map(d => d.label) }
);

// Checkbox para periodo A (2004–2014)
const selExc07A = Inputs.checkbox(
  EXC07_CATS_A,
  { label: "Filtrar categoría (2004–2014)", value: EXC07_CATS_A }
);
// Checkbox para periodo B (2016–2023)
const selExc07B = Inputs.checkbox(
  EXC07_CATS_B,
  { label: "Filtrar categoría (2016–2023)", value: EXC07_CATS_B }
);
```

```js
const delitosSeleccionados = Generators.input(selDelitoLinea);
const exc07SelA = Generators.input(selExc07A);
const exc07SelB = Generators.input(selExc07B);
```

```js
// Serie líneas delitos — eje X años, color por tipo de delito
const serieDelitos = DELITOS_DEF.flatMap(def => {
  const yearsVar = YEARS_POR_VAR[def.var];
  return yearsVar.map(year => {
    const sub = data.filter(d => d.year === year && filtroGlobal(d));
    const pct = pctSi(sub, def.var);
    return pct !== null ? { year, categoria: def.label, pct } : null;
  }).filter(Boolean);
});

const serieDelitosFiltrada = delitosSeleccionados.length === 0
  ? serieDelitos
  : serieDelitos.filter(d => delitosSeleccionados.includes(d.categoria));
```

```js
// Serie exc07 Periodo A: 2004–2014 (escala de intensidad)
const serieExc07A = YEARS_EXC07_A.flatMap(year => {
  const sub = data.filter(d =>
    d.year === year && filtroGlobal(d) &&
    d.exc07 !== null && EXC07_CATS_A.includes(d.exc07)
  );
  if (!sub.length) return [];
  const total = sub.length;
  return d3.rollups(sub, v => v.length, d => d.exc07)
    .map(([cat, cnt]) => ({ year, categoria: cat, pct: (cnt / total) * 100 }));
});

// Serie exc07 Periodo B: 2016–2023 (escala de proporción de políticos)
const serieExc07B = YEARS_EXC07_B.flatMap(year => {
  const sub = data.filter(d =>
    d.year === year && filtroGlobal(d) &&
    d.exc07 !== null && EXC07_CATS_B.includes(d.exc07)
  );
  if (!sub.length) return [];
  const total = sub.length;
  return d3.rollups(sub, v => v.length, d => d.exc07)
    .map(([cat, cnt]) => ({ year, categoria: cat, pct: (cnt / total) * 100 }));
});

const serieExc07AFiltrada = exc07SelA.length === 0 ? serieExc07A : serieExc07A.filter(d => exc07SelA.includes(d.categoria));
const serieExc07BFiltrada = exc07SelB.length === 0 ? serieExc07B : serieExc07B.filter(d => exc07SelB.includes(d.categoria));
```
  ${selDelitoLinea}

```js
Plot.plot({
  title: "Tipología de victimización por año",
  subtitle: "% de personas que reportaron haber sido víctimas en los últimos 12 meses",
  width,
  height: 360,
  x: {
    label: "Año",
    tickFormat: d => String(d),
    tickValues: YEARS_DELITOS,
    grid: true
  },
  y: { label: "% víctimas", domain: [0, 30], grid: true },
  color: { legend: true },
  marks: [
    Plot.ruleY([0]),
    Plot.lineY(serieDelitosFiltrada, {
      x: "year", y: "pct", z: "categoria",
      stroke: "categoria", strokeWidth: 2, tip: true
    }),
    Plot.dotY(serieDelitosFiltrada, {
      x: "year", y: "pct", fill: "categoria", r: 4
    }),
    Plot.text(serieDelitosFiltrada, {
      x: "year",
      y: "pct",
      text: d => d.pct.toFixed(2) + "%",
      dy: -8,
      fontSize: 9,
      fill: "white",
      stroke: "none"
    })
  ]
})
```
---

<p>Con respecto a los resultados muestran una estructura claramente diferenciada entre tipos de delito, en Fraude / estafa aparece como el delito más prevalente durante gran parte del periodo, con niveles significativamente superiores al resto, alcanzando picos cercanos al 25% (2009) y manteniéndose en niveles elevados incluso en años recientes. Esto sugiere un desplazamiento progresivo hacia delitos no violentos pero de alto impacto económico. En Amenazas constituyen el segundo tipo de victimización más frecuente, con una evolución relativamente estable entre 4% y 7%, lo que indica persistencia de formas de intimidación como mecanismo de control o presión. Por otro lado Extorsión presenta niveles moderados (alrededor de 3%–5%), con ligeras variaciones, reflejando su carácter estructural en ciertos contextos territoriales. Si nos referimos a Agresión física y intento de robo a vivienda muestran niveles más bajos y una tendencia general decreciente, lo que podría estar asociado a mejoras en seguridad urbana tradicional o cambios en patrones delictivos. Finalmente otros delitos mantienen una incidencia marginal, pero con presencia constante, lo que evidencia la diversidad de experiencias de victimización no capturadas en las categorías principales. En conjunto, se observa una transición desde delitos más visibles y violentos hacia formas más sofisticadas o menos visibles, particularmente relacionadas con fraude.</p>

${selExc07A}

```js
Plot.plot({
  title: "Percepción de corrupción entre políticos — Periodo 2004–2014 (exc07)",
  subtitle: "Escala: Nada generalizada → Muy generalizada · % de respuestas por categoría",
  width,
  height: 360,
  x: {
    label: "Año",
    tickFormat: d => String(d),
    tickValues: YEARS_EXC07_A,
    grid: true
  },
  y: { label: "%", domain: [0, 70], grid: true },
  color: {
    domain: EXC07_CATS_A,
    range: ["#16a34a", "#86efac", "#f97316", "#dc2626"],
    legend: true
  },
  marks: [
    Plot.ruleY([0]),
    Plot.lineY(serieExc07AFiltrada, {
      x: "year", y: "pct", z: "categoria",
      stroke: "categoria", strokeWidth: 2, tip: true
    }),
    Plot.dotY(serieExc07AFiltrada, {
      x: "year", y: "pct", fill: "categoria", r: 4
    }),
    Plot.text(serieExc07AFiltrada, {
      x: "year", y: "pct",
      text: d => d.pct.toFixed(1) + "%",
      dy: -10, fontSize: 10, fill: "white", stroke: "none"
    })
  ]
})
```

${selExc07B}

```js
Plot.plot({
  title: "Percepción de corrupción entre políticos — Periodo 2016–2023 (exc07)",
  subtitle: "Escala: Ninguno → Todos los políticos · % de respuestas por categoría",
  width,
  height: 360,
  x: {
    label: "Año",
    tickFormat: d => String(d),
    tickValues: YEARS_EXC07_B,
    grid: true
  },
  y: { label: "%", domain: [0, 60], grid: true },
  color: {
    domain: EXC07_CATS_B,
    range: ["#16a34a", "#86efac", "#fbbf24", "#f97316", "#dc2626"],
    legend: true
  },
  marks: [
    Plot.ruleY([0]),
    Plot.lineY(serieExc07BFiltrada, {
      x: "year", y: "pct", z: "categoria",
      stroke: "categoria", strokeWidth: 2, tip: true
    }),
    Plot.dotY(serieExc07BFiltrada, {
      x: "year", y: "pct", fill: "categoria", r: 4
    }),
    Plot.text(serieExc07BFiltrada, {
      x: "year", y: "pct",
      text: d => d.pct.toFixed(1) + "%",
      dy: -10, fontSize: 10, fill: "white", stroke: "none"
    })
  ]
})
```

<p>La percepción de corrupción entre políticos presenta niveles elevados y persistentes a lo largo del tiempo, en las categorías “algo generalizada” y “muy generalizada” concentran consistentemente la mayor proporción de respuestas, superando en varios años el 50% de los encuestados. Existe una tendencia creciente entre 2008 y 2014, donde la percepción de corrupción se intensifica, en años recientes (2016–2023), aunque se observa cierta fluctuación, la percepción sigue siendo mayoritariamente negativa, con más del 40%–50% ubicándose en niveles altos de corrupción. Por otro lado, las categorías de baja corrupción (“nada” o “poco generalizada”) permanecen en niveles muy reducidos, lo que indica una baja confianza estructural en la integridad del sistema político.</p>

---
## Sección 3: Contexto del conflicto armado
---
<p>Estas variables capturan el impacto del conflicto armado en los hogares colombianos:
muerte o desaparición de un familiar (wc1), desplazamiento forzado (wc2) y salida del país (wc3).
La visualización muestra la evolución temporal del porcentaje de hogares afectados.</p>

```js
  const yearsWC = [
    2004,2005,2006,2007,2008,2009,2010,2011,2012,2013,2014,2016,2018,2020,2025
  ];

  function buildSeriesWC(variable, label, color) {
    return yearsWC.map(year => {
      const subset = data.filter(d => d.year === year && filtroGlobal(d));
      const pct = pctSi(subset, variable);

      return pct !== null
        ? { year, pct, tipo: label, color }
        : null;
    }).filter(Boolean);
  }

  const serieWC = [
    ...buildSeriesWC("wc1", "Familiar fallecido/desaparecido", "#10b981"),
    ...buildSeriesWC("wc2", "Familiar desplazado", "#06b6d4"),
    ...buildSeriesWC("wc3", "Familiar salió del país", "#8b5cf6")
  ];
```
```js
  Plot.plot({
    title: "Impacto del conflicto armado en hogares (% afectados)",
    width,
    height: 380,

    x: {
      label: "Año",
      domain: yearsWC,
      tickValues: yearsWC,
      tickFormat: d => String(d),
      grid: true
    },

    y: {
      label: "% hogares afectados",
      domain: [0, 60],
      grid: true
    },

    color: {
      domain: [
        "Familiar fallecido/desaparecido",
        "Familiar desplazado",
        "Familiar salió del país"
      ],
      range: ["#10b981", "#06b6d4", "#8b5cf6"],
      legend: true
    },

    marks: [
      Plot.ruleY([0]),

      // líneas
      Plot.lineY(serieWC, {
        x: "year",
        y: "pct",
        stroke: "tipo",
        strokeWidth: 2.5,
        tip: true
      }),

      // puntos
      Plot.dotY(serieWC, {
        x: "year",
        y: "pct",
        fill: "white",
        stroke: "tipo",
        strokeWidth: 2,
        r: 5
      }),

      Plot.text(serieWC, {
        x: "year",
        y: "pct",
        text: d => d.pct.toFixed(2) + "%",
        dy: -12,
        fontSize: 10,
        fill: "white"
      })
    ]
  })
```
<p>Las variables asociadas al contexto del conflicto armado permiten dimensionar el impacto directo que este ha tenido sobre los hogares colombianos a lo largo del tiempo, a partir de tres dimensiones clave: la pérdida de un familiar por muerte o desaparición (wc1), el desplazamiento forzado (wc2) y la salida del país como consecuencia del conflicto (wc3). En conjunto, estas métricas ofrecen una aproximación a las huellas sociales persistentes del conflicto, más allá de los indicadores tradicionales de violencia.

Los resultados muestran que la afectación por conflicto armado ha sido significativa y relativamente estable en el tiempo, especialmente en las dimensiones de pérdida de familiares y desplazamiento. La proporción de hogares que reportan haber perdido un familiar o tener un desaparecido (wc1) se mantiene en niveles elevados durante todo el periodo analizado, oscilando entre aproximadamente 21% y 28%, con un incremento progresivo hacia los años posteriores a 2010 y un pico cercano al 28% en 2014. Aunque posteriormente se observa una leve disminución, los niveles siguen siendo altos, lo que evidencia la persistencia del impacto acumulado del conflicto en la memoria y estructura de los hogares.

Por su parte, el desplazamiento forzado (wc2) presenta una dinámica similar, aunque con niveles ligeramente inferiores. Este indicador fluctúa entre aproximadamente 16% y 26%, mostrando una tendencia creciente desde finales de la década de 2000 hasta mediados de la década de 2010. En años más recientes, si bien se observa cierta estabilización, el fenómeno continúa afectando a una proporción considerable de la población, reflejando que el desplazamiento ha sido una de las principales manifestaciones del conflicto en términos sociales y territoriales.

En contraste, la variable relacionada con la salida del país por razones del conflicto (wc3) presenta niveles considerablemente más bajos, situándose generalmente entre 4% y 10%. No obstante, su evolución muestra una tendencia creciente en los años más recientes, alcanzando su valor más alto hacia el final del periodo analizado. Esto podría indicar un aumento en estrategias de movilidad internacional como respuesta a contextos de inseguridad o falta de oportunidades asociadas al conflicto.

En conjunto, estos resultados evidencian que, aunque ciertos indicadores de violencia directa han disminuido en el tiempo, el impacto del conflicto armado sigue siendo profundo y persistente en los hogares colombianos. Las altas proporciones de afectación, particularmente en términos de pérdida de familiares y desplazamiento, sugieren que las consecuencias del conflicto trascienden los periodos de mayor intensidad y continúan influyendo en las condiciones sociales del país, consolidándose como un elemento estructural en la experiencia de seguridad y bienestar de la población.</p>

---
## Sección 4: Percepción institucional
---
<p>La percepción institucional constituye un componente fundamental para comprender la relación entre ciudadanía, Estado y seguridad. En esta sección se analizan tres dimensiones clave: la satisfacción con el funcionamiento de la democracia (pn4), el grado en que los ciudadanos consideran que los gobernantes se interesan por la gente (eff1) y la autopercepción sobre la comprensión de los asuntos políticos (eff2). En conjunto, estos indicadores permiten evaluar tanto la legitimidad del sistema político como la percepción de eficacia y cercanía institucional.

Analiza la satisfacción con la democracia (<strong>pn4</strong>) y la eficacia política interna, medida por el grado de acuerdo con que los gobernantes se interesan por la gente (<strong>eff1</strong>) y que el encuestado entiende los asuntos políticos (<strong>eff2</strong>). eff1 y eff2 usan una escala de 1 (Muy en desacuerdo) a 7 (Muy de acuerdo).
</p>

```js
  const YEARS_PN4  = [2004,2005,2006,2007,2008,2009,2010,2011,2012,2013,2014,2016,2018,2020,2021,2023,2025];
  const YEARS_EFF  = [2009,2010,2011,2012,2013,2014,2016,2018,2020,2023,2025];

  const PN4_CATS = [
    "Muy satisfecho(a)",
    "Satisfecho(a)",
    "Insatisfecho(a)",
    "Muy insatisfecho(a)"
  ];

  function mediaVariable(arr, variable) {
    const valid = arr
      .map(d => d[variable])
      .filter(v => v !== null && v !== undefined && !Number.isNaN(v));

    if (!valid.length) return null;
    return d3.mean(valid);
  }

  const seriePn4 = YEARS_PN4.flatMap(year => {
    const sub = data.filter(d =>
      d.year === year &&
      filtroGlobal(d) &&
      d.pn4 !== null &&
      PN4_CATS.includes(d.pn4)
    );

    if (!sub.length) return [];

    const total = sub.length;

    return PN4_CATS.map(cat => {
      const count = sub.filter(d => d.pn4 === cat).length;
      return {
        year,
        categoria: cat,
        pct: (count / total) * 100
      };
    });
  });

  const serieEff1 = YEARS_EFF.map(year => {
    const sub = data.filter(d => d.year === year && filtroGlobal(d));
    const media = mediaVariable(sub, "eff1");
    return media !== null ? { year, media } : null;
  }).filter(Boolean);

  const serieEff2 = YEARS_EFF.map(year => {
    const sub = data.filter(d => d.year === year && filtroGlobal(d));
    const media = mediaVariable(sub, "eff2");
    return media !== null ? { year, media } : null;
  }).filter(Boolean);
```

```js
  Plot.plot({
    title: "Satisfacción con la democracia — distribución por año (pn4)",
    width,
    height: 360,
    x: {
      label: "Año",
      tickFormat: d => String(d),
      tickValues: YEARS_PN4,
      grid: true
    },
    y: {
      label: "%",
      domain: [0, 100],
      grid: true
    },
    color: {
      domain: PN4_CATS,
      range: ["#16a34a", "#86efac", "#fca5a5", "#dc2626"],
      legend: true
    },
    marks: [
  Plot.ruleY([0]),

  // barras apiladas
  Plot.barY(
    seriePn4,
    Plot.stackY({
      x: "year",
      y: "pct",
      fill: "categoria",
      tip: true
    })
  ),

  // labels centrados en cada segmento
  Plot.text(
    seriePn4,
    Plot.stackY({
      x: "year",
      y: "pct",
      text: d => d.pct > 3 ? d.pct.toFixed(1) + "%" : "", // evita ruido
      fill: "white",
      fontSize: 10,
      textAnchor: "middle",
      dy: 0
    })
  )
]
  })
```
---
<p>La satisfacción con el funcionamiento de la democracia en Colombia muestra una transformación radical entre 2004 y 2025. Durante el período 2004–2012, el sistema político gozaba de una legitimidad relativamente alta: las categorías de satisfacción ("Muy satisfecho" + "Satisfecho") sumaban consistentemente entre el 55% y el 62% de los encuestados, con niveles de insatisfacción extrema ("Muy insatisfecho") que no superaban el 7.5%. Este escenario cambia de forma abrupta a partir de 2013, cuando la insatisfacción comienza a dominar la distribución. En 2013, los insatisfechos ya representan el 55.5% y los muy insatisfechos el 12.7%, una ruptura que se consolida en los años posteriores. El punto más crítico se registra en 2020, donde la insatisfacción acumulada supera el 80% de los encuestados, con un 27.9% en la categoría de muy insatisfecho, posiblemente asociado al impacto de la pandemia, las protestas sociales y el deterioro de la percepción institucional. En los años más recientes (2021–2025) se observa una leve recuperación de las categorías de satisfacción, que vuelven a rondar el 25%–28%, aunque la insatisfacción sigue siendo mayoritaria, lo que indica que el sistema político colombiano enfrenta un déficit de legitimidad estructural que no ha logrado revertirse completamente.</p>

```js
// ── Sección 4: variables relacionadas con pn4 ─────────────────────────────

// Años donde coinciden pn4 + eff1 + eff2
const YEARS_PN4_EFF = [2008,2009,2010,2011,2012,2013,2014,2016,2018,2020,2023,2025];

// Años donde coinciden pn4 + cp6 + cp13 + colcp8a + prot3
const YEARS_PN4_PART = [2016,2018,2020,2025];

// Orden canónico de pn4 (de más a menos satisfecho)
const PN4_ORDER = ["Muy satisfecho(a)", "Satisfecho(a)", "Insatisfecho(a)", "Muy insatisfecho(a)"];

// cp6 / cp13 / colcp8a: participación activa = cualquier respuesta distinta a "Nunca"
const esParticipante = v => v != null && v !== "Nunca";
```

```js
// Selector de año compartido para gráficas eff1/eff2 vs pn4
const selYearEff = Inputs.select(YEARS_PN4_EFF, {
  label: "Año (eficacia política)",
  value: 2025
});

// Selector de año para participación vs pn4
const selYearPart4 = Inputs.select(YEARS_PN4_PART, {
  label: "Año (participación)",
  value: 2025
});
```

```js
const yearEff   = Generators.input(selYearEff);
const yearPart4 = Generators.input(selYearPart4);
```

```js
// ── Gráfica 1: eff1 y eff2 por categoría pn4 ──────────────────────────────
// Para cada grupo pn4, media de eff1 y media de eff2
const dataEff4 = PN4_ORDER.flatMap(cat => {
  const sub = data.filter(d =>
    d.year === yearEff &&
    filtroGlobal(d) &&
    d.pn4 === cat
  );
  const mediaEff1 = sub.filter(d => d.eff1 != null).length > 0
    ? d3.mean(sub.filter(d => d.eff1 != null), d => d.eff1)
    : null;
  const mediaEff2 = sub.filter(d => d.eff2 != null).length > 0
    ? d3.mean(sub.filter(d => d.eff2 != null), d => d.eff2)
    : null;
  return [
    mediaEff1 != null ? { grupo: cat, variable: "eff1 — Gobernantes se interesan por la gente", media: mediaEff1 } : null,
    mediaEff2 != null ? { grupo: cat, variable: "eff2 — Comprendo los asuntos políticos",       media: mediaEff2 } : null
  ].filter(Boolean);
});
```

```js
// ── Gráfica 2: participación por categoría pn4 ────────────────────────────
// Para cada grupo pn4, % que participó en cp6, cp13, colcp8a, prot3
const dataPart4 = PN4_ORDER.flatMap(cat => {
  const sub = data.filter(d =>
    d.year === yearPart4 &&
    filtroGlobal(d) &&
    d.pn4 === cat
  );
  if (!sub.length) return [];

  const vars = [
    { key: "cp6",    label: "Org. religiosa (cp6)" },
    { key: "cp13",   label: "Partido político (cp13)" },
    { key: "colcp8a",label: "JAC (colcp8a)" },
    { key: "prot3",  label: "Manifestación (prot3)" }
  ];

  return vars.map(({ key, label }) => {
    const validos = sub.filter(d => d[key] != null);
    if (!validos.length) return null;
    // prot3 es siNoPartic (1/0), cp variables son texto frecuencia
    const participa = key === "prot3"
      ? validos.filter(d => d[key] === 1).length
      : validos.filter(d => esParticipante(d[key])).length;
    return {
      grupo: cat,
      variable: label,
      pct: (participa / validos.length) * 100
    };
  }).filter(Boolean);
});
```

<div style="margin-bottom:10px">${selYearEff}</div>

```js
{
  // Separar datos por variable para dos gráficas independientes
  const dataEff1 = dataEff4.filter(d => d.variable === "eff1 — Gobernantes se interesan por la gente");
  const dataEff2 = dataEff4.filter(d => d.variable === "eff2 — Comprendo los asuntos políticos");

  const W = Math.floor(width / 2) - 20;

  // Opciones comunes de eje X — sin rotación, con más margen abajo
  const xOpts = {
    domain: PN4_ORDER,
    label: null,           // label abajo lo ponemos con marginBottom
    tickSize: 6,
    tickPadding: 8,
    tickFormat: d => {
      // Abreviar para que quepan sin solapar
      const MAP = {
        "Muy satisfecho(a)":    "Muy satisfecho",
        "Satisfecho(a)":        "Satisfecho",
        "Insatisfecho(a)":      "Insatisfecho",
        "Muy insatisfecho(a)":  "Muy insatisfecho"
      };
      return MAP[d] ?? d;
    }
  };

  const mkEff = (titulo, datos, color) => Plot.plot({
    title: titulo,
    subtitle: `${yearEff} — escala 1 (desacuerdo) a 7 (acuerdo)`,
    width: W,
    height: 400,
    marginBottom: 60,
    marginLeft: 52,
    x: xOpts,
    y: { label: "Media 1–7", domain: [1, 7], grid: true },
    marks: [
      Plot.ruleY([4], { stroke: "#888", strokeDasharray: "4,3", strokeOpacity: 0.5 }),
      Plot.ruleY([0]),
      Plot.barY(datos, { x: "grupo", y: "media", fill: color, fillOpacity: 0.85, tip: true }),
      Plot.text(datos, {
        x: "grupo", y: "media",
        text: d => d.media.toFixed(2),
        dy: -10, fontSize: 11,
        fill: "currentColor", textAnchor: "middle"
      })
    ]
  });

  const c1 = mkEff("eff1 — Gobernantes se interesan por la gente", dataEff1, "#8b5cf6");
  const c2 = mkEff("eff2 — Comprendo los asuntos políticos",        dataEff2, "#06b6d4");

  const row = document.createElement("div");
  row.style.cssText = "display:flex; gap:32px; align-items:flex-start;";
  row.appendChild(c1);
  row.appendChild(c2);
  display(row);
}
```

<p>Esta gráfica muestra cómo varían la percepción de eficacia política interna (eff1: los gobernantes se interesan por la gente; eff2: comprensión de asuntos políticos) según el nivel de satisfacción con la democracia. Se esperaría una relación positiva: quienes están más satisfechos con la democracia también percibirían que los gobernantes se interesan más por la ciudadanía. Una brecha entre eff1 y eff2 dentro del mismo grupo indicaría que las personas pueden entender la política sin necesariamente confiar en que el sistema responde a sus intereses.

El análisis de la eficacia política interna según el nivel de satisfacción democrática revela patrones consistentes y teóricamente coherentes.Para el año 2025, el análisis de la eficacia política interna según el nivel de satisfacción democrática revela en el caso de eff1 (percepción de que los gobernantes se interesan por la gente), se observa una relación positiva clara con la satisfacción democrática: quienes se declaran "Muy satisfechos" asignan una media de 3.71, los "Satisfechos" de 3.84, mientras que los "Insatisfechos" caen a 3.05 y los "Muy insatisfechos" a 2.62, todos por debajo del punto medio de la escala (4). Esto indica que, independientemente del grupo, la ciudadanía percibe que los gobernantes no responden adecuadamente a sus intereses, siendo esta percepción significativamente más negativa entre quienes están insatisfechos con la democracia. En contraste, eff2 (comprensión de los asuntos políticos) muestra valores más homogéneos entre grupos: 3.95 para los muy satisfechos, 4.01 para los satisfechos, 3.66 para los insatisfechos y 3.80 para los muy insatisfechos, todos cercanos al punto medio de la escala. Esta menor variación sugiere que la comprensión subjetiva de la política no depende tanto de la satisfacción con el sistema democrático, sino que es una capacidad percibida de manera relativamente uniforme en la población. La brecha entre eff1 y eff2 dentro de cada grupo es especialmente reveladora: los ciudadanos sienten que entienden la política, pero no confían en que el sistema responda a sus necesidades, una combinación que puede alimentar tanto la frustración democrática como la movilización social.</p>

---

<div style="margin-bottom:10px">${selYearPart4}</div>

```js
{
  const dataCp6   = dataPart4.filter(d => d.variable === "Org. religiosa (cp6)");
  const dataCp13  = dataPart4.filter(d => d.variable === "Partido político (cp13)");
  const dataJac   = dataPart4.filter(d => d.variable === "JAC (colcp8a)");
  const dataProt3 = dataPart4.filter(d => d.variable === "Manifestación (prot3)");

  const W = Math.floor(width / 2) - 20;

  const xOpts = {
    domain: PN4_ORDER,
    label: null,
    tickSize: 6,
    tickPadding: 8,
    tickFormat: d => {
      const MAP = {
        "Muy satisfecho(a)":   "Muy satisfecho",
        "Satisfecho(a)":       "Satisfecho",
        "Insatisfecho(a)":     "Insatisfecho",
        "Muy insatisfecho(a)": "Muy insatisfecho"
      };
      return MAP[d] ?? d;
    }
  };

  const mkPart = (titulo, datos, color) => Plot.plot({
    title: titulo,
    subtitle: String(yearPart4),
    width: W,
    height: 360,
    marginBottom: 60,
    marginLeft: 52,
    x: xOpts,
    y: { label: "% participó", domain: [0, 100], grid: true },
    marks: [
      Plot.ruleY([0]),
      Plot.barY(datos, { x: "grupo", y: "pct", fill: color, fillOpacity: 0.85, tip: true }),
      Plot.text(datos, {
        x: "grupo", y: "pct",
        text: d => d.pct.toFixed(1) + "%",
        dy: -10, fontSize: 11,
        fill: "currentColor", textAnchor: "middle"
      })
    ]
  });

  // Fila 1: cp6 y cp13
  const row1 = document.createElement("div");
  row1.style.cssText = "display:flex; gap:32px; align-items:flex-start; margin-bottom:28px;";
  row1.appendChild(mkPart("Org. religiosa (cp6)",    dataCp6,   "#f59e0b"));
  row1.appendChild(mkPart("Partido político (cp13)", dataCp13,  "#3b82f6"));

  // Fila 2: JAC y manifestación
  const row2 = document.createElement("div");
  row2.style.cssText = "display:flex; gap:32px; align-items:flex-start;";
  row2.appendChild(mkPart("JAC (colcp8a)",       dataJac,   "#10b981"));
  row2.appendChild(mkPart("Manifestación (prot3)", dataProt3, "#ef4444"));

  const wrapper = document.createElement("div");
  wrapper.appendChild(row1);
  wrapper.appendChild(row2);
  display(wrapper);
}
```

<p>Esta visualización permite identificar si existe una relación entre la satisfacción con la democracia y la participación en distintos espacios cívicos. La hipótesis clásica de la teoría democrática sugiere que una mayor satisfacción se asocia con mayor participación institucional (partidos, JAC), mientras que la insatisfacción podría canalizarse hacia formas de participación de protesta (manifestaciones). El selector de año permite observar si estos patrones se mantienen o cambian a lo largo del tiempo.

En 2025, el análisis de la participación ciudadana según el nivel de satisfacción democrática, en cuanto a la participación en organizaciones religiosas (cp6), los niveles son elevados y prácticamente uniformes en todos los grupos de satisfacción, oscilando entre 55.4% y 57.5%, lo que indica que este espacio de participación no está mediado por la valoración del sistema político sino por factores culturales o comunitarios independientes. La participación en partidos políticos (cp13) muestra una relación levemente positiva con la satisfacción: los más satisfechos participan en un 20.0% frente al 15.5% de los muy insatisfechos, lo que sugiere que la vinculación con los canales institucionales de representación sí guarda cierta coherencia con la valoración de la democracia. La participación en Juntas de Acción Comunal (colcp8a) es prácticamente idéntica entre grupos (entre 24.2% y 27.2%), lo que refleja que este espacio de participación comunitaria responde más a dinámicas territoriales que a posiciones político-ideológicas. El resultado más llamativo lo ofrece la participación en manifestaciones o protestas (prot3): quienes están "Muy satisfechos" participaron en un 12.5%, porcentaje que desciende progresivamente hasta 5.7% entre los "Muy insatisfechos". Este hallazgo, contraintuitivo frente a la hipótesis de la protesta como canal de la insatisfacción, podría explicarse porque quienes participan activamente en manifestaciones también son ciudadanos más comprometidos políticamente en general, o porque la protesta en Colombia no está necesariamente asociada a descontento sino a culturas de movilización más amplias. En conjunto, los datos sugieren que la participación ciudadana en Colombia es multidimensional y no se reduce a una lógica de satisfacción-desafección, sino que opera a través de diferentes canales con lógicas propias.</p>

---

### Conclusiones y Apreciaciones finales

<p>Los hallazgos del Observatorio de Paz y Seguridad de Colombia configuran un retrato complejo y multidimensional de la seguridad ciudadana y la percepción institucional durante las dos últimas décadas. En conjunto, los datos permiten identificar una narrativa de mejora objetiva en materia de victimización que coexiste, de manera paradójica, con un deterioro sostenido en la percepción que los ciudadanos tienen del Estado y sus instituciones.

En el plano de la seguridad directa, el período 2004–2018 muestra una reducción real y significativa de la victimización comparable, que descendió de niveles cercanos al 9% en 2004 a alrededor del 3.5% en 2018. Esta mejora, aunque generalizada, no fue territorialmente homogénea: mientras la región Andina y la Orinoquía-Amazonía convergieron hacia niveles bajos, el Caribe y la Pacífica mantuvieron índices sistemáticamente más altos, evidenciando que la seguridad sigue siendo un fenómeno profundamente desigual en el país. En cuanto a la tipología del delito, se observa un desplazamiento desde formas más visibles de violencia hacia modalidades más difusas como el fraude y la estafa, lo que sugiere una transformación en los patrones delictivos antes que una erradicación del problema.

El contexto del conflicto armado añade una dimensión estructural de largo plazo que los indicadores de victimización cotidiana no capturan por sí solos. La proporción de hogares afectados por pérdidas de familiares, desplazamiento forzado o emigración por razones del conflicto se mantuvo persistentemente alta durante todo el período analizado, con niveles que superan el 20% en varias de estas dimensiones. Esto evidencia que las consecuencias del conflicto armado en Colombia trascienden los períodos de mayor intensidad bélica y se consolidan como una herida estructural en el tejido social, cuyo impacto acumulado sigue moldeando la experiencia cotidiana de seguridad de la población.

En el plano institucional, los datos revelan la tensión más profunda del período analizado: a pesar de las mejoras en seguridad objetiva, la satisfacción con la democracia colapsó a partir de 2013 y alcanzó su punto crítico en 2020, cuando más del 80% de los encuestados se declaró insatisfecho con el sistema político. Este deterioro no es superficial, sino que se ancla en percepciones estructurales de desconexión: los ciudadanos, independientemente de su nivel de satisfacción democrática, perciben que los gobernantes no se interesan genuinamente por sus necesidades, como lo refleja el hecho de que eff1 se mantenga consistentemente por debajo del punto medio de la escala en todos los grupos. La combinación de ciudadanos que sienten que entienden la política pero que no confían en que el sistema responda a sus intereses configura un escenario de frustración democrática latente, con potencial para canalizarse en formas de movilización o desafección cívica.

Finalmente, la percepción de corrupción política, que se mantuvo elevada y creciente entre 2008 y 2014 y que siguió siendo mayoritariamente negativa en los años posteriores, actúa como denominador común que conecta todos los hallazgos: la inseguridad, el conflicto no resuelto y la desconfianza institucional se retroalimentan en un entorno donde la ciudadanía percibe que el sistema político está capturado por intereses ajenos al bien común. En este sentido, los datos del Observatorio no solo documentan tendencias, sino que señalan con claridad que los avances en seguridad física serán insuficientes para consolidar una paz duradera mientras no se aborden simultáneamente las brechas territoriales, la legitimidad institucional y la percepción de integridad en el ejercicio del poder político.</p>
</div>