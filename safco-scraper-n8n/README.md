# Safco Dental Product Scraper (n8n Workflow)

This is a prototype scraping system built in n8n that pulls product data from Safco Dental Supply's website. It uses an agent based design where each node or group of nodes handles one responsibility.

## How It Works

The workflow has these main stages:

**Navigator** fetches each category page and extracts all the product links from it using cheerio (HTML parser). It looks for links that point to product pages.

**Rate Limiter** adds a 2 second delay between requests so we dont hammer the site.

**Extractor** visits each product page and pulls out the data using CSS selectors. It grabs the product name, brand, SKU, price, description, images, etc.

**LLM Fallback** is a conditional branch. If the CSS extractor cant find a product name on a page, it sends the raw HTML to Claude (Anthropic API) and asks it to extract the data. This only runs when the regular extraction fails.

**Validator** takes all the results, removes duplicates, cleans up whitespace, and throws out anything without a product name.

**Output** saves everything as both JSON and CSV files.

## Setup

1. Open n8n (self hosted or cloud)
2. Go to Workflows and click Import from File
3. Select safco_scraper_workflow.json
4. Set up your Anthropic API credentials in n8n (Settings > Credentials > Add Credential > Anthropic)
5. Update the LLM Fallback Extractor node to use your credential
6. Click Execute Workflow

## Output Schema

Each product has these fields: product_name, brand, sku, category, product_url, price, unit_size, availability, description, image_urls, extraction_method, and scraped_at.

See sample_output/ for examples.

## Limitations

n8n HTTP Request nodes dont run JavaScript on pages so if Safco loads products dynamically with JS those wont get picked up. A headless browser like Playwright would handle that but n8n doesnt have native support for it.

CSS selectors are based on common patterns and will break if the site redesigns.

No checkpoint or resume. If the workflow fails halfway through you have to start over.

Error handling is basic. Failed requests retry 3 times but theres no detailed error logging like you would get with a custom solution.

## How It Handles Failures

HTTP requests retry 3 times with a 5 second delay between retries. If CSS selectors find nothing the LLM fallback kicks in. Duplicates get filtered out by the validator node. If the LLM response cant be parsed the product gets skipped.

## n8n Workflow vs Custom Code: Comparison

The recruiter asked me to compare using n8n with a full AI agent stack versus writing custom code. Here is my analysis.

**Flexibility**

Custom code gives you way more flexibility. You can use any library you want (Playwright for JS rendering, Scrapy for advanced crawling, etc), handle edge cases with precise logic, and structure your code however makes sense. With n8n you are limited to what the nodes can do. For example n8n cant run a headless browser natively, so if a site loads content with JavaScript you are stuck unless you add a custom node or call an external service. Custom code also makes it easier to add features like checkpointing, detailed logging, and complex retry logic.

Winner: Custom code

**Cost**

n8n is cheaper to get started with. The self hosted version is free and you can build a working prototype in a few hours by dragging and dropping nodes. You dont need a developer to maintain it either since the visual interface is easy to understand. But if you need the LLM fallback running on every page, API costs are the same either way. Custom code has higher upfront development cost (more hours to build) but lower long term cost because you have full control over optimization and can reduce unnecessary API calls more precisely.

Winner: n8n for prototyping, custom code for long term

**Stability**

Custom code is more stable in production. You can write proper tests, use CI/CD, handle errors exactly how you want, and deploy it in Docker with monitoring. n8n workflows can break when n8n updates change node behavior, and debugging complex workflows in the visual editor is harder than debugging code in an IDE. n8n also has limitations with large datasets since everything passes through memory between nodes. For a production scraper that runs daily and needs to be reliable, custom code is the safer bet.

Winner: Custom code

**Bottom Line**

n8n is great for quick prototypes and proving a concept works. It lets you build something fast and iterate visually. But for a production system that needs to be reliable, flexible, and maintainable, custom code (Python) is the better choice. The ideal approach is to prototype in n8n to validate the logic, then rewrite in Python for production.

## How I Would Scale This for Production

I would rewrite the core scraping logic in Python using Playwright so it can handle JavaScript rendered pages. I would add a task queue so multiple categories can be scraped at the same time. I would store results in a database instead of files. I would add proxy rotation to avoid getting blocked. And I would set it up on a schedule so it runs automatically and tracks changes over time.

## Project Structure

```
safco-scraper-n8n/
    safco_scraper_workflow.json     The n8n workflow (import this)
    sample_output/
        products_sample.json        Example JSON output
        products_sample.csv         Example CSV output
    README.md                       This file
```
