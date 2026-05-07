# Bitcoin Physical Bearer Instrument

A toolkit for minting, verifying, and sweeping Bitcoin physical bearer coins with CLTV timelocks enforced by Bitcoin consensus.

**Lock target:** Block 1,951,500 · ~22 May 2046 · Bitcoin Pizza Day 36th Anniversary

---

## Overview

This project implements a physical Bitcoin bearer instrument where funds are locked by `OP_CHECKLOCKTIMEVERIFY` (BIP-65) until a fixed block height. The private key is stored in plaintext on an NTAG424 DNA NFC chip embedded in the physical coin — this is intentional. The timelock makes the key irrelevant until 2046. Security derives entirely from Bitcoin consensus, not from key secrecy.

The toolkit is four static HTML files with no server, no framework, and no runtime dependencies beyond embedded JavaScript bundles. Everything runs client-side.

---

## Architecture

### Script Construction

Each coin is a P2WSH output locking to the following redeem script:

```
<locktime> OP_CHECKLOCKTIMEVERIFY OP_DROP <pubkey> OP_CHECKSIG
```

Encoded as:

```
[len(locktime)] [locktime bytes LE] 0xb1 0x75 0x21 [33-byte compressed pubkey] 0xac
```

The P2WSH address is `bech32(OP_0 SHA256(redeemScript))`. The Miniscript descriptor is:

```
wsh(and_v(v:pk(<pubkey>),after(<lockblock>)))
```

Locktime is encoded as a minimal-push little-endian Script number. If the high bit of the last byte is set, a zero byte is appended to prevent sign-bit ambiguity.

### On-Chain Identity

Each coin broadcasts an OP_RETURN transaction at mint time as an immutable birth certificate:

```
OP_RETURN <COIN_ID>:<ADDRESS>
```

Where `COIN_ID` is a 14-character hex string (7 bytes). In V1 this is `FF` + 6 random bytes. In V2 it will be the NTAG424's factory-burned 7-byte UID (`04` prefix, ISO 14443-3).

The OP_RETURN is paid by a throwaway P2WPKH fee wallet, not the coin address — keeping the coin address's transaction history clean.

### Spending

The witness stack for spending is:

```
<sig> <redeemScript>
```

The transaction must set `nLocktime >= lockblock` and `nSequence < 0xffffffff` (RBF-compatible). The PSBT finalizer is custom — bitcoinjs-lib's auto-finalizer cannot handle arbitrary scripts.

All inputs are signed with `SIGHASH_ALL`. RBF is signalled via `nSequence = 0xfffffffe`.

---

## Files

| File | Purpose |
|------|---------|
| `index.html` | Landing page with expandable user guides |
| `mint.html` | Coin generation, OP_RETURN broadcast, NFC payload export |
| `verify.html` | Address lookup, balance, lock status, birth TX |
| `sweep.html` | Pre-signed sweep with RBF fee escalation |

All four files are fully self-contained. No CDN, no external requests except Mutinynet API calls at runtime.

---

## Cryptographic Stack

### Key Generation

Private keys are generated via `window.crypto.getRandomValues()` (CSPRNG, all modern browsers).

Public key derivation uses **noble/secp256k1 v1.7.1** (pure JavaScript, no WASM). The library is bundled via Browserify and embedded directly in `mint.html` and `sweep.html`.

```javascript
// Key derivation path
const privBytes  = crypto.getRandomValues(new Uint8Array(32));
const keyPair    = BitcoinLib.fromPrivateKey(privBytes, network);
const pubkey     = keyPair.publicKey; // 33-byte compressed secp256k1 point
```

**Important:** All key operations use `BitcoinLib.fromPrivateKey` (noble/secp256k1) for consistency. An earlier version used a custom secp256k1 implementation that produced different public keys from the same private key bytes. The two implementations must never be mixed — the redeem script encodes the public key, so the signing key must be derived from the same implementation that built the script.

### WIF Encoding

WIF is encoded manually using SHA256 (pure JS implementation) for the checksum, and Base58 encoded without external libraries:

```
[version_byte][privkey_32][0x01(compressed)][SHA256(SHA256(payload))[0:4]]
```

Version byte: `0xef` for testnet/signet, `0x80` for mainnet.

A round-trip verification is performed at mint time — the WIF is immediately decoded with `BitcoinLib.wifToBytes` and compared against the original private key bytes. If they differ, generation is aborted.

### Address Derivation

P2WSH address derivation:

```javascript
SHA256(redeemScript)       → 32-byte witness program
bech32(OP_0 || program)    → address string (tb1q... or bc1q...)
```

SHA256 is a pure JS implementation embedded in the page. Bech32 encoding uses the bitcoinjs-lib implementation from the embedded bundle. Address derivation is cross-verified against `bitcoin.payments.p2wsh()` — if the two methods produce different addresses, generation throws.

### Transaction Signing (OP_RETURN)

The fee wallet is a standard P2WPKH address. Signing uses `bitcoin.Psbt` from bitcoinjs-lib with the noble/secp256k1 signer:

```javascript
const psbt = new bitcoin.Psbt({ network });
psbt.addInput({
  hash: utxoTxid, index: utxoVout,
  sequence: 0xfffffffe,           // RBF
  witnessUtxo: { script: p2wpkh.output, value: utxoValue }
});
psbt.addOutput({ script: OP_RETURN_script, value: 0 });
psbt.signInput(0, keyPair);
psbt.finalizeAllInputs();
```

The BIP143 sighash is computed internally by bitcoinjs-lib. The scriptCode for P2WPKH signing is `OP_DUP OP_HASH160 <hash160(pubkey)> OP_EQUALVERIFY OP_CHECKSIG` as required by BIP143 §P2WPKH.

### Transaction Signing (Sweep)

The CLTV P2WSH input cannot be finalized by bitcoinjs-lib's auto-finalizer. A custom witness is constructed manually:

```javascript
psbt.signInput(i, keyPair);

// Manual witness construction
const sig      = psbt.data.inputs[i].partialSig[0].signature;
const witness  = [sig, redeemScript];
const encoded  = encodeWitness(witness); // varint-prefixed stack items
psbt.data.inputs[i].finalScriptWitness = encoded;
delete psbt.data.inputs[i].partialSig;
delete psbt.data.inputs[i].witnessScript;
```

The PSBT must have `nLocktime` set to the lock block height, and all inputs must use `nSequence = 0xfffffffe`.

---

## NFC Tag (NTAG424 DNA)

### V1 — Manual Write

The NFC payload is a JSON object written manually to the tag using NFC Tools Pro on Android. The JSON contains:

```json
{
  "v": "1.0",
  "coin_id": "FF3A7C2E9B1D44",
  "serial": "000001",
  "network": "mainnet",
  "address": "bc1q...",
  "privkey_wif": "...",
  "pubkey": "02...",
  "descriptor": "wsh(and_v(v:pk(...),after(1951500)))",
  "redeem_script": "...",
  "lock_block": 1951500,
  "lock_date_approx": "2046-05-22",
  "op_return_txid": "...",
  "op_return_block": 897442,
  "generated_ts": "2026-05-07T..."
}
```

### V2 — Automated Minting Station

The NTAG424 has a factory-burned 7-byte UID (`04` prefix, NXP manufacturer byte). This UID becomes the coin ID directly, creating a hardware-anchored identity.

The minting station reads the UID via Web NFC API (`NDEFReader`) on Chrome/Android, uses it as the coin ID, generates the wallet, broadcasts the OP_RETURN, and writes the full payload back to the tag in a single flow.

NTAG424 Secure Dynamic Messaging (SDM) enables per-tap unique URLs:

```
verify.html?uid=<encrypted_uid>&ctr=<counter>&cmac=<signature>
```

The 3-byte scan counter increments permanently on-chip, cannot be reset, and forms a provenance record of the coin's circulation history. The CMAC (AES-128) proves the counter value is authentic.

Tag specifications: 7-byte UID, AES-128 encryption, Common Criteria EAL4, 50-year data retention.

---

## Network

### Production: Mainnet

Lock block 1,951,500. At ~144 blocks/day, approximately 22 May 2046.

### Testing: Mutinynet

Mutinynet is a custom signet operated by the Mutiny Wallet team with 30-second block times. API: `https://mutinynet.com/api` (esplora-compatible). Faucet: `https://faucet.mutinynet.com`.

Mutinynet shares the `tb1` bech32 prefix with standard testnet but is a completely separate chain. WIF version byte `0xef`.

Test Mode in the mint page fetches current block height and sets `lockBlock = currentHeight + N`, allowing the full mint → lock → sweep cycle to be tested in minutes.

---

## Sweep Tool — RBF Race Mechanism

### Pre-signing

The sweep transaction is built and signed before the critical block. The signed hex is stored in memory. At lock expiry, it is broadcast immediately without any further user interaction.

### Fee Escalation

If a competing transaction spending the same UTXO is detected in the mempool, the sweep tool:

1. Rebuilds the transaction with a higher fee rate
2. Re-signs with the same key
3. Broadcasts the replacement

This uses Replace-by-Fee (BIP-125). All inputs signal RBF via `nSequence = 0xfffffffe`. The fee rate escalates by a configurable multiplier (1.5x, 2x, or 3x) up to a configured maximum. The mempool is polled every 5 seconds.

### Game Theory

The coin is fully transparent — address, private key, and redeem script are on the NFC tag, readable by anyone. Multiple holders over 21 years may copy the key. At block 1,951,500, all prior key holders can attempt to spend.

The structural advantage belongs to the holder in 2046 who prepares the sweep tool in advance. They can pre-sign the transaction, monitor the mempool from the first eligible block, and escalate fees faster than an unprepared claimant. Preparation is the only edge.

---

## Security Model

| Threat | Before 2046 | After 2046 |
|--------|------------|------------|
| Key theft | Irrelevant — CLTV rejects spend | Standard key security applies |
| Physical coin theft | Thief holds an unspendable object | Race condition — preparation wins |
| Key extraction from NFC | Key is public by design | Same as key theft |
| 51% attack | Theoretical — rewrite chain past locktime | Standard Bitcoin security |
| Cloned NFC tag | Different UID — detectable in V2 | — |

The security guarantee is: **no party can spend the locked UTXO before block 1,951,500, regardless of key possession.** This is enforced by Bitcoin consensus rules, not by any software, service, or physical mechanism.

---

## Running Locally

The pages must be served over HTTP — `file://` origin restrictions prevent the Fetch API calls to the Mutinynet API from working.

```bash
# Python
python -m http.server 8080

# Node.js
npx serve .
```

Then open `http://localhost:8080`.

---

## Dependencies (Embedded)

All dependencies are bundled at build time and embedded directly in the HTML files. No network requests are needed to load the application.

| Library | Version | Purpose |
|---------|---------|---------|
| bitcoinjs-lib | 5.2.0 | PSBT, payments, script compilation |
| noble/secp256k1 | 1.7.1 | secp256k1 key derivation and signing |
| noble/hashes | 1.3.3 | HMAC-SHA256 for RFC6979 deterministic nonce |
| buffer | — | Node.js Buffer polyfill for browser |
| qrcode.js | 1.5.4 | QR code generation (canvas) |

Bundled via Browserify. SHA256, RIPEMD160, and Bech32 also have pure JS implementations embedded for address derivation cross-verification.

---

## BIP References

| BIP | Title | Usage |
|-----|-------|-------|
| BIP-65 | OP_CHECKLOCKTIMEVERIFY | CLTV timelock enforcement |
| BIP-125 | Opt-in Full Replace-by-Fee | RBF sweep escalation |
| BIP-141 | Segregated Witness | P2WSH output type |
| BIP-143 | Transaction Signature Hashing for SegWit | Sighash computation |
| BIP-173 | Bech32 Address Format | Address encoding |
| BIP-379 | Miniscript | Descriptor format |

---

## Coin ID Format

```
V1 (software):  FF [6 random bytes]  — 14 hex chars, FF prefix flags software generation
V2 (hardware):  04 [6 NXP bytes]     — 14 hex chars, 04 prefix is NXP manufacturer byte
```

The two formats are structurally identical and the schema is forward-compatible. V2 minting is a drop-in replacement — only the coin ID generation step changes.

---

## Known Limitations

- V1 NFC write is manual (NFC Tools Pro on Android). V2 automates this via Web NFC API (Chrome/Android only — iOS does not support Web NFC write).
- The sweep tool requires the page to remain open in an active browser tab until confirmation. No background worker or server-side monitoring.
- Lock block 1,951,500 is an approximation. At 10-minute average block times the unlock will occur within days of 22 May 2046 in either direction. Variance accumulates over 20 years.
- The OP_RETURN payload links the coin ID to the address publicly. Privacy is a non-goal — full transparency is a design principle.
