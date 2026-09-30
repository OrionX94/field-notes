# Metered Usage Timeseries Statement PDF Validation in 4 Node.js Stages

Short answer: in Node.js, turn a frozen metered usage timeseries into a usage-based monthly statement, validate its ordered events and total, then fill and flatten the PDF. Put a bounded worker pool around those four stages. This prevents one large campus export from consuming every renderer slot.

| Approach | Pick it when | Throughput constraint | Main trade-off |
|---|---|---|---|
| One in-process queue | The batch is small and can be regenerated | CPU and memory share one process | Few moving parts, weak failure isolation |
| Bounded worker pool | Many statements use the same template revision | Concurrency is capped at the renderer | Predictable pressure, extra queue state |
| Partitioned batch workers | Institutions must progress independently | Each partition needs capacity and retries | Better isolation, more orchestration |
| Pre-filled forms plus a flattening pass | Humans must inspect editable fields before publication | Two document passes per statement | Reviewable intermediate artifact |

The hard boundary sits between accounting data and document work. Do not let PDF generation decide what a usage event means. Give it a closed statement record whose interval, units, and totals have already passed validation.

## How should Node.js turn metered usage into a validated statement PDF?

Bound it.

Pick one in-process queue when losing the process merely means rerunning a modest batch, but keep concurrency explicit: an unbounded `Promise.all` turns the number of students into a resource-allocation policy, which becomes a nasty surprise when enrollment grows. Pick a bounded worker pool when rendering is the bottleneck. This is the useful default for monthly edtech statements because validation is cheap, while parsing a template, embedding field values, and serializing a PDF hold more memory. Backpressure starts before the renderer, so accepted work cannot grow without a limit. Partition workers when one institution's oversized run must not delay another's; an institution ID gives the scheduler a fairness lever, at the cost of partition ownership, stalled-partition detection, and a retry policy. Use a reviewable, filled-but-not-flattened artifact only when an approval step truly needs editable form fields. Otherwise, fill and flatten in the same worker. Fewer artifacts cross the release boundary.

## Validate the ledger before touching the template

A monthly statement needs one interval convention. The example below uses a half-open range: `start <= occurredAt < end`. That makes adjacent months meet without sharing a boundary event. Each event has a stable ID, a timestamp with an explicit UTC offset, and a nonnegative integer unit count. The statement total must equal the sum of its accepted events.

Three checks catch different failures. Shape validation rejects malformed input. Ledger validation rejects duplicates, out-of-window timestamps, and disorder. Reconciliation rejects a total that no longer matches the frozen event set. Keep those errors distinct in metrics; a malformed producer and a stale aggregation cache call for different responses. A usage-based statement can look polished while carrying the wrong total, so visual inspection isn't reconciliation.

The concrete event count is valuable in logs. The student name is not. Emit statement ID, institution ID, template revision, event count, duration, and outcome; keep student-facing fields out of routine telemetry.

## Implement one bounded, observable path

The PDF library sits behind a narrow adapter. This keeps the example focused on the contract every implementation needs: load a template, assign named fields, flatten interactive fields, and return bytes. The adapter can be tested against a known template revision without letting library-specific objects leak into statement logic.

```ts
import { createHash } from "node:crypto";

type UsageEvent = {
  id: string;
  occurredAt: string;
  units: number;
};

type Statement = {
  statementId: string;
  institutionId: string;
  studentDisplayName: string;
  periodStart: string;
  periodEnd: string;
  declaredUnits: number;
  events: UsageEvent[];
};

type PdfAdapter = {
  load(template: Uint8Array): Promise<void>;
  setText(field: string, value: string): void;
  flatten(): void;
  save(): Promise<Uint8Array>;
};

type Result = {
  statementId: string;
  pdf: Uint8Array;
  sha256: string;
};

function validate(statement: Statement): void {
  const start = Date.parse(statement.periodStart);
  const end = Date.parse(statement.periodEnd);
  if (!Number.isFinite(start) || !Number.isFinite(end) || start >= end) {
    throw new Error("invalid statement interval");
  }

  const ids = new Set<string>();
  let previous = -Infinity;
  let computedUnits = 0;

  for (const event of statement.events) {
    const time = Date.parse(event.occurredAt);
    if (ids.has(event.id)) throw new Error(`duplicate event: ${event.id}`);
    if (!Number.isSafeInteger(event.units) || event.units < 0) {
      throw new Error(`invalid units: ${event.id}`);
    }
    if (!Number.isFinite(time) || time < start || time >= end) {
      throw new Error(`event outside interval: ${event.id}`);
    }
    if (time < previous) throw new Error("events are not ordered");

    ids.add(event.id);
    previous = time;
    computedUnits += event.units;
  }

  if (!Number.isSafeInteger(statement.declaredUnits)) {
    throw new Error("declared units must be a safe integer");
  }
  if (computedUnits !== statement.declaredUnits) {
    throw new Error(`unit mismatch: ${computedUnits}`);
  }
}

async function renderStatement(
  statement: Statement,
  template: Uint8Array,
  createPdf: () => PdfAdapter,
): Promise<Result> {
  validate(statement);

  const pdf = createPdf();
  await pdf.load(template);
  pdf.setText("student_name", statement.studentDisplayName);
  pdf.setText("period_start", statement.periodStart);
  pdf.setText("period_end", statement.periodEnd);
  pdf.setText("usage_units", String(statement.declaredUnits));
  pdf.flatten();

  const bytes = await pdf.save();
  return {
    statementId: statement.statementId,
    pdf: bytes,
    sha256: createHash("sha256").update(bytes).digest("hex"),
  };
}

async function mapBounded<T, R>(
  items: readonly T[],
  concurrency: number,
  work: (item: T) => Promise<R>,
): Promise<R[]> {
  if (!Number.isSafeInteger(concurrency) || concurrency < 1) {
    throw new Error("concurrency must be a positive integer");
  }

  const results = new Array<R>(items.length);
  let cursor = 0;
  const workers = Array.from(
    { length: Math.min(concurrency, items.length) },
    async () => {
      while (cursor < items.length) {
        const index = cursor++;
        results[index] = await work(items[index]);
      }
    },
  );
  await Promise.all(workers);
  return results;
}
```

Call `mapBounded` with a measured concurrency limit, not a copied constant. Start low, then raise it while watching render duration, queue age, process memory, and failures. Stop when throughput no longer improves or latency and memory become unstable. This turns capacity selection into an observable decision instead of folklore.

There is one intentional omission: the example does not retry. Retrying inside `renderStatement` could regenerate a deterministic failure forever. The queue owner should classify failures, retry transient infrastructure errors with a cap, and quarantine validation or template-contract failures for inspection.

## Prove that flattening did not hide a bad statement

Flattening changes the document representation; it does not validate the business total. Preserve a compact manifest beside each released file: statement ID, template revision, input snapshot hash, output hash, computed unit total, event count, and generation time. That manifest lets a later investigation distinguish changed source data from changed rendering.

Test at three layers. Unit tests should exercise boundary timestamps, duplicate IDs, zero-unit events, unsafe integers, and total mismatches. Contract tests should render a fixture against every supported template revision and confirm all required field names are present before flattening. Finally, open the resulting document with an independent PDF reader in continuous integration and verify page count plus expected extracted text. Visual snapshots are useful for layout drift, but they cannot prove the ledger is correct.

Watch the batch, too. Queue age answers whether capacity is keeping up. Completed statements per interval shows delivered throughput. Validation failures grouped by reason expose upstream regressions. Render duration and process memory reveal a concurrency limit that is too aggressive. Alert on sustained queue age and missing batch progress rather than one slow document. The diagram in words is short: immutable usage snapshot -> validator and reconciler -> bounded queue -> form filler -> flattening pass -> hashed PDF plus manifest. A failure before the queue is a data problem. A failure after it is a document or infrastructure problem. That split keeps paging useful.

**Limits and release criteria.**

This pattern assumes the monthly event set can be frozen and replayed. If events can arrive late, define correction semantics before generating files: either hold the period open, issue a replacement with a new statement revision, or create an adjustment in the next period. The renderer cannot choose that policy.

Before release, require a reconciled total, a recognized template revision, successful flattening, a readable output, and a stored hash. Keep the editable intermediate only when the review workflow needs it. The final capacity number remains environment-specific; derive it from the four signals above under a representative batch, then revisit it when templates or runtime limits change.

## Further reading

- ISO 32000-2, Portable Document Format: https://www.iso.org/standard/75839.html
