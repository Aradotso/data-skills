---
name: apexbank-analytics-dashboard
description: Build and customize a web-based digital banking dashboard with HTML, CSS, and JavaScript for financial tracking and analytics
triggers:
  - how do I customize the ApexBank dashboard
  - set up a digital banking analytics interface
  - modify ApexBank account balances and transactions
  - create financial dashboard with ApexBank
  - integrate custom data into ApexBank
  - style and theme the banking dashboard
  - add new features to ApexBank dashboard
  - configure mock financial data in ApexBank
---

# ApexBank Analytics Dashboard Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

ApexBank Analytics Hub is a client-side web application that provides a complete digital banking dashboard interface. Built entirely with HTML, CSS, and JavaScript, it offers account monitoring, transaction analysis, investment tracking, savings goals, and financial visualization without requiring a backend server. The dashboard uses glassmorphism design and runs completely in the browser.

## Installation

Clone the repository and serve locally:

```bash
git clone https://github.com/fhuber82/apexbank-analytics-hub.git
cd apexbank-analytics-hub
```

Serve using Python's built-in HTTP server:

```bash
python -m http.server 8000
```

Or use Node.js with `http-server`:

```bash
npx http-server -p 8000
```

Access at `http://localhost:8000`

For production deployment, upload files to any static hosting service (GitHub Pages, Netlify, Vercel, etc.).

## Project Structure

```
apexbank-analytics-hub/
├── index.html              # Main dashboard entry point
├── css/
│   ├── styles.css         # Core styling and theme
│   └── glassmorphism.css  # Glass effect styling
├── js/
│   ├── app.js             # Main application logic
│   ├── charts.js          # Chart rendering (Chart.js integration)
│   ├── accounts.js        # Account management
│   ├── transactions.js    # Transaction handling
│   ├── calculator.js      # Mortgage/loan calculators
│   └── data.js            # Mock data definitions
└── assets/
    └── images/            # UI assets and icons
```

## Configuration

### Customizing Mock Data

Edit `js/data.js` to modify default accounts, balances, and transactions:

```javascript
// js/data.js
const mockData = {
  accounts: [
    {
      id: 'acc_001',
      name: 'Primary Checking',
      type: 'checking',
      balance: 5420.50,
      currency: 'USD',
      accountNumber: '****1234'
    },
    {
      id: 'acc_002',
      name: 'Savings Account',
      type: 'savings',
      balance: 12350.00,
      currency: 'USD',
      accountNumber: '****5678'
    }
  ],
  transactions: [
    {
      id: 'txn_001',
      accountId: 'acc_001',
      date: '2026-08-03',
      description: 'Grocery Store',
      amount: -85.42,
      category: 'groceries',
      type: 'debit'
    },
    {
      id: 'txn_002',
      accountId: 'acc_001',
      date: '2026-08-02',
      description: 'Salary Deposit',
      amount: 3500.00,
      category: 'income',
      type: 'credit'
    }
  ],
  investments: [
    {
      id: 'inv_001',
      name: 'Tech Growth Fund',
      symbol: 'TECHGR',
      shares: 50,
      purchasePrice: 125.00,
      currentPrice: 142.30,
      totalValue: 7115.00
    }
  ],
  savingsGoals: [
    {
      id: 'goal_001',
      name: 'Emergency Fund',
      target: 10000,
      current: 6500,
      deadline: '2026-12-31'
    }
  ]
};
```

### Theming and Styling

Modify CSS variables in `css/styles.css`:

```css
:root {
  /* Primary Colors */
  --primary-color: #6366f1;
  --secondary-color: #8b5cf6;
  --accent-color: #ec4899;
  
  /* Background */
  --bg-primary: #0f172a;
  --bg-secondary: #1e293b;
  --bg-card: rgba(30, 41, 59, 0.7);
  
  /* Glass Effect */
  --glass-bg: rgba(255, 255, 255, 0.05);
  --glass-border: rgba(255, 255, 255, 0.1);
  --glass-blur: 10px;
  
  /* Text */
  --text-primary: #f8fafc;
  --text-secondary: #cbd5e1;
  --text-muted: #64748b;
  
  /* Status Colors */
  --success: #10b981;
  --warning: #f59e0b;
  --error: #ef4444;
}
```

## Core Functionality

### Account Management

Add new account functionality in `js/accounts.js`:

```javascript
// js/accounts.js
class AccountManager {
  constructor() {
    this.accounts = this.loadAccounts();
  }
  
  loadAccounts() {
    const stored = localStorage.getItem('apexbank_accounts');
    return stored ? JSON.parse(stored) : mockData.accounts;
  }
  
  saveAccounts() {
    localStorage.setItem('apexbank_accounts', JSON.stringify(this.accounts));
  }
  
  addAccount(accountData) {
    const newAccount = {
      id: `acc_${Date.now()}`,
      ...accountData,
      createdAt: new Date().toISOString()
    };
    this.accounts.push(newAccount);
    this.saveAccounts();
    return newAccount;
  }
  
  getAccountById(id) {
    return this.accounts.find(acc => acc.id === id);
  }
  
  updateBalance(accountId, amount) {
    const account = this.getAccountById(accountId);
    if (account) {
      account.balance += amount;
      this.saveAccounts();
      this.renderAccounts();
    }
  }
  
  getTotalBalance() {
    return this.accounts.reduce((sum, acc) => sum + acc.balance, 0);
  }
  
  renderAccounts() {
    const container = document.getElementById('accounts-container');
    container.innerHTML = this.accounts.map(account => `
      <div class="account-card glass-card" data-account-id="${account.id}">
        <div class="account-header">
          <h3>${account.name}</h3>
          <span class="account-type">${account.type}</span>
        </div>
        <div class="account-balance">
          <span class="currency">${account.currency}</span>
          <span class="amount">${account.balance.toFixed(2)}</span>
        </div>
        <div class="account-number">${account.accountNumber}</div>
      </div>
    `).join('');
  }
}

const accountManager = new AccountManager();
```

### Transaction Processing

Handle transactions in `js/transactions.js`:

```javascript
// js/transactions.js
class TransactionManager {
  constructor(accountManager) {
    this.accountManager = accountManager;
    this.transactions = this.loadTransactions();
  }
  
  loadTransactions() {
    const stored = localStorage.getItem('apexbank_transactions');
    return stored ? JSON.parse(stored) : mockData.transactions;
  }
  
  saveTransactions() {
    localStorage.setItem('apexbank_transactions', JSON.stringify(this.transactions));
  }
  
  addTransaction(transactionData) {
    const transaction = {
      id: `txn_${Date.now()}`,
      date: new Date().toISOString().split('T')[0],
      ...transactionData
    };
    
    this.transactions.unshift(transaction);
    this.accountManager.updateBalance(transaction.accountId, transaction.amount);
    this.saveTransactions();
    
    return transaction;
  }
  
  getTransactionsByAccount(accountId) {
    return this.transactions.filter(txn => txn.accountId === accountId);
  }
  
  getTransactionsByDateRange(startDate, endDate) {
    return this.transactions.filter(txn => {
      const txnDate = new Date(txn.date);
      return txnDate >= new Date(startDate) && txnDate <= new Date(endDate);
    });
  }
  
  getCategoryTotals(type = 'debit') {
    const filtered = this.transactions.filter(txn => txn.type === type);
    const totals = {};
    
    filtered.forEach(txn => {
      if (!totals[txn.category]) {
        totals[txn.category] = 0;
      }
      totals[txn.category] += Math.abs(txn.amount);
    });
    
    return totals;
  }
  
  renderTransactions(filter = {}) {
    let filtered = [...this.transactions];
    
    if (filter.accountId) {
      filtered = filtered.filter(txn => txn.accountId === filter.accountId);
    }
    if (filter.category) {
      filtered = filtered.filter(txn => txn.category === filter.category);
    }
    
    const container = document.getElementById('transactions-list');
    container.innerHTML = filtered.map(txn => `
      <div class="transaction-item ${txn.type}">
        <div class="transaction-icon">
          <i class="icon-${txn.category}"></i>
        </div>
        <div class="transaction-details">
          <div class="transaction-description">${txn.description}</div>
          <div class="transaction-date">${txn.date}</div>
        </div>
        <div class="transaction-amount ${txn.amount > 0 ? 'positive' : 'negative'}">
          ${txn.amount > 0 ? '+' : ''}${txn.amount.toFixed(2)}
        </div>
      </div>
    `).join('');
  }
}

const transactionManager = new TransactionManager(accountManager);
```

### Chart Visualization

Integrate Chart.js for analytics in `js/charts.js`:

```javascript
// js/charts.js
class ChartManager {
  constructor(transactionManager) {
    this.transactionManager = transactionManager;
    this.charts = {};
  }
  
  renderSpendingChart() {
    const ctx = document.getElementById('spending-chart').getContext('2d');
    const categoryTotals = this.transactionManager.getCategoryTotals('debit');
    
    if (this.charts.spending) {
      this.charts.spending.destroy();
    }
    
    this.charts.spending = new Chart(ctx, {
      type: 'doughnut',
      data: {
        labels: Object.keys(categoryTotals),
        datasets: [{
          data: Object.values(categoryTotals),
          backgroundColor: [
            '#6366f1',
            '#8b5cf6',
            '#ec4899',
            '#f59e0b',
            '#10b981',
            '#3b82f6'
          ],
          borderWidth: 0
        }]
      },
      options: {
        responsive: true,
        maintainAspectRatio: false,
        plugins: {
          legend: {
            position: 'bottom',
            labels: {
              color: '#cbd5e1',
              font: { size: 12 }
            }
          }
        }
      }
    });
  }
  
  renderCashFlowChart(months = 6) {
    const ctx = document.getElementById('cashflow-chart').getContext('2d');
    const monthlyData = this.getMonthlyData(months);
    
    if (this.charts.cashflow) {
      this.charts.cashflow.destroy();
    }
    
    this.charts.cashflow = new Chart(ctx, {
      type: 'line',
      data: {
        labels: monthlyData.labels,
        datasets: [
          {
            label: 'Income',
            data: monthlyData.income,
            borderColor: '#10b981',
            backgroundColor: 'rgba(16, 185, 129, 0.1)',
            tension: 0.4
          },
          {
            label: 'Expenses',
            data: monthlyData.expenses,
            borderColor: '#ef4444',
            backgroundColor: 'rgba(239, 68, 68, 0.1)',
            tension: 0.4
          }
        ]
      },
      options: {
        responsive: true,
        maintainAspectRatio: false,
        scales: {
          y: {
            beginAtZero: true,
            ticks: { color: '#cbd5e1' },
            grid: { color: 'rgba(255, 255, 255, 0.1)' }
          },
          x: {
            ticks: { color: '#cbd5e1' },
            grid: { color: 'rgba(255, 255, 255, 0.1)' }
          }
        },
        plugins: {
          legend: {
            labels: { color: '#cbd5e1' }
          }
        }
      }
    });
  }
  
  getMonthlyData(months) {
    const data = { labels: [], income: [], expenses: [] };
    const now = new Date();
    
    for (let i = months - 1; i >= 0; i--) {
      const month = new Date(now.getFullYear(), now.getMonth() - i, 1);
      const monthKey = month.toLocaleDateString('en-US', { month: 'short' });
      
      const monthStart = new Date(month.getFullYear(), month.getMonth(), 1);
      const monthEnd = new Date(month.getFullYear(), month.getMonth() + 1, 0);
      
      const transactions = this.transactionManager.getTransactionsByDateRange(
        monthStart.toISOString().split('T')[0],
        monthEnd.toISOString().split('T')[0]
      );
      
      const income = transactions
        .filter(t => t.amount > 0)
        .reduce((sum, t) => sum + t.amount, 0);
      
      const expenses = Math.abs(transactions
        .filter(t => t.amount < 0)
        .reduce((sum, t) => sum + t.amount, 0));
      
      data.labels.push(monthKey);
      data.income.push(income);
      data.expenses.push(expenses);
    }
    
    return data;
  }
}

const chartManager = new ChartManager(transactionManager);
```

### Savings Goals Tracker

Implement savings goals in `js/app.js`:

```javascript
// js/app.js
class SavingsGoalManager {
  constructor() {
    this.goals = this.loadGoals();
  }
  
  loadGoals() {
    const stored = localStorage.getItem('apexbank_goals');
    return stored ? JSON.parse(stored) : mockData.savingsGoals;
  }
  
  saveGoals() {
    localStorage.setItem('apexbank_goals', JSON.stringify(this.goals));
  }
  
  addGoal(goalData) {
    const goal = {
      id: `goal_${Date.now()}`,
      current: 0,
      ...goalData,
      createdAt: new Date().toISOString()
    };
    this.goals.push(goal);
    this.saveGoals();
    return goal;
  }
  
  updateProgress(goalId, amount) {
    const goal = this.goals.find(g => g.id === goalId);
    if (goal) {
      goal.current = Math.min(goal.current + amount, goal.target);
      this.saveGoals();
      this.renderGoals();
    }
  }
  
  getProgress(goal) {
    return (goal.current / goal.target) * 100;
  }
  
  renderGoals() {
    const container = document.getElementById('savings-goals');
    container.innerHTML = this.goals.map(goal => {
      const progress = this.getProgress(goal);
      const remaining = goal.target - goal.current;
      
      return `
        <div class="goal-card glass-card">
          <div class="goal-header">
            <h4>${goal.name}</h4>
            <span class="goal-deadline">${goal.deadline}</span>
          </div>
          <div class="goal-progress">
            <div class="progress-bar">
              <div class="progress-fill" style="width: ${progress}%"></div>
            </div>
            <div class="progress-text">${progress.toFixed(1)}%</div>
          </div>
          <div class="goal-amounts">
            <span class="current">$${goal.current.toFixed(2)}</span>
            <span class="target">/ $${goal.target.toFixed(2)}</span>
          </div>
          <div class="goal-remaining">$${remaining.toFixed(2)} remaining</div>
        </div>
      `;
    }).join('');
  }
}

const savingsGoalManager = new SavingsGoalManager();
```

### Fund Transfer System

Implement transfers between accounts:

```javascript
// js/app.js
class TransferManager {
  constructor(accountManager, transactionManager) {
    this.accountManager = accountManager;
    this.transactionManager = transactionManager;
  }
  
  transfer(fromAccountId, toAccountId, amount, description = 'Transfer') {
    const fromAccount = this.accountManager.getAccountById(fromAccountId);
    const toAccount = this.accountManager.getAccountById(toAccountId);
    
    if (!fromAccount || !toAccount) {
      throw new Error('Invalid account ID');
    }
    
    if (fromAccount.balance < amount) {
      throw new Error('Insufficient funds');
    }
    
    // Debit from source
    this.transactionManager.addTransaction({
      accountId: fromAccountId,
      description: `${description} to ${toAccount.name}`,
      amount: -amount,
      category: 'transfer',
      type: 'debit'
    });
    
    // Credit to destination
    this.transactionManager.addTransaction({
      accountId: toAccountId,
      description: `${description} from ${fromAccount.name}`,
      amount: amount,
      category: 'transfer',
      type: 'credit'
    });
    
    return {
      success: true,
      fromAccount: fromAccount.name,
      toAccount: toAccount.name,
      amount
    };
  }
  
  setupTransferForm() {
    const form = document.getElementById('transfer-form');
    
    form.addEventListener('submit', (e) => {
      e.preventDefault();
      
      const formData = new FormData(form);
      const fromAccountId = formData.get('from-account');
      const toAccountId = formData.get('to-account');
      const amount = parseFloat(formData.get('amount'));
      const description = formData.get('description') || 'Transfer';
      
      try {
        const result = this.transfer(fromAccountId, toAccountId, amount, description);
        this.showNotification('Transfer successful', 'success');
        form.reset();
      } catch (error) {
        this.showNotification(error.message, 'error');
      }
    });
  }
  
  showNotification(message, type) {
    const notification = document.createElement('div');
    notification.className = `notification notification-${type}`;
    notification.textContent = message;
    document.body.appendChild(notification);
    
    setTimeout(() => {
      notification.remove();
    }, 3000);
  }
}

const transferManager = new TransferManager(accountManager, transactionManager);
```

### Mortgage Calculator

Add financial calculator in `js/calculator.js`:

```javascript
// js/calculator.js
class MortgageCalculator {
  calculate(principal, annualRate, years) {
    const monthlyRate = annualRate / 100 / 12;
    const numPayments = years * 12;
    
    const monthlyPayment = principal * 
      (monthlyRate * Math.pow(1 + monthlyRate, numPayments)) / 
      (Math.pow(1 + monthlyRate, numPayments) - 1);
    
    const totalPayment = monthlyPayment * numPayments;
    const totalInterest = totalPayment - principal;
    
    return {
      monthlyPayment: monthlyPayment.toFixed(2),
      totalPayment: totalPayment.toFixed(2),
      totalInterest: totalInterest.toFixed(2),
      principal: principal.toFixed(2)
    };
  }
  
  generateAmortizationSchedule(principal, annualRate, years) {
    const monthlyRate = annualRate / 100 / 12;
    const numPayments = years * 12;
    const monthlyPayment = this.calculate(principal, annualRate, years).monthlyPayment;
    
    const schedule = [];
    let remainingBalance = principal;
    
    for (let month = 1; month <= numPayments; month++) {
      const interestPayment = remainingBalance * monthlyRate;
      const principalPayment = parseFloat(monthlyPayment) - interestPayment;
      remainingBalance -= principalPayment;
      
      schedule.push({
        month,
        payment: parseFloat(monthlyPayment),
        principal: principalPayment,
        interest: interestPayment,
        balance: Math.max(0, remainingBalance)
      });
    }
    
    return schedule;
  }
  
  setupCalculatorForm() {
    const form = document.getElementById('mortgage-calculator');
    const resultsDiv = document.getElementById('calculator-results');
    
    form.addEventListener('submit', (e) => {
      e.preventDefault();
      
      const formData = new FormData(form);
      const principal = parseFloat(formData.get('principal'));
      const rate = parseFloat(formData.get('rate'));
      const years = parseFloat(formData.get('years'));
      
      const results = this.calculate(principal, rate, years);
      
      resultsDiv.innerHTML = `
        <div class="calculator-results glass-card">
          <h3>Loan Summary</h3>
          <div class="result-row">
            <span>Monthly Payment:</span>
            <span class="amount">$${results.monthlyPayment}</span>
          </div>
          <div class="result-row">
            <span>Total Payment:</span>
            <span class="amount">$${results.totalPayment}</span>
          </div>
          <div class="result-row">
            <span>Total Interest:</span>
            <span class="amount">$${results.totalInterest}</span>
          </div>
          <div class="result-row">
            <span>Principal:</span>
            <span class="amount">$${results.principal}</span>
          </div>
        </div>
      `;
    });
  }
}

const mortgageCalculator = new MortgageCalculator();
```

## Application Initialization

Main app initialization in `js/app.js`:

```javascript
// js/app.js
document.addEventListener('DOMContentLoaded', () => {
  // Initialize managers
  accountManager.renderAccounts();
  transactionManager.renderTransactions();
  savingsGoalManager.renderGoals();
  
  // Render charts
  chartManager.renderSpendingChart();
  chartManager.renderCashFlowChart(6);
  
  // Setup forms
  transferManager.setupTransferForm();
  mortgageCalculator.setupCalculatorForm();
  
  // Update dashboard summary
  updateDashboardSummary();
  
  // Setup event listeners
  setupFilterListeners();
  setupThemeToggle();
});

function updateDashboardSummary() {
  const totalBalance = accountManager.getTotalBalance();
  const monthTransactions = transactionManager.getTransactionsByDateRange(
    new Date(new Date().getFullYear(), new Date().getMonth(), 1).toISOString().split('T')[0],
    new Date().toISOString().split('T')[0]
  );
  
  const monthIncome = monthTransactions
    .filter(t => t.amount > 0)
    .reduce((sum, t) => sum + t.amount, 0);
  
  const monthExpenses = Math.abs(monthTransactions
    .filter(t => t.amount < 0)
    .reduce((sum, t) => sum + t.amount, 0));
  
  document.getElementById('total-balance').textContent = `$${totalBalance.toFixed(2)}`;
  document.getElementById('month-income').textContent = `$${monthIncome.toFixed(2)}`;
  document.getElementById('month-expenses').textContent = `$${monthExpenses.toFixed(2)}`;
  document.getElementById('net-savings').textContent = `$${(monthIncome - monthExpenses).toFixed(2)}`;
}

function setupFilterListeners() {
  document.getElementById('account-filter')?.addEventListener('change', (e) => {
    transactionManager.renderTransactions({ accountId: e.target.value });
  });
  
  document.getElementById('category-filter')?.addEventListener('change', (e) => {
    transactionManager.renderTransactions({ category: e.target.value });
  });
}

function setupThemeToggle() {
  const themeToggle = document.getElementById('theme-toggle');
  const currentTheme = localStorage.getItem('apexbank_theme') || 'dark';
  
  document.documentElement.setAttribute('data-theme', currentTheme);
  
  themeToggle?.addEventListener('click', () => {
    const newTheme = document.documentElement.getAttribute('data-theme') === 'dark' ? 'light' : 'dark';
    document.documentElement.setAttribute('data-theme', newTheme);
    localStorage.setItem('apexbank_theme', newTheme);
  });
}
```

## Common Patterns

### Adding Custom Dashboard Widgets

```javascript
// Create custom widget
class CustomWidget {
  constructor(containerId) {
    this.container = document.getElementById(containerId);
  }
  
  render(data) {
    this.container.innerHTML = `
      <div class="custom-widget glass-card">
        <h3>${data.title}</h3>
        <div class="widget-content">
          ${data.content}
        </div>
      </div>
    `;
  }
}

// Use widget
const widget = new CustomWidget('widget-container');
widget.render({
  title: 'Credit Score',
  content: '<div class="score">750</div>'
});
```

### Exporting Data to CSV

```javascript
function exportTransactionsToCSV() {
  const transactions = transactionManager.transactions;
  const headers = ['Date', 'Description', 'Category', 'Amount', 'Type'];
  
  const csvContent = [
    headers.join(','),
    ...transactions.map(txn => 
      [txn.date, txn.description, txn.category, txn.amount, txn.type].join(',')
    )
  ].join('\n');
  
  const blob = new Blob([csvContent], { type: 'text/csv' });
  const url = URL.createObjectURL(blob);
  const link = document.createElement('a');
  link.href = url;
  link.download = `transactions_${new Date().toISOString().split('T')[0]}.csv`;
  link.click();
}
```

### Implementing Search Functionality

```javascript
function setupTransactionSearch() {
  const searchInput = document.getElementById('transaction-search');
  
  searchInput.addEventListener('input', (e) => {
    const query = e.target.value.toLowerCase();
    const filtered = transactionManager.transactions.filter(txn =>
      txn.description.toLowerCase().includes(query) ||
      txn.category.toLowerCase().includes(query)
    );
    
    renderFilteredTransactions(filtered);
  });
}

function renderFilteredTransactions(transactions) {
  const container = document.getElementById('transactions-list');
  container.innerHTML = transactions.map(txn => `
    <div class="transaction-item ${txn.type}">
      <div class="transaction-details">
        <div class="transaction-description">${txn.description}</div>
        <div class="transaction-date">${txn.date}</div>
      </div>
      <div class="transaction-amount">${txn.amount.toFixed(2)}</div>
    </div>
  `).join('');
}
```

## Troubleshooting

### Charts Not Rendering

Ensure Chart.js is loaded before chart initialization:

```html
<!-- Add to index.html before your scripts -->
<script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.0/dist/chart.umd.min.js"></script>
```

### Data Not Persisting

Check localStorage availability:

```javascript
function checkLocalStorage() {
  try {
    const test = '__test__';
    localStorage.setItem(test, test);
    localStorage.removeItem(test);
    return true;
  } catch (e) {
    console.error('localStorage not available:', e);
    return false;
  }
}

if (!checkLocalStorage()) {
  alert('localStorage is required for data persistence');
}
```

### CORS Errors on Local Development

Always serve via HTTP server, not `file://` protocol:

```bash
# Python 3
python -m http.server 8000

# Node.js
npx http-server -p 8000

# PHP
php -S localhost:8000
```

### Glassmorphism Effects Not Showing

Ensure backdrop-filter support and CSS properly loaded:

```css
/* Fallback for older browsers */
.glass-card {
  background: rgba(30, 41, 59, 0.7);
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px); /* Safari support */
  border: 1px solid rgba(255, 255, 255, 0.1);
}

/* Check support */
@supports (backdrop-filter: blur(10px)) {
  .glass-card {
    background: rgba(30, 41, 59, 0.5);
  }
}
```

### Mobile Responsiveness Issues

Add viewport meta tag and media queries:

```html
<meta name="viewport" content="width=device-width, initial-scale=1.0">
```

```css
@media (max-width: 768px) {
  .dashboard-grid {
    grid-template-columns: 1fr;
  }
  
  .account-card {
    font-size: 14px;
  }
}
```
