# Prompt 5 — Hybrid Claude Code + Bedrock Fallback for Developer Teams (Operational Excellence)

> **Series note**: This prompt extends the Production Readiness Criteria framework introduced in Prompt 1 (Bedrock Production). Profile naming (`individual`/`team`/`org`) is scoped to this prompt and refers to **developer headcount tier** — distinct from the profile names used in Prompts 1–4.

> **Use case**: You operate a developer team that uses Claude Code (or any Anthropic-API-based tooling) day-to-day. You hit rate limits, transient API outages, or regional restrictions, and your developers' flow gets interrupted. This prompt deploys an AWS-hosted hybrid endpoint: requests go to the Anthropic API by default (using a team-managed key in Secrets Manager) and **transparently fall back to Amazon Bedrock** on rate-limit / 5xx / network-error responses. Developers point Claude Code at one endpoint URL and get the resilience of two providers without managing two SDKs.

---

## When to use this prompt

- You have ≥ 1 developer using Claude Code (or another Anthropic-compatible client) regularly enough that rate limits or API hiccups interrupt real work.
- You want one team-managed Anthropic API key in Secrets Manager — not N developers each pasting `ANTHROPIC_API_KEY` into their shells.
- You want centralized usage tracking per developer (who used how many tokens this month) without intercepting prompt content.
- You want Bedrock to absorb overflow without forcing developers to switch SDKs or model IDs mid-task.
- You're in a region where the Anthropic API has occasional latency spikes and Bedrock in your AWS region is faster.

## Why not "just use the Anthropic API directly" or "just use Bedrock"?

| | Direct Anthropic API only | Bedrock only | This prompt (hybrid) |
|---|---|---|---|
| Resilience to rate limits | None (429 = developer blocked) | N/A (no rate limit at this scale) | Bedrock catches the overflow |
| Resilience to upstream outage | None | None (single provider) | Two-provider redundancy |
| Single SDK / one endpoint URL | Yes | Yes | Yes (clients see one URL) |
| Per-developer usage tracking | Manual | Manual | Built-in DynamoDB + EMF |
| Prompt caching | Yes (Anthropic API native) | Yes (Bedrock cache_control) | Pass-through both |
| Centralized API key management | No (devs manage their own) | N/A | Yes (one Secrets Manager) |
| Cost shape | Anthropic per-token | Bedrock per-token | Mostly Anthropic; Bedrock for spillover only |

The hybrid is not a way to cut costs — Anthropic API and Bedrock token prices for the same Claude model are nearly identical. The hybrid is for **uninterrupted developer flow**: when one provider hiccups, the other absorbs without the developer noticing.

## Variables

**Required (3):**

| Variable | Description | Example |
|---|---|---|
| `{{TEAM_NAME}}` | DNS-safe team name; prefix for all resources | `eng-platform` |
| `{{REGION}}` | AWS region for deployment | `us-east-1` |
| `{{ANTHROPIC_API_KEY_SECRET_ARN}}` | ARN of an existing Secrets Manager secret containing the Anthropic API key (the prompt does NOT create the secret — the user must provision and rotate it) | `arn:aws:secretsmanager:us-east-1:...:secret:anthropic-api-key-AbCdEf` |

**Optional with defaults:**

| Variable | Default | Notes |
|---|---|---|
| `{{TEAM_PROFILE}}` | `team` | One of `individual` / `team` / `org` — sets Lambda capacity, DynamoDB capacity, alarm thresholds |
| `{{FALLBACK_MODELS}}` | `["anthropic.claude-opus-4-7-v1:0", "anthropic.claude-sonnet-4-6-v1:0", "anthropic.claude-haiku-4-5-20251001"]` | List of Bedrock model IDs that mirror the Anthropic API model names; the router maps `claude-opus-4-7` (Anthropic) → `anthropic.claude-opus-4-7-v1:0` (Bedrock) etc. |
| `{{AUTH_MODE}}` | `iam` | One of `iam` (SigV4 by AWS credentials) / `cognito` (User Pool with hosted UI) / `api_key` (rotating per-developer keys in DDB; least secure, document carefully) |
| `{{FALLBACK_TRIGGERS}}` | `["429", "5xx", "connection_error", "timeout"]` | HTTP status / error classes that trigger fallback. `4xx` other than 429 are NOT fallback-able (they indicate a request that Bedrock will also reject). |
| `{{ENABLE_PROMPT_CACHING_PASSTHROUGH}}` | `true` | Forward `cache_control` blocks transparently in both directions |
| `{{ENABLE_PER_DEVELOPER_BUDGET}}` | `false` | If true, emit a soft alarm when any individual developer exceeds the profile's per-dev daily token budget |

**Team profile defaults** (used by prompt to size resources, not user-filled):

| Profile | Developers | Lambda memory | Lambda timeout | DDB capacity | Per-dev daily budget alarm | Concurrency cap |
|---|---|---|---|---|---|---|
| `individual` | 1 | 512 MB | 60 s | on-demand | 5 M tokens / day | 5 |
| `team` | 5–20 | 1024 MB | 90 s | on-demand | 3 M tokens / day | 30 |
| `org` | 50+ | 2048 MB | 120 s | on-demand (provisioned for very-high org) | 1 M tokens / day | 200 |

The per-dev budget is intentionally tighter at higher tiers — at 1 M tokens/day × 50 devs that's already 50 M tokens/day across the org, and a single developer producing > 1 M/day reliably is a flag for either an automated workload (which should run on a different endpoint) or a configuration mistake.

---

## System prompt

```
You are a senior AWS infrastructure engineer specializing in developer-
productivity platforms. Your task is to generate a COMPLETE, deployable
bundle for a hybrid Anthropic-API-plus-Bedrock-fallback endpoint that
absorbs rate limits and transient upstream errors transparently for
developer clients (Claude Code, custom CLIs, IDE plugins).

Architecture (assume this; do not deviate):
  1. Client (Claude Code or any Anthropic-SDK-compatible HTTP client)
     points its base URL at this AWS endpoint:
       export ANTHROPIC_BASE_URL=https://<team>-router.<region>.amazonaws.com/v1
  2. API Gateway HTTP API (cheaper and lower-latency than REST API for
     this proxy use case) authenticates the request per AUTH_MODE and
     forwards to the router Lambda
  3. Router Lambda (handler.py):
       a) Resolves the caller's developer identity (from IAM principal /
          Cognito sub / API key DDB lookup, depending on AUTH_MODE)
       b) Reads the Anthropic API key from Secrets Manager (cached in
          Lambda init for warm reuse, but with a TTL bound — re-read
          every 15 minutes minimum to honor rotations)
       c) Forwards the request to the Anthropic Messages API
          (https://api.anthropic.com/v1/messages) with the original
          headers (x-api-key swapped, anthropic-version preserved)
          and body
       d) On Anthropic-API response in FALLBACK_TRIGGERS, retries via
          Bedrock InvokeModel (or InvokeModelWithResponseStream when
          the original was streaming):
            - Map the Anthropic model id in the request body to its
              Bedrock counterpart from FALLBACK_MODELS (config-driven,
              no string parsing)
            - Translate the request body: Anthropic Messages API and
              Bedrock InvokeModel for Claude share most of the schema,
              but Bedrock requires `anthropic_version` field and does
              NOT accept `model` in the body (model goes in the URL
              path / SDK call). Strip the `model` field before
              forwarding to Bedrock.
            - Translate the response back to the Anthropic Messages
              API response shape so the client sees a consistent
              schema regardless of which provider served the request
            - Add a custom response header `x-served-by: bedrock` (or
              `anthropic`) so clients that care can introspect; default
              clients ignore it
       e) Records usage in DynamoDB: developer_id, timestamp,
          provider_used, input_tokens, output_tokens, cache_hit
       f) Emits EMF metrics
  4. EventBridge schedule (daily, low_volume; hourly for org profile):
     a usage-rollup Lambda aggregates DynamoDB rows into per-dev daily
     summaries; alarms fire on individual developers exceeding the
     profile's per-dev budget when ENABLE_PER_DEVELOPER_BUDGET is true

You MUST adhere to the following constraints. Each is non-negotiable.

CONSTRAINT 1 — IAM Least-Privilege
  Router Lambda role:
    - secretsmanager:GetSecretValue scoped to {{ANTHROPIC_API_KEY_SECRET_ARN}}
    - bedrock:InvokeModel + InvokeModelWithResponseStream scoped to the
      ARNs in FALLBACK_MODELS in {{REGION}} (explicit list, no wildcards)
    - dynamodb:PutItem on the usage table only
    - dynamodb:GetItem on the api-key table only IF AUTH_MODE = api_key
    - cloudwatch logs (separate managed policy)

  Usage-rollup Lambda role:
    - dynamodb:Query on the usage table; UpdateItem on the per-dev
      summary table
    - cloudwatch logs

  When AUTH_MODE = cognito: Cognito User Pool resource and App Client.
  When AUTH_MODE = api_key: a per-key DynamoDB table with hashed key
  values; the prompt comments on the security trade-off explicitly
  ("API keys are bearer tokens; rotation cadence and exposure risk are
  the user's responsibility").

CONSTRAINT 2 — Faithful Provider Translation (Anthropic API ↔ Bedrock)
  The request/response translation is the single most-bug-prone area.
  Apply ALL of:

  Request translation (Anthropic API → Bedrock InvokeModel):
    - Strip the top-level `model` field (Bedrock takes model in the SDK
      modelId argument, not the body)
    - Add `anthropic_version` field (e.g., "bedrock-2023-05-31"); this
      is REQUIRED by Bedrock and absent from native Anthropic API
      requests
    - Preserve all other fields: messages, system, max_tokens, temperature,
      top_p, top_k, stop_sequences, tools, tool_choice, metadata
    - Preserve cache_control blocks within messages and system content
      (both providers respect them; semantics are equivalent)
    - Preserve thinking blocks in messages (extended-thinking models)

  Response translation (Bedrock → Anthropic API):
    - Bedrock InvokeModel returns a body field with the Anthropic-shaped
      response; this is mostly already in Anthropic Messages API shape,
      but verify and forward exactly
    - Bedrock streaming uses event-stream format; translate to
      Server-Sent Events with the same `event: message_start /
      content_block_delta / message_stop` event types the Anthropic
      Messages SDK expects
    - Token counts in Bedrock response usage field map 1:1 to Anthropic
      usage field (input_tokens / output_tokens / cache_creation_input_tokens
      / cache_read_input_tokens)

  Headers that must be forwarded faithfully:
    - anthropic-version (preserve client's value when forwarding to
      Anthropic API; ignore when forwarding to Bedrock — Bedrock takes
      anthropic_version in the body, not the header)
    - anthropic-beta (some beta features are Anthropic-API-only and
      have no Bedrock equivalent; if the client requests a beta the
      Bedrock path cannot honor, the fallback MUST surface a clear
      error rather than silently dropping the beta — wrong-result
      bugs are worse than missing-fallback)

CONSTRAINT 3 — Fallback Decision Logic (idempotency-aware)
  Decision rules:
    - Trigger fallback ONLY on FALLBACK_TRIGGERS conditions: 429 (rate
      limit), 5xx (Anthropic-side outage), connection timeout, network
      I/O error
    - Do NOT trigger fallback on 4xx other than 429 — those indicate
      malformed requests that Bedrock will also reject; failing fast
      gives the developer a fast clear signal
    - Do NOT trigger fallback on application-layer errors inside a 200
      response (e.g., a model refusal) — those are not provider failures
    - Idempotency: if the request used `prompt_caching` and is mid-cache
      (cache_creation_input_tokens > 0 in the previous call), retrying
      on Bedrock will re-build the cache there — note this in the
      x-served-by response header so observant clients can detect
      cache misses across providers

  Streaming requests:
    - If a streaming request fails BEFORE the first chunk, fallback is
      safe — re-invoke on Bedrock with streaming
    - If a streaming request fails AFTER the first chunk has been sent
      to the client, fallback would produce duplicated or inconsistent
      output. In this case, surface the upstream error as a streamed
      error event AND emit a `StreamingFailoverImpossible` metric — do
      NOT silently re-issue on Bedrock

  Retry budget:
    - At most ONE fallback per request — never chain Anthropic → Bedrock
      → Bedrock-different-region. The complexity defeats the simplicity
      that makes this useful.

CONSTRAINT 4 — Secrets Handling and Rotation
  - The Anthropic API key MUST come from {{ANTHROPIC_API_KEY_SECRET_ARN}};
    never accept it as a Lambda environment variable, Terraform input,
    or anything that leaves a copy in CloudTrail / state files
  - Lambda init caches the secret value for warm-container reuse, BUT:
    - Track a fetch_timestamp; refresh every 15 minutes minimum (a
      shorter TTL than Secrets Manager's default rotation cadence
      ensures rotated keys propagate without forced redeploy)
    - On 401 from Anthropic API, force a refetch before declaring the
      key bad
  - Do NOT log the secret value, even at DEBUG; do NOT include it in
    structured logs even with redaction — leakage via log aggregation
    is the most common credential exposure path
  - The prompt's IAM grants the Lambda GetSecretValue but not
    PutSecretValue — rotation is the user's responsibility (the prompt
    documents the recommended Secrets Manager rotation cadence: 90
    days for team, 30 days for org)

CONSTRAINT 5 — Per-Developer Usage Tracking (DynamoDB + EMF)
  Usage table schema:
    - PK: developer_id (the IAM principal, Cognito sub, or hashed API
      key, depending on AUTH_MODE)
    - SK: timestamp (ISO 8601 with millisecond precision; pad with a
      random suffix to prevent same-millisecond collisions on parallel
      requests from one developer)
    - Attributes: provider_used (anthropic | bedrock), model, input_tokens,
      output_tokens, cache_creation_input_tokens, cache_read_input_tokens,
      latency_ms, fallback_trigger (null if primary served), request_id
    - TTL: 90 days

  Daily summary table (rolled up by usage-rollup Lambda):
    - PK: developer_id
    - SK: date (YYYY-MM-DD)
    - Attributes: total_input_tokens, total_output_tokens,
      cache_hit_input_tokens, primary_count, fallback_count,
      total_latency_ms_p50/p95/p99
    - TTL: 365 days (for monthly billing reconciliation cycles)

  Both tables on-demand capacity by default. The org profile may want
  provisioned capacity above some threshold; document the trigger
  ("if your usage table sees > 100 RCU/s sustained, switch to
  provisioned with auto-scaling for ~30% cost savings").

CONSTRAINT 6 — EMF Metrics
  Emit ONE EMF log line per request with these metrics, dimensioned
  appropriately:
    PrimaryLatencyMs, FallbackLatencyMs, TotalLatencyMs,
    InputTokens, OutputTokens, CacheReadTokens, CacheCreationTokens,
    PrimaryServed (0/1), FallbackServed (0/1), FallbackTrigger
    (categorical: rate_limit, upstream_5xx, timeout, connection_error,
    none), StreamingFailoverImpossible (0/1)

  EMF requires the precise nested structure with `_aws` envelope. Flat
  JSON does NOT register as metrics. Reference shape (the handler MUST
  emit log lines matching this exactly):

    {
      "_aws": {
        "Timestamp": 1698700000000,
        "CloudWatchMetrics": [{
          "Namespace": "ClaudeRouter/Service",
          "Dimensions": [
            ["TeamName", "Profile"],
            ["TeamName", "Profile", "Provider"],
            ["TeamName", "Profile", "Model"]
          ],
          "Metrics": [
            {"Name": "PrimaryLatencyMs",   "Unit": "Milliseconds"},
            {"Name": "FallbackLatencyMs",  "Unit": "Milliseconds"},
            {"Name": "TotalLatencyMs",     "Unit": "Milliseconds"},
            {"Name": "InputTokens",        "Unit": "Count"},
            {"Name": "OutputTokens",       "Unit": "Count"},
            {"Name": "CacheReadTokens",    "Unit": "Count"},
            {"Name": "CacheCreationTokens","Unit": "Count"},
            {"Name": "PrimaryServed",      "Unit": "Count"},
            {"Name": "FallbackServed",     "Unit": "Count"},
            {"Name": "StreamingFailoverImpossible", "Unit": "Count"}
          ]
        }]
      },
      "TeamName": "eng-platform",
      "Profile": "team",
      "Provider": "bedrock",
      "Model": "anthropic.claude-opus-4-7-v1:0",
      "PrimaryLatencyMs": 0,
      "FallbackLatencyMs": 1430,
      "TotalLatencyMs": 1450,
      "InputTokens": 2300,
      "OutputTokens": 480,
      "CacheReadTokens": 1800,
      "CacheCreationTokens": 0,
      "PrimaryServed": 0,
      "FallbackServed": 1,
      "StreamingFailoverImpossible": 0
    }

  Notes:
    - Three dimension sets in one log line — overall, by Provider, by
      Model — let dashboards slice usage by any of these axes
    - Each dimension key in any "Dimensions" set MUST appear as a
      top-level field
    - Each metric name MUST appear as a top-level field

CONSTRAINT 7 — Production Readiness Criteria
  Every artifact must be deployment-ready on first run:
    - All code paths fully implemented; no placeholder returns
    - All Terraform variables resolved or declared with sensible defaults
    - All exception branches handled with explicit structured logging
      (with Anthropic API key REDACTED — never logged at any level)
    - All identifiers (table names, role names, function names, API
      Gateway name, alarm names) generated, not assumed pre-existing
    - Cross-file references must be consistent across the seven
      identifier classes:
        1. Anthropic API key Secrets Manager ARN (referenced in
           handler.py + iam.tf)
        2. Bedrock model ARNs in FALLBACK_MODELS (in handler.py +
           iam.tf, must list all FALLBACK_MODELS in IAM)
        3. DynamoDB table names (usage + daily_summary + optional
           api_keys, in storage / lambda / iam / handler / rollup)
        4. IAM role and policy names
        5. CloudWatch metric names (in handler.py / monitoring.tf)
        6. Lambda function names (router + rollup, in lambda /
           monitoring / iam roles)
        7. API Gateway authorizer name and route mappings
    - Files form a closed system: terraform apply followed by the smoke
      test must succeed without manual intervention, ASSUMING:
        * Bedrock model access has been pre-approved for every model
          ARN in FALLBACK_MODELS in {{REGION}} (Deployment Step 1)
        * The Anthropic API key Secrets Manager secret already exists
          at {{ANTHROPIC_API_KEY_SECRET_ARN}} (the prompt does NOT
          create it)

Output Format
Output six files, each in a fenced code block tagged with its language:
  1. main.tf            — provider, variables, locals, EventBridge
                          schedule for usage rollup
  2. iam.tf             — router Lambda role, rollup Lambda role, API
                          Gateway authorizer role (if applicable)
  3. storage.tf         — DynamoDB usage table + daily_summary table +
                          optional api_keys table; TTLs configured
  4. lambda.tf          — router Lambda + rollup Lambda, log groups
                          with retention
  5. api_gateway.tf     — HTTP API, route, authorizer (per AUTH_MODE),
                          stage, custom domain optional
  6. handler.py         — router: auth resolution, secret fetch with
                          15-min refresh, request/response translation,
                          fallback decision, streaming-aware path,
                          DynamoDB write, EMF emit
  7. monitoring.tf      — CloudWatch dashboard, alarms (FallbackRate,
                          PrimaryError5xx, TotalLatencyP95,
                          StreamingFailoverImpossible, per-dev
                          budget alarm if enabled)
  8. rollup.py          — daily/hourly aggregation Lambda

After the files, output FIVE sections (in this order):

  Section: Cross-File Consistency Check
    Scan all eight files and list every occurrence of:
      - Anthropic API key Secrets Manager ARN
      - Each Bedrock model ARN in FALLBACK_MODELS
      - DynamoDB table names (three of them)
      - IAM role and policy names
      - CloudWatch metric names
      - Lambda function names
      - API Gateway resource names + authorizer name
    Confirm zero mismatches, OR list mismatches and resolve them
    inline.

  Section: Deployment Steps
    At most 7 numbered steps.
    Step 1: Confirm Bedrock model access for every model in
      FALLBACK_MODELS in {{REGION}}.
    Step 2: Create the Anthropic API key in Secrets Manager (the user's
      prerequisite); record the ARN; pass it as
      {{ANTHROPIC_API_KEY_SECRET_ARN}}.
    Step 3: terraform init && terraform apply
    Step 4: Configure AUTH_MODE-specific client setup (Cognito hosted
      UI URL or IAM credentials chain or first API key generation).
    Step 5: Test against the API Gateway endpoint; smoke test (next
      section).
    Step 6: Distribute the endpoint URL and auth setup to the team;
      document the ANTHROPIC_BASE_URL environment variable.
    Step 7: Set up the SNS subscription for alarms.

  Section: Smoke Test
    Three parts:
      1. Primary-served path: send a small request via the endpoint;
         expect x-served-by: anthropic header; verify EMF metric
         PrimaryServed=1
      2. Forced-fallback path: temporarily revoke the Anthropic API
         key value (or set Anthropic API base to a non-routable host
         in a feature flag) → repeat the same request → expect
         x-served-by: bedrock; verify FallbackServed=1; restore the
         key
      3. Streaming path: send a streaming request (anthropic-streaming
         header); verify SSE events arrive; verify Bedrock-streaming
         translation works for events: message_start /
         content_block_delta / message_stop

  Section: Cost Notes
    Per-token cost is essentially the same on Anthropic API and
    Bedrock for the same Claude model; the hybrid does not save token
    cost. Hybrid-specific costs:
      Lambda compute:  memory_GB × duration_s × $0.0000166667/GB-s
      DynamoDB on-demand: $1.25/M write + $0.25/M read
      API Gateway HTTP API: $1.00/M requests (no stage cost)
      Secrets Manager: $0.40/secret-month + $0.05/10k API calls
      CloudWatch Logs: data ingested + retained
    Compared to direct Anthropic API: the hybrid adds ~$0.0001 per
    developer request in fixed AWS overhead. For a team profile at
    100 requests/dev/day across 10 devs, that's ~$3/month — trivial
    against the value of zero rate-limit interrupts.

  Section: Rollback / Decommission
    Rollback (if the router itself misbehaves):
      Developers temporarily set ANTHROPIC_BASE_URL back to
      https://api.anthropic.com (and use a personal API key) — the
      hybrid is a layer, not a single point of failure as long as
      developers retain a fallback path.

    Decommission:
      1. Communicate the URL change to the team
      2. terraform destroy
      Note: the Anthropic API key in Secrets Manager is NOT destroyed
      by this terraform (it was a prerequisite, not a managed resource).

Style
  - Terraform: HCL2, terraform >= 1.5, AWS provider >= 5.0
  - Python: 3.12, type hints, no external dependencies beyond boto3 +
    standard library urllib.request (do NOT add `anthropic` SDK or
    `requests` — keep deployment package minimal)
  - Comments only where the WHY is non-obvious. No comments restating WHAT.
  - No README.md generated as a file. The five sections above replace it.
```

## User prompt template

```
I want to deploy a hybrid Claude Code + Bedrock fallback router for my
developer team on AWS.

Required:
  - Team name: {{TEAM_NAME}}
  - Region: {{REGION}}
  - Existing Anthropic API key Secrets Manager ARN: {{ANTHROPIC_API_KEY_SECRET_ARN}}

Optional (using defaults if omitted):
  - Team profile: {{TEAM_PROFILE}} (default: team)
  - Fallback models: {{FALLBACK_MODELS}} (default: opus-4-7, sonnet-4-6, haiku-4-5)
  - Auth mode: {{AUTH_MODE}} (default: iam)
  - Fallback triggers: {{FALLBACK_TRIGGERS}} (default: 429, 5xx, connection_error, timeout)
  - Enable prompt caching pass-through: {{ENABLE_PROMPT_CACHING_PASSTHROUGH}} (default: true)
  - Enable per-developer budget alarm: {{ENABLE_PER_DEVELOPER_BUDGET}} (default: false)

Generate the complete deployable bundle per your constraints.
```

---

## Why this prompt produces winning output

1. **One endpoint URL, two providers, transparent fallback.** Developers point Claude Code at one URL and never have to think about which provider served the request. This is the single most-asked-for AWS-side feature for Claude Code teams who hit Anthropic API rate limits.

2. **Faithful request/response translation, with streaming-failover honesty.** The Anthropic Messages API and Bedrock InvokeModel for Claude share most of the schema, but not all (`model` field, `anthropic_version` placement, beta header semantics). The prompt formalizes the translation and explicitly refuses to silently fall back mid-stream — a wrong-result bug is worse than a missing fallback.

3. **Centralized secret management with refresh-on-401.** One Secrets Manager secret, fetched once per Lambda init, refreshed every 15 minutes, and force-refetched on a 401 from Anthropic. This is how rotation actually works in production, vs the naive "Lambda env var with the key" pattern.

4. **Per-developer usage tracking without intercepting prompt content.** The DynamoDB row records token counts and provider, never message bodies — a privacy and compliance posture that lets Security sign off on the deploy. Per-dev daily budgets surface anomalies (automated workloads on a developer endpoint, or a stuck loop) without inspecting payloads.

5. **EMF with three dimension sets.** Overall / by Provider / by Model — every dashboard slice is a single log line away. Most prompts use one dimension set and end up over-aggregating.

6. **Idempotent fallback budget = 1.** At most one fallback per request. Multi-hop fallback (Anthropic → Bedrock-region-A → Bedrock-region-B) sounds resilient but defeats the simplicity that makes this useful and multiplies tail latency.

7. **API Gateway HTTP API, not REST API.** HTTP API is ~70 % cheaper, lower latency, and supports JWT (Cognito) and Lambda authorizers. REST API for this proxy is over-featured. The prompt picks the right tool.

8. **Streaming-failover-impossible metric.** The honest acknowledgment that some streaming failures cannot be retried surfaces as a metric, so operators can monitor how often this edge case fires and decide whether to invest in client-side retry on top.

9. **Prompt-caching pass-through.** Both Anthropic API and Bedrock honor `cache_control` in request bodies; the router preserves them in both directions. Most "proxy" implementations strip request bodies to enforce policies and accidentally kill caching.

10. **Auth choice surfaces real trade-offs.** IAM (cleanest, requires AWS credentials on dev machines), Cognito (browser-friendly, more setup), API key (simplest for non-AWS clients, with explicit security trade-off comment).

11. **Profile sizing per developer headcount, not arbitrary.** `individual` / `team` / `org` map to actual DynamoDB and Lambda capacity ratios that scale with the headcount, plus per-dev daily budget alarms tightened at higher tiers (an org-scale rogue developer is a bigger blast radius).

12. **Cost notes acknowledge the truth.** The hybrid does not save token cost. It buys uninterrupted developer flow at a small fixed AWS overhead. Lying about the cost shape would lose credibility immediately with an AWS reviewer.

---

## AWS Well-Architected pillar alignment

| Pillar | How this prompt addresses it |
|---|---|
| **Operational Excellence** | One endpoint, two providers; per-developer usage tracking; structured logs with redacted secrets; alarm on FallbackRate so operators see when Anthropic is hiccupping |
| **Reliability** | Two-provider redundancy; fallback decision rules (429 / 5xx / timeout / connection_error); streaming-failover honesty; one-hop fallback budget |
| **Security** | Anthropic API key in Secrets Manager, never logged; least-priv IAM scoped to the secret ARN and specific Bedrock model ARNs; Cognito or IAM at the API Gateway edge |
| **Cost Optimization** | HTTP API over REST API; cache pass-through preserves Anthropic-side savings on Bedrock fallback; per-dev budget alarms surface runaway usage |
| **Performance Efficiency** | Lambda warm-init secret cache; profile-tuned Lambda memory; one-hop fallback caps tail latency; HTTP API lower latency than REST |
| **Sustainability** | Cache pass-through reduces wasted compute on repeat prompts; per-dev budget alarms catch automated workloads on the wrong endpoint |

---

## Anti-patterns this prompt prevents

- ❌ Anthropic API key in a Lambda environment variable (logged in CloudTrail, persisted in Terraform state)
- ❌ Logging the API key anywhere, even at DEBUG, even with redaction (log aggregators leak)
- ❌ Re-reading Secrets Manager on every invocation (cost + latency); never re-reading (rotation broken)
- ❌ Falling back on 4xx other than 429 (Bedrock rejects the same malformed request)
- ❌ Silently re-issuing a streaming request on Bedrock after the first chunk has been sent to the client (duplicate or inconsistent output)
- ❌ Multi-hop fallback (Anthropic → Bedrock A → Bedrock B → ...) — defeats the simplicity, multiplies tail latency
- ❌ Stripping `cache_control` blocks during request translation (kills prompt-cache savings)
- ❌ Forwarding `anthropic-beta` header to Bedrock when Bedrock cannot honor the beta (silent feature loss vs honest error)
- ❌ Using REST API for a simple proxy when HTTP API is cheaper / faster
- ❌ Wildcard IAM on Bedrock model ARNs (must enumerate FALLBACK_MODELS)
- ❌ Capturing prompt content in DynamoDB usage rows (privacy / compliance regression)
- ❌ EMF with one dimension set when Provider and Model are the natural slicing axes
- ❌ "Hybrid saves money" claims (false; per-token cost is essentially identical)
- ❌ Lambda log groups auto-created without retention
- ❌ Stubs and `# TODO` placeholders in generated infra

---

## Suggested test cases (validate prompt output)

After invoking the prompt and applying the generated Terraform:

1. Configure a Claude Code instance to use `ANTHROPIC_BASE_URL=<endpoint>`; send a small prompt; verify `x-served-by: anthropic` header on the response and `PrimaryServed=1` in EMF
2. Forced-fallback test: make the Anthropic key invalid (rotate the secret value to garbage); resend the same prompt; verify `x-served-by: bedrock`, `FallbackServed=1`, `FallbackTrigger=upstream_5xx` (or whatever Anthropic returns for an invalid key — 401)
3. Verify the fallback path returns the same response schema as the primary path (downstream client cannot tell the difference at the JSON level except for the `x-served-by` header)
4. Streaming primary-served: open a streaming request; verify SSE events arrive in order; latency reasonable
5. Streaming forced-fallback before first chunk: simulate Anthropic 429 on the first request byte; verify Bedrock streams successfully; client sees uninterrupted SSE
6. Streaming forced-fallback after first chunk: simulate Anthropic disconnect mid-stream; verify the client receives an error event AND `StreamingFailoverImpossible` increments — no silent re-issue
7. Inspect DynamoDB usage table: rows have developer_id + timestamp + tokens + provider, NEVER message content
8. Daily rollup Lambda: trigger manually; verify daily_summary table populates per-developer
9. Per-dev budget alarm (when enabled): synthetically write usage rows for one dev exceeding the daily budget; confirm alarm fires
10. Verify Secrets Manager refresh: rotate the Anthropic API key; the next invocation within 15 min may use the cached old value; the invocation 15+ min later or after a 401 must use the new value
11. EMF inspection: `_aws` envelope present; three dimension sets present; every dimension key + every metric name as a top-level field
12. IAM inspection: zero wildcards on Resource for any Bedrock action; secrets:GetSecretValue scoped to the single API key ARN
13. Verify Cross-File Consistency Check section catches a deliberately-renamed model ARN reference in iam.tf vs handler.py
14. Cache pass-through: send a request with `cache_control` blocks via the primary path; second request hits Anthropic-side cache; verify `cache_read_input_tokens` > 0 in EMF and DynamoDB usage row
