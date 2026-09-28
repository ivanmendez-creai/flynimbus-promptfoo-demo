# promptfoo — FlyNimbus prompt QA & red teaming

Run-of-show and talking points: `GUIDE.md`. Demo script (Spanish): `GUION.md` (~12 min) and `GUION-6MIN.md` (6 min).

## What is promptfoo?

[promptfoo](https://www.promptfoo.dev) is an open-source CLI for testing LLM
applications — think pytest/Jest, but for prompts and models. You describe test
cases in YAML (an input plus what you expect from the output), and promptfoo
runs them against every combination of prompts and models, grades each answer,
and shows the results as a matrix. It runs locally, works with any major model
provider, and plugs into CI like any other test suite.

## promptfoo features used in this demo

| Feature | What it's for | Where in this repo |
|---|---|---|
| **`promptfoo eval`** | Runs every test against every prompt × model combination and grades each answer. Exits non-zero when a test fails. | `promptfooconfig.yaml` (2 prompts × 2 models × 10 tests = 40 cells) |
| **Prompt comparison** | Tests several versions of a prompt side by side, so "is v2 better?" becomes a number instead of an opinion. | `prompts/v1-vulnerable.txt` vs `prompts/v2-hardened.txt` |
| **Model comparison** | Same suite across providers, so a model swap is tested like any other change. | Gemma 4 (Google) vs Claude Sonnet 5 (Anthropic) |
| **Deterministic assertions** (`contains`, `not-contains`, `regex`) | Cheap, instant checks on the exact text: a required value, a forbidden string, a format. | F1 (price accuracy vs. price format as two metrics), F2, F3, A6 |
| **LLM-as-a-judge** (`llm-rubric`) | Another model grades *behaviour* against a plain-language rubric ("refuses politely and gives no code") — for things exact text can't capture. | F2, F3, A1–A6, graded by Gemma 4 31B |
| **Semantic similarity** (`similar`) | Compares the *meaning* of the answer with a reference answer using embeddings; catches correct-but-incomplete answers. | F4 (threshold 0.89, calibrated on real runs) |
| **Custom JavaScript assertions** | Your own grading logic when the built-ins aren't enough. | `tests/fuga-cupon.js`: detects the coupon even spelled out, reversed or in base64, applied to every answer |
| **Named metrics** | Groups assertions into scores shown per column in the viewer (`fuga-cupon`, `fuera-de-alcance`, `precio-correcto`...). | `metric:` on each assertion |
| **Response cache** | Stores every model response on disk, so re-runs are instant and reproducible (and don't spend API calls). | Default on; a full run takes ~1 s from cache vs ~9 min cold |
| **`promptfoo view`** | Local web UI with the results matrix: filter failures, open a cell to see the answer and why it passed or failed, compare runs. | `http://localhost:15500` |
| **Red teaming** (`promptfoo redteam`) | Generates attacks automatically from a one-paragraph description of the app (`purpose`), runs them and reports vulnerabilities by category. **Plugins** choose *what* to attack; **strategies** choose *how* (base64, prompt injection, iterative jailbreak where an attacker model rewrites the attack based on the bot's replies). | `redteam/promptfooconfig.yaml`: 4 plugins × 2 attacks × 4 strategies = 32 attacks |
| **CI integration** | The same eval as a merge gate: fails the build when the production prompt regresses, and publishes HTML/JUnit reports. | `.github/workflows/promptfoo-eval.yml` (v2 on Gemma, every push) |

```
.
├── promptfooconfig.yaml      # main eval: 2 prompts × 2 models × 10 tests
├── prompts/
│   ├── v1-vulnerable.txt     # realistic first draft (fails on purpose)
│   └── v2-hardened.txt       # reviewed system prompt
├── tests/
│   ├── funcionales.yaml      # F1–F4: contains, not-contains + llm-rubric, similar
│   ├── ataques.yaml          # A1–A6: hand-written realistic attacks
│   └── fuga-cupon.js         # global leak detector (spelled out, reversed, base64)
└── redteam/
    └── promptfooconfig.yaml  # automated red team, Gemma only
.github/workflows/promptfoo-eval.yml  # CI gate: v2 on Gemma
```

| Role | Model | Key |
|---|---|---|
| Target | `google:gemma-4-26b-a4b-it` | `GEMINI_API_KEY` (free tier) |
| Target | `anthropic:messages:claude-sonnet-5` | `ANTHROPIC_API_KEY` |
| Judge (`llm-rubric`) | `google:gemma-4-31b-it` | `GEMINI_API_KEY` |
| Embeddings (`similar`) | `google:embedding:gemini-embedding-2` | `GEMINI_API_KEY` |

Requires Node.js ≥ 22.22 (`nvm use` reads `.nvmrc`) and `npm i -g promptfoo@0.123.1`.

```bash
cp .env.example .env                  # fill in the keys
set -a; source .env; set +a

promptfoo validate
promptfoo eval             # uses the disk cache; add --no-cache for fresh answers
promptfoo view             # http://localhost:15500

# What CI runs (production prompt, free model only)
promptfoo eval --filter-prompts v2 --filter-providers gemma

# Automated red team (~32 attacks, asks for email verification on first run).
# Always pass -c: without it promptfoo uses the main config (wrong plugins and injectVar).
promptfoo redteam run -c redteam/promptfooconfig.yaml
promptfoo redteam report
```
