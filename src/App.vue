<script setup lang="ts">
// This starter template is using Vue 3 <script setup> SFCs
// Check out https://vuejs.org/api/sfc-script-setup.html#script-setup

import { ref } from "vue";
import type { Ref } from "vue";
import { getClient, Client } from "@tauri-apps/api/http";
import { SocketAddress } from "net";

interface RealTimeQuoteResponse {
  data: Array<RealTimeQuote>;
  type: string;
}

interface RealTimeQuoteErrorResponse {
  msg: string;
  type: string;
}

interface RealTimeQuote {
  c: string; // condition codes
  p: string; // price
  s: string; // symbol
  t: string; // timestamp
  v: string; // volume
}

interface Result {
  description: string;
  displaySymbol: string;
  symbol: string;
  type: string;
}

const quotes: Ref<Map<string, RealTimeQuote>> = ref(
  new Map<string, RealTimeQuote>()
);

// API connection
const socket: Ref<WebSocket | undefined> = ref();
const query: Ref<string> = ref("");
const symbol: Ref<string> = ref("");
const token: Ref<string> = ref("");

const search = async function (query: string, token: string) {
  const url = `https://finnhub.io/api/v1/search?q=${query}&token=${token}`;
  const client: Client = await getClient();
  const response: any = await client.get(url);

  if (response && response.data) {
    return response.data.result;
  }
};

const addSymbol = function (symbol: string) {
  subscribe(symbol);
};

const performSearch = async function (query: string, token: string) {
  const results = await search(query, token);
  results.forEach((result: Result) => console.log(result));
};

const formatter = Intl.NumberFormat("en-US", {
  style: "currency",
  currency: "USD",
  minimumFractionDigits: 2,
  maximumFractionDigits: 2,
});

const realTimeQuoteResponseHandler = function (event: MessageEvent) {
  let response: RealTimeQuoteResponse | RealTimeQuoteErrorResponse = JSON.parse(
    event.data
  );
  if (response) {
    switch (response.type) {
      case "error":
        let error: RealTimeQuoteErrorResponse =
          response as RealTimeQuoteErrorResponse;
        console.error(error);
        break;
      case "trade":
        let trade: RealTimeQuoteResponse = response as RealTimeQuoteResponse;
        trade.data.forEach((quote: RealTimeQuote) => {
          quotes.value.set(quote.s, quote);
        });
        break;
      default:
        return;
    }
  }
};

const subscribeHandler = function (event: any) {
  subscribe("AAPL");
  subscribe("GME");
  subscribe("TSLA");
  subscribe("SHOP");
  subscribe("BINANCE:BTCUSDT");
  subscribe("BINANCE:ETHUSDT");
  subscribe("BINANCE:ADAUSDT");
  subscribe("BINANCE:SOLUSDT");
  // subscribe("IC MARKETS:1");
};

// Connection opened -> Subscribe
const start = function (token: string) {
  socket.value = new WebSocket("wss://ws.finnhub.io?token=" + token);
  // subscribe to stocks
  socket.value.addEventListener("open", subscribeHandler);

  // Listen for messages
  socket.value.addEventListener("message", realTimeQuoteResponseHandler);
};

const stop = function () {
  if (socket.value !== undefined) {
    socket.value.removeEventListener("message", realTimeQuoteResponseHandler);
    socket.value.removeEventListener("open", subscribeHandler);
    socket.value.close();
  }
};

// Subscribe symbol to real time quotes
const subscribe = function (symbol: string) {
  if (socket.value !== undefined) {
    socket.value.send(JSON.stringify({ type: "subscribe", symbol }));
  }
};

// Unsubscribe symbol from real time quotes
const unsubscribe = function (symbol: string) {
  if (socket.value !== undefined) {
    socket.value.send(JSON.stringify({ type: "unsubscribe", symbol }));
  }
};

// Beautify symbol rendering by removing market markings
const formatSymbol = function (symbol: string) {
  return symbol.replace(/BINANCE:(?<symbol>.+)USDT/, "$1");
};

// Convert symbol to font-awesome icon identifier
const faSymbol = function (symbol: string) {
  switch (formatSymbol(symbol).toLowerCase()) {
    case "eth":
      return "ethereum";
    case "btc":
      return "btc";
    default:
      return "";
  }
};

// Render font-awesome class for a given symbol
const symbolIconString = function (symbol: string) {
  try {
    return "fa-brands fa-" + faSymbol(symbol);
  } catch (err) {
    return "";
  }
};

// Hard-coded map from symbol to boolean, wheter the symbol exists in font-awesome brands icon set
const isBranded = function (symbol: string) {
  const branded = ["BTC", "ETH"];
  if (branded.includes(formatSymbol(symbol))) {
    return true;
  } else {
    return false;
  }
};
</script>

<template>
  <div class="container">
    <div class="column">
      <ul class="subscriptions">
        <li v-for="[s, quote] in quotes" :key="s">
          {{ s }}&nbsp;<button>
            <font-awesome-icon icon="fa-solid fa-times" />
          </button>
        </li>
      </ul>
    </div>
    <div class="row">
      <table v-for="[s, quote] in quotes" :key="s" width="300px">
        <tr>
          <th>
            <b
              ><font-awesome-icon
                :icon="symbolIconString(quote.s)"
                v-if="isBranded(quote.s)"
              />
              {{ formatSymbol(quote.s) }}</b
            >
          </th>
        </tr>
        <tr>
          <td>{{ formatter.format(Number(quote.p)) }}</td>
        </tr>
        <tr>
          <td>{{ quote.v }}</td>
        </tr>
      </table>
    </div>
    <div class="row">
      <form @submit.prevent="performSearch(query, token)">
        <input type="text" name="search" placeholder="Search" v-model="query" />
        <input type="submit" value="Search" />
      </form>

      <form @submit.prevent="addSymbol(symbol)">
        <input type="text" name="stock" placeholder="Symbol" v-model="symbol" />
        <input type="submit" value="Add" />
      </form>

      <input type="text" name="api_key" placeholder="API Key" v-model="token" />
      <button @click="start(token)">Start</button>
      <button @click="stop">Stop</button>
    </div>
  </div>
</template>

<style scoped>
table {
  background-color: #0f0f0f;
}

th {
  background-color: #505050;
  border-top-left-radius: 4px;
  border-top-right-radius: 4px;
}

td {
  background-color: #2d2d2d;
}

.subscriptions {
  max-width: 300px;
  border-radius: 0;
  border: 2px solid transparent;
  padding: 0;
  font-size: 1rem;
  font-weight: 500;
  font-family: inherit;
  color: #0f0f0f;
  background-color: #ffffff;
  transition: border-color 0.25s;
  box-shadow: 0 2px 2px rgba(0, 0, 0, 0.2);
  list-style-type: none;
}

.subscriptions > li:hover {
  background-color: #396cd8;
}

.subscriptions > li {
  text-align: left;
  position: relative;
  padding-right: 1rem;
}

.subscriptions > li > button {
  border: none;
  padding: 1px;
  display: block;
  position: absolute;
  right: 0;
  top: 0;
}
</style>
