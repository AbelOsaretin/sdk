# Error Handling and Recovery

The Dorisio SDK provides custom error handlers and error recovery strategies to help you implement robust error handling in your application.

## Custom Error Handlers

You can register a custom error handler to implement custom error recovery logic such as retrying with custom delays, implementing fallback strategies, or circuit breaker patterns.

### Basic Usage

```typescript
import { DorisioClient } from 'dorisio-sdk';

const client = new DorisioClient({
  baseUrl: 'https://api.dorisio.com',
  token: 'your-token',
  errorHandler: async (error, context) => {
    if (error.statusCode === 429) {
      // Custom rate limit handling
      return { action: 'retry', delayMs: 5000 };
    }
    if (error.statusCode && error.statusCode >= 500) {
      // Custom server error handling
      return { action: 'fallback', fallbackValue: { cached: true } };
    }
    return { action: 'throw' };
  },
});
```

### Error Handler Actions

The error handler can return one of three actions:

- **`{ action: 'retry', delayMs?: number }`** - Retry the request with an optional custom delay
- **`{ action: 'fallback', fallbackValue: unknown }`** - Return a fallback value instead of throwing the error
- **`{ action: 'throw' }`** - Throw the error as normal

### Error Context

The error handler receives context information about the request:

```typescript
interface ErrorHandlerContext {
  method: 'GET' | 'POST' | 'PUT' | 'PATCH' | 'DELETE';
  path: string;
  body?: unknown;
  headers?: Record<string, string>;
  attempt?: number;
  requestId?: string;
}
```

### Registering Error Handlers

You can also register error handlers after creating the client:

```typescript
client.onError(async (error, context) => {
  console.error('Error occurred:', error, context);
  if (error.statusCode === 429) {
    return { action: 'retry', delayMs: 1000 };
  }
  return { action: 'throw' };
});
```

## Common Patterns

### Rate Limit Handling

```typescript
const client = new DorisioClient({
  baseUrl: 'https://api.dorisio.com',
  errorHandler: async (error) => {
    if (error.statusCode === 429) {
      // Exponential backoff for rate limits
      const delayMs = Math.pow(2, (context.attempt || 1)) * 1000;
      return { action: 'retry', delayMs };
    }
    return { action: 'throw' };
  },
});
```

### Circuit Breaker Pattern

```typescript
let failureCount = 0;
const FAILURE_THRESHOLD = 5;
const RESET_TIMEOUT = 60000; // 1 minute

const client = new DorisioClient({
  baseUrl: 'https://api.dorisio.com',
  errorHandler: async (error, context) => {
    if (error.statusCode && error.statusCode >= 500) {
      failureCount++;
      if (failureCount >= FAILURE_THRESHOLD) {
        // Circuit is open, return fallback
        return { action: 'fallback', fallbackValue: { fromCache: true } };
      }
      return { action: 'retry', delayMs: 1000 };
    }
    // Reset on success
    failureCount = 0;
    return { action: 'throw' };
  },
});
```

### Fallback Strategies

```typescript
const client = new DorisioClient({
  baseUrl: 'https://api.dorisio.com',
  errorHandler: async (error, context) => {
    if (error.statusCode === 503) {
      // Service unavailable, return cached data
      return { action: 'fallback', fallbackValue: getCachedData(context.path) };
    }
    return { action: 'throw' };
  },
});
```

## Async Error Handlers

Error handlers support async operations:

```typescript
const client = new DorisioClient({
  baseUrl: 'https://api.dorisio.com',
  errorHandler: async (error, context) => {
    // Perform async operations
    await logErrorToService(error, context);
    await sendAlert(error);

    if (error.statusCode === 429) {
      return { action: 'retry', delayMs: 5000 };
    }
    return { action: 'throw' };
  },
});
```

## Error Handler Failure

If the error handler itself throws an error, the SDK will fall back to normal error handling and the original error will be thrown:

```typescript
const client = new DorisioClient({
  baseUrl: 'https://api.dorisio.com',
  errorHandler: async (error) => {
    // If this throws, the original error will be thrown
    throw new Error('Handler failed');
  },
});
```
