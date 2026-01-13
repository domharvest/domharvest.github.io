---
layout: post
title: "domharvest-playwright 1.3.0: Declarative DSL and Authentication Support"
date: 2026-01-13
categories: ["web-scraping", "playwright", "javascript", "release"]
---

# domharvest-playwright 1.3.0: Declarative DSL and Authentication Support

I'm excited to announce the release of **domharvest-playwright 1.3.0**, the most significant update since the project launched. This release introduces two major features that fundamentally enhance how you build web scrapers: a **declarative DSL for data extraction** and **comprehensive authentication support**. Plus, we've significantly improved our testing infrastructure with enforced code coverage thresholds.

Let's dive into what's new.

## Declarative DSL: Cleaner Data Extraction

The biggest addition in 1.3.0 is a powerful Domain Specific Language (DSL) for declarative data extraction. If you've been writing scrapers with domharvest-playwright, you know the pattern: CSS selectors, querySelector calls, optional chaining, and trim() everywhere. It works, but it's verbose.

The new DSL eliminates that boilerplate entirely.

### Before and After

**Traditional approach (still supported):**
```javascript
const products = await harvester.harvest(
  'https://example.com/products',
  '.product',
  (el) => ({
    name: el.querySelector('h2')?.textContent?.trim(),
    price: el.querySelector('.price')?.textContent?.trim(),
    image: el.querySelector('img')?.getAttribute('src'),
    inStock: el.querySelector('.stock-badge') !== null
  })
)
```

**DSL approach (new in 1.3.0):**
```javascript
import { text, attr, exists } from 'domharvest-playwright'

const products = await harvester.harvest(
  'https://example.com/products',
  '.product',
  {
    name: text('h2'),
    price: text('.price'),
    image: attr('img', 'src'),
    inStock: exists('.stock-badge')
  }
)
```

The difference is striking. The DSL version is shorter, cleaner, and more declarative. You focus on *what* you want to extract, not *how* to extract it.

### Complete Helper Function Reference

The DSL provides six core helpers covering the most common extraction patterns:

#### 1. **text()** — Extract Text Content

```javascript
import { text } from 'domharvest-playwright'

// Basic usage
text('h1')  // Extracts and trims text from <h1>

// With default value
text('.subtitle', { default: 'No subtitle' })

// Without trimming
text('.description', { trim: false })
```

The `text()` helper automatically handles null safety and trimming, eliminating the need for `?.textContent?.trim()` everywhere.

#### 2. **attr()** — Extract Attributes

```javascript
import { attr } from 'domharvest-playwright'

// Get attribute value
attr('img', 'src')  // Get src attribute from <img>
attr('a', 'href')   // Get href from <a>

// With default value
attr('img', 'alt', { default: 'No description' })
```

Perfect for extracting URLs, IDs, data attributes, or any HTML attribute.

#### 3. **array()** — Process Collections

```javascript
import { array, text } from 'domharvest-playwright'

// Extract array of text values
array('.tag', text())

// Extract array of links
array('a', attr('href'))

// Extract array of objects
array('.review', {
  author: text('.author'),
  rating: text('.rating'),
  comment: text('.comment')
})
```

The `array()` helper simplifies working with collections, replacing verbose `Array.from()` and `.map()` chains.

#### 4. **exists()** — Check Element Presence

```javascript
import { exists } from 'domharvest-playwright'

// Check if element exists
exists('.premium-badge')  // Returns true/false
exists('.out-of-stock')   // Boolean presence check
```

Clean boolean checks without `!== null` comparisons.

#### 5. **html()** — Extract HTML Content

```javascript
import { html } from 'domharvest-playwright'

// Get innerHTML
html('.description')

// With default value
html('.content', { default: '<p>No content</p>' })
```

Useful when you need the raw HTML structure, not just text.

#### 6. **count()** — Count Elements

```javascript
import { count } from 'domharvest-playwright'

// Count matching elements
count('.review')      // Number of reviews
count('.star.filled') // Number of filled stars
```

Simple element counting without manual array length checks.

### Real-World Example: E-commerce Scraper

Here's a complete example showing how these helpers work together:

```javascript
import { DOMHarvester, text, attr, array, exists, count } from 'domharvest-playwright'

const harvester = new DOMHarvester({
  rateLimit: { requestsPerSecond: 2 }
})

await harvester.init()

try {
  const products = await harvester.harvest(
    'https://example.com/products',
    '.product-card',
    {
      // Basic text extraction
      name: text('h2.product-name'),
      price: text('.price'),
      description: text('.description', { default: 'No description' }),

      // Attribute extraction
      productId: attr('[data-product-id]', 'data-product-id'),
      image: attr('img.product-image', 'src'),
      link: attr('a.product-link', 'href'),

      // Arrays
      tags: array('.tag', text()),
      images: array('.gallery img', attr('src')),

      // Booleans
      inStock: exists('.in-stock-badge'),
      onSale: exists('.sale-badge'),

      // Counts
      reviewCount: count('.review'),

      // Nested objects
      reviews: array('.review', {
        author: text('.review-author'),
        rating: text('.review-rating'),
        date: text('.review-date'),
        comment: text('.review-comment')
      })
    }
  )

  console.log(`Extracted ${products.length} products`)
  console.log(products[0])
} finally {
  await harvester.close()
}
```

### Pure DSL vs Mixed Mode

The DSL supports two execution modes:

**Pure DSL Mode:**
When your extractor uses only DSL helpers, execution happens entirely in the browser context for optimal performance:

```javascript
// Pure DSL - optimized browser-side execution
{
  name: text('h1'),
  price: text('.price'),
  tags: array('.tag', text())
}
```

**Mixed Mode:**
You can combine DSL helpers with custom functions when needed:

```javascript
// Mixed mode - combines DSL with custom logic
{
  name: text('h1'),
  price: text('.price'),
  discount: (el) => {
    const original = parseFloat(el.querySelector('.original-price')?.textContent || '0')
    const current = parseFloat(el.querySelector('.current-price')?.textContent || '0')
    return Math.round(((original - current) / original) * 100)
  }
}
```

Mixed mode falls back to Node.js-side execution but still provides all the benefits of DSL helpers where used.

### Backward Compatibility

**Important:** The DSL is completely optional and fully backward compatible. All existing code continues to work unchanged. You can:

- Use function extractors exclusively (existing approach)
- Use DSL extractors exclusively (new approach)
- Mix both approaches in the same project
- Gradually migrate to DSL at your own pace

There are no breaking changes in this release.

## Authentication and Session Management

The second major feature in 1.3.0 is comprehensive authentication support. Many real-world scraping scenarios require authentication: accessing user profiles, scraping private repositories, extracting personalized data, or monitoring members-only areas.

Previously, you had to handle authentication manually with Playwright's API. Now, domharvest-playwright provides built-in helpers for the most common patterns.

### Form-Based Login

The `login()` helper automatically detects and fills common login forms:

```javascript
import { DOMHarvester, login } from 'domharvest-playwright'

const harvester = new DOMHarvester()
await harvester.init()

try {
  const page = await harvester.getPage()

  // Automatic login with form detection
  await login(page, 'https://example.com/login', {
    username: process.env.USERNAME,
    password: process.env.PASSWORD
  })

  // Now scrape authenticated content
  const data = await harvester.harvest(
    'https://example.com/dashboard',
    '.data',
    { value: text('.value') }
  )

  console.log('Authenticated data:', data)
} finally {
  await harvester.close()
}
```

The `login()` helper:
- Automatically detects common login form patterns
- Fills username and password fields
- Submits the form
- Waits for navigation to complete

For non-standard forms, you can provide custom selectors:

```javascript
await login(page, 'https://example.com/signin', {
  username: process.env.USERNAME,
  password: process.env.PASSWORD
}, {
  usernameSelector: '#email',
  passwordSelector: '#pwd',
  submitSelector: 'button[type="submit"]'
})
```

### Cookie Persistence

Save and restore authentication cookies to skip login on subsequent runs:

```javascript
import { DOMHarvester, login, saveCookies, loadCookies } from 'domharvest-playwright'

const harvester = new DOMHarvester()
await harvester.init()

try {
  const page = await harvester.getPage()
  const context = page.context()

  // Try loading existing cookies
  const cookiesExist = await loadCookies(context, './cookies.json')

  if (!cookiesExist) {
    // First run - perform login
    await login(page, 'https://example.com/login', {
      username: process.env.USERNAME,
      password: process.env.PASSWORD
    })

    // Save cookies for future runs
    await saveCookies(context, './cookies.json')
    console.log('Logged in and saved cookies')
  } else {
    console.log('Loaded existing cookies')
  }

  // Navigate to authenticated area
  await page.goto('https://example.com/dashboard')

  // Scrape authenticated content
  const data = await harvester.harvest(
    'https://example.com/dashboard',
    '.data',
    { value: text('.value') }
  )

  console.log('Data:', data)
} finally {
  await harvester.close()
}
```

### Complete Session Management

For the most robust authentication persistence, use the `SessionManager` class. Unlike simple cookie persistence, it saves the complete browser state:

- Cookies
- localStorage
- sessionStorage
- Origins
- Permissions

This is essential for sites that store authentication state beyond cookies.

```javascript
import { DOMHarvester, login } from 'domharvest-playwright'
import { SessionManager } from 'domharvest-playwright/auth'

const harvester = new DOMHarvester()
await harvester.init()

try {
  const page = await harvester.getPage()
  const context = page.context()

  const sessionManager = new SessionManager('./sessions')
  const sessionId = 'my-account'

  // Try loading existing session
  const loaded = await sessionManager.loadSession(context, sessionId)

  if (!loaded) {
    // First run - perform login
    await login(page, 'https://example.com/login', {
      username: process.env.USERNAME,
      password: process.env.PASSWORD
    })

    // Save complete session state
    await sessionManager.saveSession(context, sessionId)
    console.log('Logged in and saved session')
  } else {
    console.log('Loaded existing session')
  }

  // Navigate to authenticated area
  await page.goto('https://example.com/dashboard')

  // Scrape authenticated content
  const data = await harvester.harvest(
    'https://example.com/dashboard',
    '.data',
    { value: text('.value') }
  )

  console.log('Data:', data)
} finally {
  await harvester.close()
}
```

**SessionManager API:**

```javascript
const sessionManager = new SessionManager('./sessions')

// Save session
await sessionManager.saveSession(context, 'account-1')

// Load session
const loaded = await sessionManager.loadSession(context, 'account-1')

// Check if session exists
const exists = await sessionManager.hasSession('account-1')

// Delete session
await sessionManager.deleteSession('account-1')

// List all sessions
const sessions = await sessionManager.listSessions()
console.log('Available sessions:', sessions)
```

### Multi-Account Support

The SessionManager makes multi-account scraping trivial:

```javascript
import { DOMHarvester } from 'domharvest-playwright'
import { SessionManager } from 'domharvest-playwright/auth'

const accounts = [
  { id: 'account-1', username: process.env.USER1, password: process.env.PASS1 },
  { id: 'account-2', username: process.env.USER2, password: process.env.PASS2 }
]

const sessionManager = new SessionManager('./sessions')

for (const account of accounts) {
  const harvester = new DOMHarvester()
  await harvester.init()

  try {
    const page = await harvester.getPage()
    const context = page.context()

    // Load or create session for this account
    const loaded = await sessionManager.loadSession(context, account.id)

    if (!loaded) {
      await login(page, 'https://example.com/login', {
        username: account.username,
        password: account.password
      })
      await sessionManager.saveSession(context, account.id)
    }

    // Scrape with this account
    await page.goto('https://example.com/dashboard')
    const data = await harvester.harvest(
      'https://example.com/dashboard',
      '.data',
      { value: text('.value') }
    )

    console.log(`Data for ${account.id}:`, data)
  } finally {
    await harvester.close()
  }
}
```

### Real-World Example: GitHub Scraper

Here's a complete example scraping authenticated GitHub data:

```javascript
import { DOMHarvester, login, text, array } from 'domharvest-playwright'
import { SessionManager } from 'domharvest-playwright/auth'

async function scrapeGitHubDashboard() {
  const harvester = new DOMHarvester({
    rateLimit: { requestsPerSecond: 1 }
  })

  await harvester.init()

  try {
    const page = await harvester.getPage()
    const context = page.context()

    const sessionManager = new SessionManager('./sessions')
    const loaded = await sessionManager.loadSession(context, 'github')

    if (!loaded) {
      await login(page, 'https://github.com/login', {
        username: process.env.GITHUB_USERNAME,
        password: process.env.GITHUB_PASSWORD
      }, {
        usernameSelector: '#login_field',
        passwordSelector: '#password'
      })

      await sessionManager.saveSession(context, 'github')
    }

    // Scrape dashboard
    const repos = await harvester.harvest(
      'https://github.com',
      '.repo',
      {
        name: text('.repo-name'),
        description: text('.repo-description'),
        language: text('[itemprop="programmingLanguage"]'),
        stars: text('.stars'),
        updated: text('relative-time')
      }
    )

    console.log(`Found ${repos.length} repositories`)
    return repos
  } finally {
    await harvester.close()
  }
}

await scrapeGitHubDashboard()
```

### Security Best Practices

**Never hardcode credentials.** Always use environment variables:

```javascript
// Good - environment variables
await login(page, url, {
  username: process.env.USERNAME,
  password: process.env.PASSWORD
})

// Bad - hardcoded credentials
await login(page, url, {
  username: 'myuser@example.com',  // Never do this!
  password: 'mypassword123'         // Never do this!
})
```

**Store sessions outside version control:**

```bash
# .gitignore
sessions/
cookies.json
*.session
```

**Handle 2FA manually:**
For sites with two-factor authentication, run in headed mode:

```javascript
const harvester = new DOMHarvester({
  headless: false  // Visible browser for manual 2FA
})
```

**Rotate sessions periodically:**

```javascript
const sessionManager = new SessionManager('./sessions')

// Delete old sessions
if (await sessionManager.hasSession('account-1')) {
  await sessionManager.deleteSession('account-1')
}

// Force fresh login
await login(page, url, credentials)
await sessionManager.saveSession(context, 'account-1')
```

## Enhanced Testing Infrastructure

Version 1.3.0 also brings significant improvements to code quality and testing:

### Enforced Coverage Thresholds

We've implemented minimum coverage requirements enforced in CI:

- **80% minimum** for lines, functions, and statements
- **70% minimum** for branch coverage

Any PR that drops below these thresholds will fail CI, ensuring we maintain high code quality standards.

```json
{
  "coverageThreshold": {
    "global": {
      "lines": 80,
      "functions": 80,
      "statements": 80,
      "branches": 70
    }
  }
}
```

### Current Coverage Stats

The project maintains **86%+ test coverage** across all modules:

- Statement coverage: **86.62%**
- Branch coverage: **71.65%**
- Function coverage: **89.65%**
- Line coverage: **86.62%**

### DSL Module Coverage

The new DSL module has **83% test coverage**, including comprehensive tests for:

- All helper functions (text, attr, array, exists, html, count)
- Pure DSL mode execution
- Mixed mode execution
- Nested objects and complex schemas
- Error handling and edge cases
- Default values and null safety

### Authentication Module Coverage

The authentication features have **95.84% test coverage**, including:

- Form-based login with auto-detection
- Custom selector support
- Cookie persistence (save/load)
- SessionManager complete lifecycle
- Multi-account session isolation
- Error handling and validation

### Comprehensive Documentation

Along with the code, we've added extensive documentation:

- **[DSL Guide](https://domharvest.github.io/domharvest-playwright/guide/dsl.html)** - Complete API reference and practical examples
- **[Authentication Guide](https://domharvest.github.io/domharvest-playwright/guide/authentication.html)** - Real-world authentication patterns
- **[Testing Guide](https://domharvest.github.io/domharvest-playwright/guide/testing.html)** - Testing best practices and coverage requirements

## Migration Guide

Upgrading to 1.3.0 is seamless—there are no breaking changes.

### Install the Update

```bash
npm install domharvest-playwright@1.3.0
```

### Start Using DSL (Optional)

You can gradually adopt the DSL in new code or refactor existing extractors:

```javascript
// Old code (still works)
const data = await harvester.harvest(url, selector, (el) => ({
  title: el.querySelector('h1')?.textContent?.trim()
}))

// New DSL approach (recommended)
import { text } from 'domharvest-playwright'

const data = await harvester.harvest(url, selector, {
  title: text('h1')
})
```

### Add Authentication (Optional)

If you need authentication, import the new helpers:

```javascript
import { login, saveCookies, loadCookies } from 'domharvest-playwright'
// or
import { SessionManager } from 'domharvest-playwright/auth'
```

## What's Next

Looking ahead, here are some features we're considering for future releases:

- **DSL helpers for forms** - Declarative form filling
- **Advanced wait strategies** - More DSL helpers for dynamic content
- **OAuth support** - Built-in OAuth 2.0 flow handling
- **Headless 2FA helpers** - Programmatic 2FA token handling
- **Proxy rotation** - Built-in proxy management
- **Distributed scraping** - Multi-machine coordination

Have ideas? Open an issue on [GitHub](https://github.com/domharvest/domharvest-playwright/issues) or reach out on [Mastodon](https://infosec.exchange/@domharvest).

## Conclusion

Version 1.3.0 represents a major step forward for domharvest-playwright. The declarative DSL makes data extraction cleaner and more maintainable, while authentication support unlocks entire categories of scraping use cases that previously required significant custom code.

Combined with enforced test coverage and comprehensive documentation, this release solidifies domharvest-playwright as a production-ready scraping framework.

**Key highlights:**

- **Declarative DSL** with 6 core helpers (text, attr, array, exists, html, count)
- **Pure DSL mode** for optimized browser-side execution
- **Backward compatible** - all existing code works unchanged
- **Authentication support** with form login, cookie persistence, and SessionManager
- **Multi-account scraping** made simple
- **95%+ auth coverage** and 83% DSL coverage
- **Enforced 80% coverage threshold** in CI
- **Comprehensive documentation** for all new features

Whether you're building a simple scraper or a complex multi-account data extraction pipeline, 1.3.0 gives you the tools to do it cleanly and reliably.

Upgrade today and start building better scrapers!

```bash
npm install domharvest-playwright@1.3.0
```

---

**Resources:**
- [GitHub Repository](https://github.com/domharvest/domharvest-playwright)
- [npm Package](https://www.npmjs.com/package/domharvest-playwright)
- [Full Documentation](https://domharvest.github.io/domharvest-playwright/)
- [DSL Guide](https://domharvest.github.io/domharvest-playwright/guide/dsl.html)
- [Authentication Guide](https://domharvest.github.io/domharvest-playwright/guide/authentication.html)
- [Testing Guide](https://domharvest.github.io/domharvest-playwright/guide/testing.html)

*Questions or feedback? Open an issue on [GitHub](https://github.com/domharvest/domharvest-playwright/issues) or reach out on [Mastodon](https://infosec.exchange/@domharvest)!*
