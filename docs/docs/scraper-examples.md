---
sidebar_position: 3
title: Scraper Creation Examples
description: Practical examples for creating different types of scrapers with Botasaurus
---

# Scraper Creation Examples

This guide provides practical, real-world examples of creating scrapers for different scenarios using the Botasaurus framework. Each example is production-ready and includes best practices for handling common challenges.

## Table of Contents

1. [E-commerce Product Scraper](#e-commerce-product-scraper)
2. [News Article Scraper](#news-article-scraper)
3. [Social Media Scraper](#social-media-scraper)
4. [API Data Scraper](#api-data-scraper)
5. [Form Submission Scraper](#form-submission-scraper)
6. [Infinite Scroll Scraper](#infinite-scroll-scraper)
7. [Multi-Page Scraper](#multi-page-scraper)
8. [File Download Scraper](#file-download-scraper)
9. [Authentication-Required Scraper](#authentication-required-scraper)
10. [Real-Time Data Scraper](#real-time-data-scraper)

## E-commerce Product Scraper

### Basic Product Information Scraper

```python
from botasaurus import browser, Driver
from datetime import datetime
import re

@browser(
    block_images=True,  # Save bandwidth
    max_retry=3,
    parallel=3
)
def scrape_product_details(driver: Driver, product_url):
    """
    Scrapes detailed product information from e-commerce sites
    
    Args:
        product_url (str): URL of the product page
        
    Returns:
        dict: Product information including name, price, rating, etc.
    """
    
    try:
        # Navigate to product page
        driver.get(product_url)
        
        # Wait for product content to load
        driver.wait_for_element(".product-title, h1", wait=10)
        
        # Extract product name
        product_name = (
            driver.get_text(".product-title") or
            driver.get_text("h1") or
            driver.get_text("[data-testid='product-title']") or
            "Unknown Product"
        )
        
        # Extract price with multiple selectors
        price_text = (
            driver.get_text(".price-current") or
            driver.get_text(".price") or
            driver.get_text("[data-testid='price']") or
            driver.get_text(".product-price") or
            "0"
        )
        
        # Clean and parse price
        price = extract_price(price_text)
        
        # Extract rating
        rating = extract_rating(driver)
        
        # Extract availability
        availability = check_availability(driver)
        
        # Extract images
        images = extract_product_images(driver)
        
        # Extract specifications
        specifications = extract_specifications(driver)
        
        # Extract reviews count
        reviews_count = extract_reviews_count(driver)
        
        return {
            "url": product_url,
            "name": product_name.strip(),
            "price": price,
            "currency": extract_currency(price_text),
            "rating": rating,
            "reviews_count": reviews_count,
            "availability": availability,
            "images": images,
            "specifications": specifications,
            "scraped_at": datetime.utcnow().isoformat(),
            "success": True
        }
        
    except Exception as e:
        # Return error information for debugging
        return {
            "url": product_url,
            "error": str(e),
            "page_title": driver.title if hasattr(driver, 'title') else None,
            "success": False,
            "scraped_at": datetime.utcnow().isoformat()
        }

def extract_price(price_text):
    """Extract numeric price from price text"""
    if not price_text:
        return None
    
    # Remove currency symbols and extract numbers
    price_match = re.search(r'[\d,]+\.?\d*', price_text.replace('$', '').replace(',', ''))
    return float(price_match.group()) if price_match else None

def extract_currency(price_text):
    """Extract currency symbol from price text"""
    if not price_text:
        return None
    
    currency_symbols = {'$': 'USD', '€': 'EUR', '£': 'GBP', '¥': 'JPY'}
    for symbol, code in currency_symbols.items():
        if symbol in price_text:
            return code
    return 'USD'  # Default

def extract_rating(driver):
    """Extract product rating"""
    # Try multiple rating selectors
    rating_selectors = [
        ".rating-value",
        "[data-testid='rating']",
        ".stars .star-rating",
        ".review-rating"
    ]
    
    for selector in rating_selectors:
        rating_element = driver.select(selector)
        if rating_element:
            rating_text = rating_element.get_text()
            rating_match = re.search(r'(\d+\.?\d*)', rating_text)
            if rating_match:
                return float(rating_match.group(1))
    
    # Try star count method
    stars = driver.select_all(".star.filled, .star-filled")
    if stars:
        return len(stars)
    
    return None

def check_availability(driver):
    """Check product availability"""
    availability_indicators = [
        (".in-stock", "in_stock"),
        (".out-of-stock", "out_of_stock"),
        (".limited-stock", "limited_stock"),
        ("[data-testid='availability']", "check_text")
    ]
    
    for selector, status in availability_indicators:
        element = driver.select(selector)
        if element:
            if status == "check_text":
                text = element.get_text().lower()
                if "in stock" in text:
                    return "in_stock"
                elif "out of stock" in text:
                    return "out_of_stock"
                else:
                    return "unknown"
            return status
    
    return "unknown"

def extract_product_images(driver):
    """Extract product image URLs"""
    images = []
    
    # Main product image
    main_img = driver.select(".product-image img, .main-image img")
    if main_img:
        img_src = main_img.get_attribute("src") or main_img.get_attribute("data-src")
        if img_src:
            images.append({"type": "main", "url": img_src})
    
    # Thumbnail images
    thumbnails = driver.select_all(".thumbnail img, .product-thumbnails img")
    for thumb in thumbnails:
        img_src = thumb.get_attribute("src") or thumb.get_attribute("data-src")
        if img_src and img_src not in [img["url"] for img in images]:
            images.append({"type": "thumbnail", "url": img_src})
    
    return images

def extract_specifications(driver):
    """Extract product specifications"""
    specs = {}
    
    # Try table format
    spec_rows = driver.select_all(".specifications tr, .product-specs tr")
    for row in spec_rows:
        cells = row.select_all("td")
        if len(cells) >= 2:
            key = cells[0].get_text().strip()
            value = cells[1].get_text().strip()
            if key and value:
                specs[key] = value
    
    # Try list format
    if not specs:
        spec_items = driver.select_all(".spec-item, .specification-item")
        for item in spec_items:
            label = item.select(".spec-label, .label")
            value = item.select(".spec-value, .value")
            if label and value:
                specs[label.get_text().strip()] = value.get_text().strip()
    
    return specs

def extract_reviews_count(driver):
    """Extract number of reviews"""
    reviews_selectors = [
        ".reviews-count",
        ".review-count",
        "[data-testid='reviews-count']"
    ]
    
    for selector in reviews_selectors:
        element = driver.select(selector)
        if element:
            text = element.get_text()
            count_match = re.search(r'(\d+)', text.replace(',', ''))
            if count_match:
                return int(count_match.group(1))
    
    return 0

# Usage example
if __name__ == "__main__":
    product_urls = [
        "https://example-shop.com/product/laptop-123",
        "https://example-shop.com/product/phone-456",
        "https://example-shop.com/product/tablet-789"
    ]
    
    results = scrape_product_details(product_urls)
    print(f"Scraped {len(results)} products")
```

### Product Comparison Scraper

```python
@browser(reuse_driver=True, parallel=2)
def scrape_product_comparison(driver: Driver, data):
    """
    Scrape multiple products for comparison
    
    Args:
        data (dict): Contains 'products' list with URLs and 'comparison_fields'
    """
    
    products_data = []
    
    for product_url in data['products']:
        try:
            # Use fetch API for efficiency after first page
            if not driver.config.is_new:
                response = driver.requests.get(product_url)
                if response.status_code == 200:
                    soup = driver.bs4(response.text)
                    product_data = parse_product_from_soup(soup, product_url)
                    products_data.append(product_data)
                    continue
            
            # Regular navigation for first page or if fetch fails
            driver.get(product_url)
            product_data = scrape_single_product(driver, product_url)
            products_data.append(product_data)
            
        except Exception as e:
            products_data.append({
                "url": product_url,
                "error": str(e),
                "success": False
            })
    
    # Generate comparison
    comparison = generate_comparison_analysis(products_data, data.get('comparison_fields', []))
    
    return {
        "products": products_data,
        "comparison": comparison,
        "scraped_at": datetime.utcnow().isoformat()
    }

def parse_product_from_soup(soup, url):
    """Parse product data from BeautifulSoup object"""
    return {
        "url": url,
        "name": soup.select_one("h1, .product-title").get_text() if soup.select_one("h1, .product-title") else None,
        "price": extract_price_from_soup(soup),
        "rating": extract_rating_from_soup(soup),
        "method": "fetch_api"
    }

def generate_comparison_analysis(products_data, fields):
    """Generate comparison analysis between products"""
    if not products_data or not fields:
        return {}
    
    comparison = {}
    
    for field in fields:
        values = [p.get(field) for p in products_data if p.get(field) is not None]
        
        if field == 'price' and values:
            comparison[field] = {
                "min": min(values),
                "max": max(values),
                "avg": sum(values) / len(values),
                "cheapest_product": min(products_data, key=lambda x: x.get('price', float('inf')))['url']
            }
        elif field == 'rating' and values:
            comparison[field] = {
                "highest": max(values),
                "lowest": min(values),
                "avg": sum(values) / len(values),
                "best_rated_product": max(products_data, key=lambda x: x.get('rating', 0))['url']
            }
    
    return comparison
```

## News Article Scraper

### Comprehensive News Scraper

```python
@browser(
    block_images=True,
    user_agent=UserAgent.RANDOM,
    max_retry=3
)
def scrape_news_article(driver: Driver, article_url):
    """
    Scrapes news articles with content, metadata, and media
    
    Args:
        article_url (str): URL of the news article
        
    Returns:
        dict: Article content and metadata
    """
    
    try:
        driver.get(article_url)
        
        # Wait for article content
        driver.wait_for_element("article, .article-content, .post-content", wait=10)
        
        # Extract article metadata
        article_data = {
            "url": article_url,
            "title": extract_article_title(driver),
            "author": extract_author(driver),
            "publish_date": extract_publish_date(driver),
            "content": extract_article_content(driver),
            "summary": extract_summary(driver),
            "tags": extract_tags(driver),
            "category": extract_category(driver),
            "images": extract_article_images(driver),
            "videos": extract_videos(driver),
            "related_articles": extract_related_articles(driver),
            "social_shares": extract_social_shares(driver),
            "comments_count": extract_comments_count(driver),
            "word_count": 0,  # Will be calculated
            "reading_time": 0,  # Will be calculated
            "scraped_at": datetime.utcnow().isoformat(),
            "success": True
        }
        
        # Calculate derived metrics
        if article_data["content"]:
            words = len(article_data["content"].split())
            article_data["word_count"] = words
            article_data["reading_time"] = max(1, words // 200)  # Assume 200 WPM
        
        return article_data
        
    except Exception as e:
        return {
            "url": article_url,
            "error": str(e),
            "success": False,
            "scraped_at": datetime.utcnow().isoformat()
        }

def extract_article_title(driver):
    """Extract article title with multiple fallbacks"""
    title_selectors = [
        "h1.article-title",
        "h1.post-title",
        ".entry-title",
        "h1",
        "title"
    ]
    
    for selector in title_selectors:
        element = driver.select(selector)
        if element:
            title = element.get_text().strip()
            if len(title) > 10:  # Reasonable title length
                return title
    
    return driver.title

def extract_author(driver):
    """Extract article author"""
    author_selectors = [
        ".author-name",
        ".byline .author",
        "[rel='author']",
        ".post-author",
        ".article-author"
    ]
    
    for selector in author_selectors:
        element = driver.select(selector)
        if element:
            author = element.get_text().strip()
            # Clean author text
            author = re.sub(r'^(by|author:?)\s*', '', author, flags=re.IGNORECASE)
            if author and len(author) < 100:  # Reasonable author name length
                return author
    
    return None

def extract_publish_date(driver):
    """Extract article publish date"""
    from dateutil import parser
    
    # Try structured data first
    date_element = driver.select("[datetime], time")
    if date_element:
        datetime_attr = date_element.get_attribute("datetime")
        if datetime_attr:
            try:
                return parser.parse(datetime_attr).isoformat()
            except:
                pass
    
    # Try common date selectors
    date_selectors = [
        ".publish-date",
        ".post-date",
        ".article-date",
        ".date-published"
    ]
    
    for selector in date_selectors:
        element = driver.select(selector)
        if element:
            date_text = element.get_text().strip()
            try:
                return parser.parse(date_text).isoformat()
            except:
                continue
    
    return None

def extract_article_content(driver):
    """Extract main article content"""
    content_selectors = [
        ".article-content",
        ".post-content",
        ".entry-content",
        "article .content",
        ".article-body"
    ]
    
    for selector in content_selectors:
        element = driver.select(selector)
        if element:
            # Get text content while preserving paragraphs
            paragraphs = element.select_all("p")
            if paragraphs:
                content = "\n\n".join(p.get_text().strip() for p in paragraphs if p.get_text().strip())
                if len(content) > 100:  # Reasonable content length
                    return content
            
            # Fallback to all text
            content = element.get_text().strip()
            if len(content) > 100:
                return content
    
    return None

def extract_summary(driver):
    """Extract article summary or excerpt"""
    summary_selectors = [
        ".article-summary",
        ".excerpt",
        ".post-excerpt",
        ".summary",
        "[name='description']"
    ]
    
    for selector in summary_selectors:
        if selector.startswith("[name"):
            element = driver.select(selector)
            if element:
                return element.get_attribute("content")
        else:
            element = driver.select(selector)
            if element:
                summary = element.get_text().strip()
                if 50 < len(summary) < 500:  # Reasonable summary length
                    return summary
    
    return None

def extract_tags(driver):
    """Extract article tags"""
    tags = []
    
    tag_selectors = [
        ".tags a",
        ".post-tags a",
        ".article-tags a",
        ".tag-list a"
    ]
    
    for selector in tag_selectors:
        elements = driver.select_all(selector)
        for element in elements:
            tag = element.get_text().strip()
            if tag and tag not in tags:
                tags.append(tag)
    
    return tags

def extract_category(driver):
    """Extract article category"""
    category_selectors = [
        ".category",
        ".post-category",
        ".article-category",
        ".breadcrumb a:last-child"
    ]
    
    for selector in category_selectors:
        element = driver.select(selector)
        if element:
            category = element.get_text().strip()
            if category and len(category) < 50:
                return category
    
    return None

def extract_article_images(driver):
    """Extract images from article content"""
    images = []
    
    # Article content images
    content_images = driver.select_all(".article-content img, .post-content img, article img")
    
    for img in content_images:
        src = img.get_attribute("src") or img.get_attribute("data-src")
        alt = img.get_attribute("alt") or ""
        caption = ""
        
        # Try to find caption
        parent = img.find_element_by_xpath("..")
        caption_element = parent.select(".caption, figcaption")
        if caption_element:
            caption = caption_element.get_text().strip()
        
        if src:
            images.append({
                "src": src,
                "alt": alt,
                "caption": caption
            })
    
    return images

def extract_videos(driver):
    """Extract videos from article"""
    videos = []
    
    # Video elements
    video_elements = driver.select_all("video, iframe[src*='youtube'], iframe[src*='vimeo']")
    
    for video in video_elements:
        if video.tag_name.lower() == "video":
            src = video.get_attribute("src")
            poster = video.get_attribute("poster")
            videos.append({
                "type": "video",
                "src": src,
                "poster": poster
            })
        else:  # iframe
            src = video.get_attribute("src")
            if "youtube" in src:
                videos.append({
                    "type": "youtube",
                    "src": src
                })
            elif "vimeo" in src:
                videos.append({
                    "type": "vimeo",
                    "src": src
                })
    
    return videos

def extract_related_articles(driver):
    """Extract related articles"""
    related = []
    
    related_selectors = [
        ".related-articles a",
        ".recommended-articles a",
        ".more-stories a"
    ]
    
    for selector in related_selectors:
        elements = driver.select_all(selector)
        for element in elements:
            href = element.get_attribute("href")
            title = element.get_text().strip()
            
            if href and title and len(title) > 10:
                related.append({
                    "url": href,
                    "title": title
                })
    
    return related[:10]  # Limit to 10 related articles

def extract_social_shares(driver):
    """Extract social media share counts"""
    shares = {}
    
    share_selectors = {
        "facebook": [".fb-share-count", "[data-share='facebook'] .count"],
        "twitter": [".twitter-share-count", "[data-share='twitter'] .count"],
        "linkedin": [".linkedin-share-count", "[data-share='linkedin'] .count"]
    }
    
    for platform, selectors in share_selectors.items():
        for selector in selectors:
            element = driver.select(selector)
            if element:
                count_text = element.get_text().strip()
                count_match = re.search(r'(\d+)', count_text.replace(',', ''))
                if count_match:
                    shares[platform] = int(count_match.group(1))
                    break
    
    return shares

def extract_comments_count(driver):
    """Extract comments count"""
    comment_selectors = [
        ".comments-count",
        ".comment-count",
        ".disqus-comment-count"
    ]
    
    for selector in comment_selectors:
        element = driver.select(selector)
        if element:
            text = element.get_text()
            count_match = re.search(r'(\d+)', text.replace(',', ''))
            if count_match:
                return int(count_match.group(1))
    
    return 0
```

## Social Media Scraper

### Twitter/X Profile Scraper

```python
from botasaurus import browser, Driver
from botasaurus.user_agent import UserAgent
from botasaurus.window_size import WindowSize

@browser(
    user_agent=UserAgent.RANDOM,
    window_size=WindowSize.RANDOM,
    headless=True,
    max_retry=5
)
def scrape_twitter_profile(driver: Driver, profile_url):
    """
    Scrapes Twitter/X profile information
    
    Args:
        profile_url (str): URL of the Twitter profile
        
    Returns:
        dict: Profile information and recent tweets
    """
    
    try:
        # Enable human-like behavior
        driver.enable_human_mode()
        
        # Navigate naturally
        driver.google_get(profile_url)
        
        # Wait for profile to load
        driver.wait_for_element("[data-testid='UserName']", wait=15)
        
        # Extract profile information
        profile_data = {
            "url": profile_url,
            "username": extract_twitter_username(driver),
            "display_name": extract_twitter_display_name(driver),
            "bio": extract_twitter_bio(driver),
            "location": extract_twitter_location(driver),
            "website": extract_twitter_website(driver),
            "join_date": extract_twitter_join_date(driver),
            "following_count": extract_twitter_following_count(driver),
            "followers_count": extract_twitter_followers_count(driver),
            "tweets_count": extract_twitter_tweets_count(driver),
            "verified": check_twitter_verification(driver),
            "profile_image": extract_twitter_profile_image(driver),
            "banner_image": extract_twitter_banner_image(driver),
            "recent_tweets": extract_recent_tweets(driver, limit=20),
            "scraped_at": datetime.utcnow().isoformat(),
            "success": True
        }
        
        return profile_data
        
    except Exception as e:
        return {
            "url": profile_url,
            "error": str(e),
            "success": False,
            "scraped_at": datetime.utcnow().isoformat()
        }

def extract_twitter_username(driver):
    """Extract Twitter username"""
    username_selectors = [
        "[data-testid='UserName'] span",
        ".username",
        ".ProfileHeaderCard-screenname"
    ]
    
    for selector in username_selectors:
        element = driver.select(selector)
        if element:
            username = element.get_text().strip()
            if username.startswith("@"):
                return username[1:]  # Remove @ symbol
            return username
    
    return None

def extract_twitter_display_name(driver):
    """Extract Twitter display name"""
    name_selectors = [
        "[data-testid='UserName'] span:first-child",
        ".ProfileHeaderCard-name",
        ".profile-name"
    ]
    
    for selector in name_selectors:
        element = driver.select(selector)
        if element:
            name = element.get_text().strip()
            if name and not name.startswith("@"):
                return name
    
    return None

def extract_recent_tweets(driver, limit=20):
    """Extract recent tweets from timeline"""
    tweets = []
    
    # Scroll to load more tweets
    for _ in range(3):
        driver.scroll_down(wait_time=2)
        driver.short_random_sleep()
    
    # Extract tweet elements
    tweet_elements = driver.select_all("[data-testid='tweet']")[:limit]
    
    for tweet_element in tweet_elements:
        try:
            tweet_data = {
                "text": extract_tweet_text(tweet_element),
                "timestamp": extract_tweet_timestamp(tweet_element),
                "likes": extract_tweet_metric(tweet_element, "like"),
                "retweets": extract_tweet_metric(tweet_element, "retweet"),
                "replies": extract_tweet_metric(tweet_element, "reply"),
                "images": extract_tweet_images(tweet_element),
                "links": extract_tweet_links(tweet_element)
            }
            
            if tweet_data["text"]:  # Only add if we got text content
                tweets.append(tweet_data)
                
        except Exception as e:
            continue  # Skip problematic tweets
    
    return tweets

# Additional helper functions for Twitter scraping would go here...
```

## API Data Scraper

### REST API Scraper with Authentication

```python
@request(
    parallel=5,
    max_retry=3,
    cache=True
)
def scrape_api_data(request, api_config):
    """
    Scrapes data from REST APIs with authentication and pagination
    
    Args:
        api_config (dict): API configuration including endpoint, auth, params
        
    Returns:
        dict: API response data
    """
    
    try:
        # Setup authentication headers
        headers = {
            "User-Agent": "Mozilla/5.0 (compatible; DataScraper/1.0)",
            "Accept": "application/json",
            "Content-Type": "application/json"
        }
        
        # Add authentication
        auth_type = api_config.get("auth_type")
        if auth_type == "bearer":
            headers["Authorization"] = f"Bearer {api_config['token']}"
        elif auth_type == "api_key":
            headers["X-API-Key"] = api_config["api_key"]
        elif auth_type == "basic":
            import base64
            credentials = base64.b64encode(
                f"{api_config['username']}:{api_config['password']}".encode()
            ).decode()
            headers["Authorization"] = f"Basic {credentials}"
        
        # Handle pagination
        all_data = []
        page = 1
        max_pages = api_config.get("max_pages", 10)
        
        while page <= max_pages:
            # Prepare request parameters
            params = api_config.get("params", {}).copy()
            params["page"] = page
            params["limit"] = api_config.get("page_size", 100)
            
            # Make API request
            response = request.get(
                api_config["endpoint"],
                headers=headers,
                params=params,
                timeout=30
            )
            
            if response.status_code == 200:
                data = response.json()
                
                # Extract items based on API structure
                items = extract_api_items(data, api_config.get("data_path", "data"))
                
                if not items:  # No more data
                    break
                
                all_data.extend(items)
                
                # Check if there are more pages
                if not has_more_pages(data, api_config):
                    break
                
                page += 1
                
                # Rate limiting
                import time
                time.sleep(api_config.get("rate_limit_delay", 0.5))
                
            elif response.status_code == 429:  # Rate limited
                time.sleep(30)  # Wait longer for rate limit
                continue
            else:
                response.raise_for_status()
        
        return {
            "endpoint": api_config["endpoint"],
            "total_items": len(all_data),
            "data": all_data,
            "pages_scraped": page - 1,
            "scraped_at": datetime.utcnow().isoformat(),
            "success": True
        }
        
    except Exception as e:
        return {
            "endpoint": api_config.get("endpoint", "unknown"),
            "error": str(e),
            "success": False,
            "scraped_at": datetime.utcnow().isoformat()
        }

def extract_api_items(data, data_path):
    """Extract items from API response based on path"""
    try:
        # Navigate to data using dot notation (e.g., "response.items")
        current = data
        for key in data_path.split("."):
            current = current[key]
        
        return current if isinstance(current, list) else [current]
    except (KeyError, TypeError):
        return []

def has_more_pages(data, config):
    """Check if API has more pages"""
    pagination_indicators = [
        "has_more",
        "has_next_page",
        "next_page",
        "pagination.has_more"
    ]
    
    for indicator in pagination_indicators:
        try:
            current = data
            for key in indicator.split("."):
                current = current[key]
            return bool(current)
        except (KeyError, TypeError):
            continue
    
    # Fallback: check if current page is full
    items_path = config.get("data_path", "data")
    items = extract_api_items(data, items_path)
    expected_size = config.get("page_size", 100)
    
    return len(items) >= expected_size

# Usage example
api_configs = [
    {
        "endpoint": "https://api.example.com/products",
        "auth_type": "bearer",
        "token": "your_token_here",
        "data_path": "data.products",
        "max_pages": 5,
        "page_size": 50,
        "rate_limit_delay": 1.0,
        "params": {
            "category": "electronics",
            "status": "active"
        }
    }
]

results = scrape_api_data(api_configs)
```

## Form Submission Scraper

### Search Form Automation

```python
@browser(
    headless=False,  # Keep visible for debugging
    max_retry=3,
    reuse_driver=True
)
def scrape_search_results(driver: Driver, search_data):
    """
    Automates form submission and scrapes search results
    
    Args:
        search_data (dict): Search parameters and form details
        
    Returns:
        dict: Search results and metadata
    """
    
    try:
        # Navigate to search page
        driver.get(search_data["search_url"])
        
        # Wait for form to load
        driver.wait_for_element(search_data["form_selector"], wait=10)
        
        # Fill out search form
        fill_search_form(driver, search_data)
        
        # Submit form
        submit_button = driver.select(search_data.get("submit_selector", "button[type='submit']"))
        if submit_button:
            driver.enable_human_mode()  # Human-like clicking
            submit_button.click()
        else:
            # Alternative: press Enter
            search_input = driver.select(search_data["search_input_selector"])
            if search_input:
                search_input.send_keys("\n")
        
        # Wait for results to load
        driver.wait_for_element(search_data["results_selector"], wait=15)
        
        # Extract search results
        results = extract_search_results(driver, search_data)
        
        # Handle pagination if needed
        if search_data.get("scrape_all_pages", False):
            results.extend(scrape_additional_pages(driver, search_data))
        
        return {
            "search_query": search_data.get("query", ""),
            "search_url": search_data["search_url"],
            "total_results": len(results),
            "results": results,
            "scraped_at": datetime.utcnow().isoformat(),
            "success": True
        }
        
    except Exception as e:
        driver.save_screenshot("form_error.png")
        return {
            "search_url": search_data.get("search_url", ""),
            "error": str(e),
            "success": False,
            "scraped_at": datetime.utcnow().isoformat()
        }

def fill_search_form(driver, search_data):
    """Fill out search form with provided data"""
    
    # Fill search input
    if "search_input_selector" in search_data:
        search_input = driver.select(search_data["search_input_selector"])
        if search_input:
            search_input.clear()
            search_input.type(search_data.get("query", ""))
            driver.short_random_sleep()
    
    # Fill additional form fields
    form_fields = search_data.get("form_fields", {})
    for field_selector, value in form_fields.items():
        field = driver.select(field_selector)
        if field:
            field_type = field.get_attribute("type") or field.tag_name.lower()
            
            if field_type in ["text", "email", "password"]:
                field.clear()
                field.type(value)
            elif field_type == "select":
                field.select_option(value)
            elif field_type == "checkbox":
                if value and not field.is_selected():
                    field.click()
                elif not value and field.is_selected():
                    field.click()
            elif field_type == "radio":
                if value:
                    field.click()
            
            driver.short_random_sleep()
    
    # Handle dropdowns/select menus
    dropdowns = search_data.get("dropdowns", {})
    for dropdown_selector, option_value in dropdowns.items():
        dropdown = driver.select(dropdown_selector)
        if dropdown:
            dropdown.click()
            driver.short_random_sleep()
            
            # Try to select option by value, text, or index
            option_selected = False
            
            # Try by value
            option = dropdown.select(f"option[value='{option_value}']")
            if option:
                option.click()
                option_selected = True
            
            # Try by text content
            if not option_selected:
                options = dropdown.select_all("option")
                for opt in options:
                    if opt.get_text().strip().lower() == str(option_value).lower():
                        opt.click()
                        option_selected = True
                        break
            
            driver.short_random_sleep()

def extract_search_results(driver, search_data):
    """Extract search results from the page"""
    results = []
    
    result_elements = driver.select_all(search_data["results_selector"])
    
    for element in result_elements:
        try:
            result_data = {}
            
            # Extract data based on selectors
            extractors = search_data.get("extractors", {})
            for field_name, selector in extractors.items():
                field_element = element.select(selector)
                if field_element:
                    if field_name.endswith("_url") or field_name.endswith("_link"):
                        result_data[field_name] = field_element.get_attribute("href")
                    elif field_name.endswith("_image"):
                        result_data[field_name] = field_element.get_attribute("src")
                    else:
                        result_data[field_name] = field_element.get_text().strip()
            
            # Only add if we extracted meaningful data
            if any(result_data.values()):
                results.append(result_data)
                
        except Exception as e:
            continue  # Skip problematic results
    
    return results

def scrape_additional_pages(driver, search_data):
    """Scrape additional pages of search results"""
    all_results = []
    max_pages = search_data.get("max_pages", 5)
    current_page = 1
    
    while current_page < max_pages:
        # Try to find and click next page button
        next_selectors = [
            search_data.get("next_page_selector", ".next"),
            "a[aria-label='Next']",
            ".pagination .next",
            ".paging .next"
        ]
        
        next_button = None
        for selector in next_selectors:
            next_button = driver.select(selector)
            if next_button and next_button.is_enabled():
                break
        
        if not next_button:
            break  # No more pages
        
        # Click next page
        driver.enable_human_mode()
        next_button.click()
        
        # Wait for new results to load
        driver.wait_for_element(search_data["results_selector"], wait=10)
        driver.short_random_sleep()
        
        # Extract results from new page
        page_results = extract_search_results(driver, search_data)
        
        if not page_results:  # No results on this page
            break
        
        all_results.extend(page_results)
        current_page += 1
    
    return all_results

# Usage example
search_config = {
    "search_url": "https://example-site.com/search",
    "form_selector": "form.search-form",
    "search_input_selector": "input[name='q']",
    "submit_selector": "button.search-submit",
    "results_selector": ".search-result",
    "extractors": {
        "title": ".result-title",
        "description": ".result-description",
        "url": ".result-title a",
        "price": ".price"
    },
    "scrape_all_pages": True,
    "max_pages": 3,
    "next_page_selector": ".pagination .next",
    "query": "laptop computers",
    "form_fields": {
        "select[name='category']": "electronics",
        "input[name='min_price']": "100",
        "input[name='max_price']": "2000"
    },
    "dropdowns": {
        "select[name='sort']": "price_asc"
    }
}

results = scrape_search_results(search_config)
```

## Conclusion

These examples demonstrate various patterns for creating robust scrapers with Botasaurus:

1. **E-commerce scrapers** handle dynamic pricing, product variations, and complex layouts
2. **News scrapers** extract structured content with metadata and media
3. **Social media scrapers** navigate authentication and rate limiting
4. **API scrapers** handle pagination, authentication, and rate limiting
5. **Form scrapers** automate user interactions and form submissions

Each example includes error handling, data validation, and performance optimizations. Use these patterns as starting points for your own scrapers, adapting the selectors and logic to match your target websites.

Remember to always:
- Respect robots.txt and terms of service
- Implement appropriate delays to avoid overwhelming servers
- Handle errors gracefully with proper logging
- Use human-like behavior for sites with bot detection
- Monitor and maintain your scrapers as websites change