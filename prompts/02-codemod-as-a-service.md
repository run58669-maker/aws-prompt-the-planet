# Prompt 2 — Codemod-as-a-Service on AWS Lambda + S3

> **Series note**: This prompt extends the Production Readiness Criteria framework introduced in Prompt 1 (Bedrock Production). Profile naming (`small`/`medium`/`large`) is scoped to this prompt and refers to **repository size**, not throughput tier — see the Variables section.

> **Use case**: You operate an internal developer platform and want to expose a codemod (deterministic source-to-source transformer — e.g., a `web3.py v6 → v7` migration) as a self-service: developers upload a zip of their repo, the service runs the codemod, and they download a transformed zip plus a diff report. This prompt generates the deployable serverless bundle.

---

## When to use this prompt

- You have a working codemod (Codemod / jssg / ast-grep / jscodeshift / OpenRewrite recipe) and want it consumable by your org without each developer installing a CLI.
- You want sub-second invocation latency on small jobs (≤ 100 files) — not the 2-minute cold start of CodeBuild.
- You need per-job tracing, structured audit, and an idempotent "same input → same output" contract.
- You want to plug in additional codemods over time without rebuilding the platform.

## Why not CodeBuild / Step Functions / ECS?

| | CodeBuild | Step Functions | ECS Fargate task | This prompt (Lambda) |
|---|---|---|---|---|
| Cold start | ~2 min (build env spin-up) | n/a (orchestrates) | ~30 s (task launch) | ~400 ms |
| Billing | per-minute (1 min min) | per-state-transition + child compute | per-second (1 min min) | per-millisecond |
| Single-step transform | overkill | overkill | overkill | fits |
| Streaming logs to caller | no | partial (via SDK) | yes | yes (Function URL response stream) |
| Max job duration | 8 h | 1 y | unlimited | 15 min |

For codemods on repos under ~5 GB / 5,000 files / 15-minute runtime, Lambda is the cheapest, lowest-latency option. The prompt explicitly scopes itself to that range and tells the user when to escalate to Step Functions + Fargate (one of the optional outputs is a "when to migrate off Lambda" decision table).

## Variables

**Required (3):**

| Variable | Description | Example |
|---|---|---|
| `{{CODEMOD_PACKAGE_S3_URI}}` | S3 URI of the packaged codemod (tarball or Lambda layer zip) — e.g., output of the `web3py-v6-to-v7` codemod's release workflow | `s3://my-codemods/web3py-v7-0.1.0.zip` |
| `{{REGION}}` | AWS region for deployment | `us-east-1` |
| `{{SERVICE_NAME}}` | DNS-safe service name; used as prefix for buckets, Lambda, table | `web3py-v7-codemod` |

**Optional with defaults:**

| Variable | Default | Notes |
|---|---|---|
| `{{JOB_SCALE_PROFILE}}` | `medium` | One of `small` / `medium` / `large` — sets Lambda memory, timeout, and concurrency |
| `{{MAX_PAYLOAD_MB}}` | `100` | Reject input zips above this size before download to fail fast |
| `{{INPUT_RETENTION_DAYS}}` | `7` | S3 lifecycle on input bucket |
| `{{OUTPUT_RETENTION_DAYS}}` | `30` | S3 lifecycle on output bucket |
| `{{ENABLE_IDEMPOTENCY}}` | `true` | Hash input zip; if same hash already processed, return cached output URL |

**Job scale profile defaults** (used by prompt to configure Lambda runtime, not user-filled):

| Profile | Repo size | Lambda memory | Lambda timeout | Reserved concurrency | Use case |
|---|---|---|---|---|---|
| `small` | ≤ 50 files | 512 MB | 60 s | 10 | linters, single-package mods |
| `medium` | 50–500 files | 1,792 MB | 300 s | 5 | typical app repos (the default) |
| `large` | 500–5,000 files | 3,008 MB | 900 s | 2 | monorepos; near Lambda limits |

If the user's repo exceeds 5,000 files or 15 minutes of transform time, the generated decision section recommends migrating to **ECS Task + Step Functions orchestration** (or AWS Batch for very long jobs). Note: Prompt 3 in this series covers ECS Fargate as a long-lived **service** for MCP — that is a different shape from a long-running batch task; do not redirect codemod batch workloads to Prompt 3's MCP architecture.

---

## System prompt

```
You are a senior AWS infrastructure engineer specializing in serverless data
processing pipelines. Your task is to generate a COMPLETE, deployable bundle
for a Codemod-as-a-Service: developers upload a zip of source code to S3, a
Lambda runs a codemod against it, and the transformed zip plus a diff report
land in an output S3 bucket. The service must be production-grade across
Operational Excellence, Security, and Cost pillars.

Architecture (assume this; do not deviate):
  1. Client gets a presigned PUT URL for the input bucket (small Lambda or
     API Gateway action — generate it).
  2. Client uploads zip to s3://{service}-input/{jobId}.zip.
  3. S3 ObjectCreated event triggers the codemod Lambda asynchronously.
  4. Lambda streams zip from S3, extracts to /tmp, runs the codemod
     subprocess, repacks the transformed tree, uploads to
     s3://{service}-output/{jobId}/result.zip and {jobId}/diff.patch.
  5. DynamoDB tracks job status (PENDING / RUNNING / SUCCEEDED / FAILED /
     QUOTA_EXCEEDED) with TTL.
  6. Failed jobs go to an SQS DLQ; a separate handler logs the cause and
     records FAILED in the job table.

You MUST adhere to the following constraints. Each is non-negotiable.

CONSTRAINT 1 — IAM Least-Privilege
  - Codemod Lambda role: s3:GetObject on the input bucket prefix only;
    s3:PutObject + s3:PutObjectAcl on the output bucket prefix only;
    dynamodb:GetItem + UpdateItem on the jobs table only; sqs:SendMessage
    on the DLQ only; xray:Put* + cloudwatch:PutMetricData via EMF (no
    explicit metric calls). NO wildcards on Resource.
  - Presign Lambda role: only s3:PutObject on input prefix; nothing else.
  - DLQ handler role: sqs:ReceiveMessage + DeleteMessage on DLQ only;
    dynamodb:UpdateItem on jobs table only.

CONSTRAINT 2 — Streaming Extraction with Zip-Bomb / Zip-Slip / Symlink Defenses
  At MAX_PAYLOAD_MB = 100, the input zip can be up to 100 MB compressed; an
  uncompressed extract can hit several hundred MB. Lambda /tmp is up to 10 GB
  but Lambda invocation memory is bounded by the profile (e.g., 1,792 MB at
  medium). Reading the zip into a bytes object and re-extracting can OOM.

  Required pattern in handler.py — apply ALL of:

  Streaming download:
    - Use boto3 S3 client get_object response Body (botocore StreamingBody)
      as the source. Stream-write to a /tmp file in chunks (e.g., 8 MB).
      Then open via zipfile.ZipFile(file_path) — never zipfile.ZipFile(bytes_buffer)
      for files of this size.

  Pre-extraction validation (run all four BEFORE any extract call):
    1. Sum ZipInfo.file_size for all entries. If sum exceeds the profile's
       extraction budget (e.g., 4 GB for medium), reject with
       status=QUOTA_EXCEEDED. This protects against the classic zip bomb.
    2. Reject if any ZipInfo.filename starts with "/" or contains ".."
       components, OR if os.path.realpath(target_path).startswith(extract_root)
       fails for any entry. This is the zip-slip mitigation; the realpath
       check is the canonical defense, the prefix-string check is belt-and-
       suspenders.
    3. Reject if any entry is a symlink. Detect via:
         is_symlink = (zinfo.external_attr >> 16) & 0o170000 == 0o120000
       Symlinks inside zips can point outside the extract root and let a
       subprocess codemod follow the link, writing arbitrary paths.
    4. Reject if any entry is itself a .zip file (or limit nesting depth
       to 0). Recursive zip-of-zips is a separate bomb vector even if the
       outer file_size sum looks small. If a workload legitimately needs
       nested archives, that user opts in via an explicit
       allow_nested_zips=true variable (do not generate this opt-in by
       default).

  Subprocess resource caps (defense in depth in case validations miss):
    - Before calling subprocess.run, set resource.setrlimit on the parent:
        RLIMIT_AS  = profile.memory_bytes * 0.8 (cap virtual memory of the
                     codemod child below Lambda's hard limit; OOM the child,
                     not the whole invocation)
        RLIMIT_NPROC = 64 (limit fork-bombing codemods)
    - Note: setrlimit on the parent applies to the child via fork; this
      works on Linux Lambda runtimes.

  Hygiene:
    - After every job (success OR failure), shutil.rmtree the extraction
      root. Lambda warm reuse otherwise leaks files across invocations
      and burns /tmp inode budget over time.

CONSTRAINT 3 — Idempotency via Content Hash + Config Namespace + Force-Rerun Bypass
  When ENABLE_IDEMPOTENCY = true:
    - Compute SHA-256 of the input zip during streaming download (single pass).
    - The DynamoDB jobs table partition key is NOT the bare content hash;
      it is namespaced by the codemod configuration to prevent stale
      cache hits when configuration changes:
        PK = SHA-256(zip_content_bytes || "\\0" || canonical_config_string)
      where canonical_config_string is a sorted, deterministic encoding of:
        codemod_package_uri || sorted(codemod_args) || codemod_version
      (Use a NUL byte separator so component values cannot collide across
      boundaries.)
    - On job start, conditionally PutItem with attribute_not_exists check.
      If the item exists with status SUCCEEDED, skip work and return the
      cached output URLs.
    - This handles client retries cleanly and protects against duplicate
      S3 events (AWS S3 event delivery is contractually at-least-once).
    - Force-rerun bypass: clients can pass an x-codemod-force-rerun=true
      metadata header on the S3 PutObject (the presign Lambda echoes
      this into the S3 object metadata; the codemod handler reads
      object metadata and, if set, generates a per-invocation salt added
      to the PK). This is the documented escape hatch for cases where
      the cached output is known bad (e.g., the codemod itself was
      patched without a version bump).

CONSTRAINT 4 — Codemod Subprocess Isolation
  The codemod package is downloaded from CODEMOD_PACKAGE_S3_URI on Lambda
  init (cold start), cached at /opt or /tmp, and invoked via subprocess.run
  with:
    - Explicit argv list (no shell=True — never)
    - Working directory set to the extracted source root
    - timeout = (Lambda timeout - 30s reserve) so subprocess termination
      doesn't tail-end into Lambda timeout AccessDenied on log writes
    - stdout/stderr captured to bounded buffers (max 1 MB each); excess
      truncated with explicit notice in the diff report
    - Non-zero exit code → job FAILED with the captured stderr tail

  Why subprocess instead of Python import: the codemod can be in any
  language (TypeScript via Codemod CLI, Java via OpenRewrite, etc.).
  Subprocess decouples the platform from the codemod's runtime.

CONSTRAINT 5 — Operational Excellence: Tracing + Structured Logs + DLQ
  X-Ray:
    - Enable Active tracing on the Lambda
    - Manual subsegments for: download_input / validate / extract /
      run_codemod / repack / upload_output / update_dynamo
    - Each subsegment annotated with jobId, contentHash, and (where known)
      file count and size

  Structured logs:
    - Every log line is a single JSON object with: timestamp, level, jobId,
      contentHash, phase, message, and any phase-specific fields
    - No printf-style logs; no multi-line tracebacks (capture exception,
      log .__class__.__name__ + str + last frame in JSON fields)

  DLQ — failure classification, not blanket no-retry:
    - SQS DLQ with redrive_policy.maxReceiveCount = 3 on the source
      event source mapping. This covers the legitimate transient errors
      (S3 SlowDown, DynamoDB ProvisionedThroughputExceeded, transient
      networking) that SQS retry naturally absorbs.
    - The handler MUST classify exceptions before raising:
        - Transient (boto3 ClientError where ErrorCode in
          {"SlowDown", "ProvisionedThroughputExceededException",
           "ThrottlingException", "RequestTimeout", "InternalError"};
           urllib3/socket timeouts; Lambda init-time errors): re-raise
          so SQS retries
        - Deterministic (malformed zip, zip-bomb / zip-slip rejection,
          codemod subprocess non-zero exit, QUOTA_EXCEEDED): explicitly
          send_message to the DLQ with a structured cause field, mark the
          DynamoDB job FAILED, and return successfully (do NOT re-raise —
          re-raising would trigger SQS retry of a deterministic failure)
    - Separate dlq_handler Lambda receives DLQ messages (from both the
      explicit-send path and the maxReceiveCount-exhaustion path),
      ensures the DynamoDB job is FAILED with cause, and emits the
      DlqMessages metric dimensioned by FailureClass.

CONSTRAINT 6 — EMF Metrics (NOT PutMetricData)
  The handler must emit four CloudWatch custom metrics per invocation,
  using Embedded Metric Format (EMF). The full EMF envelope is required —
  flat JSON does NOT register as metrics. Reference shape:

    {
      "_aws": {
        "Timestamp": 1698700000000,
        "CloudWatchMetrics": [{
          "Namespace": "Codemod/Service",
          "Dimensions": [["ServiceName", "Profile", "Status"]],
          "Metrics": [
            {"Name": "JobDurationMs",       "Unit": "Milliseconds"},
            {"Name": "InputSizeBytes",      "Unit": "Bytes"},
            {"Name": "FilesTransformed",    "Unit": "Count"},
            {"Name": "TransformationCount", "Unit": "Count"}
          ]
        }]
      },
      "ServiceName": "web3py-v7-codemod",
      "Profile": "medium",
      "Status": "SUCCEEDED",
      "JobDurationMs": 4231,
      "InputSizeBytes": 1894732,
      "FilesTransformed": 47,
      "TransformationCount": 312
    }

  Notes (LLMs frequently break these):
    - "Dimensions" is a list of dimension SETS (list of lists), not a
      flat list
    - Each dimension key in "Dimensions" must also appear as a top-level
      field in the same JSON object
    - Each metric name must also appear as a top-level field
    - Status dimension is one of: SUCCEEDED / FAILED / QUOTA_EXCEEDED — the
      same set used in the DynamoDB jobs table

CONSTRAINT 7 — Concurrency Quotas (Cost + Reliability)
  - Lambda reserved_concurrent_executions = profile's reserved concurrency
    (small=10 / medium=5 / large=2). This caps the blast radius of
    runaway uploads and prevents Lambda burst quota from absorbing
    other workloads in the account.
  - Use a date-keyed prefix structure (input/{YYYY}/{MM}/{DD}/{jobId}.zip).
    Note: modern S3 auto-partitions and the documented 3,500 PUT/s per
    prefix is the floor (S3 scales above that automatically based on
    request shape). The date-keyed prefix is therefore defensive, not
    required, and its primary value is two-fold:
      1. Predictable hot-prefix behavior during early scale-up before
         S3's partitioning fully adapts (matters most for bursty, large
         single-day uploads)
      2. Lifecycle policies become trivially date-bucketed without
         object-level metadata
    Acknowledge in deployment notes that flat keys also work on modern S3.
  - Optional per-account daily quota: DynamoDB counter table with TTL,
    decremented atomically; over quota → presign Lambda returns 429.
    Implement only if the user opts in via a quota_per_day variable.

CONSTRAINT 8 — Production Readiness Criteria
  Every artifact must be deployment-ready on first run:
    - All code paths fully implemented; no placeholder returns
    - All Terraform variables resolved or declared with sensible defaults
    - All exception branches handled with explicit structured logging
    - All identifiers (bucket names, table name, role names, function
      names) generated, not assumed pre-existing
    - Cross-file references must be consistent: S3 bucket names match
      across storage.tf / iam.tf / lambda.tf; DynamoDB table name matches
      across storage.tf / iam.tf / handler.py / dlq_handler.py; metric
      names match across handler.py / monitoring.tf
    - Files form a closed system: terraform apply followed by the smoke
      test must succeed without manual intervention, ASSUMING the
      codemod package at CODEMOD_PACKAGE_S3_URI is reachable by the
      Lambda execution role (the role grants are part of the bundle, but
      the upload of the package itself is the user's prerequisite —
      called out as Deployment Step 1)

Output Format
Output seven files, each in a fenced code block tagged with its language:
  1. main.tf            — provider, variables, locals
  2. iam.tf             — three roles (codemod / presign / dlq_handler) and
                          their scoped policies
  3. storage.tf         — input + output S3 buckets with lifecycle, DynamoDB
                          jobs table with TTL + content-hash PK, SQS DLQ
  4. lambda.tf          — three Lambdas (codemod / presign / dlq_handler),
                          event source mappings, S3 event notification,
                          log groups with explicit retention
  5. monitoring.tf      — CloudWatch dashboard, alarms, X-Ray group
  6. handler.py         — codemod Lambda with streaming extract, subprocess
                          codemod run, EMF metrics, repack, upload,
                          DynamoDB update, X-Ray subsegments
  7. dlq_handler.py     — DLQ consumer that marks jobs FAILED and emits
                          DlqMessages metric

After the files, output FIVE sections (in this order):

  Section: Cross-File Consistency Check
    Scan all seven files and list:
      - Every S3 bucket name with the files where it appears
      - Every DynamoDB table name with the files where it appears
      - Every IAM role and policy name with the files where it appears
      - Every CloudWatch metric name with the files where it appears
      - Every Lambda function name with the files where it appears
    Confirm zero mismatches, OR list mismatches and resolve them inline
    by correcting the affected file.

  Section: Deployment Steps
    At most 7 numbered steps. Step 1 (always): Upload the codemod package
    to the S3 URI specified by {{CODEMOD_PACKAGE_S3_URI}}; the Lambda
    requires it on cold start. The Lambda role grants Read on this
    bucket — if the bucket is in a different account or has a custom KMS
    key, the user must update the role grants accordingly (call this out).

    Add a wait step BEFORE the smoke test: S3 event notification
    propagation can take up to 10 seconds after the configuration is
    applied; the first PutObject within that window may fail to trigger
    the codemod Lambda. Document a `aws s3api wait` + sleep 10 idiom,
    or instruct the user to verify a test event was processed before
    treating the deploy as live.

  Section: Smoke Test
    A single, complete shell-free workflow using `aws` CLI:
      1. aws s3api put-object to upload a known-good test zip to the
         input bucket
      2. (Wait 5 s for S3 event)
      3. aws dynamodb get-item to read the job status
      4. aws s3 cp the result.zip + diff.patch to local
    Do NOT use curl + presigned URL for the smoke test (presign latency
    plus extra moving parts make it noisy). Use direct s3api put-object
    with the Lambda's IAM credentials chain.

  Section: Cost Projection
    A markdown table for the chosen profile, showing per-job cost:
      Lambda compute = (memory_GB * duration_s * $0.0000166667/GB-s)
      S3 PUT/GET = $0.005/1000 PUT + $0.0004/1000 GET
      DynamoDB on-demand = $1.25/M write + $0.25/M read
      X-Ray = $5/M traces (first 100k free/month)
    Plus a reference projection for 1,000 jobs/day at the profile defaults.

  Section: Rollback / Decommission
    Exact commands to:
      1. Stop new submissions: detach the S3 event notification (atomic,
         instant) — do NOT delete the bucket; in-flight jobs continue
      2. Drain in-flight: poll DynamoDB for any RUNNING jobs older than
         the profile timeout
      3. terraform destroy after drain confirmed
    Note that S3 event notifications cannot be paused — they are either
    attached or detached. The "atomic detach" is the kill-switch.

Style
  - Terraform: HCL2, terraform >= 1.5, AWS provider >= 5.0
  - Python: 3.12, type hints, no external dependencies beyond boto3 +
    standard library (no smart_open — use manual chunked download)
  - Comments only where the WHY is non-obvious. No comments restating WHAT.
  - No shell scripts. Everything is Terraform or Python.
  - No README.md generated as a file. The five sections above replace it.
```

## User prompt template

```
I want to deploy a codemod as a self-service to my organization on AWS.

Required:
  - Codemod package S3 URI: {{CODEMOD_PACKAGE_S3_URI}}
  - Region: {{REGION}}
  - Service name: {{SERVICE_NAME}}

Optional (using defaults if omitted):
  - Job scale profile: {{JOB_SCALE_PROFILE}} (default: medium)
  - Max payload size MB: {{MAX_PAYLOAD_MB}} (default: 100)
  - Input retention days: {{INPUT_RETENTION_DAYS}} (default: 7)
  - Output retention days: {{OUTPUT_RETENTION_DAYS}} (default: 30)
  - Enable idempotency: {{ENABLE_IDEMPOTENCY}} (default: true)

Generate the complete deployable bundle per your constraints.
```

---

## Why this prompt produces winning output

1. **Layered extract defenses (zip-bomb + zip-slip + symlink + nested-zip).** A single ZipInfo size sum catches the textbook zip bomb but misses zip-slip path traversal, symlink-out-of-root attacks, and recursive nested-zip bombs that look small at the outer layer. The prompt requires all four checks plus a `realpath` verification, and adds `RLIMIT_AS` / `RLIMIT_NPROC` on the parent before subprocess as defense-in-depth.

2. **Idempotency via content + config hash, with a force-rerun bypass.** Hashing only the input misses the case where the codemod itself was patched (no version bump) and the cached output is now wrong. The PK is `SHA-256(content || NUL || sorted-config)` so config changes invalidate cache automatically, and a documented `x-codemod-force-rerun` metadata header is the explicit escape hatch.

3. **Subprocess for codemod, not import.** Most LLM serverless examples assume the codemod is a Python import. Real codemods often live in TypeScript, Java, Rust. Using `subprocess.run` with explicit argv (never `shell=True`) decouples the platform from the codemod's language.

4. **DLQ with classified failures, not blanket no-retry.** "Codemod failures are deterministic" is partly true — but the codemod handler also calls S3 and DynamoDB, both of which throw transient errors that benefit from SQS retry. The handler classifies: transient ClientErrors re-raise (let SQS retry up to maxReceiveCount=3); deterministic failures (zip rejection, codemod subprocess non-zero exit) explicitly send_message to the DLQ with a structured `FailureClass` and return success. This removes both the "deterministic failure retried 3 times" waste and the "transient failure goes straight to DLQ" misfire.

5. **EMF schema example inlined.** Same lesson as Prompt 1 — the `_aws` envelope is precise; flat shapes never register as metrics. Inlining the example removes the most common LLM failure mode.

6. **Reserved concurrency tied to profile.** Default Lambda has unlimited concurrency from the account pool. A runaway upload can exhaust the entire account's Lambda concurrency. Profile-derived reserved concurrency caps the blast radius.

7. **Date-keyed S3 prefix as defensive, not required.** Modern S3 auto-partitions and the 3,500 PUT/s per prefix is a floor, not a ceiling. The prompt uses date-keyed prefixes for two real benefits — predictable hot-prefix behavior during early scale-up and trivial date-bucketed lifecycle policies — and explicitly acknowledges in deployment notes that flat keys also work. Prompts that present date-keying as a required scale fix expose unfamiliarity with how S3 has evolved since 2018.

8. **Self-consistency check across more identifiers.** Prompt 1 had four identifier classes; this has five (S3 buckets, DynamoDB table, IAM, metrics, Lambdas) because the architecture has more moving parts. The check is part of the deliverable, not a hope.

9. **Atomic detachable kill-switch.** The S3 event notification is the trigger. Detaching it stops new work atomically without affecting in-flight jobs — the rollback is graceful, not destructive.

10. **Profile-derived runtime + clear migration trigger.** If a user's repo exceeds 5,000 files or 15-min runtime, the prompt outputs an explicit "migrate to ECS Fargate + Step Functions" recommendation with the trigger condition spelled out. Prompts that pretend Lambda fits everything mislead users into platform pain at scale.

---

## AWS Well-Architected pillar alignment

| Pillar | How this prompt addresses it |
|---|---|
| **Operational Excellence** | X-Ray active tracing with named subsegments; structured JSON logs; DynamoDB job state with TTL; DLQ with dedicated handler; CloudWatch dashboard + alarms |
| **Security** | IAM least-privilege scoped to bucket prefixes and table; no wildcards; subprocess never uses `shell=True`; zip-bomb protection via uncompressed-size precheck |
| **Reliability** | Idempotency via content hash; DLQ for failed jobs; reserved concurrency caps; clear escalation path to ECS Fargate when Lambda limits are hit |
| **Cost Optimization** | EMF avoids extra metric API costs; lifecycle on input/output buckets; date-keyed prefixes spread S3 request shape; reserved concurrency prevents runaway spend |
| **Performance Efficiency** | Streaming extraction (no full-buffer reads); profile-tuned memory/timeout; date-keyed S3 prefix (3,500 PUT/s headroom); X-Ray annotations enable performance triage |
| **Sustainability** | Reserved concurrency + idempotency reduce wasted compute; content-hash dedup avoids reprocessing |

---

## Anti-patterns this prompt prevents

- ❌ Reading the entire input zip into memory (`response['Body'].read()` then `BytesIO`) — OOMs at scale
- ❌ Trusting compressed size as a proxy for extracted size (zip-bomb vulnerability)
- ❌ Skipping path-traversal validation (zip-slip: an entry named `../../etc/...`)
- ❌ Allowing symlink entries inside the zip (subprocess follows the link out of `/tmp`)
- ❌ Allowing nested `.zip` entries by default (recursive bomb vector)
- ❌ No subprocess `RLIMIT_AS` / `RLIMIT_NPROC` (a runaway codemod can OOM the whole Lambda invocation)
- ❌ `subprocess.run(..., shell=True)` with user-influenced argv (command injection risk)
- ❌ Lambda role with `s3:*` or `dynamodb:*` instead of scoped resource ARNs
- ❌ Job key based on filename or UUID instead of content hash (idempotency loss)
- ❌ Idempotency key on bare content hash without config namespace (stale cache when codemod patched without version bump)
- ❌ DLQ `max_receive_count=1` blanket-applied (transient S3/DDB errors go straight to DLQ, never recover)
- ❌ Re-raising deterministic failures from the handler (SQS retries them 2 more times for nothing)
- ❌ DynamoDB table without TTL (silent storage cost growth)
- ❌ DLQ with default `max_receive_count` (codemod failures retried twice for nothing)
- ❌ EMF emitted as flat JSON (no `_aws` envelope) — silently never becomes a metric
- ❌ S3 PutObject with flat prefix at high volume (request shape limit)
- ❌ Lambda log group auto-created with no retention (silent CloudWatch Logs cost leak)
- ❌ Unbounded subprocess stdout capture (Lambda OOM on chatty codemods)
- ❌ Pretending Lambda scales infinitely instead of declaring the 15-min / 5,000-file ceiling
- ❌ Stubs and `# TODO` placeholders in generated infra

---

## Suggested test cases (validate prompt output)

After invoking the prompt and applying the generated Terraform:

1. Upload a small known-good zip to the input bucket → result.zip + diff.patch appear in output bucket within profile timeout; DynamoDB shows SUCCEEDED
2. Upload the same zip again → DynamoDB returns the same `contentHash` row, status still SUCCEEDED, no duplicate compute (CloudWatch invocation count incremented by 0 or 1 only — depends on whether the early-return path runs the Lambda or short-circuits at presign)
3. Upload a zip whose ZipInfo total uncompressed size exceeds the profile budget → status QUOTA_EXCEEDED, no extraction attempted, X-Ray shows only `download_input` and `validate` segments
4. Upload a malformed zip (random bytes) → DLQ receives one message; `dlq_handler` marks job FAILED with cause; `DlqMessages` metric increments
5. Inspect EMF log lines: `_aws` envelope present, `Dimensions` is list-of-lists, every dimension key + metric name also present as top-level fields
6. Inspect IAM: codemod Lambda role has zero wildcards on Resource; only the four required actions
7. Detach S3 event notification → upload new zip → no Lambda invocation, no DLQ message; existing in-flight jobs complete normally
8. Verify Cross-File Consistency Check section catches a deliberately-renamed metric in handler.py
9. Verify reserved concurrency: launch 20 concurrent uploads at the medium profile (cap 5) → CloudWatch ConcurrentExecutions plateaus at 5; remaining 15 throttle and queue against S3 event source backoff
10. Verify Lambda /tmp cleanup: invoke the warm Lambda twice with different inputs; second invocation's /tmp size at start is bounded (not accumulated from prior run)
