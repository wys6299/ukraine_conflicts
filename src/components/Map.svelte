<script>
  import mapboxgl from "mapbox-gl";
  import { onMount } from "svelte";

  mapboxgl.accessToken =
    "pk.eyJ1Ijoid3lzNjI5OSIsImEiOiJjbGtodDNrMnIwYmxwM2dxbDU3azZlZ3JiIn0.Hb3fx_PsGtyRh38lP1Nwcg";

  let container;
  let map;
  let hexbinLayers = {}; // Object to hold layers for each EVENT_TYPE
  let processed;
  let colorRamp = ["red", "blue", "orange", "green", "purple"];

  function updateZoomLevel() {
    const screenWidth = window.innerWidth;
    return screenWidth <= 600 ? 5 : 5.1; // Adjust these values as needed
  }

  function handleResize() {
    map.setZoom(updateZoomLevel());
  }

  onMount(() => {
    map = new mapboxgl.Map({
      container,
      style: "mapbox://styles/wys6299/clt7qstej00hs01o801l17nat",
      center: [31, 48.5],
      zoom: updateZoomLevel(),
      attributionControl: true,
    });

    window.addEventListener("resize", handleResize);

    const popup = new mapboxgl.Popup({
      closeButton: false,
      closeOnClick: false
    });

    map.on("load", function () {
      processed = fetch(
        "https://raw.githubusercontent.com/wys6299/ukraine_conflicts/main/src/data/filtered.geojson"
      )
        .then((response) => {
          if (response.ok) {
            return response.json();
          }
          throw new Error("Problem loading processed data!");
        })
        .then((data) => {
          const eventTypes = new Set(
            data.features.map((feature) => feature.properties.EVENT_TYPE)
          );

          // Create a layer for each EVENT_TYPE
          for (const [index, eventType] of [...eventTypes].entries()) {
            const filteredFeatures = data.features.filter(
              (feature) => feature.properties.EVENT_TYPE === eventType
            );

            // Add a source for this EVENT_TYPE
            map.addSource(`${eventType}-source`, {
              type: "geojson",
              data: {
                type: "FeatureCollection",
                features: filteredFeatures,
              },
            });

            // Add a layer for this EVENT_TYPE
            map.addLayer({
              id: `${eventType}-layer`,
              type: "circle",
              source: `${eventType}-source`,
              paint: {
                "circle-color": colorRamp[index], // Use a unique color for each layer
                "circle-radius": 6,
              },
            });

            // Store the layer reference in the hexbinLayers object
            hexbinLayers[eventType] = map.getLayer(`${eventType}-layer`);
          }
        })
        .catch((error) => {
          console.error("Error fetching or processing data:", error);
        });

      // Add a popup when hovering over the features
      map.on("mousemove", "circle", function (e) {
        if (e.features.length > 0) {
          const properties = e.features[0].properties;

            popup
            .setLngLat(e.lngLat)
            .setText(
              `<h3>${properties.EVENT_TYPE}</h3>
              <p>Date: ${properties.EVENT_DATE}</p>
              <p>Type: ${properties.DISORDER_TYPE}</p>
              <p>Location: ${properties.LOCATION}</p>
              <p>Notes: ${properties.NOTES}</p>
              <p>Fatalities: ${properties.FATALITIES}</p>`
            )
            .addTo(map);
        }
      });

      // Remove the popup when the mouse leaves the feature
      map.on("mouseleave", "circle", function () {
        popup.remove();
      });
    });
  });

  let isVisible = true;

  function toggleLayerVisibility(eventType) {
    const currentVisibility = map.getLayoutProperty(
      `${eventType}-layer`,
      "visibility"
    );

    const newVisibility = currentVisibility === "visible" ? "none" : "visible";

    map.setLayoutProperty(
      `${eventType}-layer`,
      "visibility",
      newVisibility
    );
}
</script>

<svelte:head>
  <link
    rel="stylesheet"
    href="https://api.mapbox.com/mapbox-gl-js/v2.14.0/mapbox-gl.css"
  />
</svelte:head>

<div class="map" class:visible={isVisible} bind:this={container} />
<div class="toggle-button-container">
  {#each Object.keys(hexbinLayers) as eventType, index}
    <button
      class="toggle-button"
      on:click={() => toggleLayerVisibility(eventType)}
    >
      {eventType}
      <span class="color-dot" style="background-color: {colorRamp[index]}"></span>
    </button>
  {/each}
</div>

<style>
  .map {
    width: 100%;
    height: 60vh;
    position: relative;
    opacity: 0;
    visibility: visible;
    transition: opacity 2s, visibility 2s;
    outline: rgb(0, 0, 0) solid 3px;
    max-width: 950px;
  }

  .map.visible {
    opacity: 1;
    visibility: visible;
  }

  .toggle-button {
    margin-bottom: 15px; /* Increase bottom margin for more spacing between buttons */
    padding: 10px 20px; /* Increase padding to make the buttons larger */
    font-size: 12px; /* Increase font size for better readability */
    background-color: #ffffff;
    border: 1px solid #cccccc;
    border-radius: 5px;
    cursor: pointer;
  }

  .toggle-button:hover {
    background-color: #f0f0f0;
  }

  .color-dot {
    display: inline-block;
    width: 12px; /* Adjust the size of the color dots accordingly */
    height: 12px;
    border-radius: 50%;
    margin-left: 10px; /* Increase left margin for more space between text and dot */
  }
</style>