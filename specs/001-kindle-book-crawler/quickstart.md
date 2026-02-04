# Quickstart Guide: Kindle Book Crawler

**Feature**: 001-kindle-book-crawler  
**Date**: 2026-02-04

## Overview

The Kindle Book Crawler is a Deno-based CLI tool that collects free Kindle book promotion data from third-party websites. It respects robots.txt, blocks Amazon domains, and outputs structured JSON data.

## Prerequisites

### Required

- **Deno 1.40+**: [Install Deno](https://deno.land/#installation)
- **npx** (for Playwright browser installation): Comes with Node.js/npm

### System Requirements

- **OS**: Windows, macOS, or Linux
- **Disk Space**: ~500MB (for Playwright browser binaries)
- **Network**: Internet access for crawling and robots.txt fetching

---

## Installation

### 1. Clone the Repository

```bash
git clone https://github.com/USER/kindle.git
cd kindle
```

### 2. Configure Deno for Playwright

The project requires `node_modules` for Playwright browser binaries.

**deno.json** (already configured in repo):

```json
{
  "nodeModulesDir": "auto",
  "imports": {
    "playwright": "npm:playwright@^1.40.0",
    "robots-parser": "npm:robots-parser@^3.0.1"
  }
}
```

### 3. Install Playwright Browsers (One-Time Setup)

```bash
deno run --allow-run=npx --allow-read --allow-write --allow-net --allow-env npm:playwright install chromium
```

This downloads the Chromium browser binary (~150MB) to `node_modules`.

**Note**: You only need Chromium for this project. Playwright also supports Firefox and WebKit, but we default to Chromium.

---

## Configuration

### Create a Configuration File

Create a `config.json` file with your crawl targets and CSS selectors:

```json
{
  "targets": [
    {
      "url": "https://example.com/free-kindle-books",
      "selectors": {
        "title": ".book-title",
        "author": ".author-name",
        "amazonLink": "a.amazon-button"
      }
    }
  ],
  "delay": 3,
  "timeout": 30,
  "outputFile": "results.json"
}
```

### Configuration Options

| Field        | Type   | Required | Default                        | Description                              |
| ------------ | ------ | -------- | ------------------------------ | ---------------------------------------- |
| `targets`    | array  | ✅ Yes   | -                              | List of URLs to crawl with CSS selectors |
| `delay`      | number | No       | 2                              | Seconds between navigations (min 2)      |
| `timeout`    | number | No       | 30                             | Page operation timeout in seconds        |
| `userAgent`  | string | No       | `KindleBookCrawler/1.0 (+URL)` | Custom User-Agent header                 |
| `outputFile` | string | No       | stdout                         | File path for JSON output                |

### Target Configuration

Each target requires:

- `url`: Full HTTP/HTTPS URL (non-Amazon domain)
- `selectors.title`: CSS selector for book title (required)
- `selectors.author`: CSS selector for author name (optional)
- `selectors.amazonLink`: CSS selector for Amazon product link (required)

### Finding CSS Selectors

Use browser DevTools to identify selectors:

1. Open the target website in a browser
2. Right-click on a book title → "Inspect Element"
3. Note the CSS class or ID (e.g., `.book-title`, `#title-123`)
4. Repeat for author and Amazon link elements
5. Test selectors in the browser console: `document.querySelector(".book-title")`

---

## Usage

### Basic Usage (stdout output)

```bash
deno run --allow-read --allow-write --allow-net --allow-env --allow-run src/main.ts config.json
```

Output will be printed to stdout as JSON.

### Save to File

```bash
# Option 1: Specify in config.json
{
  "outputFile": "results.json",
  ...
}

# Option 2: Redirect stdout
deno run --allow-read --allow-write --allow-net --allow-env --allow-run src/main.ts config.json > results.json
```

### Using a Deno Task (Recommended)

**deno.json** includes a pre-configured task:

```bash
deno task crawl config.json
```

This is equivalent to the full command above with all permissions.

---

## Understanding Permissions

Deno requires explicit permissions for security. Here's what each permission does:

| Permission      | Why Required                                                 |
| --------------- | ------------------------------------------------------------ |
| `--allow-read`  | Read `config.json`, browser binaries, `node_modules`         |
| `--allow-write` | Write output file (if `outputFile` specified), browser cache |
| `--allow-net`   | Fetch robots.txt, navigate to crawl targets                  |
| `--allow-env`   | Playwright reads environment variables (HOME, PATH, etc.)    |
| `--allow-run`   | Spawn Chromium browser process (Playwright requirement)      |

**Security Note**: `--allow-run` effectively bypasses Deno's sandbox for spawned processes. Only run configurations you trust.

---

## Example Workflow

### 1. Create Config

**config.json**:

```json
{
  "targets": [
    {
      "url": "https://freekindlebooks.example.com/today",
      "selectors": {
        "title": "h2.book-title",
        "author": ".author",
        "amazonLink": "a[href*='amazon.com']"
      }
    }
  ],
  "delay": 2,
  "outputFile": "kindle-books.json"
}
```

### 2. Run Crawler

```bash
deno task crawl config.json
```

### 3. Check Output

**kindle-books.json**:

```json
{
  "books": [
    {
      "title": "The Great Gatsby",
      "author": "F. Scott Fitzgerald",
      "sourceUrl": "https://freekindlebooks.example.com/today",
      "amazonProductUrl": "https://www.amazon.com/dp/B000FC1GHO",
      "crawlTimestamp": "2026-02-04T10:30:00.000Z"
    }
  ],
  "summary": {
    "total": 1,
    "successful": 1,
    "skipped": 0,
    "failed": 0
  }
}
```

---

## Troubleshooting

### Browser Binary Not Found

**Error**: `browserType.launch: Executable doesn't exist at ...`

**Solution**:

```bash
deno run --allow-run=npx --allow-read --allow-write --allow-net --allow-env npm:playwright install chromium
```

### robots.txt Disallows Crawling

**Error**: `Disallowed by robots.txt: https://example.com/page`

**Solution**: This is expected behavior. The crawler respects robots.txt per the project constitution. Choose different URLs that allow crawling.

### Amazon Domain Blocked

**Error**: `Amazon domain blocked: https://amazon.com/...`

**Solution**: This is intentional. The crawler never navigates to Amazon domains per the constitution. Ensure target URLs point to third-party sites only.

### Permission Denied

**Error**: `Requires read access to "config.json", run again with the --allow-read flag`

**Solution**: Use `deno task crawl config.json` or add the missing permission flag.

### Timeout Errors

**Error**: `Navigation timeout of 30000ms exceeded`

**Solution**: Increase `timeout` in `config.json` or check if the target site is slow/unavailable.

---

## Next Steps

- **Read the Implementation Plan**: [plan.md](plan.md) - Detailed technical design
- **Understand the Data Model**: [data-model.md](data-model.md) - TypeScript interfaces
- **See Configuration Schema**: [contracts/config.schema.json](contracts/config.schema.json) - JSON validation
- **View Output Schema**: [contracts/output.schema.json](contracts/output.schema.json) - Result format

---

## Constitution Compliance

This tool adheres to strict ethical guidelines:

✅ **Legal Scraping**: robots.txt checked and enforced before all navigation  
✅ **Amazon ToS**: No automated access to Amazon domains  
✅ **Transparency**: Consistent User-Agent, no stealth techniques  
✅ **Conservative Crawling**: Sequential only, explicit delays, no retries  
✅ **Deno-First**: Native Deno APIs, no Node.js patterns  
✅ **Maintainable**: Clear separation of concerns, fail-safe defaults

See [.specify/memory/constitution.md](../../.specify/memory/constitution.md) for full principles.

---

## Support

For issues, feature requests, or questions:

- **GitHub Issues**: https://github.com/USER/kindle/issues
- **Documentation**: [specs/001-kindle-book-crawler/](.)

**Remember**: This tool is for educational and research purposes. Always respect website terms of service and robots.txt directives.
