# Input from a plot

Goal: a dashboard that has "bidirectional" click interaction, meaning that drop-down menus caninfluence plots and viceversa.

The page https://observablehq.com/framework/reactivity#inputs gives valuable insights. The Observable Plot library is not enough, as pointed out here https://talk.observablehq.com/t/observable-plot-click-events/8633/5, so the lower-level library D3 is needed.

As an example, suppose that we want to display data divided in categories.

## Input objects

Observable built-in `Input` objects are great for simple interaction, i.e. the one that uses them just as inputs; but what if we want to be able to change the value both from an input object and from somewhere else (in our case, a plot)?

### the Mutable

Everything revolves around the `Mutable` concept: an object that is watched at by other components that depend on it (automagically! Thanks to Observable framework). You can define one as follows:

```js echo
const category = Mutable("first");
const setCategory = (x) => (category.value = x);
```

This code creates an object that stores the value of the category, assigning "first" as the default, and defines a setter function that is able to change this value (I noticed that it's important to define it in the same code block/cell).

This setter function is what we will pass as a callback to the `onClick` event in the plot.

### the input

Observable provides handy input objects. Let's pick one which is less obvious than the button in the documentation, a drop-down selection:

```js echo
const categoryInput = Inputs.select(["first", "second", "third"], {
  label: "Category",
});
```

```js echo
const categoryInputValue = view(categoryInput)
```

Here `view` both makes the input visible and usable, and allow assigning the value to be used later.
We also need a setter, which this time should be in _another_ cell; note that we are setting the value of the input object, while the value we assigned is, well, just a view.

```js echo
const setCategoryInput = (x) => (categoryInput.value = x);
```

### connect them

This is simple: in two separate cells, just call the setter functions with the value of the other component.

```js echo
setCategory(categoryInputValue);
```


and

```js echo
setCategoryInput(category);
```

Now we get the following:
>`category`: ${category}

>`categoryInputValue`: ${categoryInputValue}


Let's check that this works: if you select from the drop-down, you can see the values changing; if instead you change the value setting the mutable, say with a button, the drop-down gets updated ✅

```js echo
view(
  Inputs.button([
    ["first", () => setCategory("first")],
    ["second", () => setCategory("second")],
  ])
);
```


👀 Note that after pushing the button the value of `categoryInput` changes, but `categoryInputValue` _doesn't_! I'm not completely sure if this can be fixed, but it's not a big deal since the latter is only used in the `setCategory` function, which we need to call only when selecting from the drop-down... but pay attention, don't use it elsewhere!


## Plot

Let's create some sample data:

```js echo
const data = [];
const categories = ["first", "second", "third"];

for (let i = 0; i < 20; i++) {
  const x = Math.random();
  const y = Math.random();
  const category = categories[Math.floor(Math.random() * categories.length)];
  data.push({ x, y, category });
}

display(data);
```

Now let's plot it; in order to keep the code clean, we want to create a function, which could be put into another file.
Apart of the usual scatter plot code, these are the important bits:
- in order to **pass information from the page to the plot**, the `category` is passed to the plotting function; remember that it's a `Mutable`, so its updates will trigger changes in the plot;
- in order to **pass information from the plot to the page**, an event handler is added for "click" events, which calls the setter function.

```js echo
function plotData(widthContainer, data, setterFunction, chosenCategory) {
  // setup the plot
  const margin = { top: 10, right: 0, bottom: 10, left: 20 };
  const width = widthContainer //- margin.left - margin.right;
  const height = 400// - margin.top - margin.bottom;
  const color = d3.scaleOrdinal(d3.schemeObservable10);

  const svg = d3
    .create("svg")
    .attr("width", width + margin.left + margin.right)
    .attr("height", height + margin.top + margin.bottom)

  const xScale = d3.scaleLinear().domain([0, 1]).range([0, width - margin.left - margin.right]);
  const yScale = d3.scaleLinear().domain([0, 1]).range([height- margin.top - margin.bottom, 0]);

  const xAxis = d3.axisBottom(xScale);
  const yAxis = d3.axisLeft(yScale);

  svg.append("g").attr("transform", `translate(30,${height-10})`).call(xAxis);
  svg.append("g").attr("transform", `translate(30,10)`).call(yAxis);


  svg
    .selectAll("circle")
    .data(data)
    .enter()
    .append("circle")
    .attr("cx", (d) => xScale(d.x))
    .attr("cy", (d) => yScale(d.y))
    .attr("r", 5)
    .attr("stroke", (d) => color(d.category))
    .attr("fill", (d) => (d.category == chosenCategory ? color(d.category) : undefined))
    .on("click", (event, i) => {
      setterFunction(i.category);
    });

  const categories = ["first", "second", "third"];
  const legend = svg.append("g").attr("class", "legend").attr('transform', 'translate(50,10)');

  legend
    .selectAll("rect")
    .data(categories)
    .enter()
    .append("rect")
    .attr("y", (d, i) => i * 20)
    .attr("width", 10)
    .attr("height", 10)
    .attr("fill", (d) => color(d));

  legend
    .selectAll("text")
    .data(categories)
    .enter()
    .append("text")
    .attr("x", 15)
    .attr("y", (d, i) => i * 20 + 9)
    .text((d) => d)
    .attr("fill", "var(--theme-foreground)")
    .style("font-size", "12px")
    .attr("alignment-baseline", "middle");
  return svg.node();
}
```

```html echo
<div class="card">${resize((width)=>plotData(width, data, setCategory, category))}</div>
```


You can now see that clicking on a circle in the plot sets the value of the drop-down object, and vice-versa.
