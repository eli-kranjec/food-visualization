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
  import { GoogleGenerativeAI } from "@google/generative-ai";
  let API_KEY = "AIzaSyDwT4v3B07BoeitWC6VWqbYVHcrc6mpvww"; // ill make this private later

  const genAI = new GoogleGenerativeAI(API_KEY);
  const model = genAI.getGenerativeModel({ model: "gemini-1.5-flash" });

  async function AISummary(mark : Mark<FoodMarkAttrs>) : Promise<string> 
  {
    const recipeId = Number(mark.id);
  
    const recipe = dataCSV[recipeId];
  
    const ings = recipe.Ingredients;
    const prompt = "Estimate the nutritional information of this recipe given these ingredients, and provide a brief summary of it. Do not use any formatting, send a pure paragraph of text. Answer with confidence, and use the word aproximately or around if unsure of specifics.\n" + ings;
    const result = await model.generateContent(prompt);

    return result.response.text();
  }

  async function AISuggest(ingredients: string[]) : Promise<string>
  {
    const prompt = "Come up with a suggestion for a recipe to make, given these ingredients\n" + ingredients;
    const result = await model.generateContent(prompt);

    return result.response.text();
  }

  let hoveredMarkRef : Mark<FoodMarkAttrs> | null = null;



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
    umapX: Attribute<number>;
    umapY: Attribute<number>;
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

  let axisOptions: AxisOption[] = [
  { 
    id: "time", 
    label: "Preparation Time", 
    valueFn: (mark) => mark.attr("time") * 3
  },
  { 
    id: "placeholder", 
    label: "Randomized", 
    valueFn: (mark) => mark.attr("placeholder") / 3
  },
  { 
    id: "ingredientCount", 
    label: "Number of Ingredients", 
    valueFn: (mark) => mark.attr("ingredients").length * 3
  },
  {
    id: "stepCount",
    label: "Number of steps in recipe",
    valueFn: (mark) => mark.attr("instructions").length / 3
  },
  {
    id: "umapX",
    label: "Dimension 1",
    valueFn: (mark) => mark.attr("umapX") * 3
  },
  {
    id: "umapY",
    label: "Dimension 2",
    valueFn: (mark) => mark.attr("umapY") * 3
  }
];

  let selectedXAxis: string = "time";
  let selectedYAxis: string = "stepCount";
  let selectedSize : string = "ingredientCount";



  let canvasHeight: number = 650;
  let canvasWidth: number = 900;
  let gridSize: number = 2;

  let dataCSV: d3.DSVRowArray;
  let umapCSV: d3.DSVRowArray;
  let canvas: HTMLCanvasElement;
  let ticker: Ticker;
  let foodSet: MarkRenderGroup<FoodMarkAttrs>;
  let summarySet: MarkRenderGroup<SummaryMark>;
  let imageCache: Record<string, HTMLImageElement | null> = {};
  let imageLoadingPromises: Record<string, Promise<HTMLImageElement>> = {};
  let summaryPositionMap: PositionMap;
  let foodPositionMap : PositionMap;
  let imagesLoaded: boolean = false;
  let currentView: "food" | "ingredients" | "summary" | "frontPage" = "frontPage";
  let drawTransitionBegun: boolean = false;
  let animateOnlyVisibleMarks: boolean = false;
  let sampleSize: number = 50;
  let renderLimit: number = 300;
  let selectedIngredients : String[] = new Array;
  let fromFrontPage = true;
  let filteringDropdown : HTMLSelectElement | null;
  let ingredientsArray : string[] = [];

  type FilterOption = "onlyComplete" | "anyIngredient" | "allIngredients";
  let selectedFilterOption: FilterOption = "allIngredients";  
  let useSampleSize = true;

  let enteredIngredientsButtons : HTMLButtonElement[] = [];

$: selectedFilterOption = filteringDropdown?.value as FilterOption;

const basePath = import.meta.env.BASE_URL || '/';

fetch(`${basePath}cleanedIngredients.txt`)
  .then(response => {
    if (!response.ok) {
      throw new Error('Failed to load ingredients file');
    }
    return response.text();
  })
  .then(text => {
    ingredientsArray = text.split('\n').filter(ingredient => ingredient.trim().length > 0);
    // console.log(`Loaded ${ingredientsArray.length} ingredients`);
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
            if (currentView != "frontPage") enteredIngredientsButtons.push(b);
            
            showLoadingIndicator();
            
            setTimeout(() => {
              foodSet = new MarkRenderGroup(createSortedSet(selectedIngredients, selectedFilterOption));
              
              // Preload images for the newly created food marks
              preloadImages(dataCSV, true).then(() => {
                updateVisualizationSmoothly();
                hideLoadingIndicator();
              });
            }, 100);
            
            b.onclick = () => {
              selectedIngredients = selectedIngredients.filter(ing => ing !== barVal);
              enteredIngredientsBox.removeChild(b);
              if (currentView != "frontPage") enteredIngredientsButtons.splice(enteredIngredientsButtons.indexOf(b), 1);

              showLoadingIndicator();
              
              setTimeout(() => {
                foodSet = new MarkRenderGroup(createSortedSet(selectedIngredients, selectedFilterOption));
                
                // Preload images for the newly created food marks
                preloadImages(dataCSV, true).then(() => {
                  updateVisualizationSmoothly();
                  hideLoadingIndicator();
                });
              }, 100);
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

            if (currentView != "frontPage") enteredIngredientsButtons.push(b);
            
            b.onclick = () => {
              selectedIngredients = selectedIngredients.filter(ing => ing !== element);
              enteredIngredientsBox.removeChild(b);
              if (currentView != "frontPage") enteredIngredientsButtons.splice(enteredIngredientsButtons.indexOf(b), 1);
              
              showLoadingIndicator();
              
              setTimeout(() => {
                foodSet = new MarkRenderGroup(createSortedSet(selectedIngredients, selectedFilterOption));
                updateVisualizationSmoothly();
                
                hideLoadingIndicator();
              }, 100);
            };
            
            searchBar.value = "";
            ingredientBar.removeChild(result);
            
            if (currentView != "frontPage") enteredIngredientsBox.appendChild(b);
            
            showLoadingIndicator();
            
            setTimeout(() => {
              foodSet = new MarkRenderGroup(createSortedSet(selectedIngredients, selectedFilterOption));
              
              // Preload images for the newly created food marks
              preloadImages(dataCSV, true).then(() => {
                updateVisualizationSmoothly();
                hideLoadingIndicator();
              });
            }, 100);
            
            // console.log("Selected ingredients:", selectedIngredients);
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
        
        showLoadingIndicator();
        
        setTimeout(() => {
          foodSet = new MarkRenderGroup(createSortedSet(selectedIngredients, selectedFilterOption));
          
          // Preload images for the newly created food marks
          preloadImages(dataCSV, true).then(() => {
            updateVisualizationSmoothly();
            hideLoadingIndicator();
          });
        }, 100);
      };

      enteredIngredientsBox.appendChild(b);
    });
  }
}

function showLoadingIndicator() {
  let loadingIndicator = document.getElementById('loading-indicator');
  
  if (!loadingIndicator) {
    loadingIndicator = document.createElement('div');
    loadingIndicator.id = 'loading-indicator';
    loadingIndicator.textContent = 'Loading...';
    loadingIndicator.style.position = 'absolute';
    loadingIndicator.style.top = '50%';
    loadingIndicator.style.left = '50%';
    loadingIndicator.style.transform = 'translate(-50%, -50%)';
    loadingIndicator.style.padding = '10px';
    loadingIndicator.style.backgroundColor = 'rgba(0, 0, 0, 0.7)';
    loadingIndicator.style.color = 'white';
    loadingIndicator.style.borderRadius = '5px';
    loadingIndicator.style.zIndex = '1000';
    document.body.appendChild(loadingIndicator);
  } else {
    loadingIndicator.style.display = 'block';
  }
  
  const ctx = canvas.getContext('2d', { willReadFrequently: true });
  if (ctx) {
    ctx.save();
    ctx.globalAlpha = 0.5;
    ctx.fillStyle = 'rgba(255, 255, 255, 0.3)';
    ctx.fillRect(0, 0, canvas.width, canvas.height);
    ctx.restore();
  }
}

function hideLoadingIndicator() {
  const loadingIndicator = document.getElementById('loading-indicator');
  if (loadingIndicator) {
    loadingIndicator.style.display = 'none';
  }
}

function updateVisualizationSmoothly() {
  if (!foodSet || !dataCSV) return;
  
  const xOption = axisOptions.find(opt => opt.id === selectedXAxis);
  const yOption = axisOptions.find(opt => opt.id === selectedYAxis);
  const sizeOption = axisOptions.find(opt => opt.id === selectedSize);
  
  if (!xOption || !yOption || !sizeOption) return;
  
  zoom.scaleExtent([1, Math.sqrt(1 + Math.log10(foodSet.count()))]);
  
  foodSet.configure({ animationDuration: 500 });
  
  foodSet
    .animate("x")
    .animate("y")
    .animate("size")
    .animateTo("x", xOption.valueFn, { duration: 800 })
    .animateTo("y", yOption.valueFn, { duration: 800 })
    .animateTo("size", (mark) => {
      const rawValue = sizeOption.valueFn(mark);
      return Math.max(10, Math.min(40, rawValue / 10));
    }, { duration: 800 });
  
  setTimeout(() => {
    if (foodPositionMap) {
      foodPositionMap.invalidate();
      foodPositionMap.add(foodSet);
    }
    
    const mins = getMins(foodSet);
    const maxes = getMaxes(foodSet);
    
    zoom.translateExtent([
      [mins[0] - (maxes[0] - mins[0]) * 0.2, mins[1] - (maxes[1] - mins[1]) * 0.2],
      [maxes[0] + (maxes[0] - mins[0]) * 0.2, maxes[1] + (maxes[1] - mins[1]) * 0.2]
    ]);
    
    let animationTimer = setInterval(() => {
      requestAnimationFrame(drawMarks);
    }, 16);
    
    setTimeout(() => {
      clearInterval(animationTimer);
      requestAnimationFrame(drawMarks); 
    }, 850);
    
  }, 100);
  
  updateAxisLabels(xOption.label, yOption.label);
}

function getCanvasEdgeCoordinates() {
  if (!canvas) return null;
  
  const rect = canvas.getBoundingClientRect();
  
  const canvasLeft = rect.left;
  const canvasTop = rect.top;
  const canvasRight = rect.right;
  const canvasBottom = rect.bottom;
  
  type EdgePoint = {
    clientX: number;
    clientY: number;
    canvasX?: number;
    canvasY?: number;
    dataX?: number;
    dataY?: number;
  };
  
  type EdgeCoordinates = {
    topLeft: EdgePoint;
    topRight: EdgePoint;
    bottomLeft: EdgePoint;
    bottomRight: EdgePoint;
  };
  
  const edgeCoordinates: EdgeCoordinates = {
    topLeft: {
      clientX: canvasLeft,
      clientY: canvasTop
    },
    topRight: {
      clientX: canvasRight,
      clientY: canvasTop
    },
    bottomLeft: {
      clientX: canvasLeft,
      clientY: canvasBottom
    },
    bottomRight: {
      clientX: canvasRight,
      clientY: canvasBottom
    }
  };
  
  const transform = d3.zoomTransform(canvas);
  const X_SCALAR = 1 / window.devicePixelRatio;
    const Y_SCALAR = 1 / window.devicePixelRatio;
  const scaleX = (canvas.width / rect.width);
  const scaleY = (canvas.height / rect.height);
  
  const edges: (keyof EdgeCoordinates)[] = ['topLeft', 'topRight', 'bottomLeft', 'bottomRight'];
  
  edges.forEach(edge => {
    const point = edgeCoordinates[edge];
    
    const canvasX = (((point.clientX - rect.left) * X_SCALAR) - 19.76) * scaleX;
    const canvasY = (((point.clientY - rect.bottom) * Y_SCALAR) + 305) * scaleY;
    
    const [dataX, dataY] = transform.invert([canvasX, canvasY]);
    
    point.canvasX = canvasX;
    point.canvasY = canvasY;
    point.dataX = dataX;
    point.dataY = dataY;
  });
  
  return edgeCoordinates;
}

function getCanvasEdgeMaximums() {
  const edges = getCanvasEdgeCoordinates();
  if (!edges) return null;
  
  return {
    maxClientX: Math.max(edges.topLeft.clientX, edges.topRight.clientX, 
                         edges.bottomLeft.clientX, edges.bottomRight.clientX),
    maxClientY: Math.max(edges.topLeft.clientY, edges.topRight.clientY, 
                         edges.bottomLeft.clientY, edges.bottomRight.clientY),
    maxCanvasX: Math.max(
      edges.topLeft.canvasX!, edges.topRight.canvasX!, 
      edges.bottomLeft.canvasX!, edges.bottomRight.canvasX!
    ),
    maxCanvasY: Math.max(
      edges.topLeft.canvasY!, edges.topRight.canvasY!, 
      edges.bottomLeft.canvasY!, edges.bottomRight.canvasY!
    ),
    maxDataX: Math.max(
      edges.topLeft.dataX!, edges.topRight.dataX!, 
      edges.bottomLeft.dataX!, edges.bottomRight.dataX!
    ),
    maxDataY: Math.max(
      edges.topLeft.dataY!, edges.topRight.dataY!, 
      edges.bottomLeft.dataY!, edges.bottomRight.dataY!
    )
  };
}

function getCanvasEdgeBounds() {
  if (!canvas) return null;
  
  const rect = canvas.getBoundingClientRect();
  
  const canvasLeft = rect.left;
  const canvasTop = rect.top;
  const canvasRight = rect.right;
  const canvasBottom = rect.bottom;
  
  type EdgePoint = {
    clientX: number;
    clientY: number;
    canvasX?: number;
    canvasY?: number;
    dataX?: number;
    dataY?: number;
  };
  
  type EdgeCoordinates = {
    topLeft: EdgePoint;
    topRight: EdgePoint;
    bottomLeft: EdgePoint;
    bottomRight: EdgePoint;
  };
  
  const edgeCoordinates: EdgeCoordinates = {
    topLeft: {
      clientX: canvasLeft,
      clientY: canvasTop
    },
    topRight: {
      clientX: canvasRight,
      clientY: canvasTop
    },
    bottomLeft: {
      clientX: canvasLeft,
      clientY: canvasBottom
    },
    bottomRight: {
      clientX: canvasRight,
      clientY: canvasBottom
    }
  };
  
  const transform = d3.zoomTransform(canvas);
  const X_SCALAR = 1 / window.devicePixelRatio;
    const Y_SCALAR = 1 / window.devicePixelRatio;
  const scaleX = (canvas.width / rect.width);
  const scaleY = (canvas.height / rect.height);
  
  const edges: (keyof EdgeCoordinates)[] = ['topLeft', 'topRight', 'bottomLeft', 'bottomRight'];
  
  edges.forEach(edge => {
    const point = edgeCoordinates[edge];
    
    const canvasX = (((point.clientX - rect.left) * X_SCALAR) - 19.76) * scaleX;
    const canvasY = (((point.clientY - rect.bottom) * Y_SCALAR) + 305) * scaleY;
    
    const [dataX, dataY] = transform.invert([canvasX, canvasY]);
    
    point.canvasX = canvasX;
    point.canvasY = canvasY;
    point.dataX = dataX;
    point.dataY = dataY;
  });
  
  return {
    client: {
      minX: Math.min(
        edgeCoordinates.topLeft.clientX, edgeCoordinates.topRight.clientX, 
        edgeCoordinates.bottomLeft.clientX, edgeCoordinates.bottomRight.clientX
      ),
      maxX: Math.max(
        edgeCoordinates.topLeft.clientX, edgeCoordinates.topRight.clientX, 
        edgeCoordinates.bottomLeft.clientX, edgeCoordinates.bottomRight.clientX
      ),
      minY: Math.min(
        edgeCoordinates.topLeft.clientY, edgeCoordinates.topRight.clientY, 
        edgeCoordinates.bottomLeft.clientY, edgeCoordinates.bottomRight.clientY
      ),
      maxY: Math.max(
        edgeCoordinates.topLeft.clientY, edgeCoordinates.topRight.clientY, 
        edgeCoordinates.bottomLeft.clientY, edgeCoordinates.bottomRight.clientY
      )
    },
    canvas: {
      minX: Math.min(
        edgeCoordinates.topLeft.canvasX!, edgeCoordinates.topRight.canvasX!, 
        edgeCoordinates.bottomLeft.canvasX!, edgeCoordinates.bottomRight.canvasX!
      ),
      maxX: Math.max(
        edgeCoordinates.topLeft.canvasX!, edgeCoordinates.topRight.canvasX!, 
        edgeCoordinates.bottomLeft.canvasX!, edgeCoordinates.bottomRight.canvasX!
      ),
      minY: Math.min(
        edgeCoordinates.topLeft.canvasY!, edgeCoordinates.topRight.canvasY!, 
        edgeCoordinates.bottomLeft.canvasY!, edgeCoordinates.bottomRight.canvasY!
      ),
      maxY: Math.max(
        edgeCoordinates.topLeft.canvasY!, edgeCoordinates.topRight.canvasY!, 
        edgeCoordinates.bottomLeft.canvasY!, edgeCoordinates.bottomRight.canvasY!
      )
    },
    data: {
      minX: Math.min(
        edgeCoordinates.topLeft.dataX!, edgeCoordinates.topRight.dataX!, 
        edgeCoordinates.bottomLeft.dataX!, edgeCoordinates.bottomRight.dataX!
      ),
      maxX: Math.max(
        edgeCoordinates.topLeft.dataX!, edgeCoordinates.topRight.dataX!, 
        edgeCoordinates.bottomLeft.dataX!, edgeCoordinates.bottomRight.dataX!
      ),
      minY: Math.min(
        edgeCoordinates.topLeft.dataY!, edgeCoordinates.topRight.dataY!, 
        edgeCoordinates.bottomLeft.dataY!, edgeCoordinates.bottomRight.dataY!
      ),
      maxY: Math.max(
        edgeCoordinates.topLeft.dataY!, edgeCoordinates.topRight.dataY!, 
        edgeCoordinates.bottomLeft.dataY!, edgeCoordinates.bottomRight.dataY!
      )
    },
    points: edgeCoordinates
  };
}


function getVisibleMarks(transform: d3.ZoomTransform): MarkRenderGroup<FoodMarkAttrs> {
  if (!canvas || !foodSet) return [] as any;

  const bounds = getCanvasEdgeBounds();
  if (!bounds) return [] as any;
  
  const viewportRect = {
    x1: bounds.data.minX,
    y1: bounds.data.minY,
    x2: bounds.data.maxX,
    y2: bounds.data.maxY
  };
  
  return foodSet.filter((mark) => {
    const x = mark.attr("x");
    const y = mark.attr("y");
    const size = mark.attr("size");
    
    return (
      x + size >= viewportRect.x1 - size &&
      y + size >= viewportRect.y1 - size &&
      x - size <= viewportRect.x2 + size &&
      y - size <= viewportRect.y2 + size
    );
  });
}



  function loadImageOnDemand(id: number): HTMLImageElement | null {
    if (imageCache[id] !== undefined) {
      return imageCache[id];
    }
    
    imageCache[id] = null;
    
    if (!imageLoadingPromises[id]) {
      imageLoadingPromises[id] = new Promise<HTMLImageElement>((resolve, reject) => {
        if (!dataCSV || !dataCSV[id]) {
          resolve(new Image()); 
          return;
        }
        
        const img = new Image();
        
        img.onload = () => {
          imageCache[id] = img;
          resolve(img);
          requestAnimationFrame(drawMarks);
        };
        
        img.onerror = () => {
          // On error, use a placeholder image
          const placeholderImg = new Image();
          placeholderImg.src = `${basePath}placeholder_images/no-image-found.jpg`;
          placeholderImg.onload = () => {
            imageCache[id] = placeholderImg;
            resolve(placeholderImg);
            requestAnimationFrame(drawMarks);
          };
          placeholderImg.onerror = () => {
            imageCache[id] = new Image();
            resolve(new Image());
          };
        };
        
        img.src = dataCSV[id].Image_Name === "#NAME?"
          ? `${basePath}placeholder_images/no-image-found.jpg`
          : `${basePath}datasets/Food Images/${dataCSV[id].Image_Name}.jpg`;
      });
    }
    
    return null;
  }

  function preloadImages(
    dataCSV: d3.DSVRowArray,
    initialSetup: boolean = false
  ): Promise<void[]> {
    if (initialSetup) {
      const transform = d3.zoomTransform(canvas);
      const visibleMarks = getVisibleMarks(transform);
      
      return Promise.all(
        visibleMarks.map((mark) => {
          const id = Number(mark.id);
          return new Promise<void>((resolve) => {
            if (!imageCache[id]) {
              loadImageOnDemand(id);
            }
            
            const checkImage = () => {
              if (imageCache[id] && imageCache[id]?.complete) {
                resolve();
              } else {
                setTimeout(checkImage, 50);
              }
            };
            
            checkImage();
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
      img: { 
        valueFn: () => {
          return loadImageOnDemand(Number(id));
        }
      },
      ingredients: {
        valueFn: (mark) => {
          const ingArr = findIngredients(Number(id));
          mark.size = ingArr.length;
          return ingArr;
        },
      },
      size: new Attribute(0),
      time: { valueFn: (mark) => extractCookingTime(mark.id)},
      placeholder: { value: 1000 * Math.random() },
      name: {value: dataCSV[Number(id)]?.Title ?? "No Name"},
      instructions: {value: dataCSV[Number(id)]?.Instructions},
      umapX : {value: Number(umapCSV[Number(id)]?.umap_x)},
      umapY : {value: Number(umapCSV[Number(id)]?.umap_y)}
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
  
  const mins = getMins(foodSet);
  const maxes = getMaxes(foodSet);
    
  zoom.translateExtent([
    [mins[0] - (maxes[0] - mins[0]) * 0.2, mins[1] - (maxes[1] - mins[1]) * 0.2],
    [maxes[0] + (maxes[0] - mins[0]) * 0.2, maxes[1] + (maxes[1] - mins[1]) * 0.2]
    ]);
  
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
    requestAnimationFrame(drawMarks);
  }, 16);
  
  setTimeout(() => {
    clearInterval(animationTimer);
    requestAnimationFrame(drawMarks); 
  }, 2100);
  
  updateAxisLabels(xOption.label, yOption.label);
}


function updateAxisLabels(xLabel: string, yLabel: string) {
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
  // console.log("Setting up canvas");
  
  if (!canvas) {
    console.error("Canvas element not found");
    return;
  }
  
  canvas.width = canvasWidth;
  canvas.height = canvasHeight;
  
  if (foodSet) {
    const mins = getMins(foodSet);
    const maxes = getMaxes(foodSet);
    
    zoom.translateExtent([
      [mins[0] - (maxes[0] - mins[0]) * 0.2, mins[1] - (maxes[1] - mins[1]) * 0.2],
      [maxes[0] + (maxes[0] - mins[0]) * 0.2, maxes[1] + (maxes[1] - mins[1]) * 0.2]
    ]);
  }
  
  d3.select(canvas as Element)
    .on("click", handleClick)
    .on("mousemove", handleHover)
    .on("mouseleave", () => {
      hoveredMarkRef = null;
      canvas.style.cursor = 'default';
      drawMarks();
    })
    .call(zoom);
  
  if (currentView !== "frontPage") {
    const axisControls = createAxisControls();
    
    const xOption = axisOptions.find(opt => opt.id === selectedXAxis);
    const yOption = axisOptions.find(opt => opt.id === selectedYAxis);
    
    if (xOption && yOption) {
      updateAxisLabels(xOption.label, yOption.label);
    }
    
    ensureSidebarExists();
  }
}

function ensureSidebarExists() {
  let sidebar = document.getElementById("recipe-details-sidebar");
  
  if (!sidebar) {
    console.log("Creating new recipe details sidebar");
    sidebar = document.createElement('div');
    sidebar.id = 'recipe-details-sidebar';
    sidebar.className = 'details-sidebar';
    
    const clickedMarkBox = document.createElement('div');
    clickedMarkBox.id = 'clicked-mark-box';
    
    sidebar.appendChild(clickedMarkBox);
    document.body.appendChild(sidebar);
  } else {
    console.log("Recipe details sidebar already exists");
  }
  
  return sidebar;
}
const drawMarks = () => {
  if (!canvas) return;
  
  const ctx = canvas.getContext("2d", { willReadFrequently: true });
  if (!ctx) return;
  
  const transform = d3.zoomTransform(canvas);
  
  if (dataCSV && currentView === "food" && foodSet) {
    if (foodPositionMap) foodPositionMap.invalidate();
    
    const visibleMarks = getVisibleMarks(transform);
    
    // Preload images for visible marks that aren't already loading
    visibleMarks.forEach(mark => {
      const id = Number(mark.id);
      if (imageCache[id] === undefined) {
        loadImageOnDemand(id);
      }
    });
    
    if (
      visibleMarks.count() > renderLimit &&
      !drawTransitionBegun &&
      currentView === "food"
    ) {
      triggerSummaryView();
      drawTransitionBegun = true;
    } else {
      if (visibleMarks.count() > 0) {
        ctx.resetTransform();
        ctx.scale(window.devicePixelRatio, window.devicePixelRatio);
        ctx.clearRect(0, 0, canvas.clientWidth, canvas.clientHeight);
        
        const groupedMarks = groupMarks(visibleMarks);
        
        groupedMarks.forEach((group) => {
          ctx.save();
          group.forEach((mark) => {
            const { x, y, img, size } = mark.get();
            const sizeMultiplier = hoveredMarkRef && mark.id === hoveredMarkRef.id ? 1.2 : 1;
            const adjustedSize = size * sizeMultiplier;
            
            const transformedX = transform.applyX(x);
            const transformedY = transform.applyY(y);
            
            ctx.beginPath();
            ctx.arc(
              transformedX + adjustedSize,
              transformedY + adjustedSize,
              adjustedSize,
              0,
              Math.PI * 2,
              true
            );
            ctx.closePath();
            
            if (img && img.src && img.complete) {
              ctx.clip();
              ctx.drawImage(
                img,
                transformedX,
                transformedY,
                adjustedSize * 2,
                adjustedSize * 2
              );
            } else {
              ctx.fillStyle = "lightgray";
              ctx.fill();
              
              if (!imageCache[Number(mark.id)]) {
                loadImageOnDemand(Number(mark.id));
              }
            }
          });
          ctx.restore();
        });
      }
    }
  } else if (currentView === "summary" && summarySet) {
    if (summaryPositionMap) summaryPositionMap.invalidate();
    
    ctx.resetTransform();
    ctx.scale(window.devicePixelRatio, window.devicePixelRatio);
    ctx.clearRect(0, 0, canvas.clientWidth, canvas.clientHeight);
    const zoomScale = transform.k;  

    summarySet.forEach((mark) => {
      const { x, y, size } = mark.get();
      const sizeMultiplier = hoveredMarkRef && mark.id === hoveredMarkRef.id ? 1.2 : 1;
      
      const transformedX = transform.applyX(x);
      const transformedY = transform.applyY(y);
      const scaledSize = (zoomScale / size) * 20000 * sizeMultiplier; 

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

  let enteredIngredientsBox = document.getElementById("entered-ingredients-box");

  var children = enteredIngredientsBox?.children;
  if (children) 
  {
    for (var i = 0; i < children.length; i++) {
      enteredIngredientsButtons.push(children[i] as HTMLButtonElement);
    }
  }

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
  
  // console.log("Preloading images...");
  await preloadImages(dataCSV, true);
  
  imagesLoaded = true;
  
  let mins = getMins(foodSet);
  let maxes = getMaxes(foodSet);
  
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

  // console.log("Creating position maps...");
  
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
  
  // console.log("Setting up ticker...");
  
  ticker = new Ticker([foodSet, scales]).onChange(() => {
    requestAnimationFrame(drawMarks);
  });
  
  // console.log("Initialization complete");
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
  .translateExtent([[0, 0], [canvasWidth, canvasHeight]])
  .on("zoom", (e) => {
    if (e.sourceEvent) {
      const currentMarkSet = currentView === "food" ? foodSet : summarySet;
      const constrainedTransform = constrainTransformToMarks(e.transform, currentMarkSet);
      scales.transform(constrainedTransform);
      
      requestAnimationFrame(drawMarks);
    }
  });

  function extractCookingTime(id: string): number {
  const recipeId = Number(id);
  
  const recipe = dataCSV[recipeId];
  
  if (!recipe) {
    return 0; 
  }
  
  const instructions = recipe.Instructions;
  
  if (!instructions) {
    return 0; 
  }
  
  const timeRegex = /(\d+)[-–](\d+)\s*minutes|(\d+)\s*minutes|(\d+)\s*minute/gi;
  let matches = [...instructions.matchAll(timeRegex)];
  
  if (matches.length === 0) {
    return 0;
  }
  
  let cookingTimes = [];
  for (const match of matches) {
    if (match[1] && match[2]) {
      cookingTimes.push((parseInt(match[1]) + parseInt(match[2])) / 2);
    } else if (match[3]) {
      cookingTimes.push(parseInt(match[3]));
    } else if (match[4]) {
      cookingTimes.push(parseInt(match[4]));
    }
  }
  
  return cookingTimes.reduce((sum, time) => sum + time, 0);
}

  function constrainTransformToMarks(transform: d3.ZoomTransform, markSet: MarkRenderGroup<any>): d3.ZoomTransform {
  const mins = getMins(markSet);
  const maxes = getMaxes(markSet);
  
  const paddingX = (maxes[0] - mins[0]) * 0.2;
  const paddingY = (maxes[1] - mins[1]) * 0.2;
  
  const minX = mins[0] - paddingX;
  const maxX = maxes[0] + paddingX;
  const minY = mins[1] - paddingY;
  const maxY = maxes[1] + paddingY;
  
  const visibleWidth = canvasWidth / transform.k;
  const visibleHeight = canvasHeight / transform.k;
  
  const minTransformX = -maxX + visibleWidth;
  const maxTransformX = -minX;
  const minTransformY = -maxY + visibleHeight;
  const maxTransformY = -minY;
  
  let x = Math.min(maxTransformX, Math.max(minTransformX, transform.x));
  let y = Math.min(maxTransformY, Math.max(minTransformY, transform.y));
  
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
  
  const isExactIngredientMatch = (recipeIng: string, searchIngredient: string): boolean => {
    searchIngredient = searchIngredient.toLowerCase().trim();
    recipeIng = recipeIng.toLowerCase().trim();
    
    if (recipeIng === searchIngredient) return true;
   
    const pattern = new RegExp(`\\b${searchIngredient}\\b`, 'i');
    if (pattern.test(recipeIng)) return true;
    

    if (searchIngredient.endsWith('s') && 
        recipeIng === searchIngredient.slice(0, -1)) return true;
    if (recipeIng.endsWith('s') && 
        searchIngredient === recipeIng.slice(0, -1)) return true;
    
    return false;
  };
  
  dataCSV.forEach((point) => {
    let recipeIngredients: string[] = [];
    
    if (point.Ingredients) {
      if (typeof point.Ingredients === 'string' && 
          (point.Ingredients.startsWith('[') || point.Ingredients.includes(','))) {
        try {
          if (point.Ingredients.startsWith('[')) {
            recipeIngredients = JSON.parse(point.Ingredients);
          } else {
            recipeIngredients = point.Ingredients.split(',');
          }
        } catch (e) {
          recipeIngredients = point.Ingredients.split(',');
        }
      } else if (Array.isArray(point.Ingredients)) {
        recipeIngredients = point.Ingredients;
      } else {
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
          isExactIngredientMatch(recipeIng, ingredient.toString())
        )
      );
    } else if (filterSetting === "anyIngredient") {
      isMatch = ingredientList.some(ingredient => 
        recipeIngredients.some(recipeIng => 
          isExactIngredientMatch(recipeIng, ingredient.toString())
        )
      );
    } else {
      const commonExceptions = ["dish", "salt", "pepper", "water", "oil", "garnish"];
      
      const significantIngredients = recipeIngredients.filter(ing => 
        !commonExceptions.some(exception => isExactIngredientMatch(ing, exception))
      );
      
      isMatch = significantIngredients.length > 0 && 
        significantIngredients.every(recipeIng => 
          ingredientList.some(ingredient => 
            isExactIngredientMatch(recipeIng, ingredient.toString())
          )
        );
    }
    
    if (isMatch) {
      matchingMarks.push(createMark(point[""]));
    }
  });

  if (useSampleSize && matchingMarks.length > sampleSize) {
    return matchingMarks.sort(() => Math.random() - 0.5).slice(0, sampleSize);
  }

  if (matchingMarks.length === 0) return createMarkSet(sampleSize);
  
  return matchingMarks;
}

async function triggerFoodView() {
  if (currentView === "frontPage") {
    try {
      imagesLoaded = false;
      
      currentView = "food";
      
      if (!dataCSV) {
        dataCSV = await d3.csv(`${basePath}datasets/food_data.csv`);
        umapCSV = await d3.csv(`${basePath}datasets/food_data_umap.csv`);
        
        extractDataProperties();
      }
      
      await initVisualization();
      
      setupCanvas();
      
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
      const visibleMarks = getVisibleMarks(transform);

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
    let tooltip = document.getElementById("recipe-tooltip");

    if (tooltip) tooltip.style.opacity = "0";

    const rect = canvas.getBoundingClientRect();

    console.log(event.x - rect.left, event.y - rect.bottom);
    // let xRange = (450 + 19);
    // let yRange = (17 - 311);

    const X_SCALAR = 1 / window.devicePixelRatio;
    const Y_SCALAR = 1 / window.devicePixelRatio;

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
        let detailsSidebar = document.getElementById("recipe-details-sidebar");
    
        if (detailsSidebar){
          detailsSidebar.style.transform = "translateX(0)";
          detailsSidebar.style.right = "0";
      
          createRecipeCard(clickedMark, detailsSidebar);
        } else {
          console.error("Could not find detailsSidebar element");
        }
      }
    }
  }

  function handleHover(event: MouseEvent) {
  const rect = canvas.getBoundingClientRect();
  const X_SCALAR = 1 / window.devicePixelRatio;
  const Y_SCALAR = 1 / window.devicePixelRatio;

  const scaleX = (canvas.width / rect.width);
  const scaleY = canvas.height / rect.height;
  
  const canvasX = (((event.clientX - rect.left) * X_SCALAR) - 19.76) * scaleX;
  const canvasY = (((event.clientY - rect.bottom) * Y_SCALAR) + 305) * scaleY;
  
  const transform = d3.zoomTransform(canvas);
  const [dataX, dataY] = transform.invert([canvasX, canvasY]);
  
  let hoveredMark = null;

  let tooltip = document.getElementById("recipe-tooltip");
  
  if (!tooltip) {
    tooltip = document.createElement("div");
    tooltip.id = "recipe-tooltip";
    tooltip.style.position = "absolute";
    tooltip.style.padding = "12px 16px";
    tooltip.style.background = "linear-gradient(145deg, #ffffff, #fff5f5)";
    tooltip.style.boxShadow = "0 4px 15px rgba(220, 38, 38, 0.08), 0 1px 3px rgba(220, 38, 38, 0.12)";
    tooltip.style.borderRadius = "8px";
    tooltip.style.fontSize = "14px";
    tooltip.style.fontFamily = "'Inter', -apple-system, BlinkMacSystemFont, sans-serif";
    tooltip.style.color = "#4b5563";
    tooltip.style.pointerEvents = "none";
    tooltip.style.zIndex = "1000";
    tooltip.style.maxWidth = "250px";
    tooltip.style.transition = "all 0.25s cubic-bezier(0.16, 1, 0.3, 1)";
    tooltip.style.opacity = "0";
    tooltip.style.transform = "translateY(5px) scale(0.98)";
    tooltip.style.border = "1px solid rgba(254, 226, 226, 0.9)";
    document.body.appendChild(tooltip);
  }
  
  if (currentView === "summary") {
    hoveredMark = summaryPositionMap.hitTest([dataX, dataY]);
    
    canvas.style.cursor = hoveredMark ? 'pointer' : 'default';
    
    if (hoveredMark !== hoveredMarkRef) {
      hoveredMarkRef = hoveredMark;
      drawMarks(); 
    }
  } else if (currentView === "food") {
    hoveredMark = foodPositionMap.hitTest([dataX, dataY]);
    
    canvas.style.cursor = hoveredMark ? 'pointer' : 'default';
    
    if (hoveredMark !== hoveredMarkRef) {
      hoveredMarkRef = hoveredMark;
      drawMarks(); 
    }

    if (hoveredMark) {
      const recipeName = hoveredMark.attr('name');
      const cookTime = hoveredMark.attr('time');
      
      tooltip.innerHTML = `
        <div style="margin-bottom: 6px; font-weight: 600; color: #dc2626; letter-spacing: 0.01em;">${recipeName}</div>
        <div style="display: flex; align-items: center; font-size: 13px; color: #6b7280;">
          <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="#ef4444" stroke-width="2" style="margin-right: 6px;">
            <circle cx="12" cy="12" r="10"></circle>
            <polyline points="12 6 12 12 16 14"></polyline>
          </svg>
          <span>${cookTime} mins</span>
        </div>
        <div style="position: absolute; top: -3px; left: 0; right: 0; height: 3px; background: linear-gradient(90deg, #ef4444, #f87171); border-radius: 3px 3px 0 0;"></div>
      `;
      
      tooltip.style.left = `${event.clientX + 12}px`;
      tooltip.style.top = `${event.clientY - 12}px`;
      tooltip.style.opacity = "1";
      tooltip.style.transform = "translateY(0) scale(1)";
      hoveredMark.attr("ingredients").forEach((ing: string) => {
  enteredIngredientsButtons.forEach(b => {
    if (b.textContent?.trim().toLowerCase() === ing.trim().toLowerCase()) {
      b.style.backgroundColor = "red";
    }
  });
});


    } else {
      tooltip.style.opacity = "0";
      tooltip.style.transform = "translateY(5px) scale(0.98)";


      let enteredIngredientsBox = document.getElementById("entered-ingredients-box");
      var children = enteredIngredientsBox?.children;
      if (children) 
      {
        for (var i = 0; i < children.length; i++) {
          let button = children[i] as HTMLButtonElement;
          button.style.backgroundColor = "#4CAF50";
        }
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

  function createRecipeCard(clickedMark: Mark<FoodMarkAttrs>, detailsSidebar: HTMLElement): void {
    const id = Number(clickedMark.id);
    if (!imageCache[id]) {
      loadImageOnDemand(id);
    }
  let clickedMarkDisplayBox = document.getElementById("clicked-mark-box");
  
  if (clickedMarkDisplayBox && detailsSidebar) {
    clickedMarkDisplayBox.innerHTML = "";
    
    if (!document.getElementById('enhanced-recipe-styles')) {
      const styleElement = document.createElement('style');
      styleElement.id = 'enhanced-recipe-styles';
      styleElement.textContent = `
        #clicked-mark-box {
          padding: 0;
          height: 100%;
        }
        
        .recipe-card {
          height: 100%;
          background-color: #fff9f5; 
          overflow: hidden;
          position: relative;
          display: flex;
          flex-direction: column;
          color: #3a3a3a;
        }
        
        .close-button {
          position: absolute;
          top: 15px;
          right: 15px;
          background-color: rgba(255, 255, 255, 0.9);
          border: none;
          width: 40px;
          height: 40px;
          border-radius: 50%;
          display: flex;
          align-items: center;
          justify-content: center;
          font-size: 24px;
          color: #2f4858;
          cursor: pointer;
          z-index: 10;
          box-shadow: 0 3px 8px rgba(0, 0, 0, 0.15);
          transition: all 0.3s ease;
        }
        
        .close-button:hover {
          background-color: #f1f1f1;
          transform: scale(1.1);
          box-shadow: 0 5px 12px rgba(0, 0, 0, 0.2);
        }
        
        .recipe-image-container {
          padding: 20px;
          display: flex;
          justify-content: center;
          background-color: #fff;
        }
        
        .recipe-image {
          max-width: 300px;
          max-height: 200px;
          object-fit: contain;
          border-radius: 8px;
          box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
        }
        
        .recipe-title {
          padding: 22px 20px;
          background: linear-gradient(to right, #ff6b6b, #ff5252);
          color: white;
          position: relative;
          box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
          margin-top: 0;
        }
        
        .recipe-title::before {
          content: '';
          position: absolute;
          top: 0;
          left: 0;
          width: 5px;
          height: 100%;
          background-color: #f9c80e;
        }
        
        .recipe-title h3 {
          margin: 0;
          font-size: 1.5rem;
          font-weight: 600;
          letter-spacing: 0.5px;
          text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.2);
        }
        
        .recipe-content {
          padding: 30px 25px;
          flex: 1;
          overflow-y: auto;
          background-image: 
            radial-gradient(circle at 10% 20%, rgba(253, 239, 132, 0.05) 0%, transparent 20%),
            radial-gradient(circle at 90% 80%, rgba(78, 205, 196, 0.05) 0%, transparent 20%);
        }
        
        .recipe-section {
          margin-bottom: 35px;
          position: relative;
        }
        
        .recipe-section h4 {
          color: #ff6b6b;
          margin-top: 0;
          margin-bottom: 18px;
          font-size: 1.25rem;
          border-bottom: none;
          padding-bottom: 8px;
          display: inline-block;
          position: relative;
        }
        
        .recipe-section h4::after {
          content: '';
          position: absolute;
          bottom: 0;
          left: 0;
          width: 100%;
          height: 3px;
          background: linear-gradient(to right, #ff6b6b, #4ecdc4);
          border-radius: 3px;
        }
        
        .recipe-section p {
          margin: 0;
          line-height: 1.8;
          color: #3a3a3a;
        }
        
        .recipe-steps {
          white-space: pre-wrap;
          padding: 20px;
          background-color: white;
          border-radius: 12px;
          border-left: 4px solid #4ecdc4;
          box-shadow: 0 5px 15px rgba(0, 0, 0, 0.05);
          position: relative;
          z-index: 1;
        }
        
        .recipe-steps::before {
          content: '"';
          position: absolute;
          top: -15px;
          left: 15px;
          font-size: 60px;
          color: rgba(78, 205, 196, 0.1);
          font-family: Georgia, serif;
          z-index: -1;
        }
        
        .ai-loading {
          display: flex;
          flex-direction: column;
          align-items: center;
          padding: 25px 0;
        }
        
        .ai-loading .loader {
          border: 3px solid rgba(255, 107, 107, 0.1);
          border-top: 3px solid #ff6b6b;
          border-right: 3px solid #4ecdc4;
          border-bottom: 3px solid #f9c80e;
          border-radius: 50%;
          width: 35px;
          height: 35px;
          animation: spin-recipe 1.2s cubic-bezier(0.68, -0.55, 0.27, 1.55) infinite;
          margin-bottom: 15px;
        }
        
        .ai-content {
          padding: 22px;
          background-color: white;
          background-image: linear-gradient(135deg, rgba(249, 200, 14, 0.03) 0%, rgba(255, 107, 107, 0.03) 100%);
          border-radius: 12px;
          border-left: 4px solid #f9c80e;
          box-shadow: 0 5px 15px rgba(0, 0, 0, 0.05);
          position: relative;
        }
        
        .ai-content::before {
          content: '✨';
          position: absolute;
          top: -12px;
          right: 20px;
          font-size: 24px;
          color: #f9c80e;
          text-shadow: 0 0 10px rgba(249, 200, 14, 0.3);
        }
        
        .error-message {
          color: #d9534f;
          font-style: italic;
          padding: 10px 15px;
          background-color: rgba(217, 83, 79, 0.05);
          border-radius: 8px;
        }
        
        .toggle-ingredients-btn {
          background: linear-gradient(to right, #4ecdc4, #2cb5e8);
          color: white;
          border: none;
          padding: 10px 16px;
          border-radius: 25px;
          font-weight: 600;
          display: flex;
          align-items: center;
          cursor: pointer;
          box-shadow: 0 4px 8px rgba(78, 205, 196, 0.3);
          transition: all 0.3s ease;
          margin-bottom: 20px;
        }
        
        .toggle-ingredients-btn:hover {
          transform: translateY(-2px);
          box-shadow: 0 6px 12px rgba(78, 205, 196, 0.4);
        }
        
        .toggle-ingredients-btn:active {
          transform: translateY(1px);
        }
        
        .toggle-ingredients-btn .icon {
          margin-right: 8px;
          font-size: 18px;
          transition: transform 0.3s ease;
        }
        
        .toggle-ingredients-btn.active .icon {
          transform: rotate(180deg);
        }
        
        .ingredients-container {
          max-height: 0;
          overflow: hidden;
          transition: max-height 0.5s ease;
          margin-top: 0;
        }
        
        .ingredients-container.active {
          max-height: 500px;
        }
        
        .ingredients-content {
          padding: 20px;
          background-color: white;
          border-radius: 12px;
          border-left: 4px solid #f9c80e;
          box-shadow: 0 5px 15px rgba(0, 0, 0, 0.05);
          position: relative;
          z-index: 1;
          margin-top: 10px;
        }
        
        .ingredients-content ul {
          padding-left: 20px;
          margin: 0;
        }
        
        .ingredients-content li {
          margin-bottom: 8px;
          position: relative;
          padding-left: 5px;
        }
        
        .ingredients-content li::before {
          content: '•';
          color: #ff6b6b;
          font-weight: bold;
          display: inline-block;
          width: 1em;
          margin-left: -1em;
        }
        
        .ingredients-tabs {
          display: flex;
          margin-bottom: 15px;
        }
        
        .ingredients-tab {
          padding: 8px 16px;
          background-color: #f5f5f5;
          border: none;
          border-radius: 20px;
          margin-right: 10px;
          cursor: pointer;
          font-size: 14px;
          transition: all 0.2s ease;
        }
        
        .ingredients-tab.active {
          background-color: #4ecdc4;
          color: white;
          box-shadow: 0 2px 8px rgba(78, 205, 196, 0.3);
        }
        
        @keyframes spin-recipe {
          0% { transform: rotate(0deg); }
          100% { transform: rotate(360deg); }
        }
      `;
      document.head.appendChild(styleElement);
    }

    let recipeCard = document.createElement('div');
    recipeCard.className = 'recipe-card';
    
    let titleDiv = document.createElement('div');
    titleDiv.className = 'recipe-title';
    let title = document.createElement('h3');
    title.textContent = clickedMark.attr("name") as string;
    titleDiv.appendChild(title);
    recipeCard.appendChild(titleDiv);
    
    let imgSrc = (clickedMark.attr('img') as {src: string}).src;
    let imgContainer = document.createElement('div');
    imgContainer.className = 'recipe-image-container';
    let img = document.createElement('img');
    img.src = imgSrc;
    img.className = 'recipe-image';
    img.alt = clickedMark.attr("name") as string || "Recipe image";
    imgContainer.appendChild(img);
    recipeCard.appendChild(imgContainer);
    
    let closeButton = document.createElement('button');
    closeButton.className = 'close-button';
    closeButton.innerHTML = '&times;'; 
    closeButton.onclick = function() {
      detailsSidebar.classList.remove('active');
      detailsSidebar.style.transform = "translateX(100%)";
    };
    recipeCard.appendChild(closeButton);

    let contentContainer = document.createElement('div');
    contentContainer.className = 'recipe-content';
    
    let ingredientsSection = document.createElement('div');
    ingredientsSection.className = 'recipe-section';
    
    let toggleButton = document.createElement('button');
    toggleButton.className = 'toggle-ingredients-btn';
    toggleButton.innerHTML = '<span class="icon">▼</span> Show Ingredients';
    toggleButton.onclick = function() {
      const ingredientsContainer = document.getElementById('ingredients-container');
      if (ingredientsContainer) {
        ingredientsContainer.classList.toggle('active');
        toggleButton.classList.toggle('active');
        
        if (ingredientsContainer.classList.contains('active')) {
          toggleButton.innerHTML = '<span class="icon">▲</span> Hide Ingredients';
        } else {
          toggleButton.innerHTML = '<span class="icon">▼</span> Show Ingredients';
        }
      }
    };
    
    let ingredientsContainer = document.createElement('div');
    ingredientsContainer.className = 'ingredients-container';
    ingredientsContainer.id = 'ingredients-container';
    
    let tabsContainer = document.createElement('div');
    tabsContainer.className = 'ingredients-tabs';
    
    let originalTab = document.createElement('button');
    originalTab.className = 'ingredients-tab active';
    originalTab.textContent = 'Original';
    
    let cleanedTab = document.createElement('button');
    cleanedTab.className = 'ingredients-tab';
    cleanedTab.textContent = 'Cleaned';
    
    let originalIngredientsContent = document.createElement('div');
    originalIngredientsContent.className = 'ingredients-content';
    
    let cleanedIngredientsContent = document.createElement('div');
    cleanedIngredientsContent.className = 'ingredients-content';
    cleanedIngredientsContent.style.display = 'none';
    
    originalTab.onclick = function() {
      originalTab.classList.add('active');
      cleanedTab.classList.remove('active');
      originalIngredientsContent.style.display = 'block';
      cleanedIngredientsContent.style.display = 'none';
    };
    
    cleanedTab.onclick = function() {
      cleanedTab.classList.add('active');
      originalTab.classList.remove('active');
      cleanedIngredientsContent.style.display = 'block';
      originalIngredientsContent.style.display = 'none';
    };
    
    tabsContainer.appendChild(originalTab);
    tabsContainer.appendChild(cleanedTab);
    
    const id = clickedMark.id;
    
    if (typeof dataCSV !== 'undefined' && id) {
      const recipeIndex = Number(id);
      
      if (dataCSV[recipeIndex]?.Ingredients) {
        const ingredientsList = document.createElement('ul');
        const ingredients = dataCSV[recipeIndex].Ingredients.split(',');
        
        ingredients.forEach(ingredient => {
          if (ingredient.trim()) {
            const li = document.createElement('li');
            li.textContent = ingredient.trim().replaceAll("'", '').replaceAll("[", '').replaceAll("]", '');;
            ingredientsList.appendChild(li);
          }
        });
        
        originalIngredientsContent.appendChild(ingredientsList);
      } else {
        originalIngredientsContent.textContent = 'No ingredients available';
      }
      
      if (dataCSV[recipeIndex]?.Cleaned_Ingredients) {
        const cleanedList = document.createElement('ul');
        const cleanedIngredients = dataCSV[recipeIndex].Cleaned_Ingredients.split(',');
        
        cleanedIngredients.forEach(ingredient => {
          if (ingredient.trim()) {
            const li = document.createElement('li');
            li.textContent = ingredient.trim().replaceAll("'", '').replaceAll("[", '').replaceAll("]", '');
            cleanedList.appendChild(li);
          }
        });
        
        cleanedIngredientsContent.appendChild(cleanedList);
      } else {
        cleanedIngredientsContent.textContent = 'No cleaned ingredients available';
      }
    } else {
      originalIngredientsContent.textContent = 'Recipe data not available';
      cleanedIngredientsContent.textContent = 'Recipe data not available';
    }
    
    ingredientsContainer.appendChild(tabsContainer);
    ingredientsContainer.appendChild(originalIngredientsContent);
    ingredientsContainer.appendChild(cleanedIngredientsContent);
    
    ingredientsSection.appendChild(toggleButton);
    ingredientsSection.appendChild(ingredientsContainer);
    contentContainer.appendChild(ingredientsSection);
    
    let instructionsSection = document.createElement('div');
    instructionsSection.className = 'recipe-section';
    let instructionsTitle = document.createElement('h4');
    instructionsTitle.textContent = 'Instructions';
    let instructionsContent = document.createElement('div');
    instructionsContent.className = 'recipe-steps';
    instructionsContent.textContent = clickedMark.attr('instructions') as string;
    
    instructionsSection.appendChild(instructionsTitle);
    instructionsSection.appendChild(instructionsContent);
    contentContainer.appendChild(instructionsSection);
    
    let aiSection = document.createElement('div');
    aiSection.className = 'recipe-section';
    let aiTitle = document.createElement('h4');
    aiTitle.textContent = 'AI Overview';
    
    let aiLoadingContainer = document.createElement('div');
    aiLoadingContainer.className = 'ai-loading';
    
    let loadingIndicator = document.createElement('div');
    loadingIndicator.className = 'loader';
    aiLoadingContainer.appendChild(loadingIndicator);
    
    let loadingText = document.createElement('p');
    loadingText.textContent = 'Generating AI overview...';
    aiLoadingContainer.appendChild(loadingText);
    
    aiSection.appendChild(aiTitle);
    aiSection.appendChild(aiLoadingContainer);
    
    let aiContent = document.createElement('div');
    aiContent.id = 'ai-summary-content';
    aiContent.className = 'ai-content';
    aiSection.appendChild(aiContent);
    aiContent.style.display = 'none'; 
    
    contentContainer.appendChild(aiSection);
    recipeCard.appendChild(contentContainer);
    
    clickedMarkDisplayBox.appendChild(recipeCard);
    
    detailsSidebar.classList.add('active');
    
    AISummary(clickedMark).then(summary => {
      aiLoadingContainer.style.display = 'none';
      aiContent.style.display = 'block';
      
      let aiSummaryText = document.createElement('p');
      aiSummaryText.textContent = summary;
      aiContent.appendChild(aiSummaryText);
    }).catch(error => {
      aiLoadingContainer.style.display = 'none';
      aiContent.style.display = 'block';
      
      let errorMessage = document.createElement('p');
      errorMessage.textContent = 'Could not generate AI overview. Please try again.';
      errorMessage.className = 'error-message';
      aiContent.appendChild(errorMessage);
    });
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
    {#key imagesLoaded && currentView}
    <div hidden={imagesLoaded || currentView === "frontPage"}>
      <div class="loading-screen">
        <div class="loader"></div>
        <p>Loading your culinary adventure...</p>
      </div>
    </div>
    {/key}
    <svg id="axes" style="position: absolute; left: 20%; width: 70%;" class:hidden={currentView === "frontPage"}></svg>
    <canvas bind:this={canvas} style="position:absolute; left: 25%;" class:hidden={currentView === "frontPage"}></canvas>
  </div>

  {#if currentView === "food" || currentView === "summary"}
    <div id="recipe-details-sidebar" class="details-sidebar">
      <div id="clicked-mark-box"></div>
    </div>

    <div id="ingredient-bar">
      <h3>Filter by Ingredients</h3>
      <input id="search-bar" type="text" placeholder="Search ingredients...">
      <div id="entered-ingredients-box"></div>
    </div>
  {/if}

  {#if currentView === "frontPage"}
    <div class="front-page">
      <div class="front-page-content">
        <div class="logo-container">
          <span class="logo-icon">🍽️</span>
          <h1>FoodTable</h1>
        </div>
        <h2>Discover recipes based on ingredients you have</h2>
        
        <div id="ingredient-bar-front">
          <input id="search-bar-front" type="text" placeholder="Enter an ingredient...">
          <div id="entered-ingredients-box-front"></div>
        </div>
        
        <div class="filter-options">
          <select id="dropdown">
            <option value="onlyComplete">Only show recipes with all entered ingredients</option>
            <option value="anyIngredient">Show recipes with any entered ingredient</option>
            <option value="allIngredients">Only show recipes with ALL entered ingredients</option>
          </select>
        </div>
        
        <button class="primary-button" on:click={triggerFoodView}>
          <span>Explore Recipes</span>
          <span class="button-icon">→</span>
        </button>
      </div>
    </div>
  {/if}
</main>

<style>
  :root {
    --primary-color: #ff6b6b;
    --primary-dark: #ff5252;
    --secondary-color: #4ecdc4;
    --secondary-dark: #3ab7ae;
    --accent-color: #f9c80e;
    --dark-color: #2f4858;
    --light-color: #f7f7f7;
    --box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
    --card-radius: 12px;
    --transition: all 0.3s ease;
  }

  * {
    box-sizing: border-box;
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  }

  body {
    margin: 0;
    padding: 0;
    background-color: var(--light-color);
    color: var(--dark-color);
  }

  main {
    max-width: 100%;
    margin: 0;
    padding: 0;
    height: 100vh;
    position: relative;
    overflow: hidden;
  }
  
  canvas {
    z-index: 10;
    image-rendering: optimizeSpeed;
  }
  
  .loading-screen {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    background-color: white;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    padding: 2rem;
    border-radius: var(--card-radius);
    box-shadow: var(--box-shadow);
    z-index: 100;
    width: 300px;
  }

  .loader {
    border: 5px solid #f3f3f3;
    border-top: 5px solid var(--primary-color);
    border-radius: 50%;
    width: 50px;
    height: 50px;
    animation: spin 2s linear infinite;
    margin-bottom: 1rem;
  }

  @keyframes spin {
    0% { transform: rotate(0deg); }
    100% { transform: rotate(360deg); }
  }

  .hidden {
    display: none !important;
  }

  .front-page {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    display: flex;
    align-items: center;
    justify-content: center;
    background: linear-gradient(135deg, #fff6e5 0%, #ffedf2 100%);
    z-index: 11;
  }

  .front-page-content {
    max-width: 600px;
    padding: 2.5rem;
    text-align: center;
    background-color: white;
    border-radius: var(--card-radius);
    box-shadow: 0 15px 30px rgba(0, 0, 0, 0.1);
  }

  .logo-container {
    display: flex;
    align-items: center;
    justify-content: center;
    margin-bottom: 1rem;
  }

  .logo-icon {
    font-size: 2.5rem;
    margin-right: 0.5rem;
  }

  .front-page h1 {
    color: var(--primary-color);
    font-size: 3rem;
    margin: 0;
  }

  .front-page h2 {
    color: var(--dark-color);
    font-size: 1.2rem;
    font-weight: normal;
    margin-bottom: 2rem;
  }

  .primary-button {
    background-color: var(--primary-color);
    color: white;
    border: none;
    padding: 14px 28px;
    font-size: 1rem;
    border-radius: 30px;
    cursor: pointer;
    transition: var(--transition);
    margin-top: 2rem;
    font-weight: bold;
    letter-spacing: 0.5px;
    box-shadow: 0 4px 12px rgba(255, 107, 107, 0.3);
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 8px;
  }

  .button-icon {
    font-size: 1.2rem;
    transition: transform 0.3s ease;
  }

  .primary-button:hover {
    background-color: var(--primary-dark);
    transform: translateY(-2px);
    box-shadow: 0 6px 16px rgba(255, 107, 107, 0.4);
  }

  .primary-button:hover .button-icon {
    transform: translateX(4px);
  }

  #ingredient-bar, #ingredient-bar-front {
    position: relative;
    width: 100%;
    margin-bottom: 20px;
  }

  #ingredient-bar {
    position: absolute;
    top: 5%;
    right: 2%;
    width: 250px;
    background-color: white;
    padding: 20px;
    border-radius: var(--card-radius);
    box-shadow: var(--box-shadow);
    z-index: 20;
  }

  #ingredient-bar h3 {
    margin-top: 0;
    color: var(--primary-color);
    font-size: 1.1rem;
    margin-bottom: 15px;
    text-align: center;
  }

  #search-bar, #search-bar-front {
    width: 100%;
    padding: 12px 15px;
    border: 2px solid #e0e0e0;
    border-radius: 30px;
    font-size: 1rem;
    transition: var(--transition);
    margin-bottom: 10px;
  }

  #search-bar:focus, #search-bar-front:focus {
    outline: none;
    border-color: var(--secondary-color);
    box-shadow: 0 0 0 3px rgba(78, 205, 196, 0.2);
  }

  #entered-ingredients-box, #entered-ingredients-box-front {
    display: flex;
    flex-wrap: wrap;
    gap: 8px;
    margin-top: 10px;
    min-height: 30px;
  }

  .result-added-button {
    display: inline-flex;
    align-items: center;
    background-color: var(--secondary-color);
    color: white;
    padding: 8px 12px;
    margin: 2px;
    border: none;
    border-radius: 20px;
    cursor: pointer;
    font-size: 0.9rem;
    transition: var(--transition);
  }

  .result-added-button:hover {
    background-color: var(--secondary-dark);
    transform: translateY(-1px);
  }

  [name="ingredient-result-button"] {
    background-color: white;
    border: 1px solid #e0e0e0;
    padding: 10px 15px;
    margin: 5px 0;
    width: 100%;
    text-align: left;
    cursor: pointer;
    border-radius: 8px;
    transition: var(--transition);
  }

  [name="ingredient-result-button"]:hover {
    background-color: #f9f9f9;
    border-color: #ccc;
    transform: translateY(-1px);
  }

  .filter-options {
    margin: 1.5rem 0;
  }

  select {
    width: 100%;
    padding: 12px;
    border: 2px solid #e0e0e0;
    border-radius: 8px;
    background-color: white;
    font-size: 0.9rem;
    color: var(--dark-color);
    cursor: pointer;
    appearance: none;
    background-image: url("data:image/svg+xml;charset=US-ASCII,%3Csvg xmlns='http://www.w3.org/2000/svg' width='14' height='14' viewBox='0 0 24 24' fill='none' stroke='%232f4858' stroke-width='2' stroke-linecap='round' stroke-linejoin='round'%3E%3Cpolyline points='6 9 12 15 18 9'%3E%3C/polyline%3E%3C/svg%3E");
    background-repeat: no-repeat;
    background-position: right 15px center;
  }

  select:focus {
    outline: none;
    border-color: var(--secondary-color);
    box-shadow: 0 0 0 3px rgba(78, 205, 196, 0.2);
  }

  .details-sidebar {
  position: fixed;
  top: 0;
  right: 0;
  width: 400px;
  height: 100vh;
  background-color: white;
  box-shadow: -5px 0 15px rgba(0, 0, 0, 0.1);
  z-index: 50;
  overflow-y: auto;
  transform: translateX(100%);
  transition: transform 0.3s ease-in-out;
}

.details-sidebar.active {
  transform: translateX(0) !important;
}

  #clicked-mark-box {
  padding: 0;
  height: 100%;
}

.recipe-card {
  height: 100%;
  background-color: #fff9f5;
  overflow: hidden;
  position: relative;
  display: flex;
  flex-direction: column;
  color: #3a3a3a;
}

.close-button {
  position: absolute;
  top: 15px;
  right: 15px;
  background-color: rgba(255, 255, 255, 0.9);
  border: none;
  width: 40px;
  height: 40px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 24px;
  color: var(--dark-color);
  cursor: pointer;
  z-index: 10;
  box-shadow: 0 3px 8px rgba(0, 0, 0, 0.15);
  transition: var(--transition);
}

.close-button:hover {
  background-color: #f1f1f1;
  transform: scale(1.1);
  box-shadow: 0 5px 12px rgba(0, 0, 0, 0.2);
}

.recipe-header {
  position: relative;
  height: 280px;
  overflow: hidden;
}

.recipe-image {
  width: 100%;
  height: 100%;
  object-fit: cover;
  transition: transform 0.5s ease;
  filter: brightness(0.95);
}

.recipe-header:hover .recipe-image {
  transform: scale(1.05);
  filter: brightness(1.05);
}

.recipe-header::after {
  content: '';
  position: absolute;
  bottom: 0;
  left: 0;
  width: 100%;
  height: 80px;
  background: linear-gradient(to top, rgba(0, 0, 0, 0.5), transparent);
  z-index: 2;
}

.recipe-title {
  padding: 22px 20px;
  background: linear-gradient(to right, var(--primary-color), var(--primary-dark));
  color: white;
  position: relative;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
}

.recipe-title::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  width: 5px;
  height: 100%;
  background-color: var(--accent-color);
}

.recipe-title h3 {
  margin: 0;
  font-size: 1.5rem;
  font-weight: 600;
  letter-spacing: 0.5px;
  text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.2);
}

.recipe-content {
  padding: 30px 25px;
  flex: 1;
  overflow-y: auto;
  background-image: 
    radial-gradient(circle at 10% 20%, rgba(253, 239, 132, 0.05) 0%, transparent 20%),
    radial-gradient(circle at 90% 80%, rgba(78, 205, 196, 0.05) 0%, transparent 20%);
}

.recipe-section {
  margin-bottom: 35px;
  position: relative;
}

.recipe-section h4 {
  color: var(--primary-color);
  margin-top: 0;
  margin-bottom: 18px;
  font-size: 1.25rem;
  border-bottom: none;
  padding-bottom: 8px;
  display: inline-block;
  position: relative;
}

.recipe-section h4::after {
  content: '';
  position: absolute;
  bottom: 0;
  left: 0;
  width: 100%;
  height: 3px;
  background: linear-gradient(to right, var(--primary-color), var(--secondary-color));
  border-radius: 3px;
}

.recipe-section p {
  margin: 0;
  line-height: 1.8;
  color: #3a3a3a;
}

.recipe-steps {
  white-space: pre-wrap;
  padding: 20px;
  background-color: white;
  border-radius: 12px;
  border-left: 4px solid var(--secondary-color);
  box-shadow: 0 5px 15px rgba(0, 0, 0, 0.05);
  position: relative;
  z-index: 1;
}

.recipe-steps::before {
  content: '"';
  position: absolute;
  top: -15px;
  left: 15px;
  font-size: 60px;
  color: rgba(78, 205, 196, 0.1);
  font-family: Georgia, serif;
  z-index: -1;
}

.ai-loading {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 25px 0;
}

.ai-loading .loader {
  border: 3px solid rgba(255, 107, 107, 0.1);
  border-top: 3px solid var(--primary-color);
  border-right: 3px solid var(--secondary-color);
  border-bottom: 3px solid var(--accent-color);
  border-radius: 50%;
  width: 35px;
  height: 35px;
  animation: spin 1.2s cubic-bezier(0.68, -0.55, 0.27, 1.55) infinite;
  margin-bottom: 15px;
}

.ai-content {
  padding: 22px;
  background-color: white;
  background-image: linear-gradient(135deg, rgba(249, 200, 14, 0.03) 0%, rgba(255, 107, 107, 0.03) 100%);
  border-radius: 12px;
  border-left: 4px solid var(--accent-color);
  box-shadow: 0 5px 15px rgba(0, 0, 0, 0.05);
  position: relative;
}

.ai-content::before {
  content: '✨';
  position: absolute;
  top: -12px;
  right: 20px;
  font-size: 24px;
  color: var(--accent-color);
  text-shadow: 0 0 10px rgba(249, 200, 14, 0.3);
}

.error-message {
  color: #d9534f;
  font-style: italic;
  padding: 10px 15px;
  background-color: rgba(217, 83, 79, 0.05);
  border-radius: 8px;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

@media (max-width: 768px) {
  .recipe-header {
    height: 200px;
  }
  
  .recipe-title h3 {
    font-size: 1.3rem;
  }
  
  .recipe-content {
    padding: 20px;
  }
}

.recipe-section {
  margin-bottom: 30px;
  position: relative;
}

.recipe-section h4 {
  color: var(--primary-color);
  margin-top: 0;
  margin-bottom: 15px;
  font-size: 1.2rem;
  border-bottom: 2px solid var(--primary-color);
  padding-bottom: 8px;
  display: inline-block;
}

.recipe-section p {
  margin: 0;
  line-height: 1.7;
  color: #444;
}

.recipe-steps {
  white-space: pre-wrap;
  padding: 15px;
  background-color: #f9f9f9;
  border-radius: 8px;
  border-left: 3px solid var(--secondary-color);
}

.ai-loading {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 20px 0;
}

.ai-loading .loader {
  border: 3px solid #f3f3f3;
  border-top: 3px solid var(--primary-color);
  border-radius: 50%;
  width: 30px;
  height: 30px;
  animation: spin 1.5s linear infinite;
  margin-bottom: 10px;
}

.ai-content {
  padding: 15px;
  background-color: #f5f8ff;
  border-radius: 8px;
  border-left: 3px solid var(--accent-color);
}

.error-message {
  color: #d9534f;
  font-style: italic;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

@media (max-width: 768px) {
  .recipe-header {
    height: 180px;
  }
  
  .recipe-title h3 {
    font-size: 1.3rem;
  }
  
  .recipe-content {
    padding: 15px;
  }
}

    .front-page h1 {
      font-size: 2.2rem;
    }

    .front-page h2 {
      font-size: 1rem;
    }

    #ingredient-bar {
      width: 200px;
      right: 1%;
    }

    .details-sidebar {
      width: 100%;
    }

    .recipe-image {
      height: 180px;
    }
  
</style>