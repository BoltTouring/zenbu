# Zenbu 全部

**One identifier. Three payment rails. Zero friction.**

Zenbu is a unified Bitcoin payment identity built on Nostr. You give someone a single email-style address — they can pay you via Lightning, on-chain (Silent Payments), or Cashu e-cash. No invoices, no interaction, no running a web server.

```
  you@yourdomain.com
       │
       │  NIP-05 resolution
       ▼
  npub1...
       │
       │  Read kind 0 profile + kind 10019
       ▼
  ┌──────────┬──────────┬──────────┐
  │ lud16    │ sp       │ kind     │
  │          │(proposed)│ 10019    │
  ├──────────┼──────────┼──────────┤
  │Lightning │ On-Chain │ E-Cash   │
  │  Zap     │ Silent   │ Nutzap   │
  │ (NWC)    │ Payment  │(NIP-61)  │
  └──────────┴──────────┴──────────┘
```

Nostr is the sole source of identity and payment discovery. No DNS payment records, no LNURL callbacks, no parallel resolution paths.

## The Problem

Bitcoin has three practical payment rails today — Lightning, on-chain, and e-cash — but no standard way to advertise all three from a single identity. Lightning has `lud16`. Nutzaps have `kind 10019`. On-chain silent payments have... nothing. You're stuck pasting an `sp1...` address in your bio and hoping senders find it.

## The Proposal: A Nostr `sp` Profile Field

The missing piece is a standardized `sp` field in Nostr kind 0 profile metadata, mirroring `lud16`:

```json
{
  "name": "satoshi",
  "nip05": "satoshi@yourdomain.com",
  "lud16": "satoshi@getalby.com",
  "sp": "sp1qq2cy2s2..."
}
```

One new optional field. No new event kinds. No new discovery NIPs. Clients that don't understand it simply ignore it. Clients that do can offer a "Pay on-chain" button alongside the Lightning zap — the payer's wallet derives a unique Taproot output from the `sp` address, and the receiver doesn't need to be online.

### What needs to happen

1. Draft a short NIP (or amendment to NIP-01) specifying the `sp` field, format, and expected client behavior
2. Get 1–2 Nostr client teams to implement (Damus, Amethyst, Primal)
3. Get 1–2 wallet libraries (BDK, LDK) to support SP address derivation for easy client integration

## How It Works

A sender receives your identifier `you@yourdomain.com`:

| Sender's context | Resolution path | Payment rail |
|---|---|---|
| Any Nostr client | NIP-05 → npub → `lud16` | Lightning zap |
| NIP-61 client (Olas, Iris, Chachi) | NIP-05 → npub → `kind 10019` | Cashu nutzap |
| Client with `sp` support (future) | NIP-05 → npub → `sp` field | Silent Payment |
| SP wallet (manual, works today) | Copy `sp1...` from bio | Silent Payment |

## Setup

### Prerequisites

| Requirement | Purpose | Options |
|---|---|---|
| Domain you control | Host NIP-05 verification file | Any registrar |
| Nostr keypair | Your identity | Any Nostr client or nsec.app |
| Lightning wallet backend | Receive Lightning payments | [Alby Hub](https://albyhub.com) (self-custodial) |
| Silent Payments wallet | Receive on-chain privately | See [wallet table](#silent-payments-wallets) below |
| Cashu wallet | Receive e-cash nutzaps | [Cashu.me](https://cashu.me), Minibits, or [Cashu Cache](https://nostrly.com/cashu-cache) |

### 1. NIP-05 Identity

Host a JSON file at `https://yourdomain.com/.well-known/nostr.json`:

```json
{
  "names": {
    "you": "YOUR_HEX_PUBKEY_HERE"
  }
}
```

Set the CORS header:

```
Access-Control-Allow-Origin: *
```

Nginx example:

```nginx
location /.well-known/nostr.json {
    add_header Access-Control-Allow-Origin "*";
    add_header Content-Type "application/json";
}
```

Then set `you@yourdomain.com` as your NIP-05 identifier in your Nostr profile.

> **Source:** [NIP-05](https://github.com/nostr-protocol/nips/blob/master/05.md)

### 2. Lightning (lud16 + Alby Hub)

1. Set up [Alby Hub](https://albyhub.com) — self-custodial Lightning node with [NWC](https://nwc.dev) built in
2. Get your Lightning address (e.g., `you@getalby.com` or `you@yourdomain.com` with custom domain)
3. Open channels or connect to Alby's LSP for inbound liquidity
4. Set your `lud16` field in your Nostr profile to your Lightning address

Every major Nostr client reads `lud16` and shows a zap button. Alby Hub auto-generates invoices via NWC when someone zaps you.

> **Sources:** [NIP-47](https://github.com/nostr-protocol/nips/blob/master/47.md), [nwc.dev](https://nwc.dev), [albyhub.com](https://albyhub.com)

### 3. On-Chain (Silent Payments + sp Field)

[Silent Payments (BIP 352)](https://silentpayments.xyz) let you publish a single static `sp1...` address that generates a unique Taproot output for every sender. No address reuse, no interaction required.

1. Install a wallet that supports SP receiving (see table below)
2. Generate your `sp1...` address from the Receive screen
3. Add the `sp` field to your kind 0 profile metadata:

```json
{
  "lud16": "you@getalby.com",
  "sp": "sp1qq2cy2s2...YOUR_FULL_SP_ADDRESS..."
}
```

Most clients don't expose arbitrary metadata fields yet — you may need to publish the event directly using `nak`, `nostr-tool`, or a script. Until the `sp` field is widely adopted, also paste your `sp1...` address in your profile bio as a fallback.

> **Sources:** [BIP 352](https://bips.dev/352), [silentpayments.xyz](https://silentpayments.xyz)

#### Silent Payments Wallets

| Wallet | GitHub | Sending | Receiving | Privacy-Preserving Scanning |
|---|---|:---:|:---:|:---:|
| BitBox | [BitBoxSwiss/bitbox-wallet-app](https://github.com/BitBoxSwiss/bitbox-wallet-app) | ✅ | ❌ | ❌ |
| BlindBit Desktop | [setavenger/blindbit-desktop](https://github.com/setavenger/blindbit-desktop) | ✅ | ✅ | ✅ |
| BlueWallet | [bluewallet/bluewallet](https://github.com/bluewallet/bluewallet) | ✅ | ❌ | ❌ |
| Cake Wallet | [cake-tech/cake_wallet](https://github.com/cake-tech/cake_wallet) | ✅ | ✅ | ✅ |
| Dana Wallet | [cygnet3/dana](https://github.com/cygnet3/dana) | ✅ | ✅ | ✅ |
| Nunchuk Wallet | [nunchuk-io](https://github.com/nunchuk-io) | ✅ | ❌ | ❌ |
| Shakesco Wallet | [shakesco/silent](https://github.com/shakesco/silent) | ✅ | ✅ | ✅ |
| Sparrow Wallet | [sparrowwallet/sparrow](https://github.com/sparrowwallet/sparrow) | ✅ | ❌ | ❌ |
| Wasabi Wallet | [WalletWasabi/WalletWasabi](https://github.com/WalletWasabi/WalletWasabi) | ✅ | ❌ | ❌ |

**Hardware:** BitBox02 supports send + receive.

> **Source:** [silentpayments.xyz/docs/wallets](https://silentpayments.xyz/docs/wallets/)

> ⚠️ **All SP wallets are still experimental.** Back up your seed phrase and test with small amounts first.

### 4. E-Cash (NIP-60/61 Nutzaps)

Nutzaps let someone send you Cashu e-cash tokens via Nostr. Tokens accumulate in encrypted events on your relays — you claim them when you open your wallet.

1. Go to [Cashu Cache](https://nostrly.com/cashu-cache) and sign in with your Nostr key (via NIP-07 extension)
2. Select 1–2 NUT-11 compliant mints (e.g., `stablenut.umint.cash`, `mint.minibits.cash`)
3. Select your relay list (2–4 relays)
4. Click Create — this publishes:
   - **kind 17375** — Your NIP-60 wallet event (encrypted)
   - **kind 10019** — Nutzap discovery event (public — tells senders your mints, relays, and nutzap pubkey)
   - **kind 375** — Wallet backup event
5. Back up the wallet private key. **This is separate from your Nostr nsec.**

Alternative wallet apps: [Cashu.me](https://cashu.me) (web), [Minibits](https://www.minibits.cash/) (mobile), [Nutstash](https://nutstash.app/) (web).

> **Sources:** [NIP-60](https://github.com/nostr-protocol/nips/blob/master/60.md), [NIP-61](https://github.com/nostr-protocol/nips/blob/master/61.md), [cashu.space](https://cashu.space)

### 5. Relay List (NIP-65)

Publish a NIP-65 relay list so clients know where to find your `kind 10019` and other events. Most Nostr clients handle this automatically from your relay settings.

> **Source:** [NIP-65](https://github.com/nostr-protocol/nips/blob/master/65.md)

## Optional: BIP 353 (DNS Payment Instructions)

If you also want to reach senders using plain Bitcoin wallets that don't speak Nostr, you can add a BIP 353 DNS record as a complementary discovery path. This is not part of the core architecture.

1. Enable DNSSEC on your domain
2. Add a DNS TXT record:
   - **Name:** `you.user._bitcoin-payment.yourdomain.com`
   - **Value:** `bitcoin:?sp=sp1qq...&lno=lno1qq...`
3. Verify at [satsto.me](https://satsto.me)

BIP 353 and Nostr resolution don't conflict — the same email address works in both contexts.

> **Sources:** [BIP 353](https://bips.dev/353), [bolt12.org](https://bolt12.org), [twelve.cash](https://twelve.cash) (hosted usernames)

## Current Limitations

- **`sp` field doesn't exist yet** — this is the core proposal that needs developer buy-in. Until adopted, on-chain SP discovery requires sharing your address via profile bio.
- **NIP-61 adoption is small** — nutzaps work, but most Nostr users are still on NIP-57 Lightning zaps.
- **SP scanning is expensive on mobile** — wallets scan every Taproot output per block. Expect sync delays.
- **BOLT 12 on LND is incomplete** — CLN (Core Lightning) has full support.
- **No multi-rail auto-selection** — no single client auto-selects between Lightning, ecash, and on-chain yet.

## Roadmap

- **`sp` field NIP** — draft and submit for community review
- **BDK BIP 353** — `bitcoin-payment-instructions` crate from Matt Corallo for wallet integration
- **SP wallet maturity** — send+receive support expanding rapidly
- **Kind 10009 (Wallet Info)** — proposed event kind for a unified payment business card
- **NUT-13 deterministic wallets** — seed-phrase recovery for Cashu, making NIP-60 safer

## Standards Reference

| Standard | Purpose | Link |
|---|---|---|
| NIP-05 | Email-style identity → npub | [spec](https://github.com/nostr-protocol/nips/blob/master/05.md) |
| NIP-47 | Nostr Wallet Connect (NWC) | [spec](https://github.com/nostr-protocol/nips/blob/master/47.md) / [nwc.dev](https://nwc.dev) |
| NIP-60 | Cashu wallet on Nostr | [spec](https://github.com/nostr-protocol/nips/blob/master/60.md) |
| NIP-61 | Nutzaps (ecash via Nostr) | [spec](https://github.com/nostr-protocol/nips/blob/master/61.md) |
| NIP-65 | Relay list metadata | [spec](https://github.com/nostr-protocol/nips/blob/master/65.md) |
| BIP 352 | Silent Payments | [spec](https://bips.dev/352) / [silentpayments.xyz](https://silentpayments.xyz) |
| BIP 353 | DNS Payment Instructions | [spec](https://bips.dev/353) |
| BOLT 12 | Lightning Offers | [bolt12.org](https://bolt12.org) |

## License

MIT
