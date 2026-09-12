---
name: openspec-propose
description: >-
  Propose a change with all planning artifacts in one step. If an active
  OpenSpec change already covers the same work, revise that change instead of
  creating a new one. Use when the user wants a complete proposal (design,
  specs, tasks) ready for implementation.
allowed-tools: Bash(openspec:*)
license: MIT
compatibility: Requires openspec CLI.
metadata:
  author: openspec
  version: "1.1"
  generatedBy: "1.11.0"
---

Propose a change — create **or** extend planning artifacts in one step.

**Planning boundary**: This workflow creates/updates planning artifacts only. The user request that selected or triggered this workflow authorizes planning only, even if it asks to build or fix something. Do not edit project code. After the planning artifacts are complete, stop. Do not start implementation in the same response, even if the initial request asks for it. Wait for a new user request after the artifacts are presented; then start the apply workflow.

Default schema artifacts:
- proposal.md (what & why)
- `specs/<capability-path>/spec.md` (delta behavior)
- design.md (how)
- tasks.md (implementation steps)

`<capability-path>` is relative to `specs/` (e.g. `user-auth` or `identity/user-auth`). Preserve existing capability paths; follow project organization for new ones.

When the user is ready to implement, they must start the apply workflow explicitly.

---

**Store selection:** If the user names a store (a store is a standalone OpenSpec repo registered on this machine) or the work lives in one, run `openspec store list --json` to discover registered store ids, then pass `--store <id>` on the commands that read or write specs and changes (`new change`, `status`, `instructions`, `list`, `show`, `validate`, `archive`, `doctor`, `context`, `schemas`, `view`). Once selected, treat `--store <id>` as sticky for the rest of the workflow. Every unscoped example of those commands below is shorthand: before running it, append the flag. For example, run `openspec status --change "<name>" --json --store "<id>"`, not the unscoped form shown below. Other commands do not take the flag. Hints printed by commands already carry the flag; keep it on follow-ups. Without a store, commands act on the nearest local `openspec/` root.

**Input**: The user's request should include a change name (kebab-case) OR a description of what they want to build.

**Steps**

1. **Understand the request and clarify material ambiguity**

   If no clear input is provided, ask the user (open-ended, no preset options):
   > "What change do you want to work on? Describe what you want to build or fix."

   From their description, derive a kebab-case name (e.g., "add user authentication" → `add-user-auth`) — used only if a **new** change will be created.

   **IMPORTANT**: Do NOT proceed without understanding what the user wants to build.

   If the request contains ambiguity that would materially affect scope, externally observable behavior, compatibility, or acceptance criteria, ask the user before creating or revising the change. For minor details, make a reasonable assumption and record it in the planning artifacts.

2. **Active changes first (do not spawn a duplicate)**

   Before `openspec new change`, always:

   ```bash
   openspec list --json
   ```

   **Active** = any change listed by `openspec list` (not yet archived), including status `complete` / `in-progress` with unfinished or finished tasks.

   Then decide:

   | Situation | Action |
   |-----------|--------|
   | No active changes | Create a new change (step 4+) |
   | User explicitly asks for a **new / separate** change (or unrelated capability/intent) | Create a new change |
   | Exactly one active change, and the request continues / fixes / extends the same topic or capability | **Update that change** — do **not** create a new directory |
   | Several active; request clearly matches one (same capability, name, or conversation context) | **Update that change** |
   | Several active and match is ambiguous | Ask which change to update; do not create a new one until resolved |

   When updating an existing change:

   1. Announce: `Using existing change: <name>` (not creating a new one). Override: user can insist on a new change name.
   2. Follow the revision rules of [`openspec-update-change`](../openspec-update-change/SKILL.md): read all existing artifacts; fold new Why/Scope/Out of scope, requirements/scenarios, design decisions, and **new unchecked** `- [ ]` tasks; keep already `[x]` tasks checked unless the revision invalidates them (then uncheck and note why).
   3. Keep artifacts coherent in all directions (proposal ↔ specs ↔ design ↔ tasks).
   4. Do **not** invent a second change folder and “merge later”. If a mistaken new change was already scaffolded in this session for the same work, move its content into the target active change, then delete the mistaken change directory (and its `.openspec.yaml`).
   5. Skip steps 4–5 (scaffold + create-from-empty). Go to step 6 output for an **update** summary.
   6. After writes: `openspec validate <name>` and `openspec status --change "<name>"`.

3. **Determine the workflow schema** (new change only)

   Use the configured default schema unless the user explicitly requests a different workflow.

   **Use a different schema only if the user:**
   - Explicitly requests a specific schema by name → use `--schema <schema-name>`
   - Asks to "show workflows" or asks "what workflows" exist → resolve the authoritative root by running `openspec context --json` from the current working directory. If the user explicitly selected a registered store, use `openspec context --json --store "<store-id>"`. Then run `openspec schemas --json` with its working directory set to the returned `root.path` and let them choose. This preserves roots selected by a local `store:` pointer or the global `defaultStore`; when a registered store was explicitly selected, append `--store "<store-id>"` to `openspec schemas --json` as well. If context reports only `no_openspec_root`, run `openspec schemas --json` from the current working directory instead. Do not use this fallback for invalid or unavailable stores.

   Otherwise, omit `--schema` to preserve the configured default.

4. **Create the change directory** (new change only)

   Choose one schema form below. If a registered store is selected, append `--store "<store-id>"` to that command and each later OpenSpec command shown below that accepts `--store`.

   Using the configured default:
   ```bash
   openspec new change "<name>"
   ```

   Using an explicitly requested schema:
   ```bash
   openspec new change "<name>" --schema "<schema-name>"
   ```
   This creates a scaffolded change in the planning home resolved by the CLI with `.openspec.yaml`.

5. **Get the artifact build order and create every artifact** (new change only)

   ```bash
   openspec status --change "<name>" --json
   ```
   Parse the JSON to get:
   - `applyRequires`: array of artifact IDs needed before implementation (e.g., `["tasks"]`)
   - `artifacts`: list of all artifacts, each with its `status` and its `requires` edges (the artifact IDs it directly depends on)
   - `planningHome`, `changeRoot`, `artifactPaths`, and `actionContext`: path and scope context. Use these instead of assuming repo-local paths.

   Use a todo list to track progress through the artifacts.

   Loop through artifacts in dependency order (artifacts with no pending dependencies first):

   a. **For each artifact that is `ready` (dependencies satisfied)**:
      - Get instructions:
        ```bash
        openspec instructions <artifact-id> --change "<name>" --json
        ```
      - The instructions JSON includes:
        - `context`: Project background (constraints for you - do NOT include in output)
        - `rules`: Artifact-specific rules (constraints for you - do NOT include in output)
        - `template`: The structure to use for your output file
        - `instruction`: Schema-specific guidance for this artifact type
        - `skipped`/`warning`: present when the change declares skip_specs and this artifact must NOT be created - stop and pick another artifact
        - `resolvedOutputPath`: Resolved path or pattern to write the artifact
        - `dependencies`: Completed artifacts to read for context
      - Read any completed dependency files for context - always re-read them from disk, even if you saw them earlier in the conversation (the user may have edited them)
      - If the `instruction` field delegates creation to a specific skill or command, invoke it to produce the artifact instead of writing the file yourself, then verify the artifact file exists at `resolvedOutputPath`
      - Otherwise create the artifact file using `template` as the structure and write it to `resolvedOutputPath`. If `resolvedOutputPath` is a glob, follow `instruction` to choose the concrete file path
      - Apply `context` and `rules` as constraints - but do NOT copy them into the file
      - Show brief progress: "Created <artifact-id>"

   b. **Continue until every artifact in the required set exists (not just `apply.requires`)**
      - After creating each artifact, re-run `openspec status --change "<name>" --json`
      - The required set is `applyRequires` plus every artifact reachable from those by following the `requires` edges in `status --json` - walk them transitively (spec-driven closes over proposal, specs, design, tasks). Leave artifacts outside that set alone
      - `status` is file-existence only, so an `applyRequires` artifact reading `done` does NOT mean its dependencies exist - writing `tasks.md` early marks `tasks` done while `specs` was never written. Use each artifact's `requires` edges, not its `status`, to build the required set: a `done` artifact still lists what it depends on
      - An artifact already reading `status: "skipped"` is satisfied: the change declares `skip_specs` in `.openspec.yaml`, so its files must NOT exist. Never try to create one
      - Create every artifact in the required set that is missing, then re-check - creating one can unblock others
      - Skip one only when `status` already reports it `skipped`, or when its own `instruction` says it is conditional: run `openspec instructions <artifact-id> --change "<name>" --json` and skip only if its `instruction` field marks it optional (e.g. "create only if..."). Spec-driven's `design.md` qualifies; `specs` qualifies only via the `skipped` status above, never by your own judgment. Tell the user, and do not reconsider it
      - Dependencies are enablers, not gates: if a required artifact is still `blocked` only because you skipped a conditional dependency, write it anyway
      - Stop when every artifact in the required set is `done`, `skipped`, or was deliberately skipped

   c. **If an artifact requires user input** (unclear context):
      - Ask the user to clarify
      - Then continue with creation

6. **Show final status**
   ```bash
   openspec status --change "<name>"
   ```

**Output**

After completing all artifacts (create **or** update), summarize:
- Change name and location; whether this was **new** or **updated existing**
- List of artifacts created or revised with brief descriptions, plus any conditional artifact you skipped and why
- What's ready: "All artifacts needed for implementation are ready."
- Prompt: "The artifacts are ready for review. When you are ready, run `/openspec-apply-change` or ask me to apply this change." (If tasks were already partially applied and new unchecked tasks were added, say so — apply only the remaining work.)

**Artifact Creation Guidelines**

- Follow the `instruction` field from `openspec instructions` for each artifact type - it is the authoritative guidance, even for familiar artifact names
- If the `instruction` field directs you to use a specific skill or command to create the artifact, invoke it instead of writing the artifact directly
- The schema defines what each artifact should contain - follow it
- Read dependency artifacts for context before creating new ones
- Use `template` as the structure for your output file - fill in its sections
- **IMPORTANT**: `context` and `rules` are constraints for YOU, not content for the file
  - Do NOT copy `<context>`, `<rules>`, `<project_context>` blocks into the artifact
  - These guide what you write, but should never appear in the output

**Guardrails**
- The request that invoked this workflow authorizes planning only. Any implementation or apply instruction in that request does not carry forward. Do NOT implement the change, start the apply workflow, or edit project code during this workflow. After presenting the artifacts, stop and wait for a new user request to start the apply workflow
- **Prefer updating an active change** over `openspec new change` when the request continues the same work; never leave a duplicate change folder for the same intent
- Create every artifact the apply phase transitively depends on, not just the ids listed in `apply.requires` (new-change path only)
- Always read dependency artifacts before creating a new one - re-read from disk, not from conversation memory (files may have changed since you last saw them)
- Ask about ambiguities that would materially change scope, externally observable behavior, compatibility, or acceptance criteria; for minor details, make reasonable assumptions and record them
- If a change with that **exact name** already exists, ask if user wants to continue it or create a new one under a different name
- Verify each artifact file exists after writing before proceeding to next
