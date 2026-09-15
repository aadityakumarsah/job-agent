<div align="center">

# 🎯 Job Agent

**Your AI job-hunting copilot — discover, tailor, and apply autonomously but safely.**

Discover fresh roles straight from companies' official ATS APIs, LLM-rank how well they fit you, tailor a no-drift ATS-safe résumé to each, and submit applications behind an unbreakable human-approval gate.

![Python](https://img.shields.io/badge/Python-3.11+-3776AB?style=for-the-badge&logo=python&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-6366F1?style=for-the-badge)
![PRs](https://img.shields.io/badge/PRs-Welcome-brightgreen?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Alpha-e11d48?style=for-the-badge)

---

**Demo mode runs fully offline** — no API key, no network, no real world-touching. Try it in under a minute.

</div>

---

## ✨ Highlights

- 🔍 **Official ATS discovery** — reads Greenhouse, Lever, Ashby, and SmartRecruiters *public, no-auth* endpoints (plus cross-company SmartRecruiters search, Remotive, RemoteOK). No LinkedIn/Indeed scraping.
- 🧠 **LLM fit scoring** — every surviving job gets a `0–100` score with a `strong / possible / skip` verdict and `missing_requirements` reasons (Haiku-class model, strict JSON).
- 📄 **No-drift résumé tailoring** — rewrites emphasis, never fabricates. A *bidirectional* gate rejects invented metrics, uncredentialed certs, dropped employers, and scope inflation.
- 🛡️ **Human-gated apply** — needs `--submit` **and** your in-session approval. Never touches consent boxes, never handles login/captcha, never submits without you.
- 🖥️ **Local dashboard** — search / tailor / track / apply from a web UI bound to `127.0.0.1`.
- 🧩 **Chrome MV3 extension** — fills Greenhouse/Ashby/Lever forms in *your own* logged-in browser.
- 📊 **Application tracker** — statuses, notes, follow-up dates; applied jobs auto-hidden from searches.

## 🚀 Quick start

### Demo mode — no setup, no key, no network

```bash
pip install -e .
playwright install chromium        # one-time; only for the apply/browser flows

python -m job_agent search --demo    # discover + rank (mock jobs)
python -m job_agent tailor --demo    # FAKE resume → FAKE JD → sample PDF
python -m job_agent apply  --demo    # fill + "submit" a local form, end to end
```

`tailor --demo` writes an ATS-safe sample PDF + DOCX; `apply --demo` fills a local form, pauses for a simulated approval, and saves a confirmation screenshot. The whole pipeline with zero side effects.

### Real run

```bash
# 1. Install
pip install -e .

# 2. Configure
cp .env.example .env                                        # add your ANTHROPIC_API_KEY
cp search_profile.example.yaml search_profile.yaml          # roles / companies / location
cp data/answer_bank.example.yaml data/answer_bank.yaml      # your real apply answers
playwright install chromium                                 # one-time

# 3. Set up your base résumé once
#    Convert your .docx to data/career_facts.yaml (see "Résumé setup" below)

# 4. Go
python -m job_agent search                                  # fetch, filter, score, rank
python -m job_agent tailor --job <ID>                       # tailor to a match
python -m job_agent apply  --job <ID>                       # assisted apply (dry-run)
python -m job_agent apply  --job <ID> --submit              # real submit (needs your OK)
python -m job_agent dashboard                               # local web UI, port 8642
python -m job_agent applications                            # tracked log of every attempt
python -m job_agent discover                                # grow your board list
```

> A bare `python -m job_agent` with no subcommand defaults to `search`.

### Résumé setup (one-time)

```bash
python - <<'EOF'
from job_agent.tailor.extract import build_career_facts, write_career_facts

facts = build_career_facts(
    "path/to/your_resume.docx",
    email="you@email.com", phone="+1 555 000 0000",
    location="Austin, TX",
    links=["https://linkedin.com/in/you"],
    certifications=[{"name": "AWS Certified Data Engineer - Associate",
                     "issuer": "Amazon Web Services"}],
)
write_career_facts(facts, "data/career_facts.yaml")
EOF
```

This is the **source of truth** everything respects: company names, titles, durations, and metrics are locked in. Nothing is invented later.

### Useful search flags

`--profile PATH`, `--limit N`, `--days N` (recency window, default 30), `--max-age-hours N`, `--include-applied`, `--method {structured,tool}`.

## 🔧 How it works

```
src/job_agent/sources/          Greenhouse · Lever · Ashby · SmartRecruiters
                                sr-search · Remotive · RemoteOK
        │  each fetch() -> list[Job]   (coded against real API shapes)
        ▼
search.py   keyword ─▶ recency ─▶ location ─▶ seniority ─▶ dedup ─▶ experience
        │   every stage's survivor count is reported — no silent truncation
        ▼
scoring.py  LLM fit score per surviving job ─▶ ScoredJob {score, verdict, reasons}
        │
        ▼
cli.py      ranked rich table + persisted data/last_search.json
```

**Normalization.** Every board becomes one immutable `Job` model (tz-aware `posted_at`, `remote`, `country`, full description). Each source is written against captured responses in `tests/fixtures/`.

**Keyword pre-filter first.** Titles match your keywords *before* anything expensive — no LLM call is spent on an off-target role.

**Recency window (default 30 days).** Uses each board's real post date; roleless dates fall back to a small seen-ids cache.

**Location rule.** Keep remote or in-country roles; drop known-non-US even if remote; unknown countries go to the scorer.

**Optional level gates.** `max_seniority` and `experience_years` in your profile drop clearly-over-level roles *before* scoring.

**Scoring.** A low-cost Haiku-class model returns strict JSON via structured outputs, retrying once on malformed output (final fallback: keep the job, mark it `unscored`).

### 📄 Résumé tailoring — honesty by construction

- **Immutable career facts** — `.docx` → `data/career_facts.yaml` (gitignored), locked source of truth.
- **No invented metrics** — only real numbers from facts are cited; otherwise a specific *qualitative* achievement is written. Suggestions to add real figures go to the NOTES block as questions.
- **No-drift gate, bidirectional** — fabricated employers, unbanked metrics, hyped scope qualifiers ("enterprise-scale"…) *and* omitted real employers all **fail the build loudly**.
- **Verification on the artifact** — the rendered PDF's text is extracted back out and asserted selectable, with sections in order. A PDF that fails extraction is a failed build.
- **ATS-safe output** — clean single-column `.docx` → PDF via LibreOffice (bundled-font reportlab fallback).

### 🛡️ Assisted apply — cautious by construction

- **Two independent locks on submit** — `--submit` flag **and** in-session `approve`. Default is dry-run.
- **Never guesses an answer** — fields filled only from your answer bank or résumé; unmatched required fields block approval until you edit or skip.
- **Never handles credentials or captchas** — pauses, tells you what to do in the *same* browser, waits, re-checks, resumes — no lost state.
- **Full review before anything is sent** — every value + its source (e.g. `career_facts.email`, `answer_bank.salary_expectation`) plus every empty field.
- **Honesty-gated AI drafts** — free-text ("why us?") gets a Haiku draft grounded *only* in your facts + bank + this JD, tagged `[AI-DRAFT]` / `[NEEDS-INPUT]` / `[GATE-FLAGGED]`, editable inline, never auto-approved.
- **`[GROUNDED]` yes/no answers** — factual possession questions the facts settle explicitly show the fact that grounds them.

Scope: fully fillable on Greenhouse / Lever / Ashby embedded forms; Workday / iCIMS fills public fields then pauses for your login. Playwright drives the browser; all logic is unit-tested browser-free.

### 🖥️ Dashboard + Chrome extension

```bash
python -m job_agent dashboard        # http://127.0.0.1:8642 (local-only by design)
```

- **Dashboard** — ranked table joined with tracked state; Search / Tailor / Resume buttons run the real pipelines; application tracker; inline PDF résumé viewer; apply via your own Chrome or a controlled window.
- **Chrome extension** (`extension/`) — autofill Greenhouse/Ashby/Lever forms in your logged-in browser through the local backend only. Same non-negotiables: **never submits, never touches consent/legal boxes, never evades bot detection, never talks to an external server**. Popup groups results into *filled / drafts to review / needs your answer / you must confirm*.
- **Setup & manual test checklist:** [`extension/README.md`](extension/README.md).

## 📁 Project layout

```
src/job_agent/
  models.py          Job, ScoredJob (frozen Pydantic v2)
  config.py          .env + search_profile.yaml loading & validation
  http.py            shared httpx client (timeout, retries, error mapping)
  sources/           one module per ATS + JobSource base
  search.py          fetch → keyword → recency → location → seniority → dedup → experience
  geo.py             infer country from free-text location
  seniority.py       title → seniority level
  experience.py      required years-of-experience from a JD
  scoring.py         LLM fit scoring (structured + tool-use paths)
  discovery.py       board-token discovery via official ATS APIs
  cli.py             search / tailor / apply / applications / dashboard / discover
  tailor/
    extract.py       base resume (.docx) → career_facts.yaml
    career_facts.py  frozen CareerFacts models + allow-lists
    tailor.py        mega prompt + facts + JD → Sonnet → résumé + NOTES
    render_pdf.py    ATS-safe PDF + editable .docx
    verify.py        bidirectional no-drift gate + scope gate + PDF text gate
    jd_fetch.py      re-fetch full JD at tailor time
    demo/            committed FAKE facts / JD / stub response
  apply/
    answer_bank.py   frozen answer-bank models; contact merged from career_facts
    fields.py        immutable FormField / FillPlan types
    form_reader.py   read + classify form controls
    filler.py        answer-bank → FillPlan mapping; apply to the page
    grounded.py      facts-grounded yes/no answers ([GROUNDED], veto-first)
    screening.py     question routing + honesty-gated essay drafts
    review.py        human review gate (approve / edit / skip)
    handoff.py       pause/resume for login / captcha / account
    submit.py        two-lock submit gate + screenshot + JSONL log
    tracker.py       application log: statuses, notes, applied markers
    runner.py        orchestrates one application end to end
    browser.py       lazy Playwright launch (visible for real runs)
    demo_apply.py    the `apply --demo` offline flow
  dashboard/
    app.py           FastAPI app (127.0.0.1 only)
    service.py       thin layer over the CLI's functions
    apply_session.py live assisted-apply session over HTTP
    extension_api.py extension endpoints (fill-values, task hand-off)
    static/          the dashboard UI (single index.html)
extension/           Chrome MV3 extension (see extension/README.md)
prompts/             the tailoring mega prompt
tests/               pytest + respx — sources, filters, scoring, tailoring,
                     PDF, answer bank, apply, tracker, dashboard, extension API
```

## 🧑‍💻 Development

```bash
pip install -e ".[dev]"
pytest
```

Tests are fully offline: source parsers run against saved fixtures via `respx`; the scorer is exercised with a fake client covering valid output, retry-then-succeed, and the unscored fallback.

## 🔒 Privacy & safety

- `.env`, `data/`, résumé files, and career facts are **gitignored** — nothing personal leaves your machine unless you push it yourself.
- The dashboard binds **`127.0.0.1` only** and is not configurable otherwise.
- The extension talks **only** to your local backend; CORS admits `chrome-extension://` origins exclusively.
- Submission is **never** done via ATS APIs — browser-based, always human-gated.

## 🗺️ Roadmap

- [x] Slice 1 — discovery + LLM fit scoring
- [x] Slice 2 — résumé tailoring → ATS-safe PDF with a bidirectional no-drift honesty gate
- [x] Slice 3 — validated application answer bank (gitignored PII store)
- [x] Slice 4 — assisted apply in a visible browser, two-lock submit
- [x] Local dashboard + application tracker
- [x] Chrome MV3 extension with dashboard hand-off
- [x] `[GROUNDED]` facts-backed yes/no answers (veto-first)
- [x] Board-token discovery utility (`discover`)
- [x] Scan metadata + new-job tracking (NEW badges, newest-first sort)

## 📄 License

[MIT](LICENSE) © Aaditya Kumar Sah. Built for honest job hunting — use it the same way.
## Pro tips from paired review
Follow-up dates nudges help close stale applications; keep the dashboard
open while tweaking search_profile so re-scans reuse the live scorer.
