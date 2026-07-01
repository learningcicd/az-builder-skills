# Using the Graph Skills in GitHub Copilot — Repo-Agnostic Setup

At a high level, both skills turn a codebase into trustworthy documentation and diagrams — the emphasis being on trustworthy.

- This guide sets up the `azgraph` and `codegraph` skills as **native GitHub Copilot Agent Skills** living in your **user profile**, so Copilot can use them in any repo.

---

## Contents

2. [The two skills at a glance](#2-the-two-skills-at-a-glance)
3. [Requirements](#3-requirements)
4. [Install once — per developer](#4-install-once--per-developer)
5. [Install once — for the whole team](#5-install-once--for-the-whole-team)
6. [One-time `codegraph` dependency setup](#6-one-time-codegraph-dependency-setup)
7. [Using the skills](#7-using-the-skills)
8. [Where output goes](#8-where-output-goes)
9. [How the skills stay repo-agnostic](#9-how-the-skills-stay-repo-agnostic)
10. [Verifying, troubleshooting, updating](#10-verifying-troubleshooting-updating)

---

## 2. The two skills at a glance

| Skill | Answers | Method | Scope |
|---|---|---|---|
| **`azgraph`** | *"What Azure artefacts are here and how are they wired?"* | Static text/XML scan (Python stdlib only) | Function Apps, Pipelines, ARM/Bicep, NuGet/npm/pip deps, cross-repo links |
| **`codegraph`** | *"Which functions call which, and where?"* | Real AST parsing via tree-sitter | Function-level call graph across 12 languages, resolved + unresolved edges |

Both are grounded: every fact traces to a real file and line, unresolved references stay unresolved, and neither executes your code or calls external services.

---

## 3. Requirements

| Requirement | Version | Needed by |
|---|---|---|
| VS Code (or Visual Studio 2026) | current | both |
| GitHub Copilot + **agent mode** (paid tier) | latest | both |
| Python | 3.9+ | both |
| `tree-sitter` + language grammars | latest | `codegraph` only (installed once, see §6) |
| Mermaid preview extension (optional) | any | viewing `.mmd` diagrams |

`azgraph` needs **no pip packages** (standard library only). Only `codegraph` installs anything.

---

## 4. Install once — per developer

The skills folder is included as `copilot-graph-skills.zip`. Unzip it so that each skill sits in its **own directory** directly under the user-profile skills folder — the `SKILL.md` must be at the root of the skill folder, not nested.

**Target location (cross-agent standard, recommended):**

```
~/.copilot/skills/
├── azgraph/
│   ├── SKILL.md
│   ├── scripts/   (scan_azure.py, render_mindmap.py)
│   └── references/
└── codegraph/
    ├── SKILL.md
    ├── scripts/   (setup_env.sh, profile_repo.py, build_graph.py, merge_graphs.py, render_docs.py)
    └── references/
```

**macOS / Linux**
```bash
mkdir -p ~/.copilot/skills
unzip copilot-graph-skills.zip -d /tmp/graph-skills
cp -R /tmp/graph-skills/skills/* ~/.copilot/skills/
```

**Windows (PowerShell)**
```powershell
$dest = "$HOME\.copilot\skills"
New-Item -ItemType Directory -Force -Path $dest | Out-Null
Expand-Archive copilot-graph-skills.zip -DestinationPath $env:TEMP\graph-skills -Force
Copy-Item "$env:TEMP\graph-skills\skills\*" $dest -Recurse -Force
```

Then reload VS Code (`Ctrl+Shift+P` → *Developer: Reload Window*).

> **Alternate locations.** Copilot also discovers skills from `.github/skills/` inside a repo (for team-shared, version-controlled skills) and some builds also read `~/.config/github-copilot/skills/` (macOS/Linux) or `%APPDATA%\github-copilot\skills\` (Windows). If `~/.copilot/skills/` isn't picked up on your version, add the folder explicitly via the `chat.agentSkillsLocations` setting (see §5) or use the config-folder path. You can confirm the exact expected path in your client via *Configure Chat → Skills* (or type `/skills` in chat).

---

## 5. Install once — for the whole team

Two options so developers don't each unzip manually:

**Option A — shared skills directory via a setting.** Put the `azgraph/` and `codegraph/` folders in a location every developer can reach (a cloned "copilot-skills" repo, a synced folder, or a network share), then point Copilot at it in **user** `settings.json`:

```jsonc
{
  "chat.agentSkillsLocations": [
    "/path/to/shared/copilot-skills"   // contains azgraph/ and codegraph/
  ],
  // For monorepos where you open a subfolder rather than the repo root:
  "chat.useCustomizationsInParentRepositories": true
}
```

Updating the shared folder updates everyone — no re-copying. This setting can also be pushed through a `.vscode/settings.json` policy or your MDM/dotfiles tooling.

**Option B — commit into selected repos.** For a specific repo where you *do* want the skill version-controlled and shared through source control, drop the same folders under `.github/skills/` at that repo's root. This is the opposite of repo-agnostic, so use it only where you want the skill pinned to that repo. (Copilot Business/Enterprise admins can also standardise broader guidance at the **organization** level.)

For most "many repos" cases, **Option A is the one you want**: one shared copy, referenced everywhere, committed nowhere.

---

## 6. One-time `codegraph` dependency setup

`codegraph` needs tree-sitter grammars. Install them once into a virtual environment inside the skill folder; every future run reuses it. `azgraph` needs nothing here.

```bash
python -m venv ~/.copilot/skills/codegraph/.venv
source ~/.copilot/skills/codegraph/.venv/bin/activate      # Windows: ...\.venv\Scripts\Activate.ps1
bash ~/.copilot/skills/codegraph/scripts/setup_env.sh
```

`setup_env.sh` prints which grammars imported successfully. **Note any that failed** — files in those languages are skipped entirely (zero functions, zero calls), and that gap is disclosed in the output by design. Silent regex fallback is deliberately not done.

---

## 7. Using the skills

You don't reference a skill by name — you describe the task in **agent mode** and Copilot matches it against the skill descriptions, loads the right one, and runs its workflow (approving the read-only scan when prompted). When a skill activates, it shows in the chat so you know it's applied.

**Triggers `azgraph`:** "map the Azure architecture of this repo" · "what functions and triggers are in this Function App?" · "show the pipeline stages and jobs" · "what are the dependencies between these services?" · "generate an Azure mind map."

**Triggers `codegraph`:** "build a call graph for this repo" · "which functions have the highest fan-in/fan-out?" · "what does `processOrder` call, and what calls it?" · "map the cross-package calls in this monorepo" · "diagram the most connected parts of the code."

Because both live at the user-profile level, the **same prompts work in every repo you open** — no setup per project.

---

## 8. Where output goes

The skills are configured to write to a git-ignored, workspace-local folder so you can read the docs in your editor without polluting the repo:

```
<the repo you have open>/.graph-output/
├── azgraph/   (azure_graph.json, docs/overview.md, docs/repo_*.md, *.mmd)
└── codegraph/ (graph.json, docs/call_graph_overview.md, docs/functions/*.md, docs/diagrams/overview.mmd)
```

Add `.graph-output/` to your global gitignore so it never gets committed anywhere:

```bash
echo ".graph-output/" >> ~/.config/git/ignore     # or your global core.excludesfile
```

**Viewing `.mmd` diagrams:** open with the Mermaid Editor extension, paste into a ` ```mermaid ` block and open Markdown preview (`Ctrl+Shift+V`), or paste into https://mermaid.live for export.

---

## 9. How the skills stay repo-agnostic

Three design points make one central install work everywhere:

1. **Scripts take the target as an argument.** `scan_azure.py --roots <path>` and `build_graph.py --root <repo-root>` point at whatever repo is open — the skill isn't bound to any repo.
2. **`SKILL.md` references its own scripts by relative path** (`./scripts/...`), so the skill folder is self-contained wherever it's placed.
3. **Output is written next to the open workspace, not inside the skill.** The skill stays clean and reusable; results land where you're working.

In the two `SKILL.md` files you'll see `<skill-dir>` and `<workspace>` placeholders in the example commands — Copilot's agent substitutes the real absolute paths at run time (the skill directory it loaded from, and the folder you have open).

---

## 10. Verifying, troubleshooting, updating

**Verify it's installed:** type `/skills` in Copilot Chat (or *Configure Chat → Skills*). `azgraph` and `codegraph` should be listed. Then in agent mode ask *"build a call graph for this repo"* and confirm the skill activates in the chat.

| Symptom | Cause | Fix |
|---|---|---|
| Skills don't appear in `/skills` | Wrong folder, or `SKILL.md` nested one level too deep | Ensure `~/.copilot/skills/<skill>/SKILL.md` (not `.../<skill>/<skill>/SKILL.md`); reload window |
| Skill never activates | Not in agent mode, or free tier | Switch to agent mode on a paid Copilot plan |
| Skill silently fails to load | A namespace prefix was added to the `name:` field | `name` must be plain (`azgraph`), never `myorg/azgraph` or `myorg:azgraph` |
| `codegraph`: a language is missing | Its grammar didn't install | Re-run `setup_env.sh`; check its "UNAVAILABLE" line; that language is skipped until fixed |
| `codegraph`: `id_conflicts > 0` after merge | Inconsistent chunk roots or duplicate labels | Rebuild chunks with consistent roots and unique repo-relative labels |
| `azgraph`: repo shows `kind=unknown` | No Azure artefacts (or `host.json` not at repo root) | Expected for non-Azure repos; check `host.json` location |
| Monorepo subfolder skips the skills | Open folder isn't the repo root | Set `chat.useCustomizationsInParentRepositories: true` |
| `.mmd` won't render | No Mermaid extension | Install a Mermaid preview extension or use mermaid.live |

**Update a skill:** replace the `azgraph/` or `codegraph/` folder in `~/.copilot/skills/` (or the shared location from §5) with the new version and reload. Because it's centralised, that's the only copy to touch — no per-repo updates.

**Uninstall:** delete the skill folder from `~/.copilot/skills/` (and remove the `chat.agentSkillsLocations` entry if you used a shared folder). For `codegraph`, also delete its `.venv`.

---

### One-line summary

Unzip the two skill folders into `~/.copilot/skills/` (or a shared folder referenced by `chat.agentSkillsLocations`), run `codegraph`'s one-time `setup_env.sh`, then ask Copilot in agent mode — in *any* repo — for an Azure landscape or a call graph. Nothing is committed per repo, and updates happen in one place.
