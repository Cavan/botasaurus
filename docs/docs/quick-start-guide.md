---
sidebar_position: 4
title: Quick Start Guide
description: Get up and running with Botasaurus scrapers quickly with practical examples
---

# Quick Start Guide

This guide will get you up and running with Botasaurus scrapers in minutes. We'll cover the essential patterns and provide practical examples you can use immediately.

## Prerequisites

- Python 3.7 or higher
- Basic understanding of HTML and CSS selectors
- Familiarity with Python functions and dictionaries

## Installation

```bash
# Install Botasaurus
python -m pip install --upgrade botasaurus

# For development, also install optional dependencies
python -m pip install --upgrade beautifulsoup4 pandas
```

## Your First Scraper (5 minutes)

Let's create a simple scraper that extracts article titles from a news website:

```python
from botasaurus import browser, Driver

@browser
def scrape_article_titles(driver: Driver, url):
    """Extract article titles from a news website"""
    
    # Navigate to the website
    driver.get(url)
    
    # Wait for content to load
    driver.wait_for_element("h1, .title", wait=10)
    
    # Extract all article titles
    title_elements = driver.select_all("h1, h2, .article-title, .post-title")
    
    titles = []
    for element in title_elements:
        title = element.get_text().strip()
        if title and len(title) > 10:  # Filter out short titles
            titles.append(title)
    
    return {
        "url": url,
        "titles": titles,
        "count": len(titles)
    }

# Run the scraper
if __name__ == "__main__":
    result = scrape_article_titles("https://news.ycombinator.com")
    print(f"Found {result['count']} articles")
    for title in result['titles'][:5]:  # Show first 5
        print(f"- {title}")
```

**Save this as `first_scraper.py` and run:**
```bash
python first_scraper.py
```

## Quick Decision Tree: Which Decorator to Use?

```
Need to scrape a website?
├── Is it a simple HTML page without JavaScript?
│   └── Use @request (faster, less resources)
├── Does it have dynamic content, forms, or bot protection?
│   └── Use @browser (full browser capabilities)
└── Working with files, APIs, or data processing?
    └── Use @task (no browser needed)
```

## 5-Minute Examples

### 1. Fast API Scraper (@request)

Perfect for JSON APIs and simple HTML pages:

```python
from botasaurus import request

@request
def scrape_api_data(request, api_url):
    """Fast scraper for APIs and static content"""
    
    response = request.get(api_url)
    data = response.json()  # For JSON APIs
    
    return {
        "status": response.status_code,
        "data_count": len(data.get("items", [])),
        "first_item": data.get("items", [{}])[0] if data.get("items") else None
    }

# Usage
result = scrape_api_data("https://jsonplaceholder.typicode.com/posts")
```

### 2. E-commerce Product Scraper (@browser)

For dynamic websites with JavaScript:

```python
from botasaurus import browser, Driver

@browser(block_images=True)  # Save bandwidth
def scrape_product(driver: Driver, product_url):
    """Scrape product information"""
    
    driver.get(product_url)
    
    # Extract product details
    name = driver.get_text("h1, .product-title") or "Unknown Product"
    price = driver.get_text(".price, .cost, .amount") or "Price not found"
    
    # Clean price
    import re
    price_match = re.search(r'[\d,]+\.?\d*', price.replace('$', ''))
    clean_price = float(price_match.group().replace(',', '')) if price_match else None
    
    return {
        "name": name,
        "price": clean_price,
        "url": product_url
    }

# Usage
product = scrape_product("https://example-store.com/product/123")
```

### 3. Form Automation (@browser)

Automate form filling and submission:

```python
@browser
def search_and_scrape(driver: Driver, search_query):
    """Fill a search form and extract results"""
    
    # Go to search page
    driver.get("https://example.com/search")
    
    # Fill search form
    search_box = driver.select("input[name='q'], .search-input")
    search_box.type(search_query)
    
    # Submit form
    search_box.send_keys("\n")  # Press Enter
    
    # Wait for results and extract
    driver.wait_for_element(".search-results", wait=10)
    
    results = []
    for result in driver.select_all(".search-result")[:10]:  # First 10 results
        title = result.get_text(".title, h3")
        link = result.get_attribute("a", "href")
        
        if title and link:
            results.append({"title": title, "link": link})
    
    return {"query": search_query, "results": results}

# Usage
search_results = search_and_scrape("python web scraping")
```

## Essential Patterns (10 minutes)

### Pattern 1: Error Handling

Always include error handling for production scrapers:

```python
@browser(max_retry=3)
def robust_scraper(driver: Driver, url):
    """Scraper with proper error handling"""
    
    try:
        driver.get(url)
        
        # Check if page loaded correctly
        if "error" in driver.title.lower():
            raise Exception("Page shows error")
        
        # Extract data with fallbacks
        title = (
            driver.get_text("h1") or 
            driver.get_text(".title") or 
            driver.title or
            "No title found"
        )
        
        return {"url": url, "title": title, "success": True}
        
    except Exception as e:
        return {"url": url, "error": str(e), "success": False}
```

### Pattern 2: Multiple Items Processing

Process lists of URLs efficiently:

```python
@browser(parallel=3, reuse_driver=True)  # 3 browsers, reuse for efficiency
def scrape_multiple_pages(driver: Driver, url):
    """Scrape multiple pages efficiently"""
    
    driver.get(url)
    title = driver.get_text("h1")
    
    return {"url": url, "title": title}

# Process multiple URLs
urls = [
    "https://example.com/page1",
    "https://example.com/page2", 
    "https://example.com/page3"
]

results = scrape_multiple_pages(urls)  # Processes all URLs
print(f"Scraped {len(results)} pages")
```

### Pattern 3: Data Cleaning

Clean extracted data automatically:

```python
from botasaurus import task

@task
def clean_scraped_data(raw_data):
    """Clean and standardize scraped data"""
    
    import re
    
    cleaned_data = []
    for item in raw_data:
        if not item.get("success", True):  # Skip failed scrapes
            continue
            
        # Clean title
        title = item.get("title", "").strip()
        title = re.sub(r'\s+', ' ', title)  # Multiple spaces to single
        
        # Extract price if present
        price_text = item.get("price", "")
        price = None
        if price_text:
            price_match = re.search(r'[\d,]+\.?\d*', price_text.replace('$', ''))
            if price_match:
                price = float(price_match.group().replace(',', ''))
        
        cleaned_data.append({
            "title": title,
            "price": price,
            "url": item.get("url"),
            "cleaned_at": "2024-01-01T00:00:00Z"  # Add timestamp
        })
    
    return cleaned_data

# Usage: First scrape, then clean
raw_results = scrape_multiple_pages(urls)
clean_results = clean_scraped_data(raw_results)
```

## Configuration Quick Reference

### Essential Browser Configurations

```python
from botasaurus import browser, Driver
from botasaurus.user_agent import UserAgent
from botasaurus.window_size import WindowSize

# Development mode (see what's happening)
@browser(
    headless=False,          # Show browser
    close_on_crash=False,    # Keep open for debugging
)

# Production mode (fast and efficient)
@browser(
    headless=True,           # Hide browser
    block_images=True,       # Save bandwidth
    parallel=5,              # 5 concurrent browsers
    max_retry=3             # Retry failed requests
)

# Stealth mode (avoid detection)
@browser(
    user_agent=UserAgent.RANDOM,     # Random user agent
    proxy="http://user:pass@proxy",  # Use proxy
    headless=True,                   # Less detectable
)
```

### Common Configurations by Use Case

```python
# High-volume scraping
@browser(
    block_images_and_css=True,  # Maximum bandwidth savings
    reuse_driver=True,          # Reuse browser instances
    parallel=10,                # High concurrency
    cache=True                  # Cache results
)

# Bot-protected sites
@browser(
    user_agent=UserAgent.RANDOM,
    window_size=WindowSize.RANDOM,
    proxy=["proxy1", "proxy2"],     # Rotate proxies
)
def stealth_scraper(driver: Driver, url):
    driver.enable_human_mode()      # Human-like movements
    driver.google_get(url)          # Come from Google
    # ... rest of scraping logic
```

## Debugging Tips (2 minutes)

### 1. Visual Debugging

```python
@browser(headless=False, close_on_crash=False)
def debug_scraper(driver: Driver, url):
    driver.get(url)
    
    # Pause to inspect
    driver.prompt("Press Enter after inspecting the page...")
    
    # Take screenshot
    driver.save_screenshot("debug.png")
    
    # Your scraping logic here
    title = driver.get_text("h1")
    return {"title": title}
```

### 2. Element Finding

```python
def find_element_debug(driver: Driver, url):
    driver.get(url)
    
    # Try multiple selectors
    selectors = ["h1", ".title", ".headline", "[data-title]"]
    
    for selector in selectors:
        element = driver.select(selector)
        if element:
            print(f"Found element with selector: {selector}")
            print(f"Text: {element.get_text()}")
            return element.get_text()
    
    print("No elements found with any selector")
    return None
```

### 3. Page Analysis

```python
def analyze_page(driver: Driver, url):
    """Analyze page structure for scraping"""
    driver.get(url)
    
    # Get all headings
    headings = {
        "h1": [h.get_text() for h in driver.select_all("h1")],
        "h2": [h.get_text() for h in driver.select_all("h2")],
        "h3": [h.get_text() for h in driver.select_all("h3")]
    }
    
    # Get all links
    links = [a.get_attribute("href") for a in driver.select_all("a[href]")]
    
    # Get page info
    info = {
        "title": driver.title,
        "url": driver.current_url,
        "headings": headings,
        "total_links": len(links),
        "total_images": len(driver.select_all("img")),
        "has_forms": len(driver.select_all("form")) > 0
    }
    
    return info

# Use this to understand page structure before scraping
page_info = analyze_page("https://example.com")
```

## Common Selector Patterns

```python
# Text content
title = driver.get_text("h1")
price = driver.get_text(".price")

# Attributes
link = driver.get_attribute("a", "href")
image = driver.get_attribute("img", "src")

# Multiple elements
articles = driver.select_all(".article")
for article in articles:
    title = article.get_text(".title")
    link = article.get_attribute("a", "href")

# Wait for dynamic content
driver.wait_for_element(".dynamic-content", wait=10)

# Check if element exists
if driver.exists(".popup"):
    driver.click(".popup .close")
```

## File Naming and Output Control

One of the most common needs when testing scrapers is controlling where files are saved and avoiding overwrites. Here are the essential patterns:

### Basic File Naming

```python
# Default: saves as output/my_scraper.json
@browser
def my_scraper(driver: Driver, url):
    driver.get(url)
    return {"title": driver.get_text("h1")}

# Custom name: saves as output/custom_name.json
@browser(output="custom_name")
def my_scraper(driver: Driver, url):
    driver.get(url) 
    return {"title": driver.get_text("h1")}

# No auto-save: handle manually
@browser(output=None)
def my_scraper(driver: Driver, url):
    driver.get(url)
    result = {"title": driver.get_text("h1")}
    
    # Save manually with custom logic
    from botasaurus import bt
    bt.write_json(result, "my_custom_file.json")
    return result
```

### Timestamped Files (Perfect for Testing)

```python
from datetime import datetime

@browser(output=None)
def test_scraper(driver: Driver, url):
    """Generate unique files for each test run"""
    
    driver.get(url)
    result = {"title": driver.get_text("h1"), "url": url}
    
    # Create timestamp-based filename
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    filename = f"test_run_{timestamp}"
    
    from botasaurus import bt
    bt.write_json(result, f"{filename}.json")
    bt.write_csv([result], f"{filename}.csv")  # Note: CSV needs list
    
    print(f"Saved: {filename}.json and {filename}.csv")
    return result

# Each run creates new files:
# test_run_20241201_143022.json
# test_run_20241201_143530.json  
# test_run_20241201_144105.json
```

### Multiple Output Formats

```python
from botasaurus import bt

# Automatic multiple formats
@browser(output_formats=[bt.Formats.JSON, bt.Formats.CSV, bt.Formats.EXCEL])
def multi_format_scraper(driver: Driver, url):
    driver.get(url)
    return [{"title": driver.get_text("h1"), "url": url}]  # List for CSV compatibility

# Creates:
# - output/multi_format_scraper.json
# - output/multi_format_scraper.csv
# - output/multi_format_scraper.xlsx

# Manual format control
@browser(output=None)
def manual_formats(driver: Driver, url):
    driver.get(url)
    data = [{"title": driver.get_text("h1"), "url": url}]
    
    # Save in different formats with custom names
    from botasaurus import bt
    bt.write_json(data, "results.json")
    bt.write_csv(data, "results.csv") 
    bt.write_excel({"data": data}, "results.xlsx")
    
    return data
```

### Organized File Structure

```python
from pathlib import Path

@browser(output=None)
def organized_scraper(driver: Driver, data):
    """Create organized directory structure"""
    
    # Create organized directories
    domain = data.get("domain", "unknown")
    date = datetime.now().strftime("%Y-%m-%d")
    
    output_dir = Path(f"output/{date}/{domain}")
    output_dir.mkdir(parents=True, exist_ok=True)
    
    driver.get(data["url"])
    result = {"title": driver.get_text("h1")}
    
    # Save in organized structure
    filename = f"page_{data.get('page_num', 1)}.json"
    file_path = output_dir / filename
    
    from botasaurus import bt
    bt.write_json(result, str(file_path))
    
    return result

# Creates structure like:
# output/
#   2024-12-01/
#     example.com/
#       page_1.json
#       page_2.json
```

### Quick File Format Guide

| Format | Best For | Example Usage |
|--------|----------|---------------|
| **JSON** | Complex nested data, APIs | `bt.write_json(data, "file.json")` |
| **CSV** | Tabular data, Excel analysis | `bt.write_csv(list_data, "file.csv")` |
| **Excel** | Multiple sheets, formatting | `bt.write_excel({"sheet1": data}, "file.xlsx")` |
| **HTML** | Visual reports, sharing | `bt.write_file(html_content, "file.html")` |

### Common File Naming Patterns

```python
# Pattern 1: Timestamp (testing)
filename = f"test_{datetime.now().strftime('%Y%m%d_%H%M%S')}"

# Pattern 2: Content-based (organization)  
filename = f"{website_name}_{data_type}_{date}"

# Pattern 3: Unique ID (no collisions)
import uuid
filename = f"scrape_{str(uuid.uuid4())[:8]}"

# Pattern 4: Sequential (ordered)
import glob
count = len(glob.glob("output/scrape_*.json")) + 1
filename = f"scrape_{count:04d}"  # scrape_0001, scrape_0002...
```

1. **Read the [Best Practices Guide](best-practices.md)** for advanced patterns
2. **Check out [Scraper Examples](scraper-examples.md)** for complete implementations
3. **Review the main [Botasaurus Documentation](what-is-botasaurus.md)** for all features

## Quick Troubleshooting

| Problem | Solution |
|---------|----------|
| Element not found | Add `wait=10` parameter or try different selectors |
| Bot detection | Use `driver.enable_human_mode()` and random user agents |
| Slow scraping | Enable `block_images=True` and `reuse_driver=True` |
| Memory issues | Set `parallel=2` and `keep_drivers_alive=False` |
| Rate limiting | Add `driver.sleep(2)` between requests |

## Complete Example: News Scraper

Here's a complete, production-ready news scraper:

```python
from botasaurus import browser, task, Driver
from datetime import datetime
import re

@browser(
    headless=True,
    block_images=True,
    max_retry=3,
    parallel=3
)
def scrape_news_article(driver: Driver, article_url):
    """Complete news article scraper"""
    
    try:
        driver.get(article_url)
        driver.wait_for_element("h1, .title", wait=10)
        
        # Extract article data
        title = driver.get_text("h1") or driver.title
        
        # Try multiple selectors for content
        content_selectors = [".article-content", ".post-content", ".content", "article"]
        content = ""
        for selector in content_selectors:
            element = driver.select(selector)
            if element:
                content = element.get_text()
                if len(content) > 100:  # Reasonable content length
                    break
        
        # Extract metadata
        author = driver.get_text(".author, .byline") or "Unknown"
        
        # Clean author name
        author = re.sub(r'^(by|author:?)\s*', '', author, flags=re.IGNORECASE)
        
        return {
            "url": article_url,
            "title": title.strip(),
            "author": author.strip(),
            "content": content.strip(),
            "word_count": len(content.split()) if content else 0,
            "scraped_at": datetime.utcnow().isoformat(),
            "success": True
        }
        
    except Exception as e:
        return {
            "url": article_url,
            "error": str(e),
            "success": False,
            "scraped_at": datetime.utcnow().isoformat()
        }

@task
def process_articles(articles):
    """Process and filter articles"""
    
    valid_articles = []
    for article in articles:
        if not article.get("success"):
            continue
            
        # Filter by content length
        if article.get("word_count", 0) < 50:
            continue
            
        # Add reading time
        article["reading_time"] = max(1, article["word_count"] // 200)
        
        valid_articles.append(article)
    
    return valid_articles

# Usage
if __name__ == "__main__":
    urls = [
        "https://news.ycombinator.com",
        "https://techcrunch.com",
        # Add more URLs
    ]
    
    # Scrape articles
    raw_articles = scrape_news_article(urls)
    
    # Process results
    processed_articles = process_articles(raw_articles)
    
    print(f"Successfully scraped {len(processed_articles)} articles")
    
    # Show summary
    for article in processed_articles[:3]:
        print(f"- {article['title'][:60]}... ({article['word_count']} words)")
```

**Save as `news_scraper.py` and run:**
```bash
python news_scraper.py
```

You now have a solid foundation for building scrapers with Botasaurus! The framework handles the complexity while you focus on extracting the data you need.