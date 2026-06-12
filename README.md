# 🛰 OrbitGuard

**Open-source satellite conjunction screening with NASA-grade collision math — validated at
99.73% recall against the official U.S. TraCSS federal answer key.**

Every satellite operator receives collision warnings: cryptic Conjunction Data Messages with a
~99% false-alarm base rate. The commercial tools that turn those warnings into decisions cost
more per year than a university cubesat costs to build. OrbitGuard is the alternative: a
browser-based conjunction-assessment system built entirely on free public data, with the actual
operational mathematics — and an accuracy story it can *prove* rather than assert.

> **99.73% recall · 99.60% precision** against the 913,330-conjunction answer key of the U.S.
> Office of Space Commerce *Dataset for Conjunction Assessment Verification* (TraCSS) —
> TCA agreement **61 ms RMS**, miss-distance agreement **0.14 m RMS** with The Aerospace
> Corporation's validated CSieve tool, over a 7-day, 26,793-object, all-vs-all run executed on
> an 8 GB laptop. [Methodology & residual autopsies](docs/VALIDATION.md) · scorecard rendered
> live in-app · technical deep-dive in [`OrbitGuard_Technical_Report.pdf`](OrbitGuard_Technical_Report.pdf).

![Mission Control](docs/screenshots/mission-control.png)

## What it does

- **Screens your satellites against the full public catalog** — 15,000+ active objects from
  CelesTrak GP (OMM JSON) — for close approaches over a 7-day window, in seconds on a laptop.
- **Computes collision probability with the operational method**: Foster (1992) 2D
  encounter-plane quadrature (the approach NASA CARA runs), cross-checked in CI by the
  Chan (1997) analytic series and three independent mathematical referees.
- **Refuses to lie when the data can't support a number.** Public GP/TLE data carries no
  covariance, so OrbitGuard never invents one: it computes the *worst-case* probability bound
  over every error model consistent with the geometry (PcMax = R²/(e·d²)) — which can **prove an
  event safe but never order a maneuver**. When the bound can't clear an event, the verdict
  escalates to requesting CDM-grade data. *TLE data can prove safety, never danger.*
- **Issues an explained DODGE / WAIT / WATCH verdict** per event, anchored to NASA Conjunction
  Assessment Handbook thresholds, with per-object hard-body radii derived from SATCAT
  radar-cross-section data.
- **Narrates every verdict in plain language** through a strictly grounded LLM layer: the model
  receives a finished JSON evidence record and may not introduce a single number of its own — a
  digit-level validator rejects violations and a deterministic template renderer is the
  always-on fallback.
- **Renders it all in a live 3D mission control** (CesiumJS): animated assets among the real
  tracked catalog, click-to-follow any object, cinematic encounter replays with a live
  separation readout, covariance-ellipse encounter plots, and countdowns to closest approach.

## The screening funnel (a real run)

```
15,697 objects ──gate──▶ 834 ──sieve──▶ 1,967 windows ──refine──▶ 7 conjunctions    11.6 s
                (ISS vs full catalog, 7-day window, M-series laptop)
```

1. **Stage A — apogee/perigee gate** (Hoots Filter I): altitude bands that cannot intersect
   within the padded screening distance are discarded by pure geometry, O(N).
2. **Stage B — padded coarse-grid sieve**: vectorized SGP4 (`SatrecArray`) on a coarse grid; a
   pair is kept if it comes within `D + v_rel,max·Δt/2`. **The pad is a guarantee, not a
   heuristic** — no true sub-D conjunction can evade the coarse net (Alarcón-Rodríguez et al.
   2002, the "smart sieve" ESA built CRASS on).
3. **Stage C — TCA refinement**: Brent root-finding on g(t) = Δr·Δv to millisecond-level time of
   closest approach, then miss distance, relative velocity, and radial/in-track/cross-track
   decomposition.

For the federal validation run the same engine scales to **all-vs-all over 26,793 ephemerides**:
fully vectorized batched refinement (one interpolation call per *object*, not per event),
numpy run-length window bookkeeping, and memory-bounded time slices that keep a 7-day,
~10⁸-pair-window problem inside 8 GB of RAM.

## Why you can trust the numbers

The project's thesis is that **accuracy should be measured, not asserted** — in two rings.

**Ring 1 — continuous (53 automated tests, every commit):**

| Claim | Independent referee |
|---|---|
| The sieve misses nothing | Brute-force 1-second search over every pair — recall *and* precision asserted on scenes with planted conjunctions, near-misses, GEO/Molniya decoys |
| Foster Pc is exact | Closed-form noncentral-χ² result (isotropic case) to 1e-6 relative; independent adaptive 2D quadrature to machine precision; 2M-sample Monte Carlo within sampling error |
| PcMax is a theorem | σ-sweep across four decades: exact Pc never exceeds R²/(e·d²), and the bound is attained at σ = d/√2 |
| Propagation & frames are right | Skyfield's independent TEME→GCRS pipeline agrees to < 5 m; all frame math routed through one audited module |
| The harness scores honestly | Synthetic ephemerides with closed-form TCA/miss answers; duplicate events penalize precision |

**Ring 2 — the federal exam:** the U.S. Office of Space Commerce publishes a verification
dataset for conjunction-assessment systems — 20.7 GB of CCSDS-OCM ephemerides (the real
TLE-derived catalog plus synthetic maneuvering objects, historical-CDM reconstructions, and the
OSIRIS-REx reentry) with a **913,330-conjunction answer key** generated by The Aerospace
Corporation's validated CSieve tool. OrbitGuard's engine runs directly on those ephemerides
(no SGP4 in the loop — the key tests *screening*, not propagation):

| Metric | Result |
|---|---|
| Recall | **99.73%** (910,837 / 913,330 key events found) |
| Precision | **99.60%** |
| TCA error | **61 ms RMS** (matching tolerance ±5 s) |
| Miss-distance error | **0.14 m RMS** |

Residuals are published, not hidden (`data/scorecard_autopsy.json`): misses concentrate at the
10 km screening boundary where metre-level interpolation differences flip events in or out;
extra finds cluster in the CDM-reconstructed population, consistent with the key's own
documented non-exhaustiveness.

## The product

![Encounter Inspector](docs/screenshots/encounter-inspector.png)

- **Mission Control** — live globe with your assets among the real catalog, ranked verdict cards
  with countdowns, the screening funnel with timings, hover-to-identify and click-to-follow on
  every object, and an interactive first-visit guide.
- **Encounter Inspector** — a cinematic replay of each close approach: the camera frames both
  objects and zooms as they converge, a tether line recolors with live separation, comet trails
  show direction of motion, and playback controls scrub around the moment of closest approach.
  Below: the encounter-plane figure (covariance ellipse or worst-case circles), the full numeric
  evidence record with a raw-JSON toggle, and the grounded narration.
- **Validation Scorecard** — the federal-answer-key metrics rendered live, with methodology,
  run provenance (git SHA, hardware), and honest-limits statements.
- **A typed REST API** (FastAPI/OpenAPI) serving the same evidence records the UI renders —
  integration-ready for ground-segment tooling.

## Architecture

```
CelesTrak GP (OMM JSON) ─┐
SATCAT (RCS sizes) ──────┤─▶ ingest ─▶ astro (vectorized SGP4 · TEME frames) ─▶ screening (A/B/C)
TraCSS OCM ephemerides ──┘                                                          │
                                  validation harness ◀──────────────────────────────┤
                                  (answer-key scoring)                              ▼
                                                       risk: Foster · Chan · PcMax · policy
                                                                    │
                                                                    ▼  evidence records (JSON)
                                          FastAPI ──▶ Next.js + CesiumJS UI
                                             │              ▲
                                             ▼              │ static bake (serverless deploys)
                                  explain (LLM, narrate-only + digit validator + template fallback)
```

**The rule that keeps the system honest:** every number shown anywhere is computed in the Python
data plane. The LLM converts one finished JSON record into sentences; it cannot change a verdict,
a probability, or a distance.

OMM-first throughout: catalog identifiers are integers end-to-end, so the system survives the
2026 5-digit NORAD catalog-number rollover that breaks legacy TLE-only pipelines.

## Quickstart

```bash
# engine (Python 3.11+)
python3 -m venv .venv && .venv/bin/pip install -e ".[dev]"
.venv/bin/pytest                          # 53 tests
.venv/bin/python -m engine.cli fetch      # one disciplined CelesTrak download (2 h cache)
.venv/bin/python -m engine.cli screen --assets 25544 --days 7 --explain

# API + web
.venv/bin/uvicorn engine.api.app:app --port 8000
cd web && npm install && npm run dev      # http://localhost:3000
```

Works fully offline after the first fetch (committed fixture + bundled imagery — no API keys
required). Optional: `GROQ_API_KEY` in `.env` upgrades explanations from template to LLM prose.

## Deploy (shareable link, zero backend)

```bash
.venv/bin/python -m engine.cli bake      # physics -> web/public/data/*.json
```

Import the repo on [Vercel](https://vercel.com), set **Root Directory = `web`**, deploy.
Static mode enables automatically; the site is pure Next.js + client-side Cesium.

## Repository map

```
engine/ingest      CelesTrak GP + SATCAT fetchers — cache discipline enforced in code
engine/astro       OMM→SGP4, vectorized propagation, the single audited frames module
engine/screening   Stage A gate · Stage B padded sieve · Stage C Brent TCA · funnel telemetry
engine/risk        Foster · Chan · PcMax · encounter plane · DODGE/WAIT/WATCH policy · evidence
engine/explain     narrate-only LLM client · digit validator · deterministic templates
engine/validation  OCM parser · all-vs-all harness · answer-key matcher · scorecard
engine/api         FastAPI (typed OpenAPI contract)
web/               Next.js 14 + CesiumJS: mission control, encounter inspector, scorecard
tests/             the verification core (see "Why you can trust the numbers")
docs/              ALGORITHMS.md (the math) · VALIDATION.md (methodology) · screenshots
```

## Honest limits

- GP/TLE accuracy is ~1 km at epoch, degrading 1–3 km/day (in-track dominant) — which is exactly
  why TLE-grade verdicts are worst-case bounds with an escalation path, never maneuver orders.
- Low relative-velocity encounters (< 100 m/s) violate the short-encounter 2D-Pc assumptions;
  OrbitGuard flags them rather than printing a wrong number.
- The marked SIMULATED event is a training scenario (simulated geometry + CDM-grade covariance)
  that exercises the maneuver-alert path through the real probability pipeline.
- OrbitGuard is a research-grade system, not an operational conjunction-assessment service; the
  TraCSS dataset is a diagnostic, not an operational certification.

## References (abridged)

Foster & Estes 1992 (NASA JSC-25898) · Chan 1997/2008 · NASA CARA 2D-Pc implementation
recommendations (NTRS 20190028900) · NASA CA Handbook v2 2023 (NTRS 20230002470) ·
Alarcón-Rodríguez, Martínez-Fadrique & Klinkrad, ESA SP-486 (2002) · Hoots, Crawford & Roehrich
1984 · Vallado et al. 2006 (*Revisiting Spacetrack Report #3*) · Auman et al., AMOS 2025 (TraCSS
validation methodology) · Office of Space Commerce, *Dataset for Conjunction Assessment
Verification* (CC0) · CelesTrak GP/SATCAT documentation. Full list: [`docs/ALGORITHMS.md`](docs/ALGORITHMS.md).

## License

MIT — see [LICENSE](LICENSE). Built on free, public data, for the operators commercial
space-traffic services price out.
