# Mortise — threat model (honest)

_Owner-facing engineering note. Register: battery-doc §6.3 honesty — no overclaiming._
_Covers SPEC FR-9 (identity & keys), FR-10 (consent record), FR-12 (channels), FR-13
(import & verification)._

## What Mortise guarantees

Mortise is a strict no-server, on-device app. There is no Mortise backend and zero
outbound network (FR-26). Two phones exchange one HPKE-sealed envelope over any transport
(AirDrop / Messages / mail / a shared file / animated QR — a phone-to-phone *nearby* channel is
scaffolded but not shipped in v1.0, see §Transports below). Concretely, the design provides:

- **Confidentiality in transit.** The exchange payload is sealed with HPKE
  (P256-HKDF-SHA256 + AES-GCM-256) to the recipient pair's public key. Only the holder of
  that per-pair private key — which lives in the Secure Enclave / keychain
  `WhenUnlockedThisDeviceOnly` and never leaves the device (FR-9) — can open it. The
  transport is therefore untrusted by design: a `.mortise` file is safe to send over any
  channel.
- **Integrity + authenticity of the container.** HPKE is authenticated encryption: any
  bit flipped in the ciphertext or the encapsulated key fails the AEAD open (surfaced as a
  calm, specific error, never a crash). A CRC-32 over the answer stream is re-verified after
  decode, and the HPKE `info` string binds each envelope to its pairID + format version, so
  an envelope sealed for pair X cannot open under pair Y.
- **Pair-identity + consent binding.** On import the app checks the pairID three ways: the
  explicit open-pair lookup, the HPKE `info` binding, and the consent record's
  `bindingDigest` (a SHA-256 over the consent fields + pairID + the sender's public key,
  FR-10). Both sides hold and can compare the same consent fingerprint ("recorded at both
  sides").
- **Replay rejection.** A re-imported identical envelope is rejected (`duplicateEnvelope`);
  a pair already past `envelopeReceived` will not re-import.
- **Local-data protection.** All pair files carry `NSFileProtectionComplete` and are
  excluded from backup (FR-23). Keys and received raw answers are deleted with the pair;
  raw answers auto-delete once the report is generated (FR-18).

## What Mortise does NOT guarantee — stated plainly

**Forgery is detectable, not impossible.**

The per-pair keys are P256 **key-agreement** keys (for HPKE), not signing keys — so the
consent record is a cryptographic **commitment**, not a digital signature. Within the pair's
threat model this is tamper-evidence, not proof of authorship.

- A **careless fake** is catchable. The Answer-Quality validity engine (FR-8) and the
  response-telemetry patterns (FR-6) flag straight-lining, impossible speed, and internal
  inconsistency; the Initiator **recomputes** validity from the received raw answers by the
  same versioned rules and can suppress or annotate unreliable sections. The report footer
  carries "checked with rules vX" plus both protocols' quality status.
- A **motivated actor with a modified client** can defeat client-side trust. Someone who
  rebuilds the app can hand-craft an envelope that decrypts and passes validity heuristics.
  There is no server to attest the client, and **App Attest / DeviceCheck are unavailable
  without a server** — which Mortise deliberately does not have. This is an **accepted
  limitation**: Mortise is a tool for two cooperating people who each want an honest answer,
  not an adversarial-verification system. The product copy never claims otherwise.

## Network posture — zero outbound, enforced (FR-26)

_Owner-facing note for the "no server ever sees your answers" claim (SPEC Goal 4)._

Mortise makes **no network calls of any kind**. There is no Mortise backend, no analytics,
no crash SDK, and — because the app is free forever — not even StoreKit. This is not a
policy the app tries to honor at runtime; it is enforced structurally at four layers, so
the App Privacy declaration ("Data Not Collected", no tracking) is **literally** true and
test-proven rather than self-attested:

1. **Source ban (compile-time).** `.swiftlint.yml` custom rules fail the build on any
   `URLSession` type reference (`no_urlsession_anywhere`), any `dataTask`/`downloadTask`/
   `uploadTask` (`no_url_datatask_anywhere`), any `import Network` /`import CFNetwork`
   (`no_network_import_anywhere`), and any third-party analytics or cloud-AI import. There
   is **no "outside Monetization" exemption** — the studio's usual carve-out does not exist
   here because there is no Monetization module. ip_lint runs on every build too.
2. **Runtime hook (test-time).** `NetworkSanityTests` installs a `URLProtocol` that records
   **and fails** every outbound request (Mortise has ZERO sanctioned hosts, so any request
   is a violation), then exercises the full product flow — onboarding, a battery run,
   pairing/consent, the demo loopback exchange, and report assembly — and asserts the
   violation set stayed empty. A regression that added a network call would fail here even
   if it slipped past the source ban (e.g. via a dependency).
3. **Binary symbol check (artifact-time).** phase-check greps the built binary's strings for
   a `URLSession` symbol; its absence corroborates the source ban against the shipped Mach-O.
4. **No entitlements / no ATS.** The target declares no network entitlements and no
   `NSAppTransportSecurity` exception — there is no host to reach.

The transports Mortise *does* use in v1.0 are all off-internet by construction: **VisionKit** /
**CoreImage** run the QR scan/generate on-device (the pairing code + the animated multi-QR
envelope frames), **UserNotifications** schedules only *local* notifications (the opt-in return
nudge, FR-28), and the sealed `.mortise` envelope moves over whatever channel the two people
choose (AirDrop / Messages / mail / a shared file) — none of which Mortise itself opens a socket
for. A **MultipeerConnectivity** nearby (Bluetooth / peer-to-peer Wi‑Fi, no server hop) channel
is *scaffolded but not wired into the shipping UI in v1.0* — it is device-only (the FR-1 finding),
its code ships `#if DEBUG` and is exercised only by tests, and it therefore makes no v1.0 claim
and stands up no advertiser/browser at runtime (security F3). When it is wired in a later version
it will carry an `NSLocalNetworkUsageDescription` + `NSBonjourServices` and its own threat note.
`PrivacyInfo.xcprivacy` declares no collected data types and a single required-reason API
(UserDefaults, `CA92.1` — the return-nudge preference).

## Why this posture is acceptable for v1.0

Mortise's job is a serious, structured compatibility check between two people who have each
chosen to take it. The value is the honest conversation it starts, not a tamper-proof
credential. The no-server design is the whole point (privacy, "lose the phone, lose the
data"), and it is incompatible with server-side attestation. We therefore ship strong
confidentiality + integrity + careless-forgery detection, and we are honest — in code
comments, in this document, and in the consent copy — about the ceiling.
