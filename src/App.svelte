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
    time: Attribute<number>;
    placeholder: Attribute<number>;
  };

  type SummaryMark = {
    x: Attribute<number>;
    y: Attribute<number>;
    size: Attribute<number>;
    xIndex: Attribute<number>;
    yIndex: Attribute<number>;
    marks: Attribute<MarkRenderGroup<FoodMark>>;
  };

  let canvasHeight: number = 1000;
  let canvasWidth: number = 830;
  let gridSize: number = 5;

  let dataCSV: d3.DSVRowArray;
  let canvas: HTMLCanvasElement;
  let ticker: Ticker;
  let foodSet: MarkRenderGroup<FoodMark>;
  let summarySet: MarkRenderGroup<SummaryMark>;
  let imageCache: Record<string, HTMLImageElement> = {};
  let imagesLoaded: boolean = false;
  let imageCanvas: HTMLCanvasElement;
  let currentView: "food" | "ingredients" | "summary" = "food";
  let drawTransitionBegun: boolean = false;
  let animateOnlyVisibleMarks: boolean = false;
  let sampleSize: number = 1000;
  let renderLimit: number = 350;

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
          x - size <= canvas.clientWidth / 2 &&
          y - size <= canvas.clientHeight / 2
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
      x: new Attribute(0),
      y: new Attribute(0),
      img: { valueFn: () => imageCache[Number(id)] },
      ingredients: {
        valueFn: (mark) => {
          const ingArr = findIngredients(Number(id));
          mark.size = ingArr.length;
          return ingArr;
        },
      },
      size: { valueFn: (mark) => mark.size },
      time: { value: 10000 * Math.random() },
      placeholder: { value: 10000 * Math.random() },
    });

  function setupCanvas() {
    canvas.width = canvasWidth;
    canvas.height = canvasHeight;
    imageCanvas.height = canvas.height;
    imageCanvas.width = canvas.width;
    d3.select(canvas as Element)
      .on("click", handleClick)
      .call(zoom);
    drawMarks();
  }

  const drawMarks = () => {
    if (!dataCSV || !canvas) return;

    const ctx = canvas.getContext("2d", { willReadFrequently: true });
    if (!ctx) return;

    console.log(currentView);
    // let currentMarkGroup:
    //   | MarkRenderGroup<FoodMark>
    //   | MarkRenderGroup<SummaryMark> = foodSet;

    // switch (currentView) {
    //   case "food":
    //     currentMarkGroup = foodSet;
    //   case "summary":
    //     currentMarkGroup = summarySet;
    // }

    const transform = d3.zoomTransform(canvas);
    if (currentView === "food") {
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
      // if (visibleMarks.count() > 500 && !drawTransitionBegun) {
      if (
        visibleMarks.count() > renderLimit &&
        !drawTransitionBegun &&
        currentView === "food"
      ) {
        triggerSummaryView();
        drawTransitionBegun = true;
      } else {
        ctx.resetTransform();
        ctx.scale(window.devicePixelRatio, window.devicePixelRatio);
        ctx.clearRect(0, 0, canvas.clientWidth, canvas.clientHeight);
        visibleMarks.forEach((mark) => {
          const { x, y, img, size } = mark.get();
          const transformedX = Math.round(transform.applyX(x));
          const transformedY = Math.round(transform.applyY(y));
          if (img.src && img.complete) {
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
            ctx.drawImage(img, transformedX, transformedY, size * 2, size * 2);
            ctx.restore();
          }
        });
      }
    } else if (currentView === "summary") {
      ctx.resetTransform();
      ctx.scale(window.devicePixelRatio, window.devicePixelRatio);
      ctx.clearRect(0, 0, canvas.clientWidth, canvas.clientHeight);
      const zoomScale = transform.k; // Get the current zoom scale factor

      summarySet.forEach((mark) => {
        const { x, y, size } = mark.get();
        const transformedX = Math.round(transform.applyX(x));
        const transformedY = Math.round(transform.applyY(y));
        const scaledSize = (zoomScale / size) * 20000; // Adjust size based on zoom scale

        ctx.fillStyle = "blue";
        ctx.beginPath();
        ctx.arc(
          transformedX + scaledSize,
          transformedY + scaledSize,
          scaledSize,
          0,
          Math.PI * 2,
          true
        );
        ctx.fill();
        ctx.closePath();
      });

      drawTransitionBegun = false;
    }
  };

  const createSummaryMarks = (): MarkRenderGroup<SummaryMark> => {
    let arr: Mark<SummaryMark>[] = [];
    let maxes = getMaxes(foodSet);
    let mins = getMins(foodSet);

    for (let i = 0; i < gridSize; i++) {
      for (let j = 0; j < gridSize; j++) {
        let set = foodSet.filter((mark) => {
          const x = mark.attr("x");
          const y = mark.attr("y");
          return (
            x >= mins[0] + (maxes[0] / gridSize) * i &&
            x < mins[0] + (maxes[0] / gridSize) * (i + 1) &&
            y >= mins[1] + (maxes[1] / gridSize) * j &&
            y < mins[1] + (maxes[1] / gridSize) * (j + 1)
          );
        });

        let setMark = new Mark<SummaryMark>(`${i}${j}`, {
          x: new Attribute(mins[0] + (maxes[0] / gridSize) * i),
          y: new Attribute(mins[1] + (maxes[1] / gridSize) * j),
          size: new Attribute(
            ((set.count() / foodSet.count() + 0.3) * canvasWidth) / gridSize
          ),
          xIndex: new Attribute(i),
          yIndex: new Attribute(j),
          marks: new Attribute(set),
        });
        arr.push(setMark);
      }
    }

    return new MarkRenderGroup(arr);
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
      .domain([0, 1000])
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
    let xMin: number = Number.MAX_SAFE_INTEGER;
    let yMin: number = Number.MAX_SAFE_INTEGER;
    markset.map((mark) => {
      xMin = mark.attr("x") < xMin ? mark.attr("x") : xMin;
      yMin = mark.attr("y") < yMin ? mark.attr("y") : yMin;
    });

    return [xMin, yMin];
  }

  async function triggerSummaryView() {
    console.log(animateOnlyVisibleMarks);
    if (currentView === "food") {
      console.log("summary view triggered");
      const ctx = canvas.getContext("2d", { willReadFrequently: true });
      const transform = d3.zoomTransform(canvas);
      let animated = false;
      let promise = new Promise((resolve, reject) => {
        setTimeout(() => resolve("done!"), 2050);
      });
      if (ctx && transform) {
        summarySet.forEach((mark) => {
          // let marksToMove: MarkRenderGroup<FoodMark> = new MarkRenderGroup();
          // renderedMarks.forEach((m) => {
          //   if (mark.attr("marks").getMarks().includes(m)) marksToMove.addMark(m);
          // });

          // marksToMove
          //   .animateTo("x", mark.attr("x"), { duration: 2000 })
          //   .animateTo("y", mark.attr("y"), { duration: 2000 });
          // const { x, y, size } = mark.get();
          let animationMarks = animateOnlyVisibleMarks
            ? mark.attr("marks").filter((m) => {
                const x = transform.applyX(m.attr("x"));
                const y = transform.applyY(m.attr("y"));
                const size = 20;
                return (
                  x + size >= 0 &&
                  y + size >= 0 &&
                  x - size <= canvas.clientWidth / 2 &&
                  y - size <= canvas.clientHeight / 2
                );
              })
            : mark.attr("marks");

          animationMarks
            .animate("x")
            .animate("y")
            .animateTo("x", mark.attr("x"), { duration: 2000 })
            .animateTo("y", mark.attr("y"), { duration: 2000 });
        });
      }

      await promise;
      currentView = "summary";
      drawMarks();
      // foodSet
      //   .update("x", (mark) => mark.attr("time"))
      //   .update("y", (mark) => mark.attr("placeholder"));
    }
  }

  onMount(async () => {
    dataCSV = await d3.csv("src/datasets/food_data.csv");
    foodSet = new MarkRenderGroup(createMarkSet(sampleSize));
    foodSet
      .update("x", (m) => m.attr("time"))
      .update("y", (m) => m.attr("placeholder"));
    foodSet.configure({ animationDuration: 500 });
    summarySet = createSummaryMarks();
    summarySet.configure({ animationDuration: 1000 });
    await preloadImages(dataCSV, true);
    imageCanvas = document.createElement("canvas");
    imagesLoaded = true;
    let mins = getMins(foodSet);
    let maxes = getMaxes(foodSet);
    scales = new Scales({ animationDuration: 500 })
      .xRange([mins[0], maxes[0]])
      .yRange([mins[0], maxes[0]])
      .onUpdate(() => {
        const currentT = d3.zoomTransform(canvas);
        const t = scales.transform();
        if (t.k !== currentT.k || t.x !== currentT.x || t.y !== currentT.y) {
          d3.select(canvas as Element).call(
            zoom.transform,
            new d3.ZoomTransform(t.k, t.x, t.y)
          );
          createAxes();
        }
      });
    setupCanvas();
  });

  $: if (imagesLoaded) {
    ticker = new Ticker([foodSet, scales]).onChange(() => {
      requestAnimationFrame(drawMarks);
    });
    requestAnimationFrame(drawMarks);
    createAxes();
  }

  let scales: Scales;

  const zoom = d3
    .zoom()
    .scaleExtent([0.1, 10])
    .on("zoom", (e) => {
      if (e.sourceEvent) {
        scales.transform(e.transform);
        drawMarks();
      }
    });

  function createMarkSet(size: number): Mark<FoodMark>[] {
    if (!dataCSV) return [];

    const shuffled = dataCSV.sort(() => Math.random() - 0.5);
    return shuffled.slice(0, size).map((point) => createMark(point[""]));
  }

  async function triggerFoodView() {
    if (currentView === "summary") {
      console.log("food view triggered");
      const ctx = canvas.getContext("2d", { willReadFrequently: true });
      const transform = d3.zoomTransform(canvas);

      if (ctx && transform) {
        const visibleMarks = foodSet.filter((mark) => {
          const x = transform.applyX(mark.attr("x"));
          const y = transform.applyY(mark.attr("y"));
          const size = 20;
          return (
            x + size >= 0 &&
            y + size >= 0 &&
            x - size <= canvas.clientWidth / 2 &&
            y - size <= canvas.clientHeight / 2
          );
        });
        if (visibleMarks.count() < renderLimit / 4) {
          let promise = new Promise((resolve, reject) => {
            setTimeout(() => resolve("done!"), 2050);
          });
          currentView = "food";
          summarySet.forEach((mark) => {
            mark
              .attr("marks")
              .update("x", (m) => m.attr("time"))
              .update("y", (m) => m.attr("placeholder"));
          });
          let visibleSummarySet = summarySet.filter((mark) => {
            const x = transform.applyX(mark.attr("x"));
            const y = transform.applyY(mark.attr("y"));
            const size = 20;

            return (
              mark
                .attr("marks")
                .filter((m) => {
                  const x = transform.applyX(m.attr("x"));
                  const y = transform.applyY(m.attr("y"));
                  const size = 20;

                  return (
                    x + size >= 0 &&
                    y + size >= 0 &&
                    x - size <= canvas.clientWidth / 2 &&
                    y - size <= canvas.clientHeight / 2
                  );
                })
                .count() > 0 ||
              (x + size >= 0 &&
                y + size >= 0 &&
                x - size <= canvas.clientWidth / 2 &&
                y - size <= canvas.clientHeight / 2)
            );
          });

          visibleSummarySet.forEach((summaryMark) => {
            const { x, y, marks } = summaryMark.get();
            marks
              .update(
                "x",
                x + (Math.random() * 5 * Math.random() < 0.5 ? -1 : 1)
              )
              .update(
                "y",
                y + (Math.random() * 5 * Math.random() < 0.5 ? -1 : 1)
              );
          });

          visibleSummarySet.forEach((group) => {
            group
              .attr("marks")
              .filter((mark) => {
                const x = transform.applyX(mark.attr("x"));
                const y = transform.applyY(mark.attr("y"));
                const size = 40;
                return (
                  x + size >= 0 &&
                  y + size >= 0 &&
                  x - size <= canvas.clientWidth / 2 &&
                  y - size <= canvas.clientHeight / 2
                );
              })
              .animate("x")
              .animate("y")
              .animateTo("x", (m) => m.attr("time"), {
                duration: 2000,
              })
              .animateTo("y", (m) => m.attr("placeholder"), {
                duration: 2000,
              });
          });

          await promise;
        } else {
          alert("zoom in more, too many marks to render"); // try to optimize code so that this isn't neccesary
        }
      }
    }
  }

  function handleClick(e: MouseEvent) {
    const ctx = canvas.getContext("2d");

    let mousePos = [
      e.clientX - canvas.getBoundingClientRect().left,
      e.clientY - canvas.getBoundingClientRect().top,
    ];
    if (ctx) {
      if (currentView === "summary") {
        const pixel = ctx.getImageData(mousePos[0], mousePos[1], 1, 1).data;
        const color = `rgba(${pixel[0]}, ${pixel[1]}, ${pixel[2]}, ${pixel[3] / 255})`;

        if (color === "rgba(0, 0, 255, 1)") {
          triggerFoodView();
          // add specified hit detection maybe?
          // summarySet.filter((mark) => {
          //   let { x, y, size } = mark.get();
          //   return (
          //     Math.sqrt((mousePos[0] - x) ** 2 + (mousePos[1] - y) ** 2) <= size
          //   );
          // });
        } //might have to adjust based on color of summaryMarks (add adaptive?)
        console.log(color);
      } else if (currentView === "food") {
      }
    }
  }

  // placeholder/testing funcs
  function changeAnimationSettings() {
    animateOnlyVisibleMarks = !animateOnlyVisibleMarks;
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
  <button
    on:click={triggerSummaryView}
    style="position:absolute; top: 10%; left:10%;">Activate Summary View</button
  >
  <button
    on:click={triggerFoodView}
    style="position:absolute; top: 30%; left:10%;">Activate Food View</button
  >
  <button
    on:click={changeAnimationSettings}
    style="position:absolute; top: 40%; left:5%;"
    >Change animation of visible marks
  </button>
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
  }
</style>
