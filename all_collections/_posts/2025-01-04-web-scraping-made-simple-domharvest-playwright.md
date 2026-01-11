---
layout: post
title: "Web Scraping Made Simple: A Practical Guide to domharvest-playwright"
date: 2025-01-04
categories: ["web-scraping", "playwright", "tutorial", "javascript"]
---

# Web Scraping Made Simple: A Practical Guide to domharvest-playwright

Web scraping doesn't need to be complicated. While powerful tools like Playwright give you complete control over browser automation, they often require significant boilerplate code for common scraping tasks. That's where **domharvest-playwright** comes in—a production-ready library that wraps Playwright's power with built-in retry logic, rate limiting, batch processing, and enhanced error handling.

In this practical guide, I'll walk you through everything you need to know to start scraping websites efficiently with domharvest-playwright.

## What is domharvest-playwright?

domharvest-playwright is a lightweight wrapper around Playwright that specializes in extracting structured data from web pages. Unlike raw Playwright, it comes with production features out of the box:

**Built-in features:**
- Automatic retry logic with exponential and linear backoff
- Built-in rate limiting (global or per-domain)
- Batch processing with configurable concurrency
- Enhanced error handling with custom error classes
- Structured logging at multiple levels
- Screenshot capture during extraction
- Support for proxies, custom headers, and cookies

## Installation and Setup

Before we start, you'll need Node.js 18 or higher. The installation is straightforward:

```bash
# Install the package
npm install domharvest-playwright

# Install Playwright browsers (required for automation)
npx playwright install
```

Note: Playwright will download browser binaries (~500MB), so the first install might take a few minutes.

### Quick Verification

Let's verify everything works with a simple test:

```javascript
import { harvest } from 'domharvest-playwright'

const result = await harvest(
  'https://example.com',
  'h1',
  (el) => ({ heading: el.textContent?.trim() })
)

console.log('Success!', result)
```

If you see the extracted heading, you're ready to go!

## Your First Scraper: Extracting Quotes

Let's start with a practical example. We'll scrape quotes from [quotes.toscrape.com](https://quotes.toscrape.com/), a website specifically designed for practicing web scraping.

### The Simple Approach

The `harvest()` function is perfect for one-off scraping tasks:

```javascript
import { harvest } from 'domharvest-playwright'

const quotes = await harvest(
  'https://quotes.toscrape.com/',
  '.quote',
  (el) => ({
    text: el.querySelector('.text')?.textContent?.trim(),
    author: el.querySelector('.author')?.textContent?.trim(),
    tags: Array.from(el.querySelectorAll('.tag')).map(tag => tag.textContent?.trim())
  })
)

console.log(quotes)
```

**Output:**
```javascript
[
  {
    text: '"The world as we have created it is a process of our thinking..."',
    author: 'Albert Einstein',
    tags: ['change', 'deep-thoughts', 'thinking', 'world']
  },
  // ... more quotes
]
```

Let's break down what's happening:

1. **URL**: `'https://quotes.toscrape.com/'` - The page we want to scrape
2. **Selector**: `'.quote'` - CSS selector to find all quote elements
3. **Extractor**: A function that transforms each matched element into structured data
4. **Return value**: An array of extracted objects

The beauty of this approach is that you don't need to worry about:
- Launching the browser
- Navigating to the page
- Waiting for elements to load
- Closing the browser when done

It's all handled automatically.

## Using the DOMHarvester Class for More Control

For production use cases, you'll want to use the `DOMHarvester` class which gives you access to all configuration options:

```javascript
import { DOMHarvester } from 'domharvest-playwright'

const harvester = new DOMHarvester({
  headless: true,
  timeout: 30000,
  rateLimit: { requestsPerSecond: 2 }, // Built-in rate limiting
  logging: { level: 'info' },
  retries: 3, // Will retry failed requests
  backoff: 'exponential' // Exponential backoff between retries
})

await harvester.init()

try {
  const quotes = await harvester.harvest(
    'https://quotes.toscrape.com/',
    '.quote',
    (el) => ({
      text: el.querySelector('.text')?.textContent?.trim(),
      author: el.querySelector('.author')?.textContent?.trim()
    })
  )

  console.log(`Extracted ${quotes.length} quotes`)
} finally {
  await harvester.close()
}
```

## Built-in Retry Logic

One of the most powerful features is automatic retry with backoff:

```javascript
const harvester = new DOMHarvester({
  retries: 3,
  backoff: 'exponential' // or 'linear'
})

await harvester.init()

try {
  // This will automatically retry up to 3 times if it fails
  const data = await harvester.harvest(
    'https://example.com',
    '.content',
    (el) => ({ text: el.textContent })
  )
} finally {
  await harvester.close()
}
```

The library handles transient failures automatically with configurable backoff strategies.

## Batch Processing Multiple URLs

Need to scrape multiple pages? Use `harvestBatch()` with built-in concurrency control:

```javascript
const harvester = new DOMHarvester({
  rateLimit: { requestsPerSecond: 2 }
})

await harvester.init()

try {
  const configs = [
    {
      url: 'https://quotes.toscrape.com/page/1/',
      selector: '.quote',
      extractor: (el) => ({
        text: el.querySelector('.text')?.textContent?.trim(),
        author: el.querySelector('.author')?.textContent?.trim()
      })
    },
    {
      url: 'https://quotes.toscrape.com/page/2/',
      selector: '.quote',
      extractor: (el) => ({
        text: el.querySelector('.text')?.textContent?.trim(),
        author: el.querySelector('.author')?.textContent?.trim()
      })
    }
  ]

  const results = await harvester.harvestBatch(configs, {
    concurrency: 3,
    onProgress: (completed, total) => {
      console.log(`Progress: ${completed}/${total}`)
    }
  })

  console.log(`Scraped ${results.length} pages`)
} finally {
  await harvester.close()
}
```

## Custom Page Analysis with harvestCustom()

Sometimes you need more control than CSS selectors provide. The `harvestCustom()` method lets you run arbitrary JavaScript in the browser context:

```javascript
const harvester = new DOMHarvester()
await harvester.init()

try {
  const pageData = await harvester.harvestCustom(
    'https://quotes.toscrape.com/',
    () => {
      // This function runs in the browser context
      return {
        title: document.title,
        totalQuotes: document.querySelectorAll('.quote').length,
        uniqueAuthors: Array.from(new Set(
          Array.from(document.querySelectorAll('.author'))
            .map(a => a.textContent?.trim())
        )),
        hasNextPage: document.querySelector('.next') !== null
      }
    }
  )

  console.log('Page Analysis:', pageData)
} finally {
  await harvester.close()
}
```

**Important:** The function passed to `harvestCustom()` runs in the browser, not in Node.js. This means:
- You have access to `document`, `window`, etc.
- You don't have access to Node.js modules or outer scope variables
- The function must be self-contained

## Screenshot Capture

Capture screenshots during or after extraction:

```javascript
const harvester = new DOMHarvester()
await harvester.init()

try {
  // Capture screenshot of a specific page
  const screenshot = await harvester.screenshot(
    'https://quotes.toscrape.com/',
    { path: 'quotes.png', fullPage: true },
    { waitForLoadState: 'networkidle' }
  )

  console.log('Screenshot saved')
} finally {
  await harvester.close()
}
```

## Enhanced Error Handling

The library provides custom error classes for better error handling:

```javascript
import { DOMHarvester, TimeoutError, NavigationError, ExtractionError } from 'domharvest-playwright'

const harvester = new DOMHarvester({
  onError: (error) => {
    console.error('Scraping error:', error.message)
  }
})

await harvester.init()

try {
  const data = await harvester.harvest(
    'https://example.com',
    '.content',
    (el) => ({ text: el.textContent })
  )
} catch (error) {
  if (error instanceof TimeoutError) {
    console.error('Page took too long to load:', error.url)
  } else if (error instanceof NavigationError) {
    console.error('Failed to navigate:', error.url, error.cause)
  } else if (error instanceof ExtractionError) {
    console.error('Failed to extract data:', error.selector, error.operation)
  }
} finally {
  await harvester.close()
}
```

## Configuration Options

domharvest-playwright provides extensive configuration:

### Constructor Options

```javascript
const harvester = new DOMHarvester({
  // Browser settings
  headless: true,
  timeout: 30000,

  // Rate limiting
  rateLimit: {
    requestsPerSecond: 2,
    perDomain: true // Rate limit per domain
  },

  // Retry configuration
  retries: 3,
  backoff: 'exponential', // or 'linear'

  // Logging
  logging: {
    level: 'info' // 'debug', 'info', 'warn', 'error'
  },

  // Error handling
  onError: (error) => console.error(error),

  // Browser customization
  proxy: { server: 'http://proxy.example.com:8080' },
  viewport: { width: 1920, height: 1080 },
  userAgent: 'Custom User Agent',
  extraHTTPHeaders: { 'X-Custom-Header': 'value' },

  // Environment
  locale: 'en-US',
  timezoneId: 'America/New_York',
  geolocation: { latitude: 40.7128, longitude: -74.0060 },

  // JavaScript execution
  javaScriptEnabled: true
})
```

### Harvest Method Options

```javascript
await harvester.harvest(url, selector, extractor, {
  retries: 5, // Override global retry setting
  waitForLoadState: 'networkidle', // or 'load', 'domcontentloaded'
  waitForSelector: '.dynamic-content', // Wait for specific element
  screenshot: { path: 'page.png' } // Capture screenshot during extraction
})
```

## Rate Limiting

Built-in rate limiting prevents overwhelming target servers:

```javascript
// Global rate limiting
const harvester = new DOMHarvester({
  rateLimit: { requestsPerSecond: 2 }
})

// Per-domain rate limiting
const harvester2 = new DOMHarvester({
  rateLimit: {
    requestsPerSecond: 2,
    perDomain: true // Different rate limits per domain
  }
})
```

The library automatically handles request throttling—you don't need to add manual delays.

## Structured Logging

Monitor your scraping operations with built-in logging:

```javascript
const harvester = new DOMHarvester({
  logging: {
    level: 'debug' // Detailed logging
  }
})

// Logs will show:
// - Navigation events
// - Retry attempts
// - Rate limiting delays
// - Errors and warnings
```

## Best Practices

### 1. Always Close Resources

Use `try/finally` blocks to ensure the browser is always closed:

```javascript
const harvester = new DOMHarvester()
try {
  await harvester.init()
  // ... scraping logic
} finally {
  await harvester.close()
}
```

### 2. Use Built-in Rate Limiting

Don't add manual delays—use the built-in rate limiter:

```javascript
// Good: Built-in rate limiting
const harvester = new DOMHarvester({
  rateLimit: { requestsPerSecond: 2 }
})

// Not needed: Manual delays
await new Promise(resolve => setTimeout(resolve, 1000))
```

### 3. Let Retry Logic Handle Failures

Configure retries instead of writing manual retry logic:

```javascript
const harvester = new DOMHarvester({
  retries: 3,
  backoff: 'exponential'
})
// Failures are automatically retried
```

### 4. Use Batch Processing for Multiple URLs

Use `harvestBatch()` instead of loops:

```javascript
// Good: Batch processing with concurrency control
await harvester.harvestBatch(configs, { concurrency: 3 })

// Less efficient: Sequential processing
for (const url of urls) {
  await harvester.harvest(url, selector, extractor)
}
```

### 5. Monitor with Structured Logging

Enable logging for production deployments:

```javascript
const harvester = new DOMHarvester({
  logging: { level: 'info' },
  onError: (error) => {
    // Send to your monitoring system
    logToMonitoring(error)
  }
})
```

## Real-World Example: Product Monitoring

Here's a complete example using all the production features:

```javascript
import { DOMHarvester } from 'domharvest-playwright'
import { writeFile } from 'fs/promises'

async function monitorProducts(urls) {
  const harvester = new DOMHarvester({
    headless: true,
    timeout: 30000,
    rateLimit: { requestsPerSecond: 2, perDomain: true },
    retries: 3,
    backoff: 'exponential',
    logging: { level: 'info' },
    onError: (error) => console.error('Scraping error:', error.message)
  })

  await harvester.init()

  try {
    const configs = urls.map(url => ({
      url,
      selector: 'body',
      extractor: () => ({
        title: document.querySelector('h1')?.textContent?.trim(),
        price: document.querySelector('.price')?.textContent?.trim(),
        availability: document.querySelector('.stock')?.textContent?.trim(),
        timestamp: new Date().toISOString()
      })
    }))

    const results = await harvester.harvestBatch(configs, {
      concurrency: 3,
      onProgress: (completed, total) => {
        console.log(`Progress: ${completed}/${total}`)
      }
    })

    await writeFile(
      `products-${Date.now()}.json`,
      JSON.stringify(results, null, 2)
    )

    console.log(`Monitored ${results.length} products`)
    return results
  } finally {
    await harvester.close()
  }
}

const urls = [
  'https://books.toscrape.com/catalogue/a-light-in-the-attic_1000/index.html',
  'https://books.toscrape.com/catalogue/tipping-the-velvet_999/index.html'
]

await monitorProducts(urls)
```

## Conclusion

domharvest-playwright is more than a simple wrapper—it's a production-ready scraping solution with built-in retry logic, rate limiting, batch processing, and enhanced error handling. These features eliminate the need to build your own infrastructure around Playwright.

**Key takeaways:**
- Use `harvest()` for simple, one-off scraping tasks
- Use `DOMHarvester` with full configuration for production
- Leverage built-in retry logic and rate limiting
- Use `harvestBatch()` for efficient multi-URL scraping
- Monitor with structured logging and custom error classes
- Always close resources with `try/finally`

Whether you're building a one-off data extraction script or a production scraping pipeline, domharvest-playwright provides production-ready features out of the box.

Happy harvesting!

---

**Resources:**
- [GitHub Repository](https://github.com/domharvest/domharvest-playwright)
- [npm Package](https://www.npmjs.com/package/domharvest-playwright)
- [Full Documentation](https://domharvest.github.io/domharvest-playwright/)

*Questions or feedback? Reach out on [GitHub](https://github.com/domharvest) or [Mastodon](https://infosec.exchange/@domharvest)!*
