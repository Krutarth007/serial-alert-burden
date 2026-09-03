# Serial Alert Burden

A pre-deployment evaluation method for multi-label clinical decision support systems that operate over repeated examinations, with a worked demonstration on chest radiographs.

Discrimination metrics are computed over independent observations. Deployed multi-label imaging CDS violates that assumption twice over: it evaluates every examination against many findings at once, and it sees the same patient repeatedly. Two consequences are invisible in an area under the curve — how many alerts land on a single examination, and how many of them restate something already in the patient's record.

This repository contains the method, the analysis code, and the outputs behind the manuscript.

---

## The four components

**1. Alert multiplicity.** The distribution of concurrent alerts per examination. Necessary because "alerts per 100 examinations" exceeds 100 whenever a system evaluates several findings, and is routinely misread as a percentage of examinations.

**2. Decomposition.** Every fired alert is assigned exactly one mutually exclusive category — false, redundant, or first-observed-positive — reported against all fired alerts and, separately, conditional on the alert being correct.

**3. Definition sensitivity.** Redundancy under four predicates in two families. *Label history* uses prior reference labels and is retrospective. *Prediction history* uses the model's own prior firing and is what a deployed rule could condition on.

**4. Within-episode comparison.** Detection at a finding's first appearance versus its later appearances within the same run of consecutive positives, holding patient and finding constant. This controls the severity confounding that makes naive serial comparisons misleading.

---

## Headline results from the demonstration

Held-out cohort: 22,799 follow-up radiographs, 1,424 patients, 296,387 examination-finding opportunities, 13 findings. Patient-clustered 95% confidence intervals from 1,000 bootstrap replicates.

| Quantity | Estimate (95% CI) |
|---|---|
| Mean alerts per radiograph (of 13 findings) | 6.97 (6.83–7.09) |
| Radiographs receiving at least one alert | 98.2% (97.8–98.5) |
| Radiographs receiving four or more | 85.1% |
| False alerts, share of fired | 87.5% (87.2–87.8) |
| Redundant, share of **correct** alerts | 82.6% (81.6–83.7) |
| Redundant share under immediate-prior definition | 42.9% |
| Detection difference, unadjusted | 11.95 pp (10.40–13.65) |
| Detection difference, within episode | 2.30 pp (1.05–3.51) |

Three points transfer beyond this instrument:

- Multiplicity must be reported separately from alert rate.
- Redundancy estimates differ roughly twofold between two defensible definitions, so any redundancy claim must state its predicate.
- The unadjusted detection difference is about five times the within-episode difference. Serial comparisons that do not hold patient and finding constant risk substantially overstating differential detection.

---

## Repository contents

```
notebooks/
  part1_data_wiring.ipynb            Metadata linkage, label parsing, ordinal
                                     sequences, transition table, audits
  part2_modeling_simulation.ipynb    Patient-disjoint splits, frozen embeddings,
                                     linear head, threshold locking, alert stream
  part3_robustness_uncertainty.ipynb Bootstrap intervals, threshold and cohort
                                     sensitivity, within-episode control,
                                     adjudicated-label reference
outputs/                             All CSV tables and figures produced by the
                                     notebooks, organised by part
requirements.txt
LICENSE
README.md
```

Each notebook ends with a programmatic PASS/WARN/FAIL audit checklist and writes a run manifest containing library versions, random seeds, and SHA-1 fingerprints of its own outputs. Part 2 verifies it received the artifact Part 1's manifest describes; Part 3 does the same for Part 2 and refuses to proceed if the upstream audit contains a failure.

---

## Data

Neither dataset is redistributed here. Both are public.

**ChestX-ray14** — 112,120 frontal radiographs from 30,805 patients with 14 findings extracted from radiology reports by natural language processing. Available from the NIH Clinical Center. These are weak labels, not adjudicated reads.

**Radiologist-adjudicated labels** — provided as supplementary material to Majkowska et al., *Radiology* 2020;294(2):421–431, covering 1,962 ChestX-ray14 test-set images. Used here only as a within-dataset label reference, never as external validation.

**Image representation** — a DenseNet-121 pretrained on MIMIC-CXR, obtained through TorchXRayVision using the weight tag `densenet121-res224-mimic_ch`. This is deliberate: weights trained on ChestX-ray14 would have seen the evaluation patients. Preprocessing is single-channel, normalised to [-1024, 1024], centre-cropped and resized to 224 px, recorded in the cache metadata as `cxr-v2-gray1ch-224-xrvnorm`.

Embeddings are **not** committed. The array for the full corpus is roughly 900 MB. Part 2 regenerates it, chunked and resumable, in about two hours on a laptop CPU.

---

## Reproducing the analysis

```bash
git clone https://github.com/<user>/serial-alert-burden
cd serial-alert-burden
pip install -r requirements.txt
```

Download ChestX-ray14 and set `DATA_ROOT` in the Part 1 configuration cell. Run the notebooks in order. Part 2 requires `ALLOW_MODEL_DOWNLOAD = True` to fetch the pretrained weights and `ALLOW_FULL_CPU_EMBEDDING_EXTRACTION = True` before it will extract beyond 1,000 images; both refuse silently-wrong behaviour rather than proceeding.

Every notebook has a `FAST_MODE` flag that runs a subsampled smoke test into a separate output directory, stamps every table `smoke_test=True`, and cannot overwrite full-analysis results. Run it first.

Approximate timings on a laptop CPU: Part 1 a few minutes, Part 2 about two hours dominated by embedding extraction, Part 3 under five minutes.

---

## Design constraints built into the code

These are not conventions; the notebooks enforce them.

**Patient-disjoint splits**, asserted rather than assumed. The official NIH split lists are checked for patient overlap before being adopted.

**Test-side history is recomputed within the held-out split**, so no training or validation examination can enter a history window even if a future split were not patient-disjoint.

**Thresholds are locked to disk before any test quantity is computed.** No test metric informs label selection, threshold choice, or cohort definition.

**The model receives embeddings only.** No sequence position, follow-up index, prior label, or alert-history variable can reach the feature matrix. History is applied strictly after predictions are frozen.

**Index encounters are excluded** from redundancy analysis, since every positive finding is first-observed at a patient's first available examination by construction.

**Random-weight embedding caches are rejected at load time.** A cache whose metadata reports no pretrained weights raises rather than warns.

---

## What this is not

A retrospective shadow-mode simulation. No alert was delivered, and no clinician or patient was involved.

`Follow-up #` is an ordinal within-patient encounter index. It carries no calendar time, so nothing here is expressed per day, week, or month, and no onset, duration, or progression quantity is estimable.

A **redundant** alert means the label was positive earlier in the available sequence. It does not establish that a repeat alert would be clinically redundant; a finding present on a prior radiograph may still warrant re-alerting.

A **first-observed-positive** alert means the first positive weak label in the available sequence. It does not establish incident disease, a new diagnosis, or clinical actionability. Imaging outside this corpus is invisible, so it is an upper bound on first presentation.

The **label-history oracle** policy requires reference labels unavailable at deployment. It is a reference point, not an achievable bound: a prediction-history policy can suppress alerts the oracle retains, so the two trace different trade-off curves.

No suppression policy examined here is safe, beneficial, or ready for deployment. That requires prospective evaluation involving clinicians.

The adjudicated subset is a within-dataset label reference. It is a relabelling of the same images and supports no claim about external validation or transportability.

---

## Known limitations

Labels are report-derived and imperfect; instability between adjacent examinations is not separable from genuine clinical change. The within-episode design holds patient and finding constant but analyses a restricted population, so it bounds the influence of confounding rather than partitioning it exactly, and it does not remove genuine change in a finding across an episode. The validation split contained shorter patient sequences than the test split, so thresholds transferred imperfectly and the operating point should be read as one point on a curve. Calibration was not evaluated. Model discrimination is modest by design; the argument is that a system passing a conventional performance review can still produce this alert stream.

---

## Citation

Manuscript under review. Cite this repository via the archived release DOI until publication.

## License

MIT. See `LICENSE`.

Dataset licences are separate and remain with their providers.
