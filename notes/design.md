# Resume Tailor: Design Notes

Decisions made before writing the spec. SPEC.md builds on these. If the spec interview changes one, update this file.

## Purpose

A personal tool for my co-op recruiting cycle. Input: a pasted job posting. Output: a one-page LaTeX resume tailored to that posting, its PDF, and a gap report. Personal use first. Not a product.

## The core rule

Resume content comes only from verified facts in `facts/`. The job posting decides what gets selected and emphasized. It never supplies content. No number, tool, title, or claim may appear in output unless a verified fact contains it.

Why: an LLM tailoring a resume is under constant pressure to claim what the posting asks for. Manual tailoring sessions already produced invented framing ("replacing a manual agronomy lookup") and an inflated title ("Data Engineer"). The tool must make that impossible, not just unlikely.

## Fact bank

- Format: YAML, one file per role or project, validated by a Pydantic schema. Malformed facts fail loudly.
- Resumes are outputs, never inputs. The resumes in `sources/` are only for seeding facts, and they contain known embellishments (see `sources/README.md`).
- One fact per accomplishment. Proposed fields:
  - `id`, `role`
  - `what`: what I did, stated plainly
  - `details`: specifics
  - `outcome`: why it mattered, who used it
  - `facts`: numbers and flags that may be quoted (for example `farms: 78`, `synthetic_data: true`)
  - `tags`: drive selection (for example `sql`, `api-integration`, `stakeholder-facing`, `industrial`, `geospatial`)
  - `never_claim`: phrases or claims that must not appear
  - `verified`: only facts marked `true` may be used
- Locked fields at the role level: title, employer, dates, location. The model cannot change these.
- No vector database or RAG. The whole bank fits in context. Retrieval would add a failure mode (silently missing a relevant fact) for no benefit.

### Known honesty constraints to encode

Verify each one before encoding it. They come from manual review sessions.

- ESGTree title is Technical Product Manager. Never "Data Engineer" or "Software Engineer".
- ESGTree dashboard farms were synthetic sample data ("78 sample farm operations"). The dashboard was demoed to multiple Canadian banks and credit unions.
- Rotation package: "benchmarked against published provincial data" is true. "Replacing a manual agronomy lookup" is not.
- No runtime or storage improvement claims for the geospatial work. No figure exists.
- MINT: the gesture classifier was not built as of September 2026. The signal pipeline and the QC module are real. Update when that changes.
- Qenta: no "30% test coverage" figure. No concurrency or async/await claims.
- FinEm: my work is emission factor reference data and bilingual content, not engineering. Confirm with my manager that Desjardins can be named.
- Power BI: claim only if the dashboard in the oldest resume really exists.
- Engine Fault Detection: no LangChain.

## Pipeline (a fixed workflow, not an autonomous agent)

The steps are known, so a fixed pipeline is more predictable, easier to debug, and easier to evaluate.

1. Parse the posting into structured JSON: required vs preferred, keywords, likely readers (recruiter, hiring manager), logistics.
2. Map each requirement to evidence in the fact bank. Report gaps honestly. Use the strongest model here. This is where judgment lives.
3. Select roles, projects, and bullets under a one-page budget.
4. Rewrite bullets for this posting. Every bullet cites the fact IDs it came from.
5. Verify in a separate pass with fresh context (a subagent that never saw the rewrite prompt). It checks every claim against the cited facts.
6. Render: the model outputs JSON, code fills a Jinja2 LaTeX template. The model never writes raw LaTeX.
7. Compile with pdflatex and count pages with pypdf. If not exactly one page, return to step 3 with a tighter budget.

## Deterministic checks (run before any LLM judging)

- Every number in the output appears in a cited fact.
- Every tool or technology named is in the tag list.
- No `never_claim` phrase appears.
- Titles, employers, and dates match locked fields exactly.
- Exactly one page.

These run as a Claude Code hook so they cannot be skipped.

## Outputs per posting

`output/<posting-id>/`:

- `resume.tex` and `resume.pdf`
- `selection.json` (every bullet with its fact IDs)
- `requirements.md` (requirement-to-evidence table)
- `gap_report.md`

The gap report is a first-class output. "You lack Power BI, here is a weekend project that closes it" is often worth more than the resume.

## Writing rules learned from manual tailoring

- Tool jargon stays (Python, SQL, React). Domain jargon mostly goes (Holos, PCAF, CanSIS, Welch PSD). A recruiter can't evaluate it.
- Bullets: one idea, two lines max.
- Every bullet says what it was for or how big it was, not only what was built.
- Vary verbs. "Built" once or twice per page at most.
- Tailor selection and emphasis heavily, phrasing lightly. Mirroring the posting's language reads as an echo.
- Think about who reads it: a recruiter's checklist first, then a hiring manager who may not be technical. For co-op roles, also consider training cost, work term count, and whether I'd be happy in the role.
- Bold only tools and technologies. Locations are not bold.
- Date format in LaTeX: `Mon YYYY -- Mon YYYY` (for example `Sep 2024 -- Present`).
- Ongoing work in present tense, finished work in past tense.
- No em dashes anywhere in generated text.

## LaTeX

- Template: Jake Gutierrez resume template. Current version: `sources/resume-data-current.tex`. Experience entries use `\resumeProjectHeading{\textbf{Title} $|$ \emph{Org} $|$ Location}{Dates}`.
- Engine: pdflatex. The template uses `\pdfgentounicode`, which only works in pdfLaTeX.
- Jinja2 with custom delimiters, because LaTeX braces collide with `{{ }}`.
- Escape LaTeX special characters in all inserted text: & % $ # _ { } ~ ^ \

## Claude Code setup

- `CLAUDE.md`: short project rules.
- Skill `tailor`: runs the pipeline for one posting (`/tailor postings/<file>`).
- Skill `add-fact`: interviews me after I finish something and writes a new unverified fact.
- Subagent `claim-verifier`: read-only, fresh context, checks output against facts.
- Hook: runs deterministic checks and the page count.
- Permission rule: edits to `facts/` always ask me.
- Everything lives in the repo. Cloud sessions don't load user-level config or plugins.

## Build order

- v0: fact bank by hand, reviewing old resumes line by line.
- v1: renderer, compile, page check. Reproduce `sources/resume-data-current.tex` from a hand-written `selection.json`. No AI in the output path.
- v2: deterministic checks with tests, then the `tailor` skill, verifier subagent, and hook.
- v3: eval set, cover letters from the same pipeline, gap-to-learning plans.

## First eval cases

Stored in `postings/`. Expected judgments from the manual analysis:

- **Nutrien, W27 Maintenance Analytics Developer (Fort Saskatchewan, AB).** A Power BI analyst for a plant maintenance team.
  - Hard gap: Power BI, DAX, Power Query.
  - Reframe Engine Fault Detection as predictive maintenance. Lead with recall.
  - Add relevant coursework (databases, data modeling, visualization).
  - EWA is relevant: national utility, and a work-tracking system is close to a CMMS.
  - SQL evidence is thin.
  - Flags: in-person relocation, 8 vs 12 months, prefers work terms 4 and 5, possible conflict with an ESGTree extension.
- **Miller Group, W27 Digital Solutions Developer (Markham, ON).** Internal HR tools on Microsoft Power Platform.
  - Start from the software orientation, not the data one.
  - EWA is the strongest match (REST endpoints, Slack webhooks, OpenAI API).
  - Spotify beats Engine Fault here (SQL, relational schemas).
  - Put the REST API back into the geospatial bullet.
  - Hard gap: Power Automate, Power Apps, SharePoint. The employer accepts equivalent experience.
  - Flags: low pay for the GTA, mostly low-code work.

## Out of scope for now

- Scraping job boards. SCOPE is behind UBC login, so postings are pasted in.
- A web UI.
- RAG or embeddings.
- Automatic applying.
- Cover letters (v3).

## Open questions for the spec interview

- How to represent one accomplishment with several phrasings for different audiences?
- Should the tool start from a "data" or "software" base selection, or select from scratch every time?
- Skills section: derived from the tags of selected facts, or a curated list?
- Gap report format.
- How page overflow trimming decides what to cut.
- How I review and approve output before sending.
