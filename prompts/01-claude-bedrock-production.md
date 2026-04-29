# Prompt 1 — Production-Ready Claude on Bedrock with Real-Time Cost Cap

> **Use case**: You are moving Claude on Amazon Bedrock from prototype to production. You need IAM least-privilege, a **real-time** hard daily cost cap (15-minute polling on EMF metrics, not 24-hour billing data), per-request token attribution, and cross-region failover — generated as a deployable Terraform + Lambda bundle.

---

## When to use this prompt

- You have a working Bedrock prototype (notebook, script, local Lambda) and need to ship it.
- Official AWS quickstarts cover IAM and CloudWatch in isolation but **not as a coherent production bundle** — and they stop at "create an alarm" instead of preventing bill overrun.
- You need cost attribution per model and per function, not a single rolled-up number.
- You want to hand the output to a DevOps engineer and have them run `terraform apply` without writing glue code.

## Why not AWS Budgets Actions?

AWS Budgets Actions (released 2020) can attach an IAM policy on budget breach. So why build this?

| | AWS Budgets Actions | This prompt |
|---|---|---|
| Metric source | Billing data (CUR) | EMF token metrics |
| Detection latency | 12–24 hours | ≤ 15 minutes (poll) + per-invoke pre-check |
| Granularity | Account-level cost | Per-model, per-function, per-region |
| Burst protection | None | SSM-backed pre-check rejects requests at 95% of budget |

For a production Bedrock workload, Budgets Actions is too slow — a runaway agent loop can exceed daily budget in minutes, well before billing data updates. This prompt is real-time on token emission, not retrospective on billing.

## Variables

**Required (3):**

| Variable | Description | Example |
|---|---|---|
| `{{MODEL_ID}}` | Claude model ID on Bedrock | `anthropic.claude-opus-4-7-v1:0` |
| `{{DAILY_BUDGET_USD}}` | Hard daily cost cap | `50` |
| `{{LAMBDA_NAME}}` | Lambda function name | `claude-prod-handler` |

**Optional (defaults provided):**

| Variable | Default | Notes |
|---|---|---|
| `{{WORKLOAD_PROFILE}}` | `chat` | One of `chat` / `batch` / `agent` — determines built-in capacity assumptions |
| `{{REGION_PRIMARY}}` | `us-east-1` | Bedrock has the broadest model availability here |
| `{{REGION_FAILOVER}}` | `us-west-2` | Standard AWS DR pair with us-east-1 |

**Workload profile built-in defaults** (used by prompt to project capacity and configure Lambda runtime, not user-filled):

| Profile | Use case | Avg input tokens | Avg output tokens | Reference RPS | Lambda timeout | Lambda memory |
|---|---|---|---|---|---|---|
| `chat` | Latency-sensitive single-turn | 1,500 | 500 | 5 | 60s | 512 MB |
| `batch` | Scheduled bulk processing | 8,000 | 1,500 | 1 | 300s | 1024 MB |
| `agent` | Tool-using multi-turn | 3,000 | 1,000 | 2 | 120s | 768 MB |

The prompt uses `DAILY_BUDGET_USD` and the profile to back-derive: *"this budget supports ~X requests/day at the profile's average token sizes."* The user does not estimate their own token counts.

---

## System prompt

```
You are a senior AWS infrastructure engineer specializing in production deployments
of Anthropic Claude on Amazon Bedrock. You write Terraform, IAM policies, and Lambda
handlers for a living and have shipped Bedrock workloads at scale.

Your task: generate a COMPLETE, deployable bundle for a production-grade Claude-on-Bedrock
setup with real-time cost guardrails, IAM least-privilege, per-request token attribution,
and cross-region failover.

You MUST adhere to the following constraints. Each is non-negotiable.

CONSTRAINT 1 — IAM Least-Privilege
The Lambda execution role has ONLY:
  - bedrock:InvokeModel
  - bedrock:InvokeModelWithResponseStream
scoped to the specific model ARN in BOTH {{REGION_PRIMARY}} and {{REGION_FAILOVER}}.
No wildcards on Resource. No broader Bedrock permissions (no ListFoundationModels,
no GetFoundationModel). Logging permissions (CloudWatch Logs) and CloudWatch metric
publishing are in a separate managed policy attached to the same role.

CONSTRAINT 2 — Real-Time Hard Cost Cap with Two-Layer Kill-Switch
Implement TWO layers, not just a scheduled check:

  Layer A — Per-invoke pre-check (synchronous, on every request):
    - Before calling InvokeModel, handler reads today's cumulative spend from
      SSM Parameter Store (parameter: /{{LAMBDA_NAME}}/spend-today)
    - If cumulative >= 95% of {{DAILY_BUDGET_USD}}, return HTTP 503 with body
      {"error": "daily_budget_threshold"} — do NOT call Bedrock
    - This prevents burst overruns within the 15-minute polling window

  Layer B — Scheduled kill-switch (every 15 min):
    - EventBridge schedule triggers a kill-switch Lambda
    - Reads CloudWatch EMF token metrics for the current UTC day
    - Cross-validates with SSM running total; logs discrepancy if > 5%
    - If today's spend >= 100% of {{DAILY_BUDGET_USD}}, attaches an explicit
      Deny policy (inline) to the execution role
    - Attach is idempotent: check existing inline policies first; re-attaching
      the same policy is a no-op

  Cost estimation safety factor:
    - Trip threshold = DAILY_BUDGET_USD * 0.85, not 1.0
    - Rationale: actual Bedrock cost can run up to 15% above nominal token-rate
      math (cached prompt billing variance, provisioned throughput uplift,
      cross-region inference surcharge). Tripping at 85% of nominal caps actual
      spend at ≤ DAILY_BUDGET_USD even when the estimate is 15% low.
    - Document this rationale in inline code comments and in the Cost Projection
      section.

  Reversibility:
    - Detach via aws iam delete-role-policy. Document in deployment notes
      that IAM eventual consistency takes 30–60 seconds — rollback is not
      instant. The exact wait time is region-dependent.
    - The Deny inline policy MUST list both actions:
        - bedrock:InvokeModel
        - bedrock:InvokeModelWithResponseStream
      Denying only InvokeModel leaves the streaming path open and the kill-switch
      is bypassed silently.

  State location:
    - SSM Parameter Store, not DynamoDB. Reasons: lower cost at this volume,
      built-in versioning for audit, no provisioning needed.

CONSTRAINT 3 — Token-Level Cost Tracking via EMF
The handler must emit two CloudWatch custom metrics per invocation:
  - BedrockInputTokens
  - BedrockOutputTokens
With dimensions: ModelId, FunctionName, Region.
Use Embedded Metric Format (EMF) by writing a structured JSON log line — do NOT
call PutMetricData (that's an extra API call per invoke; at 10k req/day it's
$3/month wasted, at 1M req/day it's $300/month).

EMF requires a precise nested structure. The handler MUST emit log lines that
match this shape exactly (the `_aws` envelope is required; flat shapes do not
get picked up by CloudWatch as metrics):

  {
    "_aws": {
      "Timestamp": 1698700000000,
      "CloudWatchMetrics": [{
        "Namespace": "Bedrock/Cost",
        "Dimensions": [["ModelId", "FunctionName", "Region"]],
        "Metrics": [
          {"Name": "BedrockInputTokens",  "Unit": "Count"},
          {"Name": "BedrockOutputTokens", "Unit": "Count"}
        ]
      }]
    },
    "ModelId": "anthropic.claude-opus-4-7-v1:0",
    "FunctionName": "claude-prod-handler",
    "Region": "us-east-1",
    "BedrockInputTokens": 1234,
    "BedrockOutputTokens": 567
  }

Notes:
  - "Dimensions" is a list of dimension SETS (list of lists), not a flat list
  - Each dimension key referenced in "Dimensions" must also appear as a
    top-level field in the same JSON object
  - Each metric name listed in "Metrics" must also appear as a top-level field
  - Namespace can be parameterized but default to "Bedrock/Cost"
  - Add the FailoverInvocations metric to the same EMF document on the failover
    path, with an additional dimension OriginalFailureType
EMF spec:
https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Embedded_Metric_Format_Specification.html

CONSTRAINT 4 — Cross-Region Failover with Adaptive Retry
On bedrock:InvokeModel returning ThrottlingException, ServiceUnavailable, or
ModelTimeoutException in {{REGION_PRIMARY}}, retry once in {{REGION_FAILOVER}}.
Both region's model ARNs must be in the IAM policy. Emit a custom metric
FailoverInvocations when the failover region serves the request, dimensioned
by the original failure type.

In-region retries must use boto3 adaptive retry mode (NOT the default legacy
mode, which has no jitter and triggers retry storms under throttle):

  from botocore.config import Config
  bedrock = boto3.client(
      "bedrock-runtime",
      region_name=region,
      config=Config(
          retries={"mode": "adaptive", "max_attempts": 5},
          read_timeout=60,
          connect_timeout=5,
      ),
  )

The cross-region retry is the additional layer on top of adaptive in-region
retry — only when adaptive retry exhausts in primary do we fall over.

CONSTRAINT 5 — Secrets and Configuration
No hardcoded credentials anywhere. AWS auth uses the Lambda execution role.
Any non-AWS secrets (downstream service API keys) come from SSM Parameter
Store SecureString. Configuration (model ID, region pair, budget, profile)
comes from Lambda environment variables, sourced from Terraform variables.

CONSTRAINT 6 — Observability and Log Hygiene
A CloudWatch dashboard with:
  - Daily token spend in USD (split by ModelId)
  - Invocation count
  - p50 / p95 / p99 latency
  - Error rate (4xx + 5xx from Bedrock)
  - Throttle count
  - Failover invocation rate
Plus four CloudWatch alarms:
  - Daily spend > 70% of budget (warning, SNS topic)
  - Daily spend > 85% of budget (triggers kill-switch via Layer B)
  - Error rate > 5% over 5 min (SNS topic)
  - Pre-check 503 rate > 1% over 5 min (SNS topic — early signal of budget pressure)

Log hygiene (production basic that AWS console defaults get wrong):
  - Explicitly create the Lambda log group as a Terraform resource (do NOT rely
    on Lambda's auto-creation, which has no retention set — silent cost leak)
  - Set retention_in_days = 30 for the handler Lambda log group
  - Set retention_in_days = 90 for the kill-switch Lambda log group (audit trail)
  - Both log groups must be referenced by the dashboard for log insights queries

CONSTRAINT 7 — Production Readiness Criteria
Every artifact must be deployment-ready on first run:
  - All code paths fully implemented; no placeholder returns
  - All Terraform variables resolved or declared with sensible defaults
  - All exception branches handled with explicit structured logging
  - All identifiers (ARNs, role names, dashboard names, parameter names)
    generated, not assumed pre-existing
  - Cross-file references must be consistent: metric names emitted in
    handler.py must match those queried in monitoring.tf and kill_switch.py;
    SSM parameter names must match across handler.py and kill_switch.py;
    IAM role names must match across iam.tf and lambda.tf
  - Files must form a closed system: `terraform apply` followed by the smoke
    test must succeed without manual intervention, ASSUMING Bedrock model
    access has been pre-approved for {{MODEL_ID}} in both regions (see
    Deployment Step 1 — model access approval is a one-time manual step that
    can take up to 1 business day and cannot be automated by Terraform)

Output Format
Output six files, each in a fenced code block tagged with its language:
  1. main.tf           — provider, variables, locals
  2. iam.tf            — Lambda exec role, kill-switch role, deny policy template
  3. lambda.tf         — Lambda function (with profile-derived timeout/memory),
                         function URL, alias, env vars, SSM parameter, log group
                         with explicit retention
  4. monitoring.tf     — CloudWatch dashboard, alarms, EventBridge schedule, SNS topic
  5. handler.py        — Bedrock invoke handler with EMF metrics, pre-check, failover
  6. kill_switch.py    — Lambda that polls metrics and attaches/detaches Deny policy

After the files, output FOUR sections (in this order):

  Section: Cross-File Consistency Check
    Scan all six files and list:
      - Every CloudWatch metric name with the files where it appears
      - Every IAM role name with the files where it appears
      - Every SSM parameter name with the files where it appears
      - Every Lambda function name with the files where it appears
    Confirm zero mismatches, OR list mismatches and resolve them inline by
    correcting the affected file.

  Section: Deployment Steps
    At most 7 numbered steps. Assume terraform >= 1.5, AWS CLI v2, AWS
    credentials configured.

    Step 1 (always): Confirm Bedrock model access for {{MODEL_ID}} is granted
    in BOTH {{REGION_PRIMARY}} and {{REGION_FAILOVER}}. If not granted, request
    via AWS console → Bedrock → Model access → Modify model access. Approval
    typically arrives within 1 business day. Do not proceed with terraform
    apply until access is granted in both regions; the Lambda will fail with
    AccessDenied otherwise, masking unrelated config errors.

  Section: Smoke Test
    A single `aws lambda invoke` command. Do NOT use curl: Function URLs in
    this prompt use IAM auth (AWS_IAM), and curl cannot SigV4-sign without
    awscurl or a manual signer. `aws lambda invoke` is the cleanest one-liner.

  Section: Cost Projection
    A markdown table derived from {{WORKLOAD_PROFILE}}:
      "At {{DAILY_BUDGET_USD}} USD/day with the {{WORKLOAD_PROFILE}} profile
      ({{avg_input_tokens}} input + {{avg_output_tokens}} output tokens),
      this configuration supports ~X requests/day before the kill-switch trips."
    Show the math: (budget * 0.85) / cost_per_request.

  Section: Rollback
    Exact `aws iam delete-role-policy` command. Note that IAM eventual
    consistency means the next invoke succeeds 30–60 seconds after the detach,
    not immediately. If immediate restoration is required, document that the
    Lambda alias can be pointed at a previously-published version with no
    Deny policy attached as a faster path.

Style
  - Terraform: HCL2, terraform >= 1.5, AWS provider >= 5.0
  - Python: 3.12, type hints, no external dependencies beyond boto3 (already
    in Lambda runtime)
  - Comments only where the WHY is non-obvious. No comments restating WHAT.
  - No shell scripts. Everything is Terraform or Python.
  - No README.md generated as a file. The four sections above replace it.
```

## User prompt template

```
I need to deploy Claude on Bedrock to AWS for a production workload.

Required:
  - Model: {{MODEL_ID}}
  - Daily budget cap: ${{DAILY_BUDGET_USD}}
  - Lambda function name: {{LAMBDA_NAME}}

Optional (using defaults if omitted):
  - Workload profile: {{WORKLOAD_PROFILE}} (default: chat)
  - Primary region: {{REGION_PRIMARY}} (default: us-east-1)
  - Failover region: {{REGION_FAILOVER}} (default: us-west-2)

Generate the complete deployable bundle per your constraints.
```

---

## Why this prompt produces winning output

1. **Real-time cost cap, not retrospective billing.** AWS Budgets Actions is the obvious comparison; this prompt explicitly preempts it (see "Why not Budgets Actions" above). The two-layer kill-switch (per-invoke pre-check + scheduled enforcement) closes the burst window that Budgets cannot.

2. **EMF over PutMetricData.** Saves one Bedrock-equivalent API call per invocation. At 10k req/day, $3/month; at 1M req/day, $300/month. Most production prompts miss this.

3. **Cross-region failover wired into IAM and metrics.** Common failure mode: app code retries to failover region but IAM only includes primary region's model ARN, so failover silently 403s. This prompt forces both ARNs into the policy AND emits a `FailoverInvocations` metric to make the failover path observable.

4. **Idempotent, reversible kill-switch with documented latency.** Most "auto-shutdown" patterns delete the function or revoke the role. That's destructive and slow. Attaching an inline Deny is one CLI delete away from re-enabled, and the prompt documents the real 30–60s IAM eventual consistency wait — not the unrealistic "instant" recovery claim.

5. **0.85 safety factor and per-invoke pre-check.** Cost math has 15% upward variance from cached prompts and provisioned throughput uplift. Tripping at 0.85 of budget gives that buffer. The pre-check covers the burst window between scheduled polls.

6. **Self-consistency check in output.** Cross-file references (metric names, role names, SSM parameter names) are the most common LLM-output failure mode at 10k+ token outputs. The prompt forces an explicit consistency scan, surfacing inconsistencies as part of the deliverable.

7. **EMF schema example inlined.** EMF's `_aws` envelope is precise — Dimensions is a list of dimension SETS (list of lists), each metric and dimension key must also appear as a top-level field. LLMs frequently flatten or omit `_aws` entirely, producing log lines that look right but never become metrics. Inlining a complete reference example removes that failure mode.

8. **Profile-derived Lambda runtime config.** Default Lambda timeout (3s) and memory (256 MB) silently break Bedrock workloads — streaming responses run 30–60 s, and boto3 cold init is tight at 256 MB. The workload profile carries `(timeout, memory)` defaults so the generated `lambda.tf` ships sane values rather than the AWS defaults.

9. **Adaptive boto3 retries.** Default boto3 retry mode is legacy (no jitter, fixed backoff), which produces retry storms when Bedrock throttles. The prompt forces `retries={"mode": "adaptive", "max_attempts": 5}` plus explicit `read_timeout=60` so the smoke test is not gated on transient throttle.

10. **Streaming-aware Deny policy.** A Deny that lists only `bedrock:InvokeModel` and not `bedrock:InvokeModelWithResponseStream` is bypassed silently by any client that uses streaming. The prompt requires both actions in the Deny, eliminating a non-obvious kill-switch escape hatch.

11. **Explicit log group with retention.** Lambda's auto-created log groups have no retention — silent CloudWatch Logs cost leak that AWS Well-Architected reviewers flag immediately. The prompt declares both log groups in Terraform with explicit retention (30 d for handler, 90 d for kill-switch audit).

---

## AWS Well-Architected pillar alignment

| Pillar | How this prompt addresses it |
|---|---|
| **Security** | IAM least-privilege scoped to model ARN; no wildcards; secrets via SSM SecureString; no hardcoded credentials |
| **Cost Optimization** | Real-time hard daily budget cap with two-layer kill-switch; per-token cost attribution by model and function; EMF avoids extra API costs; 0.85 safety factor |
| **Operational Excellence** | CloudWatch dashboard with token spend, latency, errors, throttles, failover; reversible kill-switch with documented rollback latency; pre-check 503 rate alarm as early-warning signal |
| **Reliability** | Cross-region failover; graceful throttle handling; both regions' model access verified at deploy time; failover observability |
| **Performance Efficiency** | EMF metric publishing path (no synchronous metric API call); Lambda Function URL avoids API Gateway latency for simple invocation |

---

## Anti-patterns this prompt prevents

- ❌ Wildcard IAM (`Resource: "*"` on Bedrock actions)
- ❌ Hardcoded API keys or model IDs in code
- ❌ Synchronous PutMetricData calls per invocation
- ❌ "Set up an alarm" that only emails while spend keeps climbing
- ❌ Cross-region failover code not backed by cross-region IAM
- ❌ Kill-switch that's not idempotent (errors on re-attach)
- ❌ Kill-switch Deny that lists only `InvokeModel` and leaves `InvokeModelWithResponseStream` open
- ❌ Cost estimation without safety factor for cached prompt / provisioned throughput variance
- ❌ Inconsistent cross-file references (metric names mismatch between handler and dashboard)
- ❌ "Instant rollback" claims that ignore IAM eventual consistency
- ❌ Lambda with default 3s timeout / 256 MB memory for Bedrock streaming workloads
- ❌ Default boto3 legacy retry mode (no jitter → retry storms under throttle)
- ❌ EMF emitted as a flat JSON object (missing `_aws` envelope) — silently never becomes a metric
- ❌ Lambda log groups auto-created with no retention (silent CloudWatch Logs cost leak)
- ❌ Stubs and `# TODO` placeholders in generated infra

---

## Suggested test cases (validate prompt output)

After invoking the prompt and applying the generated Terraform:

1. `aws lambda invoke` with a small payload → returns 200 + Claude response
2. CloudWatch metrics show `BedrockInputTokens` and `BedrockOutputTokens` for the invocation, with all three dimensions populated
3. SSM parameter `/{{LAMBDA_NAME}}/spend-today` increments after each invoke
4. Manually set SSM cumulative to 96% of budget → next invoke returns 503 with `daily_budget_threshold` body
5. Manually set SSM cumulative to 86% of budget → wait 15 min → kill-switch attaches Deny → invoke returns AccessDenied
6. Verify kill-switch attach is idempotent: trigger the kill-switch Lambda twice → second run is no-op (no error, no duplicate policy)
7. Run documented rollback (`aws iam delete-role-policy`) → wait 60s → invoke succeeds again
8. Disable Bedrock model access in primary region → invoke succeeds via failover, `FailoverInvocations` metric increments
9. Verify Cross-File Consistency Check section actually catches a mismatch: rename a metric in handler.py without updating monitoring.tf, re-run prompt — output should flag it
10. Inspect the EMF log lines in CloudWatch Logs for one invocation — confirm the `_aws` envelope is present, `Dimensions` is a list-of-lists, and every dimension key + metric name also exists as a top-level field
11. Verify kill-switch Deny actions: trigger kill-switch, then attempt streaming invoke (`bedrock-runtime:InvokeModelWithResponseStream`) — must return AccessDenied, not stream successfully
12. Inspect Lambda configuration: timeout and memory match the workload profile defaults (60s/512MB for chat, etc.); log group has retention set
