# arc-pow-sigils-mcp

A standalone [Model Context Protocol](https://modelcontextprotocol.io) (MCP) server
for the **Proof of Architect** NFT collection (deterministic Architector cards) — a proof-of-work
minted NFT on **Arc mainnet** (chainId `5042`, native gas token **USDC**, 18 decimals).

The server exposes **read-only** tools over the deployed `PowMintNFTv3` contract so an
LLM agent can inspect collection stats, token data, PoW difficulty and pricing — and
even **verify a mined nonce without sending a transaction**.

- Contract (v3.4 mainnet): `0x3E20bb7be2C46f94Cab78d340D3F79Afc2a9Fed4` (Arc mainnet; override via `CONTRACT_ADDRESS`)
- RPC: `https://rpc.mainnet.arc.io` (override via `ARC_RPC_URL`)
- Transport: **stdio** (newline-delimited JSON-RPC)

---

## Quick start

### Run with npx (after publishing)

```bash
npx arc-pow-sigils-mcp
```

### Run from source

```bash
npm install
npm run build      # tsc -> dist/
npm run smoke      # spawns the server and hits the live chain
node dist/index.js # start the stdio server
```

The server speaks MCP over stdin/stdout and logs only to stderr — it is normally
launched by an MCP client, not by hand.

---

## Use in Claude Desktop

Add this to your `claude_desktop_config.json`
(`~/Library/Application Support/Claude/claude_desktop_config.json` on macOS,
`%APPDATA%\Claude\claude_desktop_config.json` on Windows):

```json
{
  "mcpServers": {
    "arc-pow-sigils": {
      "command": "npx",
      "args": ["-y", "arc-pow-sigils-mcp"],
      "env": {
        "SITE_URL": "https://proofofarchitect.builders"
      }
    }
  }
}
```

Or, pointing at a local build:

```json
{
  "mcpServers": {
    "arc-pow-sigils": {
      "command": "node",
      "args": ["/absolute/path/to/mcp/dist/index.js"],
      "env": {
        "CONTRACT_ADDRESS": "0x3E20bb7be2C46f94Cab78d340D3F79Afc2a9Fed4",
        "ARC_RPC_URL": "https://rpc.mainnet.arc.io",
        "SITE_URL": "https://proofofarchitect.builders"
      }
    }
  }
}
```

Restart Claude Desktop after editing the config.

---

## Tools

| Tool | Input | Returns |
| --- | --- | --- |
| `collection_stats` | — | `totalMinted`, `maxSupply`, `freeClaims`, `claimsLeft`, `claimedCount`, `currentWave`, `currentPrice` (wei + USDC), `mintPaused` |
| `get_token` | `tokenId` (number) | `owner`, raw `seedOf`, `displaySeed` (post-inclusion seed that drives traits/art), `mintBlock`, `pending`, `nonce`, `tokenURI`, off-chain `image` + `metadata` urls. Error content if the token doesn't exist. |
| `required_bits` | `miner` (address) | `requiredBits` (display leading zero bits), `requiredMilli` (v3.4 milli-bits), the exact `target`, plus the wave/`loadAdjust`/`streakBits` breakdown. Three difficulty layers: base **30 + 2 bits per wave**, a pace regulator, and a per-wallet streak (+2 bits per extra mint inside the cooldown window = 60 s × wave); a staking discount is applied in milli-bits. |
| `verify_nonce` | `miner` (address), `nonce` (uint256 decimal string) | `work`, `target`, `valid` — verifies a mined nonce **without a transaction** (`valid` iff `uint256(work) < target`). |
| `price_info` | — | `currentPrice` + wave math (epoch size `1000`, `priceStart` `1.0 USDC`, `x2` per wave, **no cap** — 15 waves, last wave 16 384 USDC). |
| `craft_info` | — | CraftingControllerV2 config (one-shot model): `paused`, `craftFee`, and per-tier `boostCost`/`feeFor`/`maxChosen` (tiers 0..3). Each craft is a single payable transaction — there is **no** commit/reveal step, no refund, and no salt. |

All tools return a single JSON text content block. `verify_nonce` recomputes the
preimage hash locally as
`keccak256(abi.encodePacked(chainId, contract, miner, nonce))` and checks it against
the wallet's on-chain `targetFor` — so it does not trust the RPC for the PoW verdict.

### Example: verify a mined nonce

Sample response for a placeholder wallet on the current mainnet core
(`CONTRACT_ADDRESS`, as of 2026-09):

```json
{
  "miner": "0x1111111111111111111111111111111111111111",
  "nonce": "1024085",
  "work": "0x0abff08ab4b0e81f9f54ff18d6ae1ecf695c2b7eca20527a80e3dfa3da6801d6",
  "workMatchesOnChain": true,
  "target": "107839786668602559178668060348078522694548577690162289924414440996864",
  "valid": false
}
```

Call it as `verify_nonce(miner = "0x1111111111111111111111111111111111111111", nonce = "1024085")`.
The PoW formula is the same across instances — the contract address binds the preimage, so
pass nonces mined for the contract you query. `valid` is checked against the **current**
fractional target; a nonce mined before a wave escalation or regulator tightening may no
longer pass.

---

## Environment variables

| Variable | Default | Purpose |
| --- | --- | --- |
| `CONTRACT_ADDRESS` | `0x3E20bb7be2C46f94Cab78d340D3F79Afc2a9Fed4` | PowMintNFTv3_4 address (Arc mainnet canon) |
| `CRAFT_ADDRESS` | `0xb7f32811F19579D9FC6F0e5ac925473554091a91` | CraftingControllerV2 address (one-shot crafting) |
| `ARC_RPC_URL` | `https://rpc.mainnet.arc.io` | Arc RPC endpoint. A comma-separated list is accepted; endpoints are tried in order (viem `fallback`). |
| `ARC_CHAIN_ID` | `5042` | Arc chain id (set `5042002` for testnet). |
| `ARC_EXPLORER_URL` | `https://explorer.arc.io` | Block explorer base URL. |
| `ARC_IS_TESTNET` | `false` | Set to `true` for the testnet. |
| `SITE_URL` | `https://proofofarchitect.builders` | Base site for `/api/image/{id}` and `/api/meta/{id}` links |

All variables are optional. No secrets are read or stored.

---

## Publish to the MCP registry

1. **Check the registry name.** The official
   [MCP registry](https://registry.modelcontextprotocol.io) requires a reverse-DNS
   server name that includes your GitHub username. This package is already set to
   `io.github.Proofofarchitect/arc-pow-sigils` in **both** `package.json`
   (`mcpName` field) and `server.json` (`name` field) — keep them in lockstep.

2. **Publish the npm package** (the registry resolves the stdio package by name):

   ```bash
   npm run build
   npm publish --access public
   ```

3. **Install the registry publisher CLI** and publish the server metadata (requires
the GitHub account `Proofofarchitect` for the namespace check):

   ```bash
   npx @modelcontextprotocol/mcp-publisher --help

   npx @modelcontextprotocol/mcp-publisher init      # scaffolds/validates server.json
   npx @modelcontextprotocol/mcp-publisher login github   # auth with your GitHub account
   npx @modelcontextprotocol/mcp-publisher publish   # publishes server.json
   ```

   The publisher verifies that the `name` namespace matches your authenticated
   GitHub identity (that is why the placeholder must be replaced).

4. **Verify** on the registry at
   [registry.modelcontextprotocol.io](https://registry.modelcontextprotocol.io).
   New entries are served with **preview** status while the registry is in preview.

> Keep `package.json` `version` and `server.json` `version` (and the package
> `version` field) in lockstep on every release.

---

## Development

```
src/chain.ts   viem client singleton (env-driven chain + RPC failover), contract
               read helpers, milli-bit target checks, USDC formatting, work hash
src/index.ts   McpServer + 6 registered tools over StdioServerTransport
smoke.mjs      minimal newline-delimited JSON-RPC client used by `npm run smoke`
server.json    MCP registry manifest (npm stdio)
```

The ABI is embedded in this package (see `src/chain.ts`) — it is not imported from
the website package, so `mcp/` stays independently publishable.

## License

MIT
