---
description: TypeScript coding guidelines / style – check when working on TypeScript files
globs: *.ts, *.tsx
alwaysApply: false
---
# General

- Prefer 'const myFunc = () => {}' over function declarations.
- Parallelize async calls where appropriate via `Promise.all` instead of sequential fetches in a `for` loop.
- No `any` allowed. If the input type is genuinely unknown, use `unknown` and write appropriate logic to narrow the type.
- Prefer `??` over `||`
- No unchecked index access (`array[0]` might be `undefined`). Optional chain or check and throw and error (depending on whether it is acceptable for the result to be undefined or not).
- Don't make formatting corrections to lines you aren't already modifying. Auto-fixing will handle it.
- Avoid single-letter variable names. e.g. `event`, not `e`. Exception: in `for` loops (e.g. `i` is fine).
- Variable names are all in `camelCase`. No `SCREAMING_SNAKE_CASE`.
- Check function and type signatures carefully to understand APIs and what fields are available.
  For external libraries, these should be available in a type declaration file.
  Don't guess APIs! Check the signatures.

# Avoid these!

- Don't make changes unrelated to the user's requests. You can suggest further changes at the end of your work.
- Don't get stuck fixing linting errors – stop and ask for help sooner.
- Don't attempt to run the app. Let the user handle testing.

# TypeScript Configuration for Bun

- Use Bun-optimized tsconfig.json:
  ```json
  {
    "compilerOptions": {
      // TypeScript strict mode
      "strict": true,
      "strictNullChecks": true,
      "strictFunctionTypes": true,
      "strictBindCallApply": true,
      "strictPropertyInitialization": true,
      "noImplicitThis": true,
      "alwaysStrict": true,
      
      // Bun-specific settings
      "types": ["bun-types"],
      "moduleResolution": "bundler",
      "module": "esnext",
      "target": "esnext",
      "lib": ["esnext"],
      "jsx": "react-jsx",
      
      // Additional safety
      "noUncheckedIndexedAccess": true,
      "exactOptionalPropertyTypes": true,
      "noImplicitOverride": true,
      "noUnusedLocals": true,
      "noUnusedParameters": true,
      "noFallthroughCasesInSwitch": true,
      
      // Module handling
      "esModuleInterop": true,
      "skipLibCheck": true,
      "forceConsistentCasingInFileNames": true,
      "resolveJsonModule": true,
      "allowImportingTsExtensions": true,
      "moduleDetection": "force"
    }
  }
  ```

# Bun-Specific APIs and Patterns

## File Operations

- Use Bun's file API for optimal performance:
  ```typescript
  // Reading files
  const file = Bun.file("./config.json");
  const text = await file.text();
  const json = await file.json();
  const arrayBuffer = await file.arrayBuffer();
  
  // Writing files
  await Bun.write("./output.txt", "Hello World");
  await Bun.write("./data.json", JSON.stringify(data));
  await Bun.write("./binary", new Uint8Array([1, 2, 3]));
  
  // Streaming large files
  const file = Bun.file("./large.csv");
  const stream = file.stream();
  for await (const chunk of stream) {
    process(chunk);
  }
  ```

## HTTP Server

- Use Bun.serve for high-performance servers:
  ```typescript
  Bun.serve({
    port: 3000,
    hostname: "0.0.0.0",
    
    async fetch(req: Request): Promise<Response> {
      const url = new URL(req.url);
      
      // Routing
      if (url.pathname === "/api/users" && req.method === "GET") {
        return Response.json(await getUsers());
      }
      
      if (url.pathname === "/health") {
        return new Response("OK");
      }
      
      return new Response("Not Found", { status: 404 });
    },
    
    error(error: Error): Response {
      console.error(error);
      return new Response("Internal Server Error", { status: 500 });
    },
  });
  ```

## Built-in Database Operations

- Use Bun's built-in SQLite driver:
  ```typescript
  import { Database } from "bun:sqlite";
  
  const db = new Database("app.db");
  
  // Use prepared statements
  const insertUser = db.prepare(
    "INSERT INTO users (name, email) VALUES ($name, $email)"
  );
  
  const getUser = db.prepare(
    "SELECT * FROM users WHERE id = $id"
  );
  
  // Transactions
  db.transaction(() => {
    for (const user of users) {
      insertUser.run(user);
    }
  })();
  
  // Always close when done
  db.close();
  ```

## Environment Variables

- Use Bun's built-in env handling:
  ```typescript
  // Access environment variables
  const apiKey = Bun.env.API_KEY;
  const port = Bun.env.PORT || "3000";
  
  // Validate required env vars at startup
  const requiredEnvVars = ["DATABASE_URL", "API_KEY", "JWT_SECRET"];
  for (const envVar of requiredEnvVars) {
    if (!Bun.env[envVar]) {
      throw new Error(`Missing required environment variable: ${envVar}`);
    }
  }
  ```

# Testing with Bun

- Use Bun's built-in test runner:
  ```typescript
  import { describe, it, expect, beforeEach, afterEach } from "bun:test";
  
  describe("UserService", () => {
    let service: UserService;
    
    beforeEach(() => {
      service = new UserService();
    });
    
    it("should create a user", async () => {
      const user = await service.create({
        name: "Test User",
        email: "test@example.com"
      });
      
      expect(user.id).toBeDefined();
      expect(user.name).toBe("Test User");
    });
    
    it("should validate email format", () => {
      expect(() => {
        service.create({ name: "Test", email: "invalid" });
      }).toThrow("Invalid email format");
    });
  });
  
  // Run with: bun test
  // Or specific file: bun test user.test.ts
  // Watch mode: bun test --watch
  ```

# CLI Development with Bun

- Build CLIs leveraging Bun's fast startup:
  ```typescript
  #!/usr/bin/env bun
  
  import { parseArgs } from "util";
  
  const { values, positionals } = parseArgs({
    args: Bun.argv,
    options: {
      verbose: {
        type: 'boolean',
        short: 'v',
      },
      output: {
        type: 'string',
        short: 'o',
      },
    },
    strict: true,
    allowPositionals: true,
  });
  
  // Make it executable
  // chmod +x ./my-cli.ts
  // Run directly: ./my-cli.ts
  ```

# Error Handling

- Create typed error classes:
  ```typescript
  export class AppError extends Error {
    constructor(
      message: string,
      public readonly code: string,
      public readonly statusCode?: number,
      options?: ErrorOptions
    ) {
      super(message, options);
      this.name = 'AppError';
    }
  }
  
  // Domain-specific errors
  export class ValidationError extends AppError {
    constructor(message: string, public readonly fields: Record<string, string>) {
      super(message, 'VALIDATION_ERROR', 400);
    }
  }
  
  // Usage with proper error handling
  try {
    const result = await riskyOperation();
    return result;
  } catch (error) {
    if (error instanceof ValidationError) {
      return Response.json({ 
        error: error.message, 
        fields: error.fields 
      }, { status: 400 });
    }
    
    console.error('Unexpected error:', error);
    return new Response("Internal Server Error", { status: 500 });
  }
  ```

# Performance Optimization

- Use Bun's performance APIs:
  ```typescript
  // Hashing
  const hash = Bun.hash("my-string");
  const cryptoHash = await Bun.sha256("my-string");
  
  // Password hashing
  const hashedPassword = await Bun.password.hash("user-password");
  const isValid = await Bun.password.verify("user-password", hashedPassword);
  
  // Fast deep equality checks
  const isEqual = Bun.deepEquals(obj1, obj2);
  
  // Memory-efficient string operations
  const encoder = new TextEncoder();
  const decoder = new TextDecoder();
  ```

# Async Patterns

- Use modern async patterns:
  ```typescript
  // Concurrent operations with error handling
  const results = await Promise.allSettled([
    fetchUserData(userId),
    fetchUserPosts(userId),
    fetchUserSettings(userId),
  ]);
  
  const data = results.reduce((acc, result) => {
    if (result.status === 'fulfilled') {
      return { ...acc, ...result.value };
    } else {
      console.error('Failed to fetch:', result.reason);
      return acc;
    }
  }, {});
  
  // Timeout wrapper
  async function withTimeout<T>(
    promise: Promise<T>,
    timeoutMs: number
  ): Promise<T> {
    const timeout = new Promise<never>((_, reject) =>
      setTimeout(() => reject(new Error('Operation timed out')), timeoutMs)
    );
    
    return Promise.race([promise, timeout]);
  }
  ```

# Package Management

- Use Bun's fast package manager:
  ```bash
  # Install dependencies
  bun install
  
  # Add dependencies
  bun add zod
  bun add -d @types/node
  
  # Scripts in package.json
  {
    "scripts": {
      "dev": "bun run --watch src/index.ts",
      "build": "bun build src/index.ts --outdir=dist --target=bun",
      "test": "bun test",
      "typecheck": "tsc --noEmit"
    }
  }
  ```

# Security Best Practices

- Input validation with Zod:
  ```typescript
  import { z } from 'zod';
  
  const UserSchema = z.object({
    name: z.string().min(1).max(100),
    email: z.string().email(),
    age: z.number().int().positive().max(120).optional(),
  });
  
  export async function createUser(input: unknown) {
    const validated = UserSchema.parse(input); // Throws ZodError if invalid
    
    // Hash password before storage
    if (validated.password) {
      validated.password = await Bun.password.hash(validated.password);
    }
    
    return await db.insert(users).values(validated);
  }
  ```

# Logging

- Structured logging pattern:
  ```typescript
  class Logger {
    constructor(private context: Record<string, any> = {}) {}
    
    private log(level: string, message: string, data?: Record<string, any>) {
      console.log(JSON.stringify({
        timestamp: new Date().toISOString(),
        level,
        message,
        ...this.context,
        ...data,
      }));
    }
    
    info(message: string, data?: Record<string, any>) {
      this.log('info', message, data);
    }
    
    error(message: string, error?: Error, data?: Record<string, any>) {
      this.log('error', message, {
        ...data,
        error: error ? {
          message: error.message,
          stack: error.stack,
          cause: error.cause,
        } : undefined,
      });
    }
    
    child(context: Record<string, any>) {
      return new Logger({ ...this.context, ...context });
    }
  }
  
  const logger = new Logger({ service: 'api' });
  const requestLogger = logger.child({ requestId: crypto.randomUUID() });
  ```

# Build and Deployment

- Optimize for production:
  ```typescript
  // bunfig.toml
  [install]
  production = true
  frozen-lockfile = true
  
  [build]
  minify = true
  sourcemap = "external"
  target = "bun"
  
  // Build script
  await Bun.build({
    entrypoints: ['./src/index.ts'],
    outdir: './dist',
    target: 'bun',
    minify: true,
    sourcemap: 'external',
    external: ['sqlite3'], // Mark native deps as external
  });
  ```

# WebSocket Support

- Use Bun's native WebSocket:
  ```typescript
  Bun.serve({
    port: 3000,
    
    fetch(req, server) {
      if (server.upgrade(req)) {
        return; // WebSocket upgrade successful
      }
      return new Response("Expected WebSocket", { status: 400 });
    },
    
    websocket: {
      open(ws) {
        ws.subscribe("chat");
        ws.send(JSON.stringify({ type: "welcome" }));
      },
      
      message(ws, message) {
        const data = JSON.parse(message as string);
        ws.publish("chat", JSON.stringify({
          ...data,
          timestamp: Date.now(),
        }));
      },
      
      close(ws) {
        ws.unsubscribe("chat");
      },
    },
  });
  ```
