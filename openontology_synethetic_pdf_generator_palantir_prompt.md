# Claude Build Prompt – “PDF Synth” Marketplace App

> **Objective**: Generate *all* code & config needed to ship a Palantir Marketplace App that uploads any PDF, extracts its field schema, generates **N** synthetic records (via LLM), renders typed **and** handwriting‑style filled PDFs, stores lineage objects, and presents download links in the UI.
>
> **Deliver every artefact in a single zip‑able folder structure as described below.**

---

## 1  Repository layout you must create

```
pdf‑synth/
├── compute_modules/
│   ├── extract_fields/
│   │   ├── main.py
│   │   ├── Dockerfile
│   │   └── requirements.txt
│   ├── generate_records/
│   │   ├── main.py
│   │   ├── Dockerfile          # reuse base stage ARGs
│   │   └── requirements.txt
│   └── render_pdf/
│       ├── main.py
│       ├── notify.py
│       ├── Dockerfile
│       └── requirements.txt
├── actions/
│   └── generate_synthetic_records.yaml
├── ontology/
│   └── ontology_mapping.md      # doc table for FieldSchema, SyntheticRecord, etc.
├── ui/
│   ├── README.md                # install + dev steps
│   └── src/pages/
│       ├── Upload.tsx
│       └── Results.tsx
└── pipeline/
    └── pipeline.yaml            # wires CM chain (optional if Action covers)
```

---

## 2  High‑level build contracts

1. **Compute Modules** must be **Python 3.11‑slim** containers.
2. Use **PyMuPDF** for field extraction & form filling; use **Pillow** + 3 Google handwriting TTFs for handwriting overlay.
3. LLM calls should default to **OpenAI chat‑completion** but leave room to switch via env vars.
4. Validate all generated JSON against schema with **jsonschema** before writing.
5. All writes/reads to Foundry objects must go through **OSDK ObjectSet API**. Provide helper util `foundry_io.py` so code stays clean.

---

## 3  Detailed instructions per component

### 3.1  `extract_fields/main.py`
* Input: `--pdf-path` (from raw_pdfs fileset), `--uploader`
* Output: Ontology `FieldSchema` row with keys:
  * `id` (uuid), `pdfFormId` (uuid link), `schemaJson` (dict matching AcroForm), `extractedAt`
* If the PDF has no AcroForm, locate text boxes heuristically (simple 2‑column rule) and output coordinates.

### 3.2  `generate_records/main.py`
* Args: `--field-schema-id`, `--num-samples`
* Build prompt:
  ```
  You are a synthetic data generator…
  SCHEMA: <json>
  ``
* Call OpenAI; parse; jsonschema‑validate; write **N** `SyntheticRecord` rows.

### 3.3  `render_pdf/main.py`
* Args: `--synthetic-record-id`
* For each record, create two files:
  * `typed` – fill form fields.
  * `handwriting` – overlay each value at same bbox with random handwriting font + slight jitter.
* Save to `rendered_pdfs` fileset; create `RenderedPdf` Ontology rows.
* At end, call `notify.py`.

### 3.4  `notify.py`
* Compose Foundry notification: *“PDF Synth job complete – <N> PDFs ready.”* Include direct file links.

### 3.5  `actions/generate_synthetic_records.yaml`

```yaml
name: GenerateSyntheticRecords
inputs:
  fieldSchemaId: FieldSchema
  numSamples: integer
steps:
  - id: gen
    module: generate_records
    args:
      fieldSchemaId: ${{inputs.fieldSchemaId}}
      numSamples: ${{inputs.numSamples}}
  - id: render
    module: render_pdf
    with:
      syntheticRecordIds: ${{steps.gen.output.recordIds}}
```

### 3.6  UI pages
* **Upload.tsx**
  * Drag‑drop, calls Fileset API, triggers CM `extract_fields` via REST.
  * After extraction, navigates to Schema view.
* **Results.tsx**
  * Table of `RenderedPdf` rows filtered by `pdfFormId`.
  * “Download all” button (bulk zip via Foundry link).

---

## 4  Secrets & Env Vars to support
| VAR | Default | Description |
|-----|---------|-------------|
| `OPENAI_API_KEY` | `<empty>` | LLM auth |
| `FONT_DIR` | `/fonts` | path to handwriting fonts |

Provide a `docker‑compose.override.yml` for local offline testing which mocks Foundry APIs.

---

## 5  Expected output when Claude finishes
1. Zip file (`pdf‑synth.zip`) with folder tree above.
2. A markdown summary of how to register each Compute Module & Action in Foundry.
3. Quick‑start commands:
   ```bash
   foundry osdk cm register compute_modules/*
   foundry osdk action register actions/generate_synthetic_records.yaml
   yarn && yarn dev   # inside ui/
   ```

---

### ❗ Edge‑cases Claude must address
- PDFs with mixed AcroForm + scanned pages → ignore bitmap pages.
- Massive PDFs > 25 MB → skip & log error.
- LLM hallucination → retry up to 2× with stricter system prompt.

---

## 6  Non‑code tasks to flag as **[MANUAL]**
1. Create Ontology types and properties exactly as in `ontology_mapping.md`.
2. Create two Filesets: `raw_pdfs`, `rendered_pdfs`, grant upload rights to the Marketplace App service principal.
3. Inject `OPENAI_API_KEY` secret into each CM runtime.

---

> When answered, Claude should emit (a) repo tree, (b) full source code, (c) deployment instructions, (d) edge‑case notes.

