# PoET²S simulator (v18)

Discrete-event simulator accompanying the revised manuscript *PoET²S: Folding a Digital Twin into Validator Selection
for Blockchain Consensus in Peer-to-Peer Energy Trading*. Every protocol is a message-passing state machine; latency is
measured from round start to finality with per-replica quorums, Pareto link delays, 50 Mbit/s FIFO uplinks and
signing/verification costs.

In the code, `PoMonitor`, `EACM` and `RepStake` are the **veto**, **additive** and **linear** stylised design points of
the paper (Table 1). They borrow one selection mechanism each from the cited systems and are not re-implementations.

## Changes in v18 (relative to v17, used for the first submission)

- Quorums are counted per receiving replica (v17 used one global vote set per block, which understated latency).
- The twin-verified energy term e_i is updated every slot (it was inert in v17); reported generation is capped at
  1.2 x the physical envelope.
- One slot harness for all schemes; identical attack realisations per seed across schemes (tested).
- Throughput removed as a metric (it was block size / latency). Latency accounting, Sybil, weight-trajectory and
  composite (leave-one-out, simplex) scripts added. Main experiments use 30 seeds (42–71).

## Changes in v21 (code audit)

- The twin forecast now uses the same hour as the physics (`(t - 1) % 24` after the slot tick); earlier
  versions forecast one hour ahead. All results were recomputed.
- DPoS commits on 4 delegate acknowledgements, as documented (earlier versions required 5).
- Optional `byz_withhold` for the three-phase commit; every slot is checked to have committed.

## Changes in v20

- Optional two-step fast path (`commit_mode="fast"`, FaB-style, f = 4 for K = 21, fast quorum 17, fallback to the
  three-phase commit after 50 ms). Byzantine committee members withhold accepts. Selection and detection are unchanged.
- First-commit latency (`first_commit_ms`) recorded for every scheme; scripts run_fastpath.py and run_latency_fast.py.

## Changes in v19

- `energy_offset` (default 1 = Eq. 4 as evaluated; 0 removes the per-identity constant) and `committee_mode`
  (`topk` or weighted `sortition`) options on PoET2S and the veto design point.
- Safety-violation metric (share of blocks whose validating set is at least one-third Byzantine).
- New scripts: run_hardening.py, run_sybil_hardening.py, run_sybil_sortition.py, run_matched_fp.py,
  run_seeds100.py, run_stats_v19.py; Byzantine and sensitivity sweeps now use 30 seeds.

## Reproduce

```
python3.11 -m venv .venv && . .venv/bin/activate
pip install -e .
pytest -q
python scripts/run_main.py              # Tables 6, 8; Figures 2-6   (~11 min)
python scripts/run_adaptive.py          # Figure 9
python scripts/run_hybrid_ablation.py   # Table 10
python scripts/run_byzantine.py         # Figure 8
python scripts/run_param_sensitivity.py # Figure 10
python scripts/run_scaling.py           # Figure 11
python scripts/run_latency_accounting.py# Table 7
python scripts/run_sybil.py             # Table 11
python scripts/run_weight_trajectory.py # Figure 7
python scripts/run_stats.py             # Tables 9, 13, A1
python scripts/run_hardening.py; python scripts/run_sybil_hardening.py; python scripts/run_sybil_sortition.py  # Table 12
python scripts/run_matched_fp.py        # Figure 12
python scripts/run_seeds100.py; python scripts/run_stats_v19.py   # 100-seed and safety tests
python scripts/make_figures_v18.py      # results/figures_v18/*.png
```

## Main results (N = 500, 15% Byzantine, 30 seeds, mean ± SD)

| Scheme | Latency (ms) | Caught (%) | FP / caught | Infiltration (%) |
|---|---:|---:|---:|---:|
| PoW | 8032.5 ± 988.0 | 0.0 ± 0.0 | 0.0 ± 0.0 | 13.9 ± 4.2 |
| PoS | 20.8 ± 0.4 | 100.0 ± 0.0 | 51.9 ± 1.6 | 15.2 ± 1.4 |
| PoS+Twin | 20.8 ± 0.4 | 100.0 ± 0.0 | 54.8 ± 1.4 | 15.2 ± 1.4 |
| DPoS | 19.0 ± 0.5 | 100.0 ± 0.0 | 58.6 ± 1.3 | 13.3 ± 11.8 |
| DPoS+Twin | 19.0 ± 0.5 | 100.0 ± 0.0 | 60.8 ± 1.2 | 13.3 ± 11.8 |
| PBFT | 38.9 ± 1.3 | 100.0 ± 0.0 | 66.4 ± 1.2 | 15.1 ± 4.4 |
| PBFT+Twin | 38.9 ± 1.3 | 100.0 ± 0.0 | 67.9 ± 1.2 | 15.1 ± 4.4 |
| PoMonitor | 30.2 ± 0.2 | 91.7 ± 3.1 | 9.7 ± 0.6 | 10.8 ± 3.6 |
| EACM | 30.2 ± 0.2 | 75.0 ± 4.8 | 6.0 ± 0.6 | 9.4 ± 3.5 |
| RepStake | 30.2 ± 0.2 | 91.0 ± 2.7 | 13.9 ± 0.9 | 3.8 ± 2.9 |
| PoET2S | 30.2 ± 0.2 | 91.0 ± 2.7 | 13.9 ± 0.9 | 2.6 ± 1.9 |

Latencies are relative values inside this simulator, not deployment predictions.

### Review-pass notes (v21.1)
- `scripts/run_stats.py` and `scripts/run_stats_v19.py` round paired differences to 1e-9 before the Wilcoxon test, so differences equal up to floating-point noise form proper ties. Affected Holm p-values: linear-rule infiltration 0.031 -> 0.032, proposer capture 0.43 -> 0.50; 100-seed infiltration 6.6e-9 -> 7.2e-9, proposer capture 2.7e-4 -> 3.3e-4. No conclusion changes.
- `scripts/run_tail.py` writes `results/latency_tail.csv` (per-block p95/p99 latency, PoET2S three-phase vs fast path).
- Removed stale `results/adaptive_30seed.csv` (identical copy remains in `results/v20/`).
