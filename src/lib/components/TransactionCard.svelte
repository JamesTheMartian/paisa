<script lang="ts">
  import { accountColorStyle } from "$lib/colors";
  import COLORS from "$lib/colors";
  import { iconText } from "$lib/icon";
  import {
    formatCurrency,
    postingUrl,
    restName,
    type Posting,
    type Transaction,
    firstName
  } from "$lib/utils";
  import PostingStatus from "./PostingStatus.svelte";
  import TransactionNote from "./TransactionNote.svelte";

  export let t: Transaction;
  let posting: Posting;
  $: {
    posting = t.postings[0];
  }
  
  // Helper function to determine if a transaction is an expense
  function isExpense(account: string): boolean {
    return firstName(account) === "Expenses" || firstName(account) === "Liabilities";
  }

  // Helper function to get appropriate amount class
  function getAmountClass(account: string): string {
    return isExpense(account) ? "amount-expense" : "amount-income";
  }

  function getAmountColor(account: string) {
    const normalized = getAmountClass(account);
    let color = "hsl(0, 0%, 48%)";

    if (normalized === "amount-expense") {
      color = (COLORS as Record<string, string>)["expenses"];
    } else if (normalized === "amount-income") {
      color = (COLORS as Record<string, string>)["income"];
    }

    return `color: ${color};`;
  }
  
  // Function to get appropriate icon color
  function getCategoryBorder(account: string): string {
    return accountColorStyle(firstName(account)).replace('color:', 'border-color:');
  }

  function unclearStyle(posting: Posting): string {
    var border
    var bg
    if (posting.status === "pending" || posting.status === "uncleared") {
      border = "border: dashed;"
      bg = "background: none;"
    }
    return border + bg + ";"
  }
  function unclearIconStyle(posting: Posting): string {
    var color
    if (posting.status === "pending" || posting.status === "uncleared") {
      color = "color: #8A8A9B;"
    }
    return color + ";"
  }

  // Function to get appropriate icon background color
  function getCategoryBackground(account: string): string {
    return accountColorStyle(firstName(account)).replace('color:', 'background-color:');
  }
</script>

<style>
  .transaction-item {
    display: flex;
    align-items: center;
    background-color: #ffffff;
    border-radius: 8px;
    padding: 12px 16px;
    margin-bottom: 12px;
    transition: transform 0.15s ease, box-shadow 0.15s ease;
    box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
  }
  
  .transaction-item:hover {
    transform: translateY(-2px);
    box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
  }
  
  .category-icon {
    width: 40px;
    height: 40px;
    padding-top: 4px;
    border-radius: 8px;
    border: 1px solid;
    display: flex;
    align-items: center;
    justify-content: center;
    margin-right: 16px;
    flex-shrink: 0;
    color: white;
    font-size: 16px;
  }
  
  .transaction-details {
    flex-grow: 1;
    min-width: 0; /* Ensures proper truncation */
  }
  
  .transaction-entity {
    font-weight: 500;
    margin-bottom: 4px;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }
  
  .transaction-category {
    font-size: 13px;
    color: #666666;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }
  
  .transaction-date {
    font-size: 12px;
    color: #8A8A9B;
    margin-right: 16px;
    text-align: right;
    line-height: 1.5;
    white-space: nowrap;
  }
  
  .transaction-amount {
    font-family: 'Roboto Mono', monospace;
    font-size: 16px;
    font-weight: 500;
    min-width: 100px;
    text-align: right;
    white-space: nowrap;
  }
  
  .amount-expense {
    color: #FF6B6B;
  }
  
  .amount-income {
    color: #4CD97B;
  }
  
  .status-container {
    display: inline-flex;
    align-items: center;
    margin-right: 6px;
  }
  
  .truncate {
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }
</style>

<div class="transaction-item has-background-white">
  <div class="category-icon has-text-grey truncate custom-icon" style="{getCategoryBackground(posting.account)} {unclearStyle(posting)} {getCategoryBorder(posting.account)}" title={posting.account}>
    <span style="color:#000; {unclearIconStyle(posting)}"
      >{iconText(posting.account)}</span
    >
  </div>
  
  <div class="transaction-details">
    <div class="transaction-entity">
      <a href={postingUrl(posting)}>{posting.payee}</a>
    </div>
    <div class="transaction-category">
      {restName(posting.account)}
      <TransactionNote transaction={t} />
    </div>
  </div>
  
  <div class="transaction-date">
    {posting.date.format("DD MMM")}
  </div>
  
  <div class="transaction-amount" style={getAmountColor(posting.account)}>
    {formatCurrency(posting.amount)}
  </div>
</div>
