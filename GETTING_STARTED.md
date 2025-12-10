# Getting Started with SophiaDOS

This guide helps you find the right entry point based on what you want to build.

---

## Quick Decision Tree

```
What do you want to do?
│
├── Run an existing application
│   ├── Anonymous payments → Scarcity
│   └── Decentralized social → Clout
│
├── Build a new application
│   ├── Need distributed state? → Start with HyperToken
│   ├── Need anonymous auth? → Start with Freebird  
│   ├── Need timestamps? → Start with Witness
│   └── Need all three? → Start with Scarcity as reference
│
└── Understand the architecture
    └── Read ARCHITECTURE.md
```

---

## Path 1: Run Existing Applications

### Scarcity (Zero-Cost Cryptocurrency)

```bash
git clone https://github.com/flammafex/scarcity
cd scarcity
docker compose up --build

# CLI is now available
docker compose run --rm cli wallet create
docker compose run --rm cli mint 100
docker compose run --rm cli send <recipient> 50
```

**What you get:**
- Wallet management
- Token minting
- Transfers with double-spend protection
- Split/merge operations
- HTLCs for conditional payments

📖 [Full Scarcity documentation](https://github.com/flammafex/scarcity/blob/main/README.md)

---

### Clout (Dark Social Network)

```bash
git clone https://github.com/flammafex/clout
cd clout
docker compose up --build

# Open http://localhost:3000
```

**What you get:**
- Browser-based identity (keypair in IndexedDB)
- Trust graph management
- Posting with timestamps
- Encrypted DMs
- Real-time feed updates

📖 [Full Clout documentation](https://github.com/flammafex/clout/blob/main/README.md)

---

## Path 2: Build a New Application

### Step 1: Identify Your Needs

| Need | Primitive | When to Use |
|------|-----------|-------------|
| Multiplayer state sync | HyperToken | Games, collaboration, shared documents |
| Anti-spam / rate limiting | Freebird | User-generated content, voting, access control |
| Proof of existence | Witness | Audit trails, ordering, dispute resolution |
| Anonymous transactions | All three | Financial applications, private messaging |

---

### Step 2: Start with One Primitive

#### HyperToken Only

Best for: Games, real-time collaboration, offline-first apps

```bash
# Quickstart CLI
npx hypertoken-quickstart

# Or clone and explore
git clone https://github.com/flammafex/hypertoken
cd hypertoken
npm install
npm run dev
```

**Key concepts:**
- `Chronicle` - CRDT document (Automerge)
- `Engine` - Action dispatcher
- `Stack` / `Space` - Game primitives

```typescript
import { Engine } from 'hypertoken';

const engine = new Engine();

// Dispatch actions
engine.dispatch('stack:shuffle', { seed: 42 });
engine.dispatch('agent:drawCards', { agentId: 'player1', count: 5 });

// State syncs automatically via CRDTs
```

---

#### Freebird Only

Best for: Rate limiting, anonymous voting, access control

```bash
git clone https://github.com/flammafex/freebird
cd freebird
docker compose up --build

# Issuer: http://localhost:8081
# Verifier: http://localhost:8082
```

**Key concepts:**
- `Issuer` - Creates blind-signed tokens
- `Verifier` - Validates tokens (can't correlate with issuer)
- `Day Pass` - Rate-limited anonymous authorization

```typescript
import { FreebirdClient } from '@freebird/sdk';

const client = new FreebirdClient({
  issuerUrl: 'http://localhost:8081',
  verifierUrl: 'http://localhost:8082'
});

await client.init();
const token = await client.issueToken();
const isValid = await client.verifyToken(token);
```

---

#### Witness Only

Best for: Timestamping, audit trails, ordering

```bash
git clone https://github.com/flammafex/witness
cd witness
docker compose up --build

# Gateway: http://localhost:8080
```

**Key concepts:**
- `Witness Node` - Signs attestations
- `Gateway` - Aggregates threshold signatures
- `Attestation` - Signed timestamp proof

```bash
# Timestamp a file
docker compose exec gateway witness-cli \
  --gateway http://localhost:8080 \
  timestamp --file README.md
```

---

### Step 3: Combine Primitives

Once you understand individual primitives, combine them. Use Scarcity as a reference implementation:

```typescript
// Example: Anonymous rate-limited posting

import { FreebirdAdapter } from './adapters/freebird';
import { WitnessAdapter } from './adapters/witness';
import { HyperTokenRelay } from './adapters/hypertoken';

// 1. Get anonymous authorization
const freebird = new FreebirdAdapter({ issuerUrl, verifierUrl });
const dayPass = await freebird.issueToken();

// 2. Create timestamped content
const witness = new WitnessAdapter({ gatewayUrl });
const attestation = await witness.timestamp(contentHash);

// 3. Broadcast to network
const relay = new HyperTokenRelay({ relayUrl });
await relay.broadcast({
  content,
  dayPass,
  attestation,
  signature: sign(content, privateKey)
});
```

---

## Path 3: Understand the Stack

### Read Order

1. **[README.md](README.md)** - The vision and overview
2. **[ARCHITECTURE.md](ARCHITECTURE.md)** - How primitives compose
3. **Individual project READMEs** - Deep dives

### Key Papers/References

**CRDTs:**
- [Automerge](https://automerge.org/) - The CRDT library HyperToken uses
- [A Conflict-Free Replicated JSON Datatype](https://arxiv.org/abs/1608.03960) - Kleppmann et al.

**VOPRFs:**
- [RFC 9497](https://www.rfc-editor.org/rfc/rfc9497.html) - Oblivious Pseudorandom Functions
- [Privacy Pass](https://privacypass.github.io/) - Similar protocol for rate limiting

**Threshold Signatures:**
- [Practical Threshold Signatures](https://eprint.iacr.org/2020/852) - BLS aggregation
- [Witness CoSigning](https://arxiv.org/abs/1503.08768) - Distributed timestamping

**Economics:**
- [Freigeld](https://en.wikipedia.org/wiki/Freigeld) - Gesell's demurrage currency
- [Wörgl Experiment](https://en.wikipedia.org/wiki/W%C3%B6rgl#The_W%C3%B6rgl_Experiment) - Historical precedent

---

## Common Patterns

### Pattern: Anonymous Action with Timestamp

```typescript
// User wants to do something anonymously with proof of time

// 1. Get anonymous token (proves authorization, not identity)
const token = await freebird.issueToken();

// 2. Do the action
const action = { type: 'vote', choice: 'yes' };

// 3. Timestamp it (proves when, not who)
const attestation = await witness.timestamp(hash(action));

// 4. Submit
await submitAction({ action, token, attestation });
```

### Pattern: Double-Spend Prevention

```typescript
// User wants to transfer a unique asset

// 1. Generate nullifier (unique to this spend)
const nullifier = hash(secret + assetId + timestamp);

// 2. Timestamp for ordering
const attestation = await witness.timestamp(nullifier);

// 3. Broadcast nullifier to gossip network
await hypertoken.broadcast({ type: 'nullifier', nullifier, attestation });

// 4. Recipient waits for gossip propagation
await sleep(5000);

// 5. If nullifier not seen elsewhere, accept transfer
const seenBefore = await hypertoken.checkNullifier(nullifier);
if (seenBefore) reject('Double spend detected');
```

### Pattern: Trust-Gated Content

```typescript
// Content should only be visible to trusted users

// 1. Post with timestamp
const post = { content, author: publicKey };
const attestation = await witness.timestamp(hash(post));

// 2. Broadcast to network
await hypertoken.broadcast({ post, attestation, signature });

// 3. Receivers filter by trust graph
function shouldShow(post, myTrustGraph) {
  const trustScore = myTrustGraph.getScore(post.author);
  return trustScore >= MIN_TRUST_THRESHOLD;
}
```

---

## Development Setup

### Prerequisites

| Tool | Version | Purpose |
|------|---------|---------|
| Docker | 20.10+ | Container runtime |
| Node.js | 20+ | JavaScript runtime |
| Rust | 1.70+ | WASM compilation (optional) |

### Environment

All projects use similar patterns:

```bash
# Clone
git clone https://github.com/flammafex/<project>
cd <project>

# Copy environment
cp .env.example .env

# Start with Docker
docker compose up --build

# Or run locally
npm install
npm run dev
```

### Testing

```bash
# Unit tests
npm test

# Integration tests (requires Docker services)
npm run test:integration

# Specific project tests
npm run test:hypertoken
npm run test:freebird
npm run test:witness
```

---

## Getting Help

### Documentation
- Each project has detailed README and docs/
- [ARCHITECTURE.md](ARCHITECTURE.md) explains composition

### Issues
- File on the relevant project repository
- Tag with `question` for help, `bug` for problems

### Philosophy
- [The Carpocratian Church](https://carpocratian.org/en/church/) - The "why" behind the "what"

---

## What to Build

SophiaDOS enables applications that were previously thought impossible without blockchain or central servers:

| Application Type | Stack |
|-----------------|-------|
| **Anonymous voting** | Freebird + Witness |
| **Multiplayer games** | HyperToken + (optional) Witness |
| **Private messaging** | HyperToken + Freebird |
| **Digital currency** | All three (see Scarcity) |
| **Social networks** | All three (see Clout) |
| **Collaborative documents** | HyperToken |
| **Audit logs** | Witness |
| **Rate-limited APIs** | Freebird |
| **Prediction markets** | All three |
| **Decentralized identity** | Freebird + Witness |

The primitives are building blocks. What you build is up to you.

---

<div align="center">

*Build the temple. Leave the door open.*

</div>