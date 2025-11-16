---
theme: dashboard
title: Visualization 5 - Choropleth Map
toc: false
---

# Visualization 5 - Choropleth Map

```js
import {Legend} from "https://api.observablehq.com/@d3/color-legend.js?v=4"
import {us} from "https://api.observablehq.com/@d3/us-state-choropleth/2.js?v=4"
```

```js

```


```js
const rawPopulationData = FileAttachment("data/population.csv").csv()
```

```js
const populationData = d3.group(rawPopulationData, d => d.STNAME, d => d.CTYNAME)
```

```js
const processedData = FileAttachment("data/processed_hate_crimes.csv").csv()
```

```js
const processedDataForMap = processedData.filter(d => parseInt(d.data_year) >= 2000 && !["Federal", "Guam"].includes(d.state_name))
```

```js
const byStatePerCapita = d3.rollup(
  processedDataForMap,
  v => d3.sum(v, d => d.count/(populationData.get(d.state_name).get(d.state_name)[0][parseInt(d.data_year)])), 
  d => d.general_category, 
  d => d.data_year, 
  d => d.state_name)
```

```js
const general_categories = Array.from(new Set(processedDataForMap.map(d => d.general_category)))
```

```js
const categoryDropdown = html`<div id="categorySelector">${Inputs.select(
    general_categories,
    { label: "Bias Category:" }
  )}</div>`
```

<div>${categoryDropdown}</div>

```js
const selectedCategory = general_categories[categoryDropdown.querySelector("select").value]
```

<!-- <div>${selectedCategory}</div> -->

```js
const byState = d3.rollup(processedDataForMap, v => d3.sum(v, d => d.count), d => d.general_category, d => d.data_year, d => d.state_name)
```

```js
const keys = [...byState.get(selectedCategory).keys()]
```

```js
const firstYear = parseInt(keys[0])
```

```js
const finalYear = parseInt(keys[keys.length-1])
```

```js
const dataYearRange = [firstYear, finalYear]
```

```js
function mapInputValue(input, from, to) {
  return Object.defineProperty(htl.html`<div>${input}`, 'value', {
    get: () => from(input.value),
    set: value => input.value = to(value)
  });
}
```

```js
function cssLength(v) { v == null ? null : typeof v === 'number' ? `${v}px` : `${v}` }
```

```js
const theme_Flat = `
/* Options */
:scope {
  color: #3b99fc;
  width: 240px;
}

:scope {
  position: relative;
  display: inline-block;
  --thumb-size: 15px;
  --thumb-radius: calc(var(--thumb-size) / 2);
  padding: var(--thumb-radius) 0;
  margin: 2px;
  vertical-align: middle;
}
:scope .range-track {
  box-sizing: border-box;
  position: relative;
  height: 7px;
  background-color: hsl(0, 0%, 80%);
  overflow: visible;
  border-radius: 4px;
  padding: 0 var(--thumb-radius);
}
:scope .range-track-zone {
  box-sizing: border-box;
  position: relative;
}
:scope .range-select {
  box-sizing: border-box;
  position: relative;
  left: var(--range-min);
  width: calc(var(--range-max) - var(--range-min));
  cursor: ew-resize;
  background: currentColor;
  height: 7px;
  border: inherit;
}
/* Expands the hotspot area. */
:scope .range-select:before {
  content: "";
  position: absolute;
  width: 100%;
  height: var(--thumb-size);
  left: 0;
  top: calc(2px - var(--thumb-radius));
}
:scope .range-select:focus,
:scope .thumb:focus {
  outline: none;
}
:scope .thumb {
  box-sizing: border-box;
  position: absolute;
  width: var(--thumb-size);
  height: var(--thumb-size);

  background: #fcfcfc;
  top: -4px;
  border-radius: 100%;
  border: 1px solid hsl(0,0%,55%);
  cursor: default;
  margin: 0;
}
:scope .thumb:active {
  box-shadow: inset 0 var(--thumb-size) #0002;
}
:scope .thumb-min {
  left: calc(-1px - var(--thumb-radius));
}
:scope .thumb-max {
  right: calc(-1px - var(--thumb-radius));
}
`
```

```js
function randomScope(prefix = 'scope-') {
  return prefix + (performance.now() + Math.random()).toString(32).replace('.', '-');
}
```

```js
function rangeInput(options = {}) {
  const {
    min = 0,
    max = 100,
    step = 'any',
    value: defaultValue = [min, max],
    color,
    width,
    theme = theme_Flat,
  } = options;
  
  const controls = {};
  const scope = randomScope();
  const clamp = (a, b, v) => v < a ? a : v > b ? b : v;

  // Will be used to sanitize values while avoiding floating point issues.
  const input = html`<input type=range ${{min, max, step}}>`;
  
  const dom = html`<div class=${`${scope} range-slider`} style=${{
    color,
    width: cssLength(width),
  }}>
  ${controls.track = html`<div class="range-track">
    ${controls.zone = html`<div class="range-track-zone">
      ${controls.range = html`<div class="range-select" tabindex=0>
        ${controls.min = html`<div class="thumb thumb-min" tabindex=0>`}
        ${controls.max = html`<div class="thumb thumb-max" tabindex=0>`}
      `}
    `}
  `}
  ${html`<style>${theme.replace(/:scope\b/g, '.'+scope)}`}
</div>`;

  let value = [], changed = false;
  Object.defineProperty(dom, 'value', {
    get: () => [...value],
    set: ([a, b]) => {
      value = sanitize(a, b);
      updateRange();
    },
  });

  const sanitize = (a, b) => {
    a = isNaN(a) ? min : ((input.value = a), input.valueAsNumber);
    b = isNaN(b) ? max : ((input.value = b), input.valueAsNumber);
    return [Math.min(a, b), Math.max(a, b)];
  }
  
  const updateRange = () => {
    const ratio = v => (v - min) / (max - min);
    dom.style.setProperty('--range-min', `${ratio(value[0]) * 100}%`);
    dom.style.setProperty('--range-max', `${ratio(value[1]) * 100}%`);
  };

  const dispatch = name => {
    dom.dispatchEvent(new Event(name, {bubbles: true}));
  };
  const setValue = (vmin, vmax) => {
    const [pmin, pmax] = value;
    value = sanitize(vmin, vmax);
    updateRange();
    // Only dispatch if values have changed.
    if(pmin === value[0] && pmax === value[1]) return;
    dispatch('input');
    changed = true;
  };
  
  setValue(...defaultValue);
  
  // Mousemove handlers.
  const handlers = new Map([
    [controls.min, (dt, ov) => {
      const v = clamp(min, ov[1], ov[0] + dt * (max - min));
      setValue(v, ov[1]);
    }],
    [controls.max, (dt, ov) => {
      const v = clamp(ov[0], max, ov[1] + dt * (max - min));
      setValue(ov[0], v);
    }],
    [controls.range, (dt, ov) => {
      const d = ov[1] - ov[0];
      const v = clamp(min, max - d, ov[0] + dt * (max - min));
      setValue(v, v + d);
    }],
  ]);
  
  // Returns client offset object.
  const pointer = e => e.touches ? e.touches[0] : e;
  // Note: Chrome defaults "passive" for touch events to true.
  const on  = (e, fn) => e.split(' ').map(e => document.addEventListener(e, fn, {passive: false}));
  const off = (e, fn) => e.split(' ').map(e => document.removeEventListener(e, fn, {passive: false}));
  
  let initialX, initialV, target, dragging = false;
  function handleDrag(e) {
    // Gracefully handle exit and reentry of the viewport.
    if(!e.buttons && !e.touches) {
      handleDragStop();
      return;
    }
    dragging = true;
    const w = controls.zone.getBoundingClientRect().width;
    e.preventDefault();
    handlers.get(target)((pointer(e).clientX - initialX) / w, initialV);
  }
  
  
  function handleDragStop(e) {
    off('mousemove touchmove', handleDrag);
    off('mouseup touchend', handleDragStop);
    if(changed) dispatch('change');
  }
  
  invalidation.then(handleDragStop);
  
  dom.ontouchstart = dom.onmousedown = e => {
    dragging = false;
    changed = false;
    if(!handlers.has(e.target)) return;
    on('mousemove touchmove', handleDrag);
    on('mouseup touchend', handleDragStop);
    e.preventDefault();
    e.stopPropagation();
    
    target = e.target;
    initialX = pointer(e).clientX;
    initialV = value.slice();
  };
  
  controls.track.onclick = e => {
    if(dragging) return;
    changed = false;
    const r = controls.zone.getBoundingClientRect();
    const t = clamp(0, 1, (pointer(e).clientX - r.left) / r.width);
    const v = min + t * (max - min);
    const [vmin, vmax] = value, d = vmax - vmin;
    if(v < vmin) setValue(v, v + d);
    else if(v > vmax) setValue(v - d, v);
    if(changed) dispatch('change');
  };
  
  return dom;
}
```

```js
function interval(range = [], options = {}) {
  const [min = 0, max = 1] = range;
  const {
    step = .001,
    label = null,
    value = [min, max],
    format = ([start, end]) => `${start} … ${end}`,
    color,
    width = 360,
    theme,
    __ns__ = randomScope(),
  } = options;

  const css = `
#${__ns__} {
  font: 13px/1.2 var(--sans-serif);
  display: flex;
  align-items: baseline;
  flex-wrap: wrap;
  max-width: 100%;
  width: auto;
}
@media only screen and (min-width: 30em) {
  #${__ns__} {
    flex-wrap: nowrap;
    width: ${cssLength(width)};
  }
}
#${__ns__} .label {
  width: 120px;
  padding: 5px 0 4px 0;
  margin-right: 6.5px;
  flex-shrink: 0;
}
#${__ns__} .form {
  display: flex;
  width: 100%;
}
#${__ns__} .range {
  flex-shrink: 1;
  width: 100%;
}
#${__ns__} .range-slider {
  width: 100%;
}
  `;
  
  const $range = rangeInput({min, max, value: [value[0], value[1]], step, color, width: "100%", theme});
  const $output = html`<output>`;
  const $view = html`<div id=${__ns__}>
${label == null ? '' : html`<div class="label">${label}`}
<div class=form>
  <div class=range>
    ${$range}<div class=range-output>${$output}</div>
  </div>
</div>
${html`<style>${css}`}
  `;

  const update = () => {
    const content = format([$range.value[0], $range.value[1]]);
    if(typeof content === 'string') $output.value = content;
    else {
      while($output.lastChild) $output.lastChild.remove();
      $output.appendChild(content);
    }
  };
  $range.oninput = update;
  update();
  
  return Object.defineProperty($view, 'value', {
    get: () => $range.value,
    set: ([a, b]) => {
      $range.value = [a, b];
      update();
    },
  });
}
```

```js
function offsetInterval(values, {
  value = [values[0], values[values.length - 1]],
  formatValue = v => v,
  format = ([a, b]) => `${formatValue(a)} … ${formatValue(b)}`,
  ...options
} = {}) {
  if(new Set(values).size < values.length) throw Error("All values have to be unique.");
  const valueof = i => values[i];
  const indexof = v => values.indexOf(v);
  const _format = i => formatValue(valueof(i));
  const input = interval([0, values.length - 1], {
    ...options,
    step: 1,
    value: value.map(indexof),
    format: v => format(v.map(valueof)),
  });
  return mapInputValue(input, indices => indices.map(valueof), dates => dates.map(indexof));
}
```

```js
function timeRange(start, end, options = {}) {
  let {
    interval = "day",
    step = 1,
    format = null,
    label = null,
    value = null
  } = options;
  let ts;
  switch (interval.toLowerCase()) {
    case "millisecond":
      ts = d3.utcMilliseconds(start, end, step);
      format = format || "%M:%S.%L";
      break;
    case "second":
      ts = d3.utcSeconds(start, end, step);
      format = format || "%H:%M:%S";
      break;
    case "minute":
      ts = d3.utcMinutes(start, end, step);
      format = format || "%H:%M";
      break;
    case "hour":
      ts = d3.utcHours(start, end, step);
      format = format || "%H:%M";
      break;
    case "day":
      ts = d3.utcDays(start, end, step);
      format = format || "%Y-%m-%d";
      break;
    case "week":
      ts = d3.utcWeeks(start, end, step);
      format = format || "%Y-%m-%d";
      break;
    case "month":
      ts = d3.utcMonths(start, end, step);
      format = format || "%b %Y";
      break;
    case "year":
      ts = d3.utcYears(start, end, step);
      format = format || "%Y";
      break;
    default:
      throw Error(`Unknown time interval "${interval}" provided to timeRange`);
  }

  // Because we are dealing with Date objects rather than primitives, the value setting for initial
  // selection needs to be matched against closest of the internally stored Dates rather than direct
  // equality testing.
  const findClosestDate = (targetDate) => {
    if (!targetDate || ts.length === 0) {
      return null;
    }
    let closestDate = ts[0];
    let smallestDifference = Math.abs(ts[0] - targetDate);
    for (const date of ts) {
      const difference = Math.abs(date - targetDate);
      if (difference < smallestDifference) {
        smallestDifference = difference;
        closestDate = date;
      }
    }
    return closestDate;
  };

  let minSel, maxSel;
  if (value && value.length === 2) {
    minSel = findClosestDate(value[0]);
    maxSel = findClosestDate(value[1]);
  } else {
    minSel = ts[0];
    maxSel = ts.at(-1);
  }

  return offsetInterval(ts, {
    label: label,
    formatValue: d3.utcFormat(format),
    value: [minSel, maxSel]
  });
}
```

```js
const yearsSlider = html`<div>${timeRange(new Date(dataYearRange[0]-1, 0), new Date(dataYearRange[1], 0), {
    interval: "year",
    label: "Year Range:",
    value: [new Date(dataYearRange[0]-1, 0), new Date(dataYearRange[1], 0)]
  })}</div>`
```

<div>${yearsSlider}</div>

```js
const yearRange = document.getElementsByClassName("range-output")[0]
```

```js
console.log("yearRange", yearRange)
```

```js
const years = yearRange.map(d => (d.getFullYear()+1).toString())
```

```js
function chart1() {

  const width = 975;
  const height = 610;
  
  const path = d3.geoPath();
  const format = d => `${d}%`;

  const stateNames = topojson.feature(us, us.objects.states).features.map(f => f.properties.name);
  const valuemap = new Map(stateNames.map(s => [s, (byStatePerCapita.get(selectedCategory).get(years[1]).get(s) - byStatePerCapita.get(selectedCategory).get(years[0])?.get(s)) / byStatePerCapita.get(selectedCategory).get(years[0])?.get(s)]))
  
  const color = d3.scaleDivergingPow()
    .domain([-100, 0, 100])
    .exponent(0.2)
    .interpolator(d3.interpolateRdBu);

  const counties = topojson.feature(us, us.objects.counties);
  const states = topojson.feature(us, us.objects.states);
  const statemap = new Map(states.features.map(d => [d.id, d]));

  const statemesh = topojson.mesh(us, us.objects.states, (a, b) => a !== b);

  const svg = d3.create("svg")
      .attr("width", width)
      .attr("height", height)
      .attr("viewBox", [0, 0, width, height])
      .attr("style", "max-width: 100%; height: auto;");

  svg.append("g")
      .attr("transform", `translate(${height-70},15)`)
      .append(() => Legend(color, {title: "Percentage Change in Hate Crime Incidents Per Capita", width: 350}));

  svg.append("g")
    .selectAll("path")
    .data(topojson.feature(us, us.objects.states).features)
    .join("path")
      .attr("fill", d => {
        if (!Number.isNaN(valuemap.get(d.properties.name))) {
          return color(valuemap.get(d.properties.name))
        } else if (!!byStatePerCapita.get(selectedCategory).get(years[1])?.get(d.properties.name)) { 
          // Starting year data doesn't exist, but end year does
          return "black"
        } else {
          return "#999da0"
        }
      })
      .attr("d", path)
    .append("title")
      .text(d => {
        if (!Number.isNaN(valuemap.get(d.properties.name))) {
          return `${d.properties.name}\n${valuemap.get(d.properties.name)}%`
        } else if (!!byStatePerCapita.get(selectedCategory).get(years[1])?.get(d.properties.name)) { 
          // Starting year data doesn't exist, but end year does
          return `${d.properties.name}\nMissing data for selected start year`
        } else {
          return `${d.properties.name}\nNo data for range`
        }
      });

  svg.append("path")
      .datum(topojson.mesh(us, us.objects.states, (a, b) => a !== b))
      .attr("fill", "none")
      .attr("stroke", "white")
      .attr("stroke-linejoin", "round")
      .attr("d", path);

  return svg.node();
}
```

<div class="grid grid-cols-1">
  <div class="card">
    ${chart1}
  </div>
</div>