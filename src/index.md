---
toc: false
---

<div class="hero">
  <h3>Interactive County Explorer</h3>
  <h2>By Tobin Bohley as a part of Ayaan Kazerouni's spring quarter 2026 section of CSC 477</h2>
</div>

# Data Center Explorer

```js echo=false
const counties_unmapped = [
  {
    name: "Loudoun County, VA",
    boundary: await FileAttachment("loudounbord.json").json(),
    towns: await FileAttachment("loudounplace.json").json(),
    statefp: "51",
    countyfp: "107"
  },
  {
    name: "Prince William County, VA",
    boundary: await FileAttachment("tl_2025_us_county.json").json(),
    towns: await FileAttachment("tl_2025_51_place.json").json(),
    statefp: "51",
    countyfp: "153"
  },
  {
    name: "Bucks County, PA",
    boundary: await FileAttachment("bucksbord.json").json(),
    towns: await FileAttachment("bucksplace.json").json(),
    statefp: "42",
    countyfp: "017"
  },
  {
    name: "Maricopa County, AZ",
    boundary: await FileAttachment("maricopabord.json").json(),
    towns: await FileAttachment("maricopaplace.json").json(),
    statefp: "04",
    countyfp: "13"
  },
  {
    name: "Weld County, CO",
    boundary: await FileAttachment("weld2bord@3.json").json(),
    towns: await FileAttachment("weld2place@2.json").json(),
    statefp: "08",
    countyfp: "123"
  },
  {
    name: "Catawba County, NC",
    boundary: await FileAttachment("catawbabord.json").json(),
    towns: await FileAttachment("catawbaplace.json").json(),
    statefp: "37",
    countyfp: "035"
  }
]
```

```js echo=false
const acs = await FileAttachment("acs_places.json").json();
```

```js echo=false
const counties = counties_unmapped.map(county => ({
  ...county,
  towns: {
    ...county.towns,
    features: county.towns.features.map(f => ({
      ...f,
      properties: {
        ...f.properties,
        ...(acs[f.properties.GEOID] ?? {})
      }
    }))
  }
}))
```

```js echo=false
const metrics = [
  { key: "income", label: "Median household income", census: "B19013_001E", format: d3.format("$,") },
  { key: "rent", label: "Median gross rent", census: "B25064_001E", format: d3.format("$,") },
  { key: "homeValue", label: "Median home value", census: "B25077_001E", format: d3.format("$,") },
  { key: "population", label: "Population", census: "B01003_001E", format: d3.format(",") }
]
```

```js echo=false
const us = fetch("https://cdn.jsdelivr.net/npm/us-atlas@3/counties-albers-10m.json")
  .then(r => r.json())

```
```js echo=false
//combined dats of the first two datacenters sets
const combined = (() => {
  const cleanedCounty = (name) => {
    if (!name) return "";
    return name.replace(/\s+(County|Parish|Borough|Census Area|Municipality)$/i, "").trim().toLowerCase();
  };

  const d1 = dataCenters
    .derive({
      name: d => d.name || d.operator || "Unknown",
      mw: d => null,
      status: d => "Operating",
      source: d => "dataCenters",
      join_key: aq.escape(d => d.state_abb + "|" + cleanedCounty(d.county))
    })
    .select("name", "lat", "lon", "state_abb", "county", "operator", "mw", "sqft", "status", "source", "join_key");

  const d2 = dataCenters2
    .rename({
      long: "lon",
      facility_name: "name",
      operator_name: "operator",
      facility_size_sqft: "sqft"
    })
    .derive({
      sqft: d => op.parse_int(op.replace(d.sqft || "0", /,/g, "")),
      mw: d => +d.mw || null,
      source: d => "dataCenters2",
      join_key: aq.escape(d => d.state + "|" + cleanedCounty(d.county))
    })
    .select("name", "lat", "lon", "state", "county", "operator", "mw", "sqft", "status", "source", "join_key");

  return d1.concat(d2).filter(d => d.lat != null && d.lon != null);
})();
```

```js
const dataCenters2 = aq.fromCSV(await FileAttachment("datacenters2.csv").text())
// src: 
```

```js
const dataCenters = aq.fromCSV(await FileAttachment("data_centers.csv").text())
```

```js
import * as aq from "npm:arquero";
```
```js
const {op} = aq;
```

```js echo=false
const mapView = (() => {
  // --- State ---
  let countyIndex = 0;
  let metricIndex = 0;
  let showProposed = false;
  let currentScale = 1;

  // --- Container ---
  const container = html`<div style="font-family: sans-serif;">
    <div style="display:flex; align-items:center; gap:16px; margin-bottom:12px;">
      <button id="prev">←</button>
      <span id="county-label" style="font-size:1.2em; font-weight:bold;"></span>
      <button id="next">→</button>
    </div>
    <div id="metric-btns" style="display:flex; gap:8px; margin-bottom:16px;"></div>
    <div style="margin-bottom:12px;">
      <button id="toggle-proposed" style="padding:4px 12px;">Show Proposed</button>
    </div>
    <svg id="map" width="1200" height="550"></svg>
    <div id="tooltip" style="position:fixed; background:#fff; border:1px solid #ccc; 
      padding:6px 10px; border-radius:4px; font-size:13px; pointer-events:none; opacity:0;"></div>
  </div>`;

  // --- Metric buttons ---
  const metricButtons = [];

  const metricContainer = container.querySelector("#metric-btns");
  metrics.forEach((m, i) => {
    const btn = document.createElement("button");
    btn.textContent = m.label;
    btn.style.cssText = "padding:4px 12px; cursor:pointer;";
  
    btn.addEventListener("click", () => {
      metricIndex = i;
      render();
    });
  
    metricButtons.push(btn);
    metricContainer.appendChild(btn);
  });

  // --- Nav ---
  container.querySelector("#prev").addEventListener("click", () => {
    countyIndex = (countyIndex - 1 + counties.length) % counties.length;
    render();
  });
  container.querySelector("#next").addEventListener("click", () => {
    countyIndex = (countyIndex + 1) % counties.length;
    render();
  });

  container.querySelector("#toggle-proposed").addEventListener("click", () => {
    showProposed = !showProposed;
    container.querySelector("#toggle-proposed").textContent = showProposed ? "Hide Proposed" : "Show Proposed";
    render();
  });

  const svg = d3.select(container.querySelector("#map"));
  const tooltip = d3.select(container.querySelector("#tooltip"));

  const defs = svg.append("defs");

  const pattern = defs.append("pattern")
  .attr("id", "no-data-pattern")
  .attr("patternUnits", "userSpaceOnUse")
  .attr("width", 8)
  .attr("height", 8)
  .attr("patternTransform", "rotate(45)");

pattern.append("rect")
  .attr("width", 8)
  .attr("height", 8)
  .attr("fill", "#eee");

pattern.append("line")
  .attr("x1", 0)
  .attr("y1", 0)
  .attr("x2", 0)
  .attr("y2", 8)
  .attr("stroke", "red")
  .attr("stroke-width", 2);



  
  function render() {
    svg.selectAll(".town").remove();
    svg.selectAll(".boundary").remove();
    svg.selectAll(".dc").remove();
    svg.selectAll(".legend").remove();
    const county = counties[countyIndex];
    const boundary = county.boundary;
    const towns = county.towns;
    const metric = metrics[metricIndex];
    const countyShortName = county.name.split(",")[0];

    metricButtons.forEach((btn, i) => {
      if (i === metricIndex) {
        btn.style.background = "#1976d2";
        btn.style.color = "white";
        btn.style.fontWeight = "bold";
      } else {
        btn.style.background = "";
        btn.style.color = "";
        btn.style.fontWeight = "";
      }
    });

    
    const dcs = combined
      .filter(aq.escape(d => 
        (d.status === "Operating" || (showProposed && d.status === "Proposed")) &&
        (d.county === countyShortName || d.county === countyShortName.replace(" County", ""))
      ))
      .objects();

    const operatingDcs = combined
  .filter(aq.escape(d => 
    d.status === "Operating" &&
    (d.county === countyShortName || d.county === countyShortName.replace(" County", ""))
  ))
  .objects();

    const sqftValues = operatingDcs.map(d => d.sqft).filter(v => v != null && v > 0);
    const medianSqft = d3.median(sqftValues);
    const sqft = d => (d.sqft == null || d.sqft === 0) ? medianSqft : d.sqft;


    const sqftScale = d3.scaleSqrt()
      .domain([0, d3.max(operatingDcs, d => d.sqft ?? 5000) ?? 1000000])
      .range([4, 24]);

    container.querySelector("#county-label").textContent = county.name;

    //svg.selectAll("*").remove();

    // --- Projection fitted to county boundary ---
    const projection = d3.geoIdentity()
      .reflectY(true)
      .fitSize([900, 550], boundary);

    const path = d3.geoPath().projection(projection);

    // --- Color scale ---
    const values = towns.features.map(f => f.properties[metric.key]).filter(v => v != null);
    const colorScale = d3.scaleSequential()
      .domain(d3.extent(values))
      .interpolator(d3.interpolateBlues);

    // --- Vertical Legend for chloropleth---
    const legendWidth = 18;
    const legendHeight = 200;
    
    const legend = svg.append("g")
      .attr("class", "legend")
      .attr("transform", "translate(900, 40)");
    
    const legendId = `legend-gradient-${countyIndex}-${metricIndex}`;
    
    const defs = svg.append("defs");
    
    const gradient = defs.append("linearGradient")
      .attr("id", legendId)
      .attr("x1", "0%")
      .attr("y1", "100%")
      .attr("x2", "0%")
      .attr("y2", "0%");
    
    gradient.selectAll("stop")
      .data(d3.range(0, 1.01, 0.1))
      .join("stop")
      .attr("offset", d => `${d * 100}%`)
      .attr("stop-color", d =>
        colorScale(
          colorScale.domain()[0] +
          d * (colorScale.domain()[1] - colorScale.domain()[0])
        )
      );
    
    legend.append("rect")
      .attr("width", legendWidth)
      .attr("height", legendHeight)
      .attr("fill", `url(#${legendId})`)
      .attr("stroke", "#999");
    
    const legendScale = d3.scaleLinear()
      .domain(colorScale.domain())
      .range([legendHeight, 0]);
    
    legend.append("g")
      .attr("transform", `translate(${legendWidth},0)`)
      .call(
        d3.axisRight(legendScale)
          .ticks(5)
          .tickFormat(metric.format ?? d3.format(","))
      );
    
    legend.append("text")
      .attr("x", 0)
      .attr("y", -12)
      .attr("font-size", 12)
      .attr("font-weight", "bold")
      .text(metric.label);

    // --- Datacenter Size Legend ---
    const fallbackLegendValues = [100000, 500000, 1000000];
    
    const countyMaxSqft = d3.max(operatingDcs, d => sqft(d));
    const sizeLegendValues = countyMaxSqft
      ? fallbackLegendValues.filter(v => v <= countyMaxSqft)
      : fallbackLegendValues;
    
    const sizeLegend = svg.append("g")
      .attr("class", "legend")
      .attr("transform", "translate(900, 300)");
    
    sizeLegend.append("text")
      .attr("x", 0)
      .attr("y", -15)
      .attr("font-size", 12)
      .attr("font-weight", "bold")
      .text("Datacenter Size");
    
    const maxR = sqftScale(d3.max(sizeLegendValues));
    
    [...sizeLegendValues].reverse().forEach((value, i) => {
      const r = sqftScale(value);
      const y = maxR * 2 + i * (maxR * 2.5);
    
      sizeLegend.append("circle")
        .attr("cx", r)
        .attr("cy", y)
        .attr("r", r)
        .attr("fill", "none")
        .attr("stroke", "#333");
    
      sizeLegend.append("line")
        .attr("x1", r * 2)
        .attr("x2", r * 2 + 30)
        .attr("y1", y)
        .attr("y2", y)
        .attr("stroke", "#333");
    
      sizeLegend.append("text")
        .attr("x", r * 2 + 35)
        .attr("y", y + 4)
        .attr("font-size", 11)
        .text(`${d3.format(",")(value)} sqft`);
    });

    // --- County boundary layer ---
    svg.selectAll(".boundary")
      .data(boundary.features)
      .join("path")
      .attr("class", "boundary")
      .attr("d", path)
      .attr("fill", "white")
      .attr("stroke", "#333")
      .attr("stroke-width", 2)
      .style("pointer-events", "none");

        // --- Towns layer ---
    svg.selectAll(".town")
      .data(towns.features)
      .join("path")
      .attr("class", "town")
      .attr("d", path)
      .attr("fill", d => {
        const v = d.properties[metric.key];
        return (v != null && v > 0) ? colorScale(v) : "url(#no-data-pattern)";
      })
      .attr("stroke", "#999")
      .attr("stroke-width", 0.5)
      .on("mousemove", (event, d) => {
          const value = d.properties[metric.key];
          tooltip
            .style("opacity", 1)
            .style("left", (event.clientX + 12) + "px")
            .style("top", (event.clientY - 28) + "px")
            .html(`
              <strong>${d.properties.NAME ?? d.properties.TO_TOWN}</strong><br>
              ${metric.label}: ${
                value != null
                  ? (metric.format ? metric.format(value) : value)
                  : "No data"
              }
            `);
            })
      .on("mouseleave", () => tooltip.style("opacity", 0));

    // --- Datacenter layer ---
    svg.selectAll(".dc")
      .data(dcs.slice().sort((a, b) => d3.descending(sqft(a), sqft(b))))
      .join("circle")
      .attr("class", "dc")
      .attr("cx", d => projection([d.lon, d.lat])?.[0])
      .attr("cy", d => projection([d.lon, d.lat])?.[1])
      .attr("r", d => sqftScale(sqft(d)))
      .attr("fill", d => d.status === "Proposed" ? "orange" : "red")
      .attr("stroke", "#fff")
      .attr("stroke-width", 1.5)
      .on("mousemove", (event, d) => {
        tooltip
          .style("opacity", 1)
          .style("left", (event.clientX + 12) + "px")
          .style("top", (event.clientY - 28) + "px")
          .html(`
            <strong>${d.name}</strong><br>
            Status: ${d.status}<br>
            Operator: ${d.operator ?? "Unknown"}<br>
            Size: ${d.sqft != null ? d.sqft.toLocaleString() + " sqft" : "Unknown"}
          `);
      })
      .on("mouseleave", () => tooltip.style("opacity", 0));
  }

  render();
  return container;
})();

display(mapView);
```


# Writeup

This was the result of a somewhat rushed work process over the last week-ish. While I'm happy with the look, encodings, and technical "correctness" of my visualization, it was a lot harder to convey the originally intended insights than I imagined it would be.  
My choices of visual encoding didn't change much from what I originally drafted in the planning of this assignment. I wanted the color of the map to display a chosen variable, selectable by the user. I figured this would be the best way to view close-by vs. distant effects of datacenters.  
The datacenters themselves are daunting red dots that make them stand out easily while not taking up so much of the space that you can't see what's going on behind them.  

Aside from those two choices, much of the design and implementation of this visual was forced on me by implementation/data difficulties. I would have liked to offer better metrics to users than the 4 that were easily obtainable, which was why I started off limited to a selection of only 6 browsable counties. I was hoping to find joinable data on some of the environmental factors impacted by datacenters, like water pollution measurements and electricity pricing.  
This quickly became a goal that was out of reach once I got to actually searching for good data.  
The end result is a fair bit of lost potential; all of the data I did end up using came directly from the US Census Bureau through a Python script that sent API requests and constructed a usable json of datapoints. Had I known that I would end up using data that was easily accessable/processable *en masse* from government sources, I may have instead decided to implement this using pan and zoom with the entire country as explorable data.  
To be fair if I had started this earlier I could go back and change it to be that way now, but... I didn't. Busy times.   
  
Admittedly many of my final choices of interactivity came from ease of implementation; however; I am happy with the feel and level of explorability offered here. Just less so with the variables displayed themselves.

# Data Sources

Boundaries and geographic features from the U.S. Census Bureau TIGER/Line Shapefiles. U.S. Census Bureau, Geography Division. Available at Census TIGER/Line Shapefiles. (https://www.census.gov/cgi-bin/geo/shapefiles/index.php)

Demographic data from the U.S. Census Bureau American Community Survey (ACS) 5-Year Estimates.

Data center locations: [IM3 Open Source Data Center Atlas](https://www.osti.gov/biblio/2550666), Pacific Northwest National Laboratory (2025)


<style>

.hero {
  display: flex;
  flex-direction: column;
  align-items: center;
  font-family: var(--sans-serif);
  margin: 4rem 0 8rem;
  text-wrap: balance;
  text-align: center;
}

.hero h1 {
  margin: 1rem 0;
  padding: 1rem 0;
  max-width: none;
  font-size: 14vw;
  font-weight: 900;
  line-height: 1;
  background: linear-gradient(30deg, var(--theme-foreground-focus), currentColor);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

.hero h2 {
  margin: 0;
  max-width: 34em;
  font-size: 20px;
  font-style: initial;
  font-weight: 500;
  line-height: 1.5;
  color: var(--theme-foreground-muted);
}

@media (min-width: 640px) {
  .hero h1 {
    font-size: 90px;
  }
}

</style>
