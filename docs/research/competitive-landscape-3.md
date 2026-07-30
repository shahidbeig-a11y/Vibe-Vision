# Competitive landscape and positioning gap (Issue #3)

Research date: 2026-07-30. Scope: who already visualizes codebases or plan-vs-built drift, and what gap Vibe-Vision fills for a code-illiterate vibe coder.

## 1. Codebase visualization tools

| Tool | What it draws | Who it serves | Status / pricing |
|---|---|---|---|
| **CodeSee** | Interactive file-dependency maps, PR review maps | Engineering teams (onboarding, code review) | Acquired by GitKraken, sunset as a standalone product in 2024; folded into GitKraken's line ([OpenVisio](https://openvisio.io/compare/codesee-alternative), [SourceForge](https://sourceforge.net/software/product/CodeSee/)) |
| **GitKraken (post-CodeSee)** | Git graph, repo/dependency views | Developers using a Git GUI | $4–14/user/mo individual; $8.95+/user/mo teams; Business/Enterprise $216–336/user/yr ([toolradar](https://toolradar.com/tools/gitkraken/pricing), [schneider.im](https://www.schneider.im/gitkraken-new-list-prices-for-new-sales-and-renewals/)) |
| **GitHub repo-visualizer** (GitHub Next) / AbanteAI repo-visualizer | Static SVG or interactive graph of file/folder structure and git history | Developers wanting a repo map, experimental/hobby use | Free, open source, GitHub Action ([githubnext.com](https://githubnext.com/projects/repo-visualization/), [AbanteAI](https://github.com/AbanteAI/repo-visualizer)) |
| **GitDiagram** | AI-generated Mermaid diagram of any public repo's architecture (paste URL) | Developers exploring an unfamiliar repo | Free, open source, no paywall ([gitdiagram.com](https://gitdiagram.com/), [GitHub](https://github.com/ahmedkhaleel2004/gitdiagram)) |
| **swark** | Auto-generates Mermaid architecture diagrams from source via GitHub Copilot (VS Code extension) | Developers with a Copilot seat | Free, open source ([GitHub](https://github.com/swark-io/swark)) |
| **Eraser.io (DiagramGPT / AI architecture diagrams)** | AI-drawn architecture/ER/sequence diagrams from a prompt or code | Engineers and technical PMs documenting systems | Free tier (3 files, 5 AI diagrams); Starter $15–20/mo; Business $45–60/mo ([eraser.io/pricing](https://www.eraser.io/pricing)) |

**Why none serve a code-illiterate vibe coder:** every tool in this category draws the code as it exists — folders, imports, call graphs. None of them know what the user *planned* to build, so none can render a verdict (built/partial/missing/extra). They're read-only maps for people who already read code; a non-coder gets a pretty graph with no way to judge if it's right.

## 2. Agent-observability / non-coder review products

- **AI agent observability platforms** (Braintrust, LangSmith, Arize Phoenix, Helicone, Galileo, Maxim, Datadog LLM Observability, Latitude) trace LLM/agent calls, evals, and tool-use decisions. Some (Latitude, Agenta) now let PMs/QA annotate traces "without code," and Latitude closes the loop from a detected issue to an opened PR ([Latitude buyer's guide](https://latitude.so/blog/ai-agent-observability-platforms-buyers-guide-2026)). This is agent-execution monitoring for engineering orgs — trace-level, not feature-level, and not oriented at "did the agent build what I planned."
- **CodeRabbit** generates a plain-English walkthrough of a pull request plus a diagram of how the changed components interact, and lets reviewers write custom checks in plain language ([coderabbit.ai blog](https://www.coderabbit.ai/blog/coderabbit-review-reads-a-pr-how-author-would-explain-it)). Pricing: Pro $24–30/user/mo, Pro Plus $48/user/mo, free tier for open source ([dev.to](https://dev.to/rahulxsingh/coderabbit-pricing-in-2026-free-tier-pro-plans-and-enterprise-costs-1pc4)). It explains *diffs*, still assumes a developer audience reviewing PRs in GitHub — not a founder comparing a whole app against their plan.
- **vibe-check** (open-source Claude Code/Codex skill by TexasBedouin) is the closest same-audience product: explicitly built for non-coders, takes them from idea to a buildable plan (PRD, user flows, architecture). But it stops at planning — it has "build checkpoints" where the agent narrates progress, and does not track divergence between the plan and the actual code once building starts. Free, MIT-licensed ([GitHub](https://github.com/TexasBedouin/vibe-check)).

## 3. Spec-vs-implementation / traceability checkers

Spec-driven development (SDD) tools — GitHub Spec Kit, AWS Kiro, Tessl, OpenSpec, BMAD, Google Antigravity — emerged through 2025–2026 specifically to fight plan/code drift in AI-agent coding. GitHub Spec Kit's loop (Specify → Plan → Tasks → Implement) is free, open source, agent-agnostic across 30+ coding agents ([Augment Code roundup](https://www.augmentcode.com/tools/best-spec-driven-development-tools)). These tools generate the spec that feeds the agent and re-read it during implementation, but none render a standing visual verdict of what's built vs. planned after the fact — "static spec tools produce documents that drift from implementation within hours" is the exact failure mode they're still fighting, not one they've solved with a live view.

Formal requirements-traceability tools (Xray for Jira, ACCELQ, PractiTest, Tricentis qTest, TestMonitor, SonarQube, Codecov) exist to link requirements → tests → code, mostly for regulated industries (DO-178C, ISO 26262, IEC 62304) ([Inflectra buyer's guide](https://www.inflectra.com/tools/requirements-management/10-best-requirements-traceability-tools)). They require someone who can read requirements docs *and* test code to wire the links; nothing about them is usable by someone who can't read code, and pricing/complexity targets enterprise QA orgs, not a solo builder.

## Positioning gap

Three separate product categories exist — codebase visualizers (draw the code, blind to the plan), SDD/spec tools (hold the plan, blind to drift once built), and traceability/observability platforms (link requirements to tests, built for engineers who read code). None combine all three: a non-coder's plan, the agent-built code, and a live built/partial/missing/extra verdict with evidence, zoomable into architecture. That triangle — plan-aware, code-illiterate-usable, and continuously live rather than a one-time snapshot — is open. Vibe-Vision is the first to sit at the center of it.

## Pricing anchors

| Product | Price | Segment |
|---|---|---|
| GitDiagram / swark / repo-visualizer | Free | Open-source, dev hobbyist |
| GitKraken Advanced (post-CodeSee) | $14/user/mo | Individual developer |
| Eraser.io Starter | $15–20/mo | Technical PM / small team |
| CodeRabbit Pro | $24–30/user/mo | Dev team, PR review |
| CodeRabbit Pro Plus | $48/user/mo | Dev team, heavier usage |
| Enterprise RTM (Xray/ACCELQ tier) | Custom, enterprise-only | Regulated QA orgs |
| **Reference point** | **$5/mo** | Mo's prior comparable dev tool pricing |

Given the gap Vibe-Vision fills is closest in spirit to a single-user visualization/verification tool (GitDiagram/Eraser tier) rather than a team dev-ops platform (CodeRabbit/GitKraken tier), a $5–15/mo range is defensible: above free open-source visualizers (which offer no plan-awareness) and well below team-seat tools priced for engineers, while still charging for the unique verdict-with-evidence layer no competitor offers.
