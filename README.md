# 🛡️ Echo Markets Charms App Security Audit

**Target:** `lib.rs` (Charms Protocol App)
Echo Markets is a decentralized prediction market (dual-token system) using the [Charms Protocol](https://charms.dev)

---

## 1. 🌟 Security Highlights

### A. Strict UTXO Bitcoin Accounting

The contract no relies purely on abstract token math, it inspects the physical transaction inputs and outputs (`tx.coin_ins`, `tx.coin_outs`) to ensure the Market NFT's Satoshi balance perfectly matches the token supply.

- **Exploit Prevented:** "Free Minting" and "Vault Draining."

### B. Bitcoin Dust Limit Protection

The contract enforces a hardcoded `min_bet` of `10,000` Satoshis.

- **Exploit Prevented:** Malicious actors cannot create "dust" markets or force the contract to generate uneconomical UTXOs (e.g., 1-satoshi payouts) that would cost more in miner fees to spend than they are worth.

### C. Emergency Resolution (Trustless Refunds)

If the trusted Oracle/Reclaim protocol goes offline or the creator loses their keys, user funds are not locked forever. The contract implements a `7-day (604,800 seconds)` Emergency Grace Period.

- **Exploit Prevented:** Permanent fund lockup. After 7 days past the resolution deadline, _anyone_ can bypass the signature check to resolve the market as `Invalid`, instantly unlocking 100% of user collateral for refunds.

### D. Parameter Sanity Bounds

When a market is created, the parameters are strictly bounded:

- `fee_bps <= 5000` (Max 50% fee).
- `resolution_deadline > trading_deadline`.
- **Exploit Prevented:** Creators cannot set 100%+ fees causing mathematical underflows, nor can they create chronologically impossible markets.

---

## 2. 🔍 Operation Validation Checks

Every transaction routed through `app_contract` undergoes strict conditional checks depending on the operation.

### `Create` (Market Initialization)

- **Check:** Exactly 1 input UTXO is used to fund the market.
- **Check:** At least 1 output exists (the Market NFT).
- **Check:** `min_bet >= 10000` and `fee_bps <= 5000`.
- **Check:** Timestamps are logical (`trading_deadline > 0` and `resolution_deadline > trading_deadline`).
- **Check (CRITICAL):** The market MUST start with `yes_supply == 0`, `no_supply == 0`, and `fees == 0` (Prevents pre-minting supply).
- **Check:** Status begins as `Active`.

### `Mint` (Opening a Position)

- **Check:** Market status is `Active`.
- **Check:** `current_timestamp < trading_deadline` (No past-posting).
- **Check:** `collateral_amount >= min_bet`.
- **Check (CRITICAL):** Native BTC locked. `new_sats == old_sats + collateral_amount`.
- **Check:** Equal amounts of YES and NO tokens are minted.
- **Check:** `new.yes_supply <= max_supply` (Prevents integer overflow).

### `Burn` (Closing a Position Early)

- **Check:** Market status is `Active`.
- **Check:** `current_timestamp < trading_deadline`.
- **Check:** Exactly equal amounts of YES and NO tokens are destroyed.
- **Check (CRITICAL):** Native BTC returned. `new_sats == old_sats - set_count`.

### `Resolve` (Ending the Market) _iteration to Reclaim Protocol_

- **Check:** Market status is `Active` or `TradingClosed`.
- **Check:** `current_timestamp >= resolution_deadline`.
- **Check:** Cryptographic proof is verified via Schnorr signature (`verify_resolution_signature`), **OR** the 7-day emergency timeout has passed and the outcome is being set to `Invalid`.
- **Check:** Status transitions to `Resolved`.

### `Redeem` (Claiming Winnings/Refunds)

- **Check:** Market status is `Resolved`.
- **Check:** If `Outcome::Yes`, only YES tokens are burned (`no_amount == 0`).
- **Check:** If `Outcome::No`, only NO tokens are burned (`yes_amount == 0`).
- **Check:** If `Outcome::Invalid`, any combination of YES/NO tokens can be burned (Refunds 0.5 sats per token).
- **Check (CRITICAL):** Native BTC withdrawn. `new_sats == old_sats - payout_sats`.

### `Cancel` (Creator Abort)

- **Check:** Status is NOT `Resolved`.
- **Check:** Cryptographic Schnorr signature of the `creator` matches the message `SHA256("CANCEL" || market_id)`.
- **Check:** Status transitions to `Cancelled` and resolution becomes `Invalid`.

### `ClaimFees` (Creator Payday)

- **Check:** Market status is `Resolved`.
- **Check:** Cryptographic Schnorr signature of the `creator` is verified.
- **Check:** `old.fees > 0` (Prevents empty claims).
- **Check:** `new.fees == 0` (Prevents double-claiming).
- **Check (CRITICAL):** Native BTC withdrawn. `new_sats == old_sats - old.fees`.

### `Token Transfer` (P2P Trading) _iteration to Cast DEX_

- **Check:** Handled under the `TOKEN` app tag.
- **Check:** Market status MUST be `Active` or `TradingClosed` (Tokens become soulbound/untransferable once `Resolved`).
- **Check:** Token Conservation enforced (`input_total == output_total`) to prevent unauthorized P2P minting/burning.

---

## 4. Known Limitations & Trust Assumptions

1. **Timestamp Oracles:** As noted in the documentation, the contract relies on the outer Scrolls layer sequencer to provide an honest `current_timestamp` to prevent time-based exploits.
2. **Oracle Trust:** The Cardano cross-chain logic trusts a single Oracle signature rather than verifying a full Light Client Merkle proof (Iteration to Reclaim Protocol).
