# Efficiency Analysis: bun-kernel32

**Date**: 2026-02-03
**Analyst**: Claude
**Scope**: Comprehensive review of performance and efficiency optimizations

---

## Executive Summary

This analysis identifies **7 efficiency improvements** for the bun-kernel32 library, ordered by impact. The library's architecture is sound, but several opportunities exist to reduce overhead without sacrificing correctness.

---

## 1. Method Invocation Overhead (HIGH IMPACT)

### Current Implementation

Every public static method follows this pattern:

```typescript
public static GetCurrentProcessId(): DWORD {
  return Kernel32.Load('GetCurrentProcessId')();
}
```

The `Load()` method performs:
```typescript
private static Load<T extends keyof typeof Kernel32.Symbols>(method: T): (typeof Kernel32)[T] {
  const skip = Object.getOwnPropertyDescriptor(Kernel32, method)?.configurable === false;
  if (skip) {
    return Kernel32[method];
  }
  // ... dlopen and memoize
}
```

### Problem

Even after the native function is memoized, **every call** still:
1. Enters the wrapper method
2. Calls `Load()`
3. Calls `Object.getOwnPropertyDescriptor()` to check if already bound
4. Returns `Kernel32[method]` (property access)
5. Invokes the function with spread semantics

For hot paths like `GetLastError()` or `GetTickCount64()`, this adds measurable overhead.

### Recommended Solution

Replace the static method itself with the native function after first invocation:

```typescript
public static GetCurrentProcessId(): DWORD {
  const fn = Kernel32.LoadAndBind('GetCurrentProcessId');
  return fn();
}

private static LoadAndBind<T extends keyof typeof Kernel32.Symbols>(method: T) {
  const library = dlopen('kernel32.dll', { [method]: Kernel32.Symbols[method] });
  const nativeFn = library.symbols[method];

  // Replace the static method entirely
  Object.defineProperty(Kernel32, method, {
    configurable: false,
    writable: false,
    value: nativeFn
  });

  return nativeFn;
}
```

After first call, `Kernel32.GetCurrentProcessId` points directly to the native function with **zero wrapper overhead**.

### Impact
- **Before**: 5+ operations per call (method enter, Load call, descriptor check, property access, function call)
- **After**: 1 operation per call (direct native function call)
- **Estimated speedup**: 3-5x for frequently called APIs

---

## 2. Preload Method Efficiency (MEDIUM IMPACT)

### Current Implementation

```typescript
public static Preload(methods?: (keyof typeof Kernel32.Symbols)[]): void {
  methods ??= Object.keys(Kernel32.Symbols) as (keyof typeof Kernel32.Symbols)[];

  const symbols = Object.fromEntries(
    methods
      .filter((method) => Object.getOwnPropertyDescriptor(Kernel32, method)?.configurable !== false)
      .map((method) => [method, Kernel32.Symbols[method]])
  );

  const library = dlopen('kernel32.dll', symbols);

  const propertyDescriptorMap = Object.fromEntries(
    Object.entries(library.symbols).map(([key, value]) => [key, { configurable: false, value }])
  );

  Object.defineProperties(Kernel32, propertyDescriptorMap);
}
```

### Problems

1. **Two `Object.fromEntries()` calls** with intermediate arrays
2. **Unnecessary filter/map chain** that iterates twice
3. **Descriptor check per method** even in bulk operation

### Recommended Solution

Single-pass with pre-allocated objects:

```typescript
public static Preload(methods?: (keyof typeof Kernel32.Symbols)[]): void {
  methods ??= Object.keys(Kernel32.Symbols) as (keyof typeof Kernel32.Symbols)[];

  const symbols: Record<string, FFIFunction> = {};
  const toLoad: string[] = [];

  for (const method of methods) {
    if (Object.getOwnPropertyDescriptor(Kernel32, method)?.configurable !== false) {
      symbols[method] = Kernel32.Symbols[method];
      toLoad.push(method);
    }
  }

  if (toLoad.length === 0) return;

  const library = dlopen('kernel32.dll', symbols);

  for (const method of toLoad) {
    Object.defineProperty(Kernel32, method, {
      configurable: false,
      value: library.symbols[method]
    });
  }
}
```

### Impact
- **Memory**: Eliminates 2 intermediate arrays
- **CPU**: Single iteration instead of 3 (filter + map + entries)
- **Estimated speedup**: 2x for `Preload()` calls

---

## 3. Symbols Object Memory (MEDIUM IMPACT)

### Current State

The `Symbols` object contains 1,402 FFI signatures, all loaded into memory at import time:

```typescript
private static readonly Symbols = {
  _hread: { args: [FFIType.i32, FFIType.ptr, FFIType.i32], returns: FFIType.i32 },
  _hwrite: { args: [FFIType.i32, FFIType.ptr, FFIType.i32], returns: FFIType.i32 },
  // ... 1,400+ more entries
};
```

**Estimated memory**: ~100-150KB for the Symbols object alone (each entry has args array + returns + object overhead).

### Considerations

This is a classic **lazy loading vs. memory** tradeoff:
- Current approach prioritizes lazy function binding but pays upfront memory cost
- Alternative: Code-generate symbols on demand or split into logical groups

### Potential Optimization

If memory is critical, split symbols by Windows API category:

```typescript
// symbols/process.ts
export const ProcessSymbols = {
  GetCurrentProcessId: { args: [], returns: FFIType.u32 },
  GetCurrentProcess: { args: [], returns: FFIType.u64 },
  // ...
};

// symbols/file.ts
export const FileSymbols = {
  CreateFileW: { args: [...], returns: FFIType.u64 },
  // ...
};
```

Users import only what they need:
```typescript
import { ProcessKernel32 } from 'bun-kernel32/process';
```

### Trade-off
- **Pro**: Pay-what-you-use memory model
- **Con**: API fragmentation, more complex imports
- **Recommendation**: Only pursue if memory profiling shows this is a real issue

---

## 4. Extensions Pointer Getter (LOW IMPACT)

### Current Implementation

```typescript
Object.defineProperty(prototype, 'ptr', {
  configurable: false,
  enumerable: false,
  get(this): Pointer {
    return ptr(this);
  },
});
```

### Analysis

The `ptr()` function from `bun:ffi` is called on every `.ptr` access. For code patterns like:

```typescript
const buffer = new Uint8Array(1024);
Kernel32.ReadFile(handle, buffer.ptr, buffer.length, bytesRead.ptr, null);
Kernel32.WriteFile(handle, buffer.ptr, buffer.length, bytesWritten.ptr, null);
```

The `buffer.ptr` is computed twice.

### Recommendation

This is **correct behavior** and should likely remain as-is because:
1. Bun's `ptr()` is extremely fast (direct memory address lookup)
2. Caching would require WeakMap or Symbol storage, adding overhead
3. Users can cache manually when needed: `const bufferPtr = buffer.ptr;`

**No change recommended** - current implementation is optimal.

---

## 5. TypeScript Type Generation (LOW IMPACT)

### Observation

The types file has 130+ type aliases that are all `= Pointer`:

```typescript
export type LPVOID = Pointer;
export type LPWSTR = Pointer;
export type LPCSTR = Pointer;
export type LPBYTE = Pointer;
// ... many more
```

### Analysis

These aliases exist for **documentation and API clarity**, not runtime efficiency. TypeScript erases all types at compile time, so there's zero runtime cost.

**No change needed** - this is correct design.

---

## 6. Duplicate dlopen Calls (CORRECTNESS NOTE)

### Current Behavior

When `Load()` is called for the same method twice before memoization completes (race condition), `dlopen()` could theoretically be called twice.

### Analysis

In practice:
1. JavaScript is single-threaded, so no race condition exists
2. The `configurable === false` check prevents duplicate binding
3. `dlopen()` on the same library is cached by the OS

**No change needed** - this is not a real issue.

---

## 7. Missing Test Suite (QUALITY)

### Observation

The repository has no automated tests. While not strictly an "efficiency" concern, bugs lead to inefficient debugging cycles.

### Recommendation

Add a test suite using Bun's built-in test runner:

```typescript
// test/kernel32.test.ts
import { test, expect } from 'bun:test';
import Kernel32 from '../index';

test('GetCurrentProcessId returns non-zero', () => {
  const pid = Kernel32.GetCurrentProcessId();
  expect(pid).toBeGreaterThan(0);
});

test('GetTickCount64 increases over time', async () => {
  const t1 = Kernel32.GetTickCount64();
  await Bun.sleep(10);
  const t2 = Kernel32.GetTickCount64();
  expect(t2).toBeGreaterThan(t1);
});
```

---

## Summary of Recommendations

| # | Issue | Impact | Effort | Recommendation |
|---|-------|--------|--------|----------------|
| 1 | Method invocation overhead | HIGH | Medium | Replace wrapper with native fn after first call |
| 2 | Preload iteration inefficiency | MEDIUM | Low | Single-pass implementation |
| 3 | Symbols object memory | MEDIUM | High | Consider only if memory is constrained |
| 4 | Extensions pointer getter | LOW | - | No change needed |
| 5 | Type aliases | LOW | - | No change needed |
| 6 | dlopen race condition | N/A | - | Not a real issue |
| 7 | Missing tests | QUALITY | Medium | Add test suite |

---

## Prioritized Action Items

1. **Immediate**: Implement method self-replacement pattern (Issue #1)
2. **Short-term**: Optimize Preload() implementation (Issue #2)
3. **Long-term**: Add comprehensive test suite (Issue #7)
4. **Deferred**: Evaluate symbols splitting based on real-world memory profiling

---

## Benchmarking Suggestion

Before and after implementing changes, benchmark with:

```typescript
const iterations = 1_000_000;

// Warm up
Kernel32.GetCurrentProcessId();

const start = performance.now();
for (let i = 0; i < iterations; i++) {
  Kernel32.GetCurrentProcessId();
}
const elapsed = performance.now() - start;

console.log(`${iterations} calls in ${elapsed.toFixed(2)}ms`);
console.log(`${(iterations / elapsed * 1000).toFixed(0)} calls/sec`);
```

This will quantify the actual impact of the wrapper overhead.
