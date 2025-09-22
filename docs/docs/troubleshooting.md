---
sidebar_position: 5
title: Troubleshooting Guide
description: Common issues and solutions when building scrapers with Botasaurus
---

# Troubleshooting Guide

This guide helps you quickly resolve common issues when building scrapers with Botasaurus. Each section includes the problem description, possible causes, and step-by-step solutions.

## Table of Contents

1. [Installation Issues](#installation-issues)
2. [Element Detection Problems](#element-detection-problems)
3. [Bot Detection and Blocking](#bot-detection-and-blocking)
4. [Performance Issues](#performance-issues)
5. [File and Output Issues](#file-and-output-issues)
6. [Network and Proxy Issues](#network-and-proxy-issues)
7. [Browser and Driver Issues](#browser-and-driver-issues)
8. [Data Extraction Problems](#data-extraction-problems)
9. [Debugging Techniques](#debugging-techniques)

## Installation Issues

### Chrome/Chromium Not Found

**Problem**: Error messages like "Chrome not found" or "Chromium executable not found"

**Solutions**:

```bash
# Option 1: Install Chrome manually
# On Ubuntu/Debian:
wget -q -O - https://dl.google.com/linux/linux_signing_key.pub | sudo apt-key add -
sudo sh -c 'echo "deb [arch=amd64] http://dl.google.com/linux/chrome/deb/ stable main" >> /etc/apt/sources.list.d/google-chrome.list'
sudo apt update
sudo apt install google-chrome-stable

# On CentOS/RHEL:
sudo yum install -y google-chrome-stable

# Option 2: Install through Botasaurus
python -c "from botasaurus.browser import browser; browser.install()"
```

### Dependency Conflicts

**Problem**: Package conflicts during installation

**Solutions**:

```bash
# Create a fresh virtual environment
python -m venv botasaurus_env
source botasaurus_env/bin/activate  # On Windows: botasaurus_env\Scripts\activate

# Clean install
pip install --upgrade pip
pip install botasaurus

# If still having issues, try force reinstall
pip uninstall botasaurus -y
pip install --no-cache-dir botasaurus
```

### Permission Issues

**Problem**: Permission denied errors during installation or execution

**Solutions**:

```bash
# Install for current user only
pip install --user botasaurus

# On Linux/Mac, fix Chrome permissions
sudo chmod +x /usr/bin/google-chrome

# For Docker environments
FROM python:3.9
RUN apt-get update && apt-get install -y \
    wget \
    gnupg \
    unzip \
    curl \
    xvfb

# Install Chrome
RUN wget -q -O - https://dl.google.com/linux/linux_signing_key.pub | apt-key add - \
    && echo "deb http://dl.google.com/linux/chrome/deb/ stable main" > /etc/apt/sources.list.d/google.list \
    && apt-get update \
    && apt-get install -y google-chrome-stable
```

## Element Detection Problems

### Element Not Found Errors

**Problem**: `ElementNotFound` errors or selectors returning `None`

**Diagnostic Script**:

```python
from botasaurus import browser, Driver

@browser(headless=False, close_on_crash=False)
def diagnose_elements(driver: Driver, url):
    """Diagnose element detection issues"""
    
    driver.get(url)
    
    # Wait for page to fully load
    driver.wait_for_element("body", wait=10)
    
    # Check if page loaded correctly
    print(f"Page title: {driver.title}")
    print(f"Current URL: {driver.current_url}")
    print(f"Page source length: {len(driver.page_source)}")
    
    # Try to find your target element
    target_selectors = [
        "h1",
        ".title", 
        "#main-title",
        "[data-testid='title']"
    ]
    
    for selector in target_selectors:
        elements = driver.select_all(selector)
        print(f"Selector '{selector}': Found {len(elements)} elements")
        
        if elements:
            for i, element in enumerate(elements[:3]):  # Show first 3
                print(f"  Element {i+1}: '{element.get_text()[:50]}...'")
    
    # Pause for manual inspection
    driver.prompt("Press Enter after inspecting the page...")
    
    return "Diagnosis complete"

# Run diagnosis
diagnose_elements("https://your-target-site.com")
```

**Common Solutions**:

1. **Wait for Dynamic Content**:
```python
# Instead of immediate selection
element = driver.select(".dynamic-content")

# Use wait parameter
element = driver.select(".dynamic-content", wait=10)

# Or explicit wait
driver.wait_for_element(".dynamic-content", wait=15)
element = driver.select(".dynamic-content")
```

2. **Try Alternative Selectors**:
```python
def get_title_with_fallbacks(driver):
    """Try multiple selectors for title"""
    selectors = [
        "h1.main-title",      # Specific class
        "h1",                 # Generic h1
        ".page-title",        # Class selector
        "#title",             # ID selector
        "[data-title]",       # Attribute selector
        "title"               # Fallback to page title
    ]
    
    for selector in selectors:
        element = driver.select(selector)
        if element:
            text = element.get_text().strip()
            if text and len(text) > 3:  # Reasonable title length
                return text
    
    return driver.title  # Ultimate fallback
```

3. **Handle iframes**:
```python
def scrape_iframe_content(driver: Driver, url):
    """Handle content inside iframes"""
    driver.get(url)
    
    # Check for iframes
    iframes = driver.select_all("iframe")
    print(f"Found {len(iframes)} iframes")
    
    for i, iframe in enumerate(iframes):
        try:
            # Switch to iframe
            driver.switch_to.frame(iframe)
            
            # Now try to find elements inside iframe
            content = driver.select(".content")
            if content:
                print(f"Found content in iframe {i}")
                return content.get_text()
            
            # Switch back to main content
            driver.switch_to.default_content()
            
        except Exception as e:
            print(f"Error in iframe {i}: {e}")
            driver.switch_to.default_content()
    
    return None
```

### Timing Issues

**Problem**: Elements appear after your script tries to access them

**Solutions**:

```python
from botasaurus.browser import Wait

@browser
def handle_timing_issues(driver: Driver, url):
    """Handle various timing scenarios"""
    
    driver.get(url)
    
    # 1. Wait for specific element
    driver.wait_for_element(".loader", wait=5)  # Wait for loader
    driver.wait_for_element_to_be_invisible(".loader", wait=30)  # Wait for loader to disappear
    
    # 2. Wait for multiple elements
    driver.wait_for_any_element([".content", ".main", ".data"], wait=10)
    
    # 3. Wait for text to appear
    driver.wait_for_text("Welcome", wait=10)
    
    # 4. Wait for page to be ready
    driver.wait_for_page_load()
    
    # 5. Custom wait condition
    def content_loaded():
        elements = driver.select_all(".item")
        return len(elements) > 5  # Wait for at least 5 items
    
    driver.wait_for_condition(content_loaded, timeout=20)
    
    # 6. Progressive loading (infinite scroll)
    previous_count = 0
    max_attempts = 5
    
    for attempt in range(max_attempts):
        current_count = len(driver.select_all(".item"))
        
        if current_count > previous_count:
            previous_count = current_count
            driver.scroll_to_bottom()
            driver.sleep(2)  # Wait for new content
        else:
            break  # No new content loaded
    
    return {"items_found": current_count}
```

## Bot Detection and Blocking

### Getting Blocked or Detected

**Problem**: Website shows captcha, blocks access, or returns error pages

**Detection Check**:

```python
@browser(
    user_agent=UserAgent.RANDOM,
    window_size=WindowSize.RANDOM,
    headless=True
)
def check_bot_detection(driver: Driver, url):
    """Check if you're being detected as a bot"""
    
    driver.get(url)
    
    # Check for common detection indicators
    indicators = {
        "captcha": driver.exists(".captcha, #captcha, [data-captcha]"),
        "blocked": "blocked" in driver.page_source.lower(),
        "access_denied": "access denied" in driver.page_source.lower(),
        "bot_detected": driver.is_bot_detected(),
        "cloudflare": "cloudflare" in driver.page_source.lower(),
        "unusual_traffic": "unusual traffic" in driver.page_source.lower()
    }
    
    detection_score = sum(indicators.values())
    
    return {
        "url": url,
        "detection_indicators": indicators,
        "detection_score": detection_score,
        "likely_detected": detection_score > 0,
        "page_title": driver.title,
        "status_suggestions": get_detection_solutions(indicators)
    }

def get_detection_solutions(indicators):
    """Get solutions based on detection indicators"""
    solutions = []
    
    if indicators["captcha"]:
        solutions.append("Use captcha solving service or manual intervention")
    
    if indicators["blocked"] or indicators["access_denied"]:
        solutions.append("Try different IP/proxy, reduce request frequency")
    
    if indicators["cloudflare"]:
        solutions.append("Enable human_mode, use residential proxies")
    
    if indicators["bot_detected"]:
        solutions.append("Randomize user agents, add delays, use browser profiles")
    
    return solutions
```

**Anti-Detection Strategies**:

```python
@browser(
    user_agent=UserAgent.RANDOM,
    window_size=WindowSize.RANDOM,
    proxy=["proxy1", "proxy2", "proxy3"],  # Rotate proxies
    headless=True
)
def stealth_scraper(driver: Driver, url):
    """Scraper with maximum stealth"""
    
    # Enable human-like behavior
    driver.enable_human_mode()
    
    # Natural navigation pattern
    driver.google_get(url)  # Come from Google search
    
    # Random delays
    driver.short_random_sleep()
    
    # Human-like scrolling
    driver.scroll_to_bottom(wait_time=2)
    driver.scroll_to_top(wait_time=1)
    
    # Check for detection
    if driver.is_bot_detected():
        print("Bot detection triggered!")
        
        # Try recovery strategies
        driver.sleep(30)  # Wait longer
        driver.refresh()  # Refresh page
        driver.short_random_sleep()
        
        # If still detected, abort
        if driver.is_bot_detected():
            return {"error": "Bot detected, aborting"}
    
    # Proceed with gentle scraping
    title = driver.get_text("h1")
    
    # Add delay before next action
    driver.short_random_sleep()
    
    return {"title": title, "success": True}
```

**Advanced Anti-Detection**:

```python
@browser(
    user_agent=UserAgent.RANDOM,
    window_size=WindowSize.RANDOM,
    extensions=[
        # Add browser extensions for more realistic behavior
        Extension("https://chromewebstore.google.com/detail/adblock/")
    ]
)
def advanced_stealth_scraper(driver: Driver, data):
    """Advanced anti-detection scraper"""
    
    # Randomize viewport
    driver.run_js("""
        Object.defineProperty(navigator, 'webdriver', {
            get: () => undefined,
        });
    """)
    
    # Mimic human behavior patterns
    driver.enable_human_mode()
    
    # Visit a few pages first to establish session
    driver.get("https://www.google.com")
    driver.short_random_sleep()
    
    # Search for the target site
    search_box = driver.select("input[name='q']")
    if search_box:
        search_box.type(f"site:{data['domain']}")
        search_box.send_keys("\n")
        driver.short_random_sleep()
        
        # Click on the target site from search results
        target_link = driver.select(f"a[href*='{data['domain']}']")
        if target_link:
            target_link.click()
        else:
            driver.get(data["url"])
    else:
        driver.get(data["url"])
    
    # Now proceed with scraping
    driver.wait_for_element("body", wait=10)
    
    return {"title": driver.get_text("h1")}
```

## Performance Issues

### Slow Scraping

**Problem**: Scrapers running much slower than expected

**Performance Optimization**:

```python
# Slow configuration
@browser(
    headless=False,      # Showing browser slows things down
    parallel=1,          # Single threaded
    reuse_driver=False   # Creating new browser each time
)

# Optimized configuration
@browser(
    headless=True,           # Faster without UI
    block_images=True,       # Save bandwidth and time
    block_images_and_css=True,  # Even faster
    parallel=5,              # Multiple browsers
    reuse_driver=True,       # Reuse browser instances
    cache=True               # Cache results
)
def optimized_scraper(driver: Driver, url):
    """Performance-optimized scraper"""
    
    # Use fetch API for subsequent requests (much faster)
    if not driver.config.is_new:
        response = driver.requests.get(url)
        if response.status_code == 200:
            soup = driver.bs4(response.text)
            title = soup.select_one("h1")
            return {
                "url": url,
                "title": title.get_text() if title else None,
                "method": "fetch_api"
            }
    
    # Regular navigation only for first page
    driver.get(url)
    return {
        "url": url,
        "title": driver.get_text("h1"),
        "method": "browser"
    }
```

**Bandwidth Optimization**:

```python
@browser(
    block_images_and_css=True,  # Can save 80-90% bandwidth
    reuse_driver=True
)
def bandwidth_efficient_scraper(driver: Driver, urls):
    """Minimize bandwidth usage"""
    
    results = []
    
    for i, url in enumerate(urls):
        try:
            if i == 0:
                # First request: full browser navigation
                driver.get(url)
                title = driver.get_text("h1")
                method = "browser"
            else:
                # Subsequent requests: use fetch API
                response = driver.requests.get(url)
                
                if response.status_code == 429:  # Rate limited
                    driver.sleep(5)
                    response = driver.requests.get(url)
                
                soup = driver.bs4(response.text)
                title_element = soup.select_one("h1")
                title = title_element.get_text() if title_element else None
                method = "fetch"
            
            results.append({
                "url": url,
                "title": title,
                "method": method
            })
            
            # Small delay to be respectful
            if i < len(urls) - 1:  # Don't sleep after last URL
                driver.sleep(0.5)
                
        except Exception as e:
            results.append({
                "url": url,
                "error": str(e)
            })
    
    return results
```

### Memory Leaks

**Problem**: Memory usage keeps increasing during long-running scrapes

**Memory Management**:

```python
@browser(
    keep_drivers_alive=False,  # Close drivers when done
    parallel=2,                # Limit concurrent browsers
    max_parallel=3
)
def memory_conscious_scraper(driver: Driver, url):
    """Scraper optimized for memory usage"""
    
    try:
        driver.get(url)
        
        # Extract data quickly
        data = {
            "title": driver.get_text("h1"),
            "url": url
        }
        
        # Clear DOM to free memory
        driver.run_js("document.body.innerHTML = '';")
        
        # Close extra tabs if any
        if len(driver.window_handles) > 1:
            for handle in driver.window_handles[1:]:
                driver.switch_to.window(handle)
                driver.close()
            driver.switch_to.window(driver.window_handles[0])
        
        return data
        
    finally:
        # Ensure cleanup
        try:
            driver.delete_all_cookies()
        except:
            pass

# Monitor memory usage
def monitor_memory_usage():
    """Monitor scraper memory usage"""
    import psutil
    import os
    
    process = psutil.Process(os.getpid())
    memory_info = process.memory_info()
    
    return {
        "memory_mb": memory_info.rss / 1024 / 1024,
        "memory_percent": process.memory_percent(),
        "cpu_percent": process.cpu_percent()
    }

# Usage with monitoring
@task
def scrape_with_monitoring(urls):
    """Scrape with memory monitoring"""
    
    results = []
    
    for i, url in enumerate(urls):
        # Monitor before scraping
        memory_before = monitor_memory_usage()
        
        # Scrape
        result = memory_conscious_scraper(url)
        results.append(result)
        
        # Monitor after scraping
        memory_after = monitor_memory_usage()
        
        print(f"URL {i+1}: Memory usage: {memory_after['memory_mb']:.1f}MB "
              f"(+{memory_after['memory_mb'] - memory_before['memory_mb']:.1f}MB)")
        
        # Force garbage collection if memory is high
        if memory_after['memory_mb'] > 1000:  # Over 1GB
            import gc
            gc.collect()
    
    return results
```

## File and Output Issues

### File Overwriting Problems

**Problem**: Default files keep getting overwritten during testing

**Solutions**:

```python
# Solution 1: Timestamped files
from datetime import datetime

@browser(output=None)
def test_safe_scraper(driver: Driver, url):
    """Never overwrites previous test results"""
    
    driver.get(url)
    result = {"title": driver.get_text("h1")}
    
    # Create unique filename
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S_%f")[:-3]  # Include milliseconds
    filename = f"test_{timestamp}.json"
    
    from botasaurus import bt
    bt.write_json(result, filename)
    print(f"Saved to: {filename}")
    
    return result

# Solution 2: Incremental numbering
import glob
import os

@browser(output=None) 
def incremental_scraper(driver: Driver, url):
    """Uses incremental numbering"""
    
    driver.get(url)
    result = {"title": driver.get_text("h1")}
    
    # Find next available number
    existing_files = glob.glob("output/test_*.json")
    if existing_files:
        numbers = []
        for file in existing_files:
            try:
                num = int(os.path.basename(file).split('_')[1].split('.')[0])
                numbers.append(num)
            except:
                continue
        next_num = max(numbers) + 1 if numbers else 1
    else:
        next_num = 1
    
    filename = f"test_{next_num:04d}.json"  # test_0001.json, test_0002.json...
    
    from botasaurus import bt
    bt.write_json(result, filename)
    print(f"Saved to: {filename}")
    
    return result

# Solution 3: Check and backup existing files
@browser(output=None)
def backup_aware_scraper(driver: Driver, url):
    """Creates backups of existing files"""
    
    import shutil
    from pathlib import Path
    
    driver.get(url)
    result = {"title": driver.get_text("h1")}
    
    filename = "results.json"
    file_path = Path(filename)
    
    # Create backup if file exists
    if file_path.exists():
        timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
        backup_name = f"results_backup_{timestamp}.json"
        shutil.copy(file_path, backup_name)
        print(f"Created backup: {backup_name}")
    
    from botasaurus import bt
    bt.write_json(result, filename)
    print(f"Saved to: {filename}")
    
    return result
```

### Output Format Errors

**Problem**: CSV format errors or data not displaying correctly

**Diagnostic**:

```python
def diagnose_output_format_issues(data):
    """Diagnose common output format problems"""
    
    print("Data Type Analysis:")
    print(f"- Data type: {type(data)}")
    print(f"- Is list: {isinstance(data, list)}")
    print(f"- Is dict: {isinstance(data, dict)}")
    
    if isinstance(data, list) and data:
        print(f"- List length: {len(data)}")
        print(f"- First item type: {type(data[0])}")
        if isinstance(data[0], dict):
            print(f"- First item keys: {list(data[0].keys())}")
    elif isinstance(data, dict):
        print(f"- Dict keys: {list(data.keys())}")
    
    # Check for common issues
    issues = []
    
    if isinstance(data, dict) and not isinstance(data, list):
        issues.append("CSV requires list of dictionaries, not single dict")
    
    if isinstance(data, list):
        for i, item in enumerate(data):
            if not isinstance(item, dict):
                issues.append(f"Item {i} is not a dictionary: {type(item)}")
            else:
                # Check for nested data
                for key, value in item.items():
                    if isinstance(value, (dict, list)):
                        issues.append(f"Item {i}, key '{key}' has nested data (not CSV-friendly)")
    
    if issues:
        print("\nPotential Issues:")
        for issue in issues:
            print(f"- {issue}")
    else:
        print("\nNo obvious format issues detected")

# Usage example
@browser
def problematic_scraper(driver: Driver, url):
    driver.get(url)
    
    # This will cause CSV issues - single dict instead of list
    result = {"title": driver.get_text("h1")}
    
    diagnose_output_format_issues(result)
    
    return result
```

**Solutions**:

```python
# Fix 1: Ensure list format for CSV
@browser(output_formats=[bt.Formats.JSON, bt.Formats.CSV])
def csv_friendly_scraper(driver: Driver, url):
    driver.get(url)
    
    # Single item - wrap in list for CSV compatibility
    result = {"title": driver.get_text("h1"), "url": url}
    return [result]  # CSV needs list

# Fix 2: Handle nested data for CSV
@browser(output_formats=[bt.Formats.JSON, bt.Formats.CSV])  
def flatten_for_csv(driver: Driver, url):
    driver.get(url)
    
    # Complex nested data
    result = {
        "title": driver.get_text("h1"),
        "metadata": {
            "url": url,
            "scraped_at": datetime.now().isoformat()
        },
        "links": [link.get_attribute("href") for link in driver.select_all("a")]
    }
    
    # Flatten for CSV compatibility
    flattened = {
        "title": result["title"],
        "url": result["metadata"]["url"],
        "scraped_at": result["metadata"]["scraped_at"],
        "links_count": len(result["links"]),
        "first_link": result["links"][0] if result["links"] else None
    }
    
    return [flattened]

# Fix 3: Separate formats for different data structures
@browser(output=None)
def format_specific_scraper(driver: Driver, url):
    driver.get(url)
    
    # Complex data structure
    complex_data = {
        "title": driver.get_text("h1"),
        "metadata": {"url": url, "scraped_at": datetime.now().isoformat()},
        "links": [{"text": link.get_text(), "href": link.get_attribute("href")} 
                 for link in driver.select_all("a")]
    }
    
    # Simple data for CSV
    simple_data = [{
        "title": complex_data["title"],
        "url": url,
        "links_count": len(complex_data["links"])
    }]
    
    from botasaurus import bt
    # Save complex data as JSON
    bt.write_json(complex_data, "detailed_results.json")
    
    # Save simple data as CSV 
    bt.write_csv(simple_data, "summary_results.csv")
    
    # Save both in Excel with different sheets
    bt.write_excel({
        "summary": simple_data,
        "detailed": [complex_data],
        "links": complex_data["links"]
    }, "complete_results.xlsx")
    
    return complex_data
```

### Permission and Path Issues

**Problem**: Cannot write to output directory or file paths

**Diagnostic Script**:

```python
def diagnose_file_permissions():
    """Diagnose file permission and path issues"""
    
    import os
    from pathlib import Path
    
    print("File System Diagnostic:")
    
    # Check current directory
    current_dir = Path.cwd()
    print(f"Current directory: {current_dir}")
    print(f"Current dir writable: {os.access(current_dir, os.W_OK)}")
    
    # Check output directory
    output_dir = Path("output")
    print(f"Output directory exists: {output_dir.exists()}")
    
    if output_dir.exists():
        print(f"Output dir writable: {os.access(output_dir, os.W_OK)}")
    else:
        print("Attempting to create output directory...")
        try:
            output_dir.mkdir(parents=True, exist_ok=True)
            print("✓ Output directory created successfully")
        except Exception as e:
            print(f"✗ Failed to create output directory: {e}")
    
    # Test file creation
    test_file = output_dir / "test_write.txt"
    try:
        test_file.write_text("test")
        print("✓ Can write files to output directory")
        test_file.unlink()  # Clean up
    except Exception as e:
        print(f"✗ Cannot write files: {e}")
    
    # Check disk space
    try:
        import shutil
        total, used, free = shutil.disk_usage(current_dir)
        print(f"Disk space: {free // (1024**3)}GB free of {total // (1024**3)}GB total")
    except Exception as e:
        print(f"Could not check disk space: {e}")

# Run diagnostic
diagnose_file_permissions()
```

**Solutions**:

```python
# Solution 1: Ensure directory exists and handle permissions
@browser(output=None)
def safe_file_scraper(driver: Driver, url):
    """Scraper with robust file handling"""
    
    import os
    from pathlib import Path
    
    driver.get(url)
    result = {"title": driver.get_text("h1")}
    
    # Ensure output directory exists
    output_dir = Path("output")
    try:
        output_dir.mkdir(parents=True, exist_ok=True)
    except PermissionError:
        # Fallback to current directory
        output_dir = Path(".")
        print("Warning: Using current directory due to permission issues")
    
    # Generate safe filename
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    filename = output_dir / f"results_{timestamp}.json"
    
    # Try to write file with error handling
    try:
        from botasaurus import bt
        bt.write_json(result, str(filename))
        print(f"Successfully saved to: {filename}")
    except PermissionError:
        print(f"Permission denied writing to {filename}")
        # Try alternative location
        alt_filename = Path.home() / "botasaurus_results.json"
        bt.write_json(result, str(alt_filename))
        print(f"Saved to alternative location: {alt_filename}")
    except Exception as e:
        print(f"Unexpected error saving file: {e}")
    
    return result

# Solution 2: Use temporary directory as fallback
import tempfile

@browser(output=None)
def temp_fallback_scraper(driver: Driver, url):
    """Scraper with temporary directory fallback"""
    
    driver.get(url)
    result = {"title": driver.get_text("h1")}
    
    # Try preferred location first
    preferred_path = Path("output/results.json")
    
    try:
        preferred_path.parent.mkdir(parents=True, exist_ok=True)
        from botasaurus import bt
        bt.write_json(result, str(preferred_path))
        print(f"Saved to: {preferred_path}")
    except Exception as e:
        print(f"Could not save to preferred location: {e}")
        
        # Fallback to temporary directory
        with tempfile.NamedTemporaryFile(mode='w', suffix='.json', delete=False) as temp_file:
            import json
            json.dump(result, temp_file, indent=2)
            print(f"Saved to temporary file: {temp_file.name}")
    
    return result
```

### Large File Handling

**Problem**: Memory issues when processing large datasets

**Solutions**:

```python
# Solution 1: Streaming/chunked output
@browser(output=None)
def chunked_output_scraper(driver: Driver, urls):
    """Process large datasets in chunks"""
    
    import json
    from pathlib import Path
    
    # Setup chunked output
    output_file = Path("large_results.json")
    chunk_size = 100
    processed_count = 0
    
    # Initialize file with opening bracket
    with open(output_file, 'w') as f:
        f.write('[\n')
    
    for i, url in enumerate(urls):
        try:
            driver.get(url)
            result = {
                "url": url,
                "title": driver.get_text("h1"),
                "processed_at": datetime.now().isoformat()
            }
            
            # Append to file
            with open(output_file, 'a') as f:
                if i > 0:  # Add comma for all but first item
                    f.write(',\n')
                json.dump(result, f, indent=2)
            
            processed_count += 1
            
            # Optional: Create checkpoint files
            if processed_count % chunk_size == 0:
                checkpoint_file = f"checkpoint_{processed_count}.json"
                print(f"Checkpoint: {processed_count} items processed, saved {checkpoint_file}")
                
        except Exception as e:
            print(f"Error processing {url}: {e}")
            continue
    
    # Close the JSON array
    with open(output_file, 'a') as f:
        f.write('\n]')
    
    print(f"Completed: {processed_count} items processed")
    return {"processed_count": processed_count, "output_file": str(output_file)}

# Solution 2: Memory-efficient batch processing
@browser(output=None)
def memory_efficient_scraper(driver: Driver, urls):
    """Process large datasets with memory management"""
    
    batch_size = 50
    batch_results = []
    batch_number = 1
    
    for i, url in enumerate(urls):
        try:
            driver.get(url)
            
            # Extract minimal data to conserve memory
            result = {
                "url": url,
                "title": driver.get_text("h1")[:200],  # Limit text length
                "links_count": len(driver.select_all("a"))
            }
            
            batch_results.append(result)
            
            # Save batch when full
            if len(batch_results) >= batch_size:
                batch_filename = f"batch_{batch_number:04d}.json"
                from botasaurus import bt
                bt.write_json(batch_results, batch_filename)
                print(f"Saved batch {batch_number}: {len(batch_results)} items")
                
                # Clear memory
                batch_results = []
                batch_number += 1
                
        except Exception as e:
            print(f"Error processing {url}: {e}")
            continue
    
    # Save remaining items
    if batch_results:
        batch_filename = f"batch_{batch_number:04d}.json"
        from botasaurus import bt
        bt.write_json(batch_results, batch_filename)
        print(f"Saved final batch {batch_number}: {len(batch_results)} items")
    
    return {"total_batches": batch_number, "last_batch_size": len(batch_results)}
```

## Network and Proxy Issues

### Proxy Connection Problems

**Problem**: Proxy timeouts, authentication failures, or connection errors

**Proxy Testing**:

```python
def test_proxy_connection(proxy_url):
    """Test if proxy is working"""
    import requests
    
    proxies = {
        'http': proxy_url,
        'https': proxy_url
    }
    
    try:
        # Test with a simple request
        response = requests.get(
            "http://httpbin.org/ip", 
            proxies=proxies, 
            timeout=10
        )
        
        if response.status_code == 200:
            ip_info = response.json()
            return {
                "working": True,
                "ip": ip_info.get("origin"),
                "response_time": response.elapsed.total_seconds()
            }
        else:
            return {"working": False, "error": f"Status code: {response.status_code}"}
            
    except Exception as e:
        return {"working": False, "error": str(e)}

# Test multiple proxies
proxy_list = [
    "http://user1:pass1@proxy1.com:8080",
    "http://user2:pass2@proxy2.com:8080"
]

for proxy in proxy_list:
    result = test_proxy_connection(proxy)
    print(f"Proxy {proxy}: {'✓' if result['working'] else '✗'}")
    if result['working']:
        print(f"  IP: {result['ip']}, Response time: {result['response_time']:.2f}s")
    else:
        print(f"  Error: {result['error']}")
```

**Proxy Rotation with Fallback**:

```python
@browser
def scraper_with_proxy_fallback(driver: Driver, data):
    """Scraper with automatic proxy fallback"""
    
    max_retries = 3
    
    for attempt in range(max_retries):
        try:
            # Try to access the page
            driver.get(data["url"])
            
            # Check if we got a valid page
            if "error" not in driver.title.lower() and len(driver.page_source) > 1000:
                # Success! Extract data
                return {
                    "url": data["url"],
                    "title": driver.get_text("h1"),
                    "proxy_used": driver.config.proxy,
                    "attempts": attempt + 1,
                    "success": True
                }
            else:
                raise Exception("Invalid page content")
                
        except Exception as e:
            print(f"Attempt {attempt + 1} failed: {e}")
            
            if attempt < max_retries - 1:
                # Wait before retrying
                driver.sleep(5)
    
    return {
        "url": data["url"],
        "error": "All proxy attempts failed",
        "success": False
    }

# Usage with rotating proxies
@browser(
    proxy=[
        "http://user:pass@proxy1.com:8080",
        "http://user:pass@proxy2.com:8080",
        "http://user:pass@proxy3.com:8080"
    ],
    max_retry=2
)
def rotating_proxy_scraper(driver: Driver, url):
    return scraper_with_proxy_fallback(driver, {"url": url})
```

## Browser and Driver Issues

### Chrome Crashes

**Problem**: Browser crashes, hangs, or becomes unresponsive

**Stable Browser Configuration**:

```python
@browser(
    # Stability options
    headless=True,                    # More stable than headed mode
    block_images_and_css=True,        # Reduce resource usage
    disable_images=True,              # Alternative to block_images
    
    # Chrome options for stability
    chrome_options=[
        "--no-sandbox",               # Required in some environments
        "--disable-dev-shm-usage",    # Overcome limited resource problems
        "--disable-gpu",              # Disable GPU hardware acceleration
        "--disable-extensions",       # Disable extensions
        "--disable-plugins",          # Disable plugins
        "--memory-pressure-off",      # Disable memory pressure checks
        "--max_old_space_size=4096"   # Increase memory limit
    ],
    
    # Resource limits
    max_parallel=2,                   # Limit concurrent browsers
    keep_drivers_alive=False          # Close when done
)
def stable_scraper(driver: Driver, url):
    """Scraper configured for maximum stability"""
    
    try:
        # Set timeouts to prevent hanging
        driver.implicitly_wait(10)
        driver.set_page_load_timeout(30)
        
        driver.get(url)
        
        # Quick extraction to minimize crash risk
        title = driver.get_text("h1")
        
        return {"url": url, "title": title}
        
    except Exception as e:
        print(f"Browser error: {e}")
        
        # Try to recover
        try:
            driver.refresh()
            driver.wait_for_element("body", wait=5)
            title = driver.get_text("h1")
            return {"url": url, "title": title, "recovered": True}
        except:
            return {"url": url, "error": str(e)}
```

### Driver Initialization Issues

**Problem**: Browser fails to start or initialize

**Diagnostic and Recovery**:

```python
def diagnose_browser_issues():
    """Diagnose browser setup issues"""
    
    import subprocess
    import shutil
    
    diagnostics = {}
    
    # Check Chrome installation
    chrome_paths = [
        "/usr/bin/google-chrome",
        "/usr/bin/chromium-browser", 
        "C:\\Program Files\\Google\\Chrome\\Application\\chrome.exe",
        "C:\\Program Files (x86)\\Google\\Chrome\\Application\\chrome.exe"
    ]
    
    chrome_found = False
    for path in chrome_paths:
        if shutil.which(path) or os.path.exists(path):
            diagnostics["chrome_path"] = path
            chrome_found = True
            break
    
    diagnostics["chrome_installed"] = chrome_found
    
    # Check Chrome version
    if chrome_found:
        try:
            result = subprocess.run([path, "--version"], capture_output=True, text=True)
            diagnostics["chrome_version"] = result.stdout.strip()
        except:
            diagnostics["chrome_version"] = "Could not determine"
    
    # Check system resources
    import psutil
    diagnostics["available_memory_gb"] = psutil.virtual_memory().available / (1024**3)
    diagnostics["cpu_count"] = psutil.cpu_count()
    
    # Test basic browser creation
    try:
        from botasaurus import browser, Driver
        
        @browser(headless=True)
        def test_browser(driver: Driver, data):
            driver.get("https://www.google.com")
            return {"title": driver.title}
        
        result = test_browser("test")
        diagnostics["browser_test"] = "PASSED"
        diagnostics["test_result"] = result
        
    except Exception as e:
        diagnostics["browser_test"] = "FAILED"
        diagnostics["test_error"] = str(e)
    
    return diagnostics

# Run diagnostics
print("Running browser diagnostics...")
diag_results = diagnose_browser_issues()

for key, value in diag_results.items():
    print(f"{key}: {value}")
```

## Data Extraction Problems

### Inconsistent Data

**Problem**: Sometimes getting data, sometimes not

**Robust Data Extraction**:

```python
def robust_extract_text(driver, selectors, fallback="Not found"):
    """Try multiple selectors with fallbacks"""
    
    if isinstance(selectors, str):
        selectors = [selectors]
    
    for selector in selectors:
        try:
            # Try with wait
            element = driver.select(selector, wait=3)
            if element:
                text = element.get_text().strip()
                if text and len(text) > 0:
                    return text
            
            # Try without wait (immediate)
            element = driver.select(selector)
            if element:
                text = element.get_text().strip()
                if text and len(text) > 0:
                    return text
                    
        except Exception as e:
            print(f"Selector '{selector}' failed: {e}")
            continue
    
    return fallback

def robust_extract_attribute(driver, selector, attribute, fallback=None):
    """Extract attribute with error handling"""
    
    try:
        element = driver.select(selector, wait=3)
        if element:
            value = element.get_attribute(attribute)
            if value:
                return value.strip()
    except Exception as e:
        print(f"Attribute extraction failed: {e}")
    
    return fallback

@browser
def robust_data_scraper(driver: Driver, url):
    """Scraper with robust data extraction"""
    
    driver.get(url)
    
    # Wait for page to stabilize
    driver.wait_for_element("body", wait=10)
    
    # Extract title with multiple fallbacks
    title = robust_extract_text(driver, [
        "h1.main-title",
        "h1.title", 
        "h1",
        ".page-title",
        "title"
    ])
    
    # Extract price with cleaning
    price_text = robust_extract_text(driver, [
        ".price-current",
        ".price",
        ".cost",
        "[data-price]"
    ])
    
    # Clean and parse price
    price = None
    if price_text and price_text != "Not found":
        import re
        price_match = re.search(r'[\d,]+\.?\d*', price_text.replace('$', ''))
        if price_match:
            try:
                price = float(price_match.group().replace(',', ''))
            except ValueError:
                pass
    
    # Extract images with fallbacks
    image_url = robust_extract_attribute(driver, 
        "img.main-image, .product-image img, .hero-image", 
        "src"
    )
    
    # Validate extracted data
    data_quality = {
        "has_title": title != "Not found",
        "has_price": price is not None,
        "has_image": image_url is not None,
        "page_loaded": len(driver.page_source) > 1000
    }
    
    return {
        "url": url,
        "title": title,
        "price": price,
        "image_url": image_url,
        "data_quality": data_quality,
        "quality_score": sum(data_quality.values()) / len(data_quality)
    }
```

## Debugging Techniques

### Visual Debugging

```python
@browser(headless=False, close_on_crash=False)
def visual_debug_scraper(driver: Driver, url):
    """Scraper with visual debugging capabilities"""
    
    driver.get(url)
    
    # Highlight elements you're trying to find
    driver.run_js("""
        // Highlight all h1 elements
        document.querySelectorAll('h1').forEach(el => {
            el.style.border = '3px solid red';
            el.style.backgroundColor = 'yellow';
        });
        
        // Highlight all price elements
        document.querySelectorAll('.price, .cost, [data-price]').forEach(el => {
            el.style.border = '3px solid blue';
            el.style.backgroundColor = 'lightblue';
        });
    """)
    
    # Take screenshot
    driver.save_screenshot("debug_highlighted.png")
    
    # Pause for inspection
    driver.prompt("Elements are highlighted. Press Enter to continue...")
    
    # Extract data
    title = driver.get_text("h1")
    price = driver.get_text(".price")
    
    return {"title": title, "price": price}
```

### Logging and Monitoring

```python
import logging
from datetime import datetime

# Setup logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('scraper.log'),
        logging.StreamHandler()
    ]
)

@browser(create_error_logs=True)
def logged_scraper(driver: Driver, url):
    """Scraper with comprehensive logging"""
    
    start_time = datetime.now()
    logging.info(f"Starting scrape of {url}")
    
    try:
        # Log navigation
        logging.info("Navigating to page...")
        driver.get(url)
        
        page_load_time = (datetime.now() - start_time).total_seconds()
        logging.info(f"Page loaded in {page_load_time:.2f} seconds")
        
        # Log element detection
        logging.info("Looking for title element...")
        title_element = driver.select("h1")
        
        if title_element:
            title = title_element.get_text()
            logging.info(f"Found title: {title[:50]}...")
        else:
            logging.warning("No title element found")
            title = None
        
        # Log data extraction
        logging.info("Extracting additional data...")
        
        total_time = (datetime.now() - start_time).total_seconds()
        logging.info(f"Scrape completed in {total_time:.2f} seconds")
        
        return {
            "url": url,
            "title": title,
            "scrape_time": total_time,
            "success": True
        }
        
    except Exception as e:
        logging.error(f"Scrape failed: {str(e)}")
        
        # Log page state for debugging
        try:
            logging.info(f"Page title: {driver.title}")
            logging.info(f"Current URL: {driver.current_url}")
            logging.info(f"Page source length: {len(driver.page_source)}")
        except:
            logging.error("Could not access page information")
        
        return {
            "url": url,
            "error": str(e),
            "success": False
        }
```

### Performance Profiling

```python
import time
from functools import wraps

def profile_performance(func):
    """Decorator to profile scraper performance"""
    
    @wraps(func)
    def wrapper(*args, **kwargs):
        start_time = time.time()
        start_memory = get_memory_usage()
        
        try:
            result = func(*args, **kwargs)
            
            end_time = time.time()
            end_memory = get_memory_usage()
            
            # Add performance metrics to result
            if isinstance(result, dict):
                result['_performance'] = {
                    'execution_time': end_time - start_time,
                    'memory_used_mb': end_memory - start_memory,
                    'timestamp': datetime.now().isoformat()
                }
            
            print(f"Performance: {end_time - start_time:.2f}s, "
                  f"Memory: +{end_memory - start_memory:.1f}MB")
            
            return result
            
        except Exception as e:
            end_time = time.time()
            print(f"Failed after {end_time - start_time:.2f}s: {e}")
            raise
    
    return wrapper

def get_memory_usage():
    """Get current memory usage in MB"""
    import psutil
    import os
    
    process = psutil.Process(os.getpid())
    return process.memory_info().rss / 1024 / 1024

# Usage
@browser(headless=True)
@profile_performance
def profiled_scraper(driver: Driver, url):
    """Scraper with performance profiling"""
    
    driver.get(url)
    title = driver.get_text("h1")
    
    return {"url": url, "title": title}

# Results will include performance metrics
result = profiled_scraper("https://example.com")
print(f"Execution time: {result['_performance']['execution_time']:.2f}s")
```

## Quick Reference

### Common Error Messages and Solutions

| Error Message | Likely Cause | Solution |
|---------------|--------------|----------|
| `ElementNotFoundError` | Element doesn't exist or not loaded yet | Add `wait=10` parameter or try different selectors |
| `TimeoutException` | Page taking too long to load | Increase timeout or check network connection |
| `WebDriverException` | Browser crashed or closed | Restart browser, check system resources |
| `NoSuchElementException` | Selector not found | Verify selector in browser dev tools |
| `StaleElementReferenceException` | Page changed after element was found | Re-find the element after page changes |
| `ConnectionError` | Network or proxy issue | Check internet connection and proxy settings |

### Debug Commands

```python
# Quick debugging commands to add to your scrapers:

# 1. Pause and inspect
driver.prompt("Press Enter to continue...")

# 2. Take screenshot
driver.save_screenshot("debug.png")

# 3. Print page info
print(f"Title: {driver.title}")
print(f"URL: {driver.current_url}")
print(f"Page source length: {len(driver.page_source)}")

# 4. Find all matching elements
elements = driver.select_all("your-selector")
print(f"Found {len(elements)} elements")

# 5. Check if element exists
if driver.exists(".your-element"):
    print("Element exists")
else:
    print("Element not found")

# 6. Highlight elements
driver.run_js("""
    document.querySelectorAll('your-selector').forEach(el => {
        el.style.border = '2px solid red';
    });
""")
```

Remember: When debugging, always start simple and add complexity gradually. Use visual debugging (headless=False) when developing, and comprehensive logging for production deployments.