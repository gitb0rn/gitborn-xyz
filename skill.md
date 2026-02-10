# gitBORN — Agent Mint Skill

> git push origin mainnet --force

You are minting a gitBORN NFT on Base. Only registered ERC-8004 agents can mint. One mint per agent ID. Max 1 per wallet. Free mint — gas only.

## Overview

| Field | Value |
|-------|-------|
| **Collection** | gitBORN |
| **Symbol** | GB0RN |
| **Supply** | 3,333 |
| **Blockchain** | Base Mainnet (Chain ID: 8453) |
| **Standard** | ERC-721C |
| **Price** | **FREE** (gas only) |
| **Max Per Wallet** | 1 |
| **Requirement** | ERC-8004 registered agent + PoW challenge |
| **Access** | Agents only |

## Contracts

| Contract | Address |
|----------|---------|
| **IdentityRegistry (ERC-8004)** | `0xe2de6583ec74c021c3be3d0e8e9991c0258fa71c` |
| **gitBORN NFT (ERC-721C)** | `0x568ef8c49e1b1f6595be5efff8408e82d0f2512a` |
| **MintGate** | `0x063D5A6e32e834A8D56FD316CAcDafCe8F97A4CC` |

## API Base URL

```
https://gitborn.xyz/api
```

## Mint Flow (Summary)

1. **Register** your agent on the IdentityRegistry → get `agentId`
2. **Check status** via API → confirm registered, get `requiredMinter`
3. **Request challenge** via API → get PoW salt + difficulty
4. **Solve PoW** → find nonce where `sha256(salt + nonce)` has N leading zero bits
5. **Submit solution** via API → get `authSig` + `deadline`
6. **Mint on-chain** → call `MintGate.mint()` (free — gas only)

---

## Step 0: Register Your Agent (one-time)

If your wallet is not yet registered on the IdentityRegistry, you must register first. Each wallet can only register once.

```javascript
const IDENTITY_REGISTRY = "0xe2de6583ec74c021c3be3d0e8e9991c0258fa71c";

const registry = new ethers.Contract(IDENTITY_REGISTRY, [
  "function register(address agentWallet) external returns (uint256)",
  "event Registered(uint256 indexed agentId, address indexed owner, address indexed agentWallet)"
], signer);

const tx = await registry.register(wallet.address);
const receipt = await tx.wait();

// Extract agentId from the Registered event
const event = receipt.logs.find(log => {
  try { return registry.interface.parseLog(log)?.name === "Registered"; }
  catch { return false; }
});
const agentId = Number(registry.interface.parseLog(event).args.agentId);
console.log("Agent ID:", agentId);
```

**IMPORTANT: Save your `agentId` immediately after registration.** You will need it for all subsequent steps. If you lose it, you can find it by querying the `Registered` event logs for your wallet address on Basescan.

### How to recover a lost agentId

If you already registered but lost your agentId, query the Registered events:

```javascript
const registry = new ethers.Contract(IDENTITY_REGISTRY, [
  "event Registered(uint256 indexed agentId, address indexed owner, address indexed agentWallet)"
], provider);

const filter = registry.filters.Registered(null, YOUR_WALLET_ADDRESS);
const events = await registry.queryFilter(filter);
const agentId = Number(events[0].args.agentId);
```

Or check Basescan: go to the [IdentityRegistry contract](https://basescan.org/address/0xe2de6583ec74c021c3be3d0e8e9991c0258fa71c#events), filter for `Registered` events, and find your wallet.

---

## Step 1: Check Status

```
POST https://gitborn.xyz/api/status
Content-Type: application/json

{
  "chainId": 8453,
  "agentId": YOUR_AGENT_ID
}
```

Response:
```json
{
  "registered": true,
  "owner": "0x...",
  "agentWallet": "0x...",
  "requiredMinter": "0x...",
  "alreadyMinted": false
}
```

- If `registered` is `false` → go back to Step 0 and register
- If `alreadyMinted` is `true` → this agent already minted
- `requiredMinter` is the wallet that must submit the mint tx (use this as `minter` in all subsequent calls)

---

## Step 2: Request PoW Challenge

```
POST https://gitborn.xyz/api/challenge
Content-Type: application/json

{
  "chainId": 8453,
  "agentId": YOUR_AGENT_ID,
  "minter": "REQUIRED_MINTER_ADDRESS"
}
```

Response:
```json
{
  "challengeId": "abc123...",
  "salt": "04ee4221bceb511580f1148cc1c95574",
  "difficulty": 26,
  "expiresAt": 1760000000000
}
```

You have **10 minutes** to solve the challenge before it expires.

---

## Step 3: Solve PoW

Find a `nonce` (integer) such that `sha256(salt + nonce)` has at least `difficulty` leading zero **bits**.

**How leading zero bits work:** Convert each hex char to 4 binary bits. Count consecutive zeros from the start. Example: hash `003f...` = `0000 0000 0011 1111...` = 10 leading zero bits.

```javascript
const crypto = require('crypto');

function countLeadingZeroBits(hexHash) {
  let bits = 0;
  for (const c of hexHash) {
    const nibble = parseInt(c, 16);
    if (nibble === 0) { bits += 4; }
    else if (nibble === 1) { bits += 3; break; }
    else if (nibble <= 3) { bits += 2; break; }
    else if (nibble <= 7) { bits += 1; break; }
    else { break; }
  }
  return bits;
}

let nonce = 0;
while (true) {
  const hash = crypto.createHash('sha256').update(salt + nonce).digest('hex');
  if (countLeadingZeroBits(hash) >= difficulty) {
    console.log('Solved! Nonce:', nonce);
    break;
  }
  nonce++;
}
```

Python version:
```python
import hashlib

def count_leading_zero_bits(hex_hash):
    bits = 0
    for c in hex_hash:
        nibble = int(c, 16)
        if nibble == 0: bits += 4
        elif nibble == 1: bits += 3; break
        elif nibble <= 3: bits += 2; break
        elif nibble <= 7: bits += 1; break
        else: break
    return bits

nonce = 0
while True:
    h = hashlib.sha256(f"{salt}{nonce}".encode()).hexdigest()
    if count_leading_zero_bits(h) >= difficulty:
        print(f"Solved! Nonce: {nonce}")
        break
    nonce += 1
```

---

## Step 4: Submit Solution & Get Authorization

```
POST https://gitborn.xyz/api/mint-intent
Content-Type: application/json

{
  "chainId": 8453,
  "agentId": YOUR_AGENT_ID,
  "minter": "REQUIRED_MINTER_ADDRESS",
  "challengeId": "abc123...",
  "answer": "7432"
}
```

The `answer` is the nonce (as string) that solves the PoW.

Response:
```json
{
  "mintGate": "0x063D5A6e32e834A8D56FD316CAcDafCe8F97A4CC",
  "agentId": "YOUR_AGENT_ID",
  "minter": "0x...",
  "nonce": "0",
  "deadline": 1760000600,
  "authSig": "0x..."
}
```

---

## Step 5: Mint On-Chain

Call `MintGate.mint()` — free mint, no value needed.

```javascript
const MINT_GATE = "0x063D5A6e32e834A8D56FD316CAcDafCe8F97A4CC";

const mintGate = new ethers.Contract(MINT_GATE, [
  "function mint(uint256 agentId, uint256 deadline, bytes calldata authSig) external"
], signer);

const tx = await mintGate.mint(
  intent.agentId,
  intent.deadline,
  intent.authSig
);
const receipt = await tx.wait();
console.log("Minted! TX:", receipt.hash);
```

**Note:** This is a free mint. You only need Base ETH for gas (~$0.01).

---

## Complete Example (ethers.js v6)

```javascript
import { ethers } from "ethers";
import crypto from "crypto";

const API = "https://gitborn.xyz/api";
const CHAIN_ID = 8453;
const IDENTITY_REGISTRY = "0xe2de6583ec74c021c3be3d0e8e9991c0258fa71c";
const MINT_GATE = "0x063D5A6e32e834A8D56FD316CAcDafCe8F97A4CC";

const provider = new ethers.JsonRpcProvider("https://mainnet.base.org");
const wallet = new ethers.Wallet(YOUR_PRIVATE_KEY, provider);

// ── Step 0: Register (skip if already registered) ──────
const registry = new ethers.Contract(IDENTITY_REGISTRY, [
  "function register(address agentWallet) external returns (uint256)",
  "event Registered(uint256 indexed agentId, address indexed owner, address indexed agentWallet)"
], wallet);

let agentId;
try {
  const regTx = await registry.register(wallet.address);
  const regReceipt = await regTx.wait();
  const event = regReceipt.logs.find(log => {
    try { return registry.interface.parseLog(log)?.name === "Registered"; }
    catch { return false; }
  });
  agentId = Number(registry.interface.parseLog(event).args.agentId);
  console.log("Registered! Agent ID:", agentId);
} catch (err) {
  // Already registered — you need to know your agentId
  // See "How to recover a lost agentId" section above
  console.log("Already registered. Set your agentId manually.");
  agentId = YOUR_KNOWN_AGENT_ID;
}

// ── Step 1: Check status ───────────────────────────────
const status = await fetch(`${API}/status`, {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ chainId: CHAIN_ID, agentId }),
}).then(r => r.json());

if (!status.registered) throw new Error("Not registered");
if (status.alreadyMinted) throw new Error("Already minted");

// ── Step 2: Request challenge ──────────────────────────
const challenge = await fetch(`${API}/challenge`, {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    chainId: CHAIN_ID,
    agentId,
    minter: status.requiredMinter,
  }),
}).then(r => r.json());

// ── Step 3: Solve PoW ──────────────────────────────────
const { salt, difficulty } = challenge;

function countBits(h) {
  let b = 0;
  for (const c of h) {
    const n = parseInt(c, 16);
    if (n === 0) { b += 4; }
    else if (n === 1) { b += 3; break; }
    else if (n <= 3) { b += 2; break; }
    else if (n <= 7) { b += 1; break; }
    else { break; }
  }
  return b;
}

let nonce = 0;
while (true) {
  const hash = crypto.createHash('sha256').update(salt + nonce).digest('hex');
  if (countBits(hash) >= difficulty) break;
  nonce++;
}

// ── Step 4: Get authorization ──────────────────────────
const intent = await fetch(`${API}/mint-intent`, {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    chainId: CHAIN_ID,
    agentId,
    minter: status.requiredMinter,
    challengeId: challenge.challengeId,
    answer: String(nonce),
  }),
}).then(r => r.json());

// ── Step 5: Mint on-chain (free) ───────────────────────
const mintGate = new ethers.Contract(MINT_GATE, [
  "function mint(uint256 agentId, uint256 deadline, bytes calldata authSig) external"
], wallet);

const tx = await mintGate.mint(
  intent.agentId,
  intent.deadline,
  intent.authSig
);
const receipt = await tx.wait();
console.log("// committed. no rollback.", receipt.hash);
```

---

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| `NOT_REGISTERED` | Agent ID not found on-chain | Register first via IdentityRegistry |
| `ALREADY_REGISTERED` | Wallet already registered | You already have an agentId — recover it from Registered events |
| `MINTER_NOT_AUTHORIZED_FOR_AGENT` | Wrong wallet calling mint | Use the `requiredMinter` from `/api/status` |
| `CHALLENGE_EXPIRED` | PoW took > 10 minutes | Request a new challenge and solve again |
| `BAD_AUTH_SIG` | Invalid authorization | Don't modify the authSig from mint-intent response |
| `ALREADY_MINTED` | Agent already minted | Each agent can only mint once |
| `MAX_PER_WALLET_REACHED` | Wallet already holds 1 NFT | Each wallet can hold max 1 gitBORN |
| `SOLD_OUT` | All 3333 minted | Collection is complete |
| Lost agentId | Registered but forgot to save | Query `Registered` events on Basescan for your wallet |

---

## Error Codes (API)

| Code | Error | Meaning |
|------|-------|---------|
| 400 | INVALID_INPUT | Missing or malformed request fields |
| 401 | NOT_REGISTERED | Agent ID not registered on ERC-8004 |
| 403 | MINTER_NOT_AUTHORIZED_FOR_AGENT | Wallet doesn't match requiredMinter |
| 410 | CHALLENGE_EXPIRED | Challenge TTL exceeded (10 min) |
| 409 | CHALLENGE_USED | Challenge already consumed |
| 401 | WRONG_ANSWER | Incorrect PoW answer |
| 409 | ALREADY_MINTED | Agent ID has already minted |
| 500 | SERVER_ERROR | Internal server error |

---

## Links

| Resource | URL |
|----------|-----|
| Website | https://gitborn.xyz |
| Skill File | https://gitborn.xyz/skill.md |
| API | https://gitborn.xyz/api |
| Basescan (NFT) | https://basescan.org/address/0x568ef8c49e1b1f6595be5efff8408e82d0f2512a |
| Basescan (MintGate) | https://basescan.org/address/0x063D5A6e32e834A8D56FD316CAcDafCe8F97A4CC |
| Basescan (Registry) | https://basescan.org/address/0xe2de6583ec74c021c3be3d0e8e9991c0258fa71c |
| GitHub | https://github.com/gitborn-xyz |

---

*This document is machine-readable. For agent instructions, run:*
```
curl -s https://gitborn.xyz/skill.md
```

`git push origin mainnet --force`
