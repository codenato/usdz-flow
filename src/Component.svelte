<script>
  import { getContext, onMount } from "svelte";
  import { writable } from 'svelte/store';

  const { styleable, API, routeStore, appStore } = getContext("sdk");
  const component = getContext("component");
  const apiData = writable({
    customer: {},
    transactions: []
  });
  export let customerTableId = null;
  export let transactionsTableId = null;

  let customerId = null;
  routeStore.subscribe(state => {
    customerId = state.routeParams.customer_id;
    console.log('customerId', customerId);
  });

  async function fetchData() {
    try {

      const customerResponse = await API.post({
        url: `api/${customerTableId}/search`,
        body: {
          "query": {
            "paginate": "true",
            "equal" : {
              "id": Number(customerId)
            }
          }
        }
      });

      console.log('customerResponse', customerResponse);

      const transactionsResponse = await API.post({
        url: `api/${transactionsTableId}/search`,
        body: {
          "query": {
            "equal" : {
              "customer_id": customerId
            }
          },
          "limit": 1000,
          "sort": "date",
          "sortOrder": "descending"
        }
      });

      console.log('transactionsResponse', transactionsResponse);

      apiData.set({
        customer: customerResponse?.rows[0],
        transactions: transactionsResponse?.rows
      });

      console.log('apiData', $apiData);
    } catch (error) {
      console.error('Error fetching data:', error);
    }
  }

  onMount(() => {
    fetchData();
  });

  export const snippets = {};
  function extractSnippets(appData) {

    if (appData && appData.application && Array.isArray(appData.application.snippets)) {
      appData.application.snippets.forEach(snippet => {
        if (snippet.name && snippet.code) {
          try {
            // Create a new function from the snippet code
            snippets[snippet.name] = new Function(snippet.code)();
          } catch (error) {
            console.error(`Error creating function for snippet ${snippet.name}:`, error);
          }
        }
      });
    }

    return snippets;
  }

  appStore.subscribe(state => {
    console.log('appStore', state);
    extractSnippets(state);
    console.log('snippets', snippets);
  });

  export let lastUpdate = new Date();
  export let customer = {};
  export let transactions = [];
  $: if ($apiData.customer && $apiData.transactions) {
    customer = $apiData.customer;
    transactions = $apiData.transactions;
  }
</script>

<div use:styleable={$component.styles} class="container">
  <header class="header">
    <div class="logo">
    </div>
    <div class="welcome">
      <div class="welcome-text">Bem-vindo,</div>
      <div class="welcome-name">{customer.name}</div>
    </div>
  </header>

  <main>
    <section>
      <div class="main-balance balance-card">
        <span class="transaction-icon" style="width: 30px; height: 30px;align-self: center;">
          <span style="margin-top: 2px; font-size: 18px;">$</span>
        </span>
        <div style="margin-left: 15px; display: flex; flex-direction: column;">
          <div class="balance-header">
            <span>Saldo em conta</span>
          </div>
          <div class="balance-amount">{snippets.formatToBRL(customer.balance + customer.future_balance)}</div>
        </div>
      </div>
      <div class="balance-details">
        <div class="balance-item balance-card">
          <div class="balance-label">Saldo Realizado</div>
          <div class="balance-value">{snippets.formatToBRL(customer.balance)}</div>
        </div>
        <div class="balance-item balance-card">
          <div class="balance-label">Lançamentos Futuro</div>
          <div class="balance-value">{snippets.formatToBRL(customer.future_balance)}</div>
        </div>
      </div>
      <div class="balance-details">
        <div class="balance-item balance-card">
          <div class="balance-label">Crédito Futuro</div>
          <div class="balance-value">{snippets.formatToBRL(customer.future_credit)}</div>
        </div>
        <div class="balance-item balance-card">
          <div class="balance-label">Saque Futuro</div>
          <div class="balance-value">{snippets.formatToBRL(customer.future_debit)}</div>
        </div>
      </div>
      <div class="last-update">
        Última atualização: {snippets.formatDateToBRL(lastUpdate) + ' ' + lastUpdate.getHours() + ':' + lastUpdate.getMinutes()}
      </div>
    </section>

    <h2 class="transactions-title">Transações</h2>
    <section class="transactions">
      {#each transactions as transaction}
        <div class="transaction-item">
          <div class="transaction-info">
            <div class="transaction-icon"><span>$</span></div>
            <span class="transaction-status status-{transaction.status.toLowerCase()}">{transaction.status}</span>
          </div>
          <span class="transaction-amount {transaction.amount >= 0 ? 'positive' : 'negative'}">
            {snippets.formatToBRL(transaction.amount)}
          </span>

          <div style="width: 100%; text-align: right;">
            <span class="transaction-date">{snippets.formatDateToBRL(transaction.date)}</span>
          </div>
        </div>
      {/each}
    </section>
  </main>
</div>

<style>
  :global(:root) {
    --background-color: #0F172A !important;
    --card-background: #162033;
    --text-color: white;
    --text-muted: #94A3B8;
    --primary-color: #03A6D1;
    --success-color: #4ADE80;
    --warning-color: #FACC15;
    --danger-color: #F87171;
  }

  :global(body) {
    font-family: 'Roboto', sans-serif;
    background-color: var(--background-color);
    color: var(--text-color);
    margin: 0;
    padding: 0;
  }

  .container {
    max-width: 430px;
    width: 380px;
    margin: 0 auto;
    padding: 18px 24px;
  }

  .header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 24px;
  }

  .logo {
    background-size: cover;
    background-repeat: no-repeat;
    display: flex;
    align-items: center;
    gap: 6.51px;
    background-image: url(https://imagedelivery.net/-anfAm5elOFfko_Gz_HjVQ/60dd0b00-950d-498d-9b45-4a67371f1800/public);
    width: 173px;
    height: 36px;
    background-position: left;
    margin-left: -29px;
    margin-top: 6px;
  }

  .welcome {
    text-align: right;
  }

  .welcome-text {
    color: var(--primary-color);
    font-size: 16px;
  }

  .welcome-name {
    font-size: 20px;
  }

  .main-balance {
    display: flex;
  }

  .balance-card {
    background-color: var(--card-background);
    border-radius: 8px;
    padding: 16px;
    margin-bottom: 12px;
  }

  .balance-header {
    display: flex;
    gap: 10px;
    color: var(--text-muted);
  }

  .currency-symbol {
    color: var(--primary-color);
  }

  .balance-amount {
    font-size: 30px;
    font-weight: 500;
  }

  .balance-details {
    display: flex;
    justify-content: space-between;
    gap: 12px;
  }

  .balance-item {
    flex: 1;
    background-color: var(--card-background);
    border-radius: 8px;
    padding: 16px;
  }

  .balance-label {
    color: var(--text-muted);
    font-size: 12px;
    margin-bottom: 4px;
  }

  .balance-value {
    color: var(--primary-color);
    font-size: 16px;
    font-weight: 500;
  }

  .last-update {
    color: var(--text-muted);
    font-size: 12px;
  }

  .transactions-title {
    font-size: 18px;
    font-weight: 500;
    margin: 24px 0 16px;
  }

  .transaction-item {
    background-color: var(--card-background);
    border-radius: 8px;
    padding: 14px;
    display: flex;
    flex-wrap: wrap;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 16px;
  }

  .transaction-info {
    display: flex;
    align-items: center;
    gap: 24px;
  }

  .transaction-icon {
    width: 18px;
    height: 18px;
    color: var(--primary-color);
    border-radius: 50%;
    border: 1px solid var(--primary-color);
    text-align: center;
    font-weight: bolder;
  }

  .transaction-icon span {
    display: block;
    margin-top: -1px;
  }

  .transaction-status {
    padding: 0 8px;
    border-radius: 6px;
    font-size: 14px;
  }

  .transaction-date {
    display: inline-block;
    text-align: right;
    color: var(--text-muted);
    font-size: 12px;
    margin-top: 4px;
  }

  .status-programado {
    background-color: rgba(202, 138, 4, 0.60);
    color: var(--warning-color);
  }

  .status-realizado {
    background-color: #064E3B;
    color: #34D399;
  }

  .transaction-amount {
    font-size: 16px;
  }

  .transaction-amount.positive {
    color: var(--success-color);
  }

  .transaction-amount.negative {
    color: var(--danger-color);
  }
</style>