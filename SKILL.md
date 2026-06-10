---
name: companyenrich
description: >-
  Use CompanyEnrich's full API (company + people data) in any project. Covers
  enrichment by domain, semantic company search, similar-company lookup, people
  search/lookup, work-email finding, lists, geo and taxonomy autocompletes.
  Invoke whenever the user wants to find, enrich, or integrate B2B
  company/people data, build lead-gen or prospecting features, or wire
  CompanyEnrich endpoints into their app. The bundled ENDPOINTS.md is the
  authoritative endpoint + field reference.
---

# CompanyEnrich

A B2B data API: enrich a company from its domain, search companies by natural
language, find similar companies, search/lookup people, and find work emails.
This skill lets you use every endpoint - either ad-hoc through the
`companyenrich` MCP tools, or by wiring the REST API into a user's project.

Base URL: `https://api.companyenrich.com`
Spec: `https://api.companyenrich.com/openapi/v1.json` (re-fetch if something here looks stale).

## Authentication

Resolve the API key in this order:

1. `COMPANYENRICH_API_KEY` environment variable
2. The file `~/.companyenrich/api_key`
3. A key already in conversation context (project instructions, memory, or earlier in the chat)
4. Ask the user for their key (generated at https://companyenrich.com/dashboard)

Every request sends `Authorization: Bearer <key>` and `Content-Type: application/json`.

```bash
KEY="${COMPANYENRICH_API_KEY:-$(cat ~/.companyenrich/api_key 2>/dev/null)}"
```

When the key comes from source 3 or 4, validate it with the free `GET /me`
(a 401 means it is wrong - do not store it), then persist it:

```bash
mkdir -p ~/.companyenrich && printf '%s' "$KEY" > ~/.companyenrich/api_key && chmod 600 ~/.companyenrich/api_key
```

In ephemeral sandboxes (e.g. the Claude chat app) the filesystem resets between
chats, so the stored file will not survive. The first time a user provides a key
there, tell them once to add it to their project instructions / profile so they
do not re-enter it each session. Never print the key back or include it in
summaries - refer to it only as `$KEY`. In project code, read it from an env var,
never hardcode it.

## Two ways to use it

1. **MCP tools** (`mcp__companyenrich__*`) - fast for exploration and one-off
   queries inside Claude. Available: `enrich_company`, `search_companies`,
   `find_similar_companies`, `search_people`. These cover the 4 core endpoints only.
2. **REST API** - for embedding into a project (Next.js route, script, backend).
   Gives access to ALL endpoints, including the ones MCP does not expose (batch,
   bulk/async jobs, count, scroll, workforce, lists, geo, autocompletes, people
   email). See `ENDPOINTS.md` for the full catalog.

When the user wants a feature in their app, write REST integration code. When
they just want answers/data now, prefer the MCP tools.

## Credits and spend policy

Most calls spend credits from the user's balance. `GET /me` (free) returns the
remaining balance and plan `capabilities`. Check it on first use in a session and
tell the user their balance.

**Spend rules:**
- Small calls (single enrich/lookup, or a search expected to cost <= 25 credits): just run it.
- Before anything larger - bulk jobs, async exports, scroll loops, similar-company
  searches with big page sizes, or any search where `results x cost-per-result`
  could exceed ~25 credits - first run the **free** `POST /companies/search/count`
  (same filter body) to get the exact match count, state the cost, and get the
  user's confirmation.
- Searches charge per result returned. Keep `pageSize` to what the user actually
  needs. Never default to 100.
- `expand=workforce` adds **5 credits per company** - only use it when the user
  explicitly wants workforce/headcount data.
- Preview endpoints are free but need a Scale plan (`capabilities` from `/me`); if
  unavailable, fall back to the free count endpoint.

| Action | Cost |
|---|---|
| `GET /me`, counts, previews, autocompletes, geo/industry lookups, lists CRUD, job-status reads | FREE |
| Company enrich (single / batch / bulk) | 1 credit / company |
| Company search / scroll / async export | 1 credit / company returned |
| Similar companies | 5 credits / company returned |
| People search / scroll / async export | 2 credits / person returned |
| Person lookup by email | 5 credits / successful match |
| Work-email finder (`GET /people/email`, beta) | 10 credits / newly found email |
| Workforce insights (`/companies/workforce` or `expand=workforce`) | 5 credits / company |

## Pick the right endpoint

| Goal | Endpoint |
|------|----------|
| Profile for a known domain | `GET /companies/enrich?domain=` |
| Enrich many domains (<=50) | `POST /companies/enrich/batch` |
| Enrich thousands of domains (<=10k, async) | `POST /companies/enrich/bulk` |
| Find companies from a natural-language description | `POST /companies/search` (use `semanticQuery`) |
| Count matches first (free) | `POST /companies/search/count` |
| Pull more than 10k company results | `POST /companies/search/scroll` |
| Find companies similar to one or more domains | `POST /companies/similar` |
| Search people by title/seniority/department | `POST /people/search` |
| Resolve a person from their email | `POST /people/lookup` |
| Find a person's work email | `GET /people/email` (beta) |
| Resolve country/state/city IDs for filters | `POST /geo/countries`,`/geo/states`,`/geo/cities` |
| Validate keyword/technology/position/industry values | `GET /*/autocomplete`, `GET /industries` |
| Check credit balance | `GET /me` (free) |

## Search rules (prevent wasted credits and errors)

- Prefer `semanticQuery` (natural language, searches inside profiles) over `query`
  (matches company name + domain only). Do NOT send both. `semanticWeight` is
  0-1 (default 0.7, higher favors semantic similarity).
- **Count before searching.** `POST /companies/search/count` is free and takes the
  exact same filter body. Quote the match count and intended spend before a large run.
- **Resolve filter IDs first.** `states`/`cities` take integer IDs, `regions` take
  region IDs - resolve via the free geo endpoints, never guess. `countries` are
  plain ISO-2 codes. Validate `keywords`/`technologies` via the free autocompletes.
- Keep filters light. Stacking many hard filters shrinks results fast; let the
  semantic layer handle topical intent.
- **Enum values must match exactly** (e.g. `"51-200"`, `"10m-50m"`, `"c-suite"`).
  Wrong values are silently ignored, not rejected. Full lists in `ENDPOINTS.md`.
- Pagination: page-based search allows `page * pageSize <= 10,000`. Beyond that use
  `scroll` (cursor-based, pass the returned `nextCursor`). For thousands of
  results, use async export jobs instead of looping.

## Error handling

Errors are RFC 7807 problem-details JSON (`title`, `detail`, `status`).

- `401` - invalid/expired key: re-ask the user, update `~/.companyenrich/api_key`.
- `402` - out of credits: report balance from `/me` and stop; user must top up.
- `404` - company/person not found (e.g. domain not in DB): report plainly.
- `422` - validation error: `detail`/`errors` say which filter is malformed; fix and retry.
- `429` - rate limited: back off and retry. Limit is ~500 requests/min per IP plus
  per-plan route limits; prefer batch endpoints over loops of single calls.

## Presenting results

Company and person payloads are large. Summarize only the fields relevant to the
user's ask (name, domain, industry, employees, revenue, location, key
technologies; or person name, position, seniority, company). When there are more
than a handful of records, offer to save full results to JSON/CSV instead of
dumping raw JSON into the conversation.

## Project integration pattern

Read the key from an env var, never hardcode it. Minimal fetch helper:

```js
async function ce(path, body, method = "POST") {
  const res = await fetch(`https://api.companyenrich.com${path}`, {
    method,
    headers: {
      Authorization: `Bearer ${process.env.COMPANYENRICH_API_KEY}`,
      "Content-Type": "application/json",
    },
    body: method === "GET" ? undefined : JSON.stringify(body),
  });
  if (!res.ok) throw new Error(`CompanyEnrich ${path}: ${res.status}`);
  return res.json();
}

// count first (free), then search
const { totalItems } = await ce("/companies/search/count", {
  semanticQuery: "AI startups in healthcare focused on medical imaging",
  countries: ["us"],
  employees: ["11-50", "51-200"],
});
const { items } = await ce("/companies/search", {
  semanticQuery: "AI startups in healthcare focused on medical imaging",
  countries: ["us"],
  employees: ["11-50", "51-200"],
  page: 1,
  pageSize: 25,
});

// enrich by domain (GET)
const company = await ce("/companies/enrich?domain=stripe.com", null, "GET");
```

Search response shape: `{ items, page, totalPages, totalItems }`.
`location.country` and `location.city` are objects (`{code,name,...}`), not
strings - flatten when displaying.

## Before writing integration code

1. Read `ENDPOINTS.md` for the exact endpoint, fields, and enum values.
2. If a project already calls CompanyEnrich, match its existing helper conventions.
3. Validate filter values: enum fields from `ENDPOINTS.md`; keywords / technologies
   / positions via their autocomplete endpoints; state/city IDs via the geo
   endpoints (search by name to get the integer ID).
