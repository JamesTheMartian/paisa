<script lang="ts">
  // Import the tooltip function along with other utils
  import type { Price } from "$lib/utils"; // Adjust path if needed
  import { tooltip } from "$lib/utils"; // Import the tooltip function
  import _ from "lodash";
  import dayjs from "dayjs";
  import weekOfYear from 'dayjs/plugin/weekOfYear'; // Required for week calculation
  import isBetween from 'dayjs/plugin/isBetween'; // Required for filtering
  import { onMount, onDestroy } from 'svelte';
  import * as d3 from 'd3';
  import BoxedTabs from "$lib/components/BoxedTabs.svelte"; // Assuming BoxedTabs can be used here
  import tippy from 'tippy.js'; // Import tippy
  import 'tippy.js/dist/tippy.css'; // Default tooltip theme
  import 'tippy.js/themes/light-border.css'; // Optional theme
  import 'tippy.js/animations/shift-away-subtle.css'; // Optional animation

  dayjs.extend(weekOfYear);
  dayjs.extend(isBetween);

  export let prices: Price[] = [];
  export let commodityName: string = "Commodity";

  type CandlestickData = {
    date: Date; // D3 scales work well with standard Date objects
    open: number;
    high: number;
    low: number;
    close: number;
  };

  // --- Time Range Options ---
  type TimeRangeOption = { label: string; value: string };
  const timeRangeOptions: TimeRangeOption[] = [
    { label: "1W", value: "1w" },
    { label: "4W", value: "4w" },
    { label: "1Y", value: "1y" },
    { label: "3Y", value: "3y" },
    { label: "5Y", value: "5y" },
    { label: "All", value: "all" },
  ];
  let selectedRange: string = "all"; // Default range

  // Function to aggregate daily prices into weekly OHLC data (keep as is)
  function prepareCandlestickData(dailyPrices: Price[]): CandlestickData[] {
    if (!dailyPrices || dailyPrices.length === 0) {
      return [];
    }
    // Ensure prices are sorted chronologically (oldest first)
    const sortedPrices = _.sortBy(dailyPrices, (p) => p.date.valueOf());
    const weeklyDataMap: Record<string, { date: dayjs.Dayjs; open: number; high: number; low: number; close: number; values: number[] }> = {};

    for (const price of sortedPrices) {
      const weekKey = `${price.date.year()}-${price.date.week()}`; // Unique key for each week

      if (!weeklyDataMap[weekKey]) {
        weeklyDataMap[weekKey] = {
          date: price.date.startOf('week'), // Use the start of the week for the x-axis label
          open: price.value, high: price.value, low: price.value, close: price.value, values: [price.value]
        };
      } else {
        weeklyDataMap[weekKey].high = Math.max(weeklyDataMap[weekKey].high, price.value);
        weeklyDataMap[weekKey].low = Math.min(weeklyDataMap[weekKey].low, price.value);
        weeklyDataMap[weekKey].close = price.value; // Last price becomes the close
        weeklyDataMap[weekKey].values.push(price.value);
      }
    }
    // Convert aggregated data to the format needed for D3 and sort by date
    return _.sortBy(Object.values(weeklyDataMap).map(week => ({
      date: week.date.toDate(), // Convert Dayjs object to standard Date object
      open: week.open,
      high: week.high,
      low: week.low,
      close: week.close
    })), d => d.date.valueOf()); // Ensure final data is sorted
  }

  // --- Filtering Logic ---
  function filterDataByRange(data: CandlestickData[], range: string): CandlestickData[] {
    if (range === 'all' || !data || data.length === 0) {
      return data; // Return all data if 'all' or no data
    }

    const now = dayjs();
    let startDate: dayjs.Dayjs;

    switch (range) {
      case '1w':
        startDate = now.subtract(1, 'week');
        break;
      case '4w':
        startDate = now.subtract(4, 'week');
        break;
      case '1y':
        startDate = now.subtract(1, 'year');
        break;
      case '3y':
        startDate = now.subtract(3, 'year');
        break;
      case '5y':
        startDate = now.subtract(5, 'year');
        break;
      default:
        return data; // Should not happen, but return all if invalid range
    }

    // Filter data where the date is after the calculated start date
    // Use startOf('day') on the startDate for consistent comparison
    const startOfDay = startDate.startOf('day');
    return data.filter(d => dayjs(d.date).isAfter(startOfDay));
  }


  let svgElement: SVGSVGElement;
  let containerElement: HTMLDivElement;
  let loading = true;
  let errorMessage: string | null = null;
  let allWeeklyData: CandlestickData[] = []; // Store all aggregated data
  let filteredWeeklyData: CandlestickData[] = []; // Data currently displayed
  let resizeObserver: ResizeObserver | null = null;
  let tippyInstances: any[] = []; // To hold tippy instances for cleanup

  const margin = { top: 30, right: 30, bottom: 40, left: 50 };
  const upwardColor = '#00B746'; // Green
  const downwardColor = '#EF403C'; // Red
  const wickColor = '#555555'; // Dark grey for wicks

  // --- Modified drawChart to accept data ---
  function drawChart(dataToDraw: CandlestickData[]) {
    // Destroy previous tippy instances before redrawing
    tippyInstances.forEach(instance => instance.destroy());
    tippyInstances = [];

    if (!svgElement || !containerElement || dataToDraw.length === 0) {
        // Clear previous drawing if data becomes empty
        if (svgElement) {
            d3.select(svgElement).selectAll('*').remove();
        }
        return;
    }

    const containerWidth = containerElement.clientWidth;
    const containerHeight = 350; // Fixed height for simplicity, adjust as needed

    const width = containerWidth - margin.left - margin.right;
    const height = containerHeight - margin.top - margin.bottom;

    // Clear previous SVG contents
    const svg = d3.select(svgElement)
        .attr('width', containerWidth)
        .attr('height', containerHeight)
        .attr('viewBox', [0, 0, containerWidth, containerHeight])
        .attr('style', 'max-width: 100%; height: auto; height: intrinsic;');

    svg.selectAll('*').remove(); // Clear previous render

    const g = svg.append('g')
        .attr('transform', `translate(${margin.left},${margin.top})`);

    // --- Scales (using dataToDraw) ---
    const xDomain = d3.extent(dataToDraw, d => d.date) as [Date, Date];
    // Add some padding to the x-axis domain if there's more than one data point
    if (xDomain[0] && xDomain[1] && dataToDraw.length > 1) {
        const timeDiff = xDomain[1].getTime() - xDomain[0].getTime();
        const padding = timeDiff * 0.05; // 5% padding on each side
        xDomain[0] = new Date(xDomain[0].getTime() - padding);
        xDomain[1] = new Date(xDomain[1].getTime() + padding);
    } else if (xDomain[0]) {
        // Handle single data point case - add +/- 1 week padding
        xDomain[0] = dayjs(xDomain[0]).subtract(1, 'week').toDate();
        // Use the original date for the end point calculation if only one point
        xDomain[1] = dayjs(dataToDraw[0].date).add(1, 'week').toDate();
    }


    const xScale = d3.scaleTime()
        .domain(xDomain)
        .range([0, width]);

    const yMin = d3.min(dataToDraw, d => d.low) ?? 0;
    const yMax = d3.max(dataToDraw, d => d.high) ?? 1;
    const yPadding = (yMax - yMin) * 0.1; // 10% padding

    const yScale = d3.scaleLinear()
        .domain([yMin - yPadding, yMax + yPadding]).nice() // Add padding and make ticks nice
        .range([height, 0]);

    // --- Axes ---
    const xAxis = d3.axisBottom(xScale)
        .ticks(Math.max(Math.floor(width / 100), 2)) // Adjust number of ticks based on width
        .tickFormat(d3.timeFormat('%d %b %y'));

    const yAxis = d3.axisLeft(yScale)
        .ticks(Math.max(Math.floor(height / 50), 2)) // Adjust based on height
        .tickFormat(d => {
            const num = Number(d);
            if (isNaN(num) || !isFinite(num)) { return String(d); }
            const magnitude = Math.abs(num);
            let precision = 4;
            if (magnitude >= 1) {
                 precision = Math.max(0, 4 - Math.floor(Math.log10(magnitude)));
            } else if (magnitude > 0) {
                precision = 4;
            } else {
                precision = 0;
            }
            return num.toFixed(precision);
        });

    g.append('g')
        .attr('class', 'x-axis')
        .attr('transform', `translate(0,${height})`)
        .call(xAxis)
        .selectAll('text')
          .style('text-anchor', 'end')
          .attr('dx', '-.8em')
          .attr('dy', '.15em')
          .attr('transform', 'rotate(-35)');

    g.append('g')
        .attr('class', 'y-axis')
        .call(yAxis);

    // --- Candlesticks (using dataToDraw) ---
    // Calculate candle width based on the *filtered* data length for better spacing
    const candleWidth = Math.max(1, Math.floor(width / Math.max(dataToDraw.length, 1) * 0.6));

    const candles = g.selectAll('.candlestick')
        .data(dataToDraw) // Use filtered data
        .enter()
        .append('g')
        .attr('class', 'candlestick') // Keep class for targeting
        .attr('transform', d => `translate(${xScale(d.date)}, 0)`)
        // Add data-tippy-content attribute using the tooltip util
        .attr('data-tippy-content', (d: CandlestickData) => {
            const tooltipRows: Array<Array<string | string[]>> = [
                [["Open:", "has-text-weight-bold"], [d.open.toFixed(4), "has-text-right"]],
                [["High:", "has-text-weight-bold"], [d.high.toFixed(4), "has-text-right"]],
                [["Low:", "has-text-weight-bold"], [d.low.toFixed(4), "has-text-right"]],
                [["Close:", "has-text-weight-bold"], [d.close.toFixed(4), "has-text-right"]]
            ];
            return tooltip(tooltipRows, {
                header: dayjs(d.date).format('DD MMM YYYY')
            });
        });


    // Wicks (lines)
    candles.append('line')
        .attr('class', 'wick')
        .attr('y1', d => yScale(d.high))
        .attr('y2', d => yScale(d.low))
        .attr('x1', 0)
        .attr('x2', 0)
        .attr('stroke', wickColor)
        .attr('stroke-width', 1);

    // Bodies (rects)
    candles.append('rect')
        .attr('class', 'candle-body')
        .attr('x', -candleWidth / 2)
        .attr('y', d => yScale(Math.max(d.open, d.close)))
        .attr('width', candleWidth)
        .attr('height', d => Math.max(1, Math.abs(yScale(d.open) - yScale(d.close))))
        .attr('fill', d => d.close >= d.open ? upwardColor : downwardColor)
        .attr('stroke', d => d.close >= d.open ? upwardColor : downwardColor)
        .attr('stroke-width', 1);

    // --- Title ---
    svg.append('text')
       .attr('x', margin.left)
       .attr('y', margin.top / 2)
       .attr('dy', '0.35em')
       .attr('class', 'chart-title')
       .attr('text-anchor', 'start')
       .style('font-size', '14px')
       .text(`${commodityName} Weekly Price (OHLC)`);

    // --- Initialize Tippy Tooltips ---
    // Select all elements with the 'candlestick' class within the SVG container
    const candleNodes = g.selectAll<SVGGElement, CandlestickData>('.candlestick').nodes();
    if (candleNodes.length > 0) {
        tippyInstances = tippy(candleNodes, {
            allowHTML: true,
            theme: 'light-border', // Use the imported theme
            placement: 'top',
            arrow: true,
            animation: 'shift-away-subtle', // Use the imported animation
            // content is read from data-tippy-content attribute by default
        });
    }
  }

  // --- Reactive statement to update chart based on prices OR selectedRange ---
  $: {
    loading = true;
    errorMessage = null;
    // Step 1: Always prepare the full weekly data when prices change
    allWeeklyData = prepareCandlestickData(prices);

    // Step 2: Filter the data based on the selected range
    filteredWeeklyData = filterDataByRange(allWeeklyData, selectedRange);

    // Step 3: Update error message based on filtered data
    if (filteredWeeklyData.length === 0 && allWeeklyData.length > 0) {
         errorMessage = `No data available for the selected period (${selectedRange.toUpperCase()}).`;
    } else if (filteredWeeklyData.length === 0 && allWeeklyData.length === 0) {
         errorMessage = "No price data available.";
    } else {
         errorMessage = null; // Clear error if we have data
    }

    // Step 4: Draw the chart with filtered data if elements are ready
    // Ensure containerElement is available before drawing
    if (containerElement) {
        if (!errorMessage && svgElement) {
            drawChart(filteredWeeklyData);
        } else if (errorMessage && svgElement) {
            // Clear chart if there's an error (e.g., no data for range)
            tippyInstances.forEach(instance => instance.destroy()); // Clean tippy before clearing SVG
            tippyInstances = [];
            d3.select(svgElement).selectAll('*').remove();
        }
    }


    loading = false;
  }

  onMount(() => {
    if (!containerElement) return;

    // Initial draw uses the initially filtered data
    // Defer initial draw slightly to ensure container dimensions are stable
    requestAnimationFrame(() => {
        if (!loading && !errorMessage && filteredWeeklyData.length > 0) {
            drawChart(filteredWeeklyData);
        }
    });


    // Set up ResizeObserver
    resizeObserver = new ResizeObserver(_.debounce(entries => { // Debounce resize redraw
        // Redraw with the *currently filtered* data on resize
        if (!loading && !errorMessage && filteredWeeklyData.length > 0) {
            drawChart(filteredWeeklyData);
        }
    }, 200)); // Debounce by 200ms

    resizeObserver.observe(containerElement);
  });

  onDestroy(() => {
    // Clean up observer
    if (resizeObserver && containerElement) {
        resizeObserver.unobserve(containerElement);
    }
    resizeObserver = null;

    // Clean up tippy instances on component destroy
    tippyInstances.forEach(instance => instance.destroy());
    tippyInstances = [];
  });

</script>

<div class="commodity-chart-container p-2" bind:this={containerElement}>
  <!-- Time Range Selector -->
  <div class="range-selector mb-2 is-flex is-justify-content-flex-end is-align-items-center"> <!-- Added is-align-items-center -->
      {#if prices.length > 0}
        <!-- Removed the label, BoxedTabs likely implies selection -->
      {:else if !loading} <!-- Show message only if not loading and no prices -->
        <span class="has-text-grey mr-2">No data available to display chart</span>
      {/if}
      {#if prices.length > 0} <!-- Only show tabs if there is data -->
        <BoxedTabs bind:value={selectedRange} options={timeRangeOptions} />
      {/if}
  </div>

  {#if loading}
    <div class="is-flex is-justify-content-center is-align-items-center" style="flex-grow: 1;">
        <p>Loading chart...</p>
    </div>
  {:else if errorMessage}
    <div class="is-flex is-justify-content-center is-align-items-center has-text-grey" style="flex-grow: 1;">
        <p>{errorMessage}</p>
    </div>
     <!-- Keep SVG container for consistent height even on error, but hide it -->
     <svg bind:this={svgElement} style="display: none;"></svg>
     <!-- REMOVED old tooltip div -->
     <!-- <div class="tooltip" style="opacity: 0;"></div> -->
  {:else}
    <!-- SVG element where D3 will render the chart -->
    <svg bind:this={svgElement}></svg>
    <!-- REMOVED old tooltip div -->
    <!-- <div class="tooltip" style="opacity: 0;"></div> -->
  {/if}
</div>

<style>
  .commodity-chart-container {
    min-height: 370px; /* Ensure space while loading/error */
    /* position: relative; REMOVED - No longer needed for absolute tooltip */
    display: flex; /* Use flexbox for layout */
    flex-direction: column; /* Stack children vertically */
  }

  .range-selector {
      /* Styles for the button container */
      flex-shrink: 0; /* Prevent selector from shrinking */
      min-height: 30px; /* Ensure space for the selector */
  }

  /* Ensure SVG takes remaining space */
  svg {
      flex-grow: 1; /* Allow SVG to grow */
      display: block; /* Prevent extra space below SVG */
      min-height: 0; /* Allow SVG to shrink if needed in flex context */
  }

  :global(.commodity-chart-container svg .chart-title) {
      font-size: 14px;
      fill: #333; /* Default title color */
  }

  /* Style the SVG elements using CSS */
  :global(.commodity-chart-container svg .x-axis path),
  :global(.commodity-chart-container svg .y-axis path) {
    /* stroke: #ccc; */ /* Lighter axis lines */
    display: none; /* Hide domain line */
  }

  :global(.commodity-chart-container svg .x-axis .tick line),
  :global(.commodity-chart-container svg .y-axis .tick line) {
    stroke: #eee; /* Lighter grid lines */
    stroke-dasharray: 2,2; /* Dashed grid lines */
  }

  :global(.commodity-chart-container svg .x-axis .tick text),
  :global(.commodity-chart-container svg .y-axis .tick text) {
    font-size: 10px;
    fill: #666; /* Grey axis text */
  }

  /* Dark mode adjustments */
  @media (prefers-color-scheme: dark) {
    :global(.commodity-chart-container svg .chart-title) {
        fill: #eee; /* Lighter title for dark mode */
    }

    :global(.commodity-chart-container svg .x-axis .tick line),
    :global(.commodity-chart-container svg .y-axis .tick line) {
        stroke: #444; /* Darker grid lines for dark mode */
    }
    :global(.commodity-chart-container svg .x-axis .tick text),
    :global(.commodity-chart-container svg .y-axis .tick text) {
        fill: #aaa; /* Lighter text for dark mode */
    }
    :global(.commodity-chart-container svg .wick) {
        stroke: #aaaaaa; /* Lighter wicks for dark mode */
    }
    /* REMOVED old tooltip dark mode styles */
  }

  /* REMOVED old tooltip CSS */
  /* .tooltip { ... } */
  /* :global(.tooltip .popup-table) { ... } */
  /* :global(.tooltip .popup-table td) { ... } */

  /* Optional: Customize tippy theme further if needed */
  :global(.tippy-box[data-theme~='light-border'] .tippy-content) {
      padding: 0; /* Remove default padding if table handles it */
  }
  /* Ensure Bulma table styles apply correctly inside tippy */
   :global(.tippy-box .popup-table) {
      margin-bottom: 0;
      background-color: transparent; /* Inherit background */
      color: inherit; /* Inherit text color */
      border-radius: 4px; /* Match tippy box */
      overflow: hidden;
  }
   :global(.tippy-box .popup-table td) {
      padding: 4px 8px;
      border-color: rgba(0,0,0,0.1); /* Adjust border color for theme */
  }

  /* Dark mode adjustments for tippy table */
  @media (prefers-color-scheme: dark) {
      :global(.tippy-box[data-theme~='light-border'][data-theme~='dark'] .popup-table td) {
          border-color: rgba(255,255,255,0.2); /* Lighter border for dark mode */
      }
      /* Tippy usually handles dark mode theme switching, but ensure table inherits */
      :global(.tippy-box[data-theme~='light-border'][data-theme~='dark'] .popup-table) {
          color: inherit;
      }
  }

</style>
