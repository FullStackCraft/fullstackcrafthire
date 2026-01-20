# Code Challenge: Historical Market Context API — Economic Calendar Scraper

## Overview

We are building a **Historical Market Context API** — a service that provides structured data about what happened on any given trading day: economic releases, Fed speakers, major market events, and other contextual information that explains *why* the market moved.

Think of it as answering the question every trader asks when backtesting: *"What the hell happened on March 15, 2020?"*

Your task is to build the **foundational data ingestion layer** — a reliable, extensible scraper that collects historical economic calendar data.

---

## Product Vision

The end product is an API that looks something like this:

```
GET /api/v1/context/2024-08-05
```

Response:
```json
{
  "date": "2024-08-05",
  "economic_events": [
    {
      "time": "10:00",
      "timezone": "America/New_York",
      "country": "US",
      "event": "ISM Services PMI",
      "volatility": "high",
      "actual": "51.4",
      "forecast": "51.0",
      "previous": "48.8"
    }
  ],
  "fed_speakers": [],
  "market_events": [
    {
      "time": "09:31",
      "timezone": "America/New_York", 
      "description": "Nikkei crashes -12.4% overnight, yen carry trade unwind"
    }
  ]
}
```

This challenge focuses on the **economic_events** portion — the structured calendar data.

---

## Primary Data Source

Your primary data source is **investing.com's economic calendar**:

```
https://www.investing.com/economic-calendar/
```

Spend a few minutes exploring the site. You'll notice:

- Events can be filtered by date, country, and volatility (1-3 "stars")
- Each event has: time, country, event name, actual/forecast/previous values
- The calendar goes back several years, but historical depth varies
- The site uses dynamic loading (JavaScript-rendered content)

---

## Requirements

### Phase 1: Core Scraper (Primary Deliverable)

Build a scraper that:

1. **Fetches economic calendar data for a given date range**
   - Input: start date, end date
   - Output: structured data (JSON or database records)

2. **Initially focuses on high-volatility US events**
   - These are the 3-star (⭐⭐⭐) events: NFP, CPI, FOMC, GDP, etc.
   - These move markets and are the highest-value data points

3. **Is architected to eventually support**:
   - Any country (not just US)
   - Any volatility level (1, 2, or 3 stars)
   - The filtering UI on investing.com makes this clear — your code should reflect this extensibility

4. **Handles the realities of web scraping**:
   - Rate limiting (be respectful, don't hammer the server)
   - Retries with exponential backoff
   - Session management if needed
   - Error handling and logging

5. **Stores data in PostgreSQL**
   - Design a sensible schema
   - Handle idempotency (re-running for the same date, or a full run 'from the beginning of time' shouldn't create duplicates)

### Phase 2: Historical Backfill (Secondary Consideration)

If investing.com's data doesn't go back far enough (e.g., pre-2015), you may need to supplement with other sources.

**Wayback Machine API** is one option:
```
https://archive.org/wayback/available?url=investing.com/economic-calendar/&timestamp=20150101
```

This returns archived snapshots you can then parse.

Other potential sources to research:
- FRED (Federal Reserve Economic Data) — has historical release dates
- Trading Economics archives
- News APIs with historical access

For this challenge, **document your approach** to historical backfill even if you don't fully implement it. We want to see how you think about data completeness.

---

Beyond that, but perhaps slightly out of the scope of this challenge is days when some extraneous, non-economic report related event moves markets — e.g., a major geopolitical event (trump tweeting about Xi on Friday, October 25th, 2025 which caused most indexes to sell off 3%+). I would look into what Wikipedia has for "On this day" type data, or news APIs that can be queried by date to find major events that happened on a given day.

---

## Technical Guidance

### Suggested Tech Stack

Use whatever you're comfortable with, but here's what we use internally:

- **Language**: Go please! (`go mod init hindsightapi`)
- **Database**: PostgreSQL
- **Scraping**: Playwright or Puppeteer (for JS-rendered content), or raw HTTP if you can reverse-engineer the API calls, or a go library like colly, chromedp, or rod
- **Scheduling**: Simple cron for daily runs; can discuss more robust options later

### Example Schema (Suggestion, Not Prescription)

```sql
CREATE TABLE economic_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    -- When
    event_date DATE NOT NULL,
    event_time TIME,  -- NULL if "all day" or "tentative"
    timezone VARCHAR(50) DEFAULT 'America/New_York',
    
    -- What
    country_code VARCHAR(5) NOT NULL,  -- 'US', 'EU', 'GB', etc.
    event_name VARCHAR(255) NOT NULL,
    volatility INT CHECK (volatility BETWEEN 1 AND 3),
    
    -- Values
    actual VARCHAR(50),
    forecast VARCHAR(50),
    previous VARCHAR(50),
    
    -- Metadata
    source VARCHAR(50) DEFAULT 'investing.com',
    source_url TEXT,
    scraped_at TIMESTAMP DEFAULT NOW(),
    
    -- Prevent duplicates
    UNIQUE(event_date, country_code, event_name, event_time)
);

CREATE INDEX idx_events_date ON economic_events(event_date);
CREATE INDEX idx_events_country_date ON economic_events(country_code, event_date);
```

### Example Scraper Structure (Pseudocode)

```python
class EconomicCalendarScraper:
    def __init__(self, config: ScraperConfig):
        self.config = config
        self.session = self._init_session()
    
    def scrape_date_range(
        self,
        start_date: date,
        end_date: date,
        countries: list[str] = ["US"],
        min_volatility: int = 3
    ) -> list[EconomicEvent]:
        """
        Main entry point. Iterates through dates and collects events.
        """
        events = []
        for current_date in self._date_range(start_date, end_date):
            daily_events = self._scrape_single_day(
                current_date, 
                countries, 
                min_volatility
            )
            events.extend(daily_events)
            self._rate_limit_pause()
        return events
    
    def _scrape_single_day(self, target_date: date, ...) -> list[EconomicEvent]:
        """
        Fetches and parses a single day's calendar.
        Handle retries, errors, etc. here.
        """
        # Your implementation
        pass
    
    def _parse_event_row(self, row_element) -> EconomicEvent:
        """
        Extracts structured data from a single event row.
        This is where the HTML parsing magic happens.
        """
        # Your implementation
        pass
```

### Handling Dynamic Content

investing.com loads content via JavaScript. Options:

1. **Headless browser** (Playwright/Puppeteer) — most reliable, slower
2. **Reverse-engineer XHR calls** — faster if you can find the underlying API
3. **Check for server-side rendered fallback** — some sites have this

Whichever approach you choose, document *why* you chose it.

---

## Deliverables

1. **Working scraper code**
   - Should run and produce real data
   - Include clear setup instructions (README)

2. **Database schema + migrations**
   - We should be able to spin up a fresh database

3. **Sample output**
   - Provide a JSON dump or database export of at least 30 days of scraped data
   - High-volatility US events at minimum

4. **Brief technical write-up** (1-2 pages max)
   - Architecture decisions and trade-offs
   - How you'd handle historical backfill for older data
   - How you'd extend this to other countries/volatility levels
   - Known limitations or areas for improvement
   - Rough estimate: how long to backfill to January 2010?

---

## Evaluation Criteria

We're looking for:

| Criteria | What We're Assessing |
|----------|---------------------|
| **Code quality** | Clean, readable, well-organized code. Proper error handling. |
| **Data quality** | Accurate parsing. Handles edge cases (missing values, weird formats). |
| **Reliability** | Graceful failure, retries, idempotency. Doesn't break on transient issues. |
| **Extensibility** | Easy to add new countries, volatility levels, or data sources. |
| **Pragmatism** | Solves the problem without over-engineering. Good judgment on trade-offs. |
| **Communication** | Clear documentation. Explains decisions in the write-up. |

We're *not* looking for:
- A perfect, production-ready system (this is a time-boxed challenge)
- Fancy abstractions for their own sake
- Every edge case handled — but document what you'd handle with more time

---

## Time Expectation

This challenge is designed to take approximately **6-10 hours** for an experienced developer. 

If you find yourself spending significantly more time, focus on:
1. A working scraper for a limited date range
2. Clean code for what you do build
3. A thoughtful write-up explaining what you'd do with more time

Quality over quantity.

---

## Questions?

If anything is unclear, ask. In real work, clarifying requirements early is far better than building the wrong thing.

---

## Bonus Points (Optional)

If you finish early and want to explore:

- Add a simple CLI for running backfills
- Prototype a Wayback Machine integration
- Add basic data validation / anomaly detection (e.g., "CPI came in at 847% — that's probably a parsing error")

These are genuinely optional. A clean, working core solution beats a messy solution with bells and whistles.

---

Good luck. We're excited to see what you build.
