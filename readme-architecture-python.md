# Playwright Python - Architecture Documentation

## Overview

Playwright for Python is the official Python language binding for the Playwright browser automation framework. It provides both synchronous and asynchronous APIs, enabling Python developers to automate Chromium, Firefox, and WebKit browsers with native Python idioms and patterns.

## Architecture Components

### High-Level Architecture

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│ Python          │    │ Playwright       │    │ Node.js Runtime │
│ Application     │    │ Python Bindings  │    │   + Playwright  │
│                 │    │                  │    │                 │
│ sync_api/       │◄──►│ JSON-RPC Bridge  │◄──►│ Browser Control │
│ async_api       │    │                  │    │                 │
└─────────────────┘    └──────────────────┘    └─────────────────┘
                                │                        │
                                │                        ▼
                                │               ┌─────────────────┐
                                │               │ Browser Process │
                                │               │ (Chrome/Firefox/│
                                │               │    WebKit)      │
                                │               └─────────────────┘
                                ▼
                       ┌──────────────────┐
                       │ Python Objects   │
                       │ (Page, Browser,  │
                       │  Context, etc.)  │
                       └──────────────────┘
```

### Package Structure

#### 1. **Dual API Design**
```
playwright/
├── sync_api/          # Synchronous API
│   ├── __init__.py   # sync_playwright() entry point
│   └── _generated.py # Auto-generated sync API classes
├── async_api/         # Asynchronous API  
│   ├── __init__.py   # async_playwright() entry point
│   └── _generated.py # Auto-generated async API classes
└── _impl/            # Internal implementation
    ├── __init__.py
    ├── _connection.py    # Node.js communication
    ├── _playwright.py    # Core Playwright class
    ├── _browser.py       # Browser management
    ├── _page.py         # Page automation
    └── ...              # Other implementation files
```

#### 2. **Implementation Layer** (`playwright/_impl/`)
- **Purpose**: Core implementation shared between sync and async APIs
- **Communication**: JSON-RPC over subprocess pipes
- **Design Pattern**: Internal classes with public API wrappers

#### 3. **Code Generation System**
- **Purpose**: Generates Python APIs from TypeScript definitions
- **Files**: `scripts/generate_api.py`, `scripts/generate_async_api.py`
- **Process**: Parses TypeScript → Generates Python classes
- **Benefits**: API consistency with upstream Playwright

## How It Works

### 1. **Initialization Process**

#### Synchronous API
```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch()
    page = browser.new_page()
    page.goto("https://example.com")
```

#### Asynchronous API
```python
import asyncio
from playwright.async_api import async_playwright

async def main():
    async with async_playwright() as p:
        browser = await p.chromium.launch()
        page = await browser.new_page()
        await page.goto("https://example.com")

asyncio.run(main())
```

**Internal Flow**:
1. `sync_playwright()` or `async_playwright()` starts Node.js subprocess
2. Establishes JSON-RPC communication channel
3. Creates Python wrapper objects for Playwright entities
4. Maps method calls to RPC messages

### 2. **Communication Protocol**

#### JSON-RPC Message Flow
```python
# Python method call
page.goto("https://example.com")

# Translates to JSON-RPC message
{
  "id": 1,
  "method": "goto", 
  "params": {
    "url": "https://example.com",
    "guid": "page-uuid"
  }
}

# Node.js executes and responds
{
  "id": 1,
  "result": {
    "response": {...}
  }
}
```

#### Connection Management (`_connection.py`)
```python
class Connection:
    def __init__(self):
        self._transport = subprocess.Popen([...])  # Node.js process
        self._callbacks = {}  # Pending RPC calls
        
    def send_message_to_server(self, message):
        # Send JSON-RPC to Node.js
        
    def _on_message(self, message):
        # Handle responses from Node.js
```

### 3. **Sync vs Async Implementation**

#### Synchronous Wrapper Pattern
```python
# Internal async implementation
class PageImpl:
    async def goto(self, url: str) -> None:
        # Send RPC, await response
        await self._connection.send(...)

# Synchronous wrapper
class Page:
    def goto(self, url: str) -> None:
        # Run async method in event loop
        return self._impl_obj.goto(url)
```

#### Event Loop Management
- **Sync API**: Uses `greenlet` for cooperative threading
- **Async API**: Native async/await with asyncio
- **Threading**: Each sync context runs in isolated greenlet

### 4. **Object Lifecycle Management**

#### Context Manager Pattern
```python
# Automatic cleanup
with sync_playwright() as p:
    with p.chromium.launch() as browser:
        with browser.new_context() as context:
            page = context.new_page()
            # Automatic cleanup on exit
```

#### Reference Management
```python
class PlaywrightImpl:
    def __init__(self):
        self._objects = {}  # Track browser objects by GUID
        
    def _on_close(self, guid: str):
        # Clean up Python references when objects close
        if guid in self._objects:
            del self._objects[guid]
```

## Key Features Implementation

### 1. **Auto-waiting and Assertions**
```python
# Auto-waiting built into actions
page.click("button")  # Waits for button to be actionable

# Web-first assertions
from playwright.sync_api import expect
expect(page.locator("h1")).to_have_text("Welcome")
```

**Implementation**:
- Auto-waiting implemented in Node.js Playwright core
- Python bindings inherit this behavior automatically
- Assertions use polling with configurable timeouts

### 2. **Locators System**
```python
# Locator pattern
locator = page.locator("button")
locator.click()
locator.fill("text")

# Chaining and filtering
page.locator("article").filter(has_text="News").click()
```

**Benefits**:
- Lazy evaluation until action
- Built-in retry logic
- Consistent element targeting

### 3. **Network Interception**
```python
def handle_route(route):
    if "api/data" in route.request.url:
        route.fulfill(json={"mock": "data"})
    else:
        route.continue_()

page.route("**/*", handle_route)
```

### 4. **Mobile and Device Emulation**
```python
# Device emulation
device = p.devices["iPhone 13"]
context = browser.new_context(**device)

# Custom viewport
context = browser.new_context(
    viewport={"width": 1280, "height": 720},
    user_agent="custom-agent"
)
```

## Installation and Setup

### 1. **Package Installation**
```bash
pip install playwright
```

### 2. **Browser Installation**
```bash
playwright install
# Or specific browsers
playwright install chromium firefox
```

### 3. **Dependencies**
- **Python**: 3.9+ (supports 3.9-3.13)
- **pyee**: Event emitter for Python
- **greenlet**: Cooperative threading for sync API
- **Node.js**: Bundled with browser installation

## Testing Integration

### 1. **Pytest Integration**
```python
# conftest.py
import pytest
from playwright.sync_api import sync_playwright

@pytest.fixture(scope="session")
def browser():
    with sync_playwright() as p:
        browser = p.chromium.launch()
        yield browser
        browser.close()

@pytest.fixture
def page(browser):
    context = browser.new_context()
    page = context.new_page()
    yield page
    context.close()

# test_example.py
def test_homepage(page):
    page.goto("https://example.com")
    assert page.title() == "Example"
```

### 2. **Async Testing**
```python
# pytest-asyncio integration
import pytest
from playwright.async_api import async_playwright

@pytest.mark.asyncio
async def test_async_example():
    async with async_playwright() as p:
        browser = await p.chromium.launch()
        page = await browser.new_page()
        await page.goto("https://example.com")
        title = await page.title()
        assert "Example" in title
        await browser.close()
```

### 3. **Test Configuration**
```python
# pytest.ini or pyproject.toml
[tool.pytest.ini_options]
addopts = "-v --tb=short"
asyncio_mode = "auto"  # Auto-detect async tests
markers = [
    "slow: marks tests as slow",
    "integration: marks tests as integration tests"
]
```

## Performance Considerations

### 1. **Sync vs Async Performance**
```python
# Async: Better for I/O-bound operations
async def parallel_pages():
    async with async_playwright() as p:
        browser = await p.chromium.launch()
        tasks = [
            create_page_and_navigate(browser, url) 
            for url in urls
        ]
        await asyncio.gather(*tasks)

# Sync: Simpler for sequential operations
def sequential_pages():
    with sync_playwright() as p:
        browser = p.chromium.launch()
        for url in urls:
            page = browser.new_page()
            page.goto(url)
```

### 2. **Context Reuse**
```python
# Efficient: Reuse browser context
with sync_playwright() as p:
    browser = p.chromium.launch()
    context = browser.new_context()
    
    for test_case in test_cases:
        page = context.new_page()
        run_test(page)
        page.close()  # Close page, keep context
```

### 3. **Memory Management**
```python
# Explicit cleanup for long-running scripts
browser = p.chromium.launch()
try:
    # Long-running automation
    for iteration in range(1000):
        context = browser.new_context()
        page = context.new_page()
        # Do work
        page.close()
        context.close()  # Important: clean up contexts
finally:
    browser.close()
```

## Advanced Features

### 1. **Tracing and Debugging**
```python
# Enable tracing
context = browser.new_context()
context.tracing.start(screenshots=True, snapshots=True)

# Run actions
page = context.new_page()
page.goto("https://example.com")
page.click("button")

# Save trace
context.tracing.stop(path="trace.zip")
```

### 2. **Video Recording**
```python
# Record video
context = browser.new_context(
    record_video_dir="videos/",
    record_video_size={"width": 1280, "height": 720}
)

page = context.new_page()
page.goto("https://example.com")
# Video automatically saved on context close
context.close()
```

### 3. **Screenshots and PDFs**
```python
# Screenshot
page.screenshot(path="screenshot.png")

# Element screenshot
page.locator("div.content").screenshot(path="element.png")

# PDF generation
page.pdf(path="page.pdf", format="A4")
```

### 4. **Authentication State**
```python
# Save authentication
context = browser.new_context()
page = context.new_page()
page.goto("https://example.com/login")
# Login process...
context.storage_state(path="auth.json")

# Reuse authentication
context = browser.new_context(storage_state="auth.json")
page = context.new_page()
page.goto("https://example.com/dashboard")  # Already logged in
```

## Error Handling Patterns

### 1. **Exception Hierarchy**
```python
from playwright.sync_api import PlaywrightError, TimeoutError

try:
    page.goto("https://example.com")
    page.wait_for_selector("h1", timeout=5000)
except TimeoutError:
    print("Page did not load in time")
except PlaywrightError as e:
    print(f"Playwright error: {e}")
```

### 2. **Async Error Handling**
```python
import asyncio
from playwright.async_api import async_playwright, PlaywrightError

async def safe_navigation(page, url):
    try:
        await page.goto(url)
        return True
    except PlaywrightError:
        return False
```

### 3. **Retry Patterns**
```python
from tenacity import retry, stop_after_attempt, wait_fixed

@retry(stop=stop_after_attempt(3), wait=wait_fixed(2))
def robust_click(page, selector):
    try:
        page.click(selector, timeout=5000)
    except TimeoutError:
        page.reload()
        raise  # Trigger retry
```

## Development and Debugging

### 1. **Debug Mode**
```python
# Run with debug flag
PWDEBUG=1 python test.py

# Or programmatically
browser = p.chromium.launch(headless=False, slow_mo=1000)
```

### 2. **Inspector Integration**
```python
# Pause for manual inspection
page.pause()  # Opens Playwright Inspector
```

### 3. **Console Monitoring**
```python
# Monitor console messages
def handle_console(msg):
    print(f"Console {msg.type}: {msg.text}")

page.on("console", handle_console)
page.goto("https://example.com")
```

## Best Practices

### 1. **Context Isolation**
```python
# Good: Isolated contexts for different scenarios
def test_user_workflow():
    context = browser.new_context()
    page = context.new_page()
    # Test user workflow
    context.close()

def test_admin_workflow():
    context = browser.new_context()  # Fresh context
    page = context.new_page()
    # Test admin workflow
    context.close()
```

### 2. **Resource Management**
```python
# Always use context managers
with sync_playwright() as p:
    with p.chromium.launch() as browser:
        context = browser.new_context()
        try:
            page = context.new_page()
            # Automation code
        finally:
            context.close()
```

### 3. **Locator Best Practices**
```python
# Prefer semantic locators
page.locator("role=button[name='Submit']")
page.get_by_role("button", name="Submit")

# Avoid fragile selectors
# Bad: page.locator("#btn-123-xyz")
# Good: page.get_by_test_id("submit-button")
```

### 4. **Async Best Practices**
```python
# Use async context managers
async with async_playwright() as p:
    async with p.chromium.launch() as browser:
        context = await browser.new_context()
        try:
            page = await context.new_page()
            # Async automation
        finally:
            await context.close()
```

## Integration with Python Ecosystem

### 1. **Data Science Integration**
```python
import pandas as pd
from playwright.sync_api import sync_playwright

def scrape_table_data():
    with sync_playwright() as p:
        page = p.chromium.launch().new_page()
        page.goto("https://example.com/data")
        
        # Extract table data
        table_data = page.evaluate("""
            Array.from(document.querySelectorAll('table tr')).map(row =>
                Array.from(row.querySelectorAll('td')).map(cell => cell.textContent)
            )
        """)
        
        return pd.DataFrame(table_data[1:], columns=table_data[0])
```

### 2. **Django/Flask Integration**
```python
# Django test case
from django.test import LiveServerTestCase
from playwright.sync_api import sync_playwright

class PlaywrightTestCase(LiveServerTestCase):
    @classmethod
    def setUpClass(cls):
        super().setUpClass()
        cls.playwright = sync_playwright().start()
        cls.browser = cls.playwright.chromium.launch()
    
    @classmethod 
    def tearDownClass(cls):
        cls.browser.close()
        cls.playwright.stop()
        super().tearDownClass()
    
    def test_homepage(self):
        page = self.browser.new_page()
        page.goto(self.live_server_url)
        self.assertIn("Welcome", page.content())
```

### 3. **CI/CD Integration**
```yaml
# GitHub Actions example
- name: Install dependencies
  run: |
    pip install playwright pytest
    playwright install

- name: Run tests
  run: pytest tests/
  env:
    PLAYWRIGHT_BROWSERS_PATH: ${{ github.workspace }}/browsers
```

This architecture provides Python developers with a powerful, pythonic way to automate browsers while leveraging the full capabilities of the Playwright engine through both synchronous and asynchronous interfaces.