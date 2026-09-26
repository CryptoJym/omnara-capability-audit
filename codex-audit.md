# Codex lead independent audit

Sealed before reviewing the Opus and Grok reports. Scope: frozen Omnara public source 9f9ef08158c7, BORG public source 57ae994cfbe1, selected installed memory code, and the hashed Plimsoll working-tree packet. Scores are expert judgments, not performance measurements.

## Judgment

Omnara is the stronger unified managed-agent product in the inspected source. It combines durable execution, organizations/projects, model configuration, approvals, machine pools, streaming events, a dashboard and a public API. Our combined stack is differentiated by cross-runtime memory, native owner-machine operations, explicit work ownership, physical fleet admission, and privacy-aware links between usage and accepted work. These are complementary layers. Replacing our entire stack would discard useful capabilities and create a migration problem larger than the demonstrated benefit.

The highest-value first experiment is an optional Omnara agent using a narrowly scoped BORG memory MCP server, with Omnara usage exported through a Plimsoll adapter. Start with a read-only, synthetic project; prove identity isolation, duplicate handling and native result attribution before expanding. This is an integration hypothesis, not a tested compatibility claim.

## What Omnara does particularly well

Its state and authorization contracts are explicit. Runtime leases, model-call recovery and durable events fit together in one control plane; approvals bind canonical arguments and preserve canceled interactions. Organization/project roles and machine-pool grants make team operations more legible than a collection of owner-specific tools. First-class subagents and per-agent timelines remove coordination glue an adopter would otherwise write.

The system also recognizes uncertainty: runtime recovery records ambiguous outcomes. A Postgres transaction cannot make an arbitrary remote side effect exactly once. Custom-tool executors and MCP integrations still need stable identity, reconciliation and retry contracts. Pool limits constrain allocated capacity; they do not establish the real headroom of a busy personal workstation.

## What our systems do particularly well

The memory layer has an actual scoped semantic retrieval interface, temporal graph boundary, provenance handling, selection, guarded consolidation and retention machinery. Omnara has persisted conversation state and compaction; no comparable shared semantic-memory subsystem was found in the inspected source. An external MCP server could supply that capability.

BORG makes owner-controlled computers available through authenticated MCP with native browser/UI operations, resource coordination and operation receipts. It offers a different trust model from disposable cloud sandboxes. Plimsoll has explicit reported/estimated/unknown cost fields, separate token dimensions, privacy suppression, outcome linkage and metric-coverage contracts. Those distinctions are more valuable than an undifferentiated token-cost dashboard. They still do not prove causal productivity gains or complete financial reconciliation.

## Weaknesses we should take seriously

Our strongest functionality spans multiple products, public snapshots and installed estate code. A new adopter does not automatically receive every private fleet capability. The public BORG checkout and current installed memory code differ; the local pre-existing Borg checkout was also behind the public clone. The deployed connector reported healthy graph access but no ingestion watermark or last-complete-scan value. The aggregate fleet-context call returned UNKNOWN because of its output bound, even though lower-level measurements covered seven hosts.

This audit encountered native model-name drift while requesting Grok 4.7. A native model catalog accepted the model but a controller rejected its label. This is concrete integration friction, not a lack of Grok capability. The requested Plimsoll native status command also exceeded the audit's deadline; only that read process was stopped. The cause was not determined, and the collector was not restarted.

Plimsoll Cloud's inspected README says deployed for testing/hardening and not approved for paid production. Its economics contracts must not be advertised as a complete, live ROI measurement. BORG's included small-model adapters remain qualified research artifacts rather than a demonstrated autonomous improvement loop.

## Reciprocal value

From Omnara: adopt the interaction/timeline patterns, study lease/recovery contracts and optionally use its sandbox pools. Prefer adapting interfaces over copying a whole control plane into a second work-state authority. Its Go/Postgres/Redis/blob-store stack and API billing differ materially from our native subscription-conductor operations.

To Omnara: contribute a scoped BORG MCP example, a Plimsoll usage/outcome adapter, coverage-aware metric fixtures, and an optional BYO-host admission interface. Contribute public code and synthetic fixtures only; private memory records, cloud code, account credentials and training corpora are not part of this proposal. Omnara is Apache-2.0; BORG owner code is MIT with separate third-party terms, including FalkorDB's SSPL distinction. Check exact notices and dependency boundaries for each concrete contribution.

## Verification and limits

- Fresh public source snapshots were pinned and the audit packet was hashed.
- A native Mem0 query returned candidate recall. Borg's live status read reported connector, memory and graph PASS, while graph freshness remained unknown.
- 41 focused BORG tests passed for concurrency, process handoff and independent settings. No full-stack certification follows.
- Selected Omnara unit tests were attempted with automatic toolchain changes disabled. They did not run: Go 1.27.1 is required and 1.25.6 was installed.
- Omnara hosted behavior, fresh self-host installation, remote sandbox providers, adversarial tenant boundaries and production load were not exercised.
- Private Plimsoll source and installed memory were inspected; their deployment equivalence was not assumed.

The companion RATINGS.json preserves all 24 capability judgments, sources and criticality. No overall feature total is used because the capabilities overlap and the systems optimize for different jobs.

## Lead self-check after the initial seal

Before reading the other auditors, I tightened C07 (Omnara native desktop/browser interaction) from 2/5 partial to 0/5 not found. Its implemented web search/fetch and command execution do not establish native UI automation. External browser tools can be integrated through MCP, but that possibility is not an implemented built-in capability. The initial sealed ratings remain preserved; the presented ratings include this explicit correction.

## Post-review source qualifications

After reading Opus’s audit, I verified Omnara’s production SSRF policy and the public BORG router/HTTP implementation. My C01 description of compatible local endpoints needs a qualification: production rejects loopback, private LAN and tailnet ranges; only the insecure development flag enables loopback. A scoped, reachable HTTPS endpoint or a deliberately reviewed egress extension is required.

Opus’s public BORG findings materially strengthen the case for improving execution bookkeeping before adding another control plane. The inspected public router has no normal completion-state writer in its own dispatch path, bounded receipt scans and no stale-PID lock recovery; the public conductor trusts local HTTP clients. These are source findings, not proof that the current private estate has the same deployment or is exploitable remotely.

I do not adopt absolute claims that content can never be deleted or every restart necessarily duplicates spend. The inspected Omnara application lacks a normal retention/erasure path for immutable content, and interrupted model calls can become ambiguous; administrative deletion and actual duplicate charges were not tested. Likewise, standard owner-process stdio trust is not itself a remote authentication vulnerability. The public matrix retains independent ratings; these qualifications guide the combined recommendations.

### Additional source qualification after independent review

The C12 local-endpoint statement is conditional on Omnara's deployment egress policy. The managed model/MCP HTTP clients restrict private, tailnet and production loopback destinations. Provider compatibility therefore does not prove production local inference; there is still no inspected extraction-training or promotion pipeline. The lead's C12 score remains 1, with this qualifier recorded in the presentation ratings. The initial sealed audit remains preserved.

Grok's strongest disagreement is outcome linkage: it gives our stack 4/5 where the lead gives 3 and Opus gives 2. The source does implement meaningful joins and validation, but current coverage and causal value remain unproven. Grok's suggested permission-mode work should preserve the owner's authorized unattended operation; explicit caller identity, scope and allow/deny profiles are preferable to making every action wait on a human.

Grok’s development-key wording needs a correction: Omnara’s insecure-development configuration installs a known fallback encryption key when keys are omitted, rather than leaving the effective key empty. Its adapter distinction is correct: the cited failed canary belongs to a predecessor; the shipped clean retrain is exam-qualified but has not completed that canary. Neither is production-readiness proof.