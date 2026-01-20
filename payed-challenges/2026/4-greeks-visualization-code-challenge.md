# Code Challenge: Greeks Exposure Visualization — 4D Data Problem

## Overview

We have a unique data visualization challenge: displaying **options market greek exposures** (gamma, vanna, charm) across multiple dimensions — strike price, expiration date, exposure value, and time.

This is genuinely hard. We haven't solved it ourselves. **We're looking for creative ideas, not a "right answer."**

Your challenge is to explore approaches and build a prototype that helps traders understand how dealer positioning evolves during a trading session.

---

## Background: What Are We Visualizing?

If you're not familiar with options trading, here's the context you need:

### Options and Market Makers

Options are financial contracts that give the right to buy or sell an asset at a specific price (the **strike price**) by a specific date (the **expiration**).

Market makers (big banks, trading firms) sell these options to traders. When they sell options, they take on risk. They quantify this risk using "greeks":

- **Gamma**: How much their directional exposure changes as price moves
- **Vanna**: How gamma changes as volatility changes
- **Charm**: How gamma decays over time

### Why Traders Care

When market makers have large exposures, they must hedge by trading the underlying asset (like SPY or QQQ). This hedging activity can amplify or dampen price moves.

If a trader knows:
- "Dealers are massively short gamma at the 580 strike"
- "That exposure is growing as we approach expiration"

...they can anticipate how the market might behave.

### The Data We Have

For a single point in time, we know the greek exposure at every combination of:
- **Strike price** (e.g., $575, $580, $585, $590...)
- **Expiration date** (e.g., tomorrow, next Friday, monthly, quarterly...)

And we have this data at **1-minute intervals** throughout the trading day.

---

## The Visualization Problem

Here's what makes this hard:

| Dimension | Description | Typical Range |
|-----------|-------------|---------------|
| Strike | Price levels | 200-400 active strikes |
| Expiration | Contract expiry dates | 8-12 active expirations |
| Exposure | Gamma, vanna, or charm value | Continuous (can be positive or negative) |
| Time | When during the session | 390 minutes per trading day |

**That's 4 dimensions.** Your screen has 2. Color/size gives you a pseudo-third. Animation/interaction gives you the fourth.

The challenge: make it **useful**, not just pretty.

A trader should be able to look at your visualization and answer:
- "Where is the largest exposure concentrated right now?"
- "How has positioning shifted since the open?"
- "Which expirations / strike are driving the changes?"
- "Is most of the activity / change in today's expiration, or a longer-dated one?"

---

## Sample Data Structure

You could imagine the data looking like this JSON structure:

```json
{
  "symbol": "SPY",
  "snapshots": [
    {
      "timestamp": "2025-01-15T09:30:00-05:00",
      "expirations": [
        {
          "expiration": "2025-01-15",
          "label": "0DTE",
          "strikes": [
            { "strike": 580.0, "gamma": -52000000, "vanna": 12000000, "charm": -4500000 },
            { "strike": 581.0, "gamma": -48000000, "vanna": 11200000, "charm": -4200000 },
            { "strike": 582.0, "gamma": -43000000, "vanna": 10100000, "charm": -3800000 }
            // ... more strikes
          ]
        },
        {
          "expiration": "2025-01-17",
          "label": "Weekly",
          "strikes": [
            { "strike": 580.0, "gamma": -22000000, "vanna": 8000000, "charm": -2100000 }
            // ... more strikes
          ]
        }
        // ... more expirations
      ]
    },
    {
      "timestamp": "2025-01-15T09:31:00-05:00",
      // ... next minute's snapshot
    }
    // ... 390 snapshots for a full trading day (1 minute intervals from 9:30 AM to 4:00 PM Eastern standard time)
  ]
}
```

For the challenge, just take data with the shape above for a single trading day with ~50 strikes, 4-5 expirations, and 60-90 time snapshots (1-minute intervals). It's okay if you even just randomly generate data that fits this structure, the focus is on visualization itself.

---

## Possible Approaches (Starting Points, Not Answers)

We've brainstormed some ideas, but **none of these are "the answer"** — they're starting points for your exploration. Feel free to pursue one of these, combine them, or go in a completely different direction.

### 1. Small Multiples + Time Scrubber

A grid of mini-heatmaps, one per expiration. Each shows strike (Y-axis) vs exposure (color intensity). A global time slider scrubs all charts simultaneously.

```
[0DTE]     [1DTE]     [Weekly]   [Monthly]
┌─────┐    ┌─────┐    ┌─────┐    ┌─────┐
│▓▓▓░░│    │░▓▓░░│    │░░▓░░│    │░░░▓░│
│▓▓▓▓░│    │░▓▓▓░│    │░▓▓░░│    │░░▓▓░│
│▓▓▓░░│    │░▓▓░░│    │░░▓░░│    │░░░▓░│
└─────┘    └─────┘    └─────┘    └─────┘

◄──────────────[10:30 AM]──────────────►
```

Pros: Shows all expirations at once, time is interactive
Cons: Visual noise with many expirations

### 2. Aggregated View + Drill-down

Start with a simple line chart showing *total* exposure over time (summed across all strikes/expirations). Click on any timestamp to see the detailed breakdown at that moment.

```
Total Gamma Exposure
     │    ╱╲
     │   ╱  ╲    ╱╲
     │  ╱    ╲  ╱  ╲
     └─────────────────
       9:30    12:00   4:00
              ↑ click for details
```

Pros: Simple at first glance, complexity is opt-in
Cons: Loses the "watch it evolve" feel

### 3. Animated Heatmap with Motion Trail

Strike (Y) vs Expiration (X), color = exposure. Time scrubber animates through the session.

The twist: Show "ghosts" of previous timestamps at reduced opacity, so you see where exposure *was* versus where it *is now*. Like motion blur for data.

Pros: Shows movement and direction
Cons: Could get muddy with lots of data

### 4. Differential / Change View

Instead of showing absolute exposure, show *change since last interval* (or since market open). Red = increasing exposure, blue = decreasing, intensity = magnitude.

Answers "What's moving right now?" rather than "What's the current state?"

Pros: Highlights what matters most (changes)
Cons: Loses absolute context

### 5. Ridgeline Plot (Joy Division Style)

One "ridge" per expiration, stacked vertically. X-axis = strike, height = exposure magnitude. Animate over time.

```
Monthly  ─────╱╲─────────
Weekly   ────╱╲╲╲────────
1DTE     ──╱╲╲╲╲╲╲───────
0DTE     ╱╲╲╲╲╲╲╲╲╲──────
         ← strikes →
```

Pros: Beautiful, shows distribution shape clearly
Cons: Overlapping ridges can obscure data

### 6. User-Controlled Filtering

Don't try to show everything. Let the user choose:
- Which greek (gamma / vanna / charm toggle)
- Which expirations (multi-select)
- Strike range (zoom/filter)
- Playback speed

Show a clean 2D visualization that animates, with complexity controlled by the user.

Pros: User gets exactly what they need
Cons: Requires more interaction to explore

### 7. Something We Haven't Thought Of

3D with orbit controls? A radial/polar layout? Sound mapped to exposure changes? An AI-generated narrative summary? 

We genuinely don't know the best approach. Surprise us.

---

## Requirements

### Technical Stack

| Technology | Version | Notes |
|------------|---------|-------|
| **React** | 18+ | Functional components, hooks |
| **TypeScript** | Strict mode | No `any` types |
| **Tailwind CSS** | 3.4+ | For layout and styling |

For charting/visualization, choose what fits your approach:
- **Recharts** — Good for standard charts
- **D3.js** — Maximum flexibility
- **Visx** — D3 primitives as React components
- **Three.js / React Three Fiber** — If you go 3D
- **Canvas / WebGL** — For performance with large datasets
- **Framer Motion** — For animations

Justify your library choices in your write-up. And note that we are already using Recharts in VannaCharm.com

### Functional Requirements

Your prototype must:

1. **Load and parse sample data**
2. **Display greek exposures** across strikes and expirations
3. **Show changes over time** (playable animation, movable scrubber, or other interaction)
4. **Toggle (activate / deactivate) between greeks** (gamma, vanna, charm)
5. **Be responsive** — works on desktop and tablet at minimum - mobile is a bonus
6. **Perform reasonably** — no freezing

### UX Requirements

Your visualization should help a trader answer:
- Where is exposure concentrated right now?
- How has it changed since the open?
- Which expirations are most significant?
- Where are the largest positive and negative exposures?
- "Which expirations / strike are driving the changes?"
- "Is most of the activity / change in today's expiration, or a longer-dated one?"

Include a brief "how to read this" guide or legend in the UI.

---

## Deliverables

1. **Working prototype**
   - React component that works with the existing vannacharm.com codebase
   - Loads and displays sample data
   - All functional requirements met
   - README with setup instructions

---

## Evaluation Criteria

| Criteria | What We're Assessing |
|----------|---------------------|
| **Creativity** | Did you explore the problem space? Is your approach thoughtful? |
| **Usefulness** | Can a trader actually extract insights from this? |
| **Execution** | Is the code clean? Does it work? Is it performant? |
| **Design sense** | Is it visually clear? Good use of color, space, hierarchy? |
| **Communication** | Can you explain your decisions? Is the UI self-documenting? |

We're **not** looking for:
- A polished, production-ready product
- Every feature implemented perfectly
- The "correct" answer (there isn't one)

We **are** looking for:
- Evidence of creative problem-solving
- A working prototype that demonstrates your idea
- Clear thinking about trade-offs
- Code quality in what you do build

---

## What Success Looks Like

A successful submission makes us say:

> "Oh, that's clever — I hadn't thought of showing it that way. And I can actually understand what the data is telling me."

We want to learn from your approach. The best submission might change how we think about this problem.

---

## Time Expectation

This challenge is designed to take approximately **6-8 hours**:

- ~1-2 hours exploring the problem and sketching approaches
- ~5-6 hours building the prototype

If you're spending significantly more time, focus on:
1. One well-executed visualization approach
2. Clean, working code
3. Clear explanation of your thinking

**Depth over breadth.** A single brilliant idea, well-executed, beats three half-finished experiments.

---

## Questions?

This is an open-ended challenge by design. But if something is unclear about the data structure, requirements, or expectations, ask. We'd rather clarify than have you waste time on misunderstandings.

---

## A Note on Domain Knowledge

You don't need to be an options trader to do this challenge well. You need to:
- Understand the data structure (4D: strike × expiration × exposure × time)
- Understand the goal (help someone see patterns and changes)
- Be creative about visualization approaches

Think of it like visualizing weather data across cities, altitudes, temperature, and time — even if you're not a meteorologist, you can design a useful weather dashboard.

---

Good luck. We're genuinely curious what you'll come up with.
