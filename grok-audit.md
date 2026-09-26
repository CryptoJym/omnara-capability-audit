# Independent audit: Omnara and our stack

Auditor: Grok 4.7 Extra High (`grok-4.7-xhigh` from `CURSOR_FLEET_MODEL`). Host: studio1. Bead: `[internal work reference]`. This judgment is independent of the other auditors. Their reports were not opened.

Pinned public revisions:

- Omnara `9f9ef08158c75d28d4d85dfb82f544e291a578c3` — https://github.com/omnara-ai/omnara/commit/9f9ef08158c75d28d4d85dfb82f544e291a578c3
- BORG `57ae994cfbe1cd4c1a88021b5df183cb71970c44` — https://github.com/h3ro-dev/borg/commit/57ae994cfbe1cd4c1a88021b5df183cb71970c44

Plimsoll `480524f54286b106d978abf969455123aac1520f` and plimsoll-cloud `d2a2c3b04a66b5bff60aa31bea136b6af46e3c3a` are working-tree text snapshots, not fresh public clones. `memory-runtime` is the selected installed source and has no commit pin in `SOURCE-STATE.json`. Scores of 5 are withheld because nothing here was executed live. Structured scores are in `out/RATINGS.json`.

## What each side is

Omnara is a multi-tenant agent control plane. Agents, inputs, and tool calls live in Postgres. Machines are attachable computers: a bring-your-own daemon, or a pool sandbox on Blaxel, Daytona, Modal, or Unikraft. Humans use a dashboard, Slack, cron, and webhooks. The product surface is an OpenAPI, a CLI, and a TypeScript SDK.

Our side is four separable layers, and this audit keeps them separate:

- **BORG** is the installable collective: local memory, a temporal graph, benched extraction adapters, a Codex conductor, a native connector, and an Inbox.
- **Memory** is the installed mem0/Graphiti runtime, which is not identical to the public `borg/memory` tree.
- **Plimsoll** is local usage and outcome measurement. Plimsoll Cloud is the private hosted half and is not approved to charge.
- **Ecosystem** is Inbox, Beads, and fleet conductors. They sit beside BORG. Enrolling a machine does not copy memory or create a conductor lane.

## Material weaknesses

### Ours

The Codex conductor defaults new threads to `sandbox=danger-full-access` and `approvalPolicy=never`, and if the server asks for approval anyway the conductor denies it (`public-supplement/public-supplement/borg/conductor/conductor.mjs:22-25`). Unattended work can do the most dangerous thing without a person, and a person cannot approve the exceptional case through this control plane.

Provider support is uneven. Only Codex has native thread status, mid-turn steer, and allowance routing. Grok can interrupt and resume the same session and is excluded from allowance routing. Claude is a headless launch without mid-turn steer (`launch-bus.mjs:32-49`).

Admission code fails closed when capacity or claims are unknown (`router.mjs:54-91`), and the installer does not deploy the collectors that would make those observations fresh (`docs/COMPONENTS.md:113-125`). A router without a collector is a closed gate, not a fleet.

The shipped 4B graph adapter is not the model that failed the canary. Its predecessor failed at 1 of 25 episodes, and the clean retrain has not been canaried (`adapters/README.md:52`). Weight files were not in the packet or the supplement, so the cards cannot be checked against bytes. The graph feed itself is default-off (`graph/backfill.py:12-14`). Installed graph evidence is marked derived and unverified.

Install acceptance is Apple Silicon macOS. Linux and Intel artifacts are pinned with native acceptance still pending. Windows is outside the installer (`borg/README.md:97-99`).

[Sharing note: private Cloud implementation details omitted here; the original is retained in the private audit archive. Scores are unchanged.]

### Omnara

There is no shared semantic memory, temporal graph, or forgetting pipeline. The concepts page's core entities are organizations, projects, machines, models, skills, secrets, and agents (`docs/concepts.mdx:9-33`). A transcript is durable. It is not a collective brain.

There is no subscription-allowance failover. "Allowance" in the model store is a max-output token window (`modelstore/validation.go:821`). There is no join from an agent session to a merged pull request and its checks. Usage sums `provider_reported_cost_usd` when the provider sent it (`model_usage.sql:25-32`), so missing provider cost is a hole, not a measured zero.

Host admission is machine online/asleep/offline plus project quotas. It does not see load per core or memory saturation.

`web_fetch` cannot reach localhost or private addresses (`webaccess.go:74`, `fetch.go:39`). Desktop and browser control exist only if the attached machine's shell can do them. The reserved tool list has no owned browser session.

Compose enables insecure development defaults, including empty secret-encryption keys until the operator sets them (`compose.yaml:6-8`). That is called out in the README and is still the path a first `docker compose up` takes.

Steering arrives at the next model call. It is not a mid-token interrupt of the provider stream. That is a real design, and it is a weaker interrupt than the Codex `turn/steer` and `turn/interrupt` pair.

## Worthwhile capabilities

Omnara's agent kernel is the piece we do not have in one place: inputs are stored before they are events, delivery is `queued`, `steering`, or `immediate` (`agent_inputs_ingress_store.go:65-67`), runtime-lock recovery locks the agent row first (`agent_runtime_locks.sql.go:433-435`), and subagents have a clean context plus read, message, and stop tools (`subagent_tools.go:4-26`). The sandbox contract is equally concrete: one live provider resource per machine, retry-safe wake and delete, and a replaced resource must drop stale ids (`machinepool/providers/provider.go:59-85`). Org and project roles are an actual permission lattice (`authz.go:10-64`).

Our memory and measurement layers are the piece Omnara does not have. Fast recall is a scoped Qdrant search (`mem0_recall_fast.py:1-39`). Retention can withhold ids across a crashed journal (`retention.py:1-23`). Plimsoll hashes remotes and branches and joins sessions to pull requests by branch hash, head sha, or merge sha (`linkage.ts:1-18`, `outcomes-sync.ts:121-146`), after stripping raw prompts in metadata mode (`policy.ts:207-218`). The fleet router ranks eligible Codex lanes by remaining allowance, then reset time (`router.mjs:94-101`). Native browser tools talk to a dedicated Chrome over CDP (`native_browser.py:1-32`).

## Capability matrix

Criticality is for reliable multi-agent operations, not for a general market score. Evidence tier is `source` throughout. Tests were read as coverage and were not run.

| ID | Name | Crit. | Ours | Omnara |
| --- | --- | --- | --- | --- |
| C01 | Provider and model interoperability | 5 | partial 3, BORG and Plimsoll | present 4 |
| C02 | Durable agent state and crash recovery | 5 | partial 3, BORG and Memory | present 4 |
| C03 | Runtime steering, interruption, continuation | 5 | partial 3, BORG | present 4 |
| C04 | Child-agent delegation and coordination | 5 | partial 3, BORG and ecosystem | present 4 |
| C05 | Bring-your-own machine and multi-machine tools | 4 | present 4, BORG | present 4 |
| C06 | Disposable cloud sandbox lifecycle | 3 | partial 1, BORG | present 4 |
| C07 | Native desktop and browser operations | 4 | partial 3, BORG | partial 2 |
| C08 | MCP, custom tools, and skills | 4 | partial 3, BORG and Memory | present 4 |
| C09 | Shared cross-session semantic memory | 5 | present 4, BORG and Memory | not_found 0 |
| C10 | Temporal graph and provenance-aware recall | 4 | partial 3, BORG and Memory | not_found 0 |
| C11 | Capture, consolidation, forgetting | 4 | partial 3, BORG and Memory | partial 1 |
| C12 | Local inference and extraction training | 2 | partial 2, BORG | not_found 0 |
| C13 | Operator dashboard and streaming | 4 | partial 3, BORG, Plimsoll, ecosystem | present 4 |
| C14 | Team identity, project boundaries, RBAC | 4 | partial 2, BORG and ecosystem | present 4 |
| C15 | Human approvals, tool permissions, secrets | 5 | partial 2, BORG | present 4 |
| C16 | Hooks, schedules, notifications | 3 | partial 3, BORG and Plimsoll | present 4 |
| C17 | Capacity-aware admission and work ownership | 5 | partial 3, BORG and ecosystem | partial 2 |
| C18 | Subscription allowance and account recovery | 5 | partial 3, BORG | not_found 0 |
| C19 | Usage, token, and cost observability | 4 | present 4, Plimsoll | partial 3 |
| C20 | Shipped outcome linkage and value | 5 | present 4, Plimsoll | not_found 0 |
| C21 | Privacy suppression and selective collection | 5 | present 4, Plimsoll, BORG, Memory | partial 3 |
| C22 | Idempotency, receipts, acceptance evidence | 4 | present 4, BORG, Plimsoll, ecosystem | present 3 |
| C23 | Self-hosting, installation, portability | 4 | partial 3, BORG | present 4 |
| C24 | Public API, SDK, docs, contribution | 3 | partial 3, BORG and Plimsoll | present 4 |

Row narratives and file:line sources are the `rows` array in `RATINGS.json`. The scores above match that file.

### Notes that change the reading of a row

C09 and C10 on our side are implemented and still bounded. The fast recall path returns raw cosine similarity and can rank differently from mem0's fused score (`mem0_recall_fast.py:36-39`). The installed `authority.py` gate declines until an out-of-band trust config exists, and that file is not in the public BORG memory tree (`memory-runtime/authority.py:24-27`).

C12 must not be summarized as "the 4B failed and is shipped anyway." The shipped card says the predecessor failed and the retrain is exam-qualified and uncanaried. Capture-extraction content agreement is explicitly unmeasured (`adapters/README.md:76`).

C18 on Omnara is not a near miss. Hosted credential provisioning creates a provider secret (`hosted_credentials.go:32-37`). It does not move work between subscription seats.

C21 is a difference of product intent. Omnara stores the transcript for the tenant. Plimsoll's default is to forget the transcript and keep the measurement.

## Five axes

| Axis | Ours | Omnara | Reason |
| --- | --- | --- | --- |
| Managed execution | 3 | 4 | Omnara has one agent kernel, steering queue, subagents, and sandboxes. Our steer and interrupt are strong for Codex, weaker for Grok, and absent mid-turn for Claude. |
| Collective memory | 4 | 1 | We have scoped recall, a default-off graph, and retention markers. Omnara's 1 is durable per-agent history, not a memory product. |
| Fleet operations | 4 | 3 | We fail closed on stale capacity and rank Codex allowance. Omnara has pools, daemons, grants, and RBAC, without host-pressure admission. |
| Delivery economics | 4 | 2 | Plimsoll joins usage to merged pull requests. Omnara sums provider-reported cost and stops there. |
| Product readiness | 3 | 4 | Omnara is a documented multi-tenant product with insecure dev defaults. We are a verified Apple Silicon install, with benched adapters and a cloud that cannot take live payment. |

These axes are judgments. They are not sums of the rows, so memory and BORG are not double-counted into one total.

## Reciprocal opportunities

**Import the input kernel.** Put `queued`, `steering`, and `immediate` in front of every conductor, and keep native thread readback as the provider authority. Value 5, effort L. The failure mode is two logs that disagree about which turn is live.

**Import the sandbox contract, optionally.** A Blaxel/Daytona/Modal/Unikraft-style provider beside BYO SSH machines, with one live resource per machine id. Value 4, effort L. Cloud credentials must not enter the memory home.

**Import permission modes.** Replace the Codex default of never-approve with `always_allow`, `always_ask`, and `always_deny`, and record the denial. Value 5, effort M. Unattended lanes need a timeout or they will stall.

**Contribute outcome linkage.** An Omnara export that hashes repo and branch the way Plimsoll does, and that never uploads raw prompts. Value 5, effort M. A wrong repo hash attributes spend to the wrong pull request.

**Contribute non-authoritative recall.** A scoped MCP in front of mem0, with the graph feed left default-off and with retention withholding. Value 4, effort L. The failure mode is an agent treating a recalled sentence as an instruction.

**Contribute admission results, not the whole router.** Publish fail-closed capacity for operators who run Omnara workers on their own hosts. Value 3, effort M. Pool sandboxes often have no OS load signal, so the Codex ranker must not be copied onto them.

**Do not promote the extraction adapters into either request path.** Value 2, effort S. Exam JSON validity already failed as a promotion rule once.

## Unknowns

- Whether the pinned BORG commit's adapter weights match the supplement cards. The tensors were not in either inspected tree.
- Whether Linux or Intel installation actually completes. The docs say acceptance is pending.
- Live fleet health, provider balances, and Omnara Cloud behavior. This audit did not log in or start services.
- How far the installed `memory-runtime` has drifted from public `borg/memory` beyond the files that were opened. `authority.py` is one confirmed addition.
- Whether Plimsoll or plimsoll-cloud snapshots contain uncommitted edits relative to their named commits. The contract allows that, and this audit did not diff them against remotes.
- Gemini and Grok collector coverage beyond the setup comment and the shared suppression policy. A cold-ledger doctor sentence names Claude Code or Codex as the example signal (`plimsoll/README.md:208-209`).

[Sharing note: internal runtime policy receipt is retained in the private original.]