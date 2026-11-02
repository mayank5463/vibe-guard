<div align="center">

# 🛡️ Vibe Guard

**A security layer for AI-generated code and AI-powered apps.**
Catch leaked keys and broken access control before they ship.

![Python](https://img.shields.io/badge/python-3.10%2B-blue)
![License](https://img.shields.io/badge/license-MIT-green)
![Dependencies](https://img.shields.io/badge/core%20dependencies-0-brightgreen)
![Built for](https://img.shields.io/badge/built%20for-Meta%20AI%20Hackathon-0467DF)

</div>

---

## 🚨 The problem

AI coding assistants let anyone build an app in an afternoon ("vibe coding"), but the
code they generate often ships with the same dangerous mistakes:

- 🔑 **Hardcoded API keys and passwords** pushed straight to GitHub
- 🚪 **Broken access control**: routes with no login check, users who can read or delete
  *anyone's* data by changing an ID in the URL (IDOR), and roles taken from the client
- 💉 **Injection and unsafe defaults**: string-built SQL, wide-open CORS, debug mode left on

Broken access control is the #1 web application risk in the OWASP Top 10, and leaked
secrets are one of the most common ways AI-built apps get compromised. Most beginners
building with AI never run a security review.

## ✅ What Vibe Guard does

Vibe Guard scans a project, flags these problems with file and line numbers, redacts
any secret it finds, and **fails the CI build** when something serious is detected.

| Capability | Status |
|---|---|
| Exposed secrets (AWS, OpenAI, Anthropic, GitHub, Google, Stripe keys, private keys, hardcoded passwords) | ✅ Built |
| Committed `.env` files | ✅ Built |
| Missing authentication on routes (Flask, FastAPI, Express) | ✅ Built |
| IDOR: objects fetched by ID with no ownership check | ✅ Built |
| Privilege escalation: role / `is_admin` taken from client input | ✅ Built |
| Mass assignment from raw request body | ✅ Built |
| SQL built from strings, wide-open CORS, debug mode, disabled token verification | ✅ Built |
| GitHub Action that blocks risky pull requests | ✅ Built |
| Llama-powered triage: false-positive filtering, explanations, auto-fix | 🛠️ In progress |
| Prompt-injection guard for AI agents (PromptGuard 2) | 🗺️ Planned |
| Insecure-code check on model output (CodeShield) | 🗺️ Planned |
| Dashboard (Unprotected vs Protected demo) | 🗺️ Planned |

## 🎬 Demo

```text
$ python vibeguard_scan.py demo_app

[HIGH]   app.py:8   Exposed OpenAI-style API key  sk-d... (37 chars)
[HIGH]   app.py:9   Exposed AWS access key  AKIA... (20 chars)
[HIGH]   app.py:13  Possible IDOR: object fetched by ID with no ownership check  /api/users/<int:user_id>
[HIGH]   app.py:15  SQL built with string formatting (SQL injection)
[HIGH]   app.py:18  Route has no visible authentication  /api/users/<int:user_id>
[HIGH]   app.py:26  Privilege/role taken from client input (privilege escalation)
[HIGH]   app.py:27  Mass assignment from raw request body
[HIGH]   server.js:12  Route has no auth middleware  DELETE /api/admin/users/:id
[MEDIUM] app.py:6   Wide-open CORS (any website can call this API)
[MEDIUM] app.py:38  Debug mode enabled (leaks internals, can allow code execution)
...
17 findings (12 high).
```

The protected route `@login_required /api/me` is correctly **not** flagged.
Secrets are always redacted, so the report itself can never leak a key.

## 🚀 Quick start

Requires Python 3.10+. The core scanner has **zero dependencies**.

```bash
git clone https://github.com/YOUR_USERNAME/vibe-guard.git
cd vibe-guard

# scan the intentionally insecure demo app
python vibeguard_scan.py demo_app

# scan your own project
python vibeguard_scan.py /path/to/your/project

# machine-readable output (for dashboards and the LLM layer)
python vibeguard_scan.py /path/to/your/project --json

# skip folders
python vibeguard_scan.py . --exclude demo_app tests
```

Exit code is `1` if any HIGH finding exists, so it works as a CI gate.

## 🤖 Use it in GitHub Actions

Copy `.github/workflows/vibeguard.yml` into your repo. Every push and pull request is
scanned, and risky code turns the check red.

## 🔌 Choose your model

Vibe Guard's LLM layer uses any **OpenAI-compatible** API, so the provider is just
three values in `.env` (`LLM_BASE_URL`, `LLM_MODEL`, `LLM_API_KEY`).

```bash
cp .env.example .env      # then edit .env
python check_llm.py       # confirms your key and model work
```

Supported out of the box: **Meta Model API** (Muse Spark) and OpenRouter (Meta Llama
models). Never use a "contributor" model variant on private code, because those
variants allow the provider to use your data to improve its products.

## 🧠 How it works

```
Code / PR  →  Layer 1: deterministic scanners  →  findings (JSON)
           →  Layer 2: Llama triage (in progress) →  explain / fix / drop false positives
           →  Layer 3: enforcement (CI exit code) →  block or allow merge
```

Pattern scanners find *candidates* fast and for free. An LLM then judges whether each is
a real problem, which is something regex cannot do, such as "who *should* be allowed to
call this route?". Details in [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md).

## 🗂️ Project structure

```
vibe-guard/
├── vibeguard_scan.py          # the scanner (standard library only)
├── check_llm.py               # tests your LLM key / model
├── demo_app/                  # deliberately insecure app (fake keys only!)
│   ├── app.py                 #   Flask-style example
│   └── server.js              #   Express-style example
├── tests/test_scanner.py      # unit tests
├── docs/
│   ├── ARCHITECTURE.md        # design and finding format
│   ├── OWASP_COVERAGE.md      # mapping to the OWASP Top 10 for LLM apps
│   └── HACKATHON_PLAN.md      # build checklist
├── .github/workflows/         # CI: runs Vibe Guard on every PR
├── .env.example               # template for API keys (copy to .env)
├── .gitignore                 # keeps .env and junk out of git
├── requirements.txt
└── LICENSE
```

## 🧪 Run the tests

```bash
python -m unittest discover tests -v
```

## ⚠️ Honest limitations

- Access-control checks are **heuristics**. A route protected by middleware defined in
  another file can look unprotected (false positive), and unusual frameworks are missed.
- Currently understands Flask, FastAPI and Express patterns.
- No tool guarantees security. Keep human review for critical code.
- Business rules (who may see what) need human or LLM judgement, which is why the Llama
  triage layer exists.

## 🗺️ Roadmap

1. Llama triage layer: explain each finding and generate a fix
2. LlamaFirewall integration: PromptGuard 2 and CodeShield for AI agents
3. Dependency check: detect hallucinated or typosquatted packages
4. Streamlit dashboard with a live "Unprotected vs Protected" demo
5. More frameworks (Django, Next.js API routes, Supabase RLS checks)

See [docs/OWASP_COVERAGE.md](docs/OWASP_COVERAGE.md) for how this maps to the OWASP
Top 10 for LLM applications.

## 🔐 Handling your own keys

Copy `.env.example` to `.env`, add your keys, and never commit `.env` (it is git-ignored).
If a key is ever committed by mistake, revoke and regenerate it immediately: deleting the
file does not remove it from git history.

## 👥 Team

| Name | Role |
|---|---|
| Mayank SOni | indie Developer |



