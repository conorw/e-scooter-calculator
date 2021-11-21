<script>
  // import the map component so we can use it on screen
  import Map from "./Map.svelte";
  // import the scooter drop off locations
  import data from "./data.json";

  // the total distance label
  let totalDistance = "Click to start route";
  // total distance in metres
  let totalMetres = 0.0;
  // cost worksed out by multiplying meters by cost per metre
  let totalCost = 0.0;
  let costPerMeter = 0.003;

  let mapComponent;
  // when total metres changes on the map, change the cost calculation
  $: totalCost = parseFloat(totalMetres * costPerMeter).toFixed(2);

  function finishedDrawing(event) {
    alert(`Total £${totalCost}`);
  }
</script>

<!-- draw the map component -->
<main>
  <Map on:drawend="{finishedDrawing}" data={data} bind:this={mapComponent} bind:totalDistance bind:totalMetres />
  <div class="details">
    <!-- clear the map when the button is pressed -->
    <button on:click="{mapComponent.clear}">Clear</button>
    <!-- show the values on screen -->
    <div class="distance">{totalDistance}</div>
    <div class="cost">£{totalCost}</div>
  </div>
</main>
<style>
  .details {
    position: absolute;
    width: 180px;
    color: white;
    background: gray;
    opacity: 0.5;
    margin: 20px;
    padding: 20px;
    border-radius: 5px;
    top: 0;
    right: 0;
  }
  .cost {
    font-size: xx-large;
    color: lime;
    font-weight: bold;
  }
  .distance {
    font-size: small;
    color: white;
  }
  main {
    font-family: sans-serif;
    text-align: center;
    color: white;
    width: 100%;
    height: 100%;
    position: absolute;
    left: 0;
    top: 0;
    margin-left: 0;
  }
</style>
