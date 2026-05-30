# Architecture Decision Records (ADRs)

This document captures the key technical decisions made for FinBot and the reasoning behind them.

---

## ADR-001: Python 3.12 as runtime

**Date:** 2026-05-30

**Decision:** Use Python 3.12 for all Lambda functions.

**Reasoning:**
- Most popular language for AWS Lambda
- Rich ecosystem for AI/ML (boto3, etc.)
- Easy for contributors with varying experience levels
- Type hints improve code readability
- Fast cold starts compared to Java/C#

**Alternatives considered:**
- Node.js — faster cold starts but less readable for complex logic
- Go — best performance but steeper learning curve

---

## ADR-002: DynamoDB single-table design

**Date:** 2026-05-30

**Decision:** Use a single DynamoDB table with composite keys (PK + SK) for all entities.

**Reasoning:**
- All access patterns are known upfront (user-centric queries)
- Single table = single Terraform resource, simpler IAM
- On-demand billing = $0 at low scale
- No cold starts (unlike Aurora Serverless)
- Natural fit for serverless (no connection pooling needed)

**Access patterns:**
- Get user profile: PK=USER#phone, SK=PROFILE
- Get monthly expenses: PK=USER#phone, SK begins_with(EXP#2026-06)
- Get budget: PK=USER#phone, SK=BUDGET#2026-06
- Get goals: PK=USER#phone, SK begins_with(GOAL#)

**Alternatives considered:**
- RDS/Aurora — overkill for key-value access, costs more at rest
- Multiple DynamoDB tables — more IAM complexity, no benefit

---

## ADR-003: Terraform over SAM/CDK

**Date:** 2026-05-30

**Decision:** Use Terraform for all infrastructure.

**Reasoning:**
- Industry standard — "Terraform" on a resume carries weight
- Multi-cloud portable (contributors learn transferable skills)
- Declarative and readable
- State management is explicit (S3 backend)
- Better for team collaboration than SAM (which is more single-dev)

**Alternatives considered:**
- SAM — simpler for Lambda but less portable, weaker ecosystem
- CDK — powerful but requires TypeScript knowledge, adds complexity

---

## ADR-004: Meta Cloud API (direct) over Twilio

**Date:** 2026-05-30

**Decision:** Use Meta's WhatsApp Cloud API directly, not through a provider like Twilio.

**Reasoning:**
- Free (no per-message platform fee)
- Direct control over webhook and message handling
- Better learning experience (understand the protocol)
- Meta provides test phone numbers for development
- No vendor lock-in

**Alternatives considered:**
- Twilio — faster setup but $0.005/message + monthly fee
- MessageBird — similar cost issues

---

## ADR-005: Bedrock Claude Haiku for NLU

**Date:** 2026-05-30

**Decision:** Use Amazon Bedrock with Claude Haiku for natural language understanding.

**Reasoning:**
- Cheapest model that handles Portuguese well
- Native AWS (no external API keys, IAM-based auth)
- Low latency (~500ms for short prompts)
- Sufficient for expense extraction and categorization
- Can upgrade to Sonnet later if needed

**Alternatives considered:**
- OpenAI GPT — requires external API key, not AWS-native
- Bedrock Titan — worse at Portuguese
- Claude Sonnet — better but 10x more expensive

---

## ADR-006: Monorepo

**Date:** 2026-05-30

**Decision:** Keep all code (handlers, lib, infra, tests, docs) in a single repository.

**Reasoning:**
- Small team (3-5 people)
- Shared libraries (dynamo.py, whatsapp.py) used across handlers
- Single CI/CD pipeline
- Easier onboarding (one clone, everything is there)
- Atomic changes (infra + code in same PR)

**When to split:**
- If frontend (dashboard) grows significantly
- If team exceeds 10 people
- If deploy cycles diverge (backend vs frontend)

---

## ADR-007: Single AWS account for all environments

**Date:** 2026-05-30

**Decision:** Use one AWS account (801010597252) for dev, hml, and prod, separated by resource naming.

**Reasoning:**
- Budget is $2/month — can't afford baseline costs of multiple accounts
- Control Tower baseline (Config, CloudTrail) costs ~$0.50/account/month
- 3-5 contributors, not enterprise scale
- Naming convention (`finbot-table-dev`, `finbot-table-prod`) provides sufficient isolation
- Terraform workspaces or variables handle environment switching

**When to split:**
- If FinBot gets paying customers
- If prod data needs strict isolation (compliance)
- If team grows beyond 10 people

---

## ADR-008: GitHub Projects for task management

**Date:** 2026-05-30

**Decision:** Use GitHub Projects (Kanban board) for tracking work.

**Reasoning:**
- Free and unlimited
- Integrated with PRs (auto-close issues on merge)
- Contributors already use GitHub (no new tool)
- Visual board for progress tracking
- Zero learning curve

**Alternatives considered:**
- Jira — overkill, requires separate account
- Trello — not integrated with code
- Notion — not integrated with PRs
