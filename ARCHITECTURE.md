# SophiaDOS Architecture

This document explains how the three primitives (HyperToken, Freebird, Witness) compose to enable applications like Scarcity and Clout.

---

## Core Insight: Orthogonal Primitives

SophiaDOS separates three concerns that are typically conflated:

| Concern | Traditional Approach | SophiaDOS Primitive |
|---------|---------------------|---------------------|
| **State** | Central database or blockchain | HyperToken (CRDTs) |
| **Identity** | Accounts, logins, tracking | Freebird (blind signatures) |
| **Time** | Trusted server or PoW | Witness (threshold signatures) |

By solving each independently, applications can mix and match based on their needs.

---

## Primitive 1: HyperToken (State)

### The Problem
Distributed state requires consensus. Traditional solutions:
- **Central server**: Single point of failure, trust required
- **Blockchain**: Slow, expensive, public

### The Solution
**CRDTs (Conflict-Free Replicated Data Types)** achieve consensus mathematically:

```
Traditional: "Server, may I update state?" → Server decides → Response
Blockchain:  "Network, validate my update" → Mining/staking → Confirmation
HyperToken:  "I'm updating state" → CRDTs merge → Same result everywhere
```

### How It Works

```typescript
// All peers can modify state independently
peer1.dispatch("game:move", { x: 5, y: 3 });
peer2.dispatch("game:move", { x: 2, y: 7 });

// CRDTs merge automatically - no conflicts possible
// Both moves are recorded with actor + timestamp
```

### Key Properties
- **Commutative**: Order of operations doesn't matter
- **Associative**: Grouping of operations doesn't matter  
- **Idempotent**: Duplicate operations have no effect

### Architecture
```
┌─────────────────────────────────────────┐
│              HyperToken                  │
├─────────────────────────────────────────┤
│  Chronicle (Automerge CRDT)             │
│  ├── State document                     │
│  ├── Change history                     │
│  └── Sync protocol                      │
├─────────────────────────────────────────┤
│  Engine (Game Logic)                    │
│  ├── Action dispatcher                  │
│  ├── Rule engine                        │
│  └── Game loop                          │
├─────────────────────────────────────────┤
│  Network (P2P Transport)                │
│  ├── WebSocket relay                    │
│  ├── WebRTC direct                      │
│  └── Hybrid manager                     │
└─────────────────────────────────────────┘
```

---

## Primitive 2: Freebird (Identity)

### The Problem
Authorization systems require identity. But identity enables:
- Tracking and surveillance
- Correlation across services
- Discrimination and censorship

### The Solution
**VOPRF (Verifiable Oblivious Pseudorandom Function)** separates authorization from identity:

```
Traditional: "Who are you?" → Verify identity → Grant access
Freebird:    "Can you?" → Verify token → Grant access (identity unknown)
```

### How It Works

```
User                     Issuer                  Verifier
  │                        │                        │
  │  1. Blind(input)       │                        │
  ├───────────────────────►│                        │
  │                        │                        │
  │  2. Sign(blinded)      │                        │
  │◄───────────────────────┤                        │
  │                        │                        │
  │  3. Unblind → token    │                        │
  │                        │                        │
  │  4. Present token      │                        │
  ├────────────────────────┼───────────────────────►│
  │                        │                        │
  │  5. ✓ Valid            │  (Cannot correlate     │
  │◄───────────────────────┼────────────────────────┤
  │                        │   steps 1-2 with 4-5)  │
```

### Key Properties
- **Unlinkability**: Issuer cannot correlate issuance with redemption
- **Unforgeability**: Only issuer's key can create valid tokens
- **Single-use**: Nullifier prevents double-spending

### Sybil Resistance Stack
```
┌─────────────────────────────────────────┐
│           Freebird Issuer               │
├─────────────────────────────────────────┤
│  Sybil Resistance (pick one or combine) │
│  ├── Invitation trees (web of trust)    │
│  ├── Proof of work (computational cost) │
│  ├── Rate limiting (IP/fingerprint)     │
│  └── WebAuthn (hardware attestation)    │
├─────────────────────────────────────────┤
│  Token Issuance                         │
│  ├── VOPRF blind signing                │
│  ├── Batch issuance                     │
│  └── Key rotation                       │
└─────────────────────────────────────────┘
```

---

## Primitive 3: Witness (Time)

### The Problem
Distributed systems need ordering. Traditional solutions:
- **Trusted timestamp server**: Single point of failure
- **Blockchain**: Slow (15s+), expensive (gas fees)

### The Solution
**Threshold signatures** distribute trust across multiple witnesses:

```
Traditional: "Server, what time is it?" → Trusted response
Blockchain:  "Network, include my tx" → Mining → Confirmation  
Witness:     "Witnesses, sign this hash" → Threshold met → Attestation
```

### How It Works

```
Client                   Gateway                 Witnesses (n=5, t=3)
  │                        │                        │
  │  1. Hash + request     │                        │
  ├───────────────────────►│                        │
  │                        │  2. Fan out            │
  │                        ├───────────────────────►│
  │                        │                        │
  │                        │  3. Collect sigs       │
  │                        │◄───────────────────────┤
  │                        │                        │
  │  4. Aggregated attestation (3+ signatures)     │
  │◄───────────────────────┤                        │
```

### Key Properties
- **Byzantine fault tolerance**: Requires t-of-n witnesses to collude
- **Instant**: No mining or block confirmation
- **Free**: No gas fees (optional anchoring costs)
- **Private**: Only hashes submitted

### Optional External Anchoring
```
┌─────────────────────────────────────────┐
│           Witness Gateway               │
├─────────────────────────────────────────┤
│  Threshold Signing (instant, free)      │
│  ├── Ed25519 signatures                 │
│  └── BLS aggregation (optional)         │
├─────────────────────────────────────────┤
│  External Anchors (batched, optional)   │
│  ├── Internet Archive                   │
│  ├── Transparency logs (Trillian)       │
│  ├── DNS TXT records                    │
│  └── Ethereum/EVM chains                │
└─────────────────────────────────────────┘
```

---

## Composition: How Applications Use All Three

### Scarcity (Money)

```
┌─────────────────────────────────────────────────────────────┐
│                        SCARCITY                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   MINT                                                       │
│   └── Create token with secret + amount                      │
│                                                              │
│   TRANSFER                                                   │
│   ├── Generate nullifier: H(secret || tokenId || timestamp)  │
│   ├── Get Witness attestation (ordering)          ← WITNESS  │
│   ├── Blind recipient via Freebird (privacy)      ← FREEBIRD │
│   └── Gossip nullifier via HyperToken (broadcast) ← HYPERTOKEN│
│                                                              │
│   VALIDATE                                                   │
│   ├── Check gossip network for nullifier (fast)   ← HYPERTOKEN│
│   ├── Check Witness for ordering (deterministic)  ← WITNESS  │
│   └── Confidence score based on time + peers                 │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Gossip Logic**: "Seen this nullifier before? **REJECT** (double-spend)"

### Clout (Social)

```
┌─────────────────────────────────────────────────────────────┐
│                         CLOUT                                │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   POST                                                       │
│   ├── Acquire day pass via Freebird (anti-spam)   ← FREEBIRD │
│   ├── Get Witness attestation (timestamp)         ← WITNESS  │
│   ├── Sign with identity keypair                             │
│   └── Broadcast via HyperToken relay              ← HYPERTOKEN│
│                                                              │
│   RECEIVE                                                    │
│   ├── Check trust graph (is author trusted?)                 │
│   ├── Verify Witness attestation                  ← WITNESS  │
│   └── Accept or auto-shadowban based on trust                │
│                                                              │
│   DARK SOCIAL GRAPH                                          │
│   ├── Trust signals stored locally (browser)                 │
│   ├── Encrypted trust broadcasts               ← HYPERTOKEN  │
│   └── Only recipient can decrypt                             │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Gossip Logic**: "Trust this author? **ACCEPT** (propagate signal)"

---

## The Duality

Scarcity and Clout are **algebraic inverses**:

| Dimension | Scarcity | Clout |
|-----------|----------|-------|
| Primitive | Token (value) | Post (content) |
| Core operation | `transfer()` | `post()` |
| Gossip logic | "Seen? REJECT" | "Trusted? ACCEPT" |
| Conservation | Value is conserved | Signal propagates |
| Scarcity model | Tokens are scarce | Attention is scarce |

Same infrastructure, opposite semantics.

---

## Data Flow Diagrams

### Scarcity Transfer

```
Sender                                              Recipient
   │                                                    │
   │  1. Create nullifier                               │
   │     H(secret ∥ tokenId ∥ timestamp)               │
   │                                                    │
   │  2. Get Witness attestation ──────────► Witness    │
   │     ◄─────────────────────── signed timestamp      │
   │                                                    │
   │  3. Blind recipient ─────────────────► Freebird    │
   │     ◄─────────────────────── blinded commitment    │
   │                                                    │
   │  4. Broadcast nullifier ─────────────► HyperToken  │
   │     (gossip to all peers)               P2P Network│
   │                                                    │
   │  5. Send transfer package ────────────────────────►│
   │     (nullifier + proof + commitment)               │
   │                                                    │
   │                                    6. Validate     │
   │                                       • Check gossip
   │                                       • Check Witness
   │                                       • Accept/reject
```

### Clout Post

```
Author                                              Reader
   │                                                    │
   │  1. Acquire day pass ────────────────► Freebird    │
   │     ◄─────────────────────── anonymous token       │
   │                                                    │
   │  2. Get timestamp ───────────────────► Witness     │
   │     ◄─────────────────────── signed attestation    │
   │                                                    │
   │  3. Sign post with keypair                         │
   │                                                    │
   │  4. Broadcast ───────────────────────► HyperToken  │
   │     (gossip to all peers)               P2P Network│
   │                                                    │
   │                                    5. Receive      │
   │                                       │            │
   │                              ┌────────┴────────┐   │
   │                              │ Trust check     │   │
   │                              │ • In graph?     │   │
   │                              │ • Score > min?  │   │
   │                              └────────┬────────┘   │
   │                                       │            │
   │                              ┌────────┴────────┐   │
   │                              │ YES: Accept     │   │
   │                              │ NO: Shadowban   │   │
   │                              └─────────────────┘   │
```

---

## Security Model

### Trust Assumptions

| Primitive | Trust Assumption |
|-----------|------------------|
| HyperToken | Peers are honest (or host-authoritative mode) |
| Freebird | Issuer and verifier don't collude |
| Witness | Fewer than t witnesses are compromised |

### What's Protected

| Attack | Protection |
|--------|------------|
| Double-spend | Nullifier gossip + Witness ordering |
| Spam | Freebird rate limiting |
| Sybil | Invitation trees, PoW, WebAuthn |
| Correlation | VOPRF unlinkability |
| Censorship | P2P gossip, no central authority |
| Forgery | Threshold signatures |

### What's NOT Protected

| Risk | Mitigation |
|------|------------|
| Token theft | Use TLS, secure storage |
| Timing correlation | Use Tor/mixnets |
| Quantum adversaries | Future: post-quantum crypto |
| Key compromise | Key rotation, forward secrecy |

---

## Performance Characteristics

### HyperToken
- **Latency**: <1ms local, network-bound for sync
- **Throughput**: 10,000+ ops/sec (Rust/WASM core)
- **Storage**: O(actions) for history

### Freebird
- **Issuance**: <10ms per token
- **Verification**: <5ms per token
- **Batch**: 100+ tokens/sec

### Witness
- **Attestation**: <100ms (network bound)
- **Verification**: <1ms
- **Anchoring**: Batched (hourly/daily)

### Scarcity (Combined)
- **Transfer**: <200ms end-to-end
- **Validation**: 5s (tunable confidence)
- **Energy**: ~0.0001 kWh per transaction

### Clout (Combined)
- **Post**: <500ms end-to-end
- **Feed update**: Real-time (SSE)
- **Trust computation**: O(hops × connections)

---

## Deployment Topologies

### Development (Single Machine)
```
┌─────────────────────────────────────┐
│           Docker Compose            │
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐   │
│  │HT   │ │FB   │ │WIT  │ │APP  │   │
│  │Relay│ │Issuer│ │Gate │ │    │   │
│  └─────┘ └─────┘ └─────┘ └─────┘   │
└─────────────────────────────────────┘
```

### Production (Distributed)
```
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  Region A    │  │  Region B    │  │  Region C    │
│  ┌────────┐  │  │  ┌────────┐  │  │  ┌────────┐  │
│  │Witness │  │  │  │Witness │  │  │  │Witness │  │
│  │Node 1  │  │  │  │Node 2  │  │  │  │Node 3  │  │
│  └────────┘  │  │  └────────┘  │  │  └────────┘  │
│  ┌────────┐  │  │  ┌────────┐  │  │  ┌────────┐  │
│  │Freebird│  │  │  │HT Relay│  │  │  │Gateway │  │
│  │Issuer  │  │  │  │        │  │  │  │        │  │
│  └────────┘  │  │  └────────┘  │  │  └────────┘  │
└──────────────┘  └──────────────┘  └──────────────┘
```

---

## Further Reading

- [HyperToken Documentation](https://github.com/flammafex/hypertoken/blob/main/README.md)
- [Freebird Protocol](https://github.com/flammafex/freebird/blob/main/README.md)
- [Witness Federation](https://github.com/flammafex/witness/blob/main/README.md)
- [Scarcity Security Model](https://github.com/flammafex/scarcity/blob/main/SECURITY.md)
- [Clout Trust Graph](https://github.com/flammafex/clout/blob/main/README.md)