# Playwright Go - Architecture Documentation

## Overview

Playwright for Go is a community-maintained library that provides Go language bindings for the Playwright browser automation framework. It enables Go developers to automate Chromium, Firefox, and WebKit browsers with a single, consistent API.

## Architecture Components

### Core Architecture

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   Go Application│    │  Playwright-Go  │    │ Node.js Runtime │
│                 │    │     Bridge       │    │   + Playwright  │
│  Your Go Code   │◄──►│                  │◄──►│                 │
│                 │    │ Channel/RPC API  │    │ Browser Control │
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
                       │   Go Structs     │
                       │ (Page, Browser,  │
                       │  Context, etc.)  │
                       └──────────────────┘
```

### Key Components

#### 1. **Channel Communication System** (`channel.go`, `connection.go`)
- **Purpose**: Manages communication between Go and Node.js Playwright runtime
- **Protocol**: JSON-RPC over stdio
- **Key Files**:
  - `channel.go`: Channel abstraction for object communication
  - `connection.go`: WebSocket/stdio connection management
  - `channel_owner.go`: Base class for all Playwright objects

#### 2. **Object Factory** (`objectFactory.go`)
- **Purpose**: Creates Go objects from JSON responses
- **Pattern**: Factory pattern to instantiate browser objects
- **Mapping**: Maps Playwright object types to Go structs

#### 3. **Type System** (`generated-*.go`)
- **Purpose**: Auto-generated Go types matching Playwright API
- **Files**:
  - `generated-enums.go`: Enumerations and constants
  - `generated-interfaces.go`: Interface definitions
  - `generated-structs.go`: Struct definitions
- **Generation**: Created via `scripts/generate-api.sh`

#### 4. **Core Browser Objects**
- **Browser Types** (`browser_type.go`): Launch and connect to browsers
- **Browser** (`browser.go`): Browser instance management
- **Browser Context** (`browser_context.go`): Isolated browser sessions
- **Page** (`page.go`): Individual web pages
- **Frame** (`frame.go`): Frame and iframe handling

#### 5. **Interaction APIs**
- **Locators** (`locator.go`): Element selection and interaction
- **Element Handles** (`element_handle.go`): Direct element references
- **Input** (`input.go`): Keyboard and mouse interactions
- **Network** (`network.go`): Request/response interception

## How It Works

### 1. **Initialization Process**
```go
// Start Playwright runtime
pw, err := playwright.Run()
if err != nil {
    log.Fatal(err)
}
defer pw.Stop()

// Launch browser
browser, err := pw.Chromium.Launch()
if err != nil {
    log.Fatal(err)
}
defer browser.Close()
```

**Internal Flow**:
1. `playwright.Run()` starts Node.js Playwright process
2. Establishes stdio communication channel
3. Creates connection and initializes object factory
4. Returns `Playwright` instance with browser types

### 2. **Object Lifecycle Management**
```go
// Create browser context (isolated session)
context, err := browser.NewContext()

// Create page within context
page, err := context.NewPage()

// Navigate and interact
err = page.Goto("https://example.com")
locator := page.Locator("button")
err = locator.Click()
```

**Internal Flow**:
1. Go method calls are serialized to JSON-RPC
2. Sent to Node.js Playwright runtime via stdio
3. Node.js executes browser operations
4. Results returned via JSON-RPC to Go
5. Go objects updated with response data

### 3. **Error Handling Pattern**
```go
// Go idiomatic error handling
result, err := page.Evaluate("document.title")
if err != nil {
    // Handle error
    return err
}
// Use result
```

### 4. **Type Safety and Marshaling**
- All browser responses are unmarshaled to strongly-typed Go structs
- Options are passed as Go structs with optional fields
- Enums are represented as Go constants

## Key Features Implementation

### 1. **Auto-waiting**
```go
// Automatically waits for element to be actionable
err := page.Locator("button").Click()
```
- Implemented in Node.js Playwright runtime
- Go bindings inherit this behavior automatically

### 2. **Assertions** (`assertions.go`, `locator_assertions.go`)
```go
// Web-first assertions with auto-retry
err := expect.Locator(page.Locator("h1")).ToHaveText("Welcome")
```

### 3. **Network Interception** (`route.go`)
```go
// Route and modify network requests
err := page.Route("**/*.css", func(route playwright.Route) {
    route.Abort()
})
```

### 4. **Mobile Emulation**
```go
// Launch with mobile device settings
device := playwright.Devices["iPhone 13"]
context, err := browser.NewContext(playwright.BrowserNewContextOptions{
    Viewport: device.Viewport,
    UserAgent: &device.UserAgent,
    IsMobile: &device.IsMobile,
})
```

## Installation and Setup

### 1. **Package Installation**
```bash
go get -u github.com/playwright-community/playwright-go
```

### 2. **Browser Installation**
```bash
go run github.com/playwright-community/playwright-go/cmd/playwright@v0.xxxx.x install --with-deps
```

### 3. **Runtime Requirements**
- Go 1.22+
- Node.js runtime (bundled with installation)
- Browser binaries (Chromium, Firefox, WebKit)

## Testing Integration

### 1. **With Go Testing Framework**
```go
func TestExample(t *testing.T) {
    pw, err := playwright.Run()
    require.NoError(t, err)
    defer pw.Stop()
    
    browser, err := pw.Chromium.Launch()
    require.NoError(t, err)
    defer browser.Close()
    
    page, err := browser.NewPage()
    require.NoError(t, err)
    
    _, err = page.Goto("https://example.com")
    require.NoError(t, err)
    
    title, err := page.Title()
    require.NoError(t, err)
    assert.Contains(t, title, "Example")
}
```

### 2. **Helper Utilities** (`helpers.go`)
- Device descriptors for mobile testing
- Common configuration patterns
- Utility functions for common tasks

## Performance Considerations

### 1. **Browser Process Management**
- Each `playwright.Run()` starts a new Node.js process
- Browsers are shared across contexts within same Playwright instance
- Contexts provide isolation without full browser restart

### 2. **Memory Management**
- Go objects are garbage collected normally
- Browser resources cleaned up via `defer` pattern
- Node.js process cleaned up on `pw.Stop()`

### 3. **Concurrency**
```go
// Safe for concurrent use with separate contexts
func TestConcurrent(t *testing.T) {
    pw, _ := playwright.Run()
    defer pw.Stop()
    
    browser, _ := pw.Chromium.Launch()
    defer browser.Close()
    
    var wg sync.WaitGroup
    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            context, _ := browser.NewContext()
            defer context.Close()
            
            page, _ := context.NewPage()
            page.Goto("https://example.com")
            // Parallel execution
        }()
    }
    wg.Wait()
}
```

## Debugging and Development

### 1. **Environment Variables**
- `DEBUG=pw:api`: Enable API debug logging
- `PLAYWRIGHT_BROWSERS_PATH`: Custom browser installation path

### 2. **Tracing Support** (`tracing.go`)
```go
// Start tracing
err := context.Tracing().Start(playwright.TracingStartOptions{
    Screenshots: playwright.Bool(true),
    Snapshots:   playwright.Bool(true),
})

// Stop and save trace
err = context.Tracing().Stop(playwright.TracingStopOptions{
    Path: playwright.String("trace.zip"),
})
```

### 3. **Screenshots and Videos**
```go
// Screenshot
err := page.Screenshot(playwright.PageScreenshotOptions{
    Path: playwright.String("screenshot.png"),
})

// Video recording (via context options)
context, err := browser.NewContext(playwright.BrowserNewContextOptions{
    RecordVideo: &playwright.RecordVideoOptions{
        Dir: playwright.String("videos/"),
    },
})
```

## Community and Maintenance

### Project Status
- **Community maintained**: Looking for maintainers (as of repository status)
- **Follows upstream**: Tracks Playwright Node.js releases
- **Go idioms**: Designed to feel natural for Go developers

### Contributing
- API generation via patches to upstream Playwright
- Test coverage in `tests/` directory
- Examples in `examples/` directory

### Version Compatibility
- Each minor Playwright version requires matching Go binding version
- Browser versions are locked to specific Playwright releases
- Check `go.mod` for current Playwright dependency version

## Best Practices

### 1. **Resource Management**
```go
// Always use defer for cleanup
pw, err := playwright.Run()
if err != nil {
    return err
}
defer pw.Stop()

browser, err := pw.Chromium.Launch()
if err != nil {
    return err
}
defer browser.Close()
```

### 2. **Error Handling**
```go
// Check all errors
page, err := browser.NewPage()
if err != nil {
    return fmt.Errorf("failed to create page: %w", err)
}

if _, err := page.Goto(url); err != nil {
    return fmt.Errorf("failed to navigate to %s: %w", url, err)
}
```

### 3. **Context Isolation**
```go
// Use separate contexts for test isolation
func TestScenario1(t *testing.T) {
    context, _ := browser.NewContext()
    defer context.Close()
    // Test code
}

func TestScenario2(t *testing.T) {
    context, _ := browser.NewContext() // Fresh context
    defer context.Close()
    // Test code
}
```

This architecture provides Go developers with a powerful, type-safe way to automate browsers while maintaining the performance and reliability of the underlying Playwright engine.