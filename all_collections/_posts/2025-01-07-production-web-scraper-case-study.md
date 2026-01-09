---
layout: post
title: "Building a Production Web Scraper: Real-World Lessons with domharvest-playwright"
date: 2025-01-07
categories: ["web-scraping", "playwright", "case-study", "production"]
---

# Building a Production Web Scraper: Real-World Lessons with domharvest-playwright

Theory is great, but nothing beats learning from a real project. In this case study, I'll walk you through building a production-grade web scraper using domharvest-playwright—complete with the challenges I faced, mistakes I made, and solutions that actually worked.

The goal: Build a competitive intelligence tool that monitors product listings across multiple e-commerce sites, tracking prices, availability, and product details. This needed to run reliably 24/7, handle thousands of products, and alert on significant changes.

Let's dive into what it really takes to build production-ready web scraping infrastructure.

## The Requirements

Before writing any code, I defined clear requirements:

**Functional Requirements:**
- Monitor 5,000+ products across 10 e-commerce sites
- Scrape each product every 4 hours
- Track price, availability, ratings, and descriptions
- Detect and alert on significant changes (>10% price change, out of stock)
- Export data to PostgreSQL for analysis
- Provide a REST API for accessing current data

**Non-Functional Requirements:**
- 99.5% uptime target
- Complete scraping cycle within 2 hours
- Handle site changes without manual intervention
- Respect rate limits (2 requests/second per domain)
- Memory usage under 2GB
- Cost under $50/month (using DigitalOcean droplet)

## Why domharvest-playwright?

I evaluated several options:

**Cheerio**: Fast but can't handle JavaScript-rendered content. Many modern e-commerce sites are SPAs—deal breaker.

**Puppeteer**: Powerful but required more boilerplate. I wanted to focus on extraction logic, not browser management.

**Scrapy**: Excellent for large-scale scraping, but Python-based and requires learning a new ecosystem. I wanted to stay in Node.js.

**domharvest-playwright**: Perfect balance—handles SPAs, minimal boilerplate, and gives me Playwright's power when I need it. Most importantly, I could prototype quickly and scale later.

## Architecture Overview

Here's the high-level architecture I designed:

```
┌──────────────┐      ┌──────────────┐      ┌──────────────┐
│   Scheduler  │─────▶│   Scraper    │─────▶│  PostgreSQL  │
│  (node-cron) │      │  (domharvest)│      │  (Database)  │
└──────────────┘      └──────────────┘      └──────────────┘
                             │
                             ▼
                      ┌──────────────┐
                      │ Alert System │
                      │  (Email/SMS) │
                      └──────────────┘
```

**Components:**
1. **Scheduler**: Manages scraping schedule, queues products
2. **Scraper**: Extracts data using domharvest-playwright
3. **Database**: Stores historical data, enables change detection
4. **Alert System**: Notifies on significant changes

## Implementation: Phase 1 - The Naive Approach

I started with the simplest possible implementation:

```javascript
import { DOMHarvester } from 'domharvest-playwright'
import { Pool } from 'pg'

const db = new Pool({ connectionString: process.env.DATABASE_URL })

async function scrapeProduct(url) {
  const harvester = new DOMHarvester()
  await harvester.init()

  try {
    const product = await harvester.harvestCustom(url, () => {
      return {
        title: document.querySelector('h1')?.textContent?.trim(),
        price: document.querySelector('.price')?.textContent?.trim(),
        inStock: !document.querySelector('.out-of-stock'),
        rating: document.querySelector('.rating')?.textContent?.trim()
      }
    })

    await db.query(
      'INSERT INTO products (url, title, price, in_stock, rating, scraped_at) VALUES ($1, $2, $3, $4, $5, NOW())',
      [url, product.title, product.price, product.inStock, product.rating]
    )

    return product
  } finally {
    await harvester.close()
  }
}

// Scrape all products
const products = await db.query('SELECT url FROM products')
for (const row of products.rows) {
  await scrapeProduct(row.url)
}
```

**What went wrong:**

1. **Memory explosion**: Creating a new browser for each product meant 5,000 browser instances. My 2GB droplet ran out of memory after 50 products.

2. **Extremely slow**: Each browser launch took 2-3 seconds. At this rate, scraping 5,000 products would take 4+ hours.

3. **No error handling**: If one product failed, the entire script crashed.

4. **No rate limiting**: I got IP-banned from two sites within 10 minutes.

**Time to first failure: 8 minutes.**

It was clear I needed a better approach.

## Phase 2 - Reusing Browser Instances

First fix: Reuse the browser instance:

```javascript
async function scrapeProducts(urls) {
  const harvester = new DOMHarvester({ headless: true })
  await harvester.init()

  const results = []

  try {
    for (const url of urls) {
      try {
        const product = await harvester.harvestCustom(url, () => {
          return {
            title: document.querySelector('h1')?.textContent?.trim(),
            price: document.querySelector('.price')?.textContent?.trim(),
            inStock: !document.querySelector('.out-of-stock'),
            rating: document.querySelector('.rating')?.textContent?.trim()
          }
        })

        results.push({ url, ...product, error: null })
      } catch (error) {
        console.error(`Failed to scrape ${url}: ${error.message}`)
        results.push({ url, error: error.message })
      }

      // Rate limiting: 2 requests per second
      await new Promise(resolve => setTimeout(resolve, 500))
    }
  } finally {
    await harvester.close()
  }

  return results
}
```

**Improvements:**
- Memory usage dropped from 2GB+ to ~400MB
- Speed improved dramatically (no browser launches per product)
- Per-product error handling prevents cascading failures
- Basic rate limiting avoids IP bans

**But new problems emerged:**

1. **Single point of failure**: If the browser crashes, all scraping stops
2. **No parallelization**: Still slow—~7 hours for 5,000 products
3. **Selector brittleness**: Hard-coded selectors break when sites change

## Phase 3 - Production-Grade Implementation

Here's the final, production-ready implementation:

### Step 1: Site-Specific Extractors

I created a configuration system for different sites:

```javascript
// config/extractors.js
export const extractors = {
  'example-shop.com': {
    selector: '.product-details',
    extract: (el) => ({
      title: el.querySelector('h1.product-title')?.textContent?.trim(),
      price: parseFloat(
        el.querySelector('.price .amount')?.textContent?.replace(/[^0-9.]/g, '') || '0'
      ),
      inStock: el.querySelector('.stock-status')?.textContent?.includes('In stock'),
      rating: parseFloat(
        el.querySelector('.star-rating')?.getAttribute('aria-label')?.match(/[\d.]+/)?.[0] || '0'
      ),
      description: el.querySelector('.product-description')?.textContent?.trim()
    })
  },

  'another-store.com': {
    selector: '.product-container',
    extract: (el) => ({
      title: el.querySelector('.product-name')?.textContent?.trim(),
      price: parseFloat(
        el.querySelector('.product-price')?.textContent?.replace(/[^0-9.]/g, '') || '0'
      ),
      inStock: !el.querySelector('.out-of-stock-badge'),
      rating: el.querySelectorAll('.rating-star.filled').length,
      description: el.querySelector('.description-text')?.textContent?.trim()
    })
  }
}

export function getExtractor(url) {
  const domain = new URL(url).hostname
  return extractors[domain] || extractors['default']
}
```

This makes the scraper resilient to site-specific differences and easier to maintain when sites change.

### Step 2: Worker Pool Pattern

I implemented a worker pool for parallel scraping:

```javascript
import { DOMHarvester } from 'domharvest-playwright'
import pLimit from 'p-limit'

class ScraperPool {
  constructor(size = 5) {
    this.size = size
    this.harvesters = []
    this.limit = pLimit(size)
  }

  async init() {
    // Create a pool of harvesters
    for (let i = 0; i < this.size; i++) {
      const harvester = new DOMHarvester({ headless: true })
      await harvester.init()
      this.harvesters.push(harvester)
    }
  }

  async scrape(urls) {
    const results = await Promise.all(
      urls.map(url =>
        this.limit(() => this.scrapeOne(url))
      )
    )
    return results
  }

  async scrapeOne(url) {
    // Round-robin: distribute work across harvesters
    const harvester = this.harvesters[Math.floor(Math.random() * this.size)]

    try {
      const extractor = getExtractor(url)

      const product = await harvester.harvestCustom(url, extractor.extract)

      return { url, ...product, error: null }
    } catch (error) {
      return { url, error: error.message }
    }
  }

  async close() {
    await Promise.all(this.harvesters.map(h => h.close()))
  }
}
```

**Benefits:**
- 5 concurrent scrapers = 5x faster
- Each harvester has its own browser context (isolation)
- Failed scrapers don't affect others
- Configurable concurrency

**Results:** 5,000 products now scrape in ~1.5 hours.

### Step 3: Retry Logic with Exponential Backoff

Network errors are inevitable. Retry logic is essential:

```javascript
async function scrapeWithRetry(url, maxRetries = 3) {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await this.scrapeOne(url)
    } catch (error) {
      if (attempt === maxRetries) {
        return { url, error: `Failed after ${maxRetries} attempts: ${error.message}` }
      }

      const delay = Math.min(1000 * Math.pow(2, attempt), 10000)
      console.log(`Attempt ${attempt} failed for ${url}, retrying in ${delay}ms...`)
      await new Promise(resolve => setTimeout(resolve, delay))
    }
  }
}
```

**Pattern:** Exponential backoff (1s, 2s, 4s) with a maximum of 10s.

### Step 4: Change Detection and Alerts

The database layer handles change detection:

```javascript
import { Pool } from 'pg'
import nodemailer from 'nodemailer'

const db = new Pool({ connectionString: process.env.DATABASE_URL })
const mailer = nodemailer.createTransport(process.env.SMTP_URL)

async function saveAndDetectChanges(product) {
  // Get previous data
  const previous = await db.query(
    'SELECT price, in_stock FROM products WHERE url = $1 ORDER BY scraped_at DESC LIMIT 1',
    [product.url]
  )

  // Insert new data
  await db.query(
    'INSERT INTO products (url, title, price, in_stock, rating, scraped_at) VALUES ($1, $2, $3, $4, $5, NOW())',
    [product.url, product.title, product.price, product.inStock, product.rating]
  )

  // Detect significant changes
  if (previous.rows.length > 0) {
    const prev = previous.rows[0]
    const changes = []

    // Price change > 10%
    if (Math.abs(product.price - prev.price) / prev.price > 0.10) {
      changes.push(`Price changed from $${prev.price} to $${product.price}`)
    }

    // Stock status change
    if (product.inStock !== prev.in_stock) {
      changes.push(`Stock status: ${prev.in_stock ? 'In stock' : 'Out of stock'} → ${product.inStock ? 'In stock' : 'Out of stock'}`)
    }

    // Send alerts
    if (changes.length > 0) {
      await sendAlert(product, changes)
    }
  }
}

async function sendAlert(product, changes) {
  await mailer.sendMail({
    from: 'alerts@scraper.com',
    to: 'me@example.com',
    subject: `Product Alert: ${product.title}`,
    text: `
      Product: ${product.title}
      URL: ${product.url}

      Changes detected:
      ${changes.join('\n')}
    `
  })
}
```

### Step 5: Error Monitoring and Health Checks

Production systems need observability:

```javascript
import { createLogger, format, transports } from 'winston'

const logger = createLogger({
  level: 'info',
  format: format.combine(
    format.timestamp(),
    format.errors({ stack: true }),
    format.json()
  ),
  transports: [
    new transports.File({ filename: 'error.log', level: 'error' }),
    new transports.File({ filename: 'combined.log' })
  ]
})

class MonitoredScraperPool extends ScraperPool {
  constructor(size) {
    super(size)
    this.stats = {
      total: 0,
      successful: 0,
      failed: 0,
      startTime: null
    }
  }

  async scrape(urls) {
    this.stats.startTime = Date.now()
    this.stats.total = urls.length

    const results = await super.scrape(urls)

    results.forEach(result => {
      if (result.error) {
        this.stats.failed++
        logger.error('Scrape failed', { url: result.url, error: result.error })
      } else {
        this.stats.successful++
        logger.info('Scrape succeeded', { url: result.url })
      }
    })

    return results
  }

  getStats() {
    const duration = Date.now() - this.stats.startTime
    return {
      ...this.stats,
      duration,
      successRate: (this.stats.successful / this.stats.total * 100).toFixed(2) + '%',
      averageTime: (duration / this.stats.total).toFixed(2) + 'ms'
    }
  }
}
```

### Step 6: Scheduling with node-cron

Finally, automate with a scheduler:

```javascript
import cron from 'node-cron'
import { Pool } from 'pg'

const db = new Pool({ connectionString: process.env.DATABASE_URL })

// Run every 4 hours
cron.schedule('0 */4 * * *', async () => {
  console.log('Starting scheduled scrape...')

  try {
    // Get all product URLs from database
    const result = await db.query('SELECT url FROM product_urls WHERE active = true')
    const urls = result.rows.map(row => row.url)

    // Create scraper pool
    const pool = new MonitoredScraperPool(5)
    await pool.init()

    try {
      // Scrape all products
      const results = await pool.scrape(urls)

      // Save results
      for (const product of results) {
        if (!product.error) {
          await saveAndDetectChanges(product)
        }
      }

      // Log stats
      const stats = pool.getStats()
      console.log('Scrape complete:', stats)

      // Alert on low success rate
      if (stats.successful / stats.total < 0.95) {
        await sendAlert({
          title: 'Scraper Health Alert',
          url: 'N/A'
        }, [
          `Success rate below threshold: ${stats.successRate}`,
          `Failed: ${stats.failed} / ${stats.total}`
        ])
      }
    } finally {
      await pool.close()
    }
  } catch (error) {
    logger.error('Scheduled scrape failed', { error: error.message, stack: error.stack })
  }
})

console.log('Scraper scheduler started')
```

## Production Lessons Learned

After running this in production for several months, here are the key lessons:

### 1. Selectors Will Break

**Reality:** Sites change their HTML frequently. Hard-coded selectors break constantly.

**Solution:**
- Implement health checks that detect scraping failures
- Use multiple selector strategies (fallbacks)
- Monitor success rates and alert on drops
- Build a selector update workflow (test new selectors against live sites)

### 2. Rate Limiting is Non-Negotiable

**Reality:** Sites will ban you if you scrape too aggressively.

**Solution:**
- Respect `robots.txt`
- Add delays between requests (500ms-2s minimum)
- Rotate user agents
- Consider proxies for high-volume scraping
- Monitor for 429 (Too Many Requests) responses

### 3. Memory Leaks Are Real

**Reality:** Even with proper cleanup, long-running scrapers leak memory.

**Solution:**
- Restart scraper pools periodically (every 1,000 pages)
- Monitor memory usage with process metrics
- Use Docker containers with memory limits
- Implement graceful restarts

### 4. Error Handling is Critical

**Reality:** Networks fail, sites go down, parsing breaks.

**Solution:**
- Never let one error crash the entire system
- Implement retry logic with exponential backoff
- Log everything for debugging
- Alert on high error rates
- Build self-healing mechanisms

### 5. Proxy Management is Complex

**Reality:** For high-volume scraping, you'll need proxies.

**What I learned:**
- Free proxies are unreliable (90%+ failure rate)
- Paid residential proxies work but are expensive ($5-15/GB)
- Rotating proxies help avoid bans
- Some sites detect and block data center proxies
- IP rotation strategies matter (sticky sessions vs pure rotation)

I ended up using Bright Data's residential proxies at ~$500/month for the proxy service alone, but it was worth it for reliability.

### 6. Data Quality Matters

**Reality:** Extracted data is messy and inconsistent.

**Solution:**
```javascript
function normalizeProduct(raw) {
  return {
    title: raw.title?.trim() || 'Unknown',
    price: parseFloat(raw.price?.replace(/[^0-9.]/g, '')) || 0,
    inStock: Boolean(raw.inStock),
    rating: Math.max(0, Math.min(5, parseFloat(raw.rating) || 0)),
    scrapedAt: new Date().toISOString()
  }
}
```

Always normalize, validate, and sanitize extracted data.

### 7. Monitoring is Everything

**Metrics I track:**
- Success rate per domain
- Average scraping time
- Memory usage
- Error rates by type
- Products out of stock (business metric)
- Price changes detected (business metric)

**Tools:**
- Winston for logging
- Prometheus for metrics
- Grafana for visualization
- PagerDuty for alerts

## Cost Breakdown

Here's what it actually costs to run this in production:

**Infrastructure:**
- DigitalOcean Droplet (4GB RAM, 2 vCPU): $24/month
- PostgreSQL Database (2GB): $15/month
- Backups and snapshots: $5/month

**Services:**
- Bright Data proxy service: $500/month (for 5,000 products)
- Email service (SendGrid): $10/month
- Monitoring (Grafana Cloud): Free tier

**Total: ~$554/month**

The proxy service is by far the biggest cost. For smaller projects or sites that don't aggressively block scrapers, you can skip proxies and bring costs down to ~$50/month.

## Performance Metrics

After optimization, here's what I achieved:

**Scraping Performance:**
- 5,000 products scraped in 1.5 hours
- 5 concurrent workers
- ~1.1 seconds per product
- 99.2% success rate

**System Reliability:**
- 99.7% uptime over 6 months
- Average of 3 alerts per week
- 2 critical failures (both due to site redesigns)

**Business Impact:**
- Detected 1,200+ significant price changes
- Tracked 400+ stock-out events
- Saved client $50k+ through competitive pricing intelligence

## Final Architecture

Here's the complete production system:

```
┌─────────────────────────────────────────────────────────────┐
│                       Scheduler (node-cron)                 │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                    ScraperPool (5 workers)                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │ Harvester 1  │  │ Harvester 2  │  │ Harvester 3  │ ...  │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└────────────────────────┬────────────────────────────────────┘
                         │
         ┌───────────────┼───────────────┐
         ▼               ▼               ▼
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│ PostgreSQL  │  │   Winston   │  │  Nodemailer │
│  (Storage)  │  │  (Logging)  │  │   (Alerts)  │
└─────────────┘  └─────────────┘  └─────────────┘
```

## Conclusion

Building production web scrapers is harder than it looks. The challenges aren't in the scraping itself—domharvest-playwright makes that easy—but in the surrounding infrastructure: error handling, monitoring, rate limiting, data validation, and operational concerns.

**Key takeaways:**

1. **Start simple, iterate**: My naive first version failed, but it taught me what I needed
2. **Reuse resources**: Browser instances are expensive—create once, reuse many times
3. **Expect failures**: Build retry logic, health checks, and alerts from day one
4. **Monitor everything**: You can't fix what you can't see
5. **Respect targets**: Rate limiting and robots.txt aren't optional
6. **Data quality matters**: Validate and normalize extracted data
7. **Proxies are expensive**: Budget accordingly for high-volume scraping

domharvest-playwright handled the core scraping beautifully, letting me focus on these production concerns rather than browser automation boilerplate. That's exactly what a good library should do.

Whether you're building a price monitor, a data pipeline, or a competitive intelligence tool, these patterns will help you build scrapers that actually work in production—not just in development.

---

**Want to see the full code?** The complete production scraper (with sanitized configurations) is available as a GitHub template: [production-scraper-template](https://github.com/domharvest/production-scraper-template)

**Resources:**
- [domharvest-playwright Documentation](https://domharvest.github.io/domharvest-playwright/)
- [Playwright Best Practices](https://playwright.dev/docs/best-practices)
- [Web Scraping Ethics Guide](https://www.scrapehero.com/web-scraping-ethics/)

*Building your own scraper? Share your experiences on [GitHub Discussions](https://github.com/domharvest/domharvest-playwright/discussions)!*
