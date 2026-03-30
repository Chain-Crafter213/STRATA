# STRATA — The Confidential Venture & Capital Operating System

### *"Deploy. Vest. Trade. Privately."*

---

> **STRATA** is the first FHE-powered operating system for Web3 capital management. Syndicate formation, token vesting, OTC secondary trading, LP governance, and portfolio risk scoring — all operating entirely on encrypted state. No allocation amount, no investor identity, no deal size, and no vote weight ever appears as plaintext on-chain.

---

## The Problem

Web3 venture capital is fundamentally broken by blockchain transparency.

### 1. Vesting Exposure
When a VC's token allocation unlocks, the entire market sees it on-chain in real time. MEV bots and retail traders front-run the claim transaction, dumping the token price before the investor can even execute. A $10M unlock event can trigger a 15-30% price crash driven purely by on-chain surveillance — not fundamentals.

### 2. OTC Market Panic
When early investors want to exit positions before token unlock (selling SAFTs or unvested allocations OTC), any on-chain trace of the trade signals "insider selling" to the market. Token prices collapse on sentiment, not substance. The result: a $200B OTC market that operates almost entirely off-chain through Telegram brokers, with zero settlement guarantees.

### 3. Capital Formation Opacity
Syndicate pool sizes, individual LP commitments, SAFT allocation tables, and cap tables are currently managed on Web2 spreadsheets. The reason is simple: putting this data on a public blockchain would expose every fund's deal flow, allocation strategy, and portfolio composition to competitors, counter-parties, and the general public.

### 4. Governance Coercion
When LP voting power is derived from publicly visible investment size, large investors face social pressure, targeted lobbying, and retaliation for unpopular votes. Transparent governance creates coercion risk.

---

## The Solution

STRATA is a unified protocol suite where every capital operation — from syndicate formation to token distribution to secondary trading to board governance — executes on FHE-encrypted state using Fhenix CoFHE.

The core principle: **capital moves. Data doesn't.**

Every dollar amount is an `euint64`. Every vote weight is an `euint64`. Every allocation is an `euint64`. The on-chain state is pure ciphertext. Only authorized parties — with valid CoFHE EIP-712 permits — can decrypt the specific values they are entitled to see.

---

## Protocol Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         STRATA PROTOCOL SUITE                            │
│                                                                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │ StrataVest  │  │ StrataCall  │  │ StrataOTC   │  │ StrataBoard │    │
│  │ Confidential│  │ Confidential│  │ Encrypted   │  │ Confidential│    │
│  │ Vesting     │  │ Syndicate   │  │ Secondary   │  │ LP          │    │
│  │ Engine      │  │ Pooling     │  │ Market      │  │ Governance  │    │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘    │
│         │                │                │                │            │
│         └────────────────┴────────────────┴────────────────┘            │
│                                    │                                    │
│                          ┌────────┴────────┐                           │
│                          │  StrataCore.sol  │                           │
│                          │  Shared ACL +    │                           │
│                          │  Role Management │                           │
│                          │  + Fee Engine    │                           │
│                          └────────┬────────┘                           │
│                                   │                                     │
│                          ┌────────┴────────┐                           │
│                          │  StrataRisk     │                           │
│                          │  AI Deal Scoring│                           │
│                          │  (Wave 5)       │                           │
│                          └─────────────────┘                           │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
                    ┌───────────────────────────────┐
                    │    FHENIX CoFHE COPROCESSOR    │
                    │  Task Manager │ ZK Verifier    │
                    │  Threshold Network │ CT Registry│
                    └───────────────────────────────┘
```

---

## Smart Contract Architecture

### StrataCore.sol — Shared Infrastructure

The foundation contract providing role-based access control, fee management, and shared FHE helpers used by all protocol modules.

**Roles:**
- `FOUNDER` — Creates vesting schedules, opens syndicates, submits startup metrics
- `INVESTOR` — Claims vested tokens, joins syndicates, trades OTC, votes on governance
- `AUDITOR` — Receives time-bounded, scoped decryption access for compliance verification
- `ADMIN` — Protocol governance and parameter updates

**Shared FHE helpers:**
- `_retainAccess(euint64 handle)` → `FHE.allowThis(handle)` — contract retains compute access
- `_grantDecrypt(euint64 handle, address to)` → `FHE.allow(handle, to)` — user can decrypt
- `_grantTimedAccess(euint64 handle, address auditor, uint256 expiry)` — time-bounded audit access

---

### StrataVest.sol — Confidential Token Vesting Engine (Wave 1)

The vesting module where founders distribute SAFT allocations as encrypted amounts. The public blockchain sees only ciphertext handles — no allocation size, no investor identity mapping, no unlock schedule details.

**Storage model:**
```
mapping(address => euint64) private encryptedAllocations;
mapping(address => euint64) private encryptedClaimed;
mapping(address => uint256) private vestingStart;
mapping(address => uint256) private vestingDuration;
mapping(address => uint256) private cliffEnd;
```

**Core functions:**

`allocate(address investor, InEuint64 encAmount)` — Founder deposits an encrypted SAFT allocation for an investor. The contract converts the input to a `euint64` handle via `FHE.asEuint64()`, adds it to any existing allocation via `FHE.add()`, retains compute access via `FHE.allowThis()`, and grants the investor decrypt access via `FHE.allow()`.

`claim(InEuint64 encClaimAmount)` — Investor claims a portion of their vested tokens. The contract subtracts the encrypted claim amount from the encrypted allocation via `FHE.sub()`. A vesting schedule check ensures the cliff period has passed and the linear vesting calculation determines the maximum claimable amount. The contract uses `FHE.lte()` to verify the claim doesn't exceed the vested portion, and `FHE.select()` to conditionally execute the transfer only if the claim is valid.

`viewAllocation(Permission calldata perm)` — Investor views their encrypted balance by providing a CoFHE EIP-712 permit. The contract returns the sealed output via `FHE.sealoutput()`, which the client-side SDK decrypts locally.

`batchAllocate(address[] investors, InEuint64[] amounts)` — Founder distributes allocations to multiple investors in a single transaction.

**FHE Operations:**
| Operation | Purpose |
|-----------|---------|
| `FHE.asEuint64(InEuint64)` | Convert encrypted input to handle |
| `FHE.add(allocation, amount)` | Accumulate allocations |
| `FHE.sub(allocation, claim)` | Deduct claimed tokens |
| `FHE.lte(claim, vestedAmount)` | Verify claim ≤ vested portion |
| `FHE.select(valid, claim, zero)` | Conditional claim execution |
| `FHE.allowThis(handle)` | Contract retains compute access |
| `FHE.allow(handle, investor)` | Investor can decrypt own allocation |
| `FHE.sealoutput(handle, pubKey)` | Sealed output for client decryption |

---

### StrataCall.sol — Confidential Syndicate Pooling (Wave 2)

Private capital formation where LP commitments are fully encrypted.

**Concept:** A syndicate lead opens a deal (e.g., "$2M allocation in Project X"). LPs deposit capital via encrypted transactions. Individual commitment sizes are stored as `euint64` — no LP knows what any other LP committed. The total pool size is an encrypted accumulator. The syndicate lead can view the encrypted total via their permit but cannot see individual LP amounts.

**Storage model:**
```
struct Syndicate {
    address         lead;
    euint64         encryptedCap;        // encrypted max pool size
    euint64         encryptedTotalRaised; // encrypted current total
    uint256         deadline;
    bool            active;
}

mapping(uint256 => Syndicate) private syndicates;
mapping(uint256 => mapping(address => euint64)) private lpCommitments;
```

**Core functions:**

`createSyndicate(InEuint64 encCap, uint256 deadline)` — Lead creates a syndicate with an encrypted maximum cap. Even the target raise amount is private.

`commit(uint256 syndicateId, InEuint64 encAmount)` — LP contributes capital. The contract adds the encrypted amount to the total via `FHE.add()`, then checks `FHE.lte(newTotal, cap)` to ensure the pool doesn't exceed its encrypted cap. Stablecoin settlement via Privara SDK integration.

`viewMyCommitment(uint256 syndicateId, Permission perm)` — LP views their own encrypted commitment.

`viewPoolStatus(uint256 syndicateId, Permission perm)` — Lead views encrypted total raised.

**New FHE Operations (beyond Wave 1):**
| Operation | Purpose |
|-----------|---------|
| `FHE.lte(total, cap)` | Verify pool hasn't exceeded cap |
| `FHE.gte(commitment, minimum)` | Enforce minimum LP commitment |

---

### StrataOTC.sol — Encrypted Secondary Market (Wave 3)

A dark pool specifically designed for unvested token positions and SAFT agreements.

**Concept:** An investor wants to exit a position early. They submit an encrypted ask price. A buyer submits an encrypted bid. The CoFHE coprocessor compares the two ciphertexts using `FHE.gte()`. If the bid meets or exceeds the ask, `FHE.select()` conditionally routes the tokens to the buyer and funds to the seller. The trade is mathematically invisible — no price discovery information leaks to the spot market.

**Storage model:**
```
struct Order {
    address seller;
    euint64 encryptedAskPrice;
    euint64 encryptedTokenAmount;
    bool    active;
}

struct Bid {
    address buyer;
    euint64 encryptedBidPrice;
    bool    active;
}
```

**Core functions:**

`createAsk(InEuint64 encPrice, InEuint64 encAmount)` — Seller lists unvested tokens at an encrypted price.

`submitBid(uint256 orderId, InEuint64 encBidPrice)` — Buyer submits an encrypted bid.

`matchOrder(uint256 orderId, uint256 bidId)` — The contract compares bid ≥ ask using `FHE.gte()`. On match, `FHE.select()` conditionally transfers encrypted token amounts and `FHE.min(ask, bid)` determines the settlement price (the ask price). If no match, both ciphertexts remain untouched — no information about the price gap leaks.

**New FHE Operations:**
| Operation | Purpose |
|-----------|---------|
| `FHE.gte(bid, ask)` | Check if bid meets ask |
| `FHE.select(match, tokens, zero)` | Conditional settlement |
| `FHE.min(ask, bid)` | Settlement price determination |

---

### StrataBoard.sol — Confidential LP Governance (Wave 4)

Private board voting where vote weights are derived from encrypted investment sizes.

**Concept:** LPs vote on critical decisions — releasing the next funding tranche, approving follow-on investments, changing fund terms. Each LP's voting weight equals their encrypted commitment from Wave 2 (StrataCall). Votes are tallied homomorphically: `FHE.add()` accumulates encrypted weight for each option. The result is decrypted only after the voting period ends, revealing only the winning option — never individual votes or weights.

**Core functions:**

`createProposal(string description, uint256 duration)` — Lead creates a governance proposal.

`castVote(uint256 proposalId, InEuint64 encVote)` — LP casts an encrypted vote weighted by their investment commitment. The contract multiplies vote direction (0 or 1) by their encrypted weight using `FHE.mul()` and adds to the encrypted tally via `FHE.add()`.

`finalizeProposal(uint256 proposalId)` — After the voting period, the encrypted tally is decrypted via `FHE.decrypt()` to reveal the outcome. Individual votes remain permanently encrypted.

`grantAuditAccess(uint256 proposalId, address auditor, uint256 expiry)` — Syndicate lead can grant a time-bounded decryption permit to a regulatory auditor for compliance. The auditor can verify the aggregate vote count matches the outcome but cannot see individual LP votes.

**New FHE Operations:**
| Operation | Purpose |
|-----------|---------|
| `FHE.mul(voteDir, weight)` | Weight votes by investment size |
| `FHE.decrypt(finalTally)` | Reveal outcome after period ends |

---

### StrataRisk.sol — AI-Driven Deal Scoring (Wave 5)

Encrypted portfolio intelligence where startup health metrics are scored without data exposure.

**Concept:** Portfolio companies submit financial metrics (monthly burn rate, runway, revenue growth, user acquisition cost) as encrypted values. An on-chain computation template multiplies these by proprietary risk weights using `FHE.mul()`, accumulates via `FHE.add()`, and normalizes via `FHE.div()`. The output is an encrypted "Health Score" that automatically adjusts the company's borrowing power on StrataOTC and influences governance proposals — without anyone seeing the raw financial data.

**New FHE Operations:**
| Operation | Purpose |
|-----------|---------|
| `FHE.mul(metric, weight)` | Weighted risk calculation |
| `FHE.div(sum, count)` | Score normalization |
| `FHE.gt(score, threshold)` | Health classification |

---

## Complete FHE Operations Matrix

| Operation | Wave | Contract | Purpose |
|-----------|------|----------|---------|
| `FHE.asEuint64(InEuint64)` | 1 | All | Convert encrypted input |
| `FHE.asEuint64(uint64)` | 1 | All | Create encrypted constants |
| `FHE.add(a, b)` | 1 | Vest, Call, Board | Accumulate allocations, pools, votes |
| `FHE.sub(a, b)` | 1 | Vest | Deduct claimed tokens |
| `FHE.lte(a, b)` | 1 | Vest, Call | Verify claim ≤ vested, total ≤ cap |
| `FHE.select(cond, a, b)` | 1 | Vest, OTC | Conditional execution |
| `FHE.allowThis(handle)` | 1 | All | Contract retains compute |
| `FHE.allow(handle, addr)` | 1 | All | Grant decrypt access |
| `FHE.sealoutput(handle, key)` | 1 | All | Sealed output for permits |
| `FHE.isInitialized(handle)` | 1 | Vest | Check allocation exists |
| `FHE.gte(a, b)` | 3 | OTC | Bid ≥ Ask matching |
| `FHE.min(a, b)` | 3 | OTC | Settlement price |
| `FHE.mul(a, b)` | 4 | Board, Risk | Weighted voting, risk scores |
| `FHE.div(a, b)` | 5 | Risk | Score normalization |
| `FHE.gt(a, b)` | 5 | Risk | Health classification |
| `FHE.decrypt(handle)` | 4 | Board | Reveal final vote tally |

**Total: 16 distinct FHE operations across the protocol suite.**

---

## Data Flow — Confidential Vesting Lifecycle

```
FOUNDER                          STRATA CONTRACT                      INVESTOR
───────                          ──────────────                      ────────
    │                                 │                                  │
    │  1. Encrypt allocation          │                                  │
    │     cofhe.encryptInputs([       │                                  │
    │       Encryptable.uint64(50000) │                                  │
    │     ])                          │                                  │
    │                                 │                                  │
    │  2. allocate(investor, enc) ───▶│                                  │
    │                                 │  3. FHE.asEuint64(enc)           │
    │                                 │     FHE.add(existing, amount)    │
    │                                 │     FHE.allowThis(alloc)         │
    │                                 │     FHE.allow(alloc, investor)   │
    │                                 │                                  │
    │                                 │                 After cliff ────▶│
    │                                 │                                  │
    │                                 │  4. claim(encClaimAmt) ◀─────────│
    │                                 │     FHE.lte(claim, vested) check │
    │                                 │     FHE.select(valid, claim, 0)  │
    │                                 │     FHE.sub(alloc, validClaim)   │
    │                                 │     FHE.allow(newAlloc, investor)│
    │                                 │                                  │
    │                                 │                                  │  5. viewAllocation(permit)
    │                                 │                                  │     FHE.sealoutput(alloc, pk)
    │                                 │                                  │     Client decrypts: "50,000"
    │                                 │                                  │
    │  Block explorer sees:           │  On-chain state:                 │
    │  0x8f7a2b... (ciphertext)       │  euint64 handles only            │  Only they see: 50,000
    │                                 │                                  │
```

---

## Access Control Model

| Data | Who Can Decrypt | FHE Primitive |
|------|----------------|---------------|
| Individual investor allocation | Investor only | `FHE.allow(alloc, investor)` |
| Vested amount calculation | Contract (for computation) | `FHE.allowThis(alloc)` |
| LP commitment in syndicate | LP only | `FHE.allow(commitment, lp)` |
| Syndicate total raised | Lead only | `FHE.allow(total, lead)` |
| OTC bid/ask prices | Respective submitter only | `FHE.allow(price, submitter)` |
| Vote weight | Voter only | `FHE.allow(weight, voter)` |
| Final vote tally | Public (post-period) | `FHE.decrypt(tally)` |
| Audit access | Time-bounded auditor | Scoped `FHE.allow()` with expiry |
| Raw data | NEVER public | No `FHE.allowPublic()` on positions |

### Security Invariants

1. **Allocation amounts never public** — `FHE.allowPublic()` is never called on individual allocations
2. **Cross-investor isolation** — Investor A cannot decrypt Investor B's allocation, commitment, or vote
3. **Founder cannot see claims** — Founders see total allocation but not individual claim transactions
4. **OTC price privacy** — Failed bid/ask matches reveal zero information about the price gap
5. **Gas side-channel resistance** — `FHE.select()` executes identically regardless of branch outcome
6. **Temporal audit access** — Auditor permits expire automatically, preventing permanent surveillance

---

## Threat Model

| Threat | Mitigation |
|--------|-----------|
| **MEV bots front-run vesting claims** | IMPOSSIBLE — claim amounts are encrypted. Bots see only ciphertext handles. |
| **Market surveys OTC trades** | Trade prices and volumes are encrypted. Failed matches leak zero data. |
| **Competitor reads syndicate cap** | Pool size and cap stored as euint64. Only the lead can decrypt total. |
| **Large LP coerced on governance votes** | Votes are encrypted and weighted by encrypted commitment. Individual votes never revealed. |
| **Block explorer exposes allocations** | All storage is ciphertext. Explorer shows only handle hashes. |
| **Contract admin reads investor data** | Admin role has no decrypt access on investor-specific handles. |
| **Replay attack on claims** | Nonce per claim + `FHE.sub()` updates state atomically. |

---

## Why FHE Is Required

| Alternative | Why It Fails for STRATA |
|-------------|------------------------|
| **Zero-Knowledge Proofs** | Can prove "I have ≥ X tokens vested" but cannot execute encrypted arithmetic (add, subtract, compare) on hidden balances. Cannot do blind OTC matching. |
| **Multi-Party Computation** | Requires all syndicate LPs online simultaneously for every pool operation. Impractical for capital formation with global investors across time zones. |
| **Trusted Execution Environments** | Allocation amounts exist as plaintext inside the enclave. Hardware side-channel attacks expose all investor positions. |
| **Commit-Reveal** | Reveal phase exposes allocation amounts to the public. Privacy is temporary. |
| **FHE** | Performs arithmetic on encrypted allocations, compares encrypted OTC bids, and tallies encrypted votes — all without any value ever being plaintext. Async, persistent, no hardware trust. |

Remove FHE from STRATA and the confidential vesting engine, the blind OTC matching, and the encrypted governance all cease to function. Every core operation depends on homomorphic arithmetic over ciphertext.

---

## 5-Wave Strategic Roadmap

| Wave | Module | Focus | Difficulty | Key FHE Additions |
|------|--------|-------|------------|-------------------|
| **1** | StrataVest | Confidential token vesting architecture | Foundation | `add`, `sub`, `lte`, `select`, `allow`, `sealoutput` |
| **2** | StrataCall | Confidential syndicate pooling | Easy | `gte` for minimums, Privara SDK |
| **3** | StrataOTC | Encrypted secondary market | Medium | `gte` matching, `min` settlement |
| **4** | StrataBoard | Confidential LP governance | Hard | `mul` weighting, `decrypt` finalization |
| **5** | StrataRisk | AI deal scoring | Expert | `mul`/`div` matrix ops, `gt` classification |

---

## UX Philosophy — The Privacy Console

Every page in STRATA features a persistent "Privacy Console" panel displaying three views of the same transaction:

**Public View** — What any block explorer sees: `0x8f7a2b...` raw ciphertext handle.

**Your View** — What the authenticated user sees after CoFHE permit decryption: `50,000 $STRT`.

**Access Control Status** — Who has decrypt access to this handle and why: `🟢 Owner (permanent) | 🟡 Auditor (expires 24h)`.

This makes FHE tangible. Judges and users immediately understand: the data exists, it computes, but only authorized eyes can read it.

---

## Identity

| Field | Value |
|-------|-------|
| **Name** | STRATA |
| **Meaning** | Geological layers — capital structured in confidential strata |
| **Tagline** | *"Deploy. Vest. Trade. Privately."* |
| **Domain** | strata.finance |
| **Token** | $STRT (FHERC20) |
| **Brand** | Bloomberg terminal aesthetic. Institutional. Dark. Premium. |

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| **Encryption** | Fhenix CoFHE (FHE.sol) |
| **Smart Contracts** | Solidity 0.8.25 |
| **Client SDK** | @cofhe/sdk, @cofhe/react |
| **Frontend** | React 18, TypeScript, Vite |
| **Payment Rails** | Privara SDK (stablecoin settlement) |
| **Chain** | Arbitrum Sepolia |
| **Token** | FHERC20 ($STRT) |

---

## Vision

STRATA creates the financial infrastructure Web3 venture capital has been waiting for. Token allocations vest without market surveillance. Capital formation happens without position exposure. Secondary markets operate without price information leakage. Governance executes without coercion risk. Every layer of the capital stack becomes confidential by default — not because data is hidden, but because computation no longer requires exposure.

Deploy. Vest. Trade. Privately.
