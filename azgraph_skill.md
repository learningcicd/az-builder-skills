# Azure DevOps Landscape Skill — GitHub Copilot / VSCode

> **Single-file distribution for clients.** Everything needed to run this skill
> in VSCode with GitHub Copilot is contained here: the orchestration instructions,
> both scripts, and the artefact detection reference. The README section at the
> top covers install, uninstall, and VSCode prerequisites.

---

## Table of Contents

1. [README — Install, Uninstall, Prerequisites](#1-readme--install-uninstall-prerequisites)
2. [What this skill does and does not do](#2-what-this-skill-does-and-does-not-do)
3. [When to activate this skill](#3-when-to-activate-this-skill)
4. [Workflow](#4-workflow)
5. [Output files](#5-output-files)
6. [Viewing Mermaid diagrams in VSCode](#6-viewing-mermaid-diagrams-in-vscode)
7. [Anti-hallucination rules](#7-anti-hallucination-rules)
8. [Azure-specific knowledge applied during scan interpretation](#8-azure-specific-knowledge-applied-during-scan-interpretation)
9. [Limitations to disclose to the user](#9-limitations-to-disclose-to-the-user)
10. [Reference: Azure artefact detection patterns](#10-reference-azure-artefact-detection-patterns)
11. [Scripts](#11-scripts)
    - [scan_azure.py](#scan_azurepy)
    - [render_mindmap.py](#render_mindmappy)

---

## 1. README — Install, Uninstall, Prerequisites

### Prerequisites

| Requirement | Version | Purpose |
|---|---|---|
| Python | 3.9 + | Runs the two extraction scripts |
| Git | any | Cloning repos (optional — only if repos are not already local) |
| VSCode | 1.80 + | Editor and Copilot host |
| GitHub Copilot extension | latest | Reads `.github/copilot-instructions.md` |
| Mermaid extension (optional) | any | Renders `.mmd` diagram files in VSCode |

No external Python packages are required. The scanner uses only the Python standard library (`json`, `re`, `xml.etree`, `pathlib`, `os`, `argparse`, `datetime`).

### Install — single repo

1. **Extract this skill file.** Copy `scan_azure.py` and `render_mindmap.py` from the [Scripts section](#11-scripts) below into a folder in your repo. The recommended location is:

   ```
   .copilot/
   └── skills/
       └── azgraph/
           ├── scripts/
           │   ├── scan_azure.py
           │   └── render_mindmap.py
           └── output/          ← created automatically on first run
   ```

2. **Wire up Copilot instructions.** Copy the contents of Section 2–9 of this document into your repo's Copilot instructions file. If the file already exists, append this skill's section under a `## Azure DevOps Landscape Skill` heading:

   ```bash
   # If starting fresh:
   mkdir -p .github
   cp <this-section-content> .github/copilot-instructions.md

   # If appending to an existing file:
   echo "" >> .github/copilot-instructions.md
   cat <this-section-content> >> .github/copilot-instructions.md
   ```

3. **Commit both files** so the skill travels with the repo:

   ```bash
   git add .copilot/skills/azgraph/scripts/
   git add .github/copilot-instructions.md
   git commit -m "chore: add azgraph Copilot skill"
   ```

4. **Reload VSCode** (`Ctrl+Shift+P` → `Developer: Reload Window`) to ensure Copilot picks up the new instructions file.

5. **Verify** by opening Copilot Chat and typing:
   ```
   @workspace what Azure artefacts are in this repo?
   ```
   Copilot should invoke the skill workflow rather than trying to read source files directly.

### Install — multi-root VSCode workspace (multiple repos)

For a multi-root workspace (a `.code-workspace` file pointing at several repo folders):

1. Place the `azgraph/` skill folder in any one of the repos, or in a shared tooling repo that is included in the workspace.
2. The `.github/copilot-instructions.md` file should live at the root of the **primary** repo in the workspace, or in whichever repo Copilot treats as the workspace root.
3. Update the `--roots` and `--labels` arguments in Section 4's workflow to include all repo folders visible in the workspace.

### Install — Azure DevOps pipeline integration (optional)

To run the scanner automatically on every PR and commit the docs as a pipeline artefact:

```yaml
# azure-pipelines.yml addition
- stage: AzureGraphDocs
  jobs:
    - job: GenerateDocs
      steps:
        - task: UsePythonVersion@0
          inputs:
            versionSpec: '3.11'
        - script: |
            python .copilot/skills/azgraph/scripts/scan_azure.py \
              --roots $(Build.SourcesDirectory) \
              --labels $(Build.Repository.Name) \
              --out $(Build.ArtifactStagingDirectory)/azure_graph.json
            python .copilot/skills/azgraph/scripts/render_mindmap.py \
              --graph $(Build.ArtifactStagingDirectory)/azure_graph.json \
              --out-dir $(Build.ArtifactStagingDirectory)/docs
          displayName: 'Generate Azure landscape docs'
        - task: PublishBuildArtifacts@1
          inputs:
            pathToPublish: '$(Build.ArtifactStagingDirectory)/docs'
            artifactName: 'azure-landscape-docs'
```

### Uninstall

To fully remove the skill:

```bash
# 1. Remove the scripts
rm -rf .copilot/skills/azgraph/

# 2. Remove the instructions from Copilot
#    If the instructions file was created solely for this skill:
rm .github/copilot-instructions.md

#    If the instructions file has other content, edit it manually
#    and remove the Azure DevOps Landscape Skill section.

# 3. Remove generated output (if committed)
rm -rf .copilot/skills/azgraph/output/

# 4. Commit the removal
git add -A
git commit -m "chore: remove azgraph Copilot skill"
```

### Updating the skill

To update to a newer version of this skill file:

1. Replace `scan_azure.py` and `render_mindmap.py` with the updated versions from the new skill file's Scripts section.
2. Replace the Azure DevOps Landscape Skill section in `.github/copilot-instructions.md` with the updated Sections 2–9 from the new skill file.
3. Reload VSCode.
4. Re-run the scanner to regenerate output against the updated scripts.

---

## 2. What this skill does and does not do

This skill scans Azure DevOps repositories open in VSCode and produces:

- A structured JSON graph of every Function App, pipeline, package dependency, ARM/Bicep template, and cross-repo link found in the scanned set
- Mermaid mind maps — a full landscape view, per-repo detail views, and a cross-repo dependency graph
- Markdown documentation for each repo and an overall landscape overview

Every fact in the output traces to a real file and byte offset in the scanned repos. Nothing is inferred from folder naming conventions or repo names alone. Unresolved references — packages from outside the scanned set, pipelines referencing external template repos not in the scan — are listed as unresolved rather than guessed at.

**This skill does not:**
- Execute any code from the scanned repos
- Modify any file in the scanned repos
- Connect to Azure DevOps APIs, Azure Resource Manager, or any external service
- Describe runtime behaviour (what a function "does" at runtime, what data flows through a pipeline) — only static structure
- Detect infrastructure links made via shared Azure resources (same Storage Account, same Service Bus) — these require runtime or infrastructure data, not just source files

---

## 3. When to activate this skill

**Activate** when the user asks any of the following:
- "map this repo / show me the architecture / what does this codebase do"
- "what functions are in this Function App / what triggers are used"
- "show me the pipeline / what stages and jobs does this have"
- "what are the dependencies between these repos / how are these services connected"
- "generate a mind map / create a diagram / document this Azure project"
- "what packages does X depend on / does X depend on Y"
- "what calls what across repos / what links these repos"

**Do not activate** for:
- Single-file questions ("what does this method do")
- Debugging sessions or code generation unrelated to repo structure
- Questions about live Azure resource state ("is the function app running", "what's in the queue")

---

## 4. Workflow

Always follow these steps in order. Do not skip or reorder them.

### Step 1 — Identify repos to scan

Ask the user which repos to include if not already clear from context. In a VSCode multi-root workspace, each folder is a candidate. Check for a workspace file:

```bash
cat *.code-workspace 2>/dev/null || echo "no workspace file found"
```

List the workspace folders and confirm with the user before scanning.

### Step 2 — Run the scanner (requires explicit user approval before execution)

The scanner reads files only — it does not modify anything. Run it only after the user confirms the repo list:

```bash
python .copilot/skills/azgraph/scripts/scan_azure.py \
  --roots <repo1_path> <repo2_path> \
  --labels <repo1_name> <repo2_name> \
  --out .copilot/skills/azgraph/output/azure_graph.json
```

Check the printed summary line for each repo:

| Signal | Meaning | Action |
|---|---|---|
| `kind=unknown` with 0 apps and 0 pipelines | No Azure artefacts found | Tell the user before continuing |
| Any repo with `parse_errors` in the JSON | File read or XML parse failure | Flag to user; check the `parse_errors` array in the JSON |
| `function_apps=0` on a known Function App repo | host.json not found at expected location | Check that `host.json` exists at the repo root, not in a subdirectory |

### Step 3 — Generate mind maps and documentation

```bash
python .copilot/skills/azgraph/scripts/render_mindmap.py \
  --graph .copilot/skills/azgraph/output/azure_graph.json \
  --out-dir .copilot/skills/azgraph/output/docs
```

### Step 4 — Present results to the user

After generation, tell the user:

1. **What was found** — repo kinds, function count with trigger types, pipeline stages, cross-repo link count
2. **Where the output is** — list the generated files with their paths
3. **How to view mind maps** — see Section 6
4. **Any gaps** — repos with `kind=unknown`, functions detected via attribute scan (marked `*`), skipped files

---

## 5. Output files

All outputs land in `.copilot/skills/azgraph/output/`:

| File | Description |
|---|---|
| `azure_graph.json` | Raw structured graph — complete source of truth for all downstream docs |
| `docs/overview.md` | Summary table of all repos, cross-repo link table, links to all other docs |
| `docs/repo_<name>.md` | Per-repo deep-dive: functions, pipelines, packages, IaC templates |
| `docs/mindmap.mmd` | Full landscape mind map (all repos) |
| `docs/mindmap_<name>.mmd` | Per-repo detailed mind map |
| `docs/cross_repo_links.mmd` | Cross-repo dependency graph (colour-coded by repo kind) |

---

## 6. Viewing Mermaid diagrams in VSCode

`.mmd` files contain Mermaid diagram syntax. Three ways to render them:

**Option A — Mermaid Editor extension (recommended)**
Install `tomoyukim.vscode-mermaid-editor` from the VSCode marketplace. Open any `.mmd` file and use the preview panel.

**Option B — Markdown preview**
Install `bierner.markdown-mermaid`. Paste the `.mmd` file contents into any markdown file inside a fenced code block:
````
```mermaid
<paste .mmd contents here>
```
````
Then open the markdown preview (`Ctrl+Shift+V`).

**Option C — Browser**
Paste the `.mmd` file contents into [https://mermaid.live](https://mermaid.live) for an interactive view and PNG/SVG export.

---

## 7. Anti-hallucination rules

Apply these at every step, not just when generating documentation.

- **Functions:** every function mentioned in output must trace to an entry in `azure_graph.json` → `repos[].function_apps[].functions[]` with a real `file` path. Never list a function because it "should" exist based on a folder name or repo name.
- **Trigger types:** only state a trigger type as confirmed if it came from a parsed `function.json` binding. Functions with `detection: "attribute_scan"` must be described as approximate — never state their trigger type as definitive.
- **Cross-repo links:** only assert a link if it appears in `cross_repo_links[]` with an `evidence_file`. Do not speculate about which repos "probably" depend on each other based on their names.
- **Pipeline details:** service connection names, variable group names, and task names come from YAML text extraction — do not infer what Azure subscription a service connection targets, or what secrets a variable group holds.
- **IaC resources:** ARM and Bicep resource type lists come directly from template files. Do not describe what a resource type "does" at runtime — only name it.
- **Repo kind `unknown`:** do not guess the purpose of a repo from its name. Report it as unclassified and suggest the user verify manually.
- **External repos:** if a `cross_repo_links` entry has a `to_repo` value that is not in the scanned set (e.g. `pipeline-templates`), make this explicit — it is an external reference, not a scanned repo.

---

## 8. Azure-specific knowledge applied during scan interpretation

Use this as context when explaining results to the user. It is background knowledge, not a basis for adding facts to the graph that were not extracted from source files.

**Repo kinds:**
- `function_app` ⚡ — contains Function Apps, no pipeline
- `pipeline_only` 🔧 — pipelines only, no Function Apps
- `library` 📦 — produces packages consumed by other repos
- `infrastructure` 🏗 — ARM/Bicep IaC only
- `mixed` 🔀 — Function Apps and pipelines
- `unknown` ❓ — no Azure-specific artefacts detected; tell the user

**Function App triggers:**
- `httpTrigger` — HTTP/REST endpoint, synchronous
- `timerTrigger` — scheduled execution (cron expression in `schedule` binding)
- `queueTrigger` / `serviceBusTrigger` / `eventHubTrigger` — message-driven, asynchronous
- `blobTrigger` — fires on Azure Blob Storage create/update events
- `cosmosDBTrigger` — fires on Cosmos DB change feed
- `durableOrchestrationTrigger` / `durableActivityTrigger` / `durableEntityTrigger` — Durable Functions workflow steps; if present, the app implements stateful long-running workflows
- `eventGridTrigger` — event-driven via Azure Event Grid

**Pipeline constructs:**
- `variable_groups` — reference Azure DevOps Library variable groups, which commonly hold environment-specific config and secrets
- `service_connections` — named ADO service connections to Azure subscriptions or external services (the name shown is the ADO service connection name, not the Azure subscription name)
- `extends` with a template path — the pipeline uses a hardened/approved corporate pipeline template, common in enterprise ADO governance setups
- `artifact_feeds` — Azure Artifacts NuGet/npm feeds; packages consumed from a private feed are a soft cross-repo link if the producing repo is also in the scan

**Cross-repo link kinds:**
- `package_ref` — one repo's dependency file references a package that another scanned repo produces
- `pipeline_trigger` — a pipeline in one repo references a template in another via the `@<repo>` ADO syntax
- `pipeline_repo_resource` — a pipeline declares another ADO repo as a resource (for checkout or template use)

---

## 9. Limitations to disclose to the user

Always mention these after any scan:

1. **Private ADO artifact feeds:** packages consumed from private Azure Artifacts feeds appear in dependency lists but cannot be linked to a producer repo unless that repo is also in the scan.
2. **Dynamic pipeline triggers:** pipelines using `${{ if }}` conditional expressions or runtime variables may have triggers the static YAML scan misses.
3. **v4 / isolated worker / Python v2 Functions:** detected via attribute scan — trigger types may be approximate. Marked with `*` in all output.
4. **Shared library repos:** a `library`-kind repo will only show cross-repo links if the repos consuming its packages are also in the scan.
5. **TFVC repos:** the scanner assumes standard Git directory layout. TFVC repos with `$/project/branch` path structures are not explicitly handled.
6. **Infrastructure-level links:** dependencies via shared Azure resources (same Storage Account, Service Bus namespace, Key Vault) are not detected — they require runtime or ARM deployment data, not source files.
7. **Repo completeness:** the scan covers only repos passed via `--roots`. Repos in the same ADO project that are not included will appear as external references in `cross_repo_links` with `to_repo` not matching any scanned label.

---

## 10. Reference: Azure artefact detection patterns

### Function App detection

A directory is classified as a Function App root when it contains `host.json` at its top level — this is the only reliable programmatic signal. Detection is never based on directory name alone.

**v1–v3 (function.json style):** each subdirectory containing a `function.json` is one function. The scanner reads `bindings[]` and identifies the binding whose `type` ends in `Trigger`.

**v4 / isolated worker / Python v2 (attribute style):** no `function.json` exists.
- C# isolated worker: looks for `[Function("name")]` attributes in `.cs` files
- Python v2: looks for `@app.<trigger>()` decorators in `.py` files

These are marked `detection: "attribute_scan"` in the graph and shown with `*` in docs. Trigger type accuracy is lower — always note this to the user.

**Durable Functions:** flagged when a binding type is `durableOrchestrationTrigger`, `durableActivityTrigger`, `durableEntityTrigger`, `orchestrationTrigger`, or `activityTrigger`.

### Pipeline detection

Files are treated as Azure Pipelines definitions if they match:
- Exact filename: `azure-pipelines.yml`, `azure-pipelines.yaml`, `.azure-pipelines.yml`, `.azure-pipelines.yaml`
- Any `.yml`/`.yaml` file containing both `trigger:` and (`task:` or `steps:`)

All field extraction (stages, jobs, triggers, variable groups, service connections, feeds, template refs) uses regex on raw YAML text — not a YAML parser. YAML aliases, anchors, and multi-line strings may cause misses.

### Package detection

| Manager | Files scanned | What is extracted |
|---|---|---|
| NuGet | `*.csproj`, `packages.config` | `PackageReference` and `package` elements via XML parse |
| npm | `package.json` | `dependencies`, `devDependencies`, `peerDependencies` |
| pip | `requirements*.txt`, `pyproject.toml` | Package names from requirement lines and dependency strings |

`package-lock.json`, `yarn.lock`, and transitive dependency files are not scanned — they contain resolved transitive dependencies that would create noise.

### ARM template detection

A `.json` file is an ARM template if its `$schema` field contains `deploymentTemplate` or `subscriptionDeploymentTemplate`. All other JSON files are ignored.

### Bicep template detection

All `.bicep` files are scanned. Resource types extracted from `resource <id> '<type>@<api>'` declarations. Module references extracted from `module <id> '<path>'` declarations.

### Cross-repo link detection

Links are emitted only with explicit file evidence:
- **Package links:** a package name in one repo's dependency files matches a package name found in another scanned repo's source
- **Pipeline template refs:** `template: <path>@<repo>` syntax in pipeline YAML

**Not detected** (require infrastructure or runtime data): links via shared Azure resources, Event Grid topics, Service Bus namespaces, Key Vault, or ADO artifact feeds to external repos.

### Extending the scanner

To add a new artefact type:
1. Add a `scan_<artefact>()` function in `scan_azure.py` — always include a `file` field (repo-relative path) in every dict returned
2. Call it from `scan_repo()` and add its output to the repo dict
3. Add a render section in `render_mindmap.py` — both in `build_repo_mindmap()` and `render_repo_md()`
4. Add a column to the summary table in `render_overview_md()`
5. Update Section 8 of this document with the relevant resource/trigger types

---

## 11. Scripts

Place both scripts in `.copilot/skills/azgraph/scripts/` in your repo. No pip install required — standard library only.

---

### scan_azure.py

Scans one or more repo directories for Azure artefacts and emits a structured JSON graph. Run this first, always with explicit user approval.

```python
#!/usr/bin/env python3
"""
scan_azure.py — Scan one or more local repo directories for Azure-specific
artefacts: Function Apps, pipeline definitions, shared packages, ARM/Bicep
templates, and cross-repo dependency references.

Produces a structured JSON graph that downstream scripts turn into mind maps
and documentation. Every fact emitted traces to a real file and line number —
nothing is inferred from folder naming conventions alone.

Usage:
    python scan_azure.py --roots repo1/ repo2/ --out azure_graph.json
    python scan_azure.py --roots . --out azure_graph.json --label "my-service"

Output schema:
{
  "meta": { "roots": [...], "scanned_at": "...", "repo_count": N },
  "repos": [
    {
      "label": str,          # repo folder name or --label
      "root": str,           # absolute path
      "kind": str,           # "function_app" | "pipeline_only" | "library" | "mixed" | "unknown"
      "function_apps": [...],
      "pipelines": [...],
      "packages": { "nuget": [...], "npm": [...], "pip": [...] },
      "arm_templates": [...],
      "bicep_templates": [...],
      "shared_artifacts": [...],
      "parse_errors": [...]
    }
  ],
  "cross_repo_links": [
    {
      "from_repo": str, "to_repo": str,
      "kind": str,   # "package_ref" | "pipeline_trigger" | "artifact_feed" | "template_ref"
      "detail": str, # the actual package name / feed name / template path
      "evidence_file": str, "evidence_line": int
    }
  ]
}
"""
import argparse
import json
import os
import re
import xml.etree.ElementTree as ET
from datetime import datetime, timezone
from pathlib import Path

# ---------------------------------------------------------------------------
# File patterns we care about
# ---------------------------------------------------------------------------
PIPELINE_FILENAMES = {
    "azure-pipelines.yml", "azure-pipelines.yaml",
    ".azure-pipelines.yml", ".azure-pipelines.yaml",
}
PIPELINE_DIR_PATTERNS = [".azure", "pipelines", ".pipelines", "build", "ci"]

FUNCTION_MARKERS = {
    "host.json",        # root-level host.json = Function App root
}
FUNCTION_JSON = "function.json"      # individual function trigger definition
LOCAL_SETTINGS = "local.settings.json"

ARM_EXTS = {".json"}    # ARM templates are JSON — we check for $schema
BICEP_EXT = ".bicep"

EXCLUDE_DIRS = {
    ".git", "node_modules", "bin", "obj", "dist", "build", ".next",
    "__pycache__", ".venv", "venv", "env", ".tox", "target",
    ".vs", ".idea", ".vscode", "TestResults", "coverage",
}

# ---------------------------------------------------------------------------
# Helpers
# ---------------------------------------------------------------------------

def rel(path: Path, root: Path) -> str:
    try:
        return str(path.relative_to(root))
    except ValueError:
        return str(path)


def read_json_safe(path: Path):
    try:
        with open(path, encoding="utf-8", errors="replace") as f:
            return json.load(f), None
    except Exception as e:
        return None, str(e)


def read_text_safe(path: Path):
    try:
        return path.read_text(encoding="utf-8", errors="replace"), None
    except Exception as e:
        return None, str(e)


def find_line(text: str, pattern: str) -> int:
    """Return 1-based line number of first occurrence of pattern, or 0."""
    for i, line in enumerate(text.splitlines(), 1):
        if pattern in line:
            return i
    return 0


# ---------------------------------------------------------------------------
# Azure Function App scanning
# ---------------------------------------------------------------------------

TRIGGER_TYPES = {
    "httpTrigger", "timerTrigger", "blobTrigger", "queueTrigger",
    "serviceBusTrigger", "eventHubTrigger", "cosmosDBTrigger",
    "durableOrchestrationTrigger", "durableActivityTrigger",
    "durableEntityTrigger", "eventGridTrigger", "orchestrationTrigger",
    "activityTrigger",
}


def scan_function_app(host_json_path: Path, root: Path):
    """Given a host.json, scan its directory as a Function App."""
    app_dir = host_json_path.parent
    app_rel = rel(app_dir, root)

    host_data, err = read_json_safe(host_json_path)
    host_version = None
    extensions = []
    if host_data:
        host_version = host_data.get("version", "unknown")
        ext_bundle = host_data.get("extensionBundle", {})
        if ext_bundle:
            extensions.append({
                "kind": "extension_bundle",
                "id": ext_bundle.get("id"),
                "version": ext_bundle.get("version"),
            })

    functions = []
    # v2+ style: each function is a subdirectory with a function.json
    for entry in sorted(app_dir.rglob(FUNCTION_JSON)):
        if any(p in EXCLUDE_DIRS for p in entry.parts):
            continue
        fn_data, fn_err = read_json_safe(entry)
        fn_rel = rel(entry, root)
        fn_name = entry.parent.name
        if fn_data:
            bindings = fn_data.get("bindings", [])
            trigger = next((b for b in bindings if "trigger" in b.get("type", "").lower()), None)
            functions.append({
                "name": fn_name,
                "file": fn_rel,
                "trigger_type": trigger.get("type") if trigger else None,
                "trigger_direction": trigger.get("direction") if trigger else None,
                "binding_count": len(bindings),
                "is_durable": any("durable" in b.get("type", "").lower() for b in bindings),
            })
        else:
            functions.append({
                "name": fn_name,
                "file": fn_rel,
                "trigger_type": None,
                "parse_error": fn_err,
            })

    # Also detect v4 (isolated worker) style — no function.json, uses attributes
    # We mark these as detected via code scan only if no function.json found
    if not functions:
        for cs_file in app_dir.rglob("*.cs"):
            if any(p in EXCLUDE_DIRS for p in cs_file.parts):
                continue
            text, _ = read_text_safe(cs_file)
            if text and "[Function(" in text:
                matches = re.findall(r'\[Function\("([^"]+)"\)', text)
                for m in matches:
                    trigger_match = re.search(
                        r'\[(' + '|'.join(TRIGGER_TYPES) + r')\b', text, re.IGNORECASE
                    )
                    functions.append({
                        "name": m,
                        "file": rel(cs_file, root),
                        "trigger_type": trigger_match.group(1) if trigger_match else "unknown",
                        "detection": "attribute_scan",  # not from function.json — mark clearly
                        "is_durable": "orchestration" in text.lower() and m in text,
                    })
        for py_file in app_dir.rglob("*.py"):
            if any(p in EXCLUDE_DIRS for p in py_file.parts):
                continue
            text, _ = read_text_safe(py_file)
            if text and "app.route" in text or (text and "@app." in text):
                matches = re.findall(r'@app\.\w+\(route="([^"]*)"[^)]*\)', text)
                func_matches = re.findall(r'def (\w+)\s*\(', text)
                if matches or "@app." in text:
                    for fn in func_matches[:20]:  # cap to avoid noise
                        functions.append({
                            "name": fn,
                            "file": rel(py_file, root),
                            "trigger_type": "http_or_unknown",
                            "detection": "attribute_scan",
                        })

    local_settings = {}
    ls_path = app_dir / LOCAL_SETTINGS
    if ls_path.exists():
        ls_data, _ = read_json_safe(ls_path)
        if ls_data:
            vals = ls_data.get("Values", {})
            local_settings = {
                "runtime": vals.get("FUNCTIONS_WORKER_RUNTIME"),
                "storage_set": "AzureWebJobsStorage" in vals,
                "app_insights_set": "APPINSIGHTS_INSTRUMENTATIONKEY" in vals
                    or "APPLICATIONINSIGHTS_CONNECTION_STRING" in vals,
            }

    return {
        "app_dir": app_rel,
        "host_json": rel(host_json_path, root),
        "host_version": host_version,
        "extensions": extensions,
        "function_count": len(functions),
        "functions": functions,
        "local_settings": local_settings,
    }


# ---------------------------------------------------------------------------
# Pipeline scanning
# ---------------------------------------------------------------------------

def scan_pipeline(pipeline_path: Path, root: Path):
    """Extract key facts from an azure-pipelines.yml file."""
    text, err = read_text_safe(pipeline_path)
    if not text:
        return {"file": rel(pipeline_path, root), "parse_error": err}

    result = {
        "file": rel(pipeline_path, root),
        "triggers": [],
        "stages": [],
        "jobs": [],
        "steps_summary": {},
        "variables": [],
        "variable_groups": [],
        "service_connections": [],
        "artifact_feeds": [],
        "template_refs": [],
        "repo_refs": [],
        "extends": None,
    }

    # Triggers
    if re.search(r"^\s*trigger\s*:", text, re.MULTILINE):
        result["triggers"].append("push")
    if re.search(r"^\s*pr\s*:", text, re.MULTILINE):
        result["triggers"].append("pull_request")
    if re.search(r"^\s*schedules\s*:", text, re.MULTILINE):
        result["triggers"].append("schedule")
    if re.search(r"^\s*resources\s*:", text, re.MULTILINE):
        result["triggers"].append("resource_trigger_possible")

    # Stages and jobs (name extraction only — not evaluating logic)
    result["stages"] = re.findall(r"^\s*-\s*stage\s*:\s*(\S+)", text, re.MULTILINE)
    result["jobs"] = re.findall(r"^\s*-\s*job\s*:\s*(\S+)", text, re.MULTILINE)

    # Task types used
    tasks = re.findall(r"^\s*-?\s*task\s*:\s*(\S+)", text, re.MULTILINE)
    task_counts = {}
    for t in tasks:
        task_counts[t] = task_counts.get(t, 0) + 1
    result["steps_summary"] = task_counts

    # Variable groups (service connection/env references)
    result["variable_groups"] = re.findall(r"group\s*:\s*(\S+)", text)

    # Service connections (azureSubscription is the most common pattern)
    result["service_connections"] = list(set(
        re.findall(r"azureSubscription\s*:\s*['\"]?([^'\"\n]+)['\"]?", text)
    ))

    # Artifact feed references (NuGet/npm feeds in ADO)
    result["artifact_feeds"] = list(set(
        re.findall(r"feedsToUse\s*:\s*(\S+)|vstsFeed\s*:\s*['\"]?([^'\"\n]+)['\"]?", text)
    ))

    # Template references (cross-repo pipeline reuse)
    result["template_refs"] = re.findall(r"template\s*:\s*([^\s#]+)", text)

    # Cross-repo resource references
    result["repo_refs"] = re.findall(
        r"repository\s*:\s*(\S+)|repo\s*:\s*(\S+)", text
    )

    # extends (security-hardened pipeline templates)
    extends_match = re.search(r"^\s*extends\s*:\s*\n\s*template\s*:\s*(\S+)", text, re.MULTILINE)
    if extends_match:
        result["extends"] = extends_match.group(1)

    return result


# ---------------------------------------------------------------------------
# Package dependency scanning
# ---------------------------------------------------------------------------

def scan_nuget(root: Path):
    """Scan *.csproj and packages.config for NuGet dependencies."""
    packages = []
    for f in root.rglob("*.csproj"):
        if any(p in EXCLUDE_DIRS for p in f.parts):
            continue
        try:
            tree = ET.parse(f)
            r = tree.getroot()
            ns = r.tag.split("}")[0].lstrip("{") if "}" in r.tag else ""
            prefix = f"{{{ns}}}" if ns else ""
            for ref in r.iter(f"{prefix}PackageReference"):
                name = ref.get("Include") or ref.get("include")
                version = ref.get("Version") or ref.get("version")
                if name:
                    packages.append({
                        "name": name, "version": version,
                        "file": rel(f, root), "manager": "nuget"
                    })
        except Exception:
            pass
    for f in root.rglob("packages.config"):
        if any(p in EXCLUDE_DIRS for p in f.parts):
            continue
        try:
            tree = ET.parse(f)
            for pkg in tree.getroot().iter("package"):
                name = pkg.get("id")
                version = pkg.get("version")
                if name:
                    packages.append({
                        "name": name, "version": version,
                        "file": rel(f, root), "manager": "nuget"
                    })
        except Exception:
            pass
    return packages


def scan_npm(root: Path):
    packages = []
    for f in root.rglob("package.json"):
        if any(p in EXCLUDE_DIRS for p in f.parts):
            continue
        data, _ = read_json_safe(f)
        if not data:
            continue
        for section in ("dependencies", "devDependencies", "peerDependencies"):
            for name, version in (data.get(section) or {}).items():
                packages.append({
                    "name": name, "version": version,
                    "file": rel(f, root), "manager": "npm", "section": section
                })
    return packages


def scan_pip(root: Path):
    packages = []
    for f in list(root.rglob("requirements*.txt")) + list(root.rglob("pyproject.toml")):
        if any(p in EXCLUDE_DIRS for p in f.parts):
            continue
        text, _ = read_text_safe(f)
        if not text:
            continue
        if f.suffix == ".txt":
            for line in text.splitlines():
                line = line.strip()
                if line and not line.startswith("#") and not line.startswith("-"):
                    name = re.split(r"[>=<!~\[]", line)[0].strip()
                    if name:
                        packages.append({
                            "name": name, "file": rel(f, root), "manager": "pip"
                        })
        elif f.name == "pyproject.toml":
            deps = re.findall(r'"([A-Za-z0-9_\-]+)[>=<!\[]', text)
            for d in deps:
                packages.append({"name": d, "file": rel(f, root), "manager": "pip"})
    return packages


# ---------------------------------------------------------------------------
# ARM / Bicep template scanning
# ---------------------------------------------------------------------------

def scan_arm(root: Path):
    templates = []
    for f in root.rglob("*.json"):
        if any(p in EXCLUDE_DIRS for p in f.parts):
            continue
        data, _ = read_json_safe(f)
        if not data or not isinstance(data, dict):
            continue
        schema = data.get("$schema", "")
        if "deploymentTemplate" in schema or "subscriptionDeploymentTemplate" in schema:
            resources = data.get("resources", [])
            templates.append({
                "file": rel(f, root),
                "schema": schema,
                "resource_count": len(resources),
                "resource_types": list({r.get("type") for r in resources if isinstance(r, dict) and r.get("type")}),
            })
    return templates


def scan_bicep(root: Path):
    templates = []
    for f in root.rglob("*.bicep"):
        if any(p in EXCLUDE_DIRS for p in f.parts):
            continue
        text, _ = read_text_safe(f)
        if not text:
            continue
        resource_types = re.findall(r"resource\s+\w+\s+'([^']+)'", text)
        module_refs = re.findall(r"module\s+\w+\s+'([^']+)'", text)
        templates.append({
            "file": rel(f, root),
            "resource_types": list(set(resource_types)),
            "module_refs": module_refs,
        })
    return templates


# ---------------------------------------------------------------------------
# Repo classification
# ---------------------------------------------------------------------------

def classify_repo(function_apps, pipelines, packages, arm_templates, bicep_templates):
    has_functions = len(function_apps) > 0
    has_pipelines = len(pipelines) > 0
    has_packages = any(packages[k] for k in packages)
    has_iac = len(arm_templates) > 0 or len(bicep_templates) > 0
    if has_functions and has_pipelines:
        return "mixed"
    if has_functions:
        return "function_app"
    if has_pipelines and not has_functions:
        return "pipeline_only"
    if has_packages and not has_functions and not has_pipelines:
        return "library"
    if has_iac and not has_functions and not has_pipelines:
        return "infrastructure"
    return "unknown"


# ---------------------------------------------------------------------------
# Cross-repo link detection
# ---------------------------------------------------------------------------

def detect_cross_repo_links(repos):
    """
    Compare each repo's consumed packages/feeds/template-refs against what
    other repos produce. Only emits links with explicit evidence (file + line).
    Never guesses a link from folder name similarity alone.
    """
    links = []
    # Build index: package name -> repo label
    produced = {}  # package_name -> [repo_label]
    for repo in repos:
        label = repo["label"]
        for pkg in repo["packages"].get("nuget", []) + repo["packages"].get("npm", []) + repo["packages"].get("pip", []):
            produced.setdefault(pkg["name"], []).append(label)

    # Find which repos consume packages produced by another repo in this scan
    for repo in repos:
        label = repo["label"]
        for pkg in repo["packages"].get("nuget", []) + repo["packages"].get("npm", []) + repo["packages"].get("pip", []):
            producers = [r for r in produced.get(pkg["name"], []) if r != label]
            for producer in producers:
                links.append({
                    "from_repo": label,
                    "to_repo": producer,
                    "kind": "package_ref",
                    "detail": f"{pkg['name']} ({pkg['manager']})",
                    "evidence_file": pkg["file"],
                    "evidence_line": 0,  # line resolution requires a second pass; 0 = not pinpointed
                })

    # Pipeline template cross-references
    for repo in repos:
        label = repo["label"]
        for pipeline in repo["pipelines"]:
            for tref in pipeline.get("template_refs", []):
                # Template refs like `other-repo/templates/build.yml@other-repo`
                if "@" in tref:
                    target_repo = tref.split("@")[-1].strip()
                    links.append({
                        "from_repo": label,
                        "to_repo": target_repo,
                        "kind": "pipeline_trigger",
                        "detail": tref,
                        "evidence_file": pipeline["file"],
                        "evidence_line": 0,
                    })
            for repo_ref_tuple in pipeline.get("repo_refs", []):
                repo_ref = next((r for r in repo_ref_tuple if r), None)
                if repo_ref and repo_ref != label:
                    links.append({
                        "from_repo": label,
                        "to_repo": repo_ref,
                        "kind": "pipeline_repo_resource",
                        "detail": repo_ref,
                        "evidence_file": pipeline["file"],
                        "evidence_line": 0,
                    })

    return links


# ---------------------------------------------------------------------------
# Main
# ---------------------------------------------------------------------------

def scan_repo(root_path: Path, label: str):
    root = root_path.resolve()
    parse_errors = []

    # Find Function Apps
    function_apps = []
    for hj in root.rglob("host.json"):
        if any(p in EXCLUDE_DIRS for p in hj.parts):
            continue
        # Only treat as Function App root if parent is not deep inside another app
        function_apps.append(scan_function_app(hj, root))

    # Find pipelines
    pipelines = []
    for dirpath, dirnames, filenames in os.walk(root):
        dirnames[:] = [d for d in dirnames if d not in EXCLUDE_DIRS]
        for fn in filenames:
            fp = Path(dirpath) / fn
            if fn in PIPELINE_FILENAMES:
                pipelines.append(scan_pipeline(fp, root))
            elif fn.endswith((".yml", ".yaml")):
                # Check for ADO pipeline content heuristic
                text, _ = read_text_safe(fp)
                if text and ("azure-pipelines" in fn.lower() or
                             ("trigger:" in text and ("task:" in text or "steps:" in text))):
                    pipelines.append(scan_pipeline(fp, root))

    packages = {
        "nuget": scan_nuget(root),
        "npm":   scan_npm(root),
        "pip":   scan_pip(root),
    }

    arm_templates = scan_arm(root)
    bicep_templates = scan_bicep(root)

    kind = classify_repo(function_apps, pipelines, packages, arm_templates, bicep_templates)

    return {
        "label": label,
        "root": str(root),
        "kind": kind,
        "function_apps": function_apps,
        "pipelines": pipelines,
        "packages": packages,
        "arm_templates": arm_templates,
        "bicep_templates": bicep_templates,
        "parse_errors": parse_errors,
    }


def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--roots", nargs="+", required=True,
                    help="One or more repo root directories to scan")
    ap.add_argument("--labels", nargs="*", default=None,
                    help="Optional display labels for each root (same count as --roots)")
    ap.add_argument("--out", required=True, help="Output JSON path")
    args = ap.parse_args()

    roots = [Path(r) for r in args.roots]
    labels = args.labels if args.labels else [r.name for r in roots]
    if len(labels) != len(roots):
        raise SystemExit("--labels count must match --roots count if provided")

    repos = []
    for root, label in zip(roots, labels):
        print(f"[scan_azure] scanning {label} ({root})...")
        repo = scan_repo(root, label)
        repos.append(repo)
        print(f"[scan_azure]   → kind={repo['kind']}, "
              f"function_apps={len(repo['function_apps'])}, "
              f"pipelines={len(repo['pipelines'])}, "
              f"nuget={len(repo['packages']['nuget'])}, "
              f"npm={len(repo['packages']['npm'])}, "
              f"bicep={len(repo['bicep_templates'])}, "
              f"arm={len(repo['arm_templates'])}")

    cross_links = detect_cross_repo_links(repos)
    print(f"[scan_azure] detected {len(cross_links)} cross-repo links")

    graph = {
        "meta": {
            "roots": [str(r) for r in roots],
            "labels": labels,
            "scanned_at": datetime.now(timezone.utc).isoformat(),
            "repo_count": len(repos),
            "cross_link_count": len(cross_links),
        },
        "repos": repos,
        "cross_repo_links": cross_links,
    }

    Path(args.out).parent.mkdir(parents=True, exist_ok=True)
    with open(args.out, "w") as f:
        json.dump(graph, f, indent=2)
    print(f"[scan_azure] written → {args.out}")


if __name__ == "__main__":
    main()
```

---

### render_mindmap.py

Turns a `azure_graph.json` into Mermaid mind maps and Markdown documentation. Every claim in the output traces to a field in the graph JSON.

```python
#!/usr/bin/env python3
"""
render_mindmap.py — Turn an azure_graph.json (from scan_azure.py) into:

  1. docs/mindmap.mmd              — Mermaid mindmap of the full Azure landscape
  2. docs/mindmap_<repo>.mmd       — Per-repo Mermaid mindmap
  3. docs/cross_repo_links.mmd     — Mermaid graph of cross-repo dependencies
  4. docs/overview.md              — Markdown documentation of all repos
  5. docs/repo_<label>.md          — Per-repo deep-dive documentation

ANTI-HALLUCINATION DESIGN: every claim in the output traces to a field in the
graph JSON, which itself traces to a real file/line. Architecture-level
summaries are explicitly labelled as "inferred from scan evidence" wherever
they go beyond what a single field directly states. Function counts are always
paired with their detection method (function.json vs attribute scan) so the
reader knows the confidence level.

Usage:
    python render_mindmap.py --graph azure_graph.json --out-dir docs/
"""
import argparse
import json
import re
from pathlib import Path


def safe_id(s: str) -> str:
    return re.sub(r"[^A-Za-z0-9_]", "_", s)[:40]


def safe_label(s: str) -> str:
    return str(s).replace('"', "'").replace("\n", " ")[:80]


# ---------------------------------------------------------------------------
# Mermaid mindmap builders
# ---------------------------------------------------------------------------

def build_full_mindmap(graph: dict) -> str:
    """One mindmap covering all repos and their key Azure artefacts."""
    lines = ["mindmap", "  root((Azure Landscape))"]
    for repo in graph["repos"]:
        label = repo["label"]
        kind_emoji = {
            "function_app": "⚡",
            "pipeline_only": "🔧",
            "library": "📦",
            "mixed": "🔀",
            "infrastructure": "🏗",
            "unknown": "❓",
        }.get(repo["kind"], "")
        lines.append(f"    {safe_id(label)}[{kind_emoji} {safe_label(label)}]")

        # Function Apps
        if repo["function_apps"]:
            lines.append(f"      fa_{safe_id(label)}(Function Apps)")
            for app in repo["function_apps"]:
                app_node = safe_label(app["app_dir"])
                lines.append(f"        fa_{safe_id(label)}_{safe_id(app['app_dir'])}[{app_node}]")
                for fn in app.get("functions", [])[:12]:  # cap to keep diagram readable
                    tt = fn.get("trigger_type") or "?"
                    durable = " 🔗" if fn.get("is_durable") else ""
                    lines.append(f"          fn_{safe_id(fn['name'])}({safe_label(fn['name'])}: {tt}{durable})")
                if len(app.get("functions", [])) > 12:
                    lines.append(f"          more_fns_{safe_id(label)}(... +{len(app['functions'])-12} more)")

        # Pipelines
        if repo["pipelines"]:
            lines.append(f"      pip_{safe_id(label)}(Pipelines)")
            for p in repo["pipelines"][:8]:
                pname = safe_label(Path(p["file"]).name)
                triggers = ", ".join(p.get("triggers", [])) or "none"
                lines.append(f"        pip_{safe_id(p['file'])}[{pname} | {triggers}]")

        # ARM/Bicep
        if repo["arm_templates"] or repo["bicep_templates"]:
            lines.append(f"      iac_{safe_id(label)}(Infrastructure as Code)")
            for t in repo["arm_templates"][:5]:
                lines.append(f"        arm_{safe_id(t['file'])}[ARM: {safe_label(Path(t['file']).name)}]")
            for t in repo["bicep_templates"][:5]:
                lines.append(f"        bic_{safe_id(t['file'])}[Bicep: {safe_label(Path(t['file']).name)}]")

        # Packages (summarised — full list is in docs)
        pkg_counts = {
            k: len(v) for k, v in repo["packages"].items() if v
        }
        if pkg_counts:
            summary = ", ".join(f"{v} {k}" for k, v in pkg_counts.items())
            lines.append(f"      pkg_{safe_id(label)}(Packages: {summary})")

    return "\n".join(lines)


def build_repo_mindmap(repo: dict) -> str:
    """Per-repo mindmap with full function and pipeline detail."""
    label = repo["label"]
    lines = ["mindmap", f"  root(({safe_label(label)}))"]

    for app in repo["function_apps"]:
        app_id = safe_id(app["app_dir"])
        lines.append(f"    app_{app_id}[⚡ {safe_label(app['app_dir'])}]")
        lines.append(f"      host_ver_{app_id}(host.json v{app.get('host_version') or '?'})")
        if app.get("local_settings"):
            ls = app["local_settings"]
            runtime = ls.get("runtime") or "unknown"
            lines.append(f"      runtime_{app_id}(Runtime: {runtime})")
        for fn in app.get("functions", []):
            tt = fn.get("trigger_type") or "unknown"
            durable = " [Durable]" if fn.get("is_durable") else ""
            detect = " *" if fn.get("detection") == "attribute_scan" else ""
            lines.append(f"      fn_{safe_id(fn['name'])}({safe_label(fn['name'])} | {tt}{durable}{detect})")

    for pipeline in repo["pipelines"]:
        pid = safe_id(pipeline["file"])
        pname = Path(pipeline["file"]).name
        lines.append(f"    pipeline_{pid}[🔧 {pname}]")
        triggers = pipeline.get("triggers", [])
        if triggers:
            lines.append(f"      triggers_{pid}(Triggers: {', '.join(triggers)})")
        stages = pipeline.get("stages", [])
        if stages:
            lines.append(f"      stages_{pid}(Stages)")
            for s in stages[:10]:
                lines.append(f"        stage_{safe_id(s)}({safe_label(s)})")
        vgroups = pipeline.get("variable_groups", [])
        if vgroups:
            lines.append(f"      vg_{pid}(Variable Groups)")
            for vg in vgroups[:6]:
                lines.append(f"        vg_{safe_id(vg)}({safe_label(vg)})")
        sc = pipeline.get("service_connections", [])
        if sc:
            lines.append(f"      sc_{pid}(Service Connections)")
            for s in sc[:6]:
                lines.append(f"        sc_{safe_id(s)}({safe_label(s)})")
        for tref in pipeline.get("template_refs", [])[:5]:
            lines.append(f"      tref_{safe_id(tref)}(template: {safe_label(tref)})")

    for t in repo["arm_templates"][:6]:
        tid = safe_id(t["file"])
        rtypes = ", ".join(t.get("resource_types", [])[:3])
        lines.append(f"    arm_{tid}[ARM: {safe_label(Path(t['file']).name)}]")
        if rtypes:
            lines.append(f"      armtypes_{tid}({safe_label(rtypes)})")

    for t in repo["bicep_templates"][:6]:
        tid = safe_id(t["file"])
        rtypes = ", ".join(t.get("resource_types", [])[:3])
        lines.append(f"    bic_{tid}[Bicep: {safe_label(Path(t['file']).name)}]")
        if rtypes:
            lines.append(f"      bictypes_{tid}({safe_label(rtypes)})")
        for mref in t.get("module_refs", [])[:4]:
            lines.append(f"      bicmod_{safe_id(mref)}(module: {safe_label(mref)})")

    return "\n".join(lines)


def build_cross_repo_diagram(graph: dict) -> str:
    """Mermaid graph (LR) of cross-repo dependency links."""
    links = graph.get("cross_repo_links", [])
    if not links:
        return 'graph LR\n  note["No cross-repo links detected in scanned repos"]'

    lines = ["graph LR"]
    # Repo nodes
    for repo in graph["repos"]:
        label = repo["label"]
        kind_style = {
            "function_app": ":::funcApp",
            "pipeline_only": ":::pipeline",
            "library": ":::library",
            "mixed": ":::mixed",
        }.get(repo["kind"], "")
        lines.append(f"  {safe_id(label)}[\"{safe_label(label)}\n({repo['kind']})\"]")

    # Style classes
    lines.append("  classDef funcApp fill:#0078D4,color:#fff,stroke:#005a9e")
    lines.append("  classDef pipeline fill:#107C10,color:#fff,stroke:#0a5a0a")
    lines.append("  classDef library fill:#D83B01,color:#fff,stroke:#a02d00")
    lines.append("  classDef mixed fill:#8764B8,color:#fff,stroke:#5e3f8f")

    # Apply styles
    by_kind = {}
    for repo in graph["repos"]:
        by_kind.setdefault(repo["kind"], []).append(safe_id(repo["label"]))
    for kind, ids in by_kind.items():
        class_name = {"function_app": "funcApp", "pipeline_only": "pipeline",
                      "library": "library", "mixed": "mixed"}.get(kind)
        if class_name:
            lines.append(f"  class {','.join(ids)} {class_name}")

    # Edges — deduplicate (from, to, kind) triples
    seen = set()
    kind_labels = {
        "package_ref": "📦 pkg",
        "pipeline_trigger": "🔧 pipeline",
        "pipeline_repo_resource": "🔗 repo resource",
        "artifact_feed": "🗄 feed",
        "template_ref": "📄 template",
    }
    for link in links:
        key = (link["from_repo"], link["to_repo"], link["kind"])
        if key in seen:
            continue
        seen.add(key)
        from_id = safe_id(link["from_repo"])
        to_id = safe_id(link["to_repo"])
        edge_label = kind_labels.get(link["kind"], link["kind"])
        lines.append(f"  {from_id} -->|{edge_label}| {to_id}")

    return "\n".join(lines)


# ---------------------------------------------------------------------------
# Markdown documentation builders
# ---------------------------------------------------------------------------

def render_overview_md(graph: dict) -> str:
    meta = graph["meta"]
    repos = graph["repos"]
    links = graph.get("cross_repo_links", [])
    lines = []

    lines += [
        "# Azure DevOps Repository Landscape — Overview",
        "",
        "> All data below is derived directly from scanned file content. "
        "Architecture interpretations are labelled as such. "
        "Function counts marked with `*` were detected via attribute scan "
        "(no `function.json`) and should be treated as approximate.",
        "",
        f"**Scanned at:** {meta.get('scanned_at', 'unknown')}  ",
        f"**Repos scanned:** {meta['repo_count']}  ",
        f"**Cross-repo links detected:** {meta.get('cross_link_count', len(links))}",
        "",
        "---",
        "",
        "## Repository Summary",
        "",
        "| Repo | Kind | Function Apps | Functions | Pipelines | NuGet | npm | pip | ARM | Bicep |",
        "|---|---|---|---|---|---|---|---|---|---|",
    ]
    for repo in repos:
        total_fns = sum(a["function_count"] for a in repo["function_apps"])
        lines.append(
            f"| `{repo['label']}` | {repo['kind']} | "
            f"{len(repo['function_apps'])} | {total_fns} | "
            f"{len(repo['pipelines'])} | "
            f"{len(repo['packages']['nuget'])} | "
            f"{len(repo['packages']['npm'])} | "
            f"{len(repo['packages']['pip'])} | "
            f"{len(repo['arm_templates'])} | "
            f"{len(repo['bicep_templates'])} |"
        )

    lines += ["", "---", "", "## Cross-Repo Dependency Links", ""]
    if links:
        lines += [
            "| From | To | Link kind | Detail | Evidence file |",
            "|---|---|---|---|---|",
        ]
        for link in links:
            lines.append(
                f"| `{link['from_repo']}` | `{link['to_repo']}` | "
                f"{link['kind']} | `{safe_label(link['detail'])}` | "
                f"`{link['evidence_file']}` |"
            )
    else:
        lines.append("No cross-repo links were detected within the scanned set of repos. "
                     "Links to repos outside this scan (external ADO feeds, public packages) "
                     "are not shown — they appear as unresolved references in per-repo docs.")

    lines += ["", "---", "", "## Mind Map Files", "",
              "| File | Contents |",
              "|---|---|",
              "| `mindmap.mmd` | Full landscape mind map (all repos) |",
              "| `mindmap_<repo>.mmd` | Per-repo detailed mind map |",
              "| `cross_repo_links.mmd` | Dependency graph between repos |",
              "",
              "Open `.mmd` files with the [Markdown Preview Mermaid Support]"
              "(https://marketplace.visualstudio.com/items?itemName=bierner.markdown-mermaid) "
              "extension in VSCode, or paste into [mermaid.live](https://mermaid.live).",
              ""]

    return "\n".join(lines)


def render_repo_md(repo: dict) -> str:
    label = repo["label"]
    lines = []
    lines += [
        f"# {label} — Repository Detail",
        "",
        f"**Kind:** {repo['kind']}  ",
        f"**Root:** `{repo['root']}`",
        "",
        "---",
        "",
    ]

    # Function Apps
    if repo["function_apps"]:
        lines += ["## Azure Function Apps", ""]
        for app in repo["function_apps"]:
            lines += [
                f"### `{app['app_dir']}`",
                "",
                f"- **host.json version:** {app.get('host_version') or 'not detected'}",
            ]
            ls = app.get("local_settings", {})
            if ls:
                lines.append(f"- **Worker runtime:** {ls.get('runtime') or 'not set'}")
                lines.append(f"- **Storage configured:** {ls.get('storage_set', False)}")
                lines.append(f"- **App Insights configured:** {ls.get('app_insights_set', False)}")
            if app.get("extensions"):
                for ext in app["extensions"]:
                    if ext["kind"] == "extension_bundle":
                        lines.append(f"- **Extension bundle:** `{ext['id']}` version `{ext['version']}`")

            lines += ["", f"**Functions ({app['function_count']}):**", ""]
            if app["functions"]:
                lines += [
                    "| Function | Trigger | Durable | Detection | Bindings |",
                    "|---|---|---|---|---|",
                ]
                for fn in app["functions"]:
                    detect = fn.get("detection", "function.json")
                    approx = " \\*" if detect == "attribute_scan" else ""
                    durable = "✓" if fn.get("is_durable") else ""
                    binding_count = fn.get("binding_count", "?")
                    lines.append(
                        f"| `{fn['name']}{approx}` | "
                        f"{fn.get('trigger_type') or '?'} | "
                        f"{durable} | {detect} | {binding_count} |"
                    )
                lines.append("")
                if any(fn.get("detection") == "attribute_scan" for fn in app["functions"]):
                    lines.append("> \\* Detected via attribute/decorator scan (no `function.json`). "
                                 "Trigger type may be approximate — verify against source.")
            lines.append("")

    # Pipelines
    if repo["pipelines"]:
        lines += ["## Azure Pipelines", ""]
        for p in repo["pipelines"]:
            lines += [f"### `{p['file']}`", ""]
            if p.get("parse_error"):
                lines.append(f"> ⚠ Parse error: {p['parse_error']}")
                lines.append("")
                continue
            lines.append(f"- **Triggers:** {', '.join(p.get('triggers', [])) or 'none detected'}")
            if p.get("stages"):
                lines.append(f"- **Stages:** {', '.join(p['stages'])}")
            if p.get("jobs"):
                lines.append(f"- **Jobs:** {', '.join(p['jobs'])}")
            if p.get("extends"):
                lines.append(f"- **Extends (hardened template):** `{p['extends']}`")
            if p.get("variable_groups"):
                lines.append(f"- **Variable groups:** {', '.join(p['variable_groups'])}")
            if p.get("service_connections"):
                lines.append(f"- **Service connections:** {', '.join(p['service_connections'])}")
            if p.get("artifact_feeds"):
                flat = [item for pair in p["artifact_feeds"] for item in pair if item]
                if flat:
                    lines.append(f"- **Artifact feeds:** {', '.join(flat)}")
            if p.get("template_refs"):
                lines.append("- **Template references:**")
                for t in p["template_refs"]:
                    lines.append(f"  - `{t}`")
            if p.get("steps_summary"):
                lines.append("- **Tasks used:**")
                for task, count in sorted(p["steps_summary"].items(), key=lambda x: -x[1]):
                    lines.append(f"  - `{task}` × {count}")
            lines.append("")

    # Packages
    all_pkgs = (
        repo["packages"]["nuget"] +
        repo["packages"]["npm"] +
        repo["packages"]["pip"]
    )
    if all_pkgs:
        lines += ["## Package Dependencies", ""]
        for manager in ("nuget", "npm", "pip"):
            pkgs = repo["packages"][manager]
            if pkgs:
                lines += [f"### {manager.upper()} ({len(pkgs)} packages)", ""]
                lines += ["| Package | Version | File |", "|---|---|---|"]
                for pkg in sorted(pkgs, key=lambda p: p["name"].lower())[:100]:
                    lines.append(f"| `{pkg['name']}` | {pkg.get('version') or '—'} | `{pkg['file']}` |")
                if len(pkgs) > 100:
                    lines.append(f"| *...and {len(pkgs)-100} more* | | |")
                lines.append("")

    # IaC
    if repo["arm_templates"] or repo["bicep_templates"]:
        lines += ["## Infrastructure as Code", ""]
        if repo["arm_templates"]:
            lines += [f"### ARM Templates ({len(repo['arm_templates'])})", ""]
            lines += ["| Template | Resource types |", "|---|---|"]
            for t in repo["arm_templates"]:
                rtypes = ", ".join(t.get("resource_types", [])[:5])
                lines.append(f"| `{t['file']}` | {rtypes or '—'} |")
            lines.append("")
        if repo["bicep_templates"]:
            lines += [f"### Bicep Templates ({len(repo['bicep_templates'])})", ""]
            lines += ["| Template | Resource types | Module refs |", "|---|---|---|"]
            for t in repo["bicep_templates"]:
                rtypes = ", ".join(t.get("resource_types", [])[:5])
                mrefs = ", ".join(t.get("module_refs", [])[:3])
                lines.append(f"| `{t['file']}` | {rtypes or '—'} | {mrefs or '—'} |")
            lines.append("")

    # Parse errors
    if repo.get("parse_errors"):
        lines += ["## ⚠ Parse Errors", ""]
        for e in repo["parse_errors"]:
            lines.append(f"- {e}")
        lines.append("")

    return "\n".join(lines)


# ---------------------------------------------------------------------------
# Main
# ---------------------------------------------------------------------------

def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--graph", required=True, help="azure_graph.json from scan_azure.py")
    ap.add_argument("--out-dir", required=True, help="Output directory for docs and diagrams")
    args = ap.parse_args()

    with open(args.graph) as f:
        graph = json.load(f)

    out = Path(args.out_dir)
    out.mkdir(parents=True, exist_ok=True)

    # Full mind map
    (out / "mindmap.mmd").write_text(build_full_mindmap(graph))
    print(f"[render_mindmap] wrote mindmap.mmd")

    # Per-repo mind maps
    for repo in graph["repos"]:
        safe = safe_id(repo["label"])
        (out / f"mindmap_{safe}.mmd").write_text(build_repo_mindmap(repo))
        print(f"[render_mindmap] wrote mindmap_{safe}.mmd")

    # Cross-repo dependency diagram
    (out / "cross_repo_links.mmd").write_text(build_cross_repo_diagram(graph))
    print(f"[render_mindmap] wrote cross_repo_links.mmd")

    # Overview markdown
    (out / "overview.md").write_text(render_overview_md(graph))
    print(f"[render_mindmap] wrote overview.md")

    # Per-repo docs
    for repo in graph["repos"]:
        safe = safe_id(repo["label"])
        (out / f"repo_{safe}.md").write_text(render_repo_md(repo))
        print(f"[render_mindmap] wrote repo_{safe}.md")

    print(f"[render_mindmap] all outputs → {out}/")


if __name__ == "__main__":
    main()
```

---

*Azure DevOps Landscape Skill — GitHub Copilot / VSCode edition.*
*Specialised for: Azure Function Apps, Azure DevOps Pipelines, ARM/Bicep IaC, cross-repo dependency mapping.*
