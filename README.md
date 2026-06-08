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

- A CompanyEnrich API key. Set it as an environment variable in your project:

  ```bash
  COMPANYENRICH_API_KEY=your_key_here
  ```

  The skill never hardcodes the key - it reads it from the environment. Get a key
  and check your credit balance at [companyenrich.com](https://companyenrich.com).

- Auth on every request: `Authorization: Bearer <COMPANYENRICH_API_KEY>`
- Base URL: `https://api.companyenrich.com`

## What's inside

| File | Purpose |
|------|---------|
| `SKILL.md` | When-to-use, auth, endpoint decision table, search rules, credit costs, REST integration pattern. |
| `ENDPOINTS.md` | Authoritative catalog of all ~45 endpoints with request fields, costs, and enum values. |

Built from the official OpenAPI spec at
`https://api.companyenrich.com/openapi/v1.json`. If CompanyEnrich changes their
API, re-fetch that spec to refresh the skill.

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
