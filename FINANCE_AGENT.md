# Finance Agent — Reference & Skill Docs

`finance-agent.html` · Standalone React PWA · No build step required

---

## Overview

A self-contained conversational finance agent. Open it at `/finance-agent.html` or navigate to it from the Finance tab in the main app. No backend or API key needed — all logic runs in the browser.

**Architecture at a glance:**

```
DATA LAYER          →   AGENT ENGINE        →   UI
ACCOUNTS            →   INTENTS[]           →   AgentMessage
TRANSACTIONS        →   processQuery(text)  →   BudgetBar
BUDGET_CATS         →   FALLBACK            →   Metrics strip
AUTOMATIONS         →   QUICK_REPLIES       →   Chat + Input
CONFIG
```

---

## Configuration

At the top of the `<script>` block, update `CONFIG` to personalise the agent:

```js
const CONFIG = {
  name:            'Mack',      // shown in greeting + header
  period:          'May 2026',  // shown in responses
  rothContributed: 0,           // Roth IRA $ contributed so far this year
  rothLimit:       7000,        // IRS annual contribution limit
};
```

---

## Data Layer

Four plain arrays at the top of the file. Swap these out with `fetch()` calls or localStorage reads when you're ready to connect real data.

### `ACCOUNTS`
```js
{ id, name, type, balance }
```
| Field | Type | Notes |
|---|---|---|
| `id` | number | Unique identifier |
| `name` | string | Account display name |
| `type` | string | `"Savings"`, `"Checking"`, `"Credit Card"` |
| `balance` | number | Current balance in USD |

**Live balances (Era Context, last synced May 31):**
| Account | Type | Balance |
|---|---|---|
| BILLS ON BILLS | Savings | $2,514.31 |
| MACKREAMY FUNDS | Checking | $2,546.04 |
| SPEND WISELY | Credit Card | $0.00 |

---

### `TRANSACTIONS`
```js
{ id, date, desc, amount, cat, icon }
```
| Field | Type | Notes |
|---|---|---|
| `date` | string | `"YYYY-MM-DD"` |
| `amount` | number | Positive = income, negative = expense |
| `cat` | string | Category label used for filtering |
| `icon` | string | Emoji shown next to the transaction |

---

### `BUDGET_CATS`
```js
{ name, icon, budget, spent }
```
| Field | Type | Notes |
|---|---|---|
| `budget` | number | Monthly target spend |
| `spent` | number | Actual spend this month |

Budget bars turn **yellow** when > 85% used, **red** when over budget.

---

### `AUTOMATIONS`
```js
{ name, icon, freq, amt }
```
| Field | Type | Notes |
|---|---|---|
| `freq` | string | `"Weekly"`, `"Biweekly"`, `"Monthly"` |
| `amt` | number | Amount per occurrence |

---

## Agent Skills (Intents)

Each skill is a `{ patterns, handler }` object in the `INTENTS` array. The agent scans the user's message for any matching pattern (case-insensitive substring match) and calls that handler.

---

### Greeting
**Triggers:** `hi`, `hello`, `hey`, `sup`, `what's up`, `whats up`, `yo`

Returns a full financial snapshot: total balance, monthly income, monthly spend, net cash flow, and an alert if any budget categories are over.

---

### Account Balances
**Triggers:** `balance`, `account`, `accounts`, `how much do i have`, `net worth`, `total`

Lists all accounts with current balance and account type, plus a total footer.

---

### Spending Breakdown
**Triggers:** `spend`, `spent`, `spending`, `expenses`, `expense`, `outgoing`

Shows the top 5 spending categories by amount, sorted high to low, with total expenses in the footer.

---

### Income
**Triggers:** `income`, `earn`, `salary`, `paycheck`, `paid`, `deposit`, `how much did i make`

Lists all income transactions with amounts and dates.

---

### Budget Overview
**Triggers:** `budget`, `budgets`, `over budget`, `on track`, `on budget`, `how am i doing`, `categories`

Renders budget progress bars for all categories. Bars are color-coded: normal → accent, >85% → yellow, over → red.

---

### Investing
**Triggers:** `invest`, `investing`, `robinhood`, `stocks`, `portfolio`, `market`, `trading`

Shows total moved to investments this month vs. budget target, lists all Robinhood automations with frequencies, and shows the combined monthly auto total.

---

### Roth IRA
**Triggers:** `roth`, `ira`, `retirement`, `roth ira`

Shows contribution progress toward the annual limit, the remaining amount, and the weekly contribution needed to hit the limit by the April 15 deadline.

---

### Bills & Subscriptions
**Triggers:** `bill`, `bills`, `subscription`, `subscriptions`, `recurring`, `automatic`, `automation`

Lists all automations (investing + bills) with amounts and frequencies, with a monthly total footer.

---

### Recent Transactions
**Triggers:** `transaction`, `transactions`, `recent`, `history`, `last`, `latest`, `what did i buy`

Shows the 8 most recent transactions with dates and categories. Income items are highlighted green.

---

### Cash Flow
**Triggers:** `save`, `saving`, `savings`, `net`, `leftover`, `left over`, `cash flow`, `surplus`, `deficit`

Summarises total income, total expenses, net cash flow, and current account balance. Includes a positive/negative cash flow alert.

---

### Pet Expenses
**Triggers:** `pet`, `pets`, `dog`, `cat`, `animal`, `vet`

Breaks out all transactions in the `Pets` category, flags if you're over the pet budget, and shows a total.

---

### Tips & Advice
**Triggers:** `tip`, `advice`, `suggest`, `recommendation`, `what should i`, `help me`, `optimize`

Surfaces actionable callouts: biggest budget overage, current investing rate vs. 20% target, and any unallocated cash flow.

---

### 10-Year Projection
**Triggers:** `projection`, `project`, `forecast`, `future`, `10 year`, `growth`, `compound`

Projects investment growth at 8% average annual return for 1, 3, 5, and 10 years based on current automation amounts (~$725/mo). Assumes reinvested returns, no extra lump sums.

---

## Adding a New Skill

1. Open `finance-agent.html` and find the `INTENTS` array (around line 210).
2. Add a new object before the closing `];`:

```js
{
  patterns: ['keyword1', 'keyword2'],
  handler: () => ({
    text: 'Your response text here.',

    // Optional: key/value table rows
    table: [
      { label: '📋 Label', value: '$123.00', pos: true },  // pos: true=green, false=red
    ],

    // Optional: budget progress bars
    budgets: BUDGET_CATS,

    // Optional: highlighted callout boxes
    alerts: ['Something the user should know.'],

    // Optional: bold line at the bottom of the table
    footer: 'Total: $456.00',
  }),
},
```

3. Save and reload — no build step needed.

---

## Connecting a Real AI Backend

The `sendMessage` function in `App()` has a clear swap point:

```js
// Current: keyword matching with simulated delay
setTimeout(() => {
  const response = processQuery(text);
  setMessages(prev => [...prev, response]);
  setTyping(false);
}, delay);

// Replace with: real API call
fetch('https://your-backend/chat', {
  method: 'POST',
  body: JSON.stringify({ message: text, context: { ACCOUNTS, TRANSACTIONS, BUDGET_CATS } }),
})
  .then(r => r.json())
  .then(data => {
    setMessages(prev => [...prev, { role: 'agent', id: Date.now(), text: data.reply }]);
    setTyping(false);
  });
```

---

## Theming

Edit the CSS `:root` block at the top of `<style>`:

```css
:root {
  --bg:        #ffffff;   /* page background */
  --surface:   #f7f7f7;   /* cards, bubbles */
  --surface2:  #efefef;   /* progress bar track */
  --border:    #e0e0e0;   /* all borders */
  --text:      #111111;   /* primary text */
  --text2:     #444444;   /* secondary text */
  --text3:     #888888;   /* labels, metadata */
  --accent:    #111111;   /* user bubble + buttons */
  --accent-fg: #ffffff;   /* text on accent */
  --green:     #22a060;   /* income, positive */
  --red:       #d04040;   /* expenses, over-budget */
  --yellow:    #d08820;   /* near-limit warning */
  --radius:    12px;      /* border radius everywhere */
}
```

A dark theme preset is included in the file — just uncomment it.

---

## Quick Replies

The suggestion chips shown before the first message are defined in `QUICK_REPLIES`:

```js
const QUICK_REPLIES = [
  'What are my balances?',
  'How did I spend this month?',
  'Am I on budget?',
  'Show recent transactions',
  'How much did I invest?',
  'Show my bills',
  'Any tips?',
  '10-year projection',
];
```

Add, remove, or reorder freely — they map directly to intent triggers above.

---

## File Structure

```
finance-agent.html
├── <style>
│   ├── :root design tokens
│   ├── Shell layout (.app, .header, .metrics)
│   ├── Chat bubbles (.bubble, .bubble-agent, .bubble-user)
│   ├── Rich content (.data-table, .budget-row, .alert-card)
│   └── Input + quick replies
│
└── <script type="text/babel">
    ├── DATA LAYER      — ACCOUNTS, TRANSACTIONS, BUDGET_CATS, AUTOMATIONS, CONFIG
    ├── DERIVED METRICS — income, expenses, netCash, totalBalance, overBudget
    ├── FORMATTERS      — fmt$(), fmtDate(), sign()
    ├── INTENTS         — 12 skills (see above)
    ├── FALLBACK        — catch-all response
    ├── processQuery()  — intent matching engine
    ├── QUICK_REPLIES   — suggestion chips
    ├── BudgetBar       — progress bar component
    ├── AgentMessage    — chat bubble renderer
    └── App             — main shell + state
```
