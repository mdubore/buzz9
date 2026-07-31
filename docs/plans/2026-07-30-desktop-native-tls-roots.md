# Desktop Native TLS Roots Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Make Buzz Desktop trust secure relay certificates issued by certificate authorities installed in the operating system, including StartOS local CAs.

**Architecture:** Keep the existing native Tauri WebSocket path and rustls certificate validation. Change only the desktop `tokio-tungstenite` root source from the compiled WebPKI bundle to the operating system trust store, then verify the policy statically and with an opt-in live WSS handshake.

**Tech Stack:** Rust, Tauri 2, tokio-tungstenite 0.29, rustls 0.23, Cargo tests, Debian/Tauri bundling

---

### Task 1: Add TLS trust policy regression coverage

**Files:**
- Modify: `desktop/src-tauri/src/native_websocket.rs`

**Step 1: Write the failing manifest contract test**

Add a test that parses `desktop/src-tauri/Cargo.toml`, asserts that
`tokio-tungstenite` enables `rustls-tls-native-roots`, and asserts that it does
not enable `rustls-tls-webpki-roots`.

**Step 2: Add the opt-in live WSS test**

Add an ignored async test that reads `BUZZ_TEST_WSS_URL`, establishes a
`tokio_tungstenite::connect_async` connection, and closes it cleanly. Keep the
test ignored by default because it requires a reachable external relay.

**Step 3: Run the contract test to verify it fails**

Run:

```bash
cd desktop/src-tauri
. ../../bin/activate-hermit
PKG_CONFIG_PATH=/tmp/buzz-build-deps \
LIBRARY_PATH=/tmp/buzz-build-deps/link \
PATH=/tmp/buzz-build-deps/root/usr/bin:$PATH \
CMAKE_POLICY_VERSION_MINIMUM=3.5 \
cargo test native_websocket::tests::desktop_websocket_uses_native_root_certificates -- --exact --nocapture
```

Expected: FAIL because the manifest contains
`rustls-tls-webpki-roots` and does not contain `rustls-tls-native-roots`.

### Task 2: Load native operating system roots

**Files:**
- Modify: `desktop/src-tauri/Cargo.toml`
- Modify: `desktop/src-tauri/Cargo.lock`

**Step 1: Make the minimal dependency change**

Replace the `tokio-tungstenite` feature `rustls-tls-webpki-roots` with
`rustls-tls-native-roots`.

**Step 2: Run the contract test to verify it passes**

Run the focused command from Task 1.

Expected: PASS.

**Step 3: Run native WebSocket tests**

Run:

```bash
cargo test native_websocket::tests -- --nocapture
```

Expected: all non-ignored native WebSocket tests pass.

### Task 3: Verify the StartOS relay handshake

**Files:**
- Test: `desktop/src-tauri/src/native_websocket.rs`

**Step 1: Run the ignored live test**

Run:

```bash
BUZZ_TEST_WSS_URL=wss://his-sandbag.local:50019 \
cargo test native_websocket::tests::external_wss_relay_uses_native_trust_store \
  -- --ignored --exact --nocapture
```

Expected: PASS after completing the TLS handshake and closing the socket
without disabling validation.

### Task 4: Rebuild and inspect the Debian package

**Files:**
- Create: `../Buzz_0.5.2-startos-ca1_amd64.deb`

**Step 1: Build the Tauri Debian bundle**

Run from the repository root:

```bash
. ./bin/activate-hermit
PKG_CONFIG_PATH=/tmp/buzz-build-deps \
LIBRARY_PATH=/tmp/buzz-build-deps/link \
PATH=/tmp/buzz-build-deps/root/usr/bin:$PATH \
CMAKE_POLICY_VERSION_MINIMUM=3.5 \
pnpm -C desktop tauri build --ci --bundles deb \
  --config '{"bundle":{"createUpdaterArtifacts":false}}'
```

Expected: Tauri produces `desktop/src-tauri/target/release/bundle/deb/`.

**Step 2: Copy the artifact to the current project folder**

Copy the generated package to
`../Buzz_0.5.2-startos-ca1_amd64.deb`.

**Step 3: Inspect the package**

Run `dpkg-deb --info`, `dpkg-deb --contents`, `ldd` on all packaged ELF
executables, and `sha256sum`.

Expected: package metadata is readable, required executables are present, no
dynamic library is reported missing, and a checksum is recorded.

**Step 4: Commit**

```bash
git add desktop/src-tauri/Cargo.toml desktop/src-tauri/Cargo.lock \
  desktop/src-tauri/src/native_websocket.rs
git commit -m "fix: trust native roots for desktop WebSockets"
```
