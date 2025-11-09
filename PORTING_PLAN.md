# LLM to Vercel AI SDK Port Planning Document

## Executive Summary

This document outlines comprehensive approaches for porting the `llm` Python library/application to TypeScript using the Vercel AI SDK. The goal is to recreate the core functionality of `llm` in a TypeScript environment while leveraging the Vercel AI SDK's existing capabilities.

## Table of Contents

1. [Understanding the Systems](#understanding-the-systems)
2. [Core Functionality Analysis](#core-functionality-analysis)
3. [Porting Approaches](#porting-approaches)
4. [Architecture Mapping](#architecture-mapping)
5. [Implementation Roadmap](#implementation-roadmap)
6. [Challenges and Considerations](#challenges-and-considerations)
7. [Code Examples and Comparisons](#code-examples-and-comparisons)

---

## Understanding the Systems

### The LLM Python Library

**Purpose**: A CLI tool and Python library for interacting with multiple Large Language Models through a unified interface.

**Key Characteristics**:
- Plugin-based architecture using `pluggy`
- CLI-first design with comprehensive Python API underneath
- SQLite-based persistence for logs, embeddings, and configuration
- Support for sync/async operations
- Extensive template and fragment system
- Multi-modal attachments support
- Tool/function calling capabilities
- Structured output via schemas

**Dependencies**:
- click (CLI framework)
- openai (OpenAI SDK)
- pydantic (data validation)
- sqlite-utils (database operations)
- pluggy (plugin system)
- PyYAML (configuration)

### The Vercel AI SDK

**Purpose**: A TypeScript toolkit for building AI-powered applications with unified provider support.

**Key Characteristics** (based on research):
- Provider-agnostic architecture
- Core functions: `generateText`, `streamText`, `generateObject`, `streamObject`
- Built-in streaming support
- Tool calling with type safety
- Structured output generation
- React/Next.js integration (not needed for CLI)
- Support for OpenAI, Anthropic, Google, and more

**Core Packages**:
- `ai` - Main SDK package
- Provider-specific packages (e.g., `@ai-sdk/openai`, `@ai-sdk/anthropic`)
- Stream helpers and utilities

---

## Core Functionality Analysis

### Critical Features to Port

| Feature | LLM Python | Complexity | Priority |
|---------|------------|------------|----------|
| Basic prompt execution | ✓ | Low | **Critical** |
| Multi-provider support | ✓ | Medium | **Critical** |
| Streaming responses | ✓ | Low | **Critical** |
| API key management | ✓ | Medium | **Critical** |
| CLI interface | ✓ | Medium | **Critical** |
| Conversation/chat mode | ✓ | Medium | **High** |
| Tool/function calling | ✓ | Medium | **High** |
| Structured output (schemas) | ✓ | Medium | **High** |
| SQLite logging | ✓ | Medium | **High** |
| Templates | ✓ | Medium | **High** |
| Model aliases | ✓ | Low | **Medium** |
| Attachments (multi-modal) | ✓ | Medium | **Medium** |
| Fragments | ✓ | Medium | **Medium** |
| Plugin system | ✓ | High | **Medium** |
| Embeddings | ✓ | Medium | **Low** |
| Collections (vector search) | ✓ | High | **Low** |

### Feature Mapping: LLM → Vercel AI SDK

#### Direct Mappings (SDK has equivalent)

```typescript
// LLM Python
model = llm.get_model("gpt-4")
response = model.prompt("Hello")

// Vercel AI SDK equivalent
import { openai } from '@ai-sdk/openai'
import { generateText } from 'ai'

const { text } = await generateText({
  model: openai('gpt-4'),
  prompt: 'Hello'
})
```

#### Features Requiring Custom Implementation

1. **SQLite Logging**: Need to build custom logger
2. **Plugin System**: Need custom plugin architecture
3. **Templates**: Custom template engine
4. **Fragments**: Custom fragment management
5. **CLI**: Use a Node.js CLI framework (e.g., `commander`, `oclif`, `ink`)
6. **Key Management**: Custom secure key storage
7. **Embeddings & Collections**: Custom implementation or separate library

---

## Porting Approaches

### Approach 1: Minimal Viable CLI (Recommended for Learning)

**Strategy**: Start with core functionality, gradually add features.

**Scope**:
- Basic CLI with prompt execution
- Support for 2-3 providers (OpenAI, Anthropic, Google)
- Simple key management (environment variables or config file)
- Streaming support
- Basic conversation history

**Pros**:
- Quick to implement (1-2 weeks)
- Learn Vercel AI SDK incrementally
- Early usable product
- Clear success metrics

**Cons**:
- Missing advanced features initially
- May require refactoring as features are added

**Implementation Steps**:
```
Phase 1 (Week 1):
  ├── Set up TypeScript project
  ├── Implement basic CLI using commander.js
  ├── Integrate Vercel AI SDK
  ├── Add OpenAI provider support
  └── Implement prompt execution

Phase 2 (Week 2):
  ├── Add Anthropic and Google providers
  ├── Implement streaming
  ├── Add simple config file for keys
  ├── Basic conversation mode
  └── Documentation
```

### Approach 2: Feature-Complete Port

**Strategy**: Aim for near-complete feature parity with Python version.

**Scope**:
- Full CLI with all major commands
- Plugin architecture for extensibility
- SQLite logging and history
- Templates and fragments
- Tool calling
- Structured output
- Embeddings and collections
- Multi-modal support

**Pros**:
- Feature parity with Python version
- Production-ready
- Extensible architecture

**Cons**:
- 2-3 months development time
- Complex architecture needed upfront
- Steeper learning curve

**Implementation Steps**:
```
Phase 1 - Foundation (2-3 weeks):
  ├── Project architecture
  ├── Core abstractions (Model, Response, Conversation)
  ├── Provider integration layer
  ├── CLI framework
  └── Configuration system

Phase 2 - Core Features (3-4 weeks):
  ├── Prompt execution and streaming
  ├── Conversation management
  ├── SQLite logging
  ├── Key management
  └── Model aliases

Phase 3 - Advanced Features (3-4 weeks):
  ├── Templates and fragments
  ├── Tool calling
  ├── Structured output
  ├── Multi-modal attachments
  └── Plugin system

Phase 4 - Embeddings & Polish (2-3 weeks):
  ├── Embedding models
  ├── Collection management
  ├── Vector search
  ├── Documentation
  └── Testing
```

### Approach 3: Hybrid Library + CLI

**Strategy**: Build a TypeScript library first, then add CLI on top.

**Scope**:
- TypeScript/JavaScript library for programmatic use
- CLI as a separate layer
- Designed for both Node.js and browser use (where applicable)

**Pros**:
- Reusable library for other projects
- Clean separation of concerns
- Can be used programmatically or via CLI
- Browser-compatible components

**Cons**:
- More complex architecture
- Longer initial development
- Need to consider both use cases

**Structure**:
```
llm-ts/
├── packages/
│   ├── core/           # Core abstractions
│   ├── providers/      # Provider implementations
│   ├── cli/            # CLI application
│   ├── templates/      # Template engine
│   ├── embeddings/     # Embedding functionality
│   └── plugins/        # Plugin system
└── package.json
```

### Approach 4: Incremental Migration

**Strategy**: Create TypeScript versions of individual components, use both side-by-side.

**Scope**:
- Port specific features one at a time
- Maintain Python version alongside
- Share configuration and databases
- Gradually shift users to TypeScript version

**Pros**:
- Low risk
- Can validate each component
- Users can choose which to use
- Easier to maintain backward compatibility

**Cons**:
- Maintaining two codebases
- Potential for confusion
- Database schema compatibility challenges

---

## Architecture Mapping

### Model Abstraction Layer

**Python (llm)**:
```python
class Model(ABC):
    @abstractmethod
    def execute(self, prompt, stream, response, conversation=None):
        pass

class KeyModel(Model):
    needs_key = "provider_name"
    key_env_var = "PROVIDER_API_KEY"
```

**TypeScript (proposed)**:
```typescript
interface ModelConfig {
  provider: string;
  modelId: string;
  apiKey?: string;
}

abstract class BaseModel {
  abstract execute(
    prompt: Prompt,
    options: ExecuteOptions
  ): AsyncGenerator<string> | Promise<string>;
}

class KeyModel extends BaseModel {
  readonly needsKey: string;
  readonly keyEnvVar: string;
}
```

### Plugin System

**Python (using pluggy)**:
```python
# hookspecs.py
@hookspec
def register_models(register):
    """Register LLM models"""

# plugin.py
@hookimpl
def register_models(register):
    register(MyModel())
```

**TypeScript (proposed - multiple options)**:

Option A - Simple Registry Pattern:
```typescript
// registry.ts
class ModelRegistry {
  private models = new Map<string, ModelFactory>();

  register(name: string, factory: ModelFactory) {
    this.models.set(name, factory);
  }

  get(name: string): Model | undefined {
    const factory = this.models.get(name);
    return factory?.create();
  }
}

// plugin.ts
export function registerModels(registry: ModelRegistry) {
  registry.register('my-model', new MyModelFactory());
}
```

Option B - Using a Plugin Manager:
```typescript
// Could use existing systems like:
// - jiti (for dynamic imports)
// - tapable (webpack's plugin system)
// - Custom EventEmitter-based system
```

### Configuration and Storage

**Python**:
- User directory: `~/.config/io.datasette.llm/`
- SQLite databases: `logs.db`, `embeddings.db`
- JSON files: `keys.json`, `aliases.json`
- YAML files: templates

**TypeScript**:
```typescript
import { homedir } from 'os';
import { join } from 'path';
import Database from 'better-sqlite3';

class Config {
  private configDir: string;
  private db: Database.Database;

  constructor() {
    this.configDir = process.env.LLM_USER_PATH ||
      join(homedir(), '.config', 'llm-ts');
    this.db = new Database(join(this.configDir, 'logs.db'));
  }

  // Methods for key management, aliases, etc.
}
```

**Dependencies**:
- `better-sqlite3` - SQLite bindings
- `conf` - Configuration management
- `keytar` - Secure key storage (optional)

### CLI Structure

**Python (Click)**:
```python
@cli.command()
@click.argument("prompt")
@click.option("-m", "--model")
@click.option("-s", "--system")
def prompt(prompt, model, system):
    # Implementation
```

**TypeScript Options**:

Option A - Commander.js (Similar to Click):
```typescript
import { Command } from 'commander';

const program = new Command();

program
  .command('prompt <text>')
  .option('-m, --model <model>', 'Model to use')
  .option('-s, --system <text>', 'System prompt')
  .action(async (text, options) => {
    // Implementation
  });
```

Option B - oclif (More robust):
```typescript
import { Command, Flags } from '@oclif/core';

export default class Prompt extends Command {
  static args = {
    text: Args.string({ required: true })
  };

  static flags = {
    model: Flags.string({ char: 'm' }),
    system: Flags.string({ char: 's' })
  };

  async run() {
    const { args, flags } = await this.parse(Prompt);
    // Implementation
  }
}
```

### Streaming Responses

**Python**:
```python
for chunk in model.prompt("Hello", stream=True):
    print(chunk, end="")
```

**TypeScript with Vercel AI SDK**:
```typescript
import { streamText } from 'ai';

const { textStream } = await streamText({
  model: openai('gpt-4'),
  prompt: 'Hello'
});

for await (const chunk of textStream) {
  process.stdout.write(chunk);
}
```

### Conversation Management

**Python**:
```python
conversation = model.conversation()
response1 = conversation.prompt("Hello")
response2 = conversation.prompt("How are you?")
```

**TypeScript (proposed)**:
```typescript
class Conversation {
  private messages: Message[] = [];

  async prompt(text: string): Promise<Response> {
    this.messages.push({ role: 'user', content: text });

    const result = await generateText({
      model: this.model,
      messages: this.messages
    });

    this.messages.push({
      role: 'assistant',
      content: result.text
    });

    return new Response(result);
  }
}
```

---

## Implementation Roadmap

### Recommended Learning Path: Approach 1 (Minimal Viable CLI)

#### Week 1: Foundation

**Day 1-2: Project Setup**
```bash
# Initialize project
npm init -y
npm install typescript @types/node tsx
npm install ai @ai-sdk/openai @ai-sdk/anthropic
npm install commander chalk
npm install --save-dev @types/commander

# Create tsconfig.json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "outDir": "./dist"
  }
}
```

**Tasks**:
- [ ] Initialize TypeScript project
- [ ] Set up build system (tsx/tsc)
- [ ] Create basic CLI skeleton
- [ ] Set up development environment

**Day 3-4: Basic Prompt Execution**
```typescript
// src/cli.ts
import { Command } from 'commander';
import { openai } from '@ai-sdk/openai';
import { generateText } from 'ai';

const program = new Command();

program
  .name('llm-ts')
  .description('CLI for interacting with LLMs')
  .version('0.1.0');

program
  .argument('<prompt>', 'The prompt to send')
  .option('-m, --model <model>', 'Model to use', 'gpt-4o-mini')
  .action(async (prompt, options) => {
    const { text } = await generateText({
      model: openai(options.model),
      prompt
    });
    console.log(text);
  });

program.parse();
```

**Tasks**:
- [ ] Implement basic prompt command
- [ ] Add OpenAI integration
- [ ] Handle errors gracefully
- [ ] Add basic help text

**Day 5-7: Configuration and Key Management**
```typescript
// src/config.ts
import { readFileSync, writeFileSync, existsSync, mkdirSync } from 'fs';
import { join } from 'path';
import { homedir } from 'os';

export class Config {
  private configDir: string;
  private keysFile: string;

  constructor() {
    this.configDir = join(homedir(), '.config', 'llm-ts');
    this.keysFile = join(this.configDir, 'keys.json');
    this.ensureConfigDir();
  }

  private ensureConfigDir() {
    if (!existsSync(this.configDir)) {
      mkdirSync(this.configDir, { recursive: true });
    }
  }

  getKey(provider: string): string | undefined {
    if (!existsSync(this.keysFile)) return undefined;
    const keys = JSON.parse(readFileSync(this.keysFile, 'utf-8'));
    return keys[provider];
  }

  setKey(provider: string, key: string) {
    const keys = existsSync(this.keysFile)
      ? JSON.parse(readFileSync(this.keysFile, 'utf-8'))
      : {};
    keys[provider] = key;
    writeFileSync(this.keysFile, JSON.stringify(keys, null, 2));
  }
}

// Add keys command
program
  .command('keys')
  .command('set <provider>')
  .action((provider) => {
    // Implement key setting with secure input
  });
```

**Tasks**:
- [ ] Create config directory structure
- [ ] Implement key storage
- [ ] Add `keys set` command
- [ ] Handle environment variable fallback

#### Week 2: Enhanced Features

**Day 8-10: Multiple Providers**
```typescript
// src/providers.ts
import { openai } from '@ai-sdk/openai';
import { anthropic } from '@ai-sdk/anthropic';
import { google } from '@ai-sdk/google';

export class ProviderManager {
  getModel(modelId: string, apiKey?: string) {
    const [provider, ...modelParts] = modelId.split('-');
    const model = modelParts.join('-');

    switch (provider) {
      case 'gpt':
        return openai(modelId, { apiKey });
      case 'claude':
        return anthropic(modelId, { apiKey });
      case 'gemini':
        return google(modelId, { apiKey });
      default:
        throw new Error(`Unknown provider: ${provider}`);
    }
  }
}
```

**Tasks**:
- [ ] Add Anthropic support
- [ ] Add Google Gemini support
- [ ] Implement provider detection
- [ ] Add model listing command

**Day 11-12: Streaming Support**
```typescript
program
  .option('--no-stream', 'Disable streaming')
  .action(async (prompt, options) => {
    if (options.stream) {
      const { textStream } = await streamText({
        model: getModel(options.model),
        prompt
      });

      for await (const chunk of textStream) {
        process.stdout.write(chunk);
      }
      console.log(); // New line at end
    } else {
      // Non-streaming path
    }
  });
```

**Tasks**:
- [ ] Implement streaming responses
- [ ] Add `--no-stream` flag
- [ ] Handle streaming errors
- [ ] Add progress indicators

**Day 13-14: Chat Mode**
```typescript
// src/chat.ts
import readline from 'readline/promises';

export class ChatSession {
  private messages: Array<{ role: string; content: string }> = [];

  async start(model: string) {
    const rl = readline.createInterface({
      input: process.stdin,
      output: process.stdout
    });

    console.log('Type "exit" to quit\n');

    while (true) {
      const input = await rl.question('> ');

      if (input === 'exit') break;

      this.messages.push({ role: 'user', content: input });

      const { textStream } = await streamText({
        model: getModel(model),
        messages: this.messages
      });

      let response = '';
      for await (const chunk of textStream) {
        process.stdout.write(chunk);
        response += chunk;
      }
      console.log('\n');

      this.messages.push({ role: 'assistant', content: response });
    }

    rl.close();
  }
}

program
  .command('chat')
  .option('-m, --model <model>', 'Model to use')
  .action((options) => {
    const session = new ChatSession();
    session.start(options.model);
  });
```

**Tasks**:
- [ ] Implement interactive chat
- [ ] Add conversation history
- [ ] Handle multi-line input
- [ ] Add special commands (!multi, !exit, etc.)

### Extended Roadmap (For Feature-Complete Port)

#### Phase 3: Logging and History (Week 3-4)

**Database Schema**:
```sql
CREATE TABLE conversations (
  id TEXT PRIMARY KEY,
  name TEXT,
  model TEXT,
  created_at INTEGER
);

CREATE TABLE responses (
  id TEXT PRIMARY KEY,
  conversation_id TEXT,
  prompt TEXT,
  response TEXT,
  model TEXT,
  tokens_input INTEGER,
  tokens_output INTEGER,
  timestamp INTEGER,
  FOREIGN KEY (conversation_id) REFERENCES conversations(id)
);
```

**Implementation**:
```typescript
import Database from 'better-sqlite3';

class Logger {
  private db: Database.Database;

  logResponse(prompt: string, response: string, metadata: any) {
    this.db.prepare(`
      INSERT INTO responses (id, prompt, response, model, timestamp)
      VALUES (?, ?, ?, ?, ?)
    `).run(ulid(), prompt, response, metadata.model, Date.now());
  }

  getLogs(limit: number = 10) {
    return this.db.prepare(`
      SELECT * FROM responses
      ORDER BY timestamp DESC
      LIMIT ?
    `).all(limit);
  }
}
```

**Tasks**:
- [ ] Set up SQLite database
- [ ] Create migration system
- [ ] Implement logging for all prompts
- [ ] Add `logs` command
- [ ] Add log filtering and search

#### Phase 4: Templates (Week 5)

**Template Structure**:
```yaml
# ~/.config/llm-ts/templates/summarize.yaml
name: summarize
prompt: |
  Please summarize the following text:

  $input
system: You are a helpful assistant that creates concise summaries.
model: gpt-4o-mini
options:
  temperature: 0.7
```

**Implementation**:
```typescript
import yaml from 'yaml';

interface Template {
  name: string;
  prompt?: string;
  system?: string;
  model?: string;
  options?: Record<string, any>;
}

class TemplateManager {
  private templatesDir: string;

  load(name: string): Template {
    const path = join(this.templatesDir, `${name}.yaml`);
    const content = readFileSync(path, 'utf-8');
    return yaml.parse(content);
  }

  render(template: Template, variables: Record<string, string>): string {
    let prompt = template.prompt || '';
    for (const [key, value] of Object.entries(variables)) {
      prompt = prompt.replace(new RegExp(`\\$${key}`, 'g'), value);
    }
    return prompt;
  }
}

program
  .command('templates')
  .action(() => {
    // List templates
  });

// Use template
program
  .option('-t, --template <name>', 'Use template')
  .option('--param <key=value>', 'Template parameter', collect, [])
```

**Tasks**:
- [ ] Implement template loading
- [ ] Add variable substitution
- [ ] Support system prompts in templates
- [ ] Add template listing
- [ ] Support saving templates from CLI

#### Phase 5: Tools and Structured Output (Week 6-7)

**Tool Implementation**:
```typescript
import { tool } from 'ai';
import { z } from 'zod';

const weatherTool = tool({
  description: 'Get the weather for a location',
  parameters: z.object({
    location: z.string().describe('The city and state'),
  }),
  execute: async ({ location }) => {
    // Call weather API
    return { temperature: 72, condition: 'sunny' };
  },
});

// Use in prompt
const { text } = await generateText({
  model: openai('gpt-4'),
  prompt: 'What is the weather in San Francisco?',
  tools: { weather: weatherTool },
});
```

**Structured Output**:
```typescript
import { generateObject } from 'ai';
import { z } from 'zod';

const schema = z.object({
  recipe: z.object({
    name: z.string(),
    ingredients: z.array(z.string()),
    steps: z.array(z.string()),
  }),
});

const { object } = await generateObject({
  model: openai('gpt-4'),
  schema,
  prompt: 'Generate a recipe for chocolate chip cookies',
});
```

**Tasks**:
- [ ] Implement tool registration system
- [ ] Add built-in tools (like llm_time, llm_version)
- [ ] Support tool plugins
- [ ] Implement schema-based structured output
- [ ] Add schema storage and management

---

## Challenges and Considerations

### Technical Challenges

#### 1. Plugin System Architecture

**Challenge**: Python's `pluggy` is sophisticated. TypeScript alternatives are limited.

**Solutions**:
- **Option A**: Use simple registry pattern (easier, less flexible)
- **Option B**: Use `import()` for dynamic loading (modern, powerful)
- **Option C**: Use existing plugin systems (tapable, jiti)
- **Option D**: Port pluggy concepts to TypeScript (complex, most flexible)

**Recommendation**: Start with Option A, migrate to Option B for production.

#### 2. Async/Await vs Generator Patterns

**Challenge**: Python uses generators for streaming; TypeScript uses async iterators.

**Solution**:
```typescript
// Python style
for chunk in model.prompt("Hello", stream=True):
    print(chunk)

// TypeScript equivalent
for await (const chunk of model.prompt("Hello", { stream: true })) {
  console.log(chunk);
}
```

**Key Difference**: TypeScript async iterators are `async`, Python generators can be sync.

#### 3. SQLite in Node.js

**Challenge**: Multiple SQLite libraries with different APIs and performance.

**Options**:
- `better-sqlite3`: Fast, synchronous, native bindings
- `sql.js`: WASM-based, works in browser, slower
- `sqlite3`: Async, callback-based (older)

**Recommendation**: Use `better-sqlite3` for CLI, consider `sql.js` for web version.

#### 4. Type Safety vs Flexibility

**Challenge**: Python's dynamic typing allows more flexibility; TypeScript requires type definitions.

**Strategy**:
```typescript
// Use generics for flexibility
interface Model<TOptions = Record<string, any>> {
  execute<TResult = string>(
    prompt: Prompt,
    options?: TOptions
  ): Promise<TResult>;
}

// Use type guards for runtime checks
function isStreamableModel(model: Model): model is StreamableModel {
  return 'canStream' in model && model.canStream === true;
}
```

#### 5. Configuration File Locations

**Challenge**: Cross-platform config directory paths.

**Solution**:
```typescript
import { homedir } from 'os';
import { join } from 'path';

function getConfigDir(): string {
  if (process.env.LLM_USER_PATH) {
    return process.env.LLM_USER_PATH;
  }

  const platform = process.platform;
  const home = homedir();

  switch (platform) {
    case 'darwin':
      return join(home, 'Library', 'Application Support', 'llm-ts');
    case 'win32':
      return join(process.env.APPDATA || join(home, 'AppData', 'Roaming'), 'llm-ts');
    default:
      return join(home, '.config', 'llm-ts');
  }
}
```

### Feature Gaps

#### Vercel AI SDK Provides:
✅ Multi-provider support
✅ Streaming
✅ Tool calling
✅ Structured output
✅ Type safety

#### Need Custom Implementation:
❌ SQLite logging
❌ Template system
❌ Fragment management
❌ Plugin architecture
❌ Embedding storage/search
❌ CLI interface
❌ Key management
❌ Model aliases
❌ Conversation persistence

### Migration Considerations

#### Database Compatibility

**Question**: Should the TypeScript version use the same database schema?

**Pros**:
- Users can switch between versions
- Shared history
- Easier migration

**Cons**:
- Locked into Python schema decisions
- Harder to optimize for TypeScript
- May limit TypeScript-specific features

**Recommendation**: Start with compatible schema, diverge if needed.

#### API Compatibility

**Question**: Should the programmatic API match Python's?

**Example**:
```python
# Python
import llm
model = llm.get_model("gpt-4")
response = model.prompt("Hello")
```

```typescript
// TypeScript - Option A (match Python)
import * as llm from 'llm-ts';
const model = llm.getModel('gpt-4');
const response = await model.prompt('Hello');

// TypeScript - Option B (more idiomatic)
import { LLM } from 'llm-ts';
const llm = new LLM();
const response = await llm.prompt('gpt-4', 'Hello');
```

**Recommendation**: Provide both - a Python-like API and an idiomatic TypeScript API.

---

## Code Examples and Comparisons

### Example 1: Basic Prompt Execution

**Python (llm)**:
```python
import llm

model = llm.get_model("gpt-4")
response = model.prompt("Explain quantum computing in one sentence")
print(response.text())
```

**TypeScript (proposed)**:
```typescript
import { LLM } from 'llm-ts';

const llm = new LLM();
const response = await llm.prompt('gpt-4', 'Explain quantum computing in one sentence');
console.log(response.text);
```

### Example 2: Streaming Response

**Python**:
```python
import llm
import sys

model = llm.get_model("gpt-4")
for chunk in model.prompt("Write a haiku about code", stream=True):
    print(chunk, end="")
    sys.stdout.flush()
```

**TypeScript**:
```typescript
import { LLM } from 'llm-ts';

const llm = new LLM();
const stream = await llm.prompt('gpt-4', 'Write a haiku about code', {
  stream: true
});

for await (const chunk of stream) {
  process.stdout.write(chunk);
}
```

### Example 3: Conversation

**Python**:
```python
import llm

model = llm.get_model("gpt-4")
conversation = model.conversation()

response1 = conversation.prompt("My name is Alice")
print(response1.text())

response2 = conversation.prompt("What is my name?")
print(response2.text())  # Should mention Alice
```

**TypeScript**:
```typescript
import { LLM } from 'llm-ts';

const llm = new LLM();
const conversation = llm.conversation('gpt-4');

const response1 = await conversation.prompt('My name is Alice');
console.log(response1.text);

const response2 = await conversation.prompt('What is my name?');
console.log(response2.text); // Should mention Alice
```

### Example 4: Tools/Function Calling

**Python**:
```python
import llm

def get_weather(location: str) -> str:
    """Get the weather for a location"""
    return f"The weather in {location} is sunny and 72°F"

model = llm.get_model("gpt-4")
response = model.prompt(
    "What's the weather in San Francisco?",
    tools=[get_weather]
)
print(response.text())
```

**TypeScript**:
```typescript
import { LLM } from 'llm-ts';
import { tool } from 'ai';
import { z } from 'zod';

const getWeather = tool({
  description: 'Get the weather for a location',
  parameters: z.object({
    location: z.string()
  }),
  execute: async ({ location }) => {
    return `The weather in ${location} is sunny and 72°F`;
  }
});

const llm = new LLM();
const response = await llm.prompt(
  'gpt-4',
  "What's the weather in San Francisco?",
  { tools: [getWeather] }
);
console.log(response.text);
```

### Example 5: Structured Output (Schema)

**Python**:
```python
import llm
from pydantic import BaseModel

class Person(BaseModel):
    name: str
    age: int
    occupation: str

model = llm.get_model("gpt-4")
response = model.prompt(
    "Generate a person profile for Alice",
    schema=Person
)
person = response.json()
print(person.name, person.age, person.occupation)
```

**TypeScript**:
```typescript
import { LLM } from 'llm-ts';
import { z } from 'zod';

const PersonSchema = z.object({
  name: z.string(),
  age: z.number(),
  occupation: z.string()
});

const llm = new LLM();
const response = await llm.prompt(
  'gpt-4',
  'Generate a person profile for Alice',
  { schema: PersonSchema }
);

const person = response.object;
console.log(person.name, person.age, person.occupation);
```

### Example 6: Templates

**Python**:
```bash
# Create template
llm --save summarize -s "You are a concise summarizer" "Summarize: $input"

# Use template
cat article.txt | llm -t summarize
```

**TypeScript**:
```bash
# Create template
llm-ts template create summarize \
  --system "You are a concise summarizer" \
  --prompt "Summarize: \$input"

# Use template
cat article.txt | llm-ts -t summarize
```

---

## Next Steps and Recommendations

### For Learning (Recommended Starting Point)

1. **Week 1-2: Build Minimal CLI**
   - Follow "Approach 1" implementation steps
   - Focus on core functionality
   - Get hands-on with Vercel AI SDK
   - Create working prototype

2. **Week 3: Add One Advanced Feature**
   - Choose: Templates, Logging, or Tools
   - Implement fully
   - Learn the patterns

3. **Week 4: Reflect and Plan**
   - Evaluate architecture
   - Identify pain points
   - Decide on next features

### Resources and Documentation

**Vercel AI SDK**:
- Docs: https://ai-sdk.dev/docs
- GitHub: https://github.com/vercel/ai
- Examples: https://github.com/vercel/ai/tree/main/examples

**TypeScript Tools**:
- Commander.js: https://github.com/tj/commander.js
- oclif: https://oclif.io/
- better-sqlite3: https://github.com/WiseLibs/better-sqlite3
- zod: https://zod.dev/

**Reference Implementation** (LLM Python):
- GitHub: https://github.com/simonw/llm
- Docs: https://llm.datasette.io/

### Success Metrics

**Minimal CLI Success** (Week 2):
- [ ] Can execute prompts with OpenAI
- [ ] Can switch between providers
- [ ] Has basic streaming support
- [ ] Has simple key management
- [ ] Can run interactive chat

**Feature Parity Success** (Month 3):
- [ ] All major CLI commands working
- [ ] SQLite logging functional
- [ ] Templates and fragments supported
- [ ] Tool calling works
- [ ] Plugin system extensible
- [ ] Documentation complete
- [ ] Test coverage >80%

---

## Conclusion

Porting `llm` to TypeScript using the Vercel AI SDK is a substantial but achievable project. The Vercel AI SDK handles the core LLM interaction layer well, but significant custom development is needed for the CLI, persistence, templates, and plugin system.

**Key Takeaways**:

1. **Start Small**: Begin with Approach 1 (Minimal CLI) to learn and validate
2. **Leverage the SDK**: Use Vercel AI SDK for all model interactions
3. **Custom Components**: Build your own logging, templates, and plugin systems
4. **Iterate**: Add features incrementally based on learning
5. **TypeScript Advantages**: Better type safety, modern async/await patterns
6. **Challenges**: Plugin architecture, database migrations, feature parity

**Recommended First Steps**:

```bash
# 1. Create project
mkdir llm-ts && cd llm-ts
npm init -y

# 2. Install core dependencies
npm install ai @ai-sdk/openai @ai-sdk/anthropic commander typescript tsx

# 3. Create basic structure
mkdir -p src/{commands,lib,types}
touch src/index.ts src/cli.ts

# 4. Start coding!
# Begin with basic prompt execution
```

This is an excellent learning project that will teach you:
- TypeScript best practices
- CLI application design
- Working with AI SDKs
- Database design and migrations
- Plugin architectures
- Async programming patterns

Good luck with your learning journey!
