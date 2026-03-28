<svelte:options customElement="mirla-preview" />

<script>
  // 1. Define the props that users will pass via HTML attributes
  let { 
    pid = "", 
    title = "", 
    alt = "", 
    page = "1" // HTML attributes are passed as strings, so we default to "1"
  } = $props();

  // 2. Grab the data from the global payload
  const collectionData = window.MIRLA_COLLECTION_DATA || { items: [] };
  const items = collectionData.items;
  const siteDomain = window.MIRLA_CONTEXT?.siteDomain || "";

  // 3. Derived state to find the item dynamically
  let found = $derived(items.find(d => d.pid === pid));
  
  // Calculate the array index for the image (page 1 = index 0)
  let imageIndex = $derived(Math.max(0, Number(page) - 1));
</script>

{#if found}
  <div class="mirla-preview-item">
    <a href="{siteDomain}/item/{pid}/index.html">
      {#if found.images && found.images[imageIndex]}
        <img 
          src={found.images[imageIndex]} 
          title={title || found.label || pid} 
          alt={alt || title || found.label || pid}
          loading="lazy"
        />
      {:else}
        <div class="no-image-placeholder">No image available</div>
      {/if}
    </a>
    
    <a class="silent-link" href="{siteDomain}/item/{pid}/index.html">
      {found.label || pid}
    </a>
    
    {#if title}
      <p class="custom-title"><em>{title}</em></p>
    {/if}
  </div>
{:else}
  <div class="mirla-error">
    INCORRECT: CHECK THE PID "{pid}"
  </div>
{/if}

<style>
  .mirla-preview-item {
    padding: 1.5em;
    border: 1px solid rgba(0,0,0,0.15);
    border-radius: 8px;
    max-width: 400px;
    margin: 1.5em auto;
    text-align: center;
    background: #ffffff44;
    font-family: inherit;
  }

  .mirla-preview-item img {
    width: 100%;
    height: auto;
    border-radius: 4px;
    transition: transform 0.2s;
  }

  .mirla-preview-item img:hover {
    transform: scale(1.02);
  }

  .silent-link {
    display: block;
    margin-top: 1em;
    font-weight: bold;
    color: inherit;
    text-decoration: none;
  }

  .silent-link:hover {
    text-decoration: underline;
  }

  .custom-title {
    margin-top: 0.5em;
    font-size: 0.9em;
    color: #666;
  }

  .no-image-placeholder {
    width: 100%;
    aspect-ratio: 1;
    background: #eee;
    display: flex;
    align-items: center;
    justify-content: center;
    color: #999;
    border-radius: 4px;
  }

  .mirla-error {
    padding: 1em;
    border: 1px solid red;
    background: #fff0f0;
    color: red;
    font-weight: bold;
    text-align: center;
    margin: 1em 0;
  }
</style>