---
name: companyenrich
description: >-
  Use CompanyEnrich's full API (company + people data) in any project. Covers
  enrichment by domain, semantic company search, similar-company lookup, people
  search/lookup, lists, geo and taxonomy autocompletes. Invoke whenever the user
  wants to find, enrich, or integrate B2B company/people data, build lead-gen or
  prospecting features, or wire CompanyEnrich endpoints into their app. The
  bundled ENDPOINTS.md is the authoritative endpoint + field reference.
---

# CompanyEnrich

A B2B data API: enrich a company from its domain, search companies by natural
language, find similar companies, and search/lookup people. This skill lets you
use every endpoint - either ad-hoc through the `companyenrich` MCP tools, or by
wiring the REST API into a user's project.

Base URL: `https://api.companyenrich.com`
Auth: `Authorization: Bearer <COMPANYENRICH_API_KEY>` on every request.
Spec: `https://api.companyenrich.com/openapi/v1.json` (re-fetch if something here looks stale).

## Two ways to use it

1. **MCP tools** (`mcp__companyenrich__*`) - fast for exploration and one-off
   queries directly inside Claude. Available: `enrich_company`,
   `search_companies`, `find_similar_companies`, `search_people`. These cover the
   4 core endpoints only.
2. **REST API** - for embedding into a project (Next.js route, script, backend).
   Gives access to ALL endpoints, including the ones MCP does not expose (batch,
   bulk/async jobs, count, scroll, workforce, lists, geo, autocompletes, people
   email). See `ENDPOINTS.md` for the full catalog.

When the user wants a feature in their app, write REST integration code. When
they just want answers/data now, prefer the MCP tools.

## Pick the right endpoint

| Goal | Endpoint |
|------|----------|
| I know the company's domain, want its full profile | `GET /companies/enrich?domain=` |
| Enrich many domains at once (<=50) | `POST /companies/enrich/batch` |
| Enrich thousands of domains (<=10k, async) | `POST /companies/enrich/bulk` |
| Find companies from a natural-language description | `POST /companies/search` (use `semanticQuery`) |
| Just count matches, free | `POST /companies/search/count` |
| Pull more than 10k company results | `POST /companies/search/scroll` |
| Find companies similar to one or more domains | `POST /companies/similar` |
| Search people by title/seniority/department | `POST /people/search` |
| Resolve a person from their email | `POST /people/lookup` |
| Find a person's work email | `GET /people/email` (beta) |
| Resolve country/state/city IDs for filters | `POST /geo/countries`,`/geo/states`,`/geo/cities` |
| Validate keyword/technology/position/industry values | `GET /*/autocomplete`, `GET /industries` |
| Check credit balance | `GET /me` (free) |

## Search rules (important)

- Prefer `semanticQuery` (natural language, searches inside company profiles)
  over `query` (matches company name + domain only). Do NOT send both.
- Keep filters light. Stacking many hard filters shrinks results fast. Let the
  semantic layer handle topical intent; use structured filters only for hard
  constraints (country, employee bucket, revenue, founded year).
- `semanticWeight` (0-1, default 0.7): higher favors semantic similarity.
- Pagination: `page * pageSize` cannot exceed 10000. For more, use the `scroll`
  or `async` variants. `count` returns the real total (capped) for free.
- Filter values are fixed enums - using a wrong label is silently ignored, not
  an error. Always pull exact values from `ENDPOINTS.md` (or the autocomplete
  endpoints for keywords/technologies/positions, and the geo endpoints for
  state/city IDs).

## Credits (per the spec)

- Enrich / company search: 1 credit per company returned (min 1).
- Similar companies: 5 credits per company returned.
- People search: 2 credits per person. People lookup / workforce: 5 credits.
- People email: 10 credits per newly found email (beta).
- `count`, `preview`, autocompletes, geo, `/me`, job-status reads: FREE.

Use the free `count` / `preview` endpoints to validate a query shape before
spending credits on the paid search.

## Project integration pattern

Store the key in an env var (`COMPANYENRICH_API_KEY`), never hardcode it. Minimal
fetch helper:

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

// semantic company search
const { items } = await ce("/companies/search", {
  semanticQuery: "AI startups in healthcare focused on medical imaging",
  countries: ["US"],
  employees: ["11-50", "51-200"],
  page: 1,
  pageSize: 25,
});

// enrich by domain (GET)
const company = await ce(
  "/companies/enrich?domain=stripe.com",
  null,
  "GET"
);
```

Response shape for searches: `{ items: [...], page, totalPages, totalItems }`.
`location.country` and `location.city` are objects (`{code,name,...}`), not
strings - flatten when displaying.

## Before writing integration code

1. Read `ENDPOINTS.md` in this skill folder for the exact endpoint, fields, and
   enum values needed.
2. If a project already calls CompanyEnrich, match its existing helper and field
   conventions instead of introducing a new pattern.
3. Validate filter values: enum fields from `ENDPOINTS.md`; keywords /
   technologies / positions via their autocomplete endpoints; state/city IDs via
   the geo endpoints (search by name to get the integer ID).
