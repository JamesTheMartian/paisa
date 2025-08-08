<script lang="ts">
  import Toggleable from "$lib/components/Toggleable.svelte";
  import ValueChange from "$lib/components/ValueChange.svelte";
  import { ajax, formatCurrency, type Price } from "$lib/utils";
  import { toast } from "bulma-toast";
  import _ from "lodash";
  import { onMount } from "svelte";
  import VirtualList from "svelte-tiny-virtual-list";
  import CommodityChart from '$lib/components/CommodityChart.svelte'; // Import the new chart component
  import dayjs from 'dayjs'; // Make sure dayjs is imported here too for Price type usage
  import isSameOrBefore from 'dayjs/plugin/isSameOrBefore'; // Needed for the 'change' function
  dayjs.extend(isSameOrBefore);

  let prices: Record<string, Price[]> = {};

  const ITEM_SIZE = 18;

  function change(prices: Price[], days: number, tolerance: number) {
    // Ensure prices are sorted newest first for this calculation
    const sortedPrices = _.orderBy(prices, p => p.date.valueOf(), 'desc');
    const first = sortedPrices[0];
    if (!first) return null;

    const targetDate = first.date.subtract(days, "day");
    // Find the price closest to the target date, preferring one on or before
    const last = _.find(sortedPrices, (p) => p.date.isSameOrBefore(targetDate, "day"));
    if (!last) return null;

    const diffDays = first.date.diff(last.date, "day");
    // Check tolerance: The difference should be close to `days`
    if (Math.abs(diffDays - days) <= tolerance) {
      // Avoid division by zero
      if (last.value === 0) return null;
      return (first.value - last.value) / last.value;
    }
    return null;
  }

  async function clearPriceCache() {
    const { success, message } = await ajax("/api/price/delete", { method: "POST" });
    if (!success) {
      toast({
        message: `Failed to clear price cache. reason: ${message}`,
        type: "is-danger",
        duration: 10000
      });
    } else {
      toast({
        message: "Price cache cleared.",
        type: "is-success"
      });
    }
    await fetchPrice();
  }

  async function fetchPrice() {
    // VERY IMPORTANT: Convert date strings to Dayjs objects upon fetching
    const rawData = await ajax("/api/price");
    const processedPrices: Record<string, Price[]> = {};

    for (const commodity in rawData.prices) {
        if (rawData.prices[commodity].length > 0) {
             processedPrices[commodity] = rawData.prices[commodity].map((p: any) => ({
                ...p,
                date: dayjs(p.date) // Convert string/timestamp to Dayjs object
             })).filter((p: Price) => p.date.isValid()); // Filter out invalid dates

             // Optional: Sort here once (e.g., newest first) if needed by other parts,
             // but the chart component sorts oldest->newest internally
             processedPrices[commodity] = _.orderBy(processedPrices[commodity], p => p.date.valueOf(), 'desc');
        }
    }
    prices = processedPrices;

    // Original line (replace with processing loop above):
    // ({ prices: prices } = await ajax("/api/price"));
    // prices = _.omitBy(prices, (v) => v.length === 0);
  }

  onMount(async () => {
    await fetchPrice();
  });
</script>

<section class="section tab-price">
  <div class="container is-fluid">
    <div class="columns flex-wrap">
      <div class="column is-12">
        <div class="box p-3">
          <div class="field has-addons mb-0">
            <p class="control">
              <button
                class="button is-small is-link invertable is-light is-danger"
                on:click={(_e) => clearPriceCache()}
              >
                <span class="icon is-small">
                  <i class="fas fa-trash-can" />
                </span>
                <span>Clear Price Cache</span>
              </button>
            </p>
          </div>
        </div>
      </div>
      <div class="column is-12">
        <div class="box overflow-x-auto">
          <table class="table is-narrow is-fullwidth is-light-border is-hoverable">
            <thead>
              <tr>
                <th />
                <th>Commodity Name</th>
                <th>Last Date</th>
                <th class="has-text-right">Last Price</th>
                <th class="has-text-right">1 Day</th>
                <th class="has-text-right">1 Week</th>
                <th class="has-text-right">4 Weeks</th>
                <th class="has-text-right">1 Year</th>
                <th class="has-text-right">3 Years</th>
                <th class="has-text-right">5 Years</th>
                <th>Commodity Type</th>
                <th>Commodity ID</th>
              </tr>
            </thead>
            <tbody class="has-text-grey-dark">
              {#each Object.keys(prices) as commodity}
                {@const priceArray = prices[commodity]}
                {@const p = prices[commodity][0]}
                <Toggleable>
                  <tr
                    class={active ? "is-active" : ""}
                    style="cursor: pointer;"
                    slot="toggle"
                    let:active
                    let:onclick
                    on:click={(e) => onclick(e)}
                  >
                    <td>
                      <span class="icon has-text-link">
                        <i
                          class="fas {active ? 'fa-chevron-up' : 'fa-chevron-down'}"
                          aria-hidden="true"
                        />
                      </span>
                    </td>

                    <td>{p.commodity_name}</td>
                    <td class="whitespace-nowrap">{p.date.format("DD MMM YYYY")}</td>
                    <td class="has-text-right">{formatCurrency(p.value, 4)}</td>
                    <td class="has-text-right"
                      ><ValueChange value={change(prices[commodity], 1, 0)} /></td
                    >
                    <td class="has-text-right"
                      ><ValueChange value={change(prices[commodity], 7, 2)} /></td
                    >
                    <td class="has-text-right"
                      ><ValueChange value={change(prices[commodity], 28, 4)} />
                    </td>
                    <td class="has-text-right"
                      ><ValueChange value={change(prices[commodity], 365, 7)} />
                    </td>
                    <td class="has-text-right"
                      ><ValueChange value={change(prices[commodity], 365 * 3, 7)} /></td
                    >
                    <td class="has-text-right"
                      ><ValueChange value={change(prices[commodity], 365 * 5, 7)} /></td
                    >
                    <td>{p.commodity_type}</td>
                    <td>{p.commodity_id}</td>
                  </tr>
                  <tr slot="content">
                    <td colspan="12" class="p-0 has-background-white-ter">
                        {#key commodity}
                            <CommodityChart prices={priceArray} commodityName={p.commodity_name}/>
                        {/key}
                    </td>
                  </tr>
                </Toggleable>
              {/each}
            </tbody>
          </table>
        </div>
      </div>
    </div>
  </div>
</section>
