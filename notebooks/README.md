# Notebooks

Two notebooks that differ **only** in which region the voltage is read out from. Run
either one top to bottom; they do not depend on each other.

| Notebook | `REGION` | Monitored nodes |
|----------|----------|-----------------|
| `SingleNeuronSimulationDNa02_all_inputs_IDRandAxonVDR.ipynb` | `IDRandAxonVDR_region` | ~73 (IDR + axon) |
| `SingleNeuronSimulationDNa02_all_inputs_AxonVDR_region.ipynb` | `AxonVDR_region` | ~50 (axon only) |

`RandCValues.png` is the R/C parameter table from Scheffer et al., displayed by a
markdown cell — keep it next to the notebooks.

## Before you run

Set these in your environment (see the [Credentials](../README.md#credentials) section):

| Variable | Needed when | Purpose |
|----------|-------------|---------|
| `NEUPRINT_TOKEN` | always | Fetching the skeleton, synapses, and SPICE netlist from NeuPrint |
| `NGSPICE_BIN` | only if `RUN_SIMULATION = True` | Directory holding `ngspice_con` / `ngspice` |

`NEUPRINT_APPLICATION_CREDENTIALS` is accepted as a fallback for the token. Neither
variable has a default, and the notebook stops with an explanatory error rather than
falling back to anything machine-specific.

## Cell order

1. **Cells 1–6** — imports, NeuPrint client, `NGSPICE_BIN`, then the config cell that
   sets `REPO_ROOT`, `RESULTS_DIR`, `REGION`, and `RUN_SIMULATION`.
2. **Fetch and map** — skeleton, input/output synapses, nearest-node assignment
   (`swcid`), overview plots.
3. **Region carve-out** — `navis.longest_neurite` plus coordinate windows produce `bb`
   (IDR) and `bbfront` (axon). The IDR notebook concatenates them into `nav`; the axon
   notebook uses `bbfront` alone. `mon_list` is the monitored node set.
4. **`NeuronModel`** — builds the passive RC netlist (`Ra`, `Rm`, `Cm`).
5. **The sweep** — gated on `RUN_SIMULATION`. See below.
6. **VDR map** — the headline figure: every input synapse coloured by its VDR. Driven by
   the results **file**, so it renders whether or not you ran the sweep.
7. **Trace / animation cells** — gated on `RUN_SIMULATION`, because they read the
   in-memory `sim_results` array from the last ngspice call. They print a skip notice
   when there is no live run.

## `RUN_SIMULATION`

Default is `False`: no sweep, results are read from `../results/` — the newest
`*_outs-<REGION>_*.tsv` if one exists, otherwise the committed reduced CSV for that
region.

Setting it to `True` runs one ngspice transient analysis per input synapse — **10,494
runs, hours of compute** — and writes to
`../results/DNa02-<bodyId>_All_ins-_outs-<REGION>_<timestamp>.tsv`. The region tag and
timestamp together guarantee that the two notebooks never overwrite each other and that
a re-run never clobbers an earlier sweep.

## Committing

Outputs are **stripped** in this directory, and should stay that way — it keeps diffs
readable and stops a printed token or local path from entering history. Clear outputs
before committing:

```bash
jupyter nbconvert --clear-output --inplace notebooks/*.ipynb
```

Rendered copies with figures live in [`../rendered/`](../rendered/).
