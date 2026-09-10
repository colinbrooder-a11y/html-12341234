**New code to write (~half day):**
- `src/Llm/JudgmentCheckClient.cs` (new classlib or inside Rules): loads the prompt file, calls the deployment once per rule (or batched — prompt supports per-rule instructions), parses the JSON response.
- **The citation verifier — non-negotiable:** after parsing, assert `submission_language_matched` appears **byte-for-byte** in the serialized record and `rule_text_cited` byte-for-byte in the ruleset's `rule_text`. On mismatch: reject the flag, log, and either retry once or record `cannot_determine`. This is the "citations verified or the flag is thrown out" promise from the deck — it must be code, not convention.
**Packages / auth:**
- `dotnet add package Azure.AI.OpenAI` (+ `Azure.Identity` if Entra auth granted — preferred; `DefaultAzureCredential` picks up your login).
- Config (env vars or appsettings, never committed): `AOAI_ENDPOINT`, `AOAI_CHAT_DEPLOYMENT`, and `AOAI_API_KEY` only if key auth.
**Ask in the room when granted:** endpoint URL · chat deployment *name* (deployment name ≠ model name — the SDK wants the deployment name) · Entra or key · rate limits/quota on the dev deployment.
 
**Verify:** run PipelineRunner on `testdata/kitchen-sink-record.json` → expect the 3 AP flags **plus** GOV verdicts; on `clean-record.json` → expect `no_flags_assertion: true` (the 3-way finally resolving). Personas P11/P13 are the GOV-03 calibration pair — run both; P11 should flag, P13 shouldn't.
 
**Gotchas:** temperature 0 / deterministic settings for reproducibility (the envelope stamps model+prompt versions — same inputs should re-derive the same flags); the response must be *only* JSON (prompt already demands it — strip markdown fences defensively anyway).
 
---
 
## 2. Azure OpenAI — embeddings deployment → dedupe
 
**What it does when wired:** turns idea text into vectors so similar prior submissions surface (in-conversation cards + sealed `similar_ideas`).
 
**The existing seam:**
- `src/Data/Interfaces.cs` → `IVectorRepository`: `StoreEmbeddingAsync(ideaId, float[] embedding, string embeddingModel)` and `FindSimilarAsync(ideaId, embedding, threshold, limit)` returning `SimilarIdea` objects shaped for the envelope. **There is deliberately no FileSystem stub for this interface** — similarity can't be meaningfully faked; the comment in `FileSystemRepositories.cs` says so.
- `db/schema.sql` → `idea_vectors` table. **Currently stores embeddings as JSON arrays** (nvarchar + ISJSON check) because the Azure SQL native-vector question is open (see §3). Two implementation paths are pre-written in the schema comments.
- `system-envelope.v1.json` → `embedding_ref`, `similar_ideas[]`, `dedupe_threshold_used` — all nullable today, populated when this lands.
- PipelineRunner `Versions.EmbeddingModelVersion = "pending-sandbox"` → actual deployment string.
**New code (~half day + tuning):**
- `EmbeddingClient` (same package/auth as §1, second deployment name: `AOAI_EMBED_DEPLOYMENT`): embed a canonical text projection of the record — recommend `idea_title + idea_summary + operational_mapping` concatenated; document the projection, because changing it invalidates stored vectors.
- `SqlVectorRepository` implementing IVectorRepository (path A: native `vector` type + `VECTOR_DISTANCE` if tier supports; path B: JSON arrays + cosine in C# at dev scale — fine under ~10k ideas).
**THE tripwire — dimension:** `text-embedding-3-small` = **1536**, `-large` = **3072**. The schema comment shouts about this. Confirm the deployed model *before* storing anything; changing dimension after data exists = drop + re-embed everything.
 
**Ask in the room:** which embeddings model/deployment name · (→ dimension) · same endpoint as chat or separate resource?
 
**Verify:** embed all 20 personas; `FindSimilarAsync` on P-duplicate pairs should score high, unrelated pairs low. The demo pair (correspondence-triage vs. kitchen-sink) displaying 0.81 in the UI mock is the vibe target, not a spec. **Threshold is config, not code** — record whatever's used into `dedupe_threshold_used` per envelope; OQ/EA may tune it.
 
---
 
## 3. Azure SQL Database → the data layer
 
**What it does when wired:** retires the `devstore/` FileSystem stubs; records, envelopes, transcripts, outbox, processed_events land in real tables.
 
**The existing seam — this is the cleanest one:**
- `src/Data/Interfaces.cs` — five interfaces (`IIdeaRepository`, `ITranscriptRepository`, `IOutboxRepository`, `IProcessedEventsRepository`, `IVectorRepository`). Callers (PipelineRunner today, Orchestrator soon) depend only on these.
- `src/Data/FileSystemRepositories.cs` — current implementations; every method carries a comment stating what its SQL replacement adds (transactions, indexed queries, ON-CONFLICT semantics). They stay in the repo as the offline/dev fallback — swap is DI/config, not deletion.
- `db/schema.sql` — the full T-SQL DDL, run it once against the new instance (SSMS, Azure Data Studio, or `sqlcmd`). Design notes inline: JSON documents + promoted columns, **app keeps promoted columns in sync** (that's a documented repository responsibility — `UpdateEnvelopeAsync` must write `current_state`/`filtered_reason`/`updated_at` AND the envelope JSON in one statement/transaction).
**New code (~1 day):** `src/Data/SqlRepositories.cs` — `dotnet add src/Data package Microsoft.Data.SqlClient`. Implementation notes already encoded in schema comments:
- `SaveNewIdeaAsync` = one transaction across `ideas` + `envelopes`.
- `TryMarkProcessedAsync` = the `INSERT ... SELECT ... WHERE NOT EXISTS` pattern written verbatim in the schema comment; return `@@ROWCOUNT > 0`.
- `IOutboxRepository.EnqueueAsync` must share the caller's transaction with the status write (the whole point of the outbox) — a `PostgresUnitOfWork`-style coordinator was flagged in the interface comments; same concept, SQL flavor.
**Ask in the room:** connection method — SQL auth string vs. **Entra (Authentication=Active Directory Default)**, preferred · firewall: is my client IP allowlisted / VPN route / private endpoint? (First connection timeout = network config, not code) · confirmed tier → **does it support the native `vector` type?** (closes §2's storage path decision) · who owns backups/lifecycle on the dev instance.
 
**Verify:** run PipelineRunner end-to-end with SqlRepositories wired → rows in `ideas`/`envelopes`; re-run same fixture → confirm overwrite/versioning behavior does what you intend (decide: reject duplicate idea_id vs. upsert — currently FileSystem silently overwrites; SQL should probably reject on PK and force a new id).
 
---
 
## 4. Core Tools + Azurite → the orchestrator (local)
 
**What it does when landed:** lets the Durable Functions orchestrator be built and debugged on the laptop. **This project does not exist yet** — it's the one remaining build, not a wiring job, but it's transcription:
 
**The spec to transcribe:** `docs/status-machine.md` — §5 is literally the Durable mapping table (state → orchestrator step): `WaitForExternalEvent("SupervisorDecision")`, `WaitForExternalEvent("EAVerdict")`, `WaitForExternalEvent("HubDecision")`, timers for the 1-day EA / 4-day Hub SLAs (config values, per Hub meeting), `returned_to_submitter` loop, `sealing` as an activity chain.
 
**Scaffold when tools land:**
```
func init src/Orchestrator --worker-runtime dotnet-isolated
cd src/Orchestrator && func new --template "Durable Functions orchestration"
dotnet add reference ../Domain ../Rules ../Data
```
Azurite runs as the storage emulator (`azurite` in a second terminal, or VS Code extension); `local.settings.json` gets `"AzureWebJobsStorage": "UseDevelopmentStorage=true"`.
 
**Wiring already waiting for it:** activities call the *existing* pieces — schema validation (SchemaValidator logic → an activity), `RulesEngine.EvaluateDeterministic`, envelope construction (lift from PipelineRunner's sealing stage), persistence via `IIdeaRepository` (FileSystem stub works before SQL lands — that's why the stubs exist). The EA-verdict `RaiseEvent` is what the web app's verdict button eventually POSTs to.
 
**Gotchas pre-loaded from our discussions:** Durable replay semantics — orchestrator functions must be deterministic (no direct DateTime.Now/Guid.NewGuid/IO in the orchestrator body; all of that lives in activities). Expect to be confused by replay twice; that's normal. Fake external events for testing: `func` CLI or HTTP admin API can raise events without any UI.
 
**Ask IT:** just the ticket status — both are Microsoft-published installs (`winget install Microsoft.Azure.FunctionsCoreTools` / `npm i -g azurite`), same risk profile as the .NET SDK that already got approved.
 
---
 
## 5. Azure AI Foundry → the live intake agent (batch 2)
 
**What it does:** replaces the scripted demo conversation with the real agent running `prompts/intake-agent.v2.md` (the confirmed build target — rev D).
 
**The existing seam:**
- The prompt is complete: hard rules incl. the 4 governance questions, checkpoint pacing, §2.5 similar-idea surfacing, per-answer `check_rules` calls, §12.5 wrap-up pass, REVISION MODE, field-source map.
- The tools it expects: `check_rules` (wraps `RulesEngine.EvaluateDeterministic` + judgment-check on partial records — advisory mode) and `find_similar` (wraps `IVectorRepository.FindSimilarAsync`). These need to be **HTTP endpoints** the agent can call.
**THE open question (flagged in every ask):** can a Foundry agent call tool endpoints on a dev laptop (localhost)? Almost certainly **no** without help — expected answer is a dev tunnel (VS Code port forwarding / `devtunnel`) or deploying the two tool endpoints to the batch-2 Function App first. **Sequencing consequence:** if tunnels are disallowed by policy, Function App deployment moves *ahead* of Foundry testing. Ask this before building anything Foundry-side.
 
**Also confirm:** does StateFund's Foundry setup support OpenAPI-defined tools / function calling on the model tier granted · which model backs the agent (should match or exceed the judgment-check deployment) · transcript export (the `transcripts` table + Ali's feedback expect the conversation JSON).
 
---
 
## 6. Dev Function App + storage (batch 2)
 
Deployment target for §4's orchestrator + §5's tool endpoints. Bundle the storage account in the same ask (Durable state lives there — a Function App without storage is half a resource). Consumption plan. Nothing in the codebase changes for this — it's `func azure functionapp publish` plus app settings mirroring `local.settings.json` (connection strings/endpoints from §§1–3, as App Configuration or Key Vault refs per whatever EA's standard is — ask).
 
---
 
## 7. Cross-cutting config & tripwires checklist
 
- [ ] **Embedding dimension** confirmed before any vector is stored (§2)
- [ ] **Azure SQL native vector support** confirmed → decides storage path (§3)
- [ ] **Deployment names ≠ model names** — SDK calls want deployment names (§1)
- [ ] **business_unit string values**: AP-03 compares `record.BusinessUnit` against `["Claims","Pricing","Employment","Development"]` by string equality — confirm the *exact* strings the Entra/SSO profile supplies before real users touch it (flagged in RulesEngine comments)
- [ ] **Secrets hygiene**: endpoints/keys in env vars or local.settings.json (gitignored) — never in appsettings committed to the repo
- [ ] **Version stamps**: every integration updates its `Versions` field from "pending-sandbox" to the real string — the reproducibility promise depends on it
- [ ] **Synthetic data only** from the dev environment (the standing boundary) — personas, never real submissions
- [ ] **Citation verifier is code** (§1) — the byte-exact check is the deck's central trust claim
## 8. Suggested integration order (dependency-honest)
 
1. **Core Tools/Azurite → Orchestrator** (no Azure dependency; FileSystem stubs suffice) — the one remaining *build*
2. **Azure OpenAI chat → judgment-check** (most self-contained wiring; biggest demo upgrade: the "pending" line disappears)
3. **Azure SQL → SqlRepositories** (swap under the orchestrator via DI)
4. **Embeddings → vector repo + dedupe** (needs SQL answer from #3 for storage path)
5. **Function App + Foundry → live agent** (batch 2; sequence depends on the localhost answer)
Each step independently demoable; no step blocks on a later one.
