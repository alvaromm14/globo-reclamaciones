<script>
  import Glow from "$components/Glow.svelte";
  import Tooltip from "$components/Tooltip.svelte";

  import * as topojson from "topojson-client";
  import { geoOrthographic, geoPath, geoCentroid } from "d3-geo";
  import { timer } from "d3-timer";
  import { spring } from "svelte/motion";
  import { onMount } from "svelte";
  import { drag } from "d3-drag";
  import { select } from "d3-selection";

  // Importa tu mapa base del mundo desde src/data
  import world from "$data/110m.json";
  import reclamaciones from "$data/reclamaciones.json";

  // Geometría base del mundo
  let countries = topojson.feature(world, world.objects.countries).features;
  let borders = topojson.mesh(
    world,
    world.objects.countries,
    (a, b) => a !== b,
  );

  // Claims (reclamaciones)
  let claimFeatures = reclamaciones.features;

  // Dimensiones del SVG
  let width = 400;
  $: height = width;

  // Rotación y proyección
  let rotation = spring(0, { stiffness: 0.08, damping: 0.4 });
  let degreesPerFrame = 0.5;

  $: projection = geoOrthographic()
    .scale(width / 2)
    .rotate([$rotation, 0])
    .translate([width / 2, height / 2]);

  $: path = geoPath(projection);

  // Tooltip
  let tooltipData;

  // Rotación automática
  let dragging = false;
  const t = timer(() => {
    if (dragging || tooltipData) return;
    $rotation += degreesPerFrame;
  });

  // Drag
  let globe;
  const DRAG_SENSITIVITY = 0.5;

  // Configura drag con d3
  onMount(() => {
    // Drag existente
    const element = select(globe);
    element.call(
      drag()
        .on("drag", (event) => {
          $rotation += event.dx * DRAG_SENSITIVITY;
          dragging = true;
        })
        .on("end", () => {
          dragging = false;
        }),
    );

    // Resize iframe
    function updateIframeHeight() {
      const el = document.documentElement;
      const rect = el.getBoundingClientRect();
      const styles = window.getComputedStyle(el);
      const margin =
        parseFloat(styles.marginTop) + parseFloat(styles.marginBottom);
      const height = Math.ceil(rect.height + margin);
      window.parent.postMessage({ type: "resize-iframe", value: height }, "*");
    }

    updateIframeHeight();

    if (window.ResizeObserver) {
      new ResizeObserver(() => updateIframeHeight()).observe(
        document.documentElement,
      );
    } else {
      window.addEventListener("resize", updateIframeHeight);
    }

    window.addEventListener("message", (event) => {
      if (event.data.type === "request-resize") updateIframeHeight();
    });
  });

  // Centrar tooltip si hay datos
  $: if (tooltipData) {
    const center = geoCentroid(tooltipData);
    $rotation = -center[0];
  }
</script>

<div class="chart-container" bind:clientWidth={width}>
  <svg {width} {height} bind:this={globe} class:dragging>
    <Glow />

    <!-- Fondo del globo -->
    <circle
      cx={width / 2}
      cy={height / 2}
      r={width / 2}
      fill="#b7e1d9"
      filter="url(#glow)"
      on:click={() => (tooltipData = null)}
    />

    <!-- Países -->
    {#each countries as country}
      <path d={path(country)} fill="#fef8f3" stroke="none" />
    {/each}

    <!-- Bordes -->
    <path d={path(borders)} fill="none" stroke="#808080" stroke-width="0.25" />

    <!-- Tooltip -->
    {#if tooltipData}
      <path
        d={path(tooltipData)}
        fill="transparent"
        stroke="#e23300"
        stroke-width="2"
      />
    {/if}

    <!-- Reclamaciones -->
    {#each claimFeatures as reclamacion}
      <path
        d={path(reclamacion)}
        fill="rgba(226, 113, 93, 0.6)"
        stroke="#e23300"
        stroke-width="0.5"
      />

      <!-- Hit area invisible, mucho más fácil de clicar -->
      <path
        d={path(reclamacion)}
        fill="transparent"
        stroke="transparent"
        stroke-width="12"
        style="pointer-events: stroke; fill: none;"
        on:click|stopPropagation={() => (tooltipData = reclamacion)}
        cursor="pointer"
      />
    {/each}
  </svg>

  <Tooltip data={tooltipData} />
</div>

<style>
  .chart-container {
    max-width: 900px;
    margin: 0 auto;
  }

  svg {
    overflow: visible;
  }

  .dragging {
    cursor: grabbing;
  }
</style>
