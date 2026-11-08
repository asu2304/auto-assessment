# Auto-Assessment Agent — Rebuild From Scratch

A learning rebuild of [asu2304/auto-assessment-agent](https://github.com/asu2304/auto-assessment-agent): first replicate it as-is, step by step, then add my own updates.

---

## What we're building

An AI grader. A teacher uploads a question paper or rubric plus student answer sheets (handwritten PDFs, images, DOCX or text). Several LLM "agents" then grade the sheets question by question, quoting the student's own words as evidence.

```
Question paper ─▶ Transcriber ─▶ Solver (skipped if a model answer is given)
Student sheet  ─▶ Transcriber ─▶ Evaluator ─▶ Auditor (plain Python, no LLM) ─▶ Report
                                                  └─▶ Regrade agent · Chat agent · History (SQLite)
```

| Stage | Role | Runs on |
|---|---|---|
| Transcriber | Reads handwritten/scanned PDFs and images into structured text | Gemini (vision) |
| Solver | Generates a reference solution when no model answer is supplied | Gemini |
| Evaluator | Grades criterion-by-criterion with verbatim evidence quotes; looks at page images for diagram questions | Gemini |
| Auditor | Checks score bounds and criterion-total arithmetic | Python (no LLM) |
| Regrade Agent | Re-checks a disputed criterion before changing a score | Gemini |
| Chat Agent | Answers follow-up questions grounded in the saved assessment | Gemini |

**Size of the original:** about 10.7k lines, but most of it sits in four files: `App.jsx` (3.5k lines), `styles.css` (2.7k), `web.py` (2.1k) and `agent.py` (1k). The core logic is much smaller than that.

---

## Roadmap (backend first, built from the inside out)

| Phase | What I build | What I'll learn | Done |
|---|---|---|---|
| **0** | Project setup: venv, git, folder layout | Python packaging basics, `.gitignore` | [ ] |
| **1** | `document_parser.py`: read PDF, image, DOCX and text files | Pillow, python-docx, dataclasses, bytes vs. text | [ ] |
| **2** | Pydantic models: `CriterionScore`, `QuestionEvaluation`, `AssessmentReport` | Pydantic v2, validators, JSON Schema | [ ] |
| **3** | First Gemini call with structured JSON output, plus retries | `google-genai`, structured outputs, exponential backoff | [ ] |
| **4** | The agents: Transcriber, Solver, Evaluator, Auditor, and the orchestrator that runs them | Multi-agent pipelines, prompt-injection defense, pdf2image/Poppler | [ ] |
| **5** | `web.py` part 1: FastAPI with `/api/assess` and `/api/models` | FastAPI, multipart uploads, async | [ ] |
| **6** | SQLite history, regrade, chat (including streaming) | sqlite3, server-sent events | [ ] |
| **7** | Google sign-in plus HMAC-signed session cookies | OAuth ID tokens, HMAC, cookies | [ ] |
| **8** | Agent memory (student weak areas, grading corrections), batch grading | Concurrency limits, feedback loops | [ ] |
| **9** | React + Vite frontend | React hooks, the Vite proxy, react-markdown + KaTeX | [ ] |
| **10** | Tests, `start.sh`, Docker, docker-compose, nginx | unittest, TestClient, multi-stage Docker builds, reverse proxies | [ ] |
| **11** | My own improvements | — | [ ] |

Milestones: something testable without an API key at the end of Phase 2; a working command-line grader by the end of Phase 4.

---

## Phase 0: Set up the project

1. The folder name has spaces, so always wrap the path in quotes in the terminal: `cd "…/auto assesment agent - ashutosh"`.
2. Run `git init` and commit as I go. Commit after each phase so I can look back at how things evolved.
3. Create the same layout as the original:
   ```
   auto_assessment/auto_assessment/__init__.py   (one line: __version__ = "0.1.0")
   auto_assessment/frontend/                     (leave empty for now)
   tests/
   ```
   The same name appears twice in that path on purpose. The outer folder holds the backend package and the frontend side by side. The inner one is the Python code.
4. Create a virtual environment: `python3 -m venv .venv` then `source .venv/bin/activate`.
5. **Don't copy `requirements.txt` yet.** Add packages only when I first need them, so I know why each one is there. For now, install just `pip install Pillow python-docx`.
6. Write a `.gitignore` covering `.venv/`, `__pycache__/`, `.env`, `*.db`, and `node_modules/`.
7. Run `brew install poppler` now. It's needed in Phase 4, which turns PDF pages into images.

---

## Phase 1: Build `document_parser.py`

**Goal:** given a filename and the raw file bytes, return a standard object describing what was found in the file.

Build it in this order:

1. **A `ParsedDocument` dataclass** with these fields: `filename`, `text=""`, `images` (a list of PIL images), `pdf_bytes=None`, `mime_type=None`, `error=None`.
   - Think about this: why does `images` need `field(default_factory=list)` and not `= []`? Try it the wrong way and see what Python says.
2. **`extract_content_from_file(filename, file_bytes)`**, which branches on the file extension:
   - **Empty bytes:** set `error` and return.
   - **`.pdf`:** don't parse it at all. Just store the raw bytes and set `mime_type="application/pdf"`. Reject files over 50 MB.
   - **`.png`, `.jpg`, `.jpeg`, `.webp`:** open with `Image.open(io.BytesIO(...)).convert("RGB")` and add it to `images`. Get the mime type from `Image.MIME[img.format]`.
   - **`.docx`:** write a helper `_extract_docx_text`. It loops over the paragraphs, turns paragraphs styled "Heading N" into Markdown `#` headings, then adds the tables as `cell | cell | cell` rows.
   - **`.txt`, `.md`, `.csv`:** decode as UTF-8 with `errors="ignore"`, strip whitespace, and set the matching mime type.
   - **Anything else:** set an "Unsupported file type" error.
3. **Important design choice:** the function never raises an exception. Problems go into `.error`. Why does that suit a web API that receives many files in one request?

**Test it by hand.** Create a small `.txt` file, a `.docx` with a heading and a table, and a phone photo of handwriting. Load each one with `open(path, "rb").read()`, pass it to the function, and print the result.

**Questions to think about (answer them before Phase 3):**
- Why convert images to RGB? What would go wrong with a transparent PNG later, when it gets saved as JPEG?
- Why keep PDFs as raw bytes here instead of turning them into images straight away? (Hint: Gemini can read PDFs directly, but the Transcriber later turns pages into images anyway. The trade-off comes up in Phase 4.)
- `errors="ignore"` silently drops bytes it can't decode. When could that hide a real problem?

When done: share `document_parser.py` for a review against the original, then move on to Phase 2.

---

## Phases 2–11

Detailed instructions get added here as each phase begins.

---

## Notes and challenges log

_Write down problems I hit, what caused them, and how I fixed them._

-
