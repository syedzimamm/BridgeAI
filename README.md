# Bridge AI

An end-to-end, local bridge-document review and FHWA inventory research application. Next.js dashboard, FastAPI document processing, a real 20,000-bridge research sample, and a calibrated scikit-learn classifier. No paid API or cloud account is required.

![Bridge AI overview](docs/overview.png)

![Evidence-backed document review](docs/review.png)

**Engineering research prototype. Not a substitute for a licensed structural engineer.** FHWA Poor-condition ratings are not documented collapses or proof of design flaws. The model estimates condition association, never collapse probability. No result certifies safety or code compliance.

## Start with Docker

Requires Docker Engine / Docker Desktop with Compose. From this directory:

```sh
docker compose up --build -d
```

Open <http://localhost:3000>. The app starts with honest empty states. To populate real data and train the model (once per data volume):

```sh
docker compose exec backend python -m scripts.ingest_nbi
docker compose exec backend python -m ml.train
```

Or provision and start everything in one shell command:

```sh
docker compose up --build -d && docker compose exec backend python -m scripts.ingest_nbi && docker compose exec backend python -m ml.train
```

The archive, SQLite database, trained model and private uploads live in the `bridge-data` Docker volume. Ordinary restarts preserve them. Docker uses its own data volume, separate from a native installation's `data/` directory. The archive download is about 51 MB; training runs on CPU. The web interface remains available during setup. Do not delete the volume unless you intend to delete its uploads and model.

## Native setup

Python 3.12 and Node.js 22 are recommended. Enable pnpm 11.19.0 using Corepack, or install that version of pnpm separately.

```sh
python -m venv .venv
# Linux / macOS:
source .venv/bin/activate
# Windows PowerShell instead:
# .venv\Scripts\Activate.ps1
python -m pip install -r requirements-dev.txt
corepack enable
corepack prepare pnpm@11.19.0 --activate
cd frontend
pnpm install --frozen-lockfile
cd ..
python -m scripts.ingest_nbi
python -m ml.train
python -m scripts.create_sample_plan
```

If the official download is unavailable, download the exact official archive in a browser and run:

```sh
python -m scripts.ingest_nbi --archive /path/to/2025Del.zip --seed 42
```

Run the API and frontend in separate terminals:

```sh
python -m uvicorn backend.main:app --host 127.0.0.1 --port 8000
```

```sh
cd frontend
pnpm dev
```

For a production frontend build, use `pnpm build` then `pnpm start`. Open <http://localhost:3000>; interactive API schemas are at <http://localhost:8000/docs>.

Environment variables and defaults are described in `.env.example`. Python reads exported environment variables; it does not automatically load `.env`. Next.js accepts `frontend/.env.local`. Set `BACKEND_URL` **before the frontend build** if the API is elsewhere. `DATA_DIR` controls files/model storage; `DATABASE_URL` optionally controls the database separately. The SQLAlchemy schema uses portable types and constraints and supports `postgresql+psycopg://...`; PostgreSQL integration has not been exercised in this build.

## Try the complete workflow

1. Open **Analyze Plans**, then upload `docs/sample-plan.pdf`.
2. This deliberately inconsistent, two-page **synthetic software test drawing** produces an amber missing-unit finding, an amber unresolved-detail reference, and a red conflicting-dimension finding.
3. Select the red finding to navigate and zoom to page 2. Zoom and resize: normalized overlays stay attached to the annotation.
4. Confirm, dismiss, edit an explanation, or mark the finding for engineering review. Changes persist in SQLite.
5. Review extracted dimensions and units. Scoring requires eight confirmed, inventory-compatible inputs and the existing-bridge / 2025 context. Missing inputs produce **Insufficient information**. Proposed designs and out-of-training-range values produce **Unsupported input**.
6. Download an annotated PDF, a findings PDF or a CSV. Reopen the review from **Reports**.
7. Explore the **Bridge Database** and its map; open a record's source and inspection details. **AI Model** shows actual metrics, calibration, feature importance and provenance.

The sample drawing has no real engineering meaning and is never used for ML training. For a software-only scoring demonstration, the following numeric inputs fall inside training support: age 50, length 30 m, width 10 m, maximum span 15 m, ADT 100, main spans 2, material code 3, design code 02. These are illustrative inputs, not measurements for the sample drawing or an identified bridge; do not describe their result as an engineering assessment.

## Verified data and sampling

- [Official FHWA 2025 download page](https://www.fhwa.dot.gov/bridge/nbi/ascii2025.cfm)
- [Official all-state delimited archive](https://www.fhwa.dot.gov/bridge/nbi/2025Del.zip)
- [NBI field definitions](https://www.fhwa.dot.gov/bridge/nbi/format.cfm)
- [FHWA condition definitions](https://www.fhwa.dot.gov/bridge/britab.cfm)

Verified archive SHA-256:

```text
4c54c1fb2455ce427f66b2ec306d9aa1665df006c86291cebbe9d8ac8ec54f88
```

The parser uses the actual archive headers, comma delimiters and single-quote text qualifier. Numeric dimension columns already contain metres, so they are **not** divided by ten as fixed-width raw records might be. State and structure identifiers retain leading zeros. Inspection dates retain the original MMYY representation.

Condition is derived from applicable numeric deck, superstructure, substructure and culvert ratings. N means not applicable. The minimum is Poor at 0–4, Fair at 5–6 and Good at 7–9. Missing/invalid ratings are excluded and counted; computed classes must match the official `BRIDGE_CONDITION` column. Non-highway/under-route records are excluded. Malformed schemas, inconsistent official labels and insufficient qualifying records fail visibly.

This run read **624,193 source rows**. Two state/structure identities occur twice with conflicting physical records: `25:N100108PDDOTNBI` and `51:000000000001826`. All four occurrences are excluded; no arbitrary winner is selected. The remaining eligible unique population is **41,684 Poor, 309,727 Fair and 272,778 Good**. Sampling seed 42 selects:

| Condition | Imported unique records |
|---|---:|
| Poor | 10,000 |
| Fair | 5,000 |
| Good | 5,000 |
| Total | **20,000** |

The importer stores source URL, archive hash, inventory vintage, seed, actual counts, excluded IDs, archive member, source row and per-class inclusion probabilities. The sample is stratified by condition; it is not a national prevalence sample. Repeating the same verified import is idempotent; changing inventory/seed requires a fresh database. All mutations occur in one transaction.

The generated database is about 10 MB and is still excluded from Git to keep setup reproducible and prevent later accidental inclusion of uploaded plans. Regenerate it rather than distributing a mixed public/private database. The original government archive is excluded too. `tests/fixtures/official-ak-2025.csv` is a tiny attributed, real first-row excerpt for parser regression tests. Other fixtures are explicitly synthetic software tests.

## Model and measured limitations

The shared feature allowlist is age in 2025, structure length, deck width, maximum span, ADT, main spans, material code and design code. No condition rating, structural evaluation, inspection outcome or derived label enters the feature matrix. Numeric missing values are median-imputed and categories one-hot encoded. Inference deliberately requires all eight confirmed values rather than allowing a mostly imputed plan score.

The 20,000 unique IDs are partitioned into 12,000 training, 2,000 candidate-validation, 2,000 calibration and 4,000 final test records. No bridge is shared across partitions. Candidate choice uses validation PR-AUC; test data is not used to choose a model or threshold. This is a single-vintage cross-sectional experiment, **not** a historical forecast. Multi-year training is not implemented.

Condition-stratified sampling gives the model more Poor examples. Inverse inclusion weights restore the eligible population mix during fitting, sigmoid calibration and evaluation. The unweighted test confusion matrix is separately labelled. Threshold 0.5 is a generic evaluation threshold, not an engineering rule.

| Candidate | Validation PR-AUC |
|---|---:|
| Random Forest | 0.3107 |
| Histogram gradient boosting | 0.2965 |

Selected calibrated Random Forest, held-out results:

| Metric | Value |
|---|---:|
| Precision | 0.6098 |
| Recall | 0.1045 |
| F1 | 0.1784 |
| PR-AUC (average precision) | 0.3006 |
| Brier score | 0.05471 |

**Recall is low at the 0.5 threshold. Do not use this classifier to screen out bridges needing review.** Calibration also degrades in sparse high-score bins. Maintenance, climate, inspection practices and other omitted variables confound associations. Min/max and categorical support checks are conservative input guards, not validated out-of-distribution detection. Feature importance is global permutation importance, not a causal or per-document engineering explanation.

`data/model/` contains the serialized estimator and preprocessing pipeline, feature schema, model metadata, splits and held-out predictions. `python -m ml.evaluate` recomputes metrics from the saved predictions. Never load untrusted joblib files. Full measured metadata for this implementation is in `docs/evaluation.json`.

## Architecture and review rules

```text
frontend/        Next.js + TypeScript + Tailwind; PDF.js overlays, Recharts, Leaflet
backend/         FastAPI, Pydantic schemas, SQLAlchemy, PyMuPDF/OpenCV processing
ml/              Shared feature contract, training, calibration and evaluation
scripts/         Verified FHWA ingestion and synthetic sample drawing generator
tests/           Parser, database, model contract, document, API and export tests
frontend/e2e/    Real browser upload/annotation/zoom/export test
docs/            Rules, data/model notes, evaluation and sample plan
data/            Generated local artifacts, ignored by Git
```

The database has `bridges`, `bridge_inspections`, `training_samples`, `uploaded_plans`, `extracted_features`, `analysis_findings`, `model_versions`, `source_provenance` and `schema_migrations`. Migration 1 creates the initial schema. Future structural changes must add explicit numbered migrations; `create_all` alone does not migrate existing columns.

Uploads are validated by declared MIME and decoded content, size, image pixel count and PDF page count. Only sanitized canonical PDFs are retained under server-generated UUID filenames. Rotation is removed while preserving page appearance; extraction, PDF.js and export use the same page coordinate space. Embedded files, scripts and links are stripped. Scans get an unsupported-text finding; OCR is intentionally not claimed. OpenCV computes a raster-sharpness descriptor, which does not by itself trigger an engineering warning.

See [documented review rules](docs/review-rules.md). Red means an evidence-backed **document conflict**, not a proven structural hazard. Amber means document uncertainty. No thresholds for loads, clearances, support capacities or unusual dimensions are invented. There is no crack, corrosion, code-compliance or capacity detector. No external vision service receives documents.

## Tests

```sh
python -m pytest -q
python -m ruff check backend ml scripts tests
# After official ingestion and training, also exercise real-data/model assertions:
BRIDGE_REAL_DATA_TEST=1 python -m pytest -q
# PowerShell: $env:BRIDGE_REAL_DATA_TEST='1'; python -m pytest -q
cd frontend
pnpm lint
pnpm typecheck
pnpm test
pnpm build
pnpm exec playwright install chromium
# With API and frontend running:
pnpm exec playwright test
```

GitHub Actions runs Python lint/tests, frontend type checking/unit tests/build, and the browser workflow against the synthetic software-test document. CI does not repeatedly download the full FHWA archive or fabricate model metrics. The optional real-data checks run locally after provisioning.

## Deployment and privacy boundaries

This version is a **single-user local app**. There is no authentication, tenant isolation, rate limiting, retention scheduler or worker sandbox. Compose publishes to loopback only. Keep it local for trusted documents; add those controls and isolated document workers before any shared or public deployment. Large/complex documents can consume CPU during synchronous parsing. No mandatory paid services exist; optional OCR, authenticated engineering-rule modules and external vision integrations are future extensions.

Map tiles require internet access and are requested from OpenStreetMap. Other runtime processing stays local. The map only plots the current result page, not the entire database. A missing annotation is never evidence of safety. Uploaded reports include user-supplied document text and notes; CSV exports neutralize formula prefixes.

## GitHub

The repository includes Compose, CI, ignore rules and a lockfile. After authorizing your own destination:

```sh
git remote add origin YOUR_AUTHORIZED_GITHUB_URL
git push -u origin main
```

No remote is configured and nothing is pushed by this delivery. Keep `.env`, `data/`, model artifacts and private drawings out of commits. Application source is licensed **AGPL-3.0-only**, compatible with the AGPL option of PyMuPDF. Dependency licenses still apply; FHWA data attribution is separate from the application license. Review licensing before proprietary redistribution.
