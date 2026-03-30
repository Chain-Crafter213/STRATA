# STRATA — The Confidential Venture & Capital Operating System

"Deploy. Vest. Trade. Privately."

STRATA is the first FHE-powered operating system for Web3 capital management built on Fhenix CoFHE. It provides a unified protocol suite where token vesting, syndicate pooling, OTC secondary trading, LP governance, and portfolio risk scoring all operate entirely on encrypted state. No allocation amount, investor identity, deal size, or vote weight ever appears as plaintext on-chain.

## The Problem

Web3 venture capital is broken by blockchain transparency. When allocations unlock, MEV bots front-run the price before investors act. OTC trades trigger market panic. Syndicate sizes and cap tables stay on Web2 spreadsheets because blockchains expose strategy. Transparent governance creates coercion risk. The result: institutional capital avoids on-chain infrastructure.

## How It Works

STRATA consists of five modules that build progressively:

StrataVest handles confidential token vesting. Founders deposit encrypted SAFT allocations as euint64 ciphertext. Investors claim tokens via FHE.sub() which deducts from their encrypted balance. The contract verifies claims against vesting schedules using FHE.lte() and conditionally executes via FHE.select(). Block explorers see only ciphertext handles. No bot can extract signal from an encrypted claim.

StrataCall manages confidential syndicate pooling. A syndicate lead opens a deal with an encrypted cap. LPs commit capital via Privara stablecoin settlement with individual amounts stored as encrypted euint64 values. The contract tracks an encrypted running total via FHE.add() and enforces the cap via FHE.lte(). No LP knows what any other LP committed.

StrataOTC creates an encrypted secondary market for unvested tokens. Sellers submit encrypted ask prices. Buyers submit encrypted bids. The contract compares them via FHE.gte(). On match, FHE.select() conditionally routes tokens and funds. Failed matches reveal zero information — no price gap, no participation signal.

StrataBoard enables confidential LP governance. Votes are weighted by encrypted investment size from StrataCall using FHE.mul() and tallied via FHE.add(). Only the aggregate outcome is decrypted after the voting period. Individual votes and weights stay permanently encrypted. Time-bounded audit permits allow regulatory compliance without LP exposure.

StrataRisk provides AI-driven deal scoring. Portfolio companies submit encrypted financial metrics. On-chain computation applies encrypted risk weights via FHE.mul(), aggregates via FHE.add(), and normalizes via FHE.div(). The output is an encrypted Health Score. Neither the startup's metrics nor the scoring weights are exposed.

## Architecture

Five smart contracts form the protocol suite atop a shared StrataCore.sol foundation:

StrataCore.sol provides role-based ACL (FOUNDER, INVESTOR, AUDITOR, ADMIN), shared FHE helpers, and fee management.

StrataVest.sol stores encrypted allocations per investor with vesting schedules. Core operations: FHE.asEuint64 for input conversion, FHE.add for allocation accumulation, FHE.sub for claim deduction, FHE.lte for vesting verification, FHE.select for conditional execution, FHE.sealoutput for permit-based viewing.

StrataCall.sol manages syndicate creation with encrypted caps, LP commitment deposits, pool total tracking, and Privara stablecoin integration.

StrataOTC.sol implements encrypted order book with bid-ask matching via FHE.gte, conditional settlement via FHE.select, and price determination via FHE.min.

StrataBoard.sol handles proposal creation, encrypted weighted voting via FHE.mul, tally accumulation via FHE.add, outcome finalization via FHE.decrypt, and scoped audit permit issuance.

The protocol uses 16 distinct FHE operations across its full suite: asEuint64, add, sub, mul, div, gte, lte, gt, min, select, allow, allowThis, sealoutput, isInitialized, decrypt, and asEbool.

## Five-Wave Roadmap

Wave 1 delivers the complete protocol architecture, StrataVest specification, data flow documentation, access control model, threat analysis, FHE operations matrix, and technical whitepaper covering the full five-module vision.

Wave 2 extends to StrataCall for confidential syndicate pooling with Privara stablecoin integration and encrypted cap enforcement.

Wave 3 builds StrataOTC for encrypted secondary market trading with blind bid-ask matching and zero-information failed matches.

Wave 4 implements StrataBoard for confidential LP governance with investment-weighted encrypted voting and time-bounded audit access.

Wave 5 completes StrataRisk for AI-driven encrypted deal scoring with double-blind risk assessment where neither metrics nor weights are exposed.

## Why FHE Is Required

Every module depends on FHE operations that no alternative achieves. Zero-knowledge proofs cannot perform arithmetic on encrypted balances or match two hidden prices. Multi-party computation requires all LPs online simultaneously for every operation. Trusted execution environments decrypt data inside hardware vulnerable to side-channel attacks. Commit-reveal schemes expose data on reveal. Only FHE computes on encrypted values without decryption, enabling blind matching, encrypted accumulation, and weighted computation on permanent ciphertext state.

## Vision

STRATA brings institutional capital management fully on-chain without the transparency tax. Token allocations vest invisibly. Capital forms confidentially. Secondary trading settles without market signal. Governance decides without coercion. Every layer of the venture capital stack becomes confidential by default.

Deploy. Vest. Trade. Privately.
