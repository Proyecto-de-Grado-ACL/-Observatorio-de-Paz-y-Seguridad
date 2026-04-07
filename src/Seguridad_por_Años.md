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
  // exc07 es escala de texto — se preserva tal cual
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
    domain: [0, 50],
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

```js
// Tabla: encuestados y víctimas por región y año
const tablaRegion = YEARS_VICTIMA.flatMap(year => {
  const sub = dataCore.filter(d => d.year === year && filtroGlobal(d) && d.estratopri != null);
  const byRegion = d3.rollups(
    sub,
    v => {
      const total    = v.length;
      const victimas = v.filter(d => d.victima_total === 1).length;
      return { total, victimas };
    },
    d => d.estratopri
  );
  return byRegion.map(([reg, { total, victimas }]) => ({
    Año: year,
    Región: reg,
    Encuestados: total,
    Víctimas: victimas,
    "% víctimas": total > 0 ? ((victimas / total) * 100).toFixed(1) + "%" : "N/D"
  }));
}).sort((a, b) => a.Año - b.Año || a.Región.localeCompare(b.Región));
```

```js
Inputs.table(tablaRegion, {
  columns: ["Año", "Región", "Encuestados", "Víctimas", "% víctimas"],
  sort: "Año",
  reverse: false,
  width: {
    Año: 80,
    Región: 180,
    Encuestados: 120,
    Víctimas: 100,
    "% víctimas": 110
  }
})
```
<p>El análisis territorial permite identificar heterogeneidad en la victimización, tanto por región geográfica como por tipo de zona (urbana vs. rural). Esto es clave para entender que la seguridad no evoluciona de manera uniforme en el país. Las regiones presentan trayectorias diferenciadas y volátiles. Algunas, como Pacífica y Bogotá D.C., muestran picos pronunciados en ciertos años, mientras que otras como Andina evidencian una tendencia más estable o decreciente.

En general, existe alta variabilidad interanual, lo que indica sensibilidad a contextos locales, No hay una convergencia clara entre regiones, lo que sugiere desigualdad territorial persistente en materia de seguridad y hacia los años finales, varias regiones tienden a niveles más cercanos entre sí, aunque sin eliminar completamente las brechas.
</p>
---

```js
// Selector de región para la gráfica de regiones
const selRegionChart = Inputs.select(
  ["Todas", ...([...new Set(dataCore.map(d => d.estratopri).filter(v => v != null && v !== ""))].sort())],
  { label: "Filtrar región", value: "Todas" }
);
```

```js
const regionChart = Generators.input(selRegionChart);
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

const serieRegionFiltrada = regionChart === "Todas"
  ? serieRegion
  : serieRegion.filter(d => d.category === regionChart);
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
      Plot.text(
        serieRegionFiltrada.filter(d => d.year === ultimoAnoRegion),
        {
          x: "year",
          y: "pct",
          text: d => `${d.category} ${d.pct.toFixed(1)}%`,
          dx: 8,
          textAnchor: "start",
          fontSize: 11,
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

// FIX: categorías tomadas directamente del CSV, preservando el texto original
const EXC07_CATS = [
  "Ninguno",
  "Nada generalizada",
  "Menos de la mitad",
  "La mitad de los políticos",
  "Más de la mitad",
  "Poco generalizada",
  "Algo generalizada",
  "Muy generalizada"
];
```

```js
// Selectores Sección 2
const selDelitoLinea = Inputs.select(
  ["Todas", ...DELITOS_DEF.map(d => d.label)],
  { label: "Filtrar delito", value: "Todas" }
);

const selExc07Linea = Inputs.select(
  ["Todas", ...EXC07_CATS],
  { label: "Filtrar categoría corrupción", value: "Todas" }
);
```

```js
const delitoLinea = Generators.input(selDelitoLinea);
const exc07Linea  = Generators.input(selExc07Linea);
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

const serieDelitosFiltrada = delitoLinea === "Todas"
  ? serieDelitos
  : serieDelitos.filter(d => d.categoria === delitoLinea);
```

```js
// FIX CLAVE: exc07 ahora está guardado como string en data, así que este filtro funciona
const serieExc07 = YEARS_EXC07.flatMap(year => {
  const sub = data.filter(d =>
    d.year === year &&
    filtroGlobal(d) &&
    d.exc07 !== null &&
    d.exc07 !== ""
  );
  if (!sub.length) return [];
  const total = sub.length;
  const grupos = d3.rollups(sub, v => v.length, d => d.exc07);
  return grupos.map(([cat, cnt]) => ({
    year,
    categoria: cat,
    pct: (cnt / total) * 100
  }));
});

const serieExc07Filtrada = exc07Linea === "Todas"
  ? serieExc07
  : serieExc07.filter(d => d.categoria === exc07Linea);
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

  ${selExc07Linea}

```js
Plot.plot({
  title: "Percepción de corrupción entre políticos por año (exc07)",
  subtitle: "% de respuestas por categoría",
  width,
  height: 360,
  x: {
    label: "Año",
    tickFormat: d => String(d),
    tickValues: YEARS_EXC07,
    grid: true
  },
  y: { label: "%", domain: [0, 70], grid: true },
  color: {
    domain: EXC07_CATS,
    scheme: "RdYlGn",
    legend: true
  },
  marks: [
    Plot.ruleY([0]),
    Plot.lineY(serieExc07Filtrada, {
      x: "year", y: "pct", z: "categoria",
      stroke: "categoria", strokeWidth: 2, tip: true
    }),
    Plot.dotY(serieExc07Filtrada, {
      x: "year", y: "pct", fill: "categoria", r: 4
    }),
    Plot.text(serieExc07Filtrada, {
      x: "year",
      y: "pct",
      text: d => d.pct.toFixed(2) + "%",
      dy: -10,
      fontSize: 10,
      fill: "white",
      stroke: "none"
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
<p>En el caso de la satisfacción con la democracia, los resultados muestran un deterioro progresivo a lo largo del tiempo. Durante los primeros años analizados (2004–2012), predominan las categorías de satisfacción, con más del 50% de los encuestados ubicándose como “satisfechos” y una proporción relativamente baja de insatisfacción extrema. Sin embargo, a partir de 2013 se observa un punto de quiebre, donde aumentan de manera significativa las categorías de insatisfacción. Este cambio se consolida en años posteriores, particularmente en 2020, donde la proporción de personas “muy insatisfechas” alcanza su nivel más alto, y las categorías negativas superan ampliamente a las positivas. Aunque en los años más recientes se evidencia una leve recuperación, la insatisfacción continúa siendo predominante, lo que refleja un debilitamiento sostenido en la percepción del funcionamiento democrático.</p>

<div class="aviso-nd">
  eff1 y eff2 no disponibles en 2004–2008 ni en 2021. La línea punteada marca el punto medio de la escala (4).
</div>

```js
  {
  const chartEff1 = Plot.plot({
    title: "Los gobernantes se interesan por la gente (eff1) — media escala 1–7",
    width: 500,
    height: 340,
    x: {
      label: "Año",
      tickFormat: d => String(d),
      tickValues: YEARS_EFF,
      grid: true
    },
    y: {
      label: "Media",
      domain: [1, 7],
      grid: true
    },
    marks: [
      Plot.ruleY([4], { strokeDasharray: "4,3" }),
      Plot.lineY(serieEff1, { x: "year", y: "media", stroke: "#8b5cf6", strokeWidth: 2.5 }),
      Plot.dotY(serieEff1, { x: "year", y: "media", r: 5 }),
      Plot.text(serieEff1, {
        x: "year",
        y: "media",
        text: d => d.media.toFixed(2),
        dy: -12,
        fill: "white",
        fontSize: 10
      })
    ]
  });

  const chartEff2 = Plot.plot({
    title: "Comprensión de asuntos políticos (eff2) — media escala 1–7",
    width: 500,
    height: 340,
    x: {
      label: "Año",
      tickFormat: d => String(d),
      tickValues: YEARS_EFF,
      grid: true
    },
    y: {
      label: "Media",
      domain: [1, 7],
      grid: true
    },
    marks: [
      Plot.ruleY([4], { strokeDasharray: "4,3" }),
      Plot.lineY(serieEff2, { x: "year", y: "media", stroke: "#06b6d4", strokeWidth: 2.5 }),
      Plot.dotY(serieEff2, { x: "year", y: "media", r: 5 }),
      Plot.text(serieEff2, {
        x: "year",
        y: "media",
        text: d => d.media.toFixed(2),
        dy: -12,
        fill: "white",
        fontSize: 10
      })
    ]
  });
  chartEff1.style.width = "48%";
  chartEff2.style.width = "48%";
  const container = document.createElement("div");
  container.style.display = "flex";
  container.style.gap = "24px";
  container.style.flexWrap = "wrap"; // importante para responsive

  container.appendChild(chartEff1);
  container.appendChild(chartEff2);

  display(container);
}
```
<p>Por otro lado, la percepción sobre el interés de los gobernantes por la ciudadanía (eff1) se mantiene consistentemente por debajo del punto medio de la escala (4), lo que indica una percepción generalizada de desconexión entre el Estado y la población. A lo largo del periodo analizado, este indicador presenta una caída inicial hasta 2014, seguida de una recuperación parcial en 2018, pero sin alcanzar niveles que sugieran una valoración positiva. En los años más recientes, la tendencia vuelve a estabilizarse en niveles bajos, reforzando la idea de una confianza limitada en la capacidad de los gobernantes para representar los intereses ciudadanos.

En contraste, la autopercepción de comprensión de los asuntos políticos (eff2) se mantiene relativamente estable y cercana al punto medio de la escala, con valores ligeramente superiores a los observados en eff1. Esto sugiere que, aunque los ciudadanos no perciben a los gobernantes como cercanos o receptivos, sí consideran tener cierto nivel de entendimiento sobre la política. No obstante, esta comprensión no se traduce necesariamente en mayor confianza institucional, lo que evidencia una brecha entre conocimiento político y legitimidad del sistema.

En conjunto, los resultados de esta sección reflejan un escenario de debilitamiento en la confianza institucional, caracterizado por una creciente insatisfacción con la democracia y una percepción persistente de desconexión entre gobernantes y ciudadanía. Aunque los niveles de comprensión política se mantienen relativamente estables, estos no parecen ser suficientes para contrarrestar la percepción negativa del funcionamiento institucional, lo que puede tener implicaciones importantes en términos de participación, legitimidad y gobernabilidad.</p>

---
## Sección 5: Participación Social
---
<p>
Mide la frecuencia de participación en espacios comunitarios y cívicos: organizaciones religiosas (cp6), partidos políticos (cp13), Juntas de Acción Comunal (colcp8a) y manifestaciones públicas (prot3). Las preguntas de frecuencia usan la escala: Nunca / Una o dos veces al año / Una o dos veces al mes / Una vez a la semana.
</p>

```js
  const YEARS_CP6_CP13 = [2004,2005,2006,2007,2008,2009,2010,2011,2012,2013,2014,2016,2018,2020,2023,2025];
  const YEARS_COLCP8A  = [2004,2005,2006,2007,2008,2014,2016,2018,2020,2025];
  const YEARS_PROT3    = [2010,2011,2012,2013,2014,2016,2018,2020,2021,2025];

  const FREQ_CATS = [
    "Nunca",
    "Una o dos veces al año",
    "Una o dos veces al mes",
    "Una vez a la semana"
  ];

  // FIX: usar array de números directamente, NO .map(String)
  const YEARS_PART = [...new Set([
    ...YEARS_CP6_CP13,
    ...YEARS_COLCP8A
  ])].sort((a, b) => a - b);

  // FIX: selector con números, no strings
  const selYearPart = Inputs.select(YEARS_PART, {
    label: "Año",
    value: 2018
  });
```

```js
  // FIX: celda reactiva separada — ya devuelve número porque el selector usa números
  const yearPart = Generators.input(selYearPart);
```

```js
  function distribucionFrecuencia(variable, year, yearsValidos) {
    if (!yearsValidos.includes(year)) return [];

    const sub = data.filter(d =>
      d.year === year &&
      filtroGlobal(d) &&
      d[variable] !== null &&
      FREQ_CATS.includes(d[variable])
    );

    if (!sub.length) return [];

    const total = sub.length;

    return FREQ_CATS.map(cat => {
      const count = sub.filter(d => d[variable] === cat).length;
      return {
        categoria: cat,
        pct: (count / total) * 100
      };
    });
  }

  const distCp6    = distribucionFrecuencia("cp6", yearPart, YEARS_CP6_CP13);
  const distCp13   = distribucionFrecuencia("cp13", yearPart, YEARS_CP6_CP13);
  const distColcp8 = distribucionFrecuencia("colcp8a", yearPart, YEARS_COLCP8A);

  const serieProt3 = YEARS_PROT3.map(year => {
  const sub = data.filter(d => d.year === year && filtroGlobal(d));
  const pct = pctSi(sub, "prot3");
  return pct !== null ? { year, pct } : null;
  }).filter(Boolean);
```
<div class="aviso-nd">
  cp6 y cp13: disponibles en 2004–2014, 2016, 2018, 2020, 2023 y 2025. colcp8a: disponible en 2004–2008, 2014, 2016, 2018, 2020 y 2025. prot3: disponible en 2010–2014, 2016, 2018, 2020, 2021 y 2025.
</div>

<div style="margin: 16px 0 8px 0;">
  ${selYearPart}
</div>

```js
  {
    const chartCp6 = distCp6.length
      ? Plot.plot({
          title: `Participación en organizaciones religiosas (cp6) — ${yearPart}`,
          width: Math.floor(width / 2) - 12,
          height: 300,
          marginLeft: 170,
          x: { label: "%", domain: [0, 100], grid: true },
          y: { label: null },
          color: {
            domain: FREQ_CATS,
            range: ["#dc2626", "#f59e0b", "#60a5fa", "#16a34a"],
            legend: false
          },
          marks: [
            Plot.barX(distCp6, {
              x: "pct",
              y: "categoria",
              fill: "categoria",
              tip: true
            }),
            Plot.text(distCp6, {
              x: "pct",
              y: "categoria",
              text: d => d.pct.toFixed(1) + "%",
              dx: 6,
              textAnchor: "start",
              fontSize: 11,
              fill: "white"
            })
          ]
        })
      : html`<div class="aviso-nd">cp6 no disponible en ${yearPart}</div>`;

    const chartCp13 = distCp13.length
      ? Plot.plot({
          title: `Participación en partidos políticos (cp13) — ${yearPart}`,
          width: Math.floor(width / 2) - 12,
          height: 300,
          marginLeft: 170,
          x: { label: "%", domain: [0, 100], grid: true },
          y: { label: null },
          color: {
            domain: FREQ_CATS,
            range: ["#dc2626", "#f59e0b", "#60a5fa", "#16a34a"],
            legend: false
          },
          marks: [
            Plot.barX(distCp13, {
              x: "pct",
              y: "categoria",
              fill: "categoria",
              tip: true
            }),
            Plot.text(distCp13, {
              x: "pct",
              y: "categoria",
              text: d => d.pct.toFixed(1) + "%",
              dx: 6,
              textAnchor: "start",
              fontSize: 11,
              fill: "white"
            })
          ]
        })
      : html`<div class="aviso-nd">cp13 no disponible en ${yearPart}</div>`;

    const div = document.createElement("div");
    div.style.cssText = "display:flex; gap:24px; align-items:flex-start; flex-wrap:wrap;";
    div.appendChild(chartCp6);
    div.appendChild(chartCp13);
    display(div);
  }
```
```js
  distColcp8.length
    ? Plot.plot({
        title: `Participación en Juntas de Acción Comunal (colcp8a) — ${yearPart}`,
        width,
        height: 300,
        marginLeft: 170,
        x: { label: "%", domain: [0, 100], grid: true },
        y: { label: null },
        color: {
          domain: FREQ_CATS,
          range: ["#dc2626", "#f59e0b", "#60a5fa", "#16a34a"],
          legend: false
        },
        marks: [
          Plot.barX(distColcp8, {
            x: "pct",
            y: "categoria",
            fill: "categoria",
            tip: true
          }),
          Plot.text(distColcp8, {
            x: "pct",
            y: "categoria",
            text: d => d.pct.toFixed(1) + "%",
            dx: 6,
            textAnchor: "start",
            fontSize: 11,
            fill: "white"
          })
        ]
      })
    : html`<div class="aviso-nd">colcp8a no disponible en ${yearPart}</div>`
```
```js
  Plot.plot({
    title: "Participación en manifestaciones o protestas públicas — % (prot3)",
    width,
    height: 320,
    x: {
      label: "Año",
      tickFormat: d => String(d),
      tickValues: YEARS_PROT3,
      grid: true
    },
    y: {
      label: "% participó",
      domain: [0, 20],
      grid: true
    },
    marks: [
      Plot.ruleY([0]),
      Plot.areaY(serieProt3, {
        x: "year",
        y: "pct",
        fill: "#9d174d",
        fillOpacity: 0.10
      }),
      Plot.lineY(serieProt3, {
        x: "year",
        y: "pct",
        stroke: "#9d174d",
        strokeWidth: 2.5,
        tip: true
      }),
      Plot.dotY(serieProt3, {
        x: "year",
        y: "pct",
        fill: "white",
        stroke: "#9d174d",
        strokeWidth: 2,
        r: 5
      }),
      Plot.text(serieProt3, {
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
</div>