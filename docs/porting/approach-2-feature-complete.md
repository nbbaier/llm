# Approach 2: Feature-Complete Port - Implementation Plan

## Overview

**Goal**: Create a production-ready TypeScript CLI with near-complete feature parity with the Python `llm` library.

**Timeline**: 10-12 weeks (400-500 hours)

**Outcome**: A fully-featured, extensible CLI tool with plugin architecture, SQLite logging, templates, embeddings, and all major features from the Python version.

**When to Choose This Approach**:
- You want production-ready software
- You need plugin extensibility
- You plan to use this long-term
- You want to learn advanced TypeScript patterns
- You have 2-3 months to dedicate

---

## Table of Contents

1. [Project Architecture](#project-architecture)
2. [Development Phases](#development-phases)
3. [Core Components](#core-components)
4. [Implementation Timeline](#implementation-timeline)
5. [Success Metrics](#success-metrics)
6. [Architecture Decisions](#architecture-decisions)

---

## Project Architecture

### High-Level Structure

```
llm-ts/
├── packages/                    # Monorepo structure (optional)
│   └── llm/
│       ├── src/
│       │   ├── cli/            # CLI layer
│       │   │   ├── commands/   # Command implementations
│       │   │   ├── index.ts
│       │   │   └── middleware/ # CLI middleware
│       │   ├── core/           # Core abstractions
│       │   │   ├── model.ts
│       │   │   ├── response.ts
│       │   │   ├── conversation.ts
│       │   │   ├── prompt.ts
│       │   │   └── registry.ts
│       │   ├── providers/      # Provider implementations
│       │   │   ├── base.ts
│       │   │   ├── openai.ts
│       │   │   ├── anthropic.ts
│       │   │   └── google.ts
│       │   ├── plugins/        # Plugin system
│       │   │   ├── manager.ts
│       │   │   ├── loader.ts
│       │   │   └── hooks.ts
│       │   ├── db/             # Database layer
│       │   │   ├── logs.ts
│       │   │   ├── embeddings.ts
│       │   │   ├── migrations/
│       │   │   └── schema.ts
│       │   ├── templates/      # Template engine
│       │   │   ├── manager.ts
│       │   │   ├── renderer.ts
│       │   │   └── loaders/
│       │   ├── tools/          # Tool system
│       │   │   ├── registry.ts
│       │   │   ├── executor.ts
│       │   │   └── schema.ts
│       │   ├── embeddings/     # Embedding functionality
│       │   │   ├── models.ts
│       │   │   ├── collection.ts
│       │   │   └── search.ts
│       │   ├── config/         # Configuration
│       │   │   ├── manager.ts
│       │   │   ├── keys.ts
│       │   │   └── aliases.ts
│       │   └── utils/          # Utilities
│       │       ├── errors.ts
│       │       ├── validation.ts
│       │       └── helpers.ts
│       ├── tests/
│       ├── package.json
│       └── tsconfig.json
├── docs/
├── examples/
└── README.md
```

### Core Abstractions

```typescript
// Core Model Hierarchy
abstract class BaseModel {
  abstract id: string;
  abstract canStream: boolean;
  abstract supportedFeatures: ModelFeatures;

  abstract execute(
    prompt: Prompt,
    options: ExecuteOptions
  ): Promise<Response> | AsyncGenerator<string>;
}

interface ModelFeatures {
  streaming: boolean;
  tools: boolean;
  schemas: boolean;
  attachments: string[]; // ['image', 'audio', 'video']
  maxTokens?: number;
}

// Prompt abstraction
class Prompt {
  text: string;
  system?: string;
  attachments: Attachment[];
  fragments: Fragment[];
  tools: Tool[];
  schema?: JSONSchema;
  options: Record<string, any>;
}

// Response handling
class Response {
  text: string;
  model: string;
  conversationId?: string;
  usage?: Usage;
  toolCalls?: ToolCall[];

  async log(db: Database): Promise<void>;
  json<T>(): T; // For schema-based responses
}

// Conversation management
class Conversation {
  id: string;
  messages: Message[];
  model: string;

  async prompt(text: string): Promise<Response>;
  async load(db: Database): Promise<void>;
  async save(db: Database): Promise<void>;
}
```

---

## Development Phases

### Phase 1: Foundation (Weeks 1-3)

**Goal**: Establish core architecture and basic functionality.

#### Week 1: Project Setup & Core Abstractions

**Days 1-2: Project Scaffolding**
- Set up monorepo structure (optional: use Turborepo or nx)
- Configure TypeScript with strict mode
- Set up testing framework (Vitest)
- Configure build tools (tsup or tsc)
- Set up linting (ESLint) and formatting (Prettier)

**Dependencies**:
```json
{
  "dependencies": {
    "ai": "^3.0.0",
    "@ai-sdk/openai": "^0.0.x",
    "@ai-sdk/anthropic": "^0.0.x",
    "@ai-sdk/google": "^0.0.x",
    "commander": "^11.0.0",
    "chalk": "^5.0.0",
    "better-sqlite3": "^9.0.0",
    "zod": "^3.22.0",
    "yaml": "^2.3.0",
    "ora": "^7.0.0",
    "inquirer": "^9.0.0"
  },
  "devDependencies": {
    "typescript": "^5.3.0",
    "@types/node": "^20.0.0",
    "@types/better-sqlite3": "^7.6.0",
    "vitest": "^1.0.0",
    "tsx": "^4.0.0",
    "tsup": "^8.0.0"
  }
}
```

**Days 3-5: Core Model Abstractions**

Create the base model system:

```typescript
// src/core/model.ts
export interface ModelConfig {
  id: string;
  provider: string;
  supportsStreaming: boolean;
  supportsTools: boolean;
  supportsSchemas: boolean;
  supportedAttachmentTypes: string[];
}

export abstract class BaseModel {
  constructor(
    public readonly config: ModelConfig,
    protected apiKey?: string
  ) {}

  abstract execute(
    prompt: Prompt,
    options: ExecuteOptions
  ): Promise<Response> | AsyncGenerator<string>;

  async prompt(text: string, options?: PromptOptions): Promise<Response> {
    const prompt = new Prompt(text, options);
    return this.execute(prompt, options || {});
  }

  async *stream(text: string, options?: PromptOptions): AsyncGenerator<string> {
    if (!this.config.supportsStreaming) {
      throw new Error(`${this.config.id} does not support streaming`);
    }

    const prompt = new Prompt(text, options);
    yield* this.execute(prompt, { ...options, stream: true }) as AsyncGenerator<string>;
  }
}

// src/core/registry.ts
export class ModelRegistry {
  private models = new Map<string, ModelFactory>();
  private aliases = new Map<string, string>();

  register(id: string, factory: ModelFactory): void {
    this.models.set(id, factory);
  }

  alias(alias: string, modelId: string): void {
    this.aliases.set(alias, modelId);
  }

  get(id: string): BaseModel {
    const modelId = this.aliases.get(id) || id;
    const factory = this.models.get(modelId);

    if (!factory) {
      throw new ModelNotFoundError(id);
    }

    return factory.create();
  }

  list(): ModelConfig[] {
    return Array.from(this.models.values()).map(f => f.config);
  }
}
```

**Days 6-7: Response & Conversation Abstractions**

```typescript
// src/core/response.ts
export class Response {
  constructor(
    public text: string,
    public model: string,
    public metadata: ResponseMetadata
  ) {}

  static fromAISDK(result: GenerateTextResult, model: string): Response {
    return new Response(
      result.text,
      model,
      {
        usage: result.usage,
        finishReason: result.finishReason,
        timestamp: Date.now()
      }
    );
  }

  async log(db: Database): Promise<string> {
    const id = ulid();
    await db.insert('responses', {
      id,
      model: this.model,
      text: this.text,
      ...this.metadata
    });
    return id;
  }

  json<T>(): T {
    if (!this.metadata.schema) {
      throw new Error('No schema defined for this response');
    }
    return JSON.parse(this.text) as T;
  }
}

// src/core/conversation.ts
export class Conversation {
  private messages: Message[] = [];

  constructor(
    public id: string,
    public model: string,
    systemPrompt?: string
  ) {
    if (systemPrompt) {
      this.messages.push({
        role: 'system',
        content: systemPrompt
      });
    }
  }

  async prompt(text: string, model: BaseModel): Promise<Response> {
    this.messages.push({
      role: 'user',
      content: text
    });

    const response = await model.execute(
      new Prompt(text, { messages: this.messages }),
      {}
    );

    this.messages.push({
      role: 'assistant',
      content: response.text
    });

    return response;
  }

  async save(db: Database): Promise<void> {
    await db.transaction(async (tx) => {
      // Save conversation metadata
      await tx.insert('conversations', {
        id: this.id,
        model: this.model,
        created_at: Date.now()
      });

      // Save all messages
      for (const msg of this.messages) {
        await tx.insert('messages', {
          conversation_id: this.id,
          role: msg.role,
          content: msg.content
        });
      }
    });
  }

  static async load(db: Database, id: string): Promise<Conversation> {
    const conv = await db.get('conversations', { id });
    const messages = await db.getAll('messages', { conversation_id: id });

    const conversation = new Conversation(id, conv.model);
    conversation.messages = messages;

    return conversation;
  }
}
```

#### Week 2: Database Layer

**Database Schema Design**:

```sql
-- Conversations
CREATE TABLE conversations (
  id TEXT PRIMARY KEY,
  name TEXT,
  model TEXT NOT NULL,
  created_at INTEGER NOT NULL,
  updated_at INTEGER NOT NULL
);

-- Responses (prompt/response pairs)
CREATE TABLE responses (
  id TEXT PRIMARY KEY,
  conversation_id TEXT,
  prompt TEXT NOT NULL,
  response TEXT NOT NULL,
  model TEXT NOT NULL,
  system_prompt TEXT,
  input_tokens INTEGER,
  output_tokens INTEGER,
  total_tokens INTEGER,
  created_at INTEGER NOT NULL,
  FOREIGN KEY (conversation_id) REFERENCES conversations(id)
);

-- Attachments
CREATE TABLE attachments (
  id TEXT PRIMARY KEY,
  response_id TEXT NOT NULL,
  type TEXT NOT NULL,
  path TEXT,
  url TEXT,
  content BLOB,
  FOREIGN KEY (response_id) REFERENCES responses(id)
);

-- Fragments
CREATE TABLE fragments (
  id TEXT PRIMARY KEY,
  content TEXT NOT NULL,
  created_at INTEGER NOT NULL
);

CREATE TABLE fragment_aliases (
  alias TEXT PRIMARY KEY,
  fragment_id TEXT NOT NULL,
  FOREIGN KEY (fragment_id) REFERENCES fragments(id)
);

-- Tools
CREATE TABLE tools (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL UNIQUE,
  description TEXT,
  input_schema TEXT NOT NULL,
  plugin TEXT
);

CREATE TABLE tool_calls (
  id TEXT PRIMARY KEY,
  response_id TEXT NOT NULL,
  tool_id TEXT NOT NULL,
  arguments TEXT NOT NULL,
  result TEXT,
  error TEXT,
  executed_at INTEGER NOT NULL,
  FOREIGN KEY (response_id) REFERENCES responses(id),
  FOREIGN KEY (tool_id) REFERENCES tools(id)
);

-- Schemas
CREATE TABLE schemas (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL UNIQUE,
  schema TEXT NOT NULL,
  created_at INTEGER NOT NULL
);

-- Embeddings
CREATE TABLE collections (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL UNIQUE,
  model TEXT NOT NULL,
  dimension INTEGER NOT NULL,
  created_at INTEGER NOT NULL
);

CREATE TABLE embeddings (
  id TEXT PRIMARY KEY,
  collection_id TEXT NOT NULL,
  embedding BLOB NOT NULL,
  content TEXT,
  metadata TEXT,
  created_at INTEGER NOT NULL,
  FOREIGN KEY (collection_id) REFERENCES collections(id)
);

CREATE INDEX idx_embeddings_collection ON embeddings(collection_id);
```

**Database Implementation**:

```typescript
// src/db/database.ts
import Database from 'better-sqlite3';
import { Migration } from './migrations';

export class LLMDatabase {
  private db: Database.Database;

  constructor(path: string) {
    this.db = new Database(path);
    this.db.pragma('journal_mode = WAL');
    this.db.pragma('foreign_keys = ON');
    this.runMigrations();
  }

  private runMigrations(): void {
    const migrations = this.getMigrations();
    const currentVersion = this.getSchemaVersion();

    for (let i = currentVersion; i < migrations.length; i++) {
      migrations[i].up(this.db);
      this.setSchemaVersion(i + 1);
    }
  }

  // Generic query methods
  query<T>(sql: string, params?: any[]): T[] {
    return this.db.prepare(sql).all(params) as T[];
  }

  get<T>(sql: string, params?: any[]): T | undefined {
    return this.db.prepare(sql).get(params) as T | undefined;
  }

  run(sql: string, params?: any[]): Database.RunResult {
    return this.db.prepare(sql).run(params);
  }

  transaction<T>(fn: () => T): T {
    return this.db.transaction(fn)();
  }

  // Specific methods for logging
  async logResponse(response: Response, prompt: string): Promise<string> {
    const id = ulid();

    this.run(`
      INSERT INTO responses (
        id, prompt, response, model,
        input_tokens, output_tokens, total_tokens, created_at
      ) VALUES (?, ?, ?, ?, ?, ?, ?, ?)
    `, [
      id,
      prompt,
      response.text,
      response.model,
      response.metadata.usage?.promptTokens,
      response.metadata.usage?.completionTokens,
      response.metadata.usage?.totalTokens,
      Date.now()
    ]);

    return id;
  }

  // More specific methods...
}
```

#### Week 3: Provider Integration

**Provider Base Class**:

```typescript
// src/providers/base.ts
export abstract class Provider {
  abstract name: string;
  abstract models: ModelConfig[];

  abstract createModel(
    modelId: string,
    apiKey: string
  ): BaseModel;

  supportsModel(modelId: string): boolean {
    return this.models.some(m => m.id === modelId);
  }
}

// src/providers/openai.ts
export class OpenAIProvider extends Provider {
  name = 'OpenAI';
  models = [
    {
      id: 'gpt-4o',
      provider: 'openai',
      supportsStreaming: true,
      supportsTools: true,
      supportsSchemas: true,
      supportedAttachmentTypes: ['image']
    },
    // ... more models
  ];

  createModel(modelId: string, apiKey: string): OpenAIModel {
    return new OpenAIModel(modelId, apiKey);
  }
}

class OpenAIModel extends BaseModel {
  async execute(prompt: Prompt, options: ExecuteOptions): Promise<Response> {
    const model = openai(this.config.id, { apiKey: this.apiKey });

    if (options.stream) {
      return this.streamResponse(model, prompt, options);
    }

    const result = await generateText({
      model,
      prompt: prompt.text,
      system: prompt.system,
      tools: this.convertTools(prompt.tools),
      maxTokens: options.maxTokens,
      temperature: options.temperature
    });

    return Response.fromAISDK(result, this.config.id);
  }

  private async *streamResponse(
    model: any,
    prompt: Prompt,
    options: ExecuteOptions
  ): AsyncGenerator<string> {
    const { textStream } = await streamText({
      model,
      prompt: prompt.text,
      system: prompt.system
    });

    for await (const chunk of textStream) {
      yield chunk;
    }
  }

  private convertTools(tools: Tool[]): Record<string, any> {
    // Convert our Tool format to Vercel AI SDK format
    return tools.reduce((acc, tool) => {
      acc[tool.name] = {
        description: tool.description,
        parameters: tool.inputSchema,
        execute: tool.implementation
      };
      return acc;
    }, {});
  }
}
```

**Success Criteria for Phase 1**:
- [ ] Project structure established
- [ ] Core abstractions implemented
- [ ] Database schema created and migrations working
- [ ] 3 providers integrated (OpenAI, Anthropic, Google)
- [ ] Basic tests passing (>60% coverage)
- [ ] Can execute prompts and log to database

---

### Phase 2: Core Features (Weeks 4-7)

#### Week 4: CLI Framework

**Command Structure**:

```typescript
// src/cli/index.ts
import { Command } from 'commander';

export function createCLI(registry: ModelRegistry, db: LLMDatabase): Command {
  const program = new Command()
    .name('llm')
    .description('CLI for interacting with Large Language Models')
    .version('1.0.0');

  // Register all commands
  program.addCommand(createPromptCommand(registry, db));
  program.addCommand(createChatCommand(registry, db));
  program.addCommand(createKeysCommand());
  program.addCommand(createLogsCommand(db));
  program.addCommand(createModelsCommand(registry));
  program.addCommand(createTemplatesCommand(db));
  program.addCommand(createToolsCommand(db));
  program.addCommand(createEmbedCommand(db));

  return program;
}

// src/cli/commands/prompt.ts
export function createPromptCommand(
  registry: ModelRegistry,
  db: LLMDatabase
): Command {
  return new Command('prompt')
    .argument('[text]', 'Prompt text')
    .option('-m, --model <model>', 'Model to use', 'gpt-4o-mini')
    .option('-s, --system <text>', 'System prompt')
    .option('--no-stream', 'Disable streaming')
    .option('-t, --template <name>', 'Use template')
    .option('--schema <name>', 'Use schema for structured output')
    .option('--tools <names...>', 'Enable tools')
    .option('--no-log', 'Disable logging')
    .action(async (text, options) => {
      await executePrompt(text, options, registry, db);
    });
}
```

#### Week 5: Template System

**Template Engine**:

```typescript
// src/templates/manager.ts
export interface Template {
  name: string;
  prompt?: string;
  system?: string;
  model?: string;
  attachments?: string[];
  fragments?: string[];
  tools?: string[];
  schema?: string;
  options?: Record<string, any>;
  defaults?: Record<string, any>;
}

export class TemplateManager {
  constructor(
    private templatesDir: string,
    private db: LLMDatabase
  ) {}

  load(name: string): Template {
    // Try file system first
    const filePath = join(this.templatesDir, `${name}.yaml`);
    if (existsSync(filePath)) {
      return this.loadFromFile(filePath);
    }

    // Try database
    return this.loadFromDB(name);
  }

  private loadFromFile(path: string): Template {
    const content = readFileSync(path, 'utf-8');
    return yaml.parse(content) as Template;
  }

  render(template: Template, variables: Record<string, any>): string {
    let prompt = template.prompt || '';

    // Replace variables
    for (const [key, value] of Object.entries(variables)) {
      const regex = new RegExp(`\\$${key}\\b`, 'g');
      prompt = prompt.replace(regex, String(value));
    }

    // Check for missing required variables
    const missingVars = prompt.match(/\$\w+/g);
    if (missingVars) {
      throw new Error(`Missing template variables: ${missingVars.join(', ')}`);
    }

    return prompt;
  }

  async save(template: Template): Promise<void> {
    const path = join(this.templatesDir, `${template.name}.yaml`);
    const content = yaml.stringify(template);
    writeFileSync(path, content);
  }

  list(): string[] {
    const files = readdirSync(this.templatesDir)
      .filter(f => f.endsWith('.yaml'))
      .map(f => f.replace('.yaml', ''));
    return files;
  }
}
```

**Template Usage**:

```typescript
// Usage in CLI
const templateManager = new TemplateManager(templatesDir, db);
const template = templateManager.load('summarize');

// Render with variables
const prompt = templateManager.render(template, {
  input: stdinContent,
  max_words: 100
});

// Execute with template settings
const model = registry.get(template.model || options.model);
const response = await model.prompt(prompt, {
  system: template.system,
  tools: template.tools,
  ...template.options
});
```

#### Week 6: Tool System

**Tool Implementation**:

```typescript
// src/tools/registry.ts
export class ToolRegistry {
  private tools = new Map<string, Tool>();

  register(tool: Tool): void {
    this.tools.set(tool.name, tool);
  }

  get(name: string): Tool | undefined {
    return this.tools.get(name);
  }

  getMany(names: string[]): Tool[] {
    return names.map(name => {
      const tool = this.get(name);
      if (!tool) {
        throw new Error(`Tool not found: ${name}`);
      }
      return tool;
    });
  }

  list(): Tool[] {
    return Array.from(this.tools.values());
  }

  // Create tool from function
  static fromFunction(fn: Function): Tool {
    const schema = this.generateSchema(fn);

    return {
      name: fn.name,
      description: fn.toString().match(/\/\*\*(.*?)\*\//s)?.[1].trim() || '',
      inputSchema: schema,
      implementation: fn
    };
  }

  private static generateSchema(fn: Function): JSONSchema {
    // Use TypeScript reflection or runtime inspection
    // to generate JSON schema from function signature
    const params = this.extractParameters(fn);

    return {
      type: 'object',
      properties: params.reduce((acc, p) => {
        acc[p.name] = {
          type: p.type,
          description: p.description
        };
        return acc;
      }, {}),
      required: params.filter(p => p.required).map(p => p.name)
    };
  }
}

// src/tools/executor.ts
export class ToolExecutor {
  constructor(
    private registry: ToolRegistry,
    private db: LLMDatabase
  ) {}

  async execute(
    toolCall: ToolCall,
    responseId: string
  ): Promise<ToolResult> {
    const tool = this.registry.get(toolCall.name);
    if (!tool) {
      throw new Error(`Tool not found: ${toolCall.name}`);
    }

    const startTime = Date.now();
    let result: any;
    let error: Error | undefined;

    try {
      result = await tool.implementation(toolCall.arguments);
    } catch (e) {
      error = e as Error;
    }

    // Log tool execution
    await this.db.run(`
      INSERT INTO tool_calls (
        id, response_id, tool_id, arguments, result, error, executed_at
      ) VALUES (?, ?, ?, ?, ?, ?, ?)
    `, [
      ulid(),
      responseId,
      tool.name,
      JSON.stringify(toolCall.arguments),
      result ? JSON.stringify(result) : null,
      error?.message,
      startTime
    ]);

    return {
      success: !error,
      result,
      error: error?.message,
      duration: Date.now() - startTime
    };
  }

  async executeChain(
    model: BaseModel,
    prompt: string,
    tools: Tool[],
    maxIterations: number = 10
  ): Promise<Response> {
    let currentPrompt = prompt;
    let iteration = 0;

    while (iteration < maxIterations) {
      const response = await model.prompt(currentPrompt, {
        tools
      });

      if (!response.metadata.toolCalls?.length) {
        return response;
      }

      // Execute all tool calls
      const results = await Promise.all(
        response.metadata.toolCalls.map(tc =>
          this.execute(tc, response.id)
        )
      );

      // Build next prompt with tool results
      currentPrompt = this.buildFollowUpPrompt(response, results);
      iteration++;
    }

    throw new Error(`Tool chain exceeded maximum iterations: ${maxIterations}`);
  }
}
```

#### Week 7: Structured Output (Schemas)

```typescript
// src/schemas/manager.ts
export class SchemaManager {
  constructor(private db: LLMDatabase) {}

  async save(name: string, schema: JSONSchema): Promise<void> {
    await this.db.run(`
      INSERT OR REPLACE INTO schemas (id, name, schema, created_at)
      VALUES (?, ?, ?, ?)
    `, [ulid(), name, JSON.stringify(schema), Date.now()]);
  }

  async get(name: string): Promise<JSONSchema | undefined> {
    const row = await this.db.get<{ schema: string }>(
      'SELECT schema FROM schemas WHERE name = ?',
      [name]
    );

    return row ? JSON.parse(row.schema) : undefined;
  }

  // Generate schema from Zod schema
  fromZod(zodSchema: z.ZodType): JSONSchema {
    return zodToJsonSchema(zodSchema);
  }

  // Parse and validate response against schema
  validate<T>(response: Response, schema: JSONSchema): T {
    const data = JSON.parse(response.text);

    // Validate using ajv or similar
    const valid = this.validator.validate(schema, data);
    if (!valid) {
      throw new ValidationError(this.validator.errors);
    }

    return data as T;
  }
}

// Usage with Vercel AI SDK
const schema = z.object({
  name: z.string(),
  age: z.number(),
  occupation: z.string()
});

const { object } = await generateObject({
  model: openai('gpt-4'),
  schema,
  prompt: 'Generate a person profile'
});

// Our wrapper
const response = await model.prompt('Generate a person profile', {
  schema: schemaManager.fromZod(schema)
});

const person = schemaManager.validate<Person>(response, schema);
```

**Success Criteria for Phase 2**:
- [ ] Full CLI with all major commands
- [ ] Template system working (save, load, render)
- [ ] Tool registration and execution
- [ ] Structured output with schemas
- [ ] Comprehensive logging
- [ ] Tests covering all features (>70% coverage)

---

### Phase 3: Advanced Features (Weeks 8-10)

#### Week 8: Plugin System

**Plugin Architecture**:

```typescript
// src/plugins/types.ts
export interface Plugin {
  name: string;
  version: string;
  description?: string;

  // Hooks
  registerModels?(registry: ModelRegistry): void;
  registerTools?(registry: ToolRegistry): void;
  registerTemplateLoaders?(manager: TemplateManager): void;
  registerCommands?(cli: Command): void;
}

// src/plugins/manager.ts
export class PluginManager {
  private plugins = new Map<string, Plugin>();
  private loadedPlugins = new Set<string>();

  async load(nameOrPath: string): Promise<void> {
    let plugin: Plugin;

    if (nameOrPath.startsWith('.') || nameOrPath.startsWith('/')) {
      // Load from file path
      plugin = await import(nameOrPath);
    } else {
      // Load from node_modules
      plugin = await import(nameOrPath);
    }

    this.plugins.set(plugin.name, plugin);
  }

  async initialize(
    registry: ModelRegistry,
    toolRegistry: ToolRegistry,
    templateManager: TemplateManager,
    cli: Command
  ): Promise<void> {
    for (const plugin of this.plugins.values()) {
      if (this.loadedPlugins.has(plugin.name)) {
        continue;
      }

      // Call plugin hooks
      plugin.registerModels?.(registry);
      plugin.registerTools?.(toolRegistry);
      plugin.registerTemplateLoaders?.(templateManager);
      plugin.registerCommands?.(cli);

      this.loadedPlugins.add(plugin.name);
    }
  }

  list(): Array<{ name: string; version: string; description?: string }> {
    return Array.from(this.plugins.values()).map(p => ({
      name: p.name,
      version: p.version,
      description: p.description
    }));
  }
}

// Example plugin
export const myPlugin: Plugin = {
  name: 'my-plugin',
  version: '1.0.0',
  description: 'Example plugin',

  registerModels(registry) {
    registry.register('my-model', new MyModelFactory());
  },

  registerTools(registry) {
    registry.register(myTool);
  }
};
```

**Plugin Discovery**:

```typescript
// Auto-discover plugins from package.json
async function discoverPlugins(packagePath: string): Promise<string[]> {
  const pkg = JSON.parse(readFileSync(packagePath, 'utf-8'));
  const deps = { ...pkg.dependencies, ...pkg.devDependencies };

  return Object.keys(deps).filter(name => name.startsWith('llm-plugin-'));
}
```

#### Week 9: Embeddings & Vector Search

```typescript
// src/embeddings/collection.ts
export class Collection {
  constructor(
    public id: string,
    public name: string,
    public model: string,
    public dimension: number,
    private db: LLMDatabase
  ) {}

  async embed(id: string, content: string, metadata?: any): Promise<void> {
    const embedding = await this.getEmbedding(content);

    await this.db.run(`
      INSERT INTO embeddings (id, collection_id, embedding, content, metadata, created_at)
      VALUES (?, ?, ?, ?, ?, ?)
    `, [
      id,
      this.id,
      this.serializeEmbedding(embedding),
      content,
      JSON.stringify(metadata),
      Date.now()
    ]);
  }

  async embedMulti(entries: Array<{ id: string; content: string; metadata?: any }>): Promise<void> {
    const batchSize = 100;

    for (let i = 0; i < entries.length; i += batchSize) {
      const batch = entries.slice(i, i + batchSize);
      const embeddings = await this.getEmbeddings(batch.map(e => e.content));

      await this.db.transaction(() => {
        for (let j = 0; j < batch.length; j++) {
          this.db.run(`
            INSERT INTO embeddings (id, collection_id, embedding, content, metadata, created_at)
            VALUES (?, ?, ?, ?, ?, ?)
          `, [
            batch[j].id,
            this.id,
            this.serializeEmbedding(embeddings[j]),
            batch[j].content,
            JSON.stringify(batch[j].metadata),
            Date.now()
          ]);
        }
      });
    }
  }

  async similar(query: string, limit: number = 10): Promise<SimilarResult[]> {
    const queryEmbedding = await this.getEmbedding(query);

    const embeddings = this.db.query<{ id: string; embedding: Buffer; content: string }>(
      'SELECT id, embedding, content FROM embeddings WHERE collection_id = ?',
      [this.id]
    );

    // Calculate cosine similarity
    const results = embeddings.map(e => ({
      id: e.id,
      content: e.content,
      similarity: this.cosineSimilarity(
        queryEmbedding,
        this.deserializeEmbedding(e.embedding)
      )
    }));

    return results
      .sort((a, b) => b.similarity - a.similarity)
      .slice(0, limit);
  }

  private async getEmbedding(text: string): Promise<number[]> {
    // Use OpenAI embeddings or other provider
    const { embedding } = await embed({
      model: openai.embedding('text-embedding-3-small'),
      value: text
    });
    return embedding;
  }

  private cosineSimilarity(a: number[], b: number[]): number {
    const dotProduct = a.reduce((sum, val, i) => sum + val * b[i], 0);
    const magnitudeA = Math.sqrt(a.reduce((sum, val) => sum + val * val, 0));
    const magnitudeB = Math.sqrt(b.reduce((sum, val) => sum + val * val, 0));
    return dotProduct / (magnitudeA * magnitudeB);
  }
}
```

#### Week 10: Fragments & Multi-modal

**Fragment System**:

```typescript
// src/fragments/manager.ts
export class FragmentManager {
  constructor(private db: LLMDatabase) {}

  async save(content: string, alias?: string): Promise<string> {
    const id = ulid();

    await this.db.run(
      'INSERT INTO fragments (id, content, created_at) VALUES (?, ?, ?)',
      [id, content, Date.now()]
    );

    if (alias) {
      await this.alias(id, alias);
    }

    return id;
  }

  async alias(fragmentId: string, alias: string): Promise<void> {
    await this.db.run(
      'INSERT OR REPLACE INTO fragment_aliases (alias, fragment_id) VALUES (?, ?)',
      [alias, fragmentId]
    );
  }

  async get(idOrAlias: string): Promise<string | undefined> {
    // Try as ID first
    let row = await this.db.get<{ content: string }>(
      'SELECT content FROM fragments WHERE id = ?',
      [idOrAlias]
    );

    if (!row) {
      // Try as alias
      row = await this.db.get<{ content: string }>(
        `SELECT f.content FROM fragments f
         JOIN fragment_aliases fa ON f.id = fa.fragment_id
         WHERE fa.alias = ?`,
        [idOrAlias]
      );
    }

    return row?.content;
  }

  async load(prefix: string, identifier: string): Promise<string> {
    // Support plugin-provided fragment loaders
    const loader = this.loaders.get(prefix);
    if (loader) {
      return loader.load(identifier);
    }

    throw new Error(`Unknown fragment prefix: ${prefix}`);
  }
}

// Usage in prompts
const fragments = await Promise.all(
  fragmentIds.map(id => fragmentManager.get(id))
);

const fullPrompt = `
${prompt}

Context:
${fragments.join('\n\n')}
`;
```

**Multi-modal Support**:

```typescript
// src/core/attachment.ts
export class Attachment {
  constructor(
    public type: string,
    public data: Buffer | string,
    public metadata?: Record<string, any>
  ) {}

  static fromFile(path: string): Attachment {
    const data = readFileSync(path);
    const type = this.detectType(path);
    return new Attachment(type, data, { path });
  }

  static fromURL(url: string): Attachment {
    return new Attachment('url', url, { url });
  }

  toBase64(): string {
    if (Buffer.isBuffer(this.data)) {
      return this.data.toString('base64');
    }
    return this.data;
  }

  async save(db: LLMDatabase, responseId: string): Promise<void> {
    await db.run(`
      INSERT INTO attachments (id, response_id, type, content, created_at)
      VALUES (?, ?, ?, ?, ?)
    `, [
      ulid(),
      responseId,
      this.type,
      this.data,
      Date.now()
    ]);
  }
}
```

**Success Criteria for Phase 3**:
- [ ] Plugin system working with example plugins
- [ ] Embeddings and similarity search functional
- [ ] Fragment management complete
- [ ] Multi-modal attachments supported
- [ ] All advanced features tested

---

### Phase 4: Polish & Production (Weeks 11-12)

#### Week 11: Testing & Documentation

**Comprehensive Testing**:

```typescript
// tests/integration/full-workflow.test.ts
describe('Full Workflow Integration', () => {
  it('should handle complete prompt-to-log workflow', async () => {
    const db = new LLMDatabase(':memory:');
    const registry = new ModelRegistry();

    // Register mock model
    registry.register('test-model', new MockModelFactory());

    // Execute prompt
    const model = registry.get('test-model');
    const response = await model.prompt('Test prompt');

    // Log response
    const id = await response.log(db);

    // Verify logged
    const logged = await db.get('SELECT * FROM responses WHERE id = ?', [id]);
    expect(logged).toBeDefined();
    expect(logged.response).toBe(response.text);
  });

  it('should handle conversation with tool calls', async () => {
    // ... comprehensive test
  });

  it('should handle template rendering and execution', async () => {
    // ... template test
  });
});

// tests/e2e/cli.test.ts
describe('CLI End-to-End', () => {
  it('should execute prompt via CLI', async () => {
    const result = await exec('llm "test prompt" --no-log');
    expect(result.stdout).toContain('response');
  });

  it('should use template via CLI', async () => {
    // Save template
    await exec('llm templates save test --prompt "Test: $input"');

    // Use template
    const result = await exec('echo "hello" | llm -t test');
    expect(result.stdout).toBeTruthy();
  });
});
```

**Documentation**:

- API documentation (TypeDoc)
- CLI reference (auto-generated from Commander)
- Plugin development guide
- Migration guide from Python version
- Architecture decision records (ADRs)

#### Week 12: Performance & Release

**Performance Optimization**:

```typescript
// Caching layer
export class ModelCache {
  private cache = new LRU<string, Response>({ max: 100 });

  async get(key: string): Promise<Response | undefined> {
    return this.cache.get(key);
  }

  async set(key: string, value: Response): Promise<void> {
    this.cache.set(key, value);
  }

  generateKey(prompt: Prompt, model: string): string {
    return createHash('sha256')
      .update(JSON.stringify({ prompt, model }))
      .digest('hex');
  }
}

// Connection pooling for database
export class DatabasePool {
  private connections: Database[] = [];
  private maxConnections = 5;

  getConnection(): Database {
    // Pool management logic
  }
}
```

**Release Checklist**:
- [ ] All tests passing
- [ ] Documentation complete
- [ ] Performance benchmarks met
- [ ] Security audit passed
- [ ] Cross-platform tested (Windows, macOS, Linux)
- [ ] Published to npm
- [ ] GitHub release with binaries
- [ ] Migration guide published

---

## Success Metrics

### Quantitative Metrics

**Functionality** (40%):
- [ ] (10%) Core commands work (prompt, chat, keys, models)
- [ ] (10%) Advanced features work (templates, tools, schemas)
- [ ] (10%) Plugin system functional with 2+ example plugins
- [ ] (10%) Embeddings and vector search working

**Quality** (30%):
- [ ] (10%) Test coverage >80%
- [ ] (10%) Zero critical bugs
- [ ] (10%) Performance: <100ms startup time, <50ms per command overhead

**Usability** (30%):
- [ ] (10%) Complete documentation
- [ ] (10%) Error messages are helpful
- [ ] (10%) Installation is smooth (<5 minutes)

### Qualitative Metrics

- Can perform all tasks that Python version can
- Plugin development is straightforward
- Community adoption (GitHub stars, downloads)
- Positive user feedback

---

## Architecture Decisions

### Why This Approach?

**Strengths**:
- Production-ready architecture
- Highly extensible via plugins
- Complete feature parity with Python version
- TypeScript brings type safety
- Modern async patterns
- Better cross-platform support

**Trade-offs**:
- Longer development time (10-12 weeks vs 2 weeks)
- More complex architecture
- Larger codebase to maintain
- Steeper learning curve for contributors

### Key Architectural Choices

**1. Monorepo vs Single Package**

Decision: Start with single package, can split later if needed.

Rationale: Simpler to develop and maintain initially. Can extract packages (e.g., `@llm-ts/core`, `@llm-ts/cli`) later if the project grows.

**2. Database: better-sqlite3 vs Prisma**

Decision: Use better-sqlite3 directly.

Rationale:
- Direct SQL gives more control
- Better performance
- No ORM overhead
- Simpler for CLI tool

**3. Plugin System: Custom vs Existing**

Decision: Build custom plugin system.

Rationale:
- Full control over plugin API
- Simpler than alternatives (like webpack's tapable)
- Can optimize for our use case
- Easier to understand for contributors

**4. Testing Strategy**

Approach:
- Unit tests for all core classes
- Integration tests for workflows
- E2E tests for CLI commands
- Mock external APIs for consistent tests

**5. Error Handling**

Strategy:
- Custom error classes with codes
- Structured error responses
- User-friendly messages
- Debug mode for detailed errors

---

## Risk Management

### Technical Risks

**Risk: Vercel AI SDK API changes**
- Mitigation: Pin versions, monitor changelog, abstraction layer

**Risk: SQLite performance with large datasets**
- Mitigation: Proper indexing, connection pooling, chunking

**Risk: Plugin system complexity**
- Mitigation: Start simple, iterate based on needs

### Schedule Risks

**Risk: Scope creep**
- Mitigation: Strict adherence to MVP features, track "nice-to-haves"

**Risk: Dependencies blocking progress**
- Mitigation: Identify critical path, have fallback plans

### People Risks

**Risk: Solo development burnout**
- Mitigation: Set realistic timelines, take breaks, celebrate milestones

---

## Comparison with Other Approaches

| Feature | Approach 1 (Minimal) | Approach 2 (Complete) | Python llm |
|---------|---------------------|----------------------|------------|
| Timeline | 2 weeks | 10-12 weeks | N/A |
| Features | 30% | 95% | 100% |
| Plugin System | ❌ | ✅ | ✅ |
| Logging | ❌ | ✅ | ✅ |
| Templates | ❌ | ✅ | ✅ |
| Embeddings | ❌ | ✅ | ✅ |
| Production Ready | ⚠️ | ✅ | ✅ |
| Learning Value | High | Very High | N/A |

---

## Next Steps After Completion

1. **Community Engagement**: Share on social media, gather feedback
2. **Plugin Ecosystem**: Encourage community plugins
3. **Performance Tuning**: Profile and optimize bottlenecks
4. **Feature Expansion**: Add features based on user requests
5. **Documentation**: Video tutorials, blog posts
6. **Maintenance**: Regular updates, bug fixes, security patches

---

## Conclusion

This approach creates a production-ready, extensible CLI tool with comprehensive features. While it requires significantly more time than Approach 1, it results in a tool that can truly replace the Python version and serve as a long-term solution.

**Best For**:
- Serious projects requiring production quality
- Learning advanced TypeScript architecture
- Building something for long-term use
- Contributing to open source ecosystem

**Not Recommended If**:
- You need something quickly
- You're just learning the basics
- You want to experiment first
- Limited time available (<2 months)

The investment in this approach pays off with a robust, maintainable, and extensible tool that can grow with your needs.
