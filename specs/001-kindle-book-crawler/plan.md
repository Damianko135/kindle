# Implementation Plan: Kindle Book Crawler

**Branch**: `001-kindle-book-crawler` | **Date**: 2026-02-04 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `/specs/001-kindle-book-crawler/spec.md`

**Note**: This template is filled in by the `/speckit.plan` command. See `.specify/templates/commands/plan.md` for the execution workflow.

## Summary

Build a Deno-based web crawler using Playwright that collects free Kindle book promotion data from third-party websites. The crawler reads targets from a JSON configuration file (with URLs and CSS selectors per domain), validates robots.txt compliance before navigation, extracts book metadata, and outputs structured JSON. Core technical approach: Deno native APIs for I/O, Playwright for browser automation, sequential crawling with delays, hard domain blocklist preventing Amazon access.

## Technical Context

**Language/Version**: TypeScript on Deno 1.40+  
**Primary Dependencies**: Playwright for Deno (npm:playwright@1.40.0), robots-parser (npm:robots-parser@3.0.1)  
**Storage**: File-based (JSON config input, JSON/stdout output), no database  
**Testing**: Deno's built-in test runner (`deno test`)  
**Target Platform**: Cross-platform (Windows, macOS, Linux) via Deno runtime
**Project Type**: Single project (CLI tool)  
**Performance Goals**: Sequential crawling only, 2+ second delays between navigations, 30s timeout per page  
**Constraints**: Zero parallel requests to same domain, immediate stop on HTTP 403/429/CAPTCHA, no retries  
**Scale/Scope**: Small to medium crawl jobs (10-100 URLs per run), no database persistence, stdout/file output only

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

### Principle I: Legal and Ethical Scraping
- ✅ **PASS**: robots.txt fetching mandatory before navigation (FR-001, FR-002)
- ✅ **PASS**: Crawler refuses to proceed if robots.txt disallows access (FR-002)
- ✅ **PASS**: Fail-fast approach with clear error messages (FR-007, FR-010)
- ✅ **PASS**: No bypass or circumvention mechanisms planned

### Principle II: Amazon ToS Boundary
- ✅ **PASS**: Amazon domains explicitly blocked without navigation (FR-003)
- ✅ **PASS**: Amazon links stored but never opened/validated (FR-005)
- ✅ **PASS**: Domain blocklist enforces this at code level (hardcoded guardrail)

### Principle III: Transparency Over Stealth
- ✅ **PASS**: Consistent User-Agent header specified (FR-012: `KindleBookCrawler/VERSION (+PROJECT_URL)`)
- ✅ **PASS**: No fingerprint evasion, stealth plugins, or agent rotation planned
- ✅ **PASS**: Standard Playwright usage without stealth modifications

### Principle IV: Conservative Crawling
- ✅ **PASS**: Single-page concurrency enforced (FR-006: sequential only)
- ✅ **PASS**: Explicit delays between navigations (FR-006: default 2s, configurable)
- ✅ **PASS**: Immediate stop on HTTP 403, 429, CAPTCHA (FR-007)
- ✅ **PASS**: No retry logic, fail-fast approach (FR-008)

### Principle V: Deno-First Implementation
- ✅ **PASS**: Deno 1.40+ targeted (assumption documented in spec)
- ✅ **PASS**: Deno native APIs specified (FR-011: fetch, filesystem, permissions)
- ✅ **PASS**: No Node.js modules or CommonJS patterns planned
- ⚠️ **NEEDS VERIFICATION**: Playwright for Deno compatibility (npm:playwright via Deno's npm specifier)
- ⚠️ **NEEDS CLARIFICATION**: robots.txt parser library selection (must be Deno-compatible)

### Principle VI: Maintainability Over Speed
- ✅ **PASS**: Clear separation of concerns required (config, robots.txt, navigation, extraction)
- ✅ **PASS**: Guardrails planned (domain blocklist, pre-flight checks, configuration validation)
- ✅ **PASS**: Explicit validation gates before network requests (robots.txt check, domain validation)
- ✅ **PASS**: Fail-safe defaults (refuse to proceed when uncertain)

**Overall Status**: ✅ **GATES PASSED** - 2 clarifications needed for Phase 0 research:
1. Verify Playwright for Deno compatibility and integration approach
2. Identify/select Deno-compatible robots.txt parser library

**No violations requiring justification.**

---

### Post-Design Re-Evaluation

*Re-checked after Phase 1 design (research, data model, contracts completed)*

### Principle V: Deno-First Implementation (Re-check)
- ✅ **RESOLVED**: Playwright confirmed working via npm: specifier with `nodeModulesDir: auto`
- ✅ **RESOLVED**: robots-parser selected (`npm:robots-parser@3.0.1`) - Deno-compatible via npm: specifier
- ✅ **PASS**: All dependencies use npm: specifiers as documented in constitution
- ✅ **PASS**: No Node.js-specific APIs in design (Deno.readTextFile, Deno.writeTextFile, native fetch)

**All principles remain compliant. No new violations introduced by design decisions.**

**Final Status**: ✅ **GATES PASSED** - Ready for implementation (/speckit.tasks)

## Project Structure

### Documentation (this feature)

```text
specs/[###-feature]/
├── plan.md              # This file (/speckit.plan command output)
├── research.md          # Phase 0 output (/speckit.plan command)
├── data-model.md        # Phase 1 output (/speckit.plan command)
├── quickstart.md        # Phase 1 output (/speckit.plan command)
├── contracts/           # Phase 1 output (/speckit.plan command)
└── tasks.md             # Phase 2 output (/speckit.tasks command - NOT created by /speckit.plan)
```

### Source Code (repository root)

```text
src/
├── config/
│   └── loader.ts         # Load and validate JSON configuration
├── domain/
│   ├── allowlist.ts      # Amazon domain blocklist + validation
│   └── extractor.ts      # URL parsing and domain extraction
├── robots/
│   ├── fetcher.ts        # Fetch robots.txt via Deno fetch
│   ├── parser.ts         # Parse robots.txt rules
│   └── cache.ts          # In-memory cache for robots.txt per domain
├── crawler/
│   ├── browser.ts        # Playwright browser lifecycle
│   ├── navigator.ts      # Page navigation with timeout/error handling
│   ├── scraper.ts        # DOM extraction using CSS selectors
│   └── queue.ts          # Sequential URL queue with delay management
├── output/
│   ├── formatter.ts      # JSON output formatting
│   └── writer.ts         # Write to stdout or file (Deno APIs)
├── logging/
│   └── logger.ts         # Structured logging with Deno console
└── main.ts               # CLI entry point

tests/
├── unit/
│   ├── config/
│   ├── domain/
│   ├── robots/
│   └── output/
└── integration/
    ├── crawler_flow.test.ts
    └── robots_enforcement.test.ts

types.ts                  # Shared TypeScript types (Config, BookPromotion, etc.)
deno.json                 # Deno configuration with dependencies
```

**Structure Decision**: Single project structure selected. This is a standalone CLI tool with no frontend/backend separation or mobile components. The `/src` directory organizes code by concern (config, robots.txt, crawling, output), with clear module boundaries enforcing separation. Tests mirror source structure for discoverability.

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| [e.g., 4th project] | [current need] | [why 3 projects insufficient] |
| [e.g., Repository pattern] | [specific problem] | [why direct DB access insufficient] |
