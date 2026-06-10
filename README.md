# CompanyEnrich skill for Claude Code

A [Claude Code](https://claude.com/claude-code) skill that lets you use the full
[CompanyEnrich](https://companyenrich.com) B2B data API - company enrichment,
semantic company search, similar-company lookup, people search/lookup, lists,
geo and taxonomy endpoints - inside any project.

Once installed, Claude knows every CompanyEnrich endpoint, its exact request
fields, enum values, and credit cost, plus a ready-to-use REST integration
pattern. Ask things like "find AI healthcare startups in the US with 11-50
employees" or "enrich stripe.com" or "wire CompanyEnrich search into this app"
and Claude uses the right endpoint.

It is also credit-aware: Claude counts matches with the free endpoint first,
tells you the cost, and asks before any large or expensive call - so you do not
burn credits by accident.

## Install

Clone this repo straight into your Claude Code skills folder:

```bash
git clone https://github.com/zeynepavan/companyenrich-skill ~/.claude/skills/companyenrich
```

Then in Claude Code:

```
/companyenrich
```

Or just describe a CompanyEnrich task - the skill auto-triggers.

To update later:

```bash
cd ~/.claude/skills/companyenrich && git pull
```

To uninstall, delete the folder:

```bash
rm -rf ~/.claude/skills/companyenrich
```

## Requirements

A CompanyEnrich API key (get one and check your balance at
[companyenrich.com](https://companyenrich.com)). The skill resolves it in this
order, so set it whichever way suits you:

1. `COMPANYENRICH_API_KEY` environment variable (recommended for projects)
2. `~/.companyenrich/api_key` file
3. Provide it in chat when Claude asks - it validates and stores it for next time

The skill never hardcodes the key or prints it back. Auth on every request:
`Authorization: Bearer <key>`. Base URL: `https://api.companyenrich.com`.

## What's inside

| File | Purpose |
|------|---------|
| `SKILL.md` | When-to-use, API-key handling, credit/spend policy, endpoint decision table, search rules, error handling, rate limits, and a REST integration pattern. |
| `ENDPOINTS.md` | Authoritative catalog of all ~45 endpoints with request fields, response shapes, costs, and exact enum values. |

Built from the official OpenAPI spec at
`https://api.companyenrich.com/openapi/v1.json`. If CompanyEnrich changes their
API, re-fetch that spec to refresh the skill.

## What the skill enforces

- **Spend safety** - counts matches with the free `count` endpoint before paid
  searches, quotes the cost, and confirms before bulk jobs, exports, or anything
  over ~25 credits. Keeps `pageSize` to what you actually need.
- **Correct filters** - exact enum values, ISO-2 country codes, and resolving
  state/city/region IDs via the free geo endpoints instead of guessing.
- **Robust error handling** - maps `401/402/404/422/429` to clear actions, with
  back-off on rate limits.
- **Clean output** - summarizes the relevant fields instead of dumping raw JSON,
  and offers to save large result sets to JSON/CSV.

## Coverage

- **Companies** - enrich by domain / properties / batch / bulk, search
  (+count/scroll/async), similar companies, get by ID, workforce, autocomplete.
- **People** - search (+scroll/async), lookup by email, get by ID, work-email
  resolution (beta).
- **Lists** - create/read/update/delete saved company lists.
- **Geo** - resolve region/country/state/city IDs for filters.
- **Taxonomy** - industries, keywords, technologies, positions autocompletes.
- **Account** - credit balance and job status.

## License

MIT - see [LICENSE](LICENSE).

> Not affiliated with CompanyEnrich. This is a community skill that documents and
> wraps their public API.
