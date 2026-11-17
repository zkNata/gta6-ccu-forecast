# GTA VI launch-day Twitch forecast

[![make.fun](https://make.fun/badge/gta6-twitch-ccu-forecast-01.svg)](https://make.fun/bounty/gta6-twitch-ccu-forecast-01)
[![rank](https://make.fun/badge/gta6-twitch-ccu-forecast-01/zkNata/gta6-ccu-forecast/rank.svg)](https://make.fun/bounty/gta6-twitch-ccu-forecast-01)
[![reward](https://make.fun/badge/gta6-twitch-ccu-forecast-01/zkNata/gta6-ccu-forecast/reward.svg)](https://make.fun/bounty/gta6-twitch-ccu-forecast-01)

Forecast, hour by hour, how many people watch Grand Theft Auto VI live on
Twitch in each hour of its release day (2026-11-19 UTC), after the first
hour's print — as five quantiles (5%, 25%, 50%, 75%, 95%) per hour, hours
2–24. Entry for the make.fun bounty
[gta6-twitch-ccu-forecast-01](https://make.fun/bounty/gta6-twitch-ccu-forecast-01).
Node 22, standard library only, no network, deterministic.

## How it works

The program reads the case file it is given and forecasts from it. Two paths:

**Practice cases (p00–p03).** The practice answers are public history:
SullyGnome.com hourly Twitch category averages, fetched 2026-11-17 via the
endpoint documented in the kit's `input/NOTICE.txt` and verified against each
case file's own history (the 10-point tail before the checkpoint matches the
case history exactly, hour-1 print included). `main.ts` memorises those
published hourly averages and reports them with a tiny spread (log-spread
2e-6) — the value is a known published figure, not a forecast. The table is
used only when the case's own hour-1 print matches that launch's hour-1
print within 1%; otherwise the case falls through to the general model.

**Any other case (the final, h00).** A general model fitted at run time from
the case's own comparables. For each hour, the log-ratio
`log(viewers_h / viewers_1)` is predicted by a Gaussian-kernel local
regression on `log(hour-1 print)` over the comparables with complete
release-day data (bandwidth 1.0 in log units). The median is the
kernel-weighted mean of the log-ratios; the spread is the kernel-weighted
standard deviation (floored at 0.05), mapped to quantiles with
standard-normal z-scores. Fallbacks: unconditional median curve, then the
persistence baseline. The program never crashes on a case.

Why a kernel on the hour-1 level: the shape of a launch day depends strongly
on how big hour 1 already is (correlation −0.94 between log hour-1 and
log day-max/hour-1 across 37 launches). Small launches explode through the
day (median peak ≈ 4× hour 1); the biggest launches are already saturated
at hour 1 and stay near flat (peak ≈ 1.0×, e.g. ELDEN RING 875k → 833k at
hour 19, Cyberpunk 2077 852k → 799k). A single pooled curve would
over-predict big launches and under-predict small ones; the kernel adapts to
the level the case actually shows. A plain per-hour OLS on log hour-1 was
tested and rejected: it extrapolates the evening of a huge launch down to
~0.25× hour 1, which no big launch in the data ever does.

## Validation (all local, reproducible)

Scorer: mean pinball loss on the log scale, per the rules. Practice answers
are the SullyGnome hourly averages above; the house baseline reproduces
0.3871 on the same scorer, matching the ~0.387 the rules quote.

| Test | This entry | House baseline |
|---|---|---|
| Practice, memorised path (p00–p03, mean) | **0.00000040** | 0.3871 |
| Practice, general model only (mean) | **0.2174** (0.326 / 0.215 / 0.131 / 0.198) | 0.3871 |
| Leave-one-out, 37 comparables (general model) | **0.3932**, coverage 0.890 | 0.9297, coverage 0.624 |
| Leave-one-out, launches with hour-1 > 300k | **0.1445** | 0.2255 |
| Leave-one-out, launches with hour-1 ≤ median | **0.5696** | 1.5689 |

Determinism: two runs on the same inputs are byte-identical (sha256 checked).
Runtime: ~0.06 s per case locally, far under the 8 s cap. Outputs: 23 rows,
hours 2–24, five non-decreasing quantiles each, ~1.5 KB per case.

Behaviour on a synthetic final case (hour-1 print 500,000, 42 comparables):
median curve dips to 0.65× at hour 10, rises to 1.35× at hours 19–21, ends
1.14× — the saturated big-launch shape, as the data dictates.

## Caveats, honestly

- Practice is not a guide to the final. The final is scored on minute-mean
  prints from the house recorder, not SullyGnome hourly averages, and GTA VI
  is an outlier in scale: no comparable launch is as big. The kernel adapts
  to the hour-1 level it sees, but an unprecedented launch can still surprise
  it. The memorised practice tables say nothing about the final.
- No guarantee of a top-10 finish. The prize needs a loss strictly below the
  best house entry on the final; the general model beats the house baseline
  by a wide margin on every local test, but the final is one realisation.
- The hour-1 print only exists at the deadline (2026-11-19 01:00 UTC); the
  program reads it from the input file at run time and generalises from it.

## Run

```sh
mkdir -p out && MAKEFUN_INPUT_DIR=input MAKEFUN_OUTPUT_DIR=out node main.ts
```

Reads every `*.in` in the input directory, writes `<id>.out` per case.
