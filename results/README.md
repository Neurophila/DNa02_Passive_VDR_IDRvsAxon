# Results

## Committed

| File | Size | Rows | Contents |
|------|------|------|----------|
| `idraxon_maxdf.csv` | ~1.0 MB | 10,494 | IDR + axon readout (`REGION = 'IDRandAxonVDR_region'`) |
| `axon_maxdf.csv` | ~1.0 MB | 10,494 | Axon-only readout (`REGION = 'AxonVDR_region'`) |

Both are **tab-separated** despite the `.csv` extension — read them with
`pd.read_csv(path, sep='\t')`.

### Columns

| Column | Meaning |
|--------|---------|
| `combo_id` | `"<in_swcid>_to_<out_swcid>"` |
| `in_swcid` | Skeleton node of the driven input synapse (1-based, matches SWC `rowId`) |
| `in_voltage` | Peak voltage at that input node, mV |
| `in_time` | Time of the input peak, ms |
| `out_swcid` | Monitored node that responded most strongly to this input |
| `out_voltage` | Peak voltage at that monitored node, mV |
| `out_time` | Time of the output peak, ms |

`in_time` and `out_time` are located independently, so they differ — input peaks land
near 1.2 ms, readout peaks near 3.8–5.4 ms.

**VDR** = `in_voltage / out_voltage`. Higher means more decay.

### Already reduced

These are **not** the raw sweep files. The raw sweep writes one row per
(input × every monitored node) — 10,494 × ~73 and 10,494 × ~50 rows, ~85–100 MB each.
These committed files are reduced to **one row per input synapse**, keeping the
monitored node with the highest peak voltage. So `in_swcid` is unique across all 10,494
rows, while `out_swcid` takes only the values that ever came out highest.

## Written by a re-run, not committed

With `RUN_SIMULATION = True`, each notebook writes its full sweep here as:

```
DNa02-<bodyId>_All_ins-_outs-<REGION>_<YYYYmmdd-HHMMSS>.tsv
```

The region tag keeps the two notebooks from colliding; the timestamp keeps a re-run from
overwriting an earlier sweep. These files are ~85–100 MB and are git-ignored. On the next
run with `RUN_SIMULATION = False`, the newest matching file for that region is picked up
automatically in preference to the committed reduced CSV.
