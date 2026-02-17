# UniClaw Workshop 04 — Full-Stack Uniswap Interface
### Backend → API Layer → The Graph → Human-Readable UI

> **Prerequisites:** Workshops 01–03 complete. You understand pools, signals,
> LP operations, and arb mechanics. Now you will build the interface that
> makes all of it visible, readable, and actionable to a human operator.
>
> **What this workshop builds:** A complete full-stack DeFi dashboard — a
> Node.js/Express API layer that proxies The Graph and DefiLlama, a React
> frontend that consumes it, and a design system that translates raw
> on-chain numbers into instantly understandable human information.
>
> **The guiding principle:** Every number on screen must answer a question
> a human would actually ask. Not `1823748291929` — but `$1.82M TVL ↑3.4%`.
>
> **Tag all STATE.md entries:** `WORKSHOP_04:`

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                        BROWSER                              │
│                                                             │
│   React Dashboard  ←→  React Query (cache + refetch)       │
│   Components            ↕                                   │
│   Formatters        REST calls to your API                  │
└─────────────────────────────┬───────────────────────────────┘
                              │  HTTP
┌─────────────────────────────▼───────────────────────────────┐
│                    YOUR API SERVER (Node/Express)            │
│                                                             │
│   /api/pools        /api/pool/:id      /api/position/:id    │
│   /api/tokens       /api/arb           /api/signals         │
│                             ↕                               │
│   API Key Manager   ←→   Rate Limiter  ←→  Response Cache   │
└──────┬────────────────────────┬───────────────────────────┬─┘
       │                        │                           │
       ▼                        ▼                           ▼
The Graph Gateway          DefiLlama API           Uniswap Trading API
(GraphQL, API key)         (REST, free)            (REST, API key)
V3/V4 subgraphs            TVL, prices             Quotes, routes
```

---

# PART 1 — ENVIRONMENT & API KEY SETUP

## 1.1 Project Scaffold

```bash
mkdir uniclaw-dashboard && cd uniclaw-dashboard

# Backend
mkdir server && cd server
npm init -y
npm install express cors dotenv node-fetch@2 express-rate-limit node-cache

# Frontend (separate terminal)
cd ..
npx create-react-app client
cd client
npm install @tanstack/react-query recharts date-fns axios
```

## 1.2 Environment Variables — Never Hardcode Keys

```bash
# server/.env
# -------------------------------------------
# The Graph API key — get from thegraph.com/studio
GRAPH_API_KEY=your_graph_api_key_here

# Uniswap Trading API key — get from hub.uniswap.org
UNISWAP_TRADING_API_KEY=your_trading_api_key_here

# Server config
PORT=4000
ALLOWED_ORIGIN=http://localhost:3000

# Cache TTLs (seconds)
CACHE_POOL_LIST=60
CACHE_POOL_DETAIL=30
CACHE_TOKEN_PRICE=20
CACHE_POSITIONS=30
```

```bash
# server/.gitignore  — CRITICAL: never commit keys
.env
node_modules/
```

## 1.3 Subgraph Endpoint Builder

```javascript
// server/src/config/endpoints.js
const SUBGRAPH_BASE = 'https://gateway.thegraph.com/api';

const SUBGRAPH_IDS = {
  v4: 'DiYPVdygkfjDWhbxGSqAQxwBKmfKnkWQojqeM2rkLb3G',
  v3: '5zvR82QoaXYFyDEKLZ9t6v9adgnptxYpKpSbxtgVENFV',
  v2: 'A3Np3RQbaBA6oKJgiwDJeo5T3zrYfGHPWFYayMwtNDum',
};

function subgraphUrl(version) {
  const key = process.env.GRAPH_API_KEY;
  if (!key) throw new Error('GRAPH_API_KEY not set in .env');
  return `${SUBGRAPH_BASE}/${key}/subgraphs/id/${SUBGRAPH_IDS[version]}`;
}

module.exports = {
  subgraphUrl,
  DEFILLAMA: 'https://api.llama.fi',
  DEFILLAMA_COINS: 'https://coins.llama.fi',
  UNISWAP_TRADING: 'https://trading-api.gateway.uniswap.org/v1',
};
```

---

# PART 2 — THE GRAPH QUERY LAYER

## 2.1 The GraphQL Client

```javascript
// server/src/graphql/client.js
const fetch = require('node-fetch');
const NodeCache = require('node-cache');

const queryCache = new NodeCache({ stdTTL: 30 });

async function graphQuery(url, query, variables = {}, ttl = 30) {
  const cacheKey = `${url}:${JSON.stringify({ query, variables })}`;
  const cached = queryCache.get(cacheKey);
  if (cached) return cached;

  const res = await fetch(url, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ query, variables }),
    timeout: 10000,
  });

  if (!res.ok) {
    throw new Error(`Graph request failed: ${res.status} ${res.statusText}`);
  }

  const { data, errors } = await res.json();

  if (errors) {
    // Log but don't crash — return partial data if available
    console.error('GraphQL errors:', errors);
    if (!data) throw new Error(errors[0].message);
  }

  queryCache.set(cacheKey, data, ttl);
  return data;
}

module.exports = { graphQuery };
```

## 2.2 All Queries in One Place

```javascript
// server/src/graphql/queries.js

const QUERIES = {

  // ─── Pool List ───────────────────────────────────────────────
  TOP_POOLS: `
    query TopPools($first: Int!, $skip: Int!) {
      pools(
        first: $first, skip: $skip,
        orderBy: totalValueLockedUSD, orderDirection: desc
      ) {
        id feeTier
        token0 { id symbol name decimals }
        token1 { id symbol name decimals }
        sqrtPrice tick liquidity
        totalValueLockedUSD
        volumeUSD feesUSD txCount
        token0Price token1Price
        poolDayData(first: 2, orderBy: date, orderDirection: desc) {
          date open close high low volumeUSD feesUSD tvlUSD
        }
      }
    }
  `,

  // ─── Single Pool Detail ───────────────────────────────────────
  POOL_DETAIL: `
    query PoolDetail($id: ID!) {
      pool(id: $id) {
        id feeTier sqrtPrice tick liquidity
        token0 { id symbol name decimals }
        token1 { id symbol name decimals }
        totalValueLockedUSD volumeUSD feesUSD txCount
        token0Price token1Price
        feeGrowthGlobal0X128 feeGrowthGlobal1X128
        poolDayData(first: 90, orderBy: date, orderDirection: asc) {
          date open high low close volumeUSD feesUSD tvlUSD txCount
        }
        poolHourData(first: 48, orderBy: periodStartUnix, orderDirection: desc) {
          periodStartUnix open high low close volumeUSD feesUSD tvlUSD
        }
      }
    }
  `,

  // ─── Pool Ticks ───────────────────────────────────────────────
  POOL_TICKS: `
    query PoolTicks($poolId: String!, $skip: Int!) {
      ticks(
        where: { pool: $poolId, liquidityNet_not: "0" }
        orderBy: tickIdx, orderDirection: asc
        first: 1000, skip: $skip
      ) {
        tickIdx liquidityNet liquidityGross price0 price1
      }
    }
  `,

  // ─── Recent Swaps ─────────────────────────────────────────────
  RECENT_SWAPS: `
    query RecentSwaps($poolId: String!, $first: Int!) {
      swaps(
        where: { pool: $poolId }
        orderBy: timestamp, orderDirection: desc
        first: $first
      ) {
        id timestamp
        amount0 amount1 amountUSD
        sqrtPriceX96 tick
        sender recipient
        transaction { id gasUsed gasPrice }
      }
    }
  `,

  // ─── LP Positions ─────────────────────────────────────────────
  POOL_POSITIONS: `
    query PoolPositions($poolId: String!, $first: Int!) {
      positions(
        where: { pool: $poolId, liquidity_gt: "0" }
        orderBy: liquidity, orderDirection: desc
        first: $first
      ) {
        id owner liquidity
        tickLower { tickIdx price0 price1
                    feeGrowthOutside0X128 feeGrowthOutside1X128 }
        tickUpper { tickIdx price0 price1
                    feeGrowthOutside0X128 feeGrowthOutside1X128 }
        depositedToken0 depositedToken1
        withdrawnToken0 withdrawnToken1
        collectedFeesToken0 collectedFeesToken1
        feeGrowthInside0LastX128 feeGrowthInside1LastX128
        transaction { timestamp }
      }
    }
  `,

  // ─── Token Search ─────────────────────────────────────────────
  TOKEN_SEARCH: `
    query TokenSearch($query: String!) {
      tokens(
        where: { symbol_contains_nocase: $query }
        orderBy: totalValueLockedUSD, orderDirection: desc
        first: 10
      ) {
        id symbol name decimals
        totalValueLockedUSD volumeUSD txCount
        tokenDayData(first: 1, orderBy: date, orderDirection: desc) {
          priceUSD
        }
      }
    }
  `,

  // ─── Protocol Stats ───────────────────────────────────────────
  PROTOCOL_STATS: `
    {
      uniswapDayDatas(first: 30, orderBy: date, orderDirection: desc) {
        date volumeUSD feesUSD tvlUSD txCount
      }
    }
  `,

  // ─── v4 Pools (different field names) ─────────────────────────
  V4_TOP_POOLS: `
    query V4TopPools($first: Int!) {
      pools(
        first: $first,
        orderBy: totalValueLockedUSD, orderDirection: desc
      ) {
        id feeTier hooks
        currency0 { id symbol name }
        currency1 { id symbol name }
        sqrtPrice tick liquidity
        totalValueLockedUSD volumeUSD txCount
      }
    }
  `,
};

module.exports = QUERIES;
```

---

# PART 3 — EXPRESS API SERVER

## 3.1 Main Server

```javascript
// server/src/index.js
require('dotenv').config();
const express    = require('express');
const cors       = require('cors');
const rateLimit  = require('express-rate-limit');

const app = express();

app.use(cors({ origin: process.env.ALLOWED_ORIGIN }));
app.use(express.json());

// Rate limit: prevent accidental query floods
app.use('/api/', rateLimit({
  windowMs: 60 * 1000,   // 1 minute
  max: 120,               // 120 requests per minute
  message: { error: 'Too many requests, please slow down.' }
}));

// Routes
app.use('/api/pools',    require('./routes/pools'));
app.use('/api/tokens',   require('./routes/tokens'));
app.use('/api/protocol', require('./routes/protocol'));
app.use('/api/quote',    require('./routes/quote'));

// Health check
app.get('/api/health', (req, res) => {
  res.json({
    status: 'ok',
    graph_key_set: !!process.env.GRAPH_API_KEY,
    trading_key_set: !!process.env.UNISWAP_TRADING_API_KEY,
    timestamp: new Date().toISOString(),
  });
});

// Global error handler — never leak stack traces to client
app.use((err, req, res, next) => {
  console.error('[ERROR]', err.message);
  res.status(500).json({ error: 'Internal server error', hint: err.message });
});

const PORT = process.env.PORT || 4000;
app.listen(PORT, () => console.log(`UniClaw API running on :${PORT}`));
```

## 3.2 Pools Route

```javascript
// server/src/routes/pools.js
const router = require('express').Router();
const { graphQuery }  = require('../graphql/client');
const { subgraphUrl } = require('../config/endpoints');
const QUERIES         = require('../graphql/queries');
const { formatPool, formatPoolDetail,
        formatSwap, formatPosition, formatTicks } = require('../formatters');

// GET /api/pools?version=v3&limit=20&page=0
router.get('/', async (req, res, next) => {
  try {
    const version = req.query.version || 'v3';
    const first   = Math.min(parseInt(req.query.limit)  || 20, 100);
    const skip    = (parseInt(req.query.page) || 0) * first;

    const query = version === 'v4' ? QUERIES.V4_TOP_POOLS : QUERIES.TOP_POOLS;
    const data  = await graphQuery(subgraphUrl(version), query,
                                   { first, skip }, parseInt(process.env.CACHE_POOL_LIST));

    res.json({
      pools:   (data.pools || []).map(p => formatPool(p, version)),
      version,
      count:   (data.pools || []).length,
    });
  } catch (err) { next(err); }
});

// GET /api/pools/:id?version=v3
router.get('/:id', async (req, res, next) => {
  try {
    const version = req.query.version || 'v3';
    const data    = await graphQuery(
      subgraphUrl(version), QUERIES.POOL_DETAIL,
      { id: req.params.id.toLowerCase() },
      parseInt(process.env.CACHE_POOL_DETAIL)
    );
    if (!data.pool) return res.status(404).json({ error: 'Pool not found' });
    res.json(formatPoolDetail(data.pool));
  } catch (err) { next(err); }
});

// GET /api/pools/:id/swaps?count=50
router.get('/:id/swaps', async (req, res, next) => {
  try {
    const count = Math.min(parseInt(req.query.count) || 50, 200);
    const data  = await graphQuery(
      subgraphUrl(req.query.version || 'v3'),
      QUERIES.RECENT_SWAPS,
      { poolId: req.params.id.toLowerCase(), first: count }, 15
    );
    res.json({ swaps: (data.swaps || []).map(formatSwap) });
  } catch (err) { next(err); }
});

// GET /api/pools/:id/ticks
router.get('/:id/ticks', async (req, res, next) => {
  try {
    let allTicks = [], skip = 0;
    while (true) {
      const data = await graphQuery(
        subgraphUrl(req.query.version || 'v3'),
        QUERIES.POOL_TICKS,
        { poolId: req.params.id.toLowerCase(), skip }, 60
      );
      allTicks.push(...(data.ticks || []));
      if (!data.ticks || data.ticks.length < 1000) break;
      skip += 1000;
    }
    res.json({ ticks: formatTicks(allTicks), count: allTicks.length });
  } catch (err) { next(err); }
});

// GET /api/pools/:id/positions?count=50
router.get('/:id/positions', async (req, res, next) => {
  try {
    const count = Math.min(parseInt(req.query.count) || 50, 200);
    const data  = await graphQuery(
      subgraphUrl(req.query.version || 'v3'),
      QUERIES.POOL_POSITIONS,
      { poolId: req.params.id.toLowerCase(), first: count }, 30
    );
    res.json({ positions: (data.positions || []).map(formatPosition) });
  } catch (err) { next(err); }
});

module.exports = router;
```

## 3.3 Data Formatters — The Human-Readable Layer

This is the most important file in the backend. Every raw number is
transformed into something a human can parse in under a second.

```javascript
// server/src/formatters/index.js

// ─── Number Formatters ────────────────────────────────────────

function fmtUSD(value, decimals = 2) {
  const n = parseFloat(value) || 0;
  if (n >= 1e9) return `$${(n / 1e9).toFixed(2)}B`;
  if (n >= 1e6) return `$${(n / 1e6).toFixed(2)}M`;
  if (n >= 1e3) return `$${(n / 1e3).toFixed(1)}K`;
  return `$${n.toFixed(decimals)}`;
}

function fmtPct(value, decimals = 2) {
  const n = parseFloat(value) || 0;
  const sign = n >= 0 ? '+' : '';
  return `${sign}${n.toFixed(decimals)}%`;
}

function fmtPrice(value, decimals = 6) {
  const n = parseFloat(value) || 0;
  if (n >= 10000) return n.toLocaleString('en-US', { maximumFractionDigits: 2 });
  if (n >= 1)     return n.toFixed(4);
  if (n >= 0.001) return n.toFixed(6);
  return n.toExponential(4);
}

function fmtCount(value) {
  const n = parseInt(value) || 0;
  if (n >= 1e6) return `${(n / 1e6).toFixed(1)}M`;
  if (n >= 1e3) return `${(n / 1e3).toFixed(1)}K`;
  return n.toLocaleString();
}

function fmtAge(timestamp) {
  const secs = Math.floor(Date.now() / 1000) - parseInt(timestamp);
  if (secs < 60)   return `${secs}s ago`;
  if (secs < 3600) return `${Math.floor(secs / 60)}m ago`;
  if (secs < 86400) return `${Math.floor(secs / 3600)}h ago`;
  return `${Math.floor(secs / 86400)}d ago`;
}

function fmtAddress(addr, chars = 6) {
  if (!addr) return '—';
  return `${addr.slice(0, chars)}...${addr.slice(-4)}`;
}

function fmtFeeTier(feeTier) {
  const map = { '100': '0.01%', '500': '0.05%', '3000': '0.3%', '10000': '1%' };
  return map[String(feeTier)] || `${(parseInt(feeTier) / 10000).toFixed(2)}%`;
}

function fmtLiquidity(raw) {
  const n = parseFloat(raw) || 0;
  if (n >= 1e18) return `${(n / 1e18).toFixed(2)}Q`;
  if (n >= 1e15) return `${(n / 1e15).toFixed(2)}P`;
  if (n >= 1e12) return `${(n / 1e12).toFixed(2)}T`;
  return n.toExponential(2);
}

// ─── Derived Metrics ──────────────────────────────────────────

function calcFeeAPR(feesUSD_24h, tvlUSD) {
  const fees = parseFloat(feesUSD_24h) || 0;
  const tvl  = parseFloat(tvlUSD) || 0;
  if (tvl === 0) return 0;
  return (fees / tvl) * 365 * 100;
}

function calcVolTVL(volumeUSD, tvlUSD) {
  const vol = parseFloat(volumeUSD) || 0;
  const tvl = parseFloat(tvlUSD)  || 0;
  return tvl > 0 ? vol / tvl : 0;
}

function calcDayChange(today, yesterday) {
  const t = parseFloat(today)     || 0;
  const y = parseFloat(yesterday) || 0;
  if (y === 0) return 0;
  return ((t - y) / y) * 100;
}

function priceTrend(dayData) {
  if (!dayData || dayData.length < 2) return null;
  const [latest, prev] = dayData;
  return {
    pct:       calcDayChange(latest.close, prev.close),
    direction: parseFloat(latest.close) >= parseFloat(prev.close) ? 'up' : 'down',
  };
}

// ─── Entity Formatters ────────────────────────────────────────

function formatPool(pool, version = 'v3') {
  const t0 = pool.token0 || pool.currency0 || {};
  const t1 = pool.token1 || pool.currency1 || {};
  const dayData = pool.poolDayData || [];
  const today   = dayData[0] || {};
  const yday    = dayData[1] || {};

  const feesYday = parseFloat(yday.feesUSD) || 0;
  const tvl      = parseFloat(pool.totalValueLockedUSD) || 0;
  const feeAPR   = calcFeeAPR(feesYday, tvl);
  const trend    = priceTrend(dayData);

  return {
    id:          pool.id,
    version,
    pair:        `${t0.symbol}/${t1.symbol}`,
    pairFull:    `${t0.name || t0.symbol} / ${t1.name || t1.symbol}`,
    token0:      { address: t0.id, symbol: t0.symbol, name: t0.name },
    token1:      { address: t1.id, symbol: t1.symbol, name: t1.name },

    feeTier:       fmtFeeTier(pool.feeTier),
    feeTierRaw:    parseInt(pool.feeTier),
    hooks:         pool.hooks || null,
    hasHooks:      pool.hooks && pool.hooks !== '0x0000000000000000000000000000000000000000',

    price:         fmtPrice(pool.token0Price),
    priceRaw:      parseFloat(pool.token0Price),
    priceTrend:    trend,

    tvl:           fmtUSD(tvl),
    tvlRaw:        tvl,
    tvlChange24h:  fmtPct(calcDayChange(today.tvlUSD, yday.tvlUSD)),

    volume24h:     fmtUSD(today.volumeUSD),
    volumeRaw:     parseFloat(today.volumeUSD) || 0,
    volumeChange:  fmtPct(calcDayChange(today.volumeUSD, yday.volumeUSD)),

    fees24h:       fmtUSD(today.feesUSD),
    feeAPR:        `${feeAPR.toFixed(2)}%`,
    feeAPRRaw:     feeAPR,
    feeAPRLabel:   feeAPR > 50 ? 'Very High' : feeAPR > 20 ? 'High'
                 : feeAPR > 10 ? 'Good'       : feeAPR > 5  ? 'Moderate' : 'Low',

    volTVLRatio:   calcVolTVL(today.volumeUSD, tvl).toFixed(4),
    txCount:       fmtCount(pool.txCount),
    liquidity:     fmtLiquidity(pool.liquidity),
    currentTick:   parseInt(pool.tick),
  };
}

function formatPoolDetail(pool) {
  const base = formatPool(pool);

  return {
    ...base,
    ohlcv: (pool.poolDayData || []).map(d => ({
      date:      new Date(d.date * 1000).toISOString().split('T')[0],
      open:      parseFloat(d.open),
      high:      parseFloat(d.high),
      low:       parseFloat(d.low),
      close:     parseFloat(d.close),
      volume:    parseFloat(d.volumeUSD),
      fees:      parseFloat(d.feesUSD),
      tvl:       parseFloat(d.tvlUSD),
      txCount:   parseInt(d.txCount),
      feeAPR:    calcFeeAPR(d.feesUSD, d.tvlUSD),
    })),
    hourly: (pool.poolHourData || []).reverse().map(h => ({
      time:   new Date(h.periodStartUnix * 1000).toISOString(),
      open:   parseFloat(h.open),
      close:  parseFloat(h.close),
      volume: parseFloat(h.volumeUSD),
      fees:   parseFloat(h.feesUSD),
      tvl:    parseFloat(h.tvlUSD),
    })),
  };
}

function formatSwap(swap) {
  const amt0 = parseFloat(swap.amount0);
  const isBuy = amt0 < 0;   // negative amount0 = user bought token0
  return {
    id:          swap.id,
    age:         fmtAge(swap.timestamp),
    timestamp:   parseInt(swap.timestamp),
    type:        isBuy ? 'Buy' : 'Sell',
    typeColor:   isBuy ? 'green' : 'red',
    amountUSD:   fmtUSD(swap.amountUSD),
    amountRaw:   parseFloat(swap.amountUSD),
    amount0:     Math.abs(amt0).toFixed(6),
    amount1:     Math.abs(parseFloat(swap.amount1)).toFixed(2),
    sender:      fmtAddress(swap.sender),
    recipient:   fmtAddress(swap.recipient),
    txHash:      swap.transaction?.id,
    txShort:     fmtAddress(swap.transaction?.id, 10),
    gasUSD:      fmtUSD(parseInt(swap.transaction?.gasUsed || 0) *
                        parseInt(swap.transaction?.gasPrice || 0) / 1e18 * 3200),
  };
}

function formatPosition(pos) {
  const liq    = parseFloat(pos.liquidity);
  const dep0   = parseFloat(pos.depositedToken0);
  const dep1   = parseFloat(pos.depositedToken1);
  const fees0  = parseFloat(pos.collectedFeesToken0);
  const fees1  = parseFloat(pos.collectedFeesToken1);
  const wd0    = parseFloat(pos.withdrawnToken0);
  const wd1    = parseFloat(pos.withdrawnToken1);

  const tLow  = pos.tickLower  || {};
  const tHigh = pos.tickUpper  || {};

  return {
    id:           pos.id,
    owner:        fmtAddress(pos.owner),
    ownerFull:    pos.owner,
    liquidity:    fmtLiquidity(pos.liquidity),
    liquidityRaw: liq,
    age:          fmtAge(pos.transaction?.timestamp),

    range: {
      tickLower:    parseInt(tLow.tickIdx),
      tickUpper:    parseInt(tHigh.tickIdx),
      priceLower:   fmtPrice(tLow.price0),
      priceUpper:   fmtPrice(tHigh.price0),
      priceLowerRaw: parseFloat(tLow.price0),
      priceUpperRaw: parseFloat(tHigh.price0),
      width:        parseInt(tHigh.tickIdx) - parseInt(tLow.tickIdx),
    },

    deposited: {
      token0:  dep0.toFixed(6),
      token1:  dep1.toFixed(2),
    },
    collected: {
      fees0:   fees0.toFixed(6),
      fees1:   fees1.toFixed(2),
    },
    withdrawn: {
      token0:  wd0.toFixed(6),
      token1:  wd1.toFixed(2),
    },

    whaleLabel: liq > 1e22 ? '🐋 Whale' : liq > 1e21 ? '🐬 Large' : '🐟 Retail',
  };
}

function formatTicks(ticks) {
  return ticks.map(t => ({
    tick:           parseInt(t.tickIdx),
    price:          parseFloat(t.price0),
    priceFmt:       fmtPrice(t.price0),
    liquidityNet:   t.liquidityNet,
    liquidityGross: t.liquidityGross,
    liquidityNetFmt: fmtLiquidity(Math.abs(parseInt(t.liquidityNet))),
    direction:      parseInt(t.liquidityNet) > 0 ? 'add' : 'remove',
  }));
}

module.exports = {
  formatPool, formatPoolDetail, formatSwap,
  formatPosition, formatTicks,
  fmtUSD, fmtPct, fmtPrice, fmtCount, fmtAge, fmtAddress,
};
```

---

# PART 4 — REACT FRONTEND

## 4.1 API Client

```javascript
// client/src/api/client.js
import axios from 'axios';

const api = axios.create({
  baseURL: process.env.REACT_APP_API_URL || 'http://localhost:4000/api',
  timeout: 15000,
});

// Attach error messages cleanly
api.interceptors.response.use(
  res => res.data,
  err => {
    const msg = err.response?.data?.error || err.message || 'Network error';
    return Promise.reject(new Error(msg));
  }
);

export const fetchPools      = (params) => api.get('/pools', { params });
export const fetchPool        = (id, version) => api.get(`/pools/${id}`, { params: { version } });
export const fetchPoolSwaps   = (id, count)   => api.get(`/pools/${id}/swaps`, { params: { count } });
export const fetchPoolTicks   = (id)          => api.get(`/pools/${id}/ticks`);
export const fetchPoolPositions = (id, count) => api.get(`/pools/${id}/positions`, { params: { count } });
export const fetchProtocol    = ()            => api.get('/protocol');
export const fetchQuote       = (params)      => api.post('/quote', params);
```

## 4.2 React Query Setup

```javascript
// client/src/index.js
import React from 'react';
import ReactDOM from 'react-dom/client';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import App from './App';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime:  30_000,   // data fresh for 30s
      cacheTime:  5 * 60_000,
      retry: 2,
      refetchOnWindowFocus: true,
    },
  },
});

ReactDOM.createRoot(document.getElementById('root')).render(
  <QueryClientProvider client={queryClient}>
    <App />
  </QueryClientProvider>
);
```

## 4.3 Dashboard Layout

```jsx
// client/src/App.jsx
import React, { useState } from 'react';
import { useQuery } from '@tanstack/react-query';
import { fetchPools, fetchProtocol } from './api/client';
import PoolTable    from './components/PoolTable';
import PoolDrawer   from './components/PoolDrawer';
import ProtocolBar  from './components/ProtocolBar';
import './styles/dashboard.css';

export default function App() {
  const [selectedPool, setSelectedPool] = useState(null);
  const [version, setVersion] = useState('v3');

  const { data: poolsData, isLoading, error } = useQuery({
    queryKey: ['pools', version],
    queryFn:  () => fetchPools({ version, limit: 20 }),
    refetchInterval: 60_000,
  });

  const { data: protocolData } = useQuery({
    queryKey: ['protocol'],
    queryFn:  fetchProtocol,
    refetchInterval: 120_000,
  });

  return (
    <div className="dashboard">
      <header className="dashboard__header">
        <div className="header__brand">
          <span className="brand-mark">◈</span>
          <span className="brand-name">UniClaw</span>
          <span className="brand-sub">Protocol Intelligence</span>
        </div>
        <VersionToggle value={version} onChange={setVersion} />
        <LiveIndicator />
      </header>

      {protocolData && <ProtocolBar data={protocolData} />}

      <main className="dashboard__main">
        {error && <ErrorBanner message={error.message} />}
        <PoolTable
          pools={poolsData?.pools || []}
          loading={isLoading}
          onSelect={setSelectedPool}
          selected={selectedPool?.id}
        />
      </main>

      {selectedPool && (
        <PoolDrawer
          poolId={selectedPool.id}
          version={version}
          onClose={() => setSelectedPool(null)}
        />
      )}
    </div>
  );
}

function VersionToggle({ value, onChange }) {
  return (
    <div className="version-toggle">
      {['v2', 'v3', 'v4'].map(v => (
        <button
          key={v}
          className={`version-btn ${value === v ? 'active' : ''}`}
          onClick={() => onChange(v)}
        >{v.toUpperCase()}</button>
      ))}
    </div>
  );
}

function LiveIndicator() {
  return (
    <div className="live-indicator">
      <span className="live-dot" />
      <span>Live</span>
    </div>
  );
}

function ErrorBanner({ message }) {
  return (
    <div className="error-banner">
      <span>⚠</span> {message}
    </div>
  );
}
```

## 4.4 Protocol Stats Bar

```jsx
// client/src/components/ProtocolBar.jsx
import React from 'react';

export default function ProtocolBar({ data }) {
  const stats = data?.stats || {};
  const items = [
    { label: '24h Volume',    value: stats.volume24h,  change: stats.volumeChange,  icon: '↕' },
    { label: 'Total TVL',     value: stats.tvl,        change: stats.tvlChange,     icon: '⬡' },
    { label: '24h Fees',      value: stats.fees24h,    change: stats.feesChange,    icon: '◎' },
    { label: 'Transactions',  value: stats.txCount24h, change: null,               icon: '⟳' },
  ];

  return (
    <div className="protocol-bar">
      {items.map(item => (
        <div className="protocol-stat" key={item.label}>
          <span className="stat-icon">{item.icon}</span>
          <div className="stat-body">
            <span className="stat-label">{item.label}</span>
            <span className="stat-value">{item.value || '—'}</span>
            {item.change && (
              <span className={`stat-change ${item.change.startsWith('+') ? 'up' : 'dn'}`}>
                {item.change}
              </span>
            )}
          </div>
        </div>
      ))}
    </div>
  );
}
```

## 4.5 Pool Table

```jsx
// client/src/components/PoolTable.jsx
import React, { useState } from 'react';

const COLUMNS = [
  { key: 'pair',       label: 'Pool',       width: '20%' },
  { key: 'feeTier',    label: 'Fee',        width: '7%'  },
  { key: 'tvl',        label: 'TVL',        width: '12%' },
  { key: 'volume24h',  label: 'Volume 24h', width: '12%' },
  { key: 'fees24h',    label: 'Fees 24h',   width: '10%' },
  { key: 'feeAPR',     label: 'Fee APR',    width: '10%' },
  { key: 'txCount',    label: 'Txns',       width: '8%'  },
  { key: 'price',      label: 'Price',      width: '10%' },
];

export default function PoolTable({ pools, loading, onSelect, selected }) {
  const [sortKey, setSortKey] = useState('tvlRaw');
  const [sortDir, setSortDir] = useState('desc');

  const sorted = [...pools].sort((a, b) => {
    const av = a[sortKey] ?? 0;
    const bv = b[sortKey] ?? 0;
    return sortDir === 'desc' ? bv - av : av - bv;
  });

  const handleSort = (key) => {
    if (sortKey === key) setSortDir(d => d === 'desc' ? 'asc' : 'desc');
    else { setSortKey(key); setSortDir('desc'); }
  };

  if (loading) return <TableSkeleton />;

  return (
    <div className="pool-table-wrap">
      <table className="pool-table">
        <thead>
          <tr>
            {COLUMNS.map(col => (
              <th
                key={col.key}
                style={{ width: col.width }}
                className={`th-sortable ${sortKey === col.key + 'Raw' || sortKey === col.key ? 'active' : ''}`}
                onClick={() => handleSort(col.key + 'Raw')}
              >
                {col.label}
                <span className="sort-arrow">
                  {sortKey.startsWith(col.key) ? (sortDir === 'desc' ? ' ↓' : ' ↑') : ' ↕'}
                </span>
              </th>
            ))}
          </tr>
        </thead>
        <tbody>
          {sorted.map(pool => (
            <PoolRow
              key={pool.id}
              pool={pool}
              selected={selected === pool.id}
              onClick={() => onSelect(pool)}
            />
          ))}
        </tbody>
      </table>
    </div>
  );
}

function PoolRow({ pool, selected, onClick }) {
  const trend = pool.priceTrend;
  return (
    <tr
      className={`pool-row ${selected ? 'selected' : ''} ${pool.hasHooks ? 'has-hooks' : ''}`}
      onClick={onClick}
    >
      <td className="td-pair">
        <div className="pair-name">{pool.pair}</div>
        <div className="pair-sub">
          {pool.version.toUpperCase()}
          {pool.hasHooks && <span className="hook-badge" title="Has Hooks">⚡</span>}
        </div>
      </td>
      <td><span className="fee-badge">{pool.feeTier}</span></td>
      <td>
        <span className="val-primary">{pool.tvl}</span>
        <span className={`val-change ${pool.tvlChange24h?.startsWith('+') ? 'up' : 'dn'}`}>
          {pool.tvlChange24h}
        </span>
      </td>
      <td>
        <span className="val-primary">{pool.volume24h}</span>
        <span className={`val-change ${pool.volumeChange?.startsWith('+') ? 'up' : 'dn'}`}>
          {pool.volumeChange}
        </span>
      </td>
      <td>{pool.fees24h}</td>
      <td>
        <span className={`apr-badge apr-${pool.feeAPRLabel?.toLowerCase().replace(' ', '-')}`}>
          {pool.feeAPR}
        </span>
      </td>
      <td>{pool.txCount}</td>
      <td>
        <span className="val-primary">{pool.price}</span>
        {trend && (
          <span className={`price-arrow ${trend.direction}`}>
            {trend.direction === 'up' ? '▲' : '▼'} {Math.abs(trend.pct).toFixed(2)}%
          </span>
        )}
      </td>
    </tr>
  );
}

function TableSkeleton() {
  return (
    <div className="skeleton-wrap">
      {Array.from({ length: 10 }).map((_, i) => (
        <div key={i} className="skeleton-row" style={{ animationDelay: `${i * 60}ms` }} />
      ))}
    </div>
  );
}
```

## 4.6 Pool Detail Drawer

```jsx
// client/src/components/PoolDrawer.jsx
import React, { useState } from 'react';
import { useQuery } from '@tanstack/react-query';
import { fetchPool, fetchPoolSwaps, fetchPoolPositions, fetchPoolTicks } from '../api/client';
import PriceChart    from './charts/PriceChart';
import VolumeChart   from './charts/VolumeChart';
import LiquidityMap  from './charts/LiquidityMap';
import SwapFeed      from './SwapFeed';
import PositionsList from './PositionsList';

const TABS = ['Overview', 'Swaps', 'Positions', 'Liquidity Map'];

export default function PoolDrawer({ poolId, version, onClose }) {
  const [tab, setTab] = useState('Overview');

  const { data: pool, isLoading } = useQuery({
    queryKey:  ['pool', poolId, version],
    queryFn:   () => fetchPool(poolId, version),
    refetchInterval: 30_000,
  });

  const { data: swapsData } = useQuery({
    queryKey:  ['swaps', poolId],
    queryFn:   () => fetchPoolSwaps(poolId, 100),
    enabled:   tab === 'Swaps',
    refetchInterval: 15_000,
  });

  const { data: posData } = useQuery({
    queryKey:  ['positions', poolId],
    queryFn:   () => fetchPoolPositions(poolId, 100),
    enabled:   tab === 'Positions',
  });

  const { data: tickData } = useQuery({
    queryKey:  ['ticks', poolId],
    queryFn:   () => fetchPoolTicks(poolId),
    enabled:   tab === 'Liquidity Map',
    staleTime: 120_000,
  });

  if (isLoading) return <div className="drawer drawer--loading"><Spinner /></div>;
  if (!pool) return null;

  return (
    <aside className="drawer">
      <div className="drawer__header">
        <div className="drawer__title">
          <span className="drawer__pair">{pool.pair}</span>
          <span className="fee-badge">{pool.feeTier}</span>
          {pool.hasHooks && <span className="hook-badge">⚡ Hooks</span>}
        </div>
        <button className="drawer__close" onClick={onClose}>✕</button>
      </div>

      <div className="drawer__metrics">
        <Metric label="TVL"      value={pool.tvl}      change={pool.tvlChange24h} />
        <Metric label="24h Vol"  value={pool.volume24h} change={pool.volumeChange} />
        <Metric label="Fee APR"  value={pool.feeAPR}   badge={pool.feeAPRLabel} />
        <Metric label="Price"    value={pool.price}    change={pool.priceTrend?.pct?.toFixed(2) + '%'} />
      </div>

      <nav className="drawer__tabs">
        {TABS.map(t => (
          <button key={t}
            className={`tab-btn ${tab === t ? 'active' : ''}`}
            onClick={() => setTab(t)}
          >{t}</button>
        ))}
      </nav>

      <div className="drawer__content">
        {tab === 'Overview' && pool.ohlcv && (
          <>
            <SectionLabel>Price (90 days)</SectionLabel>
            <PriceChart data={pool.ohlcv} />
            <SectionLabel>Daily Volume & Fees</SectionLabel>
            <VolumeChart data={pool.ohlcv} />
            <PoolMetaTable pool={pool} />
          </>
        )}
        {tab === 'Swaps' && (
          <SwapFeed swaps={swapsData?.swaps || []} />
        )}
        {tab === 'Positions' && (
          <PositionsList
            positions={posData?.positions || []}
            currentTick={pool.currentTick}
          />
        )}
        {tab === 'Liquidity Map' && (
          <LiquidityMap
            ticks={tickData?.ticks || []}
            currentTick={pool.currentTick}
            currentPrice={pool.priceRaw}
          />
        )}
      </div>
    </aside>
  );
}

function Metric({ label, value, change, badge }) {
  const isUp = change && (change.startsWith('+') || parseFloat(change) > 0);
  return (
    <div className="metric">
      <span className="metric__label">{label}</span>
      <span className="metric__value">{value}</span>
      {change && (
        <span className={`metric__change ${isUp ? 'up' : 'dn'}`}>{change}</span>
      )}
      {badge && <span className={`apr-badge apr-${badge?.toLowerCase().replace(' ','')}`}>{badge}</span>}
    </div>
  );
}

function SectionLabel({ children }) {
  return <div className="section-label">{children}</div>;
}

function PoolMetaTable({ pool }) {
  const rows = [
    ['Pool Address',   pool.id],
    ['Token 0',        `${pool.token0.symbol} (${pool.token0.address})`],
    ['Token 1',        `${pool.token1.symbol} (${pool.token1.address})`],
    ['Current Tick',   pool.currentTick?.toLocaleString()],
    ['Liquidity',      pool.liquidity],
    ['Total Txns',     pool.txCount],
    ['Vol/TVL Ratio',  pool.volTVLRatio],
  ];
  return (
    <table className="meta-table">
      <tbody>
        {rows.map(([k, v]) => (
          <tr key={k}><td className="meta-key">{k}</td><td className="meta-val">{v}</td></tr>
        ))}
      </tbody>
    </table>
  );
}

function Spinner() {
  return <div className="spinner" />;
}
```

## 4.7 Charts

```jsx
// client/src/components/charts/PriceChart.jsx
import React from 'react';
import { AreaChart, Area, XAxis, YAxis, Tooltip,
         ResponsiveContainer, ReferenceLine } from 'recharts';

export default function PriceChart({ data }) {
  const min = Math.min(...data.map(d => d.low));
  const max = Math.max(...data.map(d => d.high));
  const pad = (max - min) * 0.05;

  return (
    <ResponsiveContainer width="100%" height={200}>
      <AreaChart data={data} margin={{ top: 4, right: 8, bottom: 0, left: 0 }}>
        <defs>
          <linearGradient id="priceGrad" x1="0" y1="0" x2="0" y2="1">
            <stop offset="5%"  stopColor="#00e5a0" stopOpacity={0.25} />
            <stop offset="95%" stopColor="#00e5a0" stopOpacity={0} />
          </linearGradient>
        </defs>
        <XAxis dataKey="date" tick={{ fontSize: 10 }} tickLine={false}
               tickFormatter={d => d.slice(5)} interval="preserveStartEnd" />
        <YAxis domain={[min - pad, max + pad]} tick={{ fontSize: 10 }} tickLine={false}
               tickFormatter={v => v >= 1000 ? `${(v/1000).toFixed(1)}K` : v.toFixed(2)} />
        <Tooltip content={<PriceTooltip />} />
        <Area type="monotone" dataKey="close" stroke="#00e5a0" strokeWidth={2}
              fill="url(#priceGrad)" dot={false} />
      </AreaChart>
    </ResponsiveContainer>
  );
}

function PriceTooltip({ active, payload, label }) {
  if (!active || !payload?.length) return null;
  const d = payload[0].payload;
  return (
    <div className="chart-tooltip">
      <div className="tt-date">{label}</div>
      <div className="tt-row"><span>Open</span><span>{d.open?.toFixed(4)}</span></div>
      <div className="tt-row"><span>High</span><span>{d.high?.toFixed(4)}</span></div>
      <div className="tt-row"><span>Low</span> <span>{d.low?.toFixed(4)}</span></div>
      <div className="tt-row"><span>Close</span><span>{d.close?.toFixed(4)}</span></div>
      <div className="tt-divider"/>
      <div className="tt-row fee-apr"><span>Fee APR</span><span>{d.feeAPR?.toFixed(2)}%</span></div>
    </div>
  );
}
```

```jsx
// client/src/components/charts/LiquidityMap.jsx
import React from 'react';
import { BarChart, Bar, XAxis, YAxis, Tooltip,
         ResponsiveContainer, ReferenceLine, Cell } from 'recharts';

export default function LiquidityMap({ ticks, currentTick, currentPrice }) {
  if (!ticks.length) return <EmptyState message="No tick data" />;

  // Build cumulative liquidity curve around current tick
  const WINDOW = 200;   // ticks either side to show
  const inWindow = ticks.filter(t =>
    Math.abs(t.tick - currentTick) <= WINDOW
  );

  let cumLiq = 0;
  const chartData = inWindow.map(t => {
    cumLiq += parseInt(t.liquidityNet);
    return {
      tick:  t.tick,
      price: t.price?.toFixed(4),
      liq:   Math.abs(cumLiq) / 1e15,   // scale for readability
      net:   parseInt(t.liquidityNet),
      isNearCurrent: Math.abs(t.tick - currentTick) < 10,
    };
  });

  return (
    <div className="liquidity-map">
      <div className="lmap-label">
        Liquidity concentration around current price ({currentPrice?.toFixed(4)})
      </div>
      <ResponsiveContainer width="100%" height={180}>
        <BarChart data={chartData} barCategoryGap="0%">
          <XAxis dataKey="price" tick={{ fontSize: 9 }} tickLine={false}
                 interval={Math.floor(chartData.length / 6)} />
          <YAxis hide />
          <Tooltip content={<LiqTooltip />} />
          <ReferenceLine x={currentTick} stroke="#00e5a0" strokeDasharray="3 3"
                         label={{ value: 'Current', fill: '#00e5a0', fontSize: 10 }} />
          <Bar dataKey="liq" radius={[2, 2, 0, 0]}>
            {chartData.map((entry, i) => (
              <Cell key={i}
                fill={entry.isNearCurrent ? '#00e5a0'
                    : entry.tick < currentTick ? '#3b82f6' : '#8b5cf6'}
                opacity={0.8}
              />
            ))}
          </Bar>
        </BarChart>
      </ResponsiveContainer>
      <div className="lmap-legend">
        <span className="legend-dot" style={{ background: '#3b82f6' }} /> Below price
        <span className="legend-dot" style={{ background: '#00e5a0' }} /> Current range
        <span className="legend-dot" style={{ background: '#8b5cf6' }} /> Above price
      </div>
    </div>
  );
}

function LiqTooltip({ active, payload }) {
  if (!active || !payload?.length) return null;
  const d = payload[0].payload;
  return (
    <div className="chart-tooltip">
      <div className="tt-row"><span>Price</span><span>{d.price}</span></div>
      <div className="tt-row"><span>Tick</span><span>{d.tick}</span></div>
      <div className="tt-row"><span>Liquidity</span><span>{d.liq?.toFixed(2)}P</span></div>
    </div>
  );
}

function EmptyState({ message }) {
  return <div className="empty-state">{message}</div>;
}
```

## 4.8 Swap Feed

```jsx
// client/src/components/SwapFeed.jsx
import React from 'react';

export default function SwapFeed({ swaps }) {
  if (!swaps.length) return <div className="empty-state">No recent swaps</div>;

  return (
    <div className="swap-feed">
      <div className="feed-header">
        <span>Type</span><span>Size</span><span>Age</span>
        <span>Sender</span><span>Tx</span>
      </div>
      {swaps.map(swap => (
        <div key={swap.id} className={`swap-row swap-row--${swap.type.toLowerCase()}`}>
          <span className={`swap-type swap-type--${swap.type.toLowerCase()}`}>
            {swap.type === 'Buy' ? '▲' : '▼'} {swap.type}
          </span>
          <span className="swap-size">{swap.amountUSD}</span>
          <span className="swap-age">{swap.age}</span>
          <span className="swap-addr">{swap.sender}</span>
          <a className="swap-tx"
             href={`https://etherscan.io/tx/${swap.txHash}`}
             target="_blank" rel="noreferrer">
            {swap.txShort} ↗
          </a>
        </div>
      ))}
    </div>
  );
}
```

## 4.9 Positions List

```jsx
// client/src/components/PositionsList.jsx
import React from 'react';

export default function PositionsList({ positions, currentTick }) {
  return (
    <div className="positions-list">
      {positions.map(pos => {
        const inRange = currentTick >= pos.range.tickLower &&
                        currentTick <= pos.range.tickUpper;
        return (
          <div key={pos.id} className={`position-card ${inRange ? 'in-range' : 'out-range'}`}>
            <div className="pos-header">
              <span className="pos-owner">{pos.owner}</span>
              <span className="pos-whale">{pos.whaleLabel}</span>
              <span className={`pos-status ${inRange ? 'active' : 'inactive'}`}>
                {inRange ? '● In Range' : '○ Out of Range'}
              </span>
            </div>
            <div className="pos-range">
              <span className="range-label">Range</span>
              <span className="range-val">
                {pos.range.priceLower} → {pos.range.priceUpper}
              </span>
              <span className="range-width">({pos.range.width} ticks wide)</span>
            </div>
            <div className="pos-metrics">
              <div className="pos-metric">
                <span>Liquidity</span>
                <span>{pos.liquidity}</span>
              </div>
              <div className="pos-metric">
                <span>Deposited</span>
                <span>{pos.deposited.token0} / {pos.deposited.token1}</span>
              </div>
              <div className="pos-metric">
                <span>Fees Collected</span>
                <span>{pos.collected.fees0} / {pos.collected.fees1}</span>
              </div>
              <div className="pos-metric">
                <span>Age</span>
                <span>{pos.age}</span>
              </div>
            </div>
          </div>
        );
      })}
    </div>
  );
}
```

---

# PART 5 — CSS DESIGN SYSTEM

```css
/* client/src/styles/dashboard.css */

/* ─── Tokens ─────────────────────────────────────────────── */
:root {
  --bg-base:     #080b10;
  --bg-surface:  #0e1420;
  --bg-elevated: #161e2e;
  --bg-hover:    #1c2840;

  --border:      #1e2d44;
  --border-faint:#141d2d;

  --text-primary: #e8f0fe;
  --text-secondary:#7a9cc8;
  --text-muted:   #3d5478;

  --accent:      #00e5a0;
  --accent-faint:#00e5a015;
  --accent-dim:  #00b87f;

  --bull:   #00e5a0;
  --bear:   #f0534a;
  --bull-bg:#00e5a012;
  --bear-bg:#f0534a12;
  --purple: #8b5cf6;
  --blue:   #3b82f6;

  --font-mono: 'JetBrains Mono', 'Fira Code', monospace;
  --font-sans: 'DM Sans', 'Outfit', system-ui, sans-serif;
  --font-display: 'Space Grotesk', var(--font-sans);

  --radius:   8px;
  --radius-lg:14px;
  --shadow:   0 4px 24px #00000060;
}

* { box-sizing: border-box; margin: 0; padding: 0; }

body {
  background: var(--bg-base);
  color: var(--text-primary);
  font-family: var(--font-sans);
  font-size: 14px;
  line-height: 1.5;
  -webkit-font-smoothing: antialiased;
}

/* ─── Layout ─────────────────────────────────────────────── */
.dashboard {
  display: grid;
  grid-template-rows: 56px auto 1fr;
  grid-template-columns: 1fr;
  min-height: 100vh;
}
.dashboard--drawer-open {
  grid-template-columns: 1fr 440px;
}

/* ─── Header ─────────────────────────────────────────────── */
.dashboard__header {
  grid-column: 1 / -1;
  display: flex;
  align-items: center;
  gap: 24px;
  padding: 0 24px;
  background: var(--bg-surface);
  border-bottom: 1px solid var(--border);
}
.header__brand { display: flex; align-items: baseline; gap: 8px; }
.brand-mark    { color: var(--accent); font-size: 20px; }
.brand-name    { font-family: var(--font-display); font-weight: 700; font-size: 18px; }
.brand-sub     { color: var(--text-secondary); font-size: 11px; letter-spacing: 0.08em; }

.version-toggle { display: flex; gap: 4px; margin-left: auto; }
.version-btn {
  padding: 4px 14px;
  border-radius: 20px;
  border: 1px solid var(--border);
  background: transparent;
  color: var(--text-secondary);
  cursor: pointer;
  font-size: 12px;
  font-family: var(--font-mono);
  transition: all 0.15s;
}
.version-btn:hover  { border-color: var(--accent); color: var(--accent); }
.version-btn.active { background: var(--accent-faint); border-color: var(--accent); color: var(--accent); }

.live-indicator {
  display: flex; align-items: center; gap: 6px;
  color: var(--text-secondary); font-size: 12px;
}
.live-dot {
  width: 7px; height: 7px;
  border-radius: 50%;
  background: var(--accent);
  box-shadow: 0 0 8px var(--accent);
  animation: pulse 2s infinite;
}
@keyframes pulse {
  0%, 100% { opacity: 1; } 50% { opacity: 0.4; }
}

/* ─── Protocol Bar ───────────────────────────────────────── */
.protocol-bar {
  grid-column: 1 / -1;
  display: flex;
  gap: 0;
  background: var(--bg-surface);
  border-bottom: 1px solid var(--border);
  overflow-x: auto;
}
.protocol-stat {
  display: flex; align-items: center; gap: 10px;
  padding: 10px 24px;
  border-right: 1px solid var(--border-faint);
  white-space: nowrap;
}
.stat-icon    { color: var(--text-muted); font-size: 16px; }
.stat-label   { display: block; font-size: 10px; color: var(--text-muted); letter-spacing: 0.06em; text-transform: uppercase; }
.stat-value   { display: block; font-family: var(--font-mono); font-size: 15px; font-weight: 600; }
.stat-change  { font-size: 11px; font-family: var(--font-mono); }
.stat-change.up { color: var(--bull); }
.stat-change.dn { color: var(--bear); }

/* ─── Pool Table ─────────────────────────────────────────── */
.dashboard__main  { padding: 20px 24px; overflow: auto; }
.pool-table-wrap  { border-radius: var(--radius-lg); overflow: hidden; border: 1px solid var(--border); }
.pool-table       { width: 100%; border-collapse: collapse; }
.pool-table thead { background: var(--bg-elevated); }
.pool-table th    {
  padding: 10px 14px;
  text-align: left;
  font-size: 11px;
  font-weight: 600;
  letter-spacing: 0.06em;
  text-transform: uppercase;
  color: var(--text-muted);
  user-select: none;
  white-space: nowrap;
}
.th-sortable { cursor: pointer; }
.th-sortable:hover, .th-sortable.active { color: var(--text-secondary); }
.sort-arrow   { color: var(--text-muted); font-size: 10px; }

.pool-row {
  background: var(--bg-surface);
  border-bottom: 1px solid var(--border-faint);
  cursor: pointer;
  transition: background 0.1s;
}
.pool-row:hover    { background: var(--bg-hover); }
.pool-row.selected { background: var(--accent-faint); }
.pool-row td       { padding: 12px 14px; vertical-align: middle; }

.pair-name  { font-weight: 600; font-size: 14px; }
.pair-sub   { font-size: 11px; color: var(--text-muted); display: flex; gap: 6px; margin-top: 2px; }

.fee-badge {
  display: inline-block;
  padding: 2px 8px;
  border-radius: 4px;
  background: var(--bg-elevated);
  border: 1px solid var(--border);
  font-family: var(--font-mono);
  font-size: 11px;
  color: var(--text-secondary);
}
.hook-badge {
  padding: 1px 6px; border-radius: 3px;
  background: #f59e0b20; color: #f59e0b;
  font-size: 10px; font-weight: 600;
}

.val-primary { display: block; font-family: var(--font-mono); font-size: 13px; }
.val-change  { font-size: 11px; font-family: var(--font-mono); }
.val-change.up { color: var(--bull); }
.val-change.dn { color: var(--bear); }

.apr-badge {
  display: inline-block;
  padding: 2px 8px; border-radius: 4px;
  font-family: var(--font-mono); font-size: 12px; font-weight: 600;
}
.apr-badge.apr-very-high { background:#00e5a020; color:var(--bull); }
.apr-badge.apr-high      { background:#00c88020; color:#00c880; }
.apr-badge.apr-good      { background:#3b82f620; color:var(--blue); }
.apr-badge.apr-moderate  { background:#8b5cf620; color:var(--purple); }
.apr-badge.apr-low       { background:var(--bg-elevated); color:var(--text-muted); }

.price-arrow { font-size: 11px; display: block; font-family: var(--font-mono); }
.price-arrow.up { color: var(--bull); }
.price-arrow.down { color: var(--bear); }

/* ─── Skeleton ───────────────────────────────────────────── */
.skeleton-wrap { padding: 8px 0; }
.skeleton-row {
  height: 52px; margin: 2px 0;
  background: linear-gradient(90deg, var(--bg-surface) 25%, var(--bg-elevated) 50%, var(--bg-surface) 75%);
  background-size: 200% 100%;
  animation: shimmer 1.4s infinite;
  border-radius: 4px;
}
@keyframes shimmer { 0%{background-position:200% 0} 100%{background-position:-200% 0} }

/* ─── Drawer ─────────────────────────────────────────────── */
.drawer {
  grid-row: 2 / -1;
  background: var(--bg-surface);
  border-left: 1px solid var(--border);
  overflow-y: auto;
  display: flex; flex-direction: column;
}
.drawer--loading { align-items: center; justify-content: center; }

.drawer__header {
  display: flex; align-items: center; justify-content: space-between;
  padding: 16px 20px;
  border-bottom: 1px solid var(--border);
  position: sticky; top: 0;
  background: var(--bg-surface);
  z-index: 10;
}
.drawer__title  { display: flex; align-items: center; gap: 10px; }
.drawer__pair   { font-family: var(--font-display); font-weight: 700; font-size: 18px; }
.drawer__close  { background: none; border: none; color: var(--text-muted); cursor: pointer; font-size: 18px; padding: 4px 8px; }
.drawer__close:hover { color: var(--text-primary); }

.drawer__metrics {
  display: grid; grid-template-columns: 1fr 1fr;
  gap: 1px; background: var(--border);
  border-bottom: 1px solid var(--border);
}
.metric {
  background: var(--bg-surface);
  padding: 12px 16px;
  display: flex; flex-direction: column; gap: 2px;
}
.metric__label  { font-size: 10px; text-transform: uppercase; letter-spacing: 0.06em; color: var(--text-muted); }
.metric__value  { font-family: var(--font-mono); font-size: 16px; font-weight: 600; }
.metric__change { font-size: 11px; font-family: var(--font-mono); }
.metric__change.up { color: var(--bull); }
.metric__change.dn { color: var(--bear); }

.drawer__tabs {
  display: flex; gap: 0;
  border-bottom: 1px solid var(--border);
  background: var(--bg-elevated);
}
.tab-btn {
  padding: 10px 16px;
  background: none; border: none;
  color: var(--text-muted); cursor: pointer;
  font-size: 12px; font-weight: 500;
  border-bottom: 2px solid transparent;
  transition: all 0.15s;
}
.tab-btn:hover  { color: var(--text-secondary); }
.tab-btn.active { color: var(--accent); border-bottom-color: var(--accent); }

.drawer__content  { padding: 16px; flex: 1; }
.section-label    {
  font-size: 10px; text-transform: uppercase; letter-spacing: 0.08em;
  color: var(--text-muted); margin: 16px 0 8px;
}
.section-label:first-child { margin-top: 0; }

/* ─── Charts ─────────────────────────────────────────────── */
.chart-tooltip {
  background: var(--bg-elevated); border: 1px solid var(--border);
  border-radius: var(--radius); padding: 10px 14px;
  font-size: 12px; font-family: var(--font-mono);
  box-shadow: var(--shadow);
}
.tt-date   { color: var(--text-muted); margin-bottom: 6px; font-size: 11px; }
.tt-row    { display: flex; justify-content: space-between; gap: 20px; padding: 1px 0; }
.tt-row span:last-child { color: var(--text-primary); font-weight: 600; }
.tt-divider { border-top: 1px solid var(--border); margin: 6px 0; }
.tt-row.fee-apr span:last-child { color: var(--accent); }

/* ─── Liquidity Map ──────────────────────────────────────── */
.liquidity-map   { }
.lmap-label      { font-size: 11px; color: var(--text-muted); margin-bottom: 8px; }
.lmap-legend     { display: flex; gap: 16px; font-size: 11px; color: var(--text-muted); margin-top: 8px; }
.legend-dot      { display: inline-block; width: 8px; height: 8px; border-radius: 50%; margin-right: 4px; }

/* ─── Swap Feed ──────────────────────────────────────────── */
.swap-feed   { font-family: var(--font-mono); font-size: 12px; }
.feed-header {
  display: grid; grid-template-columns: 60px 80px 70px 100px 1fr;
  padding: 6px 8px; color: var(--text-muted);
  border-bottom: 1px solid var(--border);
  font-size: 10px; text-transform: uppercase; letter-spacing: 0.06em;
}
.swap-row {
  display: grid; grid-template-columns: 60px 80px 70px 100px 1fr;
  padding: 7px 8px; align-items: center;
  border-bottom: 1px solid var(--border-faint);
  transition: background 0.1s;
}
.swap-row:hover          { background: var(--bg-elevated); }
.swap-row--buy  { border-left: 2px solid var(--bull); }
.swap-row--sell { border-left: 2px solid var(--bear); }
.swap-type--buy  { color: var(--bull); }
.swap-type--sell { color: var(--bear); }
.swap-size  { font-weight: 600; }
.swap-age   { color: var(--text-muted); }
.swap-addr  { color: var(--text-secondary); }
.swap-tx    { color: var(--accent); text-decoration: none; }
.swap-tx:hover { text-decoration: underline; }

/* ─── Positions ──────────────────────────────────────────── */
.position-card {
  background: var(--bg-elevated); border-radius: var(--radius);
  border: 1px solid var(--border);
  padding: 14px; margin-bottom: 8px;
}
.position-card.in-range  { border-left: 3px solid var(--bull); }
.position-card.out-range { border-left: 3px solid var(--text-muted); }

.pos-header {
  display: flex; align-items: center; gap: 8px;
  margin-bottom: 10px; flex-wrap: wrap;
}
.pos-owner  { font-family: var(--font-mono); font-size: 12px; color: var(--text-secondary); }
.pos-whale  { font-size: 12px; }
.pos-status { margin-left: auto; font-size: 11px; }
.pos-status.active   { color: var(--bull); }
.pos-status.inactive { color: var(--text-muted); }

.pos-range {
  display: flex; align-items: center; gap: 8px;
  padding: 8px; background: var(--bg-surface); border-radius: 4px;
  margin-bottom: 10px; font-family: var(--font-mono); font-size: 12px;
}
.range-label { color: var(--text-muted); font-size: 10px; text-transform: uppercase; }
.range-val   { color: var(--text-primary); font-weight: 600; }
.range-width { color: var(--text-muted); font-size: 11px; }

.pos-metrics {
  display: grid; grid-template-columns: 1fr 1fr;
  gap: 6px;
}
.pos-metric {
  display: flex; flex-direction: column; gap: 2px;
}
.pos-metric span:first-child { font-size: 10px; color: var(--text-muted); text-transform: uppercase; letter-spacing: 0.05em; }
.pos-metric span:last-child  { font-family: var(--font-mono); font-size: 12px; }

/* ─── Meta Table ─────────────────────────────────────────── */
.meta-table     { width: 100%; border-collapse: collapse; font-size: 12px; margin-top: 12px; }
.meta-table td  { padding: 7px 8px; border-bottom: 1px solid var(--border-faint); }
.meta-key       { color: var(--text-muted); width: 40%; font-size: 11px; text-transform: uppercase; letter-spacing: 0.05em; }
.meta-val       { font-family: var(--font-mono); color: var(--text-primary); word-break: break-all; }

/* ─── Misc ───────────────────────────────────────────────── */
.empty-state { color: var(--text-muted); text-align: center; padding: 32px; font-size: 13px; }
.error-banner {
  background: #f0534a15; border: 1px solid #f0534a40;
  color: #f0534a; padding: 10px 16px; border-radius: var(--radius);
  margin-bottom: 16px; font-size: 13px;
}
.spinner {
  width: 32px; height: 32px; border-radius: 50%;
  border: 3px solid var(--border);
  border-top-color: var(--accent);
  animation: spin 0.8s linear infinite;
}
@keyframes spin { to { transform: rotate(360deg); } }
```

---

# PART 6 — RUNNING THE FULL STACK

## Start Both Servers

```bash
# Terminal 1 — backend
cd server
node src/index.js
# → UniClaw API running on :4000

# Terminal 2 — frontend
cd client
REACT_APP_API_URL=http://localhost:4000/api npm start
# → React app on http://localhost:3000
```

## Verify the Pipeline

```bash
# 1. Health check
curl http://localhost:4000/api/health

# 2. Top v3 pools
curl "http://localhost:4000/api/pools?version=v3&limit=5"

# 3. Pool detail (WETH/USDC 0.05% pool)
curl "http://localhost:4000/api/pools/0x88e6a0c2ddd26feeb64f039a2c41296fcb3f5640"

# 4. Recent swaps
curl "http://localhost:4000/api/pools/0x88e6a0c2ddd26feeb64f039a2c41296fcb3f5640/swaps?count=10"

# 5. Tick distribution
curl "http://localhost:4000/api/pools/0x88e6a0c2ddd26feeb64f039a2c41296fcb3f5640/ticks"
```

## Human-Readable Output Checklist

Every piece of raw data must pass through the formatter before leaving the API.
Run this audit before shipping:

```
[ ] All USD values: $1.82M not 1823748291.929
[ ] All percentages: +3.42% not 0.034199
[ ] All prices: 3,210.45 not 3210.4500000001
[ ] All timestamps: "3h ago" not 1739800000
[ ] All addresses: 0x1234...abcd not full 42-char string
[ ] All large integers: 1.2T not 1200000000000000000
[ ] APR label: "High" badge alongside number
[ ] Swap direction: "▲ Buy" / "▼ Sell" with colour
[ ] Position status: "● In Range" / "○ Out of Range"
[ ] Whale classification: 🐋/🐬/🐟 label on positions
[ ] Hooks flag: ⚡ badge on v4 pools with non-zero hook address
[ ] Price trend: direction arrow + percentage change
[ ] Fee APR label: Very High / High / Good / Moderate / Low
```

---

# PART 7 — PATTERNS TO KNOW

## Refresh Strategy by Data Type

| Data | Stale After | Strategy |
|------|------------|----------|
| Pool list (TVL rank) | 60s | `refetchInterval: 60_000` |
| Pool detail + charts | 30s | `refetchInterval: 30_000` |
| Swap feed | 15s | `refetchInterval: 15_000` |
| Tick distribution | 2min | `refetchInterval: 120_000` |
| Protocol totals | 2min | `refetchInterval: 120_000` |
| Position data | 30s | `refetchInterval: 30_000` |
| Token prices | 20s | `refetchInterval: 20_000` |

## API Key Security Rules

```
1. Keys live in server/.env only — never in client code, never in git
2. Client talks to YOUR server — never directly to The Graph or Uniswap API
3. Your server is the only place that holds keys
4. Add rate limiting on your own API — protects both your keys and your quota
5. Use CORS — only your frontend domain can call your server
6. Rotate keys if you accidentally commit them (they cost money if stolen)
7. Log API calls server-side — detect anomalous usage early
```

## Common Errors and Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `GRAPH_API_KEY not set` | Missing .env | Copy .env.example, fill key |
| `CORS blocked` | Wrong ALLOWED_ORIGIN | Match your frontend URL exactly |
| `429 Too Many Requests` | Rate limit hit | Increase cache TTL, reduce refetch |
| `Pool not found` | ID case mismatch | Always `.toLowerCase()` pool IDs |
| `Null data` | Subgraph not synced | Check `_meta.block.number` vs latest |
| `Field not found` | v2 vs v3 schema confusion | Use `currency0` for v4, `token0` for v3 |
| `Chart renders empty` | Data not sorted by date | Sort by `date` ASC before charting |

---

## Files to Commit After This Workshop

```
server/
  src/index.js
  src/config/endpoints.js
  src/graphql/client.js
  src/graphql/queries.js
  src/formatters/index.js
  src/routes/pools.js
  src/routes/tokens.js
  src/routes/protocol.js
  .env.example              ← keys as placeholders, safe to commit
  .gitignore                ← .env must be in here

client/
  src/api/client.js
  src/components/PoolTable.jsx
  src/components/PoolDrawer.jsx
  src/components/ProtocolBar.jsx
  src/components/SwapFeed.jsx
  src/components/PositionsList.jsx
  src/components/charts/PriceChart.jsx
  src/components/charts/VolumeChart.jsx
  src/components/charts/LiquidityMap.jsx
  src/styles/dashboard.css
```

---

*UniClaw Workshop 04 | https://github.com/ylintern/uniclawskills | MIT License*
