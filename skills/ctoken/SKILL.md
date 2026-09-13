---
name: ctoken
description: >
  Integrate Inco's Confidential Token (cToken): wrap any ERC-20 into private balances and
  private transfer amounts on Base and Base Sepolia. Covers the @inco/ctoken SDK (core client,
  React hooks, UI kit), sessions for silent balance reads, Safe / ERC-1271 accounts, and the
  public indexer REST API.
  TRIGGER: imports "@inco/ctoken", mentions cToken, confidential token, private ERC-20 balances,
  confidential transfer/send, ctoken.inco.org, ConfidentialWallet widget,
  useBalances/useDeposit/useConfidentialSend hooks.
  NOT for: writing new confidential Solidity contracts (use the lightning skill), ZK proofs.
---

# Inco Confidential Token (cToken)

cToken turns any ERC-20 into a confidential token. Balances and transfer amounts stay encrypted, only the holder can read them, and the token still moves on-chain like any other. This skill is for integrating cToken with the `@inco/ctoken` SDK. To write new confidential contracts, use the lightning skill.

**Inco is TEE-based, not FHE.** Never describe cToken as FHE.

## How it works

Wrap: approve the cToken, call `wrap(to, amount)`, your ERC-20 goes into a shared vault and you get an encrypted balance. Send: amounts travel encrypted, the tx pays a small fee. Unwrap: burn confidentially, get the underlying back. Read: balances are on-chain `bytes32` handles, decrypted off-chain through a session.

## Networks and addresses

| Network | `network` | Indexer | App |
|---|---|---|---|
| Base | `base` | `https://api.ctoken.inco.org/api` | [ctoken.inco.org](https://ctoken.inco.org) |
| Base Sepolia | `baseSepolia` | `https://api.ctoken.testnet.inco.org/api` | [ctoken.testnet.inco.org](https://ctoken.testnet.inco.org) |

The same addresses work on both networks, and the SDK ships them. Never hardcode:

| Contract | Address |
|---|---|
| `WrapperFactory` | `0x6f9a0ECD77C3Dade8Dc14a507cAbABFD746575f8` |
| `CommonVault` | `0x35A38eFec618cD1924CBf4FB84bE187323Fa7E55` |
| `CTokenSessionVerifier` | `0x8D762303c41F822D3f38f8CE0f0fD1855d602c2A` |

Base Sepolia USDC comes from the [Circle faucet](https://faucet.circle.com/). Every widget and hook runs in the [live playground](https://ctoken-tze4f72wfa-ew.a.run.app/).

## Install and pick a layer

```bash
npm install @inco/ctoken viem @inco/lightning-js
```

This skill targets `@inco/ctoken` **0.2.x**. 0.2.0 removed `CTokenClient.browser()` / `.node()` and the `indexerUrl` option; there is one `new CTokenClient(...)` for Node and browsers, `publicClient` is required, and the indexer is opt-in via `indexer`. Migrating from 0.1.x: https://docs.inco.org/ctoken/migration.

| Layer | Import | Use it when |
|---|---|---|
| Core | `@inco/ctoken` | Raw APIs, works anywhere JS runs |
| React hooks | `@inco/ctoken/react` | You have your own UI |
| UI kit | `@inco/ctoken/ui` | You want working widgets today |

## Core client

```ts
import { CTokenClient } from "@inco/ctoken";

const client = new CTokenClient({
  network: "base", // or "baseSepolia"
  publicClient,    // required: your viem read client for that chain
  walletClient,    // your signer; required for wallet reads, sessions, and writes
  indexer: true,   // off by default; true = hosted indexer, { url } = your own
});

await client.deposit({ token: usdc, amount: "100" });           // wrap
await client.confidentialSend({ token: usdc, to, amount: "25" });
await client.withdraw({ token: usdc, amount: "10" });           // unwrap

const balances = await client.balancesSettled({ tokens: [usdc] });
// balances[usdc] -> { value: 65, raw, formatted, decimals, pending: false }
// or pending: true while the ciphertext is still settling

const exact = await client.balanceExact({ token: usdc });       // { raw, formatted, decimals }
const history = await client.history({ page: 1 });              // needs indexer
const prices = await client.prices([usdc]);                     // needs indexer
```

Node and browsers use the same constructor. For scripts and backends build the `walletClient` yourself (`createWalletClient({ account: privateKeyToAccount(key), chain, transport })`); the SDK does not accept raw private keys. Own deployment? Pass `contracts` and `sessionVerifier` to override.

## React hooks

```tsx
import { CTokenProvider } from "@inco/ctoken/react";

// Under QueryClientProvider. Omit walletClient to use the surrounding wagmi provider.
<CTokenProvider network="base" publicClient={publicClient} indexer>
  <App />
</CTokenProvider>
```

```tsx
import { useBalances, useDeposit, useConfidentialSend } from "@inco/ctoken/react";
```

Hooks: `useCToken`, `useTokens`, `useResolvedTokens`, `useAssets`, `useBalances`, `useBalancesSettled`, `useBalance`, `useBalanceExact`, `usePublicBalances`, `usePublicBalance`, `usePublicBalanceExact`, `useDeposit`, `useApprove`, `useWithdraw`, `useConfidentialSend`, `useDecrypt`, `useHistory`, `useChainGuard`. Built on wagmi and react-query.

## UI kit

```tsx
import { ConfidentialWallet } from "@inco/ctoken/ui";
import "@inco/ctoken/ui/styles.css";

<ConfidentialWallet />
```

Or compose the pieces: `DepositWidget`, `WithdrawWidget`, `SendWidget`, `BalanceCard`, `PortfolioCard`, `HistoryList`.

## Sessions

Decrypting a balance needs the account's authorization. A session is one signed voucher, then a throwaway key reads silently until it expires. The SDK does this automatically on first read. Details and manual voucher flow: `references/sessions.md`.

Safe and other ERC-1271 accounts work end to end, but must go through sessions. Direct decrypts only accept EOA signatures.

## Indexer REST API

Open, rate limited per IP, localhost origins always allowed. Endpoints and semantics: `references/indexer-api.md`. Production apps should request origin whitelisting via the [whitelist form](https://docs.google.com/forms/d/e/1FAIpQLSe_lJoMc203TEgdQlL6PhPywG2EW3jdWIOo6QwvoBK0n1QCJQ/viewform?usp=header).

## Red flags

| Thought | Reality |
|---|---|
| "Read the balance right after the transfer" | Ciphertexts settle async. Use `balancesSettled`, respect `pending`. |
| "This Safe can decrypt directly" | Smart accounts always go through sessions. |
| "Parse the amount from the indexer" | The indexer only serves handles, plaintext never leaves the TEE. |
| "Hardcode the addresses" | Pass `network`, the SDK ships the addresses. |
| "`CTokenClient.browser()` / `indexerUrl`" | Gone in 0.2.0. `new CTokenClient({ network, publicClient, walletClient, indexer })`. |
| "cToken uses FHE" | It is TEE-based. Say TEE. |

## Docs

https://docs.inco.org/ctoken/overview, /sdk, /sessions, /indexer-api
