<script lang="ts">
  import {
    Mark,
    Scales,
    Ticker,
    MarkRenderGroup,
    Attribute,
  } from "counterpoint-vis";
  import * as d3 from "d3";
  import ingredientsArray from "../ingredients_filtered.js";
  import { onMount } from "svelte";

  type FoodMark = {
    title: Attribute<string>;
    x: Attribute<number>;
    y: Attribute<number>;
    img: Attribute<HTMLImageElement>;
    ingredients: Attribute<string[]>;
    size: Attribute<number>;
  };

  type SummaryMark = {
    x: Attribute<number>;
    y: Attribute<number>;
    size: Attribute<number>;
    xIndex: Attribute<number>;
    yIndex: Attribute<number>;
  };

  let canvasHeight: number = 1000;
  let canvasWidth: number = 830;

  let dataCSV: d3.DSVRowArray;
  let canvas: HTMLCanvasElement;
  let ticker: Ticker;
  let foodSet: MarkRenderGroup<FoodMark>;
  let summarySet: MarkRenderGroup<SummaryMark>;
  let imageCache: Record<string, HTMLImageElement> = {};
  let imagesLoaded: boolean = false;
  var imageCanvas: HTMLCanvasElement;
  var ctx: CanvasRenderingContext2D | null;
  var imageData: ImageData;
  var zoomedModeMarkCount: number = 5;
  let set: MarkRenderGroup<FoodMark>;
  function preloadImages(
    dataCSV: d3.DSVRowArray,
    initialSetup: boolean = false
  ): Promise<void[]> {
    dataCSV.forEach((d, i) => {
      const img = new Image();
      img.src =
        d.Image_Name === "#NAME?"
          ? "src/placeholder_images/no-image-found.jpg"
          : `src/datasets/Food Images/${d.Image_Name}.jpg`;
      imageCache[i] = img;
    });

    if (initialSetup) {
      const transform = d3.zoomTransform(canvas);
      const visibleMarks = foodSet.filter((mark) => {
        const x = transform.applyX(mark.attr("x"));
        const y = transform.applyY(mark.attr("y"));
        const size = 20;
        return (
          x + size >= 0 &&
          y + size >= 0 &&
          x - size <= canvas.clientWidth &&
          y - size <= canvas.clientHeight
        );
      });

      return Promise.all(
        visibleMarks.map((m, i) => {
          return new Promise<void>((resolve) => {
            const img = imageCache[Number(m.id)];
            if (img.complete) {
              resolve();
            } else {
              img.onload = () => resolve();
            }
          });
        })
      );
    } else {
      return Promise.resolve([]);
    }
  }

  const findIngredients = (id: number): string[] => {
    if (!dataCSV) return [];
    const ingredientsString = dataCSV[id]?.Ingredients ?? "";
    return d3.filter(ingredientsArray, (ing) =>
      ingredientsString.includes(ing)
    );
  };

  const createMark = (id: string): Mark<FoodMark> =>
    new Mark<FoodMark>(id, {
      x: { value: 10000 * Math.random() },
      y: { value: 10000 * Math.random() },
      img: { valueFn: () => imageCache[Number(id)] },
      ingredients: {
        valueFn: (mark) => {
          const ingArr = findIngredients(Number(id));
          mark.size = ingArr.length;
          return ingArr;
        },
      },
      size: { valueFn: (mark) => mark.size },
    });

  function setupCanvas() {
    // canvas.width = canvas.offsetWidth * window.devicePixelRatio;
    // canvas.height = canvas.offsetHeight * window.devicePixelRatio;
    canvas.width = canvasWidth;
    canvas.height = canvasHeight;
    imageCanvas.height = canvas.height;
    imageCanvas.width = canvas.width;
    d3.select(canvas as Element).call(zoom);
    drawFoodView();
  }

  const drawFoodView = () => {
    if (!dataCSV || !canvas) return;

    const ctx = canvas.getContext("2d", { willReadFrequently: true });
    // ctx = imageCanvas.getContext("2d", { willReadFrequently: true });
    if (!ctx || !ctx) return;

    const transform = d3.zoomTransform(canvas);
    const visibleMarks = foodSet.filter((mark) => {
      const x = transform.applyX(mark.attr("x"));
      const y = transform.applyY(mark.attr("y"));
      const size = 20;
      return (
        x + size >= 0 &&
        y + size >= 0 &&
        x - size <= canvas.clientWidth &&
        y - size <= canvas.clientHeight
      );
    });
    // new feature - d3.extent for markset?

    if (visibleMarks.count() > 500) {
      // let maxes = getMaxes(visibleMarks);
      // let mins = getMins(visibleMarks);
      ctx.resetTransform();
      ctx.scale(window.devicePixelRatio, window.devicePixelRatio);
      ctx.clearRect(0, 0, canvas.clientWidth, canvas.clientHeight);

      // summarySet
      //   .update("x", (m) => transform.applyX((maxes[0] / 5) * m.attr("xIndex")))
      //   .update("y", (m) => transform.applyY((maxes[1] / 5) * m.attr("yIndex")))
      //   .update("size", (m) => {
      //     return (set.count() / foodSet.count()) * 20000;
      //   });

      summarySet.forEach((mark) => {
        const { x, y, size } = mark.get();
        const transformedX = Math.round(transform.applyX(x));
        const transformedY = Math.round(transform.applyY(y));
        if (ctx) {
          ctx.fillStyle = "blue";
          ctx.save();
          ctx.beginPath();
          ctx.arc(
            transformedX + size,
            transformedY + size,
            size,
            0,
            Math.PI * 2,
            true
          );
          ctx.fill();
        }
      });
    } else {
      ctx.resetTransform();
      ctx.scale(window.devicePixelRatio, window.devicePixelRatio);
      ctx.clearRect(0, 0, canvas.clientWidth, canvas.clientHeight);
      visibleMarks.forEach((mark) => {
        const { x, y, img, size } = mark.get();
        const transformedX = Math.round(transform.applyX(x));
        const transformedY = Math.round(transform.applyY(y));
        if (img.src && img.complete && ctx) {
          ctx.save();
          ctx.beginPath();
          ctx.arc(
            transformedX + size,
            transformedY + size,
            size,
            0,
            Math.PI * 2,
            true
          );
          ctx.closePath();
          ctx.clip();
          ctx.drawImage(img, transformedX, transformedY, size * 2, size * 2); // round to int?
          ctx.restore();
        }
      });
    }

    // imageData = ctx.getImageData(0, 0, canvasWidth, canvasHeight);
    // ctx.putImageData(imageData, 0, 0);
  };

  const createAxes = () => {
    const svg = d3.select("#axes");
    const marginTop = 60,
      marginRight = 60,
      marginBottom = 60,
      marginLeft = 60;
    const width = window.innerWidth,
      height = window.innerHeight;

    const xScale = d3
      .scaleLinear()
      .domain(scales.xScale.range())
      .range([0, width - marginLeft - marginRight]);

    const yScale = d3
      .scaleLinear()
      .domain(scales.yScale.range())
      .range([height - marginTop - marginBottom, 0]);

    svg.attr("width", width).attr("height", height);
    svg.selectAll("*").remove();

    svg
      .append("g")
      .style("font-size", "10pt")
      .attr("transform", `translate(${marginLeft},${height - marginBottom})`)
      .call(d3.axisBottom(xScale).tickArguments([5, ",.6~s"]))
      .call((g) =>
        g
          .append("text")
          .attr("x", width - marginRight - marginLeft)
          .attr("y", 40)
          .attr("fill", "currentColor")
          .attr("text-anchor", "end")
          .text(`bottom axis →`)
      );

    svg
      .append("g")
      .style("font-size", "10pt")
      .attr("transform", `translate(${marginLeft},${marginTop})`)
      .call(d3.axisLeft(yScale).tickArguments([5, ",.6~s"]))
      .call((g) =>
        g
          .append("text")
          .attr("x", -marginLeft)
          .attr("dy", "-2em")
          .attr("fill", "currentColor")
          .attr("text-anchor", "end")
          .text(`↑ y axis`)
      );
  };

  const createMarkSet = (): Mark<FoodMark>[] =>
    dataCSV ? dataCSV.map((point) => createMark(point[""])) : [];

  onMount(async () => {
    dataCSV = await d3.csv("src/datasets/food_data.csv");
    foodSet = new MarkRenderGroup(createMarkSet());
    foodSet.configure({ animationDuration: 500 });
    summarySet = createSummaryMarks();
    await preloadImages(dataCSV, true);
    imageCanvas = document.createElement("canvas");
    imagesLoaded = true;
    // let maxes = getMaxes(foodSet);
    // let mins = getMins(foodSet);
    // set = foodSet.filter(
    //   (mark) =>
    //     mark.attr("x") <=
    //       mins[0] + (maxes[0] / zoomedModeMarkCount) * (m.attr("xIndex") + 1) &&
    //     mark.attr("y") >=
    //       mins[1] + (maxes[1] / zoomedModeMarkCount) * m.attr("yIndex") &&
    //     mark.attr("y") <=
    //       mins[1] + (maxes[1] / zoomedModeMarkCount) * (m.attr("yIndex") + 1) &&
    //     mark.attr("x") >=
    //       mins[0] + (maxes[0] / zoomedModeMarkCount) * m.attr("xIndex")
    // );

    setupCanvas();
  });

  $: {
    if (imagesLoaded) {
      ticker = new Ticker([foodSet, scales]).onChange(() => {
        requestAnimationFrame(drawFoodView);
      });
    }
  }

  $: {
    if (imagesLoaded) {
      requestAnimationFrame(drawFoodView);
      createAxes();
    }
  }

  $: if (canvas && foodSet && imagesLoaded) {
    d3.select(canvas as Element).call(zoom);
    requestAnimationFrame(drawFoodView);
  }

  let scales = new Scales({ animationDuration: 500 })
    .xRange([0, 500])
    .yRange([0, 500])
    .onUpdate(() => {
      const currentT = d3.zoomTransform(canvas);
      const t = scales.transform();
      if (t.k !== currentT.k || t.x !== currentT.x || t.y !== currentT.y) {
        d3.select(canvas as Element).call(
          zoom.transform,
          new d3.ZoomTransform(t.k, t.x, t.y)
        );
      }
    });

  const zoom = d3
    .zoom()
    .scaleExtent([0.1, 10])
    .on("zoom", (e) => {
      if (e.sourceEvent) {
        scales.transform(e.transform);
        drawFoodView();
      }
    });

  function getMaxes(markset: MarkRenderGroup<FoodMark>): number[] {
    let xMax: number = 0;
    let yMax: number = 0;
    markset.map((mark) => {
      xMax = mark.attr("x") > xMax ? mark.attr("x") : xMax;
      yMax = mark.attr("y") > yMax ? mark.attr("y") : yMax;
    });

    return [xMax, yMax];
  }

  function getMins(markset: MarkRenderGroup<FoodMark>): number[] {
    let xMin: number = 10000000;
    let yMin: number = 10000000;
    markset.map((mark) => {
      xMin = mark.attr("x") < xMin ? mark.attr("x") : xMin;
      yMin = mark.attr("y") < yMin ? mark.attr("y") : yMin;
    });

    return [xMin, yMin];
  }

  function createSummaryMarks(): MarkRenderGroup<SummaryMark> {
    let arr: Mark<SummaryMark>[] = [];
    let maxes = getMaxes(foodSet);
    let mins = getMins(foodSet);
    for (let i = 1; i < zoomedModeMarkCount ** 2 + 1; i++) {
      let set = foodSet.filter(
        (mark) =>
          mark.attr("x") <=
            mins[0] +
              (maxes[0] / zoomedModeMarkCount) *
                ((i % zoomedModeMarkCount) + 1) &&
          mark.attr("y") >=
            mins[1] +
              (maxes[1] / zoomedModeMarkCount) *
                Math.floor(i / zoomedModeMarkCount) &&
          mark.attr("y") <=
            mins[1] +
              (maxes[1] / zoomedModeMarkCount) *
                Math.floor(i / zoomedModeMarkCount) +
              1 &&
          mark.attr("x") >=
            mins[0] +
              (maxes[0] / zoomedModeMarkCount) * (i % zoomedModeMarkCount)
      );
      let setMark = new Mark<SummaryMark>(i, {
        x: new Attribute(
          10000 *
            Math.random() /* zoomedModeMarkCount) * (i % zoomedModeMarkCount) */
        ),
        y: new Attribute(
          10000 *
            Math.random() /* zoomedModeMarkCount) * Math.floor(i / zoomedModeMarkCount) */
        ),
        size: new Attribute(/*(set.count() / foodSet.count()) * 400000*/ 10000),
        xIndex: new Attribute(i % zoomedModeMarkCount),
        yIndex: new Attribute(Math.floor(i / zoomedModeMarkCount)),
      });
      arr.push(setMark);
    }

    return new MarkRenderGroup(arr);
  }
</script>

<main>
  <div id="visualization-box">
    {#key imagesLoaded}
      <div class="loading-screen" hidden={imagesLoaded}>Loading...</div>
    {/key}
    <svg id="axes" style="position: absolute; left: 20%; width: 70%;"></svg>
    <canvas bind:this={canvas}></canvas>
  </div>
</main>

<style>
  main {
    text-align: center;
    padding: 1em;
    max-width: 240px;
    margin: 0 auto;
  }
  canvas {
    z-index: 10;
    image-rendering: optimizeSpeed;
  }
  .loading-screen {
    background-color: white;
  }

  canvas,
  .loading-screen {
    position: absolute;
    bottom: 5%;
    left: 30%;
    /* width: 60%;
    height: 95%; */
  }
</style>
