# Code Challenge: Hindsight Data — Marketing Site & Subscriber Dashboard

## Overview

We are building the public-facing website for **The Hindsight Data API** (domain: `hindsightapi.com`) — a SaaS product that provides historical market context data via API. (working tagline: "Hindsight is 20/20. Now it's an API.")

The website needs to:

1. **Convert visitors into subscribers** (marketing/landing page)
2. **Allow subscribers to manage their access** (dashboard with API key + download link(s))

Your challenge is to build a **production-ready Next.js application** with authentication, a conversion-optimized landing page, and a subscriber dashboard.

For any patterns of a nearly identical site in terms of stack (Next.js, TypeScript, Tailwind, Clerk auth, go functions), Chris will give you access to vannacharm.com's repo for reference.

---

## The Product: What is Hindsight Data?

Before diving into requirements, here's what the product actually does (so you can write compelling copy):

### The Problem

Traders and quantitative researchers often analyze historical market data. When they see unusual price movements — a sudden crash, an unexpected rally — they ask: **"What happened that day?"**

Finding this context is tedious:
- Searching old news articles
- Checking economic calendar archives
- Reading through Fed statements
- Piecing together the story manually

### The Solution

Hindsight Data provides a simple API that answers: **"What market-moving events happened on this date?"**

```bash
curl https://api.hindsightdata.com/v1/context/2024-08-05 \
  -H "X-API-Key: your_api_key"
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
      "type": "major_move",
      "description": "Nikkei crashes -12.4% overnight; yen carry trade unwind triggers global sell-off"
    }
  ]
}
```

### Target Audience

- Quantitative traders building backtesting systems
- Algo developers who need to filter strategies around high-impact events
- Financial researchers studying market reactions
- Trading educators creating content

### Value Proposition

- **Save hours of research** — context for any trading day, instantly
- **Backtest smarter** — filter out days with known catalysts
- **Clean, structured data** — JSON API, not scraping news sites

---

## Technical Requirements

### Stack (Hard Requirements)

| Technology | Version | Notes |
|------------|---------|-------|
| **Next.js** | 14+ (App Router) | Use the latest stable version |
| **TypeScript** | Strict mode | No `any` types |
| **React** | 18+ | Server and Client Components |
| **Tailwind CSS** | 3.4+ | No other CSS frameworks or custom CSS |
| **Clerk** | Latest | Authentication & user management |

### Hosting Assumption

The site will be deployed to **Netlify**. Our serverless functions (if needed) will be written in Go.

---

## Pages & Features

### 1. Landing Page (`/`)

This homepage will be the primary conversion tool. It should include:

#### Hero Section
- Clear headline communicating the value prop
- Subheadline with more detail
- Primary CTA: "Get Started" or "Start Free Trial"
- Secondary CTA: "View Documentation" or "See Pricing"
- Optional: Animated or static visual showing the API in action

#### Social Proof / Trust Section
- Placeholder for testimonials (use dummy content)
- "Trusted by traders at..." with placeholder logos
- Or: "Join X+ traders using Hindsight Data"

#### How It Works
- 3-4 step visual explanation
- Step 1: Sign up
- Step 2: Get your API key
- Step 3: Query any historical date
- Step 4: Build smarter strategies

#### Features Section
- Key features with icons
- Examples: "10+ years of data", "Sub-second response times", "Economic events, Fed speakers, market news"

#### Pricing Section
- Embed the reusable pricing component (see below)
- Monthly and yearly toggle
- Feature comparison if multiple tiers

#### FAQ Section
- Common questions about the product
- Collapsible accordion style

#### Footer
- Links: Pricing, Documentation, Terms, Privacy, Contact
- Copyright

#### Conversion Best Practices to Implement

- **Above the fold**: Value prop + CTA visible without scrolling
- **Reduce friction**: Minimal form fields for signup
- **Social proof**: Near the CTA buttons
- **Clear pricing**: No hidden fees, obvious what you get
- **Trust signals**: Security badges, money-back guarantee (if applicable)
- **Mobile-first**: Must look great on mobile

### 2. Pricing Page (`/pricing`)

Dedicated pricing page using the **same reusable component** as the landing page, but with more detail:

- Full feature comparison table
- FAQ specific to pricing/billing
- "Questions? Contact us" CTA

### 3. Authentication (Clerk)

Implement using Clerk:

- **Sign Up** (`/sign-up`)
- **Sign In** (`/sign-in`)
- **User Profile** (Clerk's built-in or custom)

Clerk configuration:
- Email + password authentication (minimum)
- Optional: Google OAuth
- Redirect to dashboard after auth

### 4. Subscriber Dashboard (`/dashboard`)

**Protected route** — only accessible to authenticated users.

#### API Key Section

Display the user's API key with:
- Masked by default (show `hindsight_live_••••••••••••••••`)
- "Reveal" button to show full key
- "Copy" button with feedback ("Copied!")
- "Regenerate" button (with confirmation modal)

**Note on API Key Storage**: The API key should be stored in Clerk's user metadata. Use `privateMetadata` for the actual key (server-side only) since it's sensitive. The dashboard will need a server action or API route to fetch/display/regenerate the key.

```typescript
// Example structure in Clerk privateMetadata
{
  apiKey: "hindsight_live_550e8400e29b41d4a716446655440000",
  apiKeyCreatedAt: "2025-01-15T10:30:00Z"
}
```

#### Quick Start / API Instructions

Show users how to use their key:

```tsx
// Code block with syntax highlighting
curl https://api.hindsightdata.com/v1/context/2024-08-05 \
  -H "X-API-Key: {USER_API_KEY}"
```

Include:
- Example request
- Example response (use hardcoded dummy data)
- Link to full documentation

#### Download Section

For users who want bulk data instead of API calls:

- Date range picker (start date, end date)
- "Download CSV" button
- "Download JSON" button
- Note: For this challenge, buttons can trigger a dummy download or show a toast. The actual implementation will come later.

#### Subscription Status

Show:
- Current plan (Free Trial, Monthly, Yearly)
- Billing period end date
- "Manage Subscription" link (can link to Clerk's billing portal or Stripe customer portal)

---

## Reusable Pricing Component

Build a `<PricingSection />` component that:

### Props

```typescript
interface PricingProps {
  showToggle?: boolean;      // Show monthly/yearly toggle
  defaultInterval?: 'monthly' | 'yearly';
  variant?: 'compact' | 'full';  // For landing page vs pricing page
}
```

### Pricing Tiers

```typescript
// config/pricing.ts

export const PRICING_TIERS = [
  {
    name: 'Starter',
    description: 'For individual traders and researchers',
    monthlyPrice: 29,
    yearlyPrice: 249,  // ~30% discount
    stripeLinkMonthly: 'STRIPE_PAYMENT_LINK_STARTER_MONTHLY', // TODO: Replace with actual link
    stripeLinkYearly: 'STRIPE_PAYMENT_LINK_STARTER_YEARLY',   // TODO: Replace with actual link
    features: [
      'Full historical data (2010-present)',
      '1,000 API requests / day',
      'Economic events',
      'Fed speakers',
      'Email support',
    ],
    highlighted: false,
  },
  {
    name: 'Pro',
    description: 'For serious traders and small teams',
    monthlyPrice: 79,
    yearlyPrice: 699,
    stripeLinkMonthly: 'STRIPE_PAYMENT_LINK_PRO_MONTHLY',
    stripeLinkYearly: 'STRIPE_PAYMENT_LINK_PRO_YEARLY',
    features: [
      'Everything in Starter',
      '10,000 API requests / day',
      'Major market events & news',
      'Bulk CSV downloads',
      'Priority support',
    ],
    highlighted: true,  // Visually emphasize this tier
  },
  {
    name: 'Enterprise',
    description: 'For funds and institutions',
    monthlyPrice: null,  // "Contact us"
    yearlyPrice: null,
    stripeLinkMonthly: null,
    stripeLinkYearly: null,
    features: [
      'Everything in Pro',
      'Unlimited API requests',
      'Custom data feeds',
      'SLA guarantee',
      'Dedicated support',
    ],
    highlighted: false,
    cta: 'Contact Sales',
  },
];
```

### Toggle Behavior

- Default to yearly (better for us, better value for customer)
- Animate price change on toggle
- Show savings: "Save 30%" badge on yearly

### Stripe Integration Notes

For this challenge, the payment links will be **hardcoded placeholder strings**. In production, these would be actual Stripe Payment Links.

When a user clicks "Subscribe":
1. If not logged in → redirect to sign up, then redirect to Stripe
2. If logged in → redirect directly to Stripe Payment Link

The Stripe Payment Link should include the Clerk `userId` as a client reference ID so we can associate the subscription with the user.

```typescript
// Example: constructing the payment URL
const getCheckoutUrl = (tier: PricingTier, interval: 'monthly' | 'yearly', userId?: string) => {
  const link = interval === 'monthly' ? tier.stripeLinkMonthly : tier.stripeLinkYearly;
  if (!link) return null;
  
  // In production, append client_reference_id
  const url = new URL(link);
  if (userId) {
    url.searchParams.set('client_reference_id', userId);
  }
  return url.toString();
};
```

---

## Clerk Metadata Strategy

### The Challenge

We need to store each user's API key. Clerk provides two types of metadata:

- **`publicMetadata`**: Readable on the client, writable only from backend
- **`privateMetadata`**: Readable and writable only from backend

### Recommended Approach

Store the API key in **`privateMetadata`** because:
- API keys are secrets; they shouldn't be exposed to client-side JavaScript
- If someone inspects network requests, they won't see the raw key

### Implementation Pattern

```typescript
// Server action or API route to get the user's API key
// app/actions/api-key.ts

'use server';

import { auth, clerkClient } from '@clerk/nextjs/server';
import { v4 as uuidv4 } from 'uuid';

export async function getApiKey() {
  const { userId } = auth();
  if (!userId) throw new Error('Unauthorized');
  
  const user = await clerkClient.users.getUser(userId);
  const apiKey = user.privateMetadata.apiKey as string | undefined;
  
  return apiKey || null;
}

export async function generateApiKey() {
  const { userId } = auth();
  if (!userId) throw new Error('Unauthorized');
  
  const newKey = `hd_live_${uuidv4().replace(/-/g, '')}`;
  
  await clerkClient.users.updateUser(userId, {
    privateMetadata: {
      apiKey: newKey,
      apiKeyCreatedAt: new Date().toISOString(),
    },
  });
  
  return newKey;
}

export async function regenerateApiKey() {
  // Same as generateApiKey, but could log the old key being revoked
  return generateApiKey();
}
```

The dashboard calls these server actions to fetch/display/regenerate the key.

---

## Dummy Data & Mock Responses

For the dashboard API preview section, use this hardcoded response:

```typescript
// lib/mock-data.ts

export const MOCK_API_RESPONSE = {
  date: "2024-08-05",
  economic_events: [
    {
      time: "10:00",
      timezone: "America/New_York",
      country: "US",
      event: "ISM Services PMI",
      volatility: "high",
      actual: "51.4",
      forecast: "51.0",
      previous: "48.8"
    }
  ],
  fed_speakers: [
    {
      time: "13:30",
      timezone: "America/New_York",
      speaker: "Mary Daly",
      title: "San Francisco Fed President",
      topic: "Economic Outlook",
      location: "Commonwealth Club, San Francisco"
    }
  ],
  market_events: [
    {
      time: "09:31",
      timezone: "America/New_York",
      type: "major_move",
      description: "Nikkei crashes -12.4% overnight; yen carry trade unwind triggers global sell-off"
    }
  ]
};
```

---

## File Structure (Suggested)

```
hindsight-data/
├── app/
│   ├── (marketing)/
│   │   ├── page.tsx                 # Landing page
│   │   ├── pricing/
│   │   │   └── page.tsx             # Pricing page
│   │   └── layout.tsx               # Marketing layout (navbar, footer)
│   ├── (auth)/
│   │   ├── sign-in/[[...sign-in]]/
│   │   │   └── page.tsx
│   │   ├── sign-up/[[...sign-up]]/
│   │   │   └── page.tsx
│   │   └── layout.tsx
│   ├── (dashboard)/
│   │   ├── dashboard/
│   │   │   └── page.tsx             # Main dashboard
│   │   └── layout.tsx               # Dashboard layout (sidebar?)
│   ├── api/
│   │   └── webhooks/
│   │       └── stripe/
│   │           └── route.ts         # Stripe webhook handler (stub)
│   ├── layout.tsx                   # Root layout
│   └── globals.css
├── components/
│   ├── marketing/
│   │   ├── Hero.tsx
│   │   ├── Features.tsx
│   │   ├── HowItWorks.tsx
│   │   ├── Testimonials.tsx
│   │   ├── FAQ.tsx
│   │   └── Footer.tsx
│   ├── pricing/
│   │   ├── PricingSection.tsx       # Reusable pricing component
│   │   ├── PricingCard.tsx
│   │   └── PricingToggle.tsx
│   ├── dashboard/
│   │   ├── ApiKeyCard.tsx
│   │   ├── ApiInstructions.tsx
│   │   ├── DownloadSection.tsx
│   │   └── SubscriptionStatus.tsx
│   └── ui/
│       ├── Button.tsx
│       ├── Card.tsx
│       ├── Badge.tsx
│       └── ...                      # Shared UI components
├── lib/
│   ├── mock-data.ts
│   └── utils.ts
├── config/
│   ├── pricing.ts                   # Pricing tiers config
│   └── site.ts                      # Site metadata
├── actions/
│   └── api-key.ts                   # Server actions for API key management
├── middleware.ts                    # Clerk auth middleware
├── tailwind.config.ts
├── tsconfig.json
└── package.json
```

---

## Evaluation Criteria

| Criteria | What We're Assessing |
|----------|---------------------|
| **Code quality** | Clean TypeScript, proper component structure, no `any` types |
| **UI/UX** | Visually polished, good use of Tailwind, responsive design |
| **Conversion thinking** | Does the landing page follow best practices? Clear CTAs? |
| **Component design** | Is the pricing component truly reusable? Good props API? |
| **Auth implementation** | Clerk integrated correctly, protected routes work |
| **Server/client split** | Appropriate use of Server Components vs Client Components |
| **Attention to detail** | Loading states, error handling, copy-to-clipboard feedback |

We're **not** looking for:
- Actual Stripe integration (placeholders are fine)
- Real API backend (mock data is fine)
- Perfect copy/content (placeholder text is fine, but structure matters)
- Fancy animations (unless they serve a purpose)

We **are** looking for:
- A site we could actually deploy and start using
- Evidence you understand conversion optimization
- Clean, maintainable code we can build on

---

## Deliverables

1. **Complete Next.js application**
   - All pages and components listed above
   - Clerk authentication working
   - Responsive design (mobile, tablet, desktop)

2. **README with**
   - Setup instructions
   - Environment variables needed (Clerk keys, etc.)
   - Any design decisions or trade-offs

3. **Deployed preview** (optional but appreciated)
   - Vercel preview deployment
   - Or screenshots/video walkthrough

---

## Time Expectation

This challenge is designed to take approximately **10-15 hours**:

- ~2-3 hours on project setup and Clerk integration
- ~4-5 hours on landing page and components
- ~3-4 hours on dashboard
- ~2-3 hours on polish, responsiveness, and documentation

If you're spending significantly more time, focus on:
1. Landing page looking great
2. Pricing component working correctly
3. Dashboard with API key display

Polish over completeness.

---

## Resources

- [Next.js App Router Docs](https://nextjs.org/docs/app)
- [Clerk Next.js Quickstart](https://clerk.com/docs/quickstarts/nextjs)
- [Clerk User Metadata](https://clerk.com/docs/users/metadata)
- [Tailwind CSS Docs](https://tailwindcss.com/docs)
- [Stripe Payment Links](https://stripe.com/docs/payment-links)

---

## Questions?

If anything is unclear, ask. We'd rather answer questions upfront than have you build the wrong thing.

---

## Bonus

If you can prepare a /context/[slug].tsx page, where [slug] is the date in YYYY-MM-DD format, that fetches and displays context data in a nice way for a given date using the mock data provided, that would be a nice extra touch. This would be a potential SEO gold mine in the future to capture organic search traffic for people looking up "what happened on [date] in the markets".

Good luck. We're excited to see what you create.
