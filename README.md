# DNa02 Passive VDR — IDR + Axon vs Axon-only readout

Two parallel passive cable-theory simulations of the *Drosophila* **DNa02** descending
neuron. Every one of its **10,494 input synapses** is driven, one at a time, with a
single EPSP; the peak voltage at the input is compared to the peak voltage reached in a
monitored downstream region. The ratio is the **voltage decay ratio (VDR)**.

The two notebooks are identical apart from **where the voltage is read out**:

| Notebook | `REGION` | Monitored nodes | Represents |
|----------|----------|-----------------|------------|
| `..._IDRandAxonVDR.ipynb` | `IDRandAxonVDR_region` | ~73 | Inter-dendritic region **+** axon / DAS |
| `..._AxonVDR_region.ipynb` | `AxonVDR_region` | ~50 | Axon / DAS **only** |

The axon set is a strict subset of the IDR+axon set, so this asks a targeted question:
**does adding the IDR as a possible spike-initiation site bring inputs electrotonically
closer to their readout?**

Morphology and connectivity come from the hemibrain connectome (`DNa02_R`) and are
fetched live from NeuPrint, so **a token is required to re-run** (see
[Credentials](#credentials)). The committed results and rendered HTML need nothing.

## Result

From the completed sweeps in [`results/`](results/), pairing the two runs on the same
10,494 input synapses. VDR is `peak V at input ÷ peak V at the best-responding monitored
node`, so **higher = more decay**.

| | IDR + axon readout | Axon-only readout |
|---|---|---|
| Median VDR | **8.68** | **9.73** |
| IQR | 5.41 – 13.24 | 5.99 – 14.88 |
| Mean VDR | 10.05 | 11.28 |

The population difference resolves into two sharply separated groups:

| Group | n (% of inputs) | What happens |
|-------|-----------------|--------------|
| **Axon node still wins** | 4,885 (46.6 %) | The best-responding node is an axon node in both runs → VDR **identical** (median 9.01 vs 9.01; output voltages match to floating-point). |
| **An IDR node wins** | 5,609 (53.4 %) | An IDR node out-responds every axon node → VDR drops from 10.38 to 8.39, a **~1.28× reduction**. |

That 1.28× is strikingly uniform (median 1.28×, 90th percentile 1.28×, max 1.29×),
which points to a near-fixed electrotonic offset between the winning IDR node and the
winning axon node rather than an input-specific effect.

> **Read the sign of this correctly.** Because the IDR+axon monitor set *contains* the
> axon-only set, its per-input VDR can never be larger — the comparison is bounded by
> construction, and a signed-rank test on it is not informative (every non-zero
> difference has the same sign by definition). The meaningful findings are the **46.6 / 53.4
> split** and the **magnitude** of the reduction where the IDR wins, not the direction.

## Quick start

```bash
conda env create -f environment.yml     # creates env "vdr_ngspice" (Python 3.10)
conda activate vdr_ngspice
jupyter lab notebooks/
```

Read the results without installing or running anything: open the HTML in
[`rendered/`](rendered/).

To **re-run**, you additionally need:

1. **A NeuPrint token** — see [Credentials](#credentials). Both notebooks fetch the
   skeleton, synapses, and the SPICE netlist (`NeuronModel`) live.
2. **ngspice** — <http://ngspice.sourceforge.net/>. Point `NGSPICE_BIN` at its `bin`
   directory. On Windows also set `set filetype=binary` in
   `Spice64/share/ngspice/scripts/spinit`; on macOS/Linux the binary is `ngspice`
   rather than `ngspice_con`, so adjust the `Popen` call.

The notebooks find the repository root by walking up for the `notebooks/` + `results/`
layout, so you can launch Jupyter from anywhere in the tree.

### The `RUN_SIMULATION` switch

Both notebooks open with a config cell holding `REGION` and `RUN_SIMULATION`:

- `RUN_SIMULATION = False` *(default)* — skips the sweep and reads results from disk:
  the newest matching run in `results/`, falling back to the committed reduced CSV for
  that region. Needs no token-authenticated sweep and no ngspice.
- `RUN_SIMULATION = True` — runs the full sweep: one ngspice transient analysis per
  input synapse, 10,494 of them. This takes **hours**.

Sweep output is written to `results/` as
`DNa02-<bodyId>_All_ins-_outs-<REGION>_<timestamp>.tsv`. Because the filename carries
**both** the region and a timestamp, the two notebooks can never overwrite each other's
results, and a re-run never clobbers an earlier sweep.

Cells that need a live in-memory ngspice run (voltage traces over time, the per-timepoint
arbor maps, the animation) are gated on `RUN_SIMULATION` and print a skip notice
otherwise. The main VDR map is driven from the results file, so it renders either way.

## Credentials

**This repository contains no tokens, keys, or credentials.**

- Both notebooks read the token from the environment and fail with a clear message if it
  is absent — nothing is hard-coded:

  ```powershell
  $env:NEUPRINT_TOKEN = "<your-token>"      # PowerShell, current session
  setx NEUPRINT_TOKEN "<your-token>"        # PowerShell, persistent
  ```
  ```bash
  export NEUPRINT_TOKEN="<your-token>"       # bash
  ```

  `NEUPRINT_APPLICATION_CREDENTIALS` is accepted as a fallback name. Get a free token at
  <https://neuprint.janelia.org> → Account → Auth Token.
- Local machine paths are environment variables too (`NGSPICE_BIN`), not literals.
- Notebook outputs are **stripped** in `notebooks/`, so a printed token or local path can
  never enter history. Read the results in [`rendered/`](rendered/) instead — those were
  exported with credentials and personal paths redacted from the cell source.
- `.gitignore` blocks `.env` and `*token*` filenames.

> **Before making this repository public**, run a secret scan over the *history*, not just
> the checkout (e.g. `gitleaks detect`, or GitHub push protection, on by default for
> public repos).

## Layout

```
├── notebooks/      The two experiments (outputs stripped) + RandCValues.png
├── rendered/       HTML exports with all figures — read without re-running
├── results/        Committed reduced sweep results — see results/README.md
├── environment.yml / requirements.txt   (dependencies)
└── LICENSE
```

## Method

- **Passive cable model via ngspice.** `neuprint.NeuronModel` turns the skeleton into a
  SPICE netlist of per-compartment RC elements. Approach and R/C values follow Scheffer
  et al. 2020; see `RandCValues.png` in `notebooks/`.
  - `Ra = 1.2` (MED), `Rm = 0.8` (MED), `Cm = 1e-2`.
- **Synaptic drive.** One EPSP per run, injected at a single input synapse through a
  30 GΩ series resistor: `EXP(0 60.0 0.1 τ_rise 1.1 τ_decay 40)` with
  τ_rise = 0.6 ms, τ_decay = 2.1 ms (cholinergic values), over a 40 ms transient
  analysis at 0.1 ms steps.
- **Synapse → node mapping.** Each synapse is assigned to its nearest skeleton node by
  `cKDTree` query (`swcid = index + 1`, matching the SWC 1-based `rowId`).
- **Region definition.** Both regions are carved out of the skeleton geometrically:
  `navis.longest_neurite` gives the primary neurite, and coordinate windows on it select
  the IDR block (`bb`, y 17,000–22,000) and the axon block (`bbfront`, from `ln4`, first
  3 nodes dropped). The IDR+axon notebook concatenates them; the axon notebook uses
  `bbfront` alone.
- **VDR.** Per input synapse: peak voltage at the driven node ÷ peak voltage at the
  monitored node that responded most strongly. Peaks are taken independently, so the two
  times differ (input peaks ≈ 1.2 ms, readout ≈ 3.8–5.4 ms).

## Provenance

The notebooks are the working-copy versions with these changes, and **no change to the
simulation physics or parameters**:

1. **Credential scrub** — a hard-coded NeuPrint JWT in the client cell was replaced with
   an environment-variable read. *This token should be revoked and reissued by its owner:*
   it sat in plain text in shared-drive notebooks and does not expire until 2028. It was
   never committed to any git history.
2. **Path scrub** — hard-coded `Z:\...` ngspice and working-directory paths became
   `NGSPICE_BIN` and the `REPO_ROOT` bootstrap. The `os.chdir` cells were removed.
3. **`RUN_SIMULATION` gate** — the IDR notebook's sweep had been commented out, so it
   could not run end to end while the axon notebook's ran unconditionally. Both now share
   one gated sweep cell with region-tagged, timestamped output filenames.
4. **Results-parsing fix** — the VDR-map cell parsed the sweep file **positionally**,
   starting at column 0. The writer emits `combo_id` as column 0, so every field was read
   one slot to the left (and `int("2_to_5625")` would raise). It now selects by column
   name. This is why the figure is regenerated rather than reused.
5. **NumPy 2.0 compatibility** — the ngspice raw-file parser used `np.complex_`, removed
   in NumPy 2.0, which would abort the sweep on any current environment. Replaced with
   `np.complex128`, which is the same dtype and works on NumPy 1.x and 2.x alike.

The `rendered/` HTML is exported from the original executed notebooks, with credentials
and local paths redacted from the cell source and outputs left untouched.

## Literature

- Scheffer et al. 2020, *eLife* 9:e57443 — hemibrain connectome, R/C values
- Gouwens & Wilson 2009, *J Neurosci* 29:6239 — signal propagation in *Drosophila* central neurons
- Segev et al. 1985, *Biol Cybern* — compartmental modelling via circuit analysis
- Gornet & Scheffer, *bioRxiv* 177113 — simulating extracted connectomes

With thanks to Lou Scheffer, Stuart Berg, and Steve Plaza.

> **Caveat.** This is a purely **passive** model — no voltage-gated conductances. It
> bounds how far synaptic signals travel by cable filtering alone; it does not model
> spike initiation. VDR values are model-internal comparisons (region vs region), not
> calibrated predictions.

## License

MIT — see [LICENSE](LICENSE).
