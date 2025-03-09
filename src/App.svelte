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
  // import ingredientsArray from "../ingredients_filtered.js";
  import { onMount } from "svelte";
  import * as fs from 'fs';

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
    instructions: Attribute<string>;
  };

  type SummaryMark = {
    x: Attribute<number>;
    y: Attribute<number>;
    size: Attribute<number>;
    xIndex: Attribute<number>;
    yIndex: Attribute<number>;
    marks: Attribute<MarkRenderGroup<FoodMarkAttrs>>;
  };

  type AxisOption = {
  id: string;
  label: string;
  valueFn: (mark: Mark<FoodMarkAttrs>) => number;
};

// Add this to your component's variables section
  let axisOptions: AxisOption[] = [
  { 
    id: "time", 
    label: "Preparation Time", 
    valueFn: (mark) => mark.attr("time") / 3
  },
  { 
    id: "placeholder", 
    label: "Randomized", 
    valueFn: (mark) => mark.attr("placeholder") / 3
  },
  { 
    id: "ingredientCount", 
    label: "Number of Ingredients", 
    valueFn: (mark) => mark.attr("ingredients").length
  },
  {
    id: "stepCount",
    label: "Number of steps in recipe",
    valueFn: (mark) => mark.attr("instructions").length / 3
  }
];

  let selectedXAxis: string = "time";
  let selectedYAxis: string = "stepCount";
  let selectedSize : string = "ingredientCount";



  let canvasHeight: number = 650;
  let canvasWidth: number = 900;
  let gridSize: number = 2;

  let dataCSV: d3.DSVRowArray;
  let canvas: HTMLCanvasElement;
  let ticker: Ticker;
  let foodSet: MarkRenderGroup<FoodMarkAttrs>;
  let summarySet: MarkRenderGroup<SummaryMark>;
  let imageCache: Record<string, HTMLImageElement> = {};
  let summaryPositionMap: PositionMap;
  let foodPositionMap : PositionMap;
  let imagesLoaded: boolean = false;
  let currentView: "food" | "ingredients" | "summary" | "frontPage" = "frontPage";
  let drawTransitionBegun: boolean = false;
  let animateOnlyVisibleMarks: boolean = false;
  let sampleSize: number = 200;
  let renderLimit: number = 300;
  let selectedIngredients : String[] = new Array;
  let fromFrontPage = true;
  let filteringDropdown : HTMLSelectElement | null;
  let ingredientsArray : string[] = [];

  type FilterOption = "onlyComplete" | "anyIngredient" | "allIngredients";
  let selectedFilterOption: FilterOption = "allIngredients";  
  let useSampleSize = true;

$: selectedFilterOption = filteringDropdown?.value as FilterOption;

fetch('src/cleanedIngredients.txt')
  .then(response => {
    if (!response.ok) {
      throw new Error('Failed to load ingredients file');
    }
    return response.text();
  })
  .then(text => {
    ingredientsArray = text.split('\n').filter(ingredient => ingredient.trim().length > 0);
    console.log(`Loaded ${ingredientsArray.length} ingredients`);
  })
  .catch(error => {
    console.error('Error loading ingredients:', error);
  });

  onMount(() => {
  if (canvas) {
    canvas.width = canvasWidth;
    canvas.height = canvasHeight;
    
    d3.select(canvas as Element)
      .on("click", handleClick)
      .call(zoom);
  }

  

  filteringDropdown = document.getElementById("dropdown") as HTMLSelectElement;
  
  setupIngredientSearchBar(
    "search-bar-front", 
    "ingredient-bar-front", 
    "entered-ingredients-box-front"
  );
  
  
});




function setupIngredientSearchBar(
  searchBarId: string, 
  ingredientBarId: string, 
  enteredIngredientsBoxId: string
) {
  const searchBar = document.getElementById(searchBarId) as HTMLInputElement;
  const ingredientBar = document.getElementById(ingredientBarId) as HTMLDivElement;
  const enteredIngredientsBox = document.getElementById(enteredIngredientsBoxId) as HTMLDivElement;

  if (searchBar) {
    searchBar.addEventListener('keydown', function(event) {
      if (document.activeElement === searchBar && event.key === 'Enter') {
        if (searchBar.value) {
          let barVal = searchBar.value.toLowerCase();
          if (ingredientsArray.includes(barVal)) {
            searchBar.value = "";
            selectedIngredients.push(barVal);
            let b = document.createElement('button');
            b.textContent = barVal;
            b.classList.add('result-added-button');
            b.style.backgroundColor = "#4CAF50";
            b.style.margin = "2px";
            b.style.padding = "5px";
            b.style.borderRadius = "3px";
            b.style.color = "white";
            enteredIngredientsBox.appendChild(b);
            foodSet = new MarkRenderGroup(createSortedSet(selectedIngredients, selectedFilterOption));
            
            b.onclick = () => {
              selectedIngredients = selectedIngredients.filter(ing => ing !== barVal);
              enteredIngredientsBox.removeChild(b);
              foodSet = new MarkRenderGroup(createSortedSet(selectedIngredients, selectedFilterOption));
            };

          } else {
            console.log("Ingredient not found");
          }
        }
      }
    });

    searchBar.addEventListener('input', function(event) {
      let buttons = document.getElementsByName("ingredient-result-button");
      buttons.forEach(element => {
        if (element.parentNode === ingredientBar) {
          ingredientBar.removeChild(element);
        }
      });
      
      let foundIngredients = ingredientsArray.filter(e => 
        e.includes(searchBar.value.toLowerCase())
      );

      if (foundIngredients.length < 10) {
        foundIngredients.forEach(element => {
          let result = document.createElement('button');
          result.name = "ingredient-result-button";
          result.textContent = String(element);
          result.onclick = () => {
            selectedIngredients.push(element);
            
            let b = document.createElement('button');
            b.textContent = String(element);
            b.classList.add('result-added-button');
            b.style.backgroundColor = "#4CAF50";
            b.style.margin = "2px";
            b.style.padding = "5px";
            b.style.borderRadius = "3px";
            b.style.color = "white";
            
            b.onclick = () => {
              selectedIngredients = selectedIngredients.filter(ing => ing !== element);
              enteredIngredientsBox.removeChild(b);
              foodSet = new MarkRenderGroup(createSortedSet(selectedIngredients, selectedFilterOption));
            };
            
            searchBar.value = "";
            ingredientBar.removeChild(result);
            
            enteredIngredientsBox.appendChild(b);
            
            console.log("Selected ingredients:", selectedIngredients);
          };

          ingredientBar.appendChild(result);
        });
      }
    });
  }

  if (currentView === "food") {
    selectedIngredients.forEach(element => {
      let b = document.createElement('button');
            b.textContent = String(element);
            b.classList.add('result-added-button');
            b.style.backgroundColor = "#4CAF50";
            b.style.margin = "2px";
            b.style.padding = "5px";
            b.style.borderRadius = "3px";
            b.style.color = "white";
            
            b.onclick = () => {
              selectedIngredients = selectedIngredients.filter(ing => ing !== element);
              enteredIngredientsBox.removeChild(b);
              foodSet = new MarkRenderGroup(createSortedSet(selectedIngredients, selectedFilterOption));
            };

            enteredIngredientsBox.appendChild(b);
    });
  }
}



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
    const ingredientsString = dataCSV[id]?.Ingredients ?? "" + dataCSV[id]?.Cleaned_Ingredients ?? "";
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
      instructions: {value: dataCSV[Number(id)]?.Instructions}
    });

    function createAxisControls() {
  const axisControlsDiv = document.createElement('div');
  axisControlsDiv.id = 'axis-controls';
  axisControlsDiv.style.position = 'absolute';
  axisControlsDiv.style.top = '10px';
  axisControlsDiv.style.left = '25%';
  axisControlsDiv.style.width = '70%';
  axisControlsDiv.style.display = 'flex';
  axisControlsDiv.style.justifyContent = 'space-between';
  axisControlsDiv.style.padding = '10px';
  axisControlsDiv.style.backgroundColor = 'rgba(255, 255, 255, 0.8)';
  axisControlsDiv.style.borderRadius = '5px';
  axisControlsDiv.style.zIndex = '15';
  
  const xAxisDiv = document.createElement('div');
  xAxisDiv.className = 'axis-selector';
  
  const xAxisLabel = document.createElement('label');
  xAxisLabel.textContent = 'X-Axis: ';
  xAxisLabel.htmlFor = 'x-axis-select';
  
  const xAxisSelect = document.createElement('select');
  xAxisSelect.id = 'x-axis-select';
  
  axisOptions.forEach(option => {
    const optionElement = document.createElement('option');
    optionElement.value = option.id;
    optionElement.textContent = option.label;
    optionElement.selected = option.id === selectedXAxis;
    xAxisSelect.appendChild(optionElement);
  });
  
  xAxisSelect.addEventListener('change', (event) => {
    const target = event.target as HTMLSelectElement;
    selectedXAxis = target.value;
    updateVisualization();
  });
  
  xAxisDiv.appendChild(xAxisLabel);
  xAxisDiv.appendChild(xAxisSelect);
  
  const yAxisDiv = document.createElement('div');
  yAxisDiv.className = 'axis-selector';
  
  const yAxisLabel = document.createElement('label');
  yAxisLabel.textContent = 'Y-Axis: ';
  yAxisLabel.htmlFor = 'y-axis-select';
  
  const yAxisSelect = document.createElement('select');
  yAxisSelect.id = 'y-axis-select';
  
  axisOptions.forEach(option => {
    const optionElement = document.createElement('option');
    optionElement.value = option.id;
    optionElement.textContent = option.label;
    optionElement.selected = option.id === selectedYAxis;
    yAxisSelect.appendChild(optionElement);
  });
  
  yAxisSelect.addEventListener('change', (event) => {
    const target = event.target as HTMLSelectElement;
    selectedYAxis = target.value;
    updateVisualization();
  });
  
  yAxisDiv.appendChild(yAxisLabel);
  yAxisDiv.appendChild(yAxisSelect);
  
  // Add Size Selector
  const sizeDiv = document.createElement('div');
  sizeDiv.className = 'axis-selector';
  
  const sizeLabel = document.createElement('label');
  sizeLabel.textContent = 'Mark Size: ';
  sizeLabel.htmlFor = 'size-select';
  
  const sizeSelect = document.createElement('select');
  sizeSelect.id = 'size-select';
  
  axisOptions.forEach(option => {
    const optionElement = document.createElement('option');
    optionElement.value = option.id;
    optionElement.textContent = option.label;
    optionElement.selected = option.id === selectedSize;
    sizeSelect.appendChild(optionElement);
  });
  
  sizeSelect.addEventListener('change', (event) => {
    const target = event.target as HTMLSelectElement;
    selectedSize = target.value;
    updateVisualization();
  });
  
  sizeDiv.appendChild(sizeLabel);
  sizeDiv.appendChild(sizeSelect);
  
  axisControlsDiv.appendChild(xAxisDiv);
  axisControlsDiv.appendChild(yAxisDiv);
  axisControlsDiv.appendChild(sizeDiv);
  
  document.body.appendChild(axisControlsDiv);
  
  return axisControlsDiv;
}

function updateVisualization() {
  if (!foodSet || !dataCSV) return;
  
  const xOption = axisOptions.find(opt => opt.id === selectedXAxis);
  const yOption = axisOptions.find(opt => opt.id === selectedYAxis);
  const sizeOption = axisOptions.find(opt => opt.id === selectedSize);
  
  if (!xOption || !yOption || !sizeOption) return;
  
  foodSet
    .animate("x")
    .animate("y")
    .animate("size")
    .animateTo("x", xOption.valueFn, { duration: 2000 })
    .animateTo("y", yOption.valueFn, { duration: 2000 })
    .animateTo("size", (mark) => {
      const rawValue = sizeOption.valueFn(mark);
      return Math.max(10, Math.min(40, rawValue / 10));
    }, { duration: 2000 });
  
  if (currentView === "summary") {
    const originalPositions = new Map();
    summarySet.forEach(mark => {
      originalPositions.set(mark.id, {
        x: mark.attr("x"),
        y: mark.attr("y")
      });
    });
    
    summarySet = createSummaryMarks();
    
    summarySet.forEach(mark => {
      const originalPos = originalPositions.get(mark.id);
      if (originalPos) {
        mark.attr("x", originalPos.x);
        mark.attr("y", originalPos.y);
        
        mark
          .animate("x")
          .animate("y")
          .animateTo("x", mark.attr("x"), { duration: 2000 })
          .animateTo("y", mark.attr("y"), { duration: 2000 });
      }
    });
    
    if (summaryPositionMap) {
      summaryPositionMap.invalidate();
      summaryPositionMap.add(summarySet);
    }
  }
  
  setTimeout(() => {
    if (foodPositionMap) {
      foodPositionMap.invalidate();
      foodPositionMap.add(foodSet);
    }
  }, 2000);
  
  let animationTimer = setInterval(() => {
    drawMarks();
  }, 16);
  
  setTimeout(() => {
    clearInterval(animationTimer);
    drawMarks(); // One final draw to ensure correct positions
  }, 2100);
  
  updateAxisLabels(xOption.label, yOption.label);
}


function updateAxisLabels(xLabel: string, yLabel: string) {
  // Remove any existing labels
  const existingXLabel = document.getElementById('x-axis-label');
  const existingYLabel = document.getElementById('y-axis-label');
  
  if (existingXLabel) existingXLabel.remove();
  if (existingYLabel) existingYLabel.remove();
  
  const xAxisLabel = document.createElement('div');
  xAxisLabel.id = 'x-axis-label';
  xAxisLabel.style.position = 'absolute';
  xAxisLabel.style.bottom = '10px';
  xAxisLabel.style.left = '50%';
  xAxisLabel.style.transform = 'translateX(-50%)';
  xAxisLabel.style.fontSize = '14px';
  xAxisLabel.style.fontWeight = 'bold';
  xAxisLabel.textContent = xLabel;
  
  // Create Y axis label
  const yAxisLabel = document.createElement('div');
  yAxisLabel.id = 'y-axis-label';
  yAxisLabel.style.position = 'absolute';
  yAxisLabel.style.top = '50%';
  yAxisLabel.style.left = '25%';
  yAxisLabel.style.transform = 'translateY(-50%) rotate(-90deg)';
  yAxisLabel.style.transformOrigin = 'left center';
  yAxisLabel.style.fontSize = '14px';
  yAxisLabel.style.fontWeight = 'bold';
  yAxisLabel.textContent = yLabel;
  
  document.body.appendChild(xAxisLabel);
  document.body.appendChild(yAxisLabel);
}

function setupCanvas() {
  console.log("Setting up canvas");
  
  if (!canvas) {
    console.error("Canvas element not found");
    return;
  }
  
  canvas.width = canvasWidth;
  canvas.height = canvasHeight;
  
  // Initialize the zoom with constraints based on the initial mark set
  if (foodSet) {
    const mins = getMins(foodSet);
    const maxes = getMaxes(foodSet);
    
    // Set the initial translate extent for the zoom behavior
    zoom.translateExtent([
      [mins[0] - (maxes[0] - mins[0]) * 0.2, mins[1] - (maxes[1] - mins[1]) * 0.2],
      [maxes[0] + (maxes[0] - mins[0]) * 0.2, maxes[1] + (maxes[1] - mins[1]) * 0.2]
    ]);
  }
  
  d3.select(canvas as Element)
    .on("click", handleClick)
    .call(zoom);
  
  if (currentView !== "frontPage") {
    const axisControls = createAxisControls();
    
    // Set initial axis labels
    const xOption = axisOptions.find(opt => opt.id === selectedXAxis);
    const yOption = axisOptions.find(opt => opt.id === selectedYAxis);
    
    if (xOption && yOption) {
      updateAxisLabels(xOption.label, yOption.label);
    }
  }
  
  console.log("Canvas setup complete");
}
const drawMarks = () => {
  
  
  if (!dataCSV || !canvas) return;

  const ctx = canvas.getContext("2d", { willReadFrequently: true });

  if (!ctx) return;

  const transform = d3.zoomTransform(canvas);
  if (currentView === "food") {
    if (foodPositionMap) foodPositionMap.invalidate();
    
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
    
    if (
      visibleMarks.count() > renderLimit &&
      !drawTransitionBegun &&
      currentView === "food"
    ) {
      console.log("Too many marks visible, triggering summary view");
      triggerSummaryView();
      drawTransitionBegun = true;
    } else {
      const groupedMarks = groupMarks(visibleMarks);
      ctx.resetTransform();
      ctx.scale(window.devicePixelRatio, window.devicePixelRatio);
      ctx.clearRect(0, 0, canvas.clientWidth, canvas.clientHeight);
      
      console.log("Drawing", groupedMarks.length, "mark groups");
      
      groupedMarks.forEach((group) => {
        ctx.save();
        group.forEach((mark) => {
          const { x, y, img, size } = mark.get();
          const transformedX = transform.applyX(x);
          const transformedY = transform.applyY(y);
          if (img && img.src && img.complete) {
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
          } else {
            ctx.beginPath();
            ctx.arc(
              transformedX + size,
              transformedY + size,
              size,
              0,
              Math.PI * 2,
              true
            );
            ctx.fillStyle = "lightgray";
            ctx.fill();
            ctx.closePath();
          }
        });
        ctx.restore();
      });
    }
  } else if (currentView === "summary") {
    if (summaryPositionMap) summaryPositionMap.invalidate();
    
    ctx.resetTransform();
    ctx.scale(window.devicePixelRatio, window.devicePixelRatio);
    ctx.clearRect(0, 0, canvas.clientWidth, canvas.clientHeight);
    const zoomScale = transform.k;  

    summarySet.forEach((mark) => {
      const { x, y, size } = mark.get();
      const transformedX = transform.applyX(x);
      const transformedY = transform.applyY(y);
      const scaledSize = (zoomScale / size) * 20000; 

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
}

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

      const mins = getMins(summarySet);
      const maxes = getMaxes(summarySet);
    
      zoom.translateExtent([
        [mins[0] - (maxes[0] - mins[0]) * 0.2, mins[1] - (maxes[1] - mins[1]) * 0.2],
        [maxes[0] + (maxes[0] - mins[0]) * 0.2, maxes[1] + (maxes[1] - mins[1]) * 0.2]
      ]);

      drawMarks();
      // foodSet
      //   .update("x", (mark) => mark.attr("time"))
      //   .update("y", (mark) => mark.attr("placeholder"));
      // implement zoom out
      // createAxes(
      //   transform.rescaleX(
      //     d3.scaleLinear().domain([0, canvasWidth]).range([0, 2000])
      //   ),
      //   transform.rescaleY(
      //     d3.scaleLinear().domain([0, canvasHeight]).range([2000, 0])
      //   )
      // );
    }
  }


  async function initVisualization() {
  if (!dataCSV) {
    return;
  }

  setupIngredientSearchBar(
    "search-bar", 
    "ingredient-bar", 
    "entered-ingredients-box"
  );

  selectedFilterOption = filteringDropdown?.value as FilterOption;
    
  foodSet = new MarkRenderGroup(createSortedSet(selectedIngredients, selectedFilterOption));
  
  const xOption = axisOptions.find(opt => opt.id === selectedXAxis);
  const yOption = axisOptions.find(opt => opt.id === selectedYAxis);
  const sizeOption = axisOptions.find(opt => opt.id === selectedSize);
  
  foodSet
    .update("x", xOption ? xOption.valueFn : (m) => m.attr("time"))
    .update("y", yOption ? yOption.valueFn : (m) => m.attr("placeholder"))
    .update("size", sizeOption ? 
      (mark) => {
        const rawValue = sizeOption.valueFn(mark);
        return Math.max(10, Math.min(40, rawValue / 10));
      } : (m) => 20);
  
  
  foodSet.configure({ animationDuration: 500 });

  summarySet = createSummaryMarks();
  summarySet.configure({ animationDuration: 1000 });
  
  console.log("Preloading images...");
  await preloadImages(dataCSV, true);
  
  imagesLoaded = true;
  
  let mins = getMins(foodSet);
  let maxes = getMaxes(foodSet);
  
  // Update zoom constraints based on mark boundaries
  zoom.translateExtent([
    [mins[0] - (maxes[0] - mins[0]) * 0.2, mins[1] - (maxes[1] - mins[1]) * 0.2],
    [maxes[0] + (maxes[0] - mins[0]) * 0.2, maxes[1] + (maxes[1] - mins[1]) * 0.2]
  ]);

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

  console.log("Creating position maps...");
  
  summaryPositionMap = new PositionMap({
    maximumHitTestDistance: 20,
  }).add(summarySet);
  
  foodPositionMap = new PositionMap({
    maximumHitTestDistance: 40,
  }).add(foodSet);
  
  foodSet.configure({
    hitTest: (mark, location) => {
      // const ZOOM_SCALE = 200 / (d3.zoomTransform(canvas).k * d3.zoomTransform(canvas).k);
      const x = mark.attr('x');
      const y = mark.attr('y');
      const size = mark.attr('size') * 2; 
      const distance = Math.sqrt(Math.pow(x - location[0], 2) + Math.pow(y - location[1], 2));
      return distance <= size;
    }
  });
  
  console.log("Setting up ticker...");
  
  ticker = new Ticker([foodSet, scales]).onChange(() => {
    requestAnimationFrame(drawMarks);
  });
  
  console.log("Initialization complete");
}


  $: if (imagesLoaded) {
    ticker = new Ticker([foodSet, scales]).onChange(() => {
      requestAnimationFrame(drawMarks);
    });
    summaryPositionMap = new PositionMap({
      maximumHitTestDistance: 20,
    }).add(summarySet);
    requestAnimationFrame(drawMarks);
    // createAxes(
    //   d3.scaleLinear().domain([0, canvasWidth]).range([0, 2000]),
    //   d3.scaleLinear().domain([0, canvasHeight]).range([2000, 0])
    // );

    foodPositionMap = new PositionMap({
      maximumHitTestDistance: 40,
    }).add(foodSet);
    foodSet.configure({
      hitTest: (mark, location) => {
        // const ZOOM_SCALE = 30 / d3.zoomTransform(canvas).k;
        let x = mark.attr('x');
        let y = mark.attr('y');
        let r = mark.attr('size') * 2;
        return Math.sqrt(Math.pow(x - location[0], 2.0) + Math.pow(y - location[1], 2.0)) <= r;
      }
    })
  }

  let scales: Scales;

  const zoom = d3
  .zoom()
  .scaleExtent([1, 1.5])
  .translateExtent([[0, 0], [canvasWidth, canvasHeight]]) // Basic canvas constraint
  .on("zoom", (e) => {
    if (e.sourceEvent) {
      // Get the current marks set depending on view
      const currentMarkSet = currentView === "food" ? foodSet : summarySet;
      
      // Apply constraints based on data bounds
      const constrainedTransform = constrainTransformToMarks(e.transform, currentMarkSet);
      
      // Update scales with constrained transform
      scales.transform(constrainedTransform);
      
      // Redraw with updated transform
      drawMarks();
    }
  });

  function constrainTransformToMarks(transform: d3.ZoomTransform, markSet: MarkRenderGroup<any>): d3.ZoomTransform {
  // Get the min/max boundaries of all marks
  const mins = getMins(markSet);
  const maxes = getMaxes(markSet);
  
  // Add padding (percentage of the range)
  const paddingX = (maxes[0] - mins[0]) * 0.2;
  const paddingY = (maxes[1] - mins[1]) * 0.2;
  
  // Calculate boundaries with padding
  const minX = mins[0] - paddingX;
  const maxX = maxes[0] + paddingX;
  const minY = mins[1] - paddingY;
  const maxY = maxes[1] + paddingY;
  
  // Calculate the visible width and height based on the current scale
  const visibleWidth = canvasWidth / transform.k;
  const visibleHeight = canvasHeight / transform.k;
  
  // Calculate the limits for the transform
  const minTransformX = -maxX + visibleWidth;
  const maxTransformX = -minX;
  const minTransformY = -maxY + visibleHeight;
  const maxTransformY = -minY;
  
  // Constrain the transform values
  let x = Math.min(maxTransformX, Math.max(minTransformX, transform.x));
  let y = Math.min(maxTransformY, Math.max(minTransformY, transform.y));
  
  // Return a new transform with constrained values
  return d3.zoomIdentity.translate(x, y).scale(transform.k);
}



  function createMarkSet(size: number): Mark<FoodMarkAttrs>[] {
    if (!dataCSV) return [];

    const shuffled = dataCSV.sort(() => Math.random() - 0.5);
    return shuffled.slice(0, size).map((point) => createMark(point[""]));
  }

  function createSortedSet(ingredientList: String[], filterSetting: FilterOption): Mark<FoodMarkAttrs>[] {
  if (!dataCSV || ingredientList.length === 0) return [];
  
  const matchingMarks: Mark<FoodMarkAttrs>[] = [];
  
  dataCSV.forEach((point) => {
    // Parse ingredients properly - check if it's already an array or a string that needs parsing
    let recipeIngredients: string[] = [];
    
    if (point.Ingredients) {
      // Check if Ingredients is a string that looks like a JSON array
      if (typeof point.Ingredients === 'string' && 
          (point.Ingredients.startsWith('[') || point.Ingredients.includes(','))) {
        try {
          // Try to parse as JSON if it starts with [
          if (point.Ingredients.startsWith('[')) {
            recipeIngredients = JSON.parse(point.Ingredients);
          } else {
            // Otherwise split by comma
            recipeIngredients = point.Ingredients.split(',');
          }
        } catch (e) {
          // Fallback to comma splitting if JSON parse fails
          recipeIngredients = point.Ingredients.split(',');
        }
      } else if (Array.isArray(point.Ingredients)) {
        // If it's already an array
        recipeIngredients = point.Ingredients;
      } else {
        // Handle as a single string
        recipeIngredients = [point.Ingredients];
      }
    }
    
    recipeIngredients = recipeIngredients.map(ing => 
      typeof ing === 'string' ? ing.toLowerCase().trim() : String(ing).toLowerCase().trim()
    );
    
    let isMatch = false;
    
    if (filterSetting === "allIngredients") {
      isMatch = ingredientList.every(ingredient => 
        recipeIngredients.some(recipeIng => 
          recipeIng.includes(ingredient.toLowerCase())
        )
      );
    } else if (filterSetting === "anyIngredient") {
      isMatch = ingredientList.some(ingredient => 
        recipeIngredients.some(recipeIng => 
          recipeIng.includes(ingredient.toLowerCase())
        )
      );
    } else {
      // exclude common ingredients or phrases that might appear in instructions
      const commonExceptions = ["dish", "salt", "pepper", "water", "oil", "garnish"];
      
      const significantIngredients = recipeIngredients.filter(ing => 
        !commonExceptions.some(exception => ing.includes(exception))
      );
      
      isMatch = significantIngredients.length > 0 && 
        significantIngredients.every(recipeIng => 
          ingredientList.some(ingredient => 
            recipeIng.includes(ingredient.toLowerCase())
          )
        );
    }
    
    if (isMatch) {
      matchingMarks.push(createMark(point[""]));
    }
  });

  // Apply sample size if set
  if (useSampleSize && matchingMarks.length > sampleSize) {
    return matchingMarks.sort(() => Math.random() - 0.5).slice(0, sampleSize);
  }
  
  return matchingMarks;
}

async function triggerFoodView() {
  if (currentView === "frontPage") {
    try {
      imagesLoaded = false;
      
      currentView = "food";
      
      if (!dataCSV) {
        console.log("Loading data for the first time...");
        dataCSV = await d3.csv("src/datasets/food_data.csv");
        
        // Extract data properties for axis options
        extractDataProperties();
      }
      
      console.log("Initializing visualization...");
      await initVisualization();
      
      setupCanvas();
      
      console.log("Drawing marks for the first time...");
      drawMarks();
      
      fromFrontPage = false;
    } catch (error) {
      console.error("Error initializing visualization:", error);
      currentView = "frontPage";
      imagesLoaded = false;
    }
  } else if (currentView === "summary") {
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
        const mins = getMins(foodSet);
        const maxes = getMaxes(foodSet);
    
        zoom.translateExtent([
          [mins[0] - (maxes[0] - mins[0]) * 0.2, mins[1] - (maxes[1] - mins[1]) * 0.2],
          [maxes[0] + (maxes[0] - mins[0]) * 0.2, maxes[1] + (maxes[1] - mins[1]) * 0.2]
        ]);
        // createAxes(
        //   transform.rescaleX(
        //     d3.scaleLinear().domain([0, canvasWidth]).range([0, 2000])
        //   ),
        //   transform.rescaleY(
        //     d3.scaleLinear().domain([0, canvasHeight]).range([2000, 0])
        //   )
        // );
      } else {
        alert("zoom in more, too many marks to render"); 
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
    
    // transform.k = 1 @ initial, 0.1 min and 10 max
  
    //console.log("Canvas coordinates:", [canvasX, canvasY]);
    //console.log("Data coordinates:", [dataX, dataY]);

    if (currentView === "summary") {
      const clickedMark = summaryPositionMap.hitTest([dataX, dataY]);
    
      if (clickedMark) {
        console.log("Clicked summary mark:", clickedMark);
      
        const containedMarks = clickedMark.attr("marks");
     
        const markBBox = markBox(containedMarks.getMarks());
      
        scales.zoomTo(markBBox);
      
  
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

function extractDataProperties() {
  if (dataCSV && foodSet) {
    const hasCalories = dataCSV.some(d => d.Calories !== undefined);
    if (hasCalories) {
      axisOptions.push({
        id: "calories",
        label: "Calories",
        valueFn: (mark) => {
          const id = Number(mark.id);
          return Number(dataCSV[id]?.Calories) || 0;
        }
      });
    }
    
  }
}




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
  <div id="visualization-box">
    {#key imagesLoaded}
      <div class="loading-screen" hidden={imagesLoaded || frontPage()}>Loading...</div>
    {/key}
    <svg id="axes" style="position: absolute; left: 20%; width: 70%;" class:hidden={currentView === "frontPage"}></svg>
    <canvas bind:this={canvas} style="position:absolute; left: 25%;" class:hidden={currentView === "frontPage"}></canvas>
  </div>

  {#if currentView === "food" || currentView === "summary"}
    <button
      on:click={triggerSummaryView}
      style="position:absolute; top: 10%; left:10%;">Activate Summary View</button>
    <button
      on:click={triggerFoodView}
      style="position:absolute; top: 30%; left:10%;">Activate Food View</button>
    <button
      on:click={changeAnimationSettings}
      style="position:absolute; top: 40%; left:5%;"
      >Change animation of visible marks
    </button>

    <div id="clicked-mark-box" style="position:absolute; top: 50%; left:1%; width: 50px;"></div>

    <div id="ingredient-bar" style="position:absolute; top: 60%; right:0.01%; width: 150px;">
      <input id="search-bar" type="text" placeholder="Enter more ingredients...">
      <div id="entered-ingredients-box" style="max-width: 150px;"></div>
    </div>
  {/if}

  {#if currentView === "frontPage"}
    <div class="front-page" style="z-index: 11;">
      <h1>Welcome to FoodTable!</h1>
      <h2>Click the button below to enter (will add more to this page dw)</h2>
      <div id="ingredient-bar-front">
        <input id="search-bar-front" type="text" placeholder="Enter an ingredient...">
        <div id="entered-ingredients-box-front"></div>
      </div>
      <button on:click={triggerFoodView}>Enter the tool</button>
      <select id="dropdown">
        <option value="onlyComplete"> Only show recipes for which you've entered all ingredients</option>
        <option value="anyIngredient"> Show all recipes that feature an entered ingredient</option>
        <option value="allIngredients"> Only show recipes that feature ALL entered ingredients</option>
      </select>
    </div>
  {/if}
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

  .hidden {
    display: none !important;
  }

  #entered-ingredients-box {
    position: absolute;
    top: -100px;
    left: 0;
    right: 0;
    padding: 10px;
    display: flex;
    flex-direction: column-reverse;
    gap: 10px;
    height: 50px;
    width: 200px;
  }

#entered-ingredients-box, #entered-ingredients-box-front {
  display: flex;
  flex-wrap: wrap;
  gap: 5px;
  margin-top: 10px;
  min-height: 30px;
  max-width: 500px;
}

.result-added-button {
  display: inline-block;
  background-color: #4CAF50;
  color: white;
  padding: 5px 10px;
  margin: 2px;
  border: none;
  border-radius: 3px;
  cursor: pointer;
}

.result-added-button:hover {
  background-color: #45a049;
}

#ingredient-bar, #ingredient-bar-front {
  position: relative;
  width: 200px;
  margin-bottom: 20px;
}

#search-bar, #search-bar-front {
  width: 100%;
  padding: 8px;
  margin-bottom: 5px;
}

[name="ingredient-result-button"] {
  background-color: white;
  border: 1px solid #ddd;
  padding: 5px;
  margin: 2px 0;
  width: 100%;
  text-align: left;
  cursor: pointer;
}

[name="ingredient-result-button"]:hover {
  background-color: #f1f1f1;
}

.axis-selector {
    display: flex;
    align-items: center;
    margin: 0 10px;
  }
  
  .axis-selector label {
    margin-right: 8px;
    font-weight: bold;
  }
  
  .axis-selector select {
    padding: 5px;
    min-width: 150px;
    border-radius: 4px;
    border: 1px solid #ddd;
  }
  
  #axis-controls {
    background-color: rgba(255, 255, 255, 0.9);
    border: 1px solid #ddd;
    border-radius: 5px;
    padding: 8px;
    box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
  }
  
  #x-axis-label, #y-axis-label {
    color: #333;
    background-color: rgba(255, 255, 255, 0.7);
    padding: 2px 6px;
    border-radius: 3px;
  }
</style>