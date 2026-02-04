# Data Model: Kindle Book Crawler

**Feature**: 001-kindle-book-crawler  
**Date**: 2026-02-04  
**Phase**: 1 (Design)

## Overview

This document defines the TypeScript interfaces and types for all entities used in the Kindle Book Crawler. The model supports configuration-driven crawling with per-domain CSS selectors, robots.txt compliance, and structured book data extraction.

## Core Entities

### CrawlerConfiguration

Represents the input configuration file (`config.json`) provided by the user.

```typescript
interface CrawlerConfiguration {
  /** List of crawl targets with URLs and selectors */
  targets: TargetConfiguration[];

  /** Delay in seconds between page navigations (default: 2) */
  delay?: number;

  /** Timeout in seconds for page operations (default: 30) */
  timeout?: number;

  /** User-Agent header format (default: KindleBookCrawler/VERSION (+PROJECT_URL)) */
  userAgent?: string;

  /** Optional file path for JSON output (if omitted, writes to stdout) */
  outputFile?: string;
}
```

**Validation Rules:**

- `targets`: Must be non-empty array
- `delay`: If provided, must be ≥ 2 seconds (constitution requirement)
- `timeout`: If provided, must be > 0
- `userAgent`: If provided, must follow format `/^[^/]+\/[\d.]+ \(\+https?:\/\/.+\)$/`
- `outputFile`: If provided, must be valid file path (checked by Deno)

---

### TargetConfiguration

Represents a single crawl target with its URL and CSS selectors.

```typescript
interface TargetConfiguration {
  /** Full URL to crawl */
  url: string;

  /** CSS selectors for extracting book data */
  selectors: SelectorConfiguration;
}
```

**Derived Properties:**

- `domain`: Extracted from `url` using `new URL(url).hostname`

**Validation Rules:**

- `url`: Must be valid HTTP/HTTPS URL
- `url`: Must not point to Amazon domains (checked by domain allowlist)
- `selectors`: All required selectors must be present

---

### SelectorConfiguration

Defines CSS selectors for extracting book metadata from a page.

```typescript
interface SelectorConfiguration {
  /** CSS selector for book title (required) */
  title: string;

  /** CSS selector for author name (optional, may be null/undefined if not present) */
  author?: string;

  /** CSS selector for Amazon product link (required) */
  amazonLink: string;
}
```

**Usage Notes:**

- Selectors are applied to each book listing element on the page
- If a selector returns multiple elements, the first match is used
- `author` is optional; if selector is provided but no match found, field will be null

---

### BookPromotion

Represents a free Kindle book promotion discovered on a third-party site.

```typescript
interface BookPromotion {
  /** Book title (extracted via CSS selector) */
  title: string;

  /** Author name (optional, may be null if not found) */
  author: string | null;

  /** Source page URL where this book was discovered */
  sourceUrl: string;

  /** Amazon product URL (stored but never visited) */
  amazonProductUrl: string;

  /** ISO 8601 timestamp when this book was crawled */
  crawlTimestamp: string;
}
```

**Validation Rules:**

- `title`: Required, non-empty string
- `author`: Nullable string (null if not extracted)
- `sourceUrl`: Must be valid URL
- `amazonProductUrl`: Must be valid Amazon URL (domain checked but never navigated to)
- `crawlTimestamp`: ISO 8601 format (generated via `new Date().toISOString()`)

**Example:**

```json
{
  "title": "The Great Gatsby",
  "author": "F. Scott Fitzgerald",
  "sourceUrl": "https://freekindlebooks.example.com/classics",
  "amazonProductUrl": "https://www.amazon.com/dp/B000FC1GHO",
  "crawlTimestamp": "2026-02-04T10:30:00.000Z"
}
```

---

### CrawlTarget

Represents runtime state for a URL being crawled.

```typescript
interface CrawlTarget {
  /** Full URL to crawl */
  url: string;

  /** Domain extracted from URL */
  domain: string;

  /** robots.txt compliance status */
  robotsStatus: "allowed" | "disallowed" | "unavailable";

  /** Current crawl status */
  status: "pending" | "success" | "skipped" | "failed";

  /** Reason for skip/failure (if applicable) */
  reason?: string;

  /** CSS selectors for this target */
  selectors: SelectorConfiguration;
}
```

**State Transitions:**

- `pending` → `success`: URL crawled successfully, books extracted
- `pending` → `skipped`: URL skipped due to robots.txt disallow or Amazon domain
- `pending` → `failed`: URL navigation/extraction failed (timeout, HTTP error, etc.)

**Validation Rules:**

- `domain`: Must not be an Amazon domain (amazon.com, amazon._, a.co, amzn._)
- `robotsStatus`: Must be checked before navigation
- `reason`: Required when `status` is 'skipped' or 'failed'

---

### RobotsTxtRule

Represents parsed robots.txt directives for a domain (in-memory cache).

```typescript
interface RobotsTxtRule {
  /** Domain this rule applies to */
  domain: string;

  /** User-agent this rule was parsed for */
  userAgent: string;

  /** Parsed robots.txt content (via npm:robots-parser) */
  parser: RobotsParser; // External type from robots-parser library

  /** ISO 8601 timestamp when robots.txt was fetched */
  fetchTimestamp: string;

  /** TTL in milliseconds (24 hours = 86400000ms) */
  ttl: number;
}
```

**Cache Behavior:**

- Rules cached per domain (keyed by `domain`)
- 24-hour TTL per Google's recommendations
- Re-fetch on cache miss or expiry
- Cache cleared on crawler exit (no persistent storage)

**Validation Rules:**

- `domain`: Must be valid domain name
- `parser`: Instance of npm:robots-parser
- `ttl`: Default 86400000ms (24 hours), configurable if needed

---

## Internal Types

### CrawlResult

Represents the aggregated output from a crawl session.

```typescript
interface CrawlResult {
  /** All successfully extracted books */
  books: BookPromotion[];

  /** Summary statistics */
  summary: {
    total: number;
    successful: number;
    skipped: number;
    failed: number;
  };

  /** Optional: URLs that were skipped/failed (for debugging) */
  errors?: Array<{
    url: string;
    reason: string;
  }>;
}
```

**Output Formats:**

- **stdout** (default): `JSON.stringify(result, null, 2)`
- **file**: Same JSON written to `config.outputFile` via `Deno.writeTextFile()`

---

### LogEntry

Structured log format for audit trail (constitution requirement).

```typescript
interface LogEntry {
  /** ISO 8601 timestamp */
  timestamp: string;

  /** Log level */
  level: "info" | "warn" | "error";

  /** Log message */
  message: string;

  /** Optional context data */
  context?: {
    url?: string;
    domain?: string;
    reason?: string;
    [key: string]: any;
  };
}
```

**Logging Behavior:**

- All navigation attempts logged
- All skipped URLs logged with reason
- robots.txt fetch outcomes logged
- Domain blocklist rejections logged

---

## Domain Constants

### Amazon Domain Blocklist

Hardcoded list of blocked domains (constitution requirement II).

```typescript
const AMAZON_DOMAINS: readonly string[] = [
  "amazon.com",
  "amazon.co.uk",
  "amazon.ca",
  "amazon.de",
  "amazon.fr",
  "amazon.es",
  "amazon.it",
  "amazon.co.jp",
  "amazon.cn",
  "amazon.in",
  "amazon.com.br",
  "amazon.com.mx",
  "amazon.com.au",
  "a.co",
  "amzn.com",
  "amzn.to",
] as const;

function isAmazonDomain(domain: string): boolean {
  const normalized = domain.toLowerCase();
  return AMAZON_DOMAINS.some(
    (blocked) => normalized === blocked || normalized.endsWith(`.${blocked}`),
  );
}
```

---

## TypeScript File Organization

**types.ts** (root level):

```typescript
// Export all interfaces and types
export type {
  CrawlerConfiguration,
  TargetConfiguration,
  SelectorConfiguration,
  BookPromotion,
  CrawlTarget,
  RobotsTxtRule,
  CrawlResult,
  LogEntry,
};

// Export constants
export { AMAZON_DOMAINS, isAmazonDomain };
```

**Usage in modules:**

```typescript
import type { CrawlerConfiguration, BookPromotion } from "../types.ts";
import { isAmazonDomain } from "../types.ts";
```

---

## Validation Strategy

All entities are validated at configuration load time (`src/config/loader.ts`):

1. **Parse JSON**: `JSON.parse()` with error handling
2. **Schema validation**: Check required fields, types, ranges
3. **Domain validation**: Check Amazon blocklist for all target URLs
4. **Selector validation**: Ensure all required selectors present
5. **Early exit**: Refuse to proceed if configuration is invalid

No runtime validation needed if configuration is pre-validated.

---

## Constitution Alignment

### Principle I (Legal/Ethical Scraping)

- `RobotsTxtRule` entity enforces robots.txt checking
- `CrawlTarget.robotsStatus` tracks compliance per URL

### Principle II (Amazon ToS Boundary)

- `AMAZON_DOMAINS` constant provides hardcoded blocklist
- `isAmazonDomain()` function prevents accidental navigation
- `BookPromotion.amazonProductUrl` stored but never validated/opened

### Principle III (Transparency)

- `CrawlerConfiguration.userAgent` enforces consistent identification

### Principle IV (Conservative Crawling)

- `CrawlerConfiguration.delay` enforces minimum 2-second delays
- `CrawlTarget` tracks state to prevent retries

### Principle V (Deno-First)

- All types are pure TypeScript (no Node.js types)
- Deno native APIs used for I/O (Deno.readTextFile, Deno.writeTextFile)

### Principle VI (Maintainability)

- Clear entity boundaries with single responsibilities
- Explicit validation rules documented per entity
- Type safety enforced via TypeScript interfaces

---

## Next Steps

1. Implement JSON schema validation in `src/config/loader.ts`
2. Create TypeScript types file at root (`types.ts`)
3. Generate contracts for configuration file format (Phase 1)
4. Implement entity constructors/factories as needed
