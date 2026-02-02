# Events & Outbox Pattern (Simplified)

## Purpose
Reliable side-effects (email, notifications, external calls) using Outbox Pattern without over-engineering.
Optimized for: Prisma 6.x, Clean Architecture + Hexagonal, CQRS-lite, <= 10 events.

## Event classification (MANDATORY)
- In-memory events (best-effort): no DB durability, no retry guarantee.
- Outbox events (durable): DB-backed, async processing, retry with backoff.

Rule:
- If losing the event is unacceptable -> Outbox.
- If best-effort is fine -> In-memory.

## Layer responsibilities (STRICT)
Domain (`libs/domain`)
- Event definitions only: EventNames, `*.event.ts` payload types.
- No Prisma, no queues, no handlers.

Application (`libs/application`)
- UseCases orchestrate business flow.
- Emits domain events.
- Writes outbox inside transaction for durable events.
- Allowed: ports + tokens, DomainError.
- Forbidden: PrismaClient, BullMQ, email providers.
- Naming: `*.write-outbox.handler.ts` (handlers insert outbox rows only).

Contracts (`libs/application/contracts`)
- Define ports + tokens (e.g., `OUTBOX_WRITER_PORT`).
- Application injects by token, never adapters.

Infrastructure / Persistence
- Prisma adapters, outbox processor, queue enqueue, workers (email/PDF/etc.).
- Allowed: PrismaClient, BullMQ, external services.

## Transaction rule (NON-NEGOTIABLE)
Business update and outbox insert MUST happen in the same Prisma transaction.

Correct:
```
prisma.$transaction(tx => {
  updateBusiness(tx)
  outboxWriter.write(tx, event)
})
```

Incorrect:
- Insert outbox after commit
- Separate transactions

## Outbox processing (minimal)
Required components:
- OutboxWriter (insert row)
- OutboxProcessor (poll + retry)
- Single worker (<= 10 events)

Processing rules:
- Pick rows where `status = PENDING` and `next_run_at <= now()`.
- On success: mark SENT.
- On failure: increment attempts, set `next_run_at` (backoff).
- Use DB index `(status, next_run_at)`.

## Worker strategy (simple & safe)
PDF + Email handling:
- Do NOT call HTTP preview endpoints from workers.
- Reuse the same PDF renderer/service used by preview API.

Flow:
- Preview API: Query DB -> Render PDF -> Return response
- Worker: Query DB -> Render PDF -> Send email

## Snapshot policy (current decision)
- `finish-scan` locks data.
- Worker renders PDF from DB at processing time.
- No snapshot payload for now; acceptable if finished data is immutable.
- Future option: add JSON snapshot (no binary) without changing flow.

## Folder structure (minimal)
```
libs/
  domain/
    events/event-names.ts
    <feature>/<event>.event.ts

  application/
    eventing/
      in-memory-event-bus.ts
      subscriptions.ts
    features/
      **/*.usecase.ts
      **/*.write-outbox.handler.ts

  application/contracts/
    outbox/
      outbox-writer.port.ts
      outbox.tokens.ts

  infrastructure/
    outbox/
      outbox-writer.adapter.ts
      outbox-processor.service.ts
    messaging/bullmq/
      email.worker.ts
```

## Idempotency (recommended)
When enqueueing jobs: `jobId = outbox_event.id` to prevent duplicate processing.
