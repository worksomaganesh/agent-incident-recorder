# Agent Incident Recorder

A forensic recording and investigation layer for AI-agent systems.

**Agent Incident Recorder** is a planned tool for capturing structured agent activity, preserving an auditable event history, and helping developers reconstruct what happened during a surprising, unsafe, or failed agent run.

It is designed to complement runtime controls such as an AI Agent Firewall: the firewall aims to control actions; this project aims to make the resulting activity easier to inspect and investigate.

> **Project status:** Documentation-first starting point. The repository currently contains this README; the collector, storage layer, UI, and replay tooling have not been implemented yet. The architecture and stack below describe the intended design, not existing functionality.

## Why this project exists

When an agent edits files, invokes tools, runs commands, or encounters a policy denial, a developer may need to answer questions such as:

- What happened, and in what order?
- Which agent, session, tool, or policy decision was involved?
- What evidence was available at the time?
- Did a command fail, get denied, or simply never run?
- Has the stored event history changed since it was recorded?
- Can the incident be shared for review without exposing tokens, credentials, or private data?

Ordinary application logs often lack a common event schema and a reliable way to connect activity across components. Agent Incident Recorder aims to provide that common forensic trail.

## Design goals

- **Structured evidence:** normalize events from supported agents, gateways, and local adapters into a documented schema.
- **Timeline reconstruction:** correlate events by run, session, parent event, and timestamps.
- **Tamper evidence:** link stored events with cryptographic hashes so unexpected changes to the recorded sequence can be detected.
- **Useful investigations:** filter a run, inspect related events, and export a portable incident report.
- **Safe handling:** redact configured secrets and sensitive fields before durable storage or export.
- **Clear provenance:** preserve which adapter supplied each event and distinguish observed facts from derived interpretations.
- **Incremental integration:** begin with a mock agent and JSON event input before adding integrations.

## Intended architecture

```text
Agent / mock runner / supported gateway
                 |
                 v
       Event adapter and validator
                 |
                 v
       Redaction and normalization
                 |
                 v
       Append-only event store
          + hash-chain metadata
                 |
        +--------+---------+
        |                  |
        v                  v
 Timeline / filters   Incident export
        |
        v
 Read-only replay simulator
```

The components are intended to have narrow responsibilities:

1. **Event adapters** translate source-specific records into the common schema. They should not invent events that the source did not observe.
2. **Validator** rejects malformed events and checks required identifiers and types.
3. **Redaction layer** applies configured field and pattern rules before persistence or export.
4. **Event store** persists normalized records and integrity metadata.
5. **Timeline service** groups and orders related events, while preserving original timestamps and source sequence values.
6. **Export and replay tools** help reviewers inspect a case offline. Replay initially means reconstructing or simulating a sequence from recorded data—not re-running arbitrary commands from an incident.

## Planned event model

A normalized event is expected to include fields such as:

| Field | Purpose |
| --- | --- |
| `event_id` | Unique identifier for this recorded event |
| `schema_version` | Version of the event contract |
| `run_id` / `session_id` | Correlation identifiers |
| `timestamp` | Source event time, with timezone information where available |
| `observed_at` | Time the recorder received the event |
| `source` | Adapter or component that supplied the record |
| `event_type` | For example, tool request, policy decision, command result, or file change |
| `actor` / `target` | Agent, tool, resource, or workspace involved |
| `outcome` | Allowed, denied, succeeded, failed, or unknown, where applicable |
| `evidence` | Bounded structured details needed for investigation |
| `previous_hash` / `event_hash` | Integrity-link fields for the stored sequence |

The final schema will be versioned and validated. Raw payloads should be bounded in size, and sensitive values should be redacted or omitted whenever possible.

## Example investigation record

The following is an illustrative shape, not output from a working implementation:

```json
{
  "schema_version": "0.1",
  "event_id": "evt-example-0003",
  "run_id": "run-example-001",
  "timestamp": "2026-01-01T10:00:03Z",
  "source": "mock-agent-adapter",
  "event_type": "policy.decision",
  "actor": "example-agent",
  "target": "filesystem.write",
  "outcome": "denied",
  "evidence": {
    "reason_code": "path_outside_workspace",
    "resource": "requested/path.txt"
  }
}
```

Real exports should clearly label illustrative, inferred, missing, and source-reported data.

## Intended MVP

The first milestone should be deliberately bounded:

1. Define and document a versioned event schema.
2. Build a mock agent that emits a deterministic sequence of events.
3. Validate and normalize those events.
4. Store them locally in SQLite.
5. Link stored events with a hash chain and verify the chain on demand.
6. Provide a CLI to list runs, inspect a timeline, and export a JSON incident bundle.
7. Add tests for malformed input, missing events, ordering, redaction, and tampering.
8. Document the known limitations of each evidence source.

A web interface, live integrations, distributed storage, and automated incident classification are later possibilities—not MVP assumptions.

## Proposed technology choices

These are candidates to evaluate during implementation, not installed or verified dependencies.

- **Language:** Python 3.12+
- **Data contracts:** Pydantic or equivalent schema validation
- **Local persistence:** SQLite for the first version
- **CLI:** Typer or the standard library's argument parser
- **API (if needed):** FastAPI
- **Testing:** pytest, including adversarial and property-oriented cases
- **Quality checks:** Ruff, type checking, and GitHub Actions

Prefer the smallest dependency set that supports testable, understandable behavior.

## Security and trust boundaries

A forensic recorder is only as trustworthy as its collection path and storage controls.

- **Hash chains detect certain modifications; they do not prove that the original events were truthful or complete.** An attacker able to rewrite the full store and all associated anchors may be able to replace the history.
- Events may be absent if an adapter is bypassed, disabled, compromised, or loses connectivity. The UI and exports must not equate “no event recorded” with “nothing happened.”
- Source timestamps may be inaccurate, out of order, or controlled by the source. Preserve receipt time and source provenance.
- Redaction can miss secrets or remove context needed for an investigation. Make policies configurable and test them.
- Do not automatically execute recorded commands during replay. A future replay feature must default to inert simulation and require explicit isolation and authorization for any execution-oriented mode.
- Keep database files and incident exports private by default. Avoid recording access tokens, passwords, private keys, or complete environment dumps.

The first version should be described as a developer-focused evidence tool, not a certified forensic platform or a complete security boundary.

## Testing strategy

Planned tests should cover at least:

- valid and invalid event payloads;
- unknown schema versions and forward-compatible fields;
- duplicate identifiers, clock skew, and out-of-order arrivals;
- truncated input and oversized payloads;
- secret redaction before persistence and export;
- hash-chain validation after record modification, deletion, reordering, or truncation;
- interrupted writes and partial incident exports;
- ambiguous or missing event sources;
- deterministic timeline output for the same input dataset.

Tests should verify behavior, not only the presence of functions or files.

## Roadmap

- [ ] **Milestone 0 — Contracts:** threat model, event schema, terminology, and design decisions.
- [ ] **Milestone 1 — Local recorder:** mock source, validation, redaction, and SQLite persistence.
- [ ] **Milestone 2 — Integrity:** hash-chain generation, verification, and corruption tests.
- [ ] **Milestone 3 — Investigations:** timeline CLI, filters, and JSON export.
- [ ] **Milestone 4 — Safe replay:** deterministic, read-only simulation of recorded events.
- [ ] **Milestone 5 — Integration:** adapter for a real agent or gateway with documented provenance.
- [ ] **Milestone 6 — Release readiness:** threat-model review, reproducible setup, CI, examples, and known-limitations documentation.

## Repository principles

- Evidence and inference must be visibly distinct.
- A missing event must never be silently represented as a successful action.
- Integrity failures must be surfaced as failures, not warnings that are easy to ignore.
- Incident export must use a stable, versioned format.
- Every meaningful security claim needs a test or a stated limitation.
- The recorder should not require an LLM to validate hashes, order records, or decide whether an integrity check passed.

## Contributing

Contributions are welcome after the initial implementation begins. Start by opening an issue describing the problem, proposed behavior, threat-model implications, and how it can be tested. Keep changes small and include regression tests for bugs.

Do not submit real customer incident data, credentials, private logs, or personal information in issues or pull requests. Use synthetic fixtures.

## License

A license has not yet been selected for this repository. Until a license file is added, do not assume that the code or documentation is available for unrestricted reuse.

## Disclaimer

This project is a work in progress and is not production-ready. It does not replace endpoint monitoring, access control, secure storage, incident-response procedures, or independent validation of evidence sources.
