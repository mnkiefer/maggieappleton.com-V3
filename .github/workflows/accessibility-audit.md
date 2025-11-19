---
on:
  workflow_dispatch:
permissions:
  contents: read
  actions: read
  issues: read
  pull-requests: read
timeout-minutes: 30
network: {}
safe-outputs:
  create-issue:
    title-prefix: "[a11y] "
    labels: [accessibility, automated-audit]
    max: 20
engine: claude
tools:
  playwright:
steps:
  - name: Install dependencies
    run: npm install
  - name: Build site without webmentions
    run: |
      # Generate internal links and topics first
      node src/scripts/generate-links.js
      npx tsx src/scripts/generate-topics.ts
      # Note: Skipping webmentions fetch since WEBMENTION_API_KEY is not available in this workflow
      # This is expected for accessibility testing and won't affect the audit
      # The site will still function normally, just without displaying webmentions
      echo "Build preparation complete"
  - name: Start dev server in background
    run: npm run dev &
  - name: Wait for dev server to be ready
    run: |
      echo "Waiting for dev server to start..."
      sleep 10
      # Test if server is responsive
      for i in {1..20}; do
        if curl -s http://localhost:4321 > /dev/null; then
          echo "Dev server is ready!"
          exit 0
        fi
        echo "Waiting for server... attempt $i/20"
        sleep 2
      done
      echo "Dev server failed to start within timeout"
      exit 1
---

# Accessibility Audit Workflow

You are an accessibility testing specialist. Your task is to perform a comprehensive accessibility audit of the Maggie Appleton digital garden website.

## Context

The Astro site has been built and is now running locally at `http://localhost:4321` (Astro's default dev port). You have access to Playwright for browser automation and can take screenshots.

**Important Notes:**
- The site is running in development mode without webmentions (requires `WEBMENTION_API_KEY` which is not available in this test environment)
- This is expected and will not affect accessibility testing - webmentions are a social feature and not part of the core content
- All internal links, topics, and content have been generated and are accessible for testing

## Your Task

1. **Crawl the entire site** starting from `http://localhost:4321`:
   - Navigate through all main pages (home, essays, notes, patterns, library, etc.)
   - Follow internal links to discover all content pages
   - Build a comprehensive list of unique URLs to test
   - **Error Handling**: If you encounter any errors during crawling (e.g., 404s, broken links, JavaScript errors), create a GitHub issue documenting:
     - The error message and stack trace
     - The URL or action that triggered it
     - Steps to reproduce
     - Suggested fix if apparent

2. **For each page, check for accessibility violations**:
   - Missing alt text on images
   - Poor color contrast ratios (WCAG AA/AAA standards)
   - Missing ARIA labels and landmarks
   - Improper heading hierarchy (h1 → h2 → h3, no skipping levels)
   - Form inputs without associated labels
   - Links without descriptive text (avoid "click here")
   - Missing focus indicators for keyboard navigation
   - Improper use of semantic HTML
   - Any other WCAG 2.1 Level A/AA violations

3. **Use Playwright's accessibility testing capabilities**:
   - Use `page.accessibility.snapshot()` to capture accessibility tree
   - Use `page.evaluate()` with axe-core if needed for detailed WCAG checks
   - Take screenshots of problematic areas to provide visual context

4. **Create individual GitHub issues for each unique problem**:
   - Each issue should focus on ONE specific accessibility problem
   - Include:
     - Clear title describing the issue
     - Affected page URL(s)
     - WCAG criterion violated (e.g., "1.1.1 Non-text Content", "1.4.3 Contrast")
     - Severity level (Critical/High/Medium/Low)
     - Specific location (CSS selector or description)
     - Current state vs. expected state
     - Screenshot showing the problem (if visual)
     - Suggested fix/remediation steps
   - Format issues using markdown with proper headings and code blocks

5. **Error Handling & Reporting**:
   - If you encounter any errors while running tests (build errors, runtime errors, Playwright failures, etc.), create a GitHub issue with:
     - Title: `[Error] Brief description of the error`
     - Labels: `bug`, `automated-audit`
     - Full error message and stack trace
     - Context: What you were trying to do when the error occurred
     - Suggested fix or workaround if you can determine one
     - Example:
       ```markdown
       ## Error Description
       Playwright timeout when trying to navigate to /patterns page
       
       ## Error Message
       ```
       TimeoutError: page.goto: Timeout 30000ms exceeded
       ```
       
       ## Context
       Attempting to navigate to http://localhost:4321/patterns for accessibility testing
       
       ## Suggested Fix
       - Check if patterns page has slow-loading components
       - Increase timeout for pages with heavy D3 visualizations
       - Ensure patterns data is being loaded correctly
       ```

6. **Important Testing Guidelines**:
   - Only test pages on `localhost:4321` - do NOT make external network requests
   - Group similar violations on the same page into one issue (e.g., "Multiple images missing alt text on /essays/example")
   - Don't create duplicate issues for the same problem across multiple pages - instead list all affected pages in one issue
   - Prioritize issues by severity (Critical first)
   - Limit to the top 20 most important issues if you find many violations
   - If any part of the test fails or you're unsure about something, create an issue to document it rather than silently skipping it

## Output Format

For each issue you create, use this structure:

```markdown
## Summary
[Brief description of the accessibility problem]

## WCAG Criterion
- **Standard**: WCAG 2.1 Level [A/AA/AAA]
- **Criterion**: [Number and name, e.g., "1.4.3 Contrast (Minimum)"]

## Severity
[Critical/High/Medium/Low]

## Affected Pages
- http://localhost:3000/[page-url]
- http://localhost:3000/[another-page-url]

## Problem Description
[Detailed explanation of what's wrong]

## Current State
[What the element/page currently does]

## Expected State
[What it should do to be accessible]

## Location
[CSS selector or description of where to find this element]

## Screenshot
[If applicable, reference screenshot taken]

## Suggested Fix
[Specific code changes or recommendations]
```

Begin the audit now. Start by crawling the site to discover all pages, then systematically test each page for accessibility violations.
