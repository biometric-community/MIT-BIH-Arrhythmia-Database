# MIT-BIH Arrhythmia Database

[![License: ODC-By 1.0](https://img.shields.io/badge/License-ODC--By%201.0-green)](https://opendatacommons.org/licenses/by/1-0/)
[![Access: public](https://img.shields.io/badge/access-public-0e8a16.svg)](https://physionet.org/content/mitdb/1.0.0/)

**MIT-BIH Arrhythmia Database** — 48 half-hour two-channel ambulatory ECG recordings with beat annotations; the classic PhysioNet arrhythmia-detection benchmark.

- **Upstream source**: https://physionet.org/content/mitdb/1.0.0/
- **DOI**: https://doi.org/10.13026/C2F305
- **Original archive (this folder)**: `mitdb-1.0.0.zip` (~73.5 MiB compressed; PhysioNet lists ~104.3 MiB uncompressed)
- **License (data)**: [Open Data Commons Attribution License v1.0](https://opendatacommons.org/licenses/by/1-0/) (PhysioNet)
- **License (this folder’s helpers / docs)**: CC BY 4.0 (see [`LICENSE`](LICENSE)) — docs only; signal files follow ODC-By 1.0

| Field | Value |
|-------|-------|
| Catalog id (tbiom) | `mit-bih` |
| Category | `physio` |
| Access | `public` |
| Upstream homepage | https://physionet.org/content/mitdb/1.0.0/ |

## TL;DR

- **Task**: ECG arrhythmia detection / beat classification
- **Modality**: two-channel ambulatory ECG (WFDB `.dat` / `.hea` / `.atr`)
- **Platform**: Holter-style recordings (BIH Arrhythmia Laboratory, 1975–1979)
- **Real/Synthetic**: real
- **Records**: **48** half-hour excerpts from **47** subjects (`RECORDS` list)
- **Sampling**: 360 Hz per channel, 11-bit over a 10 mV range
- **Annotations**: cardiologist-reconciled beat labels (~110,000 annotations)
- **Archive size**: `mitdb-1.0.0.zip` ≈ **77 MiB** — under GitHub’s 100 MiB limit (no sharding required)
- **Citation**: Moody & Mark, IEEE Eng. Med. Biol. 2001 (+ PhysioNet citation)

## Table of contents

- [Download](#download)
- [Dataset structure](#dataset-structure)
- [Annotation schema](#annotation-schema)
- [Stats and splits](#stats-and-splits)
- [Quick start](#quick-start)
- [Evaluation and baselines](#evaluation-and-baselines)
- [Datasheet (data card)](#datasheet-data-card)
- [Known issues and caveats](#known-issues-and-caveats)
- [License](#license)
- [Citation](#citation)
- [Contact](#contact)

## Download

- **This folder**: GitHub-safe shards under [`parts/`](parts/) (`mitdb-1.0.0.zip.part000`, `mitdb-1.0.0.zip.part001`, plus [`parts/MANIFEST.txt`](parts/MANIFEST.txt)). After `bash setup.sh restore`, files land under `extracted/`.
- **Upstream ZIP**: https://physionet.org/content/mitdb/1.0.0/ — place as `mitdb-1.0.0.zip`, then `bash setup.sh prepare` to re-shard.
- **Helper script** (from tbiom repo root):

```bash
bash projects/datasets/scripts/download_mit_bih.sh
```

### GitHub-safe shards (&lt;100 MiB)

`mitdb-1.0.0.zip` is ~74 MiB. Local helpers (same flow as LFW / 300W):

```bash
cd projects/datasets/mit-bih
bash setup.sh prepare   # verify + split into parts/ (40 MiB each by default)
bash setup.sh restore   # cat parts → unzip into extracted/
bash setup.sh verify    # integrity check only
```

- The full zip is removed after a successful `prepare` (replaced by `parts/`)
- Override chunk size: `CHUNK_SIZE=50M bash setup.sh prepare`

## Dataset structure

```text
mit-bih/
├── README.md
├── LICENSE
├── STATUS.md
├── setup.sh
├── RECORDS                         # 48 record IDs (optional local copy)
├── parts/
│   ├── MANIFEST.txt
│   ├── mitdb-1.0.0.zip.part000
│   └── mitdb-1.0.0.zip.part001
├── mit-bih-arrhythmia-database-1.0.0/   # local extract (often not committed)
│   ├── 100.hea / 100.dat / 100.atr / ...
│   └── ...
└── extracted/                      # created by setup.sh restore
    └── mit-bih-arrhythmia-database-1.0.0/
        └── ...
```

- **Splits**: no official ML train/test partition in the zip; AAMI / custom folds are defined in papers.
- **Layout notes**: WFDB naming — each record `NNN` has header (`.hea`), signal (`.dat`), and annotation (`.atr`) files.

## Annotation schema

### WFDB records (`*.hea`, `*.dat`, `*.atr`)

- **`.hea`**: sampling rate, gains, channel names, record length
- **`.dat`**: binary ECG samples (format described in the header)
- **`.atr`**: beat / rhythm annotations (reference labels)
- **Coordinates**: time is sample index at 360 Hz (not image boxes)
- **Example paths** (verified in local extract):

```text
mit-bih-arrhythmia-database-1.0.0/100.hea
mit-bih-arrhythmia-database-1.0.0/100.dat
mit-bih-arrhythmia-database-1.0.0/100.atr
```

Use [WFDB](https://physionet.org/content/wfdb/) / `wfdb` Python package to read signals and annotations.

## Stats and splits

| Measure | Count |
|---------|------:|
| Records (`RECORDS`) | 48 |
| Subjects | 47 |
| Duration per record | ~30 minutes |
| Channels | 2 |
| Sample rate | 360 Hz |
| Zip entries (local `mitdb-1.0.0.zip`) | 705 |

## Quick start

```bash
cd projects/datasets/mit-bih
bash setup.sh restore   # writes extracted/mit-bih-arrhythmia-database-1.0.0/
```

```python
from pathlib import Path

root = Path("extracted/mit-bih-arrhythmia-database-1.0.0")
# or Path("mit-bih-arrhythmia-database-1.0.0") if you already extracted locally
records = sorted({p.stem for p in root.glob("*.hea")})
print(len(records), "headers; first", records[:5])

# Optional: pip install wfdb
# import wfdb
# sig, fields = wfdb.rdsamp(str(root / "100"))
# ann = wfdb.rdann(str(root / "100"), "atr")
```

**Dependencies**:

- `bash`, `split`, `unzip` for `setup.sh`
- Optional: `wfdb` (Python) or PhysioNet WFDB software for signals/annotations

## Evaluation and baselines

- **Primary metrics** (literature): beat classification sensitivity/PPV, alarm false positives, AAMI EC57-style scores depending on the protocol
- **Suggested baselines**: classical MIT-BIH arrhythmia detectors and modern ECG CNN/RNN papers citing Moody & Mark (2001)
- **Baseline numbers**: not reproduced in this folder — see citing literature

## Datasheet (data card)

### Motivation

Provide a shared, annotated ambulatory ECG set for evaluating arrhythmia detectors.

### Composition

48 half-hour two-channel excerpts; 23 chosen randomly from a large Holter pool and 25 selected to cover rarer clinically important arrhythmias.

### Collection process

BIH Arrhythmia Laboratory recordings (1975–1979); digitized and annotated with multi-cardiologist review.

### Preprocessing

Distributed in PhysioNet WFDB format. Annotation file `102.atr` was corrected on 2018-07-06 (see PhysioNet release notes; original kept as `102-0.atr`).

### Distribution

- **Signal / annotation files**: ODC-By 1.0 via PhysioNet
- **Helpers / docs in this folder**: CC BY 4.0 (`LICENSE`)

### Maintenance

Re-download with `download_mit_bih.sh`. Prefer PhysioNet mirrors if a zip is incomplete.

## Known issues and caveats

- Prefer `parts/` + `setup.sh restore` for redistribution; do not commit the full `mitdb-1.0.0.zip` if it is near GitHub’s size limits
- Class imbalance and patient-specific folds matter; random beat splits leak identity
- Some auxiliary files (`.xws`, `.at_`, docs) ship in the PhysioNet tree — use `RECORDS` + `.hea/.dat/.atr` for core experiments
- Related PhysioNet sets (noise stress test, P-wave annotations) are **not** included here

## License

**Data files** are licensed under **[ODC-By 1.0](https://opendatacommons.org/licenses/by/1-0/)** as published by PhysioNet.

**Packaging helpers / docs** in this folder are **[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)**. See [`LICENSE`](LICENSE).

## Citation

When using MIT-BIH, cite the database paper and PhysioNet:

```bibtex
@article{MoodyMark2001,
  title   = {The impact of the {MIT-BIH} Arrhythmia Database},
  author  = {Moody, George B. and Mark, Roger G.},
  journal = {IEEE Engineering in Medicine and Biology Magazine},
  volume  = {20},
  number  = {3},
  pages   = {45--50},
  year    = {2001}
}

@misc{mitdb100,
  title        = {{MIT-BIH} Arrhythmia Database},
  author       = {Moody, George B. and Mark, Roger G.},
  howpublished = {PhysioNet},
  year         = {2005},
  note         = {Version 1.0.0},
  doi          = {10.13026/C2F305},
  url          = {https://physionet.org/content/mitdb/1.0.0/}
}
```

Also include the current PhysioNet platform citation required on the project page.

## Contact

- **Upstream / issues**: https://physionet.org/content/mitdb/1.0.0/
- **tbiom catalog**: `projects/datasets/mit-bih/`
# MIT-BIH-Arrhythmia-Database
