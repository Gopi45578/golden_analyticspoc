# Budget Lens — Washington State Fiscal Explorer

> A natural language analytics tool that lets non-technical users explore $29.5 billion in Washington State government spending through plain-English questions, narrative answers, and interactive charts.

**Built for:** Golden Analytics — "Turn Business Data Into Answers" challenge

---

## Quick Start

```bash
npm install
npm run dev
# Open http://localhost:5173
```

No database, no backend, no API key required. The app works immediately with a built-in local query engine. Optionally connect an OpenAI API key (via the header button) for richer, AI-powered responses.

---

## What This Is

Budget Lens targets **non-technical government stakeholders** — a city councilmember trying to understand where the money went, a journalist tracking spending trends, a policy analyst who knows the questions but not the SQL.

The core interaction: **type a question in plain English → get a narrative answer + chart + transparency into how the answer was computed.**

### Key UX Decisions

1. **Natural language input as the primary interface.** No dropdowns, no filter panels, no pivot tables. The user types a question like "Which agencies spent the most?" and gets a direct answer. This is the simplest possible interaction model for a non-technical user.

2. **Narrative-first answers.** The response is a 2-4 sentence plain-English summary with highlighted dollar amounts — not a table. Tables are information-dense but cognitively expensive. A narrative answer respects the user's time and context.

3. **SQL/logic is hidden by default.** The query explanation drawer exists for transparency and trust ("how did you get this number?") but is collapsed by default. SQL is never visible unless the user explicitly asks. This satisfies the challenge constraint while still enabling power users.

4. **Suggested questions as onboarding.** Non-technical users often don't know what to ask first. The suggestion chips provide concrete starting points that demonstrate the tool's capability. The app also auto-loads a default question on launch so the user never sees an empty state.

5. **Dual-engine architecture (AI + Local).** The app works without an API key using a keyword-based local engine. When an OpenAI key is connected, it upgrades to GPT-4o-mini for flexible natural language understanding. This means the demo always works — no API key fumbling during a live walkthrough.

---

## Architecture

```
src/
├── App.tsx                    # Main orchestrator — state management, layout
├── components/
│   ├── QuestionBar.tsx        # Natural language input bar
│   ├── SuggestedQuestions.tsx  # One-click question chips
│   ├── AnswerPanel.tsx        # Narrative answer with metric highlighting
│   ├── ChartPanel.tsx         # Recharts-based bar/line/pie visualization
│   ├── SqlDrawer.tsx          # Collapsible query explanation panel
│   ├── LoadingState.tsx       # Skeleton loading with pulse animation
│   └── ApiKeyModal.tsx        # OpenAI API key configuration
├── services/
│   ├── aiEngine.ts            # Dual query engine (OpenAI + local fallback)
│   └── aiLogger.ts            # AI governance logger (singleton, in-memory)
├── types/
│   └── index.ts               # TypeScript interfaces for all data shapes
├── data/
│   └── aggregated.json        # Pre-aggregated WA State fiscal data (~35KB)
└── index.css                  # Full design system (830 lines, dark theme)
```

### Data Pipeline

The raw Excel dataset (451K records) was pre-processed by `scripts/aggregate-data.cjs` into a 35KB JSON file with:
- Agency-level totals (99 agencies)
- Category breakdowns (9 spending categories)
- Monthly spending trends (12 months)
- Vendor rankings (65K vendors, top sliced)
- Agency × Category cross-tabulations
- Agency × Month time series

**Trade-off:** Pre-aggregation means the app can't answer arbitrary drill-down questions (e.g., "How much did vendor X get from agency Y in March?"). This was intentional — it keeps the bundle small, eliminates the need for a database, and covers the 80% of questions a non-technical user would actually ask. The AI engine works against the aggregated summaries rather than raw data.

---

## AI Integration

### How It Works

When an OpenAI API key is configured:
1. The system prompt sends the full data schema + aggregated summaries to GPT-4o-mini
2. The model returns structured JSON: `{ answer, chartType, chartData, chartTitle, queryDescription, filterLogic }`
3. The response is rendered as a narrative + chart + transparency drawer

**Design decision:** The AI does NOT generate executable SQL or code. It analyzes pre-aggregated data and returns display-ready results. This is safer (no injection risk), faster (no query execution), and more predictable for a POC.

### Local Fallback Engine

Without an API key, a keyword-matching engine handles common question patterns:
- Agency questions → top agencies bar chart
- Category questions → spending breakdown pie chart
- Monthly/trend questions → time series line chart
- Vendor questions → top vendors bar chart
- Overview questions → total spending pie chart

This isn't "fake" — it's a deterministic rule engine that produces correct answers from real data. It demonstrates the architecture without external dependencies.

### AI Governance Logging

**Every AI interaction is logged** — inputs, outputs, model used, latency, status. This satisfies the enterprise B2B governance requirement.

```typescript
// From src/services/aiLogger.ts
aiLogger.log({
  id: logId,
  timestamp: new Date(),
  userQuestion: question,
  systemPrompt,        // Full prompt sent to model
  modelInput: userMessage,
  modelResponse: responseText,
  model: 'gpt-4o-mini',
  latencyMs,
  status: 'success' | 'error',
});
```

Logs are stored in-memory (singleton pattern) and printed to the browser console with color-coded output. The header shows a live log count. In production, these would persist to a database or log aggregation service (Datadog, Elasticsearch, or a Postgres audit table).

Local engine queries are also logged with `model: 'local-keyword-engine'` — governance applies regardless of which engine handles the request.

---

## AI Collaboration & Redirections

This project was built in close collaboration with an AI coding assistant. As a developer, I used AI as an execution partner, but steered all key product and architectural decisions myself. 

Here are three specific moments where I disagreed with, pushed back on, or redirected the AI's suggestions:

### 1. Rejecting the "Dashboard/Filter Sidebar" Design
* **The AI's Proposal:** When we started outlining the UI, the AI immediately generated a standard BI interface: a sidebar with multiple dropdowns (Fiscal Year, Agency, Category, Vendor) and a massive spreadsheet-style table in the center.
* **Why I Pushed Back:** A filter panel is just SQL in disguise. It assumes the user already knows what dimensions exist, what data they need, and how to read dense tables. For a city councilmember or journalist, this is still high friction.
* **The Direction I Took:** I instructed the AI to strip out all filters and replace them with a single search bar and a set of suggestion chips. The core layout should be: **Plain-text question → Narrative answer → Highlighted metric cards → Visual chart.** 

### 2. Pushing Back on "Client-Side CSV Parsing"
* **The AI's Proposal:** When discussing the dataset (`Vendor-Payments_2021-23.xlsx`), the AI suggested using `PapaParse` to download and query the raw CSV client-side in the browser on page load.
* **Why I Pushed Back:** The raw dataset has over 451,000 rows. Parsing and querying a file that size in the browser would lock up the user's main thread, crash mobile devices, and consume massive bandwidth.
* **The Direction I Took:** I redirected the AI to build a Node pre-processing script (`scripts/aggregate-data.cjs`) to aggregate the raw rows into a 35KB structured JSON file. This aggregation is what we feed into the local engine and AI context, achieving sub-second load times.

### 3. Modifying the Prompting Strategy (Schema vs. Data)
* **The AI's Proposal:** The AI initially wanted to write a prompt that would feed individual data slices to the LLM on demand.
* **Why I Pushed Back:** This would require complex state management and round-trips to figure out which data slice was needed before sending the user's prompt.
* **The Direction I Took:** Since the aggregated JSON was only 35KB, I structured the system prompt to feed the *entire* aggregated database schema and dataset directly into the LLM context. This makes the LLM completely self-sufficient and allows it to perform flexible cross-category reasoning in a single turn.

---

## Explicit Trade-offs

### 1. Pre-aggregation vs. Live Query
**Chose:** Pre-aggregated JSON embedded in the bundle  
**Deferred:** Database connection, live SQL execution  
**Why:** Eliminates infrastructure dependencies for a 30-minute POC. Covers the majority of questions a non-technical user would ask. The architecture (question → engine → structured response → render) is identical to what a live query version would use.

### 2. Client-Side AI Calls vs. Backend Proxy
**Chose:** Direct OpenAI calls from the browser (`dangerouslyAllowBrowser: true`)  
**Deferred:** Backend API proxy  
**Why:** Removes the need for a server in the POC. The API key is stored in `localStorage` and never leaves the user's browser. In production, this would route through a backend to protect keys and add rate limiting.

### 3. Keyword Engine vs. Embedding-Based Search
**Chose:** Simple keyword matching for local fallback  
**Deferred:** Vector embeddings, semantic search  
**Why:** Keyword matching is deterministic, fast, and sufficient for the 5-6 common question patterns in this dataset. Semantic search would add complexity (embedding model, vector store) without proportional UX benefit at this scale.

### 4. Static Suggested Questions vs. Dynamic Follow-ups
**Chose:** Fixed set of 5 suggestion chips  
**Deferred:** Context-aware follow-up suggestions based on previous answers  
**Why:** Static suggestions are reliable onboarding. Dynamic follow-ups would require maintaining conversation context and generating contextual recommendations — a great V2 feature.

---

## Known Limitations & Rough Edges

Because this is a 30-minute proof of concept, there are a few intentional limitations and unpolished corners:
- **Responsive Layout for Mobile:** While the app is responsive, the Recharts components (especially the Pie Chart) can overflow or wrap awkwardly on screens smaller than 375px. 
- **Chart Label Overlaps:** If the LLM generates a breakdown containing more than 8 categories, the X-axis labels on the bar chart or data labels on the pie chart can overlap. I didn't write advanced truncation/skipping logic for the labels to keep the visualization code simple.
- **Console Warnings:** Recharts 3 is still relatively new and occasionally throws minor React hydration warnings in the browser console. These do not affect functionality but would be cleaned up in a production release.
- **Keyword Match Sensitiveness:** In local fallback mode, the keyword parser is relatively simple (regex-based). If you ask *"Which agencies spent the most cash?"*, it matches "agencies" correctly, but if you ask *"Who got the most green?"*, it defaults to the main overview.

---

## POC → Production: What Would Change

| Area | POC State | Production Requirement |
|------|-----------|----------------------|
| **Data** | Pre-aggregated JSON (35KB) | Live Postgres connection with parameterized queries |
| **AI** | Client-side OpenAI calls | Backend proxy with rate limiting, key management, cost tracking |
| **Auth** | None | SSO/OAuth2, role-based access control |
| **Logging** | In-memory + console | Persistent audit table, structured logging pipeline |
| **Caching** | None | Redis/Memcached for repeated queries |
| **Error handling** | Basic try/catch | Retry logic, circuit breakers, user-friendly error states |
| **Testing** | Manual | Jest unit tests, Playwright E2E, visual regression |
| **Deployment** | `npm run dev` | Docker container, CI/CD, CDN for static assets |
| **Accessibility** | Basic semantic HTML | WCAG 2.1 AA compliance, screen reader testing |
| **Security** | `dangerouslyAllowBrowser` | CSP headers, input sanitization, API key vault |

### What Would Break at Scale
- **Pre-aggregation** fails when users need arbitrary drill-downs or cross-dataset joins
- **In-memory logging** loses data on page refresh — needs persistent storage
- **Client-side AI calls** expose the API key in browser DevTools
- **Single JSON file** can't handle multi-tenant data isolation
- **No conversation memory** means the AI can't handle follow-up questions ("What about the second one?")

---

## Tech Stack

| Technology | Purpose |
|-----------|---------|
| React 19 + TypeScript | UI framework with type safety |
| Vite 8 | Build tool with HMR |
| Recharts 3 | Bar, line, and pie chart rendering |
| OpenAI SDK | GPT-4o-mini integration (optional) |
| Vanilla CSS | Custom dark-theme design system |

---

## Why This Approach Over Alternatives

I considered three directions before committing:

1. **Dashboard with filters** — Traditional BI approach. Rejected because it requires the user to already know what they're looking for. A city councilmember doesn't want to configure a pivot table.

2. **Pre-built report pages** — Static pages showing spending by agency, by category, etc. Rejected because it's passive. The user can only see what we decided to show them. No exploration.

3. **Natural language Q&A** (chosen) — The user drives the exploration. They ask what they care about. The tool translates their intent into data and presents it in a format they can immediately understand. This is the "Canva for data" vision — the tool adapts to the user, not the other way around.

---

## Where AI Would Add More Value (Future Iterations)

- **Anomaly detection:** "Flag any agencies whose spending increased more than 20% vs. last year"
- **Comparative analysis:** "How does WA's healthcare spending compare to Oregon?"
- **Conversation memory:** Follow-up questions like "Now break that down by month"
- **Auto-generated insights:** On load, surface the 3 most interesting patterns in the data without the user asking
- **Data storytelling:** Generate a full narrative report suitable for a council meeting presentation

---

## Running the Data Pipeline

If you need to re-generate the aggregated data from the raw Excel file:

```bash
# Place Vendor-Payments_2021-23.xlsx in the project root
node scripts/aggregate-data.cjs
# Outputs src/data/aggregated.json
```

Requires `xlsx` package: `npm install xlsx`
