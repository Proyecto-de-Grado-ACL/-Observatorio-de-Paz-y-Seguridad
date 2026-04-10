---
theme: dashboard
title: Observatorio de Paz y Seguridad – Confianza en la Policía 2025
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

  ## Observatorio de Paz y Seguridad: Panorama general de confianza en la Policía (2025)

---

  <p>Este observatorio de datos presenta un análisis descriptivo de la percepción ciudadana sobre la Policía, con énfasis en los niveles de confianza, legitimidad institucional y evaluación del desempeño. A partir de datos estructurados y visualizaciones interactivas, se busca ofrecer una herramienta que permita explorar cómo varían estas percepciones entre distintos grupos poblacionales y territorios. Este análisis se basa en datos de encuesta estructurados, utilizando estadística descriptiva (promedios, distribuciones y comparaciones) sin inferencia causal.</p>

  <p>El objetivo principal de este informe descriptivo es facilitar la comprensión de patrones y brechas en la confianza hacia la Policía, sin pretender establecer relaciones causales, sino proporcionando evidencia empírica que apoye la toma de decisiones, el análisis académico y la formulación de políticas públicas. A través de filtros dinámicos y múltiples enfoques visuales, se puede identificar diferencias relevantes según variables sociodemográficas y geográficas.</p>
 
 ---

  ## Sección 1: Panorama de Confianza de la Policía a Nível Nacional

---

  <p>Esta sección presenta una visión agregada del nivel de confianza en la Policía, a través de indicadores clave y la distribución de respuestas de los encuestados. Los KPIs permiten identificar rápidamente el nivel promedio de confianza, así como la proporción de percepciones positivas y negativas. La distribución de los niveles de confianza ofrece una lectura más detallada del comportamiento de la variable, evidenciando si las respuestas se concentran en valores extremos o intermedios. En conjunto, estos elementos permiten establecer una línea base clara sobre la percepción general de la ciudadanía, la cual sirve como punto de partida para los análisis posteriores.</p>

  <p>Los filtros de género, etnia y grupo de edad permiten desagregar la información y observar cómo varía la confianza en la Policía entre distintos segmentos de la población. Al seleccionar una categoría específica, todas las visualizaciones del dashboard se actualizan dinámicamente, mostrando únicamente los datos correspondientes a ese grupo.</p>

  <!-- Load data -->
  <!-- 1. histogramas interactivos (filtardos de confianza) -->
  ```js
  import * as Plot from "npm:@observablehq/plot"
  import * as d3 from "npm:d3"
  import * as topojson from "npm:topojson-client"

  const data = await FileAttachment("data/observatorio_datos_2025.csv").csv({typed:true})
  ```

  ```js
  // Filtros — usar view() para que sean reactivos
  const gender = view(Inputs.select(
    ["Todos", ...new Set(data.map(d => d.sexo_std))],
    {label: "Género"}
  ))

  const ethnicity = view(Inputs.select(
    ["Todos", ...new Set(data.map(d => d.etid_clean))],
    {label: "Etnia"}
  ))

  const ageGroup = view(Inputs.select(
    ["Todos", ...new Set(data.map(d => d.age_gender_clean))],
    {label: "Grupo de edad"}
  ))
  ```

  ```js
  const filtered = data.filter(d =>
    (gender === "Todos" || d.sexo_std === gender) &&
    (ethnicity === "Todos" || d.etid_clean === ethnicity) &&
    (ageGroup === "Todos" || d.age_gender_clean === ageGroup)
  )
  ```

  ```js
  // Pesos poblacionales por región — Proyecciones DANE 2025
  const REGION_PESOS = {
    "Andina":             22800000,
    "Bogotá D.C.":         7942867,
    "Caribe":             11500000,
    "Pacífica":            5800000,
    "Orinoquía-Amazonía":  2014000
  }
  const TOTAL_POB = d3.sum(Object.values(REGION_PESOS))

  // Pesos urbano/rural — Censo DANE 2018 actualizado 2025
  const ZONA_PESOS = { "Urbano": 0.77, "Rural": 0.23 }

  // Promedio ponderado por región (reactivo a los filtros — usa filtered)
  const trustByRegionMean = d3.rollups(
    filtered.filter(d => d.trust_num != null && REGION_PESOS[d.estratopri]),
    rows => d3.mean(rows, d => d.trust_num),
    d => d.estratopri
  )
  const meanTrustPonderado = d3.sum(
    trustByRegionMean,
    ([region, mean]) => mean * (REGION_PESOS[region] / TOTAL_POB)
  )

  // Indicadores principales
  const meanTrust    = d3.mean(filtered, d => d.trust_num)
  const nRespondents = filtered.length
  const highTrust    = filtered.filter(d => d.trust_num >= 5).length
  const lowTrust     = filtered.filter(d => d.trust_num <= 3).length
  ```
  Esto es clave porque la confianza institucional no es homogénea: puede diferir significativamente según características sociodemográficas. Por ejemplo, ciertos grupos pueden presentar niveles de confianza más altos o más bajos debido a experiencias, contextos o percepciones diferenciadas frente a la institución. En este sentido, los filtros no solo facilitan la exploración interactiva, sino que también permiten identificar brechas y patrones relevantes, aportando una comprensión más profunda y segmentada de la percepción ciudadana sobre la Policía.

  <div class="grid grid-cols-4">
    <div class="card">
      <h2>Confianza promedio</h2>
      <span class="big">${meanTrust ? meanTrust.toFixed(2) : "NA"} / 7</span>
      <p style="font-size:0.78em;color:var(--theme-foreground-muted);margin:4px 0 0">promedio simple (muestra)</p>
      <p style="font-size:1em;font-weight:600;margin:6px 0 0;color:var(--theme-foreground)">
        ${meanTrustPonderado ? meanTrustPonderado.toFixed(2) : "NA"} / 7
        <span style="font-size:0.7em;font-weight:400;color:var(--theme-foreground-muted)"> ★ ponderado</span>
      </p>
      <p style="font-size:0.72em;color:var(--theme-foreground-muted);margin:2px 0 0">ajustado por peso poblacional regional (DANE 2025)</p>
    </div>
    <div class="card">
      <h2>Personas encuestadas</h2>
      <span class="big">${nRespondents}</span>
    </div>
    <div class="card">
      <h2>Alta confianza (5-7)</h2>
      <span class="big">${highTrust}</span>
    </div>
    <div class="card">
      <h2>Baja confianza (1–3)</h2>
      <span class="big">${lowTrust}</span>
    </div>
  </div>

  ```js
  // Distribución de confianza ciudadana
  function trustDistribution(data, {width} = {}) {
    const clean = data.filter(d => d.trust_num != null && d.trust_num > 0)
    return Plot.plot({
      title: "Distribución de confianza en la Policía",
      width,
      height: 350,
      x: {
        label: "Nivel de confianza",
        tickFormat: d3.format("d"),
        domain: [1, 2, 3, 4, 5, 6, 7]
      },
      y: {
        grid: true,
        label: "Número de personas"
      },
      marks: [
        Plot.barY(clean, Plot.groupX({y: "count"}, {x: "trust_num", fill: "trust_num", tip: true})),
        Plot.text(clean, Plot.groupX({y: "count", text: "count"}, {
          x: "trust_num",
          dy: -8,
          fontSize: 11,
          fontWeight: "500",
          fill: "var(--theme-foreground)",
          textAnchor: "middle"
        })),
        Plot.ruleY([0])
      ]
    })
  }
  ```

  <div class="grid grid-cols-1">
    <div class="card">
      ${resize((width) => trustDistribution(filtered, {width}))}
    </div>
  </div>

  ```js
  // Confianza promedio por grupos sociales
  function trustByGroup(data, variable, label, {width} = {}) {
    const grouped = d3.rollups(
      data,
      v => d3.mean(v, d => d.trust_num),
      d => d[variable]
    ).map(([group, trust]) => ({group, trust}))

    return Plot.plot({
      title: `Confianza promedio por ${label}`,
      width,
      height: 350,
      x: {label},
      y: {
        label: "Confianza promedio",
        domain: [0, 7],
        grid: true
      },
      marks: [
        Plot.barY(grouped, {x: "group", y: "trust", fill: "group", tip: true}),
        Plot.text(grouped, {
          x: "group", y: "trust",
          text: d => d.trust?.toFixed(2) ?? "",
          dy: -8, fontSize: 11, fontWeight: "500",
          fill: "var(--theme-foreground)",
          textAnchor: "middle"
        }),
        Plot.ruleY([0])
      ]
    })
  }
  ```

  <div class="grid grid-cols-3">
    <div class="card">
      ${resize((width) => trustByGroup(filtered, "sexo_std", "Género", {width}))}
    </div>
    <div class="card">
      ${resize((width) => trustByGroup(filtered, "etid_clean", "Etnia", {width}))}
    </div>
    <div class="card">
      ${resize((width) => trustByGroup(filtered, "age_gender_clean", "Grupo de edad", {width}))}
    </div>
  </div>

  <p>La confianza ciudadana en la Policía Nacional de Colombia en 2025 se ubica en un nivel moderado, con un promedio simple de 4.06 sobre 7 y un promedio ponderado por peso poblacional regional de 4.15 sobre 7, lo que indica que al ajustar por la distribución demográfica del país el resultado mejora ligeramente, aunque se mantiene en el rango intermedio de la escala. De los 1.569 encuestados, 629 (40.1%) reportaron alta confianza (niveles 5 a 7) frente a 605 (38.6%) con baja confianza (niveles 1 a 3), una distribución prácticamente simétrica que confirma la ausencia de un consenso claro en favor o en contra de la institución.
  
  La distribución por niveles revela que el valor modal es 4, con 335 personas, lo que refuerza la tendencia hacia evaluaciones neutras o de punto medio. Los niveles extremos de confianza total (7) y desconfianza total (1) también tienen una presencia notable —208 y 168 personas respectivamente— lo que sugiere que, si bien no existe polarización mayoritaria, hay una minoría de ciudadanos con posiciones muy definidas en ambos extremos.

  Al desagregar por variables sociodemográficas emergen diferencias relevantes aunque moderadas. En términos de género, los hombres presentan una confianza promedio ligeramente superior (4.18) frente a las mujeres (4.01), una brecha de 0.17 puntos que, aunque pequeña, es consistente con patrones observados en otros países donde las mujeres tienden a valorar de forma más crítica las instituciones de seguridad. Por etnia, los grupos Mestizo (4.17) y Otra (3.97) presentan los valores extremos, mientras que Blanca (4.09), Indígena (3.80), Mulata (3.83) y Negra (3.84) se ubican por debajo del promedio general, lo que podría reflejar experiencias diferenciadas frente a la Policía según identidad étnica y merece atención desde una perspectiva de equidad institucional. Finalmente, por grupo de edad, las mujeres adultas mayores registran la confianza más alta (4.42), seguidas por las mujeres adultas (4.38), mientras que los hombres adultos jóvenes presentan el nivel más bajo (3.67), un patrón que sugiere que la juventud masculina experimenta o percibe la relación con la Policía de manera más tensa que otros grupos etarios.</p>

  ---
  ### Distribución poblacional de referencia

  <div class="grid grid-cols-2">
    <div class="card">
      <h3 style="margin-bottom:8px">Por región — Colombia 2025</h3>
      <p style="font-size:0.78em;color:var(--theme-foreground-muted);margin-bottom:10px">
        Fuente: Proyecciones de población DANE 2025 con base en Censo 2018
      </p>
      <table style="width:100%;border-collapse:collapse;font-size:0.85em">
        <thead>
          <tr style="border-bottom:1px solid var(--theme-foreground-muted)">
            <th style="text-align:left;padding:4px 8px;font-weight:600">Región</th>
            <th style="text-align:right;padding:4px 8px;font-weight:600">Población 2025</th>
            <th style="text-align:right;padding:4px 8px;font-weight:600">% del total</th>
          </tr>
        </thead>
        <tbody>
          <tr style="border-bottom:1px solid var(--theme-foreground-faintest)">
            <td style="padding:5px 8px">Andina</td>
            <td style="text-align:right;padding:5px 8px">22.800.000</td>
            <td style="text-align:right;padding:5px 8px;font-weight:600">${REGION_PCT["Andina"]}%</td>
          </tr>
          <tr style="border-bottom:1px solid var(--theme-foreground-faintest)">
            <td style="padding:5px 8px">Bogotá D.C.</td>
            <td style="text-align:right;padding:5px 8px">7.942.867</td>
            <td style="text-align:right;padding:5px 8px;font-weight:600">${REGION_PCT["Bogotá D.C."]}%</td>
          </tr>
          <tr style="border-bottom:1px solid var(--theme-foreground-faintest)">
            <td style="padding:5px 8px">Caribe</td>
            <td style="text-align:right;padding:5px 8px">11.500.000</td>
            <td style="text-align:right;padding:5px 8px;font-weight:600">${REGION_PCT["Caribe"]}%</td>
          </tr>
          <tr style="border-bottom:1px solid var(--theme-foreground-faintest)">
            <td style="padding:5px 8px">Pacífica</td>
            <td style="text-align:right;padding:5px 8px">5.800.000</td>
            <td style="text-align:right;padding:5px 8px;font-weight:600">${REGION_PCT["Pacífica"]}%</td>
          </tr>
          <tr>
            <td style="padding:5px 8px">Orinoquía-Amazonía</td>
            <td style="text-align:right;padding:5px 8px">2.014.000</td>
            <td style="text-align:right;padding:5px 8px;font-weight:600">${REGION_PCT["Orinoquía-Amazonía"]}%</td>
          </tr>
        </tbody>
        <tfoot>
          <tr style="border-top:1px solid var(--theme-foreground-muted)">
            <td style="padding:5px 8px;font-weight:600">Total Colombia</td>
            <td style="text-align:right;padding:5px 8px;font-weight:600">50.056.867</td>
            <td style="text-align:right;padding:5px 8px;font-weight:600">100%</td>
          </tr>
        </tfoot>
      </table>
    </div>
    <div class="card">
      <h3 style="margin-bottom:8px">Por zona — Colombia 2025</h3>
      <p style="font-size:0.78em;color:var(--theme-foreground-muted);margin-bottom:10px">
        Fuente: Censo de Población y Vivienda DANE 2018, proyección 2025
      </p>
      <table style="width:100%;border-collapse:collapse;font-size:0.85em">
        <thead>
          <tr style="border-bottom:1px solid var(--theme-foreground-muted)">
            <th style="text-align:left;padding:4px 8px;font-weight:600">Zona</th>
            <th style="text-align:right;padding:4px 8px;font-weight:600">Población aprox.</th>
            <th style="text-align:right;padding:4px 8px;font-weight:600">% del total</th>
          </tr>
        </thead>
        <tbody>
          <tr style="border-bottom:1px solid var(--theme-foreground-faintest)">
            <td style="padding:5px 8px">🔵 Urbano</td>
            <td style="text-align:right;padding:5px 8px">38.544.000</td>
            <td style="text-align:right;padding:5px 8px;font-weight:600;color:#378ADD">77%</td>
          </tr>
          <tr>
            <td style="padding:5px 8px">🟠 Rural</td>
            <td style="text-align:right;padding:5px 8px">11.513.000</td>
            <td style="text-align:right;padding:5px 8px;font-weight:600;color:#D85A30">23%</td>
          </tr>
        </tbody>
        <tfoot>
          <tr style="border-top:1px solid var(--theme-foreground-muted)">
            <td style="padding:5px 8px;font-weight:600">Total Colombia</td>
            <td style="text-align:right;padding:5px 8px;font-weight:600">50.057.000</td>
            <td style="text-align:right;padding:5px 8px;font-weight:600">100%</td>
          </tr>
        </tfoot>
      </table>
      <p style="font-size:0.78em;color:var(--theme-foreground-muted);margin-top:12px">
        Estos pesos se aplican al cálculo del promedio ponderado de confianza en los KPIs superiores.
      </p>
    </div>
  </div>

  Esta sección presenta la distribución poblacional de referencia utilizada para el cálculo de los indicadores, desagregada por región y tipo de zona (urbano y rural). Se observa que la mayor concentración de población se encuentra en la región Andina (45.5%), seguida por el Caribe (23.0%) y Bogotá D.C. (15.9%), mientras que regiones como la Orinoquía-Amazonía representan una proporción mucho menor. Asimismo, el país presenta una fuerte concentración urbana, con un 77% de la población viviendo en zonas urbanas frente a un 23% en zonas rurales. Esta distribución es clave para el análisis, ya que se utiliza como base para ponderar los resultados de confianza, permitiendo que los indicadores reflejen de manera más precisa la realidad nacional. En este sentido, los resultados no solo representan las respuestas de la muestra, sino que están ajustados según el peso real de cada región y tipo de zona en la población total, evitando sesgos y mejorando la representatividad del análisis.

  ---

  <!-- 2. mapa de regiones -->

  ### Análisis territorial de la confianza

  A continuación, se explora cómo varía la confianza en la Policía a nivel regional, utilizando una representación geográfica que facilita la identificación de patrones territoriales. El mapa permite visualizar diferencias entre regiones del país, evidenciando posibles desigualdades en la percepción institucional. Este análisis aporta una dimensión espacial clave al observatorio, permitiendo identificar zonas donde la confianza es relativamente alta o baja. Estos hallazgos pueden estar asociados a factores contextuales como condiciones de seguridad, presencia institucional o dinámicas sociales específicas, aunque el dashboard se limita a mostrar patrones descriptivos sin establecer relaciones causales.


  ```js
  // GeoJSON de departamentos de Colombia (fuente: John Guerra / DANE)
  const colombiaGeo = await fetch(
    "https://gist.githubusercontent.com/john-guerra/43c7656821069d00dcbc/raw/be6a6e239cd5b5b803c6e7c2ec405b793a9064dd/Colombia.geo.json"
  ).then(r => r.json())
  ```

  ```js
  // Mapeo de departamentos → región (estratopri)
  const REGION_MAP = {
    // Andina
    "ANTIOQUIA": "Andina",
    "BOYACA": "Andina",
    "CALDAS": "Andina",
    "CUNDINAMARCA": "Andina",
    "HUILA": "Andina",
    "NORTE DE SANTANDER": "Andina",
    "QUINDIO": "Andina",
    "RISARALDA": "Andina",
    "SANTANDER": "Andina",
    "TOLIMA": "Andina",
    // Caribe
    "ATLANTICO": "Caribe",
    "BOLIVAR": "Caribe",
    "CESAR": "Caribe",
    "CORDOBA": "Caribe",
    "LA GUAJIRA": "Caribe",
    "MAGDALENA": "Caribe",
    "SUCRE": "Caribe",
    "SAN ANDRES": "Caribe",
    // Pacífica
    "CAUCA": "Pacífica",
    "CHOCO": "Pacífica",
    "NARIÑO": "Pacífica",
    "VALLE DEL CAUCA": "Pacífica",
    // Orinoquía-Amazonía
    "AMAZONAS": "Orinoquía-Amazonía",
    "ARAUCA": "Orinoquía-Amazonía",
    "CAQUETA": "Orinoquía-Amazonía",
    "CASANARE": "Orinoquía-Amazonía",
    "GUAINIA": "Orinoquía-Amazonía",
    "GUAVIARE": "Orinoquía-Amazonía",
    "META": "Orinoquía-Amazonía",
    "PUTUMAYO": "Orinoquía-Amazonía",
    "VAUPES": "Orinoquía-Amazonía",
    "VICHADA": "Orinoquía-Amazonía",
    // Bogotá D.C.
    "BOGOTA": "Bogotá D.C.",
    "BOGOTÁ": "Bogotá D.C.",
    "BOGOTA D.C.": "Bogotá D.C.",
  }

  // Confianza promedio por región desde los datos
  const trustByRegion = d3.rollups(
    data,
    v => d3.mean(v, d => d.trust_num),
    d => d.estratopri
  )
  const trustMap = new Map(trustByRegion)

  // Añadir región y confianza a cada feature del GeoJSON
  const geoWithTrust = {
    ...colombiaGeo,
    features: colombiaGeo.features.map(f => {
      const dpto = f.properties.NOMBRE_DPT?.toUpperCase().trim()
      const region = REGION_MAP[dpto] ?? null
      const trust = region ? trustMap.get(region) : null
      return {
        ...f,
        properties: {
          ...f.properties,
          region,
          trust
        }
      }
    })
  }
  ```

  ```js
  // Paleta divergente: rojo (baja confianza) → verde (alta confianza)
  // La escala asume trust_num de 1–7
  const trustExtent = d3.extent([...trustMap.values()])

  const colorScale = d3.scaleSequential()
    .domain(trustExtent)
    .interpolator(d3.interpolateRdYlGn)

  // Etiquetas de región con coordenadas aproximadas para anotaciones
  const REGION_CENTROIDS = [
    { region: "Andina",             x: -74.5,  y:  5.5  },
    { region: "Caribe",             x: -75.0,  y: 10.2  },
    { region: "Pacífica",           x: -77.2,  y:  4.5  },
    { region: "Orinoquía-Amazonía", x: -71.5,  y:  2.0  },
    { region: "Bogotá D.C.",        x: -74.08, y:  4.71 },
  ]
  ```

  ```js
  function trustMap_viz({width} = {}) {
    const height = width * 1.1

    // Proyección centrada en Colombia
    const projection = d3.geoMercator()
      .fitSize([width, height], geoWithTrust)

    const path = d3.geoPath(projection)

    // Calcular centroides de regiones para etiquetas
    const regionFeatures = d3.group(geoWithTrust.features, d => d.properties.region)
    const labelData = REGION_CENTROIDS.map(c => {
      const trust = trustMap.get(c.region)
      const [px, py] = projection([c.x, c.y])
      return { ...c, px, py, trust }
    }).filter(d => d.trust != null)

    const svg = d3.create("svg")
      .attr("width", width)
      .attr("height", height)
      .style("font-family", "sans-serif")

    // Fondo
    svg.append("rect")
      .attr("width", width)
      .attr("height", height)
      .attr("fill", "transparent")

    // Departamentos coloreados por región
    svg.selectAll("path")
      .data(geoWithTrust.features)
      .join("path")
      .attr("d", path)
      .attr("fill", d => d.properties.trust ? colorScale(d.properties.trust) : "#ccc")
      .attr("stroke", "white")
      .attr("stroke-width", 0.5)
      .append("title")
      .text(d => {
        const r = d.properties.region ?? "Sin datos"
        const t = d.properties.trust ? d.properties.trust.toFixed(2) : "N/A"
        return `${d.properties.NOMBRE_DPT}\nRegión: ${r}\nConfianza promedio: ${t} / 7`
      })

    // Etiquetas por región
    const labelGroup = svg.append("g")

    labelData.forEach(d => {
      // Fondo de etiqueta
      const g = labelGroup.append("g")
        .attr("transform", `translate(${d.px}, ${d.py})`)

      g.append("rect")
        .attr("x", -52).attr("y", -26)
        .attr("width", 104).attr("height", 36)
        .attr("rx", 6)
        .attr("fill", "rgba(0,0,0,0.65)")

      g.append("text")
        .attr("text-anchor", "middle")
        .attr("y", -10)
        .attr("fill", "white")
        .attr("font-size", d.region === "Bogotá D.C." ? "8px" : "9px")
        .attr("font-weight", "bold")
        .text(d.region)

      g.append("text")
        .attr("text-anchor", "middle")
        .attr("y", 6)
        .attr("fill", colorScale(d.trust))
        .attr("font-size", "11px")
        .attr("font-weight", "bold")
        .text(`${d.trust.toFixed(2)} / 7`)
    })

    // Leyenda de color
    const legendW = Math.min(200, width * 0.4)
    const legendH = 12
    const lx = width - legendW - 16
    const ly = height - 50

    const defs = svg.append("defs")
    const grad = defs.append("linearGradient").attr("id", "trust-grad")
    grad.selectAll("stop")
      .data(d3.range(0, 1.01, 0.1))
      .join("stop")
      .attr("offset", d => `${d * 100}%`)
      .attr("stop-color", d => colorScale(trustExtent[0] + d * (trustExtent[1] - trustExtent[0])))

    const lg = svg.append("g").attr("transform", `translate(${lx},${ly})`)

    lg.append("rect")
      .attr("width", legendW).attr("height", legendH)
      .attr("rx", 3)
      .attr("fill", "url(#trust-grad)")

    lg.append("text")
      .attr("y", legendH + 14).attr("fill", "currentColor")
      .attr("font-size", "10px")
      .text(`${trustExtent[0].toFixed(2)} (baja)`)

    lg.append("text")
      .attr("x", legendW).attr("y", legendH + 14)
      .attr("text-anchor", "end")
      .attr("fill", "currentColor")
      .attr("font-size", "10px")
      .text(`${trustExtent[1].toFixed(2)} (alta)`)

    lg.append("text")
      .attr("x", legendW / 2).attr("y", -6)
      .attr("text-anchor", "middle")
      .attr("fill", "currentColor")
      .attr("font-size", "10px")
      .text("Confianza promedio (escala 1–7)")

    return svg.node()
  }
  ```

  <div class="grid grid-cols-1">
    <div class="card">
      <h2>Confianza en acciones de la Policía por región</h2>
      <p style="font-size:0.85em; color: var(--theme-foreground-muted)">
        escala 1 (desacuerdo total) a 7 (acuerdo total)
      </p>
      ${resize((width) => trustMap_viz({width}))}
    </div>
  </div>

  <p>El análisis territorial de la confianza en la Policía revela diferencias regionales claras que van más allá de la variación estadística. La región Andina registra el nivel más alto con 4.45 sobre 7, siendo la única que supera con claridad el punto medio de la escala, lo que refleja una percepción relativamente favorable de la institución en el centro del país. Le siguen la Orinoquía-Amazonía con 4.10 y la Pacífica con 4.03, ambas en el rango neutro-positivo. En contraste, el Caribe (3.84) y Bogotá D.C. (3.81) se ubican por debajo del punto medio, constituyendo los territorios con mayor déficit de confianza institucional. Este patrón es especialmente crítico desde una perspectiva demográfica: Bogotá D.C. y el Caribe concentran conjuntamente cerca del 39% de la población nacional, lo que significa que la percepción negativa en estas zonas arrastra significativamente el promedio nacional hacia abajo y representa el principal desafío territorial para la institución.</p>

  <!-- 2. barras de porcentuales -->

---

  ## Sección 2: Legitimidad institucional

---

  A continuación se examina la percepción ciudadana sobre la legitimidad de la Policía, enfocándose en cómo se evalúan sus comportamientos, tanto positivos como negativos. A través de estos indicadores, se busca comprender la calidad de la interacción entre la institución y la ciudadanía, así como identificar posibles tensiones en la percepción de su actuación.

  ```js

  const VARS = [
    {
      key: "just1_clean",
      label: "Decisiones justas\nal usar la fuerza",
      type: "positive",
      // Escala: 1=Rara vez, 2=Algunas veces, 3=Frecuentemente, 4=Siempre, 5=Nunca
      positiveNums: [3, 4],
      positiveLabels: ["Frecuentemente", "Siempre"]
    },
    {
      key: "resp1_clean",
      label: "Trato respetuoso\na ciudadanos",
      type: "positive",
      positiveNums: [3, 4],
      positiveLabels: ["Frecuentemente", "Siempre"]
    },
    {
      key: "poley1_clean",
      label: "Solicitan\nsobornos",
      type: "negative",
      // Escala: 1=Nunca, 2=Rara vez, 3=Algunas veces, 4=Frecuentemente, 5=Siempre
      positiveNums: [4, 5],
      positiveLabels: ["Frecuentemente", "Siempre"]
    },
    {
      key: "poley2_clean",
      label: "Violan derechos\nhumanos",
      type: "negative",
      positiveNums: [4, 5],
      positiveLabels: ["Frecuentemente", "Siempre"]
    },
    {
      key: "poley3_clean",
      label: "Protegen\na delincuentes",
      type: "negative",
      positiveNums: [4, 5],
      positiveLabels: ["Frecuentemente", "Siempre"]
    }
  ]

  // Detecta tipo del primer valor válido y calcula % según corresponda
  function calcPct(variable) {
    const valid = data.filter(d => d[variable.key] != null && d[variable.key] !== "")
    if (valid.length === 0) return 0
    const firstVal = valid[0][variable.key]
    const isNumeric = typeof firstVal === "number"
    const pos = isNumeric
      ? valid.filter(d => variable.positiveNums.includes(d[variable.key]))
      : valid.filter(d => variable.positiveLabels.includes(d[variable.key]))
    return (pos.length / valid.length) * 100
  }

  const stats = VARS.map(v => ({ ...v, pct: calcPct(v) }))
  ```
  ```js
  // ─── Gauge semicircular — muestra % que responde Frecuentemente/Siempre ───────
  function gauge(pct, { width = 260 } = {}) {
  const w = width
  const h = w * 0.62
  const cx = w / 2
  const cy = h * 0.88
  const r = w * 0.36
  const strokeW = w * 0.085

  // Color único por intensidad: azul suave → azul fuerte
  const fillColor = d3.interpolateBlues(0.4 + (pct / 100) * 0.55)

  const startAngle = -Math.PI
  const endAngle = 0
  const valueAngle = startAngle + (pct / 100) * (endAngle - startAngle)

  function arcPath(start, end) {
    const x1 = cx + r * Math.cos(start), y1 = cy + r * Math.sin(start)
    const x2 = cx + r * Math.cos(end),   y2 = cy + r * Math.sin(end)
    const large = (end - start) > Math.PI ? 1 : 0
    return `M ${x1} ${y1} A ${r} ${r} 0 ${large} 1 ${x2} ${y2}`
  }

  const nx = cx + r * 0.8 * Math.cos(valueAngle)
  const ny = cy + r * 0.8 * Math.sin(valueAngle)

  const svg = d3.create("svg")
    .attr("width", w)
    .attr("height", h)
    .attr("viewBox", `0 0 ${w} ${h}`)
    .style("overflow", "visible")

  // Track gris
  svg.append("path")
    .attr("d", arcPath(startAngle, endAngle))
    .attr("fill", "none")
    .attr("stroke", "#e5e7eb")
    .attr("stroke-width", strokeW)
    .attr("stroke-linecap", "round")

  // Arco de valor
  svg.append("path")
    .attr("d", arcPath(startAngle, valueAngle))
    .attr("fill", "none")
    .attr("stroke", fillColor)
    .attr("stroke-width", strokeW)
    .attr("stroke-linecap", "round")
    .attr("opacity", 0.92)

  // Aguja
  svg.append("line")
    .attr("x1", cx)
    .attr("y1", cy)
    .attr("x2", nx)
    .attr("y2", ny)
    .attr("stroke", "#374151")
    .attr("stroke-width", w * 0.018)
    .attr("stroke-linecap", "round")

  // Centro
  svg.append("circle")
    .attr("cx", cx)
    .attr("cy", cy)
    .attr("r", strokeW * 0.45)
    .attr("fill", fillColor)

  // Valor % — más arriba para que no tape la manecilla
  svg.append("text")
    .attr("x", cx)
    .attr("y", cy - r * 1.25)
    .attr("text-anchor", "middle")
    .attr("font-size", `${w * 0.13}px`)
    .attr("font-weight", "bold")
    .attr("fill", fillColor)
    .text(`${pct.toFixed(1)}%`)

  // Min / Max
  svg.append("text")
    .attr("x", cx - r - strokeW * 0.5)
    .attr("y", cy + 16)
    .attr("text-anchor", "middle")
    .attr("font-size", `${w * 0.052}px`)
    .attr("fill", "#9ca3af")
    .text("0%")

  svg.append("text")
    .attr("x", cx + r + strokeW * 0.5)
    .attr("y", cy + 16)
    .attr("text-anchor", "middle")
    .attr("font-size", `${w * 0.052}px`)
    .attr("fill", "#9ca3af")
    .text("100%")

  return svg.node()
}
  ```

  ```js
  // ─── Lollipop comparativo ─────────────────────────────────────────────────────
  function lollipop({ width } = {}) {
    const colorFn = d => d.type === "positive"
      ? (d.pct >= 60 ? "#22c55e" : d.pct >= 35 ? "#eab308" : "#ef4444")
      : (d.pct >= 40 ? "#ef4444" : d.pct >= 20 ? "#f97316" : "#22c55e")

    return Plot.plot({
      title: "Comparativo de indicadores de legitimidad",
      width,
      height: 320,
      marginLeft: 140,
      marginRight: 60,
      x: {
        label: "% respuestas Frecuentemente + Siempre →",
        domain: [0, 100],
        grid: true
      },
      y: {
        label: null,
        domain: stats.map(d => d.label)
      },
      marks: [
        Plot.ruleX([0]),
        Plot.link(stats, {
          x1: 0, x2: "pct",
          y1: "label", y2: "label",
          stroke: d => colorFn(d),
          strokeWidth: 2.5,
          strokeOpacity: 0.5
        }),
        Plot.dot(stats, {
          x: "pct", y: "label",
          r: 10,
          fill: d => colorFn(d),
          tip: true,
          title: d => `${d.label.replace("\n"," ")}\n${d.pct.toFixed(1)}% — ${d.type === "positive" ? "positivo" : "problema"}`
        }),
        Plot.text(stats, {
          x: "pct", y: "label",
          text: d => `${d.pct.toFixed(1)}%`,
          dx: 18, fontSize: 11, fontWeight: "bold",
          fill: d => colorFn(d)
        })
      ]
    })
  }
  ```

  ---

  ### Percepción de comportamiento policial — frecuencia percibida

  <div class="grid grid-cols-2">
    <div class="card" style="text-align:center">
      <h3>Decisiones justas al usar la fuerza</h3>
      <p style="font-size:0.8em;color:var(--theme-foreground-muted)">
        ¿Qué tan frecuente es que los policías tomen decisiones justas al usar la fuerza?
      </p>
      ${gauge(stats[0].pct, { width: 280 })}
    </div>
    <div class="card" style="text-align:center">
      <h3>Trato respetuoso a ciudadanos</h3>
      <p style="font-size:0.8em;color:var(--theme-foreground-muted)">
        ¿Qué tan frecuente es que los policías traten con respeto a las personas?
      </p>
      ${gauge(stats[1].pct, { width: 280 })}
    </div>
  </div>
  
  <p>Los dos indicadores positivos de legitimidad muestran niveles diferenciados que revelan una asimetría importante en la percepción ciudadana. El trato respetuoso a ciudadanos alcanza el 41.2%, lo que indica que cerca de cuatro de cada diez encuestados considera que esta conducta ocurre con frecuencia o siempre, posicionándose como la fortaleza relativa más clara de la institución en términos de interacción cotidiana. Sin embargo, las decisiones justas al usar la fuerza solo alcanzan el 25.2%, es decir, apenas uno de cada cuatro ciudadanos percibe que el uso de la fuerza es ejercido con justicia de manera frecuente. Esta brecha de 16 puntos porcentuales entre ambos indicadores es analíticamente relevante: sugiere que la Policía puede ser percibida como amable en el trato superficial pero no como justa en el ejercicio de su autoridad más crítica, una distinción que en la literatura sobre legitimidad policial se asocia directamente con la legitimidad procedimental y que tiene mayor peso que el trato cortés en la construcción de confianza duradera.</p>

  <div class="grid grid-cols-3">
    <div class="card" style="text-align:center">
      <h3>Solicitan sobornos</h3>
      <p style="font-size:0.8em;color:var(--theme-foreground-muted)">
        ¿Qué tan frecuente es que los policías pidan sobornos?
      </p>
      ${gauge(stats[2].pct, { width: 240 })}
    </div>
    <div class="card" style="text-align:center">
      <h3>Violan derechos humanos</h3>
      <p style="font-size:0.8em;color:var(--theme-foreground-muted)">
        ¿Qué tan frecuente es que los policías violen derechos humanos?
      </p>
      ${gauge(stats[3].pct, { width: 240 })}
    </div>
    <div class="card" style="text-align:center">
      <h3>Protegen a delincuentes</h3>
      <p style="font-size:0.8em;color:var(--theme-foreground-muted)">
        ¿Qué tan frecuente es que los policías protejan a delincuentes?
      </p>
      ${gauge(stats[4].pct, { width: 240 })}
    </div>
  </div>

  <p>Los tres indicadores negativos presentan porcentajes que, si bien son minoritarios en términos absolutos, no pueden considerarse marginales dado el peso simbólico de las conductas que miden. La percepción de que los policías protegen a delincuentes de forma frecuente o siempre alcanza el 15.2%, siendo el indicador negativo más alto de la sección. Le siguen la solicitud de sobornos con 12.3% y la violación de derechos humanos con 10.8%. Estos valores implican que entre uno de cada siete y uno de cada nueve ciudadanos percibe estas prácticas como recurrentes dentro de la institución, lo cual constituye una señal de alerta importante. Aunque ninguno de los tres supera el umbral del 20%, su persistencia como percepciones instaladas en la ciudadanía limita estructuralmente la capacidad de la Policía de consolidar legitimidad, ya que la desconfianza asociada a la corrupción y el abuso de autoridad tiende a ser resistente al cambio y a propagarse socialmente con mayor facilidad que las percepciones positivas.</p>

  ---

  ### Vista comparativa de los 5 indicadores

  Este gráfico presenta una comparación directa de los cinco indicadores de legitimidad institucional, diferenciando entre comportamientos positivos y negativos de la Policía. Cada punto representa el porcentaje de personas que perciben que una conducta ocurre con frecuencia, lo que permite visualizar de forma clara la magnitud relativa de cada indicador. Es importante tener en cuenta que, para los indicadores positivos, valores más altos reflejan una mejor percepción, mientras que en los indicadores negativos un mayor porcentaje indica una mayor presencia de problemas percibidos. De esta manera, el gráfico facilita la comparación simultánea de distintas dimensiones del comportamiento institucional.

  <div class="grid grid-cols-1">
    <div class="card">
      <p style="font-size:0.82em;color:var(--theme-foreground-muted)">
        Para indicadores <strong>positivos</strong> (just1, resp1): mayor % = mejor percepción.<br>
        Para indicadores <strong>negativos</strong> (poley1–3): mayor % = mayor problema percibido.
      </p>
      ${resize((width) => lollipop({ width }))}
    </div>
  </div>

  <p>La vista comparativa de los cinco indicadores sintetiza de forma clara el perfil de legitimidad institucional percibida en 2025. El trato respetuoso (41.2%) se posiciona como el indicador más favorable, seguido a distancia por las decisiones justas al usar la fuerza (25.2%), que a pesar de ser el segundo indicador positivo aparece en rojo en la visualización, lo que señala que su nivel es insuficiente para ser considerado una fortaleza. En el lado negativo, los tres indicadores se ubican en la zona verde del gráfico —lo que en este caso indica baja prevalencia percibida del problema— con protección a delincuentes (15.2%) como el más preocupante, seguido por solicitud de sobornos (12.3%) y violación de derechos humanos (10.8%). En conjunto, el perfil resultante describe una institución que es percibida como razonablemente respetuosa en el trato cotidiano, pero que enfrenta dudas significativas sobre la justicia de sus decisiones más críticas y mantiene una presencia no despreciable de percepciones de corrupción e irregularidades, lo que configura una legitimidad parcial y frágil que requiere intervenciones focalizadas en las dimensiones procedimentales del ejercicio policial.</p>
  ---

  ## Sección 3: Desempeño y efectividad policial

  ---

  En esta sección se analiza la percepción ciudadana sobre el desempeño y la efectividad de la Policía, evaluando distintas funciones operativas y su comportamiento en el territorio. A través de múltiples visualizaciones, se busca identificar cómo varía la evaluación del desempeño institucional según el contexto y qué tan consistente es esta percepción entre regiones y tipos de zona.

  ---

  ```js
  // ── Configuración de variables de desempeño ───────────────────────────────────
  const PERF_VARS = [
    { key: "pr5_num",    label: "Investiga delitos (pr5)"          },
    { key: "effec1_num", label: "Controla tráfico de drogas (effec1)" },
    { key: "effec2_num", label: "Mantiene seguridad (effec2)"      }
  ]

  const REGIONES = ["Andina", "Caribe", "Pacífica", "Orinoquía-Amazonía", "Bogotá D.C."]
  const ZONA     = ["Urbano", "Rural"]

  const VAR_COLORS = ["#378ADD", "#1D9E75", "#BA7517"]
  ```

  ```js
  // ── Datos agrupados ───────────────────────────────────────────────────────────

  // 1. Promedio por región × variable (para heatmap)
  const byRegionVar = []
  for (const v of PERF_VARS) {
    const rolled = d3.rollups(
      data.filter(d => REGIONES.includes(d.estratopri) && d[v.key] != null && d[v.key] !== ""),
      rows => d3.mean(rows, d => +d[v.key]),
      d => d.estratopri
    )
    for (const [region, mean] of rolled)
      byRegionVar.push({ region, variable: v.label, mean })
  }

  // 2. Promedio por zona × variable (para dot plot)
  const byUrbVar = []
  for (const v of PERF_VARS) {
    const rolled = d3.rollups(
      data.filter(d => ZONA.includes(d.ur) && d[v.key] != null && d[v.key] !== ""),
      rows => d3.mean(rows, d => +d[v.key]),
      d => d.ur
    )
    for (const [zona, mean] of rolled)
      byUrbVar.push({ zona, variable: v.label, mean })
  }

  ```

  ```js
  // ── Pesos y etiquetas con % poblacional ──────────────────────────────────────
  // Reutiliza REGION_PESOS y TOTAL_POB definidos arriba
  const REGION_PCT = Object.fromEntries(
    Object.entries(REGION_PESOS).map(([r, p]) => [r, ((p / TOTAL_POB) * 100).toFixed(1)])
  )
  // Etiquetas enriquecidas para eje Y del heatmap
  const REGIONES_LABEL = REGIONES.map(r => `${r} (${REGION_PCT[r]}%)`)
  const REGION_TO_LABEL = Object.fromEntries(REGIONES.map(r => [r, `${r} (${REGION_PCT[r]}%)`]))

  // Datos del heatmap con etiqueta enriquecida
  const byRegionVarLabeled = byRegionVar.map(d => ({
    ...d,
    regionLabel: REGION_TO_LABEL[d.region] ?? d.region
  }))
  ```

  ### Mapa de calor: región × variable

  Este mapa de calor muestra la percepción de efectividad de la Policía en distintas funciones como la investigación de delitos, el control del tráfico de drogas y el mantenimiento de la seguridad desagregada por región. Cada celda representa el valor promedio en una escala de 1 a 7, donde los colores permiten identificar rápidamente niveles relativos de desempeño: tonos más verdes indican una mejor percepción, mientras que tonos más cercanos al rojo reflejan evaluaciones más bajas. Esta visualización facilita la comparación simultánea entre regiones y variables, permitiendo identificar patrones territoriales en la percepción de la efectividad institucional.

  ```js
  // Gráfico 2 — Heatmap
  function heatmap({ width } = {}) {
    return Plot.plot({
      width,
      height: 230,
      marginLeft: 220,
      marginRight: 100,
      marginTop: 20,
      marginBottom: 55,
      x: { label: null, domain: PERF_VARS.map(v => v.label) },
      y: { label: null, domain: REGIONES_LABEL },
      color: {
        type: "sequential",
        scheme: "RdYlGn",
        domain: [1, 7],
        legend: true,
        label: "Promedio (1–7)"
      },
      marks: [
        Plot.cell(byRegionVarLabeled, {
          x: "variable", y: "regionLabel", fill: "mean",
          tip: true,
          title: d => `${d.region}\n${d.variable}\nPromedio: ${d.mean?.toFixed(2)}`
        }),
        Plot.text(byRegionVarLabeled, {
          x: "variable", y: "regionLabel",
          text: d => d.mean?.toFixed(1) ?? "",
          fill: d => (d.mean ?? 4) > 5 ? "#174504" : (d.mean ?? 4) < 3 ? "#FAEEDA" : "#2C2C2A",
          fontSize: 13, fontWeight: "500"
        })
      ]
    })
  }
  ```

  <div class="grid grid-cols-1">
    <div class="card">
      <p style="font-size:0.82em;color:var(--theme-foreground-muted)">
        Verde = alta percepción de efectividad · Rojo = baja percepción. El número en cada celda es el promedio exacto.
      </p>
      ${resize(width => heatmap({ width }))}
    </div>
  </div>

  <p>El mapa de calor de efectividad policial por región y función revela dos patrones transversales claros. Primero, el control del tráfico de drogas (effec1) es sistemáticamente la función peor evaluada en todas las regiones, con promedios que van desde 3.3 en Bogotá D.C. hasta 4.0 en la Andina, ninguno de los cuales supera el punto medio de la escala de manera contundente. Esto indica una debilidad estructural percibida en esta función específica que no depende del territorio sino de la institución en su conjunto. Segundo, el mantenimiento de la seguridad (effec2) es consistentemente la función mejor valorada, liderada por la Andina con 4.6, seguida por Orinoquía-Amazonía con 4.1 y Pacífica con 3.9. La investigación de delitos (pr5) ocupa una posición intermedia en todas las regiones. A nivel territorial, la región Andina domina en las tres funciones con los valores más altos (4.4, 4.0, 4.6), mientras que Bogotá D.C. presenta los promedios más bajos en las tres dimensiones (3.7, 3.3, 3.7), lo que resulta paradójico dado que es la ciudad con mayor concentración de recursos policiales del país y sugiere que una mayor presencia institucional no se traduce automáticamente en mejor percepción de efectividad.</p>

  ---

  ### Brecha Urbano vs Rural por variable

  Por otra parte, aquí se compara la percepción de efectividad de la Policía entre zonas urbanas y rurales para distintas funciones. Cada fila representa una variable específica y muestra los valores promedio en cada tipo de zona, mientras que los puntos y el triángulo indican qué zona presenta una mejor percepción y la magnitud de la diferencia. Esta visualización permite identificar de manera clara y directa si existen brechas territoriales en la evaluación del desempeño policial.

  ```js
  // Gráfico 3 — Slope chart Urbano vs Rural
  function slopeChart({ width } = {}) {
    const varLabels = PERF_VARS.map(v => v.label)
    const zonaColor = d3.scaleOrdinal().domain(ZONA).range(["#378ADD", "#D85A30"])

    // Pares para las líneas de pendiente
    const pairs = varLabels.map(v => ({
      variable: v,
      urbano: byUrbVar.find(d => d.variable === v && d.zona === "Urbano")?.mean,
      rural:  byUrbVar.find(d => d.variable === v && d.zona === "Rural")?.mean
    })).filter(d => d.urbano != null && d.rural != null)

    // Todos los puntos individuales
    const points = byUrbVar.filter(d => varLabels.includes(d.variable))

    const h = 80 + varLabels.length * 90
    const marginL = 60, marginR = 60, marginT = 40, marginB = 30
    const innerW = width - marginL - marginR
    const xU = marginL + innerW * 0.2   // columna Urbano
    const xR = marginL + innerW * 0.8   // columna Rural
    const yScale = d3.scaleLinear().domain([1, 7]).range([h - marginB, marginT])

    // Una fila por variable con su propio eje Y parcial
    const varYCenter = varLabels.map((v, i) => ({
      v,
      cy: marginT + i * 90 + 30
    }))

    // Para slope chart, mapeamos cada variable a una posición Y fija y el valor al tamaño del punto
    // Usamos un eje Y por variable: cada variable ocupa una banda horizontal
    const bandH = 90
    const bandScale = (i) => marginT + i * bandH + bandH / 2

    const svg = d3.create("svg")
      .attr("width", width).attr("height", h)
      .style("font-family", "sans-serif").style("overflow", "visible")

    // Etiquetas de columna con % poblacional
    svg.append("text").attr("x", xU).attr("y", 16)
      .attr("text-anchor", "middle").attr("font-size", "13px").attr("font-weight", "600")
      .attr("fill", "#378ADD").text("Urbano")
    svg.append("text").attr("x", xU).attr("y", 30)
      .attr("text-anchor", "middle").attr("font-size", "10px").attr("font-weight", "400")
      .attr("fill", "#378ADD").attr("opacity", 0.75).text("(77% pob. nacional)")
    svg.append("text").attr("x", xR).attr("y", 16)
      .attr("text-anchor", "middle").attr("font-size", "13px").attr("font-weight", "600")
      .attr("fill", "#D85A30").text("Rural")
    svg.append("text").attr("x", xR).attr("y", 30)
      .attr("text-anchor", "middle").attr("font-size", "10px").attr("font-weight", "400")
      .attr("fill", "#D85A30").attr("opacity", 0.75).text("(23% pob. nacional)")

    varLabels.forEach((v, i) => {
      const cy   = bandScale(i)
      const pair = pairs.find(p => p.variable === v)
      if (!pair) return

      // Escala local: mapea 1–7 al ancho disponible entre columnas
      const localScale = d3.scaleLinear().domain([1, 7]).range([xU - 30, xR + 30])
      const xUval = localScale(pair.urbano)
      const xRval = localScale(pair.rural)

      // Línea de fondo (referencia de banda)
      svg.append("line")
        .attr("x1", xU - 35).attr("x2", xR + 35)
        .attr("y1", cy).attr("y2", cy)
        .attr("stroke", "#e5e7eb").attr("stroke-width", 1)

      // Etiqueta de variable (izquierda)
      svg.append("text")
        .attr("x", xU - 42).attr("y", cy + 4)
        .attr("text-anchor", "end").attr("font-size", "11px")
        .attr("fill", "currentColor").attr("opacity", 0.75)
        .text(v.replace(/\n/g, " ").replace(/\(.*\)/, "").trim())

      // Línea de pendiente
      const lineColor = pair.urbano > pair.rural ? "#378ADD" : "#D85A30"
      svg.append("line")
        .attr("x1", xUval).attr("x2", xRval)
        .attr("y1", cy).attr("y2", cy)
        .attr("stroke", lineColor).attr("stroke-width", 2.5).attr("opacity", 0.5)

      // Punto Urbano
      svg.append("circle").attr("cx", xUval).attr("cy", cy).attr("r", 16)
        .attr("fill", "#378ADD").attr("opacity", 0.9)
      svg.append("text").attr("x", xUval).attr("y", cy + 4)
        .attr("text-anchor", "middle").attr("font-size", "11px")
        .attr("font-weight", "600").attr("fill", "white")
        .text(pair.urbano.toFixed(1))

      // Punto Rural
      svg.append("circle").attr("cx", xRval).attr("cy", cy).attr("r", 16)
        .attr("fill", "#D85A30").attr("opacity", 0.9)
      svg.append("text").attr("x", xRval).attr("y", cy + 4)
        .attr("text-anchor", "middle").attr("font-size", "11px")
        .attr("font-weight", "600").attr("fill", "white")
        .text(pair.rural.toFixed(1))

      // Diferencia
      const diff = pair.urbano - pair.rural
      const midX = (xUval + xRval) / 2
      svg.append("text")
        .attr("x", midX).attr("y", cy - 20)
        .attr("text-anchor", "middle").attr("font-size", "10px")
        .attr("fill", diff > 0 ? "#378ADD" : diff < 0 ? "#D85A30" : "#888")
        .attr("font-weight", "500")
        .text(diff !== 0 ? `${diff > 0 ? "▲" : "▼"} ${Math.abs(diff).toFixed(2)}` : "sin diferencia")
    })

    return svg.node()
  }
  ```

  <div class="grid grid-cols-1">
    <div class="card">
      <p style="font-size:0.82em;color:var(--theme-foreground-muted)">
        Cada fila muestra el promedio percibido en zona <span style="color:#378ADD"><strong>Urbana</strong></span> vs <span style="color:#D85A30"><strong>Rural</strong></span> (escala 1–7). El triángulo indica qué zona percibe mayor efectividad y por cuánto.
      </p>
      ${resize(width => slopeChart({ width }))}
    </div>
  </div>

  <p>La comparación entre zonas urbanas y rurales en la percepción de efectividad policial arroja un resultado que desafía la intuición: las diferencias son mínimas en las tres funciones y no siguen una dirección uniforme. Para la investigación de delitos, la zona urbana puntúa marginalmente más alto (▲ 0.08), lo que podría reflejar mayor acceso a servicios de denuncia y seguimiento. Sin embargo, para el control del tráfico de drogas (▼ 0.04) y el mantenimiento de la seguridad (▼ 0.06), la percepción es ligeramente más favorable en zonas rurales que en urbanas, con diferencias de apenas cuatro y seis centésimas respectivamente. Estos resultados indican que la percepción de efectividad policial en Colombia en 2025 no está determinada de manera significativa por el tipo de territorio, sino que responde a factores más transversales como la función específica evaluada o la región. En términos prácticos, las brechas urbano-rural en efectividad percibida son tan pequeñas que no justificarían por sí solas estrategias diferenciadas por tipo de zona, aunque sí merecen seguimiento longitudinal para detectar si la tendencia se mantiene o se revierte.</P>
  ---

  ### Tiempo de respuesta policial percibido (infrax)

  Por último se visualiza la percepción ciudadana sobre los tiempos de respuesta de la Policía, desagregada por zona (urbana y rural) y por región. Cada barra representa la distribución porcentual de respuestas en distintos rangos de tiempo, desde respuestas rápidas (menos de 10 minutos) hasta percepciones extremas como “nunca llega”. El uso de colores permite identificar rápidamente la calidad del servicio percibido, donde los tonos verdes indican tiempos más favorables y los tonos rojos reflejan respuestas más lentas o ineficientes.

  ```js
  const INFRAX_ORDER = [
    "Menos de 10 minutos",
    "Entre 10 y hasta 30 minutos",
    "Más de 30 minutos y hasta una hora",
    "Más de 1 hora y hasta 3 horas",
    "Más de 3 horas",
    "[No Leer] No hay Policía/ No llegaría nunca"
  ]
  const INFRAX_SHORT = {
    "Menos de 10 minutos":                          "< 10 min",
    "Entre 10 y hasta 30 minutos":                  "10–30 min",
    "Más de 30 minutos y hasta una hora":           "30 min–1 h",
    "Más de 1 hora y hasta 3 horas":               "1–3 h",
    "Más de 3 horas":                               "> 3 h",
    "[No Leer] No hay Policía/ No llegaría nunca": "Nunca llega"
  }
  const INFRAX_COLORS = ["#1D9E75","#5DCAA5","#EF9F27","#D85A30","#A32D2D","#4B1528"]
  const shortLabels   = INFRAX_ORDER.map(c => INFRAX_SHORT[c])

  function calcInfrax(groupKey, groups) {
    const result = []
    for (const grp of groups) {
      const subset = data.filter(d => d[groupKey] === grp && d.infrax_clean != null && d.infrax_clean !== "")
      const total  = subset.length
      if (total === 0) continue
      let cumulative = 0
      for (const cat of INFRAX_ORDER) {
        const pct = (subset.filter(d => d.infrax_clean === cat).length / total) * 100
        result.push({
          group: grp,
          category: INFRAX_SHORT[cat],
          catFull: cat,
          x1: cumulative,
          x2: cumulative + pct,
          pct
        })
        cumulative += pct
      }
    }
    return result
  }

  const infraxData = [
    ...calcInfrax("ur", ["Urbano", "Rural"]).map(d => ({ ...d, tipo: "Zona" })),
    ...calcInfrax("estratopri", REGIONES).map(d => ({ ...d, tipo: "Región" }))
  ]
  const allGroups = ["Urbano", "Rural", ...REGIONES]
  ```

  ```js
  function stackedInfrax({ width } = {}) {
    return Plot.plot({
      width,
      height: 420,
      marginLeft: 185,
      marginRight: 20,
      marginTop: 20,
      marginBottom: 60,
      x: {
        label: "% de respuestas →",
        domain: [0, 100],
        grid: true,
        tickFormat: d => `${d}%`
      },
      y: { label: null, domain: allGroups },
      color: {
        domain: shortLabels,
        range:  INFRAX_COLORS,
        legend: true
      },
      marks: [
        Plot.ruleY(["Rural"], {
          stroke: "var(--theme-foreground-muted)",
          strokeDasharray: "3 3",
          strokeWidth: 0.8,
          dy: 24
        }),
        Plot.barX(infraxData, {
          x1: "x1", x2: "x2",
          y: "group",
          fill: "category",
          tip: true,
          title: d => `${d.group}\n${d.catFull}\n${d.pct.toFixed(1)}%`,
          insetTop: 5, insetBottom: 5
        }),
        // Etiquetas % dentro del segmento — solo si pct > 5%
        Plot.text(infraxData.filter(d => d.pct > 1), {
          x: d => (d.x1 + d.x2) / 2,
          y: "group",
          text: d => `${d.pct.toFixed(0)}%`,
          fill: "white",
          fontSize: 10,
          fontWeight: "600",
          textAnchor: "middle",
          dy: 1
        }),
        Plot.ruleX([0])
      ]
    })
  }
  ```

  <div class="grid grid-cols-1">
    <div class="card">
      <p style="font-size:0.82em;color:var(--theme-foreground-muted)">
        Verde = respuesta rápida · Rojo oscuro = "nunca llegaría". Las dos primeras filas corresponden a zona (Urbano/Rural), las siguientes a región.
      </p>
      ${resize(width => stackedInfrax({ width }))}
    </div>
  </div>

  <p>La distribución del tiempo de respuesta percibido es quizás el indicador más crítico del dashboard en términos operativos. A nivel general, solo el 5% de los encuestados urbanos y el 1% de los rurales percibe que la Policía llega en menos de 10 minutos, mientras que la mayoría de respuestas se concentra en el rango de 10 a 60 minutos. Sin embargo, los segmentos más preocupantes son los tiempos superiores a una hora: en zona urbana, el 31% percibe tiempos de respuesta mayores a una hora (20% entre 1–3 h y 11% más de 3 h), cifra que sube al 41% en zona rural (22% y 15% respectivamente), evidenciando una desventaja sistemática del campo frente a la ciudad. A nivel regional, la Pacífica presenta el perfil más crítico con un 8% que percibe que la Policía "nunca llegaría", seguida de Bogotá D.C. con el mismo porcentaje, lo cual resulta llamativo para la capital. La Andina muestra el mejor perfil relativo con el 43% de respuestas en los rangos más rápidos (< 10 min y 10–30 min), mientras que la Orinoquía-Amazonía y Bogotá D.C. concentran la mayor proporción en el rango de 30 min–1 hora (39% y 38%), sugiriendo respuestas lentas pero no ausentes. En conjunto, el tiempo de respuesta emerge como el eslabón más débil de la cadena operativa y un factor con alto potencial de impacto en la percepción de efectividad y confianza institucional.</p>

  ---

  ## Conclusiones y Apreciaciones finales

  ---
  <p>El Observatorio de Confianza en la Policía 2025 configura un retrato institucional de moderación con tensiones específicas. La confianza promedio de 4.06 sobre 7 —ligeramente superior al punto neutro de la escala— indica que la ciudadanía colombiana no rechaza a la Policía como institución, pero tampoco le otorga una valoración positiva consolidada. Esta posición intermedia no es estática ni homogénea: está atravesada por brechas territoriales, sociodemográficas y funcionales que revelan dónde se concentran los retos más urgentes.

  Territorialmente, el contraste entre la región Andina —con los niveles más altos en confianza, efectividad y tiempo de respuesta— y Bogotá D.C. y el Caribe —con los peores indicadores en casi todas las dimensiones— es el hallazgo más estructuralmente relevante del análisis. Que la capital del país y la región más poblada del Caribe sean precisamente los focos de menor confianza y peor desempeño percibido implica que el déficit institucional se concentra donde más población hay, amplificando su impacto en el promedio nacional y en la legitimidad del sistema de seguridad en su conjunto.

  En el plano de la legitimidad, la brecha entre el trato respetuoso (41.2%) y la justicia en el uso de la fuerza (25.2%) es el hallazgo más relevante desde el punto de vista teórico. Una institución percibida como amable pero no como justa enfrenta un problema de legitimidad procedimental que es más difícil de corregir que la percepción de maltrato, porque implica dudas sobre los principios que guían las decisiones institucionales y no solo sobre el tono de la interacción. La presencia sostenida —aunque minoritaria— de percepciones de corrupción, protección a delincuentes y violación de derechos humanos refuerza esta fragilidad de base.

  Finalmente, el tiempo de respuesta emerge como la dimensión operativa con mayor potencial de impacto en la percepción ciudadana. Con solo un 5% de encuestados urbanos que percibe respuestas en menos de diez minutos, y un 41% rural que reporta esperas superiores a una hora, existe una brecha entre la presencia física de la Policía y su capacidad de respuesta oportuna que erosiona la confianza independientemente de cómo sea el trato cuando finalmente llega. Cerrar esta brecha, especialmente en zonas rurales y en regiones como la Pacífica y Bogotá D.C., representa la intervención con mayor potencial de retorno en términos de percepción ciudadana a corto plazo. En síntesis, fortalecer la confianza en la Policía en Colombia requiere actuar simultáneamente en tres frentes: mejorar los tiempos de respuesta operativa, reforzar la percepción de justicia procedimental en el uso de la fuerza, y desarrollar estrategias territorialmente diferenciadas que prioricen las regiones con mayor déficit de legitimidad institucional.</p>

  ---

<div class="container">
</div>