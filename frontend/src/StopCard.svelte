<script lang="ts">
  import { createEventDispatcher } from "svelte";
  import type { StopArrival } from "./lib/types";
  import { currentTime } from "./lib/stores";

  export let stop: StopArrival;
  export let hiddenRoutes: string[] = [];
  export let selectedRoute: string | undefined = undefined;
  export let browsing: boolean = false;

  const dispatch = createEventDispatcher();

  let showFilters = false;

  function toggleRoute(route: string) {
    dispatch("toggleHidden", route);
  }

  function getIcon(type: string): string {
    switch (type) {
      case "bus":
        return "🚌";
      case "commercialbus":
        return "🚍";
      case "regionalbus":
        return "🚍";
      case "train":
        return "🚆";
      case "tram":
        return "🚋";
      case "trolleybus":
        return "🚎";
      default:
        return "❓";
    }
  }

  function formatTime(timestamp: number): string {
    const date = new Date(timestamp);
    return date.toLocaleTimeString("en-US", {
      hour: "2-digit",
      minute: "2-digit",
      hour12: false,
    });
  }

  $: allArrivals = Object.entries(stop.arrivals).flatMap(([type, routes]) =>
    Object.entries(routes).flatMap(([route, arrivals]) =>
      arrivals.map((arrival) => ({
        route,
        type,
        timestamp: arrival.time,
        time: formatTime(arrival.time),
        isLowEntry: arrival.isLowEntry,
      }))
    )
  );

  $: getCountdown = (timestamp: number) => {
    const now = $currentTime;
    const diff = Math.floor((timestamp - now) / 1000);
    if (diff <= 0) return "Now";
    const minutes = Math.floor(diff / 60);
    const seconds = diff % 60;
    return minutes > 0 ? `${minutes}m` : `${seconds}s`;
  };

  $: allTimes = [...new Set(allArrivals.map((a) => a.time))].sort();

  $: visibleRoutes = Object.entries(stop.arrivals)
    .flatMap(([type, routes]) =>
      Object.entries(routes)
        .filter(([route]) => browsing || !hiddenRoutes.includes(route))
        .map(([route]) => route)
    )
    .sort((a, b) => {
      if (browsing && selectedRoute) {
        if (a === selectedRoute) return -1;
        if (b === selectedRoute) return 1;
      }
      return 0;
    });
</script>

<div class="stop-card">
  <div class="stop-header">
    <div class="title-group">
      <h3>{stop.name}</h3>
      <span class="stop-id">{stop.id}</span>
    </div>
    <div class="actions">
      {#if !browsing}
        <button
          class="icon-btn secondary"
          onclick={() => (showFilters = !showFilters)}
          title="Filter routes"
        >
          ⚙️
        </button>
        <button
          class="icon-btn secondary remove"
          onclick={() => dispatch("remove")}
          title="Remove stop"
        >
          ✕
        </button>
      {/if}
    </div>
  </div>

  {#if showFilters && !browsing}
    <div class="filters">
      <p class="filter-title">Visible Routes</p>
      <div class="filter-grid">
        {#each Object.entries(stop.arrivals) as [type, routes]}
          {#each Object.entries(routes) as [route]}
            <label class="filter-item">
              <input
                type="checkbox"
                checked={!hiddenRoutes.includes(route)}
                onchange={() => toggleRoute(route)}
              />
              <span class="route-badge small">{route}</span>
            </label>
          {/each}
        {/each}
      </div>
    </div>
  {/if}

  <div class="arrivals">
    {#if allTimes.length === 0}
      <div class="empty-state">
        <p>No arrivals available at this time.</p>
      </div>
    {:else}
      <div class="routes-container">
        {#each visibleRoutes as route}
          {@const routeArrivals = allArrivals.filter((a) => a.route === route)}
          {@const routeType = routeArrivals[0]?.type || ""}
          <div class="route-card">
            <div class="route-info">
              <span class="route-type-icon">{getIcon(routeType)}</span>
              <span class="route-badge">{route}</span>
            </div>
            <div class="times-group">
              {#each routeArrivals as arrival}
                <div
                  class="arrival-token"
                  class:imminent={getCountdown(arrival.timestamp) === "Now"}
                >
                  <span class="countdown"
                    >{getCountdown(arrival.timestamp)}</span
                  >
                  <span class="clock-time">{arrival.time}</span>
                  {#if arrival.isLowEntry}
                    <span class="accessibility-icon" title="Low entry">♿</span>
                  {/if}
                </div>
              {/each}
            </div>
          </div>
        {/each}
      </div>
    {/if}
  </div>
</div>

<style>
  .stop-card {
    border: 1px solid var(--border-color);
    border-radius: var(--radius-lg);
    padding: 1.5rem;
    margin-bottom: 1.5rem;
    background: var(--card-bg);
    box-shadow: var(--shadow-sm);
    transition: box-shadow 0.2s ease;
  }

  .stop-card:hover {
    box-shadow: var(--shadow-md);
  }

  .stop-header {
    display: flex;
    justify-content: space-between;
    align-items: flex-start;
    margin-bottom: 1.5rem;
  }

  .title-group h3 {
    font-size: 1.125rem;
    font-weight: 700;
    margin-bottom: 0.25rem;
  }

  .stop-id {
    font-size: 0.75rem;
    color: var(--secondary-text);
    font-family: monospace;
  }

  .actions {
    display: flex;
    gap: 0.5rem;
  }

  .icon-btn {
    padding: 0.5rem;
    width: 2rem;
    height: 2rem;
    border-radius: 50%;
    font-size: 1rem;
    line-height: 1;
  }

  .icon-btn.remove:hover {
    color: #ef4444;
    border-color: #ef4444;
  }

  .filters {
    margin-bottom: 1.5rem;
    padding: 1rem;
    background: var(--bg-color);
    border-radius: var(--radius-md);
  }

  .filter-title {
    font-size: 0.75rem;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.05em;
    color: var(--secondary-text);
    margin-bottom: 0.75rem;
  }

  .filter-grid {
    display: flex;
    flex-wrap: wrap;
    gap: 0.75rem;
  }

  .filter-item {
    display: flex;
    align-items: center;
    gap: 0.375rem;
    cursor: pointer;
  }

  .route-badge {
    background: var(--accent-color);
    color: var(--accent-text-color);
    padding: 0.25rem 0.5rem;
    border-radius: 6px;
    font-size: 0.875rem;
    font-weight: 700;
    min-width: 2.5rem;
    text-align: center;
  }

  .route-badge.small {
    font-size: 0.75rem;
    min-width: 2rem;
  }

  .routes-container {
    display: flex;
    flex-direction: column;
    gap: 1rem;
  }

  .route-card {
    display: flex;
    align-items: center;
    gap: 1rem;
    padding: 0.5rem 0;
  }

  .route-info {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    min-width: 5rem;
  }

  .route-type-icon {
    font-size: 1.25rem;
  }

  .times-group {
    display: flex;
    gap: 0.75rem;
    overflow-x: auto;
    padding: 0.25rem 0;
    scrollbar-width: none;
  }

  .times-group::-webkit-scrollbar {
    display: none;
  }

  .arrival-token {
    display: flex;
    flex-direction: column;
    align-items: center;
    padding: 0.5rem 0.75rem;
    background: var(--bg-color);
    border: 1px solid var(--border-color);
    border-radius: var(--radius-md);
    min-width: 4rem;
    position: relative;
  }

  .arrival-token.imminent {
    border-color: var(--accent-color);
    background: rgba(59, 130, 246, 0.1);
  }

  .countdown {
    font-size: 0.875rem;
    font-weight: 700;
    color: var(--accent-color);
  }

  .clock-time {
    font-size: 0.625rem;
    color: var(--secondary-text);
  }

  .accessibility-icon {
    position: absolute;
    top: -4px;
    right: -4px;
    font-size: 0.75rem;
    background: var(--card-bg);
    border: 1px solid var(--border-color);
    border-radius: 50%;
    width: 1rem;
    height: 1rem;
    display: flex;
    align-items: center;
    justify-content: center;
  }

  .empty-state {
    text-align: center;
    padding: 2rem;
    color: var(--secondary-text);
    background: var(--bg-color);
    border-radius: var(--radius-md);
  }
</style>
