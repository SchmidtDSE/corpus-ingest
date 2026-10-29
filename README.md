# text-ingest

Builds AI-ready, cloud-native corpora from document collections: scientific papers, reports
and grey literature, or legal and policy documents. A collection on S3 becomes parquet tables
of documents, passages, references and places, with bibliographic metadata, provenance and a
Croissant description. Nothing in it is specific to one application; each corpus's recipe in
[`text-workflows`](https://github.com/SchmidtDSE/text-workflows) sets its sources, document
kind, languages and gazetteers. Processing runs as resumable jobs on Kubernetes,
following the patterns of [`cng-datasets`](https://github.com/boettiger-lab/datasets), which
does the same for spatial data.

**Status:** design. Nothing is implemented yet. The design and the pilot that informs it are in
[`text-workflows/PLAN.md`](https://github.com/SchmidtDSE/text-workflows/blob/main/PLAN.md).
Recipes for each corpus live in
[`text-workflows`](https://github.com/SchmidtDSE/text-workflows).

## Stages

| Stage | What it does | Tools | Output |
|---|---|---|---|
| Inventory | Lists and hashes every object, recording its original path and folder | `rclone`, sha256 | `manifest.parquet` |
| Deduplication | Collapses identical files to one document, keeping every path and folder; flags near-duplicates | sha256, then text similarity | `doc_id` = content hash |
| Text layer check | Decides per page whether OCR is needed | PyMuPDF | page flags |
| OCR | Adds a text layer to scanned pages, next to the original | OCRmyPDF (Tesseract) | `pdf-ocr/<doc_id>.pdf` |
| Structure | Title, authors, sections and passages with pages, and parsed references, with a parser chosen by document kind | GROBID for scholarly papers, a layout parser for reports, the source markup for legal XML and HTML, PyMuPDF as fallback | `structure/`, `passages.parquet`, `references.parquet` |
| Metadata | Bibliographic record, language, licence and access status; jurisdiction, version and effective dates for legal text | Crossref, OpenAlex and Unpaywall for scholarly work; the issuing body or legislative source otherwise | `documents.parquet` |
| Places | Place names and coordinates in passages and documents, matched to the recipe's gazetteers, as H3 cells with a precision class | gazetteers chosen per corpus (for example GeoNames, Marine Regions, GEBCO feature names, WDPA, PAD-US), coordinate patterns, an extraction model | `places.parquet` |
| Search index | Optional full-text and embedding indexes | DuckDB FTS, an open-weights embedding model | index files |
| Publish | Validates outputs and writes the corpus description | `mlcroissant`, corpus checks | `croissant.json`, README, `AGENTS.md`, `stac-collection.json` |

## Design rules

- **Originals are never modified.** Everything derived goes under `derived/<snapshot>/`,
  keyed by `doc_id`.
- **Snapshots are immutable.** A passage id is a hash of the document, its position and its
  text, so coded tables that cite it stay valid when documents are added.
- **Every stage is resumable.** Outputs are keyed by content hash, so a rerun skips finished
  work and new documents are processed incrementally.
- **Provenance travels in the file.** Each parquet records the tool version, stage, OCR and
  GROBID versions and build time in its key-value metadata.
- **Checks run before publishing:** text coverage after OCR, passages per page, unresolved
  identifiers, duplicates, and places that fall where the corpus says they cannot, such as on
  land for a marine corpus.
- **Passages are the unit.** A paragraph of a paper or the smallest provision of a statute,
  with its section path kept for citation.
- **Location has levels.** Places found in a passage, the document's study area that a passage
  without a place inherits (flagged as inherited), and the H3 resolution set by the recipe.
- **Metadata follows standards made for document collections.** Croissant describes tables and
  files, CSL-JSON field names hold each document's bibliographic record, and GROBID's TEI is
  kept, or the source markup for legal text. A thin STAC collection points to the Croissant file so catalog tools can find the
  corpus. Derived spatial products, such as places on the H3 grid, are ordinary STAC
  collections.
- **Access is optional.** Corpora may sit in public or private buckets. Credentials come from
  the environment and are scoped to the buckets a job reads.
- **GROBID runs as a cluster service,** deployed from `k8s/grobid/` in this repository.

## Licence

BSD-3-Clause. See [DISCLOSURE.md](DISCLOSURE.md) for the use of software tools.
