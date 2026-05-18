# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

<!-- BEGIN BEADS INTEGRATION v:1 profile:minimal hash:7510c1e2 -->
## Beads Issue Tracker

This project uses **bd (beads)** for issue tracking. Run `bd prime` to see full workflow context and commands.

### Quick Reference

```bash
bd ready              # Find available work
bd show <id>          # View issue details
bd update <id> --claim  # Claim work
bd close <id>         # Complete work
```

### Rules

- Use `bd` for ALL task tracking — do NOT use TodoWrite, TaskCreate, or markdown TODO lists
- Run `bd prime` for detailed command reference and session close protocol
- Use `bd remember` for persistent knowledge — do NOT use MEMORY.md files

**Architecture in one line:** issues live in a local Dolt DB; sync uses `refs/dolt/data` on your git remote; `.beads/issues.jsonl` is a passive export. See https://github.com/gastownhall/beads/blob/main/docs/SYNC_CONCEPTS.md for details and anti-patterns.

## Session Completion

**When ending a work session**, you MUST complete ALL steps below. Work is NOT complete until `git push` succeeds.

**MANDATORY WORKFLOW:**

1. **File issues for remaining work** - Create issues for anything that needs follow-up
2. **Run quality gates** (if code changed) - Tests, linters, builds
3. **Update issue status** - Close finished work, update in-progress items
4. **PUSH TO REMOTE** - This is MANDATORY:
   ```bash
   git pull --rebase
   git push
   git status  # MUST show "up to date with origin"
   ```
5. **Clean up** - Clear stashes, prune remote branches
6. **Verify** - All changes committed AND pushed
7. **Hand off** - Provide context for next session

**CRITICAL RULES:**
- Work is NOT complete until `git push` succeeds
- NEVER stop before pushing - that leaves work stranded locally
- NEVER say "ready to push when you are" - YOU must push
- If push fails, resolve and retry until it succeeds
<!-- END BEADS INTEGRATION -->

---

## ask-repo: Purpose

This repository is an LLM-based expert system for maintaining a knowledge base (KB) about software repositories and associated documents. Claude operates as a KB administrator: adding repos, scanning them, recording knowledge as Beads issues, and answering questions by retrieving KB content. The human may also directly manage Beads issues.

## Directory Layout

```
repo/NAME/     git submodule for each source repo under study
docs/NAME/     auxiliary documentation associated with repo NAME
docs/any/      documentation not tied to a single repo, or general-purpose
```

Ground truth for all collected knowledge lives in these directories.

---

## Knowledge Base: Beads Issue Conventions

The KB is stored entirely as Beads issues. Issues fall into two roles: **node issues** (facts) and **edge issues** (relationships). Both use custom Beads issue types defined in `.beads/config.yaml` under `types.custom`.

### KB Issue States

| State | Meaning |
|---|---|
| `open` | Newly recorded fact or relationship — unconfirmed |
| `in_progress` | Actively being confirmed (multi-agent/human coordination) |
| `closed` | Confirmed as true/appropriate — **only close at explicit user request** |
| `blocked` | Set automatically by Beads when dependencies are unresolved |

**Claude must not close KB issues unilaterally.** Close only when the user requests it, including batch-close directives.

### Node Issue Types (facts)

Define new types in `.beads/config.yaml` → `types.custom` as needed. Before creating a new type, check whether an existing one fits to avoid proliferation. Core types:

| Type | Represents |
|---|---|
| `repo` | A source code repository under study |
| `scan` | A scanning event of a repo (records date, scope, granularity) |
| `package` | A sub-package or module within a repo |
| `component` | A named software component (service, plugin, binary, subsystem) |
| `class` | A specific class or struct within source code |
| `interface` | An API, protocol, or user-facing interface |
| `facade` | A simplified interface over a more complex subsystem |
| `concept` | A domain concept or abstraction |
| `algorithm` | A specific algorithm or computational method |
| `parameter` | A configuration parameter or mechanism |
| `build` | A build target, rule, or build-system construct |
| `test` | A test, test suite, or test framework |
| `document` | A document in `docs/` |

### Edge Issue Types (relationships)

Edge issues sit between two node issues and carry additional information about the relationship. Core types:

| Type | Represents |
|---|---|
| `scanned` | Links a `scan` node (child) to its `repo` node (parent) |
| `uses` | One component uses/depends on another |
| `implements` | A component implements an interface or abstract contract |
| `inherits` | Class or type inherits from another |
| `extends` | Extends behavior without full inheritance (mixins, decorators, etc.) |
| `mimics` | Replicates the behavior or API of another node without formal relation |
| `breaks` | A change or component breaks the contract of another |
| `obsoletes` | One node supersedes and deprecates another |
| `duplicates` | Two nodes redundantly implement the same thing |
| `contains` | Containment (package contains component, etc.) |
| `configures` | A parameter governs a component's behavior |
| `documents` | A document describes a node |
| `competes-with` | Two components address the same feature (bidirectional — use `related`) |

### Dependency Conventions

Beads dependency types are used as follows:

**`related`** — for **bidirectional** relationships passing through an edge issue:
```
node-A  --[related]-->  edge-issue  --[related]-->  node-B
```

**`parent-child`** — for **directional** relationships passing through an edge issue, where "child points to parent" (the dependent/derived node is the child):
```
node-A (child)  --[parent-child]-->  edge-issue  --[parent-child]-->  node-B (parent)
```
Use this when the relationship has a clear source and target (A derives from, uses, or is contained by B). Do **not** use `parent-child` directly between two node issues — all hierarchy and containment goes through an interstitial edge issue.

**`discovered-from`** — for provenance: the source node contributed to identifying the target node. Used directly between node issues, no edge issue required.

**Other pre-defined Beads dep types** may be used directly between node issues when no additional relationship metadata is needed and the semantics fit precisely.

### Title Writing for Keyword Recall

KB node and edge issue titles must be written to optimize `bd search` recall:
- Include the canonical name of the entity exactly as it appears in source code
- Include the repo name or package path as a prefix when disambiguation is needed
- Edge issue titles should name both endpoints: `"USES: ComponentA → ComponentB"`

---

## Scanning Workflow

### Adding a New Repo

When the user instructs Claude to add a repo:
1. Add a git submodule at `repo/NAME/` and ensure contents are present
2. Create a `repo` KB node issue with open state
3. Prompt the user for scan granularity and scope (see below) before scanning
4. Run the scan and create a `scan` KB node issue
5. Connect them with a `scanned` edge issue: `scan` (child) --[parent-child]--> `scanned` --[parent-child]--> `repo` (parent); the edge body holds scan date and process details
6. Populate KB with findings as additional node and edge issues

### Scan Granularity

**Always ask the user for granularity and scope before scanning** unless they specified it. Present options with approximate cost/time indication. Example:

- **Overview** (fast): README, top-level structure, build files, major entry points only
- **Package-level** (moderate): All packages/modules, their public APIs and inter-package dependencies
- **Component-level** (expensive): All classes/services/plugins, their relationships and configurations
- **Deep** (very expensive): Full function/method-level detail, test suites, all config parameters

Record what was scanned (scope, granularity, date) in the `scan` KB node and the `scanned` edge issue. Claude should also record scan coverage in Beads memories so future sessions know what has and has not been examined.

Claude is free to scan deeper at any time to answer a specific question, and should create a new `scan` node for each such targeted scan.

---

## Query and Answer Workflow

When the user asks a question about repos or documents:

1. **Retrieve**: Run `bd search <keywords>` to find relevant KB nodes and edges. Read `bd memories [search <keywords>]` for relevant behavioral context.
2. **Augment**: Read retrieved issues to build context. Follow dependency chains to find connected nodes.
3. **Answer**: Respond using the retrieved KB content plus any direct source inspection needed.
4. **Update**: At the end of the conversation (not inline), add or update KB issues and memories with anything new learned. Coherence takes priority over real-time updates.

---

## Beads Memories

Use `bd remember` / `bd memories` for **meta and behavioral** information about the ask-repo system itself — not for scanned content (that goes in KB issues). Examples of what belongs in memories:

- User preferences: verbosity level, preferred components when alternatives compete
- System behavior overrides set during this setup or by later user instruction
- Scan coverage summary (which repos, which granularities have been scanned)
- Notes about the KB itself (e.g., "node type X is overloaded, prefer Y instead")

---

## Non-Interactive Shell Commands

Always use non-interactive flags to avoid hanging on confirmation prompts (`cp`, `mv`, `rm` may be aliased with `-i` on this system):

```bash
cp -f source dest       # NOT: cp source dest
mv -f source dest       # NOT: mv source dest
rm -f file              # NOT: rm file
rm -rf directory        # NOT: rm -r directory
```

Other commands that may prompt: use `ssh/scp -o BatchMode=yes`, `apt-get -y`.
