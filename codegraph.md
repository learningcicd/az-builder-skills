# Codegraph — Code Property Graph Skill

> **Single-file reference for clients.** This document contains the full orchestration guide, all reference material, and all scripts needed to run the codegraph skill. Everything a Claude agent (or any AI coding assistant) needs to extract and document a repository's function call graph is here in one place.

---

## Table of Contents

1. [What this skill does and does not do](#1-what-this-skill-does-and-does-not-do)
2. [Workflow overview](#2-workflow-overview)
3. [Step 0 — Setup](#3-step-0--setup)
4. [Step 1 — Get the repo on disk](#4-step-1--get-the-repo-on-disk)
5. [Step 2 — Profile the repo](#5-step-2--profile-the-repo)
6. [Step 3 — Extraction strategy](#6-step-3--extraction-strategy)
7. [Step 4 — Generate documentation](#7-step-4--generate-documentation)
8. [Step 5 — Semantic summaries (optional)](#8-step-5--semantic-summaries-optional)
9. [Anti-hallucination rules](#9-anti-hallucination-rules)
10. [Reference: Large-repo and monorepo strategy](#10-reference-large-repo-and-monorepo-strategy)
11. [Reference: Language support](#11-reference-language-support)
12. [Scripts](#12-scripts)
    - [setup_env.sh](#setupenvsh)
    - [profile_repo.py](#profile_repopy)
    - [build_graph.py](#build_graphpy)
    - [merge_graphs.py](#merge_graphspy)
    - [render_docs.py](#render_docspy)

---

## 1. What this skill does and does not do

This skill extracts **structural facts** from source code using real AST parsing (tree-sitter): which functions exist, where they're defined, and which calls happen where, resolved against an exact symbol table wherever possible. It does **not** guess at runtime behavior, infer what code "probably does" from naming, or fabricate relationships that weren't actually parsed. If a call can't be statically resolved (external library, dynamic dispatch, reflection), it is reported as unresolved — never silently dropped, never guessed at.

This distinction matters: the whole point of this skill is to be a source of ground truth the user can trust, not a plausible-sounding narrative. Hold to that throughout — see [Anti-hallucination rules](#9-anti-hallucination-rules) before writing any documentation.

---

## 2. Workflow overview

1. **Get the repo on disk** — clone or use a local path the user gave you
2. **Profile it** — language mix, size, monorepo structure
3. **Decide strategy** — single pass vs. chunked, based on the profile
4. **Extract the graph** — run the tree-sitter extractor (chunked or whole)
5. **Merge** — combine chunk graphs if chunked, resolving cross-chunk calls
6. **Generate documentation** — markdown + diagrams, strictly from graph data
7. **Report honestly** — coverage stats, what was skipped and why, before any narrative summary

Scripts live in the [Scripts section](#12-scripts) below. Read this document fully before running anything — the strategy decision in step 3 changes how steps 4–5 are invoked.

---

## 3. Step 0 — Setup

The extractor requires tree-sitter and per-language grammar packages. Run the following script **only after explicit user approval**, as it installs packages into the active Python environment:

```bash
bash setup_env.sh
```

The script reports which language grammars were successfully installed. If a language shows as unavailable, files in that language will be skipped entirely — not parsed with degraded accuracy, not approximated via regex. Any skipped languages must be disclosed in the final documentation.

---

## 4. Step 1 — Get the repo on disk

If given a GitHub URL, clone it shallow (history isn't needed for structural analysis):

```bash
git clone --depth 1 <url> /home/claude/repo_workspace/<repo-name>
```

If given a local path or uploaded files, use them directly. If the user references a private repo or one requiring auth that you can't access, say so plainly rather than attempting to fabricate results from the repo name alone.

---

## 5. Step 2 — Profile the repo

Before parsing a single function body, understand the shape of what you're dealing with:

```bash
python profile_repo.py --root /home/claude/repo_workspace/<repo-name> --out /home/claude/profile.json
```

Read the output. It tells you:

| Field | Meaning |
|---|---|
| `size_tier` | `small` / `medium` / `large` / `very_large` — used only to pick a strategy, never reported as a precise metric |
| `is_monorepo` | Whether multiple distinct project markers were found (`package.json`, `pyproject.toml`, `go.mod`, `Cargo.toml`, `pom.xml`, `composer.json`) |
| `boundaries` | Candidate chunk units — real package/workspace members if markers were found, top-level directories as fallback |
| `recommended_strategy` | `single_pass` or `chunked_by_boundary` |

**Use this as a starting recommendation, not gospel.** You have judgment the script doesn't: if `boundaries` look lopsided (one package is 95% of the code), sub-chunk that big one by directory while leaving the small ones whole. If the profile shows a monorepo with 15 tiny packages each under 1000 LOC, grouping a few together is more efficient than 15 separate passes.

---

## 6. Step 3 — Extraction strategy

### Strategy A — Single pass (small/medium, non-monorepo)

```bash
python build_graph.py --root <repo-root> --out /home/claude/graph.json
```

Default for anything the profiler marked `single_pass`. Simpler, faster, no merge step needed.

**Reliability fallback:** if a "medium" tier single-pass run times out or fails, don't retry blindly. Fall back to chunking by top-level directories and proceed with Strategy B. Note the fallback in your final report so the user understands what happened.

### Strategy B — Chunked by boundary (large or monorepo)

Run the extractor once per boundary, using `--chunk-label` matching the repo-relative path so the merge step can attribute functions correctly and resolve cross-chunk calls cleanly:

```bash
for boundary in <each boundary.path from profile.json>; do
  python build_graph.py \
    --root <repo-root>/<boundary> \
    --out /home/claude/chunks/<safe-name>.json \
    --chunk-label "<boundary>"
done
```

Run these **sequentially**, not in parallel — sequential runs make failures attributable to a specific chunk.

If a single chunk is itself enormous, sub-chunk it by subdirectory using a label like `"packages/big-pkg/src/auth"` (always full repo-relative path, never just the leaf directory name).

**After each chunk run, check `parsed_file_count` vs `file_count` in the summary.** A chunk with most files failing is a signal to inspect `parse_error` fields before proceeding — don't silently accumulate broken chunks into a merge.

### Merge step (Strategy B only)

```bash
python merge_graphs.py --chunks-dir /home/claude/chunks --out /home/claude/graph.json
```

Check the summary for `id_conflicts > 0` — this means chunks were built against inconsistent root paths, or two chunks were given the same `--chunk-label`. Resolve before proceeding.

---

## 7. Step 4 — Generate documentation

```bash
python render_docs.py --graph /home/claude/graph.json --out-dir /home/claude/docs --top-n 25
```

Produces:

| Output file | Contents |
|---|---|
| `docs/call_graph_overview.md` | Coverage stats, top fan-out/fan-in functions, unresolved call summary |
| `docs/functions/<chunk>.md` | Per-chunk function listings with their exact outgoing calls |
| `docs/diagrams/overview.mmd` | Mermaid diagram of the highest-connectivity functions only — a full-repo diagram is unreadable |

All content is templated directly from graph JSON fields. **Do not hand-edit these files to add explanatory prose about what functions "do" unless you've separately verified that by reading the actual function source** — see Step 5 below.

---

## 8. Step 5 — Semantic summaries (optional)

The structural docs from Step 4 are reliable by construction. If the user additionally wants natural-language descriptions of *what each function does* (not just what it calls), that requires reading actual function source and is where hallucination risk reappears. If you do this:

- Read the actual function body (using the `file`, `start_line`, `end_line` fields from the graph) before describing it — never describe a function from its name alone.
- Keep summaries scoped to what's visibly in the function body. If behavior depends on an unresolved external call, say so rather than guessing.
- Mark every such summary as a separate, clearly-labeled section — e.g., *"Function descriptions (interpreted from source — verify if behavior-critical)"* — so the user can distinguish structural fact from interpretation.
- For very large repos, prioritize high fan-in/fan-out functions from the overview doc rather than attempting to summarize everything.

---

## 9. Anti-hallucination rules

These apply throughout, not just in Step 5.

- **Functions:** every function listed in documentation must trace to an actual parsed AST node with a real file path and line number. Never list a function because it "should" exist based on a pattern seen elsewhere.
- **Call edges:** every edge must trace to an actual call-expression node. Never add an edge because two functions "probably" interact.
- **Unresolved calls:** stay unresolved. Never pick the "most likely" candidate when a name is ambiguous or external. The scripts enforce this via `resolved` / `ambiguous_candidates` fields — don't override it in your own writing.
- **Skipped languages:** if a grammar wasn't available, that language contributes **zero** functions and **zero** calls to the graph. State this explicitly — don't let silence imply full coverage. Check `meta.skipped_languages` in the graph JSON.
- **Unparsed files:** listed under "Unparsed files" in `docs/call_graph_overview.md`. Repeat this disclosure in your summary — never let the user assume 100% coverage.
- **Architecture claims:** ground any higher-level claim in specific graph evidence (e.g., "the `routes/` package has the highest fan-in, consistent with it being the central dispatcher") rather than general impressions.

---

## 10. Reference: Large-repo and monorepo strategy

### Why chunking matters

Parsing an entire large repository in one Python process risks three failure modes: the process running out of memory while holding thousands of ASTs, a single pass simply taking too long and getting interrupted, and — most importantly — a partial failure silently producing a graph that looks complete but covers only part of the repo. Chunking turns one large fragile operation into many small reliable ones, each independently verifiable before being merged.

### Choosing chunk boundaries

In order of preference:

1. **Real package/workspace boundaries.** A directory containing `package.json`, `pyproject.toml`, `go.mod`, `Cargo.toml`, `pom.xml`/`build.gradle`, or `composer.json` is a meaningful unit — it's how the codebase's own authors divided responsibility. `profile_repo.py` detects these automatically and reports them as `boundaries` with `kind: "package"`.

2. **Workspace members within a monorepo tool.** Tools like pnpm/yarn/lerna workspaces, Nx, Turborepo, or a Python monorepo with multiple `pyproject.toml` files all produce multiple markers — `profile_repo.py`'s `is_monorepo` flag goes true when more than one marker is found. Each member should get its own chunk and its own `--chunk-label` matching its repo-relative path.

3. **Top-level directories, as fallback.** If no markers are found (a flat script collection, or a build system not in the marker list), `profile_repo.py` falls back to top-level directories (`kind: "top_level_dir"`). These may not be semantically meaningful — be honest about this being a fallback.

4. **Sub-chunking within a boundary.** If a single package boundary is enormous (a 200k-LOC monolith), split further by subdirectory. Keep `--chunk-label` as the full repo-relative path (e.g. `"packages/big-pkg/src/auth"`, not just `"auth"`) so labels stay globally unique.

### Size tiers

| Tier | LOC range | Default strategy |
|---|---|---|
| `small` | < 30k | Always single-pass, even if monorepo-shaped |
| `medium` | 30k–150k | Single-pass if one project; chunked if monorepo |
| `large` | 150k–600k | Always chunked |
| `very_large` | 600k+ | Always chunked |

These thresholds are starting points. Reliability observed in practice should override the size-tier heuristic.

### Cross-chunk call resolution

When chunks are merged, calls that couldn't be resolved within their own chunk get a second resolution pass against the entire merged symbol table. A function in `packages/api` calling a function in `packages/shared-utils` resolves correctly once merged, even though neither chunk alone could resolve it.

The same conservative rule applies at merge time: if a call's target name matches more than one function across the whole repo, it stays unresolved with `ambiguous_candidates` listed. This is correct behavior for generic names like `run`, `execute`, `handle` — not a bug.

### Handling extremely large individual files

`build_graph.py --max-file-kb` (default 1024 KB) skips files above the size threshold entirely. These are filtered out before parsing is attempted. If a specific huge file needs covering (e.g. a massive generated API client), raise `--max-file-kb` for that one chunk specifically, not globally.

### What monorepo handling does NOT mean

It does not mean producing one graph per package and stopping. That silently misses cross-package calls, which are often exactly what matters most in a monorepo. The merge step's cross-chunk resolution is not optional polish — it's the core value of treating a monorepo as one logical system rather than N unrelated ones.

---

## 11. Reference: Language support

### Supported languages

| Language | Extensions | Function-like nodes extracted | Call nodes extracted |
|---|---|---|---|
| Python | `.py` | `function_definition` (includes methods) | `call` |
| JavaScript | `.js`, `.jsx`, `.mjs`, `.cjs` | function declarations, methods, function expressions, arrow functions | `call_expression` |
| TypeScript | `.ts` | same as JavaScript | `call_expression` |
| TSX | `.tsx` | same as JavaScript | `call_expression` |
| Go | `.go` | function and method declarations | `call_expression` |
| Java | `.java` | method and constructor declarations | method invocations, object creation |
| Ruby | `.rb` | `method` | `call`, `method_call` |
| Rust | `.rs` | `function_item` | `call_expression`, `macro_invocation` |
| C | `.c`, `.h` | `function_definition` | `call_expression` |
| C++ | `.cpp`, `.cc`, `.hpp` | `function_definition` | `call_expression` |
| C# | `.cs` | method, constructor, local function declarations | `invocation_expression` |
| PHP | `.php` | function and method declarations | function/method call expressions |

This list is intentionally conservative: a node type is only listed here if it's been verified against actual grammar output. If a construct isn't listed (e.g. Python lambdas), it's not extracted — under-extraction is safer than mis-extraction.

### Unsupported languages

If a grammar couldn't be installed or a file extension isn't in the registry, those files are **excluded entirely** — they contribute zero functions and zero calls. This is recorded in `meta.skipped_languages` and must be surfaced in any user-facing summary. Never fall back to regex silently — an unannounced accuracy drop is worse than an announced gap.

### Extending to a new language

If a `tree-sitter-<language>` PyPI package exists:

1. Add the extension(s) → `(module_name, ts_language_name)` mapping to `LANGUAGE_MODULES` in `build_graph.py`.
2. Determine correct node type names by parsing a small sample and walking the tree printing `node.type` for each node. Do not guess from a related language's grammar.
3. Add the verified sets to `FUNCTION_NODE_TYPES`, `CALL_NODE_TYPES`, and `CLASS_NODE_TYPES`.
4. Add the package name to `setup_env.sh`'s `PACKAGES` array.
5. Test with a small synthetic sample that has at least one resolved in-file call and one external/unresolvable call.

### Why tree-sitter over regex

Regex-based call detection produces large numbers of false positives (keywords, type casts, string contents) and false negatives (multi-line calls, unusual syntax). Tree-sitter answers "is this a call expression" structurally from the grammar, not from text shape. A regex-based count looks identical in format to a real one while being substantially less trustworthy — that's exactly what this skill is designed to avoid.

---

## 12. Scripts

Copy each script to your working directory before running. All scripts are standalone Python 3 with no imports beyond `tree-sitter` packages (installed by `setup_env.sh`) and the standard library.

---

### setup_env.sh

Install tree-sitter and all language grammars, then verify which ones actually imported successfully.

```bash
```bash
#!/usr/bin/env bash
# setup_env.sh — Install tree-sitter and language grammars needed for graph
# extraction. Safe to re-run. Reports which grammars actually installed so
# the calling skill knows up front which languages will be fully supported
# vs. skipped for this run.
set -uo pipefail

PACKAGES=(
  tree-sitter
  tree-sitter-python
  tree-sitter-javascript
  tree-sitter-typescript
  tree-sitter-go
  tree-sitter-java
  tree-sitter-ruby
  tree-sitter-rust
  tree-sitter-c
  tree-sitter-cpp
  tree-sitter-c-sharp
  tree-sitter-php
)

echo "[setup_env] installing tree-sitter grammars..."
pip install --break-system-packages -q "${PACKAGES[@]}" 2>&1 | tail -20

echo "[setup_env] verifying which grammars actually import..."
python3 - << 'EOF'
mods = [
    "tree_sitter_python", "tree_sitter_javascript", "tree_sitter_typescript",
    "tree_sitter_go", "tree_sitter_java", "tree_sitter_ruby", "tree_sitter_rust",
    "tree_sitter_c", "tree_sitter_cpp", "tree_sitter_c_sharp", "tree_sitter_php",
]
ok, fail = [], []
for m in mods:
    try:
        __import__(m)
        ok.append(m)
    except Exception as e:
        fail.append((m, str(e)))

print(f"[setup_env] {len(ok)}/{len(mods)} grammars available: {', '.join(ok)}")
if fail:
    print(f"[setup_env] UNAVAILABLE (these languages will be skipped, not faked): {', '.join(m for m, _ in fail)}")
EOF
```

---

### profile_repo.py

Scan a repository's structure and size without parsing any function bodies, then recommend a chunking strategy. Always run this first.

```python
#!/usr/bin/env python3
"""
profile_repo.py — Inspect a repository and propose a chunking strategy before
any graph extraction happens. This is the planning step: it never parses
function bodies, only scans structure, so it stays fast even on huge repos.

Usage:
    python profile_repo.py --root <dir> --out profile.json

Output JSON:
{
  "root": str,
  "total_files": int, "total_source_files": int, "total_loc_estimate": int,
  "languages": {"python": {"files": N, "loc": N}, ...},
  "size_tier": "small" | "medium" | "large" | "very_large",
  "boundaries": [
    {"path": str, "kind": "package"|"workspace_member"|"top_level_dir",
     "marker_file": str|null, "language": str|null, "file_count": int, "loc_estimate": int}
  ],
  "is_monorepo": bool,
  "recommended_strategy": "single_pass" | "chunked_by_boundary",
  "notes": [str, ...]
}

Boundary detection looks for well-known project markers (pyproject.toml,
package.json, go.mod, Cargo.toml, pom.xml, *.csproj, composer.json) at any
depth, and treats each marker's directory as a candidate chunk. If no
markers are found, falls back to top-level directories under root.
"""
import argparse
import json
import os
from pathlib import Path

DEFAULT_EXCLUDE_DIRS = {
    ".git", "node_modules", "vendor", "venv", ".venv", "env", "__pycache__",
    "dist", "build", ".next", "target", ".tox", "site-packages", ".mypy_cache",
    ".pytest_cache", "coverage", ".idea", ".vscode", "out", "bin", "obj",
}

SOURCE_EXTS = {
    "py": "python", "js": "javascript", "jsx": "javascript", "mjs": "javascript", "cjs": "javascript",
    "ts": "typescript", "tsx": "typescript", "go": "go", "java": "java", "rb": "ruby",
    "rs": "rust", "c": "c", "h": "c", "cpp": "cpp", "cc": "cpp", "hpp": "cpp",
    "cs": "c_sharp", "php": "php",
}

PROJECT_MARKERS = {
    "pyproject.toml": "python", "setup.py": "python", "Pipfile": "python",
    "package.json": "javascript", "go.mod": "go", "Cargo.toml": "rust",
    "pom.xml": "java", "build.gradle": "java", "build.gradle.kts": "java",
    "composer.json": "php",
}

# Size tiers are measured in *function count* once known, but at the
# profiling stage we only have LOC, so this is a fast pre-estimate using
# lines of code as a proxy (roughly: very rough average of 1 function per
# 15-25 LOC across languages). This is only used to pick a strategy, not
# reported as ground truth anywhere downstream.
LOC_TIER_THRESHOLDS = [
    ("small", 30_000),
    ("medium", 150_000),
    ("large", 600_000),
]  # anything above the last threshold is "very_large"


def estimate_loc(path: Path) -> int:
    try:
        with open(path, "rb") as f:
            return sum(1 for _ in f)
    except OSError:
        return 0


def scan(root: Path, exclude_dirs):
    languages = {}
    total_source_files = 0
    total_files = 0
    marker_dirs = {}  # relative dir path -> marker filename

    for dirpath, dirnames, filenames in os.walk(root):
        dirnames[:] = [d for d in dirnames if d not in exclude_dirs and not d.startswith(".")]
        rel_dir = os.path.relpath(dirpath, root)
        for fn in filenames:
            total_files += 1
            if fn in PROJECT_MARKERS and rel_dir not in marker_dirs:
                marker_dirs[rel_dir] = fn
            ext = fn.rsplit(".", 1)[-1].lower() if "." in fn else ""
            lang = SOURCE_EXTS.get(ext)
            if lang:
                total_source_files += 1
                loc = estimate_loc(Path(dirpath) / fn)
                entry = languages.setdefault(lang, {"files": 0, "loc": 0})
                entry["files"] += 1
                entry["loc"] += loc

    return languages, total_files, total_source_files, marker_dirs


def build_boundaries(root: Path, marker_dirs, exclude_dirs):
    """Each marker directory becomes a candidate chunk boundary. Nested
    markers (a package inside another package, e.g. a monorepo workspace
    member) are kept as separate boundaries rather than merged, since that
    nesting is usually intentional (workspaces, lerna/pnpm/yarn packages)."""
    boundaries = []
    for rel_dir, marker in sorted(marker_dirs.items()):
        abs_dir = root / rel_dir if rel_dir != "." else root
        file_count = 0
        loc_estimate = 0
        for dirpath, dirnames, filenames in os.walk(abs_dir):
            dirnames[:] = [d for d in dirnames if d not in exclude_dirs and not d.startswith(".")]
            for fn in filenames:
                ext = fn.rsplit(".", 1)[-1].lower() if "." in fn else ""
                if ext in SOURCE_EXTS:
                    file_count += 1
                    loc_estimate += estimate_loc(Path(dirpath) / fn)
        boundaries.append({
            "path": rel_dir,
            "kind": "package",
            "marker_file": marker,
            "language": PROJECT_MARKERS.get(marker),
            "file_count": file_count,
            "loc_estimate": loc_estimate,
        })
    return boundaries


def fallback_top_level_boundaries(root: Path, exclude_dirs):
    boundaries = []
    try:
        entries = [e for e in root.iterdir() if e.is_dir() and e.name not in exclude_dirs and not e.name.startswith(".")]
    except OSError:
        entries = []
    for d in sorted(entries):
        file_count = 0
        loc_estimate = 0
        for dirpath, dirnames, filenames in os.walk(d):
            dirnames[:] = [dn for dn in dirnames if dn not in exclude_dirs and not dn.startswith(".")]
            for fn in filenames:
                ext = fn.rsplit(".", 1)[-1].lower() if "." in fn else ""
                if ext in SOURCE_EXTS:
                    file_count += 1
                    loc_estimate += estimate_loc(Path(dirpath) / fn)
        if file_count > 0:
            boundaries.append({
                "path": str(d.relative_to(root)),
                "kind": "top_level_dir",
                "marker_file": None,
                "language": None,
                "file_count": file_count,
                "loc_estimate": loc_estimate,
            })
    return boundaries


def size_tier(total_loc):
    for tier, threshold in LOC_TIER_THRESHOLDS:
        if total_loc < threshold:
            return tier
    return "very_large"


def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--root", required=True)
    ap.add_argument("--out", required=True)
    ap.add_argument("--exclude-dirs", default=None)
    args = ap.parse_args()

    root = Path(args.root).resolve()
    exclude_dirs = set(DEFAULT_EXCLUDE_DIRS)
    if args.exclude_dirs:
        exclude_dirs |= set(x.strip() for x in args.exclude_dirs.split(",") if x.strip())

    languages, total_files, total_source_files, marker_dirs = scan(root, exclude_dirs)
    total_loc = sum(v["loc"] for v in languages.values())
    tier = size_tier(total_loc)

    boundaries = build_boundaries(root, marker_dirs, exclude_dirs)
    is_monorepo = len(marker_dirs) > 1

    if not boundaries:
        boundaries = fallback_top_level_boundaries(root, exclude_dirs)

    notes = []
    if tier in ("small",):
        recommended = "single_pass"
        notes.append("Repo is small enough to parse in a single pass; chunking is unnecessary overhead.")
    elif tier in ("medium",) and not is_monorepo:
        recommended = "single_pass"
        notes.append("Repo is medium-sized but a single logical project; attempt single pass first, fall back to chunked if extraction proves unreliable (timeouts, memory pressure).")
    else:
        recommended = "chunked_by_boundary"
        if is_monorepo:
            notes.append(f"Detected {len(marker_dirs)} distinct project markers — this looks like a monorepo. Chunk per package/workspace member.")
        else:
            notes.append("Repo is large; chunk by top-level architectural boundary even though it is a single logical project, to keep each extraction pass reliable.")

    if not boundaries:
        notes.append("No clear package/workspace boundaries found; will need to chunk by directory depth or file batches instead — see references/large-repo-strategy.md.")

    unsupported_exts = []
    profile = {
        "root": str(root),
        "total_files": total_files,
        "total_source_files": total_source_files,
        "total_loc_estimate": total_loc,
        "languages": languages,
        "size_tier": tier,
        "boundaries": sorted(boundaries, key=lambda b: -b["loc_estimate"]),
        "is_monorepo": is_monorepo,
        "recommended_strategy": recommended,
        "notes": notes,
    }

    Path(args.out).parent.mkdir(parents=True, exist_ok=True)
    with open(args.out, "w") as f:
        json.dump(profile, f, indent=2)

    print(f"[profile_repo] {total_source_files} source files, ~{total_loc} LOC, tier={tier}, "
          f"monorepo={is_monorepo}, strategy={recommended}, boundaries={len(boundaries)} -> {args.out}")


if __name__ == "__main__":
    main()
```

---

### build_graph.py

Core extractor. Parses source files via tree-sitter ASTs and emits a graph JSON containing function nodes and call edges. Run once for single-pass, or once per chunk for large/monorepo repos.

```python
#!/usr/bin/env python3
"""
build_graph.py — Extract a code property graph (functions + call edges) from
a directory of source files using tree-sitter.

Every fact emitted in the output graph is derived directly from a parsed AST
node. Nothing is inferred, guessed, or filled in from naming conventions
alone. Calls that cannot be statically resolved to a known function are kept
as edges with resolved=false rather than silently dropped or guessed at —
this is the core anti-hallucination mechanism: downstream doc generation
must never claim a resolved target for an unresolved call.

Usage:
    python build_graph.py --root <dir> --out <graph.json> [--languages py,js,ts,go,...]
                           [--exclude-dirs node_modules,.git,...] [--max-file-kb 1024]

Output JSON schema:
{
  "meta": {...},
  "files": [{"path": str, "language": str, "loc": int, "parsed": bool, "parse_error": str|null}],
  "functions": [{"id": str, "name": str, "file": str, "start_line": int, "end_line": int,
                 "class": str|null, "is_method": bool, "signature": str}],
  "calls": [{"caller_id": str, "callee_name": str, "callee_id": str|null,
             "file": str, "line": int, "resolved": bool}],
  "unresolved_summary": {"callee_name": count, ...}
}
"""
import argparse
import json
import os
import sys
import hashlib
from pathlib import Path

# ---------------------------------------------------------------------------
# Language registry: maps file extensions to tree-sitter grammar modules and
# the AST node types we care about for that grammar. Every language we claim
# to support here has been verified to expose these node types.
# ---------------------------------------------------------------------------

LANGUAGE_MODULES = {
    "py":   ("tree_sitter_python", "python"),
    "js":   ("tree_sitter_javascript", "javascript"),
    "jsx":  ("tree_sitter_javascript", "javascript"),
    "mjs":  ("tree_sitter_javascript", "javascript"),
    "cjs":  ("tree_sitter_javascript", "javascript"),
    "ts":   ("tree_sitter_typescript", "typescript"),
    "tsx":  ("tree_sitter_typescript", "tsx"),
    "go":   ("tree_sitter_go", "go"),
    "java": ("tree_sitter_java", "java"),
    "rb":   ("tree_sitter_ruby", "ruby"),
    "rs":   ("tree_sitter_rust", "rust"),
    "c":    ("tree_sitter_c", "c"),
    "h":    ("tree_sitter_c", "c"),
    "cpp":  ("tree_sitter_cpp", "cpp"),
    "cc":   ("tree_sitter_cpp", "cpp"),
    "hpp":  ("tree_sitter_cpp", "cpp"),
    "cs":   ("tree_sitter_c_sharp", "c_sharp"),
    "php":  ("tree_sitter_php", "php"),
}

# Node types per language for function-like definitions and call expressions.
# These are intentionally conservative: if a construct isn't listed, it is
# not extracted, rather than risk misclassifying it.
FUNCTION_NODE_TYPES = {
    "python": {"function_definition"},
    "javascript": {"function_declaration", "method_definition", "function_expression", "arrow_function"},
    "typescript": {"function_declaration", "method_definition", "function_expression", "arrow_function"},
    "tsx": {"function_declaration", "method_definition", "function_expression", "arrow_function"},
    "go": {"function_declaration", "method_declaration"},
    "java": {"method_declaration", "constructor_declaration"},
    "ruby": {"method"},
    "rust": {"function_item"},
    "c": {"function_definition"},
    "cpp": {"function_definition"},
    "c_sharp": {"method_declaration", "constructor_declaration", "local_function_statement"},
    "php": {"function_definition", "method_declaration"},
}

CALL_NODE_TYPES = {
    "python": {"call"},
    "javascript": {"call_expression"},
    "typescript": {"call_expression"},
    "tsx": {"call_expression"},
    "go": {"call_expression"},
    "java": {"method_invocation", "object_creation_expression"},
    "ruby": {"call", "method_call"},
    "rust": {"call_expression", "macro_invocation"},
    "c": {"call_expression"},
    "cpp": {"call_expression"},
    "c_sharp": {"invocation_expression"},
    "php": {"function_call_expression", "member_call_expression"},
}

CLASS_NODE_TYPES = {
    "python": {"class_definition"},
    "javascript": {"class_declaration"},
    "typescript": {"class_declaration"},
    "tsx": {"class_declaration"},
    "go": {"type_declaration"},
    "java": {"class_declaration", "interface_declaration"},
    "ruby": {"class"},
    "rust": {"impl_item", "struct_item"},
    "c": set(),
    "cpp": {"class_specifier", "struct_specifier"},
    "c_sharp": {"class_declaration"},
    "php": {"class_declaration"},
}

DEFAULT_EXCLUDE_DIRS = {
    ".git", "node_modules", "vendor", "venv", ".venv", "env", "__pycache__",
    "dist", "build", ".next", "target", ".tox", "site-packages", ".mypy_cache",
    ".pytest_cache", "coverage", ".idea", ".vscode", "out", "bin", "obj",
}


def fid(*parts):
    """Stable, deterministic id for a function node based on its identity, not its content."""
    raw = "::".join(str(p) for p in parts)
    return hashlib.sha1(raw.encode("utf-8")).hexdigest()[:16]


class LanguageSupport:
    """Lazily loads and caches tree-sitter Language/Parser objects per language name."""

    def __init__(self):
        self._cache = {}
        self._unavailable = set()

    def get(self, lang_name):
        if lang_name in self._cache:
            return self._cache[lang_name]
        if lang_name in self._unavailable:
            return None
        try:
            from tree_sitter import Language, Parser
            mod_name = None
            for ext, (m, l) in LANGUAGE_MODULES.items():
                if l == lang_name:
                    mod_name = m
                    break
            if mod_name is None:
                self._unavailable.add(lang_name)
                return None
            module = __import__(mod_name)
            # tree-sitter-typescript exposes language_typescript()/language_tsx();
            # most others expose a single language() function.
            if mod_name == "tree_sitter_typescript":
                fn = module.language_typescript if lang_name == "typescript" else module.language_tsx
                ts_lang = Language(fn())
            else:
                ts_lang = Language(module.language())
            parser = Parser(ts_lang)
            self._cache[lang_name] = parser
            return parser
        except Exception as e:
            sys.stderr.write(f"[build_graph] grammar unavailable for {lang_name}: {e}\n")
            self._unavailable.add(lang_name)
            return None


def node_text(node, source):
    return source[node.start_byte:node.end_byte].decode("utf-8", errors="replace")


def extract_name(node, lang, source):
    """Pull the identifier name out of a function/class definition node."""
    name_node = node.child_by_field_name("name")
    if name_node is not None:
        return node_text(name_node, source)
    # Fallback: scan immediate children for an identifier-like node (covers
    # constructs like JS method_definition where 'name' field may differ).
    for child in node.children:
        if child.type in ("identifier", "property_identifier", "field_identifier", "type_identifier"):
            return node_text(child, source)
    return "<anonymous>"


def extract_callee_name(node, lang, source):
    """Pull the textual name being called out of a call-expression node.
    For attribute/member calls (e.g. obj.method()) this returns the rightmost
    identifier plus an attribute marker, never a guessed canonical path."""
    func_node = node.child_by_field_name("function")
    if func_node is None:
        # languages like Ruby/PHP vary field names; fall back to first child
        func_node = node.children[0] if node.children else None
    if func_node is None:
        return None
    if func_node.type in ("identifier", "type_identifier"):
        return node_text(func_node, source)
    if func_node.type in ("attribute", "member_expression", "field_expression", "selector_expression", "scoped_identifier"):
        # take the rightmost identifier as the callable name (e.g. `.qux` in `baz.qux()`)
        prop = func_node.child_by_field_name("attribute") or func_node.child_by_field_name("property") or func_node.child_by_field_name("field")
        if prop is not None:
            return node_text(prop, source)
        # last resort: last identifier-like descendant
        last = None
        for child in func_node.children:
            if "identifier" in child.type:
                last = child
        if last is not None:
            return node_text(last, source)
    return node_text(func_node, source)[:80]  # cap length, never fabricate


def detect_language(path: Path):
    ext = path.suffix.lstrip(".").lower()
    entry = LANGUAGE_MODULES.get(ext)
    return entry[1] if entry else None


def collect_files(root: Path, exclude_dirs, max_file_kb, only_languages=None):
    files = []
    for dirpath, dirnames, filenames in os.walk(root):
        dirnames[:] = [d for d in dirnames if d not in exclude_dirs and not d.startswith(".")]
        for fn in filenames:
            p = Path(dirpath) / fn
            lang = detect_language(p)
            if lang is None:
                continue
            if only_languages and lang not in only_languages:
                continue
            try:
                size_kb = p.stat().st_size / 1024
            except OSError:
                continue
            if size_kb > max_file_kb:
                continue
            files.append(p)
    return files


def walk_collect(node, target_types, out):
    if node.type in target_types:
        out.append(node)
    for child in node.children:
        walk_collect(child, target_types, out)


def find_enclosing_class(node, class_types):
    p = node.parent
    while p is not None:
        if p.type in class_types:
            return p
        p = p.parent
    return None


def process_file(path: Path, root: Path, lang_name: str, parser, langsup, chunk_label):
    rel = str(path.relative_to(root))
    display_path = f"{chunk_label}/{rel}" if chunk_label else rel
    try:
        source = path.read_bytes()
    except OSError as e:
        return {"path": display_path, "language": lang_name, "loc": 0, "parsed": False, "parse_error": str(e)}, [], []

    try:
        tree = parser.parse(source)
    except Exception as e:
        return {"path": display_path, "language": lang_name, "loc": 0, "parsed": False, "parse_error": str(e)}, [], []

    root_node = tree.root_node
    loc = source.count(b"\n") + 1
    file_meta = {"path": display_path, "language": lang_name, "loc": loc, "parsed": True, "parse_error": None}

    func_types = FUNCTION_NODE_TYPES.get(lang_name, set())
    call_types = CALL_NODE_TYPES.get(lang_name, set())
    class_types = CLASS_NODE_TYPES.get(lang_name, set())

    func_nodes = []
    walk_collect(root_node, func_types, func_nodes)

    functions = []
    func_node_to_id = {}
    for fn in func_nodes:
        name = extract_name(fn, lang_name, source)
        cls = find_enclosing_class(fn, class_types)
        cls_name = extract_name(cls, lang_name, source) if cls is not None else None
        start_line = fn.start_point[0] + 1
        end_line = fn.end_point[0] + 1
        # chunk_label is included so that identically-named relative paths in
        # different chunks (e.g. two packages each with an index.js) can
        # never collide once merged by merge_graphs.py.
        fn_id = fid(chunk_label, rel, cls_name, name, start_line)
        func_node_to_id[fn.id] = fn_id
        sig_line = node_text(fn, source).split("\n")[0][:160]
        functions.append({
            "id": fn_id,
            "name": name,
            "file": display_path,
            "start_line": start_line,
            "end_line": end_line,
            "class": cls_name,
            "is_method": cls is not None,
            "signature": sig_line,
            "chunk": chunk_label,
        })

    # Map every byte range to its innermost enclosing function for caller attribution.
    def enclosing_function_id(node):
        p = node
        while p is not None:
            if p.id in func_node_to_id:
                return func_node_to_id[p.id]
            p = p.parent
        return None

    call_nodes = []
    walk_collect(root_node, call_types, call_nodes)

    calls = []
    for cn in call_nodes:
        callee_name = extract_callee_name(cn, lang_name, source)
        if not callee_name:
            continue
        caller_id = enclosing_function_id(cn)
        calls.append({
            "caller_id": caller_id,  # may be null: a module-level call outside any function
            "callee_name": callee_name,
            "callee_id": None,       # resolved later, in a second pass, against the full symbol table
            "file": display_path,
            "line": cn.start_point[0] + 1,
            "resolved": False,
        })

    return file_meta, functions, calls


def resolve_calls(functions, calls):
    """Second pass: resolve callee_name -> callee_id using an exact-name symbol
    table built from extracted functions. Ambiguous names (multiple functions
    share it) are left unresolved with resolved=false and a note, rather than
    guessing one. This is the key anti-hallucination boundary for the graph
    layer: we never pick an arbitrary candidate."""
    by_name = {}
    for f in functions:
        by_name.setdefault(f["name"], []).append(f["id"])

    unresolved_summary = {}
    for c in calls:
        candidates = by_name.get(c["callee_name"], [])
        if len(candidates) == 1:
            c["callee_id"] = candidates[0]
            c["resolved"] = True
        else:
            c["resolved"] = False
            if len(candidates) > 1:
                c["ambiguous_candidates"] = candidates
            unresolved_summary[c["callee_name"]] = unresolved_summary.get(c["callee_name"], 0) + 1
    return unresolved_summary


def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--root", required=True)
    ap.add_argument("--out", required=True)
    ap.add_argument("--languages", default=None, help="comma-separated language names to restrict to, e.g. python,javascript")
    ap.add_argument("--exclude-dirs", default=None, help="comma-separated extra directory names to exclude")
    ap.add_argument("--max-file-kb", type=float, default=1024.0)
    ap.add_argument("--chunk-label", default=None, help="optional label identifying which chunk this graph represents")
    args = ap.parse_args()

    root = Path(args.root).resolve()
    exclude_dirs = set(DEFAULT_EXCLUDE_DIRS)
    if args.exclude_dirs:
        exclude_dirs |= set(x.strip() for x in args.exclude_dirs.split(",") if x.strip())
    only_languages = set(x.strip() for x in args.languages.split(",")) if args.languages else None

    langsup = LanguageSupport()
    files_to_process = collect_files(root, exclude_dirs, args.max_file_kb, only_languages)

    all_files_meta = []
    all_functions = []
    all_calls = []
    skipped_languages = {}

    for path in files_to_process:
        lang_name = detect_language(path)
        parser = langsup.get(lang_name)
        rel = str(path.relative_to(root))
        display_path = f"{args.chunk_label}/{rel}" if args.chunk_label else rel
        if parser is None:
            skipped_languages[lang_name] = skipped_languages.get(lang_name, 0) + 1
            all_files_meta.append({
                "path": display_path, "language": lang_name,
                "loc": 0, "parsed": False, "parse_error": "grammar_unavailable"
            })
            continue
        file_meta, functions, calls = process_file(path, root, lang_name, parser, langsup, args.chunk_label)
        all_files_meta.append(file_meta)
        all_functions.extend(functions)
        all_calls.extend(calls)

    unresolved_summary = resolve_calls(all_functions, all_calls)

    graph = {
        "meta": {
            "root": str(root),
            "chunk_label": args.chunk_label,
            "file_count": len(all_files_meta),
            "parsed_file_count": sum(1 for f in all_files_meta if f["parsed"]),
            "function_count": len(all_functions),
            "call_count": len(all_calls),
            "resolved_call_count": sum(1 for c in all_calls if c["resolved"]),
            "skipped_languages": skipped_languages,
        },
        "files": all_files_meta,
        "functions": all_functions,
        "calls": all_calls,
        "unresolved_summary": unresolved_summary,
    }

    Path(args.out).parent.mkdir(parents=True, exist_ok=True)
    with open(args.out, "w") as f:
        json.dump(graph, f, indent=2)

    print(f"[build_graph] {graph['meta']['parsed_file_count']}/{graph['meta']['file_count']} files parsed, "
          f"{graph['meta']['function_count']} functions, {graph['meta']['call_count']} calls "
          f"({graph['meta']['resolved_call_count']} resolved) -> {args.out}")


if __name__ == "__main__":
    main()
```

---

### merge_graphs.py

Combines per-chunk graph JSON files into one unified graph, running a second resolution pass to resolve calls that cross chunk boundaries.

```python
#!/usr/bin/env python3
"""
merge_graphs.py — Combine multiple per-chunk graph.json files (as produced by
build_graph.py) into a single unified graph, and attempt to resolve calls
that point across chunk boundaries (e.g. package A calling into package B).

This is the step that makes chunking safe to use: without it, a function in
chunk A calling a function in chunk B would be permanently unresolved even
though both sides were actually parsed correctly. Cross-chunk resolution
uses the same conservative exact-name-match rule as build_graph.py's
single-chunk resolver: ambiguous matches stay unresolved rather than being
guessed.

Usage:
    python merge_graphs.py --chunks chunk1.json chunk2.json ... --out merged.json
    # or
    python merge_graphs.py --chunks-dir <dir-of-json-files> --out merged.json
"""
import argparse
import json
from pathlib import Path


def load_chunks(paths):
    chunks = []
    for p in paths:
        with open(p) as f:
            chunks.append(json.load(f))
    return chunks


def merge(chunks):
    all_files = []
    all_functions = []
    all_calls = []
    chunk_labels = []
    skipped_languages = {}

    for c in chunks:
        label = c.get("meta", {}).get("chunk_label") or c.get("meta", {}).get("root")
        chunk_labels.append(label)
        all_files.extend(c.get("files", []))
        all_functions.extend(c.get("functions", []))
        all_calls.extend(c.get("calls", []))
        for lang, count in c.get("meta", {}).get("skipped_languages", {}).items():
            skipped_languages[lang] = skipped_languages.get(lang, 0) + count

    # Function ids are content-derived (file path + class + name + line) via
    # build_graph.py's fid(), so they are already globally unique as long as
    # file paths passed to each chunk run were relative to a *consistent*
    # root. We verify there are no collisions with conflicting metadata,
    # which would indicate the chunks were built against inconsistent roots.
    by_id = {}
    id_conflicts = []
    deduped_functions = []
    for f in all_functions:
        existing = by_id.get(f["id"])
        if existing is None:
            by_id[f["id"]] = f
            deduped_functions.append(f)
        elif existing != f:
            id_conflicts.append({"id": f["id"], "a": existing, "b": f})

    by_name = {}
    for f in deduped_functions:
        by_name.setdefault(f["name"], []).append(f["id"])

    newly_resolved = 0
    for c in all_calls:
        if c.get("resolved"):
            continue
        candidates = by_name.get(c["callee_name"], [])
        if len(candidates) == 1:
            c["callee_id"] = candidates[0]
            c["resolved"] = True
            c["resolved_cross_chunk"] = True
            newly_resolved += 1
        elif len(candidates) > 1:
            c["ambiguous_candidates"] = candidates

    unresolved_summary = {}
    for c in all_calls:
        if not c["resolved"]:
            unresolved_summary[c["callee_name"]] = unresolved_summary.get(c["callee_name"], 0) + 1

    merged = {
        "meta": {
            "chunk_count": len(chunks),
            "chunk_labels": chunk_labels,
            "file_count": len(all_files),
            "parsed_file_count": sum(1 for f in all_files if f.get("parsed")),
            "function_count": len(deduped_functions),
            "call_count": len(all_calls),
            "resolved_call_count": sum(1 for c in all_calls if c["resolved"]),
            "newly_resolved_cross_chunk": newly_resolved,
            "id_conflicts": len(id_conflicts),
            "skipped_languages": skipped_languages,
        },
        "files": all_files,
        "functions": deduped_functions,
        "calls": all_calls,
        "unresolved_summary": unresolved_summary,
        "id_conflicts": id_conflicts,
    }
    return merged


def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--chunks", nargs="*", default=None, help="explicit list of chunk graph.json paths")
    ap.add_argument("--chunks-dir", default=None, help="directory containing chunk graph.json files (non-recursive *.json)")
    ap.add_argument("--out", required=True)
    args = ap.parse_args()

    paths = list(args.chunks) if args.chunks else []
    if args.chunks_dir:
        paths.extend(sorted(str(p) for p in Path(args.chunks_dir).glob("*.json")))
    if not paths:
        raise SystemExit("No chunk files provided via --chunks or --chunks-dir")

    chunks = load_chunks(paths)
    merged = merge(chunks)

    Path(args.out).parent.mkdir(parents=True, exist_ok=True)
    with open(args.out, "w") as f:
        json.dump(merged, f, indent=2)

    m = merged["meta"]
    print(f"[merge_graphs] merged {m['chunk_count']} chunks: {m['function_count']} functions, "
          f"{m['call_count']} calls ({m['resolved_call_count']} resolved, "
          f"{m['newly_resolved_cross_chunk']} newly resolved cross-chunk), "
          f"{m['id_conflicts']} id conflicts -> {args.out}")
    if m["id_conflicts"] > 0:
        print("[merge_graphs] WARNING: id conflicts detected — chunks may have been built against "
              "inconsistent root paths. Check id_conflicts in the output before trusting the merge.")


if __name__ == "__main__":
    main()
```

---

### render_docs.py

Turns a graph JSON into markdown documentation and a Mermaid call-flow diagram. Every sentence it writes is templated from a literal field in the graph — no free-text generation of structural claims.

```python
#!/usr/bin/env python3
"""
render_docs.py — Turn a graph.json (from build_graph.py or merge_graphs.py)
into markdown documentation and Mermaid call-flow diagrams.

ANTI-HALLUCINATION DESIGN: every sentence this script writes is templated
from a literal field in the graph JSON (a function name, a file path, a line
number, a call count). There is no free-text generation step here — this
script never infers what a function "probably does" from its name. If you
need natural-language *explanations* of behavior (not just structure), that
requires a separate, explicitly-labeled step where an LLM reads actual
function source and is told to mark uncertainty — see SKILL.md's
"Generating semantic summaries" section. This script alone only describes
structure: what calls what, how many times, whether resolved.

Usage:
    python render_docs.py --graph merged.json --out-dir docs/ [--top-n 25]

Outputs:
    docs/call_graph_overview.md   — repo-level stats, top callers/callees, unresolved calls
    docs/functions/<chunk>.md     — one file per chunk listing functions and their calls
    docs/diagrams/overview.mmd    — Mermaid diagram of the highest-fan-out/fan-in functions
"""
import argparse
import json
from collections import defaultdict
from pathlib import Path


def safe_label(name: str) -> str:
    return name.replace('"', "'")[:60]


def build_indices(graph):
    func_by_id = {f["id"]: f for f in graph["functions"]}
    out_edges = defaultdict(list)   # caller_id -> [call records]
    in_edges = defaultdict(list)    # callee_id -> [call records]
    for c in graph["calls"]:
        if c.get("caller_id"):
            out_edges[c["caller_id"]].append(c)
        if c.get("resolved") and c.get("callee_id"):
            in_edges[c["callee_id"]].append(c)
    return func_by_id, out_edges, in_edges


def render_overview(graph, func_by_id, out_edges, in_edges, top_n):
    meta = graph["meta"]
    lines = []
    lines.append("# Call Graph Overview")
    lines.append("")
    lines.append("All figures below are computed directly from parsed source — every count is exact "
                  "for the files that parsed successfully; nothing here is estimated or inferred.")
    lines.append("")
    lines.append("## Coverage")
    lines.append("")
    lines.append(f"- Files scanned: {meta.get('file_count', 0)}")
    lines.append(f"- Files successfully parsed: {meta.get('parsed_file_count', 0)}")
    unparsed = meta.get('file_count', 0) - meta.get('parsed_file_count', 0)
    if unparsed > 0:
        lines.append(f"- **Files NOT parsed (excluded from this graph): {unparsed}** — "
                      f"see \"Unparsed files\" section below before treating this graph as complete.")
    lines.append(f"- Functions/methods identified: {meta.get('function_count', 0)}")
    lines.append(f"- Call sites identified: {meta.get('call_count', 0)}")
    resolved = meta.get('resolved_call_count', 0)
    total_calls = meta.get('call_count', 0)
    pct = (resolved / total_calls * 100) if total_calls else 0
    lines.append(f"- Call sites resolved to a known function in this repo: {resolved} ({pct:.0f}%)")
    lines.append(f"  - The remaining {total_calls - resolved} calls target either external/third-party "
                  f"code, builtins, dynamically-dispatched methods, or functions outside the scanned scope. "
                  f"These are listed, not guessed at, in \"Unresolved calls\" below.")
    if meta.get("chunk_count"):
        lines.append(f"- Built from {meta['chunk_count']} chunk(s): {', '.join(str(c) for c in meta.get('chunk_labels', []))}")
    skipped = meta.get("skipped_languages", {})
    if skipped:
        lines.append("")
        lines.append("**Languages detected but not parsed (no grammar available in this environment):**")
        for lang, count in sorted(skipped.items(), key=lambda x: -x[1]):
            lines.append(f"- {lang}: {count} file(s) skipped entirely — none of their functions or calls appear anywhere in this graph")
    lines.append("")

    # Unparsed files detail
    unparsed_files = [f for f in graph["files"] if not f.get("parsed")]
    if unparsed_files:
        lines.append("## Unparsed files")
        lines.append("")
        lines.append("These files were not included in the graph below. Do not assume their functions "
                      "are absent from the codebase — they were simply not analyzed.")
        lines.append("")
        for f in unparsed_files[:200]:
            reason = f.get("parse_error") or "unknown"
            lines.append(f"- `{f['path']}` — {reason}")
        if len(unparsed_files) > 200:
            lines.append(f"- ...and {len(unparsed_files) - 200} more")
        lines.append("")

    # Top callers (highest fan-out)
    fan_out = sorted(out_edges.items(), key=lambda kv: -len(kv[1]))[:top_n]
    lines.append(f"## Highest fan-out (top {top_n} functions by number of outgoing calls)")
    lines.append("")
    lines.append("| Function | File | Line | Outgoing calls |")
    lines.append("|---|---|---|---|")
    for fn_id, calls in fan_out:
        f = func_by_id.get(fn_id)
        if not f:
            continue
        lines.append(f"| `{safe_label(f['name'])}` | `{f['file']}` | {f['start_line']} | {len(calls)} |")
    lines.append("")

    # Top callees (highest fan-in) — only counts resolved, in-repo calls
    fan_in = sorted(in_edges.items(), key=lambda kv: -len(kv[1]))[:top_n]
    lines.append(f"## Highest fan-in (top {top_n} functions by number of resolved incoming calls)")
    lines.append("")
    lines.append("| Function | File | Line | Incoming calls (resolved, in-repo only) |")
    lines.append("|---|---|---|---|")
    for fn_id, calls in fan_in:
        f = func_by_id.get(fn_id)
        if not f:
            continue
        lines.append(f"| `{safe_label(f['name'])}` | `{f['file']}` | {f['start_line']} | {len(calls)} |")
    lines.append("")

    # Unresolved calls summary
    unresolved_summary = graph.get("unresolved_summary", {})
    if unresolved_summary:
        top_unresolved = sorted(unresolved_summary.items(), key=lambda kv: -kv[1])[:top_n]
        lines.append(f"## Most frequent unresolved call targets (top {len(top_unresolved)})")
        lines.append("")
        lines.append("These are call targets that could not be matched to a function definition found "
                      "in the scanned source. This is expected for standard library calls, third-party "
                      "package calls, and dynamic dispatch — it is not necessarily a sign of a problem.")
        lines.append("")
        lines.append("| Call target name | Occurrences |")
        lines.append("|---|---|")
        for name, count in top_unresolved:
            lines.append(f"| `{safe_label(name)}` | {count} |")
        lines.append("")

    id_conflicts = graph.get("id_conflicts", [])
    if id_conflicts:
        lines.append("## ⚠ ID conflicts detected during merge")
        lines.append("")
        lines.append(f"{len(id_conflicts)} function id(s) had conflicting metadata across chunks. "
                      "This usually means chunks were built against inconsistent root paths. "
                      "Treat the merged graph with caution until this is resolved.")
        lines.append("")

    return "\n".join(lines)


def render_chunk_functions(graph, func_by_id, out_edges, chunk_label):
    funcs = [f for f in graph["functions"] if f.get("chunk") == chunk_label or (chunk_label is None and "chunk" not in f)]
    funcs.sort(key=lambda f: (f["file"], f["start_line"]))
    lines = [f"# Functions — {chunk_label or 'repository'}", ""]
    lines.append(f"{len(funcs)} function(s)/method(s) found in this chunk. Each entry lists its exact "
                  "outgoing calls as found in source — no behavior is described beyond what was parsed.")
    lines.append("")
    current_file = None
    for f in funcs:
        if f["file"] != current_file:
            current_file = f["file"]
            lines.append(f"## `{current_file}`")
            lines.append("")
        cls_prefix = f"{f['class']}." if f.get("class") else ""
        lines.append(f"### `{cls_prefix}{f['name']}` (line {f['start_line']}-{f['end_line']})")
        lines.append("")
        lines.append(f"```\n{f['signature']}\n```")
        calls = out_edges.get(f["id"], [])
        if calls:
            lines.append("")
            lines.append("Calls:")
            for c in calls:
                if c.get("resolved"):
                    target = func_by_id.get(c["callee_id"])
                    target_desc = f"`{target['name']}` in `{target['file']}:{target['start_line']}`" if target else f"`{c['callee_name']}` (resolved id not found)"
                    cross = " *(cross-chunk)*" if c.get("resolved_cross_chunk") else ""
                    lines.append(f"- line {c['line']}: → {target_desc}{cross}")
                else:
                    lines.append(f"- line {c['line']}: → `{c['callee_name']}` (unresolved — external, builtin, or dynamic)")
        else:
            lines.append("")
            lines.append("Calls: none detected.")
        lines.append("")
    return "\n".join(lines)


def render_mermaid(graph, func_by_id, out_edges, in_edges, top_n=20):
    """Diagram the highest-connectivity functions only — a full-repo diagram
    is unreadable and not the point. We pick functions with the largest
    fan-in + fan-out as the structurally significant ones."""
    scored = []
    seen = set()
    for fn_id, f in func_by_id.items():
        score = len(out_edges.get(fn_id, [])) + len(in_edges.get(fn_id, []))
        if score > 0:
            scored.append((score, fn_id))
    scored.sort(key=lambda x: -x[0])
    top_ids = set(fn_id for _, fn_id in scored[:top_n])

    lines = ["graph LR"]
    edges_emitted = set()
    for fn_id in top_ids:
        f = func_by_id[fn_id]
        node_label = f"{fn_id}[\"{safe_label(f['name'])}<br/>{Path(f['file']).name}\"]"
        lines.append(f"    {node_label}")
    for c in graph["calls"]:
        if not c.get("resolved"):
            continue
        caller = c.get("caller_id")
        callee = c.get("callee_id")
        if caller in top_ids and callee in top_ids:
            key = (caller, callee)
            if key not in edges_emitted:
                edges_emitted.add(key)
                lines.append(f"    {caller} --> {callee}")
    if len(lines) == 1:
        lines.append('    note["No densely-connected functions found to diagram"]')
    return "\n".join(lines)


def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--graph", required=True)
    ap.add_argument("--out-dir", required=True)
    ap.add_argument("--top-n", type=int, default=25)
    args = ap.parse_args()

    with open(args.graph) as f:
        graph = json.load(f)

    out_dir = Path(args.out_dir)
    (out_dir / "functions").mkdir(parents=True, exist_ok=True)
    (out_dir / "diagrams").mkdir(parents=True, exist_ok=True)

    func_by_id, out_edges, in_edges = build_indices(graph)

    overview = render_overview(graph, func_by_id, out_edges, in_edges, args.top_n)
    (out_dir / "call_graph_overview.md").write_text(overview)

    chunk_labels = sorted(set(f.get("chunk") for f in graph["functions"]))
    if chunk_labels == [None]:
        content = render_chunk_functions(graph, func_by_id, out_edges, None)
        (out_dir / "functions" / "repository.md").write_text(content)
    else:
        for label in chunk_labels:
            content = render_chunk_functions(graph, func_by_id, out_edges, label)
            safe_name = (label or "root").replace("/", "_")
            (out_dir / "functions" / f"{safe_name}.md").write_text(content)

    mermaid = render_mermaid(graph, func_by_id, out_edges, in_edges, top_n=min(args.top_n, 30))
    (out_dir / "diagrams" / "overview.mmd").write_text(mermaid)

    print(f"[render_docs] wrote overview, {max(1, len(chunk_labels))} function doc(s), and a diagram to {out_dir}")


if __name__ == "__main__":
    main()
```

---

*Generated from `codegraph.skill` — a Claude AI skill for code property graph extraction.*
