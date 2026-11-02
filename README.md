# Prompt for repo coding agent — session-day prep report

Copy everything below the line into the agent chat (Agent mode, repo root as context). It produces a report — it should NOT edit code.

---

You are working in the `intake-pipeline` repository (C#/.NET 8+, Azure-targeted AI intake/governance pipeline). Today I have a 2-hour provisioning session where the following Azure resources will be created and connected: **Azure OpenAI (one chat deployment + one embeddings deployment), Azure SQL Database, a Function App + storage account, Azure Functions Core Tools + Azurite installed locally, and Azure AI Foundry access.**

**Do not modify any files.** Produce a single markdown report called **SESSION-PREP.md** content (output it in chat) with the sections below. Every claim must cite an exact `path:line` you verified by reading the actual file — if something I believe exists doesn't, say so explicitly rather than inventing it.

## Section 1 — Pre-session fixes (blockers for persisting valid data)
Verify each suspected issue and give me: confirmed/not-confirmed, exact location(s), the minimal fix, and the test that proves it:
1. `stack_match` vs `assessment_status` mismatch between `schema/intake-record.v11.json` (tech_components definition) and the C# domain model — confirm whether a round-trip drops or renames a schema-required property.
2. Envelopes stamping an empty `ruleset_version` — find where the ruleset file's version lives vs. where the loader reads it, and where PipelineRunner writes the envelope's versions block.
3. Any place the literal filename typo `evelope` appears (code, scripts, fixtures).
4. Any *string values* (not comments) containing typos that would fossilize into data or events — especially event-type strings like `IdeaSumitted`.
5. Untracked/ignored-file hygiene: anything like `debug.log` that should be gitignored.

## Section 2 — Exact plug-in locations for each resource landing today
For each resource, give a table: **seam (path:line) · what gets added/changed there · config/env var name expected · the verification command/test to run the moment the resource exists.** Ground every row in the actual code:
- **Azure OpenAI chat** → the llm_judged seam: where pending llm rules are surfaced (rules engine), where the judgment-check prompt file lives, where PipelineRunner records `ChatModelVersion`/pending placeholders, where a new JudgmentCheckClient would be called from.
- **Azure OpenAI embeddings** → the vector seam: the IVectorRepository contract, the envelope fields it populates (embedding_ref, similar_ideas, dedupe_threshold_used), where EmbeddingModelVersion is stamped, and what in `db/` depends on the embedding dimension.
- **Azure SQL** → the persistence seam: every repository interface and its FileSystem stub counterpart, which schema file is the authoritative DDL to run (flag if more than one schema file exists and which targets which engine), and where a SqlRepositories implementation would register.
- **Core Tools/Azurite → orchestrator**: confirm whether any orchestrator project exists yet; identify the status-machine documentation section that defines the states/waits it must implement; list the existing classes/methods its activities would call (validation, rules evaluation, envelope construction, repositories).
- **Foundry** → the agent seam: the agent prompt file, and the existing code that `check_rules` / `find_similar` tool endpoints would wrap.

## Section 3 — Config inventory
List every env var / setting name the codebase (or its docs) already expects for these integrations, with where each is referenced. Flag any naming inconsistencies between docs and code (e.g., placeholder strings that differ).

## Section 4 — Smoke verification sequence
An ordered checklist of commands to run during the session as each resource lands (build/test commands that already exist in this repo, plus the gaps where no verification exists yet).

## Section 5 — Anything else inconsistent
Brief: anything you noticed while reading that would embarrass us mid-session (broken references, stale docs contradicting code, solution-file issues). Max 10 items, one line each, with locations.

Keep the whole report tight and scannable — tables over prose. Accuracy over completeness: a verified `path:line` is worth ten guesses.
