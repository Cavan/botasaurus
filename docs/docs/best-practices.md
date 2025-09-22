---
sidebar_position: 2
title: Best Practices for Botasaurus
description: Comprehensive guide to building effective scrapers with the Botasaurus framework
---

# Best Practices for Botasaurus

This guide provides comprehensive best practices for building effective, maintainable, and scalable web scrapers using the Botasaurus framework. Whether you're a beginner or an experienced developer, these practices will help you create robust scrapers that can handle real-world challenges.

## Table of Contents

1. [Getting Started - First Scraper](#getting-started---first-scraper)
2. [Decorator Selection Guide](#decorator-selection-guide)
3. [Configuration Best Practices](#configuration-best-practices)
4. [Anti-Detection Strategies](#anti-detection-strategies)
5. [Performance Optimization](#performance-optimization)
6. [Error Handling and Debugging](#error-handling-and-debugging)
7. [Data Management](#data-management)
8. [Scaling Strategies](#scaling-strategies)
9. [Security Considerations](#security-considerations)
10. [Production Deployment](#production-deployment)

## Getting Started - First Scraper

### Basic Scraper Structure

Every Botasaurus scraper follows a simple pattern using decorators. Here's how to create your first scraper:

```python
# Simple browser-based scraper
from botasaurus import browser, Driver

@browser
def scrape_website(driver: Driver, data):
    """
    Basic scraper function structure
    Args:
        driver: Browser driver instance
        data: Input data (URL, search query, etc.)
    Returns:
        dict: Scraped data
    """
    # Navigate to the website
    driver.get(data)
    
    # Extract data
    title = driver.get_text("h1")
    
    # Return structured data
    return {
        "url": data,
        "title": title,
        "scraped_at": driver.run_js("return new Date().toISOString()")
    }

# Run the scraper
if __name__ == "__main__":
    scrape_website("https://example.com")
```

### Request-Based Scraper (Lightweight Alternative)

For simple HTML parsing without JavaScript execution:

```python
from botasaurus import request
from bs4 import BeautifulSoup

@request
def scrape_with_requests(request, data):
    """
    Lightweight scraper using HTTP requests
    Faster and more resource-efficient for static content
    """
    response = request.get(data)
    soup = BeautifulSoup(response.text, 'html.parser')
    
    return {
        "url": data,
        "title": soup.find("h1").get_text() if soup.find("h1") else None,
        "status_code": response.status_code
    }

# Usage
scrape_with_requests("https://example.com")
```

## Decorator Selection Guide

### When to Use @browser

Use the `@browser` decorator when you need:

- **JavaScript execution**: For React, Vue, Angular applications
- **User interactions**: Clicking buttons, filling forms, scrolling
- **Bot detection bypass**: Accessing Cloudflare-protected sites
- **Complex navigation**: Multi-step workflows

```python
@browser(
    headless=False,  # Set to True for production
    block_images=True,  # Save bandwidth
    wait_for_complete_page_load=True
)
def scrape_spa_website(driver: Driver, data):
    """Scraper for Single Page Applications"""
    driver.get(data["url"])
    
    # Wait for dynamic content
    driver.wait_for_element(".dynamic-content", wait=10)
    
    # Interact with the page
    driver.click("button.load-more")
    driver.wait_for_element(".additional-content")
    
    # Extract data after JavaScript execution
    results = []
    elements = driver.select_all(".item")
    
    for element in elements:
        results.append({
            "title": element.get_text(".title"),
            "price": element.get_text(".price"),
            "rating": element.get_attribute(".rating", "data-rating")
        })
    
    return results
```

### When to Use @request

Use the `@request` decorator when you need:

- **Fast processing**: Static HTML pages
- **API endpoints**: JSON responses
- **Bulk scraping**: High-volume data collection
- **Resource efficiency**: Minimal memory usage

```python
@request(
    proxy="http://username:password@proxy.com:8080",  # Optional
    max_retry=3,
    parallel=5
)
def scrape_api_data(request, data):
    """Efficient API data scraping"""
    headers = {
        "User-Agent": "Mozilla/5.0 (compatible; Bot)",
        "Accept": "application/json"
    }
    
    response = request.get(
        f"https://api.example.com/data/{data['id']}", 
        headers=headers
    )
    
    if response.status_code == 200:
        return response.json()
    else:
        return {"error": f"Failed with status {response.status_code}"}
```

### When to Use @task

Use the `@task` decorator for:

- **Data processing**: File parsing, data transformation
- **Non-web tasks**: Working with databases, APIs
- **Custom integrations**: Third-party library usage

```python
from botasaurus import task
import pandas as pd

@task
def process_scraped_data(data):
    """Process and clean scraped data"""
    df = pd.DataFrame(data)
    
    # Clean and transform data
    df['price'] = df['price'].str.replace('$', '').astype(float)
    df['title'] = df['title'].str.strip().str.title()
    
    # Filter valid entries
    df = df.dropna(subset=['title', 'price'])
    
    return df.to_dict('records')
```

## Configuration Best Practices

### Essential Browser Configurations

```python
@browser(
    # Performance optimizations
    headless=True,                    # Production mode
    block_images=True,               # Save bandwidth
    block_images_and_css=True,       # Maximum savings
    
    # Proxy configuration
    proxy="http://user:pass@proxy:port",  # Or list for rotation
    
    # Profile management
    profile="user1",                 # Persistent sessions
    tiny_profile=True,              # Lightweight profiles
    
    # Retry and timing
    max_retry=3,                    # Auto-retry failed requests
    retry_wait=2,                   # Wait between retries
    
    # Parallel processing
    parallel=3,                     # Concurrent browser instances
    max_parallel=5,                 # Maximum parallel limit
    
    # Resource management
    reuse_driver=True,              # Reuse browser instance
    keep_drivers_alive=False,       # Close when done
    
    # Output configuration
    output="custom_results.json",   # Custom output file
    cache=True,                     # Enable caching
)
def production_scraper(driver: Driver, data):
    """Production-ready scraper configuration"""
    pass
```

### Dynamic Configuration

Configure scrapers dynamically based on input data:

```python
def get_proxy(data):
    """Select proxy based on target region"""
    proxy_map = {
        "US": "http://user:pass@us-proxy.com:8080",
        "EU": "http://user:pass@eu-proxy.com:8080",
        "ASIA": "http://user:pass@asia-proxy.com:8080"
    }
    return proxy_map.get(data.get("region"), None)

def get_profile(data):
    """Select profile based on account type"""
    return f"profile_{data.get('account_type', 'default')}"

@browser(
    proxy=get_proxy,
    profile=get_profile,
    headless=True
)
def region_aware_scraper(driver: Driver, data):
    """Scraper that adapts to different regions and accounts"""
    driver.get(data["url"])
    return {"region": data["region"], "data": "scraped_content"}

# Usage with different configurations
data_list = [
    {"url": "https://us-site.com", "region": "US", "account_type": "premium"},
    {"url": "https://eu-site.com", "region": "EU", "account_type": "basic"}
]

region_aware_scraper(data_list)
```

## Anti-Detection Strategies

### Human-Like Behavior

```python
from botasaurus import browser, Driver
from botasaurus.user_agent import UserAgent
from botasaurus.window_size import WindowSize

@browser(
    user_agent=UserAgent.RANDOM,
    window_size=WindowSize.RANDOM,
    proxy=["proxy1", "proxy2", "proxy3"]  # Rotate proxies
)
def stealth_scraper(driver: Driver, data):
    """Scraper with anti-detection measures"""
    
    # Enable human-like mouse movements
    driver.enable_human_mode()
    
    # Natural navigation patterns
    driver.google_get(data["url"])  # Come from Google
    
    # Random delays between actions
    driver.short_random_sleep()
    
    # Human-like interactions
    driver.scroll_to_bottom(wait_time=2)
    driver.short_random_sleep()
    
    # Check for detection
    if driver.is_bot_detected():
        print("Bot detection triggered!")
        return {"error": "Bot detected"}
    
    # Gentle data extraction
    driver.click("button.show-more", human_mode=True)
    driver.wait_for_element(".content")
    
    return {"data": driver.get_text(".content")}
```

### Gradual Loading Strategies

```python
@browser(reuse_driver=True, max_retry=5)
def gradual_scraper(driver: Driver, data):
    """Implement gradual loading to avoid detection"""
    
    # First visit - establish session
    if driver.config.is_new:
        driver.google_get("https://www.google.com")
        driver.short_random_sleep()
        driver.get("https://target-site.com")
        driver.long_random_sleep()  # Let the site "trust" us
    
    # Subsequent requests via fetch API (reduces proxy usage)
    if hasattr(data, 'url') and not driver.config.is_new:
        response = driver.requests.get(data['url'])
        
        if response.status_code == 429:  # Rate limited
            driver.sleep(60)  # Wait longer
            response = driver.requests.get(data['url'])
        
        return {"html": response.text}
    
    # Regular navigation for first page
    driver.get(data["url"])
    return {"html": driver.page_source}
```

## Performance Optimization

### Bandwidth Optimization

```python
@browser(
    block_images_and_css=True,  # Can reduce bandwidth by 80-90%
    reuse_driver=True,
    max_parallel=3
)
def bandwidth_efficient_scraper(driver: Driver, data):
    """Minimize bandwidth usage for large-scale scraping"""
    
    # Use fetch API after initial page load
    if not driver.config.is_new:
        # Much more efficient for subsequent requests
        response = driver.requests.get(data["url"])
        soup = driver.bs4(response.text)
        
        return {
            "title": soup.select_one("h1").get_text(),
            "bandwidth_saved": True
        }
    
    # Initial page load
    driver.get(data["url"])
    return {"title": driver.get_text("h1")}

# This approach can save 90%+ bandwidth costs
urls = ["https://example.com/page1", "https://example.com/page2"]
bandwidth_efficient_scraper(urls)
```

### Memory Management

```python
@browser(
    keep_drivers_alive=False,  # Close drivers when done
    parallel=2,  # Limit concurrent instances
)
def memory_conscious_scraper(driver: Driver, data):
    """Scraper optimized for memory usage"""
    
    try:
        driver.get(data["url"])
        
        # Extract only what you need
        result = {
            "title": driver.get_text("h1"),
            "description": driver.get_text("meta[name='description']", "content")
        }
        
        # Clear unnecessary data
        driver.run_js("document.body.innerHTML = '';")  # Clear DOM
        
        return result
    
    finally:
        # Ensure cleanup
        if len(driver.window_handles) > 1:
            driver.close()  # Close extra windows
```

### Caching Strategies

```python
from botasaurus import Cache

@browser(cache=True)
def cached_scraper(driver: Driver, data):
    """Implement smart caching for repeated requests"""
    
    cache_key = f"product_{data['id']}"
    
    # Check if we have recent data
    cached_data = Cache.get(cache_key)
    if cached_data and cached_data.get('timestamp'):
        # Use cached data if less than 1 hour old
        import time
        if time.time() - cached_data['timestamp'] < 3600:
            return cached_data['data']
    
    # Fresh scrape
    driver.get(data["url"])
    result = {
        "id": data["id"],
        "name": driver.get_text(".product-name"),
        "price": driver.get_text(".price"),
        "timestamp": time.time()
    }
    
    # Cache the result
    Cache.set(cache_key, result)
    
    return result
```

## Error Handling and Debugging

### Robust Error Handling

```python
@browser(
    max_retry=3,
    create_error_logs=True,
    close_on_crash=False  # Keep browser open for debugging
)
def robust_scraper(driver: Driver, data):
    """Scraper with comprehensive error handling"""
    
    try:
        # Set timeout for operations
        driver.set_page_load_timeout(30)
        
        # Navigate with error checking
        driver.get(data["url"])
        
        # Wait for critical elements
        if not driver.select(".main-content", wait=10):
            raise Exception("Main content not found")
        
        # Extract data with fallbacks
        title = (
            driver.get_text("h1.title") or 
            driver.get_text("h1") or 
            driver.get_text("title") or
            "No title found"
        )
        
        # Validate extracted data
        if not title or title == "No title found":
            driver.save_screenshot("error_screenshot.png")
            raise Exception("Failed to extract title")
        
        return {
            "url": data["url"],
            "title": title,
            "success": True
        }
        
    except Exception as e:
        # Log detailed error information
        error_info = {
            "url": data["url"],
            "error": str(e),
            "page_title": driver.title if hasattr(driver, 'title') else None,
            "current_url": driver.current_url if hasattr(driver, 'current_url') else None,
            "success": False
        }
        
        # Take screenshot for debugging
        try:
            driver.save_screenshot(f"error_{data.get('id', 'unknown')}.png")
        except:
            pass
        
        return error_info
```

### Development vs Production Configuration

```python
from botasaurus.config import (
    production_config, 
    production_browser_config,
    production_error_tolerant_config
)

# Development configuration
@browser(
    headless=False,          # See what's happening
    close_on_crash=False,    # Debug errors
    create_error_logs=True,  # Detailed logs
    output="dev_results.json"
)
def dev_scraper(driver: Driver, data):
    """Development scraper with debugging features"""
    driver.get(data["url"])
    
    # Add debugging pauses
    if data.get("debug"):
        driver.prompt("Press Enter to continue...")
    
    return {"title": driver.get_text("h1")}

# Production configuration
@browser(**production_browser_config)
def prod_scraper(driver: Driver, data):
    """Production scraper with optimized settings"""
    driver.get(data["url"])
    return {"title": driver.get_text("h1")}
```

## Data Management

### File Naming and Output Control

Botasaurus provides flexible options for controlling where and how your scraped data is saved. By default, data is saved using the function name, but you can customize this behavior extensively.

#### Basic File Naming

```python
from botasaurus import browser, Driver
from datetime import datetime

# Default: saves as output/scrape_products.json
@browser
def scrape_products(driver: Driver, data):
    driver.get(data["url"])
    return {"title": driver.get_text("h1")}

# Custom filename: saves as output/my_custom_name.json
@browser(output="my_custom_name")
def scrape_products(driver: Driver, data):
    driver.get(data["url"])
    return {"title": driver.get_text("h1")}

# No automatic saving - handle manually
@browser(output=None)
def scrape_products(driver: Driver, data):
    driver.get(data["url"])
    result = {"title": driver.get_text("h1")}
    
    # Handle output manually
    from botasaurus import bt
    bt.write_json(result, "custom_location.json")
    return result
```

#### Timestamped Files for Testing

When testing scrapers, you often want to generate multiple files without overwriting previous results:

```python
from datetime import datetime
import uuid

@browser(output=None)  # Disable automatic output
def test_scraper(driver: Driver, data):
    """Testing scraper that generates timestamped files"""
    
    driver.get(data["url"])
    
    result = {
        "url": data["url"],
        "title": driver.get_text("h1"),
        "scraped_at": datetime.utcnow().isoformat()
    }
    
    # Generate timestamped filename
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    filename = f"test_run_{timestamp}"
    
    # Or use UUID for uniqueness
    unique_id = str(uuid.uuid4())[:8]
    filename_uuid = f"test_run_{unique_id}"
    
    # Save with timestamp
    from botasaurus import bt
    bt.write_json(result, f"{filename}.json")
    bt.write_csv(result, f"{filename}.csv")
    
    print(f"Saved results to {filename}.json and {filename}.csv")
    
    return result

# Usage
test_data = {"url": "https://example.com"}
test_scraper(test_data)
# Output files: test_run_20241201_143022.json, test_run_20241201_143022.csv
```

#### Dynamic File Naming

Create dynamic filenames based on scraped data or input parameters:

```python
@browser(output=None)
def scrape_with_dynamic_naming(driver: Driver, data):
    """Scraper with dynamic file naming based on content"""
    
    driver.get(data["url"])
    
    title = driver.get_text("h1")
    company = driver.get_text(".company-name") or "unknown"
    
    # Clean title for filename
    import re
    clean_title = re.sub(r'[^\w\s-]', '', title).strip()
    clean_title = re.sub(r'[-\s]+', '_', clean_title)
    
    # Create filename from scraped data
    filename = f"{company}_{clean_title}_{datetime.now().strftime('%Y%m%d')}"
    
    result = {
        "title": title,
        "company": company,
        "url": data["url"],
        "scraped_at": datetime.utcnow().isoformat()
    }
    
    # Save with dynamic name
    from botasaurus import bt
    bt.write_json(result, f"{filename}.json")
    
    return result

# Usage
data = {"url": "https://company.com/job-posting"}
scrape_with_dynamic_naming(data)
# Output: TechCorp_Software_Engineer_Position_20241201.json
```

#### Batch Processing with Organized Output

For processing multiple items while keeping files organized:

```python
@browser(output=None)
def batch_scraper_with_organization(driver: Driver, data):
    """Batch scraper with organized file structure"""
    
    import os
    from pathlib import Path
    
    # Create organized directory structure
    base_dir = Path("output")
    date_dir = base_dir / datetime.now().strftime("%Y-%m-%d")
    domain_dir = date_dir / data.get("domain", "unknown")
    
    # Create directories if they don't exist
    domain_dir.mkdir(parents=True, exist_ok=True)
    
    driver.get(data["url"])
    
    result = {
        "url": data["url"],
        "title": driver.get_text("h1"),
        "domain": data.get("domain"),
        "batch_id": data.get("batch_id"),
        "scraped_at": datetime.utcnow().isoformat()
    }
    
    # Generate organized filename
    batch_id = data.get("batch_id", "default")
    filename = f"{batch_id}_page_{data.get('page_num', 1)}"
    
    # Save in organized structure
    from botasaurus import bt
    file_path = domain_dir / f"{filename}.json"
    bt.write_json(result, str(file_path))
    
    print(f"Saved to: {file_path}")
    
    return result

# Usage for batch processing
batch_data = [
    {"url": "https://example.com/page1", "domain": "example", "batch_id": "batch_001", "page_num": 1},
    {"url": "https://example.com/page2", "domain": "example", "batch_id": "batch_001", "page_num": 2},
]

for item in batch_data:
    batch_scraper_with_organization(item)

# Output structure:
# output/
#   2024-12-01/
#     example/
#       batch_001_page_1.json
#       batch_001_page_2.json
```

### Output Formats and Options

Botasaurus supports multiple output formats. Here's a comprehensive guide:

#### Built-in Format Options

```python
from botasaurus import browser, bt, Driver

# Save in multiple formats automatically
@browser(output_formats=[bt.Formats.JSON, bt.Formats.CSV, bt.Formats.EXCEL])
def multi_format_scraper(driver: Driver, data):
    """Scraper that saves in multiple formats"""
    
    driver.get(data["url"])
    
    return {
        "title": driver.get_text("h1"),
        "description": driver.get_text(".description"),
        "price": driver.get_text(".price"),
        "url": data["url"]
    }

# This will create:
# - output/multi_format_scraper.json
# - output/multi_format_scraper.csv  
# - output/multi_format_scraper.xlsx
```

#### Available Format Constants

```python
# All available format options
formats_guide = {
    "bt.Formats.JSON": "JavaScript Object Notation - good for nested data",
    "bt.Formats.CSV": "Comma-separated values - good for tabular data", 
    "bt.Formats.EXCEL": "Excel spreadsheet - good for analysis",
    "bt.Formats.HTML": "HTML table - good for viewing in browser",
    "bt.Formats.XML": "XML format - good for data exchange"
}

@browser(output_formats=[
    bt.Formats.JSON,
    bt.Formats.CSV, 
    bt.Formats.EXCEL,
    bt.Formats.HTML
])
def comprehensive_format_example(driver: Driver, data):
    """Example showing all major formats"""
    
    driver.get(data["url"])
    
    # Structure data for best compatibility across formats
    return [
        {
            "id": 1,
            "title": driver.get_text("h1"),
            "description": driver.get_text(".description"),
            "price": driver.get_text(".price"),
            "url": data["url"],
            "scraped_at": datetime.utcnow().isoformat()
        }
    ]
```

#### Manual Format Control

For maximum control over output formatting:

```python
@browser(output=None)
def manual_format_control(driver: Driver, data):
    """Complete control over output formats and structure"""
    
    driver.get(data["url"])
    
    # Extract data
    products = []
    for product in driver.select_all(".product"):
        products.append({
            "name": product.get_text(".name"),
            "price": product.get_text(".price"),
            "rating": product.get_text(".rating"),
            "image_url": product.get_attribute("img", "src")
        })
    
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    
    # Save as JSON with pretty formatting
    from botasaurus import bt
    bt.write_json(products, f"products_{timestamp}.json", indent=2)
    
    # Save as CSV with custom headers
    bt.write_csv(products, f"products_{timestamp}.csv")
    
    # Save as Excel with multiple sheets
    excel_data = {
        "products": products,
        "summary": [{
            "total_products": len(products),
            "scraped_from": data["url"],
            "scraped_at": datetime.utcnow().isoformat()
        }]
    }
    bt.write_excel(excel_data, f"products_report_{timestamp}.xlsx")
    
    # Save as HTML table
    html_content = create_html_table(products, f"Products from {data['url']}")
    bt.write_file(html_content, f"products_{timestamp}.html")
    
    # Save raw data for debugging
    debug_data = {
        "input_data": data,
        "page_source_length": len(driver.page_source),
        "elements_found": len(driver.select_all(".product")),
        "products": products
    }
    bt.write_json(debug_data, f"debug_{timestamp}.json", indent=2)
    
    return products

def create_html_table(data, title):
    """Helper function to create HTML table"""
    if not data:
        return f"<html><body><h1>{title}</h1><p>No data found</p></body></html>"
    
    headers = list(data[0].keys())
    
    html = f"""
    <!DOCTYPE html>
    <html>
    <head>
        <title>{title}</title>
        <style>
            table {{ border-collapse: collapse; width: 100%; }}
            th, td {{ border: 1px solid #ddd; padding: 8px; text-align: left; }}
            th {{ background-color: #f2f2f2; }}
            tr:nth-child(even) {{ background-color: #f9f9f9; }}
        </style>
    </head>
    <body>
        <h1>{title}</h1>
        <table>
            <tr>
                {''.join(f'<th>{header}</th>' for header in headers)}
            </tr>
    """
    
    for row in data:
        html += "<tr>"
        for header in headers:
            value = str(row.get(header, ""))
            html += f"<td>{value}</td>"
        html += "</tr>"
    
    html += """
        </table>
    </body>
    </html>
    """
    
    return html
```

#### Advanced File Management Patterns

```python
class FileManager:
    """Utility class for advanced file management"""
    
    def __init__(self, base_dir="output"):
        self.base_dir = Path(base_dir)
        self.session_id = datetime.now().strftime("%Y%m%d_%H%M%S")
    
    def get_unique_filename(self, base_name, extension="json"):
        """Generate unique filename to avoid overwrites"""
        counter = 1
        filename = f"{base_name}.{extension}"
        file_path = self.base_dir / filename
        
        while file_path.exists():
            filename = f"{base_name}_{counter}.{extension}"
            file_path = self.base_dir / filename
            counter += 1
        
        return str(file_path)
    
    def save_with_backup(self, data, filename):
        """Save data and create backup of existing file"""
        file_path = self.base_dir / filename
        
        # Create backup if file exists
        if file_path.exists():
            backup_path = self.base_dir / f"{filename}.backup_{self.session_id}"
            file_path.rename(backup_path)
            print(f"Created backup: {backup_path}")
        
        # Save new data
        from botasaurus import bt
        bt.write_json(data, str(file_path))
        print(f"Saved to: {file_path}")

@browser(output=None)
def advanced_file_management_scraper(driver: Driver, data):
    """Scraper with advanced file management"""
    
    file_manager = FileManager("output/advanced")
    
    driver.get(data["url"])
    
    result = {
        "url": data["url"],
        "title": driver.get_text("h1"),
        "content": driver.get_text(".content"),
        "links": [link.get_attribute("href") for link in driver.select_all("a[href]")]
    }
    
    # Save with unique filename (no overwrites)
    unique_file = file_manager.get_unique_filename("scrape_results")
    from botasaurus import bt
    bt.write_json(result, unique_file)
    
    # Save with backup (preserves previous version)
    file_manager.save_with_backup(result, "latest_results.json")
    
    # Save summary in different formats
    summary = {
        "url": data["url"],
        "title": result["title"],
        "links_count": len(result["links"]),
        "content_length": len(result["content"]),
        "scraped_at": datetime.utcnow().isoformat()
    }
    
    bt.write_csv([summary], "summary.csv")
    bt.write_excel({"summary": [summary], "full_data": [result]}, "complete_report.xlsx")
    
    return result
```

#### Quick Reference: File Naming Strategies

```python
# Strategy 1: Timestamp-based (good for testing)
filename = f"scrape_{datetime.now().strftime('%Y%m%d_%H%M%S')}"

# Strategy 2: Content-based (good for organization)
filename = f"{domain}_{page_type}_{date}"

# Strategy 3: UUID-based (guaranteed unique)
import uuid
filename = f"scrape_{str(uuid.uuid4())[:8]}"

# Strategy 4: Sequential numbering
import glob
existing_files = glob.glob("output/scrape_*.json")
next_number = len(existing_files) + 1
filename = f"scrape_{next_number:04d}"  # scrape_0001.json

# Strategy 5: Hierarchical organization
filename = f"{category}/{subcategory}/{item_id}_{timestamp}"
```

This comprehensive file management system gives you complete control over where and how your scraped data is saved, preventing overwrites during testing and enabling organized data collection workflows.

### Data Cleaning and Validation

```python
@task
def clean_scraped_data(raw_data):
    """Clean and validate scraped data"""
    
    import re
    from urllib.parse import urljoin, urlparse
    
    cleaned_data = []
    
    for item in raw_data:
        if not item.get("data"):
            continue
            
        # Clean text fields
        title = item["data"].get("title", "").strip()
        if not title:
            continue
            
        # Normalize text
        title = re.sub(r'\s+', ' ', title)  # Multiple whitespace to single
        
        # Clean price data
        price_text = item["data"].get("price", "")
        price = None
        if price_text:
            price_match = re.search(r'[\d,]+\.?\d*', price_text.replace('$', ''))
            if price_match:
                price = float(price_match.group().replace(',', ''))
        
        # Validate URLs
        url = item.get("url", "")
        if url and not urlparse(url).scheme:
            continue
            
        cleaned_item = {
            "title": title,
            "price": price,
            "url": url,
            "cleaned_at": datetime.utcnow().isoformat()
        }
        
        cleaned_data.append(cleaned_item)
    
    return cleaned_data
```

### Custom Output Formats

```python
@browser(
    output=None  # Disable automatic output to handle manually
)
def custom_output_scraper(driver: Driver, data):
    """Scraper with custom output handling"""
    
    driver.get(data["url"])
    
    result = {
        "title": driver.get_text("h1"),
        "links": [link.get_attribute("href") for link in driver.select_all("a[href]")]
    }
    
    # Custom output handling
    from botasaurus import bt
    
    # Save as Excel with multiple sheets
    bt.write_excel({
        "summary": [{"url": data["url"], "title": result["title"]}],
        "links": [{"url": link} for link in result["links"]]
    }, "custom_output.xlsx")
    
    # Save as HTML report
    html_content = f"""
    <html>
        <head><title>Scrape Report</title></head>
        <body>
            <h1>{result['title']}</h1>
            <p>Found {len(result['links'])} links</p>
            <ul>
                {"".join(f'<li><a href="{link}">{link}</a></li>' for link in result['links'][:10])}
            </ul>
        </body>
    </html>
    """
    
    bt.write_file(html_content, "report.html")
    
    return result
```

## Scaling Strategies

### Parallel Processing

```python
@browser(
    parallel=5,              # Run 5 browsers simultaneously
    max_parallel=10,         # Maximum concurrent limit
    reuse_driver=True        # Reuse browser instances
)
def scalable_scraper(driver: Driver, data):
    """Scraper designed for parallel execution"""
    
    # Handle concurrent access to shared resources
    import threading
    lock = threading.Lock()
    
    driver.get(data["url"])
    
    result = {
        "url": data["url"],
        "title": driver.get_text("h1"),
        "thread_id": threading.current_thread().ident
    }
    
    # Thread-safe operations
    with lock:
        # Shared resource access
        print(f"Processing {data['url']}")
    
    return result

# Process large datasets efficiently
urls = [f"https://example.com/page/{i}" for i in range(100)]
results = scalable_scraper(urls)
```

### Batch Processing

```python
@browser(reuse_driver=True)
def batch_scraper(driver: Driver, data_batch):
    """Process multiple items in a single browser session"""
    
    results = []
    
    for item in data_batch:
        try:
            # Navigate to the page
            if driver.config.is_new:
                driver.google_get(item["url"])
            else:
                # Use fetch API for subsequent requests
                response = driver.requests.get(item["url"])
                if response.status_code == 200:
                    soup = driver.bs4(response.text)
                    title = soup.select_one("h1")
                    
                    results.append({
                        "url": item["url"],
                        "title": title.get_text() if title else None,
                        "method": "fetch_api"
                    })
                    continue
            
            # Fallback to regular navigation
            driver.get(item["url"])
            results.append({
                "url": item["url"],
                "title": driver.get_text("h1"),
                "method": "browser_navigation"
            })
            
        except Exception as e:
            results.append({
                "url": item["url"],
                "error": str(e)
            })
    
    return results

# Group URLs into batches
def create_batches(urls, batch_size=10):
    for i in range(0, len(urls), batch_size):
        yield [{"url": url} for url in urls[i:i + batch_size]]

# Process in batches
all_urls = [f"https://example.com/page/{i}" for i in range(100)]
for batch in create_batches(all_urls):
    batch_results = batch_scraper(batch)
```

## Security Considerations

### Secure Proxy Management

```python
import os
from botasaurus import browser

# Load sensitive data from environment variables
PROXY_USER = os.getenv("PROXY_USER")
PROXY_PASS = os.getenv("PROXY_PASS")
PROXY_HOST = os.getenv("PROXY_HOST")

def get_secure_proxy(data):
    """Securely construct proxy URLs"""
    if not all([PROXY_USER, PROXY_PASS, PROXY_HOST]):
        return None
    
    return f"http://{PROXY_USER}:{PROXY_PASS}@{PROXY_HOST}:8080"

@browser(proxy=get_secure_proxy)
def secure_scraper(driver: Driver, data):
    """Scraper with secure proxy configuration"""
    driver.get(data["url"])
    return {"title": driver.get_text("h1")}
```

### Data Sanitization

```python
@browser
def secure_data_scraper(driver: Driver, data):
    """Scraper that sanitizes extracted data"""
    
    import html
    import re
    
    driver.get(data["url"])
    
    # Extract raw data
    raw_title = driver.get_text("h1")
    raw_description = driver.get_text(".description")
    
    # Sanitize data
    def sanitize_text(text):
        if not text:
            return ""
        
        # HTML decode
        text = html.unescape(text)
        
        # Remove potential script injection
        text = re.sub(r'<script.*?>.*?</script>', '', text, flags=re.DOTALL | re.IGNORECASE)
        
        # Remove potentially dangerous characters
        text = re.sub(r'[<>"\';]', '', text)
        
        return text.strip()
    
    return {
        "title": sanitize_text(raw_title),
        "description": sanitize_text(raw_description),
        "url": data["url"]  # URL should be validated at input
    }
```

## Production Deployment

### Production-Ready Configuration

```python
from botasaurus.config import production_browser_config
from botasaurus.user_agent import UserAgent

@browser(
    **production_browser_config,
    # Additional production settings
    output="production_results.json",
    create_error_logs=True,
    max_retry=5,
    proxy=get_production_proxy_list(),
    user_agent=UserAgent.RANDOM,
)
def production_scraper(driver: Driver, data):
    """Production-ready scraper with full configuration"""
    
    try:
        # Set timeouts
        driver.implicitly_wait(10)
        driver.set_page_load_timeout(30)
        
        # Navigate with retries
        max_attempts = 3
        for attempt in range(max_attempts):
            try:
                driver.google_get(data["url"])
                break
            except Exception as e:
                if attempt == max_attempts - 1:
                    raise e
                driver.sleep(2 ** attempt)  # Exponential backoff
        
        # Extract data with validation
        title = driver.get_text("h1")
        if not title:
            raise ValueError("No title found")
        
        return {
            "url": data["url"],
            "title": title,
            "scraped_at": datetime.utcnow().isoformat(),
            "success": True
        }
        
    except Exception as e:
        # Log error but don't crash
        return {
            "url": data["url"],
            "error": str(e),
            "scraped_at": datetime.utcnow().isoformat(),
            "success": False
        }
```

### Health Monitoring

```python
@task
def monitor_scraper_health():
    """Monitor scraper performance and health"""
    
    import psutil
    import time
    
    # System metrics
    cpu_percent = psutil.cpu_percent(interval=1)
    memory = psutil.virtual_memory()
    disk = psutil.disk_usage('/')
    
    # Chrome process monitoring
    chrome_processes = []
    for process in psutil.process_iter(['pid', 'name', 'memory_info']):
        if 'chrome' in process.info['name'].lower():
            chrome_processes.append({
                'pid': process.info['pid'],
                'memory_mb': process.info['memory_info'].rss / 1024 / 1024
            })
    
    health_report = {
        "timestamp": time.time(),
        "system": {
            "cpu_percent": cpu_percent,
            "memory_percent": memory.percent,
            "disk_percent": disk.percent,
        },
        "chrome_processes": len(chrome_processes),
        "total_chrome_memory_mb": sum(p['memory_mb'] for p in chrome_processes),
        "status": "healthy" if cpu_percent < 80 and memory.percent < 80 else "warning"
    }
    
    return health_report
```

### Environment-Specific Deployment

```python
import os

# Environment detection
ENVIRONMENT = os.getenv("ENVIRONMENT", "development")

def get_environment_config():
    """Get configuration based on environment"""
    
    base_config = {
        "max_retry": 3,
        "cache": True,
    }
    
    if ENVIRONMENT == "production":
        return {
            **base_config,
            "headless": True,
            "block_images_and_css": True,
            "parallel": 5,
            "create_error_logs": True,
            "raise_exception": False,  # Don't crash on errors
        }
    elif ENVIRONMENT == "staging":
        return {
            **base_config,
            "headless": True,
            "parallel": 2,
            "create_error_logs": True,
        }
    else:  # development
        return {
            **base_config,
            "headless": False,
            "parallel": 1,
            "close_on_crash": False,  # Keep browser open for debugging
        }

@browser(**get_environment_config())
def environment_aware_scraper(driver: Driver, data):
    """Scraper that adapts to different environments"""
    
    if ENVIRONMENT == "development":
        print(f"Scraping {data['url']} in development mode")
    
    driver.get(data["url"])
    return {"title": driver.get_text("h1")}
```

## Conclusion

Following these best practices will help you build robust, efficient, and maintainable web scrapers with Botasaurus. Remember to:

1. **Start simple** with basic scrapers and gradually add complexity
2. **Choose the right decorator** for your specific use case
3. **Implement proper error handling** and logging
4. **Optimize for performance** early in development
5. **Plan for scale** from the beginning
6. **Security-first approach** when handling sensitive data
7. **Monitor and maintain** your scrapers in production

The Botasaurus framework provides powerful tools to handle the complexities of modern web scraping. By following these patterns and practices, you'll be able to create scrapers that are not only effective but also maintainable and scalable for long-term use.

For more specific examples and advanced techniques, explore the other documentation sections and the Botasaurus GitHub repository.