# Approach 3: Hybrid Library + CLI - Implementation Plan

## Overview

**Goal**: Build a TypeScript library first for programmatic use, then add a CLI layer on top.

**Timeline**: 8-10 weeks (350-450 hours)

**Outcome**: A reusable TypeScript/JavaScript library that can be used programmatically in Node.js applications, with a powerful CLI built on top of it. The library should also work in browser environments where applicable.

**When to Choose This Approach**:
- You want to use LLM functionality in your own applications
- You need both programmatic API and CLI access
- You're building a platform/framework that needs LLM capabilities
- You want maximum reusability and flexibility
- You care about browser compatibility for some features

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Library-First Design](#library-first-design)
3. [Development Phases](#development-phases)
4. [API Design](#api-design)
5. [CLI Layer](#cli-layer)
6. [Browser Compatibility](#browser-compatibility)
7. [Success Metrics](#success-metrics)

---

## Architecture Overview

### Package Structure

```
llm-ts/
├── packages/
│   ├── core/                    # @llm-ts/core - Core library
│   │   ├── src/
│   │   │   ├── client.ts       # Main LLM client
│   │   │   ├── models/         # Model abstractions
│   │   │   ├── providers/      # Provider implementations
│   │   │   ├── types/          # TypeScript types
│   │   │   └── index.ts        # Public API
│   │   ├── package.json
│   │   └── tsconfig.json
│   ├── plugins/                 # @llm-ts/plugins - Plugin system
│   │   ├── src/
│   │   │   ├── manager.ts
│   │   │   ├── types.ts
│   │   │   └── index.ts
│   │   └── package.json
│   ├── templates/               # @llm-ts/templates - Template engine
│   │   ├── src/
│   │   │   ├── manager.ts
│   │   │   ├── renderer.ts
│   │   │   └── index.ts
│   │   └── package.json
│   ├── tools/                   # @llm-ts/tools - Tool system
│   │   ├── src/
│   │   │   ├── registry.ts
│   │   │   ├── executor.ts
│   │   │   └── index.ts
│   │   └── package.json
│   ├── embeddings/              # @llm-ts/embeddings - Embeddings
│   │   ├── src/
│   │   │   ├── collection.ts
│   │   │   ├── search.ts
│   │   │   └── index.ts
│   │   └── package.json
│   ├── storage/                 # @llm-ts/storage - Storage layer
│   │   ├── src/
│   │   │   ├── sqlite.ts
│   │   │   ├── memory.ts       # In-memory for browser
│   │   │   └── index.ts
│   │   └── package.json
│   └── cli/                     # llm-ts - CLI application
│       ├── src/
│       │   ├── commands/
│       │   ├── index.ts
│       │   └── utils/
│       ├── bin/
│       │   └── llm.js
│       └── package.json
├── examples/                    # Usage examples
│   ├── node/                    # Node.js examples
│   ├── browser/                 # Browser examples
│   └── plugins/                 # Plugin examples
├── docs/
│   ├── api/                     # API documentation
│   └── guides/                  # User guides
└── README.md
```

### Design Principles

**1. Library-First**
- Core functionality as pure library
- CLI is thin wrapper around library
- Same APIs available programmatically

**2. Environment Agnostic**
- Core works in Node.js and browser
- Environment-specific adapters where needed
- Feature detection for capabilities

**3. Composable**
- Small, focused packages
- Mix and match as needed
- Clear dependencies between packages

**4. Type-Safe**
- Comprehensive TypeScript types
- Exported types for consumers
- Generic types for flexibility

---

## Library-First Design

### Core Library (@llm-ts/core)

**Public API Design**:

```typescript
// Simple usage
import { LLM } from '@llm-ts/core';

const llm = new LLM({
  provider: 'openai',
  apiKey: process.env.OPENAI_API_KEY
});

const response = await llm.prompt('What is TypeScript?');
console.log(response.text);

// Advanced usage with options
const response = await llm.prompt('Explain concisely', {
  model: 'gpt-4o',
  system: 'You are a helpful teacher',
  temperature: 0.7,
  maxTokens: 500
});

// Streaming
for await (const chunk of llm.stream('Tell me a story')) {
  process.stdout.write(chunk);
}

// Conversations
const conversation = llm.conversation('gpt-4o');
await conversation.send('Hello!');
await conversation.send('How are you?');
console.log(conversation.messages); // Full history

// Tools
const weatherTool = {
  name: 'get_weather',
  description: 'Get weather for a location',
  parameters: {
    type: 'object',
    properties: {
      location: { type: 'string' }
    }
  },
  handler: async ({ location }) => {
    return { temp: 72, condition: 'sunny' };
  }
};

const response = await llm.prompt('What is the weather in SF?', {
  tools: [weatherTool]
});

// Structured output
import { z } from 'zod';

const schema = z.object({
  name: z.string(),
  age: z.number(),
  city: z.string()
});

const response = await llm.prompt('Generate a person', {
  schema
});

const person = response.object; // Typed as { name: string; age: number; city: string }
```

**Core Client Implementation**:

```typescript
// packages/core/src/client.ts
export class LLM {
  private provider: Provider;
  private config: LLMConfig;

  constructor(config: LLMConfig) {
    this.config = config;
    this.provider = this.createProvider(config.provider, config.apiKey);
  }

  async prompt(
    text: string,
    options?: PromptOptions
  ): Promise<Response> {
    const model = this.getModel(options?.model);
    const prompt = this.buildPrompt(text, options);

    const result = await generateText({
      model,
      prompt: prompt.text,
      system: prompt.system,
      tools: this.convertTools(prompt.tools),
      temperature: options?.temperature,
      maxTokens: options?.maxTokens
    });

    return this.wrapResponse(result, options);
  }

  async *stream(
    text: string,
    options?: PromptOptions
  ): AsyncGenerator<string> {
    const model = this.getModel(options?.model);
    const prompt = this.buildPrompt(text, options);

    const { textStream } = await streamText({
      model,
      prompt: prompt.text,
      system: prompt.system
    });

    for await (const chunk of textStream) {
      yield chunk;
    }
  }

  conversation(model?: string, system?: string): Conversation {
    return new Conversation(this, model || this.config.defaultModel, system);
  }

  // Provider management
  use(provider: Provider): void {
    this.provider = provider;
  }

  // Plugin support
  plugin(plugin: Plugin): void {
    plugin.register(this);
  }

  private createProvider(name: string, apiKey?: string): Provider {
    const provider = PROVIDERS[name];
    if (!provider) {
      throw new Error(`Unknown provider: ${name}`);
    }
    return new provider(apiKey);
  }

  private getModel(modelId?: string): LanguageModel {
    const id = modelId || this.config.defaultModel;
    return this.provider.createModel(id);
  }
}

// packages/core/src/conversation.ts
export class Conversation {
  private messages: Message[] = [];

  constructor(
    private llm: LLM,
    private model: string,
    system?: string
  ) {
    if (system) {
      this.messages.push({ role: 'system', content: system });
    }
  }

  async send(text: string, options?: PromptOptions): Promise<Response> {
    this.messages.push({ role: 'user', content: text });

    const response = await this.llm.prompt(text, {
      ...options,
      model: this.model,
      messages: this.messages
    });

    this.messages.push({
      role: 'assistant',
      content: response.text
    });

    return response;
  }

  async *stream(text: string, options?: PromptOptions): AsyncGenerator<string> {
    this.messages.push({ role: 'user', content: text });

    let fullResponse = '';
    for await (const chunk of this.llm.stream(text, {
      ...options,
      model: this.model,
      messages: this.messages
    })) {
      fullResponse += chunk;
      yield chunk;
    }

    this.messages.push({
      role: 'assistant',
      content: fullResponse
    });
  }

  clear(): void {
    const system = this.messages.find(m => m.role === 'system');
    this.messages = system ? [system] : [];
  }

  export(): ConversationData {
    return {
      model: this.model,
      messages: this.messages,
      timestamp: Date.now()
    };
  }

  static import(llm: LLM, data: ConversationData): Conversation {
    const conv = new Conversation(llm, data.model);
    conv.messages = data.messages;
    return conv;
  }
}
```

### Type Definitions

```typescript
// packages/core/src/types/index.ts

export interface LLMConfig {
  provider: string;
  apiKey?: string;
  defaultModel?: string;
  baseURL?: string;
  timeout?: number;
}

export interface PromptOptions {
  model?: string;
  system?: string;
  temperature?: number;
  maxTokens?: number;
  tools?: Tool[];
  schema?: JSONSchema | z.ZodType;
  messages?: Message[];
  attachments?: Attachment[];
}

export interface Response {
  text: string;
  model: string;
  usage?: Usage;
  metadata?: Record<string, any>;
  object?: any; // For schema-based responses
  toolCalls?: ToolCall[];
}

export interface Message {
  role: 'system' | 'user' | 'assistant' | 'tool';
  content: string;
  name?: string;
  toolCallId?: string;
}

export interface Tool {
  name: string;
  description: string;
  parameters: JSONSchema;
  handler: (args: any) => Promise<any> | any;
}

export interface ConversationData {
  model: string;
  messages: Message[];
  timestamp: number;
}

// Re-export for convenience
export { LLM } from './client';
export { Conversation } from './conversation';
export { Provider } from './providers/base';
```

---

## Development Phases

### Phase 1: Core Library (Weeks 1-3)

**Week 1: Foundation**
- Set up monorepo with Turborepo or pnpm workspaces
- Create @llm-ts/core package structure
- Implement basic LLM client
- Add provider abstraction
- OpenAI provider implementation

**Week 2: Advanced Features**
- Streaming support
- Conversation management
- Tool/function calling
- Structured output with Zod

**Week 3: Testing & Polish**
- Comprehensive unit tests
- Integration tests with mocked APIs
- API documentation with TypeDoc
- Example applications

**Deliverables**:
```bash
npm install @llm-ts/core
```

```typescript
import { LLM } from '@llm-ts/core';

const llm = new LLM({
  provider: 'openai',
  apiKey: process.env.OPENAI_API_KEY
});

const response = await llm.prompt('Hello!');
```

### Phase 2: Extended Packages (Weeks 4-5)

**@llm-ts/plugins**:
```typescript
import { LLM } from '@llm-ts/core';
import { PluginManager } from '@llm-ts/plugins';

const llm = new LLM(config);
const plugins = new PluginManager();

// Load plugin
await plugins.load('llm-plugin-custom-provider');
plugins.register(llm);
```

**@llm-ts/templates**:
```typescript
import { TemplateManager } from '@llm-ts/templates';

const templates = new TemplateManager('./templates');

const template = templates.load('summarize');
const rendered = template.render({ input: 'Long text...' });

const response = await llm.prompt(rendered, template.options);
```

**@llm-ts/tools**:
```typescript
import { ToolRegistry } from '@llm-ts/tools';

const tools = new ToolRegistry();

// Register from function
tools.registerFunction(function getWeather(location: string) {
  return { temp: 72, condition: 'sunny' };
});

// Use in prompts
const response = await llm.prompt('Weather in SF?', {
  tools: tools.getAll()
});
```

**@llm-ts/embeddings**:
```typescript
import { EmbeddingClient } from '@llm-ts/embeddings';

const embeddings = new EmbeddingClient(config);

// Create collection
const collection = await embeddings.collection('docs');

// Add documents
await collection.add([
  { id: '1', text: 'TypeScript is great' },
  { id: '2', text: 'JavaScript is flexible' }
]);

// Search
const results = await collection.search('programming languages', { limit: 5 });
```

**@llm-ts/storage**:
```typescript
import { SQLiteStorage } from '@llm-ts/storage';

const storage = new SQLiteStorage('./data.db');

// Log responses
await storage.logResponse(response, prompt);

// Query logs
const logs = await storage.getLogs({ model: 'gpt-4o', limit: 10 });

// Browser-compatible in-memory storage
import { MemoryStorage } from '@llm-ts/storage/memory';
const memStorage = new MemoryStorage();
```

### Phase 3: CLI Layer (Weeks 6-7)

**CLI Architecture**:

```typescript
// packages/cli/src/index.ts
import { Command } from 'commander';
import { LLM } from '@llm-ts/core';
import { TemplateManager } from '@llm-ts/templates';
import { ToolRegistry } from '@llm-ts/tools';
import { SQLiteStorage } from '@llm-ts/storage';

// Initialize library components
const config = loadConfig();
const llm = new LLM(config);
const templates = new TemplateManager(config.templatesDir);
const tools = new ToolRegistry();
const storage = new SQLiteStorage(config.dbPath);

// Create CLI
const program = new Command()
  .name('llm')
  .description('CLI powered by @llm-ts packages')
  .version('1.0.0');

// Prompt command - thin wrapper
program
  .argument('[text]', 'Prompt text')
  .option('-m, --model <model>', 'Model to use')
  .option('-t, --template <name>', 'Use template')
  .action(async (text, options) => {
    let prompt = text;

    // Handle template
    if (options.template) {
      const template = templates.load(options.template);
      prompt = template.render({ input: text });
    }

    // Execute using library
    const response = await llm.prompt(prompt, {
      model: options.model
    });

    // Log using library
    await storage.logResponse(response, prompt);

    // Output
    console.log(response.text);
  });

// Chat command - uses Conversation class
program
  .command('chat')
  .option('-m, --model <model>', 'Model to use')
  .action(async (options) => {
    const conversation = llm.conversation(options.model);

    // Interactive loop using library's conversation
    while (true) {
      const input = await askQuestion('> ');
      if (input === 'exit') break;

      const response = await conversation.send(input);
      console.log(response.text);
    }
  });

program.parse();
```

**Key Principle**: CLI is just a thin layer that:
1. Handles user input/output
2. Loads configuration
3. Calls library methods
4. Formats output

All logic lives in the library packages!

### Phase 4: Browser Compatibility (Week 8)

**Browser-Specific Adaptations**:

```typescript
// packages/core/src/adapters/browser.ts
export class BrowserLLM extends LLM {
  constructor(config: LLMConfig) {
    super(config);

    // Use fetch instead of node-specific HTTP
    // Store data in IndexedDB instead of SQLite
    // Handle CORS appropriately
  }
}

// Usage in browser
import { BrowserLLM } from '@llm-ts/core/browser';

const llm = new BrowserLLM({
  provider: 'openai',
  apiKey: getApiKeyFromLocalStorage()
});

const response = await llm.prompt('Hello!');
```

**Build Configuration**:

```javascript
// vite.config.ts
export default defineConfig({
  build: {
    lib: {
      entry: {
        index: 'src/index.ts',
        browser: 'src/adapters/browser.ts'
      },
      formats: ['es', 'cjs']
    },
    rollupOptions: {
      external: ['better-sqlite3', 'fs', 'path'] // Node-only modules
    }
  }
});
```

**Package.json Exports**:

```json
{
  "name": "@llm-ts/core",
  "exports": {
    ".": {
      "import": "./dist/index.js",
      "require": "./dist/index.cjs"
    },
    "./browser": {
      "import": "./dist/browser.js"
    }
  }
}
```

### Phase 5: Polish & Documentation (Weeks 9-10)

**Comprehensive Documentation**:

```
docs/
├── getting-started/
│   ├── installation.md
│   ├── quick-start.md
│   └── core-concepts.md
├── guides/
│   ├── prompting.md
│   ├── conversations.md
│   ├── tools.md
│   ├── templates.md
│   └── embeddings.md
├── api/
│   ├── core.md           # Auto-generated from TypeDoc
│   ├── plugins.md
│   ├── templates.md
│   └── tools.md
├── cli/
│   ├── commands.md
│   └── configuration.md
└── examples/
    ├── node-app.md
    ├── browser-app.md
    └── custom-plugin.md
```

**Example Applications**:

```
examples/
├── node/
│   ├── basic-prompt/
│   ├── chatbot/
│   ├── tool-calling/
│   └── custom-provider/
├── browser/
│   ├── chat-widget/
│   ├── semantic-search/
│   └── code-assistant/
└── plugins/
    ├── custom-model/
    ├── custom-tool/
    └── template-loader/
```

---

## API Design Patterns

### Fluent API

```typescript
const response = await llm
  .model('gpt-4o')
  .system('You are a helpful assistant')
  .temperature(0.7)
  .prompt('Explain TypeScript');
```

### Builder Pattern

```typescript
const prompt = llm
  .builder()
  .text('Summarize this:')
  .fragment('my-article')
  .tool(weatherTool)
  .schema(PersonSchema)
  .build();

const response = await llm.execute(prompt);
```

### Event-Driven

```typescript
llm.on('response', (response) => {
  console.log('Got response:', response.text);
});

llm.on('tool-call', (call) => {
  console.log('Tool called:', call.name);
});

await llm.prompt('What is the weather?', {
  tools: [weatherTool]
});
```

### Middleware

```typescript
llm.use(async (context, next) => {
  console.log('Before prompt:', context.prompt);
  const response = await next();
  console.log('After response:', response.text);
  return response;
});
```

---

## CLI Layer Design

### Command Structure

```
llm [command] [options]

Commands:
  llm [text]           Execute a prompt (default)
  llm chat            Start interactive chat
  llm models          List available models
  llm keys            Manage API keys
  llm templates       Manage templates
  llm tools           List available tools
  llm embed           Generate embeddings
  llm similar         Find similar items
  llm logs            View prompt history
```

### Implementation Philosophy

**❌ Don't**: Implement logic in CLI
```typescript
// Bad - logic in CLI
program.action(async (text) => {
  const model = openai('gpt-4o', { apiKey });
  const result = await generateText({ model, prompt: text });
  console.log(result.text);
});
```

**✅ Do**: Call library methods
```typescript
// Good - use library
program.action(async (text, options) => {
  const response = await llm.prompt(text, options);
  console.log(response.text);
});
```

### Configuration Management

```typescript
// packages/cli/src/config.ts
import { LLM } from '@llm-ts/core';
import { loadConfig } from './utils';

export function createLLMFromConfig(): LLM {
  const config = loadConfig(); // Load from ~/.llm/config.json

  return new LLM({
    provider: config.defaultProvider,
    apiKey: config.keys[config.defaultProvider],
    defaultModel: config.defaultModel
  });
}

// In commands
const llm = createLLMFromConfig();
```

---

## Browser Compatibility

### What Works in Browser

✅ Core prompt execution
✅ Streaming responses
✅ Conversation management
✅ Tool calling
✅ Structured output
✅ In-memory storage

### What Requires Adaptation

⚠️ SQLite → IndexedDB
⚠️ File system → Browser storage APIs
⚠️ Node.js streams → Web Streams API

### Example: Browser Chat App

```html
<!DOCTYPE html>
<html>
<head>
  <title>LLM Chat</title>
</head>
<body>
  <div id="chat"></div>
  <input id="input" type="text">
  <button onclick="send()">Send</button>

  <script type="module">
    import { BrowserLLM } from 'https://esm.sh/@llm-ts/core/browser';

    const llm = new BrowserLLM({
      provider: 'openai',
      apiKey: localStorage.getItem('openai-key')
    });

    const conversation = llm.conversation();

    window.send = async function() {
      const input = document.getElementById('input');
      const text = input.value;
      input.value = '';

      // Add user message
      addMessage('user', text);

      // Get response
      const response = await conversation.send(text);
      addMessage('assistant', response.text);
    };

    function addMessage(role, text) {
      const chat = document.getElementById('chat');
      const div = document.createElement('div');
      div.textContent = `${role}: ${text}`;
      chat.appendChild(div);
    }
  </script>
</body>
</html>
```

---

## Success Metrics

### Library Adoption

**npm Downloads**: Target 1000+/month after 3 months

**GitHub Stars**: Target 500+ after 6 months

**Usage Patterns**:
- 70% programmatic (library)
- 30% CLI

### API Quality

**Type Safety**: 100% TypeScript coverage

**Documentation**: Every public API documented

**Examples**: 20+ example applications

**Test Coverage**: >85%

### Performance

**Bundle Size**:
- @llm-ts/core: <50KB minified
- Browser build: <100KB minified

**Startup Time**:
- CLI cold start: <100ms
- Library import: <10ms

**Memory Usage**:
- Single conversation: <10MB
- With history (100 messages): <50MB

### Developer Experience

**Installation Time**: <30 seconds

**Time to First Prompt**: <2 minutes

**Learning Curve**: Basic usage in 5 minutes

**Error Messages**: Helpful with suggestions

---

## Comparison Matrix

| Aspect | Approach 1 | Approach 2 | Approach 3 |
|--------|-----------|-----------|-----------|
| Primary Use | CLI | CLI | Library + CLI |
| Programmatic API | ❌ | ⚠️ (Limited) | ✅ (Full) |
| Browser Support | ❌ | ❌ | ✅ |
| Reusability | Low | Medium | High |
| Timeline | 2 weeks | 10-12 weeks | 8-10 weeks |
| Complexity | Low | High | Medium-High |
| Best For | Learning | Production CLI | Platform Building |

---

## Migration from Python

### API Comparison

**Python**:
```python
import llm

model = llm.get_model("gpt-4o")
response = model.prompt("Hello")
print(response.text())
```

**TypeScript Library**:
```typescript
import { LLM } from '@llm-ts/core';

const llm = new LLM({ provider: 'openai' });
const response = await llm.prompt('Hello', { model: 'gpt-4o' });
console.log(response.text);
```

### Feature Mapping

| Python Feature | TypeScript Package |
|---------------|-------------------|
| `llm.get_model()` | `@llm-ts/core` LLM client |
| `model.prompt()` | `llm.prompt()` |
| `model.conversation()` | `llm.conversation()` |
| Templates | `@llm-ts/templates` |
| Tools | `@llm-ts/tools` |
| Embeddings | `@llm-ts/embeddings` |
| Plugins | `@llm-ts/plugins` |
| Logging | `@llm-ts/storage` |

---

## Example Use Cases

### 1. Custom Application

```typescript
// Building a customer support chatbot
import { LLM } from '@llm-ts/core';
import { ToolRegistry } from '@llm-ts/tools';

const llm = new LLM(config);
const tools = new ToolRegistry();

// Register custom tools
tools.registerFunction(async function lookupOrder(orderId: string) {
  return await db.orders.findById(orderId);
});

tools.registerFunction(async function createTicket(issue: string) {
  return await supportSystem.createTicket(issue);
});

// Use in application
export async function handleCustomerQuery(query: string) {
  const response = await llm.prompt(query, {
    system: 'You are a helpful customer support agent',
    tools: tools.getAll()
  });

  return response.text;
}
```

### 2. Documentation Generator

```typescript
import { LLM } from '@llm-ts/core';
import { readFileSync } from 'fs';

const llm = new LLM(config);

async function generateDocs(codeFile: string) {
  const code = readFileSync(codeFile, 'utf-8');

  const response = await llm.prompt(
    `Generate documentation for:\n\n${code}`,
    {
      model: 'gpt-4o',
      system: 'You are a technical writer',
      schema: DocumentationSchema
    }
  );

  return response.object; // Typed documentation
}
```

### 3. CLI Tool Builder

```typescript
// Other developers can build CLIs using your library
import { LLM } from '@llm-ts/core';
import { Command } from 'commander';

const llm = new LLM(config);

const program = new Command()
  .name('my-ai-tool')
  .command('analyze <file>')
  .action(async (file) => {
    const content = readFileSync(file, 'utf-8');
    const response = await llm.prompt(`Analyze: ${content}`);
    console.log(response.text);
  });

program.parse();
```

---

## Conclusion

This approach provides the best of both worlds: a powerful, reusable library for programmatic use, and a fully-featured CLI built on top of it.

**Strengths**:
- Maximum reusability
- Can be used in applications
- Browser compatibility
- Clear separation of concerns
- Multiple packages for specific needs
- Excellent for building platforms

**Trade-offs**:
- More complex than Approach 1
- Requires thinking about multiple use cases
- More packages to maintain
- Need to balance library and CLI features

**Best For**:
- Building products that use LLMs
- Creating platforms or frameworks
- Developers who want programmatic access
- Teams building multiple AI-powered applications
- Open source projects expecting broad adoption

**ROI**: The investment in library architecture pays dividends when you or others want to use LLM functionality in applications, not just from the command line.
