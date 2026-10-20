# Worked examples

These examples are informative and fictional. They show logical records, not executable migrations or a complete application. All three examples share scope `workspace:demo`; references such as `vehicle:v-4318` resolve to existing domain entities in that scope. `user:operator`, `user:customer`, `agent:operations`, and `system:workshop` are registered actors.

The JSON snapshots use the exact logical field names from the [specification](0.1.md). Optional fields are omitted when unused. Lists under a table name represent its rows. Read the timelines as the sequence of writes; the snapshots show the final state of mutable tasks.

## 1. Insurance arrives before the scheduled review

At 08:00 on October 20, a customer promises to upload insurance tomorrow. An agent records the report, the observation, and a decision to wait until October 21 at 09:00 UTC, with earlier review on `insurance.uploaded`.

At 10:00 on October 20, a document upload event identifies the document as its primary subject and the vehicle as a related subject. It triggers review of the waiting decision. An authorized operator verifies the coverage; the reviewer records a superseding, resolved decision. Merely uploading a file would not establish valid coverage.

The resolved decision creates a notification task. The task moves from `pending` to `in_progress` to `completed`, with events preserving those transitions and the actual notification result.

```json
{
  "events": [
    {
      "id": "e-promise",
      "scope_id": "workspace:demo",
      "subject_ref": "vehicle:v-4318",
      "recorded_at": "2026-10-20T08:00:00Z",
      "recorded_by": "user:operator",
      "event_type": "customer.promise.received",
      "occurred_at": "2026-10-20T08:00:00Z",
      "summary": "Customer promised to upload insurance tomorrow.",
      "source_ref": "message:m-1",
      "payload": { "customer_ref": "user:customer" }
    },
    {
      "id": "e-upload",
      "scope_id": "workspace:demo",
      "subject_ref": "document:insurance-1",
      "recorded_at": "2026-10-20T10:00:00Z",
      "recorded_by": "user:customer",
      "event_type": "insurance.uploaded",
      "occurred_at": "2026-10-20T10:00:00Z",
      "summary": "Customer uploaded a new insurance document.",
      "related_subject_refs": ["vehicle:v-4318"],
      "source_ref": "document:insurance-1",
      "payload": {}
    },
    {
      "id": "e-verified",
      "scope_id": "workspace:demo",
      "subject_ref": "vehicle:v-4318",
      "recorded_at": "2026-10-20T10:04:00Z",
      "recorded_by": "user:operator",
      "event_type": "insurance.coverage.verified",
      "occurred_at": "2026-10-20T10:04:00Z",
      "summary": "Operator verified that the uploaded policy covers this vehicle.",
      "related_subject_refs": ["document:insurance-1"],
      "payload": { "document_ref": "document:insurance-1" }
    },
    {
      "id": "e-notify-started",
      "scope_id": "workspace:demo",
      "subject_ref": "vehicle:v-4318",
      "recorded_at": "2026-10-20T10:06:00Z",
      "recorded_by": "agent:operations",
      "event_type": "task.started",
      "occurred_at": "2026-10-20T10:06:00Z",
      "summary": "Agent claimed the customer notification task.",
      "payload": {
        "task_id": "t-notify",
        "from_status": "pending",
        "to_status": "in_progress"
      }
    },
    {
      "id": "e-notified",
      "scope_id": "workspace:demo",
      "subject_ref": "vehicle:v-4318",
      "recorded_at": "2026-10-20T10:07:00Z",
      "recorded_by": "agent:operations",
      "event_type": "customer.notified",
      "occurred_at": "2026-10-20T10:07:00Z",
      "summary": "Insurance confirmation was delivered to the customer.",
      "source_ref": "message:m-2",
      "payload": {
        "task_id": "t-notify",
        "from_status": "in_progress",
        "to_status": "completed"
      }
    }
  ],
  "observations": [
    {
      "id": "o-promise",
      "scope_id": "workspace:demo",
      "subject_ref": "vehicle:v-4318",
      "recorded_at": "2026-10-20T08:01:00Z",
      "recorded_by": "agent:operations",
      "observation_type": "insurance.customer-promise",
      "observed_at": "2026-10-20T08:00:00Z",
      "assertion": "Customer says the insurance will be uploaded tomorrow.",
      "state": "asserted",
      "evidence_refs": [{ "table": "events", "id": "e-promise" }],
      "payload": {}
    },
    {
      "id": "o-covered",
      "scope_id": "workspace:demo",
      "subject_ref": "vehicle:v-4318",
      "recorded_at": "2026-10-20T10:04:30Z",
      "recorded_by": "user:operator",
      "observation_type": "insurance.coverage",
      "observed_at": "2026-10-20T10:04:00Z",
      "assertion": "Verified policy covers this vehicle at the time of review.",
      "state": "asserted",
      "evidence_refs": [{ "table": "events", "id": "e-verified" }],
      "payload": { "document_ref": "document:insurance-1" }
    }
  ],
  "decisions": [
    {
      "id": "d-wait",
      "scope_id": "workspace:demo",
      "subject_ref": "vehicle:v-4318",
      "recorded_at": "2026-10-20T08:02:00Z",
      "recorded_by": "agent:operations",
      "decision_type": "insurance.follow-up",
      "concern": "insurance",
      "outcome": "wait",
      "conclusion": "Wait for the promised insurance upload.",
      "rationale": "The customer promised to provide it tomorrow; another reminder is premature.",
      "plan": "Review when insurance is uploaded, or tomorrow at 09:00 UTC if it has not arrived.",
      "evidence_refs": [{ "table": "observations", "id": "o-promise" }],
      "payload": {},
      "review_at": "2026-10-21T09:00:00Z",
      "review_on_event_types": ["insurance.uploaded"]
    },
    {
      "id": "d-insurance-resolved",
      "scope_id": "workspace:demo",
      "subject_ref": "vehicle:v-4318",
      "recorded_at": "2026-10-20T10:05:00Z",
      "recorded_by": "agent:operations",
      "decision_type": "insurance.follow-up",
      "concern": "insurance",
      "outcome": "resolved",
      "conclusion": "Resolve this missing-insurance follow-up.",
      "rationale": "The uploaded document has been verified for this vehicle.",
      "plan": "Stop waiting for this upload and notify the customer. Future expiry is a separate concern.",
      "evidence_refs": [{ "table": "observations", "id": "o-covered" }],
      "payload": {},
      "supersedes_decision_id": "d-wait"
    }
  ],
  "tasks": [
    {
      "id": "t-notify",
      "scope_id": "workspace:demo",
      "subject_ref": "vehicle:v-4318",
      "recorded_at": "2026-10-20T10:05:30Z",
      "recorded_by": "agent:operations",
      "task_type": "customer.notify",
      "title": "Tell the customer the insurance follow-up is complete.",
      "status": "completed",
      "assignee_ref": "agent:operations",
      "payload": { "customer_ref": "user:customer" },
      "updated_at": "2026-10-20T10:07:00Z",
      "decision_id": "d-insurance-resolved",
      "result_event_id": "e-notified"
    }
  ]
}
```

No movement occurred, so `event_lines` is empty. `d-wait` remains historical; `d-insurance-resolved` is the current decision for the insurance concern. The customer-promise observation remains a record of what was reported, rather than a claim that the upload is still outstanding.

If the file had not arrived, `d-wait` would become due at its scheduled time. If verification failed, review would record a different successor with the failed verification as evidence and a new plan; an upload alone must not resolve the concern.

## 2. Workshop follow-up for the same vehicle

At 10:10, the customer reports unusual fuel consumption. That is evidence of a report, rather than proof of a mechanical fault. The agent chooses to arrange an inspection and creates a pending booking task. An actual workshop booking will trigger review of this decision.

```json
{
  "events": [
    {
      "id": "e-fuel-report",
      "scope_id": "workspace:demo",
      "subject_ref": "vehicle:v-4318",
      "recorded_at": "2026-10-20T10:10:00Z",
      "recorded_by": "user:operator",
      "event_type": "customer.fuel-concern.reported",
      "occurred_at": "2026-10-20T10:10:00Z",
      "summary": "Customer reported higher fuel consumption than usual.",
      "source_ref": "message:m-3",
      "payload": { "customer_ref": "user:customer" }
    }
  ],
  "observations": [
    {
      "id": "o-fuel-report",
      "scope_id": "workspace:demo",
      "subject_ref": "vehicle:v-4318",
      "recorded_at": "2026-10-20T10:10:30Z",
      "recorded_by": "agent:operations",
      "observation_type": "vehicle.fuel-concern",
      "observed_at": "2026-10-20T10:10:00Z",
      "assertion": "Customer reports increased fuel consumption; cause is unverified.",
      "state": "asserted",
      "evidence_refs": [{ "table": "events", "id": "e-fuel-report" }],
      "payload": {}
    }
  ],
  "decisions": [
    {
      "id": "d-inspect",
      "scope_id": "workspace:demo",
      "subject_ref": "vehicle:v-4318",
      "recorded_at": "2026-10-20T10:11:00Z",
      "recorded_by": "agent:operations",
      "decision_type": "workshop.inspection-plan",
      "concern": "fuel-inspection",
      "outcome": "act",
      "conclusion": "Arrange a workshop inspection of the reported fuel issue.",
      "rationale": "The report needs investigation before choosing a repair.",
      "plan": "Ask an operator to book an inspection; review after booking or tomorrow at 09:00 UTC.",
      "evidence_refs": [{ "table": "observations", "id": "o-fuel-report" }],
      "payload": {},
      "review_at": "2026-10-21T09:00:00Z",
      "review_on_event_types": ["workshop.visit.booked"]
    }
  ],
  "tasks": [
    {
      "id": "t-book-inspection",
      "scope_id": "workspace:demo",
      "subject_ref": "vehicle:v-4318",
      "recorded_at": "2026-10-20T10:12:00Z",
      "recorded_by": "agent:operations",
      "task_type": "workshop.book-inspection",
      "title": "Arrange a workshop inspection for the fuel concern.",
      "status": "pending",
      "assignee_ref": "user:operator",
      "payload": { "vehicle_ref": "vehicle:v-4318" },
      "updated_at": "2026-10-20T10:12:00Z",
      "decision_id": "d-inspect",
      "due_at": "2026-10-21T08:00:00Z"
    }
  ]
}
```

Insurance and inspection now have separate current decisions for the same vehicle. An agent sees a resolved insurance-upload concern and outstanding workshop work through the same record structure. Neither conclusion alone establishes that the vehicle is fit to return to service; that requires the applicable domain checks.

## 3. Equipment transfer

A workshop mounts one serialized printer in the vehicle. The event describes the occurrence; its lines describe custody moving between accounts. The workshop previously held this printer, and domain checks authorize the transfer.

```json
{
  "events": [
    {
      "id": "e-printer-mounted",
      "scope_id": "workspace:demo",
      "subject_ref": "vehicle:v-4318",
      "recorded_at": "2026-10-20T11:00:00Z",
      "recorded_by": "system:workshop",
      "event_type": "equipment.mounted",
      "occurred_at": "2026-10-20T11:00:00Z",
      "summary": "Workshop mounted printer p-7 in the vehicle.",
      "related_subject_refs": ["equipment:p-7"],
      "payload": { "equipment_ref": "equipment:p-7" }
    }
  ],
  "event_lines": [
    {
      "id": "line-workshop-out",
      "scope_id": "workspace:demo",
      "subject_ref": "vehicle:v-4318",
      "recorded_at": "2026-10-20T11:00:00Z",
      "recorded_by": "system:workshop",
      "event_id": "e-printer-mounted",
      "account_ref": "account:workshop",
      "resource_ref": "equipment:p-7",
      "quantity": "-1",
      "unit": "item"
    },
    {
      "id": "line-vehicle-in",
      "scope_id": "workspace:demo",
      "subject_ref": "vehicle:v-4318",
      "recorded_at": "2026-10-20T11:00:00Z",
      "recorded_by": "system:workshop",
      "event_id": "e-printer-mounted",
      "account_ref": "account:vehicle-v-4318",
      "resource_ref": "equipment:p-7",
      "quantity": "1",
      "unit": "item"
    }
  ]
}
```

For this event, printer, and unit, `-1 + 1 = 0`. These lines show a change, not the opening balance: balances need the earlier acquisition history too. A later demount or correction creates another balanced event. It does not edit this transfer. No observation, decision, or task is required merely to record the movement.

## Illustrative queries

These queries assume the canonical logical names are exposed as SQL columns. Bind parameters through the application's database driver. They illustrate reads; they do not define a physical schema or complete review queue.

### Current decisions across features

```sql
SELECT d.id, d.concern, d.outcome, d.conclusion, d.review_at
FROM decisions AS d
WHERE d.scope_id = :scope_id
  AND d.subject_ref = :subject_ref
  AND NOT EXISTS (
    SELECT 1 FROM decisions AS successor
    WHERE successor.scope_id = d.scope_id
      AND successor.supersedes_decision_id = d.id
  )
ORDER BY d.concern;
```

For `workspace:demo` and `vehicle:v-4318`, this returns `d-inspect` and `d-insurance-resolved`, with their distinct concerns.

### Scheduled reviews that are due

This example assumes a database-native timestamp mapping so `<=` compares instants. A string-backed implementation must normalize precision or use its database's timestamp conversion.

```sql
SELECT d.id, d.subject_ref, d.concern, d.review_at
FROM decisions AS d
WHERE d.scope_id = :scope_id
  AND d.review_at IS NOT NULL
  AND d.review_at <= :now
  AND NOT EXISTS (
    SELECT 1 FROM decisions AS successor
    WHERE successor.scope_id = d.scope_id
      AND successor.supersedes_decision_id = d.id
  );
```

At `2026-10-21T09:00:00Z`, the combined examples return only `d-inspect`. The superseded `d-wait` must not reappear merely because its old review time has arrived. Event-triggered reviews need the matching and durable bookkeeping described in the specification, beyond this scheduled query.

### Pending work and its current justification

```sql
SELECT t.id, t.title, d.concern, d.conclusion
FROM tasks AS t
JOIN decisions AS d
  ON d.scope_id = t.scope_id AND d.id = t.decision_id
WHERE t.scope_id = :scope_id
  AND t.subject_ref = :subject_ref
  AND t.status = 'pending'
  AND NOT EXISTS (
    SELECT 1 FROM decisions AS successor
    WHERE successor.scope_id = d.scope_id
      AND successor.supersedes_decision_id = d.id
  );
```

This returns `t-book-inspection`. Tasks without a decision need a separate branch in a complete work-inbox query. Executors must still check permissions, domain preconditions, and pending reviews before acting.

## Acceptance scenarios

Implementers can use these scenarios to assess their mapping and behavior. This specification release includes no executable test suite.

| Scenario                                                      | Expected behavior                                                                                                                               |
| ------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| No upload before the waiting decision's scheduled time        | The current waiting decision becomes due, remains visible, and can be retried after a failed review.                                            |
| Insurance upload before the review time                       | The event's related vehicle and type trigger earlier review; upload alone does not establish valid coverage.                                    |
| Matching event has a different subject or scope               | It does not trigger this decision; cross-scope references and evidence access are rejected.                                                     |
| Evidence arrives late or shares a timestamp with the decision | Recording order determines whether it is new; old occurrence time or equal timestamps do not drop new evidence.                                 |
| Evidence arrives while a review is running                    | Evidence outside the review context remains pending against the resulting current decision.                                                     |
| Review reaffirms waiting                                      | A successor preserves the earlier judgment and records a renewed rationale and review conditions.                                               |
| Two reviewers replace the same current decision               | At most one successor commits; the stale reviewer reloads rather than forking the chain.                                                        |
| An event is delivered twice                                   | One source occurrence and one pending review intent are preserved without duplicate actions.                                                    |
| A decision is superseded while its task is pending            | The old task cannot execute under the obsolete justification; it is cancelled or replaced.                                                      |
| A relevant review is pending immediately before execution     | Resolve the review before an action it could invalidate; record actual outcomes if an external action is already committed.                     |
| A task completes or fails                                     | Completion links a result event; failure has a reason; state transitions remain inspectable. A retry uses a new task with effect deduplication. |
| Independent insurance and inspection decisions coexist        | Both are current under their own concern keys; resolving one does not close the other.                                                          |
| Two observations conflict                                     | Both remain inspectable; recency alone does not establish truth. A withdrawal explicitly supersedes an assertion.                               |
| A printer transfers between workshop and vehicle              | Atomic lines sum to exactly zero for the same event, resource, and unit; custody is checked separately.                                         |
| A movement is corrected                                       | A new compensating event preserves the original transfer. Ordinary events remain valid with no lines.                                           |
| A projection is rebuilt                                       | Canonical history produces the same current decisions and balances without repeating external effects.                                          |
