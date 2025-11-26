<script lang="ts">
  import "bootstrap/dist/css/bootstrap.min.css";
  import "bootstrap/dist/js/bootstrap.min.js";
  import "@fortawesome/fontawesome-free/css/all.min.css";
  import ReportProblem from "./ReportProblem.svelte";
  import Loader from "./Loader.svelte";
  import Auth from "./Auth.svelte";
  import favicon from "../assets/favicon.ico?url";
  import logo from "../assets/logo.svg?url";
  import arrow from "../assets/arrow.png?url";
  import { MapLibre } from "svelte-maplibre";
  import { onMount, untrack } from "svelte";
  import { backend } from "./";
  import type { Map } from "maplibre-gl";
  import {
    basemapStyles,
    Basemaps,
    Geocoder,
    StandardControls,
  } from "svelte-utils/map";
  import { Layout, leftTarget } from "svelte-utils/two_column_layout";
  import { Modal } from "svelte-utils";
  import * as backendPkg from "../../backend/pkg";
  import MainMode from "./main/MainMode.svelte";

  let map: Map | undefined = $state();
  let loaded = $state(false);
  let basemap = $state("Maptiler OpenStreetMap");
  let show = $state(true);

  onMount(async () => {
    await backendPkg.default();
  });

  // svelte-ignore state_referenced_locally
  let initialStyle = basemapStyles.get(basemap)!;
  // TODO Upstream to svelte-maplibre. It feels a bit brittle how we detect the sources and layers created here, vs from the basemap
  $effect(() => {
    if (basemap) {
      untrack(() => {
        map?.setStyle(basemapStyles.get(basemap)!, {
          transformStyle: (previousStyle, nextStyle) => {
            if (!previousStyle) {
              return nextStyle;
            }

            let customLayers = previousStyle.layers.filter((l) =>
              ["circle-", "line-", "fill-", "heatmap-", "symbol-"].some(
                (prefix) => l.id.startsWith(prefix),
              ),
            );
            let layers = nextStyle.layers.concat(customLayers);
            let sources = nextStyle.sources;

            for (let [key, value] of Object.entries(
              previousStyle.sources || {},
            )) {
              if (key.startsWith("geojson-") || key.startsWith("heatmap")) {
                sources[key] = value;
              }
            }

            return {
              ...nextStyle,
              sources: sources,
              layers: layers,
            };
          },
        });
      });
    }
  });
</script>

<svelte:head>
  <link rel="icon" type="image/x-icon" href={favicon} />
</svelte:head>

<Layout>
  {#snippet left()}
    <div>
      <div class="d-flex align-items-center">
        <img
          src={logo}
          style="height: 30px"
          class="me-3"
          alt="A/B Street logo"
        />
        <h2 class="me-3">Speedwalk</h2>
        <!-- svelte-ignore a11y_invalid_attribute -->
        <a href="#" onclick={() => (show = true)} title="About Speedwalk">
          <i class="fa-solid fa-circle-info"></i>
        </a>
      </div>

      <Auth />

      {#if map}
        <div bind:this={leftTarget.value}></div>
      {/if}
    </div>
  {/snippet}

  {#snippet main()}
    <div style="position:relative; width: 100%; height: 100vh;">
      <MapLibre
        style={initialStyle}
        hash
        bind:map
        bind:loaded
        onerror={(e) => console.log(e.error)}
        images={[{ id: "arrow", url: arrow }]}
      >
        <StandardControls {map} />
        <Geocoder {map} {loaded} />
        <!--<MapContextMenu {map} />-->
        <Basemaps bind:basemap />

        {#if map}
          <ReportProblem {map} />

          {#if $backend}
            <MainMode {map} />
          {:else}
            <Loader {map} />
          {/if}
        {/if}
      </MapLibre>
    </div>
  {/snippet}
</Layout>

<Modal bind:show>
  <h1>Welcome to Speedwalk</h1>

  <p>
    This tool helps you quickly assess how sidewalks are mapped in OSM. You can
    find and fix common tagging problems. The tool assumes you understand the
    correct <a
      href="https://wiki.openstreetmap.org/wiki/Sidewalks"
      target="_blank"
    >
      sidewalk mapping conventions
    </a>
    . If you are unsure about some edits you make, you can download the changeset
    file and check in JOSM, rather than uploading directly.
  </p>

  <p>
    This is an <a
      href="https://github.com/a-b-street/speedwalk"
      target="_blank"
    >
      open source project
    </a>
    developed without funding by
    <a href="https://abstreet.uk" target="_blank">A/B Street Ltd</a>
    . Please file a Github issue or contact
    <a href="mailto:dustin@abstreet.uk" target="_blank">dustin@abstreet.uk</a>
    with feedback.
    <b>This is an alpha tool; there will be problems!</b>
  </p>

  <button class="btn btn-primary" onclick={() => (show = false)}>Start</button>
</Modal>
