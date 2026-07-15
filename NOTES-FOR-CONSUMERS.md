# Notes for consumers of ftso-accuracy.json

This file sits next to `ftso-accuracy.json` in the public data repo. It explains
how to read the artifact — especially for automated consumers. The JSON itself
carries a machine-readable `advisory` block with the same facts; prefer parsing
that (`advisory.advisories[]`, each `{id, severity, message, since,
expected_resolution}`) and treat this document as the prose companion.

## What this artifact is

Per-feed accuracy of the FlareForward FTSO provider across all 63 Flare FTSOv2
feeds, regenerated every ~10 minutes. Accuracy means: how often our value lands
inside the field's reward band — **primary** = strictly inside the weighted IQR
`(q25, q75)` (boundary scores 0.5), **secondary** = within consensus median
±500ppm. Windows: 6h = 240 voting rounds, 24h = 960 rounds, 90s per round.

## Two measurement sources — do not mix them up

1. **On-chain** (`feeds`, `summary`, top-level windows; `source:
   "onchain_reveal"`): our actual on-chain reveal, decoded and graded. The
   strongest evidence, but it only exists while we are registered and
   submitting. During a registration gap these windows are **null** — null
   means "no on-chain submissions to grade", never "0% accurate".
2. **QA** (`qa` block; `source: "qa_local_provider_vs_field_band"`): our local
   provider's values graded every round against the same field reward band,
   which is decoded from the ~76 field voters' on-chain reveals and stays
   fresh regardless of whether we submit. As of schema v2 this path is
   independent of the Flare Data Availability API for grading and scale
   verification. Read `qa.qa_coverage` for how many of the 63 feeds have a
   verified integer scale (scales are snapped to exact powers of ten, never
   guessed) and `qa_lag_rounds` for freshness of the QA recorder itself.

## Current advisories (2026-07-21)

- **On-chain grading paused for reward epoch 417** (severity: warning). We are
  not in epoch 417's signing policy, so on-chain windows are null. Expected to
  resume with epoch 418 registration, ~2026-07-23T18:00Z. The `status` block
  (`submitting`, `gap_reason`, `expected_resume_utc`, lag counters) tracks
  this live; `submitting` flips back automatically when reveals resume.
- **QA source is DA-independent** (info). qa grading uses only decoded
  on-chain reveal data for its reference band. It kept producing numbers
  through the 2026-07-21 DA API outage.
- **qa understates ETH/USD and XRP/USD** (info). Validated against on-chain
  grading on a 59,220-row overlap window: 96.2% row-level agreement overall,
  but qa reads ETH secondary ~32pp and XRP ~28pp BELOW their on-chain truth
  (a known local sampling bias). No feed is overstated. Treat qa as a
  conservative floor; do not quote qa ETH/XRP as our protocol accuracy.
- **Epochs 412–416 are verified history** (info). The `epochs` table entries
  for reward epochs 412–416 are fully verified, decoded-on-chain results and
  remain trustworthy throughout the pause.

## Reading rules (for humans and agents)

1. Check `status.submitting` first. `true` → use on-chain numbers. `false` →
   explain the gap using `status.gap_reason` / `expected_resume_utc`, then
   fall back to `qa` (labeled as QA, with its caveat).
2. Any window with `n == 0` is "no data", never "0%".
3. `qa.qa_coverage.qa_lag_rounds` > ~240 → the QA recorder itself is stale;
   treat qa 6h as unavailable.
4. `advisory.human_summary` is a one-paragraph current-state summary safe to
   surface directly to users.
5. Additive schema policy: new keys may appear at any time; existing keys keep
   their meaning. Pin on `schema_version` if you need strictness.
