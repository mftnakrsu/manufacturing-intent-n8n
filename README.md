# Company Intent Signals (n8n + Supabase)

![n8n](https://img.shields.io/badge/built%20with-n8n-orange)
![Supabase](https://img.shields.io/badge/db-Supabase-3FCF8E)
![License](https://img.shields.io/badge/license-MIT-blue)



**Goal:** Collect **intent signals** (product launches, partnerships, events, capacity expansion, hiring, M\&A, automation interest) for target manufacturing companies from **Google News RSS**, score them in **n8n**, and **upsert** results into **Supabase**.

* **Stack:** n8n (workflow) + Supabase (Postgres + REST)
* **Source:** **Single source** → Google News RSS
* **Output:** `company_intent` table (dedup by `url` via upsert)

---

## Table of Contents

* [Architecture](#architecture)
* [Requirements](#requirements)
* [Setup](#setup)

  * [Supabase schema](#1-supabase-schema)
  * [n8n workflow](#2-n8n-workflow)
* [Run & Scheduling](#run--scheduling)
* [Queries](#queries)
* [Extensibility](#extensibility)
* [Troubleshooting](#troubleshooting)
* [Security](#security)
* [License](#license)

---

## Architecture

```
Schedule (daily)
   ↓
Set(Companies: 20 manufacturing targets)
   ↓
Split Out (1 item = 1 company)
   ↓
RSS Read  ← "COMPANY manufacturing when:7d"
   ↓
Tag & Score (Code) → label signals + sum score
   ↓
IF (score > 0)
   ↓
HTTP Request → Supabase REST Upsert (on_conflict=url)
```

* **Idempotent:** `url` is UNIQUE + upsert → safe re-runs
* **Flexible:** store full RSS item in `raw` (JSONB) so you can reprocess with new rules later

---

## Requirements

* n8n (Cloud or Self-hosted)
* Supabase project + **Service Role Key** (easiest if RLS is enabled)
* (Optional) GitHub repo for workflow JSON + docs

---

## Setup

### 1) Supabase schema

Run once in SQL editor:

```sql
create table if not exists company_intent (
  id bigserial primary key,
  company text not null,
  signal_type text,
  title text,
  url text not null,
  published_at timestamptz,
  source text not null default 'GoogleNewsRSS',
  score int not null default 0,
  raw jsonb not null default '{}'::jsonb,
  created_at timestamptz not null default now()
);

create unique index if not exists uq_company_intent_url on company_intent(url);
create index if not exists idx_company_intent_published_at on company_intent(published_at desc);
create index if not exists idx_company_intent_company on company_intent(company);
```

> Filling `company`: either (a) pass it through from upstream nodes, or (b) **detect in Tag & Score** from title/summary (fallback included below).

---

### 2) n8n workflow

#### 2.1 Set (Companies)

Use a single **Set** node to define your 20 targets:

```json
{
  "companies": [
    "Bosch", "Siemens", "ABB", "Schneider Electric", "Rockwell Automation",
    "Honeywell", "Emerson", "GE Vernova", "Fanuc", "KUKA",
    "Yaskawa", "Mitsubishi Electric", "Danfoss", "SKF",
    "ThyssenKrupp", "Valeo", "Magna", "Continental", "TRUMPF", "DMG Mori"
  ]
}
```

#### 2.2 Split Out

* **Item Lists → Split Out** to fan out the `companies` array.

#### 2.3 companies → company (Code/Function Item)

* **Run Once for Each Item**

```js
return { json: { company: $json.companies } };
```

#### 2.4 RSS Read

* **Node:** RSS Read
* **URL (Expression, single line):**

```js
={{ 'https://news.google.com/rss/search?q='
  + encodeURIComponent($json.company + ' manufacturing when:7d')
  + '&hl=en&gl=US&ceid=US:en' }}
```

* **Options:** Continue On Fail = **ON**

#### 2.5 Tag & Score (Code)

* **Run Once for Each Item**, **Language: JavaScript**

```js
// 1) Company detection (fallback via title/summary)
const CANDS = [
  'Bosch','Siemens','ABB','Schneider Electric','Rockwell Automation','Honeywell','Emerson',
  'GE Vernova','Fanuc','KUKA','Yaskawa','Mitsubishi Electric','Danfoss','SKF',
  'ThyssenKrupp','Valeo','Magna','Continental','TRUMPF','DMG Mori'
];
const hayRaw = (String($json.title||'') + ' ' + String($json.contentSnippet||$json.content||''));
const hay = hayRaw.toLowerCase();
let detected = null;
for (const c of CANDS) {
  const re = new RegExp(`\\b${c.toLowerCase().replace(/[.*+?^${}()|[\\]\\\\]/g,'\\$&')}\\b`);
  if (re.test(hay)) { detected = c; break; }
}
const company = $json.company || detected || 'Unknown';

// 2) Signal rules (+ summed score)
const signals = [
  { type:'hiring',         score:1, kw:['hiring','recruit','vacanc','career','stellen'] },
  { type:'product_launch', score:3, kw:['launch','unveil','introduc','release','new product','rollout'] },
  { type:'partnership',    score:3, kw:['partner','partnership','collaborat','alliance','mou'] },
  { type:'capacity',       score:2, kw:['factory','plant','manufactur','expand','opens','capacity','production line','line expansion'] },
  { type:'event',          score:2, kw:['expo','fair','conference','summit','booth','showcase','exhibition','trade show','tradeshow'] },
  { type:'mna',            score:3, kw:['acquire','acquisition','merger','buyout','takeover'] },
  { type:'automation',     score:1, kw:['automation','industrial automation','robotics','cobot','plc','mes','scada'] },
];

const title = String($json.title || '').toLowerCase();
const desc  = String($json.contentSnippet || $json.content || '').toLowerCase();
const text  = `${title} ${desc}`;

let total=0, kinds=[];
for (const s of signals) if (s.kw.some(k => text.includes(k))) { total += s.score; kinds.push(s.type); }

const publishedISO = $json.pubDate
  ? new Date($json.pubDate).toISOString()
  : ($json.isoDate ? new Date($json.isoDate).toISOString() : null);

const url = $json.link || $json.guid || '';

return {
  json: {
    company,
    signal_type: kinds.length ? kinds.join(',') : 'none',
    title: $json.title || '',
    url,
    published_at: publishedISO,
    source: 'GoogleNewsRSS',
    score: total,
    raw: $json
  }
};
```

#### 2.6 IF (filter meaningful)

* **Condition:** Number → `score` **greater than** `0`
* **True** branch → DB upsert.

#### 2.7 Supabase Upsert (HTTP Request)

* **Method:** `POST`
* **URL:** `https://<PROJECT-REF>.supabase.co/rest/v1/company_intent?on_conflict=url`
* **Send Headers:** ON

  * `apikey: <SERVICE_ROLE_KEY>`
  * `Authorization: Bearer <SERVICE_ROLE_KEY>`
  * `Content-Type: application/json`
  * `Prefer: resolution=merge-duplicates,return=representation`
* **Send Body:** ON → **RAW JSON** (single expression, **array**):

```js
={{ JSON.stringify([{
  company:      $json.company || 'Unknown',
  signal_type:  $json.signal_type || null,
  title:        $json.title || null,
  url:          $json.url,
  published_at: $json.published_at || null,
  source:       'GoogleNewsRSS',
  score:        $json.score || 0,
  raw:          $json.raw || {}
}]) }}
```

> Tip: If your n8n Supabase node lacks native “Upsert”, this HTTP approach is the simplest.
> Alternative inside n8n: **Create (Continue On Fail: ON) → Update (Select `url = {{$json.url}}`)** to emulate upsert.

---

## Run & Scheduling

* **Manual test:** click **Execute workflow** in n8n.
* **Production:** Schedule Trigger → **Daily 08:00 (Europe/Istanbul)** → **Activate**.
* Turn **Continue On Fail: ON** for RSS Read and HTTP Upsert for resilience.

---

## Queries

Last 7 days by signal type:

```sql
SELECT signal_type, COUNT(*) AS signals, SUM(score) AS total_score
FROM company_intent
WHERE published_at >= NOW() - INTERVAL '7 days'
GROUP BY signal_type
ORDER BY total_score DESC, signals DESC;
```

Top companies in 30 days:

```sql
SELECT company, COUNT(*) AS signals, SUM(score) AS score
FROM company_intent
WHERE published_at >= NOW() - INTERVAL '30 days'
GROUP BY company
ORDER BY score DESC
LIMIT 20;
```

Daily score trend:

```sql
SELECT DATE_TRUNC('day', published_at) AS day, SUM(score) AS score
FROM company_intent
GROUP BY day
ORDER BY day DESC;
```

---

## Extensibility

* **Add sources:** company blog RSS, careers RSS (Greenhouse/Lever), industry events RSS.
* Keep **Tag & Score** and **Upsert** as-is; only add/modify the collector node and set `source` accordingly.
* Add alerts (Slack/Email) for `score >= X`.

---

## Troubleshooting

**Duplicate key (url unique):**

* Use upsert: `?on_conflict=url` + header `Prefer: resolution=merge-duplicates`.
* Or emulate: Create (COF ON) → Update (`url = {{$json.url}}`).

**“column "https" does not exist”:**

* You have extra Query Params. Keep **only** `on_conflict=url`.
* Do not leak body values into the query string.

**400 Bad Request:**

* Body must be a **JSON array** (`[ { … } ]`) when calling PostgREST.
* Headers must include `apikey`, `Authorization: Bearer …`, `Content-Type: application/json`, `Prefer: …`.

**company is empty:**

* Add variations to the `CANDS` list (e.g., “Bosch Rexroth”, “Siemens AG”).
* Or pass `company` through from upstream nodes.

---

## License

MIT

---

