---
layout: home

hero:
  name: DOMHarvest
  text: Playwright-powered web scraping
  tagline: Extract DOM elements with precision and speed
  image:
    src: /DomHarvest_Logo.png
    alt: DOMHarvest Logo
  actions:
    - theme: brand
      text: Get Started
      link: /guide/getting-started
    - theme: alt
      text: View on GitHub
      link: https://github.com/domharvest/domharvest-playwright

features:
  - icon: 🎯
    title: Declarative DSL
    details: Extract data with a clean DSL using text(), attr(), array() helpers - no more verbose querySelector chains.
  - icon: 🔐
    title: Authentication Built-in
    details: Complete session management with login helpers, cookie persistence, and multi-account support.
  - icon: 🚀
    title: Built on Playwright
    details: Leverages Playwright's battle-tested browser automation for reliable, cross-browser scraping.
  - icon: ⚡
    title: Production Ready
    details: Rate limiting, retry logic with exponential backoff, proxy support, and structured logging.
  - icon: 📸
    title: Screenshot Support
    details: Capture full-page or clipped screenshots during scraping with customizable options.
  - icon: 🔄
    title: Batch Processing
    details: Harvest multiple URLs concurrently with progress tracking and configurable concurrency.
---

## Quick Example

```javascript
import { harvest, text, array } from 'domharvest-playwright'

// Extract quotes using the declarative DSL
const quotes = await harvest(
  'https://quotes.toscrape.com/',
  '.quote',
  {
    text: text('.text'),
    author: text('.author'),
    tags: array('.tag', text())
  }
)

console.log(quotes)
// [{ text: "The world as we...", author: "Albert Einstein", tags: ["change", "world"] }, ...]
```

## Why DOMHarvest?

DOMHarvest makes web scraping simple and reliable by leveraging Playwright's battle-tested browser automation. Whether you're building a data pipeline, monitoring websites, or extracting content for analysis, DOMHarvest provides the tools you need with minimal setup.

## Features at a Glance

- **Declarative DSL**: Use `text()`, `attr()`, `array()`, `exists()`, `html()`, `count()` for clean, readable extraction
- **Authentication helpers**: `login()`, `fillLoginForm()`, `SessionManager` for authenticated scraping
- **Rate limiting**: Built-in global and per-domain rate limiting to avoid overwhelming servers
- **Retry with backoff**: Automatic retries with exponential or linear backoff strategies
- **Batch harvesting**: Process multiple URLs concurrently with `harvestBatch()`
- **Custom extractors**: Mix DSL with custom functions when you need complex logic
- **Screenshot capture**: Take screenshots during or after scraping operations
- **Proxy support**: Route requests through proxy servers with authentication
- **Full Playwright access**: Direct access to browser, context, and page for advanced use cases
