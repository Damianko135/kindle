<!--
Sync Impact Report - Constitution v1.1.0
========================================
Version Change: 1.0.0 → 1.1.0
Rationale: MINOR bump - Added new principle (Deno-first implementation)

Modified Principles:
  - Principle V: NEW - Deno-first implementation (targeting Deno runtime, native APIs)
  - Principle VI: RENUMBERED (was V) - Maintainability over speed (unchanged content)

Added Sections: None
Removed Sections: None

Templates Requiring Updates:
  ✅ plan-template.md - Constitution Check section compatible (generic gates structure)
  ✅ spec-template.md - Requirements section compatible (principles inform functional requirements)
  ✅ tasks-template.md - Task structure compatible (Deno-first principle will guide technology choices)
  ✅ agent-file-template.md - Checked, no principle-specific references
  ✅ checklist-template.md - Checked, no principle-specific references

Follow-up TODOs: None
-->

# Kindle Scraper Constitution

## Core Principles

### I. Legal and Ethical Scraping (NON-NEGOTIABLE)

Web scraping MUST respect legal and ethical boundaries at all times. The system SHALL:

- Check and respect `robots.txt` for every domain before any access attempt
- Refuse to crawl if `robots.txt` is unavailable, unreachable, or explicitly disallows access
- Fail fast with clear error messages when scraping is not permitted
- Never attempt to bypass or circumvent access restrictions

**Rationale**: Legal compliance is absolute. No feature justifies violating website terms or applicable law. This protects both users and the project from legal liability.

### II. Amazon ToS Boundary (ABSOLUTE)

Automated access to Amazon-owned domains is FORBIDDEN. The system MUST:

- Never programmatically visit `amazon.com` or any Amazon-owned domain
- Never validate, scrape, or crawl Amazon URLs
- Allow storage of Amazon links for reference purposes only (user-initiated manual access)
- Display clear warnings when Amazon links are present but cannot be accessed programmatically

**Rationale**: Amazon's Terms of Service explicitly prohibit automated access. Violation risks account termination and legal action. Kindle data must be accessed through official APIs or manual export only.

### III. Transparency Over Stealth

The system MUST identify itself clearly and never attempt to evade detection. Required practices:

- Use a consistent, explicit, and informative `User-Agent` header (format: `KindleScraper/[VERSION] (+[PROJECT_URL])`)
- NO fingerprint evasion techniques
- NO stealth plugins or bot circumvention tools
- NO rotation of user agents or headers to appear as different clients

**Rationale**: Honest identification respects website operators and reduces the appearance of malicious activity. Stealth techniques signal bad intent and invite blocking or legal scrutiny.

### IV. Conservative Crawling

The system MUST crawl conservatively to minimize server load and detection risk:

- Single-page concurrency ONLY (no parallel requests to the same domain)
- Explicit delays between navigations (minimum 2 seconds, configurable upward)
- IMMEDIATE stop on detection signals: HTTP 403, 429, CAPTCHA, bot-detection challenges
- Graceful degradation with clear user notification when access is blocked

**Rationale**: Aggressive crawling is indistinguishable from a DDoS attack. Conservative behavior respects server resources and reduces the likelihood of being blocked or triggering defensive measures.
Deno-First Implementation

The project MUST target Deno as the runtime environment, not Node.js:

- Use Deno's native APIs: `fetch`, permissions model, filesystem (`Deno.readTextFile`, `Deno.writeTextFile`)
- Avoid Node-specific modules, polyfills, or CommonJS patterns (`require`, `module.exports`)
- Leverage Deno's built-in TypeScript support without transpilation
- Use Deno's permission system to enforce access controls (`--allow-net`, `--allow-read`)
- Prefer `jsr:` and `npm:` specifiers in `deno.json` imports over bare module specifiers

**Rationale**: Deno provides modern security primitives (granular permissions), native TypeScript, and a simpler dependency model. Using Node.js patterns defeats these advantages and creates unnecessary complexity.

### VI.

### V. Maintainability Over Speed

Code structure and safety MUST take precedence over performance optimizations:

- Clear separation of concerns: browser automation, robots.txt checking, URL management
- Guardrails MUST prevent accidental policy violations (e.g., domain allow-lists, pre-flight checks)
- Explicit validation gates before any network request
- Fail-safe defaults: when in doubt, refuse to proceed

**Rationale**: A fast but legally non-compliant scraper is worthless. Defensive coding and clear boundaries prevent accidental violations that could harm users or the project's reputation.

## Implementation Guardrails

All code MUST implement the following safeguards:

1. **Domain Allow-List**: Maintain explicit list of permitted domains; reject unlisted domains
2. **Pre-Flight Checks**: Before any navigation, validate robots.txt compliance and domain permission
3. **Circuit Breaker**: After any block signal (403, 429, CAPTCHA), halt all operations for that domain
4. **Audit Logging**: Log all attempted navigations with timestamps, domains, and outcomes
5. **Configuration Validation**: Reject configurations that could violate principles (e.g., concurrency > 1)

## Governance

- This constitution supersedes all other development practices and preferences
- Amendments require:
  - Documented 1ustification for the change
  - Impact analysis on existing code and templates
  - Version increment per semantic versioning rules (see below)
- All implementations, PRs, and code reviews MUST verify principle compliance
- When principles conflict with requirements, principles WIN—implementation refuses and explains why

**Versioning Policy**:

- **MAJOR**: Principle removal, redefinition, or backward-incompatible governance change
- **MINOR**: New principle added or existing principle materially expanded
- **PATCH**: Clarifications, wording improvements, typo fixes

**Version**: 1.0.0 | **Ratified**: 2026-02-04 | **Last Amended**: 2026-02-04
