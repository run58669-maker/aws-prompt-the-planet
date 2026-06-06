# Anthropic Claude on AWS — Production Prompt Series

A 6-part series of production-ready prompts for the [AWS Prompt the Planet Challenge](https://dorahacks.io/hackathon/awsprompttheplanet/detail). Each prompt generates a complete, deployable Terraform + Lambda bundle for a real Claude-on-AWS production scenario, covering the full AWS Well-Architected Framework.

These are not prompts for Claude to "answer questions about AWS." They are prompts for Claude to **generate the infrastructure code** that ships an AI workload to production.

---

## The series

| # | Prompt | Pillar focus | What it generates |
|---|---|---|---|
| 1 | [Production-Ready Claude on Bedrock](prompts/01-claude-bedrock-production.md) | Cost Optimization | Bedrock + Lambda + IAM with real-time hard cost cap, two-layer kill-switch, EMF token metrics, cross-region failover |
| 2 | [Codemod-as-a-Service](prompts/02-codemod-as-a-service.md) | Operational Excellence | Lambda + S3 + DynamoDB serverless codemod runner with streaming extract, zip-bomb defenses, idempotency by content hash, classified DLQ |
| 3 | [Custom MCP Server on ECS Fargate](prompts/03-mcp-server-on-ecs-fargate.md) | Reliability | ECS Fargate + ALB hosting Model Context Protocol servers with sticky SSE sessions, SIGTERM graceful drain, two-layer auto scaling, VPC endpoints |
| 4 | [Production-Grade RAG with Bedrock Knowledge Bases](prompts/04-rag-with-bedrock-kb-citations.md) | Performance Efficiency | Bedrock KB + Claude with hard-contract citations, Haiku groundedness verifier, profile-aware vector store + cache, continuous eval harness with F1 alarm |
| 5 | [Hybrid Claude Code + Bedrock Fallback](prompts/05-claude-code-hybrid-bedrock-fallback.md) | Operational Excellence | API Gateway + Lambda router that forwards to Anthropic API with transparent Bedrock fallback on rate limits, per-developer usage tracking |
| 6 | [Production Agentic Loop on AWS](prompts/06-agentic-loop-step-functions.md) | Security | Step Functions-orchestrated Claude tool-use agent with per-tool least-privilege, reversibility-tiered human approval gates, dual cost+step budget, and tool-result injection defense |

---

## What every prompt in this series shares

Every prompt is engineered against a common framework so the outputs are consistent in shape and easy to operate together:

1. **Production Readiness Criteria.** No `# TODO` stubs, no ellipses, all identifiers generated, all exception branches handled with structured logging, files form a closed system that `terraform apply` cleanly.
2. **Cross-File Consistency Check section.** Every prompt forces a self-scan across its identifier classes (metric names, IAM roles, ARNs, Lambda function names, etc.) so the most common LLM-output failure mode at 10k+ token outputs — drift between files — surfaces as part of the deliverable.
3. **EMF (Embedded Metric Format) over PutMetricData.** Saves an extra API call per invocation; at production volumes this is ~$3–$300/month wasted otherwise. Every prompt inlines a complete `_aws`-envelope JSON example with multi-dimension sets.
4. **`Why not <obvious AWS-native alternative>?` preempt section.** Reviewers' first instinct is to ask "why not use AWS Budgets Actions / Step Functions / App Runner / CodeBuild?" — every prompt addresses the obvious comparison up front with a side-by-side table.
5. **3 required + several optional-with-defaults variables.** Required variables capture identity (region, service name, source data location); optional variables expose tuning knobs but never force the user to estimate values they cannot reasonably know.
6. **Workload profiles with sane defaults.** Each prompt has a 3-tier profile (`small/medium/large` or domain-specific equivalents) that sets capacity, scaling, and cost trade-offs without requiring the user to size every component manually. Profile naming is explicitly scoped per prompt to avoid cross-prompt confusion.
7. **Wait steps in deployment.** AWS resources have async readiness (S3 event propagation, ECS image pull + ALB health, Bedrock KB ingestion, IAM eventual consistency). Each deployment step section calls out where to wait and gives the exact polling command.
8. **Anti-patterns + Suggested test cases sections.** A list of the failure modes the prompt prevents, and a list of test cases that validate the generated bundle works as intended.

---

## How to use a prompt

1. Open the prompt's markdown file.
2. Decide your workload profile (the prompt explains how to pick).
3. Fill in the required variables in the **User prompt template** section.
4. Send the **System prompt** + **User prompt template** to Claude (Anthropic API, Bedrock, or claude.ai); the prompt is calibrated for `claude-opus-4-7` but works on Sonnet.
5. The model returns a complete deployable bundle: Terraform files, Lambda handlers, monitoring, plus inline sections for cross-file consistency check, deployment steps, smoke test, cost projection, and rollback.
6. Run the deployment steps; pay attention to the wait commands.

The prompts are designed so that a competent infra engineer can `terraform apply` the output and have a working production-shaped service in under an hour, assuming AWS account access and Bedrock model approvals are in place.

---

## License

MIT — see [LICENSE](LICENSE).

Submitted to the AWS Prompt the Planet Challenge by [@run58669-maker](https://github.com/run58669-maker). Built on the lessons of an earlier hackathon submission, [`web3py-v6-to-v7-codemod`](https://github.com/run58669-maker/web3py-v6-to-v7-codemod) (Boring AI, 2026).
