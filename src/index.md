---
theme: dashboard
title: Observatorio de Paz y Seguridad
toc: false
---

<style>
.hero {
  max-width: 1100px;
  margin: 0 auto;
  padding: 24px 20px 8px 20px;
}

.hero h1 {
  font-size: 2.4rem;
  margin-bottom: 0.4rem;
  line-height: 1.1;
}

.hero .subtitle {
  font-size: 1.15rem;
  color: var(--theme-foreground-muted);
  margin-bottom: 1.25rem;
}

.hero .intro {
  max-width: 900px;
  line-height: 1.7;
  font-size: 1rem;
  text-align: justify;
  margin-bottom: 1.5rem;
}

.cards {
  max-width: 1100px;
  margin: 0 auto;
  padding: 8px 20px 24px 20px;
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(320px, 1fr));
  gap: 18px;
}

.card {
  background: var(--theme-background-alt, rgba(255,255,255,0.03));
  border: 1px solid var(--theme-foreground-faint, rgba(255,255,255,0.12));
  border-radius: 12px;
  padding: 22px 22px 18px 22px;
}

.card h2 {
  margin: 0 0 10px 0;
  font-size: 1.35rem;
}

.card p {
  line-height: 1.65;
  color: var(--theme-foreground);
  text-align: justify;
  margin-bottom: 1rem;
}

.badge {
  display: inline-block;
  font-size: 0.8rem;
  padding: 4px 10px;
  border-radius: 999px;
  margin-bottom: 12px;
  background: rgba(255,255,255,0.08);
  color: var(--theme-foreground-muted);
}

.actions {
  display: flex;
  gap: 10px;
  flex-wrap: wrap;
  margin-top: 12px;
}

.button {
  display: inline-block;
  padding: 10px 14px;
  border-radius: 8px;
  text-decoration: none;
  font-weight: 600;
  border: 1px solid var(--theme-foreground-faint, rgba(255,255,255,0.12));
  background: rgba(255,255,255,0.04);
  color: var(--theme-foreground);
}

.button:hover {
  background: rgba(255,255,255,0.08);
}

.section-note {
  max-width: 1100px;
  margin: 0 auto;
  padding: 0 20px 28px 20px;
  color: var(--theme-foreground-muted);
  line-height: 1.6;
}

hr {
  max-width: 1100px;
  margin: 18px auto 24px auto;
}
</style>

<div class="hero">
  <h1>Observatorio de Paz y Seguridad</h1>
  <div class="subtitle">Percepción ciudadana sobre seguridad, institucionalidad y confianza en Colombia</div>

  <p class="intro">
    Este observatorio reúne dos dashboards interactivos construidos a partir de encuestas de percepción pública,
    con el objetivo de explorar cómo han evolucionado en Colombia la victimización, la seguridad, la confianza en la Policía,
    la legitimidad institucional y la participación social. La plataforma permite comparar resultados a lo largo del tiempo,
    identificar diferencias entre grupos poblacionales y facilitar una lectura descriptiva de indicadores clave para el análisis
    académico y la toma de decisiones informadas.
  </p>
</div>

<hr>

<div class="cards">
  <div class="card">
    <div class="badge">Dashboard 1</div>
    <h2>Confianza en la Policía 2025</h2>
    <p>
      Este dashboard presenta un análisis descriptivo de la percepción ciudadana sobre la Policía en Colombia durante 2025.
      Incluye indicadores de confianza, legitimidad institucional y evaluación del desempeño, con desagregaciones por región,
      zona y características sociodemográficas.
    </p>
    <div class="actions">
      <a class="button" href="./Confianza_en_la_Policía_2025">Abrir dashboard</a>
    </div>
  </div>

  <div class="card">
    <div class="badge">Dashboard 2</div>
    <h2>Seguridad en Colombia a través de los años</h2>
    <p>
      Este dashboard analiza la evolución temporal de la victimización, la tipología del delito, el impacto del conflicto armado,
      la percepción institucional y la participación social. Permite examinar tendencias históricas y contrastes territoriales
      en distintos momentos de la serie de datos.
    </p>
    <div class="actions">
      <a class="button" href="./Seguridad_por_Años">Abrir dashboard</a>
    </div>
  </div>
</div>

<div class="section-note">
  El observatorio está diseñado como una herramienta reproducible y accesible para explorar patrones, contrastes y transformaciones
  en la percepción ciudadana sobre seguridad y paz en Colombia.
</div>