# Ranking And Citation Contract

## 1. Scope

This document is the normative launch contract for SV-US-009. It defines the internal
`rank_and_cite` application boundary consumed later by the chat workflow. It does not add an
installation-facing HTTP route and it does not let a caller or model widen the persisted retrieval
allowlist.

The launch policy version is `ask-sunny-ranking-v1`. Any change to signal meaning, weights, sort
order, URL canonicalization, review aggregation, or uncertainty rules requires a new version and a
new committed evaluation baseline.

## 2. Inputs And Trust Boundary

The boundary accepts only candidates returned by the allowlist-scoped SV-US-008 retrieval service,
the normalized query and filters used for that retrieval, optional factual claim/source references,
and optional volatile-detail requirements. It must not accept raw database rows, SQL, arbitrary
URLs, or caller assertions that a source is allowed.

Retrieval candidates are enriched with these safe ranking fields:

```json
{
  "source_kind": "directorist_listing",
  "data_source_key": "directorist:events",
  "data_source_label": "Event Directory",
  "source_id": "2001",
  "title": "Community Workshop",
  "url": "https://example.com/events/community-workshop",
  "result_role": "listing",
  "matched_metadata": {"categories": ["workshops"], "event_start": "2026-07-18T10:00:00Z"},
  "matched_constraints": ["categories", "date", "locations"],
  "scores": {"vector": 0.84, "bm25": 7.2, "fused": 0.91},
  "ranking_features": {
    "is_featured": false,
    "source_updated_at": "2026-07-15T08:00:00Z",
    "review_rating": null,
    "promotion_values": {},
    "volatile": {
      "date": {"value": "2026-07-18T10:00:00Z", "observed_at": "2026-07-15T08:00:00Z"}
    }
  }
}
```

The repository emits only configured promotion keys into `promotion_values`; it never exposes the
whole metadata object through this boundary. A review candidate retains its direct review identity,
title, URL, rating, and an active allowed `parent` listing with the same safe ranking fields. Review
and parent URLs must both pass the same URL rule.

`matched_constraints` contains only server-owned filter dimension names that were actually applied
and satisfied: `categories`, `locations`, `amenities`, `price`, `rating`, `taxonomies`, `metadata`,
`date`, and `distance`. It does not contain raw values. Repository filtering remains authoritative;
the field exists to explain and evaluate ranking, not to weaken filtering.

## 3. Launch Ranking Policy

All input numbers are finite and clamped to `0..1`. Intermediate and output scores are rounded to
six decimal places. Raw BM25 values are diagnostic only and never enter final application ranking.

### 3.1 Relevance signals

`retrieval` is the normalized SV-US-008 fused score. `exact` is:

- `1.0` when the normalized query equals the normalized title;
- `0.75` when the whole normalized query is a title phrase;
- `0.50` when every query token occurs across the title and compact matched metadata;
- `0` otherwise.

Normalization lowercases, Unicode-normalizes with NFKC, collapses whitespace, and removes no words
or numbers.

`structured` is `1` when no structured filters were requested. Otherwise it is the number of unique
requested filter dimensions present in `matched_constraints`, divided by the number requested.
Date, location, category, amenity, price, rating, taxonomy, metadata, and distance therefore remain
independently visible signals even though eligible retrieval rows normally satisfy every mandatory
filter.

```text
relevance_score = 0.70 * retrieval
                + 0.20 * exact
                + 0.10 * structured
```

### 3.2 Quality signals

`freshness` is based on the server clock and `source_updated_at`: `1` at age `0..7` days, `0.5` at
`>7..30` days, `0.25` at `>30..365` days, and `0` when older, missing, invalid, or in the future.

Eligible retrieved reviews are grouped by their active allowed parent listing. `review` is the
average finite rating divided by `5`, multiplied by `min(review_count, 5) / 5`. Missing ratings do
not enter the average. It is `0` when there is no rated evidence.

```text
quality_score = 0.60 * review + 0.40 * freshness
```

### 3.3 Promotion signals

`is_featured` contributes `0.5`. Configured promotion matches contribute the remaining `0.5` as
`sum(matched weights) / sum(configured weights)`, or `0` when no promotion fields are configured.
Promotion fields apply only to Directorist listing metadata and never to reviews or WordPress
content.

The server environment variable `RANKING_PROMOTION_CONFIG_JSON` defaults to `[]` and contains at
most 20 objects with exactly:

```json
{
  "key": "sponsorship_tier",
  "values": ["gold", "silver"],
  "weight": 1,
  "disclosure": "Sponsored placement"
}
```

Keys are unique stable lowercase metadata keys, `values` is a non-empty array of at most 20 unique
JSON primitive values, `weight` is finite in `(0,1]`, and `disclosure` is non-empty public text of at
most 120 characters. Unknown fields, invalid JSON, duplicate keys/values, objects/arrays as values,
and private metadata fail startup configuration validation. Primitive comparison is type-sensitive;
strings are compared after trimming and Unicode NFKC normalization. A matching configuration always
emits its disclosure even though its score may not affect order.

`RANKING_FEATURED_DISCLOSURE` defaults to `Featured`. It is emitted when `is_featured=true`.

### 3.4 Sort and deduplication

Results sort lexicographically by:

1. `relevance_score` descending;
2. `quality_score` descending;
3. `promotion_score` descending;
4. stable tuple `(source_kind, data_source_key, source_id)` ascending.

This ordering is a product invariant: no featured or configured promotion signal can compensate for
lower semantic, exact, or structured relevance. Promotion only breaks an exact relevance-and-quality
tie.

Before output, candidates deduplicate first by stable source tuple and then by canonical public URL,
keeping the earlier candidate in the policy order. A valid target uses `http` or `https`, has a
non-empty hostname, contains no username or password, drops its fragment, removes a default port,
lowercases scheme and hostname, and serializes through the standard URL parser. An invalid target is
removed from recommendations and citations. Query strings are retained because a source may require
them for its direct public page.

## 4. Review Evidence Aggregation

A review is never a recommendation card. Review evidence groups by the parent listing stable tuple.
If the parent listing is already a candidate, its retrieved reviews contribute the review quality
signal and remain separately citable. If it is absent, the ranker may create one parent listing
candidate from the review's already validated `parent` identity/title/URL; its retrieval, exact, and
structured signals come from the best-ranked review in that group. Duplicate reviews count once by
review stable tuple, and no more than the bounded retrieved review set contributes.

Conflicting parent identity or URL data fails closed for that review group. A synthesized parent is
subject to the same URL validation, ranking, and deduplication rules as a directly retrieved listing.

## 5. Recommendation Contract

Each recommendation card contains:

```json
{
  "source_kind": "directorist_listing",
  "data_source_key": "directorist:events",
  "data_source_label": "Event Directory",
  "source_id": "2001",
  "title": "Community Workshop",
  "url": "https://example.com/events/community-workshop",
  "reason": "Matches the requested date, location, and category.",
  "is_featured": false,
  "disclosures": [],
  "volatile_details": [],
  "uncertainties": [],
  "ranking": {
    "policy_version": "ask-sunny-ranking-v1",
    "relevance_score": 0.91,
    "quality_score": 0.4,
    "promotion_score": 0
  }
}
```

The reason is deterministic and evidence-based. It names satisfied structured dimensions in this
order: date, location, distance, category, amenity, price, rating, taxonomy, metadata. Otherwise it
uses `Exact title match.`, `Strong semantic match.`, or `Relevant retrieved source.` in that order.
It never cites featured or promotion state as the reason for relevance.

`disclosures` is a stable, deduplicated array of `{type, label}` objects. `type=featured` uses the
configured featured label; `type=promotion:<key>` uses the matching promotion disclosure. Internal
scores are safe product diagnostics but contain no query, content, raw metadata, or provider data.

## 6. Citations And Unsupported Claims

A factual claim reference has a caller-owned `claim_id` and one or more exact stable source tuples.
The citation assembler resolves those tuples only against the valid retrieval/review evidence set
received by this boundary. It cannot construct a citation from caller-supplied title or URL data.

Each deduplicated citation contains `citation_id` (`citation-1`, `citation-2`, ... in stable source
tuple order), `claim_ids`, `source_kind`, `data_source_key`, `data_source_label`, `source_id`, `title`,
`url`, and `evidence_role=primary|review`. A review may be cited directly while its parent remains the
recommendation card. Claim IDs with no valid resolved source appear in `unsupported_claim_ids`; the
later answer workflow must remove, qualify, or regenerate those claims rather than present them as
facts.

## 7. Uncertainty

The caller may request any of `date`, `availability`, and `operating_hours` for each candidate. The
ranker emits a `volatile_details` member with exactly `field`, `status="confirmed"`, `value`, and
`observed_at` only when the candidate contains a non-empty public value and a valid observation
timestamp. Date values must be valid timestamps or bounded date ranges.
Availability evidence older than 24 hours and operating-hours evidence older than 30 days is stale;
future timestamps are invalid.

Missing, invalid, stale, or conflicting evidence produces an uncertainty object with exactly
`field`, `status="uncertain"`, and `reason=missing_evidence|invalid_evidence|stale_evidence|conflicting_evidence`.
An uncertain field has no invented `value`. Recommendation reasons never assert an uncertain
volatile detail. The later answer workflow must preserve these labels and direct source links.

## 8. Internal Response

The application boundary returns:

```json
{
  "policy_version": "ask-sunny-ranking-v1",
  "recommendations": [],
  "citations": [],
  "unsupported_claim_ids": [],
  "diagnostics": {
    "input_candidates": 0,
    "valid_evidence": 0,
    "review_groups": 0,
    "duplicate_identities_removed": 0,
    "duplicate_urls_removed": 0,
    "invalid_urls_removed": 0
  }
}
```

The boundary is pure and deterministic when supplied a clock. No public route is added until the
SV-US-011 chat contract uses it.

## 9. Evaluation Set And Release Gate

The repository stores a reviewed `evaluation/ranking-launch-v1.json` fixture set. Every case includes
a stable ID, query, filters, candidates, expected ordered recommendation tuples, optional claim
references, and expected uncertainties/disclosures. It contains exact-title, semantic, date,
location, category, amenity, metadata, freshness, review-parent, featured/promotion, duplicate,
invalid-URL, citation, zero-result, and insufficient-evidence cases.

The deterministic evaluator commits `evaluation/ranking-launch-v1-baseline.json` with fixture count,
top-1 accuracy, top-3 recall, citation correctness, promotion inversions, structured-filter
violations, invalid recommendation targets, and uncertainty correctness. SV-US-009 cannot complete
unless top-1 accuracy, top-3 recall, citation correctness, and uncertainty correctness are `1`, and
promotion inversions, filter violations, and invalid targets are `0`. Any future policy change must
update the policy version, reviewed fixtures, and baseline together.
