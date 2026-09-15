# Common Pile v0.1 — CoreML copy (dataset info)

*Last updated: 2026-09-15. Owner: shaneb. For CoreML pretraining experiments.*
*Token counts MEASURED 2026-09-15 by scanning the HDF5 (`tokens = num_sequences x 8192`); filtering/size context from Mostafa's Slack notes (2026-09) and The Common Pile v0.1 paper (arXiv:2506.05209).*

## Quick answer (paths + tokens)

- **Recommended data to train on (HDF5, tokenized, globally shuffled):**
  `/cb/home/mostafae/research/outdirs/datasets/experiments/20260908_commonpilev0.1_filtered_reprocess/dataset_shuffled/train/`  (val: `.../dataset_shuffled/val/`)  — on **mb306** (node `cs306-wse022-us-sr01`).
- **Seq length:** 8192 tok/seq · **Tokenizer:** Llama-3, vocab **128256** (`/cb/home/gaviag/datasets/common_pile/tokenizer/`).
- **Our filtered, 23-source training set (measured):** **~402.8B train tokens** (402,829,606,912; 49,173,536 sequences) + ~2.42B val.
- **Full Common Pile v0.1 *Filtered*:** ~1.84 TB text ≈ ~400B Llama-3 tokens (our copy matches this). **Raw:** ~8 TB ≈ ~1.6T tokens.
- Per-source token counts: [Sources & token counts](#sources--token-counts).

> **Note on an earlier number:** an older processing (`20260527_common_pile_layer_drop`) was smaller
> (~241B tokens across the same 23 sources). The **current `20260908` reprocess is the full ~403B** —
> use it. If you see 241B quoted anywhere, it's stale.

## Filtered vs. Raw, and total size

Common Pile v0.1 exists in two forms (paper Table 6; sizes per Gavia/Mostafa):

| Variant | Text size | ≈ Llama-3 tokens | Notes |
|---|---|---|---|
| **Filtered** (what we use) | ~1.84 TB | **~400B** | English-only + quality/toxicity/PII/boilerplate filtered |
| Raw | ~8 TB | ~1.6T | unfiltered; option if we need many more tokens |

Our filtered+shuffled training copy **measures ~402.8B tokens** (counted directly), consistent with
the ~400B "filtered" figure. It already **excludes** StackExchange, CC YouTube, and 6 tiny sources
(see [Excluded sources](#excluded-sources-vs-paper-table-7)).

> **"Raw" vs what we have — avoid confusion.** Everything on `mb306` under `common_pile/` is the
> **Filtered** variant, and already **tokenized** (Llama-3, vocab 128256). Specifically,
> `/cb/home/gaviag/datasets/common_pile/data/` is the **tokenized filtered `.bin` source**
> (not raw text, and not the unfiltered ~8 TB "Raw" Common Pile). It's only "raw" in the sense of
> *pre-HDF5-packing*. The unfiltered **Raw** Common Pile (~8 TB / ~1.6T tokens) is **not present here** —
> it would have to be sourced/tokenized separately if needed.

**Filtering applied to produce the Filtered set** (paper Table 6):
- **Language:** English only (FastText language classifier; other languages removed).
- **Web-text quality:** low-threshold DataComp-LM–style quality classifier on Creative-Commons Common Crawl.
- **OCR quality:** drop docs with pervasive OCR errors (low likelihood under a unigram LM trained on the Trillion Word Corpus).
- **Toxicity:** two FastText toxicity classifiers (Jigsaw Toxic Comment Challenge).
- **PII:** redact emails / phone numbers / IPs → placeholders (e.g. `<EMAIL_ADDRESS>`).
- **Boilerplate & repetition:** source-specific regexes remove page numbers, preambles, license statements, etc.

## Locations (on mb306 / `cs306-wse022-us-sr01`)

| What | Path |
|---|---|
| **Filtered tokenized `.bin` source** (input to our HDF5) — 31 `*_filtered` source dirs, ~797 GB | `/cb/home/gaviag/datasets/common_pile/data/` (holds `common_pile_train_*.bin`; incl. stackexchange/youtube/news, which we exclude from training) |
| Build scripts | `/cb/home/gaviag/datasets/common_pile/` (`bin_to_hdf5.py`, `compute_weights.py`, `common_pile.py`) |
| Tokenizer (vocab 128256) | `/cb/home/gaviag/datasets/common_pile/tokenizer/` |
| **HDF5 shuffled (RECOMMENDED)** — 2026-09-08 reprocess, ~403B tok | `/cb/home/mostafae/research/outdirs/datasets/experiments/20260908_commonpilev0.1_filtered_reprocess/dataset_shuffled/{train,val}/` |
| HDF5 raw (unshuffled) — same reprocess | `.../20260908_commonpilev0.1_filtered_reprocess/dataset/{train,val}/` |
| HDF5 original (unshuffled) — 2026-05-27, ~241B tok (stale) | `/cb/home/mostafae/research/outdirs/drop_path/experiments/20260527_common_pile_layer_drop/dataset/{train,val}/` |
| Alt capped variants (Gavia) | `/cb/home/gaviag/datasets/common_pile/hdf5_8192_10B/`, `.../hdf5_8192_100B/` |

**Use the shuffled copy** (`dataset_shuffled/`). Mostafa: *"I have done global shuffling of the dataset
here that hopefully will be less troublesome."* Unshuffled copies cause loss artifacts with `shuffle: false`
(see [Known issues](#known-issues--caveats)).

## How it was processed (2026-09-08 reprocess)

```bash
OUT_DIR=/cb/home/mostafae/research/outdirs/datasets/experiments/20260908_commonpilev0.1_filtered_reprocess
mkdir -p "$OUT_DIR"
LOG="$OUT_DIR/runs.log"

# 1) Gavia's Common Pile preprocessing: tokenized .bin -> packed HDF5 sequences (len 8192)
python /cb/home/gaviag/datasets/common_pile/bin_to_hdf5.py \
    --input_dir /cb/home/gaviag/datasets/common_pile/data \
    --output_dir "$OUT_DIR/dataset" \
    --seq_len 8192 \
    2>&1 | tee -a "$LOG"

# 2) global offline shuffle (within + across each source's .h5 files)
python /cb/home/mostafae/research/monolith_config_gen/mostafae/utils/shuffle_h5_data.py \
    --input_dir "$OUT_DIR/dataset" \
    --output_dir "$OUT_DIR/dataset_shuffled" \
    2>&1 | tee -a "$LOG"
```

Train and val are **separate directory splits made before shuffling** → they contain **different
documents** (expected holdout; see caveats).

## Getting a different tokenization (or the unfiltered Raw)

The `.bin`/HDF5 on mb306 are **Llama-3 tokenized** (vocab 128256) and **cannot be reused for a different
tokenizer** — you re-run Gavia's pipeline, which pulls the **text from HuggingFace** and tokenizes on the
fly. There is **no local text copy**: `common_pile.py` streams each subset via
`load_dataset("common-pile/<subset>", split="train")` (cached under `HF_HOME`) and tokenizes with
`AutoTokenizer.from_pretrained(--tokenizer)` (BOS prepended per document; int32 `.bin` shards).

Recipe (scripts in `/cb/home/gaviag/datasets/common_pile/`; see its `Makefile` for exact targets — the
heavy run was done on a scratch node `jnodea2`, then rsync'd to mb306):

```bash
# 1) download text from HF + tokenize with a DIFFERENT tokenizer -> .bin (int32 tokens, BOS/doc)
python common_pile.py \
    --tokenizer /path/to/OTHER_tokenizer \
    --output_dir /path/to/new_bins
    # optional: --datasets peS2o_filtered github_archive_filtered ...   # subset
    #           --shard_size 100000000  --nprocs N ;  set HF_HOME for the download cache

# 2) pack to HDF5 (choose seq_len)
python bin_to_hdf5.py --input_dir /path/to/new_bins --output_dir /path/to/new_hdf5 --seq_len 8192

# 3) global shuffle (recommended for shuffle:false training)
python /cb/home/mostafae/research/monolith_config_gen/mostafae/utils/shuffle_h5_data.py \
    --input_dir /path/to/new_hdf5 --output_dir /path/to/new_hdf5_shuffled
```

Notes:
- **Unfiltered Raw variant:** point `common_pile.py` at the raw HF datasets instead of the `*_filtered`
  ones (edit its `DATASETS` list / the `common-pile/<name>` id), then steps 2–3 as above (~8 TB / ~1.6T tokens).
- `.bin` stores **int32** tokens (fine for any vocab) with a **BOS prepended per document**.
- Set the training YAML `vocab_size` to the new tokenizer's vocab size.
- Run on a node with HF access + scratch space; it re-downloads ~all of Common Pile and re-tokenizes (compute + bandwidth heavy).

## HDF5 format

- Per source: `<name>_000000.h5 …` + a `MANIFEST`. Dataset key `data`, shape **`(<=100000, 3, 8192)`**,
  dtype **`int32`** (100k sequences per full file, ~9.83 GB/file). **Channel 0 = input_ids** (0..128255).
- Index → file: `file = seq_index // 100000`.

## Sources & token counts

Measured 2026-09-15 from the shuffled HDF5. "1 epoch at" = training tokens drawn before that source is
seen once at its weight (smallest = binding constraint).

| Source (`*_filtered`) | Train tokens | Train sequences (8192) | % of tokens | Config weight | Normalized weight | 1 epoch at (train tokens) |
|---|---:|---:|---:|---:|---:|---:|
| uspto_filtered | 152,758,386,688 | 18,647,264 | 37.92% | 0.04133 | 0.04808 | 3177B |
| stackv2_edu_filtered | 62,289,666,048 | 7,603,719 | 15.46% | 0.12783 | 0.14871 | 419B |
| peS2o_filtered | 39,412,088,832 | 4,811,046 | 9.78% | 0.27409 | 0.31886 | 124B |
| pubmed_filtered | 35,224,297,472 | 4,299,841 | 8.74% | 0.03683 | 0.04285 | 822B |
| caselaw_access_project_filtered | 17,263,476,736 | 2,107,358 | 4.29% | 0.01941 | 0.02258 | 765B |
| wikimedia_filtered | 13,980,123,136 | 1,706,558 | 3.47% | 0.08616 | 0.10023 | 139B |
| cccc_filtered | 13,887,700,992 | 1,695,276 | 3.45% | 0.08716 | 0.10140 | 137B |
| pre_1929_books_filtered | 10,464,231,424 | 1,277,372 | 2.60% | 0.01161 | 0.01351 | 775B |
| github_archive_filtered | 10,108,518,400 | 1,233,950 | 2.51% | 0.06064 | 0.07055 | 143B |
| biodiversity_heritage_library_filtered | 8,515,592,192 | 1,039,501 | 2.11% | 0.0022 | 0.00256 | 3327B |
| library_of_congress_filtered | 7,963,901,952 | 972,156 | 1.98% | 0.0022 | 0.00256 | 3112B |
| usgpo_filtered | 7,675,002,880 | 936,890 | 1.91% | 0.0023 | 0.00268 | 2868B |
| arxiv_papers_filtered | 6,010,593,280 | 733,715 | 1.49% | 0.02932 | 0.03411 | 176B |
| project_gutenberg_filtered | 4,807,999,488 | 586,914 | 1.19% | 0.005 | 0.00582 | 827B |
| wikiteam_filtered | 2,844,975,104 | 347,287 | 0.71% | 0.01371 | 0.01595 | 178B |
| doab_filtered | 2,700,517,376 | 329,653 | 0.67% | 0.01801 | 0.02095 | 129B |
| uk_hansard_filtered | 1,911,947,264 | 233,392 | 0.47% | 0.01441 | 0.01676 | 114B |
| ubuntu_irc_filtered | 1,659,748,352 | 202,606 | 0.41% | 0.00791 | 0.00920 | 180B |
| regulations_filtered | 1,180,499,968 | 144,104 | 0.29% | 0.00761 | 0.00885 | 133B |
| stackv2_html_filtered | 1,002,897,408 | 122,424 | 0.25% | 0.00225974 | 0.00263 | 381B |
| data_provenance_initiative_filtered | 717,348,864 | 87,567 | 0.18% | 0.0051 | 0.00593 | 121B |
| arxiv_abstracts_filtered | 424,452,096 | 51,813 | 0.11% | 0.0036 | 0.00419 | 101B |
| pressbooks_filtered | 25,640,960 | 3,130 | 0.01% | 0.0009 | 0.00105 | 24B |
| **TOTAL (23 sources)** | **402,829,606,912** (~402.8B) | **49,173,536** | 100% | 0.8596 | 1.00000 | — |

Val: ~2.42B tokens (2,419,974,144) across 28 source dirs (~100M tokens each; the val dir
holds a few extra tiny sources — foodista, libretexts, oercommons, public_domain_review, PEPs — but the
dataloader uses the same 23 as train).

**Epoch limit (practical):** the meaningful constraint is the highest-weight source, **`peS2o`**
(32% weight, ~39B tokens) — it completes 1 epoch and begins repeating at **~124B training tokens**;
the rest of the high-weight cluster (cccc ~137B, wikimedia ~139B, github ~143B) follows soon after.
So training up to roughly **~120B tokens is ~single-epoch** on the sources that dominate the loss.
(Tiny sources repeat earlier — pressbooks at ~24B — but at 0.01% weight that's negligible.)
Note weights are **not** natural token proportions: e.g. `uspto` is 38% of tokens but only 4.8% weight,
while `peS2o` is 10% of tokens but 31.9% weight (up-weighted).

## Excluded sources (vs. paper Table 7)

Dropped and **not** renormalized (loader renormalizes at load; raw weights sum to ~0.860):
StackExchange (~13.47%), CC YouTube (~0.47%), Foodista, LibreTexts, News, OERCommons, Public Domain
Review, PEPs. Kept subset ≈ 85.96% of the paper's Table-7 mixing weights.

## Weighting used

Config (raw) weights in the table/YAML (sum ~0.860); the H5 `Mixture` normalizes to sum 1 at load
(normalized column). Derived from paper Table 7 with YouTube + StackExchange removed
(see `compute_weights.py`).

## Sample YAML (train dataloader)

```yaml
    train_dataloader:
      batch_size: 160
      data_processor: GptHDF5MapDataProcessor
      mixture:
        - data_dir: /cb/home/mostafae/research/outdirs/datasets/experiments/20260908_commonpilev0.1_filtered_reprocess/dataset_shuffled/train/arxiv_abstracts_filtered
          weight: 0.0036
        - data_dir: /cb/home/mostafae/research/outdirs/datasets/experiments/20260908_commonpilev0.1_filtered_reprocess/dataset_shuffled/train/arxiv_papers_filtered
          weight: 0.02932
        - data_dir: /cb/home/mostafae/research/outdirs/datasets/experiments/20260908_commonpilev0.1_filtered_reprocess/dataset_shuffled/train/biodiversity_heritage_library_filtered
          weight: 0.0022
        - data_dir: /cb/home/mostafae/research/outdirs/datasets/experiments/20260908_commonpilev0.1_filtered_reprocess/dataset_shuffled/train/caselaw_access_project_filtered
          weight: 0.01941
        - data_dir: /cb/home/mostafae/research/outdirs/datasets/experiments/20260908_commonpilev0.1_filtered_reprocess/dataset_shuffled/train/cccc_filtered
          weight: 0.08716
        - data_dir: /cb/home/mostafae/research/outdirs/datasets/experiments/20260908_commonpilev0.1_filtered_reprocess/dataset_shuffled/train/data_provenance_initiative_filtered
          weight: 0.0051
        - data_dir: /cb/home/mostafae/research/outdirs/datasets/experiments/20260908_commonpilev0.1_filtered_reprocess/dataset_shuffled/train/doab_filtered
          weight: 0.01801
        - data_dir: /cb/home/mostafae/research/outdirs/datasets/experiments/20260908_commonpilev0.1_filtered_reprocess/dataset_shuffled/train/github_archive_filtered
          weight: 0.06064
        - data_dir: /cb/home/mostafae/research/outdirs/datasets/experiments/20260908_commonpilev0.1_filtered_reprocess/dataset_shuffled/train/library_of_congress_filtered
          weight: 0.0022
        - data_dir: /cb/home/mostafae/research/outdirs/datasets/experiments/20260908_commonpilev0.1_filtered_reprocess/dataset_shuffled/train/peS2o_filtered
          weight: 0.27409
        - data_dir: /cb/home/mostafae/research/outdirs/datasets/experiments/20260908_commonpilev0.1_filtered_reprocess/dataset_shuffled/train/pre_1929_books_filtered
          weight: 0.01161
        - data_dir: /cb/home/mostafae/research/outdirs/datasets/experiments/20260908_commonpilev0.1_filtered_reprocess/dataset_shuffled/train/pressbooks_filtered
          weight: 0.0009
        - data_dir: /cb/home/mostafae/research/outdirs/datasets/experiments/20260908_commonpilev0.1_filtered_reprocess/dataset_shuffled/train/project_gutenberg_filtered
          weight: 0.005
        - data_dir: /cb/home/mostafae/research/outdirs/datasets/experiments/20260908_commonpilev0.1_filtered_reprocess/dataset_shuffled/train/pubmed_filtered
          weight: 0.03683
        - data_dir: /cb/home/mostafae/research/outdirs/datasets/experiments/20260908_commonpilev0.1_filtered_reprocess/dataset_shuffled/train/regulations_filtered
          weight: 0.00761
        - data_dir: /cb/home/mostafae/research/outdirs/datasets/experiments/20260908_commonpilev0.1_filtered_reprocess/dataset_shuffled/train/stackv2_edu_filtered
          weight: 0.12783
        - data_dir: /cb/home/mostafae/research/outdirs/datasets/experiments/20260908_commonpilev0.1_filtered_reprocess/dataset_shuffled/train/stackv2_html_filtered
          weight: 0.00225974
        - data_dir: /cb/home/mostafae/research/outdirs/datasets/experiments/20260908_commonpilev0.1_filtered_reprocess/dataset_shuffled/train/ubuntu_irc_filtered
          weight: 0.00791
        - data_dir: /cb/home/mostafae/research/outdirs/datasets/experiments/20260908_commonpilev0.1_filtered_reprocess/dataset_shuffled/train/uk_hansard_filtered
          weight: 0.01441
        - data_dir: /cb/home/mostafae/research/outdirs/datasets/experiments/20260908_commonpilev0.1_filtered_reprocess/dataset_shuffled/train/usgpo_filtered
          weight: 0.0023
        - data_dir: /cb/home/mostafae/research/outdirs/datasets/experiments/20260908_commonpilev0.1_filtered_reprocess/dataset_shuffled/train/uspto_filtered
          weight: 0.04133
        - data_dir: /cb/home/mostafae/research/outdirs/datasets/experiments/20260908_commonpilev0.1_filtered_reprocess/dataset_shuffled/train/wikimedia_filtered
          weight: 0.08616
        - data_dir: /cb/home/mostafae/research/outdirs/datasets/experiments/20260908_commonpilev0.1_filtered_reprocess/dataset_shuffled/train/wikiteam_filtered
          weight: 0.01371
      num_samples: 750000000   # epoch cap in sequences (real epoch ~49M seq -> cap not hit)
      repeat: true
      shuffle: false           # interleaved sampling; REQUIRES pre-shuffled on-disk data (dataset_shuffled/)
      shuffle_seed: 1
      use_worker_cache: false
      vocab_size: 128256
    val_dataloader:            # same 23 sources/weights -> .../dataset_shuffled/val/
      ...
```

## Known issues & caveats

1. **Use `dataset_shuffled/`.** With `shuffle: false` + interleaving, unshuffled HDF5 groups content by
   write-order → reproducible loss steps at `.h5` file boundaries (e.g. `stackv2_edu`: prose-heavy repos
   ~2.8 → pure source code ~1.6–1.8 at a file boundary). Not duplicates/bad data — just ordering.
2. **Train vs val = different documents** (pre-shuffle split) → expect a standing ~0.2 train/val loss gap.
   Benign; shuffling doesn't make val i.i.d. with train. For a tight yardstick, use a random per-sequence holdout.
3. **`num_samples: 750000000`** caps the epoch only *down*; the real epoch is ~49M sequences,
   so it isn't hit — the loader loops past one epoch.

## References

- The Common Pile v0.1 — arXiv:2506.05209 (Table 6 raw-vs-filtered; Tables 3–4 peS2o split; Table 7 mixture).
- Build/scripts: `/cb/home/gaviag/datasets/common_pile/` (`bin_to_hdf5.py`, `compute_weights.py`); shuffle: `/cb/home/mostafae/research/monolith_config_gen/mostafae/utils/shuffle_h5_data.py` (Mostafa).
- Reprocess log: `.../20260908_commonpilev0.1_filtered_reprocess/runs.log`.
- Ordering-artifact analysis: `~/ws/training_artifacts/260821_common_diffusion/analysis/`.
