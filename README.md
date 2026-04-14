# Idea Parking Lot

A full-stack idea management web app built as a data engineering portfolio project. Capture, organize, and analyze ideas from any device with real-time sync, a Postgres backend, and a live analytics dashboard.

**Live demo:** https://saumya-maurya.github.io/idea-parking-lot

---

## What it does

- **Quick capture** — park an idea in seconds with a tag, status, and energy level
- **Status tracking** — move ideas through a pipeline: Raw → In Progress → Parked → Shipped → Discarded
- **Cross-device sync** — data persists in Postgres and syncs in real time across all devices
- **Search & filter** — search by text, tag, or notes; filter by status; sort by energy or priority
- **Expandable notes** — add links, next steps, and context to any idea
- **Analytics dashboard** — pipeline funnel, 30-day timeline, energy distribution, tag breakdown, and key metrics

---

## Tech stack

| Layer | Technology |
|---|---|
| Frontend | Vanilla HTML, CSS, JavaScript |
| Hosting | GitHub Pages (static, free) |
| Database | Supabase (Postgres) |
| Auth | Supabase Auth (email + password) |
| Real-time | Supabase Realtime (postgres_changes) |
| Charts | Chart.js |

---

## Architecture

```
Browser (GitHub Pages)
    │
    ├── Auth layer (Supabase Auth)
    │       └── JWT session tokens
    │
    ├── Data layer (Supabase REST API)
    │       ├── INSERT  → park new idea
    │       ├── SELECT  → load all user ideas
    │       ├── UPDATE  → edit text / status / notes / priority
    │       └── DELETE  → remove idea
    │
    └── Real-time layer (Supabase Realtime)
            └── postgres_changes subscription
                    → live sync across tabs and devices
```

---

## Data Engineering concepts demonstrated

### 1. Data Modeling
The `ideas` table is designed with analytics in mind — every field is typed and constrained. Status and energy are stored as enums, enabling efficient aggregation queries.

```sql
create table ideas (
  id          uuid      default gen_random_uuid() primary key,
  user_id     uuid      references auth.users not null,
  text        text      not null,
  tag         text,
  status      text      default 'raw',
  energy      text      default 'medium',
  priority    boolean   default false,
  notes       text      default '',
  created_at  timestamptz default now()
);
```

### 2. Row Level Security (RLS)
Data access is enforced at the database level — not just the application layer. Each user can only read and write their own rows.

```sql
alter table ideas enable row level security;

create policy "Users manage own ideas"
  on ideas for all
  using (auth.uid() = user_id)
  with check (auth.uid() = user_id);
```

### 3. Real-time Data Pipeline
The app subscribes to Postgres change events via Supabase Realtime. Every INSERT, UPDATE, and DELETE propagates to all connected clients without polling.

```javascript
sb.channel('ideas-ch')
  .on('postgres_changes', {
    event: '*',
    schema: 'public',
    table: 'ideas',
    filter: `user_id=eq.${currentUser.id}`
  }, payload => handleRealtimeChange(payload))
  .subscribe();
```

### 4. Funnel Analysis (Pipeline Metrics)
The analytics dashboard computes conversion rates between pipeline stages — a standard data engineering pattern for measuring throughput.

```
Raw → In Progress → Parked → Shipped
 X        Y%          Z%       W%
```

Conversion rate = `count(next_stage) / count(current_stage) * 100`

### 5. Time-Series Bucketing
Ideas are bucketed by day over a 30-day rolling window to show idea velocity over time — the same pattern used in event analytics pipelines.

```javascript
// Build day buckets for last 30 days
for (let i = days - 1; i >= 0; i--) {
  const d = new Date();
  d.setDate(d.getDate() - i);
  dayBuckets[d.toISOString().slice(0, 10)] = 0;
}
// Assign each idea to its day bucket
ideas.forEach(idea => {
  const day = idea.created_at.slice(0, 10);
  if (dayBuckets[day] !== undefined) dayBuckets[day]++;
});
```

### 6. Aggregation Queries (Client-side)
Status counts, tag frequency distributions, energy breakdowns, and ship rates are computed as in-memory aggregations — mirroring the logic of SQL GROUP BY queries.

---

## Analytics dashboard

The analytics view provides:

- **Pipeline funnel** — idea count and conversion % at each stage
- **30-day timeline** — bar chart of ideas parked per day
- **Energy distribution** — donut chart of low / medium / high energy ideas
- **Status breakdown** — horizontal bar chart of all statuses
- **Top tags** — bar chart of most-used tags
- **Key metrics** — ship rate, ideas this week, oldest raw idea, priority count

---

## Running locally

No build step needed — just open `index.html` in a browser.

```bash
git clone https://github.com/Saumya-Maurya/idea-parking-lot.git
cd idea-parking-lot
open index.html
```

Note: Auth and sync require the Supabase connection. For local development, the existing Supabase project will work as long as you're online.

---

## Extending this project

Some ideas for taking this further as a data engineering project:

- **Audit log table** — log every status change with a timestamp (SCD Type 2 pattern)
- **ETL pipeline** — Python script to pull ideas data, transform it, and load summaries into a separate analytics table
- **GitHub Actions cron** — schedule a daily ETL job to snapshot idea metrics
- **Notion integration** — sync shipped ideas to a Notion database via API
- **AI enrichment** — auto-tag and score ideas using the Claude API when parked

---

## Project structure

```
idea-parking-lot/
└── index.html      # entire app — HTML, CSS, and JS in one file
└── README.md       # this file
```

---

## Author

Saumya Maurya — [github.com/Saumya-Maurya](https://github.com/Saumya-Maurya)
