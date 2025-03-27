<script lang="ts">
  import {
    intervalText,
    nextUnpaidSchedule,
    scheduleIcon,
    totalRecurring
  } from "$lib/transaction_sequence";
  import { formatCurrencyCrude, now, type TransactionSequence } from "$lib/utils";

  export let transactionSequece: TransactionSequence;

  let schedule = nextUnpaidSchedule(transactionSequece);
  let dueDate = schedule.scheduled;
  const icon = scheduleIcon(schedule);
  
  // Calculate days until payment is due
  const daysUntilDue = dueDate.diff(now(), 'days');
  
  // Calculate percentage for progress bar
  const progressPercentage = dueDate.isBefore(now()) 
    ? 100 
    : Math.max(0, Math.min(100, 100 - (daysUntilDue / transactionSequece.interval) * 100));
  
  // Determine amount category for color coding
  const amount = totalRecurring(transactionSequece);
  function getAmountCategory(amount: number): string {
    const absAmount = Math.abs(amount);
    if (absAmount >= 5000) return "high-amount";
    if (absAmount >= 1000) return "medium-amount";
    return "low-amount";
  }
  
  const amountCategory = getAmountCategory(amount);
  
  // Get progress bar color
  function getProgressColor(): string {
    if (amountCategory === "high-amount") return "#F87171";
    if (amountCategory === "medium-amount") return "#FBBF24";
    return "#34D399";
  }
  
  const progressColor = getProgressColor();
  
  // Get frequency class
  function getFrequencyClass(interval: number): string {
    if (interval <= 31) return "monthly";
    if (interval <= 120) return "quarterly";
    return "yearly";
  }
  
  const frequencyClass = getFrequencyClass(transactionSequece.interval);

  // Capitalaize interval text
  function capIntervalText(ts: TransactionSequence): string {
    const text = intervalText(transactionSequece)
    return text.charAt(0).toUpperCase() + text.slice(1);
  }
</script>

<div class="payment-card {amountCategory}">
  <span class="payment-frequency {frequencyClass}">{capIntervalText(transactionSequece)}</span>
  <div class="payment-title">{transactionSequece.key}</div>
  <div class="payment-amount">{formatCurrencyCrude(amount)}</div>
  <div class="progress-container">
    <div class="progress-bar">
      <div class="progress-fill" style="width: {progressPercentage}%; background-color: {progressColor};"></div>
    </div>
  </div>
  <div class="payment-info">
    <div>Due {dueDate.fromNow()}</div>
    <div>{dueDate.format("DD MMM")}</div>
  </div>
</div>

<style>
  .payment-card {
    background-color: var(--card-bg, #202025);
    border-radius: 10px;
    padding: 16px;
    transition: transform 0.15s ease, box-shadow 0.15s ease;
    position: relative;
    overflow: hidden;
    height: 100%;
    color: var(--text-color, #E4E4E7);
  }
  
  .payment-card:hover {
    transform: translateY(-3px);
    box-shadow: 0 6px 12px rgba(0, 0, 0, 0.15);
  }
  
  .payment-title {
    font-weight: 600;
    font-size: 16px;
    margin-bottom: 4px;
    white-space: nowrap;
    text-overflow: ellipsis;
    overflow: hidden;
  }
  
  .payment-amount {
    font-family: var(--mono-font, monospace);
    font-size: 20px;
    font-weight: 700;
    margin-bottom: 16px;
  }
  
  .high-amount .payment-amount {
    color: var(--high-amount-color, #F87171);
  }
  
  .medium-amount .payment-amount {
    color: var(--medium-amount-color, #FBBF24);
  }
  
  .low-amount .payment-amount {
    color: var(--low-amount-color, #34D399);
  }
  
  .progress-container {
    margin-bottom: 10px;
  }
  
  .progress-bar {
    height: 6px;
    width: 100%;
    background-color: var(--progress-bg, rgba(255, 255, 255, 0.1));
    border-radius: 3px;
    overflow: hidden;
    position: relative;
  }
  
  .progress-fill {
    height: 100%;
    border-radius: 3px;
    position: absolute;
    top: 0;
    left: 0;
  }
  
  .payment-info {
    display: flex;
    justify-content: space-between;
    font-size: 13px;
    color: var(--secondary-text, #ACACBE);
  }
  
  .payment-frequency {
    background-color: rgba(255, 255, 255, 0.1);
    border-radius: 4px;
    padding: 2px 6px;
    font-size: 11px;
    position: absolute;
    top: 16px;
    right: 16px;
  }
  
  .monthly {
    color: var(--monthly-color, #7DD3FC);
  }
  
  .yearly {
    color: var(--yearly-color, #A5B4FC);
  }
  
  .quarterly {
    color: var(--quarterly-color, #C4B5FD);
  }
  
  /* Add CSS variables for theme support */
  :global([data-theme="light"]) .payment-card {
    --card-bg: #fefefe;
    --text-color: #333333;
    --secondary-text: #666666;
    --progress-bg: rgba(0, 0, 0, 0.1);
    --high-amount-color: #10b981;
    --medium-amount-color: #f59e0b;
    --low-amount-color: #ef4444;
    --monthly-color: #0891b2;
    --yearly-color: #4f46e5;
    --quarterly-color: #8b5cf6;
    
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  }
</style>