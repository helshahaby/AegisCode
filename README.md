# AegisCode — Autonomous Incident Commander

**Author:** Hossam Elshahaby · **Event:** HackSprint NYC
**Application** [website](aegiscode.lovable.app)
**Demo** [Video](https://youtu.be/-pvNaeoXhjQ)

AegisCode is a multi-agent AI system that detects cloud outages, decides on the safest fix, writes and tests a patch in isolation, validates it against security guardrails, and deploys only after a human approves.

## Problem

On-call engineers lose hours triaging noisy alerts. Downtime costs revenue and trust, and rushed hot-fixes often introduce new risks: wildcard IAM policies, leaked secrets, untested changes.

## Solution — four agents, one safe pipeline

| Agent         | Sponsor tool                        | Responsibility                                                                                                                   |
| ------------- | ----------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| **Sentinel**  | AWS Sandbox (CloudWatch) + Arize AI | Analyses telemetry, ranks root-cause hypotheses with confidence and blast radius                                                 |
| **Commander** | ElevenLabs                          | Picks ONE least-privilege, reversible remediation, documents rejected alternatives and rollback, speaks a voice brief to on-call |
| **Developer** | Daytona + Anchor Browser            | Writes the minimal patch in an isolated sandbox, defines tests and staging checks                                                |
| **Validator** | Arize AI guardrails                 | LLM review for correctness/hallucination risk **plus** deterministic policy rules that cannot be overridden by model output      |

Then: **Human approval gate → AWS us-east-1 rollout**. Every step is recorded as a trace span.

## Safety design

- Deterministic guardrails (`src/lib/guardrails.ts`) block wildcard IAM (`"Action": "*"`), `AdministratorAccess`, AWS keys, hard-coded credentials, public S3 and destructive commands; open ingress produces a warning.
- A blocking finding vetoes deployment even if the Validator LLM approves.
- **Red-team toggle** in the UI injects a malicious wildcard IAM policy to prove the veto works live.
- No production change without explicit human approval; rollback plan always produced.
- All AI calls run server-side; keys never reach the browser.

## App features

- 3 realistic outage scenarios (Lambda OOM, RDS connection exhaustion, S3/CloudFront 403)
- Live agent pipeline with status, latency and reasoning for each step
- Voice briefings (browser speech; ElevenLabs adapter in spec)
- Arize-style trace panel
- Approve / Reject deployment gate

## Tech stack

TanStack Start (React 19, Vite, Tailwind v4) · server functions on Cloudflare Workers · Lovable AI Gateway (LLM reasoning) · Vitest.

Daytona, Anchor Browser, AWS deploy and ElevenLabs are represented by simulated adapters in this demo build; production wiring follows the snippets in the architectural specification (Daytona SDK workspace creation, Phoenix/OpenInference tracing, ElevenLabs TTS, Anchor staging verify).

## Run

```bash
bun install
bun run dev
bunx vitest run src/lib/guardrails.test.ts   # safety rule tests
```

## Demo script (3 min)

1. Pick "Checkout Lambda out of memory" → **Trigger incident & respond**.
2. Watch Sentinel hypotheses, Commander decision + voice brief, Developer patch, Anchor checks.
3. Validator approves → click **Approve deploy** → telemetry restored.
4. Enable **Inject unsafe patch** and rerun → guardrail veto, deployment blocked.
