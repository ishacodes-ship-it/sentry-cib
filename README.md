# SENTRY-CIB console

A single-file browser console that shows how SENTRY-CIB (Sentinel for Coordinated Inauthentic Behavior in Reviews) detects a coordinated review campaign while it is still forming. It runs a synthetic review stream, or your own events, through a cascade of detectors and shows every step.

Keep this repository private until the invention is filed. This README describes the mechanism, so a public repo counts as a public disclosure.

## Run it

Open `index.html` in a browser, or serve the folder:

```bash
python3 -m http.server 5178
```

Then open `http://localhost:5178`. There is no build step and no backend. Everything runs in the page.

## What this project is and how it came about

1. **Starting point.** The manuscript "Early-Stage Detection of Coordinated Online Review Campaigns via Dynamic Bipartite Graph Modeling and Multimodal Confidence Gating" (Isha S U, Manya Dev, Hemanya Dargar). It describes a PyTorch research prototype and reports single-seed results on synthetic data. The authors flag those results as preliminary.
2. **Console.** This repo is the working front end built from that design. Tier 0, Tier 1 and the Hawkes fit follow the manuscript's equations. The temporal graph network, the sentence-embedding index and the learned gate are replaced by labelled stand-ins (see "What is real").
3. **Validation.** The console was run headless in Chrome 154 on 2 October 2026 across 12 random seeds. Results are below.
4. **Patent work.** The architecture was written up as an Invention Disclosure Format form with a prior-art search. Those documents are not in this repo.

## How the cascade works

Each review is an edge in a bipartite reviewer-to-product graph. A 72 hour window of edges is kept per product.

| Tier | What it does | Cost per review |
|---|---|---|
| Tier 0 | Welford running mean and variance of each product's ratings give a rating deviation and z-score. Account age is the reviewer's lifetime review count. | constant |
| Tier 1 | A count-min sketch keyed by product. Each cell holds a current-tick count and a decayed history. The score is the minimum over four rows of (count - expected)^2 / expected, multiplied by an amplifier for rating deviation and unverified purchase. A per-product CUSUM watches arrivals per hour. Either detector turns a product into a candidate for 6 hours and freezes its baseline and rating statistics. | constant |
| Tier 2 | Runs only on candidates. Structural score: decay-weighted share of reviewers with 2 or fewer lifetime reviews, unverified share, rating outliers, density, and a Hawkes branching ratio. The Hawkes fit counts only if it beats both a homogeneous Poisson model and a one-step change-point model on the same events, so a smooth viral spike does not read as self-excitation. Semantic score: size of the largest cluster of similar texts in the last 6 hours. A gate, g = sigmoid(4 x (structural - semantic)), blends the two. Alert at fused score 0.6 or more with 5 or more edges. | only on candidates |
| Tier 3 | Analyst queue. Each alert has an evidence bundle (subgraph, Hawkes trace, cluster rows, score trail, JSON export) and a single confirm or dismiss decision. | human |

Because Tier 1 is keyed by product, an account with one lifetime review is scored through the product it targets. A volume threshold that ignores accounts with fewer than 5 reviews never sees these accounts.

## Using the console

- **Scenario buttons** inject a ring with copy-paste text, a ring with fluent varied text, or a benign viral spike.
- **Tier 2 branch** switches between gated fusion, structural only and text only. Set it to text only and inject the fluent ring: the text branch misses it.
- **Ground truth** toggle shows which reviews and alerts belong to injected campaigns.
- **Scoreboard** compares SENTRY-CIB with two baselines that only fire at their own trigger points: a static snapshot at each 72 hour window close, and a volume threshold on accounts with 5 or more reviews.
- **Replay data** loads a JSONL, JSON or CSV file. Fields recognised: user (`user_id`, `user`, `reviewer`, `author`), product (`product_id`, `item`, `asin`, `business_id`), time (`timestamp`, `ts`, `date`), rating (`rating`, `stars`, `overall`), text (`text`, `review`), verified (`verified`). `sample_events.jsonl` holds 2,630 synthetic events with one planted ring on product `P07`.
- **Keys:** Space play or pause, 1 2 3 inject, J and K move through the queue, E evidence, C confirm, D dismiss, R reset, Esc close.

## Results from the console (synthetic data only)

Twelve seeds, six injected campaigns each, 32 products: 48 rings and 24 benign spikes.

| Branch | Rings alerted | False alerts | Low-history accounts caught |
|---|---|---|---|
| Gated fusion | 48 of 48 | 4 | 1,152 of 1,152 |
| Structural only | 48 of 48 | 7 | 1,152 of 1,152 |
| Text only | 27 of 48 (24 of 24 copy-paste, 3 of 24 fluent) | 1 | 648 of 1,152 |

- Median ring reviews already posted when the alert fired: 5 of 30.
- Static snapshot baseline alerted 21 of 48 rings. Volume threshold alerted 34 of 48, and by construction caught none of the low-history accounts.
- Tier 1 did not forward 91.3% of events and forwarded 93.5% of ring reviews, so Tier 2 can never see more than about 93.5% of a ring.
- Replay of `sample_events.jsonl`: one alert, on the planted product 46 minutes after the ring began, no false alert.
- Tier 0 plus Tier 1 throughput on a dense 300,000 event test: between 483,871 and 7.6 million events per second in two browser instances. Tier 2 and rendering are excluded.

## What is real and what is a stand-in

Real: Welford features, sketch chi-square, decay, CUSUM with baseline freeze, the Hawkes fit and its null-model test, the alert logic, the evidence bundle.

Stand-ins, labelled in the interface:

- Temporal graph neural network: decay-weighted aggregation with a logistic readout.
- Sentence-transformer plus FAISS: hashed bag-of-words vectors with cosine similarity.
- Learned gate: the fixed rule above.

## Limits

- One synthetic generator, written by the same authors as the detector.
- One machine, parameters tuned for the demo, no confidence intervals, no real review dataset.
- The 4 against 7 false-alert difference between gated and structural-only is small and was not tested for significance.
- Numbers here are not a measure of performance on a live platform.

## Files

- `index.html`: the whole console.
- `sample_events.jsonl`: synthetic replay file.
