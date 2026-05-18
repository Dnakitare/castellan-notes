# Authority Model

Castellan's authority model is the concrete embodiment of the authority-laundering taxonomy applied to a working system. This document defines the shape of authority in the PoC and where each taxonomy gap is closed.

## The objects

### Principal
The identity on whose behalf an action is taken. In the PoC, principals are one of:
- `operator:<id>` — a human shift operator who submitted the ticket
- `system:orchestrator` — the orchestrator itself, for system actions not on behalf of a human
- `system:scheduler` — reserved for future scheduled actions; unused in the PoC

A principal is never `agent:<name>`. Agents do not hold authority. They exercise it on behalf of a principal.

### Capability
A scoped, time-limited grant of authority to perform a specific class of action. A capability is a concrete object in the capability ledger with the following fields (full schema in `event-schema.md`):

- `capability_id` — UUID
- `principal` — who this authority is on behalf of
- `requesting_agent` — which agent process requested it
- `scope` — what the capability permits (see below)
- `issued_at` — timestamp
- `expires_at` — timestamp; capabilities are always time-limited
- `parent_capability_id` — if this capability was issued in the context of another, the chain
- `decision` — `granted` | `denied`
- `decision_reasoning` — human-readable text from the broker
- `consumed_at` — set when the capability is used; capabilities are single-use in the PoC

### Scope
The set of actions a capability permits. Scope is structured, not free text:

```json
{
  "resource": "inventory.part" | "inventory.reservation" | "workorder" | "history.asset",
  "action": "read" | "reserve" | "draft" | "route_for_approval",
  "constraints": {
    "asset_id": "PUMP-3B",
    "part_id": "BEARING-X42",
    "quantity_max": 2,
    "workorder_severity_max": "medium"
  }
}
```

A capability's scope is the tightest envelope that allows the intended action. The broker rejects requests that ask for more scope than the principal's role permits or than the situation justifies.

### Principal role
Each principal has a role with declared maximum scope. For the PoC:
- `operator` may authorize: read history, read inventory, reserve parts up to a limit, draft work orders, route for approval. **May not** authorize: dispatching work, modifying SOPs, granting capabilities to other agents.
- The role is hardcoded in the broker's config for the PoC. Future ADR can move it to a config file or external policy service.

## How an action happens

1. Orchestrator decides an action is needed (e.g., reserve `BEARING-X42` quantity 1 for `PUMP-3B`).
2. Orchestrator calls `broker.request_capability` with:
   - `principal` — the operator who submitted the ticket
   - `requesting_agent` — `orchestrator`
   - `scope` — the tightest scope that allows the action
   - `justification` — short text: why this action, in service of what
   - `parent_capability_id` — if this is in the context of an outer capability, that ID
3. Broker evaluates:
   - Is the principal known and active?
   - Does the principal's role permit this scope?
   - Are the constraints reasonable (quantity within limit, severity within limit, asset matches the ticket)?
   - Is the parent capability, if any, still valid and does it permit this child scope?
4. Broker issues or denies. Either way, the decision is logged to the capability ledger and a `capability.decision` event is emitted to the audit bus.
5. If granted, the orchestrator receives the `capability_id` and uses it as the `capability_token` in the subsequent tool call.
6. The tool adapter receives the call, extracts the `capability_token`, calls `broker.validate_capability(token, expected_scope)`, and proceeds only if the broker confirms.
7. On success, the tool adapter calls `broker.consume_capability(token)` (single-use) and emits a `tool.invoked` event.

## How the taxonomy gaps are closed

The six gap families from the taxonomy, mapped to where they are closed in Castellan:

### Scope / Intent
The agent does not get ambient access to tools. Every action requires a capability whose `scope` declares precisely what is permitted, with structured constraints. The `justification` field captures the intent in human-readable form, audited. **Closed at**: broker request/issue.

### Point / Trajectory
Capabilities are single-use and bound to a specific resource and action. A capability to reserve `BEARING-X42` quantity 1 cannot be reused to reserve a different part or a different quantity. **Closed at**: broker's `consume_capability` and the scope constraints.

### Lifetime
Every capability has `issued_at` and `expires_at`. Default expiry in the PoC is short (60 seconds). Capabilities not consumed in their lifetime are dead. **Closed at**: broker's expiry check on validate.

### Principal
Every capability names a concrete principal. The agent is never the principal. The broker rejects requests where the principal is unknown or where the requesting agent has no claim to be acting on that principal's behalf. **Closed at**: broker request evaluation.

### Provenance
The capability ledger records the requesting agent, the principal, the justification, the issuing broker decision, and the parent capability. The audit log records the surrounding context (the model output that prompted the request, the retrieved evidence). Provenance is reconstructible from the audit log alone. **Closed at**: audit bus and capability ledger.

### Composition
When an action is taken in the context of an outer capability (e.g., drafting a work order is part of the larger ticket-handling flow), the inner capability records its parent. The broker enforces that the inner scope is a subset of the outer scope. **Closed at**: broker's parent-capability check.

## What the broker explicitly refuses

These are the refusal cases the PoC demonstrates on purpose:

- **Unknown principal.** Ticket arrives with an operator ID that isn't in the principal registry.
- **Out-of-role action.** Operator-role tries to request a capability for `workorder.dispatch`. The action doesn't exist; the role wouldn't include it if it did.
- **Scope creep.** Agent requests a capability for "all parts for PUMP-3B" when the procedure names two specific parts. Broker requires the narrowest viable scope.
- **Expired parent.** Inner capability requested after the outer capability has expired.
- **Constraint violation.** Quantity exceeds limit, severity higher than role permits, asset doesn't match the ticket.

Each refusal produces a `capability.decision` event with `decision: denied` and human-readable reasoning. The agent receives the denial and escalates rather than retrying with different parameters.

## What is deliberately simple

- Capability tokens are UUIDs, not signed JWTs. The broker is the only authority. Distributed validation comes later.
- Principal roles are hardcoded. Policy externalization comes later.
- There is no delegation between principals. An operator cannot grant capabilities to another operator. Future scope.
- The broker is a single process. HA comes later.

These simplifications are honest — the PoC is about demonstrating the shape, not the production hardening.
