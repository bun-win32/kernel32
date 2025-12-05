# Fix Windows Handle Types in bun-\* FFI Packages

This package has incorrect FFI type definitions for Windows handles. On 64-bit Windows, handles (HWND, HDC, HBITMAP, HFONT, HGLRC, HMENU, HINSTANCE, HMODULE, HGDIOBJ, HBRUSH, HPEN, HRGN, HICON, HCURSOR, etc.) are 64-bit values that must be passed as `FFIType.u64`, not `FFIType.ptr`.

Using `FFIType.ptr` causes sign-extension issues where a handle like `570502638` becomes `18446744073004980000`, resulting in ERROR_INVALID_HANDLE (6) when passed to other Windows APIs.

## Task

1. In `structs/*.ts`, find all FFI symbol definitions in the `Symbols` object where handle types are used as parameters or return values.

2. Change `FFIType.ptr` to `FFIType.u64` for these handle types:

   - HWND, HDC, HBITMAP, HFONT, HGLRC, HMENU, HINSTANCE, HMODULE
   - HGDIOBJ, HBRUSH, HPEN, HRGN, HPALETTE, HMETAFILE, HENHMETAFILE
   - HICON, HCURSOR, HACCEL, HDWP, HHOOK, HMONITOR, HWINEVENTHOOK
   - HDESK, HWINSTA, HKL, HTOUCHINPUT, HGESTUREINFO, HRAWINPUT
   - HGLRC (OpenGL render context)
   - Any other `H*` handle types

3. In `types/*.ts`, update the corresponding TypeScript type aliases to use `bigint` instead of `Pointer` for handle types.

4. Keep `FFIType.ptr` for actual pointer types that point to data structures (like `LPVOID`, `LPCWSTR`, `LPARAM` when used as a pointer, struct pointers, etc.).

## Example Transformation

```typescript
// BEFORE (incorrect):
GetDC: { args: [FFIType.ptr], returns: FFIType.ptr },
CreateWindowExW: { args: [..., FFIType.ptr, FFIType.ptr, FFIType.ptr, FFIType.ptr], returns: FFIType.ptr },

// AFTER (correct):
GetDC: { args: [FFIType.u64], returns: FFIType.u64 },
CreateWindowExW: { args: [..., FFIType.u64, FFIType.u64, FFIType.u64, FFIType.ptr], returns: FFIType.u64 },
//                         hWndParent^  hMenu^  hInstance^  lpParam^ (stays ptr - it's data)
```

## Rule of Thumb

- If the parameter name starts with `h` and represents a Windows handle → `FFIType.u64`
- If it's `NULL`/`0` when unused → `FFIType.u64` (handles accept 0/null as bigint `0n`)
- If it points to a struct, string, or buffer → `FFIType.ptr`

## Applies To

- `bun-gdi32`
- `bun-user32`
- `bun-opengl32`
- `bun-kernel32`

## After Making Changes

1. Commit the changes:

   ```bash
   git add -A && git commit -m "fix: use FFIType.u64 for Windows handle types"
   ```

2. Push to remote:

   ```bash
   git push
   ```

3. Publish to npm:
   ```bash
   bun publish
   ```
