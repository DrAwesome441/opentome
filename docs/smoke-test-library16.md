# Smoke test — tier-0 parser against the existing Mangarr library (16 series)

Run 2026-08-26. **Not the validation spike** — this sample is Nick's own curated,
mainstream, English-print library, so the numbers are optimistic by construction and do
not count as validation. Its purpose was fast feedback on a real corpus.

**Clean room preserved:** only the *series names* were taken from `mangarr` (Nick's own
facts about his library). The GCD artifact was never read, never used as ground truth,
never used as a tiebreaker.

## Result

| Metric | Value |
|---|---|
| Volumes extracted | **536** across 16 series |
| **JP day-precision dates** | **99.4%** (533/536) |
| JP ISBN | 99.4% |
| **EN day-precision dates** | **84.3%** (452/536) |
| EN ISBN | 84.3% |
| Chapter composition | 69.2% (371/536) — see below |

Every date carries a source URL from the article's own `<ref>` (shueisha.co.jp for JP,
viz.com for EN), so records are attributable to the publisher rather than to Wikipedia.

Against the plan's bars: **JP clears ≥95% comfortably; EN at 84.3% does not.**

## The real finding: matching is the problem, exactly as predicted

The first run used a naive "take the top search hit" strategy. It silently returned:

| Series | Matched article | Result |
|---|---|---|
| `gyo` | *List of Kingdom chapters* | 80 confident, wrong volumes |
| `my dress up darling` | *List of Ranma ½ chapters* | 93 confident, wrong volumes |
| `solo leveling` | *List of Hunter × Hunter chapters* | 39 confident, wrong volumes |
| `sword art online` | *List of Sword Art Online **characters*** | 0 volumes |

**25% of the library silently bound to the wrong series**, each returning a full set of
dates and ISBNs that looked perfect and were entirely false. This is the
"confidently wrong is worse than absent" failure the plan named as decisive — reproduced
on the first run, on real data.

**Fix:** a token-containment guard in `find_article()` — every significant token of the
series name must appear in the candidate article title, with list-articles scored up and
`characters|episodes|anime|film` scored down. Mismatch rate after the guard: **0/16.**
Gyo correctly resolved to 2 volumes; Sword Art Online to its manga volumes list.

This is the single most transferable lesson so far: **the guard, not the parser, is what
makes the data trustworthy.**

## Two modelling issues surfaced

1. **Franchise articles conflate release lines.** `re zero` → 105 volumes and
   `sword art online` → 89 volumes, because those articles cover the entire franchise
   (light novels + manga + every spin-off) rather than one release line. The
   `work → release line → volume` model handles this, but the parser currently flattens
   it. Needs per-line separation.
2. **Chapter composition is correctly absent for webtoons.** Solo Leveling (manhwa) and
   Sword Art Online return 0 chapters because they have no chapters-in-volumes
   structure. Excluding those two (125 volumes), chapter composition is **90.3%**
   (371/411). The plan's "score webtoons on a different question" rule is right and
   should be applied to this metric.

## Engineering notes

- Wikipedia returned **HTTP 429** partway through the first full run. Added a 1.1s
  serial throttle, exponential backoff, and a **disk cache** (`.cache/`) keyed by request
  URL — so re-runs hit zero network, which the plan required for reproducibility.
- The User-Agent should carry a real contact before any larger run; Wikipedia's policy
  asks for one and throttles anonymous-looking clients.
- Chapter lists use two conventions: `{{Numbered list|start=N|…}}` (numbers implicit in
  item count) and `* Chapter N: Title` bullets. Both are handled. Only chapter *numbers*
  are extracted — never titles or summaries.

## Files

- `tier0/wikipedia_volumes.py` — the parser
- `tier0/run_library.py` — this 16-series runner
- `.cache/` — cached API responses (gitignore before committing)
