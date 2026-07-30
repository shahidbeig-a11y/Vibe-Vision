# Research: Tapping the User's Own Agent (Issue #4)

Vibe-Vision's locked decision is that the user's own agent or key powers every model call — normalizing the plan doc, mapping code evidence to features, and periodically re-checking the deterministic analysis. No hosted model service in v1. This note evaluates the mechanisms available to do that, and — critically — surfaces a hard compliance wall that changes the recommended integration surface.

## The headline finding: Anthropic banned third-party use of Claude subscription auth (Feb 2026)

Anthropic's own current Agent SDK documentation (`code.claude.com/docs/en/agent-sdk/overview`, fetched live) states, verbatim:

> "Unless previously approved, Anthropic does not allow third party developers to offer claude.ai login or rate limits for their products, including agents built on the Claude Agent SDK. Use the API key authentication methods described in the Quickstart instead."

This is a company-wide policy, not an SDK quirk — it was enforced server-side starting January–February 2026 and covered widely in the press (Anthropic blocked subscription OAuth tokens from unofficial clients; multiple third-party wrapper products, including tools bundling Claude access into other products, were locked out). The Claude Code authentication docs confirm the mechanics: `CLAUDE_CODE_OAUTH_TOKEN` (minted via `claude setup-token`) and the default `/login` subscription OAuth are both real, working credentials — but "using OAuth tokens obtained through Claude Free, Pro, or Max accounts in any other product, tool, or service" is a Consumer Terms of Service violation unless Anthropic pre-approves the integration.

This directly rules out the cleanest version of "ride the user's Claude subscription" — building on the Agent SDK and letting it drive the `/login` OAuth flow inside Vibe-Vision. That path is explicitly named as disallowed.

## Mechanism-by-mechanism

### 1. Claude Agent SDK using the user's Claude.ai subscription OAuth — BLOCKED

- **Technical stability:** Works today; the SDK and CLI both resolve subscription OAuth by default.
- **ToS compliance: hard blocker.** Explicitly named and banned in current official docs (quoted above). Not a gray area — Anthropic wrote "including agents built on the Claude Agent SDK" specifically to close this door.
- **Latency/cost:** Would have been free to the user (covered by their existing plan) — irrelevant since it's not usable.
- **Coverage:** N/A.

### 2. Claude Agent SDK / `claude -p` using an API key or a `claude setup-token` CI token — VIABLE, with a documented tension

- **Technical stability:** High. `claude -p` (headless/print mode) is a first-class, documented feature: text/json/stream-json output, exit codes, `--max-turns`, `--max-budget-usd`, `--permission-mode`, piped stdin. Non-Python/TS callers are explicitly told by Anthropic to shell out to the CLI with `-p --output-format json` rather than use the SDK directly — this is the sanctioned pattern for exactly Vibe-Vision's use case (a non-JS/Python product driving the agent loop).
- **ToS compliance:** API-key auth (`ANTHROPIC_API_KEY`) is unambiguously compliant — it's the explicit recommended replacement in the same doc that bans subscription OAuth. `CLAUDE_CODE_OAUTH_TOKEN` (`claude setup-token`) sits in a real tension: Anthropic's own docs describe it as intended "for CI pipelines and scripts where browser login isn't available," which sounds exactly like a third-party orchestrator — but the Feb 2026 enforcement wave and press coverage describe a blanket rule that OAuth tokens from consumer plans are not permitted "in any other product, tool, or service," without carving out setup-token explicitly. Treat this token as usable for the user's own scripts, not as a sanctioned foundation for a commercial third-party product, until Anthropic clarifies.
- **Latency/cost:** Runs on the user's machine; cost is whatever they already pay Anthropic (subscription-covered if using their Claude Code login and staying under its usage caps, or metered if they point it at an API key). No added network hop for Vibe-Vision.
- **Coverage:** Requires the user to already have Claude Code installed and authenticated (a real subset of "vibe coders," not universal — plenty use Cursor, Codex, or Copilot instead). Among Claude Code users, close to 100% coverage since `claude -p` ships with every install.

### 3. Invoking other installed agent CLIs headlessly (Codex CLI, Gemini CLI, Cursor CLI)

- **Codex CLI (`codex exec`):** Documented, stable headless mode; same model/tools as interactive use, no TUI. Supports ChatGPT Plus/Pro sign-in or an API key. No explicit third-party-wrapper ban was found in OpenAI's current public docs during this research — general ToS misrepresentation/sharing clauses presumably still apply, but nothing as explicit as Anthropic's Feb 2026 statement turned up.
- **Gemini CLI:** Documented non-interactive/headless mode (`-p`/`--prompt`, auto-triggered in non-TTY contexts), text/JSON/NDJSON output. No explicit subscription-reuse restriction found in the public docs surveyed.
- **Cursor CLI (`cursor-agent`):** Headless `-p` print mode exists and is explicitly described as scriptable (pipe JSON output to other programs), but it requires an active Cursor subscription and is still in beta — flags can change release to release. No ToS statement on third-party wrapping was found.
- **Coverage:** Each covers only its own user base (Codex → ChatGPT Plus/Pro or OpenAI API users; Gemini CLI → Google AI users; Cursor → Cursor subscribers). None alone matches the reach of "any vibe coder with an API key."
- **Caveat:** This research did not turn up an equivalent explicit ban from OpenAI or Google, but that absence is not proof of safety — it may just mean the press hasn't covered an equivalent crackdown, or one hasn't happened yet. Don't treat silence as a green light equivalent to a documented policy.

### 4. Pasted API key (Anthropic / OpenAI / OpenRouter) called directly

- **Technical stability:** Highest of all options — a stable, versioned, documented HTTP API with official SDKs in every major language. No dependency on a separate CLI being installed, on its version, or on its headless-mode flags changing.
- **ToS compliance:** Fully compliant — this is the explicit, sanctioned integration path for every provider. Anthropic's own ban notice names API-key auth as the required alternative.
- **Latency/cost:** One extra network hop to the provider vs. an already-running local CLI, but negligible in practice; cost is metered per-token to the user's own account, fully transparent, no subscription-cap ambiguity.
- **Coverage:** Universal in principle (any vibe coder can generate a key from any of the three providers in minutes), but real coverage is bounded by how many users are willing to leave the flow to fetch and paste a key versus how many already have Claude Code, Codex, or Cursor sitting authenticated on their machine.

## Ranked recommendation for v1

1. **Pasted API key (Anthropic primary, OpenAI/OpenRouter as fallbacks)** — the only mechanism with zero compliance risk, the most stable long-term surface, and no dependency on the user having a specific CLI installed. Make this the default and the one every user can fall back to.
2. **Headless `claude -p` invocation of the user's already-installed, already-logged-in Claude Code, using `ANTHROPIC_API_KEY` when present** — for users who already have Claude Code set up, this is a lower-friction onboarding path with identical compliance posture to (1), since it's the same API-key auth Claude Code itself resolves to. Do not build this path around `CLAUDE_CODE_OAUTH_TOKEN` or bare subscription `/login` reuse.
3. **Codex CLI / Gemini CLI headless invocation** — secondary, opportunistic coverage for users on those ecosystems; treat as "nice to have" additions once the Claude path is proven, not a v1 dependency, given the immaturity of public ToS guidance for these two.
4. **Do not build on the Claude Agent SDK riding a user's Claude.ai subscription OAuth.** It is explicitly named and banned in Anthropic's current documentation; building on it would mean shipping a feature Anthropic can and does disable server-side without notice.

## Hard compliance blocker (repeated for visibility)

> Anthropic: "Unless previously approved, Anthropic does not allow third party developers to offer claude.ai login or rate limits for their products, including agents built on the Claude Agent SDK." — `code.claude.com/docs/en/agent-sdk/overview`, live as of this research.

Any v1 design that depends on capturing or replaying a user's Claude.ai Pro/Max subscription session inside Vibe-Vision is out — Anthropic enforces this server-side and has already done so against other tools industry-wide in 2026.
