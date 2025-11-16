<script>
  // Importamos funciones del ciclo de vida de Svelte
  import { onMount, onDestroy } from "svelte";

  // Importamos Chart.js para las gráficas
  import Chart from "chart.js/auto";

  // Importamos el parser YAML
  import yaml from "js-yaml";

  // Referencias a los canvas donde van los gráficos
  let canvasTemp, canvasPower;

  // Variables donde guardaremos las instancias de Chart.js
  let chartTemp, chartPower;

  // Valores actuales a mostrar
  let lastTemp = "--";
  let lastPower = "--";

  // Modo oscuro (si lo quieres usar)
  let darkMode = false;

  // Para mostrar errores
  let error = null;

  // Conversión Kelvin-decikelvin a Celsius
  const dKtoC = (x) => x / 10 - 273.15;

  // Conversión MW a kWh
  const MWtoKWh = (x) => parseFloat(x) * 1000;

  // Índices para **carga incremental**
  // Esto evita reprocesar todo el archivo YAML
  let idxTemp = 0;
  let idxPower = 0;

  // Máximo de puntos visibles para que Chart.js vaya rápido
  const MAX_POINTS = 600;

  // Función que recorta la gráfica si hay demasiados puntos
  function trimChart(chart) {
    const len = chart.data.labels.length;
    if (len > MAX_POINTS) {
      const cut = len - MAX_POINTS;
      chart.data.labels.splice(0, cut);        // eliminamos etiquetas viejas
      chart.data.datasets[0].data.splice(0, cut); // eliminamos valores viejos
    }
  }

  // ----------------------------------------------
  //   FUNCIÓN PRINCIPAL: Cargar y procesar datos YAML
  // ----------------------------------------------
  async function cargarDatos() {
    try {
      // Cargamos el archivo YAML
      const res = await fetch("/data.yml");
      if (!res.ok) return; // Si falla, salimos sin romper nada

      // Parseamos el YAML
      const yamlData = yaml.load(await res.text());

      // Tiempo actual hh:mm:ss → solo usamos datos hasta el momento real
      const now = new Date().toTimeString().slice(0, 8);

      // Listas completas desde el YAML
      const tempList = yamlData.temperature.values;
      const powerList = yamlData.power.values;

      // Guardamos los últimos valores actuales (para la tarjeta de arriba)
      lastTemp = dKtoC(tempList.at(-1).value).toFixed(2);
      lastPower = MWtoKWh(powerList.at(-1).value).toFixed(2);

      // ---------------------------------------------
      // Cargar SOLO los datos nuevos desde la última vez
      // ---------------------------------------------
      const newTemp = [];
      for (let i = idxTemp; i < tempList.length; i++) {
        if (tempList[i].time > now) break; // No cargar datos futuros
        newTemp.push(tempList[i]);
      }

      const newPower = [];
      for (let i = idxPower; i < powerList.length; i++) {
        if (powerList[i].time > now) break;
        newPower.push(powerList[i]);
      }

      // ---------------------------------------------
      // Agregar puntos nuevos en la gráfica de temperatura
      // ---------------------------------------------
      for (const v of newTemp) {
        chartTemp.data.labels.push(v.time);
        chartTemp.data.datasets[0].data.push(dKtoC(v.value).toFixed(2));
      }

      // ---------------------------------------------
      // Agregar puntos nuevos en la gráfica de potencia
      // ---------------------------------------------
      for (const v of newPower) {
        chartPower.data.labels.push(v.time);
        chartPower.data.datasets[0].data.push(MWtoKWh(v.value).toFixed(2));
      }

      // Evitar que las gráficas tengan demasiados puntos
      trimChart(chartTemp);
      trimChart(chartPower);

      // Actualizamos los gráficos una sola vez (muy rápido)
      chartTemp.update();
      chartPower.update();

      // Actualizamos los índices para no recargar datos viejos
      idxTemp = tempList.length;
      idxPower = powerList.length;

    } catch (e) {
      error = "Error al cargar datos: " + e.message;
    }
  }

  // ----------------------------------------------
  //   onMount: se ejecuta cuando el componente aparece
  // ----------------------------------------------
  onMount(() => {
    // Inicializar gráfica de temperatura
    chartTemp = new Chart(canvasTemp, {
      type: "line",
      data: {
        labels: [],
        datasets: [{
          label: "Temperatura (°C)",
          data: [],
          borderColor: "#ff5252",
          backgroundColor: "rgba(255,82,82,0.25)",
          tension: 0.3     // suaviza la curva
        }]
      },
      options: { responsive: true, maintainAspectRatio: false }
    });

    // Inicializar gráfica de potencia
    chartPower = new Chart(canvasPower, {
      type: "line",
      data: {
        labels: [],
        datasets: [{
          label: "Potencia (kWh)",
          data: [],
          borderColor: "#2979ff",
          backgroundColor: "rgba(41,121,255,0.25)",
          tension: 0.3
        }]
      },
      options: { responsive: true, maintainAspectRatio: false }
    });

    // Cargar datos por primera vez
    cargarDatos();

    // Recargar cada 5 segundos (rápido y eficiente)
    const interval = setInterval(cargarDatos, 5000);

    // Limpiar intervalo si el componente desaparece
    onDestroy(() => clearInterval(interval));
  });

  // Alternar modo oscuro (opcional)
  function toggleDark() {
    darkMode = !darkMode;
    document.body.classList.toggle("dark", darkMode);
  }
</script>

<!-- Título principal -->
<h1 class="titulo">Monitor de Temperatura y Potencia</h1>

<!-- Mostrar error si lo hay -->
{#if error}
  <p class="error">{error}</p>
{/if}

<!-- Contenido general -->
<main>
  <!-- Tarjetas superiores con valores actuales -->
  <section class="cards">
    <div class="card">
      <h3>Temperatura Actual</h3>
      <p>{lastTemp} °C</p>
    </div>

    <div class="card">
      <h3>Potencia Actual</h3>
      <p>{lastPower} kWh</p>
    </div>
  </section>

  <!-- Gráficas -->
  <section class="charts">

    <div class="chart-card">
      <h3>Temperatura</h3>
      <div class="chart-wrapper">
        <canvas bind:this={canvasTemp}></canvas>
      </div>
    </div>

    <div class="chart-card">
      <h3>Potencia</h3>
      <div class="chart-wrapper">
        <canvas bind:this={canvasPower}></canvas>
      </div>
    </div>

  </section>
</main>
