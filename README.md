# AI Idea Intake Pipeline

From static form to conversation: an AI-assisted intake and triage pipeline for AI use-case ideas at State Fund. A conversational agent completes a rigid, versioned intake record; governance rules flag issues **during** the conversation with verbatim citations; ideas are deduplicated against history; and every decision is made by a human — supervisor, then Enterprise Architecture, then the AI Hub.

**The rule that governs everything here: the AI retrieves, flags, and cites — humans decide.**

---

## Status at a glance

| Piece | State |
|---|---|
| Intake record + envelope schemas (JSON Schema, versioned) | ✅ built, validated |
| Governance ruleset + deterministic rules engine (AP-01/02/03, cited flags) | ✅ built, tested |
| End-to-end pipeline runner (validate → flag → seal, self-validating envelopes) | ✅ built — `.\demo.ps1` |
| Test suite (round-trip, persona-based rules tests, null tri-state) | ✅ green |
| Interactive UI demo (5 roles, scripted intake chat, flag cards) | ✅ built — open `ui-demo/intake-demo-full.html` |
| Data layer design (Azure SQL DDL + repository interfaces + dev stubs) | ✅ designed; SQL impl pending instance |
| Intake agent + judgment-check prompts (rev D confirmed) | ✅ written, versioned |
| Durable Functions orchestrator | ⏳ next build — gated on Core Tools/Azurite install |
| LLM checks (GOV-02/03/04), embeddings dedupe | ⏳ seams built; gated on Azure OpenAI |
| Real persistence | ⏳ gated on Azure SQL instance |
| Live agent (Foundry), OneTrust write-back, Oro export | 🔭 integration phase |

**Where every pending piece plugs in:** see [`docs/integration-map.md`](docs/integration-map.md) — per-resource seams, files, config, and verification steps.

---

## Quick start

```powershell
# validate any record against the intake schema
dotnet run --project src\SchemaValidator -- schema\intake-record.v11.json testdata\valid-record.json

# run the test suite
dotnet test src\Tests

# the three-beat demo: flags+citations / clean+unsure / rejection
.\demo.ps1

# the UI walkthrough (no install — opens in any browser)
ui-demo\intake-demo-full.html
```

The demo's kitchen-sink record deliberately trips all three deterministic governance rules — watch for verbatim rule text and the submitter's own words in each flag, and the honest `llm_judged pending` line marking the Azure OpenAI seam.

---

## Repository layout

```
schema/     intake-record.v11.json, system-envelope.v1.json — the data contracts (JSON Schema 2020-12)
rules/      ruleset.v1.json — governance rules: 3 active deterministic (AI-AP-*), 3 active llm_judged (AI-GOV-*)
prompts/    intake-agent.v2.md (confirmed build target), judgment-check.v1.md — versioned LLM prompts
docs/       status-machine.md (rev D — THE workflow spec), ui-design.v1.md, hub-sliver.v1.md,
            integration-map.md (per-resource wiring guide)
db/         schema.sql — Azure SQL DDL: JSON docs + promoted columns, outbox, per-consumer idempotency
testdata/   fixtures (valid / clean / unsure / kitchen-sink / broken) + personas.v1.md (20+2 test personas)
ui-demo/    intake-demo-full.html — self-contained 5-role interactive demo
src/
  SchemaValidator/   CLI: validate any JSON against any schema, exact error pointers
  Domain/            POCOs mirroring the schemas (IntakeRecord, SystemEnvelope)
  Rules/             ruleset loader + deterministic rules engine (null-safe tri-state)
  Tests/             xUnit: round-trips, persona-shaped rules tests
  PipelineRunner/    end-to-end: validate → flag → sealed, version-stamped envelope (self-validated)
  Data/              repository interfaces + filesystem dev stubs (devstore/); SQL impl pending
demo.ps1    the three-beat runnable demo
```

## Design decisions worth knowing (details in each artifact)

- **Documents + promoted columns** (db/schema.sql): records/envelopes stored as validated JSON; only hot-path fields are real columns — schema revs never force migrations.
- **Deterministic first, LLM second:** highest-stakes rules are plain C# (zero variance, asked explicitly); principle-level rules are LLM-judged with **byte-exact citation verification** — a flag whose quote doesn't match is rejected.
- **"Not sure" is a tri-state null**, never a violation — it routes to an open question (tested: the P09 persona).
- **Flags inform, never block.** No idea is stopped by automation; submitter responses to flags travel to every reviewer.
- **Outbox + per-consumer idempotency** (`(event_id, consumer)` PK): status writes and their events commit together; duplicate event deliveries no-op.
- **Human-confirmed dedupe only** (Hub decision): the system surfaces similar ideas; only people close ideas as duplicates.
- **Every envelope stamps versions** (schema, prompt, model, ruleset) — any flag can be re-derived.
- **Known gap, disclosed:** AI-AP-03 triggers on the submitter's business_unit, not the decision domain (fix specced; persona P12 flips when fixed).

## Workflow (rev D — confirmed)

`intake_in_progress → submitted → sealing → supervisor_review_pending → ea_review_pending → hub_review_pending → hub_approved | hub_denied`, with `returned_to_submitter` loops on every decline (revisions are new, linked submissions via `revised_from`), `withdrawn` from any non-terminal state, and `filtered_pre_hub(duplicate | ea_declined | supervisor_declined | pre_approved)` early exits. Full spec, SLAs (EA 1d + Hub 4d working values), and the Durable Functions mapping: [`docs/status-machine.md`](docs/status-machine.md).

## Milestone tags

- `spec-complete-v1` — schemas, ruleset, prompts, status machine as reviewed artifacts
- `prototype-tier1-v1` — working end-to-end governance pipeline (the demo)

## Next

1. **Orchestrator** (on Core Tools landing): transcribe status-machine §5 into Durable Functions — activities call the existing engine/repositories
2. **Judgment-check wiring** (on Azure OpenAI): the most self-contained integration; kills the "pending" line
3. **SQL repositories** (on the Azure SQL instance): swap for the filesystem stubs via the existing interfaces
4. **Embeddings dedupe** → **Foundry agent** → integration phase

Per-step details, asks, and tripwires: [`docs/integration-map.md`](docs/integration-map.md).
