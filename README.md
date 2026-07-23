# Mandarin Tonal Processing

A pipeline for extracting and statistically analyzing syllable-level tonal features from Mandarin speech recordings. It goes from raw audio + orthographic transcripts to syllable-aligned F0 contours, DCT-parameterized tone shapes, and inferential statistics comparing the four lexical tones. The pipeline can directly run from colab with files uploaded in google drive.

The pipeline has three stages, each corresponding to one script (all exported from Google Colab notebooks):

| Stage | Script | Output |
|---|---|---|
| 1. Forced alignment | `copy_of_ctc_forcealign.py` | Syllable boundaries (JSON + Praat TextGrid) |
| 2. Acoustic feature extraction | `ctcf0.py` | Speaker-normalized F0 contours, DCT coefficients, HNR (CSV) |
| 3. Statistical analysis | `ctcstats.py` | Descriptive tables, ANOVA / Tukey HSD / mixed models, figures |

---

## Stage 1 — CTC forced alignment

`copy_of_ctc_forcealign.py`

Aligns Chinese-character transcripts to audio at the syllable level using a CTC forced aligner (ONNX runtime).

- Characters are converted to toneless pinyin with `pypinyin` (`Style.TONE3`, digits stripped) and expanded into a flat phoneme sequence. Tone information is deliberately removed at this stage so that alignment depends only on segmental content — tone is recovered later as an acoustic measurement, not assumed from the orthography.
- `get_alignments` / `get_spans` produce frame-level phoneme spans, which are then **regrouped back into syllables**: each character's start is the minimum non-blank frame across its phonemes, and its end is the maximum.
- Per-utterance output is a JSON array of `{text, start, end, score}` records, one per character.
- A converter writes Praat `.TextGrid` files with a single `syllable` interval tier. The gap-free variant (`json_to_textgrid_no_gap`) snaps each interval's `xmax` to the midpoint between the current syllable's end and the next one's start, so intervals are contiguous and Praat won't reject the tier.
- A QC pass reports total syllable count, mean/min/max duration, mean alignment confidence, and flags outliers (implausibly short or long intervals, low-confidence spans).

**Note:** the script contains two alignment passes. The first operates on a flat phoneme list and treats `stride` as seconds; the second — the one to use — maps phonemes back to syllables and correctly converts `stride` from milliseconds. The later pass overwrites the earlier JSON output.

## Stage 2 — F0 extraction and tonal parameterization

`ctcf0.py`

For each aligned syllable:

1. **F0 tracking** — `librosa.pyin` over the syllable interval (`fmin=63 Hz`, `fmax=400 Hz`, `frame_length=640`, `hop_length=128`, sr = 16 kHz).
2. **Unvoiced interpolation** — linear interpolation across unvoiced frames, with edge values held constant at the boundaries.
3. **Speaker normalization** — Hz converted to semitones relative to the speaker's median F0, computed per file across all voiced frames: `12 · log2(f0 / median)`.
4. **Time normalization** — the contour is resampled to 10 equidistant points, making contours comparable across syllables of different durations.
5. **DCT parameterization** — an orthonormal DCT over the 10-point contour; coefficients `C0`, `C1`, `C2` are retained as level, slope, and curvature terms respectively.
6. **Voicing quality** — `voiced_ratio` per syllable, and HNR estimated by normalized autocorrelation over the lag range corresponding to 50–400 Hz: `HNR(dB) = 10·log10(R / (1 − R))`.

Tone labels are then recovered from the character via `pypinyin` (`Style.TONE3`), with neutral tone coded as `0`.

**Cleaning filters** applied before analysis:

- `C0` present and within `[−10, 15]`; `C1` within `[−10, 10]`
- `voiced_ratio > 0.3`
- tone label present and non-neutral (neutral tone is excluded — it has no canonical contour target)

Outputs, written to the project root:

- `f0_all.csv` — every syllable, contour flattened to `f0_t1 … f0_t10`
- `f0_with_tone.csv` — with tone labels
- `f0_clean.csv` — filtered analysis set, plus the `HNR` column
- `f0_sample.png`, `f0_by_tone.png` — per-tone contour plots (individual traces + mean)

## Stage 3 — Statistics

`ctcstats.py`

- Descriptive table: n, mean and SD of `C0`/`C1`/`C2`, mean duration and HNR, by tone.
- **One-way ANOVA** across the four tones for `C0`, `C1`, `C2`, `duration_ms`, `HNR`.
- **Tukey HSD** post-hoc pairwise comparisons, reporting only significant pairs.
- **Linear mixed-effects models** (`statsmodels.mixedlm`), `feature ~ tone` with a random intercept for `file`, so that utterance-level variance (speaker, recording, register) is partialled out rather than treated as independent observations. T1 serves as the reference level.
- Figures: DCT coefficient boxplots by tone (`stats_boxplot.png`), and mean F0 contours by tone with ±1 SE bands (`stats_f0_mean.png`).

---

## Expected directory layout

The scripts read from Google Drive and assume this structure under `project_root`:

```
project_root/
├── processed_audio/     # 16 kHz mono WAV
├── transcripts/         # UTF-8 .txt, one per WAV, matching basename
├── alignments/          # written by stage 1
├── textgrids/           # written by stage 1
└── f0_results/          # written by stage 2
```

`project_root` is hard-coded at the top of each script and needs to be edited to match your own path.

## Dependencies

```
ctc-forced-aligner
onnxruntime
pypinyin
librosa
numpy
scipy
pandas
matplotlib
statsmodels
```

## Citation and reuse

If you use this pipeline, or code derived from it, in research, teaching, or any commercial context, please cite it and credit the author:

> [Andi Xiao] (2026). *Mandarin Tonal Processing: a CTC-alignment and F0 pipeline for syllable-level tone analysis.* GitHub. https://github.com/[andix43]/Mandarin_Tonal_Processing

BibTeX:

```bibtex
@software{mandarin_tonal_processing,
  author  = {[Andi Xiao]},
  title   = {Mandarin Tonal Processing: a CTC-alignment and F0 pipeline
             for syllable-level tone analysis},
  year    = {2026},
  url     = {https://github.com/[andix43]/Mandarin_Tonal_Processing}
}
```

For commercial use, or to adapt this pipeline as part of a larger published work, please get in touch first: [ax2132@nyu.edu].

## License

Released under [MIT] — attribution to the author is required in any redistribution or derivative work.

## Notes and limitations

- The scripts are Colab exports and run top-to-bottom as notebooks rather than as a CLI; they mount Google Drive on import.
- Alignment is CTC-based rather than HMM-based (no MFA), so boundaries are approximate at syllable edges. TextGrid output is provided specifically so alignments can be spot-checked or hand-corrected in Praat.
- Speaker normalization is computed per file. If a single file contains multiple speakers, the median will be pooled across them and the semitone values will be misleading.
- HNR here is an autocorrelation-based estimate, not Praat's implementation; values are usable for relative comparison across tones but should not be treated as interchangeable with Praat's `To Harmonicity`.
- Inline comments and console output are in Chinese.
