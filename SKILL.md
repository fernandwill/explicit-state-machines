---
name: explicit-state-machines
description: Scan the codebase for implicit state machines and make each one explicit.
disable-model-invocation: true
---

An **implicit state machine** is state whose legal values and legal transitions exist only in scattered code: the rules live in `if` chains, not in one named place. Making it **explicit** means one source of truth that names every state and every allowed transition, and code that cannot reach an **illegal state**.

## 1. Scan

Search the whole codebase (skip vendored, generated, and gitignored folders). Signals:

- A `status` / `state` / `phase` / `stage` / `step` field, column, or enum.
- Several booleans that together encode one state (`isLoading` + `isError` + `data`, `isPaid` + `isShipped`), especially combinations that must never be true together.
- Nullable fields whose presence means a state (`approvedAt`, `cancelledAt`, `deletedAt`).
- Guards before writes (`if (order.status !== 'PAID') throw`), repeated in more than one place.
- String literals compared across files instead of one enum.
- Timestamps or flags flipped by cron jobs, webhooks, queues, or retries.
- UI flows: wizards, modals, async request lifecycles, multi-step forms.

Done when every hit is either a candidate or dismissed with a one-line reason.

## 2. Map each candidate

For each candidate, write down from the code (cite `file:line`):

- **States**: every value it can hold, including ones produced only by error paths.
- **Transitions**: every place that changes it, from which state to which, and the trigger.
- **Illegal states and transitions** the current code allows (a race, a missing guard, a flag combination nothing prevents).
- **Persistence**: whether the state is stored (DB column, cache, URL) and who else reads it (API contract, other services, analytics).

Done when every write site of the state is listed; a write you did not find is a transition the explicit version will break.

## 3. Report and ask

Present a table, worst first: candidate, location, states, transition count, write sites, illegal states found, proposed fix, risk. Proposed fix follows the codebase's existing patterns first:

- An existing state machine library or pattern already in the repo.
- Otherwise a typed state enum or discriminated union plus one transition function or table that every write goes through.

Flag any fix that changes a DB column, an API contract, or more than 3 files: those need the user's approval under AGENTS.md "Ask first". Ask the user which candidates to convert. Stop here until they answer.

## 4. Convert

For each approved candidate:

1. Write tests first for the transitions as they behave today, and for each illegal transition the new code must reject. See them go red where today's code allows the illegal path.
2. Add the explicit definition (states + allowed transitions) in one place.
3. Route every write site from step 2 through it. Preserve today's legal behaviour exactly; only illegal paths change.
4. Stored state: keep the stored values unchanged, or follow expand-then-contract for any schema change.

Done when no write site sets the state directly (search for the field again to prove it), and the tests, lint, and typecheck pass.
