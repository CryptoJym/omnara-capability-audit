# Independent audit: Omnara vs BORG, shared memory and Plimsoll

- **Bead:** [internal work reference]
- **Auditor:** the independent Opus 5.5 auditor. This was a delegated fleet worker on Studio1 (`claude-opus-5-5` per the runtime context; effort not exposed).
- **Written:** 2026-09-25, America/Denver.
- **Independence:** I did not see, search for or infer the other audits. I also did not use Mem0 recall, because it could surface them. The lead owns synthesis.

## Sources

| System | Pin | Basis |
|---|---|---|
| Omnara | `9f9ef08158c75d28d4d85dfb82f544e291a578c3` | Fresh public clone |
| BORG | `57ae994cfbe1cd4c1a88021b5df183cb71970c44` | Fresh public clone |
| Plimsoll | `480524f5…` | Working-tree snapshot, may include uncommitted changes |
| Plimsoll Cloud | `d2a2c3b0…` | Working-tree snapshot, may include uncommitted changes |
| memory-runtime | — | Selected currently installed files |

- **Omnara links:** `https://github.com/omnara-ai/omnara/blob/9f9ef08158c75d28d4d85dfb82f544e291a578c3/<path>#L<n>`. Both public HEADs still equalled these pins at audit time.
- **BORG links:** `https://github.com/h3ro-dev/borg/blob/57ae994cfbe1cd4c1a88021b5df183cb71970c44/<path>#L<n>`.

## Evidence rules

- **Documented:** READMEs, docs and papers.
- **Source:** code.
- **Coverage:** test files that exist. They were not executed.
- **Live:** used only where I observed behaviour in this session itself.
- Citations are `path:line` relative to `input/packet/`. `work/borg-public/…` and `work/omnara-public/…` are files from the same public commits that the packet filter dropped.

---

## 1. Bottom line

1. **These are different kinds of systems, and they mostly complement each other.**
   - **Omnara** owns the agent loop. It runs its own harness against model APIs and keeps every agent's state in one Postgres kernel.
   - **Our stack** orchestrates vendor CLI agents: Codex app-server lanes, Claude Code seats and the Grok CLI. Around them it adds:
     - shared memory (Mem0 + Graphiti);
     - native machine tools;
     - capacity- and allowance-aware admission;
     - agent coordination (Inbox + Beads);
     - spend and outcome telemetry (Plimsoll).
   - Omnara cannot drive subscription CLI seats (C25). Our stack has no managed-execution kernel comparable to Omnara's (C02).

2. **Omnara's real strength is execution reliability and product polish.** Evidence:
   - An immutable event ledger enforced by database triggers: `omnara/migrations/000004_events_messages_turns.sql:74-89` ([link](https://github.com/omnara-ai/omnara/blob/9f9ef08158c75d28d4d85dfb82f544e291a578c3/migrations/000004_events_messages_turns.sql#L74-L89)).
   - A lease-fenced runtime lock per agent (90 s), plus a reaper that marks in-flight model calls "outcome ambiguous" (`omnara/internal/storage/executionstore/agent_runtime_locks_store.go:19-21`).
   - Resumable SSE.
   - Contract-first RBAC: the server refuses to start if any API operation lacks a policy (`omnara/internal/httpapi/openapi_policies.go:420-457`).
   - 3,620 Go test functions in 576 files (counted, not run).

3. **Omnara's gaps are exactly where our stack is differentiated:**
   - No shared memory (C09–C11).
   - No link from runs to shipped outcomes (C20).
   - Admission is static slot counts, not measured capacity (C17).
   - No allowance or account routing (C18).
   - No desktop or browser control (C07).
   - No path to local or tailnet models in production. Its network filter blocks private ranges and `100.64.0.0/10`, the Tailscale range (`omnara/internal/ssrf/ssrf.go:39-48`, `:11-32`).

4. **Our stack's core ideas are ahead of Omnara's in several places.** Admission fails closed on unknown telemetry (`work/borg-public/conductor/router/router.mjs:54-92`). Memory recall is provenance-checked (`borg/memory/bin/mem0_recall_fast.py:554-589`). Outcome linkage keeps an `UNKNOWN` discipline (Plimsoll). **But the shipped implementations have reliability cliffs and security defaults I would fix before anything else.** Most material, all verified by me:
   - **The BORG router will eventually refuse all dispatch.**
     - No shipped code writes a terminal receipt state (`work/borg-public/conductor/router/router.mjs:145`, where only `PRE_START_FAILED` is ever written, at `:688`).
     - The ledger scan rejects above 10,000 entries or 4 MiB (`work/borg-public/conductor/router/receipt-reader-worker.mjs:5-7`).
     - A failed scan refuses dispatch (`router.mjs:636-643`).
     - `dispatch.lock` has no stale-PID recovery (`router.mjs:419-435`).
     - A dispatched workspace stays claimed until someone hand-edits JSON.
   - **The Codex conductor's HTTP API is unauthenticated.**
     - It is on loopback, exposes a raw JSON-RPC passthrough `/rpc` and starts threads with `danger-full-access` and `approvalPolicy: never` by default (`work/borg-public/conductor/conductor.mjs:87-98`, `:577-580`).
     - Any local process or local user can drive it.
   - **The memory boundary is thin.**
     - stdio gets full access by design (`memory-runtime/mem0-mcp-server-v2:380-387`).
     - Qdrant is bound to loopback with no API key (`borg/installer/services.py:44-50`).
     - MCP `memory_delete` is an immediate hard delete with no receipt (`memory-runtime/mem0-mcp-server-v2:2617-2633`).
   - **Plimsoll's privacy and trust shortcuts:**
     - "Hashed" identifiers are unsalted SHA-256 truncated to 16 hex characters, so a dictionary reverses them (`plimsoll/packages/shared/src/policy.ts:121-127`).
[Sharing note: private Cloud implementation details omitted here; the original is retained in the private audit archive. Scores are unchanged.]
[Sharing note: private Cloud implementation details omitted here; the original is retained in the private audit archive. Scores are unchanged.]
[Sharing note: private Cloud implementation details omitted here; the original is retained in the private audit archive. Scores are unchanged.]

5. **Omnara also has material weaknesses:**
   - Every built-in tool, including `run_command`, defaults to `always_allow` (`omnara/internal/toolcatalog/catalog.go:357`).
   - Slack approvals resolve with no Omnara role check (`omnara/internal/httpapi/integration_action_routes.go:26-70`, `:147-158`).
   - The compose file defaults to insecure dev mode. That mode includes a fixed, publicly known encryption key (`omnara/compose.yaml:6`; `omnara/internal/config/config.go:334-338`).
   - Agent content is stored immutably and can never be deleted. Org deletion is a soft delete, yet the docs say it "removes everything" (`omnara/internal/storage/queries/identity.sql:22-28` vs `omnara/docs/organization/members.mdx:50`).
   - SIGTERM cancels in-flight turns with no drain (`omnara/cmd/worker/main.go:72-75`).

6. **Recommendation: do not replace our fleet execution plane with Omnara.** Import its patterns instead:
   - a durable, lease-fenced run ledger with replayable event sequences;
   - a fail-closed operation-to-policy table;
   - idempotent spawn keys;
   - the queued/steering input model.

   Treat a self-hosted Omnara pilot as optional. It fits only API-billed, client-facing agents that need RBAC, Slack and disposable sandboxes, and only after hardening. Reciprocal contributions to Omnara are cheap: a private-egress allowlist, local MCP, defect reports and a memory-provider pattern. They carry less direct value for James.

---

## 2. What each system is (from source)

**Omnara** (Go + TypeScript, Apache-2.0)
- A managed-agent platform. It has:
  - an API, worker and maintenance processes over Postgres, Redis and blob storage;
  - a harness that calls OpenAI Responses, Chat Completions and Anthropic Messages, including the OpenRouter and Bedrock variants;
  - an outbound-only machine daemon (`omnarad`) for your own machines;
  - four sandbox providers: Blaxel, Daytona, Modal and Unikraft;
  - remote MCP, custom tools and skills;
  - a subagent tree;
  - org and project RBAC;
  - a web dashboard, TypeScript SDK and CLI, and Slack;
  - cron triggers and event webhooks.
- It is pre-1.0 (`omnara/SECURITY.md:5-7`).

**BORG** (Python + Node, MIT; public release)
- A local-first "collective" installer and platform:
  - an authenticated MCP connector with native files, processes, jobs, browser, UI, credential handles and SSH fleet routing;
  - a Codex app-server conductor with a router, plus Grok and launch-bus bridges;
  - the vendored Agent Inbox (45 files from an owner snapshot, `borg/coordination/PROVENANCE.md:3-13`) and pinned upstream Beads;
  - an installer with pinned runtimes;
  - the memory, graph and training layer.
- Only macOS arm64 is verified (`borg/README.md:98-99`).

**Shared memory** (installed runtime plus the BORG-published copy)
- Mem0 on local Qdrant, with scoped MCP servers and recall/capture hooks for Codex, Claude, Grok and fleet peers.
- Graphiti on FalkorDB, partitioned by scope.
- Nightly "dream" retention.
- Local extraction on Ollama, plus LoRA students that are all benched.

**Plimsoll** (TypeScript, Apache-2.0) and **Plimsoll Cloud** (private Next.js)
- A local telemetry collector for Claude Code, Codex, Gemini and Grok. It suppresses content before anything is stored, joins sessions to pull requests and computes Validated Delivery Yield (VDY).
- The Cloud is a multi-tenant work-intelligence app. By its own README it is "deployed for test and hardening; not approved for paid production" (`plimsoll-cloud/README.md:10-14`).

**Ecosystem, separate from BORG core**
- The estate's shared Inbox hub, canonical Beads, fleet conductors, the fleet-delegate seat gate, Estate Ops, account recovery and Jev.
- These are **documented** in `[private local path]`. They are not in the packet, and I did not verify them beyond what this session itself exercised:
  - the seat-gate admission probe;
  - the Inbox and Jev hooks that registered this session;
  - this delegated run itself.
- The lead checks BORG runtime and fleet health separately.

---

## 3. Capability matrix (my scores)

- **Scale:** 0–5 capability readiness for James's reliable multi-agent operations.
- **Crit:** criticality for that use case.
- **Tier:** S = source, S+L = source plus narrow live observation in this session, D = documented.
- There is no total: rows overlap (notably C01/C18/C25).
- Full findings and citations are in `RATINGS.json`; per-row evidence is in `evidence/`.

| ID | Capability | Crit | Ours | Tier | Omnara | Tier | Main difference |
|---|---|---|---|---|---|---|---|
| C01 | Provider/model interoperability | 4 | 3 (partial) | S+L | 4 (present) | S | Omnara: 3 API protocols with provider-switch replay. Ours: deep Codex, gated Grok, launch-only Claude in BORG core |
| C02 | Durable state and crash recovery | 5 | 2 (partial) | S | 4 (present) | S | Omnara's Postgres kernel with leases and ambiguous-outcome reaping vs our in-memory conductor and a self-locking router ledger |
| C03 | Steering, interruption, continuation | 4 | 3 (partial) | S | 4 (present) | S | Omnara: queued/steering/immediate inputs plus backlog control. Ours: native Codex steer with a turn guard, polled lossy events |
| C04 | Child-agent delegation and coordination | 5 | 3 (partial) | S+L | 3 (present) | S | Ours: richer authority model (Inbox leases, grants, CAS) that dispatch does not use. Omnara: integrated but tree-only |
| C05 | BYO machine and multi-machine tools | 5 | 3 (present) | S | 4 (present) | S | Omnara's outbound daemon with custody outbox vs our identity-pinned SSH tool routing |
| C06 | Disposable cloud sandboxes | 2 | 1 (partial) | S | 4 (present) | S | Omnara has 4 providers with an idempotent lifecycle; we have only a local Tart desktop lease, off by default |
| C07 | Native desktop/browser operations | 3 | 2 (partial) | S | 1 (partial) | S | Ours: thin CDP browser and AppleScript UI. Omnara: static fetch and search only |
| C08 | MCP and custom tools/skills | 4 | 3 (present) | S | 4 (present) | S | Omnara: OAuth MCP, durable custom tools, skills. Ours: rich native tool gateway but no owner-mounted extensions |
| C09 | Shared cross-session memory | 4 | 3 (present) | S | 0 (not found) | S | Ours only |
| C10 | Temporal graph and provenance recall | 3 | 3 (present) | S | 0 (not found) | S | Ours only; Omnara has a provenance substrate (immutable log) but no recall |
| C11 | Capture, consolidation, retention | 3 | 3 (partial) | S | 1 (partial) | S | Ours: grounded capture and reversible retention. Omnara: per-agent compaction only, and nothing can be deleted |
| C12 | Local inference and training/eval | 2 | 3 (present) | S | 1 (partial) | S | Ours: local extraction plus an honest canary harness. Omnara: its network filter blocks local endpoints in production |
| C13 | Operator dashboard and streaming | 4 | 2 (partial) | S (D eco) | 4 (present) | S | Omnara: resumable SSE, dashboard, SDK, Slack. Ours: Inbox console and analytics, no agent console |
| C14 | Identity, project boundaries, RBAC | 3 | 2 (partial) | S | 4 (present) | S | Omnara: fail-closed per-operation RBAC. Ours: owner_all connector, scope ACLs, prototype Cloud trust |
| C15 | Approvals, tool permissions, secrets | 4 | 2 (partial) | S | 3 (present) | S | Ours: strong secret redaction but no permission scoping. Omnara: good machinery, unsafe defaults |
| C16 | Hooks, schedules, notifications | 3 | 3 (present) | S+L | 3 (present) | S | Ours: hook-centric (verified live in this session). Omnara: cron, webhooks, Slack |
| C17 | Capacity-aware admission and ownership | 5 | 3 (present) | S+L | 2 (partial) | S | Ours: measured, fail-closed admission design. Omnara: static slots, but excellent lease ownership |
| C18 | Allowance and account recovery routing | 5 | 3 (partial) | S (D eco) | 1 (partial) | S | Ours: allowance-ranked Codex lanes. Omnara: same-provider retry only |
| C19 | Usage, token, cost observability | 4 | 3 (present) | S | 3 (present) | S | Plimsoll observes CLI seats; Omnara has exact provider cost for its own agents |
| C20 | Shipped outcome linkage and value | 4 | 2 (partial) | S+D | 0 (not found) | S | Ours only, and live coverage was last measured at 13 of 2,423 sessions |
| C21 | Privacy suppression and selective collection | 3 | 3 (partial) | S | 1 (partial) | S | Plimsoll suppresses before storage; memory and Omnara store content broadly |
| C22 | Idempotency, receipts, acceptance evidence | 5 | 3 (present) | S | 3 (partial) | S | Both strong on core paths with gaps; Omnara has no admin audit log, ours is uneven |
| C23 | Self-hosting, install, portability | 3 | 3 (partial) | S | 3 (present) | S | Both carefully pinned; ours macOS-only with no upgrade path; Omnara has unsafe compose defaults |
| C24 | Public API/SDK, docs, contribution | 2 | 2 (partial) | S | 4 (present) | S | Omnara: OpenAPI, SDK, CI. Ours: good docs, no SDK or spec, thin CI |
| C25 *(added)* | Subscription-seat CLI agents as engines | 5 | 3 (present) | S+L | 0 (not found) | S | Ours only; overlaps C01 and C18 |
| C26 *(added)* | Release integrity and cross-component contracts | 4 | 2 (partial) | S | 4 (present) | S | Our installed/published and repo-to-repo drift vs Omnara's single OpenAPI source |

### Axes (not totals; each is a judgment over the related rows)

| Axis | Ours | Omnara | Why |
|---|---|---|---|
| Managed execution (C01–C08, C25) | 2 | 4 | Omnara owns a durable kernel. Our execution depends on vendor CLIs plus a thin conductor with in-memory state and manual recovery |
| Collective memory (C09–C12) | 3 | 1 | Ours is real and provenance-aware but thinly bounded and single-node. Omnara has only per-agent compaction and human-uploaded skills |
| Fleet operations (C05, C06, C16–C18) | 3 | 3 | Ours has the better admission and allowance design, with operational cliffs. Omnara has production-grade machine and sandbox lifecycle, but static admission and no account routing |
| Delivery economics (C18–C20) | 3 | 2 | Plimsoll measures CLI-seat cost and links outcomes (low live coverage). Omnara has exact provider cost, no value linkage and no budgets |
| Product readiness (C13–C15, C21–C24, C26) | 2 | 4 | Omnara has CI, API, SDK, RBAC and signed releases (pre-1.0, unsafe compose defaults). Ours is macOS-only, has no SDK or spec, thin CI, drift and a prototype Cloud trust boundary |

---

## 4. Material weaknesses: our side, ranked by operational impact

Each item was confirmed by my own read of the cited lines unless marked "(reviewer)". Those were read by one of my evidence reviewers and not re-read by me.

1. **The router ledger locks itself up (BORG, C02/C17/C22).** Only `PRE_START_FAILED` is ever written as a terminal state (`router.mjs:145`, `:688`). Dispatched or uncertain receipts therefore stay active claims forever. Three consequences:
   - A workspace, its parents and its children cannot be dispatched again until someone edits the receipt. Tests confirm the conflict: `work/borg-public/conductor/tests/router-reliability.test.mjs:63` (reviewer).
   - The ledger scan is all-or-error, with limits of 10,000 entries and 4 MiB (`receipt-reader-worker.mjs:5-7`, `:43-49`). A failed scan refuses every dispatch (`router.mjs:638-643`).
   - A crash leaves `dispatch.lock` in place, and nothing checks whether the process that wrote it is alive (`router.mjs:419-435`).

   The documentation says reconciliation happens (`borg/conductor/docs/LAUNCH-RELIABILITY.md:39-41`, reviewer), but I found no code that does it.
2. **Unauthenticated local control plane with full-access defaults (BORG, C15).**
   - Conductor routes carry no credential or Host/Origin checks (grep of `conductor.mjs` for authorization, bearer or origin returned nothing).
   - `/rpc` forwards any JSON-RPC method (`:577-580`).
   - Thread start defaults to `danger-full-access` and `approvalPolicy: never` (`:91-92`).
   - Grok runs with `--always-approve` (`work/borg-public/conductor/providers/grok-conductor.mjs:375`).
   - The connector accepts only an `owner_all` principal (`borg/connector/borg_context_server.py:122-130`). Through the Cloudflare gateway that full tool set is reachable from web ChatGPT, and through `fleet_call` from every enrolled host (reviewer: `cloudflare_gateway.py:106-156`).

   James's doctrine wants uninhibited agents, and I respect that. This is not about approvals. It is about who can reach the control plane: every local process today, and a prompt-injected web session through the gateway.
3. **Execution state is in memory; recovery is manual (BORG, C02/C03).**
   - The thread map and a 5,000-event ring shared by all threads live in process memory (`conductor.mjs:392`; reviewer: `:442-455`).
   - The ring has no gap signal, so a slow reader silently loses events.
   - An account switch kills in-flight threads, and "mass-relaunch" is manual (`borg/docs/PAPER-3-conductors.md:67-71`, reviewer).
4. **The memory boundary exists only at the HTTP MCP door (Memory, C09/C14/C21).**
   - stdio has full access (`memory-runtime/mem0-mcp-server-v2:380-387`).
   - Qdrant runs on loopback without an API key (`borg/installer/services.py:44-50`), and hooks and tools talk to it directly.
   - `memory_search` tells agents to "Use this FIRST" for decisions and preferences, with no candidate framing (`memory-runtime/mem0-mcp-server-v2:1932-1937`). That conflicts with "memory is never the authority". The rule is enforced only by framing in the hooks. `memory-runtime/authority.py` is imported by nothing in the packet and "always declines" by design (`:24-27`).
5. **Memory privacy and erasure gaps (Memory, C11/C21).**
   - The older Claude/Grok SessionEnd capture hook stores extracted facts first, then deletes matches and logs the dropped text verbatim (`borg/memory/bin/mem0-capture-hook:286-295`).
   - The public identifier floor returns `[]` when its config is missing, contradicting its own comment (`borg/memory/bin/mem0_scope_lib.py:195-212`). The installed copy fails closed.
   - Deleted text persists in `retired-log.jsonl` (reviewer: `borg/memory/bin/mem0ctl:1560-1599`).
   - MCP delete is a hard delete with no receipt (`memory-runtime/mem0-mcp-server-v2:2617-2633`).
   - No at-rest encryption was found.
6. **Installed vs published drift (Memory/BORG, C23/C26).** Neither version is a superset of the other:
   - Only the installed runtime has fail-closed floors, never-widen `set_scope`, and no replay of `memory_add`. Compare `memory-runtime/mem0-mcp-server-v2:2180-2185` with the public copy, which replays adds on any exception at `borg/memory/bin/mem0-mcp-server-v2:1897-1900`.
   - The public copy has read-only graph queries that the installed runtime lacks.
   - The identity schema string differs, so the same event gets a different memory ID in each (`memory-runtime/mem0_capture_identity.py:19`, reviewer).
   - The installed runtime hard-codes estate paths and host names.
7. **Plimsoll trust boundary and privacy (C14/C21/C22).**
   - Protected-field hashes are unsalted and truncated (`plimsoll/packages/shared/src/policy.ts:121-127`). The project's own public issue records reversing one (reviewer: `plimsoll/issues/0061…:25`; I do not reproduce it).
[Sharing note: private Cloud implementation details omitted here; the original is retained in the private audit archive. Scores are unchanged.]
[Sharing note: private Cloud implementation details omitted here; the original is retained in the private audit archive. Scores are unchanged.]
[Sharing note: private Cloud implementation details omitted here; the original is retained in the private audit archive. Scores are unchanged.]
8. **Outcome linkage is unproven at fleet scale (Plimsoll, C20).** The project's own audit found 13 of 2,423 sessions linked and called fleet-wide cost per merged PR "not yet publishable" (`plimsoll/docs/agent-economics-join-audit-2026-08-20.md:31-34`). Linkage is GitHub-only and run by hand, one repo at a time.
9. **Plimsoll repo-to-repo contract drift (C19/C26).**
[Sharing note: private Cloud implementation details omitted here; the original is retained in the private audit archive. Scores are unchanged.]
[Sharing note: private Cloud implementation details omitted here; the original is retained in the private audit archive. Scores are unchanged.]
[Sharing note: private Cloud implementation details omitted here; the original is retained in the private audit archive. Scores are unchanged.]
10. **Test and CI coverage is thin where the risk is.**
    - BORG CI runs the release guard, 3 installer test modules (22 tests) and the conductor Node tests (`work/borg-public/.github/workflows/verify.yml:37-48`).
    - About 280 connector, coordination and installer Python tests, and every memory, graph and training test, never run in CI.
    - Both dream test files skip at import (reviewer).
11. **Uneven provider depth (BORG, C01/C25).** Claude dispatch in BORG core is a detached `claude -p`: it ignores `packet.model`, does not track exit, and reports completion only by the agent writing a file (`work/borg-public/conductor/providers/launch-bus.mjs:140-159`). The installer does not provision Grok, and the docs say to leave it disabled (reviewer: `borg/docs/SETUP.md:192-199`). The ecosystem's fleet-delegate path is what actually carries Claude work: this audit ran through it.
12. **Public-release hygiene.**
    - An owner-specific default memory scope literal remains in the public connector (`borg/connector/borg_context_server.py:117`).
    - Agent instructions name a `conductor-route` command that does not ship (reviewer).
    - The `activity` reader expects estate ledgers that nothing writes (reviewer).
    - The richer Peekaboo desktop backend is refused at config load (`borg_context_server.py:158-160`), yet 19 tests cover it (reviewer).

## 5. Material weaknesses: Omnara, ranked for James's use case

1. **It cannot use subscription CLI seats (C25/C18).**
   - The agent loop is Omnara's own harness on API keys.
   - I found no adapter that drives Codex app-server, Claude Code or the Grok CLI. The search for `chatgpt|codex|claude code|subscription` over `internal`, `cmd` and `docs` found only model-name pricing and MCP client docs.
   - Quota and billing errors are terminal (reviewer: `omnara/internal/model/providererrors/classify.go:123-126`).
   - Adopting Omnara as the execution plane would move the fleet from allowance-routed seats to per-token API billing.
2. **Private networks are unreachable by design (C01/C08/C12).**
   - The network filter blocks private addresses, `100.64.0.0/10` (Tailscale) and loopback outside dev mode for model endpoints, MCP servers and webhooks (`omnara/internal/ssrf/ssrf.go:11-48`; `omnara/cmd/worker/main.go:142-143`, `:186-195`).
   - Model base URLs must be HTTPS unless loopback (reviewer: `omnara/internal/storage/modelstore/validation.go:246-251`).
   - In a production self-host, tailnet Ollama, BORG memory MCP and LAN services need public HTTPS exposure.
3. **Permissive defaults and weak approval authority (C15).**
   - Every built-in tool, including `run_command`, `create_machine` and `spawn_agent`, defaults to `always_allow` (`omnara/internal/toolcatalog/catalog.go:357`).
   - Resolving an approval needs only `AgentOperate`, the same permission as sending input (`openapi_policies.go:384`).
   - Slack approvals check only the Slack signature, the app/workspace identity, an active install and that the target matches; they then resolve as the Slack user (`integration_action_routes.go:26-70`, `:147-158`). I found no approver allowlist.
   - `secret_env_overlay` values are injected into every process (`omnara/docs/agents/configuration.mdx:194`), so `run_command` can print them. The output then persists immutably.
4. **No data lifecycle (C11/C21).**
   - Triggers block UPDATE and DELETE on events, content blocks, model outputs and tool results (reviewer: `omnara/migrations/000006_processes_artifacts_usage.sql:195-209`; `000005…:510-564`).
   - Org and project deletion is a soft delete (`omnara/internal/storage/queries/identity.sql:22-28`), while the docs say "removes everything" (`omnara/docs/organization/members.mdx:50`).
   - End-user actors cannot be deleted (reviewer).
   - That is hard to square with client-data deletion duties.
5. **Unsafe self-hosting defaults (C23).**
   - The compose file defaults to `OMNARA_ALLOW_INSECURE_DEV_DEFAULTS=1`, open signup and fixed database credentials, and publishes Redis (no auth) and Postgres on all interfaces (`omnara/compose.yaml:1-8`, `:18-40`, `:102`).
   - Dev mode substitutes a fixed, publicly known encryption key (`omnara/internal/config/config.go:334-338`).
   - The documented recipe for exposing the stack publicly keeps these defaults (reviewer: `omnara/docs/self-hosting/deployment.mdx:58-62`).
   - There is no backup, restore or down-migration guidance.
6. **No deploy drain, and model calls are at-least-once (C02).** SIGTERM cancels the root context that every turn derives from (`omnara/cmd/worker/main.go:72-75`). Every rolling deploy therefore re-runs in-flight model calls as ambiguous retries, which means duplicate spend. Postgres is a single point of failure.
7. **Static admission (C17).** A fixed 4 turns per worker (reviewer: `omnara/internal/config/config.go:214`), declared pool quotas and no measured host load. Claims are global FIFO with no tenant fairness. The per-org kill switch has no writer in the repository.
8. **Machine trust (C05).**
   - Daemon tokens have no expiry column (`omnara/migrations/000007_org_byo_machines.sql:5-21`).
   - Self-update checks only a SHA-256 taken from the same origin (`omnara/internal/omnarad/update.go:595-607`). Release attestations are published but not verified by clients (reviewer).
   - BYO commands run with no OS isolation.
9. **Coverage and doc gaps.**
   - `Idempotency-Key` is accepted on 11 of 86 mutating public operations (my count over `omnara/api/openapi/openapi.yaml`).
   - There is no admin audit log.
   - Webhooks drop after a 10-minute window.
   - Cron has no docs page.
   - Grant revocation behaves differently from the docs (reviewer: `omnara/docs/organization/model-providers.mdx:107` vs `omnara/internal/modelprovider/resolver.go:112-128`).
   - No browser or desktop control; without an Exa key, `web_search` goes to Exa's keyless public endpoint (`omnara/internal/webaccess/exa.go:39-54`).

## 6. Worth keeping from each side

**Omnara, worth importing as patterns**
- **Durable run ledger.** DB-enforced invariants, lease-fenced runtime locks renewed at about lease/3 with self-cancel before expiry, a reaper, and explicit `outcome_ambiguous` / "external outcome unknown" states.
- **Idempotent side-effect contracts.** Record before calling out; deterministic sandbox allocation names; `spawn:<toolCallID>`; cancel keyed `agent-cancel:<id>:after:<seq>`; cron keyed trigger plus due time.
- **Queued/steering/immediate input model**, with backlog move, promote and demote, and idempotent cancel.
- **Fail-closed operation-to-policy table**, enforced at startup and covered by tests.
- **Durable SSE** with sequence IDs, `Last-Event-ID` replay and heartbeats.
- **Honest cost completeness:** "N of M calls reported no cost".
- **Repository lint invariants:** database-owned time and no raw sleeps (reviewer: `omnara/tools/omnaralint/omnaralint.go:122-138`).

**Ours, worth protecting, and the parts Omnara lacks**
- **Fail-closed admission** on unknown, stale or hot telemetry, with workspace identity by realpath plus inode.
- **Allowance-ranked account lanes** that never retry an uncertain start on another account.
- **Inbox authority re-validation at delivery time.** Instructions whose authority lapsed are rejected before lease (`borg/coordination/comms/hub/store.py:1443-1456`). Also CAS reassignment and attenuated delegable grants.
- **Provenance-aware graph recall.** A derived fact reaches a prompt only while its source rows are unchanged.
- **Evidence-grounded capture** for the newer memory path.
- **An honest local-model canary harness**, which refused to promote a 400/400 exam winner that failed in the real pipeline.
- **Plimsoll's pre-persistence suppression, per-item verified acknowledgements, conservation-checked allocation and `UNKNOWN` discipline.**
- **Candid non-claims:** `completionVerified: false` and `work_acceptance: not_asserted`.

## 7. The combined stack

**The best combined shape keeps vendor CLI seats as the main engines.** It hardens our control plane with Omnara's execution-ledger patterns and keeps Omnara, if used at all, as a separate API-billed tier.

Constraints on any Omnara integration:
1. **Memory access needs a public, scoped endpoint.** Omnara agents can reach BORG memory only through a public HTTPS remote MCP endpoint, because of the network filter. Today the only public BORG door is the Cloudflare gateway, which exposes the full `owner_all` tool set. A scoped memory-only profile would be needed first.
2. **Omnara agents cannot use the fleet's subscription allowances.** Plimsoll would need an Omnara source adapter (usage API and event webhooks) to put them in the same cost and outcome ledger.
3. **Deletion duties.** Omnara stores all content immutably, so client work that has deletion duties should not run there without an upstream retention feature.
4. **Two execution planes.** Allowance routing, seat gates and Inbox ownership do not see Omnara runs. Omnara does not see our capacity telemetry.

**A combined stack gains:** sandboxes (C06), RBAC and a Slack surface (C13/C14), and a mature API/SDK for client-facing agents (C24).

**It does not gain:** fleet reliability. That depends on fixing our own router, conductor and memory boundaries.

## 8. Recommendations (ranked; the same set is in `RATINGS.json`)

| # | Direction | Recommendation | Value | Effort |
|---|---|---|---|---|
| 1 | neither (internal) | Make the router self-healing: write terminal states from native readback, reconcile and archive receipts, recover a stale `dispatch.lock` by checking PID liveness, and add a ledger-size alarm | 5 | S |
| 2 | neither (internal) | Authenticate the conductor (per-lane token, Host/Origin checks). Gate or remove raw `/rpc`. Make full access an explicit request rather than the default | 5 | S |
| 3 | import | Adopt Omnara's durable run-ledger pattern for conductor threads: durable sequences, lease-fenced ownership, reaper, ambiguous outcomes, replayable events | 5 | M |
| 4 | import | Carry one work correlation ID across router receipts, Inbox assignments, Beads, memory capture identity and Plimsoll sessions | 4 | M |
| 5 | neither (internal) | Close the memory boundary: authenticate stdio or restrict it to owner processes, add Qdrant/FalkorDB auth, route MCP delete through tombstones with receipts, remove store-then-delete logging, frame `memory_search` output as candidates, run memory tests in CI | 4 | M |
| 6 | neither (internal) | Reconcile installed and published memory into one source. Make Plimsoll collector and Cloud share a package with contract fingerprints | 4 | M |
[Sharing note: private Cloud implementation details omitted here; the original is retained in the private audit archive. Scores are unchanged.]
| 8 | import | Adopt a fail-closed operation-to-policy table for BORG HTTP and MCP surfaces, and add scoped connector profiles (read-only, memory-only) for web clients | 4 | M |
| 9 | neither | Do not replace BORG conductors or fleet dispatch with Omnara | 4 | S |
| 10 | import | Adopt idempotent spawn keys, depth and instance caps, and the queued/steering/immediate input model in fleet-delegate, the launch bus and Inbox instructions | 3 | S |
| 11 | neither (internal) | Run the existing connector, coordination, memory, graph and training tests in CI | 3 | S |
| 12 | contribute | Upstream a configurable private-egress allowlist (tailnet and LAN CIDRs, default closed) plus local/stdio MCP support | 3 | M |
| 13 | import | Optional pilot: self-host Omnara only for API-billed, client-facing agents that need RBAC, Slack and sandboxes, after hardening: dev mode off, `run_command` set to always_ask, Slack approvers restricted, scoped memory MCP | 2 | L |
| 14 | contribute | Report Omnara defects. Use its private security channel for the Slack approval path and the insecure public-exposure recipe; file public issues for the doc mismatches, the keyless Exa default, the missing SIGTERM drain, non-expiring daemon tokens and unverified update attestation | 2 | S |
| 15 | contribute | Offer a provenance-aware memory provider (candidate framing, `derived_unverified`, source-currency checks) and Plimsoll-style outcome linkage as Omnara integrations | 2 | L |
| 16 | import | Add an Omnara source adapter to Plimsoll if any Omnara pilot runs | 2 | M |

No external message was sent in this audit. Recommendations 12, 14 and 15 would each need an orchestration-tier agent with full context before any upstream send.

## 9. Documentation vs code mismatches found

**Ours**
- BORG `README.md:52` says the conductor can "stream" Codex threads. In fact events are polled pagination over an in-memory ring.
- `LAUNCH-RELIABILITY.md` says receipts are reconciled; nothing writes the reconciled states.
- `PAPER-3-conductors.md` says Node 18+ and `/turn/interrupt {threadId}`. The code pins Node 24.21.0 and forwards `turnId` (reviewer).
- PAPER-2 describes consolidation as "merges ≥0.90, contradiction review 0.75–0.90". In code, apply is held in the public copy and the lane is off by default in the installed runtime (reviewer).
- PAPER-2 says a drop filter runs "before storage". The Claude/Grok hook stores first and deletes after.
- PAPER-2 says the dream is covered by tests. Its tests skip at import.
- The public `mem0_scope_lib.py` comment says the floor loader reports a missing config; the code returns `[]`.
- Plimsoll says its `delivery-ack` copies are "byte-identical"; they differ by 46 lines.
- Plimsoll's check count is given as 14, 61 and about 109 in different places, and its changelog stops at 0.7.5 (reviewer).

**Omnara**
- Org deletion is documented as "removes everything" but is a soft delete, and content is undeletable.
- Grant revocation is documented as not affecting running agents; the code enforces the grant on every call.
- Steering is documented as "joins the current turn"; the code opens a new turn.
- The docs' example of an MCP server on an internal host is blocked by the network filter.
- The README claims Ollama support; production blocks loopback and private endpoints.
- The docs say pools reconcile "toward targets"; there are no pool-size targets (reviewer).

## 10. How I verified

- **Inputs.**
  - All 3,015 entries in `INPUT-MANIFEST.sha256` matched.
  - `git ls-remote` showed both public HEADs still at the pinned commits.
  - I found that the packet filter dropped files. For BORG it dropped every `.mjs`/`.js` file, including the conductor, router and Inbox web UI, which was material. For Omnara it dropped 64 files, including `go.mod`, CI workflows and `hosted_credentials.go`.
  - I fetched the dropped files read-only from the same public commits:
    - **BORG:** 61 files, all matching the SHA-256 values in the in-packet `borg/RELEASE-INVENTORY.json`.
    - **Omnara:** 22 files. They are integrity-bound to the pinned commit by git object hashes; no separate inventory exists.
- **Evidence gathering.**
  - Five read-only Opus reviewers worked in parallel, one each for Omnara execution, Omnara governance, the BORG platform, memory and Plimsoll. Their full evidence files are in `out/evidence/`.
  - I then personally re-read about 35 of the highest-impact citations in both stacks. All matched; I made no corrections. They included:
    - Omnara: the ledger triggers, lease constants, `always_allow` default, Slack approval path, policy table, compose defaults, dev key, network filter and blocked ranges, Exa keyless path, SIGTERM handling, updater checksum, soft delete, token schema and secret overlay.
    - BORG: conductor defaults and routes, router terminal set, lock, failed-scan refusal, scan limits, `claude -p` launch, Grok `--always-approve`, CI scope, connector `owner_all`, SSH pinning, Peekaboo refusal and Inbox authority recheck.
    - Memory: stdio full access, delete path, Qdrant environment, capture-hook logging, public floor fail-open, `memory_add` replay drift, the `memory_search` docstring and that `authority.py` is not imported.
    - Plimsoll: unsalted hash, shared secret, unsigned-allowed path, 13 of 2,423, dry-run deletion, collector version pin and the 46-line ack diff.
  - I independently counted Omnara's `Idempotency-Key` coverage (149 operations; 86 mutating; 11 with the key) and its tests (576 files; 3,620 `func Test`).
  - `node --check` passed on 9 BORG conductor and platform `.mjs` files and 5 Inbox web JS files, using Node v24.21.0, the version BORG pins.
- **Live observations (narrow, this session only).** The fleet-delegate seat gate on this host admitted my subagents with measured load, memory, pressure, open files and disk (see `RECEIPT.json`). The Inbox and Jev hooks registered this session and reported checkpoint health.
- **What I did not do.** I ran no product tests, builds or services, and made no edits, deployments or external messages.

## 11. Limitations and unknowns

- Scores are expert judgment from source and docs, not benchmarks. No 5s were given. Test files are coverage, not proof.
- Not inspected:
  - Omnara Cloud's private services (the credential service, billing, and whatever writes the kill switch);
  - BORG adapter weights, `site/` JS and the `fleet_browser` worker;
  - Plimsoll dotfiles and CI, `dashboard.html`, and the Cloud Prisma schema;
  - the installed `mem0ctl`, `mem0_recall_fast.py`, Claude hook and graph binaries. The public copies stood in for them.
- Runtime state is **unknown** from source:
  - which memory flags are set (graph, JEV, capture student);
  - which capture hook each harness runs;
  - whether the offsite backup schedule is installed;
  - live Plimsoll linkage coverage today;
  - the Plimsoll Cloud signing configuration.
- Ecosystem capabilities (shared hub, fleet conductors, account recovery, Estate Ops) are documented only here. The lead checks runtime.
- Inferred, not reproduced: Omnara's head-of-line wakeup hazard, the macOS `os.freemem` effect on BORG admission, and the exact receipt count at which the router ledger fills. The entry and byte limits themselves are verified.
- Plimsoll and Plimsoll Cloud are working-tree snapshots, so no public commit links are given for them.
- Items marked "(reviewer)" rest on the evidence reviewers' reads, not on my own re-reads.

## Appendix: files in `out/`

- `AUDIT.md`: this report.
- `RATINGS.json`: the matrix with components, findings and sources.
- `RECEIPT.json`: runtime identity, sources, checks, children and admission.
- `evidence/evidence-omnara-exec.md`: Omnara C01–C08, C16, C22.
- `evidence/evidence-omnara-gov.md`: Omnara C09–C15, C17–C21, C23, C24.
- `evidence/evidence-borg.md`: BORG platform rows plus the adapters question.
- `evidence/evidence-memory.md`: memory rows plus installed vs public drift.
- `evidence/evidence-plimsoll.md`: Plimsoll and Plimsoll Cloud rows plus contract drift.

Every evidence file was written by one of my read-only reviewers. I spot-checked their citations as described in section 10.