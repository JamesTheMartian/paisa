<script lang="ts">
  import { accountColorStyle } from "$lib/colors";
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
  function isExpense(amount: number): boolean {
    return amount < 0;
  }
  
  // Helper function to get appropriate amount class
  function getAmountClass(amount: number): string {
    return isExpense(amount) ? "amount-expense" : "amount-income";
  }
  
  // Function to get appropriate icon background color
  function getCategoryBorder(account: string): string {
    // You can map account types to specific colors or use your existing color system
    return accountColorStyle(firstName(account)).replace('color:', 'border-color:');
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
    border-radius: 8px;
    border: solid;
    border-width: 1px;
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
  
  /* Filter buttons styling */
  .filter-container {
    display: flex;
    gap: 8px;
    margin-bottom: 16px;
  }
  
  .filter-btn {
    background-color: #f0f0f0;
    border: none;
    color: #666;
    padding: 6px 12px;
    border-radius: 6px;
    font-size: 13px;
    cursor: pointer;
    transition: all 0.2s ease;
  }
  
  .filter-btn.active {
    background-color: #006064;
    color: white;
  }
  
  .filter-btn:hover:not(.active) {
    background-color: #e0e0e0;
  }
</style>

<div class="transaction-item has-background-white">
  <div class="category-icon has-text-grey truncate custom-icon" style={getCategoryBorder(posting.account)} title={posting.account}>
    <span style={accountColorStyle(firstName(posting.account))}
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
    <div class="status-container">
      <PostingStatus {posting} />
    </div>
    {posting.date.format("DD MMM YYYY")}
  </div>
  
  <div class="transaction-amount {getAmountClass(posting.amount)}">
    {formatCurrency(posting.amount)}
  </div>
</div>