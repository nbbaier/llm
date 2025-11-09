# Approach 1: Minimal Viable CLI - Detailed Implementation Plan

## Overview

**Goal**: Create a minimal but functional CLI that demonstrates core LLM interaction capabilities using the Vercel AI SDK.

**Timeline**: 2 weeks (80-100 hours)

**Outcome**: A working TypeScript CLI tool that can execute prompts, stream responses, manage API keys, and support basic conversations across multiple providers.

---

## Table of Contents

1. [Project Setup](#project-setup)
2. [Week 1: Foundation](#week-1-foundation)
3. [Week 2: Enhanced Features](#week-2-enhanced-features)
4. [Success Metrics](#success-metrics)
5. [Testing Strategy](#testing-strategy)
6. [Deliverables](#deliverables)
7. [Risk Mitigation](#risk-mitigation)

---

## Project Setup

### Day 0: Environment Preparation (2-4 hours)

#### Prerequisites

- Node.js 18+ installed
- npm or pnpm installed
- Git configured
- Code editor (VS Code recommended with TypeScript extensions)

#### Initial Setup

```bash
# Create project directory
mkdir llm-ts
cd llm-ts

# Initialize npm project
npm init -y

# Install core dependencies
npm install ai @ai-sdk/openai @ai-sdk/anthropic @ai-sdk/google commander chalk ora dotenv

# Install development dependencies
npm install --save-dev typescript @types/node tsx vitest @types/commander prettier eslint @typescript-eslint/parser @typescript-eslint/eslint-plugin

# Initialize TypeScript
npx tsc --init
```

#### Project Structure

```
llm-ts/
├── src/
│   ├── index.ts              # Main entry point
│   ├── cli.ts                # CLI command definitions
│   ├── config/
│   │   ├── config.ts         # Configuration management
│   │   └── keys.ts           # API key management
│   ├── providers/
│   │   ├── index.ts          # Provider registry
│   │   ├── openai.ts         # OpenAI provider
│   │   ├── anthropic.ts      # Anthropic provider
│   │   └── google.ts         # Google provider
│   ├── models/
│   │   ├── model.ts          # Base model interface
│   │   └── response.ts       # Response wrapper
│   ├── chat/
│   │   └── session.ts        # Chat session management
│   └── utils/
│       ├── stream.ts         # Streaming utilities
│       └── errors.ts         # Error handling
├── tests/
│   └── ... (test files mirror src/)
├── package.json
├── tsconfig.json
└── README.md
```

#### tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "lib": ["ES2022"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "tests"]
}
```

#### package.json scripts

```json
{
  "name": "llm-ts",
  "version": "0.1.0",
  "description": "CLI for interacting with LLMs using Vercel AI SDK",
  "type": "module",
  "bin": {
    "llm-ts": "./dist/index.js"
  },
  "scripts": {
    "dev": "tsx src/index.ts",
    "build": "tsc",
    "test": "vitest",
    "test:watch": "vitest --watch",
    "lint": "eslint src --ext .ts",
    "format": "prettier --write \"src/**/*.ts\"",
    "start": "node dist/index.js"
  }
}
```

**Success Criteria**:
- ✅ Project compiles without errors
- ✅ Can run `npm run dev -- --help`
- ✅ All dependencies installed correctly

---

## Week 1: Foundation

### Day 1-2: Core CLI Structure and Basic Prompt Execution (16 hours)

#### Step 1.1: Create Base CLI Framework (4 hours)

**File: `src/index.ts`**

```typescript
#!/usr/bin/env node
import { cli } from './cli.js';

cli.parse(process.argv);
```

**File: `src/cli.ts`**

```typescript
import { Command } from 'commander';
import chalk from 'chalk';
import { version } from '../package.json' assert { type: 'json' };

export const cli = new Command();

cli
  .name('llm-ts')
  .description('CLI for interacting with Large Language Models')
  .version(version);

// We'll add commands here
cli
  .argument('[prompt]', 'The prompt to send to the model')
  .option('-m, --model <model>', 'Model to use', 'gpt-4o-mini')
  .option('-s, --system <text>', 'System prompt')
  .option('--no-stream', 'Disable streaming output')
  .action(async (prompt, options) => {
    if (!prompt) {
      cli.help();
      process.exit(0);
    }

    // We'll implement this
    console.log(chalk.red('Not yet implemented'));
  });
```

**Testing**:
```bash
npm run dev -- --help
npm run dev -- --version
```

**Success Criteria**:
- ✅ Help text displays correctly
- ✅ Version command works
- ✅ Error handling for missing prompt

#### Step 1.2: Implement Configuration System (4 hours)

**File: `src/config/config.ts`**

```typescript
import { existsSync, mkdirSync, readFileSync, writeFileSync } from 'fs';
import { join } from 'path';
import { homedir } from 'os';

export class Config {
  private configDir: string;
  private configFile: string;
  private data: Record<string, any>;

  constructor() {
    this.configDir = this.getConfigDir();
    this.configFile = join(this.configDir, 'config.json');
    this.ensureConfigDir();
    this.load();
  }

  private getConfigDir(): string {
    if (process.env.LLM_TS_USER_PATH) {
      return process.env.LLM_TS_USER_PATH;
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

  private ensureConfigDir(): void {
    if (!existsSync(this.configDir)) {
      mkdirSync(this.configDir, { recursive: true });
    }
  }

  private load(): void {
    if (existsSync(this.configFile)) {
      const content = readFileSync(this.configFile, 'utf-8');
      this.data = JSON.parse(content);
    } else {
      this.data = {};
    }
  }

  private save(): void {
    writeFileSync(this.configFile, JSON.stringify(this.data, null, 2));
  }

  get<T>(key: string, defaultValue?: T): T | undefined {
    return this.data[key] ?? defaultValue;
  }

  set(key: string, value: any): void {
    this.data[key] = value;
    this.save();
  }

  delete(key: string): void {
    delete this.data[key];
    this.save();
  }

  getConfigDir(): string {
    return this.configDir;
  }
}

// Singleton instance
export const config = new Config();
```

**File: `src/config/keys.ts`**

```typescript
import { existsSync, readFileSync, writeFileSync } from 'fs';
import { join } from 'path';
import { config } from './config.js';
import * as readline from 'readline/promises';

export class KeyManager {
  private keysFile: string;
  private keys: Record<string, string>;

  constructor() {
    this.keysFile = join(config.getConfigDir(), 'keys.json');
    this.load();
  }

  private load(): void {
    if (existsSync(this.keysFile)) {
      const content = readFileSync(this.keysFile, 'utf-8');
      this.keys = JSON.parse(content);
    } else {
      this.keys = {};
    }
  }

  private save(): void {
    writeFileSync(this.keysFile, JSON.stringify(this.keys, null, 2), {
      mode: 0o600 // Readable/writable by owner only
    });
  }

  getKey(provider: string): string | undefined {
    // Check environment variable first
    const envVarName = `${provider.toUpperCase()}_API_KEY`;
    if (process.env[envVarName]) {
      return process.env[envVarName];
    }

    // Check stored keys
    return this.keys[provider];
  }

  async setKey(provider: string): Promise<void> {
    const rl = readline.createInterface({
      input: process.stdin,
      output: process.stdout
    });

    // Hide input (not perfect but works on most terminals)
    const key = await rl.question(`Enter API key for ${provider}: `);
    rl.close();

    this.keys[provider] = key.trim();
    this.save();
    console.log(`API key for ${provider} saved successfully`);
  }

  listKeys(): string[] {
    return Object.keys(this.keys);
  }

  deleteKey(provider: string): void {
    delete this.keys[provider];
    this.save();
  }
}

export const keyManager = new KeyManager();
```

**Testing**:
```typescript
// tests/config.test.ts
import { describe, it, expect } from 'vitest';
import { Config } from '../src/config/config';

describe('Config', () => {
  it('should create config directory', () => {
    const config = new Config();
    expect(config.getConfigDir()).toBeDefined();
  });

  it('should set and get values', () => {
    const config = new Config();
    config.set('test', 'value');
    expect(config.get('test')).toBe('value');
  });
});
```

**Success Criteria**:
- ✅ Config directory created on first run
- ✅ Can save and load configuration
- ✅ Keys stored with restricted permissions (600)
- ✅ Environment variables override stored keys

#### Step 1.3: Implement Provider System (4 hours)

**File: `src/providers/index.ts`**

```typescript
import { openai } from '@ai-sdk/openai';
import { anthropic } from '@ai-sdk/anthropic';
import { google } from '@ai-sdk/google';
import { LanguageModel } from 'ai';
import { keyManager } from '../config/keys.js';

export interface ProviderConfig {
  name: string;
  models: string[];
  keyRequired: boolean;
}

export const PROVIDERS: Record<string, ProviderConfig> = {
  openai: {
    name: 'OpenAI',
    models: ['gpt-4o', 'gpt-4o-mini', 'gpt-4-turbo', 'gpt-3.5-turbo'],
    keyRequired: true
  },
  anthropic: {
    name: 'Anthropic',
    models: ['claude-3-5-sonnet-20241022', 'claude-3-opus-20240229', 'claude-3-haiku-20240307'],
    keyRequired: true
  },
  google: {
    name: 'Google',
    models: ['gemini-1.5-pro', 'gemini-1.5-flash', 'gemini-2.0-flash-exp'],
    keyRequired: true
  }
};

export function getModel(modelId: string): LanguageModel {
  // Detect provider from model name
  let provider: string;

  if (modelId.startsWith('gpt-')) {
    provider = 'openai';
  } else if (modelId.startsWith('claude-')) {
    provider = 'anthropic';
  } else if (modelId.startsWith('gemini-')) {
    provider = 'google';
  } else {
    throw new Error(`Unknown model: ${modelId}`);
  }

  // Get API key
  const apiKey = keyManager.getKey(provider);
  if (!apiKey) {
    throw new Error(
      `No API key found for ${provider}. Set it with: llm-ts keys set ${provider}`
    );
  }

  // Return appropriate model
  switch (provider) {
    case 'openai':
      return openai(modelId, { apiKey });
    case 'anthropic':
      return anthropic(modelId, { apiKey });
    case 'google':
      return google(modelId, { apiKey });
    default:
      throw new Error(`Provider not implemented: ${provider}`);
  }
}

export function listModels(): { provider: string; models: string[] }[] {
  return Object.entries(PROVIDERS).map(([key, config]) => ({
    provider: config.name,
    models: config.models
  }));
}
```

**Success Criteria**:
- ✅ Can detect provider from model name
- ✅ Throws clear error when API key missing
- ✅ Environment variables work for API keys
- ✅ Supports OpenAI, Anthropic, and Google

#### Step 1.4: Implement Basic Prompt Execution (4 hours)

**File: `src/models/response.ts`**

```typescript
export class Response {
  constructor(
    public text: string,
    public model: string,
    public usage?: {
      promptTokens?: number;
      completionTokens?: number;
      totalTokens?: number;
    }
  ) {}

  toString(): string {
    return this.text;
  }
}
```

**Update `src/cli.ts`** with actual implementation:

```typescript
import { generateText, streamText } from 'ai';
import { getModel } from './providers/index.js';
import ora from 'ora';

// ... previous code ...

cli
  .argument('[prompt]', 'The prompt to send to the model')
  .option('-m, --model <model>', 'Model to use', 'gpt-4o-mini')
  .option('-s, --system <text>', 'System prompt')
  .option('--no-stream', 'Disable streaming output')
  .action(async (promptText, options) => {
    if (!promptText) {
      // Check if stdin has data
      if (process.stdin.isTTY) {
        cli.help();
        process.exit(0);
      }

      // Read from stdin
      const chunks: Buffer[] = [];
      for await (const chunk of process.stdin) {
        chunks.push(chunk);
      }
      promptText = Buffer.concat(chunks).toString('utf-8').trim();
    }

    try {
      const model = getModel(options.model);

      if (options.stream) {
        // Streaming mode
        const { textStream } = await streamText({
          model,
          prompt: promptText,
          system: options.system
        });

        for await (const chunk of textStream) {
          process.stdout.write(chunk);
        }
        console.log(); // New line at end
      } else {
        // Non-streaming mode
        const spinner = ora('Thinking...').start();

        const { text, usage } = await generateText({
          model,
          prompt: promptText,
          system: options.system
        });

        spinner.stop();
        console.log(text);

        if (usage) {
          console.log(chalk.dim(`\n[Tokens: ${usage.totalTokens}]`));
        }
      }
    } catch (error) {
      console.error(chalk.red('Error:'), error.message);
      process.exit(1);
    }
  });
```

**Testing**:
```bash
# Test basic prompt
npm run dev -- "What is TypeScript?"

# Test with system prompt
npm run dev -- "Explain concisely" -s "You are a helpful assistant"

# Test with stdin
echo "What is Node.js?" | npm run dev

# Test non-streaming
npm run dev -- "Count to 5" --no-stream
```

**Success Criteria**:
- ✅ Can execute basic prompts
- ✅ Streaming works by default
- ✅ Non-streaming mode shows spinner
- ✅ Can read from stdin
- ✅ System prompts work
- ✅ Clear error messages

---

### Day 3-5: Key Management CLI (12 hours)

#### Step 3.1: Add Keys Commands (6 hours)

**Update `src/cli.ts`**:

```typescript
import inquirer from 'inquirer';

// Keys command
const keysCmd = cli.command('keys').description('Manage API keys');

keysCmd
  .command('set <provider>')
  .description('Set API key for a provider')
  .action(async (provider) => {
    try {
      await keyManager.setKey(provider);
    } catch (error) {
      console.error(chalk.red('Error:'), error.message);
      process.exit(1);
    }
  });

keysCmd
  .command('list')
  .description('List saved API keys')
  .action(() => {
    const keys = keyManager.listKeys();
    if (keys.length === 0) {
      console.log('No API keys saved');
    } else {
      console.log('Saved API keys:');
      keys.forEach(key => console.log(`  - ${key}`));
    }
  });

keysCmd
  .command('delete <provider>')
  .description('Delete API key for a provider')
  .action(async (provider) => {
    const { confirm } = await inquirer.prompt([{
      type: 'confirm',
      name: 'confirm',
      message: `Delete API key for ${provider}?`,
      default: false
    }]);

    if (confirm) {
      keyManager.deleteKey(provider);
      console.log(chalk.green(`API key for ${provider} deleted`));
    }
  });

keysCmd
  .command('path')
  .description('Show path to keys file')
  .action(() => {
    console.log(config.getConfigDir());
  });
```

**Add to package.json**:
```bash
npm install inquirer @types/inquirer
```

**Success Criteria**:
- ✅ Can set keys interactively
- ✅ Can list saved keys (without showing values)
- ✅ Can delete keys with confirmation
- ✅ Can show path to config directory

#### Step 3.2: Add Models Command (3 hours)

```typescript
// Models command
cli
  .command('models')
  .description('List available models')
  .option('-v, --verbose', 'Show detailed information')
  .action((options) => {
    const models = listModels();

    if (options.verbose) {
      models.forEach(({ provider, models }) => {
        console.log(chalk.bold(`\n${provider}:`));
        models.forEach(model => {
          const hasKey = !!keyManager.getKey(provider.toLowerCase());
          const status = hasKey ? chalk.green('✓') : chalk.red('✗');
          console.log(`  ${status} ${model}`);
        });
      });

      console.log(chalk.dim('\n✓ = API key configured'));
      console.log(chalk.dim('✗ = API key not configured'));
    } else {
      models.forEach(({ provider, models }) => {
        console.log(chalk.bold(`${provider}:`));
        models.forEach(model => console.log(`  ${model}`));
      });
    }
  });
```

**Success Criteria**:
- ✅ Lists all supported models
- ✅ Verbose mode shows key status
- ✅ Clear, readable output

#### Step 3.3: Configuration Testing (3 hours)

```typescript
// tests/keys.test.ts
import { describe, it, expect, beforeEach } from 'vitest';
import { KeyManager } from '../src/config/keys';
import { existsSync, rmSync } from 'fs';

describe('KeyManager', () => {
  it('should save and retrieve keys', () => {
    const km = new KeyManager();
    km.keys['test'] = 'test-key';
    km.save();
    expect(km.getKey('test')).toBe('test-key');
  });

  it('should respect environment variables', () => {
    process.env.OPENAI_API_KEY = 'env-key';
    const km = new KeyManager();
    expect(km.getKey('openai')).toBe('env-key');
    delete process.env.OPENAI_API_KEY;
  });
});
```

**Success Criteria**:
- ✅ All tests pass
- ✅ Keys file has correct permissions
- ✅ Environment variables take precedence

---

### Day 6-7: Model Options and Error Handling (8 hours)

#### Step 6.1: Add Model Options (4 hours)

**Update CLI to support more options**:

```typescript
cli
  .argument('[prompt]', 'The prompt to send to the model')
  .option('-m, --model <model>', 'Model to use', 'gpt-4o-mini')
  .option('-s, --system <text>', 'System prompt')
  .option('--no-stream', 'Disable streaming output')
  .option('-t, --temperature <number>', 'Temperature (0-2)', parseFloat)
  .option('--max-tokens <number>', 'Maximum tokens to generate', parseInt)
  .option('-o, --option <key=value>', 'Model-specific option', collect, [])
  .action(async (promptText, options) => {
    // ... existing code ...

    // Parse options
    const modelOptions: any = {};
    if (options.temperature !== undefined) {
      modelOptions.temperature = options.temperature;
    }
    if (options.maxTokens !== undefined) {
      modelOptions.maxTokens = options.maxTokens;
    }

    // Parse custom options
    options.option.forEach(opt => {
      const [key, value] = opt.split('=');
      modelOptions[key] = tryParseValue(value);
    });

    // Use in prompt
    const { textStream } = await streamText({
      model,
      prompt: promptText,
      system: options.system,
      ...modelOptions
    });

    // ... rest of code ...
  });

function collect(value: string, previous: string[]) {
  return previous.concat([value]);
}

function tryParseValue(value: string): any {
  if (value === 'true') return true;
  if (value === 'false') return false;
  if (!isNaN(Number(value))) return Number(value);
  return value;
}
```

**Testing**:
```bash
npm run dev -- "Tell me a joke" -t 1.5
npm run dev -- "Be creative" --max-tokens 100
npm run dev -- "Test" -o topP=0.9
```

**Success Criteria**:
- ✅ Temperature option works
- ✅ Max tokens option works
- ✅ Custom options parsed correctly
- ✅ Invalid options show clear errors

#### Step 6.2: Enhanced Error Handling (4 hours)

**File: `src/utils/errors.ts`**

```typescript
export class LLMError extends Error {
  constructor(message: string, public code?: string) {
    super(message);
    this.name = 'LLMError';
  }
}

export class ConfigError extends LLMError {
  constructor(message: string) {
    super(message, 'CONFIG_ERROR');
    this.name = 'ConfigError';
  }
}

export class APIKeyError extends LLMError {
  constructor(provider: string) {
    super(
      `No API key found for ${provider}.\n` +
      `Set it with: llm-ts keys set ${provider}\n` +
      `Or set ${provider.toUpperCase()}_API_KEY environment variable`,
      'API_KEY_ERROR'
    );
    this.name = 'APIKeyError';
  }
}

export class ModelError extends LLMError {
  constructor(message: string) {
    super(message, 'MODEL_ERROR');
    this.name = 'ModelError';
  }
}

export function handleError(error: any): void {
  if (error instanceof APIKeyError) {
    console.error(chalk.red('API Key Error:'));
    console.error(error.message);
  } else if (error instanceof ConfigError) {
    console.error(chalk.red('Configuration Error:'));
    console.error(error.message);
  } else if (error instanceof ModelError) {
    console.error(chalk.red('Model Error:'));
    console.error(error.message);
  } else if (error.code === 'ECONNREFUSED') {
    console.error(chalk.red('Network Error:'));
    console.error('Unable to connect to API. Check your internet connection.');
  } else if (error.status === 401) {
    console.error(chalk.red('Authentication Error:'));
    console.error('Invalid API key. Check your key with: llm-ts keys list');
  } else if (error.status === 429) {
    console.error(chalk.red('Rate Limit Error:'));
    console.error('Rate limit exceeded. Please try again later.');
  } else {
    console.error(chalk.red('Error:'));
    console.error(error.message || error);
  }
  process.exit(1);
}
```

**Update all command actions to use error handler**:

```typescript
.action(async (promptText, options) => {
  try {
    // ... existing code ...
  } catch (error) {
    handleError(error);
  }
});
```

**Success Criteria**:
- ✅ Clear error messages for all common scenarios
- ✅ Network errors handled gracefully
- ✅ API key errors show helpful instructions
- ✅ Rate limit errors provide guidance

---

## Week 2: Enhanced Features

### Day 8-10: Interactive Chat Mode (12 hours)

#### Step 8.1: Basic Chat Session (6 hours)

**File: `src/chat/session.ts`**

```typescript
import * as readline from 'readline/promises';
import { streamText } from 'ai';
import { getModel } from '../providers/index.js';
import chalk from 'chalk';

interface Message {
  role: 'user' | 'assistant' | 'system';
  content: string;
}

export class ChatSession {
  private messages: Message[] = [];
  private rl: readline.Interface;

  constructor(
    private modelId: string,
    private systemPrompt?: string
  ) {
    this.rl = readline.createInterface({
      input: process.stdin,
      output: process.stdout,
      prompt: chalk.blue('> ')
    });

    if (systemPrompt) {
      this.messages.push({
        role: 'system',
        content: systemPrompt
      });
    }
  }

  async start(): Promise<void> {
    console.log(chalk.bold(`Chatting with ${this.modelId}`));
    console.log(chalk.dim('Type "exit" or "quit" to exit'));
    console.log(chalk.dim('Type "!help" for more commands\n'));

    this.rl.prompt();

    for await (const line of this.rl) {
      const input = line.trim();

      // Handle special commands
      if (input === 'exit' || input === 'quit') {
        break;
      }

      if (input === '!help') {
        this.showHelp();
        this.rl.prompt();
        continue;
      }

      if (input === '!clear') {
        this.messages = this.messages.filter(m => m.role === 'system');
        console.log(chalk.dim('Conversation cleared'));
        this.rl.prompt();
        continue;
      }

      if (input === '!history') {
        this.showHistory();
        this.rl.prompt();
        continue;
      }

      if (!input) {
        this.rl.prompt();
        continue;
      }

      await this.processMessage(input);
      this.rl.prompt();
    }

    this.rl.close();
    console.log(chalk.dim('\nGoodbye!'));
  }

  private async processMessage(input: string): Promise<void> {
    this.messages.push({
      role: 'user',
      content: input
    });

    try {
      const model = getModel(this.modelId);
      const { textStream } = await streamText({
        model,
        messages: this.messages
      });

      let response = '';
      for await (const chunk of textStream) {
        process.stdout.write(chunk);
        response += chunk;
      }
      console.log('\n');

      this.messages.push({
        role: 'assistant',
        content: response
      });
    } catch (error) {
      console.error(chalk.red('\nError:'), error.message);
    }
  }

  private showHelp(): void {
    console.log(chalk.bold('\nAvailable commands:'));
    console.log('  exit, quit     Exit the chat');
    console.log('  !help          Show this help');
    console.log('  !clear         Clear conversation history');
    console.log('  !history       Show conversation history');
    console.log('  !multi         Enter multi-line mode (not yet implemented)');
    console.log();
  }

  private showHistory(): void {
    console.log(chalk.bold('\nConversation history:'));
    this.messages.forEach((msg, idx) => {
      if (msg.role === 'system') return;
      const prefix = msg.role === 'user' ? chalk.blue('User:') : chalk.green('Assistant:');
      const content = msg.content.substring(0, 100);
      const suffix = msg.content.length > 100 ? '...' : '';
      console.log(`${idx}. ${prefix} ${content}${suffix}`);
    });
    console.log();
  }
}
```

**Add chat command to CLI**:

```typescript
cli
  .command('chat')
  .description('Start an interactive chat session')
  .option('-m, --model <model>', 'Model to use', 'gpt-4o-mini')
  .option('-s, --system <text>', 'System prompt')
  .action(async (options) => {
    try {
      const session = new ChatSession(options.model, options.system);
      await session.start();
    } catch (error) {
      handleError(error);
    }
  });
```

**Testing**:
```bash
npm run dev chat
npm run dev chat -m claude-3-5-sonnet-20241022
npm run dev chat -s "You are a helpful coding assistant"
```

**Success Criteria**:
- ✅ Interactive chat works
- ✅ Maintains conversation context
- ✅ Special commands work (!help, !clear, !history)
- ✅ Can exit gracefully
- ✅ Handles errors in conversation

#### Step 8.2: Multi-line Input Support (3 hours)

**Update `src/chat/session.ts`**:

```typescript
export class ChatSession {
  private multiLineMode = false;
  private multiLineBuffer: string[] = [];

  // ... existing code ...

  async start(): Promise<void> {
    console.log(chalk.bold(`Chatting with ${this.modelId}`));
    console.log(chalk.dim('Type "exit" or "quit" to exit'));
    console.log(chalk.dim('Type "!multi" for multi-line input'));
    console.log(chalk.dim('Type "!help" for more commands\n'));

    this.rl.prompt();

    for await (const line of this.rl) {
      const input = line.trim();

      // Handle multi-line mode
      if (this.multiLineMode) {
        if (input === '!end') {
          const fullInput = this.multiLineBuffer.join('\n');
          this.multiLineBuffer = [];
          this.multiLineMode = false;
          await this.processMessage(fullInput);
          this.rl.prompt();
        } else if (input === '!cancel') {
          this.multiLineBuffer = [];
          this.multiLineMode = false;
          console.log(chalk.dim('Multi-line input cancelled'));
          this.rl.prompt();
        } else {
          this.multiLineBuffer.push(line);
          process.stdout.write(chalk.dim('... '));
        }
        continue;
      }

      // Handle !multi command
      if (input === '!multi') {
        this.multiLineMode = true;
        console.log(chalk.dim('Entering multi-line mode. Type !end when done, !cancel to abort'));
        process.stdout.write(chalk.dim('... '));
        continue;
      }

      // ... rest of existing code ...
    }
  }

  private showHelp(): void {
    console.log(chalk.bold('\nAvailable commands:'));
    console.log('  exit, quit     Exit the chat');
    console.log('  !help          Show this help');
    console.log('  !clear         Clear conversation history');
    console.log('  !history       Show conversation history');
    console.log('  !multi         Enter multi-line mode');
    console.log('    !end         Finish multi-line input');
    console.log('    !cancel      Cancel multi-line input');
    console.log();
  }
}
```

**Success Criteria**:
- ✅ Can enter multi-line mode
- ✅ Can complete multi-line input with !end
- ✅ Can cancel with !cancel
- ✅ Clear visual feedback for multi-line mode

#### Step 8.3: Chat Enhancement and Polish (3 hours)

```typescript
// Add message counting
private messageCount = 0;

private async processMessage(input: string): Promise<void> {
  this.messageCount++;

  this.messages.push({
    role: 'user',
    content: input
  });

  const startTime = Date.now();

  try {
    const model = getModel(this.modelId);
    const { textStream, usage } = await streamText({
      model,
      messages: this.messages
    });

    let response = '';
    for await (const chunk of textStream) {
      process.stdout.write(chunk);
      response += chunk;
    }

    const duration = Date.now() - startTime;
    console.log(chalk.dim(`\n[${duration}ms]`));

    this.messages.push({
      role: 'assistant',
      content: response
    });
  } catch (error) {
    console.error(chalk.red('\nError:'), error.message);
  }
}

// Add !save command to save conversation
if (input.startsWith('!save')) {
  const filename = input.split(' ')[1] || `chat-${Date.now()}.json`;
  this.saveConversation(filename);
  this.rl.prompt();
  continue;
}

private saveConversation(filename: string): void {
  const data = {
    model: this.modelId,
    timestamp: new Date().toISOString(),
    messages: this.messages
  };

  writeFileSync(filename, JSON.stringify(data, null, 2));
  console.log(chalk.green(`Conversation saved to ${filename}`));
}
```

**Success Criteria**:
- ✅ Shows response time
- ✅ Can save conversations
- ✅ Message count tracking
- ✅ Polished user experience

---

### Day 11-12: Stdin Support and Piping (8 hours)

#### Step 11.1: Enhanced Stdin Handling (4 hours)

**Update main prompt handler**:

```typescript
cli
  .argument('[prompt]', 'The prompt to send to the model')
  .option('-m, --model <model>', 'Model to use', 'gpt-4o-mini')
  .option('-s, --system <text>', 'System prompt')
  .option('--no-stream', 'Disable streaming output')
  .option('-t, --temperature <number>', 'Temperature', parseFloat)
  .option('--stdin', 'Explicitly read from stdin')
  .action(async (promptText, options) => {
    try {
      let prompt = promptText;
      let stdinContent = '';

      // Read from stdin if available
      if (!process.stdin.isTTY || options.stdin) {
        const chunks: Buffer[] = [];
        for await (const chunk of process.stdin) {
          chunks.push(chunk);
        }
        stdinContent = Buffer.concat(chunks).toString('utf-8').trim();
      }

      // Combine prompt and stdin
      if (stdinContent && prompt) {
        prompt = `${prompt}\n\n${stdinContent}`;
      } else if (stdinContent) {
        prompt = stdinContent;
      } else if (!prompt) {
        cli.help();
        process.exit(0);
      }

      const model = getModel(options.model);

      if (options.stream) {
        const { textStream } = await streamText({
          model,
          prompt,
          system: options.system,
          temperature: options.temperature
        });

        for await (const chunk of textStream) {
          process.stdout.write(chunk);
        }
        console.log();
      } else {
        const spinner = ora('Thinking...').start();

        const { text } = await generateText({
          model,
          prompt,
          system: options.system,
          temperature: options.temperature
        });

        spinner.stop();
        console.log(text);
      }
    } catch (error) {
      handleError(error);
    }
  });
```

**Testing**:
```bash
# Pipe file content
cat README.md | npm run dev -- "Summarize this"

# Pipe command output
ls -la | npm run dev -- "Explain these files"

# Use with system prompt
echo "function add(a, b) { return a + b; }" | npm run dev -- "Explain" -s "You are a code reviewer"
```

**Success Criteria**:
- ✅ Can read from stdin
- ✅ Combines stdin with prompt
- ✅ Works in pipes
- ✅ Handles large inputs

#### Step 11.2: Output Formatting Options (4 hours)

```typescript
cli
  .argument('[prompt]', 'The prompt to send to the model')
  // ... existing options ...
  .option('-j, --json', 'Output response as JSON')
  .option('-q, --quiet', 'Suppress all non-essential output')
  .action(async (promptText, options) => {
    try {
      // ... existing code ...

      if (options.stream && !options.json) {
        for await (const chunk of textStream) {
          process.stdout.write(chunk);
        }
        console.log();
      } else {
        if (!options.quiet) {
          const spinner = ora('Thinking...').start();
        }

        const result = await generateText({
          model,
          prompt,
          system: options.system,
          temperature: options.temperature
        });

        if (!options.quiet) {
          spinner.stop();
        }

        if (options.json) {
          console.log(JSON.stringify({
            text: result.text,
            model: options.model,
            usage: result.usage
          }, null, 2));
        } else {
          console.log(result.text);
        }
      }
    } catch (error) {
      handleError(error);
    }
  });
```

**Success Criteria**:
- ✅ JSON output works
- ✅ Quiet mode suppresses extras
- ✅ Can be used in scripts
- ✅ Parseable output

---

### Day 13-14: Documentation, Testing, and Polish (8 hours)

#### Step 13.1: Comprehensive Testing (4 hours)

**File: `tests/integration.test.ts`**

```typescript
import { describe, it, expect, beforeAll } from 'vitest';
import { exec } from 'child_process';
import { promisify } from 'util';

const execAsync = promisify(exec);

describe('CLI Integration Tests', () => {
  beforeAll(() => {
    // Ensure test API key is set
    process.env.OPENAI_API_KEY = 'test-key';
  });

  it('should show help', async () => {
    const { stdout } = await execAsync('npm run dev -- --help');
    expect(stdout).toContain('CLI for interacting with Large Language Models');
  });

  it('should list models', async () => {
    const { stdout } = await execAsync('npm run dev models');
    expect(stdout).toContain('OpenAI');
    expect(stdout).toContain('gpt-4');
  });

  it('should handle stdin', async () => {
    const { stdout } = await execAsync('echo "test" | npm run dev -- "echo this"');
    expect(stdout).toBeTruthy();
  });
});
```

**Run tests**:
```bash
npm test
```

**Success Criteria**:
- ✅ All unit tests pass
- ✅ Integration tests pass
- ✅ Test coverage >70%

#### Step 13.2: Documentation (2 hours)

**Create `README.md`**:

```markdown
# llm-ts

A TypeScript CLI for interacting with Large Language Models using the Vercel AI SDK.

## Installation

\`\`\`bash
npm install -g llm-ts
\`\`\`

## Quick Start

\`\`\`bash
# Set API key
llm-ts keys set openai

# Run a prompt
llm-ts "What is TypeScript?"

# Start a chat
llm-ts chat
\`\`\`

## Commands

### Prompt Execution

\`\`\`bash
llm-ts [prompt] [options]
\`\`\`

Options:
- `-m, --model <model>`: Model to use (default: gpt-4o-mini)
- `-s, --system <text>`: System prompt
- `-t, --temperature <number>`: Temperature (0-2)
- `--no-stream`: Disable streaming
- `-j, --json`: Output as JSON

### Chat

\`\`\`bash
llm-ts chat [options]
\`\`\`

Interactive commands:
- `exit`, `quit`: Exit chat
- `!help`: Show help
- `!clear`: Clear history
- `!multi`: Multi-line input
- `!save [filename]`: Save conversation

### Key Management

\`\`\`bash
llm-ts keys set <provider>    # Set API key
llm-ts keys list              # List saved keys
llm-ts keys delete <provider> # Delete key
llm-ts keys path              # Show config path
\`\`\`

### Models

\`\`\`bash
llm-ts models           # List models
llm-ts models -v        # Show key status
\`\`\`

## Examples

\`\`\`bash
# Basic prompt
llm-ts "Explain quantum computing"

# With system prompt
llm-ts "Explain briefly" -s "You are a teacher"

# Read from file
cat code.ts | llm-ts "Review this code"

# Different model
llm-ts "Tell me a joke" -m claude-3-5-sonnet-20241022

# JSON output for scripts
llm-ts "Count to 5" -j | jq .text
\`\`\`

## Supported Providers

- OpenAI (gpt-4o, gpt-4o-mini, etc.)
- Anthropic (claude-3-5-sonnet, etc.)
- Google (gemini-1.5-pro, etc.)

## Configuration

Config directory: `~/.config/llm-ts`

Files:
- `keys.json`: API keys
- `config.json`: Settings

## License

MIT
```

**Success Criteria**:
- ✅ README complete
- ✅ All commands documented
- ✅ Examples provided
- ✅ Clear installation instructions

#### Step 13.3: Polish and Package (2 hours)

**Update `package.json`**:

```json
{
  "name": "llm-ts",
  "version": "0.1.0",
  "description": "CLI for interacting with LLMs using Vercel AI SDK",
  "type": "module",
  "bin": {
    "llm-ts": "./dist/index.js"
  },
  "files": [
    "dist",
    "README.md"
  ],
  "keywords": [
    "llm",
    "cli",
    "ai",
    "openai",
    "anthropic",
    "chatgpt",
    "typescript"
  ],
  "repository": {
    "type": "git",
    "url": "your-repo-url"
  }
}
```

**Build and test**:
```bash
npm run build
node dist/index.js --help
```

**Success Criteria**:
- ✅ Builds without errors
- ✅ Binary works when installed globally
- ✅ Package ready for publishing

---

## Success Metrics

### Week 1 Success Criteria

**Must Have** (Blocking):
- [ ] Project compiles and runs
- [ ] Can execute basic prompts with OpenAI
- [ ] API key management works
- [ ] Can list available models
- [ ] Error handling provides clear messages

**Should Have** (Important):
- [ ] Supports 3 providers (OpenAI, Anthropic, Google)
- [ ] Streaming responses work
- [ ] Can read from stdin
- [ ] System prompts work

**Nice to Have** (Optional):
- [ ] Progress indicators
- [ ] Colored output
- [ ] Model options (temperature, etc.)

### Week 2 Success Criteria

**Must Have** (Blocking):
- [ ] Interactive chat works
- [ ] Conversation context maintained
- [ ] Can exit chat gracefully
- [ ] Basic documentation complete

**Should Have** (Important):
- [ ] Multi-line input works
- [ ] Special chat commands (!clear, !help)
- [ ] Can save conversations
- [ ] JSON output mode

**Nice to Have** (Optional):
- [ ] Conversation history display
- [ ] Response timing
- [ ] Comprehensive tests

### Overall Project Success

**Functionality** (40 points):
- [ ] (10) Can execute prompts with all 3 providers
- [ ] (10) Streaming and non-streaming modes work
- [ ] (10) Interactive chat maintains context
- [ ] (10) Key management is secure and functional

**Usability** (30 points):
- [ ] (10) Clear help text and documentation
- [ ] (10) Good error messages
- [ ] (10) Works with stdin/pipes

**Code Quality** (30 points):
- [ ] (10) TypeScript types used properly
- [ ] (10) Tests written and passing
- [ ] (10) Code is organized and readable

**Target Score**: 75/100 (75% of features working well)

---

## Testing Strategy

### Unit Tests

Test individual components in isolation:

```typescript
// tests/config.test.ts
describe('Config', () => {
  it('should create config directory');
  it('should save and load values');
  it('should handle missing config');
});

// tests/keys.test.ts
describe('KeyManager', () => {
  it('should save keys securely');
  it('should retrieve keys');
  it('should respect env variables');
});

// tests/providers.test.ts
describe('ProviderManager', () => {
  it('should detect provider from model name');
  it('should throw on invalid model');
  it('should throw on missing key');
});
```

### Integration Tests

Test CLI commands end-to-end:

```bash
# Test basic execution
npm run dev -- "test prompt"

# Test with stdin
echo "test" | npm run dev -- "prompt"

# Test keys command
npm run dev keys list

# Test models command
npm run dev models
```

### Manual Testing Checklist

Week 1:
- [ ] Basic prompt execution
- [ ] Set/list/delete keys
- [ ] List models
- [ ] Different providers
- [ ] Streaming on/off
- [ ] System prompts
- [ ] Error handling

Week 2:
- [ ] Start chat
- [ ] Multiple messages
- [ ] Clear history
- [ ] Multi-line input
- [ ] Save conversation
- [ ] Exit gracefully

---

## Deliverables

### Week 1 Deliverables

1. **Working CLI Binary** (`dist/index.js`)
   - Executable via `npm run dev`
   - All core commands functional

2. **Key Management System**
   - Secure storage (600 permissions)
   - Environment variable support
   - List/set/delete commands

3. **Provider Integration**
   - OpenAI support
   - Anthropic support
   - Google support

4. **Documentation**
   - README with examples
   - Help text for all commands
   - Error messages

### Week 2 Deliverables

1. **Interactive Chat**
   - Chat command
   - Special commands
   - Multi-line support
   - Conversation persistence

2. **Enhanced Features**
   - Stdin support
   - JSON output
   - Model options

3. **Testing**
   - Unit tests (>70% coverage)
   - Integration tests
   - Manual test results

4. **Polish**
   - Colored output
   - Progress indicators
   - Error handling
   - Documentation complete

### Final Deliverable

A complete, working CLI tool that:
- Executes prompts across multiple providers
- Provides interactive chat capabilities
- Manages API keys securely
- Is well-documented and tested
- Can be installed and used globally

---

## Risk Mitigation

### Technical Risks

**Risk**: API rate limits during testing
- **Mitigation**: Use mock responses for automated tests
- **Mitigation**: Implement retry logic with backoff

**Risk**: API key security
- **Mitigation**: File permissions (600)
- **Mitigation**: Never log keys
- **Mitigation**: Environment variable support

**Risk**: Large stdin inputs
- **Mitigation**: Stream processing
- **Mitigation**: Token limit warnings

**Risk**: Cross-platform compatibility
- **Mitigation**: Test on Windows/Mac/Linux
- **Mitigation**: Use platform-agnostic paths

### Schedule Risks

**Risk**: Week 1 takes longer than expected
- **Mitigation**: Focus on core features first
- **Mitigation**: Cut non-essential features
- **Mitigation**: Extend to 3 weeks if needed

**Risk**: API changes during development
- **Mitigation**: Pin dependency versions
- **Mitigation**: Monitor Vercel AI SDK releases

### Scope Risks

**Risk**: Feature creep
- **Mitigation**: Stick to defined scope
- **Mitigation**: Track "nice to have" separately
- **Mitigation**: Time-box each feature

---

## Next Steps After Completion

Once the minimal CLI is complete, consider:

1. **Add Logging**: SQLite-based prompt/response logging
2. **Add Templates**: Simple template system
3. **Add Tools**: Function calling support
4. **Publish to npm**: Make it available globally
5. **Add More Providers**: Extend to other providers
6. **Performance Optimization**: Caching, faster startup

---

## Conclusion

This approach provides a solid foundation for learning the Vercel AI SDK while building something immediately useful. The 2-week timeline is aggressive but achievable with focused effort.

**Key Success Factors**:
- Focus on core features first
- Test continuously
- Keep scope limited
- Document as you go
- Celebrate small wins

**Learning Outcomes**:
- Hands-on Vercel AI SDK experience
- TypeScript CLI development
- Async programming patterns
- API integration best practices
- Configuration management
- Error handling strategies

Good luck with your implementation! 🚀
