<svelte:options customElement="mirla-barchart" />

<script>
  import { onMount, onDestroy } from 'svelte';
  import * as d3 from 'd3';

  // 1. Svelte 5 Props (Web Component attributes are always strings)
  let { 
    key = "",     // The metadata attribute to graph
    top = "10",   // How many top items to show
    color = "#4a90e2" // Added color customization!
  } = $props();

  // 2. Data Source
  const collectionData = window.MIRLA_COLLECTION_DATA || { items: [] };
  const items = collectionData.items;

  let svgElement = $state();
  let resizeObserver;

  // 3. Reactive Data Processing
  let chartData = $derived.by(() => {
    if (!key || items.length === 0) return [];

    const counts = {};
    for (let item of items) {
      const val = item[key];
      // Ignore empty or undefined values
      if (val !== undefined && val !== "" && val !== null) {
        counts[val] = (counts[val] || 0) + 1;
      }
    }

    let sorted = Object.entries(counts)
      .map(([k, count]) => ({ key: k, count }))
      .sort((a, b) => d3.descending(a.count, b.count));

    const topN = parseInt(top, 10);
    if (!isNaN(topN) && topN > 0) {
      sorted = sorted.slice(0, topN);
    }

    return sorted;
  });

  // Calculate required height based on the number of bars to prevent squishing
  let chartHeight = $derived(chartData.length > 0 ? (chartData.length * 40) + 60 : 300);

  // 4. D3 Drawing Logic
  function drawChart() {
    if (!svgElement || chartData.length === 0) return;

    const svg = d3.select(svgElement);
    svg.selectAll("*").remove(); // Clear previous renders

    // Read the actual computed width of the container
    const rect = svgElement.parentElement.getBoundingClientRect();
    const width = rect.width;
    const height = chartHeight; 

    svg.attr("viewBox", [0, 0, width, height]);

    // Margins (Left is wide to accommodate text labels)
    const margin = { top: 20, right: 30, bottom: 40, left: 140 }; 

    // Scales
    const x = d3.scaleLinear()
      .domain([0, d3.max(chartData, d => d.count)])
      .range([margin.left, width - margin.right])
      .nice();

    const y = d3.scaleBand()
      .domain(chartData.map(d => d.key))
      .range([margin.top, height - margin.bottom])
      .padding(0.2);

    // Axes
    const xAxis = g => g
      .attr("transform", `translate(0,${height - margin.bottom})`)
      .call(d3.axisBottom(x).ticks(Math.max(width / 80, 2)).tickSizeOuter(0))
      .call(g => g.select(".domain").remove()); // cleaner look

    const yAxis = g => g
      .attr("transform", `translate(${margin.left},0)`)
      .call(d3.axisLeft(y).tickSizeOuter(0))
      .selectAll(".tick text")
      .call(wrapText, margin.left - 15); // Wrap long categorical labels!

    // Draw Bars
    svg.append("g")
      .attr("fill", color)
      .selectAll("rect")
      .data(chartData)
      .join("rect")
        .attr("x", x(0))
        .attr("y", d => y(d.key))
        .attr("width", d => Math.max(0, x(d.count) - x(0)))
        .attr("height", y.bandwidth())

    // Append Axes
    svg.append("g").call(xAxis);
    svg.append("g").call(yAxis);
  }

  // Helper function to wrap long SVG text labels
  function wrapText(text, width) {
    text.each(function() {
      let textNode = d3.select(this),
          words = textNode.text().split(/\s+/).reverse(),
          word,
          line = [],
          lineNumber = 0,
          lineHeight = 1.1, // ems
          y = textNode.attr("y"),
          dy = parseFloat(textNode.attr("dy") || 0.32),
          tspan = textNode.text(null).append("tspan").attr("x", -9).attr("y", y).attr("dy", dy + "em");
      while (word = words.pop()) {
        line.push(word);
        tspan.text(line.join(" "));
        if (tspan.node().getComputedTextLength() > width) {
          line.pop();
          tspan.text(line.join(" "));
          line = [word];
          tspan = textNode.append("tspan").attr("x", -9).attr("y", y).attr("dy", ++lineNumber * lineHeight + dy + "em").text(word);
        }
      }
    });
  }

  // 5. Lifecycle & Resizing
  onMount(() => {
    // ResizeObserver is much more efficient than window.on('resize')
    resizeObserver = new ResizeObserver(() => {
      requestAnimationFrame(drawChart);
    });
    
    if (svgElement && svgElement.parentElement) {
      resizeObserver.observe(svgElement.parentElement);
    }
  });

  onDestroy(() => {
    if (resizeObserver) resizeObserver.disconnect();
  });

  // Redraw automatically if props change
  $effect(() => {
    chartData; color; 
    drawChart();
  });
</script>

<div class="mirla-viz-container" style="height: {chartHeight}px;">
  {#if chartData.length === 0}
    <div class="no-data">No data found for key "{key}"</div>
  {:else}
    <svg bind:this={svgElement}></svg>
  {/if}
</div>

<style>
  .mirla-viz-container {
    width: 100%;
    margin: 2em 0;
    font-family: inherit; /* Inherits the Publii theme font */
  }
  
  svg {
    width: 100%;
    height: 100%;
    overflow: visible;
  }
  
  .no-data {
    display: flex;
    align-items: center;
    justify-content: center;
    height: 100%;
    background: #f9f9f9;
    color: #888;
    border: 1px dashed #ccc;
    border-radius: 4px;
    font-family: monospace;
  }

  /* Make D3 text inherit the theme font */
  :global(mirla-barchart svg text) {
    font-family: inherit;
    font-size: 1rem;
    fill: currentColor;
  }
</style>