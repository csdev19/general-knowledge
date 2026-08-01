# Sourcing real-world reference data (and seeding it)

_How to fill an app with real, structured, geo-tagged reference data — e.g. countries
→ attractions for a trip planner — legally, verifiably, and in a shape that loads
straight into the schema. Captured from the trip-planner "fill the app with places"
work so the approach is not lost._

## Who should do it: an agent with tools, not a plain chat

For **structured, geo-referenced** data the agent (web tools + structured output) beats
a plain LLM chat:

- **Plain chat** → nice prose from memory, fast, but **no verified coordinates, no
  guaranteed structure, no live web**, and you copy-paste + reshape by hand.
- **Agent** → emits JSON that matches the target table, **verifies coordinates**,
  dedupes, validates coherence, writes files, and feeds the seed pipeline. This is the
  missing link that turns "a list" into "loaded data".

Use the model's knowledge for **curation and reasoning** (which places matter, why go,
best season) — but **never trust an LLM's coordinates or facts**; verify every lat/lng
against a real source.

## Sources — legality first

| Source | License | Use for | Notes |
| --- | --- | --- | --- |
| **Wikidata** | **CC0** (no attribution) | **primary** — coords (`P625`), canonical name, category (`instance of`) | Query via SPARQL or `wbgetentities`. Structured + free. The mine of gold. |
| **Wikipedia** | CC BY-SA | descriptions / context | Attribution required — keep the source URL. |
| **OpenStreetMap / Nominatim** | ODbL | geocoding / addresses | Attribution + usage policy (≤1 req/s, cache results). |
| Official place APIs (Google Places, etc.) | proprietary | only if licensed | Pay + ToS; fine when you have a key and a budget. |

**Do NOT scrape** TripAdvisor / Google Maps / Yelp: their ToS prohibit it, they fight
scrapers, the HTML is fragile, and it is legal risk for zero upside when Wikidata already
has the structured facts. "Web scraping" for this problem = **Wikidata + web_search
enrichment**, not HTML scraping.

## The pipeline (repeatable per city/country)

1. **Curate** a candidate list (model knowledge): the places actually worth visiting.
2. **Resolve** each against **Wikidata** → verified coordinates, canonical name, category.
3. **Enrich** (description, best season, typical duration, cost) via Wikipedia / web_search.
4. **Normalize** to the target schema (for trip-planner `locations`: `name`,
   `defaultTags` ∈ the [[tagging-system]] slugs, `latitude`, `longitude`, `address`,
   `description`, `reason`, `expectedDurationHours`, `expectedCost`, `pricingType`).
5. **Validate** — coords in range, dedupe by name + proximity, category in the allowed
   set (same spirit as the CLI's `validateScenario`).
6. **Store** `data/attractions/<country>.json` in the repo → seed via the Ripuy CLI
   (`ripuy seed …` / `ripuy trip create -`).

## Provenance is a field, not an afterthought

Every record carries `source`, `sourceUrl`, and `license`. Learned the hard way: the gym
exercise dataset shipped without a link/licence and became unusable-until-verified. Bake
provenance in from row one so a CC BY-SA description can be attributed and a CC0 fact can
be used freely.

## Shape follows the target table

The output shape is **the app's own table**, not a generic "place" — so it loads with no
translation. For trip-planner that is `locations` (see the Convex `schema.ts`); mirror
whatever the destination app uses ([[schemas-first]]).

## Where to start

**Japan first** for trip-planner — dual purpose: it fills the app *and* feeds a real
upcoming trip. Then Perú (Lima/Cusco), then broaden. Generate a small sample (~12 places)
first to lock the shape before scaling a country.

## See also
- [[tagging-system]] — the category slugs attractions map onto.
- [[schemas-first]] — the shape is the schema; validate against it.
- [[ci-cd-pipeline-strategy]] — keep any scraper/enricher deps light ([deps weight](../conventions/ci-cd-pipeline-strategy.md)).
