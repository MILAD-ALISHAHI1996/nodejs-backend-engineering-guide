# Building a Production-Ready PDF Generation Pipeline in Node.js with Puppeteer, BullMQ, and Redis

## 1. The Real Problem

In many ERP, CRM, finance, warehouse, and reporting systems, users eventually need downloadable reports.

Usually, these reports come in formats like:

* Invoice PDF
* Pre-invoice PDF
* Customer order report
* Financial report
* Inventory report
* Production report

At first, generating a PDF looks simple:

```text
Receive filters from the user
        ↓
Fetch data from the database
        ↓
Render HTML
        ↓
Convert HTML to PDF
        ↓
Return the file to the user
```

A simple implementation may look like this:

```ts
const result =
  await reportRepository.getCustomerOrderItems(
    criteria
  );

const html =
  htmlTemplate.render(result);

const pdfResult =
  await pdfGenerator.generate({
    html,
    filePrefix: "customer-order-items",
  });
```

This works perfectly in development.

The problem starts when the same code runs in production with real data, real users, and real report sizes.

A user may request a report without any filter. Instead of 100 rows, the system may need to process 10,000 or 20,000 rows.

At that point, PDF generation is no longer a simple file export.

It becomes a heavy pipeline involving:

1. A database query
2. Large JavaScript objects in memory
3. A large HTML string
4. Chromium parsing and rendering
5. CSS layout calculation
6. PDF generation
7. File storage

The first lesson was clear:

> PDF generation is not only an I/O operation. It consumes database resources, memory, CPU, and Chromium runtime resources.

---

## 2. Why Generating PDF Directly Inside the API Is Dangerous

A naive implementation could be placed directly inside an API endpoint:

```ts
router.get(
  "/reports/customer-order-items/pdf",
  async (req, res) => {
    const criteria = req.query;

    const result =
      await reportRepository.getCustomerOrderItems(
        criteria
      );

    const html =
      htmlTemplate.render(result);

    const pdfResult =
      await pdfGenerator.generate({
        html,
        filePrefix: "customer-order-items",
      });

    return res.download(pdfResult.path);
  }
);
```

This code is simple and readable, but it has a serious architectural issue.

The entire PDF generation process runs inside the request-response lifecycle.

That means:

```text
User sends request
        ↓
API keeps connection open
        ↓
Database query runs
        ↓
HTML is generated
        ↓
Chromium starts
        ↓
PDF is generated
        ↓
File is returned
```

If the PDF takes 5 seconds, the request stays open for 5 seconds.

If it takes 30 seconds, the request stays open for 30 seconds.

If Chromium crashes, consumes too much memory, or hangs, the API process is directly affected.

In a real system, this is risky.

Imagine this situation:

```text
User A → PDF Report → Chromium
User B → PDF Report → Chromium
User C → PDF Report → Chromium
User D → Login Request
User E → Dashboard Request
```

Now, normal requests like login or dashboard loading may also become slow because the API server is busy generating PDFs.

That is why PDF generation should not be tightly coupled to the main API request lifecycle.

---

## 3. Moving PDF Generation to a Queue-Based Architecture

To solve this, we moved PDF generation to a background worker.

The API no longer generates the PDF directly.

Instead, it only creates a job in a queue.

The final architecture became:

```text
Client
  │
  │  Request PDF generation
  ▼
API
  │
  │  Add job to queue
  ▼
BullMQ Queue
  │
  │  Store job in Redis
  ▼
Redis
  │
  │  Worker reads job
  ▼
PDF Worker
  │
  │  Fetch data + render HTML + generate PDF
  ▼
Puppeteer
  │
  ▼
PDF File
```

With this approach, the API can respond quickly:

```json
{
  "jobId": "12345",
  "status": "queued"
}
```

The actual PDF generation happens in a separate process.

This gives us a better separation of responsibilities:

```text
API:
- Validate request
- Add job to queue
- Return jobId

Queue:
- Store jobs
- Manage job states
- Handle retries
- Track completed and failed jobs

Worker:
- Process job
- Query database
- Render HTML
- Run Puppeteer
- Generate PDF
```

The important point is this:

> A queue does not make PDF generation cheaper. It only moves the heavy work away from the API process.

We still need limits, validation, monitoring, and resource control.

---

## 4. Creating the BullMQ Queue

We used BullMQ with Redis as the queue backend.

```ts
// src/modules/reports/infrastructure/queue/pdf.queue.ts

import { Queue } from "bullmq";

export const pdfQueue = new Queue("pdf", {
  connection: {
    host: process.env.REDIS_HOST ?? "127.0.0.1",
    port: Number(process.env.REDIS_PORT ?? 6379),
  },
});
```

This creates a queue named:

```ts
"pdf"
```

The name is very important.

The worker must listen to the exact same queue name:

```ts
new Worker("pdf", handler, options);
```

If the queue name and worker name do not match, jobs will be added successfully, but no worker will process them.

The Redis connection is configured through environment variables:

```ts
host: process.env.REDIS_HOST ?? "127.0.0.1",
port: Number(process.env.REDIS_PORT ?? 6379),
```

In local development, Redis may run on `127.0.0.1`.

In Docker or production, the host is usually the Redis service name:

```env
REDIS_HOST=redis
REDIS_PORT=6379
```

At this point, this file only creates the queue.

It does not process jobs.

Processing is the responsibility of the worker.

---

## 5. Adding a PDF Job to the Queue

To avoid coupling the application layer directly to BullMQ, we used a queue adapter.

```ts
// src/modules/reports/infrastructure/queue/BullPdfQueue.ts

import { Queue } from "bullmq";

import { PdfQueuePort } from "../../application/ports/PdfQueuePort";

export class BullPdfQueue implements PdfQueuePort {
  constructor(private readonly queue: Queue) {}

  async addCustomerOrderItemsReport(
    payload: Parameters<
      PdfQueuePort["addCustomerOrderItemsReport"]
    >[0]
  ): Promise<{ jobId: string }> {
    const job = await this.queue.add(
      "customer-order-items-pdf",
      payload,
      {
        attempts: 3,
        removeOnComplete: false,
        removeOnFail: false,
      }
    );

    return {
      jobId: String(job.id),
    };
  }
}
```

The important part is:

```ts
this.queue.add(
  "customer-order-items-pdf",
  payload,
  options
);
```

The first argument is the job name:

```ts
"customer-order-items-pdf"
```

This allows one queue to support multiple PDF job types:

```text
customer-order-items-pdf
invoice-pdf
pre-invoice-pdf
warehouse-report-pdf
```

The worker can then decide which handler to run based on `job.name`.

The second argument is the payload.

A good payload should contain only the information required to generate the report.

For example:

```ts
{
  criteria: {
    customerId: 120,
    fromDate: "2026-01-01",
    toDate: "2026-01-31"
  }
}
```

We should not store the full report data inside Redis.

This is a bad idea:

```ts
await queue.add("pdf", {
  rows: hugeDataArray
});
```

This is better:

```ts
await queue.add("pdf", {
  criteria
});
```

The worker can fetch the data from the database later.

This keeps Redis lighter and avoids storing large datasets in the queue.

---

## 6. Understanding the Risk of Retry Attempts

In the first version, the job options included:

```ts
attempts: 3
```

This means that if a job fails, BullMQ will retry it up to three times.

This is useful for temporary failures such as:

* Temporary database disconnection
* Temporary storage failure
* Temporary Redis issue
* Network instability

But it can be dangerous for memory-related failures.

Assume a user requests a huge PDF without filters.

The worker loads too much data, Chromium consumes too much memory, and the job fails.

Retrying the same job will not solve the problem.

It will repeat the same expensive operation:

```text
Attempt 1 → Huge data → High RAM → Fail
Attempt 2 → Huge data → High RAM → Fail
Attempt 3 → Huge data → High RAM → Fail
```

So the server gets hit three times by the same heavy job.

For heavy PDF generation jobs, this is often safer:

```ts
const job = await this.queue.add(
  "customer-order-items-pdf",
  payload,
  {
    attempts: 1,

    removeOnComplete: {
      age: 60 * 60,
      count: 100,
    },

    removeOnFail: {
      age: 24 * 60 * 60,
      count: 500,
    },
  }
);
```

This configuration avoids unnecessary retries for deterministic failures.

It also prevents Redis from growing forever by keeping only a limited number of completed and failed jobs.

---

## 7. Running the PDF Worker as a Separate Process

The worker is started from a separate entry point.

```ts
// src/workers/pdf.worker.runner.ts

import "reflect-metadata";
import "../config/env";

import { sourceDataSource } from "../config/database";

import { PdfWorker } from "../modules/reports/infrastructure/queue/workers/PdfWorker";
import { ReportTypeOrmRepository } from "../modules/reports/infrastructure/persistence/typeorm/ReportTypeOrmRepository";
import { CustomerOrderItemsHtmlTemplate } from "../modules/reports/infrastructure/pdf/CustomerOrderItemsHtmlTemplate";
import { PuppeteerPdfGenerator } from "../modules/reports/infrastructure/pdf/PuppeteerPdfGenerator";

async function bootstrap(): Promise<void> {
  console.log("SOURCE_DB_USER:", process.env.SOURCE_DB_USER);

  if (!sourceDataSource.isInitialized) {
    await sourceDataSource.initialize();
  }

  const reportRepository =
    new ReportTypeOrmRepository(sourceDataSource);

  new PdfWorker(
    reportRepository,
    new CustomerOrderItemsHtmlTemplate(),
    new PuppeteerPdfGenerator()
  );

  console.log("PDF worker is running...");
}

bootstrap().catch((error) => {
  console.error("Failed to start PDF worker:", error);
  process.exit(1);
});
```

This file shows that the worker is a separate process.

The API and worker can be started independently:

```bash
npm run start:api
npm run start:pdf-worker
```

In Docker, they can run as separate services:

```text
backend-api
pdf-worker
redis
```

This separation is important.

If the PDF worker consumes too much CPU or memory, the API process can remain alive.

This gives the system better operational stability.

---

## 8. Why the Worker Must Initialize Its Own Database Connection

Inside the worker bootstrap, we have:

```ts
if (!sourceDataSource.isInitialized) {
  await sourceDataSource.initialize();
}
```

This is necessary because the worker runs in a separate Node.js process.

We cannot assume that because the API is connected to the database, the worker is also connected.

Each process has its own memory, lifecycle, and database connections.

The worker must do its own initialization:

```text
Load environment variables
        ↓
Initialize database connection
        ↓
Create repository
        ↓
Create PDF worker
        ↓
Start consuming jobs
```

Without this step, the worker may receive a job and then fail because the database connection is not initialized.

This is a common mistake when moving logic from an API process into a background worker.

---

## 9. Building the PDF Worker Class

The worker class receives three dependencies:

```ts
export class PdfWorker {
  private readonly worker: Worker;

  constructor(
    private readonly reportRepository: IReportRepository,
    private readonly htmlTemplate: CustomerOrderItemsHtmlTemplate,
    private readonly pdfGenerator: PuppeteerPdfGenerator
  ) {
    this.worker = new Worker(
      "pdf",
      this.handleJob.bind(this),
      {
        connection: {
          host: process.env.REDIS_HOST ?? "127.0.0.1",
          port: Number(process.env.REDIS_PORT ?? 6379),
        },

        concurrency: 1,

        lockDuration: 5 * 60 * 1000,
        stalledInterval: 30 * 1000,
        maxStalledCount: 1,
      }
    );

    this.registerEvents();
  }
}
```

The responsibilities are separated clearly:

```text
reportRepository:
Fetches report data from the database.

htmlTemplate:
Transforms report data into HTML.

pdfGenerator:
Converts HTML into PDF using Puppeteer.
```

This design is better than putting everything inside one large function.

If we later change the HTML template, the repository and worker do not need to change.

If we replace Puppeteer with another PDF engine, only the PDF generator implementation changes.

This follows a clean architecture approach where infrastructure details are hidden behind interfaces and adapters.

---

## 10. Why `concurrency: 1` Matters

The worker configuration includes:

```ts
concurrency: 1
```

This means the worker processes only one PDF job at a time.

At first, it may look better to increase this value:

```ts
concurrency: 2
concurrency: 4
concurrency: 8
```

But PDF generation with Puppeteer is expensive.

Each job may create a Chromium page or even a Chromium instance.

Multiple jobs can quickly increase memory usage:

```text
Job 1 → Chromium → RAM
Job 2 → Chromium → RAM
Job 3 → Chromium → RAM
Job 4 → Chromium → RAM
```

On a small or medium server, this can easily cause memory pressure or process crashes.

For PDF workers, starting with `concurrency: 1` is safer.

After measuring real memory usage and execution time, the value can be tuned.

The rule is:

> Do not increase concurrency before measuring memory and CPU usage under real report sizes.

---

## 11. Routing Jobs by Job Name

When BullMQ passes a job to the worker, we route it based on `job.name`.

```ts
private async handleJob(
  job: Job<CustomerOrderItemsPdfJob>
) {
  console.log("PDF job started:", {
    id: job.id,
    name: job.name,
  });

  switch (job.name) {
    case "customer-order-items-pdf":
      return this.handleCustomerOrderItemsPdf(job);

    default:
      throw new Error(
        `Unknown PDF job: ${job.name}`
      );
  }
}
```

This makes the worker extensible.

Today, it supports one job type:

```text
customer-order-items-pdf
```

Later, we can add more:

```ts
switch (job.name) {
  case "customer-order-items-pdf":
    return this.handleCustomerOrderItemsPdf(job);

  case "invoice-pdf":
    return this.handleInvoicePdf(job);

  case "pre-invoice-pdf":
    return this.handlePreInvoicePdf(job);

  default:
    throw new Error(`Unknown PDF job: ${job.name}`);
}
```

The queue remains the same, but the worker can handle multiple report types.

This is useful when all PDF jobs share the same infrastructure but have different templates and data sources.

---

## 12. The Main PDF Handler

The core handler looks like this:

```ts
private async handleCustomerOrderItemsPdf(
  job: Job<CustomerOrderItemsPdfJob>
) {
  const { criteria } = job.data;

  const result =
    await this.reportRepository.getCustomerOrderItems(
      criteria
    );

  const html =
    this.htmlTemplate.render(result);

  const pdfResult =
    await this.pdfGenerator.generate({
      html,
      filePrefix: "customer-order-items",
    });

  return {
    success: true,
    ...pdfResult,
  };
}
```

This method looks simple, but each line has a cost.

First:

```ts
const { criteria } = job.data;
```

The worker reads the report filters from the job payload.

Then:

```ts
await this.reportRepository.getCustomerOrderItems(criteria);
```

The worker loads data from the database.

If the criteria are not controlled, this can load thousands of records into Node.js memory.

Then:

```ts
this.htmlTemplate.render(result);
```

The data is transformed into an HTML string.

This means the data now exists in memory in at least two forms:

```text
JavaScript objects
HTML string
```

Then:

```ts
this.pdfGenerator.generate(...)
```

The HTML is sent into Chromium.

Chromium parses it, builds the DOM, calculates layout, loads fonts, renders pages, and exports a PDF.

So the real pipeline is:

```text
Database rows
    ↓
JavaScript objects
    ↓
HTML string
    ↓
Chromium DOM
    ↓
PDF pages
    ↓
PDF file
```

This is why report size matters so much.

---

## 13. The Dangerous Case: No Filters

The most dangerous input is an empty criteria object:

```ts
criteria = {};
```

This usually means the user did not apply any filter.

The repository may then generate a broad query:

```sql
SELECT *
FROM CustomerOrderItems;
```

Or in a query builder:

```ts
const query =
  repository.createQueryBuilder("item");
```

Without `WHERE`, without date range, and without customer limitation, the result can become huge:

```text
10,000 rows
20,000 rows
50,000 rows
```

For a normal API response, this is already bad.

For PDF generation, it is much worse.

The data must go through all these stages:

```text
Database result
JavaScript objects
HTML string
Chromium DOM
PDF file
```

A report with thousands of rows can easily produce hundreds of PDF pages.

The first production rule should be:

> Never allow PDF export without meaningful filters.

---

## 14. Validating Criteria Before Query Execution

The best place to reject invalid PDF requests is before executing the database query.

```ts
private validateCriteria(
  criteria: CustomerOrderItemsPdfJob["criteria"]
) {
  if (!criteria || Object.keys(criteria).length === 0) {
    throw new Error(
      "At least one filter is required for PDF export."
    );
  }

  const hasValidFilter =
    Object.entries(criteria).some(
      ([key, value]) => {
        if (key === "limit") return false;
        if (key === "offset") return false;
        if (value === undefined) return false;
        if (value === null) return false;
        if (value === "") return false;

        return true;
      }
    );

  if (!hasValidFilter) {
    throw new Error(
      "PDF export without filter is not allowed."
    );
  }
}
```

Here, `limit` and `offset` are not considered meaningful filters.

This request is limited, but not filtered:

```json
{
  "limit": 1000,
  "offset": 0
}
```

A meaningful filter should be something like:

```json
{
  "customerId": 120
}
```

Or:

```json
{
  "fromDate": "2026-01-01",
  "toDate": "2026-01-31"
}
```

Or:

```json
{
  "preInvoiceId": 19184
}
```

This distinction matters.

A limit only controls quantity.

A business filter controls scope.

For PDF reports, both are important.

---

## 15. Enforcing a Maximum Row Limit

Even if filters exist, the result can still be too large.

For example, a customer may have 20,000 order items in a large date range.

So we need a maximum row limit:

```ts
private readonly MAX_ROWS_FOR_PDF = 1000;
```

Then we create a safe criteria object:

```ts
const safeCriteria = {
  ...criteria,
  limit: Math.min(
    Number((criteria as any).limit ?? this.MAX_ROWS_FOR_PDF),
    this.MAX_ROWS_FOR_PDF
  ),
};
```

This means that even if the user sends:

```json
{
  "limit": 100000
}
```

The system converts it to:

```json
{
  "limit": 1000
}
```

This is a defensive programming technique.

Never trust the client to send a safe limit.

The backend must enforce the maximum.

After loading data, we can also check the row count:

```ts
if (rows > this.MAX_ROWS_FOR_PDF) {
  throw new Error(
    `PDF row limit exceeded. Max allowed rows: ${this.MAX_ROWS_FOR_PDF}`
  );
}
```

This second check protects the system if the repository ignores the limit or returns a paginated object incorrectly.

---

## 16. Detecting the Number of Rows

Depending on repository design, the result may be returned as an array:

```ts
return rows;
```

Or as a paginated object:

```ts
return {
  items,
  total,
  page,
  limit
};
```

To support both formats, we can detect rows like this:

```ts
const rows = Array.isArray(result)
  ? result.length
  : Array.isArray((result as any)?.items)
    ? (result as any).items.length
    : 0;
```

This allows the worker to log and validate the result size regardless of the repository response shape.

Then we log it:

```ts
console.log("PDF data loaded:", {
  jobId: job.id,
  rows,
  memory: this.getMemoryUsage(),
});
```

This log helps answer an important production question:

> Did the worker fail because the query returned too much data?

Without row count logs, debugging PDF worker failures becomes guesswork.

---

## 17. Why Row Count Is Not Enough

A row limit is necessary, but it is not enough.

Not all rows have the same size.

For example:

```text
500 short rows
```

may produce a small HTML document.

But:

```text
500 rows with long descriptions, addresses, notes, and many columns
```

may produce a much larger HTML document.

So we also need to measure the HTML size.

```ts
const htmlSizeMB =
  Buffer.byteLength(html, "utf8") / 1024 / 1024;
```

Then we can enforce a limit:

```ts
private readonly MAX_HTML_SIZE_MB = 20;
```

And validate it:

```ts
if (htmlSizeMB > this.MAX_HTML_SIZE_MB) {
  throw new Error(
    `PDF HTML size is too large. Max allowed size: ${this.MAX_HTML_SIZE_MB}MB`
  );
}
```

This protects Chromium.

Large HTML is expensive because Chromium must:

```text
Parse HTML
Build DOM
Apply CSS
Calculate layout
Render pages
Generate PDF
```

So HTML size is often a better risk signal than row count alone.

---

## 18. Memory Profiling Inside the Worker

To debug PDF generation properly, we need memory logs.

Node.js provides:

```ts
process.memoryUsage();
```

We wrapped it like this:

```ts
private getMemoryUsage() {
  const memory = process.memoryUsage();

  return {
    rssMB: Math.round(memory.rss / 1024 / 1024),
    heapUsedMB: Math.round(memory.heapUsed / 1024 / 1024),
    heapTotalMB: Math.round(memory.heapTotal / 1024 / 1024),
    externalMB: Math.round(memory.external / 1024 / 1024),
  };
}
```

These values mean different things:

```text
heapUsed:
Memory used by JavaScript objects.

heapTotal:
Memory reserved by V8 for the JavaScript heap.

rss:
Total memory used by the process from the operating system.

external:
Memory used outside the V8 heap, such as Buffers.
```

For Puppeteer workloads, `rss` is especially important.

Why?

Because Chromium-related memory and native allocations may increase RSS even when `heapUsed` does not look very high.

If we only monitor `heapUsed`, we may miss the real memory pressure.

A better production log includes both data size and memory:

```ts
console.log("PDF html generated:", {
  jobId: job.id,
  htmlSizeMB: Number(htmlSizeMB.toFixed(2)),
  memory: this.getMemoryUsage(),
});
```

---

## 19. Stage-by-Stage Logging

The worker should log each major stage.

At job start:

```ts
console.log("PDF job started:", {
  id: job.id,
  name: job.name,
  attemptsMade: job.attemptsMade,
  memory: this.getMemoryUsage(),
});
```

After loading data:

```ts
console.log("PDF data loaded:", {
  jobId: job.id,
  rows,
  memory: this.getMemoryUsage(),
});
```

After rendering HTML:

```ts
console.log("PDF html generated:", {
  jobId: job.id,
  htmlSizeMB: Number(htmlSizeMB.toFixed(2)),
  memory: this.getMemoryUsage(),
});
```

After generating the PDF:

```ts
console.log("PDF file generated:", {
  jobId: job.id,
  pdfResult,
  memory: this.getMemoryUsage(),
});
```

These logs help identify where the bottleneck happens:

```text
Memory increases after query:
The dataset is too large.

Memory increases after HTML render:
The template produces too much HTML.

Memory increases during PDF generation:
Chromium is the expensive part.
```

Without this kind of logging, debugging PDF issues in production becomes very difficult.

---

## 20. Handling Worker Events

BullMQ provides important worker events.

```ts
this.worker.on("completed", ...);
this.worker.on("failed", ...);
this.worker.on("error", ...);
this.worker.on("stalled", ...);
```

For completed jobs:

```ts
this.worker.on("completed", (job, result) => {
  console.log("PDF job completed:", {
    id: job.id,
    result,
  });
});
```

For failed jobs:

```ts
this.worker.on("failed", (job, error) => {
  console.error("PDF job failed:", {
    id: job?.id,
    name: job?.name,
    attemptsMade: job?.attemptsMade,
    error: error.message,
    stack: error.stack,
    memory: this.getMemoryUsage(),
  });
});
```

This gives us important production information:

```text
Which job failed?
What was the job type?
How many attempts were made?
What was the error?
What was memory usage at failure time?
```

For worker-level errors:

```ts
this.worker.on("error", (error) => {
  console.error("PDF worker error:", {
    error: error.message,
    stack: error.stack,
    memory: this.getMemoryUsage(),
  });
});
```

This event is different from a job failure.

A job failure means a job could not be completed.

A worker error may indicate a broader issue with the worker process, Redis connection, or BullMQ internals.

---

## 21. Understanding Stalled Jobs

BullMQ can mark a job as stalled.

A stalled job means the worker picked up the job but failed to renew its lock in time.

Common causes include:

* Worker process crashed
* Event loop was blocked
* CPU was under heavy pressure
* Process was killed
* Job took too long
* Server ran out of memory

The configuration:

```ts
lockDuration: 5 * 60 * 1000,
stalledInterval: 30 * 1000,
maxStalledCount: 1,
```

`lockDuration` defines how long the worker owns the job lock.

`stalledInterval` defines how often BullMQ checks for stalled jobs.

`maxStalledCount` defines how many times a job may be recovered from a stalled state.

For heavy PDF generation, this is important:

```ts
maxStalledCount: 1
```

If a job stalls because it is too heavy, running it repeatedly may make the situation worse.

The worker logs stalled jobs like this:

```ts
this.worker.on("stalled", (jobId) => {
  console.warn("PDF job stalled:", {
    jobId,
    memory: this.getMemoryUsage(),
  });
});
```

This helps us detect jobs that may not fail normally but still break worker execution.

---

## 22. A More Production-Safe Worker

After adding validation, limits, memory logs, and better error handling, the worker becomes safer.

```ts
export class PdfWorker {
  private readonly worker: Worker;

  private readonly MAX_ROWS_FOR_PDF = 1000;
  private readonly MAX_HTML_SIZE_MB = 20;

  constructor(
    private readonly reportRepository: IReportRepository,
    private readonly htmlTemplate: CustomerOrderItemsHtmlTemplate,
    private readonly pdfGenerator: PuppeteerPdfGenerator
  ) {
    this.worker = new Worker(
      "pdf",
      this.handleJob.bind(this),
      {
        connection: {
          host: process.env.REDIS_HOST ?? "127.0.0.1",
          port: Number(process.env.REDIS_PORT ?? 6379),
        },

        concurrency: 1,
        lockDuration: 5 * 60 * 1000,
        stalledInterval: 30 * 1000,
        maxStalledCount: 1,
      }
    );

    this.registerEvents();
  }

  private async handleJob(
    job: Job<CustomerOrderItemsPdfJob>
  ) {
    console.log("PDF job started:", {
      id: job.id,
      name: job.name,
      attemptsMade: job.attemptsMade,
      memory: this.getMemoryUsage(),
    });

    switch (job.name) {
      case "customer-order-items-pdf":
        return this.handleCustomerOrderItemsPdf(job);

      default:
        throw new Error(`Unknown PDF job: ${job.name}`);
    }
  }

  private async handleCustomerOrderItemsPdf(
    job: Job<CustomerOrderItemsPdfJob>
  ) {
    const { criteria } = job.data;

    this.validateCriteria(criteria);

    const safeCriteria = {
      ...criteria,
      limit: Math.min(
        Number((criteria as any).limit ?? this.MAX_ROWS_FOR_PDF),
        this.MAX_ROWS_FOR_PDF
      ),
    };

    const result =
      await this.reportRepository.getCustomerOrderItems(
        safeCriteria
      );

    const rows = Array.isArray(result)
      ? result.length
      : Array.isArray((result as any)?.items)
        ? (result as any).items.length
        : 0;

    console.log("PDF data loaded:", {
      jobId: job.id,
      rows,
      memory: this.getMemoryUsage(),
    });

    if (rows > this.MAX_ROWS_FOR_PDF) {
      throw new Error(
        `PDF row limit exceeded. Max allowed rows: ${this.MAX_ROWS_FOR_PDF}`
      );
    }

    const html = this.htmlTemplate.render(result);

    const htmlSizeMB =
      Buffer.byteLength(html, "utf8") / 1024 / 1024;

    console.log("PDF html generated:", {
      jobId: job.id,
      htmlSizeMB: Number(htmlSizeMB.toFixed(2)),
      memory: this.getMemoryUsage(),
    });

    if (htmlSizeMB > this.MAX_HTML_SIZE_MB) {
      throw new Error(
        `PDF HTML size is too large. Max allowed size: ${this.MAX_HTML_SIZE_MB}MB`
      );
    }

    const pdfResult =
      await this.pdfGenerator.generate({
        html,
        filePrefix: "customer-order-items",
      });

    console.log("PDF file generated:", {
      jobId: job.id,
      pdfResult,
      memory: this.getMemoryUsage(),
    });

    return {
      success: true,
      rows,
      htmlSizeMB: Number(htmlSizeMB.toFixed(2)),
      ...pdfResult,
    };
  }
}
```

This worker does more than generate PDFs.

It protects the system.

It prevents unfiltered PDF exports, limits rows, measures HTML size, logs memory, and makes production debugging much easier.

---

## 23. When PDF Is the Wrong Output Format

PDF is excellent for presentation and printing.

It is not always the right format for large datasets.

If the user wants to export 20,000 rows, PDF is usually a bad choice.

In that case, Excel or CSV is better.

A practical rule:

```text
Up to 1,000 rows:
PDF is usually acceptable.

1,000 to 5,000 rows:
Depends on the template and server capacity.

More than 5,000 rows:
Excel or CSV is usually a better option.
```

The reason is simple:

```text
PDF:
Designed for reading, printing, and sharing.

Excel:
Designed for filtering, sorting, analyzing, and processing data.
```

If a report is too large to read comfortably, it probably should not be a PDF.

This is not only a technical decision.

It is also a product decision.

The system should guide users toward the right export format based on data size.

---

## 24. Final Production Lessons

The main lesson from this implementation was that PDF generation is not just about choosing a library.

Puppeteer is powerful, but production PDF generation needs an entire pipeline.

A stable PDF generation pipeline should include:

```text
Validation
Queue
Worker
Repository
HTML Template
Puppeteer
Storage
Monitoring
Limits
Fallback Export Format
```

Using Puppeteer directly inside an API endpoint may work for small reports.

But in production, problems appear when:

* Reports become large
* Multiple users export PDFs at the same time
* Users request reports without filters
* Chromium consumes too much memory
* Jobs are retried unnecessarily
* Worker logs are not detailed enough

The most important lesson is this:

> A queue moves heavy work away from the API, but it does not reduce the cost of that work.

We still need to control the size of the data, the size of the HTML, the memory usage of the worker, and the behavior of failed jobs.

In the end, building a reliable PDF generator in Node.js is not only a Puppeteer task.

It is a backend architecture problem.

It requires thinking about:

```text
Database load
Memory usage
CPU usage
Redis job storage
Worker lifecycle
Chromium behavior
User experience
Export format strategy
```

Puppeteer gives us high-quality PDF output.

BullMQ gives us background processing.

Redis gives us job persistence.

But production stability comes from the limits, monitoring, and architectural decisions around them.
