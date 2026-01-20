# Code Challenge: AMT JOY Historical TPO API — Cleanup & Productionization

## Overview

We have an existing project called **AMT JOY** (https://amtjoy.com) that provides historical Market Profile / TPO (Time Price Opportunity) data for futures contracts. The core functionality works, but it grew organically as a hobby project and needs to be cleaned up before we can:

1. Offer it as a paid SaaS / API product
2. Expand from futures (NQ) to stock tickers (AAPL, TSLA, NVDA, etc.)

Your challenge is to **assess the current state, propose a cleanup plan, and prototype the improvements**.

This is a **refactoring and productionization challenge**, not a greenfield build.

---

## Background: What is Market Profile / TPO?

If you're not familiar with Market Profile, here's the context:

### The Concept

Market Profile is a charting technique developed by J. Peter Steidlmayer at the Chicago Board of Trade. Instead of traditional candlestick charts, it shows **how much time** price spent at each level during a trading session.

The trading day is divided into 30-minute periods, each labeled with a letter:
- A period: 9:30 - 10:00 AM
- B period: 10:00 - 10:30 AM
- C period: 10:30 - 11:00 AM
- ... and so on through the day

Each letter is called a **TPO (Time Price Opportunity)**. Stack them up and you get a profile that shows where the market spent most of its time.

### Key Metrics Derived from TPO

| Metric | Description |
|--------|-------------|
| **Value Area High (VAH)** | Upper bound of where 70% of trading occurred |
| **Value Area Low (VAL)** | Lower bound of where 70% of trading occurred |
| **Point of Control (POC)** | The single price with the most TPOs (most time spent) |
| **Initial Balance (IB)** | The range established in A and B periods (first hour) |
| **Session Type** | Classification: trend, normal, normal variation, neutral, etc. |
| **Single Prints** | Price levels visited only once (potential future targets) |
| **Poor High/Low** | Weak extremes that may get revisited |
| **Excess** | Strong rejection at highs/lows |

### Why Traders Care

- **Value Area** acts as support/resistance
- **Session Type** tells you whether to trade with the trend or fade the range
- **Single Prints** are magnets for future price action
- Futures traders have used this for decades; stock traders are increasingly adopting it

### Visual Example

A TPO chart might look like this (simplified):

```
Price   | TPOs
--------|------------------
5920    | A
5919    | AB
5918    | ABC
5917    | ABCD
5916    | ABCDE      ← IB High
5915    | ABCDEF
5914    | ABCDEFG    ← POC (most time spent here)
5913    | ABCDEFG
5912    | ABCDEF
5911    | ABCDE      ← IB Low
5910    | ABCD
5909    | ABC
5908    | AB
5907    | A
```

---

## Current State of AMT JOY

### What Exists

We have a working system for **NQ (Nasdaq 100 futures)** that:

- Calculates TPO profiles from raw price data
- Derives all the key metrics (VAH, VAL, POC, IB, session type, etc.)
- Stores results as JSON files
- Serves data via an API

You can see the live product at: **https://amtjoy.com**

### The Problems

The implementation has issues typical of a hobby project that grew organically:

1. **Inconsistent data storage**
   - JSON files, not a proper database
   - File naming conventions vary
   - Some days missing, some duplicated

2. **Calculation inconsistencies**
   - Early calculations had bugs that were later fixed
   - Some historical data was calculated with old (incorrect) logic
   - No easy way to identify which records need recalculation

3. **Hardcoded for futures**
   - Assumes futures market hours (overnight + RTH sessions)
   - Tick sizes and price increments are hardcoded
   - Adding stocks would require significant changes

4. **No data validation**
   - Bad data gets stored without error
   - No checksums or integrity verification
   - Hard to know if a calculation is trustworthy

5. **Fragile API**
   - Reads directly from JSON files
   - No caching strategy
   - Would not scale to many symbols

### Sample of Current JSON Structure

```json
{
  "symbol": "NQ",
  "date": "2025-01-15",
  "session_type": "trend",
  "bias": "up",
  "value_area_high": 21485.50,
  "value_area_low": 21342.25,
  "poc": 21415.75,
  "initial_balance_high": 21398.00,
  "initial_balance_low": 21342.25,
  "session_high": 21523.75,
  "session_low": 21298.50,
  "single_prints": [
    [21478.00, 21485.50],
    [21298.50, 21315.25]
  ],
  "poor_high": false,
  "poor_low": true,
  "excess_high": true,
  "excess_low": false,
  "tpo_structure": {
    "A": [21375.00, 21342.25],
    "B": [21398.00, 21358.50],
    "C": [21425.25, 21385.75],
    // ... through M period
  },
  "volume": 892451,
  "cash_session_return": 0.0082
}
```

This structure is mostly fine — the problem is how it's generated and stored.

---

## The Challenge

### Part 1: Assessment & Cleanup Plan (Primary Deliverable)

Review the current implementation (we'll provide access to the codebase and sample data) and produce:

#### 1. Current State Assessment

- Inventory of what exists and what's broken
- Data quality analysis: How many sessions are valid? How many need recalculation?
- Code quality observations: What's salvageable? What needs rewriting?

#### 2. Migration Plan

Propose how to:

- Move from JSON files to a proper database
- Identify and recalculate bad historical data
- Preserve valid data during migration
- Validate data integrity

#### 3. Database Schema Design

Design a schema that:

- Stores all TPO metrics efficiently
- Supports multiple symbols (futures AND stocks)
- Handles different market hours (futures: 23 hours, stocks: 6.5 hours)
- Handles different tick sizes (NQ: 0.25, AAPL: 0.01)
- Makes common queries fast

#### 4. Stock Expansion Strategy

Document how the system needs to change to support stocks:

- What assumptions are currently baked in that need to change?
- How do you handle pre-market/after-hours for stocks? (Most traders only care about RTH)
- How do you source the underlying price data for stocks?
- Which stocks would you prioritize? (Hint: liquidity and options volume matter)

### Part 2: Prototype Implementation (Secondary Deliverable)

Build a working prototype that demonstrates:

1. **Database schema** (PostgreSQL preferred, but justify alternatives)

2. **Data migration script** that:
   - Reads existing JSON files
   - Validates the data
   - Flags suspicious records
   - Inserts into the new database

3. **API endpoints** matching the existing contract:

```
GET /api/v1/tpo/{symbol}/{date}
GET /api/v1/tpo/{symbol}/range?start=2025-01-01&end=2025-01-15
GET /api/v1/tpo/{symbol}/latest
```

4. **Symbol configuration system** that handles:
   - Different tick sizes
   - Different market hours
   - Futures vs stocks flag

### Part 3: Data Quality Framework

Propose (and ideally prototype) a system for ongoing data quality:

- How do you validate a TPO calculation is correct?
- What sanity checks should run on ingestion?
- How do you handle late corrections? (e.g., exchange revises volume data)
- How do you track calculation version? (so you can re-run if logic changes)

---

## Key Differences: Futures vs Stocks

Your solution needs to handle these differences gracefully:

| Aspect | Futures (NQ, ES) | Stocks (AAPL, TSLA) |
|--------|------------------|---------------------|
| Market hours | Nearly 24h (6pm-5pm ET with break) | 9:30am-4pm ET |
| Tick size | 0.25 (NQ), 0.25 (ES) | 0.01 |
| TPO period | 30 min | 30 min |
| Overnight session | Yes, important | No (ignore pre/post market for TPO) |
| Data source | CME | NYSE/NASDAQ |
| Volume data | Reliable | Reliable |

The system should be configurable per symbol, not hardcoded.

---

## Suggested Database Schema (Starting Point)

This is a starting point, not a prescription. Critique and improve it.

```sql
-- Symbol configuration
CREATE TABLE symbols (
    id SERIAL PRIMARY KEY,
    symbol VARCHAR(20) UNIQUE NOT NULL,
    name VARCHAR(100),
    asset_type VARCHAR(20) NOT NULL,  -- 'future', 'stock', 'etf'
    tick_size DECIMAL(10, 4) NOT NULL,
    market_open TIME NOT NULL,        -- RTH open
    market_close TIME NOT NULL,       -- RTH close
    has_overnight BOOLEAN DEFAULT FALSE,
    timezone VARCHAR(50) DEFAULT 'America/New_York',
    active BOOLEAN DEFAULT TRUE
);

-- Daily TPO sessions
CREATE TABLE tpo_sessions (
    id SERIAL PRIMARY KEY,
    symbol_id INT REFERENCES symbols(id),
    session_date DATE NOT NULL,
    
    -- Session classification
    session_type VARCHAR(30),         -- trend, normal, normal_variation, neutral, etc.
    bias VARCHAR(10),                 -- up, down, neutral
    
    -- Core levels
    poc DECIMAL(12, 4),
    value_area_high DECIMAL(12, 4),
    value_area_low DECIMAL(12, 4),
    
    -- Initial balance
    ib_high DECIMAL(12, 4),
    ib_low DECIMAL(12, 4),
    
    -- Session extremes
    session_high DECIMAL(12, 4),
    session_low DECIMAL(12, 4),
    
    -- Session stats
    volume BIGINT,
    cash_session_return DECIMAL(8, 6),
    
    -- Quality flags
    poor_high BOOLEAN,
    poor_low BOOLEAN,
    excess_high BOOLEAN,
    excess_low BOOLEAN,
    
    -- Metadata
    calculation_version VARCHAR(20),  -- Track which algo version produced this
    calculated_at TIMESTAMP DEFAULT NOW(),
    
    UNIQUE(symbol_id, session_date)
);

-- TPO letter distribution (the actual profile shape)
CREATE TABLE tpo_periods (
    id SERIAL PRIMARY KEY,
    session_id INT REFERENCES tpo_sessions(id) ON DELETE CASCADE,
    period_letter CHAR(1) NOT NULL,   -- A, B, C, ... M
    price_high DECIMAL(12, 4) NOT NULL,
    price_low DECIMAL(12, 4) NOT NULL,
    
    UNIQUE(session_id, period_letter)
);

-- Single prints (ranges visited only once)
CREATE TABLE single_prints (
    id SERIAL PRIMARY KEY,
    session_id INT REFERENCES tpo_sessions(id) ON DELETE CASCADE,
    price_high DECIMAL(12, 4) NOT NULL,
    price_low DECIMAL(12, 4) NOT NULL
);

-- Indexes
CREATE INDEX idx_sessions_symbol_date ON tpo_sessions(symbol_id, session_date);
CREATE INDEX idx_sessions_date ON tpo_sessions(session_date);
```

Questions to consider:
- Is normalizing tpo_periods and single_prints worth it, or should they be JSONB columns?
- Do we need to store the full tick-by-tick TPO matrix, or just the derived metrics?
- How do you handle sessions that span midnight (futures)?

---

## Evaluation Criteria

| Criteria | What We're Assessing |
|----------|---------------------|
| **Assessment quality** | Can you accurately diagnose problems in existing code? |
| **Migration thinking** | Do you have a safe, incremental approach? No "big bang" rewrites? |
| **Schema design** | Is it flexible enough for stocks? Normalized appropriately? |
| **Data quality mindset** | Do you think about validation, versioning, integrity? |
| **Pragmatism** | Are you improving what exists, not rewriting for the sake of it? |
| **Communication** | Can you explain trade-offs clearly? |

We're **not** looking for:
- A complete rewrite of the calculation engine
- Perfect handling of every edge case
- Enterprise-grade infrastructure

We **are** looking for:
- Good judgment about what to keep vs. change
- A realistic migration path
- Evidence you've dealt with messy real-world data before

---

## Deliverables

1. **Solution Code**
   - Schema migrations
   - JSON → Database migration script with validation
   - API endpoints (3 minimum)
   - Symbol configuration system
   - README with setup instructions

---

## Time Expectation

This challenge is designed to take approximately **8-12 hours**:

- ~2-3 hours reviewing existing code and data
- ~2-3 hours on assessment and planning documents
- ~4-5 hours on prototype implementation
- ~1 hour on documentation and polish

Focus on demonstrating judgment over completeness. A thoughtful partial solution beats a rushed complete one.

---

## Questions?

If anything is unclear about the existing system or requirements, ask. Part of this challenge is gathering requirements effectively.

---

## What Happens Next

If your solution looks good, the next step would be:

1. Walk us through your design decisions
2. Discuss how you'd handle specific edge cases
3. Talk about what you'd prioritize for a v1 launch

We're looking for someone who can own this product end-to-end, not just write code to spec.

---

Good luck. We're excited to see how you approach cleaning up a real-world messy codebase.
