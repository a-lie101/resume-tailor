# Resume Tailor: Specification

Derived from `notes/design.md` and the spec interview (2026-09-27).
When this file conflicts with `design.md`, this file wins.

---

## 1. Purpose

Personal CLI tool for co-op recruiting. Input: a job posting (file path
or pasted text). Output: a one-page tailored LaTeX resume and PDF, a
selection record, a gap report, and an approval gate before anything
compiles.

Content rule: every claim in the output traces to a verified fact in
`facts/`. The posting drives selection and emphasis only. It never
supplies content.

---

## 2. Repository layout

```
resume-tailor/
  facts/                         # one YAML file per role or project
    esgtree.yaml
    qenta.yaml
    ewa.yaml
    mint.yaml
    finem.yaml
    engine-fault.yaml
    spotify.yaml
    geospatial.yaml
  skills.yaml                    # curated skill list (flat, typed, tagged)
  postings/                      # job postings, one .txt per posting
  output/                        # generated per posting; gitignored
    <posting-id>/
      resume.tex
      resume.pdf
      selection.json
      requirements.md
      gap_report.md
  sources/                       # old resumes for seeding facts only; read-only
  notes/
    design.md
  templates/
    resume.tex.jinja              # Jinja2 template with custom delimiters
  tailor/                        # Python package
    __init__.py
    pipeline.py                  # orchestrates all steps
    parse_posting.py             # step 1
    select.py                    # steps 2-3
    rewrite.py                   # step 4
    verify.py                    # step 5, verifier subagent
    render.py                    # step 6
    compile.py                   # step 7
    checks.py                    # deterministic checks (also runs as hook)
    models.py                    # Pydantic schemas for facts and interfaces
    skills.py                    # skills filtering and grouping
  pyproject.toml
  CLAUDE.md
  SPEC.md
  .gitignore
  .claude/
    settings.json                # hook and permission-rule config
```

`output/` is gitignored. Everything else is committed.

---

## 3. Fact schema

One YAML file per role or project. Role-level locked fields appear once
at the top; individual accomplishments are a list under `facts`.

```yaml
role:
  id: esgtree                               # unique slug
  title: "Technical Product Manager"        # locked; never altered in output
  employer: "ESGTree"                       # locked
  dates: "Sep 2024 -- Present"             # locked; format: Mon YYYY -- Mon YYYY
  location: "Remote"                        # locked

facts:
  - id: esgtree-dashboard
    what: "Built and shipped an interactive farm-level carbon dashboard"
    details: "Displays emissions breakdowns by scope and farm; used by credit analysts"
    outcome: "Demoed to multiple Canadian banks and credit unions"
    facts:
      farms: 78
      synthetic_data: true
    tags:
      - data-visualization
      - stakeholder-facing
      - python
      - react
    never_claim:
      - "78 real farms"
      - "production data"
    verified: true
```

### Field rules

| Field | Required | Notes |
|---|---|---|
| `role.id` | yes | slug, unique across all files |
| `role.title` | yes | locked; verbatim in output |
| `role.employer` | yes | locked |
| `role.dates` | yes | locked; `Mon YYYY -- Mon YYYY` format |
| `role.location` | yes | locked |
| `id` | yes | `<role-id>-<slug>`, unique across all files |
| `what` | yes | plain-language description |
| `details` | no | specifics |
| `outcome` | no | why it mattered |
| `facts` | no | key-value pairs; only these values may be quoted as specific figures |
| `tags` | yes | at least one; drive selection and skills filtering |
| `never_claim` | no | phrases that must not appear in any output |
| `verified` | yes | only `true` facts enter the pipeline; `false` facts are not passed to any model step |

### Number quoting rule

Only numbers presented as a direct, specific figure must match a value
in the `facts` subfield verbatim. Approximations ("over 70") are
permitted only if they do not exceed the true value. Invented figures
that have no `facts` entry at all are always a failure.

---

## 4. Skills file

`skills.yaml` is a flat list. The pipeline filters it to entries whose
`tag` appears in the tags of selected facts, then groups by `type` to
produce the LaTeX skills rows.

```yaml
skills:
  - name: "Python"
    tag: python
    type: languages
  - name: "SQL"
    tag: sql
    type: languages
  - name: "React"
    tag: react
    type: frameworks
  - name: "Git"
    tag: git
    type: developer-tools
```

Valid `type` values (determines row label in LaTeX): `languages`,
`frameworks`, `developer-tools`, `cloud`, `other`. Add new types by
adding entries; no code change needed.

---

## 5. Pipeline

Invoked by the `/tailor` skill. Steps run sequentially in
`pipeline.py`. A failure at any step aborts and prints the reason.

### Step 1: Parse posting (`parse_posting.py`)

**Input**: posting text (from file or stdin).  
**Output**: `posting.json` written to `output/<posting-id>/`.

```json
{
  "id": "miller-w27-digital-solutions",
  "title": "Digital Solutions Developer",
  "employer": "Miller Group",
  "required": ["Power Automate", "Power Apps", "SharePoint", "REST APIs"],
  "preferred": ["Python", "SQL"],
  "keywords": ["internal tools", "HR", "Microsoft Power Platform"],
  "readers": ["recruiter", "hiring manager"],
  "logistics": {
    "location": "Markham, ON",
    "duration_months": 12,
    "work_term": "W27"
  }
}
```

The posting ID is derived from the filename (`miller-w27-digital-solutions.txt`
-> `miller-w27-digital-solutions`). For stdin input, the pipeline
auto-saves to `postings/<employer>-<date>.txt` before parsing.

### Steps 2-3: Map and select (`select.py`)

**Input**: `posting.json` + all facts with `verified: true`.  
**Output**: `selection.json` (see §6).

The model maps each requirement from `posting.json` to the best
available fact, assigns a relevance score (1-10) to each candidate
bullet, and selects roles and bullets under a one-page budget. No
orientation bias; all verified facts are candidates regardless of role
type. Facts with `verified: false` are never passed to this step.

### Step 4: Rewrite bullets (`rewrite.py`)

**Input**: `selection.json`.  
**Output**: `selection.json` updated with `bullet_text` per bullet entry.

Each bullet must cite the fact IDs it draws from in the `fact_ids`
field. Writing rules from `design.md` apply: vary verbs, one idea per
bullet, two lines max, bold tools and technologies only, ongoing work in
present tense, no em dashes.

### Step 5: Verify (`verify.py`)

A subagent (`claim-verifier`) with fresh context receives
`selection.json` and the raw fact objects for each cited ID. It has no
access to the rewrite prompt.

**Checks performed:**
- Every claim in `bullet_text` traces to a cited fact field.
- No number in `bullet_text` appears unless it is a value in the
  `facts` subfields of the cited facts (or an approximation that does
  not exceed it).
- No `never_claim` phrase appears in any bullet.
- Locked fields (title, employer, dates, location) match the role
  record exactly.

**Output format:**

Pass: `{"result": "PASS"}`  
Fail: `{"result": "FAIL", "failures": [{"bullet_id": "...", "claim": "...", "reason": "..."}]}`

On FAIL: pipeline aborts and prints all failures. Does not proceed to
the approval gate or compile.

### Approval gate

After a verifier PASS, the pipeline prints two sections and prompts
for confirmation before compiling.

**Printed:**
1. **Hard gaps**: requirements from `posting.json` that have no
   matching fact (absent or empty `fact_ids` in `requirements_map`).
2. **Verifier result**: PASS confirmation, or failures list if somehow
   reached with warnings.

**Prompt**: `Approve and compile? [y/N]`

On N: pipeline exits without writing `resume.tex` or `resume.pdf`.

### Step 6: Render (`render.py`)

**Input**: approved `selection.json` + `skills.yaml`.  
**Output**: `output/<posting-id>/resume.tex`.

The model outputs a JSON render object (role blocks, bullet lists,
skills rows). Python fills the Jinja2 template at
`templates/resume.tex.jinja`. The model never writes raw LaTeX.

Jinja2 custom delimiters: `((*` / `*))` for blocks, `((=` / `=))` for
variables. This avoids collision with LaTeX braces.

All text inserted into the template has LaTeX special characters
escaped: `& % $ # _ { } ~ ^ \`.

Header (name, email, phone, links) and education section are hardcoded
in the template. The pipeline does not modify them.

### Step 7: Compile and page-check (`compile.py`)

Run `pdflatex` twice (for cross-references). Count pages with `pypdf`.

**If page count > 1, attempt in order:**

1. **Tightening pass**: re-run step 4 with an explicit instruction to
   shorten verbose bullets (remove filler words, tighten prose). Do not
   drop any bullets. Recompile.
2. **If still > 1**: drop the bullet with the lowest `score` in
   `selection.json`. Recompile. Repeat until exactly one page.
3. After any bullet change, re-run the verifier (step 5) and the
   approval gate before finalizing.

---

## 6. Interfaces

### `selection.json`

```json
{
  "posting_id": "miller-w27-digital-solutions",
  "roles": [
    {
      "role_id": "ewa",
      "title": "Software Developer Co-op",
      "employer": "EWA",
      "dates": "Jan 2025 -- Apr 2025",
      "location": "Remote",
      "bullets": [
        {
          "id": "ewa-slack-webhook",
          "fact_ids": ["ewa-slack", "ewa-openai"],
          "score": 9,
          "bullet_text": "Built a Slack webhook integration that ..."
        }
      ]
    }
  ],
  "skills_tags": ["python", "react", "sql"],
  "requirements_map": [
    {
      "requirement": "REST APIs",
      "fact_ids": ["ewa-rest-endpoint"],
      "gap": false
    },
    {
      "requirement": "Power Automate",
      "fact_ids": [],
      "gap": true,
      "gap_severity": "hard"
    }
  ]
}
```

### `gap_report.md`

```markdown
# Gap Report: Miller Group W27 Digital Solutions Developer

## Requirements vs Evidence

| Requirement     | Fact IDs          | Status    |
|----------------|-------------------|-----------|
| REST APIs       | ewa-rest-endpoint | matched   |
| Power Automate  | --                | hard gap  |
| Power Apps      | --                | hard gap  |

## Action Items

- **Power Automate**: No matching fact. ...
- **Power Apps**: No matching fact. ...
```

### `requirements.md`

Full requirement-to-evidence table from step 1 with fact IDs filled in
after step 2. One row per requirement.

---

## 7. Deterministic checks (`checks.py`)

Runs as a Claude Code hook after any write to `output/`. Also callable
directly. Cannot be skipped.

1. Every number in `resume.tex` inserted text appears verbatim in a
   `facts` subfield of a cited fact.
2. Every tool or technology named in `resume.tex` appears as a tag in a
   cited fact for the corresponding role.
3. No `never_claim` phrase from any cited fact appears in `resume.tex`.
4. Locked fields (title, employer, dates, location) in `resume.tex`
   match the role YAML record exactly.
5. `resume.pdf` is exactly one page (checked via `pypdf`).
6. No em dashes appear in the inserted text of `resume.tex`.

On any failure: print the violation with the line reference and exit
non-zero. The pipeline treats a non-zero exit as a hard failure.

---

## 8. Claude Code setup

### Skills

- `/tailor <path-or-paste>`: runs the full pipeline for one posting.
  Accepts a file path (`postings/foo.txt`) or a text paste (auto-saves
  to `postings/<derived-id>.txt` before running).
- `/add-fact`: interviews the user after completing work and writes a
  new fact entry with `verified: false`. Never modifies existing facts
  or sets `verified: true`.

### Subagent

`claim-verifier`: read-only, fresh context, implements step 5. Receives
`selection.json` and the raw fact objects. Has no access to the rewrite
prompt or any prior pipeline context.

### Hook

`.claude/settings.json` registers `tailor/checks.py` as a post-tool-use
hook on writes to `output/`.

### Permission rule

`.claude/settings.json` requires a confirmation prompt before any edit
to `facts/`.

---

## 9. Milestones

### v0: Fact bank

**Goal**: populate `facts/` by hand from `sources/`. Encode every
honesty constraint from `design.md`. Mark facts `verified: true` only
after manual review.

**Runnable check**:
```bash
uv run python -m tailor.models
```
Pydantic validates all YAML files in `facts/`. Exits 0 if all files are
valid; prints the first schema error otherwise. Every `never_claim`
phrase from `design.md` must be encoded before this milestone is done.

### v1: Renderer and compile

**Goal**: given a hand-written `selection.json`, produce a correct
one-page PDF. The output must reproduce `sources/resume-data-current.tex`
faithfully (no AI in the output path at this stage; validate the
template and renderer only).

**Runnable check**:
```bash
uv run python -m tailor.render output/v1-test selection-v1-test.json
uv run python -m tailor.compile output/v1-test
uv run python -m tailor.checks output/v1-test
```
Expected: `output/v1-test/resume.pdf` is exactly one page, all
deterministic checks exit 0, visual comparison against
`sources/resume-data-current.tex` shows only expected differences (no
missing bullets, no garbled characters, correct locked fields).

---

## 10. Out of scope

- Scraping job boards (SCOPE is behind UBC login; postings are pasted).
- A web UI.
- RAG, embeddings, or vector search.
- Automatic application submission.
- Cover letters (planned for v3).
- Multi-page or academic CV format.
- Inventing content for gaps (gaps are reported, not filled).

---

## 11. End-to-end verification

After v1 ships, run the full pipeline on both eval postings:

```bash
uv run python -m tailor.pipeline postings/nutrien-w27-maintenance-analytics.txt
uv run python -m tailor.pipeline postings/miller-w27-digital-solutions.txt
```

For each run, confirm all of the following:

1. `output/<id>/resume.pdf` opens and is exactly one page.
2. `output/<id>/gap_report.md` names the hard gaps documented in
   `design.md`:
   - Nutrien: Power BI, DAX, Power Query.
   - Miller: Power Automate, Power Apps, SharePoint.
3. All deterministic checks (`tailor/checks.py`) exit 0.
4. Every bullet in `selection.json` has at least one entry in
   `fact_ids`. No bullet cites zero facts.
5. The verifier returned `PASS` (visible in pipeline stdout).
6. No em dashes, no `never_claim` phrases, no altered locked fields
   appear in either PDF.
7. The ESGTree title reads "Technical Product Manager" in both outputs.
