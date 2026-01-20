# Code Challenge: Greeks Historical API — Data Architecture & System Design

## Overview

We are building a **Historical Greeks Exposure API** — a service that provides time-series data of options market greek exposures (gamma, vanna, charm) at the strike level, at minute-by-minute granularity.

This data is valuable for quantitative traders and researchers who want to:
- Backtest strategies based on dealer positioning
- Analyze how greek exposures correlate with price moves
- Study market microstructure around options expiration

**We are already collecting this data in near real-time.** Your challenge is to design and prototype the **storage layer and API** that makes this data queryable and sellable as a product.

This is primarily a **data architecture and systems design challenge**, not a scraping challenge.

---

## Background: What Are Greeks Exposures?

If you're not familiar with options greeks, here's the minimum context you need:

### The Basics

Options are financial contracts. Market makers (dealers) hold large inventories of these contracts, which exposes them to various risks. These risks are quantified as "greeks":

- **Gamma**: How much the dealer's directional exposure changes as price moves
- **Vanna**: How gamma changes as volatility changes  
- **Charm**: How gamma decays over time

### Why This Matters

When dealers are heavily exposed (e.g., large negative gamma), they must hedge by trading the underlying asset. This hedging activity can amplify or dampen price moves. Traders who understand dealer positioning can anticipate these flows.

### The Data We Collect

For each **symbol** (e.g., SPY, QQQ), at each **strike price** (e.g., $580, $585, $590...), across all active future **expirations** (e.g., Jan 19, Jan 24, Feb 21...), we calculate and store:

```
{
  "timestamp": "2025-01-15T14:32:00Z",
  "symbol": "SPY",
  "expiration": "2025-01-19",
  "strike": 585.0,
  "gamma_exposure": -142500000,   // in dollars
  "vanna_exposure": 23400000,
  "charm_exposure": -8900000
}
```

This is recorded **every minute** during market hours (9:30 AM - 4:00 PM ET = 390 minutes/day).

---

## The Data Dimensions

Here's where it gets interesting. Let's do the math:

### Per Symbol (e.g., SPY)

| Dimension | Typical Count |
|-----------|---------------|
| Active expirations | 8-12 (weeklies + monthlies + LEAPs) |
| Strikes with meaningful OI per expiration | 200-400 |
| Greek values per strike | 3 (gamma, vanna, charm) |
| Minutes per trading day | 390 |
| Trading days per year | 252 |

### Napkin Math

Conservative estimate for **one symbol**:

```
10 expirations × 300 strikes × 3 greeks × 390 minutes = 3.5M data points/day
```

Per year: **~880 million data points per symbol**

For SPY + QQQ: **~1.75 billion data points/year**

If we expand to ES, NQ, and other liquid names: **5-10 billion+ data points/year**

### Storage Estimate

Each data point might look like:
- Timestamp: 8 bytes
- Symbol: 4 bytes (or foreign key)
- Expiration: 4 bytes (date)
- Strike: 4 bytes (float)
- Gamma: 8 bytes (double)
- Vanna: 8 bytes (double)  
- Charm: 8 bytes (double)

**~44 bytes per row raw**, but with overhead, indexes, etc., budget **60-100 bytes/row**.

For 2 symbols, 1 year: **~100-175 GB uncompressed**

This is manageable but non-trivial. Naive approaches will be slow and expensive.

---

## The Challenge

Design and prototype a system that:

1. **Stores this data efficiently**
2. **Serves it via an API with reasonable query performance**
3. **Scales to multiple years of history**
4. **Is cost-effective to operate**

---

## Requirements

### Part 1: Data Architecture Design (Primary Deliverable)

Produce a **design document** that addresses:

#### Storage Strategy

- What database/storage technology would you use? Why?
- Row-store (Postgres, MySQL) vs. column-store (ClickHouse, TimescaleDB, DuckDB) vs. object storage (Parquet on S3)?
- How would you partition the data? By date? By symbol? By expiration?
- What indexes would you create? What are the trade-offs?
- How would you handle compression?

#### Query Patterns

The API needs to support queries like:

```
# Get all strikes for SPY on a specific day
GET /api/v1/greeks/SPY/2025-01-15

# Get a specific strike's time series
GET /api/v1/greeks/SPY/2025-01-15?strike=585&expiration=2025-01-17

# Get aggregate exposure (sum across all strikes) 
GET /api/v1/greeks/SPY/2025-01-15/aggregate

# Get data for a time range within a day
GET /api/v1/greeks/SPY/2025-01-15?start_time=14:00&end_time=15:00
```

For each query pattern:
- What's the expected data volume returned?
- How would you optimize for it?
- What's the expected latency?

#### Data Lifecycle

- How do you handle data ingestion? (We push ~3.5M rows/day per symbol)
- Hot vs. warm vs. cold storage tiers?
- Retention policy considerations?

#### Cost Analysis

- Rough estimate of storage costs (pick a cloud provider)
- Rough estimate of compute costs for typical query load
- What are the cost drivers? How would you optimize?

### Part 2: Prototype Implementation (Secondary Deliverable)

Build a working prototype that demonstrates your architecture:

1. **Schema/table creation** for your chosen storage solution
2. **Data loader** that can ingest sample data (we'll provide a sample dataset, or you can generate synthetic data)
3. **API endpoints** for at least 2 of the query patterns above
4. **Performance benchmarks** — show query times with realistic data volumes

The prototype doesn't need to be production-ready, but it should prove your design works.

### Part 3: Pitfalls & Failure Modes

Document what could go wrong and how you'd handle it:

- What happens if ingestion falls behind?
- What if a query requests too much data? (e.g., full year, all strikes)
- How do you handle schema evolution? (e.g., we add a new greek)
- What if the data has gaps or errors?
- Backup and disaster recovery considerations?

---

## Technical Constraints

### What We're Currently Running

- **Current storage**: Raw minute-by-minute data is being written to [assume Postgres or flat files — you can propose migration]
- **Infrastructure**: We're cloud-agnostic but lean toward AWS or a simple VPS setup
- **Budget**: This is a small SaaS, not a hedge fund. Cost-efficiency matters.

### API Requirements

- **Auth**: Simple API key (UUID), no rate limiting needed initially
- **Response format**: JSON
- **Latency target**: < 2 seconds for single-day queries, < 10 seconds for bulk queries
- **Availability**: 99% is fine (this is historical data, not trading-critical)

---

## Sample Data Structure

Here's what a single minute's data might look like for one expiration:

```json
{
  "timestamp": "2025-01-15T14:32:00-05:00",
  "symbol": "SPY",
  "expiration": "2025-01-17",
  "strikes": [
    { "strike": 580.0, "gamma": -52000000, "vanna": 12000000, "charm": -4500000 },
    { "strike": 581.0, "gamma": -48000000, "vanna": 11200000, "charm": -4200000 },
    { "strike": 582.0, "gamma": -43000000, "vanna": 10100000, "charm": -3800000 },
    // ... 200-400 more strikes
    { "strike": 620.0, "gamma": -12000000, "vanna": 3400000, "charm": -1100000 }
  ]
}
```

For the prototype, you can either:
- Generate synthetic data that matches this structure
- Request a sample dataset from us

---

## Suggested Approaches (Not Prescriptive)

### Option A: TimescaleDB (Postgres Extension)

- Familiar Postgres interface
- Good compression for time-series
- Hypertables with automatic partitioning
- Might struggle at 10B+ rows

### Option B: ClickHouse

- Purpose-built for analytical queries on large datasets
- Excellent compression (often 10-20x)
- Fast aggregations
- Steeper learning curve, different operational model

### Option C: Parquet on S3 + DuckDB

- Cheapest storage (S3 pricing)
- Parquet gives excellent compression
- DuckDB can query directly, or use Athena
- Higher query latency, but might be acceptable

### Option D: Hybrid

- Hot data (last 30 days) in Postgres/TimescaleDB
- Cold data in Parquet/S3
- API layer abstracts the difference

We're genuinely open to your recommendation. Make the case for your choice.

---

## Deliverables

1. **Design Document** (3-5 pages)
   - Architecture overview with diagram
   - Storage technology choice with justification
   - Schema design
   - Query optimization strategy
   - Cost estimate
   - Pitfalls and mitigation strategies

2. **Prototype Code**
   - Working schema/migrations
   - Data loader (with synthetic data generator or sample ingestion)
   - At least 2 API endpoints
   - Basic performance benchmarks
   - README with setup instructions

3. **Benchmark Results**
   - Query times for each endpoint
   - Data volume used in testing
   - Any bottlenecks identified

---

## Evaluation Criteria

| Criteria | What We're Assessing |
|----------|---------------------|
| **Systems thinking** | Do you understand the scale and constraints? Do you see the trade-offs? |
| **Pragmatism** | Is your solution appropriately sized? Not over-engineered, not under-engineered? |
| **Cost awareness** | Can you estimate and optimize costs? This is a bootstrapped product. |
| **Query performance** | Do your design choices support the access patterns we need? |
| **Operational simplicity** | Can a small team run this? Or does it require a dedicated data eng team? |
| **Communication** | Is your design document clear? Can we understand your reasoning? |

We're **not** looking for:
- The most sophisticated possible solution
- Perfect handling of every edge case
- Production-ready code (prototype quality is fine)

We **are** looking for:
- Evidence you understand the problem
- Good judgment on trade-offs
- A solution we could actually build and operate

---

## Time Expectation

This challenge is designed to take approximately **8-12 hours**:

- ~3-4 hours on the design document
- ~4-6 hours on the prototype
- ~1-2 hours on benchmarking and polish

If you're spending significantly more time, focus on:
1. A thorough design document
2. A minimal but working prototype
3. Honest assessment of limitations

---

## Questions to Consider

As you work through this, think about:

- If a customer wants to download a full year of data, how do you handle that?
- What's the marginal cost of adding a third symbol? A tenth symbol?
- How would you handle a query like "show me all days where gamma exposure exceeded X"?
- If we wanted to add real-time streaming (not just historical), how would that change your design?
- What monitoring/alerting would you set up?

You don't need to solve all of these, but thinking about them will inform your design.

---

## Bonus Points (Optional)

If you finish early:

- Add a third query pattern (aggregations across multiple days)
- Prototype a data export endpoint (bulk CSV/Parquet download)
- Add basic caching layer
- Dockerize the solution
- Compare performance across two storage options

---

Good luck. We're excited to see how you think through this problem.
