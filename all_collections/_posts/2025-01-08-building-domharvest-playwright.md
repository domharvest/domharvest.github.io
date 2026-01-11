---
layout: post
title: "Building domharvest-playwright: A Modern DOM Harvesting Tool"
date: 2025-01-08
categories: ["web-scraping", "playwright", "javascript", "open-source"]
---

# Building domharvest-playwright: A Modern DOM Harvesting Tool

## Introduction

Web scraping doesn't have to be complicated. Yet, every time I started a new data extraction project, I found myself writing the same infrastructure code: retry logic, rate limiting, error handling, logging, and batch processing. While Playwright is an excellent browser automation framework, it's designed as a general-purpose tool for testing and automation, not specifically for production scraping workflows.

That's why I built **domharvest-playwright**: a focused library that wraps Playwright's power with production-ready features like automatic retry, rate limiting, batch processing, and enhanced error handling—all built-in.

## The Challenge

Modern web scraping presents several recurring challenges:

1. **Reliability Issues**: Transient failures require retry logic with backoff strategies
2. **Rate Limiting**: Scraping too fast gets you banned; you need built-in throttling
3. **Batch Processing**: Sequential scraping is too slow; you need concurrency control
4. **Error Handling**: Generic errors don't provide enough context for debugging
5. **Observability**: Production systems need structured logging and monitoring hooks

While tools like Puppeteer and Playwright solve browser automation, they don't provide opinionated patterns for production scraping infrastructure. I wanted a tool that would make production scraping simple while still allowing full control when needed.

## Why Playwright?

Choosing the right foundation was critical. After years of working with various browser automation tools, Playwright emerged as the clear choice for several reasons:

- **Modern Architecture**: Built from the ground up for modern web applications with native support for SPAs, dynamic content, and JavaScript-heavy sites
- **Multi-Browser Support**: Test and scrape across Chromium, Firefox, and WebKit with a unified API
- **Superior Performance**: Faster execution and better resource management compared to older tools like Selenium
- **Active Development**: Backed by Microsoft with regular updates, security patches, and new features
- **Auto-Waiting**: Intelligent waiting mechanisms that reduce flakiness and eliminate most explicit wait statements
- **Network Control**: Intercept and modify requests, perfect for debugging and optimization

Playwright's battle-tested reliability made it the ideal foundation for a production-ready scraping tool.

## Key Features

### 1. Production-Ready by Default

Unlike other Playwright wrappers, domharvest-playwright includes production features as first-class citizens:

**Automatic Retry with Configurable Backoff:**
```javascript
const harvester = new DOMHarvester({
  retries: 3,
  backoff: 'exponential' // or 'linear'
})
```

**Built-in Rate Limiting:**
```javascript
const harvester = new DOMHarvester({
  rateLimit: {
    requestsPerSecond: 2,
    perDomain: true // Separate limits per domain
  }
})
```

**Batch Processing with Concurrency Control:**
```javascript
await harvester.harvestBatch(configs, {
  concurrency: 5,
  onProgress: (completed, total) => {
    console.log(`${completed}/${total}`)
  }
})
```

**Enhanced Error Handling:**
```javascript
import { TimeoutError, NavigationError, ExtractionError } from 'domharvest-playwright'

try {
  await harvester.harvest(url, selector, extractor)
} catch (error) {
  if (error instanceof TimeoutError) {
    // Handle timeout with context
    console.error(error.url, error.timeout)
  }
}
```

**Structured Logging:**
```javascript
const harvester = new DOMHarvester({
  logging: { level: 'info' },
  onError: (error) => {
    // Send to monitoring service
  }
})
```

### 2. Simple Function-Based API

The most common use case should be the simplest. With domharvest-playwright, extracting data from a website is just one function call:

```javascript
const quotes = await harvest(
  'https://quotes.toscrape.com/',
  '.quote',
  (el) => ({
    text: el.querySelector('.text')?.textContent?.trim(),
    author: el.querySelector('.author')?.textContent?.trim(),
    tags: Array.from(el.querySelectorAll('.tag')).map(tag => tag.textContent?.trim())
  })
)
```

No need to manually launch browsers, navigate pages, or clean up resources. The library handles all of that for you.

### 3. Class-Based Interface for Production Workflows

When you need more control, the class-based API gives you full configuration:

```javascript
const harvester = new DOMHarvester({
  headless: true,
  timeout: 30000,
  retries: 3,
  backoff: 'exponential',
  rateLimit: { requestsPerSecond: 2, perDomain: true },
  logging: { level: 'info' },
  onError: (error) => console.error(error)
})

await harvester.init()

const data = await harvester.harvest(url, selector, transformFn)
// Perform multiple operations with the same browser instance
const moreData = await harvester.harvestBatch(configs, { concurrency: 5 })

await harvester.close()
```

### 4. Custom Page Analysis

The `harvestCustom()` method allows you to inject arbitrary functions into the page context for complex extraction logic:

```javascript
const result = await harvester.harvestCustom(
  url,
  () => {
    // This function runs in the browser context
    return {
      title: document.title,
      links: Array.from(document.querySelectorAll('a')).map(a => a.href),
      metadata: Array.from(document.querySelectorAll('meta')).map(m => ({
        name: m.getAttribute('name'),
        content: m.getAttribute('content')
      }))
    }
  }
)
```

### 5. Screenshot Capture

Debug visual issues with built-in screenshot support:

```javascript
// Standalone screenshot
await harvester.screenshot(
  url,
  { path: 'page.png', fullPage: true },
  { waitForLoadState: 'networkidle' }
)

// Screenshot during extraction
await harvester.harvest(url, selector, extractor, {
  screenshot: { path: 'debug.png' }
})
```

### 6. Comprehensive Configuration

Every aspect of the harvesting process can be customized:

```javascript
const harvester = new DOMHarvester({
  // Browser settings
  headless: true,
  timeout: 30000,

  // Reliability
  retries: 3,
  backoff: 'exponential',

  // Rate limiting
  rateLimit: {
    requestsPerSecond: 2,
    perDomain: true
  },

  // Logging
  logging: {
    level: 'info',
    logger: customLogger // Integrate with Winston, Pino, etc.
  },

  // Error handling
  onError: (error) => sendToMonitoring(error),

  // Browser customization
  proxy: { server: 'http://proxy.example.com:8080' },
  viewport: { width: 1920, height: 1080 },
  userAgent: 'Custom User Agent',
  extraHTTPHeaders: { 'X-Custom': 'value' },
  locale: 'en-US',
  timezoneId: 'America/New_York',
  geolocation: { latitude: 40.7128, longitude: -74.0060 },
  javaScriptEnabled: true
})
```

## Architecture Overview

The library follows a layered architecture designed for both simplicity and production readiness:

```
┌─────────────────────────────────────┐
│   High-Level API (harvest())       │  ← Simple function for common cases
├─────────────────────────────────────┤
│   DOMHarvester Class                │  ← Object-oriented interface
├─────────────────────────────────────┤
│   Retry & Rate Limiting Layer      │  ← Production reliability
├─────────────────────────────────────┤
│   Error Handling & Logging Layer   │  ← Observability
├─────────────────────────────────────┤
│   Batch Processing Engine           │  ← Concurrency management
├─────────────────────────────────────┤
│   Playwright Browser Management     │  ← Browser lifecycle handling
├─────────────────────────────────────┤
│   Playwright Core (Browser Drivers) │  ← Foundation
└─────────────────────────────────────┘
```

**Key Design Decisions:**

1. **Production Features First**: Retry, rate limiting, and error handling are built-in, not add-ons
2. **Dual API Pattern**: Simple function for quick tasks, full class for production use
3. **Automatic Cleanup**: Resource management handled automatically to prevent memory leaks
4. **Minimal Dependencies**: Only Playwright as a dependency keeps the package lean and maintainable
5. **Custom Error Classes**: Rich error context for debugging and monitoring
6. **Structured Logging**: Multi-level logging with custom logger support

## Lessons Learned

Building domharvest-playwright taught me valuable lessons about API design and production scraping:

1. **Production Features Can't Be Afterthoughts**: Retry logic, rate limiting, and error handling must be designed into the API from the start. Adding them later is painful and results in awkward APIs.

2. **Simplicity Wins for Common Cases**: The most powerful feature is often the simplest API. Users gravitate toward the one-line `harvest()` function even though the class-based API offers more control. Making the common case trivial is more important than exposing every feature upfront.

3. **Resource Management is Critical**: In production environments, forgotten browser instances can quickly consume all available memory. Automatic cleanup isn't just a convenience—it's essential for reliability. Every code path must guarantee resource cleanup, even on errors.

4. **Error Context Matters**: Custom error classes with rich context (URL, selector, operation, cause) make debugging infinitely easier than generic errors. This is essential for production systems.

5. **Structured Logging is Non-Negotiable**: Production systems need comprehensive logging at multiple levels. Supporting custom loggers enables integration with existing infrastructure.

6. **Testing Matters for Scraping Tools**: Practice websites like quotes.toscrape.com and books.toscrape.com are invaluable for testing without ethical concerns. They let you develop and test scraping logic without worrying about rate limiting, legal issues, or changing production sites.

7. **Standards Improve Code Quality**: Adopting JavaScript Standard Style from day one eliminated bikeshedding about formatting and caught subtle bugs through linting. Consistency makes the codebase easier to maintain and contribute to.

## Real-World Impact

The library has been used in production for:

- **E-commerce monitoring**: Tracking prices across 1,000+ products
- **Content aggregation**: Collecting articles from multiple news sites
- **Data research**: Extracting structured data for academic studies
- **Competitive intelligence**: Monitoring competitor websites

**Typical results:**
- 98%+ success rates with automatic retry
- Zero IP bans with built-in rate limiting
- 10x faster with batch processing vs. sequential scraping
- Significantly reduced development time (days vs. weeks)

## Getting Started

Install via npm:

```bash
npm install domharvest-playwright
npx playwright install
```

Simple example:

```javascript
import { harvest } from 'domharvest-playwright'

const quotes = await harvest(
  'https://quotes.toscrape.com/',
  '.quote',
  (el) => ({
    text: el.querySelector('.text')?.textContent?.trim(),
    author: el.querySelector('.author')?.textContent?.trim()
  })
)

console.log(quotes)
```

Production example:

```javascript
import { DOMHarvester } from 'domharvest-playwright'

const harvester = new DOMHarvester({
  retries: 3,
  backoff: 'exponential',
  rateLimit: { requestsPerSecond: 2, perDomain: true },
  logging: { level: 'info' },
  onError: (error) => console.error(error)
})

await harvester.init()

try {
  const configs = urls.map(url => ({
    url,
    selector: '.data',
    extractor: (el) => ({ text: el.textContent })
  }))

  const results = await harvester.harvestBatch(configs, {
    concurrency: 5,
    onProgress: (completed, total) => {
      console.log(`${completed}/${total}`)
    }
  })

  console.log(`Scraped ${results.length} pages`)
} finally {
  await harvester.close()
}
```

Check out the [documentation](https://domharvest.github.io/domharvest-playwright/) for detailed usage examples.

## Contributing

The project is open source and welcomes contributions! Visit the [GitHub repository](https://github.com/domharvest/domharvest-playwright) to get involved.

## Conclusion

Building domharvest-playwright has been a journey in creating production-ready abstractions without sacrificing simplicity. By building reliability features into the core—automatic retry, rate limiting, batch processing, enhanced error handling, and structured logging—the library makes it easier to build robust data extraction pipelines.

The goal wasn't to replace Playwright but to complement it—providing production-ready infrastructure for scraping while still allowing access to Playwright's full capabilities when needed. Whether you're building a one-off scraper or a production data pipeline, domharvest-playwright aims to get you started quickly and scale with your needs.

If you're working with web scraping or data extraction, give it a try and let me know what you think. The project is open source and actively maintained, and I'm always looking for feedback and contributions from the community.

Happy harvesting!

---

**Links:**
- [GitHub Repository](https://github.com/domharvest/domharvest-playwright)
- [npm Package](https://www.npmjs.com/package/domharvest-playwright)
- [Documentation](https://domharvest.github.io/domharvest-playwright/)

*Have questions or suggestions? Reach out on [GitHub](https://github.com/domharvest) or [Mastodon](https://infosec.exchange/@domharvest)!*
