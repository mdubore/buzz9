# Desktop Native TLS Roots Design

## Problem

Buzz Desktop opens its relay session through `tokio-tungstenite`. The desktop
crate currently enables `rustls-tls-webpki-roots`, so the native WebSocket
client trusts only the public WebPKI root bundle compiled into the application.
It does not trust private certificate authorities installed in the operating
system trust store.

StartOS uses a private local certificate authority for HTTPS and secure
WebSocket endpoints such as `wss://his-sandbag.local:50019`. The StartOS CA is
installed and trusted by the host operating system, but Buzz Desktop rejects
the relay certificate because the native WebSocket stack never loads those
native roots.

## Decision

Enable the `rustls-tls-native-roots` feature for the desktop crate's
`tokio-tungstenite` dependency instead of `rustls-tls-webpki-roots`.

This keeps TLS certificate-chain and hostname validation enabled. It does not
add an insecure certificate bypass, custom certificate exception, or relay
specific trust rule. Public certificates continue to work because the
operating system trust store includes public roots, while locally installed
private roots such as the StartOS CA also become available.

## Scope

The dependency change applies only to `desktop/src-tauri`. Server crates and
other workspace binaries retain their existing TLS behavior. The relay URL and
StartOS package configuration are unchanged.

## Verification

1. Add a manifest contract test that requires the desktop WebSocket dependency
   to use native roots and rejects an accidental return to WebPKI-only roots.
2. Add an ignored live WebSocket test controlled by `BUZZ_TEST_WSS_URL`.
3. Run the contract test before the dependency change and observe the expected
   failure.
4. Switch the feature and rerun the focused and native WebSocket tests.
5. Run the live test against the installed StartOS CA and relay endpoint.
6. Rebuild the Debian package and inspect its metadata, files, linked
   libraries, and checksum before installation.
