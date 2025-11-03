<script lang="ts">
  import Auth from "./Auth.svelte";
  import favicon from "../assets/favicon.ico?url";
  import { MapLibre } from "svelte-maplibre";
  import { PolygonToolLayer } from "maplibre-draw-polygon";
  import { onMount, untrack } from "svelte";
  import { backend } from "./";
  import "bootstrap/dist/css/bootstrap.min.css";
  import type { Map } from "maplibre-gl";
  import {
    bbox,
    basemapStyles,
    Basemaps,
    StandardControls,
    //MapContextMenu,
  } from "svelte-utils/map";
  import { OverpassSelector } from "svelte-utils/overpass";
  import { Loading } from "svelte-utils";
  import { Layout, leftTarget } from "svelte-utils/two_column_layout";
  import * as backendPkg from "../../backend/pkg";
  import SidewalksMode from "./sidewalks/SidewalksMode.svelte";

  let loading = $state("");
  let map: Map | undefined = $state();

  onMount(async () => {
    await backendPkg.default();
  });

  let fileInput: HTMLInputElement = $state();
  async function loadFile(e: Event) {
    try {
      loading = "Loading from file";
      let bytes = await fileInput.files![0].arrayBuffer();
      $backend = new backendPkg.Speedwalk(new Uint8Array(bytes));
      zoomFit();
    } catch (err) {
      window.alert(`Bad input file: ${err}`);
    } finally {
      loading = "";
    }
  }

  async function gotXml(e: CustomEvent<{ xml: string }>) {
    try {
      let bytes = new TextEncoder().encode(e.detail.xml);
      $backend = new backendPkg.Speedwalk(new Uint8Array(bytes));
      zoomFit();
    } catch (err) {
      window.alert(`Couldn't import from Overpass: ${err}`);
    } finally {
      loading = "";
    }
  }

  function zoomFit() {
    map!.fitBounds(bbox(JSON.parse($backend!.getNodes())), {
      animate: false,
      padding: 10,
    });
  }

  function clear() {
    $backend = null;
  }

  let basemap = $state("Maptiler OpenStreetMap");
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

<Loading {loading} />

<Layout>
  {#snippet left()}
    <h1>Speedwalk</h1>

    {#if $backend}
      <button class="btn btn-secondary" onclick={clear}>
        Load another area
      </button>
    {:else if map}
      <div>
        <label class="form-label">
          Load an osm.pbf or osm.xml file
          <input
            class="form-control"
            bind:this={fileInput}
            onchange={loadFile}
            type="file"
          />
        </label>
      </div>

      <p class="fst-italic my-3">or...</p>

      <OverpassSelector
        {map}
        on:gotXml={gotXml}
        on:loading={(e) => (loading = e.detail)}
        on:error={(e) => window.alert(e.detail)}
      />
    {/if}

    <Auth />

    <div bind:this={leftTarget.value}></div>
  {/snippet}

  {#snippet main()}
    <div style="position:relative; width: 100%; height: 100vh;">
      <MapLibre
        style={initialStyle}
        hash
        bind:map
        on:error={(e) => {
          console.log(e.error);
        }}
      >
        <StandardControls {map} />
        <!--<MapContextMenu {map} />-->
        <Basemaps bind:basemap />

        {#if $backend && map}
          <SidewalksMode {map} />
        {:else}
          <PolygonToolLayer />
        {/if}
      </MapLibre>
    </div>
  {/snippet}
</Layout>
