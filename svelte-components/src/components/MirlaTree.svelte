<svelte:options customElement="mirla-tree" />

<script>
  import { onMount, onDestroy } from 'svelte';
  import * as d3 from 'd3';
  import ItemsBar from "$lib/ItemsBar.svelte";

  // 1. Svelte 5 Props
  let { 
    keys = "",     
    color = "#4a90e2",
    accent = "#ffcc00"
  } = $props();

  // 2. Data Source
  const collectionData = window.MIRLA_COLLECTION_DATA || { items: [] };
  const metadata = collectionData.items;
  const siteDomain = window.MIRLA_CONTEXT?.siteDomain || "";

  // 3. Reactive State
  let svgElement = $state();
  let resizeObserver;
  let selectedItems = $state([]); 
  let activeNodeId = $state(null);

  // 4. Data Processing
  let treeRoot = $derived.by(() => {
    if (!keys || metadata.length === 0) return null;
    const keyList = keys.split(',').map(s => s.trim()).filter(s => s !== "");
    const rollup = d3.rollups(metadata, v => v.length, ...keyList.map(k => (d => d[k] || "N/A")));

    function getChildren(parentList) {
      return parentList.map(parent => {
        const obj = { name: parent[0] };
        if (typeof parent[1] === "number") { obj.value = parent[1]; } 
        else { obj.children = getChildren(parent[1]); }
        return obj;
      });
    }

    const hierarchyData = { name: "Root", children: getChildren(rollup) };
    return d3.hierarchy(hierarchyData).count();
  });

  let leafCount = $derived(treeRoot ? treeRoot.leaves().length : 0);
  let chartHeight = $derived(Math.max(300, leafCount * 30));

  function handleReset() {
    selectedItems = [];
    activeNodeId = null;
    if (svgElement) {
      d3.select(svgElement).selectAll("circle")
        .transition().duration(500)
        .attr("fill", color)
        .attr("stroke-width", 1);
    }
  }

  function getItemsForNode(node) {
    if (node.depth === 0) return metadata;
    const keyList = keys.split(',').map(s => s.trim()).filter(s => s !== "");
    let currentNode = node;
    const filters = [];
    while (currentNode && currentNode.depth > 0) {
      filters.push({ key: keyList[currentNode.depth - 1], value: currentNode.data.name });
      currentNode = currentNode.parent;
    }
    return metadata.filter(item => filters.every(f => String(item[f.key] || "N/A") === String(f.value)));
  }

  // 6. D3 Drawing Logic
  function drawTree() {
    if (!svgElement || !treeRoot) return;

    const svg = d3.select(svgElement);
    svg.selectAll("*").remove();

    const rect = svgElement.parentElement.getBoundingClientRect();
    const width = rect.width;
    const height = chartHeight;
    const margin = { top: 20, right: 120, bottom: 20, left: 60 };

    svg.attr("viewBox", [0, 0, width, height]);

    const treeLayout = d3.tree().size([height - margin.top - margin.bottom, width - margin.left - margin.right]);
    treeLayout(treeRoot);

    const g = svg.append("g").attr("transform", `translate(${margin.left},${margin.top})`);

    g.append("g")
      .attr("fill", "none")
      .attr("stroke", "#ccc")
      .attr("stroke-width", 1.5)
      .attr("stroke-linecap", "round")
      .selectAll("path")
      .data(treeRoot.links())
      .join("path")
        .attr("d", d3.linkHorizontal().x(d => d.y).y(d => d.x));

    const node = g.append("g")
      .selectAll("g")
      .data(treeRoot.descendants())
      .join("g")
        .attr("transform", d => `translate(${d.y},${d.x})`)
        .style("cursor", "pointer")
        .on("click", function(event, d) {
          selectedItems = getItemsForNode(d);
          activeNodeId = d.data.name + d.depth;
          
          // Cohesive visual feedback
          g.selectAll("circle").attr("fill", color).attr("stroke-width", 1);
          d3.select(this).select("circle")
            .attr("fill", accent)
            .attr("stroke-width", 4);
        });

    node.append("circle")
      .attr("fill", d => (activeNodeId === (d.data.name + d.depth)) ? accent : color)
      .attr("stroke", "black")
      .attr("stroke-width", d => (activeNodeId === (d.data.name + d.depth)) ? 2 : 1)
      .attr("r", 6);

    node.append("text")
      .attr("dy", "0.31em")
      .attr("x", d => d.children ? -10 : 10)
      .attr("text-anchor", d => d.children ? "end" : "start")
      .text(d => d.data.name === "Root" ? "" : d.data.name)
      .attr("paint-order", "stroke")
      .attr("stroke", "white")
      .attr("stroke-width", 4)
      .attr("stroke-opacity", 0.6)
      .attr("stroke-linecap", "round")
      .attr("stroke-linejoin", "round");
  }

  onMount(() => {
    resizeObserver = new ResizeObserver(() => requestAnimationFrame(drawTree));
    if (svgElement?.parentElement) resizeObserver.observe(svgElement.parentElement);
  });

  onDestroy(() => resizeObserver?.disconnect());

  $effect(() => { 
    treeRoot; color; accent; 
    drawTree(); 
  });
</script>

<div class="mirla-tree-container">
  <div class="svg-wrapper" style="height: {chartHeight}px;">
    {#if !keys}
      <div class="error">Define 'keys' attribute (e.g., keys="Category,Year")</div>
    {:else}
      <svg bind:this={svgElement}></svg>
    {/if}
  </div>

  <ItemsBar 
    items={selectedItems} 
    {siteDomain} 
    onclose={handleReset} 
  />
</div>

<style>
  .mirla-tree-container {
    width: 100%;
    margin: 2em 0;
    overflow: visible; 
  }

  .svg-wrapper {
    width: 100%;
    overflow: visible;
  }

  svg {
    width: 100%;
    height: 100%;
    display: block;
  }

  .error {
    padding: 2rem;
    text-align: center;
    color: #999;
    font-family: monospace;
    border: 1px dashed #ccc;
  }

  :global(mirla-tree svg text) {
    font-family: inherit;
    font-size: 11px;
    pointer-events: none;
    fill: currentColor;
  }
</style>