# CarTalk Forum Scraper — Automotive Lead Generation Pipeline

## Project Overview

This project is an automated pipeline that scrapes car repair posts from the CarTalk community forum, analyzes them using AI, and saves structured data to a Supabase database. It automatically extracts shop and mechanic information for lead generation purposes.

---

## Architecture

```
CarTalk Forum → n8n Workflow → Groq AI → Supabase Database
```

### Tech Stack

| Tool | Purpose |
|---|---|
| **n8n** | Workflow automation |
| **Groq AI (llama-3.3-70b)** | Content classification & data extraction |
| **Supabase** | PostgreSQL database |
| **CarTalk API** | Forum data source |

---

## How It Works

### Step 1 — Schedule Trigger
The workflow runs automatically every 60 minutes.

### Step 2 — Fetch Forum Data
Fetches the latest repair topics from the CarTalk forum:
```
GET https://community.cartalk.com/c/repair-and-maintenance/6.json
```

### Step 3 — Parse Topics
Builds a list of up to 30 topic URLs. The generic category page (ID: 12) is filtered out automatically.

### Step 4 — Groq AI Analysis
Each topic title is analyzed by the AI to extract:
- Intent classification
- Vehicle information (year, make, model, engine)
- DTC codes and symptoms
- Lead detection (shops, mechanics, contact info)

### Step 5 — Combine Data
Merges the AI response with the topic metadata into a single structured object.

### Step 6 — Save to Database
Data is saved to three tables:
- `forum_topics` — record of every processed topic
- `issues` — structured repair data
- `leads` — shop and mechanic contact information

---

## Database Schema

### forum_topics
```sql
CREATE TABLE forum_topics (
  id SERIAL PRIMARY KEY,
  topic_id BIGINT UNIQUE NOT NULL,
  url TEXT,
  title TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### issues
```sql
CREATE TABLE issues (
  id SERIAL PRIMARY KEY,
  topic_id BIGINT UNIQUE,
  source TEXT DEFAULT 'cartalk',
  url TEXT,
  title TEXT,
  intent TEXT,
  dtc_codes JSONB,
  symptoms JSONB,
  vehicle_json JSONB,
  raw_markdown TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### leads
```sql
CREATE TABLE leads (
  id SERIAL PRIMARY KEY,
  topic_id BIGINT,
  url TEXT,
  name TEXT,
  city TEXT,
  region TEXT,
  phone TEXT,
  email TEXT,
  website TEXT,
  confidence FLOAT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

---

## AI Classification

### Intent Types

| Intent | Description |
|---|---|
| `diagnostic_question` | User asking about a car problem |
| `repair_success_story` | A problem that has been fixed |
| `shop_recommendation` | Recommending a specific shop |
| `shop_request` | Looking for a mechanic or shop |
| `other` | Everything else |

### Lead Detection Rules

| Condition | Confidence Score |
|---|---|
| Phone / Email / Website found | 0.9 |
| Shop name + City found | 0.7 |
| Shop name only | 0.5 |
| No lead detected | 0.0 |

A lead is only saved when `is_lead = true`, which requires at least one identifying detail about a business or mechanic.

---

## Setup Guide

### Prerequisites
You will need accounts on the following platforms:
- [n8n.io](https://n8n.io) — workflow automation (cloud or self-hosted)
- [Supabase](https://supabase.com) — database (free tier available)
- [Groq](https://console.groq.com) — AI API (free tier available)

---

### Step 1 — Supabase Setup
1. Create a new Supabase project
2. Open the **SQL Editor** and run the three table creation scripts above
3. Go to **Settings → API** and copy your Project URL and Service Role Key

---

### Step 2 — Groq API Key
1. Go to [console.groq.com](https://console.groq.com)
2. Sign up with Google or GitHub
3. Go to **API Keys** and create a new key
4. Copy the key — it is only shown once

---

### Step 3 — Import Workflow into n8n
1. Open n8n
2. Click **"..."** menu → **"Import from file"**
3. Select the `cartalk_final_v3.json` file
4. The workflow will appear with all nodes connected

---

### Step 4 — Configure Credentials

**Supabase credential in n8n:**

| Field | Value |
|---|---|
| Host | `your-project-id.supabase.co` |
| Service Role Secret | From Supabase → Settings → API |

**Groq API key in n8n:**
Open the **Groq AI** node and update the Authorization header:
```
Bearer YOUR_GROQ_API_KEY
```

---

### Step 5 — Test the Workflow
1. Click **"Execute Workflow"** in n8n
2. All nodes should turn green
3. Check your Supabase tables for saved data

---

## Workflow Nodes

| Node | Function |
|---|---|
| Schedule Trigger | Runs the workflow every hour |
| Get Forum | Calls the CarTalk API |
| Parse Topics | Builds the topic URL list |
| Groq AI | Sends each title to the AI for analysis |
| Combine Data | Merges AI output with topic metadata |
| Save Topic | Inserts row into `forum_topics` |
| Save Issue | Inserts row into `issues` |
| Is Lead? | Checks if `is_lead = true` |
| Save Lead | Inserts row into `leads` (only when lead found) |

---

## Useful SQL Queries

### View All Data
```sql
SELECT * FROM forum_topics ORDER BY created_at DESC;
SELECT * FROM issues ORDER BY created_at DESC;
SELECT * FROM leads ORDER BY confidence DESC;
```

### View All Leads with Post Titles
```sql
SELECT 
  ft.title,
  l.name,
  l.city,
  l.phone,
  l.email,
  l.website,
  l.confidence
FROM leads l
JOIN forum_topics ft ON ft.topic_id = l.topic_id
ORDER BY l.confidence DESC;
```

### View Diagnostic Issues
```sql
SELECT 
  title,
  intent,
  dtc_codes,
  symptoms,
  vehicle_json
FROM issues
WHERE intent = 'diagnostic_question'
ORDER BY created_at DESC;
```

### Count Records Per Table
```sql
SELECT COUNT(*) AS total FROM forum_topics;
SELECT COUNT(*) AS total FROM issues;
SELECT COUNT(*) AS total FROM leads;
```

### Reset All Tables
```sql
TRUNCATE TABLE forum_topics;
TRUNCATE TABLE issues;
TRUNCATE TABLE leads;
```

---

## Troubleshooting

### Groq Rate Limit (Error 429)
The free Groq tier has rate limits. In the **Groq AI** node, go to **Settings** and configure:
- Retry On Fail: ON
- Max Tries: 3
- Wait Between Tries: 3000ms

### Duplicate Key Error
This happens when the same topic is processed twice. Truncate the tables and re-run, or add a unique constraint with `ON CONFLICT DO NOTHING` in Supabase.

### NULL Values in Issues Table
Check the **Combine Data** node. Make sure it reads from `$('Parse Topics').item.json` for topic data and `$json.choices[0].message.content` for the Groq response.

### Only 1 Row Saving
This is caused by a `Get a row` or `IF` node processing only the first item. Remove those nodes and connect `Parse Topics` directly to `Groq AI` so all topics are processed.

---

## Project Files

```
cartalk-scraper/
├── cartalk_final_v3.json    # n8n workflow (import this into n8n)
├── README.md                # This file
```

---

## Important Notes

- CarTalk forum posts are mostly technical discussions — leads are less frequent but do appear in shop recommendation threads
- The Groq free tier has rate limits — sending many requests simultaneously may trigger a 429 error; the built-in retry logic handles this automatically
- Topic ID 12 is the generic category page and is automatically excluded from processing
- The workflow checks for new topics every 60 minutes once activated
- To activate the workflow, toggle the **Active** switch in the top right corner of the n8n workflow editor
