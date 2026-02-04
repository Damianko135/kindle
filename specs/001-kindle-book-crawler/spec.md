# Feature Specification: Kindle Book Crawler

**Feature Branch**: `001-kindle-book-crawler`  
**Created**: 2026-02-04  
**Status**: Draft  
**Input**: User description: "Build a Deno-based web crawler using Playwright that collects information about free Kindle book promotions from third-party websites."

## Clarifications

### Session 2026-02-04

- Q: How should users provide URLs to the crawler? → A: Configuration file only (e.g., `config.json` with URL list)
- Q: How should the crawler identify and extract book data from varying HTML structures across different third-party websites? → A: Configuration-based CSS selectors per domain (users specify selectors in config)
- Q: What format should the User-Agent header use to identify the crawler? → A: `KindleBookCrawler/1.0 (+https://github.com/USER/kindle)` (standard format with contact)
- Q: Where should the crawler write its JSON output? → A: Configurable: stdout by default, optional file path in config
- Q: What timeout should apply to page navigation and data extraction? → A: 30 seconds per page

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Crawl Single Website for Free Kindle Books (Priority: P1)

A developer wants to collect information about free Kindle book promotions from a specific third-party website. They create a configuration file specifying the target URL, invoke the crawler, which validates robots.txt compliance, extracts book data (title, author, Amazon link), and outputs structured JSON. This delivers immediate value as a standalone tool for manual data collection.

**Why this priority**: This is the core MVP. A single-site crawler that respects robots.txt and extracts book data provides immediate value and proves all foundational components work (robots.txt checking, page navigation, data extraction, JSON output).

**Independent Test**: Can be fully tested by providing a configuration file with one URL and its CSS selectors, verifying robots.txt is checked, and confirming JSON output contains book title, author, source URL, and Amazon link.

**Acceptance Scenarios**:

1. **Given** a configuration file containing a valid URL to a third-party site listing free Kindle books, **When** the crawler is invoked, **Then** robots.txt is fetched and checked, the page is crawled if allowed, and JSON output contains all extracted book data
2. **Given** a URL where robots.txt disallows crawling, **When** the crawler is invoked, **Then** no navigation occurs, and a log message explains the URL was skipped due to robots.txt restrictions
3. **Given** a URL to an Amazon domain, **When** the crawler is invoked, **Then** the crawler immediately rejects it with a log message explaining Amazon domains are forbidden
4. **Given** a page with 10 free Kindle book listings, **When** the crawler processes it, **Then** JSON output contains exactly 10 book records with title, author (if present), source URL, and Amazon product link

---

### User Story 2 - Crawl Multiple Pages Sequentially (Priority: P2)

A developer wants to crawl multiple pages from the same or different third-party websites in a single run. They specify multiple URLs in a configuration file. The crawler processes URLs sequentially with explicit delays between requests, respects per-domain robots.txt rules, and aggregates all results into a single JSON output.

**Why this priority**: Extends the MVP to handle realistic batch collection scenarios. This enables users to gather data from multiple sources without running the crawler repeatedly.

**Independent Test**: Can be fully tested by providing a configuration file with 3-5 URLs from different domains (each with CSS selectors), verifying each domain's robots.txt is checked independently, delays are enforced between navigations, and JSON output contains aggregated results from all allowed URLs.

**Acceptance Scenarios**:

1. **Given** a list of 5 URLs from 3 different domains, **When** the crawler runs, **Then** robots.txt is fetched once per unique domain, each allowed URL is crawled sequentially with configured delays, and JSON contains books from all successful crawls
2. **Given** a list where 2 URLs are disallowed by robots.txt, **When** the crawler runs, **Then** those 2 URLs are skipped with log messages, remaining URLs are crawled, and JSON contains only data from allowed URLs
3. **Given** a configured delay of 3 seconds between requests, **When** crawling 3 URLs, **Then** the total runtime is at least 6 seconds (2 delays between 3 requests)
4. **Given** a URL that returns HTTP 403, **When** the crawler encounters it, **Then** it logs the error, skips that URL without retrying, and continues processing remaining URLs

---

### User Story 3 - Handle Failures Gracefully (Priority: P3)

The crawler encounters various failure scenarios (network errors, missing data, malformed pages) and handles them without crashing. Each failure is logged with specific details (URL, error type, reason), and the crawler continues processing remaining URLs.

**Why this priority**: Essential for production reliability. Users need to understand why specific URLs failed without losing data from successful crawls.

**Independent Test**: Can be fully tested by providing a mix of valid URLs, URLs with network issues, and URLs with missing book data, verifying the crawler completes without crashing and produces logs explaining each failure.

**Acceptance Scenarios**:

1. **Given** a page that times out during navigation, **When** the crawler attempts to access it, **Then** the timeout is logged, the URL is skipped, and the crawler continues with the next URL
2. **Given** a page with a book listing missing the title field, **When** the crawler extracts data, **Then** that book is logged as having incomplete data, skipped from output, and extraction continues for remaining books on the page
3. **Given** a domain where robots.txt is unreachable (network error), **When** the crawler checks robots.txt, **Then** the entire domain is marked as disallowed, logged with the network error, and all URLs from that domain are skipped
4. **Given** a page that triggers Playwright CAPTCHA detection, **When** the crawler encounters it, **Then** the URL is immediately skipped with a log message explaining bot detection was encountered

---

### Edge Cases

- What happens when robots.txt is partially malformed but includes a Disallow rule?
- How does the system handle a page with 0 book listings (empty result)?
- What happens when the same book appears on multiple crawled pages?
- How does the system handle extremely large pages (1000+ books on one page)?
- What happens when a domain's robots.txt is valid but returns HTTP 403 on the actual page?
- How does the system handle redirects (301/302) to Amazon domains?
- What happens when the Amazon product URL is relative instead of absolute?

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST fetch and parse robots.txt using Deno's native fetch API before attempting to crawl any URL
- **FR-002**: System MUST refuse to crawl any URL if robots.txt is unavailable, unreachable, or explicitly disallows access
- **FR-003**: System MUST reject all URLs pointing to Amazon domains (amazon.com, amazon.*, a.co, amzn.*) without navigation
- **FR-004**: System MUST extract book title, author name (if available), source page URL, and Amazon product URL using CSS selectors specified in the configuration file
- **FR-005**: System MUST store Amazon product URLs in output without opening, validating, or navigating to them
- **FR-006**: System MUST crawl pages sequentially (single-page concurrency) with configurable delays between navigations (default: 2 seconds minimum)
- **FR-007**: System MUST stop immediately when encountering HTTP 403, 429, CAPTCHA, or bot-detection signals and log the reason
- **FR-008**: System MUST skip failed URLs without retrying and continue processing remaining URLs
- **FR-009**: System MUST output structured JSON containing all successfully extracted book data to stdout by default, with optional file output if specified in configuration
- **FR-010**: System MUST log each skipped URL with the specific reason (robots.txt violation, disallowed domain, HTTP error, etc.)
- **FR-011**: System MUST use Deno's native APIs (fetch, filesystem, permissions) and avoid Node.js-specific modules
- **FR-012**: System MUST identify itself with a User-Agent header in the format `KindleBookCrawler/VERSION (+PROJECT_URL)` where VERSION is the crawler version and PROJECT_URL is the project repository or contact URL
- **FR-013**: System MUST NOT attempt authentication, login, or access paywalled content
- **FR-014**: System MUST validate that target URLs are publicly accessible non-Amazon websites
- **FR-015**: System MUST read target URLs from a configuration file (JSON format)
- **FR-016**: System MUST apply a configurable timeout (default 30 seconds) to page navigation and data extraction operations

### Key Entities

- **Book Promotion**: Represents a free Kindle book offer discovered on a third-party site
  - Attributes: title (string, required), author (string, optional), source URL (string, required), Amazon product URL (string, required), crawl timestamp (datetime, required)
  
- **Crawl Target**: Represents a URL to be crawled
  - Attributes: URL (string, required), domain (string, required), robots.txt status (allowed/disallowed/unavailable), crawl status (pending/success/skipped/failed), failure reason (string, optional)

- **robots.txt Rule**: Represents parsed robots.txt directives for a domain
  - Attributes: domain (string, required), user-agent (string, required), allowed paths (array of patterns), disallowed paths (array of patterns), fetch timestamp (datetime, required)
- **Crawler Configuration**: Represents the input configuration file
  - Attributes: targets (array of Target objects, required), delay (number in seconds, optional, default 2), timeout (number in seconds, optional, default 30), user-agent (string, optional, default format: `KindleBookCrawler/VERSION (+PROJECT_URL)`), output-file (string, optional, if omitted output goes to stdout)

- **Target Configuration**: Represents a crawl target with selectors
  - Attributes: url (string, required), selectors (object with title/author/amazonLink CSS selectors, required), domain (string, derived from URL)


## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Crawler successfully extracts book data from at least 90% of robots.txt-allowed URLs with valid HTML structure
- **SC-002**: robots.txt is fetched and evaluated within 2 seconds for any domain before page navigation begins
- **SC-003**: Zero automated navigations to Amazon domains occur during any crawl operation
- **SC-004**: Crawler processes 10 URLs with at least 2 seconds delay between each navigation (total runtime ≥ 18 seconds for sequential processing)
- **SC-005**: Failed URLs (HTTP errors, timeouts, missing data) are logged with specific reasons and do not cause crawler termination
- **SC-006**: JSON output from a successful crawl is parseable and contains all required fields (title, source URL, Amazon link) for 100% of extracted books
- **SC-007**: Crawler completes execution and outputs results even when 50% of input URLs are disallowed by robots.txt or fail

## Assumptions *(optional)*

- Third-party websites have HTML structures that can be described with CSS selectors
- Book titles and Amazon links are always present in the HTML; author names may be optional
- robots.txt files are hosted at standard locations (domain root: /robots.txt)
- Deno runtime version 1.40+ is available with necessary permissions (--allow-net, --allow-read, --allow-write)
- Playwright for Deno is available and compatible with the target Deno version
- Users will provide a JSON configuration file containing target URLs and optional settings
- Configuration file path is provided as a command-line argument (e.g., `deno run main.ts config.json`)
- Default delay between navigations is 2 seconds but can be configured in the configuration file
- Default timeout for page operations is 30 seconds but can be configured in the configuration file
- Default User-Agent follows RFC 7231 format with project URL (configurable in config file)

## Out of Scope *(optional)*

- Price verification on Amazon (Amazon URLs are stored but never accessed)
- CAPTCHA solving or anti-bot circumvention techniques
- Authentication, login, or access to paywalled/premium content
- Real-time monitoring or scheduled crawling (single execution per invocation)
- Database persistence (JSON output only; users handle storage)
- Retry logic for failed requests (fail-fast approach)
- Proxy support or IP rotation
- Browser fingerprint evasion or stealth techniques
- Parallel/concurrent crawling of multiple domains
- HTML parsing fallback strategies for JavaScript-heavy sites
- Node.js compatibility or CommonJS module support
