# Implementing Decision Schema

This guide is informative. The [0.1 specification](0.1.md) defines the requirements.

## Adopt a shared layer incrementally

Start with one real concern, such as missing insurance. Keep documents and vehicles in their existing domain tables. Add the shared logical tables once, then have other features use them.

1. Choose stable subject references and an ownership scope.
2. Document the first event, observation, decision, and task types and their payloads.
3. Write domain changes and their business events atomically where possible. Across systems, use durable delivery and reconciliation.
4. Record evidence, decisions, review conditions, and work through shared operations.
5. Add a second feature and verify that the same queries can reconstruct both concerns for one subject.

Avoid separate `insurance_decisions` and `workshop_decisions` contracts with incompatible lifecycles. Keep specialized validation in domain code and expose the shared fields consistently. A type vocabulary is necessary: sharing columns while leaving every payload undocumented does not make the information understandable.

## Storage and validation

Five physical SQL tables are a useful starting point, but the standard defines logical records. Evidence arrays may be JSON or normalized relationships. Indexes, source deduplication keys, review queues, and read projections belong to the implementation.

Validate input shapes at the boundary, then validate references, scope, permissions, and domain rules. Database constraints should support application validation where possible. For decisions, protect both the single root per subject/concern and the single successor per predecessor inside a transaction or equivalent atomic operation. A “check, then insert” sequence without concurrency protection is insufficient.

Useful indexes include:

- Scope, subject, and recording order for history.
- Scope, subject, concern, and predecessor for current decisions.
- Current decision review time for scheduled scans.
- Event type and subject references for event-trigger matching.
- Task assignee, status, and due time for work queues.
- Event-line resource, account, and unit for balances.

Use exact decimal arithmetic for movement lines. Validate and commit the complete balanced group atomically. A balanced transfer still needs domain checks such as whether a serialized printer is actually held by the sending account.

## Reconstruct context for humans and agents

For an authorized scope and subject, load recent events, current observations, current decisions for each concern, and outstanding tasks. Include supporting evidence, relevant related-subject records, and reasons a review is due. Bound and page results; make older history accessible by reference.

Expose disagreements and unknowns. “No inspection observation exists” does not mean “the inspection passed.” Clearly distinguish a customer report from a verified assessment.

Store the decision's useful business explanation, chosen plan, and evidence references. Raw chat history, tool transcripts, and private model reasoning traces are unnecessary for this contract. An agent should be able to explain a decision from these shared records.

## Process scheduled and event-triggered reviews

One implementation can use a database-backed review queue:

1. Atomically record an event and durable work to match it against current decisions.
2. Match its scope, type, primary subject, and related subjects. Coalesce pending reviews for each current decision while retaining triggering event IDs.
3. Also enqueue current decisions whose review time has arrived.
4. Claim review work and load the current chain head, evidence, and a recording-order watermark.
5. Evaluate the concern and append a successor using an atomic predecessor check.
6. Reconcile pending tasks and mark only the reviewed trigger IDs or watermark as handled.
7. Retain or enqueue a review for evidence recorded after the context watermark, even if it arrived before the successor was committed.

Record a successor even when the result remains “wait”; its new rationale and review conditions explain the reaffirmation. A failed review leaves the old decision visible and reviewable. Multiple events can justify one review; redelivery of one event must not manufacture additional judgments.

Use recording order, not only `occurred_at`, when handling late input. An inspection performed yesterday but imported today can still invalidate a decision made this morning. Do not use wall-clock timestamps alone to order concurrent commits.

The standard names the review conditions and observable behavior, while the queue, scheduler, event bus, outbox, worker, and locking strategy remain replaceable infrastructure.

## Execute tasks reliably

Claim a pending task atomically before marking it in progress. Recheck permission, the linked decision, outstanding relevant reviews, and domain preconditions immediately before an external action. When a new decision replaces an old one, reconcile its pending tasks and create replacement work where appropriate.

The check and the external action may not share a transaction. Document the action's commitment boundary: a message already submitted to a provider cannot always be recalled when a concurrent review changes the decision. Record what actually happened, then let subsequent decisions respond to it.

Use a stable idempotency key at an external provider when supported. After an ambiguous delivery failure, reconcile the provider outcome before creating a retry task. A new task ID must not accidentally bypass deduplication for the same intended action. Completion records a result event; status changes remain inspectable through events or another documented history mechanism.

## Present projected state

A customer or operator table can show one row per vehicle, with open concerns and next tasks. Treat it as a rebuildable read model rather than another place to edit decisions.

Link summaries to the evidence and decision chain. Show scheduled review times, event-triggered reviews, failures, and overdue work. A resolved insurance concern and an open inspection concern must remain separate even when they share a subject.

## Protect access and maintain the system

Apply the caller's permissions to record lists, evidence, related subjects, task execution, and search. Avoid revealing restricted source contents through a readable decision summary. Same-scope membership is necessary but may not be sufficient; source records can have stricter permissions.

Monitor review lag, due decisions, failed reviews, duplicate ingestion, task failures, and stale pending tasks. Support replaying event matching and rebuilding projections without repeating external actions.

Document retention and redaction, including how deleted source evidence is represented and how projections are repaired. Keep enough authorized provenance to distinguish unavailable evidence from evidence that never existed.

## Relationship to existing patterns

Decision Schema can coexist with Activity Schema for analytics, domain-specific transactional tables, event sourcing, and orchestration frameworks. It does not require reconstructing the entire application from events. The shared operational records are the contract; infrastructure can change independently.
