# Playwright TypeScript/JavaScript - Architecture Documentation

## Overview

Playwright for TypeScript/JavaScript is the original and core implementation of the Playwright browser automation framework. It serves as both a standalone automation library and the foundation for all other language bindings. This is the most feature-complete implementation with built-in test runner, advanced debugging tools, and comprehensive testing capabilities.

## Architecture Components

### High-Level Monorepo Architecture

```
playwright/
├── packages/                    # Monorepo packages
│   ├── playwright-core/        # Core browser automation
│   ├── playwright/             # Main package with browser install
│   ├── @playwright/test/       # Test runner and utilities
│   ├── playwright-ct-*/        # Component testing packages
│   ├── html-reporter/          # HTML test reporter
│   ├── trace-viewer/           # Trace visualization
│   └── web/                   # Web UI components
├── src/                        # Source code
│   ├── client/                # Client-side API
│   ├── server/                # Browser server implementation
│   ├── protocol/              # Browser communication protocol
│   ├── test/                  # Test runner implementation
│   └── utils/                 # Shared utilities
├── tests/                      # Comprehensive test suite
├── browser_patches/            # Browser engine patches
└── utils/                     # Build and development tools
```

### Core Package Architecture

#### 1. **playwright-core** (Foundation)
```
playwright-core/
├── lib/
│   ├── client/               # Browser API implementation
│   │   ├── browser.ts       # Browser management
│   │   ├── page.ts          # Page automation
│   │   ├── locator.ts       # Element location/interaction
│   │   └── channelOwner.ts  # RPC communication base
│   ├── server/              # Browser server
│   │   ├── chromium/        # Chromium-specific implementation
│   │   ├── firefox/         # Firefox-specific implementation
│   │   ├── webkit/          # WebKit-specific implementation
│   │   ├── browserServer.ts # Browser process management
│   │   └── playwright.ts    # Main orchestrator
│   ├── protocol/            # Browser communication
│   │   ├── channels.ts      # RPC channel definitions
│   │   └── serializers.ts   # Data serialization
│   └── utils/               # Shared utilities
└── types/                   # TypeScript definitions
```

#### 2. **@playwright/test** (Test Runner)
```
@playwright/test/
├── lib/
│   ├── test/                # Test framework core
│   │   ├── testRunner.ts    # Test execution engine
│   │   ├── fixtures.ts      # Test fixtures system
│   │   ├── expect.ts        # Assertion library
│   │   └── config.ts        # Configuration management
│   ├── reporters/           # Test reporters
│   │   ├── html.ts          # HTML reporter
│   │   ├── json.ts          # JSON reporter
│   │   └── junit.ts         # JUnit reporter
│   └── transform/           # Code transformation
│       ├── babelBundle.ts   # Babel integration
│       └── esmLoader.ts     # ES module loading
└── index.ts                 # Public API exports
```

## Communication Architecture

### 1. **Multi-Process Design**
```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│ Node.js Process │    │ Browser Process  │    │ Renderer Process│
│                 │    │                  │    │                 │
│ Playwright API  │◄──►│ Browser Server   │◄──►│ Web Content     │
│ Test Runner     │    │ (CDP/WS Protocol)│    │ JavaScript      │
│ User Code       │    │                  │    │ DOM/Events      │
└─────────────────┘    └──────────────────┘    └─────────────────┘
        │                       │
        │                       ▼
        │              ┌─────────────────┐
        │              │ Browser Patches │
        │              │ Custom Protocol │
        │              │ Features        │
        │              └─────────────────┘
        ▼
┌─────────────────┐
│ RPC Channel     │
│ Communication   │
│ (JSON Protocol) │
└─────────────────┘
```

### 2. **Channel Communication System**
```typescript
// Client-Server RPC Architecture
class ChannelOwner {
  _connection: Connection;
  _guid: string;
  
  async _sendMessage(method: string, params: any): Promise<any> {
    return this._connection.sendMessageToServer({
      guid: this._guid,
      method,
      params
    });
  }
}

// Example: Page navigation
class Page extends ChannelOwner {
  async goto(url: string): Promise<Response> {
    const result = await this._sendMessage('goto', { url });
    return result.response;
  }
}
```

## Key Subsystems Deep Dive

### 1. **Browser Management System**

#### Browser Server (`src/server/`)
```typescript
// Browser launcher abstraction
class BrowserType {
  async launch(options: LaunchOptions): Promise<Browser> {
    const server = new BrowserServer(this._name, options);
    await server.start();
    return new Browser(server);
  }
}

// Browser-specific implementations
class ChromiumBrowser extends BrowserTypeImpl {
  // Chromium-specific launch logic
}

class FirefoxBrowser extends BrowserTypeImpl {
  // Firefox-specific launch logic  
}

class WebKitBrowser extends BrowserTypeImpl {
  // WebKit-specific launch logic
}
```

#### Browser Context Isolation
```typescript
class BrowserContext extends ChannelOwner {
  _pages = new Set<Page>();
  _routes: Route[] = [];
  
  async newPage(): Promise<Page> {
    // Creates isolated page within context
    const page = await this._sendMessage('newPage');
    this._pages.add(page);
    return page;
  }
  
  async route(url: string, handler: RouteHandler): Promise<void> {
    // Network interception setup
    this._routes.push(new Route(url, handler));
  }
}
```

### 2. **Locator System**

#### Smart Element Selection
```typescript
class Locator {
  _selector: string;
  _page: Page;
  
  // Auto-waiting click
  async click(options?: ClickOptions): Promise<void> {
    await this._page._sendMessage('locatorClick', {
      selector: this._selector,
      options
    });
  }
  
  // Filtering and chaining
  filter(options: { hasText?: string }): Locator {
    return new Locator(this._page, `${this._selector} >> text=${options.hasText}`);
  }
  
  // Element state checks
  async isVisible(): Promise<boolean> {
    return this._evaluate(element => element.offsetParent !== null);
  }
}
```

#### Selector Engine
```typescript
// Built-in selector engines
const selectorEngines = {
  css: new CSSEngine(),
  xpath: new XPathEngine(), 
  text: new TextEngine(),
  'data-testid': new TestIdEngine(),
  role: new AriaRoleEngine()
};

// Custom selector registration
playwright.selectors.register('vue', vueEngine);
```

### 3. **Test Runner Architecture**

#### Test Discovery and Execution
```typescript
class TestRunner {
  async run(config: FullConfig): Promise<void> {
    // 1. Discover test files
    const testFiles = await this._collectTestFiles(config);
    
    // 2. Load and parse tests
    const testSuite = await this._loadTests(testFiles);
    
    // 3. Execute with parallelization
    const results = await this._runTestSuite(testSuite, config);
    
    // 4. Generate reports
    await this._generateReports(results, config);
  }
}
```

#### Fixture System
```typescript
// Built-in fixtures
const test = base.extend<{ page: Page; browser: Browser }>({
  browser: async ({}, use) => {
    const browser = await playwright.chromium.launch();
    await use(browser);
    await browser.close();
  },
  
  page: async ({ browser }, use) => {
    const page = await browser.newPage();
    await use(page);
    await page.close();
  }
});

// Custom fixtures
const test = base.extend<{ todoApp: TodoApp }>({
  todoApp: async ({ page }, use) => {
    const app = new TodoApp(page);
    await app.goto();
    await use(app);
  }
});
```

#### Parallel Execution
```typescript
class Dispatcher {
  async run(): Promise<void> {
    const workers = Array.from({ length: this._config.workers }, 
      () => new Worker(this._config));
    
    const testQueue = [...this._testSuite.tests()];
    
    await Promise.all(workers.map(worker => 
      this._runWorker(worker, testQueue)));
  }
}
```

### 4. **Assertion System**

#### Web-First Assertions
```typescript
class Expect {
  async toHaveText(expected: string): Promise<void> {
    await this._locator.waitFor({ state: 'attached' });
    
    // Retry assertion with polling
    await this._retryAssertion(async () => {
      const text = await this._locator.textContent();
      if (text !== expected) {
        throw new AssertionError(`Expected "${expected}", got "${text}"`);
      }
    });
  }
  
  async toBeVisible(): Promise<void> {
    await this._retryAssertion(async () => {
      const visible = await this._locator.isVisible();
      if (!visible) {
        throw new AssertionError('Element is not visible');
      }
    });
  }
}

// Usage
await expect(page.locator('h1')).toHaveText('Welcome');
await expect(page.locator('.loading')).toBeHidden();
```

### 5. **Network Interception**

#### Route Management
```typescript
class Route {
  constructor(
    private _url: string | RegExp,
    private _handler: RouteHandler
  ) {}
  
  async fulfill(response: FulfillOptions): Promise<void> {
    await this._page._sendMessage('routeFulfill', {
      url: this._url,
      response
    });
  }
  
  async abort(errorCode?: string): Promise<void> {
    await this._page._sendMessage('routeAbort', {
      url: this._url,
      errorCode
    });
  }
}

// Usage patterns
await page.route('**/api/data', route => {
  route.fulfill({ json: { mock: 'data' } });
});

await page.route(/\.(png|jpg)$/, route => route.abort());
```

## Advanced Features

### 1. **Tracing System**

#### Trace Collection
```typescript
class Tracing {
  async start(options: TracingOptions): Promise<void> {
    await this._context._sendMessage('tracingStart', {
      screenshots: options.screenshots,
      snapshots: options.snapshots,
      sources: options.sources
    });
  }
  
  async stop(options?: { path?: string }): Promise<void> {
    const traceData = await this._context._sendMessage('tracingStop');
    if (options?.path) {
      await fs.writeFile(options.path, traceData);
    }
  }
}

// Usage in tests
test('user flow', async ({ context, page }) => {
  await context.tracing.start({ screenshots: true, snapshots: true });
  
  await page.goto('/login');
  await page.fill('#email', 'user@example.com');
  await page.click('#submit');
  
  await context.tracing.stop({ path: 'trace.zip' });
});
```

#### Trace Viewer Integration
```typescript
// Trace analysis
class TraceViewer {
  static async show(tracePath: string): Promise<void> {
    const server = new TraceViewerServer();
    await server.start();
    
    // Open browser to trace viewer
    const browser = await chromium.launch({ headless: false });
    const page = await browser.newPage();
    await page.goto(`http://localhost:${server.port}?trace=${tracePath}`);
  }
}
```

### 2. **Component Testing**

#### Component Test Packages
```typescript
// React component testing
import { test, expect } from '@playwright/experimental-ct-react';
import { Button } from './Button';

test('button click', async ({ mount }) => {
  const component = await mount(
    <Button onClick={() => console.log('clicked')}>
      Click me
    </Button>
  );
  
  await component.click();
  await expect(component).toHaveText('Click me');
});
```

#### Multiple Framework Support
- `@playwright/experimental-ct-react`
- `@playwright/experimental-ct-vue`
- `@playwright/experimental-ct-svelte`
- `@playwright/experimental-ct-solid`

### 3. **Visual Testing**

#### Screenshot Comparison
```typescript
test('visual regression', async ({ page }) => {
  await page.goto('/dashboard');
  
  // Full page screenshot
  await expect(page).toHaveScreenshot('dashboard.png');
  
  // Element screenshot
  await expect(page.locator('.chart')).toHaveScreenshot('chart.png');
  
  // Cross-browser comparison
  await expect(page).toHaveScreenshot('dashboard.png', { 
    threshold: 0.2,
    maxDiffPixels: 100 
  });
});
```

#### Image Comparison Engine
```typescript
class ImageComparator {
  compare(
    expected: Buffer,
    actual: Buffer,
    options: CompareOptions
  ): CompareResult {
    // Pixel-level comparison
    const diff = this._pixelDiff(expected, actual);
    
    if (diff.percentage > options.threshold) {
      return {
        pass: false,
        diff: diff.buffer,
        percentage: diff.percentage
      };
    }
    
    return { pass: true };
  }
}
```

### 4. **Reporter System**

#### Built-in Reporters
```typescript
// HTML Reporter
class HtmlReporter implements Reporter {
  async onEnd(result: FullResult): Promise<void> {
    const report = this._generateHtmlReport(result);
    await fs.writeFile('playwright-report/index.html', report);
    
    // Auto-open browser
    if (this._config.open) {
      await this._openReport();
    }
  }
}

// Custom Reporter
class SlackReporter implements Reporter {
  async onTestEnd(test: TestCase, result: TestResult): Promise<void> {
    if (result.status === 'failed') {
      await this._sendSlackNotification(test, result);
    }
  }
}
```

## Configuration Architecture

### 1. **Layered Configuration System**
```typescript
// playwright.config.ts
export default defineConfig({
  // Global settings
  timeout: 30000,
  retries: 2,
  workers: process.env.CI ? 2 : undefined,
  
  // Browser configuration
  use: {
    browserName: 'chromium',
    headless: !process.env.DEBUG,
    viewport: { width: 1280, height: 720 },
    screenshot: 'only-on-failure',
    trace: 'retain-on-failure'
  },
  
  // Project-specific settings
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] }
    },
    {
      name: 'firefox', 
      use: { ...devices['Desktop Firefox'] }
    },
    {
      name: 'mobile',
      use: { ...devices['iPhone 13'] }
    }
  ],
  
  // Test organization
  testDir: './tests',
  testMatch: '**/*.spec.ts',
  outputDir: 'test-results',
  
  // Reporting
  reporter: [
    ['html'],
    ['json', { outputFile: 'results.json' }],
    ['junit', { outputFile: 'results.xml' }]
  ]
});
```

### 2. **Environment-Specific Configs**
```typescript
// config/staging.config.ts
export default defineConfig({
  ...baseConfig,
  use: {
    ...baseConfig.use,
    baseURL: 'https://staging.example.com'
  },
  projects: [
    {
      name: 'staging-smoke',
      testMatch: '**/smoke/*.spec.ts'
    }
  ]
});
```

## Browser Patches Architecture

### 1. **Custom Protocol Extensions**
```
browser_patches/
├── chromium/
│   ├── patches/          # Chromium engine patches
│   ├── protocol/         # Custom CDP extensions
│   └── browser_protocol/ # Browser-specific features
├── firefox/
│   ├── patches/          # Firefox engine patches
│   └── protocol/         # Custom debugging protocol
└── webkit/
    ├── patches/          # WebKit engine patches
    └── protocol/         # Custom protocol
```

### 2. **Enhanced Browser Features**
- **Accessibility**: Enhanced accessibility tree access
- **Network**: Advanced network interception
- **Debugging**: Extended debugging capabilities
- **Performance**: Performance monitoring APIs
- **Security**: Enhanced security context control

## Development and Build System

### 1. **Build Pipeline**
```typescript
// utils/build/build.js
class BuildSystem {
  async build(): Promise<void> {
    // 1. TypeScript compilation
    await this._compileTypeScript();
    
    // 2. Bundle client libraries
    await this._bundleClient();
    
    // 3. Generate protocol definitions
    await this._generateProtocol();
    
    // 4. Package distribution
    await this._packageDistribution();
  }
}
```

### 2. **Code Generation**
```typescript
// utils/generate_types/
class TypeGenerator {
  generateApiTypes(): void {
    // Generate TypeScript definitions from protocol
    const protocol = this._loadProtocol();
    const types = this._generateTypes(protocol);
    this._writeTypeDefinitions(types);
  }
}
```

### 3. **Testing Infrastructure**
```typescript
// Comprehensive test suite
tests/
├── library/              # Core library tests
├── page/                 # Page API tests  
├── playwright-test/      # Test runner tests
├── installation/         # Installation tests
├── stress/              # Performance tests
├── electron/            # Electron integration
└── android/             # Android testing
```

## Performance Optimizations

### 1. **Lazy Loading**
```typescript
class PlaywrightClient {
  private _browserTypes?: Map<string, BrowserType>;
  
  get chromium(): BrowserType {
    if (!this._browserTypes) {
      this._initializeBrowserTypes();
    }
    return this._browserTypes.get('chromium')!;
  }
}
```

### 2. **Connection Pooling**
```typescript
class ConnectionPool {
  private _connections = new Map<string, Connection>();
  
  async getConnection(endpoint: string): Promise<Connection> {
    if (!this._connections.has(endpoint)) {
      const connection = await this._createConnection(endpoint);
      this._connections.set(endpoint, connection);
    }
    return this._connections.get(endpoint)!;
  }
}
```

### 3. **Resource Management**
```typescript
class ResourceManager {
  private _cleanup: (() => Promise<void>)[] = [];
  
  register(cleanup: () => Promise<void>): void {
    this._cleanup.push(cleanup);
  }
  
  async dispose(): Promise<void> {
    await Promise.all(this._cleanup.map(fn => fn()));
    this._cleanup.length = 0;
  }
}
```

## Best Practices and Patterns

### 1. **Page Object Model**
```typescript
class LoginPage {
  constructor(private page: Page) {}
  
  private emailInput = this.page.locator('#email');
  private passwordInput = this.page.locator('#password');
  private submitButton = this.page.locator('#submit');
  
  async login(email: string, password: string): Promise<void> {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.submitButton.click();
  }
  
  async expectLoginError(): Promise<void> {
    await expect(this.page.locator('.error')).toBeVisible();
  }
}
```

### 2. **Custom Fixtures**
```typescript
// test/fixtures.ts
export const test = base.extend<{
  loginPage: LoginPage;
  authenticatedPage: Page;
}>({
  loginPage: async ({ page }, use) => {
    const loginPage = new LoginPage(page);
    await use(loginPage);
  },
  
  authenticatedPage: async ({ page, loginPage }, use) => {
    await loginPage.login('user@example.com', 'password');
    await use(page);
  }
});
```

### 3. **Test Organization**
```typescript
// tests/auth/login.spec.ts
test.describe('Login Flow', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/login');
  });
  
  test('successful login', async ({ loginPage }) => {
    await loginPage.login('valid@email.com', 'password');
    await expect(page).toHaveURL('/dashboard');
  });
  
  test('failed login', async ({ loginPage }) => {
    await loginPage.login('invalid@email.com', 'wrongpassword');
    await loginPage.expectLoginError();
  });
});
```

This architecture provides a comprehensive, scalable foundation for browser automation and testing, serving as both a powerful standalone tool and the backbone for all other Playwright language implementations.