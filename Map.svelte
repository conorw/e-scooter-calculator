<script>
  import { onMount } from "svelte";
  import "ol/ol.css";
  import Draw from "ol/interaction/Draw";
  import Map from "ol/Map";
  import { fromLonLat } from "ol/proj";
  import Overlay from "ol/Overlay";
  import Feature from "ol/Feature";
  import Point from "ol/geom/Point";
  import View from "ol/View";
  import { Circle as CircleStyle, Fill, Stroke, Style, Icon } from "ol/style";
  import { LineString, Polygon } from "ol/geom";
  import OSM from "ol/source/OSM";
  import VectorSource from "ol/source/Vector";
  import { Tile as TileLayer, Vector as VectorLayer } from "ol/layer";
  import { getArea, getLength } from "ol/sphere";
  import { unByKey } from "ol/Observable";
  import { createEventDispatcher } from "svelte";

  const dispatch = createEventDispatcher();
  let mapDiv;

  // the point data
  export let data;

  export let totalDistance = "";
  export let totalMetres = 0.0;

  // use open street map for base layer
  const raster = new TileLayer({
    source: new OSM({
      url: "http://{a-c}.basemaps.cartocdn.com/dark_all/{z}/{x}/{y}.png"
    })
  });

  const iconStyle = new Style({
    fill: new Fill({
      color: "rgba(255, 255, 255, 0.2)"
    }),
    stroke: new Stroke({
      color: "#ffcc33",
      width: 2
    }),
    image: new Icon({
      anchor: [0.5, 46],
      anchorXUnits: "fraction",
      anchorYUnits: "pixels",
      src: "./icon_white.png"
    })
  });

  // add the fetaures to an array
  const source = new VectorSource({
    features: data.map(
      t =>
        new Feature({
          id: t.location,
          geometry: new Point(fromLonLat([t.lon, t.lat])),
          name: t.location,
          style: iconStyle
        })
    )
  });

  const vector = new VectorLayer({
    source: source,
    style: iconStyle
  });

  /**
   * Currently drawn feature.
   * @type {import("../src/ol/Feature.js").default}
   */
  let sketch;

  /**
   * The help tooltip element.
   * @type {HTMLElement}
   */
  let helpTooltipElement;

  /**
   * Overlay to show the help messages.
   * @type {Overlay}
   */
  let helpTooltip;

  /**
   * The measure tooltip element.
   * @type {HTMLElement}
   */
  let measureTooltipElement;

  /**
   * Overlay to show the measurement.
   * @type {Overlay}
   */
  let measureTooltip;

  /**
   * Message to show when the user is drawing a polygon.
   * @type {string}
   */
  const continuePolygonMsg = "Click to add stop. Double click to end";

  /**
   * Message to show when the user is drawing a line.
   * @type {string}
   */
  const continueLineMsg = "Click to add stop. Double click to end";

  /**
   * Handle pointer move.
   * @param {import("../src/ol/MapBrowserEvent").default} evt The event.
   */
  const pointerMoveHandler = function(evt) {
    if (evt.dragging) {
      return;
    }
    /** @type {string} */
    let helpMsg = "Click to start route";

    if (sketch) {
      const geom = sketch.getGeometry();
      if (geom instanceof Polygon) {
        helpMsg = continuePolygonMsg;
      } else if (geom instanceof LineString) {
        helpMsg = continueLineMsg;
      }
    }

    helpTooltipElement.innerHTML = helpMsg;
    helpTooltip.setPosition(evt.coordinate);

    helpTooltipElement.classList.remove("hidden");
  };

  export function clear() {
    console.log("clear called", draw);
    map.getInteractions().forEach(interaction => {
      if (interaction instanceof Draw) {
        console.log("remove", interaction);
        map.removeInteraction(interaction);
      }
    });
    var features = source.getFeatures().forEach(feature => {
      console.log(feature);
      if (!feature.values_.id) {
        source.removeFeature(feature);
      }
    });
    totalDistance = "";
    totalMetres = 0.0;
    document.querySelectorAll(".ol-tooltip").forEach(function(a) {
      a.remove();
    });
    helpTooltipElement = null;
    reset();
    addInteraction();
  }
  let map;

  let draw; // global so we can remove it later

  let listener;

  /**
   * Format length output.
   * @param {LineString} line The line.
   * @return {string} The formatted length.
   */
  const formatLength = function(line) {
    const length = getLength(line);
    let output;
    totalMetres = length;
    if (length > 100) {
      output = Math.round((length / 1000) * 100) / 100 + " " + "km";
    } else {
      output = Math.round(length * 100) / 100 + " " + "m";
    }
    return output;
  };

  /**
   * Format area output.
   * @param {Polygon} polygon The polygon.
   * @return {string} Formatted area.
   */
  const formatArea = function(polygon) {
    const area = getArea(polygon);
    let output;
    if (area > 10000) {
      output = Math.round((area / 1000000) * 100) / 100 + " " + "km<sup>2</sup>";
    } else {
      output = Math.round(area * 100) / 100 + " " + "m<sup>2</sup>";
    }
    return output;
  };

  function addInteraction() {
    const type = "LineString";
    draw = new Draw({
      source: source,
      type: type,
      style: new Style({
        fill: new Fill({
          color: "rgba(255, 255, 255, 0.2)"
        }),
        stroke: new Stroke({
          color: "rgba(255, 255, 0, 0.5)",
          lineDash: [10, 10],
          width: 2
        }),
        image: new CircleStyle({
          radius: 5,
          stroke: new Stroke({
            color: "rgba(255, 255, 0, 0.7)"
          }),
          fill: new Fill({
            color: "rgba(255, 255, 255, 0.2)"
          })
        })
      })
    });
    map.addInteraction(draw);

    createMeasureTooltip();
    createHelpTooltip();

    draw.on("drawstart", function(evt) {
      // set sketch
      sketch = evt.feature;

      /** @type {import("../src/ol/coordinate.js").Coordinate|undefined} */
      let tooltipCoord = evt.coordinate;

      listener = sketch.getGeometry().on("change", function(evt) {
        const geom = evt.target;
        let output;

        output = formatLength(geom);
        tooltipCoord = geom.getLastCoordinate();

        totalDistance = output;
        measureTooltipElement.innerHTML = output;
        measureTooltip.setPosition(tooltipCoord);
      });
    });

    draw.on("drawend", function() {
      reset();
      dispatch("drawend");
    });
  }
  function reset() {
    measureTooltipElement.className = "ol-tooltip ol-tooltip-static";
    measureTooltip.setOffset([0, -7]);
    // unset sketch
    sketch = null;
    // unset tooltip so that a new one can be created
    measureTooltipElement = null;
    createMeasureTooltip();
    unByKey(listener);
  }
  /**
   * Creates a new help tooltip
   */
  function createHelpTooltip() {
    if (helpTooltipElement) {
      helpTooltipElement.parentNode.removeChild(helpTooltipElement);
    }
    helpTooltipElement = document.createElement("div");
    helpTooltipElement.className = "ol-tooltip hidden";
    helpTooltip = new Overlay({
      element: helpTooltipElement,
      offset: [15, 0],
      positioning: "center-left"
    });
    map.addOverlay(helpTooltip);
  }

  /**
   * Creates a new measure tooltip
   */
  function createMeasureTooltip() {
    if (measureTooltipElement) {
      measureTooltipElement.parentNode.removeChild(measureTooltipElement);
    }
    measureTooltipElement = document.createElement("div");
    measureTooltipElement.className = "ol-tooltip ol-tooltip-measure";
    measureTooltip = new Overlay({
      element: measureTooltipElement,
      offset: [0, -15],
      positioning: "bottom-center",
      stopEvent: false,
      insertFirst: false
    });
    map.addOverlay(measureTooltip);
  }

  onMount(() => {
    map = new Map({
      layers: [raster, vector],
      target: mapDiv,
      view: new View({
        center: fromLonLat([-1.8904, 52.4862]),
        zoom: 13
      })
    });

    map.on("pointermove", pointerMoveHandler);

    //change map color
    map.on("postcompose", function(e) {
      // document.querySelector('canvas').style.filter="invert(90%)";
    });

    map.getViewport().addEventListener("mouseout", function() {
      helpTooltipElement.classList.add("hidden");
    });

    addInteraction();
  });
</script>

<style>
  .map {
    width: 100%;
    height: 100%;
    position: absolute;
  }
  .ol-tooltip {
    position: relative;
    background: rgba(0, 0, 0, 0.5);
    border-radius: 4px;
    color: white;
    padding: 4px 8px;
    opacity: 0.7;
    white-space: nowrap;
    font-size: 12px;
    cursor: default;
    user-select: none;
  }
  .ol-tooltip-measure {
    opacity: 1;
    font-weight: bold;
    color: white;
  }
  .ol-tooltip-static {
    background-color: #ffcc33;
    color: white;
    border: 1px solid white;
  }
  .ol-tooltip-measure:before,
  .ol-tooltip-static:before {
    border-top: 6px solid rgba(0, 0, 0, 0.5);
    border-right: 6px solid transparent;
    border-left: 6px solid transparent;
    content: "";
    position: absolute;
    bottom: -6px;
    margin-left: -7px;
    left: 50%;
  }
  .ol-tooltip-static:before {
    border-top-color: #ffcc33;
  }
</style>

<div bind:this={mapDiv} id="map" class="map"></div>
