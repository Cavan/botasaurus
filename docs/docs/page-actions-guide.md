---
sidebar_position: 6
title: Page Actions Guide
description: Complete guide to performing page actions like clicking tabs, navigation, and complex interactions with Botasaurus
---

# Page Actions Guide

This comprehensive guide covers advanced page interactions with Botasaurus, including tab management, navigation, dynamic content handling, and complex user workflows that go beyond basic element clicking.

## Understanding "Tabs" in Web Automation

When working with web automation, the term "tabs" can refer to two different things:

1. **Page Element Tabs** - UI elements within a webpage that switch content (like navigation tabs, content panels). These are clickable elements that change what's displayed on the same page.
2. **Browser Tabs** - Actual browser windows/tabs that contain different web pages.

This guide covers both types, with the primary focus on **page element tabs** which are more commonly used in web scraping and automation workflows.

## Browser Tab Management (Window/Tab Handling)

This section covers managing browser tabs and windows - opening new browser tabs, switching between them, and managing multiple browser windows.

### Opening New Tabs

```python
from botasaurus import browser, Driver

@browser
def work_with_multiple_tabs(driver: Driver, data):
    # Open initial page
    driver.get("https://example.com")
    
    # Method 1: Open link in new tab using JavaScript
    driver.run_js("window.open('https://example.com/page2', '_blank')")
    
    # Method 2: Ctrl+Click to open in new tab
    link = driver.select("a[href='/page2']")
    link.click(modifier="ctrl")  # Opens in new tab
    
    # Method 3: Middle-click to open in new tab
    link.click(button="middle")
    
    return {"tabs_opened": "success"}
```

### Switching Between Tabs

```python
@browser
def switch_tabs_example(driver: Driver, data):
    # Start with one tab
    driver.get("https://example.com/tab1")
    
    # Open second tab
    driver.run_js("window.open('https://example.com/tab2', '_blank')")
    
    # Get all window handles
    all_tabs = driver.window_handles
    print(f"Number of tabs: {len(all_tabs)}")
    
    # Switch to the second tab (index 1)
    driver.switch_to.window(all_tabs[1])
    
    # Extract data from second tab
    tab2_title = driver.title
    
    # Switch back to first tab (index 0)
    driver.switch_to.window(all_tabs[0])
    
    # Extract data from first tab
    tab1_title = driver.title
    
    return {
        "tab1_title": tab1_title,
        "tab2_title": tab2_title
    }
```

### Closing Tabs

```python
@browser
def manage_tabs_lifecycle(driver: Driver, data):
    # Open main page
    driver.get("https://example.com")
    
    # Open multiple tabs
    for i in range(3):
        driver.run_js(f"window.open('https://example.com/page{i+2}', '_blank')")
    
    all_tabs = driver.window_handles
    print(f"Opened {len(all_tabs)} tabs")
    
    # Process each tab and close it
    results = []
    for i, tab_handle in enumerate(all_tabs[1:], 1):  # Skip first tab
        driver.switch_to.window(tab_handle)
        
        # Extract data
        title = driver.title
        results.append({"tab": i, "title": title})
        
        # Close current tab
        driver.close()
    
    # Switch back to main tab
    driver.switch_to.window(all_tabs[0])
    
    return {"processed_tabs": results}
```

## Page Navigation

### Basic Navigation

```python
@browser
def navigation_example(driver: Driver, data):
    # Navigate to initial page
    driver.get("https://example.com/form")
    
    # Fill out form
    driver.type("input[name='search']", "botasaurus")
    driver.click("button[type='submit']")
    
    # Wait for results page
    driver.wait_for_element(".search-results", wait=10)
    results_title = driver.title
    
    # Go back to form
    driver.back()
    
    # Verify we're back at the form
    assert driver.exists("input[name='search']"), "Not back at form page"
    
    # Go forward to results again
    driver.forward()
    
    # Refresh the page
    driver.refresh()
    driver.wait_for_element(".search-results", wait=10)
    
    return {
        "navigation": "success",
        "results_title": results_title
    }
```

### Advanced Navigation Patterns

```python
@browser
def multi_step_navigation(driver: Driver, data):
    """Navigate through a multi-step process with back/forward"""
    
    # Step 1: Start page
    driver.get("https://example.com/wizard/step1")
    driver.type("input[name='name']", "John Doe")
    driver.click("button.next")
    
    # Step 2: Second page
    driver.wait_for_element(".step2-content", wait=10)
    driver.select("select[name='country']").select_by_text("United States")
    driver.click("button.next")
    
    # Step 3: Review page
    driver.wait_for_element(".step3-content", wait=10)
    
    # Go back to edit something
    driver.back()
    driver.back()
    
    # Make changes on step 1
    driver.clear("input[name='name']")
    driver.type("input[name='name']", "Jane Smith")
    driver.click("button.next")
    
    # Navigate forward through steps again
    driver.forward()
    
    # Final submission
    driver.click("button.submit")
    
    return {"wizard_completed": True}
```

## Page Element Interactions

This section covers interacting with UI elements within web pages, particularly tab elements that switch content, accordions, modals, and other dynamic components.

### Clicking Page Tabs to Load Content

**This addresses the most common use case** - clicking on tab elements within a webpage to load different content sections. These are UI elements like navigation tabs, content switchers, or tabbed panels.

```python
@browser
def handle_content_tabs(driver: Driver, data):
    """Handle tabbed interfaces that load content dynamically"""
    
    driver.get("https://example.com/dashboard")
    
    # Wait for initial content
    driver.wait_for_element(".tab-container", wait=10)
    
    tab_data = {}
    
    # Get all tab buttons
    tabs = driver.select_all(".tab-button")
    
    for i, tab in enumerate(tabs):
        # Click tab
        tab.click()
        
        # Wait for content to load (important for dynamic content)
        driver.wait_for_element(f".tab-content-{i+1}", wait=10)
        
        # Optional: Add small delay for animations
        driver.sleep(1)
        
        # Extract content from this tab
        content = driver.get_text(".tab-content")
        tab_name = tab.get_text()
        
        tab_data[tab_name] = {
            "content": content,
            "loaded_at": driver.run_js("return Date.now()")
        }
    
    return tab_data
```

#### Common Tab Element Patterns

Here are examples of different types of tab elements you might encounter:

```python
@browser
def handle_different_tab_types(driver: Driver, data):
    """Examples of different tab element patterns"""
    
    driver.get("https://example.com")
    
    # Pattern 1: Navigation tabs (most common)
    nav_tabs = driver.select_all(".nav-tabs li")
    for tab in nav_tabs:
        tab.click()
        driver.wait_for_element(".tab-pane.active", wait=5)
        content = driver.get_text(".tab-pane.active")
    
    # Pattern 2: Button-style tabs
    button_tabs = driver.select_all(".tab-buttons button")
    for tab in button_tabs:
        tab.click()
        # Wait for specific content area to update
        driver.wait_for_element(".content-area[data-loaded='true']", wait=10)
    
    # Pattern 3: Link-based tabs with hash navigation
    link_tabs = driver.select_all(".tabs a[href^='#']")
    for tab in link_tabs:
        tab.click()
        # Wait for URL hash to change
        tab_id = tab.get_attribute("href").replace("#", "")
        driver.wait_for_element(f"#{tab_id}.active", wait=5)
    
    # Pattern 4: Custom tab elements with data attributes
    custom_tabs = driver.select_all("[data-tab]")
    for tab in custom_tabs:
        tab.click()
        tab_target = tab.get_attribute("data-tab")
        driver.wait_for_element(f"[data-tab-content='{tab_target}'].active", wait=5)
    
    return {"tabs_processed": len(nav_tabs + button_tabs + link_tabs + custom_tabs)}
```

### Working with Accordions and Collapsible Content

```python
@browser
def handle_accordions(driver: Driver, data):
    """Expand and interact with accordion/collapsible content"""
    
    driver.get("https://example.com/faq")
    
    faq_data = []
    
    # Find all accordion headers
    accordion_headers = driver.select_all(".accordion-header")
    
    for header in accordion_headers:
        # Click to expand
        header.click()
        
        # Wait for content to be visible
        driver.wait_for_element(".accordion-content:not(.hidden)", wait=5)
        
        # Extract question and answer
        question = header.get_text()
        answer = driver.get_text(".accordion-content")
        
        faq_data.append({
            "question": question,
            "answer": answer
        })
        
        # Optional: Collapse before moving to next (if needed)
        header.click()
        driver.sleep(0.5)
    
    return faq_data
```

### Modal and Popup Handling

```python
@browser
def handle_modals_and_popups(driver: Driver, data):
    """Handle modals, popups, and overlay content"""
    
    driver.get("https://example.com/products")
    
    products = []
    product_links = driver.select_all(".product-card .view-details")
    
    for link in product_links[:5]:  # Limit to first 5 products
        # Click to open modal
        link.click()
        
        # Wait for modal to appear
        modal = driver.wait_for_element(".modal", wait=10)
        
        # Extract product details from modal
        product_name = driver.get_text(".modal .product-title")
        product_price = driver.get_text(".modal .price")
        product_description = driver.get_text(".modal .description")
        
        products.append({
            "name": product_name,
            "price": product_price,
            "description": product_description
        })
        
        # Close modal
        close_button = driver.select(".modal .close-button")
        if close_button:
            close_button.click()
        else:
            # Alternative: Press Escape key
            driver.press_key("Escape")
        
        # Wait for modal to disappear
        driver.wait_for_element(".modal", wait=5, state="hidden")
    
    return products
```

## Complex Interaction Workflows

### Multi-Step Form with Conditional Logic

```python
@browser
def complex_form_workflow(driver: Driver, form_data):
    """Handle complex forms with conditional fields and validation"""
    
    driver.get("https://example.com/complex-form")
    
    # Step 1: Basic information
    driver.type("input[name='firstName']", form_data["first_name"])
    driver.type("input[name='lastName']", form_data["last_name"])
    driver.type("input[name='email']", form_data["email"])
    
    # Step 2: Conditional fields based on user type
    user_type = form_data.get("user_type", "individual")
    driver.select("select[name='userType']").select_by_value(user_type)
    
    # Wait for conditional fields to appear
    if user_type == "business":
        driver.wait_for_element("input[name='companyName']", wait=5)
        driver.type("input[name='companyName']", form_data["company_name"])
        driver.type("input[name='taxId']", form_data["tax_id"])
    
    # Step 3: Handle dynamic address fields
    driver.type("input[name='address']", form_data["address"])
    
    # Country selection affects state/province field
    country_select = driver.select("select[name='country']")
    country_select.select_by_text(form_data["country"])
    
    # Wait for state field to update based on country
    driver.sleep(1)
    
    if form_data["country"] == "United States":
        state_select = driver.wait_for_element("select[name='state']", wait=5)
        driver.select("select[name='state']").select_by_text(form_data["state"])
    else:
        # For other countries, there might be a text input
        if driver.exists("input[name='province']"):
            driver.type("input[name='province']", form_data.get("province", ""))
    
    # Step 4: File upload (if required)
    if form_data.get("document_required"):
        file_input = driver.select("input[type='file']")
        file_input.send_keys(form_data["document_path"])
        
        # Wait for file upload confirmation
        driver.wait_for_element(".upload-success", wait=30)
    
    # Step 5: Submit and handle confirmation
    submit_button = driver.select("button[type='submit']")
    submit_button.click()
    
    # Handle potential validation errors
    if driver.exists(".error-message"):
        errors = driver.select_all(".error-message")
        error_messages = [error.get_text() for error in errors]
        return {"success": False, "errors": error_messages}
    
    # Wait for success page or confirmation
    confirmation = driver.wait_for_element(".success-message", wait=15)
    confirmation_text = confirmation.get_text()
    
    return {
        "success": True,
        "confirmation": confirmation_text,
        "form_data": form_data
    }
```

### E-commerce Shopping Flow

```python
@browser
def shopping_cart_workflow(driver: Driver, products_to_buy):
    """Complete shopping flow from product selection to checkout"""
    
    driver.get("https://example-shop.com")
    
    cart_items = []
    
    # Step 1: Search and add products to cart
    for product in products_to_buy:
        # Search for product
        search_box = driver.select("input[name='search']")
        search_box.clear()
        search_box.type(product["name"])
        driver.click("button[type='submit']")
        
        # Wait for search results
        driver.wait_for_element(".search-results", wait=10)
        
        # Select first matching product
        first_product = driver.select(".product-item:first-child")
        product_link = first_product.select("a")
        product_link.click()
        
        # Wait for product page
        driver.wait_for_element(".product-details", wait=10)
        
        # Select quantity
        if product.get("quantity", 1) > 1:
            qty_input = driver.select("input[name='quantity']")
            qty_input.clear()
            qty_input.type(str(product["quantity"]))
        
        # Add to cart
        add_to_cart_btn = driver.select(".add-to-cart")
        add_to_cart_btn.click()
        
        # Wait for cart confirmation
        driver.wait_for_element(".cart-notification", wait=5)
        
        # Record added item
        item_name = driver.get_text(".product-title")
        item_price = driver.get_text(".product-price")
        
        cart_items.append({
            "name": item_name,
            "price": item_price,
            "quantity": product.get("quantity", 1)
        })
        
        # Go back to search or continue shopping
        driver.get("https://example-shop.com")
    
    # Step 2: View cart and proceed to checkout
    cart_icon = driver.select(".cart-icon")
    cart_icon.click()
    
    # Wait for cart page
    driver.wait_for_element(".cart-items", wait=10)
    
    # Verify cart contents
    cart_total = driver.get_text(".cart-total")
    
    # Proceed to checkout
    checkout_btn = driver.select(".proceed-checkout")
    checkout_btn.click()
    
    # Step 3: Fill shipping information
    driver.wait_for_element(".checkout-form", wait=10)
    
    shipping_info = {
        "firstName": "John",
        "lastName": "Doe",
        "address": "123 Main St",
        "city": "Anytown",
        "zipCode": "12345"
    }
    
    for field, value in shipping_info.items():
        driver.type(f"input[name='{field}']", value)
    
    # Continue to payment
    driver.click(".continue-to-payment")
    
    # Step 4: Payment information (demo only - don't use real payment info)
    driver.wait_for_element(".payment-form", wait=10)
    
    # This would typically be where you handle payment
    # For demo purposes, we'll just verify we reached this step
    payment_form_exists = driver.exists(".payment-form")
    
    return {
        "cart_items": cart_items,
        "cart_total": cart_total,
        "reached_payment": payment_form_exists,
        "workflow_completed": True
    }
```

## Advanced Page State Management

### Handling Page State Changes

```python
@browser
def monitor_page_state_changes(driver: Driver, data):
    """Monitor and respond to dynamic page state changes"""
    
    driver.get("https://example.com/live-dashboard")
    
    # Initial state
    initial_data = driver.get_text(".data-counter")
    
    # Click refresh/update button
    refresh_btn = driver.select(".refresh-data")
    refresh_btn.click()
    
    # Wait for state change using multiple strategies
    def wait_for_data_change():
        current_data = driver.get_text(".data-counter")
        return current_data != initial_data
    
    # Method 1: Custom wait condition
    driver.wait.until(wait_for_data_change, timeout=30)
    
    # Method 2: Wait for specific element state
    driver.wait_for_element(".loading-spinner", wait=5, state="visible")
    driver.wait_for_element(".loading-spinner", wait=30, state="hidden")
    
    # Method 3: Wait for attribute changes
    status_element = driver.select(".status-indicator")
    driver.wait.until(
        lambda: status_element.get_attribute("class") == "status-indicator updated",
        timeout=15
    )
    
    updated_data = driver.get_text(".data-counter")
    
    return {
        "initial_data": initial_data,
        "updated_data": updated_data,
        "state_changed": initial_data != updated_data
    }
```

### Session Management Across Pages

```python
@browser
def maintain_session_across_pages(driver: Driver, credentials):
    """Maintain session state while navigating multiple pages"""
    
    # Step 1: Login
    driver.get("https://example.com/login")
    driver.type("input[name='username']", credentials["username"])
    driver.type("input[name='password']", credentials["password"])
    driver.click("button[type='submit']")
    
    # Wait for successful login
    driver.wait_for_element(".dashboard", wait=10)
    
    # Step 2: Navigate to different sections while maintaining session
    sections_data = {}
    
    sections = [
        {"name": "Profile", "url": "/profile"},
        {"name": "Settings", "url": "/settings"}, 
        {"name": "Reports", "url": "/reports"}
    ]
    
    for section in sections:
        # Navigate to section
        driver.get(f"https://example.com{section['url']}")
        
        # Verify still logged in (check for logout button or user info)
        if not driver.exists(".user-menu"):
            raise Exception(f"Session lost on {section['name']} page")
        
        # Extract section-specific data
        section_title = driver.get_text("h1")
        section_content = driver.get_text(".main-content")
        
        sections_data[section["name"]] = {
            "title": section_title,
            "content_preview": section_content[:200]  # First 200 chars
        }
    
    # Step 3: Logout
    user_menu = driver.select(".user-menu")
    user_menu.click()
    
    logout_btn = driver.select(".logout-button")
    logout_btn.click()
    
    # Verify logout
    driver.wait_for_element(".login-form", wait=10)
    
    return {
        "sections_accessed": sections_data,
        "session_maintained": True,
        "logout_successful": True
    }
```

## Best Practices and Common Patterns

### Robust Tab Management

```python
@browser
def robust_tab_management(driver: Driver, urls):
    """Robust pattern for managing multiple tabs with error handling"""
    
    results = []
    original_window = driver.current_window_handle
    
    try:
        for i, url in enumerate(urls):
            # Open new tab
            driver.run_js("window.open('about:blank', '_blank')")
            
            # Switch to new tab
            all_windows = driver.window_handles
            driver.switch_to.window(all_windows[-1])
            
            try:
                # Navigate and extract data
                driver.get(url)
                driver.wait_for_element("body", wait=30)
                
                title = driver.title
                results.append({
                    "url": url,
                    "title": title,
                    "success": True
                })
                
            except Exception as e:
                results.append({
                    "url": url,
                    "error": str(e),
                    "success": False
                })
            
            finally:
                # Always close the current tab (if not the original)
                if driver.current_window_handle != original_window:
                    driver.close()
                    
                # Switch back to original window
                driver.switch_to.window(original_window)
    
    except Exception as e:
        # Ensure we're back to original window even on failure
        try:
            driver.switch_to.window(original_window)
        except:
            pass
        raise e
    
    return results
```

### Handling Slow-Loading Dynamic Content

```python
@browser
def handle_slow_dynamic_content(driver: Driver, data):
    """Pattern for handling slow-loading dynamic content"""
    
    driver.get("https://example.com/slow-page")
    
    # Pattern 1: Progressive loading detection
    content_sections = []
    
    # Wait for initial page structure
    driver.wait_for_element(".content-container", wait=30)
    
    # Wait for each content section to load
    expected_sections = [".section-1", ".section-2", ".section-3"]
    
    for section_selector in expected_sections:
        try:
            # Wait for section to appear
            section = driver.wait_for_element(section_selector, wait=60)
            
            # Wait for section content to be populated (not just empty div)
            driver.wait.until(
                lambda: len(section.get_text().strip()) > 0,
                timeout=30,
                poll_frequency=1
            )
            
            content = section.get_text()
            content_sections.append({
                "section": section_selector,
                "content": content,
                "loaded": True
            })
            
        except Exception as e:
            content_sections.append({
                "section": section_selector,
                "error": str(e),
                "loaded": False
            })
    
    # Pattern 2: Wait for loading indicators to disappear
    loading_selectors = [".loading", ".spinner", ".skeleton-loader"]
    
    for selector in loading_selectors:
        if driver.exists(selector):
            driver.wait_for_element(selector, wait=60, state="hidden")
    
    # Pattern 3: Wait for final state indicators
    driver.wait_for_element(".content-loaded", wait=60)
    
    return {
        "sections": content_sections,
        "fully_loaded": True
    }
```

## Complete Real-World Example

Here's a complete example that combines multiple page action patterns:

```python
@browser
def complete_workflow_example(driver: Driver, data):
    """Complete example combining multiple page interaction patterns"""
    
    # Start at main page
    driver.get("https://example.com")
    
    workflow_results = {
        "steps": [],
        "tabs_used": 0,
        "navigation_count": 0
    }
    
    # Step 1: Navigate to product catalog
    driver.click("nav a[href='/products']")
    driver.wait_for_element(".product-grid", wait=10)
    workflow_results["steps"].append("Reached product catalog")
    workflow_results["navigation_count"] += 1
    
    # Step 2: Open product details in new tabs
    product_links = driver.select_all(".product-card a")[:3]
    
    original_window = driver.current_window_handle
    product_details = []
    
    for i, product_link in enumerate(product_links):
        # Open in new tab using Ctrl+Click
        product_link.click(modifier="ctrl")
        workflow_results["tabs_used"] += 1
        
        # Switch to new tab
        all_tabs = driver.window_handles
        driver.switch_to.window(all_tabs[-1])
        
        # Wait for product page to load
        driver.wait_for_element(".product-details", wait=15)
        
        # Extract product information
        name = driver.get_text(".product-name")
        price = driver.get_text(".product-price")
        description = driver.get_text(".product-description")
        
        product_details.append({
            "name": name,
            "price": price,
            "description": description[:100]  # First 100 chars
        })
        
        # Close tab and return to original
        driver.close()
        driver.switch_to.window(original_window)
    
    workflow_results["steps"].append(f"Collected {len(product_details)} product details")
    
    # Step 3: Navigate to contact form
    driver.click("nav a[href='/contact']")
    driver.wait_for_element(".contact-form", wait=10)
    workflow_results["navigation_count"] += 1
    
    # Step 4: Fill and submit contact form
    form_data = {
        "name": "Test User",
        "email": "test@example.com",
        "message": f"Interested in: {', '.join([p['name'] for p in product_details])}"
    }
    
    for field, value in form_data.items():
        driver.type(f"input[name='{field}'], textarea[name='{field}']", value)
    
    # Submit form
    driver.click("button[type='submit']")
    
    # Wait for confirmation
    success_message = driver.wait_for_element(".success-message", wait=15)
    confirmation_text = success_message.get_text()
    
    workflow_results["steps"].append("Form submitted successfully")
    
    # Step 5: Go back and verify we can navigate back
    driver.back()
    
    # Verify we're back at the form
    if driver.exists(".contact-form"):
        workflow_results["steps"].append("Successfully navigated back to form")
    
    # Go forward to confirmation again
    driver.forward()
    workflow_results["navigation_count"] += 2
    
    return {
        "workflow_results": workflow_results,
        "product_details": product_details,
        "confirmation": confirmation_text,
        "completed": True
    }

# Usage example
if __name__ == "__main__":
    result = complete_workflow_example({})
    print(f"Workflow completed with {len(result['workflow_results']['steps'])} steps")
    for step in result['workflow_results']['steps']:
        print(f"✓ {step}")
```

This guide covers the most common page action patterns you'll need when building complex web scrapers and automation scripts with Botasaurus. Each example is designed to be practical and can be adapted to your specific use cases.

## Quick Reference

### Tab Management
- `driver.run_js("window.open(url, '_blank')")` - Open new tab
- `driver.window_handles` - Get all tab handles
- `driver.switch_to.window(handle)` - Switch to tab
- `driver.close()` - Close current tab

### Navigation
- `driver.back()` - Navigate back
- `driver.forward()` - Navigate forward  
- `driver.refresh()` - Refresh page

### Dynamic Content
- `driver.wait_for_element(selector, wait=X)` - Wait for element
- `driver.wait_for_element(selector, state="hidden")` - Wait for element to hide
- `element.click()` - Click tabs, buttons, etc.

### Best Practices
- Always handle exceptions when working with multiple tabs
- Use explicit waits for dynamic content
- Clean up tabs to prevent memory issues
- Verify page state before performing actions
- Use human-like delays when needed: `driver.sleep(1)`