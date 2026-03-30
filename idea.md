# STRATA — Idea & Vision Document

---

## Core Thesis

Web3 venture capital operates on public blockchains designed for transparency. This transparency, while valuable for decentralization, is catastrophic for institutional capital management. Every token unlock triggers front-running. Every OTC trade signals insider sentiment. Every syndicate size reveals deal flow strategy. Every governance vote exposes investor alignment.

STRATA eliminates this conflict by moving every capital operation to encrypted state using Fully Homomorphic Encryption. The blockchain still settles, validates, and enforces — but it never sees the numbers.

---

## The Five Modules

STRATA is a modular protocol suite. Each module solves a specific capital management problem. Together, they form a complete operating system for confidential venture operations.

### Module 1: StrataVest — Confidential Token Vesting

**The pain:** When token allocations unlock, on-chain watchers see the vesting contract release tokens to known VC addresses. Automated bots and retail traders interpret this as sell pressure and front-run the price. Investors lose value not because they traded — but because the market saw they COULD trade.

**STRATA's approach:** Founders deposit encrypted allocation amounts mapped to investor addresses. Every allocation is a `euint64` ciphertext handle. The vesting schedule (cliff, duration) is enforced by the contract using `FHE.lte()` to verify claims never exceed the vested portion. When an investor claims, `FHE.sub()` deducts from their encrypted balance and `FHE.select()` ensures the operation is valid — all without revealing the amount. Block explorers show only ciphertext changes. No MEV bot can extract signal from an encrypted claim.

**The investor experience:** Connect wallet → CoFHE permit auto-generates → The dApp decrypts their allocation locally → They see "50,000 $STRT vested" while Etherscan sees `0x8f7a2b...` → They claim an encrypted amount → Their balance updates → The market sees nothing.

---

### Module 2: StrataCall — Confidential Syndicate Pooling

**The pain:** When a top-tier VC leads a $5M round, the syndicate structure reveals everything: who the co-investors are, how much each committed, and how large the total raise was. Competitors reverse-engineer deal flow. Portfolio companies lose negotiation leverage. LPs in the fund lose confidentiality.

**STRATA's approach:** A syndicate lead creates a pool with an encrypted cap (even the target raise is private). LPs commit capital via Privara stablecoin settlement — their individual commitment sizes stored as encrypted `euint64` values. The contract maintains an encrypted running total via `FHE.add()` and enforces the cap via `FHE.lte(total, cap)`. No LP knows what any other LP committed. The lead can view the encrypted total via their permit but cannot see individual commitments.

**Additional features:** Minimum commitment enforcement via `FHE.gte()`, deadline-based pool closing, automatic pro-rata distribution of allocation based on encrypted commitment ratios.

---

### Module 3: StrataOTC — Encrypted Secondary Market

**The pain:** The OTC market for unvested tokens and SAFTs is a $200B+ industry operating almost entirely off-chain through Telegram brokers. The reason: any on-chain trace of a pre-unlock token sale signals "insider selling" and causes immediate market panic. There are zero settlement guarantees in the current Telegram-broker model.

**STRATA's approach:** A seller submits an encrypted ask price and an encrypted token amount. A buyer submits an encrypted bid price. The contract compares the two using `FHE.gte(bid, ask)`. If the bid meets or exceeds the ask, `FHE.select()` conditionally routes encrypted tokens to the buyer and encrypted funds to the seller. The settlement price is determined by `FHE.min(ask, bid)` (seller gets their asking price). If the match fails, both ciphertexts remain untouched — zero information about the price gap leaks. The spot market never panics because the trade is mathematically invisible.

**Key innovation:** Failed order matches reveal NOTHING. In traditional dark pools, a failed match at least reveals that a buyer and seller participated. In STRATA, the encrypted comparison produces an `ebool` result that only the counterparties can decrypt. Even the fact of participation is opaque to external observers.

---

### Module 4: StrataBoard — Confidential LP Governance

**The pain:** When LP governance is transparent, large investors face coercion. If everyone knows that Investor A holds 40% of the fund and votes "no" on releasing the next tranche, Investor A faces political pressure, relationship damage, and potential retaliation. Transparent governance creates oligarchic capture.

**STRATA's approach:** LPs cast encrypted votes weighted by their encrypted investment commitment from StrataCall (Module 2). The contract multiplies vote direction by encrypted weight using `FHE.mul()` and tallies via `FHE.add()`. The final result is decrypted only after the voting period ends, revealing only the aggregate outcome — never individual votes, weights, or directions.

**Selective compliance:** Using ACL-scoped permits, the syndicate lead can grant a time-bounded decryption window to a regulatory auditor. The auditor can verify that the aggregate tally matches the declared outcome, satisfying compliance requirements without doxxing individual LP positions or votes. The permit expires automatically.

---

### Module 5: StrataRisk — AI-Driven Deal Scoring

**The pain:** Portfolio monitoring requires startups to share sensitive financial metrics (burn rate, runway, revenue, user acquisition cost) with their investors. Currently, this data flows through email and spreadsheets, creating security risks and no automated intelligence.

**STRATA's approach:** Portfolio companies submit encrypted financial metrics. An on-chain computation template applies proprietary risk weights via `FHE.mul()`, aggregates via `FHE.add()`, and normalizes via `FHE.div()`. The output is an encrypted "Health Score" that automatically adjusts the company's borrowing power on StrataOTC and influences governance proposals. The risk model's weights are also encrypted — preventing the portfolio company from gaming the score.

**Double-blind scoring:** The startup doesn't know the weight formula. The scoring model doesn't know the raw metrics. Only the final encrypted score is produced, decryptable by the authorized fund.

---

## Why This Protocol Suite Matters

### For Founders
- Distribute SAFT allocations without revealing cap table structure
- Portfolio metrics stay confidential while still enabling automated risk assessment
- No front-running of vesting unlocks preserves token value for all stakeholders

### For Investors
- Claim vested tokens without triggering market-wide sell signals
- Trade unvested positions OTC with on-chain settlement guarantees and zero price leakage
- Vote on governance without coercion or public position exposure
- Commit to syndicates without peers knowing commitment size

### For the Ecosystem
- STRATA brings institutional capital management on-chain without the transparency tax
- The protocol demonstrates that FHE enables an entirely new category of financial infrastructure
- Each module is independently valuable but compositionally powerful

---

## Why Each Module Requires FHE

| Module | Critical FHE Operation | Why No Alternative Works |
|--------|----------------------|------------------------|
| **StrataVest** | `FHE.sub()` on encrypted balances | ZK can't do encrypted arithmetic on balances. Commit-reveal exposes on reveal. |
| **StrataCall** | `FHE.add()` encrypted pool accumulation | MPC requires all LPs online simultaneously. TEE trusts hardware. |
| **StrataOTC** | `FHE.gte()` blind bid-ask matching | ZK requires one party to know the other's value to construct proof. |
| **StrataBoard** | `FHE.mul()` weighted encrypted votes | ZK can't multiply two encrypted values. MPC requires interactive voting. |
| **StrataRisk** | `FHE.mul() + FHE.div()` matrix scoring | No alternative can compute on TWO sets of encrypted inputs (metrics + weights). |

---

## The UX Principle: Make FHE Visible

Every transaction in STRATA displays the "Privacy Console" — a three-pane view showing:

1. **What the blockchain sees:** Raw ciphertext handle (`0x8f7a2b...`)
2. **What you see:** Decrypted value after CoFHE permit (`50,000 $STRT`)
3. **Who has access:** Live ACL status with grant type and expiry (`🟢 Owner | 🟡 Auditor: 23h remaining`)

This is not a debug tool — it is the product's value proposition made visual. Users understand FHE not through documentation but through direct comparison: "the chain sees gibberish, I see my portfolio."

---

## Market Context

- **Token vesting market:** $15B+ in locked token allocations across Web3
- **OTC SAFT market:** $200B+ annual volume (mostly off-chain, no settlement guarantees)
- **VC syndicate market:** $50B+ in crypto venture annual deployment
- **LP governance:** Every fund with >1 LP needs private voting infrastructure

STRATA addresses operational infrastructure for the entire Web3 capital stack — not a single application, but the operating system for how capital forms, distributes, trades, and governs.
