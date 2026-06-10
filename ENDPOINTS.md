# CompanyEnrich - Full Endpoint Reference

Authoritative source: `https://api.companyenrich.com/openapi/v1.json` (re-fetch if
this drifts). Base URL `https://api.companyenrich.com`. Auth header on every call:
`Authorization: Bearer <COMPANYENRICH_API_KEY>`.

Cost legend: credits are deducted per the spec. FREE = no credits.

## Companies - Enrich

| Method + path | Cost | Notes |
|---|---|---|
| `GET /companies/enrich?domain=` | 1/call | Preferred enrichment. Optional `expand=workforce`. |
| `POST /companies/enrich` | 1/call | Enrich by properties (best match). Body = `EnrichRequest`. |
| `POST /companies/enrich/batch` | 1/domain | Up to 50 domains. Body `{ domains: string[] }`. |
| `POST /companies/enrich/bulk` | 1/domain (on completion) | Async, up to 10,000 domains. Returns jobId; webhook on finish. |
| `GET /companies/enrich/bulk/{jobId}` | FREE | Job status; includes `results_url` when done. |
| `GET /companies/enrich/bulk/jobs` | FREE | List bulk jobs. `page,pageSize,status`. |

`EnrichRequest` (POST by properties) - provide at least one of:
`name`, `linkedinUrl`, `linkedinId`, `twitterUrl`, `facebookUrl`,
`instagramUrl`, `youTubeUrl`.

## Companies - Search

| Method + path | Cost | Notes |
|---|---|---|
| `POST /companies/search` | 1/result | Main search. Body = `CompanySearchPageInput`. Max `page*pageSize` = 10,000. |
| `POST /companies/search/preview` | FREE | Top 25 only. Scale plan required. Body = `CompanySearchInput`. |
| `POST /companies/search/count` | FREE | Total match count. Body = `CompanySearchInput`. |
| `POST /companies/search/scroll` | 1/result | Cursor pagination beyond 10k. Body = `CompanySearchScrollInput`. |
| `POST /companies/search/async` | 1/result (reserved) | Export up to 50,000. Returns jobId. |
| `GET /companies/search/async/{jobId}` | FREE | Export job status + `results_url`. |
| `GET /companies/search/async/jobs` | FREE | List export jobs. |

### CompanySearchInput / CompanySearchPageInput fields

`CompanySearchPageInput` = `CompanySearchInput` + `page` (int, >0) and `pageSize`
(int, 1-100).

Text / semantic:
- `semanticQuery: string` - natural-language description. Preferred. Do not combine with `query`.
- `query: string` - matches company name + domain only.
- `semanticWeight: number` (0-1, default 0.7) - higher favors semantic similarity.

Location (hard filters):
- `countries: string[]` - ISO 3166-1 alpha-2 codes. Lowercase works and is what the
  API examples use, e.g. `["us","gb"]`.
- `regions: string[]` - region IDs (see `GET /geo/regions`).
- `states: number[]` - state IDs (resolve via `POST /geo/states`).
- `cities: number[]` - city IDs (resolve via `POST /geo/cities`).

Firmographics (enum arrays - exact values below):
- `employees: CompanyEmployees[]`
- `revenue: CompanyRevenue[]`
- `category: CompanyCategory[]` + `categoryOperator: "And" | "Or"` (default `And`)
- `type: CompanyType[]`
- `fundingRounds: CompanyFundingRound[]`
- `foundedYear: { min?, max? }`
- `fundingYear: { min?, max? }`
- `fundingAmount: { min?, max? }` (CompanyFundingAmountRange, USD)

Topical:
- `keywords: string[]` + `keywordsOperator: "And" | "Or"` - validate via `GET /keywords/autocomplete`.
- `technologies: string[]` + `technologiesOperator: "And" | "Or"` - validate via `GET /technologies/autocomplete`.
- `naicsCode: number[]` - 2 to 6 digit NAICS codes. A shorter code includes all 6-digit codes beneath it.

Workforce (note: department here uses UNDERSCORE slugs, unlike people `department`):
- `workforceSize: [{ department?, min?, max? }]` - absolute headcount filter.
- `workforceGrowth: { period?, department?, minPercent?, maxPercent? }` - growth filter.
  - `period`: `m6` | `y1` | `y2`
  - `department` (both fields): `overall`, `c_suite`, `product_management`,
    `engineering_technical`, `design`, `education`, `finance`, `human_resources`,
    `information_technology`, `legal`, `marketing`, `medical_health`, `operations`,
    `sales`, `consulting`

Other:
- `lists: string[]` - restrict to saved list IDs.
- `require: FeatureRequirement[]` - features that must exist on the company. Values:
  `linkedin`, `twitter`, `facebook`, `instagram`, `angellist`, `crunchbase`,
  `youtube`, `country`, `city`, `state`, `revenue`, `foundedYear`, `anyFunding`.
- `exclude: CompanyExcludeFilters` - negative filters; same shape: `domains,
  regions, countries, states, cities, type, category, employees, revenue,
  naicsCode, keywords, technologies, fundingRounds`. A company matching any
  exclude filter is dropped.

Response: `{ items: [...], page, totalPages, totalItems }`. `totalItems` is capped
(historically at 10,000) - it is not always the true match count; use `count`.
Each item's `location` is structured: `{ country:{code,name,latitude,longitude},
state, city:{id,name,...}, address, postal_code, phone }`.

## Companies - Similar

| Method + path | Cost | Notes |
|---|---|---|
| `POST /companies/similar` | 5/result | Body = `CompanySimilarPageInput`. Up to 100 per call. |
| `POST /companies/similar/preview` | FREE | Top 25. Scale plan. Body = `CompanySimilarInput2`. |
| `POST /companies/similar/count` | FREE | Match count. |
| `POST /companies/similar/scroll` | 5/result | Cursor pagination. |

`CompanySimilarInput2` core fields:
- `domains: string[]` - seed companies, 1 to 10 (required).
- `similarityWeight: number` (-1 to 1, default 0) - higher favors more-similar,
  lower favors more-established companies.
- `minScore: number` (0-1) - drops weak matches.
- Plus all the same firmographic / location / topical / `exclude` filters as
  company search above.

At 5 credits/result, default `pageSize` <= 10 unless told otherwise. The response
includes `metadata.scores` mapping company id -> similarity score (0-1) - surface
scores when presenting. `POST /companies/similar/count` is free.

## Companies - Single + utility

| Method + path | Cost | Notes |
|---|---|---|
| `GET /companies?id=` | 1/call | Get one company by CompanyEnrich ID. Optional `expand=workforce`. |
| `GET /companies/workforce?id=` or `?domain=` | 5/call | Workforce insights (headcount, range, growth). Exactly one of id/domain. |
| `GET /companies/autocomplete?query=` | FREE | Up to 10 domain matches for a partial name. |

## People

| Method + path | Cost | Notes |
|---|---|---|
| `POST /people/search` | 2/result | Body = `PersonSearchPageInput`. Max `page*pageSize`=10,000. |
| `POST /people/search/preview` | FREE | Top 10. Scale plan. |
| `POST /people/search/scroll` | 2/result | Cursor pagination beyond 10k (`cursor`). |
| `POST /people/search/async` | 2/result (reserved) | Export up to 50,000. |
| `GET /people/search/async/{jobId}` / `.../jobs` | FREE | Export job status / list. |
| `POST /people/lookup` | 5/call | Body `{ email }`. Resolves person from email. |
| `GET /people?id=` | 2/call | Get one person by CompanyEnrich ID. |
| `GET /people/email?id=&domain=` | 10/email (beta) | Resolve a work email. |
| `POST /people/email/bulk` + `GET /people/email/bulk/{jobId}` | up to 10/item (beta) | Bulk email enrichment, up to 1,000 items: `{ items: [{ person_id, domain? }], webhook_url? }`. |

### PersonSearchInput / PersonSearchPageInput fields

`PersonSearchPageInput` = `PersonSearchInput` + `page`, `pageSize` (1-100).

- `positionQuery: string[]` - queries against current job title. Use this for
  role/title search, NOT `query`.
- `query: string` - matches the company name + domain (scopes by company).
- `seniority: PersonSeniority[]`
- `department: PersonDepartment[]` (large taxonomy, e.g. `engineering-technical/data-science`)
- `countries: string[]` - alpha-2.
- `domains: string[]` - up to 100 company domains to scope people to.
- `companyFilter: { search?: CompanySearchInput, similar?: CompanySimilarInput }`
  - scope people by a company search/similar query.
- Date filters (ISO date-times): `atCurrentCompanyAfter/Before` (company join date),
  `atCurrentPositionAfter/Before` (position start). Good for "recently hired VPs".
- `exclude: { positionQuery, countries, department, domains, seniority }`.

`companyFilter` costs nothing extra - you only pay for the people returned. There
is no free `count` for people; use `preview` (Scale plan, top 10) or a
`pageSize: 1` probe to gauge a query before a large pull, and confirm spend.

**PersonInfo response** has NO email (`id`, `name`, `first_name`, `last_name`,
`location`, `socials`, `experiences`, `position`, `seniority`, `department`,
`image_url`). Emails come only from the work-email finder below.

**Work-email finder response** (`GET /people/email`):
`{ person, person_id, domain, status, email, certainty }`.
- `status`: `pending`, `found`, `not-found`, `unavailable`. On `pending`, retry the
  same call after a short wait - re-accessing an already-found email is not
  re-charged (only newly accessed emails cost credits).
- `certainty`: `unknown`, `medium`, `high`, `very-high`. Always report it with the email.
- `domain` is optional; if omitted, the person's experience domains are tried in
  seniority order. If provided, it must match one of the person's experiences.

## Pagination, scroll and async export (shapes)

**Page-based response:** `{ items, page, totalPages, totalItems }`.

**Scroll response:** `{ items, metadata: { totalMatching }, nextCursor,
previousCursor, totalItems, startIndex }`. Omit `cursor` on the first call, then
pass the returned `nextCursor` as `cursor`. Each scroll page bills per result -
track cumulative spend against what the user approved when looping.

**Async export body** (`POST /companies/search/async`):
`{ count: <required>, webhook_url?: string, search?: {filter body}, similar?: {similar body} }`.
Credits are reserved when the job is created - always confirm with the user first.
Poll `GET /companies/search/async/{jobId}` -> `status`, `requested_count`,
`total_matching`, `exported_count`, `credits_charged`, `results_url` (download when
`status` = `completed`). People async export is the same shape with people filters.

`status` (JobStatus): `pending`, `processing`, `completing`, `completed`, `failed`.

## Lists (saved company lists)

| Method + path | Notes |
|---|---|
| `GET /lists/companies` | All lists for the user. |
| `POST /lists/companies` | Create a list from a search or similar query. Costs same as the underlying search. Body = `CreateListInput`. |
| `GET /lists/companies/{id}` | Get one list. |
| `PUT /lists/companies/{id}` | Update list properties. |
| `DELETE /lists/companies/{id}` | Delete list + entries. |

## Geo (resolve IDs for state/city/region filters) - all FREE

| Method + path | Notes |
|---|---|
| `GET /geo/regions` | All regions (region IDs for `regions` filter). |
| `GET /geo/countries/{countryCode}` | Country by alpha-2 code. |
| `POST /geo/countries` | Search countries by name (<=100/page). |
| `POST /geo/states` | Search states by name / country codes -> state IDs. |
| `POST /geo/cities` | Search cities by name / country codes -> city IDs. |

To use `states`/`cities` filters in company search you must first resolve the
integer ID here - the search endpoints take IDs, not names.

## Taxonomy / autocomplete - all FREE

| Method + path | Notes |
|---|---|
| `GET /industries` | Full list of company industries. |
| `GET /keywords/autocomplete?query=` | Valid keyword values for the `keywords` filter. |
| `GET /technologies/autocomplete?query=` | Valid technology values for the `technologies` filter. |
| `GET /positions/autocomplete?query=` | Valid job-position values. |

## Account / jobs - FREE

| Method + path | Notes |
|---|---|
| `GET /me` | API key, credit balance, account capabilities. |
| `GET /jobs` | All async jobs (bulk enrich, exports). Filter `status`, `type`. |
| `GET /jobs/{jobId}` | One job's details. |

## Enum reference

- `CompanyEmployees`: `1-10`, `11-50`, `51-200`, `201-500`, `501-1K`, `1K-5K`, `5K-10K`, `over-10K`
- `CompanyRevenue`: `under-1m`, `1m-10m`, `10m-50m`, `50m-100m`, `100m-200m`, `200m-1b`, `over-1b`
- `CompanyCategory`: `b2b`, `b2c`, `b2g`, `e-commerce`, `media`, `service-provider`, `mobile`, `saas`
- `CompanyType`: `private`, `public`, `self-employed`, `self-owned`, `partnership`, `nonprofit`, `educational`, `government`
- `CompanyFundingRound`: `seed`, `debt_financing`, `angel`, `venture`, `series_a`..`series_h`, `other`
- `PersonSeniority`: `owner`, `founder`, `c-suite`, `partner`, `vp`, `head`, `director`, `manager`, `senior`, `entry`, `intern`
- `PersonDepartment`: large nested taxonomy with HYPHEN slugs - root values
  (`c-suite`, `engineering-technical`, `product-management`, `design`, `finance`,
  `marketing`, `sales`, `human-resources`, `operations`, `legal`, `education`,
  `medical-health`, `information-technology`, `consulting`) plus sub-departments
  like `engineering-technical/data-science` or `c-suite/marketing-executive`. A
  root value includes all its sub-departments. Pull exact sub-values from the
  spec's `PersonDepartment` enum, or map a title phrase via `GET /positions/autocomplete`.
  (Do not confuse with the workforce `department` enums above, which use UNDERSCORES.)
- `FeatureRequirement` (for `require`): `linkedin`, `twitter`, `facebook`, `instagram`, `angellist`, `crunchbase`, `youtube`, `country`, `city`, `state`, `revenue`, `foundedYear`, `anyFunding`
- Filter operators (`categoryOperator`, `keywordsOperator`, `technologiesOperator`): `And` (default) | `Or` - capitalized.
- `JobStatus` (async/bulk jobs): `pending`, `processing`, `completing`, `completed`, `failed`

Wrong enum/field values are silently ignored (no 400). If a filter seems to have
no effect, verify the exact value against this list or the relevant autocomplete
endpoint.
