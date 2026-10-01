# LAMB–SKEWER
Long-read Allocation Model Builder and Structural Karyotype Estimation, Weighting, and Event Resolver

**Molecule-aware normalization, structural-variant discovery, and copy-number estimation for long-read sequencing data.**

LAMB–SKEWER connects cross-sample read-length normalization with structural-variant analysis while preserving individual molecules as evidence. LAMB builds the normalization model; SKEWER uses the original alignments and that model to identify structural events, recount supporting molecules, and report raw and normalized results.

## Programs

| Program | Purpose |
| --- | --- |
| **LAMB** · Long-read Allocation Model Builder | Balances read-length composition across samples using shared length strata. Produces per-molecule normalization weights, a reproducible selection manifest, and diagnostic reports. Physical BAM downsampling is optional. |
| **SKEWER** · Structural Karyotype Estimation, Weighting, and Event Resolver | Reconstructs long-read alignment paths to detect duplications, deletions, and inversions. Uses bundled SVCROWS to define canonical events, then estimates molecule support, interval copy counts, regional copy number, and exploratory duplication heterogeneity. |

SKEWER defines event boundaries from alignment evidence before estimating copy number. Raw and LAMB-weighted results remain available separately. Region-level counting is the default; a GFF is not required.

## Requirements

- Linux or a Linux environment such as WSL2, with Bash, `tar`, and Python 3 available for setup.
- Conda available on your `PATH`. The supplied environment files install the analysis dependencies.
- Coordinate-sorted long-read BAM files with their BAI or CSI indexes. Preserve supplementary alignments for structural-variant discovery.
- For SKEWER, the reference FASTA used to generate the BAMs. All samples should use the same reference and chromosome names.

LAMB does not require a reference FASTA. SKEWER includes LAMB 1.5.0 and the SVCROWS source locally; setup does not download either program from GitHub. Conda dependencies require package-channel access or a populated package cache.

## Installation

The commands below match **LAMB 1.5.0** and **SKEWER 1.1.0**. Place both archives in your working directory. Adjust filenames and paths if using another release.

### 1. Unpack the archives

The LAMB archive contains its files directly, so extract it into a dedicated folder. The SKEWER archive already contains a versioned top-level folder.

```bash
mkdir -p LAMB_v1.5.0

tar -xzf LAMB_v1.5.0.tar.gz -C LAMB_v1.5.0
tar -xzf SKEWER_v1.1.0.tar.gz

# Save absolute installation paths for the commands below.
LAMB_ROOT="$PWD/LAMB_v1.5.0"
SKEWER_ROOT="$PWD/SKEWER_v1.1.0"
```

Keep the hidden `.lamb` and `.skewer` directories with their wrappers. They contain the program implementations.

### 2. Set up LAMB

```bash
bash "$LAMB_ROOT/setup_lamb.sh"

conda env create \
  --name lamb \
  --file "$LAMB_ROOT/bin/modules/yaml/lamb.yml"

conda run --no-capture-output -n lamb \
  bash "$LAMB_ROOT/bin/modules/lamb.sh" --help
```

The setup script checks the extracted layout and sets executable permissions. The separate Conda command installs dependencies. These examples explicitly name the environment `lamb` in lowercase.

### 3. Set up SKEWER

```bash
bash "$SKEWER_ROOT/bin/modules/.skewer/setup_skewer.sh" --standalone

bash "$SKEWER_ROOT/bin/modules/skewer.sh" --help
```

SKEWER setup creates its own Conda environment under `.skewer/env` and installs the bundled SVCROWS package. Its wrapper selects that environment automatically; manual activation is unnecessary.

If you only want SKEWER to run its bundled LAMB automatically, you can skip the separate LAMB installation.

## Basic workflow

Run the following commands from your analysis directory. Replace the example paths with your own inputs. If opening a new terminal, set `LAMB_ROOT` and `SKEWER_ROOT` again to the absolute installation paths above.

```bash
BAM_DIR="/absolute/path/to/original_bams"
REFERENCE="/absolute/path/to/reference.fasta"
```

### 1. Normalize with LAMB

```bash
conda run --no-capture-output -n lamb \
  bash "$LAMB_ROOT/bin/modules/lamb.sh" \
  --bam-dir "$BAM_DIR" \
  --require-index \
  --length-strata 20 \
  --min-read-length 1000 \
  --max-read-length 100000 \
  --min-bin-molecules 10 \
  --sparse-bin-policy merge \
  --mode maximalist \
  --seed 0 \
  --threads 4 \
  --out-dir ./lamb_out
```

This runs inventory, stratification, allocation, manifest generation, validation, and reporting. The length limits are example analysis settings, not universal requirements; choose them to suit your dataset. Length strata describe **read-length ranges**, not genomic windows. Sparse-bin merging can reduce the final number of strata below the requested 20.

| Allocation mode | Behavior |
| --- | --- |
| `minimalist` | Uses a conservative common allocation based on the per-bin minimum across samples, with equal target molecule counts per bin across samples. |
| `maximalist` | Optimizes a shared target length distribution to maximize the lowest sample retention, then total retention. Sample totals can differ. |

LAMB's mapping-quality measurements are diagnostic and do not determine normalization weights. The example retains the default statistical workflow, without producing downsampled BAMs.

### 2. Analyze structural events with SKEWER

```bash
bash "$SKEWER_ROOT/bin/modules/skewer.sh" \
  --bam-dir "$BAM_DIR" \
  --reference "$REFERENCE" \
  --lamb-dir ./lamb_out \
  --require-index \
  --threads 4 \
  --out-dir ./skewer_out
```

**Use the original BAMs, including supplementary alignments.** SKEWER combines these with the LAMB model and manifest. Do not substitute LAMB's physically downsampled BAMs for this workflow. Discovery preserves eligible structural evidence even from molecules with zero normalization weight.

Sample IDs must match between the two programs. When using `--bam-dir`, IDs are BAM filenames without the `.bam` suffix. Both programs also support `--sample-sheet` with tab-separated `sample_id` and `bam` columns for explicit naming.

### Alternative: let SKEWER run LAMB automatically

After installing SKEWER, omit `--lamb-dir` to run its bundled LAMB:

```bash
bash "$SKEWER_ROOT/bin/modules/skewer.sh" \
  --bam-dir "$BAM_DIR" \
  --reference "$REFERENCE" \
  --length-strata 20 \
  --lamb-mode maximalist \
  --require-index \
  --threads 4 \
  --out-dir ./skewer_auto_out
```

Bundled LAMB outputs are saved under `skewer_auto_out/01_inventory/lamb/`. This shortcut does not reproduce all the length and sparse-bin settings in the separate LAMB example. Use a separate LAMB run when you need those controls. Without an explicit `--lamb-mode`, SKEWER defaults to `minimalist`.

## Results to open first

Paths below are relative to each program's output directory.

| Program | File | Contents |
| --- | --- | --- |
| LAMB | `06_report/lamb_report.html` | Read-length distributions, retention summaries, and per-sample normalization plots. |
| LAMB | `03_model/allocation.tsv` | Per-sample, per-stratum targets and weights. |
| LAMB | `04_manifest/molecule_manifest.tsv.gz` | Per-molecule normalization and deterministic selection information. |
| SKEWER | `07_report/index.html` | Summary report with links to individual event pages. |
| SKEWER | `03_canonical/events.tsv` | Canonical SV IDs, event types, and read-defined intervals. |
| SKEWER | `04_reads/read_observations.tsv.gz` | Molecule-level observations, support, flank evidence, and copy labels. |
| SKEWER | `05_abundance/counts_raw.tsv`, `counts_weighted.tsv` | Raw and normalized support and copy totals. |
| SKEWER | `05_abundance/copy_number.tsv` | Regional depth-based copy-number estimates. |
| SKEWER | `06_heterogeneity/heterogeneity.tsv` | Exploratory duplication copy-distribution diagnostics. |

Keep each report directory together when copying HTML reports. An observed molecule is not necessarily an SV-supporting molecule: reference-like and internal reads provide useful context without proving a diagnostic junction. Exact copy labels such as `2` differ from lower-bound labels such as `2+`.

## Visualize an event

SKEWER can export an evidence bundle with the canonical SV ID, local background reads, supporting molecules, alignment paths, and saved CN/depth information.

```bash
bash "$SKEWER_ROOT/bin/modules/skewer.sh" visualize \
  --run-dir ./skewer_out \
  --svid SV_YOUR_ID \
  --output-dir ./skewer_visualization
```

Replace `SV_YOUR_ID` with an ID from `03_canonical/events.tsv`. Omit `--svid` to select up to three well-supported events per event type. Open the bundle's `index.html`, or load a sample's `igv_session.xml` in IGV Desktop. Copy the complete bundle for portable viewing.

## Resume a run

Both programs reuse valid completed stages:

```bash
conda run --no-capture-output -n lamb \
  bash "$LAMB_ROOT/bin/modules/lamb.sh" --out-dir ./lamb_out

bash "$SKEWER_ROOT/bin/modules/skewer.sh" --out-dir ./skewer_out
```

Keep the original inputs and output directories available. Changed settings or inputs can require rebuilding stages. A resumed unfinished stage starts again from its beginning.

## Optional physical downsampling

To materialize the deterministic subset already selected by LAMB:

```bash
conda run --no-capture-output -n lamb \
  bash "$LAMB_ROOT/bin/modules/lamb.sh" materialize \
  --out-dir ./lamb_out \
  --threads 4
```

Indexed BAMs are written to `lamb_out/07_materialized_bams/`. Every alignment record belonging to a selected molecule is retained. This option is separate from the original-BAM SKEWER workflow above.

## Existing MalariAPI installations

SKEWER can also be installed into an existing MalariAPI checkout:

```bash
bash "$SKEWER_ROOT/bin/modules/.skewer/setup_skewer.sh" \
  --mapi-root "$HOME/MalariAPI"

mapi skewer --help
```

After installation, use `mapi skewer` in place of the standalone wrapper. For LAMB already installed as a MAPI module, use `mapi lamb` in place of the Conda-plus-wrapper command. MAPI is optional for the standalone workflow.

## Scope and documentation

SKEWER currently targets tandem duplications, deletions, and inversion evidence. Insertions, translocations, general complex-event assembly, and haplotype phasing are outside this release. Heterogeneity results describe an exploratory capture model and should not be interpreted directly as cell fractions.

For full parameters, run either wrapper with `--help`. Detailed documentation ships in:

- LAMB: `docs/LAMB_OUTPUT_MANIFEST.md` and `docs/LAMB_V1_MIGRATION.md` beneath its installation root.
- SKEWER: `bin/modules/.skewer/docs/README.md`, `METHODS.md`, `OUTPUTS.md`, and `VISUALIZATION.md` beneath its installation root.


  
