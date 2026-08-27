# HypeFuel

**Swap USDC for native HYPE on HyperEVM without holding any gas.**

HyperEVM charges gas in HYPE. Bridge in stablecoins and hold no HYPE, and you are stuck: you
cannot swap for HYPE, because swapping needs gas. Gas.zip and SmolRefuel do not cover HyperEVM,
so people end up with a wallet full of money they cannot move.

HypeFuel fixes that. The user signs one message (no transaction, no gas) and a relayer
broadcasts it, taking their USDC and delivering native HYPE in the same transaction.

| | |
|---|---|
| App | https://hypefuel.me |
| API | https://api.hypefuel.me |
| Contract | [`0x42b06b1d9a07Fc3925C518dbf9475E7cA80DC8DF`](https://hyperevmscan.io/address/0x42b06b1d9a07Fc3925C518dbf9475E7cA80DC8DF) |
| Implementation | [`0xDF4FdA0F992D7E0Ed73d81726bAD854214eA373e`](https://hyperevmscan.io/address/0xDF4FdA0F992D7E0Ed73d81726bAD854214eA373e) |
| Chain | HyperEVM mainnet (999) |

The contract address is a proxy and is the one to integrate against; it does not change across
upgrades. The implementation is listed only so an upgrade can be checked against source.

## How it works

The central constraint is that a wallet with no gas cannot send a transaction, so it cannot
emit an on-chain event either. Watching the chain for user intent is therefore impossible; the
authorization has to reach us off-chain, which is what the relayer API is for.

```
User                       Relayer API                  HypeFuel contract
 |                              |                              |
 |-- POST /v1/quote ----------->|-- quote() ------------------->| reads HYPE price from
 |<-- order + price ------------|<------------------------------| HyperCore precompiles
 |                              |                              |
 | sign EIP-3009 (no gas)       |                              |
 |-- POST /v1/fill ------------>|-- fill(order, sig) --------->| pulls USDC via EIP-3009,
 |                              |   (relayer pays gas)         | sends HYPE to the signer
 |<-- HYPE in wallet -----------|                              |
```

### One signature covers the whole order

Payment uses **EIP-3009 `receiveWithAuthorization`** rather than EIP-2612 `permit`. It moves
tokens in a single atomic step, carries its own expiry, and refuses to execute unless the caller
is the named payee, so the signature is safe to hand to a relayer, because only the HypeFuel
contract can ever spend it.

EIP-3009 signs over a fixed set of fields: `(from, to, value, validAfter, validBefore, nonce)`.
None of them can carry order data such as `minHypeOut`. HypeFuel resolves this by **deriving the
authorization's `nonce` from a hash of the entire order**:

```solidity
nonce = keccak256(abi.encode(ORDER_TYPEHASH, user, usdcIn, minHypeOut, validAfter, validBefore, salt));
```

Because the token verifies the signature over that nonce, tampering with any field changes the
nonce and invalidates the signature. One signature commits to the whole order, and the token's
own `authorizationState` mapping provides replay protection for free.

### Pricing

The contract reads HYPE's price on-chain from Hyperliquid's HyperCore precompiles at execution
time, via [`hyper-evm-lib`](https://github.com/hyperliquid-dev/hyper-evm-lib):

- `oraclePx` (`0x…807`) for the HYPE perp: the validator-median price, expensive to manipulate.
- `spotPx` (`0x…808`) for HYPE/USDC spot, as a cross-check.

Fills revert if the two feeds disagree by more than `maxOracleDeviationBps`, and the contract
prices at the **higher** of the two, so pushing either feed down cannot extract extra HYPE. An
attacker would have to move both feeds at once.

### Fees

**3% of the amount, with a $0.15 minimum**, capped on-chain at 5% and $1.00 by immutable
constants that governance cannot exceed.

The flat floor exists because per-fill costs do not scale with order size. A live mainnet fill
costs about **$0.0009** in gas (159k gas at ~0.3 gwei), so the floor is not really about gas.
It is about making a $1 top-up worth processing at all. The percentage takes over above $5.

| Order | Fee | Effective |
|---|---|---|
| $1 | $0.15 | 15% |
| $5 | $0.15 | 3% (crossover) |
| $10 | $0.30 | 3% |
| $50 | $1.50 | 3% |

### Keeping inventory topped up

Every fill receives USDC and dispenses HYPE, so over time running the contract reduces inventory
and increases USDC holdings. The 3% fee means 97% of volume has to be recycled back into HYPE,
indefinitely. The USDC accumulated by the contract is the exact funds needed to repurchase that
HYPE, so nothing requires subsidising. The contract stays fully backed in value terms and merely
ends up holding the wrong asset.

`rebalance()` completes the cycle on-chain. It is permissionless, just like `fill`, and pays no
bounty, because the motivation already exists: whoever wants inventory to be available is the next
person who wants a fill. A user whose fill just reverted as `insufficient_liquidity` can rebalance
and then fill.

The swap runs against **Project X's USDC/WHYPE 0.05% pool**
([`0x6c9A…9285`](https://hyperevmscan.io/address/0x6c9A33E3b592C0d65B3Ba59355d5Be0d38259285)), the
deepest USDC/HYPE venue on HyperEVM at roughly $14M of liquidity, around 15x HyperSwap's
comparable pool. HypeFuel interacts with the pool contract directly rather than via a router,
eliminating both the need to trust a router address and any outstanding token approval.

The pool price is never relied upon. `minHypeOut` is calculated from the same HyperCore feeds that
price fills, which an AMM manipulator cannot move, so a skewed pool makes the rebalance revert
rather than buy HYPE at an inflated price. A rebalance is never time-critical, so failing closed
and retrying later is the appropriate behaviour.

The two feeds do different jobs. The spend is sized off the *lower* of them, the cautious side
when buying, so a disagreement between the feeds can never overspend. The bound on what comes back
uses the *higher* one, because a bound priced off the lower feed becomes unreachable the moment the
feeds sit further apart than the slippage tolerance: fills would carry on selling across the whole
`maxOracleDeviationBps` band while every refill reverted, draining inventory with no way to restock
it. The worst a swap can pay is therefore the higher feed plus the tolerance, and since fills sell
at that same higher feed, recycling stays profitable for as long as `feeBps` (300) sits above the
tolerance (100).

Bounding off the higher feed widens what a manipulator could in principle take, so it was measured
against the live pool with the spot feed held at the top of the 5% band. Skewing the pool with $1M
makes a full refill overpay by **$4.03** and costs the manipulator **$991.79** on the round trip
to set up and unwind, and past roughly that point the bound rejects the swap outright rather than
paying more. Extraction scales with the size of a refill while the cost of moving the pool does
not, so at current liquidity the two only meet if the float grows around 250-fold. A thinner pool
would bring that threshold down; either is when this is worth revisiting.

Against real liquidity the all-in cost of a rebalance measures about **5 bps**, rising to 13 bps
even at $50k. The tolerance is set at 100 bps, providing ample room for an honest swap while
limiting what a sandwich can capture.

Two parameters determine when it fires. `hypeTarget` is the level a rebalance refills to, and
`hypeFloor` is the level at or below which one becomes permitted. The gap between them serves a
purpose: a successful rebalance is meant to lift the balance clear of the floor, so another cannot
occur until real depletion takes place, and nobody can chain swaps to bleed the pool fee at our
expense. The setter keeps that gap honest by rejecting any floor a rebalance executing at the
edge of the slippage tolerance would fail to clear when the feeds agree, and by rejecting a floor
of zero, which would demand a balance of exactly zero and let anyone deny it forever with a
single wei. A spread between the feeds, or USDC truncation, can still leave a small deficit.

### The keeper

Nothing about `rebalance()` needs a keeper. Any user can call it, and one whose fill just failed
has every reason to. A cron simply means nobody has to notice first. The relayer Worker runs one
every minute:

```
cron ──▶ pendingRebalanceUsdc() ──▶ 0      ──▶ done, one eth_call spent
                                └──▶ $n    ──▶ simulate ──▶ rebalance()
```

`pendingRebalanceUsdc()` reproduces every precondition in `rebalance()` and returns zero when any
of them fails, so the keeper needs no opinion of its own about when to act, and a disagreement between
the two is impossible because only one of them decides. It simulates before broadcasting, so a
skewed pool or a price that moved mid-block costs nothing and is logged with a reason.

The keeper runs on the relayer's key. That is a deliberate choice rather than an oversight:
`rebalance()` grants no privileges, so the key is only ever paying gas, and sharing it leaves one
HYPE balance to keep funded instead of two. The cost is that a rebalance can occasionally take a
nonce a fill wanted, which `submitFill` already retries through. Give the keeper its own key if
fill volume ever makes that contention worth removing.

## Layout

```
packages/
  contracts/   Foundry. HypeFuel.sol plus 96 tests, including fork tests against real USDC.
  sdk/         @hypefuel/sdk: order encoding, typed data, quote math. Browser/Node/Workers.
  api/         Cloudflare Worker relayer. Quote, simulate, broadcast, plus the rebalance keeper.
  web/         Vite + React app: landing page, swap UI, integration docs.
    assets-src/  SVG sources for the icons and social card. See "Generated assets".
    public/      Generated output, plus robots.txt and the _headers policy.
scripts/       Repo checks that need no dependencies.
```

The SDK is shared by the API and the web app, so the order encoding and quote arithmetic exist
in exactly one place on the TypeScript side.

`packages/web/src/content/site.ts` plays the same role for copy. The titles, descriptions and the
landing page FAQ live there once, and `vite.config.ts` reads them at build time to emit the
`FAQPage` structured data and `sitemap.xml`. An answer therefore cannot be reworded on the page and
left stale in the markup that Google reads.

Two things about the web build are worth knowing before you change it:

Each route gets its own HTML file. The build writes `app/index.html` and `docs/index.html` with the
titles, descriptions and canonical URLs from `site.ts`, because social crawlers and unfurlers do not
run React and would otherwise read the landing page's metadata for every link. `html_handling` is
set to `drop-trailing-slash` so `/docs` is the path those files are served from, matching what the
canonical tags claim.

Nothing inline may execute. `public/_headers` sets a Content-Security-Policy with `script-src
'self'`, so an inline `<script>` or an `onclick` attribute will be blocked and the page will break.
The JSON-LD block is exempt because a data block is not script. Adding a third-party origin for
script, style, fonts or network calls means adding it to that policy too.

### Generated assets

Everything binary under `packages/web/public/` is rendered from the SVG sources beside it, so edit
the SVG rather than the PNG:

```bash
brew install librsvg imagemagick
packages/web/assets-src/render.sh
```

The favicon ships in three cuts because one does not survive every context: `icon-small.svg` fattens
the bolt and drops the hairline border, which is the difference between a mark and a smudge at 16px;
`icon-square.svg` is full-bleed because iOS and Android apply their own corner mask; `icon-rounded`
is the one browsers show at 96px and up.

## Development

Requires [Foundry](https://getfoundry.sh), Node 22+ and pnpm 9+. Wrangler 4 refuses to run on
older Node versions, so deploying needs 22.

```bash
git clone --recurse-submodules https://github.com/chase-mew/hype-fuel
cd hype-fuel
pnpm install
pnpm --filter @hypefuel/sdk build   # the API and web app import the built SDK

pnpm test                            # contracts + SDK
pnpm dev:web                         # web app on :5173
pnpm dev:api                         # relayer on :8787
```

### Testing

```bash
cd packages/contracts
forge test --no-match-path 'test/*.fork.t.sol'   # hermetic, no network
forge test --match-path 'test/*.fork.t.sol'      # against real mainnet USDC and live oracles
```

Two suites, deliberately:

- **Hermetic tests** run against `MockUSDC` and `MockV3Pool`, faithful reimplementations of
  Circle's FiatTokenV2_2 and a Uniswap V3 pool. Fast, offline, and able to drive execution price
  directly so manipulated and partial fills are easy to exercise.
- **Fork tests** run the same scenarios against the real token, the real Project X pool and live
  precompiles. These exist to prove the mocks are faithful: both suites assert the same output
  values, so a divergence from mainnet shows up as a failure rather than a surprise in production.
  `Rebalance.fork.t.sol` also skews the real pool with a $3M swap to confirm the oracle bound
  rejects a manipulated price against genuine liquidity.

The order commitment hash is pinned to a shared vector asserted in **both**
`HypeFuel.t.sol` and the SDK's test suite. If the Solidity and TypeScript encoders ever
diverge, that fails in CI instead of silently rejecting every signature the frontend produces.

There are also two live scripts that spend real funds:

```bash
PRIVATE_KEY=0x… FUEL_ADDRESS=0x… bun packages/api/scripts/live-fill.ts 2      # direct contract
PRIVATE_KEY=0x… API_URL=https://…  bun packages/api/scripts/live-api-fill.ts 3 # through the API
```

## Deployment

### Contract

HypeFuel sits behind an ERC-1967 proxy. Publish the implementation, then deploy the proxy with the
`initialize` call as its constructor argument, which leaves no window in which the proxy exists
uninitialized. Then point it at the Project X pool.

```bash
cd packages/contracts
export ETH_RPC_URL=https://rpc.hyperliquid.xyz/evm

# 1. Implementation. Cap the gas explicitly: Forge's estimate buffer would otherwise
#    ask for more than a small block allows.
forge create src/HypeFuel.sol:HypeFuel \
  --rpc-url hyperevm --private-key $PK --broadcast --gas-limit 2600000

# 2. Proxy, initialized atomically.
INIT=$(cast calldata 'initialize(address,address,uint256,uint256,uint256,uint256,uint256)' \
  $OWNER $USDC 300 150000 1000000 50000000 500)
forge create lib/hyper-evm-lib/lib/openzeppelin-contracts/contracts/proxy/ERC1967/ERC1967Proxy.sol:ERC1967Proxy \
  --rpc-url hyperevm --private-key $PK --broadcast --constructor-args $IMPL $INIT

# 3. Rebalancing. Validated on-chain: setPool rejects anything that is not a USDC/WHYPE pair.
cast send $PROXY 'setPool(address)' 0x6c9A33E3b592C0d65B3Ba59355d5Be0d38259285 --private-key $PK
cast send $PROXY 'setRebalanceConfig(uint256,uint256,uint256,uint256)' \
  1800000000000000000 900000000000000000 1000000 100 --private-key $PK
```

`script/Deploy.s.sol` does the same thing in one transaction batch and is the better record of
intent, but it needs `FOUNDRY_BYTECODE_HASH=ipfs` to run: `foundry.toml` sets
`bytecode_hash = "none"` for reproducible bytecode, and Forge 1.5 identifies a script's target
contract through that metadata hash, so without the override it fails with `No contract bytecode`.
That override also changes the bytecode being deployed, which is why the live deployment used
`forge create`. Only scripts are affected; `forge test` and `forge build` are fine.

The implementation contains no state and accepts no constructor parameters, so an upgrade is just
publishing a fresh one and pointing the proxy at it. Nothing needs re-supplying, which is
deliberate: the USDC address lives in storage rather than in an immutable, so no upgrade can set it
wrongly.

```bash
forge create src/HypeFuel.sol:HypeFuel \
  --rpc-url hyperevm --private-key $PK --broadcast --gas-limit 2600000
cast send $PROXY 'upgradeToAndCall(address,bytes)' $NEW_IMPL 0x --private-key $PK

# Then confirm state survived, and register the new implementation with the explorer.
cast call $PROXY 'owner()(address)' && cast call $PROXY 'usdc()(address)'
```

Verification uses an Etherscan V2 key, which covers HyperEVM as chain 999:

```bash
forge verify-contract $IMPL src/HypeFuel.sol:HypeFuel --chain 999 --watch \
  --etherscan-api-key $ETHERSCAN_API_KEY \
  --verifier-url 'https://api.etherscan.io/v2/api?chainid=999'
```

**Publishing an implementation is close to the small-block ceiling.** At ~11KB of runtime code it
costs about 2.45M gas, against a small-block limit that is currently 3M. It fits, so no big-block
toggle is needed, but the headroom is only ~20%, so if the implementation grows much further, enable
big blocks on the deployer for the publish step and turn them off again afterwards. Every other
transaction here is under 500k.

> Moving to a proxy changed the address once, in July 2026. `deployments.json` records both the
> current deployment and the retired one, which is drained and paused. From here the address is
> stable across upgrades.

### Relayer and web app

```bash
cd packages/api
wrangler secret put RELAYER_PRIVATE_KEY     # never committed, never exposed to CI
wrangler deploy

cd ../web
pnpm build && wrangler deploy
```

Set `FUEL_ADDRESS` in `packages/api/wrangler.jsonc` after deploying the contract. The same file
declares the keeper's cron trigger, so `wrangler deploy` is what makes it live; confirm the output
ends with `schedule: * * * * *`.

Each worker also declares its own hostname (`hypefuel.me` for the web app, `api.hypefuel.me` for
the relayer), so the routing lives in the repo rather than only in the dashboard. Both keep
`workers_dev` on: declaring a route otherwise switches the `workers.dev` address off, and those
addresses were published before the domain existed.

`.github/workflows/deploy.yml` does both automatically on push to `main`, given
`CLOUDFLARE_API_TOKEN` and `CLOUDFLARE_ACCOUNT_ID` repository secrets.

`CLOUDFLARE_ACCOUNT_ID` is already set. To finish wiring up automatic deploys, create a token
with the **Edit Cloudflare Workers** template at
<https://dash.cloudflare.com/profile/api-tokens> and add it:

```bash
gh secret set CLOUDFLARE_API_TOKEN
```

Until then, deploy by hand with the commands above; the workflow reaches Cloudflare and stops
there.

## Operating it

The contract holds HYPE inventory and accumulates USDC from fills.

```bash
# Add HYPE inventory (plain transfer)
cast send $FUEL --value 1ether --private-key $PK --rpc-url hyperevm

# Collect USDC revenue
cast send $FUEL "withdrawUsdc(address,uint256)" $TREASURY $AMOUNT --private-key $PK

# Recover HYPE
cast send $FUEL "withdrawHype(address,uint256)" $TREASURY $AMOUNT --private-key $PK

# Emergency stop; withdrawals keep working while paused
cast send $FUEL "setPaused(bool)" true --private-key $PK

# Recycle accumulated USDC into HYPE inventory. Anyone can call this.
cast send $FUEL "rebalance()" --private-key $PK --rpc-url hyperevm

# What a rebalance would spend right now, or 0 if none is needed
cast call $FUEL "pendingRebalanceUsdc()(uint256)" --rpc-url hyperevm
```

Inventory replenishes itself: the contract accumulates USDC from fills, and the keeper spends it
back on HYPE every time inventory falls to the floor. So the one balance that still needs watching
is the **relayer EOA's HYPE** for gas, which funds both fills and keeper runs. At ~$0.0009 per
transaction, a small balance lasts a long time.

Raise `hypeTarget` and `hypeFloor` together as inventory grows; they are currently sized for a
3.5 HYPE float, roughly four back-to-back maximum-size fills. The keeper needs no configuration of
its own, and only ever does what `pendingRebalanceUsdc()` tells it to.

```bash
# What the keeper is doing, live
npx wrangler tail --format pretty     # from packages/api
```

## Design decisions worth knowing

**HYPE always goes to the signer.** There is no separate recipient field. Adding one would let a
malicious integrator redirect a user's funds while showing them a legitimate-looking prompt;
locking the destination to the payer makes that impossible.

**`fill` is permissionless.** Every fill is profitable for the contract by construction, so
anyone can relay one. That removes us as a liveness bottleneck: if our relayer goes down, users
or integrators can submit orders themselves with any funded wallet.

**The contract is upgradeable.** HypeFuel operates behind an ERC-1967 proxy using UUPS, so its
address stays constant permanently. This introduces trust and is worth stating explicitly: the owner
can substitute the logic. It does not widen custody risk, since the owner could already withdraw the
inventory, but it does mean a replaced implementation could disregard the terms of an in-flight
signature. Orders are short-lived, so that exposure window is small.

**Fee ceilings are constants.** `MAX_FEE_BPS` and `MAX_MIN_FEE_USDC` bound the fee within any given
implementation. Because an upgrade can lift them, a signer's real guarantee is the `minHypeOut`
their signature commits to.

**Rebalancing pays no bounty.** Adding one would create the only reason a caller might want to
trigger swaps repeatedly. Leaving it out means a caller can never extract value from `rebalance()`,
and the honest incentive of wanting a fill to be servable is enough.

**The keeper is a convenience, not a dependency.** It exists so nobody has to notice inventory is
low, but every path it uses is open to anyone. If the Worker stops, the next user to hit
`insufficient_liquidity` can restock the contract themselves.

**Smart-contract wallets work.** USDC's `bytes`-signature overload validates through a checker
that understands ERC-1271, so Safe and ERC-4337 accounts can sign too.

## Security notes

Not audited. The contract is small and covered by 95 tests, but it holds funds, so treat the
inventory as at-risk capital and keep it sized to demand.

The main economic risk is oracle manipulation, mitigated by using the validator-median perp
oracle, cross-checking it against spot, pricing at the higher feed, and capping order size.
`setPaused` is the emergency stop, and it halts rebalancing as well as fills.

Rebalancing introduces a dependency on an AMM. The pool is set by the owner and validated to be a
USDC/WHYPE pair when assigned, and because the output bound comes from the oracle rather than the
pool, a manipulated pool costs a reverted transaction rather than money.

Upgradeability concentrates trust in the owner key. It is currently an EOA; a multisig or a timelock
would be the natural next step, and neither requires a further address change.

Found something? [SECURITY.md](SECURITY.md) has the private disclosure process and spells out what a
user is trusting. Please do not open a public issue for anything that puts funds at risk.

## Contributing

[CONTRIBUTING.md](CONTRIBUTING.md) covers setup, how to run the suites, and the house style. Two
things that catch people out: the Foundry dependencies are submodules, so clone with
`--recurse-submodules`, and `pnpm lint` fails on em dashes, en dashes and curly quotes anywhere in
the repo, UI copy and code comments included.

## Licence

[MIT](LICENSE)
