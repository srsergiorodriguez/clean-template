<svelte:options customElement="mirla-network" />

<script>
  import { onMount, onDestroy } from 'svelte';
  import * as d3 from 'd3';
  import ItemsBar from "$lib/ItemsBar.svelte";

  let { 
    sourceKey = "", 
    targetKey = "", 
    color = "#4a90e2",
    accentSource = "#e06363",
    accentTarget = "#f4f4f4"
  } = $props();

  const collectionData = window.MIRLA_COLLECTION_DATA || { items: [] };
  const metadata = collectionData.items;
  const siteDomain = window.MIRLA_CONTEXT?.siteDomain || "";

  let svgElement = $state();
  let resizeObserver;
  let selectedItems = $state([]); 
  let activeNodeId = $state(null);
  let simulation;
  let zoomBehavior;

  const splitValues = (val) => {
    if (val === null || val === undefined) return [];
    if (typeof val !== 'string') return [String(val)];
    return val.split(/[;,]+/).map(s => s.trim()).filter(s => s !== "");
  };

  let graph = $derived.by(() => {
    if (!sourceKey || !targetKey || metadata.length === 0) return { nodes: [], links: [] };
    const nodeMap = new Map();
    const links = [];
    metadata.forEach(item => {
      const sources = splitValues(item[sourceKey]);
      const targets = splitValues(item[targetKey]);
      sources.forEach(src => {
        if (!nodeMap.has(src)) nodeMap.set(src, { id: src, type: 'source', count: 0 });
        nodeMap.get(src).count++;
        targets.forEach(tgt => {
          if (!nodeMap.has(tgt)) nodeMap.set(tgt, { id: tgt, type: 'target', count: 0 });
          nodeMap.get(tgt).count++;
          links.push({ source: src, target: tgt });
        });
      });
    });
    return { nodes: Array.from(nodeMap.values()), links };
  });

  function fitToView(transitionDuration = 750) {
    if (!svgElement || graph.nodes.length === 0) return;
    const width = svgElement.parentElement.clientWidth;
    const height = 500;
    let minX = d3.min(graph.nodes, d => d.x);
    let maxX = d3.max(graph.nodes, d => d.x);
    let minY = d3.min(graph.nodes, d => d.y);
    let maxY = d3.max(graph.nodes, d => d.y);
    const graphWidth = maxX - minX;
    const graphHeight = maxY - minY;
    const centerX = (minX + maxX) / 2;
    const centerY = (minY + maxY) / 2;
    const padding = 0.7; // Generous padding to account for wide labels on edges
    const scale = Math.min(2, padding / Math.max(graphWidth / width, graphHeight / height));
    const transform = d3.zoomIdentity.translate(width / 2, height / 2).scale(scale).translate(-centerX, -centerY);
    
    if (transitionDuration > 0) {
      d3.select(svgElement).transition().duration(transitionDuration).call(zoomBehavior.transform, transform);
    } else {
      d3.select(svgElement).call(zoomBehavior.transform, transform);
    }
  }

  function handleReset() {
    selectedItems = [];
    activeNodeId = null;
    if (svgElement) {
      const svg = d3.select(svgElement);
      svg.selectAll("circle").attr("stroke-width", 1);
      svg.selectAll("text").attr("opacity", 1);
      svg.selectAll("line").attr("stroke", "#ccc").attr("stroke-width", 1);
      fitToView(750);
    }
  }

  function initSimulation() {
    if (!svgElement || graph.nodes.length === 0) return;
    if (simulation) simulation.stop();
    const svg = d3.select(svgElement);
    svg.selectAll("*").remove();
    const width = svgElement.parentElement.clientWidth;
    const height = 500;
    svg.attr("viewBox", [0, 0, width, height]);

    const defs = svg.append("defs");
    defs.append("clipPath").attr("id", "clip-network").append("rect").attr("width", width).attr("height", height);

    const radiusScale = d3.scaleSqrt()
      .domain([1, d3.max(graph.nodes, d => d.count)])
      .range([4, 16]);

    const gMain = svg.append("g").attr("clip-path", "url(#clip-network)");
    const gContent = gMain.append("g");

    zoomBehavior = d3.zoom().on("zoom", ({transform}) => {
      gContent.attr("transform", transform);
    });
    svg.call(zoomBehavior);

    // BALANCED FORCES: Better breathing room, stronger gravity
    simulation = d3.forceSimulation(graph.nodes)
      .force("link", d3.forceLink(graph.links).id(d => d.id).distance(100))
      .force("charge", d3.forceManyBody().strength(-300)) // Stronger repulsion for label space
      .force("x", d3.forceX(width / 2).strength(0.15)) // Higher gravity to counteract repulsion
      .force("y", d3.forceY(height / 2).strength(0.15))
      .force("collide", d3.forceCollide(d => radiusScale(d.count) + 5)); // Just for dots

    for (let i = 0; i < 200; ++i) simulation.tick();

    const link = gContent.append("g")
      .attr("stroke", "#ccc")
      .attr("stroke-opacity", 0.6)
      .selectAll("line").data(graph.links).join("line");

    const node = gContent.append("g").selectAll("g").data(graph.nodes).join("g")
      .style("cursor", "pointer")
      .on("click", (event, d) => { activeNodeId = d.id; filterNetwork(d.id); })
      .call(d3.drag()
        .on("start", (e, d) => { if (!e.active) simulation.alphaTarget(0.3).restart(); d.fx = d.x; d.fy = d.y; })
        .on("drag", (e, d) => { d.fx = event.x; d.fy = event.y; })
        .on("end", (e, d) => { if (!e.active) simulation.alphaTarget(0); d.fx = null; d.fy = null; }));

    node.append("circle")
      .attr("r", d => radiusScale(d.count))
      .attr("fill", d => d.type === 'source' ? accentSource : accentTarget)
      .attr("stroke", "black").attr("stroke-width", 1);

    node.append("text")
      .attr("x", d => radiusScale(d.count) + 5)
      .attr("dy", "0.31em")
      .text(d => d.id)
      .attr("paint-order", "stroke").attr("stroke", "white").attr("stroke-width", 4)
      .attr("stroke-opacity", 0.6).attr("stroke-linecap", "round").attr("stroke-linejoin", "round");

    simulation.on("tick", () => {
      link.attr("x1", d => d.source.x).attr("y1", d => d.source.y)
          .attr("x2", d => d.target.x).attr("y2", d => d.target.y);
      node.attr("transform", d => `translate(${d.x},${d.y})`);
    });

    fitToView(0);

    function filterNetwork(nodeId) {
      selectedItems = metadata.filter(item => {
        const vals = [...splitValues(item[sourceKey]), ...splitValues(item[targetKey])];
        return vals.includes(nodeId);
      });
      const neighbors = new Set([nodeId]);
      graph.links.forEach(l => {
        if (l.source.id === nodeId) neighbors.add(l.target.id);
        if (l.target.id === nodeId) neighbors.add(l.source.id);
      });
      gContent.selectAll("circle").attr("stroke-width", d => neighbors.has(d.id) ? 4 : 1);
      gContent.selectAll("text").attr("opacity", d => neighbors.has(d.id) ? 1 : 0.2);
      gContent.selectAll("line").attr("stroke", d => (d.source.id === nodeId || d.target.id === nodeId) ? '#000' : '#ccc');
    }
  }

  onMount(() => {
    resizeObserver = new ResizeObserver(() => requestAnimationFrame(initSimulation));
    resizeObserver.observe(svgElement.parentElement);
  });
  onDestroy(() => { simulation?.stop(); resizeObserver?.disconnect(); });
  $effect(() => { graph; color; accentSource; accentTarget; initSimulation(); });
</script>

<div class="mirla-network-container">
  <div class="svg-wrapper">
    <svg bind:this={svgElement}></svg>
    {#if activeNodeId}
      <button class="reset-zoom-btn" onclick={handleReset} title="Fit All">↺</button>
    {/if}
  </div>
  <ItemsBar items={selectedItems} {siteDomain} onclose={handleReset} />
</div>

<style>
  .mirla-network-container { width: 100%; margin: 2em 0; position: relative; }
  .svg-wrapper { width: 100%; height: 500px; position: relative; overflow: hidden; border: 1px solid rgba(0,0,0,0); border-radius: 8px; }
  svg { width: 100%; height: 100%; display: block; cursor: grab; }
  svg:active { cursor: grabbing; }
  .reset-zoom-btn { position: absolute; top: 10px; left: 10px; background: rgba(255, 255, 255, 0.4); border: 1px solid rgba(0,0,0,0.1); border-radius: 50%; width: 32px; height: 32px; cursor: pointer; display: flex; align-items: center; justify-content: center; font-size: 1.2rem; z-index: 10; }
  :global(mirla-network svg text) { font-family: inherit; font-size: 11px; fill: currentColor; pointer-events: none; }
</style>