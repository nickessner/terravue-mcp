# TerraVue MCP Server

**Analyze any US home from Claude or ChatGPT: buy vs rent, affordability, and ZIP-level
appreciation — grounded in real data and real simulation, never the model's guess.**

TerraVue is a **remote** MCP server. There is nothing to install, no package to pull, and no
API key. Point your client at the URL and it works.

```
https://api.terravue.app/mcp
```

- **Transport:** streamable HTTP
- **Auth:** none (public research preview)
- **Official registry:** [`app.terravue/terravue`](https://registry.modelcontextprotocol.io/v0/servers?search=terravue)
- **Web app:** <https://www.terravue.app>

---

## Why this exists

Most rent-vs-buy tools apply a national average to your situation and hand back one
confident number. The number is usually wrong in a direction you can't see, because the
inputs that matter most — how *your* ZIP has actually appreciated, what rent actually costs
there, what you give up by not investing the down payment — are exactly what gets averaged
away.

TerraVue computes those from source data and returns a **probability**, not a prediction.

---

## Connecting

### Claude (web, desktop, or mobile)

Settings → **Connectors** → **Add custom connector** → paste `https://api.terravue.app/mcp`.
There is a walkthrough with screenshots at <https://www.terravue.app/connect>.

### Claude Code

```bash
claude mcp add --transport http terravue https://api.terravue.app/mcp
```

### ChatGPT

Settings → **Connectors** → add a custom connector with the same URL. Availability depends
on your plan.

### Any other MCP client

```json
{
  "mcpServers": {
    "terravue": {
      "type": "streamable-http",
      "url": "https://api.terravue.app/mcp"
    }
  }
}
```

---

## Tools

| Tool | Answers | Required |
|---|---|---|
| `analyze_address` | What is this ZIP actually like? | `address` |
| `buy_vs_rent` | Should I buy this, or keep renting? | `home_price`, `monthly_rent` |
| `affordability` | Can I afford this specific home? | `home_price`, `annual_income` |
| `affordable_price` | How much house can I afford? | `annual_income` |
| `compare_areas` | Which of these areas is best? | `addresses` |
| `request_capability` | Records something TerraVue can't do yet | `description` |

### `analyze_address`

Takes a full street address or a bare 5-digit ZIP. Returns the ZIP's appreciation forecast
from FHFA records going back to 1975, an estimated market rent, regional tax and insurance
defaults, and neighborhood signals — school ratings, walkability, education level, PFAS
water-quality findings, and grocery/dining/trail/water proximity.

Given a full street address it also returns sub-ZIP census-tract context: how that specific
pocket's prices, incomes and build era compare to the rest of its ZIP. That is reported as
**position, not forecast** — see [Limitations](#what-it-deliberately-does-not-do).

*Optional:* `bedrooms`, `property_type` (`sfr` | `condo` | `townhome` | `multi`)

### `buy_vs_rent`

A deterministic 30-year simulation plus a **500-scenario Monte Carlo**. Returns the
probability that buying beats renting, the breakeven hold year, monthly cost comparison,
cash to close, and net-worth outcomes by percentile.

The renter isn't assumed to set money on fire: the down payment and any monthly difference
get invested, so both sides are compared on the same money over the same clock.

Pass `address` to ground the simulation in that ZIP's real appreciation forecast rather than
a generic assumption.

*Optional:* `address`, `down_payment_pct`, `fixed_rate`, `analysis_years`, `hoa_monthly`,
`property_tax_rate`, `home_appreciation_rate`, `investment_return_rate`

### `affordability`

What a lender actually looks at. Returns the monthly PITI payment, front-end and back-end
DTI ratios, the income needed at a 28% front-end ratio, estimated after-tax money left over
each month, the cash cushion remaining after closing, and a plain
**Comfortable / Qualifiable-tight / Stretch** verdict.

DTI alone answers "will they lend me this." The leftover-per-month figure answers "does this
fit my life," which is usually the question people actually mean.

*Optional:* `address`, `down_payment_pct`, `fixed_rate`, `hoa_monthly`, `monthly_debts`,
`monthly_expenses`, `liquid_savings`, `marginal_tax_rate`, `property_tax_rate`,
`insurance_rate`

### `affordable_price`

The reverse of `affordability`: solves for the maximum home price under lender DTI limits
and your cash on hand, and tells you **which limit binds** — income or cash.

*Optional:* `address`, `down_payment_pct`, `fixed_rate`, `hoa_monthly`, `monthly_debts`,
`liquid_savings`, `front_end_dti_pct`, `back_end_dti_pct`, `property_tax_rate`,
`insurance_rate`

### `compare_areas`

Ranks several ZIPs or addresses side by side on appreciation, school rating, walkability,
education level and water quality, with each figure grounded in the same source data as the
single-area tools.

*Optional:* `bedrooms`

---

## Try it

Once connected, ask in plain language:

> Should I buy a $650k house in 80302 if I'm paying $2,800 in rent?

> I make $140k and have $80k saved. What can I actually afford?

> Compare 78704, 80302 and 27514 for schools and appreciation.

> Is 11375 a good place to buy right now? What's the rent like?

> We're deciding between buying at $500k or renting at $2,400 for the next 6 years.

---

## Where the numbers come from

| Signal | Source |
|---|---|
| Home price appreciation | FHFA House Price Index, ZIP5 level, 1975–2025 (~670k observations) |
| Market rent | Zillow ZORI, plus Census ACS by bedroom count |
| Property tax & insurance | Regional defaults by state |
| Schools | SchoolDigger ratings |
| Water quality | EPA UCMR5 PFAS findings by water system |
| Education, income, build era | Census ACS, at census-tract level |
| Walkability | Walk Score |
| Grocery / dining / trails / water | OpenStreetMap |

Appreciation is a **blend of 60% long-run and 40% recent-3yr CAGR** for the ZIP, adjusted
for local income growth and the macro rate environment.

---

## What it deliberately does not do

These are design decisions, not missing features.

- **It does not predict the future.** The appreciation figure is an estimate built from
  history and labelled as one. A 500-run Monte Carlo returns a distribution precisely
  because a single number would be a lie about certainty.

- **It does not forecast individual neighborhoods.** Sub-ZIP appreciation edges
  **mean-revert** — in our own census-tract analysis, past outperformance correlates around
  −0.3 with future outperformance. So sub-ZIP data is reported as *where this home sits
  within its ZIP today*, never as a prediction about that pocket.

- **It does not value a specific property.** Rent and price figures are ZIP-level. They are
  good at the market level and bad at the individual-unit level, and the tools say so.

- **It does not read listings.** Give it an address and a price; it will not scrape Zillow.

- **It is not financial advice.** It is a simulator with stated assumptions, all of which
  you can override.

---

## Privacy

The server never receives your conversation — MCP transmits tool calls only.

Usage telemetry is deliberately coarse: which tool was called, the resolved ZIP and state,
and numeric parameters. **Typed street addresses are never stored.** Full policy at
<https://www.terravue.app/privacy>.

---

## Status

Public research preview. No auth, no rate-limit tier beyond basic per-IP throttling, free to
use. If a tool returns something that looks wrong, that is worth reporting —
<support@terravue.app>.

## Links

- Web app — <https://www.terravue.app>
- Connection walkthrough — <https://www.terravue.app/connect>
- Methodology — <https://www.terravue.app>
- Privacy — <https://www.terravue.app/privacy>
- Terms — <https://www.terravue.app/terms>

## License

Documentation and manifest in this repository: MIT. The TerraVue service itself is a hosted
product and its source is not included here.
