---
layout: post
title: "Inside domharvest-playwright: Architecture, Design Patterns, and Advanced Techniques"
date: 2025-01-06
categories: ["web-scraping", "playwright", "architecture", "javascript", "deep-dive"]
---

# Inside domharvest-playwright: Architecture, Design Patterns, and Advanced Techniques

While many developers use domharvest-playwright for its simple API, there's a sophisticated architecture underneath that makes it both powerful and production-ready. In this deep dive, we'll explore the design decisions, patterns, and advanced techniques that differentiate domharvest-playwright from a simple Playwright wrapper.

This article is for developers who want to understand not just *how* to use the library, but *why* it works the way it does and what makes it production-ready.

## Architectural Overview

domharvest-playwright follows a layered architecture that progressively builds abstractions over Playwright's browser automation:

```
┌─────────────────────────────────────────┐
│   High-Level API (harvest function)    │  ← Simple convenience layer
├─────────────────────────────────────────┤
│   DOMHarvester Class                    │  ← Full-featured interface
├─────────────────────────────────────────┤
│   Retry & Rate Limiting Layer          │  ← Production reliability
├─────────────────────────────────────────┤
│   Error Handling & Logging Layer       │  ← Observability
├─────────────────────────────────────────┤
│   Batch Processing Engine               │  ← Concurrency management
├─────────────────────────────────────────┤
│   Playwright Core (Browser Drivers)     │  ← Foundation
└─────────────────────────────────────────┘
```

This layered design follows the principle of **progressive disclosure**: simple tasks use simple APIs, complex tasks have access to deeper layers.

## Design Philosophy

### 1. Production-Ready by Default

Unlike many scraping libraries that require you to build reliability features yourself, domharvest-playwright includes them as first-class features:

**Automatic Retry with Backoff**
- Configurable retry counts
- Exponential or linear backoff strategies
- Per-request retry overrides

**Built-in Rate Limiting**
- Global or per-domain throttling
- Prevents overwhelming target servers
- Automatic request queueing

**Enhanced Error Handling**
- Custom error classes with context
- Error callbacks for monitoring integration
- Graceful degradation

**Structured Logging**
- Multiple log levels (debug, info, warn, error)
- Custom logger support
- Production-ready observability

### 2. Dual API Pattern

The library provides two complementary interfaces:

**Function-Based API** (`harvest()`):
```javascript
const data = await harvest(url, selector, extractor)
```

**Class-Based API** (`DOMHarvester`):
```javascript
const harvester = new DOMHarvester(options)
await harvester.init()
const data = await harvester.harvest(url, selector, extractor)
await harvester.close()
```

This pattern serves different use cases:
- Function API: Optimized for one-time operations, automatic cleanup
- Class API: Optimized for reuse, explicit lifecycle management, full configuration

## Core Features Deep Dive

### Retry Logic with Backoff Strategies

The retry mechanism is sophisticated and configurable:

```javascript
const harvester = new DOMHarvester({
  retries: 3,
  backoff: 'exponential' // or 'linear'
})
```

**How it works:**

1. **Exponential Backoff**: Delays increase exponentially (1s, 2s, 4s, 8s, ...)
   - Ideal for transient failures
   - Gives target servers time to recover
   - Industry-standard approach

2. **Linear Backoff**: Delays increase linearly (1s, 2s, 3s, 4s, ...)
   - More predictable timing
   - Good for rate-limit recovery

**Per-request overrides:**
```javascript
await harvester.harvest(url, selector, extractor, {
  retries: 5 // Override global setting for this request
})
```

### Rate Limiting Engine

The built-in rate limiter prevents overwhelming target servers:

```javascript
const harvester = new DOMHarvester({
  rateLimit: {
    requestsPerSecond: 2,
    perDomain: true // Separate limits per domain
  }
})
```

**Implementation details:**

- **Token bucket algorithm**: Ensures consistent request rates
- **Per-domain tracking**: When `perDomain: true`, each domain gets its own rate limit
- **Automatic queueing**: Requests are queued and released at the configured rate
- **No manual delays needed**: The library handles all throttling

**Why this matters:**
- Prevents IP bans
- Respects server resources
- Enables polite scraping at scale

### Batch Processing with Concurrency Control

The `harvestBatch()` method processes multiple URLs efficiently:

```javascript
const configs = [
  { url: 'https://example.com/1', selector: '.data', extractor: fn1 },
  { url: 'https://example.com/2', selector: '.data', extractor: fn2 },
  { url: 'https://example.com/3', selector: '.data', extractor: fn3 }
]

const results = await harvester.harvestBatch(configs, {
  concurrency: 3,
  onProgress: (completed, total) => {
    console.log(`${completed}/${total} complete`)
  }
})
```

**Key capabilities:**

1. **Configurable concurrency**: Control how many requests run in parallel
2. **Progress callbacks**: Monitor progress in real-time
3. **Combined with rate limiting**: Concurrency + rate limiting work together
4. **Error isolation**: One failed URL doesn't affect others

**Architecture:**

```
Request Queue → Rate Limiter → Concurrency Pool → Browser
     [100 URLs]     [2 req/s]      [3 parallel]     [Playwright]
```

Each layer enforces its constraints independently.

### Enhanced Error Handling

Custom error classes provide rich context:

```javascript
import { TimeoutError, NavigationError, ExtractionError } from 'domharvest-playwright'

try {
  await harvester.harvest(url, selector, extractor)
} catch (error) {
  if (error instanceof TimeoutError) {
    console.log('Timeout:', error.url, error.timeout)
  } else if (error instanceof NavigationError) {
    console.log('Navigation failed:', error.url, error.cause)
  } else if (error instanceof ExtractionError) {
    console.log('Extraction failed:', error.selector, error.operation)
  }
}
```

**Error properties:**
- `url`: The URL being processed
- `selector`: The CSS selector (for ExtractionError)
- `operation`: What operation failed
- `cause`: Original error for debugging

**Error callbacks for monitoring:**
```javascript
const harvester = new DOMHarvester({
  onError: (error) => {
    // Send to monitoring service
    logger.error('Scraping error', {
      url: error.url,
      type: error.constructor.name,
      message: error.message
    })
  }
})
```

### Structured Logging System

Multi-level logging for different environments:

```javascript
const harvester = new DOMHarvester({
  logging: {
    level: 'debug' // 'debug', 'info', 'warn', 'error'
  }
})
```

**Log levels:**
- **debug**: Verbose output (navigation, selectors, timing)
- **info**: Important events (retries, rate limiting)
- **warn**: Potential issues (slow pages, partial failures)
- **error**: Failures (timeouts, navigation errors)

**Custom loggers:**
```javascript
const harvester = new DOMHarvester({
  logging: {
    level: 'info',
    logger: {
      debug: (msg) => winston.debug(msg),
      info: (msg) => winston.info(msg),
      warn: (msg) => winston.warn(msg),
      error: (msg) => winston.error(msg)
    }
  }
})
```

This enables integration with existing logging infrastructure.

## Advanced Techniques

### Technique 1: Custom Page Functions with harvestCustom()

For complex extraction logic, `harvestCustom()` gives you full control:

```javascript
const data = await harvester.harvestCustom(
  'https://example.com',
  () => {
    // This runs in the browser context
    return {
      title: document.title,
      links: Array.from(document.querySelectorAll('a')).map(a => ({
        href: a.href,
        text: a.textContent?.trim()
      })),
      metadata: Array.from(document.querySelectorAll('meta')).map(m => ({
        name: m.getAttribute('name'),
        content: m.getAttribute('content')
      }))
    }
  }
)
```

**Key points:**
- Function executes in browser context (has access to `document`, `window`)
- No access to Node.js scope or modules
- Must be self-contained
- Can return complex nested data structures

### Technique 2: Screenshot Capture

Capture visual snapshots during scraping:

```javascript
// Standalone screenshot
await harvester.screenshot(
  'https://example.com',
  { path: 'page.png', fullPage: true },
  { waitForLoadState: 'networkidle' }
)

// Screenshot during extraction
await harvester.harvest(url, selector, extractor, {
  screenshot: { path: 'extraction.png' }
})
```

**Use cases:**
- Debugging selector issues
- Archiving page state
- Visual verification
- Compliance documentation

### Technique 3: Combining Retry and Rate Limiting

These features work together seamlessly:

```javascript
const harvester = new DOMHarvester({
  retries: 3,
  backoff: 'exponential',
  rateLimit: { requestsPerSecond: 2, perDomain: true }
})

// Requests are:
// 1. Rate limited (max 2/sec per domain)
// 2. Retried on failure (up to 3 times)
// 3. With exponential backoff between retries
```

**Execution flow:**
```
Request → Rate Limit Check → Execute → Success/Failure
                                ↓
                             Failure → Backoff Delay → Retry → Rate Limit Check → ...
```

### Technique 4: Batch Processing with Progress Monitoring

Monitor long-running batch operations:

```javascript
let startTime = Date.now()

const results = await harvester.harvestBatch(configs, {
  concurrency: 5,
  onProgress: (completed, total) => {
    const elapsed = Date.now() - startTime
    const rate = completed / (elapsed / 1000)
    const eta = (total - completed) / rate

    console.log(`Progress: ${completed}/${total}`)
    console.log(`Rate: ${rate.toFixed(2)} pages/sec`)
    console.log(`ETA: ${eta.toFixed(0)} seconds`)
  }
})
```

### Technique 5: Dynamic Retry Configuration

Override retry settings per request:

```javascript
// Important pages get more retries
await harvester.harvest(criticalUrl, selector, extractor, {
  retries: 10
})

// Fast-fail for less critical pages
await harvester.harvest(optionalUrl, selector, extractor, {
  retries: 1
})
```

### Technique 6: Browser Customization

Full control over browser environment:

```javascript
const harvester = new DOMHarvester({
  proxy: { server: 'http://proxy.example.com:8080' },
  viewport: { width: 1920, height: 1080 },
  userAgent: 'Mozilla/5.0 Custom Agent',
  extraHTTPHeaders: {
    'Accept-Language': 'en-US',
    'X-Custom-Header': 'value'
  },
  locale: 'en-US',
  timezoneId: 'America/New_York',
  geolocation: { latitude: 40.7128, longitude: -74.0060 },
  cookies: [
    { name: 'session', value: 'abc123', domain: 'example.com' }
  ]
})
```

**Use cases:**
- Proxy rotation
- Geo-specific content
- Authenticated scraping
- Custom headers for API access

## Performance Considerations

### Memory Management

**Browser reuse is critical:**

```javascript
// Bad: Creates new browser for each page
for (const url of urls) {
  const data = await harvest(url, selector, extractor)
}

// Good: Reuses same browser
const harvester = new DOMHarvester()
await harvester.init()
for (const url of urls) {
  await harvester.harvest(url, selector, extractor)
}
await harvester.close()
```

**Why it matters:**
- Browser launch: ~2-3 seconds
- Memory per browser: ~100-200MB
- For 100 pages: 200-300 seconds vs. 10-20 seconds

### Concurrency vs. Rate Limiting Trade-offs

Finding the right balance:

```javascript
// Aggressive: Fast but risky
const aggressive = new DOMHarvester({
  rateLimit: { requestsPerSecond: 10 },
  retries: 1
})

await aggressive.harvestBatch(configs, { concurrency: 10 })

// Conservative: Slower but safer
const conservative = new DOMHarvester({
  rateLimit: { requestsPerSecond: 2 },
  retries: 5
})

await conservative.harvestBatch(configs, { concurrency: 3 })
```

**Guidelines:**
- Start conservative, increase gradually
- Monitor error rates
- Respect robots.txt
- Consider target server capacity

### Wait Strategies

Different load states for different needs:

```javascript
// Fast: Wait for DOM only
await harvester.harvest(url, selector, extractor, {
  waitForLoadState: 'domcontentloaded'
})

// Balanced: Wait for load event
await harvester.harvest(url, selector, extractor, {
  waitForLoadState: 'load'
})

// Thorough: Wait for network idle
await harvester.harvest(url, selector, extractor, {
  waitForLoadState: 'networkidle'
})

// Custom: Wait for specific element
await harvester.harvest(url, selector, extractor, {
  waitForSelector: '.dynamic-content'
})
```

## Real-World Architecture Example

Here's how to structure a production scraping system:

```javascript
import { DOMHarvester } from 'domharvest-playwright'
import winston from 'winston'

class ProductionScraper {
  constructor() {
    this.logger = winston.createLogger({
      level: 'info',
      format: winston.format.json(),
      transports: [
        new winston.transports.File({ filename: 'error.log', level: 'error' }),
        new winston.transports.File({ filename: 'combined.log' })
      ]
    })

    this.harvester = new DOMHarvester({
      headless: true,
      timeout: 30000,
      retries: 3,
      backoff: 'exponential',
      rateLimit: {
        requestsPerSecond: 2,
        perDomain: true
      },
      logging: {
        level: 'info',
        logger: {
          debug: (msg) => this.logger.debug(msg),
          info: (msg) => this.logger.info(msg),
          warn: (msg) => this.logger.warn(msg),
          error: (msg) => this.logger.error(msg)
        }
      },
      onError: (error) => {
        this.logger.error('Scraping error', {
          url: error.url,
          type: error.constructor.name,
          message: error.message
        })
      }
    })
  }

  async init() {
    await this.harvester.init()
  }

  async scrapeProducts(urls) {
    const configs = urls.map(url => ({
      url,
      selector: '.product',
      extractor: (el) => ({
        title: el.querySelector('h2')?.textContent?.trim(),
        price: parseFloat(
          el.querySelector('.price')?.textContent?.replace(/[^0-9.]/g, '') || '0'
        ),
        inStock: !el.querySelector('.out-of-stock')
      })
    }))

    return await this.harvester.harvestBatch(configs, {
      concurrency: 5,
      onProgress: (completed, total) => {
        this.logger.info(`Progress: ${completed}/${total}`)
      }
    })
  }

  async close() {
    await this.harvester.close()
  }
}

// Usage
const scraper = new ProductionScraper()
await scraper.init()

try {
  const products = await scraper.scrapeProducts(urls)
  console.log(`Scraped ${products.length} products`)
} finally {
  await scraper.close()
}
```

## Best Practices

### 1. Always Use Try/Finally for Cleanup

```javascript
const harvester = new DOMHarvester()
try {
  await harvester.init()
  // ... scraping logic
} finally {
  await harvester.close()
}
```

### 2. Configure Logging for Production

```javascript
const harvester = new DOMHarvester({
  logging: { level: 'info' }, // Not 'debug' in production
  onError: (error) => {
    // Send to monitoring service
  }
})
```

### 3. Use Batch Processing for Multiple URLs

```javascript
// Good: Efficient batch processing
await harvester.harvestBatch(configs, { concurrency: 3 })

// Less efficient: Sequential processing
for (const config of configs) {
  await harvester.harvest(config.url, config.selector, config.extractor)
}
```

### 4. Set Appropriate Retry Counts

```javascript
// Critical data: More retries
await harvester.harvest(url, selector, extractor, { retries: 10 })

// Optional data: Fewer retries
await harvester.harvest(url, selector, extractor, { retries: 1 })
```

### 5. Monitor and Adjust Rate Limits

```javascript
const harvester = new DOMHarvester({
  rateLimit: { requestsPerSecond: 2 }, // Start conservative
  onError: (error) => {
    if (error.message.includes('429')) {
      // Adjust rate limit if getting throttled
    }
  }
})
```

## Conclusion

domharvest-playwright's architecture demonstrates that simplicity and production-readiness are not mutually exclusive. The library provides:

**Built-in reliability:**
- Automatic retry with configurable backoff
- Rate limiting to prevent bans
- Enhanced error handling with custom classes
- Structured logging for observability

**Scalability:**
- Batch processing with concurrency control
- Per-domain rate limiting
- Efficient browser reuse
- Memory-conscious design

**Flexibility:**
- Simple function API for quick tasks
- Full-featured class API for production
- Extensive configuration options
- Direct Playwright access when needed

**Key takeaways:**

1. **Production features are first-class**, not add-ons
2. **Retry and rate limiting** work together seamlessly
3. **Batch processing** enables efficient multi-URL scraping
4. **Enhanced error handling** provides context for debugging
5. **Structured logging** enables production observability

Whether you're building a simple scraper or a production data pipeline, domharvest-playwright provides the infrastructure you need without requiring you to build it yourself.

---

**Resources:**
- [GitHub Repository](https://github.com/domharvest/domharvest-playwright)
- [Full Documentation](https://domharvest.github.io/domharvest-playwright/)
- [Playwright API Docs](https://playwright.dev/docs/api/class-playwright)

*Have architectural questions? Discuss on [GitHub Issues](https://github.com/domharvest/domharvest-playwright/issues)!*
