---
layout: post
title: "Building domharvest-playwright: A Modern DOM Harvesting Tool"
date: 2025-01-08
categories: ["web-scraping", "playwright", "javascript", "open-source"]
---

# Building domharvest-playwright: A Modern DOM Harvesting Tool

## Introduction

Web scraping doesn't have to be complicated. Yet, every time I started a new data extraction project, I found myself writing the same boilerplate code: launching browsers, navigating pages, waiting for elements, extracting data, and cleaning up resources. While Playwright is an excellent browser automation framework, it's designed as a general-purpose tool for testing and automation, not specifically for DOM harvesting workflows.

That's why I built **domharvest-playwright**: a focused library that wraps Playwright's power into a simple, intuitive API specifically designed for extracting structured data from websites.

## The Challenge

Modern web scraping presents several recurring challenges:

1. **Boilerplate Overhead**: Every scraping script needs browser initialization, navigation, waiting logic, and cleanup code
2. **Inconsistent Patterns**: Different developers solve the same problems in different ways, making code harder to maintain
3. **Error-Prone Resource Management**: Forgetting to close browsers or pages can lead to memory leaks in production
4. **Complex Extraction Logic**: Transforming raw DOM elements into structured data requires repetitive querySelector chains and null checks

While tools like Puppeteer and Playwright solve browser automation, they don't provide opinionated patterns for the specific use case of DOM harvesting. I wanted a tool that would make the common case simple while still allowing full control when needed.

## Why Playwright?

Choosing the right foundation was critical. After years of working with various browser automation tools, Playwright emerged as the clear choice for several reasons:

- **Modern Architecture**: Built from the ground up for modern web applications with native support for SPAs, dynamic content, and JavaScript-heavy sites
- **Multi-Browser Support**: Test and scrape across Chromium, Firefox, and WebKit with a unified API
- **Superior Performance**: Faster execution and better resource management compared to older tools like Selenium
- **Active Development**: Backed by Microsoft with regular updates, security patches, and new features
- **Auto-Waiting**: Intelligent waiting mechanisms that reduce flakiness and eliminate most explicit wait statements
- **Network Control**: Intercept and modify requests, perfect for bypassing certain anti-scraping measures

Playwright's battle-tested reliability made it the ideal foundation for a production-ready scraping tool.

## Key Features

### 1. Simple Function-Based API

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

### 2. Class-Based Interface for Complex Workflows

When you need more control, the class-based API gives you full access to the browser instance:

```javascript
const harvester = new DOMHarvester()
await harvester.init()

const data = await harvester.harvest(url, selector, transformFn)
// Perform multiple operations with the same browser instance
const moreData = await harvester.harvestCustom(url, customPageFunction)

await harvester.close()
```

### 3. Custom Page Analysis

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

### 4. Configurable Behavior

Every aspect of the harvesting process can be customized:

- **Headless/Headed Mode**: Debug visually or run in production headless mode
- **Timeouts**: Configure wait times for slow-loading pages
- **Browser Selection**: Choose between Chromium, Firefox, or WebKit
- **Custom Selectors**: Use any valid CSS selector for element targeting

## Architecture Overview

The library follows a layered architecture designed for simplicity and flexibility:

```
┌─────────────────────────────────────┐
│   High-Level API (harvest())       │  ← Simple function for common cases
├─────────────────────────────────────┤
│   DOMHarvester Class                │  ← Object-oriented interface
├─────────────────────────────────────┤
│   Playwright Browser Management     │  ← Browser lifecycle handling
├─────────────────────────────────────┤
│   Page Navigation & Evaluation      │  ← DOM interaction layer
├─────────────────────────────────────┤
│   Playwright Core (Browser Drivers) │  ← Foundation
└─────────────────────────────────────┘
```

**Key Design Decisions:**

1. **ES Modules**: Modern JavaScript with native module support for better tree-shaking and compatibility
2. **Automatic Cleanup**: Resource management handled automatically to prevent memory leaks
3. **Function Serialization**: Transform functions are serialized and executed in the browser context for maximum flexibility
4. **Minimal Dependencies**: Only Playwright as a dependency keeps the package lean and maintainable

## Lessons Learned

Building domharvest-playwright taught me valuable lessons about API design and web scraping:

1. **Simplicity Wins**: The most powerful feature is often the simplest API. Users gravitate toward the one-line `harvest()` function even though the class-based API offers more control. Making the common case trivial is more important than exposing every feature upfront.

2. **Resource Management is Critical**: In production environments, forgotten browser instances can quickly consume all available memory. Automatic cleanup isn't just a convenience—it's essential for reliability. Every code path must guarantee resource cleanup, even on errors.

3. **Context Switching is Tricky**: Serializing functions to run in the browser context introduces subtle gotchas. Variables from the outer scope don't transfer, and debugging is harder. Clear documentation and good error messages are essential to help users understand the execution model.

4. **Testing Matters for Scraping Tools**: Practice websites like quotes.toscrape.com and books.toscrape.com are invaluable for testing without ethical concerns. They let you develop and test scraping logic without worrying about rate limiting, legal issues, or changing production sites.

5. **Standards Improve Code Quality**: Adopting JavaScript Standard Style from day one eliminated bikeshedding about formatting and caught subtle bugs through linting. Consistency makes the codebase easier to maintain and contribute to.

## Future Roadmap

The library is production-ready, but there's always room for improvement. Here's what's on the horizon:

- **Retry Logic**: Automatic retries with exponential backoff for flaky networks or rate-limited sites
- **Parallel Harvesting**: Built-in support for concurrent scraping of multiple URLs with configurable concurrency limits
- **Data Export Formats**: Native support for exporting to JSON, CSV, and other common formats
- **Request Interception Helpers**: Simplified API for blocking images, stylesheets, or other resources to speed up scraping
- **Screenshot & PDF Generation**: Capture visual content alongside structured data
- **Proxy Support**: Easy configuration of proxy servers for distributed scraping
- **Performance Metrics**: Built-in timing and performance tracking for optimization

Contributions and feature requests are welcome on GitHub!

## Getting Started

Install via npm:

```bash
npm install domharvest-playwright
```

Check out the [documentation](https://domharvest.github.io/domharvest-playwright/) for detailed usage examples.

## Contributing

The project is open source and welcomes contributions! Visit the [GitHub repository](https://github.com/domharvest/domharvest-playwright) to get involved.

## Conclusion

Building domharvest-playwright has been a journey in creating developer-friendly abstractions without sacrificing power. By focusing on the most common web scraping workflows and wrapping them in an intuitive API, the library makes it easier to build reliable data extraction pipelines.

The goal wasn't to replace Playwright but to complement it—providing a higher-level interface for DOM harvesting while still allowing access to Playwright's full capabilities when needed. Whether you're building a one-off scraper or a production data pipeline, domharvest-playwright aims to get you started quickly and scale with your needs.

If you're working with web scraping or data extraction, give it a try and let me know what you think. The project is open source and actively maintained, and I'm always looking for feedback and contributions from the community.

Happy harvesting!

---

**Links:**
- [GitHub Repository](https://github.com/domharvest/domharvest-playwright)
- [npm Package](https://www.npmjs.com/package/domharvest-playwright)
- [Documentation](https://domharvest.github.io/domharvest-playwright/)

*Have questions or suggestions? Reach out on [GitHub](https://github.com/domharvest) or [Mastodon](https://infosec.exchange/@domharvest)!*
