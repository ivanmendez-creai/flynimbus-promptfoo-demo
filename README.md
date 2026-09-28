# promptfoo — FlyNimbus prompt QA & red teaming

Run-of-show and talking points: `GUIDE.md`. Demo script (Spanish): `GUION.md` (~12 min) and `GUION-6MIN.md` (6 min).

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
