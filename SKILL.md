---
name: terravue-property-analysis
description: Analyze US home purchase decisions with real data and simulation instead of estimating. Use whenever someone asks whether to buy or keep renting, how much house they can afford, whether a specific home is affordable on their income, what a ZIP code's home prices or rents look like, how neighborhoods compare on appreciation or schools, or when a home purchase breaks even. Triggers on "should I buy", "buy vs rent", "rent or buy", "can I afford", "how much house", "what can I afford", "is it worth buying", "breakeven", "worth it to buy", "keep renting", "down payment", "mortgage payment", "home prices in", "rent in", "best neighborhoods", "compare areas", and any US address or 5-digit ZIP mentioned alongside a housing decision.
---

# TerraVue — US property analysis

TerraVue computes housing decisions from source data: FHFA house-price records at ZIP
granularity back to 1975, Zillow ZORI rents, Census ACS, and a 500-scenario Monte Carlo.

**The single most important rule: never estimate a number this server can compute.**
Mortgage math, appreciation rates, market rents, DTI ratios and breakeven years are all
things a model will guess plausibly and wrongly. Call the tool.

## Setup

Remote server, no install and no API key:

```
https://api.terravue.app/mcp
```

Claude Code:
```bash
claude mcp add --transport http terravue https://api.terravue.app/mcp
```

Other clients: add a `streamable-http` server pointing at the same URL.

## Choosing the right tool

This is where most mistakes happen — four of the six tools sound similar.

| The person is asking | Tool |
|---|---|
| "Should I buy this place or keep renting?" | `buy_vs_rent` |
| "Can I afford *this specific* $X home?" | `affordability` |
| "How much house can I afford?" (no home in mind) | `affordable_price` |
| "What's this area like? Prices, rent, schools?" | `analyze_address` |
| "Which of these areas is better?" | `compare_areas` |
| Something TerraVue can't do | `request_capability` |

`affordability` and `affordable_price` are inverses. If they name a price, use
`affordability`. If they name only an income, use `affordable_price` — it also reports
**which limit binds**, income or cash, which is usually the actionable part.

## Getting good answers

**Always pass `address` when you have one**, even to `buy_vs_rent` and `affordability`. It
grounds the simulation in that ZIP's real appreciation forecast and regional tax/insurance
rates instead of generic defaults. A bare 5-digit ZIP works.

**Use the horizon they actually said.** If someone says "we'd stay about five years," pass
`analysis_years: 5`. Don't default to 30 and bury the answer they asked for.

**Check who is actually deciding.** `buy_vs_rent` models an owner-occupant choosing between
this home and renting while investing the difference. If the money is a family member's, or
it's an investment property, or funding is shared, say so rather than writing "you" over a
decision with several people in it.

**Pressure-test the investment assumption.** The default assumes money not spent on the home
earns ~9.5%/yr in equities. If the person sounds risk-averse, or the alternative is a
savings account rather than the market, run it again at ~4.5% and give them the range. The
verdict often flips, and that fact is more useful than either single number.

**Report the probability, not a verdict.** The tool returns the share of 500 scenarios where
buying wins. "Buying wins in 62% of scenarios, with breakeven around year 7" is honest.
"You should buy" is not what was computed.

## What it will not do

Say so plainly rather than filling the gap with a guess:

- **No forecasting of individual neighborhoods.** Sub-ZIP appreciation edges mean-revert
  (past outperformance correlates about −0.3 with future). Sub-ZIP data describes where a
  home sits within its ZIP *today* — never a prediction about that pocket.
- **No valuation of a specific property.** Figures are ZIP-level: good for a market, poor
  for one unit. If they know their actual rent or price, use their number.
- **No listing lookup.** It will not fetch a Zillow URL. Ask for address and price.
- **US only.** No international coverage.
- **Not financial advice.** It's a simulator with stated, overridable assumptions.

When someone wants something outside this, call `request_capability` with a description of
what they asked for. That is how the roadmap gets set.

## Worked example

> "We pay $2,800 in Boulder and we're looking at a $750k place. Worth buying if we might
> move in six years?"

1. `analyze_address("80302")` — appreciation forecast, market rent, tax and insurance rates
2. `buy_vs_rent(home_price=750000, monthly_rent=2800, address="80302", analysis_years=6)`
3. Lead with the probability and the breakeven year. If breakeven lands *after* year six,
   that is the answer and it should be the first sentence.
4. Offer the conservative run (`investment_return_rate=4.5`) if they don't sound like
   someone with everything in index funds.

## More

- Web app: <https://www.terravue.app>
- Setup walkthrough: <https://www.terravue.app/connect>
- Privacy: <https://www.terravue.app/privacy> — tool calls only, typed addresses never stored
