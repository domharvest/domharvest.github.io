---
layout: post
title: "Inside domharvest-playwright: Architecture, Design Patterns, and Advanced Techniques"
date: 2025-01-06
categories: ["web-scraping", "playwright", "architecture", "javascript", "deep-dive"]
---

# Inside domharvest-playwright: Architecture, Design Patterns, and Advanced Techniques

While many developers use domharvest-playwright for its simple API, there's a sophisticated architecture underneath that makes it both powerful and flexible. In this deep dive, we'll explore the design decisions, patterns, and advanced techniques that make domharvest-playwright production-ready.

This article is for developers who want to understand not just *how* to use the library, but *why* it works the way it does.

## Architectural Overview

domharvest-playwright follows a layered architecture that progressively builds abstractions over Playwright's browser automation:

```
┌─────────────────────────────────────────┐
│   High-Level API (harvest function)    │  ← Simple convenience layer
├─────────────────────────────────────────┤
│   DOMHarvester Class                    │  ← Object-oriented interface
├─────────────────────────────────────────┤
│   Resource Management Layer             │  ← Browser lifecycle handling
├─────────────────────────────────────────┤
│   Function Serialization & Evaluation   │  ← Page context execution
├─────────────────────────────────────────┤
│   Playwright Core (Browser Drivers)     │  ← Foundation
└─────────────────────────────────────────┘
```

This layered design follows the principle of **progressive disclosure**: simple tasks use simple APIs, complex tasks have access to deeper layers.

## Design Philosophy

### 1. Separation of Concerns

The library cleanly separates three distinct responsibilities:

**Browser Management**
- Launching and closing browsers
- Creating and managing contexts
- Resource cleanup

**Navigation and Waiting**
- Page navigation
- Element waiting and timeouts
- Error handling

**Data Extraction**
- Selector evaluation
- Element transformation
- Result aggregation

This separation makes the code testable, maintainable, and easier to reason about.

### 2. Dual API Pattern

The library provides two complementary interfaces:

**Function-Based API** (`harvest()`):
```javascript
const data = await harvest(url, selector, extractor, options)
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
- Class API: Optimized for reuse, explicit lifecycle management

Let's examine how this is implemented under the hood.

## Implementation Deep Dive

### The harvest() Function: Simplicity Through Abstraction

The `harvest()` function is a thin wrapper that creates, uses, and destroys a `DOMHarvester` instance:

```javascript
export async function harvest(url, selector, extractor, options = {}) {
  const harvester = new DOMHarvester(options)

  try {
    await harvester.init()
    return await harvester.harvest(url, selector, extractor)
  } finally {
    await harvester.close()
  }
}
```

This implementation demonstrates several patterns:

1. **Factory Pattern**: Creates the harvester instance
2. **Resource Acquisition Is Initialization (RAII)**: Guarantees cleanup via try/finally
3. **Facade Pattern**: Hides complexity behind a simple interface

The trade-off is clear: simplicity for one-time use, with the cost of creating a new browser instance each time.

### The DOMHarvester Class: Power Through Flexibility

The `DOMHarvester` class is where the real work happens. Let's examine its structure:

```javascript
class DOMHarvester {
  constructor(options = {}) {
    this.options = {
      headless: options.headless ?? true,
      timeout: options.timeout ?? 30000
    }
    this.browser = null
    this.context = null
    this.page = null
  }

  async init() {
    // Launch browser with Playwright
    this.browser = await chromium.launch({
      headless: this.options.headless
    })

    // Create browser context (isolated session)
    this.context = await this.browser.newContext()

    // Create a page in that context
    this.page = await this.context.newPage()
  }

  async harvest(url, selector, extractor) {
    // Navigate to URL with timeout
    await this.page.goto(url, {
      timeout: this.options.timeout,
      waitUntil: 'domcontentloaded'
    })

    // Wait for selector to appear
    await this.page.waitForSelector(selector, {
      timeout: this.options.timeout
    })

    // Execute extraction in browser context
    return await this.page.$$eval(selector, (elements, extractorStr) => {
      const extractorFn = eval(`(${extractorStr})`)
      return elements.map(el => extractorFn(el))
    }, extractor.toString())
  }

  async close() {
    if (this.browser) {
      await this.browser.close()
    }
  }
}
```

This implementation reveals several important patterns:

### Pattern 1: Explicit Initialization

The constructor doesn't do any heavy work—it just sets up state. The actual browser launch happens in `init()`. This is intentional:

**Benefits:**
- Constructor can't fail (no async operations)
- Explicit control over when browser launches
- Easier to test and mock
- Clear lifecycle stages

**Trade-off:**
- Requires calling `init()` before use
- More verbose than automatic initialization

### Pattern 2: Context Isolation

The class creates a browser context, not just a browser:

```javascript
this.browser = await chromium.launch(...)
this.context = await this.browser.newContext()
this.page = await this.context.newPage()
```

**Why contexts matter:**
- Isolate cookies, storage, and cache
- Enable parallel operations without interference
- Simulate different users or sessions
- Essential for production scraping

### Pattern 3: Function Serialization

The most interesting part is how extractor functions are handled:

```javascript
return await this.page.$$eval(selector, (elements, extractorStr) => {
  const extractorFn = eval(`(${extractorStr})`)
  return elements.map(el => extractorFn(el))
}, extractor.toString())
```

This is necessary because the extractor function needs to run **in the browser context**, not in Node.js. Let's break down what happens:

1. **Serialization**: `extractor.toString()` converts the function to a string
2. **Injection**: The string is passed to the browser context
3. **Reconstruction**: `eval()` recreates the function in the browser
4. **Execution**: The function runs on each matched element

**Implications:**
- Extractor functions can't access outer scope variables
- Functions must be self-contained
- Complex logic needs to be embedded in the function

This is a fundamental constraint of browser automation, not a limitation of the library.

## Advanced Techniques

### Technique 1: Direct Playwright API Access

The `DOMHarvester` class exposes its internal Playwright instances:

```javascript
const harvester = new DOMHarvester()
await harvester.init()

// Access the page directly
await harvester.page.goto('https://example.com')
await harvester.page.waitForSelector('.dynamic-content')

// Use Playwright's full API
const screenshot = await harvester.page.screenshot()
const cookies = await harvester.context.cookies()
const title = await harvester.page.title()
```

This design choice follows the **Principle of Least Surprise**: when you need more power, the full Playwright API is right there.

### Technique 2: Custom Page Evaluation

The `harvestCustom()` method provides a different pattern for complex extraction:

```javascript
async harvestCustom(url, pageFunction) {
  await this.page.goto(url, {
    timeout: this.options.timeout,
    waitUntil: 'domcontentloaded'
  })

  return await this.page.evaluate(pageFunction)
}
```

**Usage:**
```javascript
const data = await harvester.harvestCustom(url, () => {
  // This runs in the browser
  return {
    title: document.title,
    links: Array.from(document.querySelectorAll('a')).map(a => a.href),
    scripts: Array.from(document.scripts).map(s => s.src)
  }
})
```

This pattern is more powerful than `harvest()` because:
- You control the entire extraction logic
- You can aggregate data across selectors
- You can implement custom waiting or filtering logic

### Technique 3: Handling Dynamic Content

Modern websites often load content dynamically. Here's how to handle it:

```javascript
const harvester = new DOMHarvester({ timeout: 60000 })
await harvester.init()

try {
  // Navigate to page
  await harvester.page.goto('https://example.com/infinite-scroll')

  // Wait for initial content
  await harvester.page.waitForSelector('.item', { timeout: 5000 })

  // Scroll to trigger lazy loading
  await harvester.page.evaluate(() => {
    window.scrollTo(0, document.body.scrollHeight)
  })

  // Wait for new content to appear
  await harvester.page.waitForFunction(
    () => document.querySelectorAll('.item').length > 20,
    { timeout: 10000 }
  )

  // Now extract all items
  const items = await harvester.page.$$eval('.item', items =>
    items.map(item => ({
      title: item.querySelector('h2')?.textContent,
      content: item.querySelector('p')?.textContent
    }))
  )

  console.log(`Extracted ${items.length} items`)
} finally {
  await harvester.close()
}
```

This demonstrates:
- Custom wait conditions with `waitForFunction()`
- Programmatic scrolling to trigger lazy loading
- Dynamic content extraction after loading completes

### Technique 4: Network Interception

Sometimes the best approach isn't scraping HTML at all—it's intercepting API calls:

```javascript
const harvester = new DOMHarvester()
await harvester.init()

try {
  // Set up request interception
  await harvester.page.route('**/api/data', route => {
    route.continue()
  })

  // Listen for responses
  const apiData = await new Promise(resolve => {
    harvester.page.on('response', async response => {
      if (response.url().includes('/api/data')) {
        const json = await response.json()
        resolve(json)
      }
    })

    // Trigger the API call by navigating
    harvester.page.goto('https://example.com')
  })

  console.log('Intercepted API data:', apiData)
} finally {
  await harvester.close()
}
```

This technique:
- Captures structured data before it's rendered
- Avoids parsing HTML entirely
- Often more reliable than DOM scraping
- Reveals the actual data source

### Technique 5: Parallel Scraping with Multiple Contexts

For production scraping, you often need to scrape multiple pages in parallel:

```javascript
import { chromium } from 'playwright'

async function parallelScrape(urls, concurrency = 5) {
  const browser = await chromium.launch({ headless: true })

  // Process URLs in chunks
  const results = []

  for (let i = 0; i < urls.length; i += concurrency) {
    const chunk = urls.slice(i, i + concurrency)

    // Create a context and page for each URL in the chunk
    const chunkResults = await Promise.all(
      chunk.map(async url => {
        const context = await browser.newContext()
        const page = await context.newPage()

        try {
          await page.goto(url, { timeout: 30000 })
          const data = await page.$$eval('.item', items =>
            items.map(item => ({ text: item.textContent }))
          )
          return { url, data, error: null }
        } catch (error) {
          return { url, data: null, error: error.message }
        } finally {
          await context.close()
        }
      })
    )

    results.push(...chunkResults)

    // Rate limiting between chunks
    await new Promise(resolve => setTimeout(resolve, 1000))
  }

  await browser.close()
  return results
}

// Scrape 50 URLs with 5 concurrent requests
const results = await parallelScrape(urls, 5)
```

**Key concepts:**
- Create one browser instance
- Use multiple contexts for isolation
- Process URLs in chunks to limit concurrency
- Handle errors per-URL to avoid failing entire batch
- Add rate limiting between chunks

## Performance Considerations

### Memory Management

Browser instances consume significant memory. Here's how to optimize:

**1. Reuse Browser Instances**
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

**2. Close Contexts When Done**
```javascript
// Create multiple contexts for parallel work
const contexts = await Promise.all(
  [1, 2, 3].map(() => browser.newContext())
)

// Use them
// ...

// Close them
await Promise.all(contexts.map(ctx => ctx.close()))
```

**3. Set Resource Limits**
```javascript
const context = await browser.newContext({
  // Block unnecessary resources
  bypassCSP: true,
  javaScriptEnabled: true,
  ignoreHTTPSErrors: true
})

// Block images and stylesheets
await context.route('**/*', route => {
  const type = route.request().resourceType()
  if (type === 'image' || type === 'stylesheet') {
    route.abort()
  } else {
    route.continue()
  }
})
```

### Timeout Strategy

Timeouts are critical for reliability. Here's a sophisticated approach:

```javascript
class SmartHarvester extends DOMHarvester {
  async harvest(url, selector, extractor, retries = 3) {
    for (let attempt = 1; attempt <= retries; attempt++) {
      try {
        // Exponential backoff for timeout
        const timeout = this.options.timeout * attempt

        await this.page.goto(url, {
          timeout,
          waitUntil: 'domcontentloaded'
        })

        await this.page.waitForSelector(selector, { timeout })

        return await this.page.$$eval(selector, (elements, extractorStr) => {
          const extractorFn = eval(`(${extractorStr})`)
          return elements.map(el => extractorFn(el))
        }, extractor.toString())

      } catch (error) {
        if (attempt === retries) throw error

        console.log(`Attempt ${attempt} failed, retrying...`)
        await new Promise(resolve => setTimeout(resolve, 1000 * attempt))
      }
    }
  }
}
```

This implements:
- Retry logic with exponential backoff
- Increasing timeouts for each retry
- Configurable retry count

## Comparing with Alternatives

Let's compare domharvest-playwright with other scraping approaches:

### vs. Raw Playwright

**domharvest-playwright:**
```javascript
const data = await harvest(url, '.item', el => ({
  title: el.querySelector('h2')?.textContent
}))
```

**Raw Playwright:**
```javascript
const browser = await chromium.launch()
const context = await browser.newContext()
const page = await context.newPage()
try {
  await page.goto(url)
  await page.waitForSelector('.item')
  const data = await page.$$eval('.item', items =>
    items.map(item => ({
      title: item.querySelector('h2')?.textContent
    }))
  )
} finally {
  await browser.close()
}
```

**Trade-off:** domharvest-playwright reduces boilerplate by ~60% for common cases, but you can still access raw Playwright when needed.

### vs. Cheerio

**Cheerio** is faster but can't handle JavaScript-rendered content:

```javascript
// Cheerio: Fast but no JS execution
const $ = cheerio.load(html)
const data = $('.item').map((i, el) => ({
  title: $(el).find('h2').text()
})).get()

// domharvest-playwright: Slower but handles SPAs
const data = await harvest(url, '.item', el => ({
  title: el.querySelector('h2')?.textContent
}))
```

**When to use each:**
- Cheerio: Static HTML, known structure, speed critical
- domharvest-playwright: Dynamic content, SPAs, JavaScript-heavy sites

### vs. Puppeteer

Puppeteer is similar to Playwright, but Playwright has advantages:

**Playwright Benefits:**
- Multi-browser support (Chromium, Firefox, WebKit)
- Better auto-waiting mechanisms
- More active development
- Better network interception

domharvest-playwright builds on these advantages while adding DOM extraction patterns.

## Best Practices for Production

### 1. Implement Circuit Breakers

Prevent cascading failures:

```javascript
class CircuitBreaker {
  constructor(threshold = 5, timeout = 60000) {
    this.failures = 0
    this.threshold = threshold
    this.timeout = timeout
    this.state = 'closed' // closed, open, half-open
    this.nextAttempt = Date.now()
  }

  async execute(fn) {
    if (this.state === 'open') {
      if (Date.now() < this.nextAttempt) {
        throw new Error('Circuit breaker is open')
      }
      this.state = 'half-open'
    }

    try {
      const result = await fn()
      this.onSuccess()
      return result
    } catch (error) {
      this.onFailure()
      throw error
    }
  }

  onSuccess() {
    this.failures = 0
    this.state = 'closed'
  }

  onFailure() {
    this.failures++
    if (this.failures >= this.threshold) {
      this.state = 'open'
      this.nextAttempt = Date.now() + this.timeout
    }
  }
}

// Usage
const breaker = new CircuitBreaker()
const harvester = new DOMHarvester()

for (const url of urls) {
  try {
    await breaker.execute(async () => {
      return await harvester.harvest(url, selector, extractor)
    })
  } catch (error) {
    console.error(`Failed to scrape ${url}: ${error.message}`)
  }
}
```

### 2. Add Comprehensive Logging

```javascript
class LoggedHarvester extends DOMHarvester {
  async harvest(url, selector, extractor) {
    const start = Date.now()
    console.log(`[${new Date().toISOString()}] Starting harvest: ${url}`)

    try {
      const result = await super.harvest(url, selector, extractor)
      const duration = Date.now() - start

      console.log(`[${new Date().toISOString()}] Success: ${url} (${duration}ms, ${result.length} items)`)

      return result
    } catch (error) {
      const duration = Date.now() - start

      console.error(`[${new Date().toISOString()}] Failed: ${url} (${duration}ms) - ${error.message}`)

      throw error
    }
  }
}
```

### 3. Use Type Safety with TypeScript

```typescript
import { DOMHarvester } from 'domharvest-playwright'

interface Product {
  title: string
  price: number
  inStock: boolean
}

async function scrapeProducts(url: string): Promise<Product[]> {
  const harvester = new DOMHarvester()
  await harvester.init()

  try {
    return await harvester.harvest<Product>(
      url,
      '.product',
      (el): Product => ({
        title: el.querySelector('h2')?.textContent?.trim() ?? '',
        price: parseFloat(
          el.querySelector('.price')?.textContent?.replace(/[^0-9.]/g, '') ?? '0'
        ),
        inStock: el.querySelector('.stock')?.textContent?.includes('In stock') ?? false
      })
    )
  } finally {
    await harvester.close()
  }
}
```

## Conclusion

domharvest-playwright's architecture demonstrates that simple APIs don't require sacrificing power or flexibility. The key design decisions—layered architecture, dual API pattern, explicit lifecycle management, and direct Playwright access—create a library that scales from quick scripts to production systems.

**Key takeaways:**

1. **Layered architecture** provides progressive disclosure of complexity
2. **Function serialization** is necessary for browser context execution
3. **Explicit initialization** gives control over resource lifecycle
4. **Direct API access** enables advanced techniques when needed
5. **Context isolation** is essential for parallel and production scraping

Whether you're building a simple scraper or a production data pipeline, understanding these patterns will help you use domharvest-playwright effectively and extend it when needed.

---

**Resources:**
- [GitHub Repository](https://github.com/domharvest/domharvest-playwright)
- [Full Documentation](https://domharvest.github.io/domharvest-playwright/)
- [Playwright API Docs](https://playwright.dev/docs/api/class-playwright)

*Have architectural questions? Discuss on [GitHub Issues](https://github.com/domharvest/domharvest-playwright/issues)!*
