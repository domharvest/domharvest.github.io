---
layout: post
title: "Web Scraping Made Simple: A Practical Guide to domharvest-playwright"
date: 2025-01-04
categories: ["web-scraping", "playwright", "tutorial", "javascript"]
---

# Web Scraping Made Simple: A Practical Guide to domharvest-playwright

Web scraping doesn't need to be complicated. While powerful tools like Playwright give you complete control over browser automation, they often require significant boilerplate code for common scraping tasks. That's where **domharvest-playwright** comes in—a focused library that makes DOM extraction intuitive without sacrificing Playwright's power.

In this practical guide, I'll walk you through everything you need to know to start scraping websites efficiently with domharvest-playwright.

## What is domharvest-playwright?

domharvest-playwright is a lightweight wrapper around Playwright that specializes in one thing: extracting structured data from web pages. It provides a clean, intuitive API for the most common scraping scenarios while still giving you access to Playwright's full capabilities when you need them.

**Key benefits:**
- Simple one-function API for quick scraping tasks
- Automatic resource management (no forgotten browser instances)
- CSS selector-based extraction with custom transformations
- Cross-browser support (Chromium, Firefox, WebKit)
- Built with modern JavaScript (ES modules)

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

## Scraping Product Data

Let's try something more complex: extracting product information from an e-commerce site. We'll use [books.toscrape.com](https://books.toscrape.com/):

```javascript
import { harvest } from 'domharvest-playwright'

const books = await harvest(
  'https://books.toscrape.com/',
  '.product_pod',
  (el) => ({
    title: el.querySelector('h3 a')?.getAttribute('title'),
    price: el.querySelector('.price_color')?.textContent?.trim(),
    availability: el.querySelector('.availability')?.textContent?.trim(),
    rating: el.querySelector('.star-rating')?.className.split(' ')[1],
    imageUrl: el.querySelector('img')?.getAttribute('src')
  })
)

console.log(`Found ${books.length} books`)
console.log(books[0])
```

**Output:**
```javascript
{
  title: 'A Light in the Attic',
  price: '£51.77',
  availability: 'In stock',
  rating: 'Three',
  imageUrl: 'media/cache/2c/da/2cdad67c44b002e7ead0cc35693c0e8b.jpg'
}
```

This demonstrates extracting multiple data points from each product, including attributes and nested elements.

## Advanced: Scraping Multiple Pages

For real-world scraping, you often need to visit multiple pages. When doing this, use the `DOMHarvester` class to reuse the same browser instance—it's much more efficient than creating a new browser for each page.

```javascript
import { DOMHarvester } from 'domharvest-playwright'

async function scrapeMultiplePages(maxPages = 3) {
  const harvester = new DOMHarvester({ headless: true })
  await harvester.init()

  const allQuotes = []

  try {
    for (let page = 1; page <= maxPages; page++) {
      console.log(`Scraping page ${page}...`)

      const quotes = await harvester.harvest(
        `https://quotes.toscrape.com/page/${page}/`,
        '.quote',
        (el) => ({
          text: el.querySelector('.text')?.textContent?.trim(),
          author: el.querySelector('.author')?.textContent?.trim()
        })
      )

      allQuotes.push(...quotes)

      // Be respectful: wait 1 second between requests
      await new Promise(resolve => setTimeout(resolve, 1000))
    }

    console.log(`Total quotes collected: ${allQuotes.length}`)
    return allQuotes
  } finally {
    // Always close the browser to prevent memory leaks
    await harvester.close()
  }
}

const quotes = await scrapeMultiplePages(3)
```

**Key points:**
- Create a `DOMHarvester` instance once
- Call `init()` before using it
- Reuse it for multiple pages
- Always use `try/finally` to ensure cleanup
- Add delays between requests to be respectful to the server

## Filtering and Transforming Data

Often, you don't want all the data—you want to filter and transform it. Let's find only top-rated books that are in stock:

```javascript
import { DOMHarvester } from 'domharvest-playwright'

const harvester = new DOMHarvester()
await harvester.init()

try {
  const books = await harvester.harvest(
    'https://books.toscrape.com/',
    '.product_pod',
    (el) => ({
      title: el.querySelector('h3 a')?.getAttribute('title'),
      price: parseFloat(
        el.querySelector('.price_color')?.textContent?.replace(/[^0-9.]/g, '') || '0'
      ),
      rating: el.querySelector('.star-rating')?.className.split(' ')[1],
      inStock: el.querySelector('.availability')?.textContent?.includes('In stock')
    })
  )

  // Filter for top-rated books in stock
  const topBooks = books
    .filter(book => book.inStock && ['Four', 'Five'].includes(book.rating))
    .sort((a, b) => b.price - a.price)

  console.log(`Found ${topBooks.length} top-rated books in stock:`)
  topBooks.forEach(book => {
    console.log(`- ${book.title} (${book.rating} stars) - £${book.price}`)
  })
} finally {
  await harvester.close()
}
```

This example shows how to:
- Parse numeric data (converting price string to number)
- Filter results based on multiple criteria
- Sort the results

## Custom Page Analysis with harvestCustom()

Sometimes you need more control than CSS selectors provide. The `harvestCustom()` method lets you run arbitrary JavaScript in the browser context:

```javascript
import { DOMHarvester } from 'domharvest-playwright'

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
        allTags: Array.from(new Set(
          Array.from(document.querySelectorAll('.tag'))
            .map(t => t.textContent?.trim())
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

## Extracting Table Data

Tables are a common source of structured data. Here's how to handle them:

```javascript
import { DOMHarvester } from 'domharvest-playwright'

const harvester = new DOMHarvester()
await harvester.init()

try {
  const tableData = await harvester.harvestCustom(
    'https://example.com/data-table',
    () => {
      const rows = Array.from(document.querySelectorAll('table tbody tr'))

      return rows.map(tr => {
        const cells = Array.from(tr.querySelectorAll('td'))
        return {
          column1: cells[0]?.textContent?.trim(),
          column2: cells[1]?.textContent?.trim(),
          column3: cells[2]?.textContent?.trim()
        }
      })
    }
  )

  console.log(`Extracted ${tableData.length} rows`)
} finally {
  await harvester.close()
}
```

## Saving Your Data

Once you've scraped the data, you'll want to save it. Here's how to export to JSON and CSV:

```javascript
import { harvest } from 'domharvest-playwright'
import { writeFile } from 'fs/promises'

const quotes = await harvest(
  'https://quotes.toscrape.com/',
  '.quote',
  (el) => ({
    text: el.querySelector('.text')?.textContent?.trim(),
    author: el.querySelector('.author')?.textContent?.trim(),
    tags: Array.from(el.querySelectorAll('.tag')).map(tag => tag.textContent?.trim())
  })
)

// Save as JSON
await writeFile(
  'quotes.json',
  JSON.stringify(quotes, null, 2),
  'utf-8'
)

// Save as CSV
const csv = [
  'text,author,tags',
  ...quotes.map(q =>
    `"${q.text}","${q.author}","${q.tags.join('; ')}"`
  )
].join('\n')

await writeFile('quotes.csv', csv, 'utf-8')

console.log(`Saved ${quotes.length} quotes to files`)
```

## Configuration Options

domharvest-playwright provides two main configuration options:

### headless Mode

By default, the browser runs in headless mode (invisible). For debugging, you can make it visible:

```javascript
const harvester = new DOMHarvester({ headless: false })
```

This lets you watch the browser navigate and interact with pages, which is invaluable for troubleshooting.

### timeout

The default timeout is 30 seconds. Adjust it based on your target sites:

```javascript
// For fast, static pages
const fastHarvester = new DOMHarvester({ timeout: 10000 })

// For slow, dynamic content
const slowHarvester = new DOMHarvester({ timeout: 60000 })
```

## Error Handling

Always handle errors gracefully to avoid crashes:

```javascript
import { DOMHarvester } from 'domharvest-playwright'

const harvester = new DOMHarvester()

try {
  await harvester.init()

  const data = await harvester.harvest(
    'https://example.com',
    '.some-selector',
    (el) => ({ text: el.textContent })
  )

  console.log('Success:', data)
} catch (error) {
  console.error('Scraping failed:', error.message)

  // Handle specific errors
  if (error.message.includes('timeout')) {
    console.error('Page took too long to load. Try increasing timeout.')
  }
} finally {
  // Always close, even if there was an error
  await harvester.close()
}
```

## Best Practices

Based on my experience with domharvest-playwright, here are some key recommendations:

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

### 2. Reuse Harvester Instances

For multiple pages, create one instance and reuse it:

```javascript
// Good: One browser instance
const harvester = new DOMHarvester()
await harvester.init()
for (const url of urls) {
  await harvester.harvest(url, selector, extractor)
}
await harvester.close()

// Bad: Creates a new browser for each page
for (const url of urls) {
  const data = await harvest(url, selector, extractor)
}
```

### 3. Be Respectful with Rate Limiting

Always add delays between requests:

```javascript
for (const url of urls) {
  await harvester.harvest(url, selector, extractor)
  await new Promise(resolve => setTimeout(resolve, 1000)) // 1 second delay
}
```

### 4. Check robots.txt

Before scraping a site, check its `robots.txt` file to ensure you're allowed to scrape it:

```
https://example.com/robots.txt
```

### 5. Use harvest() for Quick Tasks

For one-off scraping, the simple function is perfect:

```javascript
// Quick and clean
const data = await harvest(url, selector, extractor)
```

### 6. Use DOMHarvester for Multiple Operations

For anything more complex, use the class:

```javascript
const harvester = new DOMHarvester()
await harvester.init()
// ... multiple operations
await harvester.close()
```

## Troubleshooting Common Issues

### Browser Installation Failed

If Playwright can't install browsers, try:

```bash
# Use sudo on Linux/macOS
sudo npx playwright install

# Or set a custom directory
PLAYWRIGHT_BROWSERS_PATH=~/pw-browsers npx playwright install
```

### Timeout Errors

If pages are timing out:

1. Increase the timeout: `new DOMHarvester({ timeout: 60000 })`
2. Check if the page loads in a regular browser
3. Try with headless mode off to see what's happening

### Elements Not Found

If your selector returns empty results:

1. Run with `headless: false` to inspect the page
2. Use browser DevTools to test your selectors
3. Check if content loads dynamically (might need `harvestCustom()` with waits)

### Memory Leaks

If your script consumes increasing memory:

1. Ensure you're calling `close()` in a `finally` block
2. Don't create multiple harvester instances unnecessarily
3. For long-running processes, consider restarting the harvester periodically

## Real-World Example: Price Monitoring

Let's build a practical price monitoring scraper:

```javascript
import { DOMHarvester } from 'domharvest-playwright'
import { writeFile } from 'fs/promises'

async function monitorPrices(urls) {
  const harvester = new DOMHarvester({ headless: true })
  await harvester.init()

  const results = []

  try {
    for (const url of urls) {
      console.log(`Checking ${url}...`)

      try {
        const product = await harvester.harvestCustom(url, () => {
          return {
            title: document.querySelector('h1')?.textContent?.trim(),
            price: document.querySelector('.price')?.textContent?.trim(),
            availability: document.querySelector('.stock')?.textContent?.trim(),
            timestamp: new Date().toISOString()
          }
        })

        results.push({ url, ...product, error: null })
      } catch (error) {
        results.push({ url, error: error.message })
      }

      // Rate limiting
      await new Promise(resolve => setTimeout(resolve, 2000))
    }

    // Save results
    await writeFile(
      `prices-${Date.now()}.json`,
      JSON.stringify(results, null, 2)
    )

    console.log(`Monitored ${results.length} products`)
    return results
  } finally {
    await harvester.close()
  }
}

// Usage
const urls = [
  'https://books.toscrape.com/catalogue/a-light-in-the-attic_1000/index.html',
  'https://books.toscrape.com/catalogue/tipping-the-velvet_999/index.html'
]

const prices = await monitorPrices(urls)
```

This example demonstrates:
- Handling multiple URLs
- Error handling per URL (doesn't fail if one URL errors)
- Timestamping data
- Saving results to dated files
- Rate limiting between requests

## Next Steps

Now that you understand the basics of domharvest-playwright, here are some ways to level up your scraping skills:

1. **Explore Dynamic Content**: Learn how to handle JavaScript-heavy SPAs using direct Playwright API access
2. **Handle Authentication**: Use Playwright's context to log in before scraping
3. **Work with APIs**: Consider using the Network tab to find APIs that might be easier than scraping HTML
4. **Build Robust Scrapers**: Add retry logic, better error handling, and monitoring
5. **Scale Your Scraping**: Look into proxies and distributed scraping for large-scale projects

## Conclusion

domharvest-playwright strikes an excellent balance between simplicity and power. The `harvest()` function makes quick scraping tasks trivial, while the `DOMHarvester` class and direct Playwright access give you the flexibility to handle complex scenarios.

Key takeaways:
- Use `harvest()` for simple, one-off scraping tasks
- Use `DOMHarvester` when scraping multiple pages
- Always close resources with `try/finally`
- Be respectful: add rate limiting and check `robots.txt`
- Start with `headless: false` when debugging

Whether you're building a one-off data extraction script or a production scraping pipeline, domharvest-playwright provides the tools you need without the complexity you don't.

Happy harvesting!

---

**Resources:**
- [GitHub Repository](https://github.com/domharvest/domharvest-playwright)
- [npm Package](https://www.npmjs.com/package/domharvest-playwright)
- [Full Documentation](https://domharvest.github.io/domharvest-playwright/)
- [Practice Sites](https://toscrape.com/)

*Questions or feedback? Reach out on [GitHub](https://github.com/domharvest) or [Mastodon](https://infosec.exchange/@domharvest)!*
