# Playwright MCP - Architecture Documentation

## Overview

Playwright MCP (Model Context Protocol) is a server that provides browser automation capabilities specifically designed for AI/LLM integration. It enables AI agents to interact with web browsers through structured accessibility snapshots rather than visual screenshots, making it ideal for AI-powered automation and testing.

## Architecture Components

### High-Level Architecture

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   AI Client     │    │  Playwright MCP  │    │ Playwright Core │
│ (Claude, GPT,   │    │     Server       │    │   (Node.js)     │
│  VSCode, etc.)  │◄──►│                  │◄──►│                 │
│                 │    │ MCP Tools/API    │    │ Browser Control │
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
                       │ Accessibility    │
                       │   Snapshots      │
                       │ (Structured Data)│
                       └──────────────────┘
```

### Core Components

#### 1. **MCP Server Framework** (`src/server.ts`, `src/connection.ts`)
- **Purpose**: Implements Model Context Protocol specification
- **Transport**: SSE (Server-Sent Events) or stdio
- **Communication**: JSON-RPC with AI clients
- **Key Features**:
  - Tool registration and execution
  - Resource management
  - Error handling and validation

#### 2. **Browser Context Factory** (`src/browserContextFactory.ts`)
- **Purpose**: Manages browser lifecycle and context creation
- **Modes**:
  - **Persistent**: Saves user data between sessions
  - **Isolated**: Fresh context for each session
- **Configuration**: Supports both programmatic and file-based config

#### 3. **Tools System** (`src/tools.ts`)
- **Purpose**: Exposes browser automation as MCP tools
- **Tool Categories**:
  - Interactions (click, type, drag)
  - Navigation (go to URL, back/forward)
  - Resources (screenshots, PDFs, network logs)
  - Utilities (wait, resize, close)
  - Tabs management
  - Testing utilities

#### 4. **Page Snapshot System** (`src/pageSnapshot.ts`)
- **Purpose**: Generates accessibility-based page representations
- **Format**: Structured text with element references
- **Benefits**: No visual processing needed, deterministic element targeting

#### 5. **Configuration System** (`src/config.ts`)
- **Purpose**: Unified configuration management
- **Sources**: CLI arguments, JSON files, programmatic options
- **Features**: Validation, defaults, environment-specific settings

## How It Works

### 1. **Server Initialization**
```bash
npx @playwright/mcp@latest --browser chrome --headless
```

**Internal Flow**:
1. Parse CLI arguments and load configuration
2. Initialize MCP server with specified transport
3. Create browser context factory
4. Register all available tools
5. Start listening for AI client connections

### 2. **AI Client Integration**
```json
{
  "mcpServers": {
    "playwright": {
      "command": "npx",
      "args": ["@playwright/mcp@latest"]
    }
  }
}
```

**Connection Flow**:
1. AI client starts MCP server via stdio
2. Handshake establishes protocol version
3. Client discovers available tools
4. Tools are exposed in AI interface

### 3. **Tool Execution Flow**
```
AI Request → MCP Server → Playwright API → Browser → Response → AI Client
```

**Example - Click Tool**:
1. AI decides to click an element
2. Calls `browser_click` tool with element description and reference
3. MCP server validates parameters
4. Executes `page.locator(ref).click()` via Playwright
5. Returns success/failure to AI client

### 4. **Accessibility Snapshot Generation**
```typescript
// Simplified snapshot generation
const snapshot = await page.locator('html').ariaSnapshot();
return {
  content: snapshot,
  format: 'text/plain'
};
```

**Snapshot Structure**:
```
button "Submit Form" [ref="button-123"]
textbox "Email Address" [ref="input-456"] 
heading "Welcome" level=1 [ref="h1-789"]
```

## Tool Categories Deep Dive

### 1. **Interaction Tools**
- **browser_snapshot**: Capture page accessibility tree
- **browser_click**: Click elements using references
- **browser_type**: Type text into form fields
- **browser_drag**: Drag and drop operations
- **browser_hover**: Hover over elements
- **browser_select_option**: Select dropdown options
- **browser_press_key**: Keyboard input

### 2. **Navigation Tools**
- **browser_navigate**: Go to specific URLs
- **browser_navigate_back**: Browser back button
- **browser_navigate_forward**: Browser forward button

### 3. **Resource Tools**
- **browser_take_screenshot**: Capture visual screenshots
- **browser_pdf_save**: Generate PDF documents
- **browser_network_requests**: Monitor network activity
- **browser_console_messages**: Access console logs

### 4. **Utility Tools**
- **browser_wait_for**: Wait for conditions
- **browser_resize**: Change viewport size
- **browser_close**: Close browser
- **browser_install**: Install browsers
- **browser_file_upload**: Handle file uploads
- **browser_handle_dialog**: Manage browser dialogs

### 5. **Tab Management Tools**
- **browser_tab_list**: List open tabs
- **browser_tab_new**: Open new tabs
- **browser_tab_select**: Switch between tabs
- **browser_tab_close**: Close specific tabs

### 6. **Testing Tools**
- **browser_generate_playwright_test**: Generate test code

### 7. **Vision Mode Tools** (Optional)
When `--vision` flag is used:
- **browser_screen_capture**: Take screenshots
- **browser_screen_click**: Click using X,Y coordinates
- **browser_screen_move_mouse**: Move mouse cursor
- **browser_screen_drag**: Drag using coordinates
- **browser_screen_type**: Type text (no element targeting)

## Configuration Architecture

### 1. **CLI Configuration**
```bash
npx @playwright/mcp@latest \
  --browser chrome \
  --headless \
  --device "iPhone 15" \
  --viewport-size "1280,720" \
  --user-data-dir ./profile
```

### 2. **JSON Configuration File**
```json
{
  "browser": {
    "browserName": "chromium",
    "isolated": false,
    "launchOptions": {
      "headless": true,
      "executablePath": "/path/to/browser"
    },
    "contextOptions": {
      "viewport": { "width": 1280, "height": 720 },
      "userAgent": "custom-agent"
    }
  },
  "capabilities": ["core", "tabs", "pdf"],
  "vision": false,
  "outputDir": "./output"
}
```

### 3. **Programmatic Configuration**
```typescript
import { createConnection } from '@playwright/mcp';

const connection = await createConnection({
  browser: {
    browserName: 'chromium',
    launchOptions: { headless: true }
  }
});
```

## Browser Context Management

### 1. **Persistent Mode** (Default)
```
User Profile Location:
- Windows: %USERPROFILE%\AppData\Local\ms-playwright\mcp-{channel}-profile
- macOS: ~/Library/Caches/ms-playwright/mcp-{channel}-profile  
- Linux: ~/.cache/ms-playwright/mcp-{channel}-profile
```

**Benefits**:
- Login state preserved
- Extensions and settings maintained
- Realistic user behavior simulation

### 2. **Isolated Mode**
```bash
npx @playwright/mcp@latest --isolated --storage-state auth.json
```

**Benefits**:
- Clean state for each session
- Controlled initial conditions
- Better for testing scenarios

## Transport Mechanisms

### 1. **Stdio Transport** (Default)
- Used when launched by AI clients
- Direct process communication
- Automatic process lifecycle management

### 2. **SSE Transport** (Server Mode)
```bash
npx @playwright/mcp@latest --port 8931
```

**Client Configuration**:
```json
{
  "mcpServers": {
    "playwright": {
      "url": "http://localhost:8931/sse"
    }
  }
}
```

**Use Cases**:
- Remote server deployment
- Multiple client connections
- Environment without direct process control

## AI Integration Patterns

### 1. **Accessibility-First Approach**
```
AI sees: button "Sign In" [ref="btn-signin"]
AI calls: browser_click(element="Sign In button", ref="btn-signin")
Result: Element clicked successfully
```

### 2. **Vision Mode (Optional)**
```
AI sees: Screenshot with X,Y coordinates
AI calls: browser_screen_click(x=150, y=200)
Result: Click performed at coordinates
```

### 3. **Multi-Step Workflows**
```
1. browser_snapshot() → Get page structure
2. browser_click(ref="login-btn") → Click login
3. browser_type(ref="email", text="user@example.com") → Fill email
4. browser_type(ref="password", text="secret") → Fill password
5. browser_click(ref="submit") → Submit form
6. browser_wait_for(text="Dashboard") → Wait for redirect
```

## Performance Considerations

### 1. **Snapshot Generation**
- Accessibility snapshots are faster than screenshots
- Structured data reduces AI processing time
- Deterministic element references improve reliability

### 2. **Browser Lifecycle**
- Persistent mode: Fast startup, shared state
- Isolated mode: Clean state, slower startup
- Context reuse within sessions

### 3. **Network Optimization**
```bash
# Block unnecessary resources
npx @playwright/mcp@latest \
  --blocked-origins "ads.google.com;analytics.com"
```

## Security and Permissions

### 1. **Origin Filtering**
```bash
# Allow only specific domains
npx @playwright/mcp@latest \
  --allowed-origins "example.com;trusted-site.com"
```

### 2. **Capability Restrictions**
```bash
# Limit available tools
npx @playwright/mcp@latest \
  --caps "core,tabs" # Only core and tab tools
```

### 3. **Sandbox Settings**
```bash
# Disable sandbox (when needed)
npx @playwright/mcp@latest --no-sandbox
```

## Debugging and Development

### 1. **Trace Collection**
```bash
npx @playwright/mcp@latest --save-trace
```
- Saves Playwright traces to output directory
- Useful for debugging failed automations
- Can be viewed in Playwright trace viewer

### 2. **Debug Logging**
```bash
DEBUG=mcp:* npx @playwright/mcp@latest
```

### 3. **Output Directory**
```bash
npx @playwright/mcp@latest --output-dir ./debug-output
```
- Screenshots saved with timestamps
- PDFs and traces collected
- Network logs and console messages

## Docker Deployment

### 1. **Docker Image**
```bash
docker run -i --rm mcr.microsoft.com/playwright/mcp
```

**Limitations**:
- Headless Chromium only
- No persistent profile support
- Suitable for CI/CD environments

### 2. **Custom Docker Build**
```dockerfile
FROM mcr.microsoft.com/playwright:latest
COPY config.json /app/config.json
WORKDIR /app
CMD ["npx", "@playwright/mcp@latest", "--config", "config.json"]
```

## Integration Examples

### 1. **VS Code/Cursor Integration**
```json
{
  "mcpServers": {
    "playwright": {
      "command": "npx",
      "args": ["@playwright/mcp@latest", "--headless"]
    }
  }
}
```

### 2. **Claude Desktop Integration**
```json
{
  "mcpServers": {
    "playwright": {
      "command": "npx", 
      "args": [
        "@playwright/mcp@latest",
        "--isolated",
        "--save-trace"
      ]
    }
  }
}
```

### 3. **Custom AI Application**
```typescript
import { createConnection } from '@playwright/mcp';
import { SSEServerTransport } from '@modelcontextprotocol/sdk/server/sse.js';

const connection = await createConnection({
  browser: { launchOptions: { headless: true } }
});

const transport = new SSEServerTransport('/messages', response);
await connection.server.connect(transport);
```

## Best Practices

### 1. **Element References**
- Always use exact `ref` values from snapshots
- Include human-readable descriptions for AI context
- Validate element existence before interactions

### 2. **Error Handling**
- Implement retry logic for transient failures
- Use wait conditions for dynamic content
- Handle dialog boxes and popups

### 3. **Resource Management**
- Close tabs when no longer needed
- Use isolated mode for testing
- Configure appropriate timeouts

### 4. **AI Prompt Engineering**
- Provide clear tool descriptions
- Use structured workflows
- Validate AI understanding with snapshots

This architecture enables powerful AI-driven browser automation while maintaining the reliability and performance characteristics of Playwright, specifically optimized for AI agent integration.