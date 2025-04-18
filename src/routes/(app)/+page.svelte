<script lang="ts">
  import * as cashFlow from "$lib/cash_flow";
  import COLORS from "$lib/colors";
  import LastNMonths from "$lib/components/LastNMonths.svelte";
  import TransactionCard from "$lib/components/TransactionCard.svelte";
  import * as expense from "$lib/expense/monthly";
  import { enrichTrantionSequence, sortTrantionSequence } from "$lib/transaction_sequence";
  import {
    ajax,
    formatCurrency,
    formatFloat,
    type Budget,
    type CashFlow,
    type Networth,
    type Posting,
    type Transaction,
    type TransactionSequence,
    type Legend,
    now,
    type GoalSummary,
    type AssetBreakdown
  } from "$lib/utils";
  import _ from "lodash";
  import { onMount } from "svelte";

  import BudgetCard from "$lib/components/BudgetCard.svelte";
  import LevelItem from "$lib/components/LevelItem.svelte";
  import ZeroState from "$lib/components/ZeroState.svelte";
  import { MasonryGrid } from "@egjs/svelte-grid";
  import { refresh } from "../../store";
  import UpcomingCard from "$lib/components/UpcomingCard.svelte";
  import GoalSummaryCard from "$lib/components/GoalSummaryCard.svelte";
  import LegendCard from "$lib/components/LegendCard.svelte";
  import BalanceCard from "$lib/components/BalanceCard.svelte";

  let UntypedMasonryGrid = MasonryGrid as any;

  let cashflowLegends: Legend[] = [];
  let month = now().format("YYYY-MM");
  let goalSummaries: GoalSummary[] = [];
  let transactionSequences: TransactionSequence[] = [];
  let cashFlows: CashFlow[] = [];
  let expenses: { [key: string]: Posting[] } = {};
  let xirr = 0;
  let networth: Networth;
  let renderer: (data: Posting[]) => void;
  let totalExpense = 0;
  let transactions: Transaction[] = [];
  let budgetsByMonth: Record<string, Budget> = {};
  let currentBudget: Budget;
  let selectedExpenses: Posting[] = [];
  let isEmpty = false;
  let checkingBalances: Record<string, AssetBreakdown> = {};

  $: if (renderer) {
    selectedExpenses = expenses[month] || [];
    renderer(selectedExpenses);
    totalExpense = _.sumBy(selectedExpenses, (p) => p.amount);
  }

  async function initDemo() {
    await ajax("/api/init", { method: "POST" });
    refresh();
  }

  onMount(async () => {
    ({
      expenses,
      cashFlows,
      goalSummaries,
      budget: { budgetsByMonth },
      transactionSequences,
      networth: { networth, xirr },
      checkingBalances: { asset_breakdowns: checkingBalances },
      transactions
    } = await ajax("/api/dashboard"));

    goalSummaries = _.sortBy(goalSummaries, (g) => -g.priority);

    if (_.isEmpty(transactions)) {
      isEmpty = true;
    } else {
      isEmpty = false;
    }

    const postings = _.chain(expenses).values().flatten().value();
    const z = expense.colorScale(postings);
    renderer = expense.renderCurrentExpensesBreakdown(z);
    currentBudget = budgetsByMonth[month];

    const { renderer: cashflowRenderer, legends } = cashFlow.renderMonthlyFlow(
      "#d3-current-cash-flow",
      {
        rotate: false,
        balance: _.last(cashFlows)?.balance || 0
      }
    );
    cashflowRenderer(cashFlows);
    cashflowLegends = legends;
    transactionSequences = _.take(
      sortTrantionSequence(enrichTrantionSequence(transactionSequences)),
      16
    );
  });

  // Add filter state
  let currentFilter = "all"; // "all", "income", "expense"

  const incomeAccounts = ["Income", "Assets"];

  function filterTransactions(transactions: Transaction[]) {
    if (currentFilter === "all") return transactions;
    if (currentFilter === "income") {
      return transactions.filter(t => {
        // Check if any posting account starts with "Income"
        return t.postings.some(posting => posting.account.startsWith("Income"));
      });
    }
    if (currentFilter === "expense") {
      // Transactions that don't have any Income accounts are expenses
      return transactions.filter(t => {
        return t.postings.some(posting => posting.account.startsWith("Expenses"));
      });
    }
    return transactions;
  }
</script>

<section class="section" class:is-hidden={!isEmpty}>
  <div class="container is-fluid">
    <div class="columns">
      <div class="column is-12">
        <ZeroState item={!isEmpty}>
          <div class="has-text-left" style="max-width: 640px;">
            <p class="mb-2">
              Looks like you are new here, you can either get started or look at a demo setup
            </p>
            <div>
              <p class="is-size-4">I want to get started</p>
              <ol class="ml-5 mt-2 mb-4">
                <li>
                  Go to <a href="/more/config">configuration</a> page and set your default currency and
                  locale.
                </li>
                <li>
                  Go to <a href="/ledger/editor">editor</a> page and start adding transactions to your
                  journal.
                </li>
              </ol>
              <p class="is-size-4">I want to view a Demo</p>
              <p class="ml-3"></p>
              <ol class="ml-5 mt-2 mb-4">
                <li>
                  Click the button below to load a demo setup. This will load a demo journal with
                  relevant config.
                </li>
                <li>
                  Once you are done playing around, you can go to <a href="/ledger/editor">editor</a
                  > page and select all the content and delete them.
                </li>
                <li>
                  Go to <a href="/more/config">configuration</a> page and click the reset to defaults
                  button.
                </li>
              </ol>

              <a on:click={(_e) => initDemo()} class="button is-link">Setup Demo</a>
            </div>
          </div>
        </ZeroState>
      </div>
    </div>
  </div>
</section>

<section class="section tab-networth" class:is-hidden={isEmpty}>
  <div class="container is-fluid">
    <div class="tile is-ancestor is-align-items-start">
      <div class="tile is-4 is-vertical">
        <div class="tile is-parent">
          <div class="tile is-child">
            <div class="content">
              <p class="subtitle">
                <a class="secondary-link has-text-grey" href="/assets/networth">Assets</a>
              </p>
              <div class="content">
                <div>
                  {#if networth}
                    <nav class="level grid-2">
                      <LevelItem
                        narrow
                        title="Net worth"
                        color={COLORS.primary}
                        value={formatCurrency(networth.balanceAmount)}
                      />

                      <LevelItem
                        narrow
                        title="Net Investment"
                        color={COLORS.secondary}
                        value={formatCurrency(networth.netInvestmentAmount)}
                      />
                    </nav>
                    <nav class="level grid-2">
                      <LevelItem
                        narrow
                        title="Gain / Loss"
                        color={networth.gainAmount >= 0 ? COLORS.gainText : COLORS.lossText}
                        value={formatCurrency(networth.gainAmount)}
                      />

                      <LevelItem narrow title="XIRR" value={formatFloat(xirr)} />
                    </nav>
                  {/if}
                </div>
              </div>
            </div>
          </div>
        </div>

        {#if !_.isEmpty(checkingBalances)}
          <div class="tile is-parent">
            <article class="tile is-child">
              <div class="content">
                <p class="subtitle">
                  <a class="secondary-link has-text-grey" href="/assets/balance">Checking Balance</a
                  >
                </p>
                <div class="content">
                  <UntypedMasonryGrid gap={10} maxStretchColumnSize={400} align="stretch">
                    {#each _.values(checkingBalances) as assetBreakdown}
                      <div class="is-flex-grow-1">
                        <BalanceCard {assetBreakdown} />
                      </div>
                    {/each}
                  </UntypedMasonryGrid>
                </div>
              </div>
            </article>
          </div>
        {/if}

        <div class="tile is-parent">
          <article class="tile is-child min-w-0">
            <p class="subtitle">
              <a class="secondary-link has-text-grey" href="/cash_flow/monthly">Cash Flow</a>
            </p>
            <div class="content box px-2 pb-0">
              <ZeroState item={cashFlows}>
                <strong>Oops!</strong> You have not made any transactions in the last 3 months.
              </ZeroState>

              <LegendCard legends={cashflowLegends} clazz="mb-2 overflow-x-auto" />

              <svg
                class:is-not-visible={_.isEmpty(cashFlows)}
                id="d3-current-cash-flow"
                height="250"
                width="100%"
              />
            </div>
          </article>
        </div>
        {#if currentBudget}
          <div class="tile is-parent">
            <div class="tile is-child">
              <div class="content">
                <p class="subtitle">
                  <a class="secondary-link has-text-grey" href="/expense/budget">Budget</a>
                </p>
                <div class="content">
                  <div>
                    {#each currentBudget.accounts as accountBudget (accountBudget)}
                      <BudgetCard compact {accountBudget} />
                    {/each}
                  </div>
                </div>
              </div>
            </div>
          </div>
        {/if}
        {#if !_.isEmpty(goalSummaries)}
          <div class="tile">
            <div class="tile is-parent is-12">
              <article class="tile is-child">
                <div class="content">
                  <p class="subtitle">
                    <a class="secondary-link has-text-grey" href="/more/goals">Goals</a>
                  </p>
                  <div class="content">
                    {#each goalSummaries as goal}
                      <GoalSummaryCard {goal} small />
                    {/each}
                  </div>
                </div>
              </article>
            </div>
          </div>
        {/if}
      </div>
      <div class="tile is-vertical">
        <div class="tile is-parent is-12">
          <article class="tile is-child">
            <p class="subtitle is-flex is-justify-content-space-between is-align-items-end">
              <span
                ><a class="secondary-link has-text-grey" href="/expense/monthly">Expenses</a>
                <span class="is-size-5 has-text-weight-bold px-2" style="color: {COLORS.expenses}"
                  >{formatCurrency(totalExpense)}</span
                ></span
              >
              <LastNMonths n={3} bind:value={month} />
            </p>
            <div class="content box px-3">
              <ZeroState item={selectedExpenses}>
                <strong>Hurray!</strong> You have no expenses this month.
              </ZeroState>
              <svg id="d3-current-month-breakdown" width="100%" />
            </div>
          </article>
        </div>
        {#if !_.isEmpty(transactionSequences)}
          <div class="tile">
            <div class="tile is-parent is-12">
              <article class="tile is-child">
                <div class="content">
                  <p class="subtitle">
                    <a class="secondary-link has-text-grey" href="/cash_flow/recurring">Recurring</a
                    >
                  </p>
                  <div class="content box">
                    <div
                      class="grid grid-rows-1 overflow-hidden recurring-grid"
                      style="grid-auto-rows: 0px; grid-template-columns: repeat(auto-fit, minmax(130px, 200px)); margin-left: 16px;"
                    >
                      {#each transactionSequences as ts (ts)}
                        <UpcomingCard transactionSequece={ts} />
                      {/each}
                    </div>
                  </div>
                </div>
              </article>
            </div>
          </div>
        {/if}
        {#if !_.isEmpty(transactions)}
          <div class="tile">
            <div class="tile is-parent is-12">
              <article class="tile is-child">
                <div class="content">
                  <div class="subtitle transaction-top">
                    <a class="secondary-link has-text-grey" href="/ledger/transaction"
                      >Recent Transactions</a
                    >
                    <div class="filter-container">
                      <button class="filter-btn {currentFilter === 'all' ? 'active' : ''}" 
                              on:click={() => currentFilter = 'all'}>All</button>
                      <button class="filter-btn {currentFilter === 'income' ? 'active' : ''}" 
                              on:click={() => currentFilter = 'income'}>Income</button>
                      <button class="filter-btn {currentFilter === 'expense' ? 'active' : ''}" 
                              on:click={() => currentFilter = 'expense'}>Expenses</button>
                    </div>
                  </div>
                  <div>
                    <UntypedMasonryGrid gap={10} maxStretchColumnSize={500} align="stretch">
                      {#each _.take(filterTransactions(transactions), 18) as transaction}
                      {@const t = (currentFilter === 'all' ? transaction : transaction)}
                        <div class="mr-3 is-flex-grow-1">
                          <TransactionCard {t} />
                        </div>
                      {/each}
                    </UntypedMasonryGrid>
                  </div>
                </div>
              </article>
            </div>
          </div>
        {/if}
      </div>
    </div>
  </div>
</section>

<style lang="scss">
  .subtitle {
    margin-bottom: 0.5rem !important;
  }

  .subtitle a.secondary-link {
    text-transform: uppercase;
    font-size: 1rem;
  }

  .recurring-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
    gap: 16px;
  }

  .content {
    border-radius: 10px;
  }
  
  .transaction-top {
    display: flex;
    justify-content: space-between;
    align-items: center;
  }

  /* Filter buttons styling */
  .filter-container {
    display: flex;
    gap: 8px;
  }

  .filter-btn {
    border: none;
    padding: 6px 12px;
    border-radius: 6px;
    font-size: 13px;
    cursor: pointer;
    transition: all 0.2s ease;
    background-color: #f0f0f0;
    color: #666;
  }

  html[data-theme=dark] * .filter-btn:not(.active) {
    background-color: hsl(215, 18%, 10%);
    color: #ccc;
  }

  .filter-btn.active {
    background-color: #006064;
    color: white;
  }

  .filter-btn:hover:not(.active) {
    background-color: #e0e0e0;
  }

  html[data-theme=dark] * .filter-btn:hover:not(.active) {
    background-color: hsl(215, 18%, 15%);
  }
</style>
