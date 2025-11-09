# Approach 4: Incremental Migration - Implementation Plan

## Overview

**Goal**: Port features one at a time from Python to TypeScript, maintaining both versions side-by-side until complete.

**Timeline**: 12-16 weeks (Flexible - can pause between components)

**Outcome**: Gradual, validated transition from Python to TypeScript with both tools coexisting during the migration period.

**When to Choose This Approach**:
- You have existing Python `llm` users who depend on it
- You want to de-risk the migration
- You prefer working on one feature at a time
- You want to validate each component before moving on
- You need to maintain backward compatibility
- You're migrating an existing codebase (not starting fresh)

---

## Table of Contents

1. [Migration Strategy](#migration-strategy)
2. [Coexistence Architecture](#coexistence-architecture)
3. [Component Migration Order](#component-migration-order)
4. [Data Compatibility](#data-compatibility)
5. [Testing Strategy](#testing-strategy)
6. [Rollout Plan](#rollout-plan)
7. [Success Criteria](#success-criteria)

---

## Migration Strategy

### Core Principles

**1. Incremental Value**
- Each migrated component provides immediate value
- Can ship intermediate versions
- Users can adopt incrementally

**2. Parallel Operation**
- Both Python and TypeScript versions work simultaneously
- Share configuration and data where possible
- Gradual user migration, not forced

**3. Validation at Each Step**
- Test each component thoroughly before moving on
- User feedback incorporated early
- Can pause or adjust direction

**4. Backward Compatibility**
- Maintain Python CLI throughout
- TypeScript CLI can invoke Python for missing features
- Eventually Python can be deprecated gracefully

---

## Coexistence Architecture

### File System Structure

```
llm-project/
├── python/                     # Original Python implementation
│   ├── llm/
│   ├── tests/
│   └── setup.py
├── typescript/                 # New TypeScript implementation
│   ├── packages/
│   │   ├── core/
│   │   ├── cli/
│   │   └── ...
│   └── package.json
├── shared/                     # Shared resources
│   └── config/
│       └── .llm/              # Shared config directory
│           ├── keys.json      # Shared API keys
│           ├── aliases.json   # Shared aliases
│           ├── logs.db        # Shared database!
│           └── config.json    # Shared settings
└── README.md
```

### Shared Configuration

**Key Design Decision**: Use the same configuration directory!

```typescript
// TypeScript reads Python's config
import { homedir } from 'os';
import { join } from 'path';

function getConfigDir(): string {
  // Use same location as Python version
  return process.env.LLM_USER_PATH ||
    join(homedir(), '.config', 'io.datasette.llm');
}
```

**Benefits**:
- API keys work in both versions
- Model aliases shared
- Unified configuration experience
- Easier for users to try TypeScript version

### Shared Database Schema

**Critical**: TypeScript must use the same SQLite schema!

```typescript
// TypeScript uses Python's database structure
export class SharedDatabase {
  constructor(path: string) {
    this.db = new Database(path);

    // Check schema version matches Python
    const version = this.getSchemaVersion();
    if (version !== EXPECTED_PYTHON_SCHEMA_VERSION) {
      throw new Error(
        `Database schema mismatch. ` +
        `Expected Python schema version ${EXPECTED_PYTHON_SCHEMA_VERSION}, ` +
        `got ${version}`
      );
    }
  }

  // Read and write to same tables as Python
  logResponse(response: Response): void {
    // Use exact same schema as Python
    this.db.run(`
      INSERT INTO responses (
        id, model, prompt, system, response,
        prompt_json, options_json,
        response_json, conversation_id,
        duration_ms, datetime_utc
      ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
    `, [...]);
  }
}
```

### Command Namespacing

**Option A: Separate Commands**
```bash
# Python version (existing)
llm "prompt text"

# TypeScript version
llm-ts "prompt text"

# Or with subcommand
llm ts "prompt text"
```

**Option B: Feature Flags**
```bash
# Use TypeScript for specific features
llm "prompt" --engine=typescript

# Set default engine
llm config set default-engine typescript
```

**Recommendation**: Start with Option A (separate commands), migrate to Option B once TypeScript feature-complete.

---

## Component Migration Order

### Phase 1: Core Infrastructure (Weeks 1-3)

**Component 1.1: Configuration & Key Management**

*Why First?* Foundation for everything else, shared by both versions.

```typescript
// Week 1: Config reader
export class ConfigManager {
  private configDir: string;

  constructor() {
    // Use Python's config directory!
    this.configDir = join(homedir(), '.config', 'io.datasette.llm');
  }

  getKey(provider: string): string | undefined {
    // Read from Python's keys.json
    const keysFile = join(this.configDir, 'keys.json');
    if (!existsSync(keysFile)) return undefined;

    const keys = JSON.parse(readFileSync(keysFile, 'utf-8'));
    return keys[provider];
  }

  getAlias(alias: string): string | undefined {
    // Read from Python's aliases.json
    const aliasesFile = join(this.configDir, 'aliases.json');
    if (!existsSync(aliasesFile)) return undefined;

    const aliases = JSON.parse(readFileSync(aliasesFile, 'utf-8'));
    return aliases[alias];
  }
}
```

**Validation**:
- [ ] Can read API keys set by Python CLI
- [ ] Can read model aliases set by Python CLI
- [ ] Respects same environment variables
- [ ] Shares default model settings

**Component 1.2: Basic Provider Abstraction**

```typescript
// Week 2-3: Model execution
export class OpenAIProvider {
  async execute(prompt: string, options: PromptOptions): Promise<Response> {
    const { generateText } = await import('ai');
    const { openai } = await import('@ai-sdk/openai');

    const result = await generateText({
      model: openai(options.model || 'gpt-4o-mini'),
      prompt,
      system: options.system
    });

    return {
      text: result.text,
      model: options.model || 'gpt-4o-mini',
      usage: result.usage
    };
  }
}
```

**Deliverable**: `llm-ts "prompt"` works with OpenAI models using shared config.

### Phase 2: Core Features (Weeks 4-7)

**Component 2.1: Prompt Execution (Week 4)**

```bash
# Python
llm "What is TypeScript?"

# TypeScript (same behavior)
llm-ts "What is TypeScript?"

# Both log to same database!
llm logs | tail -2  # Shows both Python and TypeScript executions
```

**Implementation**:
```typescript
export async function executePrompt(
  text: string,
  options: CLIOptions,
  config: ConfigManager,
  db: SharedDatabase
): Promise<void> {
  const provider = getProvider(options.model, config);
  const response = await provider.execute(text, options);

  // Log to shared database
  await db.logResponse({
    prompt: text,
    response: response.text,
    model: response.model,
    // ... using Python's schema
  });

  console.log(response.text);
}
```

**Validation**:
- [ ] Same output format as Python
- [ ] Logs appear in `llm logs`
- [ ] Compatible with Python's log viewing
- [ ] Same error handling

**Component 2.2: Streaming (Week 5)**

```typescript
export async function executeStreamingPrompt(
  text: string,
  options: CLIOptions
): Promise<void> {
  const provider = getProvider(options.model);
  const stream = await provider.stream(text, options);

  for await (const chunk of stream) {
    process.stdout.write(chunk);
  }
  console.log(); // Newline
}
```

**Validation**:
- [ ] Streaming matches Python's behavior
- [ ] Same output format
- [ ] Can interrupt with Ctrl+C

**Component 2.3: Conversations (Weeks 6-7)**

```typescript
export class Conversation {
  async save(db: SharedDatabase): Promise<void> {
    // Save using Python's schema
    await db.run(`
      INSERT INTO conversations (id, name, model)
      VALUES (?, ?, ?)
    `, [this.id, this.name, this.model]);

    // Save messages
    for (const msg of this.messages) {
      await db.run(`
        INSERT INTO messages (conversation_id, role, content)
        VALUES (?, ?, ?)
      `, [this.id, msg.role, msg.content]);
    }
  }

  static async load(db: SharedDatabase, id: string): Promise<Conversation> {
    // Load from Python's schema
    // Can load conversations created by Python CLI!
  }
}
```

**Deliverable**: Can continue conversations started in Python CLI!

```bash
# Start in Python
llm chat -m gpt-4o --save my-chat
> Hello
Hello! How can I help you today?
> exit

# Continue in TypeScript!
llm-ts chat --continue my-chat
> What did we just talk about?
We just greeted each other...
```

### Phase 3: Advanced Features (Weeks 8-11)

**Component 3.1: Templates (Week 8-9)**

```bash
# Create template in Python
llm --save summarize -s "You are a summarizer" "Summarize: $input"

# Use in TypeScript!
echo "Long text..." | llm-ts -t summarize
```

```typescript
export class TemplateManager {
  loadFromPython(name: string): Template {
    // Read from Python's templates directory
    const templateFile = join(
      config.userDir,
      'templates',
      `${name}.yaml`
    );

    const content = readFileSync(templateFile, 'utf-8');
    return yaml.parse(content) as Template;
  }
}
```

**Component 3.2: Tools (Week 10)**

**Challenge**: Python tools use Python functions!

**Solution**: TypeScript reimplements core tools, can't use Python-defined tools.

```typescript
// Reimplement default tools in TypeScript
export const defaultTools = {
  llm_version: () => ({ version: '1.0.0-ts' }),
  llm_time: () => ({
    utc: new Date().toISOString(),
    local: new Date().toString()
  })
};
```

**Trade-off**: Python plugins won't work in TypeScript version (expected).

**Component 3.3: Attachments (Week 11)**

```typescript
export class Attachment {
  static fromFile(path: string): Attachment {
    const data = readFileSync(path);
    return new Attachment({
      type: detectMimeType(path),
      content: data
    });
  }

  async save(db: SharedDatabase, responseId: string): Promise<void> {
    // Save using Python's attachments schema
    await db.run(`
      INSERT INTO attachments (id, response_id, type, content)
      VALUES (?, ?, ?, ?)
    `, [ulid(), responseId, this.type, this.content]);
  }
}
```

### Phase 4: Plugin System (Weeks 12-14)

**Component 4.1: Plugin Architecture**

**Decision Point**: TypeScript plugins separate from Python plugins.

```typescript
// TypeScript plugin (won't work with Python CLI)
export const myTSPlugin: TSPlugin = {
  name: 'my-ts-plugin',
  version: '1.0.0',

  registerModels(registry) {
    registry.register('my-model', MyModelFactory);
  }
};
```

**Python plugins continue working in Python CLI**.

**Component 4.2: Embeddings (Weeks 13-14)**

```typescript
export class Collection {
  constructor(name: string, db: SharedDatabase) {
    // Use Python's embeddings schema
    this.name = name;
    this.db = db;
  }

  async embed(id: string, text: string): Promise<void> {
    const embedding = await getEmbedding(text);

    // Store in Python's format
    await this.db.run(`
      INSERT INTO embeddings (id, collection_id, embedding, content)
      VALUES (?, ?, ?, ?)
    `, [id, this.collectionId, serializeEmbedding(embedding), text]);
  }

  async similar(query: string): Promise<SimilarResult[]> {
    // Can search embeddings created by Python CLI!
  }
}
```

---

## Data Compatibility

### Database Schema Compatibility

**Critical Rules**:

1. **Never modify Python's schema from TypeScript**
2. **Read-only access to Python-specific tables** (if any)
3. **Write using exact same format**
4. **Handle NULL values same way**

**Schema Validation**:

```typescript
export class SchemaValidator {
  async validate(db: Database): Promise<ValidationResult> {
    const issues: string[] = [];

    // Check table exists
    const tables = await db.all(
      "SELECT name FROM sqlite_master WHERE type='table'"
    );

    const requiredTables = [
      'responses', 'conversations', 'attachments',
      'tools', 'tool_calls', 'embeddings', 'collections'
    ];

    for (const table of requiredTables) {
      if (!tables.some(t => t.name === table)) {
        issues.push(`Missing table: ${table}`);
      }
    }

    // Check column compatibility
    for (const table of requiredTables) {
      const info = await db.all(`PRAGMA table_info(${table})`);
      // Validate columns match Python version
    }

    return {
      valid: issues.length === 0,
      issues
    };
  }
}
```

### Configuration Format Compatibility

**keys.json** - Exact same format:
```json
{
  "openai": "sk-...",
  "anthropic": "sk-ant-..."
}
```

**aliases.json** - Exact same format:
```json
{
  "4": "gpt-4o",
  "fast": "gpt-4o-mini",
  "claude": "claude-3-5-sonnet-20241022"
}
```

**templates/*.yaml** - Exact same format:
```yaml
name: summarize
prompt: "Summarize: $input"
system: "You are a concise summarizer"
model: gpt-4o-mini
options:
  temperature: 0.7
```

---

## Testing Strategy

### Compatibility Testing

**Test Matrix**:

| Operation | Python First | TypeScript First | Mixed |
|-----------|-------------|------------------|-------|
| Set API key | ✓ Use in TS | ✓ Use in Py | N/A |
| Create alias | ✓ Use in TS | ✓ Use in Py | N/A |
| Save template | ✓ Use in TS | ✓ Use in Py | N/A |
| Start conversation | ✓ Continue in TS | ✓ Continue in Py | ✓ |
| Log response | ✓ View in Py | ✓ View in TS | ✓ |
| Create embedding | ✓ Search in TS | ✓ Search in Py | ✓ |

**Automated Tests**:

```typescript
// tests/compatibility/keys.test.ts
describe('Key Compatibility', () => {
  it('TypeScript can read Python-set keys', async () => {
    // Set key using Python CLI
    await exec('python -m llm keys set test test-key-123');

    // Read in TypeScript
    const config = new ConfigManager();
    const key = config.getKey('test');

    expect(key).toBe('test-key-123');
  });

  it('Python can read TypeScript-set keys', async () => {
    // Set key using TypeScript CLI
    await exec('llm-ts keys set test ts-key-456');

    // Read in Python
    const result = await exec('python -m llm keys list');

    expect(result.stdout).toContain('test');
  });
});

// tests/compatibility/database.test.ts
describe('Database Compatibility', () => {
  it('TypeScript can read Python logs', async () => {
    // Log with Python
    await exec('python -m llm "test prompt" --model gpt-4o-mini');

    // Read with TypeScript
    const db = new SharedDatabase(getDbPath());
    const logs = await db.getLogs({ limit: 1 });

    expect(logs[0].prompt).toBe('test prompt');
  });

  it('Python can read TypeScript logs', async () => {
    // Log with TypeScript
    await exec('llm-ts "ts test prompt"');

    // Read with Python
    const result = await exec('python -m llm logs -n 1');

    expect(result.stdout).toContain('ts test prompt');
  });
});
```

### Integration Testing

**Scenario Testing**:

```typescript
describe('End-to-End Scenarios', () => {
  it('Complete workflow across both CLIs', async () => {
    // 1. Set key in Python
    await pythonCLI('keys set openai sk-test');

    // 2. Create template in Python
    await pythonCLI('--save greet "Hello $name"');

    // 3. Use template in TypeScript
    const result = await tsCLI('-t greet', { stdin: 'Alice' });
    expect(result.stdout).toContain('Alice');

    // 4. View log in Python
    const logs = await pythonCLI('logs -n 1');
    expect(logs.stdout).toContain('greet');
  });
});
```

---

## Rollout Plan

### Stage 1: Alpha (Weeks 1-4)

**Goal**: Basic functionality working, shared config

**Features**:
- ✅ Basic prompt execution
- ✅ Shared API keys
- ✅ Shared database logging

**User Base**: Internal testing only

**Installation**:
```bash
npm install -g llm-ts@alpha
```

**Feedback Focus**: Configuration compatibility

### Stage 2: Beta (Weeks 5-8)

**Goal**: Core features complete, stable API

**Features**:
- ✅ Conversations
- ✅ Streaming
- ✅ Templates
- ✅ Multiple providers

**User Base**: Early adopters, documented as beta

**Installation**:
```bash
npm install -g llm-ts@beta
```

**Feedback Focus**: Feature parity, performance

### Stage 3: Release Candidate (Weeks 9-12)

**Goal**: Feature-complete, production-ready

**Features**:
- ✅ All core features
- ✅ Comprehensive tests
- ✅ Documentation

**User Base**: Public beta

**Installation**:
```bash
npm install -g llm-ts@rc
```

**Feedback Focus**: Bugs, edge cases

### Stage 4: Stable Release (Week 13+)

**Goal**: Production release, recommended for new users

**Features**:
- ✅ All features working
- ✅ < 5 known bugs
- ✅ Full documentation

**User Base**: General availability

**Installation**:
```bash
npm install -g llm-ts
# or
llm install-ts  # Helper in Python version
```

**Migration Path**: Python version continues working, gradual deprecation over 12 months

---

## Migration Decision Points

### When to Use TypeScript Version?

**Recommend TypeScript For**:
- New users
- Features better in TypeScript (e.g., better Node.js integration)
- When needing programmatic API
- Cross-platform consistency

**Recommend Python For**:
- Existing workflows
- Python-specific plugins
- Until TypeScript reaches feature parity

### Deprecation Timeline

**Year 1**: Both versions supported
- Active development on TypeScript
- Bug fixes only for Python
- Clear migration guides

**Year 2**: Python in maintenance mode
- Security fixes only
- TypeScript is recommended
- Python still available

**Year 3**: Python deprecated
- TypeScript only
- Python archived
- Clear migration tools available

---

## Success Criteria

### Technical Success

**Compatibility** (Critical):
- [ ] 100% config file compatibility
- [ ] 100% database schema compatibility
- [ ] Can migrate between versions freely

**Feature Parity** (Important):
- [ ] 90%+ feature coverage
- [ ] All core features working
- [ ] Performance within 20% of Python

**Quality** (Important):
- [ ] < 10 compatibility bugs
- [ ] Test coverage >75%
- [ ] Documentation complete

### User Success

**Adoption** (3 months):
- [ ] 20% of Python users try TypeScript
- [ ] 10% switch primarily to TypeScript
- [ ] Positive feedback (>70% satisfaction)

**Adoption** (6 months):
- [ ] 50% of users using TypeScript
- [ ] New users default to TypeScript
- [ ] Community plugins for TypeScript

**Adoption** (12 months):
- [ ] 80% using TypeScript primarily
- [ ] Python version in maintenance mode
- [ ] Clear path to full migration

---

## Risk Management

### Technical Risks

**Risk: Schema divergence**
- **Impact**: Data corruption, compatibility broken
- **Likelihood**: Medium
- **Mitigation**: Schema validation on every start, comprehensive tests
- **Fallback**: Schema version locking, refuse to start if mismatch

**Risk: Performance regression**
- **Impact**: Users frustrated, don't adopt
- **Likelihood**: Low
- **Mitigation**: Benchmarks, profiling, optimization
- **Fallback**: Document performance differences, optimize critical paths

**Risk: Feature gaps**
- **Impact**: Users can't migrate
- **Likelihood**: High (initially)
- **Mitigation**: Clear feature matrix, hybrid workflows
- **Fallback**: Keep Python for missing features

### Process Risks

**Risk: Dual maintenance burden**
- **Impact**: Burnout, bugs in both versions
- **Likelihood**: High
- **Mitigation**: Clear ownership, automated tests, feature freeze on Python
- **Fallback**: Extend timeline, reduce scope

**Risk: User confusion**
- **Impact**: Support burden, poor experience
- **Likelihood**: Medium
- **Mitigation**: Clear docs, version indicators, migration guide
- **Fallback**: Better error messages, troubleshooting guide

---

## Comparison with Other Approaches

| Aspect | Approach 1 | Approach 2 | Approach 3 | Approach 4 |
|--------|-----------|-----------|-----------|-----------|
| Risk | Low | Medium | Medium | Very Low |
| Time to Value | Fast | Slow | Medium | Continuous |
| Maintenance Burden | Low | Medium | Medium | High (temporarily) |
| User Disruption | None (new tool) | None (new tool) | None (new tool) | Minimal |
| Compatibility | N/A | N/A | N/A | Critical |
| Best For | Learning | Greenfield | Library | Migration |

---

## Conclusion

This approach minimizes risk by allowing incremental validation and continuous user feedback. While it has the longest total timeline and highest temporary complexity (maintaining both versions), it provides the safest path for existing Python users.

**Key Advantages**:
- Lowest risk
- Continuous validation
- Users can migrate at their own pace
- Can pivot if issues found
- Maintains backward compatibility

**Key Challenges**:
- Dual maintenance burden
- Longer total time
- Complex compatibility testing
- User confusion possible
- Requires discipline to complete

**Best For**:
- Established tools with users
- Risk-averse organizations
- Teams that prefer incremental progress
- When backward compatibility is critical
- When you have time for a gradual migration

**Success Factors**:
- Strict compatibility testing
- Clear communication
- Regular releases
- Active user feedback
- Commitment to completion

This is the "safest" approach but requires the most patience and discipline to execute well.
