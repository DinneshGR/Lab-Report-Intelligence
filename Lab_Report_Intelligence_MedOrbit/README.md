# Lab Report Intelligence — MedOrbit

A Colab-runnable pipeline that reads a patient's last 3 lab report PDFs (digital or scanned), extracts and normalizes test results, builds a predictive multi-visit summary, and emits a JSON payload shaped for MedOrbit's pre-consult-brief and consult-summary agents.

**File to run:** `Lab_Report_Intelligence_MedOrbit.ipynb` → upload to Google Colab, Runtime → Run all.

---

## What it does

1. **Ingests PDFs** — digital (text-based) and scanned (image-based), with automatic per-page OCR fallback.
2. **Extracts test data** — test name, value, unit, reference range, high/low flag — using a tolerant regex library tuned to the line/table grammars common across Indian lab report formats (SRL, Metropolis, Dr Lal PathLabs, Thyrocare, Apollo, hospital in-house, etc.), plus an optional LLM fallback for formats regex misses.
3. **Normalizes** — folds lab-specific test naming into one canonical schema so the same analyte lines up correctly across visits, even if labs spelled or formatted it differently.
4. **Builds a 3-visit timeline** and computes:
   - direction and % change per test
   - **deteriorating-trend flags** — tests moving toward/past an abnormal boundary across visits even while still "Normal" today
   - **cross-panel correlations** — small, explainable rule set (e.g. rising LDL + rising HbA1c → cardiometabolic cluster; falling eGFR + rising creatinine → renal trend)
5. **Generates a doctor-ready summary** — headline, abnormal-now, trending-to-watch, cross-panel patterns. Works with or without an LLM API key (deterministic template fallback).
6. **Generates consultation suggestions** — discussion points, follow-up tests to consider, what to monitor next visit — explicitly framed as decision support, never diagnosis.
7. **Emits a MedOrbit-ready JSON payload** and a stub `push_to_medorbit()` function to wire it into the existing pre-consult-brief and consult-summary agents.

---

## How it was built — design decisions

- **PDF ingestion is two-path by design** (`pdfplumber` for digital, `pdf2image` + `pytesseract` OCR for scanned), decided **per page**, not per file — a single report that mixes a digital first page with a stamped/scanned second page is handled correctly.
- **Extraction is rule-based first, LLM second.** Indian lab reports follow a small number of recurring line/table grammars, so regex + table parsing handles the bulk of cases cheaply and deterministically. The LLM path (Google **Gemini**, free tier, via the `google-genai` SDK) is only invoked as a *fallback* when a report yields suspiciously few rows — this keeps cost, latency, and hallucination risk low while still catching unusual formats. The notebook runs fully without any API key (`USE_LLM = False` by default).
- **Canonicalization is an explicit, editable dictionary** (`CANONICAL_TESTS`), not a fuzzy black box — so it's obvious which aliases map to which analyte, and easy to extend as you encounter new report formats.
- **Trend logic is transparent and rule-based**, not a trained model — every flag (deteriorating trend, cross-panel correlation) is traceable to a specific, readable condition. This matters clinically: a doctor can see *why* something was surfaced.
- **Two-tier generation for summary and suggestions**: an LLM phrases the final text when available (constrained to only the structured data already computed — it can't introduce numbers that weren't extracted), and a deterministic template produces the same structure without a key. Same downstream shape either way.
- **Integration is a stable JSON contract**, not a tight coupling to MedOrbit internals — `build_medorbit_payload()` defines the schema; `push_to_medorbit()` is a thin, swappable HTTP call (defaults to `dry_run=True` so nothing fires from Colab by accident).

---

## Notebook section guide

| Section | Purpose |
|---|---|
| 0 | Installs (`pdfplumber`, `pdf2image`, `pytesseract`, `google-genai`) + config (`USE_LLM`, Gemini API key) |
| 1 | PDF ingestion: digital text extraction with OCR fallback per page |
| 2 | Rule-based line/table parsers for common Indian lab report grammars |
| 3 | Optional LLM extraction fallback for formats the regex misses |
| 4 | Canonical test-name dictionary + normalization into one schema |
| 5 | Builds the 3-visit patient timeline DataFrame (single source of truth for everything downstream) |
| 6 | Trend computation, deterioration flags, cross-panel correlation rules |
| 7 | Doctor-ready summary generation (LLM-assisted or rule-based) |
| 8 | AI-assisted consultation suggestions (discussion points / follow-up tests / monitoring) |
| 9 | MedOrbit integration: JSON payload builder + `push_to_medorbit()` stub |
| 10 | `run_pipeline()` — the single end-to-end entry point |
| 11 | Demo run on synthetic data (no PDFs needed, runs immediately in Colab) |
| 12 | How to point the pipeline at real PDFs + production notes/limitations |

---

## Running it

**Quick demo (no files needed):** Run all cells — Section 11 runs the full trend/summary/suggestion/payload logic against a built-in synthetic 3-visit dataset.

**With real reports:**
```python
reports = [
    LabReportInput(pdf_path="/content/report_2025_03.pdf", report_date="2025-03-10"),
    LabReportInput(pdf_path="/content/report_2025_06.pdf", report_date="2025-06-12"),
    LabReportInput(pdf_path="/content/report_2025_09.pdf", report_date="2025-09-15"),
]
result = run_pipeline(patient_id="PATIENT-1234", reports=reports, dry_run=True)
print(result["summary"])
```
Upload the 3 PDFs via Colab's Files sidebar first.

**To enable LLM-assisted phrasing and extraction fallback:** set `USE_LLM = True` and paste a **free** Gemini API key (get one at [Google AI Studio](https://aistudio.google.com/app/apikey)) in Section 0.

---

## Limitations to know before production use

- Regex coverage is a starting library, not exhaustive — extend `LINE_PATTERNS` (Section 2) and `CANONICAL_TESTS` (Section 4) as real report formats are seen.
- OCR quality depends on scan resolution/skew; bump `dpi` in `extract_pdf_text()` or add a deskew step if scans are poor.
- The consultation-suggestions output is decision support only, never a diagnosis — the disclaimer field in the payload should stay visible wherever this is rendered.
- No patient-data persistence is done in this notebook — production wiring should handle PHI/auth appropriately and clean up temp files from Colab's ephemeral disk.
- `push_to_medorbit()` defaults to `dry_run=True`; point it at MedOrbit's real authenticated endpoints (or swap for a direct in-process call) before using it live.
