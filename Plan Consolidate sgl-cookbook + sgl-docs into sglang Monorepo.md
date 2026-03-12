# Plan: Consolidate sgl-cookbook + sgl-docs into sglang Monorepo

## Context

Three repos serve SGLang documentation today:
- **sgl-project/sglang** — main repo, Sphinx docs at `docs/`, deployed to **docs.sglang.io** via GitHub Pages
- **sgl-project/sgl-docs** — Mintlify docs at `lmsysorg.mintlify.app`, has TWO tabs: "Docs" (full reference) + "Cookbook" (model guides)
- **sgl-project/sgl-cookbook** — Docusaurus cookbook at cookbook.sglang.io (being deprecated)

**Key insight:** sgl-docs already has the full Sphinx content converted to Mintlify MDX. No conversion work needed — just move it into sglang.

**Goal:** Move sgl-docs into `sglang/docs/`, replace Sphinx, preserve Claude skills, preserve contributor credit from both community repos via git subtree merges, ensure CI/CD has zero breakage, and make the contributor workflow smooth.

---

## Part 1: Understanding the Mintlify Repo (sgl-docs)

This section explains the sgl-docs architecture so contributors and PMs can understand the system.

### Site Architecture: Two-Tab Design

```
┌─────────────────────────────────────────────────────┐
│  SGLang Documentation          [Docs] [Cookbook]     │
├─────────────────────────────────────────────────────┤
│                                                     │
│  "Docs" Tab (Reference)        "Cookbook" Tab        │
│  ├── Get Started               ├── Autoregressive   │
│  ├── Basic Usage               │   ├── Qwen (7)     │
│  ├── Advanced Features (24)    │   ├── DeepSeek (6) │
│  ├── SGLang Diffusion          │   ├── Llama (3)    │
│  ├── Hardware Platforms        │   ├── GLM (9)      │
│  ├── Supported Models          │   └── ... (18 vendors) │
│  ├── Developer Guide           ├── Diffusion        │
│  └── References                │   ├── FLUX, Wan... │
│                                ├── Omni (FishAudio) │
│                                ├── SpecBundle       │
│                                └── Benchmarks       │
└─────────────────────────────────────────────────────┘
```

**"Docs" tab** = Reference documentation (how SGLang works, APIs, configuration)
**"Cookbook" tab** = Practical deployment recipes (how to run Model X on Hardware Y)

### File Organization

```
sgl-docs/
├── docs.json              ← THE source of truth: nav structure, theme, branding
├── index.mdx              ← Homepage (blog cards, feature highlights)
├── custom.css             ← Table/scrollbar styling
├── favicon.png, logo/, fonts/  ← Branding assets
│
├── docs/                  ← "Docs" tab content
│   ├── get-started/       │   installation.mdx, quickstart.mdx
│   ├── basic_usage/       │   OpenAI APIs, Ollama, native API...
│   ├── advanced_features/ │   24 pages: quantization, LoRA, speculative decoding...
│   ├── hardware-platforms/│   AMD, TPU, CPU, Ascend NPU, XPU
│   ├── supported-models/  │   Text gen, retrieval, specialized
│   ├── developer-guide/   │   Contribution, Docker, benchmarking
│   └── references/        │   FAQ, env vars, production metrics
│
├── cookbook/               ← "Cookbook" tab content
│   ├── autoregressive/    │   18 vendor folders, 50+ model guides
│   ├── diffusion/         │   FLUX, Wan, Qwen-Image, Z-Image, MOVA
│   ├── omni/              │   FishAudio S2-Pro (TTS)
│   └── base/benchmarks/   │   Benchmark guides
│
├── src/snippets/          ← Interactive deployment config components (JSX)
│   ├── autoregressive/    │   50+ *-deployment.jsx files
│   └── diffusion/         │   wan22-deployment.jsx, mova-deployment.jsx...
│
├── cards/                 ← Card images for overview pages
│   └── logos/             │   Vendor logos
│
├── scripts/               ← Automation
│   └── update_lmsys_sglang_blogs.py  ← Blog sync (cron every 12h)
├── src/generated/         ← Auto-generated blog data JSON
│
└── AGENTS.md              ← Claude agent instructions (accuracy rules, conventions)
```

### How docs.json Controls Everything

`docs.json` is the single config file that defines:
- **Navigation** — Two tabs, groups within each tab, page ordering
- **Branding** — Colors (#d55816 orange), fonts (Approach), logo
- **Theme** — "aspen", dark/light modes, grid background
- **SEO** — Google verification, meta descriptions
- **Contextual options** — Copy, Claude, ChatGPT, Cursor, VSCode integration
- **Footer** — GitHub, Slack, Discord, X links
- **Redirects** — URL mapping (currently empty)

Every new page MUST be added to docs.json or it won't appear in navigation.

### How a Cookbook Page Works

Each model deployment guide has two parts:

**1. MDX doc file** (`cookbook/autoregressive/Qwen/Qwen3.mdx`):
```mdx
---
title: Qwen3
metatags:
    description: "Deploy Qwen3 with SGLang..."
tag: NEW
---

import { Qwen3Deployment } from '/src/snippets/autoregressive/qwen3-deployment'

## 1. Model Introduction
[Description, features, links...]

## 2. Deployment
<Qwen3Deployment />         ← Interactive config generator renders here

## 3. Model Invocation
[curl examples, Python client code...]
```

**2. JSX deployment snippet** (`src/snippets/autoregressive/qwen3-deployment.jsx`):
```jsx
export const Qwen3Deployment = () => {
  // Config options (hardware, model size, quantization)
  const options = { ... };
  // State management, dark mode detection
  const [values, setValues] = useState(...);
  // Generate sglang serve command from selections
  const generateCommand = () => { ... };
  // Render radio buttons + command output
  return (<div>...</div>);
};
```

The snippet is a **self-contained React component** with inline styles and dark mode support. No external dependencies. Users click options and see the exact `sglang serve` command to run.

### Category Differences: Autoregressive vs Diffusion vs Omni

The three model categories have different maturity levels, serving infrastructure, and documentation patterns:

```
                    Autoregressive          Diffusion               Omni
────────────────────────────────────────────────────────────────────────────
Maturity            Mature (50+ models)     Established (5 vendors) Nascent (1 model)
Serving command     sglang serve            sglang serve            python -m sglang_omni.cli.cli serve
Runtime             Core SGLang             Core SGLang             sglang-omni (separate project)
Installation        pip install sglang      pip install sglang      Docker + sglang-omni package
Parallelism knobs   --tp, --dp, --ep        --num-gpus,             Config YAML file
                                            --ring-degree,
                                            --ulysses-degree
JSX snippets        50+ (all models)        4 (FLUX, Wan, etc.)     0 (none yet)
Docs tab section    No (implicit)           Yes ("SGLang Diffusion") No (not yet)
Cookbook tab         18 vendor groups        5 vendor groups         1 vendor (FishAudio)
```

**What this means for the /add-model skill:**

The skill must ask which category upfront and branch accordingly:

| Step | Autoregressive | Diffusion | Omni |
|---|---|---|---|
| Template MDX | Standard (Model Intro → Deployment → Invocation) | Standard + Cache/Optimization section | Standard + Installation (Docker) + Architecture |
| JSX snippet | Hardware + Model Size + Quantization selectors | Hardware + Resolution selectors | TBD (no pattern yet — create when 2nd omni model added) |
| Server command | `sglang serve --model-path X --tp Y` | `sglang serve --model-path X --num-gpus Y --ring-degree Z` | `python -m sglang_omni.cli.cli serve --model-path X --config Y` |
| docs.json location | Cookbook → Autoregressive Models → Vendor | Cookbook → Diffusion Models → Vendor | Cookbook → SGLang Omni → Vendor |

**Future growth pattern:**

As Omni matures (more TTS models, audio models, multimodal):
1. It will need JSX deployment snippets (create pattern when 2nd model arrives)
2. It will need a "Docs tab" section like Diffusion has ("SGLang Omni" reference docs)
3. The /add-model skill will need an omni-specific template

For NOW: the /add-model skill handles autoregressive and diffusion. Omni models are rare enough to be added manually following the S2-Pro pattern.

### Deployment Flow

```
Contributor pushes to main → Mintlify cloud detects change → Auto-builds → Deploys
```

No CI/CD workflow needed for deployment. Mintlify handles it entirely. The only workflow is the blog sync cron job.

---

## Part 2: Contributor Submission & Review Workflow

### Current State (3 repos = confusion)

| Action | Where to submit? | Review by? |
|---|---|---|
| Fix a typo in API docs | sgl-docs (Mintlify) | Informal |
| Add new model guide | sgl-cookbook (Docusaurus) | Informal |
| Fix Python code | sglang (main repo) | CODEOWNERS + CI |
| Update hardware guide | sgl-docs (Mintlify) | Informal |

**Problem:** Contributors don't know which repo to use. Cookbook has Claude skills but is being deprecated. sgl-docs has the content but no governance.

### Future State (1 repo = clarity)

| Action | Where | Path | Reviewed by |
|---|---|---|---|
| Fix a typo in API docs | sglang PR | `docs/docs/` | @docs-maintainers |
| Add new model guide | sglang PR | `docs/cookbook/` | @cookbook-maintainers |
| Fix Python code | sglang PR | `python/sglang/` | CODEOWNERS (kernel devs) |
| Update hardware guide | sglang PR | `docs/docs/hardware-platforms/` | @docs-maintainers |

**Everything in one place.** Clear ownership via CODEOWNERS. No confusion.

### Step-by-Step: Adding a New Model

```
1. Fork sgl-project/sglang (shallow clone OK: git clone --depth 1)
2. Create branch: git checkout -b add-model-xyz

3. Use /add-model Claude skill OR manually create:
   a. docs/cookbook/<category>/<Vendor>/<Model>.mdx    ← deployment guide
   b. docs/src/snippets/<category>/<name>-deployment.jsx  ← config generator
   c. Update docs/docs.json → Cookbook tab navigation

4. Test locally:
   cd docs && npx mintlify dev     # Preview at localhost:3000

5. Commit and push to fork

6. Create PR to sgl-project/sglang
   → Auto-labeled: "documentation" + "cookbook"
   → Mintlify validation runs (broken-links, build) — NO GPU CI
   → CODEOWNERS requests review from @cookbook-maintainers

7. Reviewer checks (using /review-pr skill):
   - Autoregressive/Diffusion: uses sglang serve (not deprecated launch_server)
   - Omni: uses correct sglang-omni command + Docker setup documented
   - Port consistency (30000 for autoregressive, 30002 for diffusion, 8000 for omni)
   - AMD GPU memory specs correct (if applicable)
   - docs.json updated under correct category group
   - Deployment snippet has named export (if snippet exists)

8. Merge → Mintlify auto-deploys
```

### Step-by-Step: Editing Reference Docs

```
1. Fork (or click "Edit on GitHub" pencil icon on any page)
2. Edit MDX file in docs/docs/
3. If new page: add to docs/docs.json
4. PR → auto-labeled "documentation" → reviewed by @docs-maintainers → merge
```

### Review Standards

From AGENTS.md (non-negotiable rules for docs):
- **Never guess** flags, defaults, or behavior — verify against sglang source code
- **Always specify** platform (NVIDIA/AMD/TPU/etc.), model identifier, parallelism knobs
- **Keep examples copy-pasteable** with consistent placeholders
- **Every page needs** `title` and `description` in frontmatter
- **Use active voice**, second person ("you"), short scannable sections
- **Verify links** with `mintlify broken-links`

### Governance Integration

To make docs contributions smooth in the monorepo:

**1. CODEOWNERS** (`.github/CODEOWNERS`):
```
docs/docs.json                    @docs-maintainers
docs/cookbook/                     @cookbook-maintainers
docs/src/snippets/                @cookbook-maintainers
docs/docs/                        @docs-maintainers
```

**2. Auto-labeler** (`.github/labeler.yml`):
```yaml
cookbook:
  - docs/cookbook/**
  - docs/src/snippets/**
```

**3. PR template exemption**:
```markdown
<!-- For documentation-only PRs (docs/ changes), you can skip
     Accuracy Tests and Benchmarking. Just describe your changes. -->
```

**4. CI path filtering** (CRITICAL):
- `execute-notebook.yml` trigger: `docs-legacy/**` (NOT `docs/**`)
- Docs PRs never trigger GPU CI — only lightweight Mintlify validation

---

## Part 3: Migration Plan

### Contributor Credit Preservation Strategy

**Why this matters:** sgl-cookbook has 180 commits from 32 contributors and sgl-docs has 75 commits from 15 human contributors. Many are open source community members whose GitHub profiles and `git blame` attribution depend on their commits being in the active repo's history. A flat copy erases all of this.

**Approach: Git subtree merges + CONTRIBUTORS.md + archived repos**

| Method | What it preserves | Applies to |
|---|---|---|
| `git subtree add` (no squash) | Full commit history, `git blame`, `git log`, GitHub contributor graph | sgl-docs (active content) |
| `git subtree add` (no squash) + remove files | Commit history in `git log`, GitHub contributor graph (not `git blame` — files removed) | sgl-cookbook (deprecated content, different format) |
| `docs/CONTRIBUTORS.md` | Human-readable acknowledgment with GitHub links | Both repos |
| Archived public repos | Complete original history, issues, PRs | Both repos |

**sgl-docs contributors (15 human):**
Richardczl98 (11), adhyan-jain (11), AdityaVKochar (6), Maitri-shah29 (5), adarshxs (4), Krishang-Zinzuwadia (4), richardchenczl (3), pokymono (3), wisclmy0611 (3), divyamagrawal06 (3), IshhanKheria (2), nimeshas (2), Ishitajoshii (1), JignasP (1), Nakul-Sinha (1)

**sgl-cookbook contributors (32):**
NLRX-WJC (42), Richardczl98 (20), ChangLiu0709 (16), GoldenGrapeGentleman (15), JustinTong0323 (14), Fridge003 (10), jiapingW (8), longGGGGGG (8), indianspeedster (7), Kangyan-Zhou (6), haic0 (4), Qiaolin-Yu (4), ispobock (4), Mahdi-CV (2), zijiexia (2), leejnau (2), zhaochenyang20 (1), jiacao-amd (1), danielafrimi (1), ajith-sirra-amd (1), zhenghax (1), venkywonka (1), wenscarl (1), narain1 (1), mmangkad (1), mickqian (1), michaelzhang-ai (1), kedarpotdar-nv (1), kartikx (1), sodacorsair (1), dougyster (1), ch-wan (1)

### Target Layout in sglang Repo

```
sglang/
├── docs/                    # Mintlify content root (subtree-merged from sgl-docs)
│   ├── docs.json            # Nav + theme config
│   ├── index.mdx            # Homepage
│   ├── custom.css, favicon.png, fonts/, logo/
│   ├── docs/                # "Docs" tab
│   ├── cookbook/             # "Cookbook" tab
│   ├── sglang-diffusion/
│   ├── src/snippets/        # JSX config generators
│   ├── src/generated/       # Auto-generated blog data
│   ├── cards/               # Card images
│   ├── scripts/             # Blog sync script
│   └── CONTRIBUTORS.md      # Credit for sgl-docs + sgl-cookbook contributors
├── mint.json                # Monorepo config: {"mintlifyDir": "./docs"}
├── docs-legacy/             # Jupyter notebooks (kept for CI)
├── tools/                   # release_lookup, performance_dashboard
├── .claude/
│   ├── AGENTS.md            # From sgl-docs
│   └── skills/
│       ├── add-jit-kernel/  # EXISTING (unchanged)
│       ├── add-sgl-kernel/  # EXISTING (unchanged)
│       ├── sglang-bisect-ci-regression/  # EXISTING (unchanged)
│       ├── write-sglang-test/  # EXISTING (unchanged)
│       └── cookbook/         # NEW subfolder for docs skills
│           ├── add-model/   # NEW (adapted from sgl-cookbook)
│           └── review-pr/   # NEW (adapted from sgl-cookbook)
└── python/sglang/           # Unchanged
```

### Phase 1: Fork & Branch

1. Fork `sgl-project/sglang`, create branch `consolidate-docs`

### Phase 2: Clear Sphinx, Preserve Notebooks

1. Move `.ipynb` files from `docs/` → `docs-legacy/` (preserving structure)
2. Move `docs/Makefile` → `docs-legacy/Makefile`
3. Move `docs/requirements.txt` → `docs-legacy/requirements.txt` (needed by notebook CI)
4. Move `docs/release_lookup/` → `tools/release_lookup/`
5. Move `docs/performance_dashboard/` → `tools/performance_dashboard/`
6. Remove everything remaining in `docs/`: `conf.py`, `make.bat`, `serve.sh`, `deploy.py`, `_static/`, `_templates/`, `index.rst`, all `.rst`, all `.md`, `wrap_run_llm.py`
7. **Remove the `docs/` directory entirely** — `git subtree add` requires the prefix directory to not exist
8. Commit: `chore: clear Sphinx docs, preserve notebooks in docs-legacy/`

### Phase 3: Subtree-Merge sgl-docs (preserves full contributor history)

This replaces the previous "copy" approach. Every commit from sgl-docs lands in sglang's git history with the original author, date, and message.

```bash
# Add sgl-docs as a remote
git remote add sgl-docs https://github.com/sgl-project/sgl-docs.git
git fetch sgl-docs

# Subtree-merge into docs/ — full history, no squash
git subtree add --prefix=docs sgl-docs main

# Verify: git log --oneline docs/ should show all 75 sgl-docs commits
# Verify: git blame docs/cookbook/autoregressive/Qwen/Qwen3.mdx shows original authors
```

**What this gives us:**
- `git blame` on any file in `docs/` shows the original author and date
- `git log docs/` shows the full sgl-docs commit history
- GitHub contributor graph for sglang includes all 15 sgl-docs contributors
- Contributors' GitHub profiles reflect their work in the active monorepo

**Post-merge cleanup** (separate commit): The subtree merge brings in ALL tracked files from sgl-docs, including files we don't want in `docs/`. Clean up in a follow-up commit:

```bash
# Move AGENTS.md to its correct location
git mv docs/AGENTS.md .claude/AGENTS.md

# Remove files that belong to sgl-docs repo structure, not our docs/
git rm docs/README.md docs/LICENSE docs/CONTRIBUTING.md
git rm -r docs/.github/    # sgl-docs workflows (not needed, GitHub only reads root .github/)

# Add CONTRIBUTORS.md to .mintignore so Mintlify doesn't try to render it
echo "CONTRIBUTORS.md" >> docs/.mintignore

git commit -m "chore: clean up subtree artifacts, move AGENTS.md to .claude/"
```

Note: `.git/` is the only thing excluded automatically by `git subtree add`. All other tracked files (README, LICENSE, .github/, etc.) are included and must be removed manually.

### Phase 3b: Subtree-Merge sgl-cookbook (preserves contributor history in git log)

sgl-cookbook content is Docusaurus-formatted and already superseded by the Mintlify versions in sgl-docs. We merge it to a temporary prefix and then remove the files. This preserves all 180 commits and 32 contributors in `git log` and the GitHub contributor graph, even though the files themselves are removed.

```bash
# Add sgl-cookbook as a remote
git remote add sgl-cookbook https://github.com/sgl-project/sgl-cookbook.git
git fetch sgl-cookbook

# Subtree-merge into a temporary prefix — full history, no squash
git subtree add --prefix=cookbook-archive sgl-cookbook main

# Remove the archived files (history stays in git log)
git rm -r cookbook-archive/
git commit -m "chore: remove cookbook-archive files (history preserved in git log)

The sgl-cookbook content has been superseded by Mintlify versions in docs/cookbook/.
This subtree merge preserved all 180 commits from 32 contributors in git history.
The archived files are removed but full attribution is retained via git log."
```

**What this gives us:**
- `git log` shows all 180 sgl-cookbook commits with original authors
- GitHub contributor graph for sglang includes all 32 sgl-cookbook contributors
- Contributors' GitHub profiles reflect their work
- No stale Docusaurus files cluttering the repo (removed after merge)

**Limitation:** `git blame` won't work on sgl-cookbook files (they're deleted). For per-line attribution, refer to the archived repo. This is acceptable since the Mintlify versions in `docs/cookbook/` are rewrites, not 1:1 copies.

### Phase 3c: Create CONTRIBUTORS.md

Create `docs/CONTRIBUTORS.md` as a human-readable acknowledgment:

```markdown
# Documentation Contributors

The SGLang documentation was consolidated from two community-maintained
repositories into this monorepo. The following people contributed to the
original projects. Full commit history is preserved in this repo's git log
via subtree merges.

## From sgl-project/sgl-docs (Mintlify documentation)

| Contributor | Contributions |
|---|---|
| [@Richardczl98](https://github.com/Richardczl98) | 11 |
| [@adhyan-jain](https://github.com/adhyan-jain) | 11 |
| [@AdityaVKochar](https://github.com/AdityaVKochar) | 6 |
| [@Maitri-shah29](https://github.com/Maitri-shah29) | 5 |
| [@adarshxs](https://github.com/adarshxs) | 4 |
| [@Krishang-Zinzuwadia](https://github.com/Krishang-Zinzuwadia) | 4 |
| [@richardchenczl](https://github.com/richardchenczl) | 3 |
| [@pokymono](https://github.com/pokymono) | 3 |
| [@wisclmy0611](https://github.com/wisclmy0611) | 3 |
| [@divyamagrawal06](https://github.com/divyamagrawal06) | 3 |
| [@IshhanKheria](https://github.com/IshhanKheria) | 2 |
| [@nimeshas](https://github.com/nimeshas) | 2 |
| [@Ishitajoshii](https://github.com/Ishitajoshii) | 1 |
| [@JignasP](https://github.com/JignasP) | 1 |
| [@Nakul-Sinha](https://github.com/Nakul-Sinha) | 1 |

## From sgl-project/sgl-cookbook (Docusaurus cookbook, now superseded)

| Contributor | Contributions |
|---|---|
| [@NLRX-WJC](https://github.com/NLRX-WJC) | 42 |
| [@Richardczl98](https://github.com/Richardczl98) | 20 |
| [@ChangLiu0709](https://github.com/ChangLiu0709) | 16 |
| [@GoldenGrapeGentleman](https://github.com/GoldenGrapeGentleman) | 15 |
| [@JustinTong0323](https://github.com/JustinTong0323) | 14 |
| [@Fridge003](https://github.com/Fridge003) | 10 |
| [@jiapingW](https://github.com/jiapingW) | 8 |
| [@longGGGGGG](https://github.com/longGGGGGG) | 8 |
| [@indianspeedster](https://github.com/indianspeedster) | 7 |
| [@Kangyan-Zhou](https://github.com/Kangyan-Zhou) | 6 |
| [@haic0](https://github.com/haic0) | 4 |
| [@Qiaolin-Yu](https://github.com/Qiaolin-Yu) | 4 |
| [@ispobock](https://github.com/ispobock) | 4 |
| [@Mahdi-CV](https://github.com/Mahdi-CV) | 2 |
| [@zijiexia](https://github.com/zijiexia) | 2 |
| [@leejnau](https://github.com/leejnau) | 2 |
| [@zhaochenyang20](https://github.com/zhaochenyang20) | 1 |
| [@jiacao-amd](https://github.com/jiacao-amd) | 1 |
| [@danielafrimi](https://github.com/danielafrimi) | 1 |
| [@ajith-sirra-amd](https://github.com/ajith-sirra-amd) | 1 |
| [@zhenghax](https://github.com/zhenghax) | 1 |
| [@venkywonka](https://github.com/venkywonka) | 1 |
| [@wenscarl](https://github.com/wenscarl) | 1 |
| [@narain1](https://github.com/narain1) | 1 |
| [@mmangkad](https://github.com/mmangkad) | 1 |
| [@mickqian](https://github.com/mickqian) | 1 |
| [@michaelzhang-ai](https://github.com/michaelzhang-ai) | 1 |
| [@kedarpotdar-nv](https://github.com/kedarpotdar-nv) | 1 |
| [@kartikx](https://github.com/kartikx) | 1 |
| [@sodacorsair](https://github.com/sodacorsair) | 1 |
| [@dougyster](https://github.com/dougyster) | 1 |
| [@ch-wan](https://github.com/ch-wan) | 1 |

## Archived Repositories

Original repositories with complete history, issues, and pull requests:
- [sgl-project/sgl-docs](https://github.com/sgl-project/sgl-docs) (archived)
- [sgl-project/sgl-cookbook](https://github.com/sgl-project/sgl-cookbook) (archived)
```

### Phase 4: Monorepo Config

Create `mint.json` at repo root:
```json
{ "mintlifyDir": "./docs" }
```

### Phase 5: CI/CD Updates

**5a. `release-docs.yml`** — Rewrite entirely:
- Runner: `ubuntu-latest` (was `1-gpu-runner`)
- Steps: `npx mintlify validate --directory docs` + `npx mintlify broken-links --directory docs`
- Remove: Sphinx build, notebook execution, force-push to sgl-project.github.io
- Remove triggers for `python/sglang/version.py` and `python/sglang/**`

**5b. `execute-notebook.yml`** — Update trigger path:
- Change `docs/**` → `docs-legacy/**` in paths filter
- Change `cd docs` → `cd docs-legacy` in run steps

**5c. Add `sync-lmsys-sglang-blogs.yml`** — Copy from sgl-docs, update paths:
- Script: `docs/scripts/update_lmsys_sglang_blogs.py`
- Outputs: `docs/index.mdx`, `docs/src/generated/lmsys_sglang_blogs.json`

**5d. sgl-project.github.io** — After Mintlify custom domain set to docs.sglang.io:
- Push redirect `index.html` to sgl-project.github.io

### Phase 6: Skills & Governance

- ADD `cookbook/add-model/SKILL.md` and `cookbook/review-pr/SKILL.md` to `.claude/skills/` (do NOT touch 4 existing skills)
- Update `.github/CODEOWNERS` with docs/ entries
- Update `.github/labeler.yml` with `cookbook` label
- Add docs-only exemption to PR template

### Phase 7: Deprecate (post-merge)

1. sgl-cookbook: deprecation notice + archive
2. sgl-docs: deprecation notice + archive
3. Domain: configure docs.sglang.io → Mintlify custom domain in dashboard
4. Redirect: cookbook.sglang.io → docs.sglang.io/cookbook

---

## Part 4: CI/CD Bug Prevention

| # | Bug | Fix |
|---|---|---|
| 1 | `execute-notebook.yml` triggers GPU CI on every MDX edit | Change trigger from `docs/**` to `docs-legacy/**` |
| 2 | `release-docs.yml` tries to run Sphinx on GPU runner | Rewrite for Mintlify validation on `ubuntu-latest` |
| 3 | Mintlify can't find config | `mint.json` at repo root with `"mintlifyDir": "./docs"`, `docs.json` inside `docs/` |
| 4 | Blog sync writes to wrong paths | Update workflow paths to `docs/scripts/`, `docs/index.mdx` |
| 5 | Overwrite existing Claude skills | ADD new skills, never touch existing 4 |
| 6 | sgl-project.github.io serves stale Sphinx | Set up redirect BEFORE removing Sphinx deployment |
| 7 | Missing assets (fonts/CSS/logo) | Subtree merge brings ALL sgl-docs assets into `docs/` automatically |
| 8 | Cookbook PRs blocked by kernel CODEOWNERS | Separate CODEOWNERS entries for `docs/` |
| 9 | Pre-commit hooks reject MDX | Verify pre-commit scope targets Python/JS only |
| 10 | docs-legacy Makefile can't find notebooks | Update Makefile paths for new directory |
| 11 | Flat copy loses contributor credit from sgl-docs + sgl-cookbook | Use `git subtree add` (no squash) to preserve full commit history and attribution |
| 12 | `git subtree add` fails because `docs/` directory exists | Phase 2 must fully remove `docs/` directory before subtree merge |
| 13 | `docs-legacy/requirements.txt` deleted during Sphinx cleanup | Phase 2 step 3: move (not delete) `requirements.txt` to `docs-legacy/` |
| 14 | Subtree merge brings unwanted files (README.md, LICENSE, .github/) into `docs/` | Post-merge cleanup commit removes these artifacts |
| 15 | `docs/CONTRIBUTORS.md` rendered by Mintlify as a doc page | Add `CONTRIBUTORS.md` to `docs/.mintignore` |

---

## Part 5: Verification Checklist

### Docs Build
- [ ] `cd docs && npx mintlify dev` renders at localhost:3000
- [ ] Both "Docs" and "Cookbook" tabs work
- [ ] `npx mintlify broken-links --directory docs` — zero broken links
- [ ] JSX deployment snippets render interactive config generators
- [ ] Homepage loads with blog cards

### CI/CD
- [ ] `execute-notebook.yml` trigger = `docs-legacy/**`
- [ ] `release-docs.yml` runner = `ubuntu-latest`, no Sphinx
- [ ] `sync-lmsys-sglang-blogs.yml` paths correct
- [ ] Grep all workflows for `sphinx`, `make html`, `conf.py` — zero matches

### Contributor Workflow
- [ ] CODEOWNERS has `docs/cookbook/` and `docs/docs/` entries
- [ ] labeler.yml has `cookbook` label
- [ ] PR template has docs-only exemption
- [ ] 6 total Claude skills (4 existing + 2 new)
- [ ] `/add-model` paths: `docs/cookbook/`, `docs/src/snippets/`, `docs/docs.json`

### Contributor Credit
- [ ] `git log --oneline docs/` shows sgl-docs commits with original authors (not just merge commit)
- [ ] `git blame docs/cookbook/autoregressive/Qwen/Qwen3.mdx` shows original author (not consolidation committer)
- [ ] `git log --all --oneline | grep "sgl-cookbook"` shows sgl-cookbook history present
- [ ] `docs/CONTRIBUTORS.md` lists all 15 sgl-docs + 32 sgl-cookbook contributors
- [ ] CONTRIBUTORS.md links to archived repos

### Subtree Cleanup
- [ ] `docs/README.md`, `docs/LICENSE`, `docs/CONTRIBUTING.md` do NOT exist (removed in cleanup)
- [ ] `docs/.github/` does NOT exist (removed in cleanup)
- [ ] `docs/AGENTS.md` does NOT exist (moved to `.claude/AGENTS.md`)
- [ ] `CONTRIBUTORS.md` is listed in `docs/.mintignore`

### Content
- [ ] `docs/docs.json` matches sgl-docs nav exactly
- [ ] All cards, fonts, logo, CSS present
- [ ] `.claude/AGENTS.md` present (moved from subtree, not copied from source)
- [ ] `tools/release_lookup/` and `tools/performance_dashboard/` present

---

## Part 6: First Attempt Post-Mortem (2026-03-10)

### What happened

PR #20308 was created using a flat copy approach (no history preservation). It was closed without merging due to merge conflicts and the contributor credit gap identified below.

**PR #20308**: https://github.com/sgl-project/sglang/pull/20308
**Status**: Closed (not merged) — will be superseded by new PR using subtree merge approach

### Bugs found and lessons learned

**Bug A (CRITICAL): `release-docs.yml` uses non-existent `mintlify build` command**
- The Mintlify CLI does NOT have a `build` command
- Correct command: `mintlify validate` (strict mode, exits on warnings/errors)
- **Lesson**: Incorporated into Phase 5a — use `mintlify validate`, not `mintlify build`

**Bug B (CRITICAL): `docs-legacy/requirements.txt` missing**
- `execute-notebook.yml` line 40: `pip install -r docs-legacy/requirements.txt`
- The file was deleted during Sphinx cleanup
- **Lesson**: Incorporated into Phase 2 step 3 — move (not delete) `requirements.txt`

**Bug C (PRE-EXISTING, not our fault): 3 nav warnings + 14 broken links**
- Confirmed identical in sgl-docs source repo — these exist upstream
- Missing nav pages: `docs/sglang-diffusion/performance/profiling`, `docs/sglang-diffusion/support-new-models`, `docs/sglang-diffusion/contributing`
- 14 broken internal links in sglang-diffusion MDX files
- NOT blocking for our migration — upstream issue

**Bug D (DESIGN): Flat copy erases contributor credit**
- sgl-cookbook has 32 contributors (180 commits) and sgl-docs has 15 human contributors (75 commits)
- Many are open source community members whose GitHub profiles depend on commit attribution
- A flat copy shows only the consolidation committer in `git blame` and contributor graphs
- **Lesson**: Incorporated as Phase 3/3b — use `git subtree add` (no squash) to preserve full history

### Verification results from first attempt (kept for reference)

| Check | Status | Notes |
|---|---|---|
| `mintlify validate` in docs/ | PASS (3 pre-existing warnings) | Same as sgl-docs source |
| `mintlify broken-links` in docs/ | 14 pre-existing | Same as sgl-docs source |
| `execute-notebook.yml` trigger = `docs-legacy/**` | PASS | |
| `release-docs.yml` runner = `ubuntu-latest` | PASS | |
| Zero Sphinx refs in workflows | PASS | |
| 6 Claude skills (4 existing + 2 new) | PASS | |
| All sgl-docs assets present | PASS | 183 MDX, 43 JSX, fonts, logo, cards, CSS |

These checks remain valid — the subtree approach changes how content arrives, not the final file layout.

---

## Resolved Decisions

1. **Domain**: Mintlify currently at `lmsysorg.mintlify.app`. Post-migration: configure custom domain `docs.sglang.io` in Mintlify dashboard.
2. **Standalone tools**: Move `release_lookup` + `performance_dashboard` to `tools/`.
3. **Notebook CI**: Keep in `docs-legacy/` as integration tests.
4. **Skills organization**: Cookbook skills under `.claude/skills/cookbook/`, core sglang skills at `.claude/skills/` root.
5. **Contributor credit**: Use `git subtree add` (no squash) for both sgl-docs and sgl-cookbook to preserve full commit history and attribution. Supplement with `docs/CONTRIBUTORS.md` and keep archived repos public.

## Post-Merge Steps

1. Configure `docs.sglang.io` custom domain in Mintlify dashboard (point to `sgl-project/sglang`, content dir `docs/`)
2. Set up `cookbook.sglang.io` → `docs.sglang.io/cookbook` redirect
3. Push redirect page to `sgl-project.github.io`
4. Archive `sgl-docs` and `sgl-cookbook` repos with deprecation notices

## Next Steps

1. Create new branch `consolidate-docs-v2` from latest `main`
2. Execute Phases 2 → 6 using subtree merge approach (Phase 3/3b/3c)
3. Open new PR superseding #20308
4. Verify contributor credit checks pass (Part 5 checklist)
