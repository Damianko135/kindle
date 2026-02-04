# Research: Kindle Book Crawler Technical Decisions

**Feature**: 001-kindle-book-crawler  
**Date**: 2026-02-04  
**Phase**: 0 (Research & Resolution)

## Overview

This document consolidates research findings for resolving technical unknowns identified in the Technical Context. All decisions documented here inform the implementation plan and ensure constitution compliance.

## Research Tasks

### Task 1: Playwright Compatibility with Deno 1.40+

**Question**: Can Playwright be used in Deno via npm: specifier, and what is the recommended integration approach?

#### Decision
**Use Playwright via npm: specifier with local node_modules**

Playwright is compatible with Deno 1.40+ through the `npm:playwright` specifier. However, it requires specific setup due to browser binary management requirements.

#### Rationale
- Deno's npm compatibility layer supports Playwright successfully
- Playwright is actively maintained and battle-tested for browser automation
- Deno team has fixed multiple Playwright-specific bugs, indicating ongoing support
- No viable Deno-native alternatives exist with equivalent maturity

#### Integration Details

**Import Pattern:**
```typescript
import { chromium } from "npm:playwright@1.40.0";
```

**Required Deno Configuration:**
```json
{
  "nodeModulesDir": "auto"
}
```

**Required Permissions:**
- `--allow-read` - Browser binaries, config files, node_modules
- `--allow-write` - Browser data, cache, temp files
- `--allow-net` - Download binaries, network requests
- `--allow-env` - Environment variables (HOME, PATH, etc.)
- `--allow-run` - **Critical** - Spawn browser processes
- `--node-modules-dir=auto` - Required for browser binary installation

**Setup Steps:**
1. Add `"nodeModulesDir": "auto"` to `deno.json`
2. Run `deno run --allow-run=npx --allow-read --allow-write --allow-net --allow-env npm:playwright install` (one-time)
3. Import via `npm:playwright@1.40.0` in code
4. Run crawler with all required permissions

#### Limitations
- **Browser binaries**: Requires separate `playwright install` step (cannot auto-install on first run)
- **node_modules required**: Unlike most npm packages, Playwright needs local `node_modules` for browser binaries
- **Security trade-off**: `--allow-run` permission allows spawned processes unrestricted access
- **Not compatible with `deno compile`**: Binary dependencies prevent single-executable compilation
- **Platform-specific quirks**: Windows has had specific subprocess issues (fixed in recent Deno versions)

#### Alternatives Considered
- **Deno-native browser automation**: No mature alternatives exist (puppeteer-deno archived, no active projects)
- **Web-native CDP**: Too low-level, would require significant implementation effort
- **Custom solution**: Not practical for browser automation complexity

---

### Task 2: robots.txt Parsing Library Selection

**Question**: What Deno-compatible library should be used for parsing and enforcing robots.txt rules?

#### Decision
**Use npm:robots-parser@3.0.1 via npm: specifier**

The `robots-parser` npm package is the recommended solution for robots.txt parsing in this Deno project.

#### Rationale
- **Proven maturity**: 2M+ weekly npm downloads, actively maintained
- **Standards-compliant**: Implements RFC 9309 (robots.txt specification)
- **Deno-compatible**: Uses modern Web APIs (URL object), no Node.js-specific dependencies
- **Feature-complete**: Supports all required features (user-agent matching, wildcards, Allow/Disallow rules)
- **TypeScript support**: Built-in type declarations included
- **Battle-tested**: Follows Google's robots.txt implementation patterns

#### Implementation Details

**Import Pattern:**
```typescript
import robotsParser from "npm:robots-parser@3.0.1";
```

**Core API:**
```typescript
const robots = robotsParser(robotsTxtUrl, robotsTxtContent);
robots.isAllowed(targetUrl, userAgent);  // Returns boolean
robots.getCrawlDelay(userAgent);         // Returns number or undefined
robots.getSitemaps();                    // Returns string[]
```

**Key Features:**
- **User-agent matching**: Case-insensitive, supports wildcards (`googlebot*`)
- **Rule precedence**: Most specific (longest matching) rule wins; on conflicts, least restrictive applies
- **Wildcards**: `*` = 0+ characters, `$` = end of URL (e.g., `/*.php$` matches only URLs ending in `.php`)
- **Path matching**: Case-sensitive, relative to domain root, must start with `/`
- **Error handling**: Gracefully handles malformed robots.txt (ignores invalid lines)

**Constitution-Aligned Behavior:**
```typescript
// Fetch robots.txt
const response = await fetch(`${domain}/robots.txt`, {
  headers: { "User-Agent": "KindleBookCrawler/1.0 (+PROJECT_URL)" }
});

// Fail-safe approach per Constitution Principle I
if (!response.ok) {
  // 4xx/5xx: Disallow domain (fail-safe, not per spec which allows on 404)
  console.error(`robots.txt unavailable (${response.status}): disallowing domain`);
  return false;  // Skip all URLs for this domain
}

const robotsTxt = await response.text();
const robots = robotsParser(response.url, robotsTxt);

// Check if crawling is allowed
if (!robots.isAllowed(targetUrl, userAgent)) {
  console.log(`❌ Disallowed by robots.txt: ${targetUrl}`);
  return false;
}
```

**Caching Strategy:**
- Cache parsed `robots.txt` per domain in memory
- 24-hour TTL per Google's recommendations
- Re-fetch on cache miss or expiry
- No persistent storage (in-memory only for crawler session)

#### Edge Cases Supported
- Partial malformed files (ignore invalid lines, parse valid ones)
- Missing robots.txt → Constitution requires DISALLOW (fail-safe) vs spec allows (permissive)
- Relative vs absolute URLs (library handles both)
- Percent-encoded paths and UTF-8 characters
- Conflicting Allow/Disallow rules for same path

#### Alternatives Considered
1. **Deno-native libraries (deno.land/x, JSR)**: No mature robots.txt parsers found
2. **Custom implementation**: Would violate Constitution Principle VI (Maintainability over Speed) - significant effort to handle edge cases correctly, maintenance burden
3. **Other npm packages** (`robots-txt-parse`, `robots-txt-guard`): Less mature, fewer downloads, lacking TypeScript support

---

## Summary of Technical Decisions

### Dependencies Added
1. **Playwright** (`npm:playwright@1.40.0`) - Browser automation
   - Requires: `nodeModulesDir: auto` in `deno.json`
   - Setup: `npx playwright install` (one-time)
   - Permissions: `--allow-read`, `--allow-write`, `--allow-net`, `--allow-env`, `--allow-run`

2. **robots-parser** (`npm:robots-parser@3.0.1`) - robots.txt parsing
   - No special setup required
   - Permissions: `--allow-net` (for fetch)

### Configuration Requirements

**deno.json:**
```json
{
  "nodeModulesDir": "auto",
  "imports": {
    "playwright": "npm:playwright@^1.40.0",
    "robots-parser": "npm:robots-parser@^3.0.1"
  },
  "tasks": {
    "crawl": "deno run --allow-read --allow-write --allow-net --allow-env --allow-run src/main.ts"
  }
}
```

**Permissions Summary:**
- `--allow-read`: Config file, browser binaries, node_modules
- `--allow-write`: Optional file output, browser cache/data
- `--allow-net`: robots.txt fetching, page navigation
- `--allow-env`: Playwright environment variables
- `--allow-run`: Browser process spawning (Playwright requirement)

### Constitution Compliance

All decisions align with project constitution:

- **Principle I (Legal/Ethical Scraping)**: robots-parser with fail-safe approach (disallow on errors)
- **Principle II (Amazon ToS)**: No conflict (decisions don't affect Amazon blocking)
- **Principle III (Transparency)**: Standard Playwright usage, no stealth modifications
- **Principle IV (Conservative Crawling)**: No conflict (sequential crawling enforced at queue level)
- **Principle V (Deno-First)**: Both libraries use npm: specifier as documented in constitution; no Node.js-specific code
- **Principle VI (Maintainability)**: Mature, well-documented libraries chosen over custom implementations

### Open Questions Resolved
✅ **Playwright Deno compatibility**: Confirmed working via npm: specifier with node_modules setup  
✅ **robots.txt parser library**: Selected npm:robots-parser@3.0.1 as best option

### Next Steps (Phase 1)
1. Generate `data-model.md` with TypeScript interfaces for entities
2. Create `quickstart.md` with setup instructions (including Playwright install step)
3. Update agent context with new dependencies
4. Proceed to implementation planning
