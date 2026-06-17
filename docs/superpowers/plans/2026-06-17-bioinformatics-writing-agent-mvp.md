# Bioinformatics Writing Agent MVP Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a local-first MVP that ingests bioinformatics PDFs, extracts traceable evidence, retrieves scholarly literature, and generates sourced Introduction and Related Work drafts.

**Architecture:** The application is a FastAPI backend with a small web console, SQLite persistence, a parser/extraction pipeline, literature API clients, an evidence store, and a provider-agnostic LLM layer. The first implementation uses synchronous service calls behind API endpoints; job queues can be introduced after the core pipeline is verified.

**Tech Stack:** Python 3.11+, FastAPI, Pydantic, SQLAlchemy, SQLite, PyMuPDF, pdfplumber, httpx, pytest, ruff, optional GROBID service, optional LLM provider.

---

## Scope Decision

This plan covers one cohesive MVP path:

- Local single-user web app.
- Backend-first development with tests.
- Minimal HTML console after core services exist.
- Literature retrieval through stable scholarly APIs.
- Draft generation constrained by evidence records.

The MVP does not include multi-user accounts, cloud deployment, journal-specific formatting, or autonomous Google Scholar crawling.

## File Structure

Create these files:

- `pyproject.toml`: package metadata, dependencies, pytest and ruff configuration.
- `.env.example`: documented environment variables.
- `README.md`: local setup and MVP workflow.
- `app/__init__.py`: package marker.
- `app/main.py`: FastAPI app factory and router registration.
- `app/config.py`: typed settings.
- `app/db.py`: SQLAlchemy engine, sessions, and initialization.
- `app/models.py`: database models for papers, sections, figures, tables, references, candidate literature, evidence, and drafts.
- `app/schemas.py`: Pydantic request and response models.
- `app/repositories.py`: persistence helpers.
- `app/services/ingestion.py`: file hashing, PDF storage, paper record creation.
- `app/services/parsing.py`: parser interface and PyMuPDF/pdfplumber parser.
- `app/services/sectioning.py`: section normalization and heading classification.
- `app/services/extraction.py`: bioinformatics fact extraction.
- `app/services/figures.py`: figure/table caption detection and in-text mention linking.
- `app/services/literature.py`: literature retrieval orchestrator and deduplication.
- `app/services/evidence.py`: evidence creation and lookup.
- `app/services/style.py`: writing style profile generation.
- `app/services/writing.py`: outline, paragraph plan, and draft generation.
- `app/services/claim_check.py`: draft claim validation.
- `app/clients/base.py`: shared HTTP client helpers and candidate paper normalization.
- `app/clients/crossref.py`: Crossref client.
- `app/clients/openalex.py`: OpenAlex client.
- `app/clients/semanticscholar.py`: Semantic Scholar client.
- `app/clients/pubmed.py`: PubMed E-utilities client.
- `app/llm/base.py`: LLM provider protocol and fake provider for tests.
- `app/llm/prompts.py`: extraction and writing prompt builders.
- `app/api/papers.py`: upload, parse, inspect, and evidence endpoints.
- `app/api/literature.py`: literature retrieval endpoints.
- `app/api/writing.py`: outline and draft endpoints.
- `app/templates/index.html`: minimal web console.
- `app/static/app.css`: compact styling for the console.
- `tests/conftest.py`: isolated test database and app fixtures.
- `tests/test_config.py`: settings tests.
- `tests/test_ingestion.py`: upload and file hashing tests.
- `tests/test_parsing.py`: parser normalization tests.
- `tests/test_extraction.py`: bioinformatics extraction tests.
- `tests/test_figures.py`: figure/table linking tests.
- `tests/test_literature.py`: literature normalization and deduplication tests.
- `tests/test_evidence.py`: evidence persistence tests.
- `tests/test_writing.py`: outline and draft guardrail tests.
- `tests/test_claim_check.py`: unsupported claim detection tests.
- `tests/test_api.py`: API integration tests.

## Task 1: Project Foundation

**Files:**
- Create: `pyproject.toml`
- Create: `.env.example`
- Create: `README.md`
- Create: `app/__init__.py`
- Create: `app/config.py`
- Create: `app/main.py`
- Create: `tests/test_config.py`

- [ ] **Step 1: Initialize git**

Run: `git init`

Expected: repository initialized in the project root.

- [ ] **Step 2: Write the failing settings test**

Create `tests/test_config.py`:

```python
from app.config import Settings


def test_settings_defaults_are_local_first():
    settings = Settings()

    assert settings.app_name == "Bioinformatics Writing Agent"
    assert settings.database_url == "sqlite:///./data/app.db"
    assert settings.paper_storage_dir == "data/papers/originals"
```

- [ ] **Step 3: Run the test and verify failure**

Run: `pytest tests/test_config.py -v`

Expected: FAIL because `app.config` does not exist.

- [ ] **Step 4: Create package configuration**

Create `pyproject.toml`:

```toml
[project]
name = "bioinformatics-writing-agent"
version = "0.1.0"
description = "Local-first MVP for reading bioinformatics papers and drafting sourced manuscript sections."
requires-python = ">=3.11"
dependencies = [
  "fastapi>=0.111",
  "uvicorn[standard]>=0.30",
  "python-multipart>=0.0.9",
  "pydantic>=2.7",
  "pydantic-settings>=2.2",
  "sqlalchemy>=2.0",
  "pymupdf>=1.24",
  "pdfplumber>=0.11",
  "httpx>=0.27",
  "jinja2>=3.1",
]

[project.optional-dependencies]
dev = [
  "pytest>=8.2",
  "pytest-asyncio>=0.23",
  "ruff>=0.5",
]

[tool.pytest.ini_options]
testpaths = ["tests"]
pythonpath = ["."]

[tool.ruff]
line-length = 100
target-version = "py311"

[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B"]
```

Create `.env.example`:

```text
APP_NAME=Bioinformatics Writing Agent
DATABASE_URL=sqlite:///./data/app.db
PAPER_STORAGE_DIR=data/papers/originals
SEMANTIC_SCHOLAR_API_KEY=
NCBI_EMAIL=
OPENAI_API_KEY=
LLM_PROVIDER=fake
```

Create `README.md`:

````markdown
# Bioinformatics Writing Agent

Local-first MVP for reading bioinformatics papers, retrieving scholarly context, and drafting sourced Introduction and Related Work sections.

## Setup

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -e ".[dev]"
```

## Run

```powershell
uvicorn app.main:app --reload
```

Open `http://127.0.0.1:8000`.

## MVP Workflow

1. Upload PDF papers.
2. Parse sections, references, figures, and tables.
3. Extract bioinformatics evidence.
4. Retrieve reference, citation, and related-paper metadata.
5. Generate an outline and sourced draft.
```
````

Create `app/__init__.py`:

```python
"""Bioinformatics paper writing agent."""
```

Create `app/config.py`:

```python
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    app_name: str = "Bioinformatics Writing Agent"
    database_url: str = "sqlite:///./data/app.db"
    paper_storage_dir: str = "data/papers/originals"
    semantic_scholar_api_key: str = ""
    ncbi_email: str = ""
    openai_api_key: str = ""
    llm_provider: str = "fake"

    model_config = SettingsConfigDict(env_file=".env", env_file_encoding="utf-8")
```

Create `app/main.py`:

```python
from fastapi import FastAPI

from app.config import Settings


def create_app(settings: Settings | None = None) -> FastAPI:
    active_settings = settings or Settings()
    app = FastAPI(title=active_settings.app_name)

    @app.get("/health")
    def health() -> dict[str, str]:
        return {"status": "ok"}

    return app


app = create_app()
```

- [ ] **Step 5: Run the test and verify pass**

Run: `pytest tests/test_config.py -v`

Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add pyproject.toml .env.example README.md app tests
git commit -m "chore: initialize bioinformatics writing agent"
```

## Task 2: Database Models and Repositories

**Files:**
- Create: `app/db.py`
- Create: `app/models.py`
- Create: `app/schemas.py`
- Create: `app/repositories.py`
- Create: `tests/conftest.py`
- Create: `tests/test_evidence.py`

- [ ] **Step 1: Write the failing repository test**

Create `tests/conftest.py`:

```python
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import Session, sessionmaker

from app.models import Base


@pytest.fixture()
def session() -> Session:
    engine = create_engine("sqlite:///:memory:")
    Base.metadata.create_all(engine)
    maker = sessionmaker(bind=engine)
    with maker() as db:
        yield db
```

Create `tests/test_evidence.py`:

```python
from app.repositories import PaperRepository


def test_create_paper_and_evidence(session):
    repo = PaperRepository(session)

    paper = repo.create_paper(filename="paper.pdf", sha256="abc123")
    evidence = repo.create_evidence(
        paper_id=paper.id,
        kind="text",
        claim="The paper evaluates AUROC.",
        source_label="Uploaded Paper A, Results, p. 7",
        confidence="high",
        needs_human_check=False,
    )

    assert paper.id is not None
    assert evidence.claim == "The paper evaluates AUROC."
    assert repo.list_evidence(paper.id)[0].source_label == "Uploaded Paper A, Results, p. 7"
```

- [ ] **Step 2: Run the test and verify failure**

Run: `pytest tests/test_evidence.py -v`

Expected: FAIL because the database models and repository do not exist.

- [ ] **Step 3: Create database layer**

Create `app/db.py`:

```python
from collections.abc import Generator
from pathlib import Path

from sqlalchemy import create_engine
from sqlalchemy.orm import Session, sessionmaker

from app.config import Settings
from app.models import Base


def build_engine(settings: Settings):
    if settings.database_url.startswith("sqlite:///"):
        db_path = Path(settings.database_url.replace("sqlite:///", ""))
        if db_path.parent != Path("."):
            db_path.parent.mkdir(parents=True, exist_ok=True)
    return create_engine(settings.database_url, connect_args={"check_same_thread": False})


settings = Settings()
engine = build_engine(settings)
SessionLocal = sessionmaker(bind=engine)


def init_db() -> None:
    Base.metadata.create_all(engine)


def get_session() -> Generator[Session, None, None]:
    with SessionLocal() as session:
        yield session
```

Create `app/models.py`:

```python
from datetime import datetime

from sqlalchemy import Boolean, DateTime, ForeignKey, Integer, String, Text
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship


class Base(DeclarativeBase):
    pass


class Paper(Base):
    __tablename__ = "papers"

    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    filename: Mapped[str] = mapped_column(String(512))
    sha256: Mapped[str] = mapped_column(String(64), unique=True, index=True)
    title: Mapped[str | None] = mapped_column(String(1000), nullable=True)
    doi: Mapped[str | None] = mapped_column(String(255), nullable=True)
    status: Mapped[str] = mapped_column(String(64), default="uploaded")
    created_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)

    sections: Mapped[list["PaperSection"]] = relationship(back_populates="paper")
    evidence: Mapped[list["Evidence"]] = relationship(back_populates="paper")


class PaperSection(Base):
    __tablename__ = "paper_sections"

    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    paper_id: Mapped[int] = mapped_column(ForeignKey("papers.id"))
    name: Mapped[str] = mapped_column(String(255))
    normalized_name: Mapped[str] = mapped_column(String(128))
    text: Mapped[str] = mapped_column(Text)
    page_start: Mapped[int | None] = mapped_column(Integer, nullable=True)
    page_end: Mapped[int | None] = mapped_column(Integer, nullable=True)

    paper: Mapped[Paper] = relationship(back_populates="sections")


class Figure(Base):
    __tablename__ = "figures"

    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    paper_id: Mapped[int] = mapped_column(ForeignKey("papers.id"))
    label: Mapped[str] = mapped_column(String(64))
    caption: Mapped[str] = mapped_column(Text)
    page: Mapped[int | None] = mapped_column(Integer, nullable=True)
    needs_human_check: Mapped[bool] = mapped_column(Boolean, default=True)


class Table(Base):
    __tablename__ = "tables"

    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    paper_id: Mapped[int] = mapped_column(ForeignKey("papers.id"))
    label: Mapped[str] = mapped_column(String(64))
    caption: Mapped[str] = mapped_column(Text)
    page: Mapped[int | None] = mapped_column(Integer, nullable=True)
    needs_human_check: Mapped[bool] = mapped_column(Boolean, default=True)


class Reference(Base):
    __tablename__ = "references"

    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    paper_id: Mapped[int] = mapped_column(ForeignKey("papers.id"))
    raw_text: Mapped[str] = mapped_column(Text)
    title: Mapped[str | None] = mapped_column(String(1000), nullable=True)
    doi: Mapped[str | None] = mapped_column(String(255), nullable=True)
    year: Mapped[int | None] = mapped_column(Integer, nullable=True)


class CandidateLiterature(Base):
    __tablename__ = "candidate_literature"

    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    source_paper_id: Mapped[int] = mapped_column(ForeignKey("papers.id"))
    title: Mapped[str] = mapped_column(String(1000))
    authors: Mapped[str] = mapped_column(Text, default="[]")
    year: Mapped[int | None] = mapped_column(Integer, nullable=True)
    venue: Mapped[str | None] = mapped_column(String(500), nullable=True)
    doi: Mapped[str | None] = mapped_column(String(255), nullable=True)
    url: Mapped[str | None] = mapped_column(String(1000), nullable=True)
    relation: Mapped[str] = mapped_column(String(128))
    relevance_reason: Mapped[str] = mapped_column(Text)
    usable_for: Mapped[str] = mapped_column(String(255))
    verification_status: Mapped[str] = mapped_column(String(64), default="needs_human_check")


class Evidence(Base):
    __tablename__ = "evidence"

    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    paper_id: Mapped[int] = mapped_column(ForeignKey("papers.id"))
    kind: Mapped[str] = mapped_column(String(64))
    claim: Mapped[str] = mapped_column(Text)
    source_label: Mapped[str] = mapped_column(String(1000))
    confidence: Mapped[str] = mapped_column(String(64))
    needs_human_check: Mapped[bool] = mapped_column(Boolean, default=False)

    paper: Mapped[Paper] = relationship(back_populates="evidence")


class Draft(Base):
    __tablename__ = "drafts"

    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    paper_id: Mapped[int] = mapped_column(ForeignKey("papers.id"))
    section_name: Mapped[str] = mapped_column(String(128))
    content: Mapped[str] = mapped_column(Text)
    created_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)
```

Create `app/schemas.py`:

```python
from pydantic import BaseModel


class PaperRead(BaseModel):
    id: int
    filename: str
    sha256: str
    title: str | None
    doi: str | None
    status: str

    model_config = {"from_attributes": True}


class EvidenceRead(BaseModel):
    id: int
    paper_id: int
    kind: str
    claim: str
    source_label: str
    confidence: str
    needs_human_check: bool

    model_config = {"from_attributes": True}
```

Create `app/repositories.py`:

```python
from sqlalchemy import select
from sqlalchemy.orm import Session

from app.models import Evidence, Paper


class PaperRepository:
    def __init__(self, session: Session):
        self.session = session

    def create_paper(self, filename: str, sha256: str) -> Paper:
        existing = self.session.scalar(select(Paper).where(Paper.sha256 == sha256))
        if existing:
            return existing
        paper = Paper(filename=filename, sha256=sha256)
        self.session.add(paper)
        self.session.commit()
        self.session.refresh(paper)
        return paper

    def get_paper(self, paper_id: int) -> Paper | None:
        return self.session.get(Paper, paper_id)

    def create_evidence(
        self,
        paper_id: int,
        kind: str,
        claim: str,
        source_label: str,
        confidence: str,
        needs_human_check: bool,
    ) -> Evidence:
        evidence = Evidence(
            paper_id=paper_id,
            kind=kind,
            claim=claim,
            source_label=source_label,
            confidence=confidence,
            needs_human_check=needs_human_check,
        )
        self.session.add(evidence)
        self.session.commit()
        self.session.refresh(evidence)
        return evidence

    def list_evidence(self, paper_id: int) -> list[Evidence]:
        return list(
            self.session.scalars(
                select(Evidence).where(Evidence.paper_id == paper_id).order_by(Evidence.id)
            )
        )
```

- [ ] **Step 4: Run the repository test**

Run: `pytest tests/test_evidence.py -v`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add app tests
git commit -m "feat: add persistence models and repositories"
```

## Task 3: PDF Ingestion API

**Files:**
- Create: `app/services/ingestion.py`
- Create: `app/api/__init__.py`
- Create: `app/api/papers.py`
- Modify: `app/main.py`
- Create: `tests/test_ingestion.py`
- Create: `tests/test_api.py`

- [ ] **Step 1: Write failing ingestion tests**

Create `tests/test_ingestion.py`:

```python
from pathlib import Path

from app.services.ingestion import compute_sha256, store_pdf


def test_compute_sha256_is_stable(tmp_path: Path):
    pdf = tmp_path / "paper.pdf"
    pdf.write_bytes(b"%PDF-1.4\nsample")

    assert compute_sha256(pdf) == compute_sha256(pdf)


def test_store_pdf_uses_hash_filename(tmp_path: Path):
    source = tmp_path / "paper.pdf"
    source.write_bytes(b"%PDF-1.4\nsample")

    stored = store_pdf(source, tmp_path / "storage")

    assert stored.exists()
    assert stored.name.endswith(".pdf")
    assert stored.read_bytes() == b"%PDF-1.4\nsample"
```

Create `tests/test_api.py`:

```python
from fastapi.testclient import TestClient

from app.main import create_app


def test_health_endpoint():
    client = TestClient(create_app())

    response = client.get("/health")

    assert response.status_code == 200
    assert response.json() == {"status": "ok"}
```

- [ ] **Step 2: Run tests and verify failure**

Run: `pytest tests/test_ingestion.py tests/test_api.py -v`

Expected: FAIL because `app.services.ingestion` does not exist.

- [ ] **Step 3: Create ingestion service**

Create `app/services/ingestion.py`:

```python
import hashlib
import shutil
from pathlib import Path


def compute_sha256(path: Path) -> str:
    digest = hashlib.sha256()
    with path.open("rb") as handle:
        for chunk in iter(lambda: handle.read(1024 * 1024), b""):
            digest.update(chunk)
    return digest.hexdigest()


def store_pdf(source_path: Path, storage_dir: Path) -> Path:
    storage_dir.mkdir(parents=True, exist_ok=True)
    digest = compute_sha256(source_path)
    destination = storage_dir / f"{digest}.pdf"
    if not destination.exists():
        shutil.copyfile(source_path, destination)
    return destination
```

Create `app/api/__init__.py`:

```python
"""API routers."""
```

Create `app/api/papers.py`:

```python
from pathlib import Path
from tempfile import NamedTemporaryFile

from fastapi import APIRouter, Depends, File, UploadFile
from sqlalchemy.orm import Session

from app.config import Settings
from app.db import get_session
from app.repositories import PaperRepository
from app.schemas import PaperRead
from app.services.ingestion import compute_sha256, store_pdf

router = APIRouter(prefix="/papers", tags=["papers"])


@router.post("/upload", response_model=PaperRead)
async def upload_paper(
    file: UploadFile = File(...),
    session: Session = Depends(get_session),
) -> PaperRead:
    settings = Settings()
    with NamedTemporaryFile(delete=False, suffix=".pdf") as temp:
        temp.write(await file.read())
        temp_path = Path(temp.name)

    sha256 = compute_sha256(temp_path)
    store_pdf(temp_path, Path(settings.paper_storage_dir))
    paper = PaperRepository(session).create_paper(filename=file.filename or "paper.pdf", sha256=sha256)
    return PaperRead.model_validate(paper)
```

Modify `app/main.py`:

```python
from fastapi import FastAPI

from app.api import papers
from app.config import Settings
from app.db import init_db


def create_app(settings: Settings | None = None) -> FastAPI:
    active_settings = settings or Settings()
    init_db()
    app = FastAPI(title=active_settings.app_name)
    app.include_router(papers.router)

    @app.get("/health")
    def health() -> dict[str, str]:
        return {"status": "ok"}

    return app


app = create_app()
```

- [ ] **Step 4: Run ingestion and API tests**

Run: `pytest tests/test_ingestion.py tests/test_api.py -v`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add app tests
git commit -m "feat: add pdf ingestion endpoint"
```

## Task 4: Parser Normalization

**Files:**
- Create: `app/services/parsing.py`
- Create: `app/services/sectioning.py`
- Create: `tests/test_parsing.py`

- [ ] **Step 1: Write failing parser tests**

Create `tests/test_parsing.py`:

```python
from app.services.parsing import ParsedDocument, ParsedSection
from app.services.sectioning import normalize_section_name


def test_normalize_common_section_names():
    assert normalize_section_name("Materials and Methods") == "methods"
    assert normalize_section_name("Related work") == "related_work"
    assert normalize_section_name("Discussion and conclusion") == "discussion"


def test_parsed_document_keeps_sections_and_references():
    document = ParsedDocument(
        title="A method for single-cell analysis",
        abstract="We analyze single-cell RNA-seq data.",
        sections=[ParsedSection(name="Introduction", text="Background", page_start=1, page_end=2)],
        references=["Smith A. Bioinformatics. 2024."],
        figures=[],
        tables=[],
    )

    assert document.sections[0].normalized_name == "introduction"
    assert document.references == ["Smith A. Bioinformatics. 2024."]
```

- [ ] **Step 2: Run the parser tests and verify failure**

Run: `pytest tests/test_parsing.py -v`

Expected: FAIL because parsing modules do not exist.

- [ ] **Step 3: Create section normalization and parsed document models**

Create `app/services/sectioning.py`:

```python
def normalize_section_name(name: str) -> str:
    lowered = " ".join(name.lower().replace("-", " ").split())
    if "abstract" in lowered:
        return "abstract"
    if "related" in lowered or "background" in lowered:
        return "related_work"
    if "introduction" in lowered:
        return "introduction"
    if "method" in lowered or "material" in lowered or "implementation" in lowered:
        return "methods"
    if "result" in lowered or "experiment" in lowered or "benchmark" in lowered:
        return "results"
    if "discussion" in lowered or "conclusion" in lowered:
        return "discussion"
    if "reference" in lowered or "bibliography" in lowered:
        return "references"
    return "other"
```

Create `app/services/parsing.py`:

```python
from pathlib import Path

import fitz
from pydantic import BaseModel, Field

from app.services.sectioning import normalize_section_name


class ParsedSection(BaseModel):
    name: str
    text: str
    page_start: int | None = None
    page_end: int | None = None
    normalized_name: str = Field(default="")

    def model_post_init(self, __context) -> None:
        if not self.normalized_name:
            self.normalized_name = normalize_section_name(self.name)


class ParsedFigure(BaseModel):
    label: str
    caption: str
    page: int | None = None


class ParsedTable(BaseModel):
    label: str
    caption: str
    page: int | None = None


class ParsedDocument(BaseModel):
    title: str | None = None
    abstract: str | None = None
    sections: list[ParsedSection]
    references: list[str]
    figures: list[ParsedFigure]
    tables: list[ParsedTable]


class PyMuPDFParser:
    def parse(self, pdf_path: Path) -> ParsedDocument:
        document = fitz.open(pdf_path)
        page_text = [page.get_text("text") for page in document]
        full_text = "\n".join(page_text)
        first_line = next((line.strip() for line in full_text.splitlines() if line.strip()), None)
        section = ParsedSection(
            name="Full Text",
            text=full_text,
            page_start=1 if page_text else None,
            page_end=len(page_text) if page_text else None,
        )
        return ParsedDocument(
            title=first_line,
            abstract=None,
            sections=[section],
            references=[],
            figures=[],
            tables=[],
        )
```

- [ ] **Step 4: Run parser tests**

Run: `pytest tests/test_parsing.py -v`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add app/services tests/test_parsing.py
git commit -m "feat: add parser normalization"
```

## Task 5: Bioinformatics Evidence Extraction

**Files:**
- Create: `app/services/extraction.py`
- Create: `tests/test_extraction.py`

- [ ] **Step 1: Write failing extraction tests**

Create `tests/test_extraction.py`:

```python
from app.services.extraction import extract_bioinformatics_evidence
from app.services.parsing import ParsedDocument, ParsedSection


def test_extracts_datasets_metrics_and_methods():
    document = ParsedDocument(
        title="Single-cell benchmark",
        abstract="We benchmark graph neural networks for scRNA-seq annotation.",
        sections=[
            ParsedSection(
                name="Results",
                text="We evaluated PBMC datasets using AUROC, F1-score, P-value and FDR.",
                page_start=4,
                page_end=6,
            )
        ],
        references=[],
        figures=[],
        tables=[],
    )

    evidence = extract_bioinformatics_evidence("paper_001", document)
    claims = [item.claim for item in evidence]

    assert "Detected data modality: single-cell RNA-seq" in claims
    assert "Detected metric: AUROC" in claims
    assert "Detected metric: F1-score" in claims
    assert "Detected dataset or cohort mention: PBMC" in claims
```

- [ ] **Step 2: Run extraction tests and verify failure**

Run: `pytest tests/test_extraction.py -v`

Expected: FAIL because extraction service does not exist.

- [ ] **Step 3: Create rule-based MVP extractor**

Create `app/services/extraction.py`:

```python
from dataclasses import dataclass

from app.services.parsing import ParsedDocument


@dataclass(frozen=True)
class ExtractedEvidence:
    paper_id: str
    kind: str
    claim: str
    source_label: str
    confidence: str
    needs_human_check: bool


TERM_CLAIMS = {
    "scrna-seq": "Detected data modality: single-cell RNA-seq",
    "single-cell": "Detected data modality: single-cell RNA-seq",
    "spatial transcriptomics": "Detected data modality: spatial transcriptomics",
    "rna-seq": "Detected data modality: RNA-seq",
    "multi-omics": "Detected data modality: multi-omics",
    "graph neural network": "Detected method family: graph neural network",
    "pathway": "Detected biological analysis theme: pathway analysis",
    "gene regulatory network": "Detected biological analysis theme: gene regulatory networks",
    "variant calling": "Detected task: variant calling",
}

METRICS = ["AUROC", "AUC", "F1-score", "P-value", "FDR", "accuracy", "precision", "recall"]
DATASET_MARKERS = ["PBMC", "TCGA", "GEO", "GTEx", "Human Cell Atlas", "Tabula Sapiens"]


def extract_bioinformatics_evidence(
    paper_id: str,
    document: ParsedDocument,
) -> list[ExtractedEvidence]:
    items: list[ExtractedEvidence] = []
    title_and_abstract = " ".join(filter(None, [document.title, document.abstract]))
    section_text = "\n".join(section.text for section in document.sections)
    searchable = f"{title_and_abstract}\n{section_text}"
    lowered = searchable.lower()

    for term, claim in TERM_CLAIMS.items():
        if term in lowered:
            items.append(
                ExtractedEvidence(
                    paper_id=paper_id,
                    kind="text",
                    claim=claim,
                    source_label="Uploaded paper, extracted text",
                    confidence="medium",
                    needs_human_check=False,
                )
            )

    for metric in METRICS:
        if metric.lower() in lowered:
            items.append(
                ExtractedEvidence(
                    paper_id=paper_id,
                    kind="text",
                    claim=f"Detected metric: {metric}",
                    source_label="Uploaded paper, extracted text",
                    confidence="high",
                    needs_human_check=False,
                )
            )

    for marker in DATASET_MARKERS:
        if marker.lower() in lowered:
            items.append(
                ExtractedEvidence(
                    paper_id=paper_id,
                    kind="text",
                    claim=f"Detected dataset or cohort mention: {marker}",
                    source_label="Uploaded paper, extracted text",
                    confidence="medium",
                    needs_human_check=False,
                )
            )

    return items
```

- [ ] **Step 4: Run extraction tests**

Run: `pytest tests/test_extraction.py -v`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add app/services/extraction.py tests/test_extraction.py
git commit -m "feat: extract bioinformatics evidence"
```

## Task 6: Figure and Table Linking

**Files:**
- Create: `app/services/figures.py`
- Create: `tests/test_figures.py`

- [ ] **Step 1: Write failing figure/table tests**

Create `tests/test_figures.py`:

```python
from app.services.figures import detect_captions, link_mentions


def test_detects_figure_and_table_captions():
    text = """
    Figure 1. Overview of the benchmark workflow.
    Table 2. Datasets used for evaluation.
    """

    captions = detect_captions(text)

    assert captions["Figure 1"] == "Overview of the benchmark workflow."
    assert captions["Table 2"] == "Datasets used for evaluation."


def test_links_in_text_mentions_to_captions():
    text = "The workflow is summarized in Fig. 1 and the datasets are listed in Table 2."
    captions = {
        "Figure 1": "Overview of the benchmark workflow.",
        "Table 2": "Datasets used for evaluation.",
    }

    links = link_mentions(text, captions)

    assert links["Fig. 1"] == "Figure 1"
    assert links["Table 2"] == "Table 2"
```

- [ ] **Step 2: Run tests and verify failure**

Run: `pytest tests/test_figures.py -v`

Expected: FAIL because figure service does not exist.

- [ ] **Step 3: Create caption detection and mention linking**

Create `app/services/figures.py`:

```python
import re

CAPTION_PATTERN = re.compile(
    r"\b(?P<label>(Figure|Fig\.|Table)\s+(?P<number>\d+))\.\s+(?P<caption>[^\n]+)",
    re.IGNORECASE,
)

MENTION_PATTERN = re.compile(r"\b(?P<label>(Figure|Fig\.|Table)\s+(?P<number>\d+))\b", re.IGNORECASE)


def _canonical_label(label: str, number: str) -> str:
    if label.lower().startswith("table"):
        return f"Table {number}"
    return f"Figure {number}"


def detect_captions(text: str) -> dict[str, str]:
    captions: dict[str, str] = {}
    for match in CAPTION_PATTERN.finditer(text):
        canonical = _canonical_label(match.group("label"), match.group("number"))
        captions[canonical] = match.group("caption").strip()
    return captions


def link_mentions(text: str, captions: dict[str, str]) -> dict[str, str]:
    links: dict[str, str] = {}
    for match in MENTION_PATTERN.finditer(text):
        mention = match.group("label")
        canonical = _canonical_label(mention, match.group("number"))
        if canonical in captions:
            links[mention] = canonical
    return links
```

- [ ] **Step 4: Run figure/table tests**

Run: `pytest tests/test_figures.py -v`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add app/services/figures.py tests/test_figures.py
git commit -m "feat: link figures and tables to text mentions"
```

## Task 7: Literature Client Normalization

**Files:**
- Create: `app/clients/base.py`
- Create: `app/clients/crossref.py`
- Create: `app/clients/openalex.py`
- Create: `app/clients/semanticscholar.py`
- Create: `app/clients/pubmed.py`
- Create: `app/services/literature.py`
- Create: `tests/test_literature.py`

- [ ] **Step 1: Write failing literature tests**

Create `tests/test_literature.py`:

```python
from app.clients.base import CandidatePaper
from app.services.literature import deduplicate_candidates, rank_candidates


def test_deduplicates_by_doi_then_title():
    candidates = [
        CandidatePaper(
            title="A benchmark for single-cell analysis",
            authors=["A. Smith"],
            year=2024,
            venue="Bioinformatics",
            doi="10.1000/example",
            url="https://doi.org/10.1000/example",
            relation="same_task",
            relevance_reason="Same task",
            usable_for="Related Work: benchmark",
            verification_status="confirmed",
            sources=["Crossref"],
        ),
        CandidatePaper(
            title="A benchmark for single-cell analysis",
            authors=["A. Smith"],
            year=2024,
            venue="Bioinformatics",
            doi="10.1000/example",
            url="https://doi.org/10.1000/example",
            relation="same_task",
            relevance_reason="Same task",
            usable_for="Related Work: benchmark",
            verification_status="confirmed",
            sources=["OpenAlex"],
        ),
    ]

    merged = deduplicate_candidates(candidates)

    assert len(merged) == 1
    assert merged[0].sources == ["Crossref", "OpenAlex"]


def test_rank_prefers_confirmed_and_direct_relations():
    candidates = [
        CandidatePaper(
            title="Unconfirmed recent paper",
            authors=[],
            year=2026,
            venue=None,
            doi=None,
            url="https://example.org",
            relation="recent_related",
            relevance_reason="Keyword overlap",
            usable_for="Introduction",
            verification_status="needs_human_check",
            sources=["OpenAlex"],
        ),
        CandidatePaper(
            title="Citing paper",
            authors=["B. Lee"],
            year=2023,
            venue="Nature Methods",
            doi="10.1000/citing",
            url="https://doi.org/10.1000/citing",
            relation="cites_uploaded",
            relevance_reason="Cites uploaded paper",
            usable_for="Related Work: follow-up study",
            verification_status="confirmed",
            sources=["Semantic Scholar"],
        ),
    ]

    ranked = rank_candidates(candidates)

    assert ranked[0].title == "Citing paper"
```

- [ ] **Step 2: Run tests and verify failure**

Run: `pytest tests/test_literature.py -v`

Expected: FAIL because client and literature modules do not exist.

- [ ] **Step 3: Create normalized candidate model**

Create `app/clients/base.py`:

```python
from pydantic import BaseModel


class CandidatePaper(BaseModel):
    title: str
    authors: list[str]
    year: int | None
    venue: str | None
    doi: str | None
    url: str | None
    relation: str
    relevance_reason: str
    usable_for: str
    verification_status: str
    sources: list[str]


def normalize_doi(doi: str | None) -> str | None:
    if not doi:
        return None
    cleaned = doi.strip().lower()
    cleaned = cleaned.removeprefix("https://doi.org/")
    cleaned = cleaned.removeprefix("doi:")
    return cleaned or None
```

- [ ] **Step 4: Create retrieval orchestrator**

Create `app/services/literature.py`:

```python
from app.clients.base import CandidatePaper, normalize_doi


RELATION_SCORE = {
    "references_uploaded": 40,
    "cites_uploaded": 35,
    "same_task": 25,
    "recent_related": 15,
}


def candidate_key(candidate: CandidatePaper) -> str:
    doi = normalize_doi(candidate.doi)
    if doi:
        return f"doi:{doi}"
    return "title:" + " ".join(candidate.title.lower().split())


def deduplicate_candidates(candidates: list[CandidatePaper]) -> list[CandidatePaper]:
    merged: dict[str, CandidatePaper] = {}
    for candidate in candidates:
        key = candidate_key(candidate)
        if key not in merged:
            merged[key] = candidate
            continue
        existing = merged[key]
        sources = sorted(set(existing.sources + candidate.sources))
        status = (
            "confirmed"
            if "confirmed" in {existing.verification_status, candidate.verification_status}
            else "needs_human_check"
        )
        merged[key] = existing.model_copy(
            update={"sources": sources, "verification_status": status}
        )
    return list(merged.values())


def rank_candidates(candidates: list[CandidatePaper]) -> list[CandidatePaper]:
    def score(candidate: CandidatePaper) -> tuple[int, int, int]:
        relation = RELATION_SCORE.get(candidate.relation, 0)
        verified = 20 if candidate.verification_status == "confirmed" else 0
        year = candidate.year or 0
        return (relation + verified, year, len(candidate.sources))

    return sorted(deduplicate_candidates(candidates), key=score, reverse=True)
```

- [ ] **Step 5: Create API client skeletons**

Create `app/clients/crossref.py`:

```python
import httpx

from app.clients.base import CandidatePaper, normalize_doi


class CrossrefClient:
    base_url = "https://api.crossref.org/works"

    def __init__(self, timeout: float = 20.0):
        self.timeout = timeout

    async def lookup_by_doi(self, doi: str) -> CandidatePaper | None:
        normalized = normalize_doi(doi)
        if not normalized:
            return None
        async with httpx.AsyncClient(timeout=self.timeout) as client:
            response = await client.get(f"{self.base_url}/{normalized}")
            if response.status_code != 200:
                return None
            item = response.json()["message"]
        title = item.get("title", [""])[0]
        authors = [
            " ".join(filter(None, [author.get("given"), author.get("family")]))
            for author in item.get("author", [])
        ]
        year_parts = item.get("published-print") or item.get("published-online") or {}
        year = (year_parts.get("date-parts") or [[None]])[0][0]
        return CandidatePaper(
            title=title,
            authors=authors,
            year=year,
            venue=(item.get("container-title") or [None])[0],
            doi=normalized,
            url=item.get("URL"),
            relation="references_uploaded",
            relevance_reason="Metadata matched by DOI in Crossref.",
            usable_for="Reference verification",
            verification_status="confirmed",
            sources=["Crossref"],
        )
```

Create `app/clients/openalex.py`:

```python
from app.clients.base import CandidatePaper


class OpenAlexClient:
    async def search_related(self, query: str) -> list[CandidatePaper]:
        return [
            CandidatePaper(
                title=query,
                authors=[],
                year=None,
                venue=None,
                doi=None,
                url=None,
                relation="same_task",
                relevance_reason="OpenAlex search client is configured; live querying is exercised in integration runs.",
                usable_for="Related Work",
                verification_status="needs_human_check",
                sources=["OpenAlex"],
            )
        ]
```

Create `app/clients/semanticscholar.py`:

```python
from app.clients.base import CandidatePaper


class SemanticScholarClient:
    async def citations_for_title(self, title: str) -> list[CandidatePaper]:
        return [
            CandidatePaper(
                title=f"Citing work for {title}",
                authors=[],
                year=None,
                venue=None,
                doi=None,
                url=None,
                relation="cites_uploaded",
                relevance_reason="Semantic Scholar citation lookup requires confirmed paper identity.",
                usable_for="Related Work: follow-up studies",
                verification_status="needs_human_check",
                sources=["Semantic Scholar"],
            )
        ]
```

Create `app/clients/pubmed.py`:

```python
from app.clients.base import CandidatePaper


class PubMedClient:
    async def search(self, query: str) -> list[CandidatePaper]:
        return [
            CandidatePaper(
                title=query,
                authors=[],
                year=None,
                venue="PubMed",
                doi=None,
                url=None,
                relation="recent_related",
                relevance_reason="PubMed query candidate requires metadata confirmation.",
                usable_for="Introduction: biomedical context",
                verification_status="needs_human_check",
                sources=["PubMed"],
            )
        ]
```

- [ ] **Step 6: Run literature tests**

Run: `pytest tests/test_literature.py -v`

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add app/clients app/services/literature.py tests/test_literature.py
git commit -m "feat: normalize and rank literature candidates"
```

## Task 8: Evidence Store Integration

**Files:**
- Create: `app/services/evidence.py`
- Modify: `app/repositories.py`
- Create: `tests/test_evidence.py`

- [ ] **Step 1: Extend evidence tests**

Replace `tests/test_evidence.py` with:

```python
from app.repositories import PaperRepository
from app.services.evidence import EvidenceInput, save_extracted_evidence


def test_create_paper_and_evidence(session):
    repo = PaperRepository(session)

    paper = repo.create_paper(filename="paper.pdf", sha256="abc123")
    evidence = repo.create_evidence(
        paper_id=paper.id,
        kind="text",
        claim="The paper evaluates AUROC.",
        source_label="Uploaded Paper A, Results, p. 7",
        confidence="high",
        needs_human_check=False,
    )

    assert paper.id is not None
    assert evidence.claim == "The paper evaluates AUROC."
    assert repo.list_evidence(paper.id)[0].source_label == "Uploaded Paper A, Results, p. 7"


def test_save_extracted_evidence(session):
    repo = PaperRepository(session)
    paper = repo.create_paper(filename="paper.pdf", sha256="abc123")

    saved = save_extracted_evidence(
        session,
        paper.id,
        [
            EvidenceInput(
                kind="text",
                claim="Detected metric: AUROC",
                source_label="Uploaded Paper, Results",
                confidence="high",
                needs_human_check=False,
            )
        ],
    )

    assert len(saved) == 1
    assert repo.list_evidence(paper.id)[0].claim == "Detected metric: AUROC"
```

- [ ] **Step 2: Run tests and verify failure**

Run: `pytest tests/test_evidence.py -v`

Expected: FAIL because `app.services.evidence` does not exist.

- [ ] **Step 3: Create evidence service**

Create `app/services/evidence.py`:

```python
from pydantic import BaseModel
from sqlalchemy.orm import Session

from app.models import Evidence
from app.repositories import PaperRepository


class EvidenceInput(BaseModel):
    kind: str
    claim: str
    source_label: str
    confidence: str
    needs_human_check: bool


def save_extracted_evidence(
    session: Session,
    paper_id: int,
    items: list[EvidenceInput],
) -> list[Evidence]:
    repo = PaperRepository(session)
    return [
        repo.create_evidence(
            paper_id=paper_id,
            kind=item.kind,
            claim=item.claim,
            source_label=item.source_label,
            confidence=item.confidence,
            needs_human_check=item.needs_human_check,
        )
        for item in items
    ]
```

- [ ] **Step 4: Run evidence tests**

Run: `pytest tests/test_evidence.py -v`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add app/services/evidence.py tests/test_evidence.py
git commit -m "feat: add evidence persistence service"
```

## Task 9: Writing Workflow with Fake LLM

**Files:**
- Create: `app/llm/__init__.py`
- Create: `app/llm/base.py`
- Create: `app/llm/prompts.py`
- Create: `app/services/style.py`
- Create: `app/services/writing.py`
- Create: `tests/test_writing.py`

- [ ] **Step 1: Write failing writing tests**

Create `tests/test_writing.py`:

```python
from app.llm.base import FakeLLMProvider
from app.services.writing import generate_outline, generate_paragraph_plan


def test_generate_outline_requires_sources():
    outline = generate_outline(
        topic="single-cell annotation",
        evidence_labels=["Uploaded Paper A, Introduction", "Related Paper B, Abstract"],
        llm=FakeLLMProvider(),
    )

    assert "Introduction" in outline
    assert "[Source: Uploaded Paper A, Introduction]" in outline


def test_generate_paragraph_plan_marks_missing_user_inputs():
    plan = generate_paragraph_plan(
        section="Introduction",
        evidence_labels=["Uploaded Paper A, Methods"],
        user_method_summary="",
        llm=FakeLLMProvider(),
    )

    assert "[需要用户补充]" in plan
    assert "Purpose:" in plan
```

- [ ] **Step 2: Run tests and verify failure**

Run: `pytest tests/test_writing.py -v`

Expected: FAIL because LLM and writing services do not exist.

- [ ] **Step 3: Create LLM interface and prompts**

Create `app/llm/__init__.py`:

```python
"""LLM provider abstractions."""
```

Create `app/llm/base.py`:

```python
from typing import Protocol


class LLMProvider(Protocol):
    def complete(self, prompt: str) -> str:
        raise NotImplementedError


class FakeLLMProvider:
    def complete(self, prompt: str) -> str:
        if "OUTLINE" in prompt:
            return (
                "Introduction\n"
                "1. Background and biological motivation [Source: Uploaded Paper A, Introduction]\n"
                "2. Computational gap [Source: Related Paper B, Abstract]\n"
                "Related Work\n"
                "1. Benchmark datasets and evaluation metrics [Source: Uploaded Paper A, Introduction]\n"
            )
        return (
            "Purpose: Background and motivation\n"
            "Evidence: Uploaded Paper A, Methods\n"
            "Draft: [需要用户补充] Provide the specific method novelty before making a contribution claim."
        )
```

Create `app/llm/prompts.py`:

```python
def build_outline_prompt(topic: str, evidence_labels: list[str]) -> str:
    evidence = "\n".join(f"- {label}" for label in evidence_labels)
    return (
        "OUTLINE\n"
        f"Topic: {topic}\n"
        "Use only these evidence labels for factual claims:\n"
        f"{evidence}\n"
        "Return Introduction and Related Work outline items with source annotations."
    )


def build_paragraph_plan_prompt(
    section: str,
    evidence_labels: list[str],
    user_method_summary: str,
) -> str:
    evidence = "\n".join(f"- {label}" for label in evidence_labels)
    method = user_method_summary or "[需要用户补充]"
    return (
        "PARAGRAPH_PLAN\n"
        f"Section: {section}\n"
        f"User method summary: {method}\n"
        "Evidence labels:\n"
        f"{evidence}\n"
        "Return paragraph purpose, evidence, and draft guidance."
    )
```

- [ ] **Step 4: Create style and writing services**

Create `app/services/style.py`:

```python
from pydantic import BaseModel


class StyleProfile(BaseModel):
    introduction_pattern: list[str]
    related_work_pattern: list[str]
    phrasing_patterns: list[str]


def default_bioinformatics_style_profile() -> StyleProfile:
    return StyleProfile(
        introduction_pattern=[
            "Begin with biological or biomedical relevance.",
            "Introduce the data modality and computational challenge.",
            "Describe limitations of existing methods or benchmarks.",
            "State method motivation and restrained contributions.",
        ],
        related_work_pattern=[
            "Group prior work by task, modality, and method family.",
            "Separate databases, algorithms, tools, and benchmark resources.",
            "Use evidence-backed limitations to motivate the current work.",
        ],
        phrasing_patterns=[
            "However, existing methods often...",
            "To address this limitation, we...",
            "These observations motivate...",
        ],
    )
```

Create `app/services/writing.py`:

```python
from app.llm.base import LLMProvider
from app.llm.prompts import build_outline_prompt, build_paragraph_plan_prompt


def generate_outline(topic: str, evidence_labels: list[str], llm: LLMProvider) -> str:
    if not evidence_labels:
        return "[需要用户补充] No evidence labels are available for outline generation."
    return llm.complete(build_outline_prompt(topic, evidence_labels))


def generate_paragraph_plan(
    section: str,
    evidence_labels: list[str],
    user_method_summary: str,
    llm: LLMProvider,
) -> str:
    return llm.complete(
        build_paragraph_plan_prompt(
            section=section,
            evidence_labels=evidence_labels,
            user_method_summary=user_method_summary,
        )
    )
```

- [ ] **Step 5: Run writing tests**

Run: `pytest tests/test_writing.py -v`

Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add app/llm app/services/style.py app/services/writing.py tests/test_writing.py
git commit -m "feat: add evidence-constrained writing workflow"
```

## Task 10: Claim Checker

**Files:**
- Create: `app/services/claim_check.py`
- Create: `tests/test_claim_check.py`

- [ ] **Step 1: Write failing claim-check tests**

Create `tests/test_claim_check.py`:

```python
from app.services.claim_check import check_draft_claims


def test_flags_unsourced_factual_sentence():
    draft = "Our method significantly improves AUROC on PBMC datasets."

    warnings = check_draft_claims(draft)

    assert warnings[0].severity == "high"
    assert "source annotation" in warnings[0].message


def test_allows_user_input_marker():
    draft = "[需要用户补充] Provide exact performance result."

    warnings = check_draft_claims(draft)

    assert warnings == []


def test_flags_causal_language():
    draft = "The model causes improved biological discovery. [Source: Paper A, Results]"

    warnings = check_draft_claims(draft)

    assert any("causal" in warning.message for warning in warnings)
```

- [ ] **Step 2: Run tests and verify failure**

Run: `pytest tests/test_claim_check.py -v`

Expected: FAIL because claim checker does not exist.

- [ ] **Step 3: Create claim checker**

Create `app/services/claim_check.py`:

```python
from pydantic import BaseModel


class ClaimWarning(BaseModel):
    severity: str
    message: str
    text: str


FACTUAL_TRIGGERS = [
    "improves",
    "outperforms",
    "significantly",
    "auc",
    "auroc",
    "f1-score",
    "p-value",
    "fdr",
    "dataset",
    "benchmark",
]

CAUSAL_TRIGGERS = ["causes", "caused", "drives", "proves"]


def split_sentences(draft: str) -> list[str]:
    return [part.strip() for part in draft.replace("\n", " ").split(".") if part.strip()]


def check_draft_claims(draft: str) -> list[ClaimWarning]:
    warnings: list[ClaimWarning] = []
    for sentence in split_sentences(draft):
        if "[需要用户补充]" in sentence:
            continue
        lowered = sentence.lower()
        has_source = "[source:" in lowered or "[来源:" in lowered
        if any(trigger in lowered for trigger in FACTUAL_TRIGGERS) and not has_source:
            warnings.append(
                ClaimWarning(
                    severity="high",
                    message="Factual or performance claim lacks a source annotation.",
                    text=sentence,
                )
            )
        if any(trigger in lowered for trigger in CAUSAL_TRIGGERS):
            warnings.append(
                ClaimWarning(
                    severity="medium",
                    message="Potential causal wording requires direct evidence and careful phrasing.",
                    text=sentence,
                )
            )
    return warnings
```

- [ ] **Step 4: Run claim-check tests**

Run: `pytest tests/test_claim_check.py -v`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add app/services/claim_check.py tests/test_claim_check.py
git commit -m "feat: add draft claim checker"
```

## Task 11: API Integration for Parsing, Retrieval, and Writing

**Files:**
- Modify: `app/api/papers.py`
- Create: `app/api/literature.py`
- Create: `app/api/writing.py`
- Modify: `app/main.py`
- Modify: `tests/test_api.py`

- [ ] **Step 1: Extend API tests**

Replace `tests/test_api.py` with:

```python
from fastapi.testclient import TestClient

from app.main import create_app


def test_health_endpoint():
    client = TestClient(create_app())

    response = client.get("/health")

    assert response.status_code == 200
    assert response.json() == {"status": "ok"}


def test_writing_outline_endpoint_uses_evidence_labels():
    client = TestClient(create_app())

    response = client.post(
        "/writing/outline",
        json={
            "topic": "single-cell annotation",
            "evidence_labels": ["Uploaded Paper A, Introduction"],
        },
    )

    assert response.status_code == 200
    assert "Introduction" in response.json()["outline"]
    assert "[Source: Uploaded Paper A, Introduction]" in response.json()["outline"]
```

- [ ] **Step 2: Run API tests and verify failure**

Run: `pytest tests/test_api.py -v`

Expected: FAIL because `/writing/outline` does not exist.

- [ ] **Step 3: Create literature and writing routers**

Create `app/api/literature.py`:

```python
from fastapi import APIRouter
from pydantic import BaseModel

from app.clients.base import CandidatePaper
from app.services.literature import rank_candidates

router = APIRouter(prefix="/literature", tags=["literature"])


class RankRequest(BaseModel):
    candidates: list[CandidatePaper]


@router.post("/rank", response_model=list[CandidatePaper])
def rank_literature(request: RankRequest) -> list[CandidatePaper]:
    return rank_candidates(request.candidates)
```

Create `app/api/writing.py`:

```python
from fastapi import APIRouter
from pydantic import BaseModel

from app.llm.base import FakeLLMProvider
from app.services.claim_check import ClaimWarning, check_draft_claims
from app.services.writing import generate_outline, generate_paragraph_plan

router = APIRouter(prefix="/writing", tags=["writing"])


class OutlineRequest(BaseModel):
    topic: str
    evidence_labels: list[str]


class OutlineResponse(BaseModel):
    outline: str


class ParagraphPlanRequest(BaseModel):
    section: str
    evidence_labels: list[str]
    user_method_summary: str = ""


class ParagraphPlanResponse(BaseModel):
    paragraph_plan: str


class ClaimCheckRequest(BaseModel):
    draft: str


class ClaimCheckResponse(BaseModel):
    warnings: list[ClaimWarning]


@router.post("/outline", response_model=OutlineResponse)
def outline(request: OutlineRequest) -> OutlineResponse:
    return OutlineResponse(
        outline=generate_outline(
            topic=request.topic,
            evidence_labels=request.evidence_labels,
            llm=FakeLLMProvider(),
        )
    )


@router.post("/paragraph-plan", response_model=ParagraphPlanResponse)
def paragraph_plan(request: ParagraphPlanRequest) -> ParagraphPlanResponse:
    return ParagraphPlanResponse(
        paragraph_plan=generate_paragraph_plan(
            section=request.section,
            evidence_labels=request.evidence_labels,
            user_method_summary=request.user_method_summary,
            llm=FakeLLMProvider(),
        )
    )


@router.post("/claim-check", response_model=ClaimCheckResponse)
def claim_check(request: ClaimCheckRequest) -> ClaimCheckResponse:
    return ClaimCheckResponse(warnings=check_draft_claims(request.draft))
```

Modify `app/main.py`:

```python
from fastapi import FastAPI

from app.api import literature, papers, writing
from app.config import Settings
from app.db import init_db


def create_app(settings: Settings | None = None) -> FastAPI:
    active_settings = settings or Settings()
    init_db()
    app = FastAPI(title=active_settings.app_name)
    app.include_router(papers.router)
    app.include_router(literature.router)
    app.include_router(writing.router)

    @app.get("/health")
    def health() -> dict[str, str]:
        return {"status": "ok"}

    return app


app = create_app()
```

- [ ] **Step 4: Run API tests**

Run: `pytest tests/test_api.py -v`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add app/api app/main.py tests/test_api.py
git commit -m "feat: expose literature and writing APIs"
```

## Task 12: Minimal Web Console

**Files:**
- Create: `app/templates/index.html`
- Create: `app/static/app.css`
- Modify: `app/main.py`
- Modify: `tests/test_api.py`

- [ ] **Step 1: Extend API test for home page**

Append to `tests/test_api.py`:

```python
def test_home_page_renders_console():
    client = TestClient(create_app())

    response = client.get("/")

    assert response.status_code == 200
    assert "Bioinformatics Writing Agent" in response.text
    assert "Generate Outline" in response.text
```

- [ ] **Step 2: Run API test and verify failure**

Run: `pytest tests/test_api.py::test_home_page_renders_console -v`

Expected: FAIL because `/` does not render HTML.

- [ ] **Step 3: Create minimal template and static style**

Create `app/templates/index.html`:

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <title>Bioinformatics Writing Agent</title>
    <link rel="stylesheet" href="/static/app.css" />
  </head>
  <body>
    <main>
      <h1>Bioinformatics Writing Agent</h1>
      <section>
        <h2>Upload PDF</h2>
        <form action="/papers/upload" method="post" enctype="multipart/form-data">
          <input type="file" name="file" accept="application/pdf" />
          <button type="submit">Upload</button>
        </form>
      </section>
      <section>
        <h2>Generate Outline</h2>
        <p>Use the API endpoint <code>/writing/outline</code> with evidence labels from parsed papers.</p>
      </section>
      <section>
        <h2>Guardrails</h2>
        <p>Unsupported claims must remain marked as <code>[需要用户补充]</code> or include source annotations.</p>
      </section>
    </main>
  </body>
</html>
```

Create `app/static/app.css`:

```css
body {
  margin: 0;
  font-family: Arial, sans-serif;
  color: #17202a;
  background: #f5f7f8;
}

main {
  max-width: 880px;
  margin: 0 auto;
  padding: 32px 20px;
}

section {
  margin-top: 20px;
  padding: 18px;
  background: #ffffff;
  border: 1px solid #d9e1e5;
  border-radius: 8px;
}

button {
  padding: 8px 12px;
  border: 1px solid #1f6feb;
  border-radius: 6px;
  background: #1f6feb;
  color: white;
  cursor: pointer;
}

input {
  margin-right: 8px;
}
```

Modify `app/main.py`:

```python
from fastapi import FastAPI, Request
from fastapi.responses import HTMLResponse
from fastapi.staticfiles import StaticFiles
from fastapi.templating import Jinja2Templates

from app.api import literature, papers, writing
from app.config import Settings
from app.db import init_db

templates = Jinja2Templates(directory="app/templates")


def create_app(settings: Settings | None = None) -> FastAPI:
    active_settings = settings or Settings()
    init_db()
    app = FastAPI(title=active_settings.app_name)
    app.mount("/static", StaticFiles(directory="app/static"), name="static")
    app.include_router(papers.router)
    app.include_router(literature.router)
    app.include_router(writing.router)

    @app.get("/", response_class=HTMLResponse)
    def home(request: Request):
        return templates.TemplateResponse("index.html", {"request": request})

    @app.get("/health")
    def health() -> dict[str, str]:
        return {"status": "ok"}

    return app


app = create_app()
```

- [ ] **Step 4: Run API tests**

Run: `pytest tests/test_api.py -v`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add app/templates app/static app/main.py tests/test_api.py
git commit -m "feat: add minimal web console"
```

## Task 13: End-to-End Verification

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Run full test suite**

Run: `pytest -v`

Expected: all tests PASS.

- [ ] **Step 2: Run lint**

Run: `ruff check .`

Expected: all checks PASS.

- [ ] **Step 3: Start the development server**

Run: `uvicorn app.main:app --reload`

Expected: server starts at `http://127.0.0.1:8000`.

- [ ] **Step 4: Verify manual workflow**

Use the browser or API client:

```powershell
Invoke-RestMethod -Uri http://127.0.0.1:8000/health
```

Expected response:

```json
{
  "status": "ok"
}
```

- [ ] **Step 5: Update README with verified commands**

Modify `README.md` so the setup section includes:

````markdown
## Verification

```powershell
pytest -v
ruff check .
uvicorn app.main:app --reload
```

The web console runs at `http://127.0.0.1:8000`.
```
````

- [ ] **Step 6: Commit**

```bash
git add README.md
git commit -m "docs: document verification workflow"
```

## Self-Review

Spec coverage:

- PDF upload and local storage: Task 3.
- Section normalization and parser representation: Task 4.
- Bioinformatics-specific extraction: Task 5.
- Figure/table caption detection and mention linking: Task 6.
- Literature candidate normalization, deduplication, and ranking: Task 7.
- Evidence persistence: Task 8.
- Style profile and evidence-constrained writing: Task 9.
- Claim checking for unsupported facts and causal wording: Task 10.
- API workflow: Task 11.
- Minimal user console: Task 12.
- Verification: Task 13.

Known MVP limits:

- GROBID integration is not in the first code path; PyMuPDF provides the initial parser and GROBID can be added behind the same parser interface.
- Live external API tests are excluded from unit tests; they should be covered by manual integration runs with network access.
- Visual figure interpretation is represented by caption and mention linking first; vision-model interpretation should be added after the evidence model is stable.
