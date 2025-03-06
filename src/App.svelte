<script lang="ts">
  import {
    Mark,
    Scales,
    Ticker,
    MarkRenderGroup,
    Attribute,
    markBox,
    PositionMap,
  } from "counterpoint-vis";
  import * as d3 from "d3";
  import ingredientsArray from "../ingredients_filtered.js";
  import { onMount } from "svelte";

  type FoodMarkAttrs = {
    title: Attribute<string>;
    x: Attribute<number>;
    y: Attribute<number>;
    img: Attribute<HTMLImageElement>;
    ingredients: Attribute<string[]>;
    size: Attribute<number>;
    time: Attribute<number>;
    placeholder: Attribute<number>;
    name: Attribute<string>;
  };

  type SummaryMark = {
    x: Attribute<number>;
    y: Attribute<number>;
    size: Attribute<number>;
    xIndex: Attribute<number>;
    yIndex: Attribute<number>;
    marks: Attribute<MarkRenderGroup<FoodMarkAttrs>>;
  };

  let canvasHeight: number = 650;
  let canvasWidth: number = 900;
  let gridSize: number = 5;

  let dataCSV: d3.DSVRowArray;
  let canvas: HTMLCanvasElement;
  let ticker: Ticker;
  let foodSet: MarkRenderGroup<FoodMarkAttrs>;
  let summarySet: MarkRenderGroup<SummaryMark>;
  let imageCache: Record<string, HTMLImageElement> = {};
  let summaryPositionMap: PositionMap;
  let foodPositionMap : PositionMap;
  let imagesLoaded: boolean = false;
  let imageCanvas: HTMLCanvasElement;
  let currentView: "food" | "ingredients" | "summary" | "frontPage" = "frontPage";
  let drawTransitionBegun: boolean = false;
  let animateOnlyVisibleMarks: boolean = false;
  let sampleSize: number = 200;
  let renderLimit: number = 201;
  let searchBar: HTMLInputElement;
  let ingredientBar : HTMLDivElement;
  let selectedIngredients : String[] = new Array;




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
        const size = mark.attr("size");
        return (
          x + size >= 0 &&
          y + size >= 0 &&
          x + size <= canvas.clientWidth / 2 &&
          y + size <= canvas.clientHeight / 2
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

  const createMark = (id: string): Mark<FoodMarkAttrs> =>
    new Mark<FoodMarkAttrs>(id, {
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
      size: new Attribute(0),
      time: { value: 1000 * Math.random() },
      placeholder: { value: 1000 * Math.random() },
      name: {value: dataCSV[Number(id)]?.Title ?? "No Name"},
    });

  function setupCanvas() {
    canvas.width = canvasWidth;
    canvas.height = canvasHeight;
    imageCanvas.height = canvas.height;
    imageCanvas.width = canvas.width;
    d3.select(canvas as Element)
      .on("click", handleClick)
      .call(zoom);
    // drawMarks();
  }

  const drawMarks = () => {
    if (!dataCSV || !canvas) return;

    const ctx = canvas.getContext("2d", { willReadFrequently: true });
    if (!ctx) return;


    // let currentMarkGroup:
    //   | MarkRenderGroup<FoodMarkAttrs>
    //   | MarkRenderGroup<SummaryMark> = foodSet;

    // switch (currentView) {
    //   case "food":
    //     currentMarkGroup = foodSet;
    //   case "summary":
    //     currentMarkGroup = summarySet;
    // }

    const transform = d3.zoomTransform(canvas);
    if (currentView === "food") {
      foodPositionMap.invalidate();
      const visibleMarks = foodSet.filter((mark) => {
        const x = transform.applyX(mark.attr("x"));
        const y = transform.applyY(mark.attr("y"));
        const size = mark.attr("size");
        return (
          x + size >= 0 &&
          y + size >= 0 &&
          x + size <= canvas.clientWidth &&
          y + size <= canvas.clientHeight
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
        const groupedMarks = groupMarks(visibleMarks);
        ctx.resetTransform();
        ctx.scale(window.devicePixelRatio, window.devicePixelRatio);
        ctx.clearRect(0, 0, canvas.clientWidth, canvas.clientHeight);
        groupedMarks.forEach((group) => {
          ctx.save();
          group.forEach((mark) => {
            const { x, y, img, size } = mark.get();
            const transformedX = transform.applyX(x);
            const transformedY = transform.applyY(y);
            if (img.src && img.complete) {
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
              ctx.drawImage(
                img,
                transformedX,
                transformedY,
                size * 2,
                size * 2
              );              
            }
          });
          ctx.restore();
        });
      }
    } else if (currentView === "summary") {
      summaryPositionMap.invalidate();
      ctx.resetTransform();
      ctx.scale(window.devicePixelRatio, window.devicePixelRatio);
      ctx.clearRect(0, 0, canvas.clientWidth, canvas.clientHeight);
      const zoomScale = transform.k; // Get the current zoom scale factor

      summarySet.forEach((mark) => {
        const { x, y, size } = mark.get();
        const transformedX = transform.applyX(x);
        const transformedY = transform.applyY(y);
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

  const groupMarks = (marks: MarkRenderGroup<FoodMarkAttrs>) => {
    const groupSize = 100;
    const groups = [];
    let currentGroup: Mark<FoodMarkAttrs>[] = [];

    marks.forEach((mark) => {
      if (
        currentGroup.length === 0 ||
        shouldGroupTogether(currentGroup[0], mark, groupSize)
      ) {
        currentGroup.push(mark);
      } else {
        groups.push(currentGroup);
        currentGroup = [mark];
      }
    });
    if (currentGroup.length > 0) {
      groups.push(currentGroup);
    }

    return groups;
  };
  const shouldGroupTogether = (
    markA: Mark<FoodMarkAttrs>,
    markB: Mark<FoodMarkAttrs>,
    size: number
  ) => {
    const xDiff = Math.abs(markA.attr("x") - markB.attr("x"));
    const yDiff = Math.abs(markA.attr("y") - markB.attr("y"));
    return xDiff < size && yDiff < size;
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

  const createAxes = (
    xScale: d3.ScaleLinear<number, number, never>,
    yScale: d3.ScaleLinear<number, number, never>
  ) => {
    const svg = d3.select("#axes");
    const marginTop = 60,
      marginRight = 60,
      marginBottom = 60,
      marginLeft = 60;
    const width = window.innerWidth,
      height = window.innerHeight;

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

  function getMaxes(
    markset: MarkRenderGroup<FoodMarkAttrs> | MarkRenderGroup<SummaryMark>
  ): number[] {
    let xMax: number = 0;
    let yMax: number = 0;
    if (markset == summarySet) {
      markset.forEach((mark) => {
        xMax = mark.attr("x") > xMax ? mark.attr("x") : xMax;
        yMax = mark.attr("y") > yMax ? mark.attr("y") : yMax;
      });
    } else if (markset == foodSet) {
      markset.forEach((mark) => {
        xMax = mark.attr("x") > xMax ? mark.attr("x") : xMax;
        yMax = mark.attr("y") > yMax ? mark.attr("y") : yMax;
      });
    }

    return [xMax, yMax];
  }

  function getMins(
    markset: MarkRenderGroup<FoodMarkAttrs> | MarkRenderGroup<SummaryMark>
  ): number[] {
    let xMin: number = Number.MAX_SAFE_INTEGER;
    let yMin: number = Number.MAX_SAFE_INTEGER;
    if (markset == foodSet) {
      markset.map((mark) => {
        xMin = mark.attr("x") < xMin ? mark.attr("x") : xMin;
        yMin = mark.attr("y") < yMin ? mark.attr("y") : yMin;
      });
    } else if (markset == summarySet) {
      markset.map((mark) => {
        xMin = mark.attr("x") < xMin ? mark.attr("x") : xMin;
        yMin = mark.attr("y") < yMin ? mark.attr("y") : yMin;
      });
    }

    return [xMin, yMin];
  }

  async function triggerSummaryView() {
    if (currentView === "food") {
      const ctx = canvas.getContext("2d", { willReadFrequently: true });
      const transform = d3.zoomTransform(canvas);
      let animated = false;
      let promise = new Promise((resolve, reject) => {
        setTimeout(() => resolve("done!"), 2050);
      });
      if (ctx && transform) {
        summarySet.forEach((mark) => {
          // let marksToMove: MarkRenderGroup<FoodMarkAttrs> = new MarkRenderGroup();
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
                const size = m.attr("size");
                return (
                  x + size >= 0 &&
                  y + size >= 0 &&
                  x + size <= canvas.clientWidth / 2 &&
                  y + size <= canvas.clientHeight / 2
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
      // implement zoom out
      createAxes(
        transform.rescaleX(
          d3.scaleLinear().domain([0, canvasWidth]).range([0, 2000])
        ),
        transform.rescaleY(
          d3.scaleLinear().domain([0, canvasHeight]).range([2000, 0])
        )
      );
    }
  }

  onMount(async () => {
    dataCSV = await d3.csv("src/datasets/food_data.csv");
    foodSet = new MarkRenderGroup(createMarkSet(sampleSize));
    foodSet
      .update("x", (m) => m.attr("time"))
      .update("y", (m) => m.attr("placeholder"))
      .update("size", (m) => 20);
      // .update("size", (m) => m.attr("ingredients").length);
    foodSet.configure({ animationDuration: 500 });

    // foodSet.forEach((m) => console.log(m.attr("name") + ", " + (m.attr("x") + ", " + m.attr("y"))));
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
        }
        if (!!summaryPositionMap) summaryPositionMap.invalidate();
        if (!!foodPositionMap) foodPositionMap.invalidate();
      });

    setupCanvas();

    searchBar = document.getElementById("search-bar") as HTMLInputElement;
    ingredientBar = document.getElementById("ingredient-bar") as HTMLDivElement;
    let enteredIngredientsBox = document.getElementById("entered-ingredients-box") as HTMLDivElement;

    searchBar.addEventListener('keydown', function(event) {
    if (document.activeElement === searchBar && event.key === 'Enter') {
    console.log("key down");
    if (searchBar?.value) 
    {
      console.log("bar");
      if (ingredientsArray.includes(searchBar.value.toLowerCase())) 
      {
        console.log("Ingredient found"); // add ingredient below search bar
        searchBar.value = "";
      } else 
      {
        console.log("Ingredient not found"); // add error message below search bar
      }
    }
  }
});

  searchBar.addEventListener('input', function(event) {

    let buttons = document.getElementsByName("ingredient-result-button");
    
    buttons.forEach(element => {
      ingredientBar.removeChild(element);
    });
    let foundIngredients = ingredientsArray.filter(e => e.includes(searchBar.value));

    if (foundIngredients.length < 10) {
      foundIngredients.forEach(element => {
      let result = document.createElement('button');
      result.name = "ingredient-result-button";
      result.textContent = String(element);
      result.onclick = () => {
        let b = document.createElement('button');
        b.textContent = String(element) + " added to list";
        b.classList.add('result-added-button');
        b.style.backgroundColor = "red";
        b.style.position = "absoulte";
        selectedIngredients.push(element);
        ingredientBar.removeChild(result);
        enteredIngredientsBox.appendChild(b);
        
      };

      ingredientBar.appendChild(result);
    });
    }
  })



  });

  $: if (imagesLoaded) {
    ticker = new Ticker([foodSet, scales]).onChange(() => {
      requestAnimationFrame(drawMarks);
    });
    summaryPositionMap = new PositionMap({
      maximumHitTestDistance: 20,
    }).add(summarySet);
    requestAnimationFrame(drawMarks);
    createAxes(
      d3.scaleLinear().domain([0, canvasWidth]).range([0, 2000]),
      d3.scaleLinear().domain([0, canvasHeight]).range([2000, 0])
    );

    foodPositionMap = new PositionMap({
      maximumHitTestDistance: 20,
    }).add(foodSet);
    foodSet.configure({
      hitTest: (mark, location) => {
        let x = mark.attr('x');
        let y = mark.attr('y');
        let r = mark.attr('size');
        return Math.sqrt(Math.pow(x - location[0], 2.0) + Math.pow(y - location[1], 2.0)) <= r;
      }
    })
  }

  let scales: Scales;

  const zoom = d3
    .zoom()
    .scaleExtent([0.1, 10])
    .on("zoom", (e) => {
      if (e.sourceEvent) {
        scales.transform(e.transform);
        drawMarks();
        const transform = d3.zoomTransform(canvas);
        createAxes(
          transform.rescaleX(
            d3.scaleLinear().domain([0, canvasWidth]).range([0, 2000])
          ),
          transform.rescaleY(
            d3.scaleLinear().domain([0, canvasHeight]).range([2000, 0])
          )
        );
      }
    });

  function createMarkSet(size: number): Mark<FoodMarkAttrs>[] {
    if (!dataCSV) return [];

    const shuffled = dataCSV.sort(() => Math.random() - 0.5);
    return shuffled.slice(0, size).map((point) => createMark(point[""]));
  }

  async function triggerFoodView() {
    if (currentView === "summary") {
      const ctx = canvas.getContext("2d", { willReadFrequently: true });
      const transform = d3.zoomTransform(canvas);

      if (ctx && transform) {
        const visibleMarks = foodSet.filter((mark) => {
          const x = transform.applyX(mark.attr("x"));
          const y = transform.applyY(mark.attr("y"));
          const size = mark.attr("size");
          return (
            x + size >= 0 &&
            y + size >= 0 &&
            x + size <= canvas.clientWidth &&
            y + size <= canvas.clientHeight
          );
        });
        if (visibleMarks.count() < renderLimit * 0.9) {
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
            const size = mark.attr("size");

            return (
              mark
                .attr("marks")
                .filter((m) => {
                  const x = transform.applyX(m.attr("x"));
                  const y = transform.applyY(m.attr("y"));
                  const size = m.attr("size");

                  return (
                    x + size >= 0 &&
                    y + size >= 0 &&
                    x + size <= canvas.clientWidth &&
                    y + size <= canvas.clientHeight
                  );
                })
                .count() > 0 ||
              (x + size >= 0 &&
                y + size >= 0 &&
                x + size <= canvas.clientWidth &&
                y + size <= canvas.clientHeight)
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
          // implement zoom in
          createAxes(
            transform.rescaleX(
              d3.scaleLinear().domain([0, canvasWidth]).range([0, 2000])
            ),
            transform.rescaleY(
              d3.scaleLinear().domain([0, canvasHeight]).range([2000, 0])
            )
          );
        } else {
          alert("zoom in more, too many marks to render"); // try to optimize code so that this isn't neccesary
        }
      }
    }
  }

  function handleClick(event: MouseEvent) {
    const rect = canvas.getBoundingClientRect();
    // let xRange = (450 + 19);
    // let yRange = (17 - 311);

    const X_SCALAR = 0.5;
    const Y_SCALAR = 0.5;


    const scaleX = (canvas.width / rect.width);
    const scaleY = canvas.height / rect.height;

  
    const canvasX = (((event.clientX  - rect.left) * X_SCALAR) - 19.76) * scaleX;
    const canvasY = (((event.clientY - rect.bottom) * Y_SCALAR) + 305) * scaleY;
  
    const transform = d3.zoomTransform(canvas);
    const [dataX, dataY] = transform.invert([canvasX, canvasY]);
  
    console.log("Canvas coordinates:", [canvasX, canvasY]);
    //console.log("Data coordinates:", [dataX, dataY]);

    // Depending on the current view, use the appropriate PositionMap for hit testing
    if (currentView === "summary") {
      const clickedMark = summaryPositionMap.hitTest([dataX, dataY]);
    
      if (clickedMark) {
        console.log("Clicked summary mark:", clickedMark);
      
        // Get the marks contained within the summary mark
        const containedMarks = clickedMark.attr("marks");
        // Calculate the bounding box of these marks to zoom to
        const markBBox = markBox(containedMarks.getMarks());
      
        // Zoom to this area with some padding
        scales.zoomTo(markBBox);
      
        // Switch to food view after zoom animation completes
        setTimeout(() => {
          triggerFoodView();
        }, 1200);
    }
  } else if (currentView === "food") {
    const clickedMark = foodPositionMap.hitTest([dataX, dataY]);
    
    if (clickedMark) {
      console.log("Food item:", clickedMark.attr("name"));
      let clickedMarkDisplayBox = document.getElementById("clicked-mark-box");
      if (clickedMarkDisplayBox) {
        clickedMarkDisplayBox.innerHTML = " ";

        let imgSrc = clickedMark.attr('img').src;
        let img = document.createElement('img');
        let p = document.createElement('p');
        let textNode = document.createTextNode("Clicked Recipe : " + clickedMark.attr("name"));
        p.appendChild(textNode);
        img.src = imgSrc;
        img.width /= 2;
        img.height /= 2;

        clickedMarkDisplayBox.appendChild(img);
        clickedMarkDisplayBox.appendChild(p);

      } 
    }
  }
}



  // placeholder/testing funcs
  function changeAnimationSettings() {
    animateOnlyVisibleMarks = !animateOnlyVisibleMarks;
  }

  function frontPage() : boolean {
    return currentView === "frontPage";
  }

  function foodView() : boolean {
    return currentView === "food";
  }

</script>

<main>
<div hidden={!foodView}>
  <div id="visualization-box">
    {#key imagesLoaded}
      <div class="loading-screen" hidden={imagesLoaded}>Loading...</div>
    {/key}
    <svg id="axes" style="position: absolute; left: 20%; width: 70%;"></svg>
    <canvas bind:this={canvas} style="position:absolute; left: 25%;"></canvas>
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

  <div id="clicked-mark-box" style="position:absolute; top: 50%; left:1%; width: 50px;"></div>

  <div id="ingredient-bar" style="position:absolute; top: 60%; right:0.01%; width: 150px;">
    <input id="search-bar" type="text" placeholder="Enter an ingredient...">
    <div id="entered-ingredients-box"></div>
    </div>
  </div>

  <div class="front-page" hidden={!frontPage} style="z-index: 11;">
    <head>
      <h1>Welcome to FoodTable!</h1>
      <h2>Enter your ingredients to begin!</h2>
      <button on:click={triggerFoodView}>Enter the tool</button>
    </head>
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
    z-index: 5;
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

  #entered-ingredients-box {
    position: absolute;  /* Position it above #searchBar */
      top: -100px;          /* Positioned above the search bar */
      left: 0;
      right: 0;
      padding: 10px;
      display: flex;
      flex-direction: column-reverse; /* Flexbox to make elements go up */
      gap: 10px;
      height: 50px;
      width: 200px;
  }

  
</style>