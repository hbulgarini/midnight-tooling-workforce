# Midnight SDK Browser & Mobile Compatibility Analysis

This document analyzes the Midnight SDK's technical requirements and platform incompatibilities when running in browser and mobile environments.

---

## Executive Summary

The Midnight SDK packages are designed primarily for **Node.js environments**. To run them in the browser, developers must:

1. **Polyfill Node.js APIs** (`Buffer`, `process`) that don't exist in browsers
2. **Handle WebAssembly modules** that contain core cryptographic primitives
3. **Transform CommonJS modules** to work with ESM-native bundlers
4. **Support top-level await** for async WASM initialization

**Key Finding:** WASM is not optional—it is the **only viable method** to run Midnight's cryptographic stack in browsers. The alternatives (pure JavaScript, WebCrypto API, native extensions) are either too slow, incompatible, or not cross-platform.

**Mobile Finding:** The current WASM-based architecture is fundamentally incompatible with iOS native app development due to Apple's JIT restrictions on JavaScriptCore.

---

## Table of Contents

1. [Vite Configuration Overview](#1-vite-configuration-overview)
2. [Buffer Polyfill](#2-buffer-polyfill)
3. [Process Polyfill](#3-process-polyfill)
4. [WebAssembly (WASM) Modules](#4-webassembly-wasm-modules)
5. [CommonJS Compatibility](#5-commonjs-compatibility)
6. [Additional Configuration](#6-additional-configuration)
7. [Package Dependencies Summary](#7-package-dependencies-summary)
8. [Performance Considerations](#8-performance-considerations)
9. [Troubleshooting Common Issues](#9-troubleshooting-common-issues)
10. [Deep Dive: Why WASM is Required](#10-deep-dive-why-wasm-is-required)
11. [Midnight Ledger: Rust Implementation Analysis](#11-midnight-ledger-rust-implementation-analysis)
12. [Cryptographic Primitives](#12-cryptographic-primitives)
13. [Alternatives to WASM: Analysis](#13-alternatives-to-wasm-analysis)
14. [Future Considerations](#14-future-considerations)
15. [Mobile Platform Incompatibility and Potential Fixes](#15-mobile-platform-incompatibility-and-potential-fixes)
16. [Active Implementation: ledger-uniffi (Native FFI for Mobile)](#16-active-implementation-ledger-uniffi-native-ffi-for-mobile)
17. [Alternative Approach: Polygen (WASM AOT Compilation)](#17-alternative-approach-polygen-wasm-aot-compilation)
18. [Alternative Approach: Manual Rust FFI (Direct Native Modules)](#18-alternative-approach-manual-rust-ffi-direct-native-modules)

---

## 1. Vite Configuration Overview

```typescript
// vite.config.ts
plugins: [
  nodePolyfills({
    include: ['buffer', 'process'],
    globals: { Buffer: true, process: true },
  }),
  wasm(),                    // WebAssembly support
  react(),                   // React JSX transform
  viteCommonjs(),           // CommonJS compatibility
  topLevelAwait(),          // Async module initialization
],
```

---

## 2. Buffer Polyfill

### Why It's Needed

`Buffer` is a Node.js global class for binary data manipulation. Browsers have `Uint8Array` and `ArrayBuffer`, but not `Buffer`. The Midnight SDK uses `Buffer` extensively for:

- Hex encoding/decoding
- Cryptographic operations
- Address formatting
- Serialization

### Packages That Use Buffer

| Package | Usage | Purpose |
|---------|-------|---------|
| `@midnight-ntwrk/compact-runtime` | `Buffer.from(s, 'hex')`, `Buffer.from(s).toString('hex')` | Hex encoding/decoding in `utils.js` |
| `@midnight-ntwrk/compact-runtime` | `Buffer.from(Bytes32Descriptor...)` | Coin commitment creation in `zswap.js` |
| `@midnight-ntwrk/wallet-sdk-address-format` | `Buffer.from(address, 'hex')` | Bech32m address encoding |
| `@midnight-ntwrk/wallet-sdk-hd` | `Buffer.from(secretKey)` | HD wallet key derivation |

### Code Examples from SDK

```javascript
// @midnight-ntwrk/compact-runtime/dist/utils.js
export const fromHex = (s) => Buffer.from(s, 'hex');
export const toHex = (s) => Buffer.from(s).toString('hex');

// @midnight-ntwrk/compact-runtime/dist/zswap.js
circuitContext.currentQueryContext = circuitContext.currentQueryContext
  .insertCommitment(
    Buffer.from(Bytes32Descriptor.fromValue(createCoinCommitment(coinInfo, recipient).value))
      .toString('hex'),
    circuitContext.currentZswapLocalState.currentIndex
  );
```

### Typical DApp Code Using Buffer

```typescript
// Example: Converting between hex strings and byte arrays
new Uint8Array(Buffer.from(addresses.shieldedCoinPublicKey, 'hex'))
Buffer.from(userPublicKey).toString('hex')
```

---

## 3. Process Polyfill

### Why It's Needed

Many npm packages check `process.env.NODE_ENV` for development/production mode detection. Some libraries also check `process.browser` to detect browser environments.

### Implementation

```typescript
// src/globals.ts
if (typeof globalThis.process === 'undefined') {
  globalThis.process = {
    env: {
      NODE_ENV: import.meta.env.MODE || 'production',
    },
    version: '',
    cwd: () => '/',
  };
}

// For environments that expect process.browser
if (typeof process !== 'undefined' && !process.browser) {
  process.browser = true;
}
```

---

## 4. WebAssembly (WASM) Modules

### WASM Files in the SDK

| Package | WASM File | Size | Purpose |
|---------|-----------|------|---------|
| `@midnight-ntwrk/ledger-v6` | `midnight_ledger_wasm_bg.wasm` | **8.8 MB** | Ledger operations, ZK verification |
| `@midnight-ntwrk/onchain-runtime-v1` | `midnight_onchain_runtime_wasm_bg.wasm` | **1.2 MB** | On-chain runtime execution |

**Total WASM size: ~10 MB**

### What the WASM Modules Provide

The `ledger-v6` WASM module exports critical cryptographic functions:

```typescript
// Core Token Operations
nativeToken()
feeToken()
shieldedToken()
unshieldedToken()

// Coin Operations
createCoinInfo(type_, value)
createShieldedCoinInfo(type_, value)
coinNullifier(coin_info, coin_secret_key)
coinCommitment(coin, coin_public_key)

// Address Operations
addressFromKey(key)
encodeContractAddress(addr)
decodeContractAddress(addr)
encodeUserAddress(addr)
decodeUserAddress(addr)

// Proof Operations
createProvingTransactionPayload(tx, proving_data)
createProvingPayload(serialized_preimage, overwrite_binding_input, key_material)
createCheckPayload(serialized_preimage, ir)
parseCheckResult(result)

// Utility
sampleDustSecretKey()
sampleCoinPublicKey()
partitionTranscripts(calls, params)
```

### How WASM is Loaded

```javascript
// midnight_ledger_wasm.js (browser entry point)
import * as wasm from "./midnight_ledger_wasm_bg.wasm";
export * from "./midnight_ledger_wasm_bg.js";
import { __wbg_set_wasm } from "./midnight_ledger_wasm_bg.js";
__wbg_set_wasm(wasm);
wasm.__wbindgen_start();
```

### Browser vs Node.js Entry Points

Both WASM packages provide conditional exports:

```json
{
  "exports": {
    "browser": "./midnight_ledger_wasm.js",
    "node": "./midnight_ledger_wasm_fs.js"
  }
}
```

The `_fs.js` variant uses Node.js filesystem APIs to load the WASM, while the browser variant uses the ESM import.

### Why `vite-plugin-wasm` is Required

Vite needs special handling for `.wasm` imports:
1. The WASM file must be served with correct MIME type
2. The instantiation must be async
3. The module must be properly bundled

### Why `vite-plugin-top-level-await` is Required

WASM initialization is asynchronous. The SDK uses top-level await to ensure WASM is ready before exporting:

```javascript
// The WASM module uses this pattern internally
const wasm = await WebAssembly.instantiate(...);
export const someFunction = wasm.exports.someFunction;
```

Without this plugin, the code would fail because browsers don't natively support top-level await in all bundling scenarios.

---

## 5. CommonJS Compatibility

### Module Format Analysis

| Category | Count | Examples |
|----------|-------|----------|
| **Pure ESM** (`"type": "module"`) | 17 | `compact-runtime`, `ledger-v6`, `wallet-sdk-*` |
| **Pure CommonJS** (`.cjs` only) | 1 | `zswap` |
| **Dual Published** (`.cjs` + `.mjs`) | 9 | `midnight-js-*` packages |

### Pure CommonJS Package

```json
// @midnight-ntwrk/zswap/package.json
{
  "name": "@midnight-ntwrk/zswap",
  "main": "./zswap.cjs",
  "exports": {
    ".": {
      "import": "./zswap.cjs",
      "require": "./zswap.cjs"
    }
  }
}
```

### Dual-Published Packages

```json
// @midnight-ntwrk/midnight-js-contracts/package.json
{
  "main": "dist/index.cjs",
  "module": "dist/index.mjs",
  "exports": {
    ".": {
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs"
    }
  }
}
```

### Why `viteCommonjs` is Required

The `@midnight-ntwrk/zswap` package is CommonJS-only. Vite is ESM-native and needs the CommonJS plugin to:
1. Transform `require()` calls to `import`
2. Transform `module.exports` to `export`
3. Handle mixed ESM/CJS dependencies

**Note:** If `@midnight-ntwrk/zswap` ships as ESM in the future, this plugin could be removed.

---

## 6. Additional Configuration

### Global Definitions

```typescript
// vite.config.ts
define: {
  'process.env.NODE_ENV': JSON.stringify(mode),
  'process.env': {},
  global: 'globalThis',
},
```

### Dependency Optimization

```typescript
optimizeDeps: {
  esbuildOptions: {
    define: {
      global: 'globalThis',
    },
  },
  exclude: [
    '@midnight-ntwrk/onchain-runtime',  // Excluded due to WASM
  ],
},
```

### Build Options

```typescript
build: {
  commonjsOptions: {
    transformMixedEsModules: true,  // Handle mixed ESM/CJS
  },
},
```

---

## 7. Package Dependencies Summary

### Core Midnight SDK Packages

| Package | Version | Module Format | Uses Buffer | Uses WASM |
|---------|---------|---------------|-------------|-----------|
| `@midnight-ntwrk/compact-runtime` | 0.11.0-rc.1 | ESM | Yes | No |
| `@midnight-ntwrk/dapp-connector-api` | 4.0.0-beta.2 | ESM | No | No |
| `@midnight-ntwrk/ledger-v6` | 6.1.0-alpha.6 | ESM | Yes | **Yes (8.8MB)** |
| `@midnight-ntwrk/midnight-js-contracts` | 3.0.0-alpha.11 | Dual | No | No |
| `@midnight-ntwrk/midnight-js-fetch-zk-config-provider` | 3.0.0-alpha.11 | Dual | No | No |
| `@midnight-ntwrk/midnight-js-http-client-proof-provider` | 3.0.0-alpha.11 | Dual | No | No |
| `@midnight-ntwrk/midnight-js-indexer-public-data-provider` | 3.0.0-alpha.11 | Dual | No | No |
| `@midnight-ntwrk/midnight-js-network-id` | 3.0.0-alpha.11 | Dual | No | No |
| `@midnight-ntwrk/midnight-js-types` | 3.0.0-alpha.11 | Dual | No | No |
| `@midnight-ntwrk/zswap` | 4.0.0 | **CJS** | Yes | No |

### Transitive Dependencies

| Package | Module Format | Uses Buffer | Uses WASM |
|---------|---------------|-------------|-----------|
| `@midnight-ntwrk/onchain-runtime-v1` | ESM | No | **Yes (1.2MB)** |
| `@midnight-ntwrk/wallet-sdk-address-format` | ESM | Yes | No |
| `@midnight-ntwrk/wallet-sdk-hd` | ESM | Yes | No |

---

## 8. Performance Considerations

### Bundle Size Impact

| Component | Size | Impact |
|-----------|------|--------|
| WASM modules | ~10 MB | Large initial download, but cached |
| Buffer polyfill | ~50 KB | Minimal |
| Process polyfill | ~5 KB | Negligible |

### Loading Strategy

1. **WASM modules are loaded asynchronously** - doesn't block initial render
2. **Prover keys are fetched on-demand** - only when proof generation is needed
3. **Code splitting** - Vite automatically splits the bundle

---

## 9. Troubleshooting Common Issues

### "Buffer is not defined"
Ensure `globals.ts` is imported at the app entry point before any SDK imports.

### "Cannot use import statement outside a module"
The CommonJS plugin may not be transforming a dependency. Add it to `optimizeDeps.include`.

### WASM loading fails
Check that `vite-plugin-wasm` is configured and the WASM files are accessible.

### "Top-level await is not available"
Ensure `vite-plugin-top-level-await` is in the plugins array.

---

## 10. Deep Dive: Why WASM is Required

### The Fundamental Problem

The Midnight blockchain implements a sophisticated zero-knowledge proof system that requires:

1. **Elliptic curve cryptography** on non-standard curves (BLS12-381, embedded curves)
2. **Zero-knowledge proof verification** involving millions of field operations
3. **Deterministic execution** across different platforms (browser, Node.js, native)
4. **Constant-time operations** to prevent timing attacks

**JavaScript cannot provide these guarantees.**

### Why Not Pure JavaScript?

| Limitation | Impact |
|------------|--------|
| No native 256-bit integers | BigInt is 10-100x slower than native for crypto |
| No constant-time operations | Vulnerable to timing attacks |
| No SIMD instructions | Cannot parallelize field arithmetic |
| Non-deterministic floating point | Results may vary across platforms |
| Memory safety | No guarantees against buffer overflows |

### Architectural Decision from Midnight

From Midnight Architecture decissions:

> "Ledger Libs are being implemented in Rust language, but expose a TypeScript interface that can be used in Node.js and browsers through targeting WASM."

This decision ensures:
- **Single source of truth** for ledger logic
- **Eliminates duplication** between client-side and server-side implementations
- **Deterministic execution** across all platforms

### What WASM Contains

The WASM modules are compiled from Rust and contain:

| Component | Purpose | Why Rust/WASM |
|-----------|---------|---------------|
| Field arithmetic | 256-bit modular arithmetic | Performance, constant-time |
| Curve operations | Point addition, scalar multiplication | Performance |
| Hash functions | SHA-256, Poseidon | Determinism |
| Merkle trees | Membership proofs | Performance |
| Proof verification | ZK-SNARK verification | Millions of operations |
| Impact VM | Smart contract execution | Determinism |

---

## 11. Midnight Ledger: Rust Implementation Analysis

### Codebase Overview

The `midnight-ledger` repository is a **Rust workspace** containing 27+ interconnected crates:

| Metric | Value |
|--------|-------|
| **Language** | Rust (Edition 2024) |
| **Codebase Size** | 165 Rust source files |
| **Core Ledger Module** | ~13,735 lines of Rust |
| **License** | Apache 2.0 |
| **Version** | 7.0.0-alpha.1 |

### WASM Build Process

```
Rust Crates → cargo (wasm32 target) → WASM binary
  → wasm-bindgen → JavaScript glue code
  → wasm-opt → optimized WASM
  → TypeScript declarations
  → ESM module + Node.js wrapper
```

**Build Toolchain:**
- **Target:** `wasm32-unknown-unknown`
- **Binding Layer:** wasm-bindgen 0.2.104
- **Optimization:** Binaryen's `wasm-opt` with `-Os` flag

**WASM Compilation Profile:**
```toml
[profile.wasm]
opt-level = "z"      # Optimize for size
lto = true           # Link-time optimization
codegen-units = 1    # Single codegen unit for better optimization
panic = "abort"      # No unwinding in WASM
```

### WASM Packages Produced

| Package | npm Name | Purpose |
|---------|----------|---------|
| ledger-wasm | `@midnight-ntwrk/ledger` | Transaction assembly, verification |
| onchain-runtime-wasm | `@midnight-ntwrk/onchain-runtime` | Contract state management |
| zkir-wasm | `@midnight-ntwrk/zkir-v2` | ZK IR proving/verification |

### Crate Dependency Graph

```
ledger-wasm ──┬──→ ledger ──────┬──→ zswap ──→ onchain-runtime ──→ onchain-vm
              │                 ├──→ zkir   ─┐
              ├──→ onchain-runtime-wasm       │
              └──→ transient-crypto ←────────┘
                       ↑
                    (ZK core)
                       ↓
            base-crypto ← serialize ← storage
```

### Key Rust Crates

| Crate | Purpose |
|-------|---------|
| `ledger` | Transaction construction, dust computation, intent verification |
| `zswap` | Shielded token system, nullifiers, commitments |
| `onchain-vm` | Impact VM (smart contract execution engine) |
| `onchain-runtime` | Contract state transitions |
| `transient-crypto` | ZK-specific cryptography |
| `base-crypto` | SHA-256, Schnorr signatures, serialization |
| `zkir` | Zero-Knowledge IR compilation and execution |

---

## 12. Cryptographic Primitives

### External Cryptographic Libraries

| Library | Version | Purpose |
|---------|---------|---------|
| `midnight-curves` | ^0.2.0 | Elliptic curve operations & field arithmetic |
| `midnight-proofs` | ^0.7.0 | Zero-knowledge proof system |
| `midnight-circuits` | ^6.0.0 | Circuit representation and compilation |
| `midnight-zk-stdlib` | ^1.0.0 | Zero-knowledge standard library |
| `k256` | ^0.13.4 | secp256k1 implementation (Schnorr signatures) |
| `sha2` | ^0.10.9 | SHA-256 hashing |
| `ff` | ^0.13.1 | Finite field arithmetic |
| `group` | ^0.13.0 | Elliptic curve group operations |

### Cryptographic Modules

#### base-crypto (midnight-base-crypto)

| Component | Implementation |
|-----------|----------------|
| **Hash** | SHA-256 based persistent hashing, 32-byte outputs |
| **Signatures** | BIP340 Schnorr over secp256k1 via k256 |
| **RNG** | Cryptographically secure random number generation |
| **FAB** | Fixed Alignment Blocks for on-chain data |

#### transient-crypto (ZK-specific)

| Component | Implementation |
|-----------|----------------|
| **Primary Field** | `Fr` (scalar field of BLS12-381) |
| **Embedded Curve** | `EmbeddedGroupAffine` points |
| **Commitments** | Pedersen-style commits with randomness |
| **Merkle Trees** | Fixed-height (32) for accumulator functionality |
| **Encryption** | Public-key encryption for shielded transactions |

### Why These Curves Cannot Be Replaced

| Curve | Used For | WebCrypto Support |
|-------|----------|-------------------|
| BLS12-381 | ZK proofs, pairing operations | **No** |
| Embedded curve (Jubjub-like) | In-circuit operations | **No** |
| secp256k1 | Schnorr signatures | **No** (only ECDSA) |

WebCrypto only supports P-256, P-384, P-521, and Ed25519. The curves required by Midnight's ZK system are **not available** in any browser API.

---

## 13. Alternatives to WASM: Analysis

### Alternatives Evaluated 

#### 1. FFI (Foreign Function Interface)

| Aspect | Details |
|--------|---------|
| **Status** | Used for Node.js only |
| **Limitation** | Browsers have no FFI capability |
| **Decision** | Insufficient for browser support |

#### 2. Pure JavaScript Implementation

| Aspect | Details |
|--------|---------|
| **Status** | Not viable |
| **Performance** | 10-100x slower than WASM for crypto operations |
| **Security** | No constant-time guarantees |
| **Decision** | Rejected |

#### 3. WebCrypto API

| Aspect | Details |
|--------|---------|
| **Status** | Partially viable |
| **Limitation** | Only supports standard curves (P-256, not BLS12-381) |
| **Missing** | Pairing operations, field arithmetic |
| **Decision** | Cannot replace WASM |

#### 4. Native Browser Extensions

| Aspect | Details |
|--------|---------|
| **Status** | Safari only |
| **Limitation** | Not cross-platform |
| **Decision** | Not viable for wide browser support |

#### 5. HTTP-based Proof Server

| Aspect | Details |
|--------|---------|
| **Status** | Current interim solution for proving |
| **Limitation** | WebKit/Safari restrict localhost HTTP from HTTPS |
| **Privacy** | Requires external process, complicates "data stays on device" |
| **Decision** | Workaround, not replacement |

#### 6. Native Messaging API

| Aspect | Details |
|--------|---------|
| **Status** | Proposed for wallet extensions |
| **Limitation** | Only available to browser extensions |
| **Decision** | Good for wallets, not for embedded dApp functionality |

#### 7. WebGPU

| Aspect | Details |
|--------|---------|
| **Status** | Future possibility |
| **Limitation** | Not yet mature for ZK proofs |
| **Decision** | Monitor for future |

### Summary: Why Alternatives Fail

```
┌─────────────────────────────────────────────────────────────────┐
│                    WASM is Required Because:                    │
├─────────────────────────────────────────────────────────────────┤
│ 1. Curves used (BLS12-381, embedded) not in WebCrypto          │
│ 2. ZK proof verification needs millions of field operations    │
│ 3. JavaScript BigInt is 10-100x slower than native             │
│ 4. Determinism required across browser, Node.js, native        │
│ 5. Constant-time operations prevent timing attacks             │
│ 6. No other cross-browser solution for native code execution   │
└─────────────────────────────────────────────────────────────────┘
```

### WASM Performance Characteristics

> "Generating proofs in a default, single-threaded setup is unacceptably slow - although successful, it took minutes to generate a Zswap spend proof. On the other hand - enabling WASM to run in a multithreaded setup (by leveraging Web Workers, shared array buffers and `wasm-bindgen-rayon` crate) brings performance to noticeably slower than native, but somewhat acceptable levels (usually 20-30s for a Zswap spend proof on a M1 Max/10 core machine)."

**WASM Multi-threading Requirements:**
- Unstable Rust compiler feature `target-features=+atomics`
- Shared array buffers (require COOP/COEP headers)
- Web Workers (WASM blocks its running thread)

---

## 14. Future Considerations

### Short-Term Improvements

1. **Smaller WASM Bundles**
   - Tree-shaking unused functions
   - Separate proving and verification modules

2. **Streaming Compilation**
   - Load WASM progressively while showing UI
   - Use `WebAssembly.compileStreaming()`

3. **Service Worker Caching**
   - Cache 10MB WASM for offline capability
   - Faster subsequent loads

### Medium-Term (SDK Evolution)

1. **Dedicated Browser Builds**
   - Pre-bundled polyfills
   - Optimized WASM for browser-only features

2. **Native Messaging for Wallets**
   - Offload proving to native companion app
   - Better performance for complex proofs

3. **`@midnight-ntwrk/zswap` as ESM**
   - Would eliminate need for `viteCommonjs` plugin

### Long-Term (Ecosystem Changes)

1. **WebCrypto Expansion**
   - If browsers add BLS12-381 support (unlikely soon)
   - Would reduce WASM size significantly

2. **WASM GC (Garbage Collection)**
   - Reduce memory overhead
   - Better integration with JavaScript

3. **WebGPU for Proving**
   - GPU-accelerated proof generation
   - Could dramatically improve proving performance

---

## 15. Mobile Platform Incompatibility and Potential Fixes

### The Core Problem: WASM Restrictions by Platform and Framework

The Midnight SDK's reliance on WebAssembly creates compatibility challenges for mobile development. It's important to distinguish between **platform-level restrictions** and **framework-specific limitations**.

---

### Platform-Level Restrictions

#### iOS: Hard Platform Restriction

Apple prohibits Just-In-Time (JIT) compilation for all apps outside of Safari/WKWebView for security reasons. This is an **OS-level policy** that affects all native iOS development regardless of framework choice.

| iOS Environment | JIT Allowed | WASM Support | Notes |
|-----------------|-------------|--------------|-------|
| Safari / WKWebView | ✅ Yes | ✅ Full | Browser context has JIT privileges |
| Native Apps (any framework) | ❌ No | ❌ Broken | Apple security policy since iOS 14 |
| iOS Simulator | ✅ Yes | ✅ Works | **Misleading** - does not reflect device behavior |

**Critical Finding:** WebAssembly works in the iOS Simulator but **fails on actual devices**. This creates a dangerous false positive during development.

**Performance Impact:** When JIT is disabled, JavaScript performance degrades by approximately **7.5x**. For ZK proof operations that already take 20-30 seconds in optimized multi-threaded WASM, this makes the SDK completely unusable.

#### Android: No Platform Restrictions

Android has **no platform-level restrictions** on JIT compilation or WebAssembly execution. Native Android development can:

- Embed V8 or any JavaScript engine with full WASM/JIT support
- Use any WASM runtime (Wasmer, Wasmtime, wasm3, etc.)
- Compile Rust directly to native `.so` libraries via JNI/NDK
- Execute JIT-compiled code without restriction

| Android Environment | JIT Allowed | WASM Support | Notes |
|---------------------|-------------|--------------|-------|
| Native Apps (Kotlin/Java + NDK) | ✅ Yes | ✅ Full | No platform restrictions |
| Chrome / WebView | ✅ Yes | ✅ Full | V8 engine with full JIT |
| Embedded V8 | ✅ Yes | ✅ Full | Can embed directly in native apps |

---

### React Native-Specific Limitations

React Native introduces its own limitations on **both platforms** due to its choice of JavaScript engines. These are framework limitations, not platform restrictions.

#### React Native on iOS

React Native uses JavaScriptCore (JSC), which is subject to Apple's JIT prohibition:

| Configuration | WASM Support | Reason |
|---------------|--------------|--------|
| JSC (default) | ❌ Broken | Apple JIT prohibition applies |
| Hermes | ❌ No support | Hermes doesn't implement WASM |

#### React Native on Android

React Native's JavaScript engines do not support WASM, but this is an **engine choice**, not a platform limitation:

| JS Engine | WASM Support | Details |
|-----------|--------------|---------|
| **Hermes** (RN default) | ❌ No | WASM not implemented; Hermes team states "WASM doesn't make sense" for RN |
| **JSC-Android** | ❌ Disabled | Explicitly disabled via `--no-webassembly` build flag |

**Key Point:** Unlike iOS, these Android limitations could be overcome by:
- Using a custom V8 integration instead of Hermes
- Building JSC-Android with WASM enabled
- Bypassing React Native's JS engine entirely with native modules

---

### Summary: Platform vs Framework Restrictions

| Platform | Native Development | React Native | Root Cause |
|----------|-------------------|--------------|------------|
| **iOS** | ❌ WASM broken | ❌ WASM broken | Apple JIT prohibition (platform policy) |
| **Android** | ✅ No restrictions | ❌ Engine limitations | RN engine choices (can be worked around) |

---

### SharedArrayBuffer Limitations

Multi-threaded WASM (required for acceptable proof generation times) depends on `SharedArrayBuffer`, which has additional restrictions:

- Requires COOP/COEP headers for cross-origin isolation
- Historically disabled on mobile Chrome (now re-enabled with headers)
- Inconsistent support across mobile browsers and WebViews

### Why Current Workarounds Fail (iOS & React Native)

The following workarounds attempt to run WASM in restricted environments but fall short for ZK proof generation:

| Workaround | Limitation |
|------------|------------|
| WebView-based WASM | Still subject to memory limits; poor UX |
| wasm3 interpreter | 10-100x slower than JIT-compiled WASM |
| Polyfills (react-native-wasm) | Only provides WebAssembly API shim, doesn't solve JIT issue |
| Browser-in-app | Memory constraints, battery drain, not truly native |

**Note:** For Android native development, these workarounds are unnecessary. Developers can embed V8 or use native WASM runtimes with full JIT support.

### Potential Fix #1: Native FFI Compilation

The most portable solution is to **bypass WASM entirely** by compiling the Midnight Rust crates directly to native iOS/Android libraries. This approach:

- **iOS:** Required due to Apple's JIT prohibition
- **Android:** Optional but recommended for consistency and maximum performance

#### How It Works

```
Rust Crates → cargo (iOS/Android targets) → Native Libraries
  → UniFFI/swift-bridge → Swift/Kotlin bindings
  → Mobile SDK wrapper → React Native/Flutter integration
```

#### Target Platforms

| Platform | Rust Target | Output | Integration |
|----------|-------------|--------|-------------|
| iOS Device | `aarch64-apple-ios` | `.a` static library | Xcode + FFI |
| iOS Simulator | `aarch64-apple-ios-sim` | `.a` static library | Xcode + FFI |
| Android ARM64 | `aarch64-linux-android` | `.so` shared library | JNI |
| Android ARM32 | `armv7-linux-androideabi` | `.so` shared library | JNI |
| Android x86_64 | `x86_64-linux-android` | `.so` shared library | JNI |

#### Proof This Approach Works: MoPro Framework

The **MoPro (Mobile Prover)** project has successfully implemented this pattern for other ZK systems:

- Supports Circom, Halo2, and Noir proving systems
- **Supports BLS12-381 curve** (same as Midnight)
- Generates bindings for Swift (iOS), Kotlin (Android), React Native, and Flutter
- Achieves **10x performance improvement** over browser-based WASM
- Reduces integration time from "15 days to hours"

#### Implementation Requirements for Midnight

| Step | Description | Effort |
|------|-------------|--------|
| 1. C FFI Interface | Add `#[no_mangle] pub extern "C"` exports to core Rust crates | Medium |
| 2. Cross-compilation | Add iOS/Android targets to build pipeline | Low |
| 3. Binding Generation | Use UniFFI to auto-generate Swift/Kotlin wrappers | Medium |
| 4. Mobile SDK Layer | Create TypeScript bridge for React Native/Flutter | High |
| 5. Mobile Optimization | Reduce memory footprint, add GPU acceleration | High |

#### Example FFI Interface

```rust
// midnight-ledger/src/mobile_ffi.rs
use std::os::raw::c_char;
use std::ffi::{CString, CStr};

#[no_mangle]
pub extern "C" fn midnight_verify_proof(
    proof_data: *const c_char,
    public_inputs: *const c_char,
) -> bool {
    let proof = unsafe { CStr::from_ptr(proof_data) };
    let inputs = unsafe { CStr::from_ptr(public_inputs) };
    // Native verification - no WASM overhead, full device performance
    internal_verify(proof, inputs)
}

#[no_mangle]
pub extern "C" fn midnight_create_transaction(
    intent_data: *const c_char,
) -> *mut c_char {
    // Native transaction creation
    let result = internal_create_tx(...);
    CString::new(result).unwrap().into_raw()
}
```

#### Swift Integration Example

```swift
import MidnightCore

// Direct native calls - no JavaScript bridge
let isValid = midnight_verify_proof(proofData, publicInputs)
let transaction = midnight_create_transaction(intentData)
```

### Potential Fix #2: Proof Server Delegation

For scenarios where on-device proving is not critical, delegate proof generation to a remote server.

#### Architecture

```
Mobile App (UI + Transaction Building)
    ↓ (witness data)
Proof Server (Native Rust)
    ↓ (completed proof)
Mobile App (Submit to Network)
```

#### Trade-offs

| Advantage | Disadvantage |
|-----------|--------------|
| No mobile SDK changes needed | Requires trusted server infrastructure |
| Works on all platforms | Latency for proof generation |
| Centralized optimization | Privacy implications (witness data leaves device) |
| Easier to update proving logic | Single point of failure |

#### When to Use

- Enterprise applications with controlled infrastructure
- Applications where proving data is not privacy-sensitive
- As an interim solution while native SDK is developed

### Potential Fix #3: Hybrid Approach

Combine native FFI for critical operations with proof server delegation for heavy computation.

#### Split by Operation Type

| Operation | Execution Location | Rationale |
|-----------|-------------------|-----------|
| Address generation | Native (on-device) | Privacy-critical |
| Transaction building | Native (on-device) | Fast, no privacy concerns |
| Signature creation | Native (on-device) | Security-critical |
| Proof verification | Native (on-device) | Fast enough on mobile |
| **Proof generation** | **Server-side** | Too slow on mobile |

#### Implementation

```
┌─────────────────────────────────────────────────────────────┐
│                     Mobile Application                       │
├─────────────────────────────────────────────────────────────┤
│  Native FFI Layer (Rust → Swift/Kotlin)                     │
│  ├── Address operations      ✓ On-device                    │
│  ├── Transaction building    ✓ On-device                    │
│  ├── Signature creation      ✓ On-device                    │
│  └── Proof verification      ✓ On-device                    │
├─────────────────────────────────────────────────────────────┤
│  Network Layer                                               │
│  └── Proof generation        → Proof Server (optional)      │
└─────────────────────────────────────────────────────────────┘
```

### Potential Fix #4: GPU-Accelerated Mobile Proving

For future consideration: leverage mobile GPU hardware for proof generation.

#### Technologies

| Platform | GPU API | Status |
|----------|---------|--------|
| iOS | Metal | Active research (MoPro) |
| Android | Vulkan | Experimental |
| Cross-platform | WebGPU | Not ready for mobile |

#### Performance Potential

GPU acceleration for Multi-Scalar Multiplication (MSM)—a key ZK proving operation—shows promising results on Apple Silicon. The MoPro project is actively researching Metal-based acceleration for mobile devices.

#### Challenges

- Significant engineering investment
- Battery consumption concerns
- Heat management on mobile devices
- Not all operations are GPU-parallelizable

### Comparison of Mobile Solutions

| Solution | Performance | Privacy | Engineering Complexity |
|----------|-------------|---------|------------------------|
| Native FFI (Full) | Excellent | Full on-device | High |
| Proof Server | Good | Partial (witness leaves device) | Low |
| Hybrid Approach | Very Good | Good (only proofs delegated) | Medium |
| GPU Acceleration | Excellent | Full on-device | Very High |

### Industry Precedent

Several ZK projects have successfully implemented native mobile SDKs, demonstrating technical feasibility:

| Project | Technical Approach | Performance Result |
|---------|-------------------|-------------------|
| **Reclaim Protocol** | Migrated from snarkjs (WASM) to gnark via native FFI | 40s → 4-5s proving time |
| **EZKL** | Native iOS package via Swift Package Manager using Halo2 | Full on-device ZK inference |
| **MoPro** | UniFFI-based Rust→Swift/Kotlin bindings with BLS12-381 support | 10x faster than browser WASM |
| **zkPassport** | Native Noir integration via Barretenberg FFI | Production mobile deployment |

### Technical Constraints Summary

The current WASM-based SDK architecture is fundamentally incompatible with iOS native app development due to Apple's JIT restrictions. The technical options are:

1. **Native FFI compilation** bypasses WASM entirely and provides full device performance
2. **Proof server delegation** avoids client-side proving but introduces privacy trade-offs
3. **Hybrid approaches** balance privacy and performance by splitting operations
4. **GPU acceleration** requires significant R&D but offers the best long-term performance potential

---

## Appendix A: Architecture Decision Records

### ADR 0004: Workshops without Browser Support

> "It allows components like ledger or zero knowledge system (which are implemented in Rust) connect through an FFI interface to node.js (which is simpler than WebAssembly) - this is essential for claims that private data stays on the device"

**Decision:** FFI for Node.js workshops, WASM for browser support.

### ADR 0016: Forks and Change Management

> "To separate Midnight Ledger from runtime through native runtime interface for the time being"

**Decision:** Ledger logic isolated in WASM, upgradeable independently.

### Proposal 0020: Local Proving Modalities

**Evaluated approaches:**
1. HTTP-based proof server (current)
2. Native Messaging API (for extensions)
3. WASM-based proving (promising but not production-ready)

**Conclusion:** WASM proving is the future, but requires toolchain maturation.

---

## Appendix B: Quick Reference

### Required Vite Plugins

| Plugin | Purpose | Removable If... |
|--------|---------|-----------------|
| `nodePolyfills` | Buffer, process | SDK provides browser builds |
| `wasm` | WASM loading | Never (fundamental) |
| `topLevelAwait` | Async WASM init | Never (fundamental) |
| `viteCommonjs` | CJS compatibility | `zswap` ships as ESM |

### WASM Module Sizes

| Module | Size | Contains |
|--------|------|----------|
| ledger-v6 | 8.8 MB | Crypto, proofs, addresses |
| onchain-runtime-v1 | 1.2 MB | Impact VM, state |
| **Total** | **10 MB** | |

### Cryptographic Curves

| Curve | Purpose | In WebCrypto |
|-------|---------|--------------|
| BLS12-381 | ZK proofs | No |
| Embedded (Jubjub-like) | In-circuit ops | No |
| secp256k1 | Signatures | No |

### Mobile Platform Support

| Platform | Native Development | React Native | Root Cause |
|----------|-------------------|--------------|------------|
| iOS Native | ❌ WASM broken | ❌ WASM broken | Apple JIT prohibition (platform policy) |
| iOS Safari/WKWebView | ✅ Works | N/A | JIT allowed in browser context |
| Android Native | ✅ No restrictions | ❌ Engine limitations | RN uses Hermes/JSC (workaround possible) |
| Android Chrome/WebView | ✅ Works | N/A | V8 engine with full JIT |

### Mobile Solution Comparison

| Approach | Privacy | Performance | Complexity |
|----------|---------|-------------|------------|
| Proof Server Delegation | Partial | Good | Low |
| Hybrid (FFI + Server) | Good | Very Good | Medium |
| Full Native FFI | Full | Excellent | High |
| GPU Acceleration | Full | Excellent | Very High |

### Active Implementation: ledger-uniffi

| Component | Status |
|-----------|--------|
| UniFFI Framework | ✅ UniFFI 0.29 integrated |
| Android Build (ARM64) | ✅ Working |
| iOS Build (Simulator) | ✅ Working |
| React Native Module | ✅ Scaffolding complete |
| Demo App | ✅ End-to-end working |
| API Implementation | ⚠️ Partial |
| Test Suite | ❌ Pending |
| Documentation | ❌ Pending |

### Alternative: Polygen (WASM AOT)

| Aspect | Status |
|--------|--------|
| Approach | WASM → C → Native (AOT) |
| iOS JIT bypass | ✅ Solved |
| Thread support | ❌ Not available |
| Best for | Non-proving operations |
| Limitation | Proof generation would be slow (minutes) |

### Alternative: Manual Rust FFI

| Aspect | Status |
|--------|--------|
| Approach | Rust → JNI/Swift → Native |
| Binding generation | Manual (JNI + Swift code) |
| Control | Maximum flexibility |
| Boilerplate | Significant |
| Best for | Few functions, custom memory management |
| Trade-off | More effort, fewer dependencies |

---

---

## 16. Active Implementation: ledger-uniffi (Native FFI for Mobile)

https://github.com/midnightntwrk/midnight-ledger/pull/42

This section documents active work to implement the **Native FFI Compilation** solution described in Section 15 ("Potential Fix #1"). The `ledger-uniffi` project provides UniFFI bindings for the Midnight Ledger crate, enabling native mobile integration without WASM dependencies.

### Project Overview

| Attribute | Details |
|-----------|---------|
| **Goal** | Provide a mobile-friendly FFI surface for the Rust ledger library |
| **Approach** | Mozilla UniFFI for type-safe bindings to iOS (Swift) and Android (Kotlin) |
| **Analogy** | Conceptually similar to `ledger-wasm` (which targets Web via WASM), but targeting mobile (React Native) instead |
| **Status** | Experimental and work-in-progress |

### Project Structure

```
ledger-uniffi/
├── src/                          # Rust UniFFI wrapper code
│   ├── lib.rs                    # Main exports and UniFFI scaffolding
│   ├── types.rs                  # Type definitions
│   ├── crypto.rs                 # Cryptographic operations
│   ├── tx.rs                     # Transaction handling
│   ├── intent.rs                 # Intent management
│   ├── zswap_*.rs                # ZSwap-related functionality
│   ├── conversions.rs            # Type conversions between FFI and internal types
│   ├── errors.rs                 # Error handling
│   └── objects/                  # Object wrappers
│       ├── transaction.rs
│       ├── proof.rs
│       ├── token_types.rs
│       ├── parameters.rs
│       ├── cost_model.rs
│       └── dust.rs
├── react-native-ledger-ffi/      # React Native bridge module
│   ├── android/                  # Android native module (Kotlin)
│   └── ios/                      # iOS native module (Swift)
├── rn-demo-app/                  # Sample React Native demonstration app
├── run.sh                        # Automated build script for both platforms
└── Cargo.toml                    # Rust dependencies
```

### Exposed API Functions

The UniFFI bindings expose the following functions to mobile platforms:

#### Token Operations
| Function | Description |
|----------|-------------|
| `native_token()` | Returns the native token type |
| `fee_token()` | Returns the fee (dust) token type |
| `shielded_token()` | Returns the shielded token type |
| `unshielded_token()` | Returns the unshielded token type |

#### Coin Operations
| Function | Description |
|----------|-------------|
| `create_shielded_coin_info(token_type, value)` | Creates shielded coin info with specified type and value |
| `coin_nullifier(coin_info, coin_secret_key)` | Computes coin nullifier for spending |
| `coin_commitment(coin_info, coin_public_key)` | Computes coin commitment |
| `sample_coin_public_key()` | Generates a random coin public key |
| `sample_encryption_public_key()` | Generates a random encryption public key |

#### Address Operations
| Function | Description |
|----------|-------------|
| `address_from_key(key)` | Derives user address from verifying key |

#### Proof Operations
| Function | Description |
|----------|-------------|
| `create_proving_payload(preimage, overwrite_binding_input, key_material)` | Creates payload for proof generation |
| `create_check_payload(preimage, ir)` | Creates payload for proof checking |
| `parse_check_result(result)` | Parses proof check results |

### Build Pipeline

The `run.sh` script automates the complete build process:

```
Rust Source Code
    ↓
┌───────────────────────────────────────────────────────────────┐
│ Android Build                                                  │
│   Target: aarch64-linux-android                                │
│   Output: libledger_uniffi.so                                  │
│   Bindings: Kotlin (UniFFI generated)                          │
└───────────────────────────────────────────────────────────────┘
    ↓
┌───────────────────────────────────────────────────────────────┐
│ iOS Build                                                      │
│   Target: aarch64-apple-ios-sim                                │
│   Output: libledger_uniffi.a                                   │
│   Bindings: Swift (UniFFI generated)                           │
└───────────────────────────────────────────────────────────────┘
    ↓
React Native Module (react-native-ledger-ffi)
    ↓
Demo App (rn-demo-app)
```

#### Quick Start Commands

```bash
# Build for both platforms
./run.sh

# Build only for Android
./run.sh android

# Build only for iOS
./run.sh ios
```

### Current Implementation State

#### What Works
- End-to-end data flow from Rust library → UniFFI bindings → React Native module → mobile UI
- Basic function calls (e.g., `hello()` greeting string) surfaced and displayed in demo app
- Build pipeline for both Android and iOS platforms

#### What's Incomplete
| Component | Status |
|-----------|--------|
| Rust wrapper implementations | Partial - some functions are placeholders |
| Public API design | Not finalized - will change |
| Data types exposed via FFI | Not finalized |
| Test suite | None - unit/integration/proptests pending |
| Transaction parsing | Not yet implemented |
| Error handling | Partial coverage |

### Dependencies

The UniFFI bindings depend on the core Midnight Ledger crates:

```toml
[dependencies]
uniffi = "0.29"
serialize = { path = "../serialize", package = "midnight-serialize" }
ledger = { path = "../ledger", package = "midnight-ledger" }
base-crypto = { path = "../base-crypto", package = "midnight-base-crypto" }
coin-structure = { path = "../coin-structure", package = "midnight-coin-structure" }
transient-crypto = { path = "../transient-crypto", package = "midnight-transient-crypto" }
```

### Planned Work

1. **Finalize Native Module API**
   - Complete function implementations
   - Stabilize data types for FFI boundary
   - Add comprehensive error handling

2. **Testing**
   - Rust-side unit and integration tests
   - React Native module tests (JS/TS)
   - End-to-end paths with demo app

3. **Documentation**
   - API reference documentation
   - Integration guides for iOS and Android

### Relation to Section 15 Solutions

This implementation directly addresses the **Native FFI Compilation** approach (Potential Fix #1):

| Aspect | Planned (Section 15) | Implemented (ledger-uniffi) |
|--------|----------------------|----------------------------|
| FFI Framework | UniFFI/swift-bridge | UniFFI 0.29 |
| iOS Target | `aarch64-apple-ios` | `aarch64-apple-ios-sim` |
| Android Targets | ARM64, ARM32, x86_64 | ARM64 (`aarch64-linux-android`) |
| Binding Languages | Swift, Kotlin | Swift, Kotlin |
| React Native Integration | TypeScript bridge | `react-native-ledger-ffi` module |

### Bypassing WASM Limitations

By compiling Rust directly to native libraries, this approach:

1. **Avoids iOS JIT restrictions** - Native code runs without WASM/JavaScript
2. **Avoids React Native engine limitations** - No dependency on Hermes or JSC WASM support
3. **Enables full performance** - Native execution speed without interpreter overhead
4. **Maintains privacy** - All cryptographic operations stay on-device

---

## 17. Alternative Approach: Polygen (WASM AOT Compilation)

Polygen is a WebAssembly toolkit for React Native developed by Callstack that uses **Ahead-of-Time (AOT) compilation** instead of JIT. This approach offers an alternative path to running WASM modules on mobile platforms.

### How Polygen Works

```
WASM Module (.wasm)
    ↓ wasm2c (WABT toolkit)
C Source Code (.c/.h)
    ↓ Native compiler (Xcode/NDK)
Native Library (.a/.so)
    ↓ JSI bridge
React Native JavaScript
```

Polygen converts WASM modules to C code at **build time** using the `wasm2c` tool from the WebAssembly Binary Toolkit (WABT). The generated C code is then compiled into native iOS/Android libraries and accessed via React Native's JSI (JavaScript Interface).

### Key Benefits

| Benefit | Description |
|---------|-------------|
| **Bypasses iOS JIT prohibition** | WASM is pre-compiled to native C—no runtime JIT needed |
| **Standard WebAssembly API** | Provides a polyfill matching the WebAssembly JavaScript API |
| **Works on both platforms** | Same AOT approach for iOS and Android |
| **Near-native performance** | Compiled C code executes at native speed |

### Feature Support Status

| Feature | Status | Notes |
|---------|--------|-------|
| WebAssembly 2.0 | ✅ Supported | Core specification |
| Exceptions | ❌ Not supported | |
| **Threads** | ❌ Not supported | Critical limitation for proving |
| Garbage Collection | ❌ Not supported | |
| Multiple Memories | 🟡 Partial | |
| Mutable Globals | 🟡 Partial | |

### Thread Limitation Impact

The lack of thread support in Polygen has significant implications for ZK proof generation. From the Midnight architecture documentation:

> "Generating proofs in a default, single-threaded setup is unacceptably slow... it took minutes to generate a Zswap spend proof. On the other hand - enabling WASM to run in a multithreaded setup... brings performance to... usually 20-30s."

Without thread support, proof generation via Polygen would take **minutes rather than seconds**. However, many ledger operations do not require multi-threading and could benefit from this approach.

### Applicability to Midnight Operations

| Operation Type | Polygen Viability | Rationale |
|----------------|-------------------|-----------|
| Proof generation | ⚠️ Limited | Single-threaded execution would be significantly slower |
| Proof verification | ✅ Viable | Verification is less compute-intensive |
| Transaction building | ✅ Viable | Does not require threading |
| Address operations | ✅ Viable | Simple cryptographic operations |
| Coin operations | ✅ Viable | Commitment/nullifier computation |

### Comparison: Polygen vs ledger-uniffi

| Aspect | Polygen | ledger-uniffi |
|--------|---------|---------------|
| **Approach** | WASM → C → Native | Rust → Native (FFI) |
| **Thread support** | ❌ No | ✅ Yes (native threads) |
| **API style** | WebAssembly API | Custom FFI API |
| **Code reuse** | Uses existing WASM modules | Requires FFI wrapper code |
| **Build complexity** | Moderate | Higher |
| **Performance** | Near-native | Native |
| **Status** | Active development | Experimental |

### Potential Hybrid Strategy

Polygen could complement the ledger-uniffi approach:

1. **ledger-uniffi** for core cryptographic operations requiring native threading
2. **Polygen** for WASM modules where single-threaded execution is acceptable (e.g., `onchain-runtime-wasm`)
3. **Server-side proving** as an option for heavy proof generation workloads

### Integration Example

```javascript
// polygen.config.mjs
import { localModule, polygenConfig } from '@callstack/polygen-config';

export default polygenConfig({
  modules: [
    localModule('node_modules/@midnight-ntwrk/onchain-runtime-v1/midnight_onchain_runtime_wasm_bg.wasm'),
  ],
});
```

```javascript
// App.js
import '@callstack/polygen/polyfill';

// Standard WebAssembly API now works in React Native
const module = await WebAssembly.compile(wasmBuffer);
const instance = await WebAssembly.instantiate(module, imports);
```

### Conclusion

Polygen offers a viable path for running certain WASM modules on mobile without JIT, but its lack of thread support makes it **complementary rather than a replacement** for the native FFI approach. For Midnight's ZK proving operations, ledger-uniffi remains the preferred solution due to its support for native multi-threading.

---

## 18. Alternative Approach: Manual Rust FFI (Direct Native Modules)

https://rust-dd.com/post/building-a-rust-native-module-for-react-native-on-ios-and-android

An alternative to UniFFI's automated binding generation is building Rust native modules manually using direct FFI. This approach provides maximum control over the native interface at the cost of more boilerplate code.

### How It Works

```
Rust Library (staticlib + cdylib)
    ↓
┌─────────────────────────────────────────────────────────────┐
│ iOS                          │ Android                      │
│ ─────                        │ ───────                      │
│ .a static library            │ .so shared library           │
│ C header file                │ JNI bindings in Rust         │
│ Swift/ObjC bridge code       │ Kotlin external declarations │
│ CocoaPods integration        │ jniLibs directory            │
└─────────────────────────────────────────────────────────────┘
    ↓
React Native Module (Expo or bare)
    ↓
JavaScript API
```

### Rust Library Configuration

```toml
# Cargo.toml
[lib]
crate-type = [
    "staticlib",  # for iOS (.a file)
    "cdylib"      # for Android (.so file)
]
```

### Build Targets

| Platform | Rust Target | Output | Use Case |
|----------|-------------|--------|----------|
| iOS Device | `aarch64-apple-ios` | `.a` | Physical iPhones/iPads |
| iOS Simulator | `aarch64-apple-ios-sim` | `.a` | Apple Silicon simulators |
| Android ARM64 | `aarch64-linux-android` | `.so` | Modern Android devices |
| Android ARM32 | `armv7-linux-androideabi` | `.so` | Older Android devices |
| Android x86_64 | `x86_64-linux-android` | `.so` | Android emulators |
| Android x86 | `i686-linux-android` | `.so` | Older emulators |

### Android JNI Bindings

Manual JNI bindings must be written for each exposed function:

```rust
#[cfg(target_os = "android")]
pub mod android {
    use jni::objects::JClass;
    use jni::sys::jint;
    use jni::JNIEnv;

    #[no_mangle]
    pub unsafe extern "C" fn Java_com_midnight_ledger_LedgerModule_coinNullifier(
        env: JNIEnv,
        _class: JClass,
        coin_info: jbyteArray,
        secret_key: jbyteArray,
    ) -> jbyteArray {
        // Manual conversion and function call
    }
}
```

### iOS Swift Bridge

Swift code must declare and call C functions:

```swift
// Bridge header declares C functions
// rust_ledger.h
int32_t rust_add(int32_t a, int32_t b);
uint8_t* coin_nullifier(const uint8_t* coin_info, const uint8_t* secret_key);

// Swift module calls them
@objc(LedgerModule)
class LedgerModule: NSObject {
    @objc func coinNullifier(_ coinInfo: Data, secretKey: Data,
                              resolver: @escaping RCTPromiseResolveBlock,
                              rejecter: @escaping RCTPromiseRejectBlock) {
        // Call Rust function and handle result
    }
}
```

### Kotlin Integration

```kotlin
class LedgerModule : Module() {
    companion object {
        init {
            System.loadLibrary("midnight_ledger")
        }
    }

    // Declare external functions matching JNI exports
    external fun coinNullifier(coinInfo: ByteArray, secretKey: ByteArray): ByteArray
}
```

### Comparison: Manual FFI vs UniFFI

| Aspect | Manual FFI | UniFFI (ledger-uniffi) |
|--------|------------|------------------------|
| **Binding generation** | Manual JNI + Swift code | Auto-generated from Rust |
| **Type safety** | Manual conversion | Automatic type mapping |
| **Maintenance** | High - update bindings for each change | Low - regenerate bindings |
| **Control** | Maximum flexibility | Constrained by UniFFI patterns |
| **Boilerplate** | Significant | Minimal |
| **Error handling** | Manual propagation | Automatic Result mapping |
| **Complex types** | Manual serialization | Built-in support |

### When to Use Manual FFI

| Scenario | Recommendation |
|----------|----------------|
| Few simple functions | ✅ Manual FFI is straightforward |
| Many functions with complex types | ❌ UniFFI saves significant effort |
| Need custom memory management | ✅ Manual FFI provides control |
| Rapid iteration during development | ❌ UniFFI regenerates bindings quickly |
| Integration with existing C API | ✅ Manual FFI maps directly |

### Project Structure

```
rust-module/
├── rust_backend/
│   ├── Cargo.toml
│   └── src/
│       └── lib.rs              # Rust implementation + JNI bindings
├── ios/
│   ├── rust/
│   │   ├── librust_backend.a   # Compiled static library
│   │   └── rust_backend.h      # C header
│   └── LedgerModule.swift      # Swift bridge
├── android/
│   └── src/main/
│       ├── jniLibs/
│       │   ├── arm64-v8a/
│       │   │   └── librust_backend.so
│       │   ├── armeabi-v7a/
│       │   │   └── librust_backend.so
│       │   └── x86_64/
│       │       └── librust_backend.so
│       └── kotlin/
│           └── LedgerModule.kt  # Kotlin declarations
└── src/
    └── index.ts                 # JavaScript API
```

### Advantages for Midnight

1. **No UniFFI dependency** - Fewer build dependencies
2. **Direct control** - Custom memory management for large cryptographic structures
3. **Optimized serialization** - Can use binary formats tuned for specific types
4. **Gradual adoption** - Can expose functions incrementally

### Disadvantages for Midnight

1. **Significant boilerplate** - Each function needs JNI + Swift + Kotlin code
2. **Error-prone** - Manual type conversions can introduce bugs
3. **Maintenance burden** - API changes require updating multiple files
4. **Complex types** - Structs like `ProofPreimage` require manual serialization

### Conclusion

Manual Rust FFI provides maximum control but requires significantly more development effort than UniFFI. For a large API surface like the Midnight Ledger (with dozens of functions and complex types), **UniFFI's automated approach is more practical**. Manual FFI may be appropriate for performance-critical code paths where custom memory management is beneficial.
