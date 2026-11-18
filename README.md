# Gemoon

A meme-token launch protocol on Uniswap V4. A single controller call creates the token, a Meme/USDG
pool with a hook, and a single-sided position holding the entire supply. The hook takes a USDG fee on
every swap; the vault splits it between the meme's creator and stakers and converts it into reward assets.

## Contracts

| Contract | File | Role |
|---|---|---|
| GemoonController | `src/contracts/Gemoon.sol` | Entry point: deploys a meme — token (via TokenDeployer), pool and position. Proxy, OwnableUpgradeable. |
| TokenDeployer | `src/contracts/TokenDeployer.sol`, interface `ITokenDeployer.sol` | Creates the token of a new meme on a controller call and returns its address, nothing more: the whole supply is minted to the controller. Only the controller set in the constructor may call `deployToken`. Not a proxy, no owner, no state. New token code means a new TokenDeployer plus `controller.setTokenDeployer`, with no controller upgrade; memes already created keep their own code. |
| HookManager | `src/contracts/hooks/HookManager.sol`, interface `IHookManager.sol` | Uniswap V4 hook: a USDG fee on every swap (1%..5% chosen by the creator; during the pool's first 15 seconds it starts at 80% and decays linearly), the whole fee goes to the vault, which decides how to split it. Not a proxy (not upgradeable), no owner: vault and controller are set in the constructor, there are no admin functions. The address is mined to match the permission bits. |
| Vault | `src/contracts/vault/Vault.sol` | One contract, one logical vault per meme: receives the whole fee from the hook and splits it (protocol share `protocolShareBps` → `protocolRecipient`, the rest to the creator and stakers), staking, conversion, payouts. Proxy, Ownable2Step. |
| GmoonReferrals | `src/contracts/Referrals.sol`, interface `IReferrals.sol` | Recipient of the protocol share (`Vault.protocolRecipient`): referral codes, binding a user to a referrer, `refShareBips` of the protocol fee of the referred users' memes goes to the referrer, the rest to the protocol. Proxy, Ownable2Step, Pausable (referral payouts only). |
| UniswapV3SwapAdapter | `src/contracts/adapters/UniswapV3SwapAdapter.sol` | Swaps the vault's USDG for reward assets in Uniswap V3 pools; computes the minimum output itself from the pool TWAP. Ownable2Step, not a proxy: the vault switches adapters via `setSwapAdapter`. |
| GemoonToken | `src/contracts/Token.sol` | Meme ERC20 with metadata and admins, ERC-1046 `tokenURI` (base64 JSON built on-chain from the token data; an admin can set a custom URI via `setTokenURI`) and ERC-165, a holder counter `totalHolders`, and the fee distribution `feeInfo` (total, creator, vault, protocol; fixed at deploy and included in `tokenURI`). Created by TokenDeployer, the whole supply goes to the constructor's recipient (the controller). Not upgradeable. |
| ProjectToken | `src/contracts/ProjectToken.sol`, script `script/DeployProjectToken.s.sol` | A standalone project token outside the controller: a copy of GemoonToken without `feeInfo` and admins, `tokenURI` is a plain link (get/set); the owner (Ownable2Step, the deployer wallet) can change the link, image and description. The whole supply (1 bn) is minted to the deployer. The script creates a Uniswap V4 pool against USDG without a hook (fee `PROJECT_POOL_FEE`) and puts the whole supply into a single-sided position; the position NFT goes to the deployer. Not upgradeable. Robinhood: `0xCef233054553A3e27d23f19D3DfBA1A86eC84C62`. |
| PositionDeployer | `src/contracts/utils/PositionDeployer.sol` | Internal library inlined into the controller: mints the position through PositionManager and Permit2. Has no address of its own. |
| Ticks, PriceMath | `src/contracts/utils/Ticks.sol`, `src/contracts/utils/Price.sol` | External libraries, linked at deploy time. |
| GemoonLens | `src/contracts/lens/GemoonLens.sol`, interface `IGemoonLens.sol` | Read-only batch reads for the frontend: everything about a list of memes in one `eth_call`. No state, owner or funds, `view` only. Not a proxy: when protocol addresses change it is redeployed (CREATE2, `make deploy-lens`). |

All key contracts sit behind a `TransparentUpgradeableProxy`, each with its own `ProxyAdmin`.
GemoonLens is the exception: it stores nothing, so there is nothing to upgrade; a new version is a new deployment.
TokenDeployer also has no proxy: it is not upgraded but replaced (`setTokenDeployer`), so the token code
of new memes can change without upgrading the controller.
There is a single pair token for the whole protocol: the controller, hook and vault are built around the same USDG.

## Interaction flow

### Deployment, once

`DeployGemoon` brings everything up in a single run in the order Vault, GemoonController, TokenDeployer,
GmoonReferrals, HookManager, UniswapV3SwapAdapter. The hook receives the vault, controller and referrals
addresses (the protocol-share recipient) in `initialize`. The controller is the only one that may call the
hook's `onlyController` functions (`notifyPoolCreated`). The hook gets the vault and controller addresses in
its constructor and never changes them: the hook has no owner.
The controller receives the hook, vault, TokenDeployer and PositionManager through setters,
the vault receives the hook, controller and adapter. Assets from `DEPLOY_ASSETS_FILE` (robinhood assets)
are allowed in the vault and get a route on the adapter in the same run if their USDG/asset pool
exists in the factory and provides a TWAP; the rest are skipped with the reason in the report. At the end ownership
moves to `GEMOON_OWNER`; the vault, referrals and adapter need a separate `acceptOwnership`.

A half-deployed state is safe: the controller does not deploy tokens without the hook, vault, TokenDeployer and
PositionManager, the hook rejects pools of unregistered memes, the vault accepts fees only from
its own hook.

### Meme lifecycle

1. **Creation.** The creator calls `GemoonController.deployToken(config)` with the token config,
   reward addresses and a set of reward assets with weights. The controller adds itself and the caller
   as token admins and creates the token via `tokenDeployer`: TokenDeployer deploys the token, the whole supply
   is minted to the controller, and the token's `creator()` is the caller of `deployToken`. The controller checks that
   it received exactly the full supply (`INITIAL_SUPPLY_X18`), otherwise `TokenSupplyMismatch`. Next it
   registers the meme in the vault, initializes the Meme/USDG pool with
   the hook and a zero LP fee, and mints the whole supply into a single-sided position above the starting price.
   The position NFT stays with the controller. Events: `TokenCreated`, `PoolCreated`, `PositionCreated`.
   If `config.devBuy.memeAmount` is non-zero, right after creating the pool the controller opens
   the PoolManager via `unlock` and, in `unlockCallback`, buys that amount of the meme (exact output) for
   `msg.sender`: USDG is pulled from the caller straight into the PoolManager (an approval to the
   controller is required, not more than `maxPairIn`), and the meme goes to the caller. All in the same transaction, so nobody
   trades before the creator. The hook charges the controller the meme's base rate without the anti-snipe
   surcharge. The limit is 80% of the supply. Event `DevBuy`.

2. **Trading.** Traders go through any Uniswap V4 router, e.g. the Universal Router.
   On every swap the hook takes a USDG fee: in `beforeSwap` if the USDG amount is specified
   by the user, otherwise in `afterSwap`. The base rate, from 1% to 5%, is chosen by the creator in
   `rewardsConfig.swapFeeBips`. For the first `DYNAMIC_FEE_THRESHOLD` seconds (30) after pool creation the
   rate is higher: 80% at creation, decaying linearly to the base rate (`feeBipsAt`). The exception is
   the controller's own swaps, i.e. the dev buy: they always pay the base rate. The hook records the creation time
   itself in `afterInitialize` of the first pool of a meme created by the controller
   (`poolTimestamps`, event `PoolTimestampRecorded`); pools created by anyone else, and
   later pools of the same meme, do not touch it. The hook receives the rate from the
   controller via `notifyPoolCreated(meme, feeBips)` — once per meme
   (`PoolAlreadyNotified`) and only in the block where the pool is created (`PoolNotCreatedInThisBlock`). Therefore
   neither the controller nor the protocol owner (the hook has no owner) can enable 80% on an already
   trading pool, restart or shift the window. The hook only computes the fee amount and
   knows nothing about how it is split. The fee accumulates in the hook as ERC6909 claims, separately per
   meme.

3. **Fee payout.** At the end of each swap the hook tries to pay out everything accumulated for that
   meme in one go to the vault and immediately calls `Vault.notifyFees(meme, amount)`. The vault does the split:
   `protocolShareBps` (2500 bps by default, a quarter) of the amount, including the window surcharge, is transferred at once
   to `protocolRecipient`, the rest stays with the meme. At a 1% rate that is 0.25% and 0.75% of the
   trade, at 5% — 1.25% and 3.75%. Rounding is down, in favour of the meme's vault. If `protocolRecipient` has code
   (the referrals contract), the vault calls `IProtocolFeeReceiver.notifyFee(meme,
   toProtocol)` after the transfer; an EOA is not notified. A recipient contract must implement this interface,
   otherwise `notifyFees` reverts, and with it the hook's whole payout (the swap goes through, the fee stays
   accrued in the hook). The share and recipient are vault-owner settings (`setProtocolShare`,
   `setProtocolRecipient`), global: a new value applies from the next payout for all
   memes. While the share is > 0 and no recipient is set, `notifyFees` reverts with
   `ProtocolRecipientNotSet` and the fee waits in the hook.
   If the PoolManager has no USDG yet, as on a fresh deploy, the payout reverts inside try/catch,
   the swap goes through and the amount stays accrued. Anyone can collect it later via
   `HookManager.distribute(meme)`, or the next swap will do it.

4. **Vault accounting.** `notifyFees` splits the part remaining after the protocol as follows: the share
   `creatorShareBps` to the creator (0..50%, `MAX_CREATOR_SHARE_BPS`, above that `InvalidCreatorShare`;
   chosen by the creator in `deployToken`, `config.vaultCreatorShareBps`), the rest pro rata to the meme's current
   stakers. If there are no stakers, everything goes to the creator. A staker earns only on fees
   that arrived while they were staked. Accruals accumulate in USDG inside the open epoch.

5. **Conversion.** An epoch is closed by a threshold, there is no keeper. As soon as the USDG accumulated in the open
   epoch reaches `conversionThreshold`, `notifyFees` in the same transaction calls
   `Vault.convertFees(meme)`: the vault swaps all the epoch's USDG for reward assets by weight through the swap
   adapter and closes the epoch. Gas is paid by the swapper whose swap crossed the threshold. The conversion is wrapped in
   try/catch: if the swap in the adapter fails, the accrual stays, `ConversionFailed` is emitted, and
   the next attempt is no sooner than `CONVERSION_COOLDOWN`.
   **Gas for conversion.** Because of try/catch, running out of gas inside the conversion would not revert the swap, and
   gas estimation in wallets, routers and 0x would find a limit at which the conversion always silently
   dies of gas. So before paying out, the hook asks the vault for `conversionGasRequired(meme)`
   and reverts the swap with `NotEnoughGasForConversion(needed, available)` if there is less gas than needed,
   taking the 63/64 rule over three nested calls into account. Gas estimation then raises the limit by itself.
   The vault's answer is capped at `MAX_CONVERSION_GAS` (3M; the real vault maximum is 2.55M with 8
   assets): a vault with an inflated estimate cannot block trading, and a conversion that lacks gas
   fails inside try/catch and the fee stays accumulated. Both vault calls from a
   swap are gas-limited: the estimate query — `CONVERSION_QUERY_GAS` (100k), the payout — exactly the
   estimate plus `PAYOUT_GAS_OVERHEAD`. A vault that burns all the gas it receives makes a swap more expensive
   by no more than that value, but does not block it.
   The requirement applies to **every** swap of a meme with auto-conversion enabled, not only the one
   crossing the threshold: otherwise a swap estimated while the threshold was far would fail if, before it was included
   in a block, other trades had almost reached the threshold. Only the gas actually used is paid.
   Budget: `CONVERSION_BASE_GAS` (150k) + `CONVERSION_GAS_PER_SWAP` (300k) per asset
   except USDG, plus the hook's `PAYOUT_GAS_OVERHEAD` (300k; the heaviest payout with a split in the vault is ~240k). Measured on Robinhood Chain: a conversion
   into two assets through Uniswap V3 costs 570k–660k. With threshold 0 (auto-conversion off)
   there is no requirement. Anyone can call `convertFees`
   under the same threshold condition, as insurance if the automation gets stuck. Price protection lives in the
   adapter: the minimum output is computed from the pool TWAP minus the pool fee and a per-asset
   tolerance. Only after the epoch is closed do the accruals become claimable, already in the purchased assets.

6. **Stakers and creator.** Meme holders stake via `stake` or `stakeWithPermit`,
   claim rewards via `claim` in full or for selected assets, unstake via `unstake`
   keeping their rewards, or `exit` for everything at once. `unstake` works even when paused.
   The creator gets their share via `claimCreatorRewards`; anyone can call it, the funds always
   go to the current creator. The creator role is transferred in two steps: `transferCreator`,
   `acceptCreator`.

### Referrals

`GmoonReferrals` is set as the protocol recipient in the vault (`Vault.protocolRecipient`) and receives the whole
protocol share of fees. It accepts `notifyFee` from the vault (`setVault`) and also from old-scheme
hooks that paid the protocol share themselves (`isFeeSource`).

1. A referrer registers a code: `registerCode("code")`, 1..32 characters `[a-z0-9_-]`, the first to claim
   a code owns it forever, one referrer may have several codes.
2. A user binds with `useCode("code")`. Once and forever, you cannot refer yourself. Any number of
   users can be referred by one code.
3. `deployToken` calls `registerToken(meme, msg.sender)`: the meme is attached to the **deployer's**
   referrer at deploy time. A binding made later does not affect memes already created.
4. On each payout the vault transfers the protocol share and calls `notifyFee`. The meme's referrer
   is credited `amount * refShareBips / 10000` (by default 25% of the protocol share, rounded
   down in favour of the protocol). Memes without a referrer and unknown memes go entirely to the protocol.
5. Referrer payouts: `claim()` by the referrer or `distribute(address[])` by anyone (funds go only to
   the referrers themselves, zeros and duplicates are skipped).
6. Protocol: `withdrawProtocolFees(to)` (owner only) withdraws `balance - totalPending`, i.e.
   everything except the debt to referrers.

Pause (`setPaused`) stops only `claim` and `distribute`. Accrual, code registration and
`withdrawProtocolFees` always work, so a pause neither breaks fee payouts nor blocks the
protocol. Invariant: `balanceOf(referrals) >= totalPending`, checked on every accrual;
`notifyFee` without an actual transfer reverts with `BalanceTooLow`. `refShareBips` is global and
applies to fees that arrive after the change.

### Swap adapter: why Uniswap V3

The vault knows only the `ISwapAdapter` interface: give USDG, receive the asset to its own balance, the output is
measured by the balance difference. The vault does not care which DEX is behind it; the adapter is replaced via
`setSwapAdapter`. The current `UniswapV3SwapAdapter` implementation was chosen for three reasons:

- **Reward-asset liquidity.** USDG/asset pools for tokenized stocks and wBTC live in
  Uniswap V3. The protocol's V4 pools are only Meme/USDG with the Gemoon hook and are not
  meant for reward assets.
- **Built-in oracle.** There is no keeper, so the minimum output is computed on-chain and a reference
  price is needed. A V3 pool has one out of the box: `observe` returns accumulated ticks, from which a TWAP over the
  window is computed. V4 has no built-in oracle; we would need an oracle hook on every pool or
  depend on Chainlink, which may not exist for tokenized stocks.
- **Factory instead of a pool address.** `setRoute(asset, fee, slippage)` asks the factory for the pool by the
  triple (USDG, asset, fee tier). The owner cannot substitute an arbitrary contract for a pool, and the
  `fee` is guaranteed to match the pool's real fee, which the adapter subtracts from the quote.
  The factory address is immutable.

If on the target network USDG/asset liquidity turns out to be elsewhere, another adapter is written for
the same interface with its own price source; the vault and its accounting do not change.

### Administration

- **Token.** Admins change the image and description; the admin set is changed through
  `GemoonController.changeAdmin`. Only an admin of that meme can call it: the meme itself sees the
  controller as the caller, which is also its admin, so the controller checks the caller.
- **Vault.** The owner manages the asset allowlist, conversion threshold, swap adapter, pause, and also
  the fee split: the protocol share (`setProtocolShare`, 0..100% of the fee, immediately for all
  memes) and its recipient (`setProtocolRecipient`). After `TransferToGovernance` the vault owner
  is a Safe without a timelock, so a change of the protocol share takes effect without delay.
  Threshold 0 disables automatic conversion. Pause stops staking, claiming and conversion,
  but not fee intake or unstaking. `rescueERC20` for tokens sent by mistake.
- **Swap adapter.** The owner sets a route per asset: the fee tier of the USDG/asset pool from the
  Uniswap V3 factory and the tolerance from the TWAP in bps, no more than 10%. The TWAP window is shared, from 1 minute to an hour.
  `sweep` withdraws leftovers from the adapter; between swaps it holds nothing.
- **Hook.** No owner, no admin functions (version 3). The vault, controller and fee rates are set
  in the constructor and never change; the fee split lives in the vault. The vault and controller are proxies, their
  upgrades do not affect the hook; a different vault or controller address requires a new hook.
- **Controller.** The owner changes the hook, vault, PositionManager with Permit2, the referrals contract
  (`setReferrals`, zero disables registration), the TokenDeployer (`setTokenDeployer`) and withdraws
  funds held by the controller itself.
- **TokenDeployer.** No owner, no admin functions: the controller is set in the constructor, the token code
  is baked into the bytecode. The token code of new memes changes by replacing the deployer: `make deploy-token-deployer`
  (script `DeployTokenDeployer`) deploys a new TokenDeployer with the token code of the current commit and, if the
  broadcaster owns the controller, immediately calls `setTokenDeployer`; otherwise it prints the call for the
  owner (after `TransferToGovernance` — a call from the Safe). The controller is not upgraded,
  already created memes do not change. `setTokenDeployer` accepts only a deployer bound to this
  controller (`TokenDeployerControllerMismatch`). The new token code is trusted: the controller
  only checks that the whole supply reached it; everything else (e.g. the absence of extra minting)
  is reviewed before `setTokenDeployer` just like a controller upgrade. The new token's constructor
  must mint the whole `INITIAL_SUPPLY_X18` to the caller of `deployToken` (the controller).
- **Referrals.** The owner changes `refShareBips` (at most 100%), the vault (`setVault`), hook, controller, pauses payouts
  to referrals, withdraws the protocol part and rescues foreign tokens (`rescueERC20`, except the
  pair token).
- **Upgrades.** Each proxy is upgraded through its own ProxyAdmin with the scripts `ProxyVaultUpgrade`,
  `ProxyGemoonControllerUpgrade`, `ProxyReferralsUpgrade`. **HookManager is not upgradeable:** it has
  no proxy or ProxyAdmin; the code and fee rates (`TOTAL_FEE_BIPS`, `MAX_FEE_BIPS`, the meme
  rate bounds) are fixed at deploy time. There is no fee split in the hook: everything goes to the upgradeable vault.
  The hook has no owner: fees cannot be redirected or the controller changed, and nobody can raise the fee
  of a live pool. A new hook version is a new contract at a new address: pools of
  already created memes stay on the old hook, only new memes move to the new one. Changing the hook in an
  existing deployment: `make deploy-hook` (script `DeployHook`: deploy, `Vault.setHook`,
  `Controller.setHook`, wiring check), then `make deploy-lens` (the Lens stores the hook address).
  The vault and referrals keep accepting fees from all hooks assigned earlier (`isHook`),
  the controller remembers each meme's hook (`hookOf`), and GemoonLens reads each meme through its hook.
  An old hook cannot be revoked: otherwise its memes' fees would get stuck. `VerifyWiring`
  checks that the wiring is intact (referrals — if `REFERRALS_PROXY_ADDRESS` is set).

### GemoonLens: batch reads for the frontend

The frontend reads state not with hundreds of `eth_call`s but with one call per screen. All functions are `view`,
take an array of addresses, the result is in input order, duplicates are allowed.

| Function | Returns | Screen |
|---|---|---|
| `tokens(address[] memes, address account)` → `TokenView[]` | metadata, market, public vault state, the position of `account` | Explore, token page, staking |
| `markets(address[] memes)` → `(TokenMarket[], uint256[] staked)` | market and `totalStaked` only, no strings | periodic price refresh |
| `totalStaked(address[] memes)` → `uint256[]` | `vault.totalStaked` for each | hero statistics |
| `quotes(address[] assets, uint256 amountIn)` → `(uint256[], bool[] ok)` | how much of an asset `amountIn` USDG buys at the current adapter | reward valuation in USD |
| `protocol()` | USDG (address, decimals, symbol), adapter, conversion threshold and cooldown, hook, PoolManager | once per session |
| `creator(address meme)` → `(creatorOf, assets[], accrued[])` | the vault's creator and their accrued share | Creator rewards block |
| `creators(address[] memes)` → `CreatorRewards[]` | the same for a list of memes, with an `ok` flag | a creator with several memes |

A meme's pool is derived the same way as in the controller: `PoolKey{sort(USDG, meme), fee 0, tickSpacing 200,
hooks = HookManager}`, `poolId = keccak256(abi.encode(key))`, the price from the PoolManager's slot0 (same as
`StateView.getSlot0`). USDG and the adapter are read from the vault on every call.

**Flags and zeros.** No failure breaks the batch: a failed read yields zero, and where zero is a
valid value, also a `false` flag.

| Field | When `false` / zero | What to show |
|---|---|---|
| `meta.ok` | the address has no code or `totalSupply()` did not answer | no token; only `meme`, `market.poolId`, `market.memeIsToken0` are filled |
| `meta.decimalsOk` | no `decimals()` or the answer does not fit in uint8 | `–` instead of amounts |
| `meta.creatorOk` | no `creator()` (old tokens) or the answer is not an address | `–` |
| `meta.socialOk` | no `getSocialMedia()` or at least one of the four strings is invalid; then all four are empty | do not show links |
| `name`, `symbol`, `description`, `image` | no function, revert, or the answer is not an ABI string (e.g. a `bytes32` name) | empty string |
| `market.sqrtPriceX96 == 0` | the pool is not created or not initialized | no price |
| `market.burnedOk` | at least one `balanceOf` (0x0 or 0xdEaD) did not answer; `burned` = 0 | `–` |
| `market.pendingFeesOk` | `hook.pendingFees` did not answer | `–` |
| `vault.registeredOk` | `vault.isRegistered` did not answer; `registered = false` means "unknown", not "no vault" | `–` |
| `vault.assets[i].symbolOk`, `decimalsOk` | the asset has no `symbol()`/`decimals()` or the answer is invalid | `–` |
| `vault.registered == false` with `registeredOk` | the meme has no vault (not created through the controller) | all vault fields and vault calls are skipped |
| `position` | `account == address(0)` | zeros, per-account calls are not made |
| `quotes` `ok[i]` | the adapter has no route, no adapter set, or it failed | `–`; USDG is always 1:1 with `ok = true` |
| `creators` `ok` | the vault did not answer `creatorOf` or `creatorAccrued` | `–` |

`vault.registered == true` is the only sign that an address really is a meme of this
protocol: the Lens reads any address it is given. The `description`/`image` strings are set by the
meme creator; the frontend must escape them.

**Batch size limits** (cold-call gas, like a real `eth_call`; nodes usually cap it at 50M, the spec
requires 30M for `tokens` and 15M for `markets`/`totalStaked`):

| Call | Gas | Source |
|---|---|---|
| `tokens(70, account)`, 5 assets, stake, socials, 280 B description, 120 B image | 18.4M | `GemoonLensGasTest` |
| `tokens(70, 0x0)`, same | 14.5M | `GemoonLensGasTest` |
| `tokens(70, account)`, 70 real devnet memes | 14.2M | `GemoonLensDevnetForkTest` |
| `tokens(70, account)`, 5 assets, description and image **2 KB each** | 40.8M | `GemoonLensGasTest` |
| `tokens(50, account)`, strings of 2 KB | 27.5M | `GemoonLensGasTest` |
| `markets(200)` | 6.0M | `GemoonLensGasTest` |
| `totalStaked(200)` | 1.4M | `GemoonLensGasTest` |
| `creators(70)`, real devnet memes | 2.3M | `GemoonLensDevnetForkTest` |

Recommended chunks: `tokens` — **50 addresses** (stays within 30M even with 2 KB strings; with typical
string lengths 70 fit with headroom), `markets`/`totalStaked` — 200. With 2 KB strings
about 60% of a token's gas goes to reading the strings from storage (128 cold SLOADs); the Lens cannot
reduce this. Full `forge test --gas-report` output: `docs/gemoon-lens-gas-report.txt`.

**Deployment.** `make deploy-lens` (`script/DeployLens.s.sol`): takes `CONTROLLER_PROXY_ADDRESS`,
`VAULT_PROXY_ADDRESS`, `HOOK_ADDRESS`, `POOL_MANAGER` and `LENS_SALT` (default 0) from
`env/<ENVIRONMENT>.mk`, checks that the controller, vault, hook and PoolManager point at each other, and deploys through the
CREATE2 factory `0x4e59b44847b379578588920cA78FbF26c0B4956C`. The address depends only on the salt,
arguments and bytecode, so after a fork reset the same command brings the Lens up at the same address,
and if it already exists, sends nothing. The bytecode changes with any contract edit or compiler
setting change — and then the address changes too. A regular transaction, no `anvil_*`.

Devnet (chain 1488): `0xf5e9342Ffe373D9a3B4a986870cbC6E66dDCbF15`, block 84351591
(tx `0xefea4f43376ea117553188d801761d721830cbe6c80b8597c4dfa8aa54940e10`), `LENS_ADDRESS` in `env/dev.mk`.
The first version, without socials, asset flags and `creators`, remains at `0x43A7E53e63A834c3FB7FcE55d84F2879cB5A82Ec` (block 84351590);
it has a different ABI.

## Protocol parameters

| Parameter | Value | Where |
|---|---|---|
| Meme supply | 1 000 000 000 tokens, 18 decimals | `INITIAL_SUPPLY_X18`, `IGemoon.sol` |
| Initial market cap | 20 000 USDG | `INITIAL_MARKET_CAP`, `IGemoon.sol` |
| Initial price | 50 000 whole memes per 1 whole USDG (supply / initial market cap); the decimals of both tokens are read when the pool is created, the token0/token1 order does not affect the price | `PRICE_PER_TOKEN`, `IGemoon.sol`; `_configurePool`, `Gemoon.sol` |
| Tick spacing | 200 | `TICK_SPACING`, `IGemoon.sol` |
| Pool LP fee | 0, the fee is taken by the hook | `Gemoon.sol` |
| Meme fee | 1%..5%, chosen by the creator in `deployToken` | `MIN_MEME_FEE_BIPS`, `MAX_MEME_FEE_BIPS`, `HookManager.sol` |
| Protocol share | a quarter (2500 bps) of every fee payout (including the anti-snipe surcharge), the rest to the meme; stored in the vault, the vault owner changes it globally, at most 100% | `protocolShareBps`, `setProtocolShare`, `Vault.sol` |
| Fallback fee | 1.25% for memes the controller did not report; set in the constructor and never changes | `TOTAL_FEE_BIPS`, `HookManager` constructor |
| Anti-snipe fee | 80% at pool creation, linearly to the base rate over 15 seconds; dev buy pays the base rate | `MAX_FEE_BIPS`, `DYNAMIC_FEE_THRESHOLD`, `HookManager.sol` |
| Dev buy | up to 80% of the supply, exact output, in the `deployToken` transaction | `MAX_DEV_BUY_BIPS`, `MAX_DEV_BUY_X18`, `IGemoon.sol` |
| Creator share | 0..50% of every vault fee (`MAX_CREATOR_SHARE_BPS` = 5000), chosen by the creator in `deployToken`, does not change after registration | `VaultInfo.creatorShareBps`, `creatorShareBpsOf`, `MAX_CREATOR_SHARE_BPS`, `Vault.sol` |
| Referral share | 25% (2500 bips) of the protocol share of a referred user's meme, set by the owner, at most 100% | `refShareBips`, `Referrals.sol` |
| Referral code | 1..32 characters `[a-z0-9_-]` | `MAX_CODE_LENGTH`, `Referrals.sol` |
| Reward assets per vault | up to 8 | `MAX_ASSETS`, `Vault.sol` |
| Epoch conversion threshold | set by the owner in USDG units, 0 disables the automation | `setConversionThreshold`, `Vault.sol` |
| Cooldown after a failed conversion | 1 minute | `CONVERSION_COOLDOWN`, `Vault.sol` |
| Gas for conversion | 150k + 300k per asset except USDG, plus 300k for the payout (split in the vault, transfer to the protocol and referrals `notifyFee`); any swap of a meme with threshold > 0 and less gas than that reverts, only the gas actually used is paid | `CONVERSION_BASE_GAS`, `CONVERSION_GAS_PER_SWAP`, `Vault.sol`; `PAYOUT_GAS_OVERHEAD`, `HookManager.sol` |
| Per-asset TWAP tolerance | set by the adapter owner, 1% recommended, ceiling 10% | `setRoute`, `MAX_SLIPPAGE_BPS`, `UniswapV3SwapAdapter.sol` |
| TWAP window | 1 minute … 1 hour, 10 minutes by default in the script | `setTwapWindow`, `UniswapV3SwapAdapter.sol` |

Fee split after the anti-snipe window with default parameters, as a percentage of the trade amount:

| Part | Rule | At 1% rate | At 5% rate |
|---|---|---|---|
| Meme fee | `swapFeeBips`, 100..500 | 1% | 5% |
| Protocol | 25% of the fee (`protocolShareBps` = 2500) | 0.25% | 1.25% |
| of which to the referrer | 25% of the protocol share (`refShareBips` = 2500), only if the meme has a referrer | 0.0625% | 0.3125% |
| Meme vault | the remaining 75% of the fee | 0.75% | 3.75% |
| of which to the creator | the vault's `creatorShareBps`, 0..50% | 0..0.375% | 0..1.875% |
| of which to stakers | the rest, at least half of the vault | 0.375..0.75% | 1.875..3.75% |

A detailed map: where exactly each percentage is computed (file, function, line), how it is rounded,
what is fixed forever and what the owner can change — `docs/FEES.md`.

## Deployment

Environment variables live in `env/<ENVIRONMENT>.mk`: `env/dev.mk` (devnet) and `env/prod.mk`
(mainnet), neither is committed. The environment is selected by the required variable `ENVIRONMENT=dev|prod`:
`make <target> ENVIRONMENT=dev`, `ENVIRONMENT=prod ./deploy.sh`. Without it, or with another value, the
Makefile and the scripts in `tools/scripts` fail with an error. Only `clean`, `compile`, `unit-tests`
and `prepare-abi` work without `ENVIRONMENT`: the latter writes the addresses into the ABI README straight from `env/dev.mk` and
`env/prod.mk` (dev and prod columns; a missing file or an empty value gives a dash). For prod the Makefile additionally requires non-empty
`GEMOON_OWNER` and `GEMOON_PROXY_ADMIN_OWNER`; the deployer is not substituted as the default owner there.
File format: `KEY=VALUE` lines without quotes or `$`, readable by both make and bash. There must be no `.env`
file: forge loads it by itself, and its values would leak into any environment.
An empty value counts as unset.

| Variable | Purpose |
|---|---|
| `RPC`, `PRIVATE_KEY` | Network and deployer key. |
| `POOL_MANAGER`, `POSITION_MANAGER`, `PERMIT2` | Uniswap V4 on the target network. |
| `USDG_ADDRESS` | Pair token of all pools. |
| `PROTOCOL_FEE_RECIPIENT` | Manual scenarios only (`set-protocol-recipient RECIPIENT=…`). A full deploy sets the referrals contract as the vault's recipient. |
| `REFERRALS_REF_SHARE_BIPS` | Referrer's share in bips of the protocol share, 2500 by default. |
| `HOOK_TOTAL_FEE_BIPS` | The hook's fallback fee for memes without their own rate (bips of the trade), 125 by default. |
| `VAULT_PROTOCOL_SHARE_BPS` | The protocol share in the vault (bps of the fee), 2500 by default. Later changed with `make set-protocol-share`. |
| `VAULT_CONVERSION_THRESHOLD` | Epoch conversion threshold in USDG units, required. 0 disables the automation. |
| `UNISWAP_V3_FACTORY` | Uniswap V3 factory with USDG/asset pools, required. |
| `ADAPTER_TWAP_WINDOW`, `ADAPTER_MAX_SLIPPAGE_BPS` | The adapter's TWAP window (600 s by default) and route tolerance (100 bps by default). |
| `DEPLOY_ASSETS_FILE` | Reward asset list, `tools/scripts/robinhood-assets.csv` by default. |
| `VAULT_ALLOWED_ASSETS` | Extra assets without a route (e.g. USDG). |
| `VAULT_SWAP_ADAPTER`, `SWAP_ADAPTER`, `LENS_ADDRESS` | Filled in by `deploy.sh`. |
| `GEMOON_OWNER`, `GEMOON_PROXY_ADMIN_OWNER` | Final owner of the contracts and owner of the ProxyAdmin. The deployer by default. |
| `*_PROXY_ADDRESS`, `*_PROXY_ADMIN_ADDRESS` | Filled in from the deploy logs, needed by upgrades and verification. |
| `TOKEN_DEPLOYER_ADDRESS` | The controller's current TokenDeployer. Filled in by `deploy.sh`; after `make deploy-token-deployer`, enter the printed `TOKEN DEPLOYER`. Needed by `verify-token-deployer` and contracts-map; the controller itself reads the deployer from its storage. |

```sh
# All targets below, except tests and build, run with ENVIRONMENT=dev|prod, e.g. make verify-wiring ENVIRONMENT=dev
ENVIRONMENT=dev ./deploy.sh [--simulate] [--no-lens]  # full deploy: all contracts, assets and routes, env/dev.mk, Lens, report
make deploy-gemoon            # the same without writing addresses and Lens: Vault, Controller, Referrals, Hook, Adapter, assets, ownership
make verify-wiring            # read-only check that the contracts point at each other
make upgrade-vault-proxy      # new vault implementation
make upgrade-controller-proxy # new controller implementation
make deploy-token-deployer    # new token code for new memes: TokenDeployer + setTokenDeployer, the controller is untouched
make deploy-vault             # a standalone vault, if the existing one must be replaced
make deploy-swap-adapter      # UniswapV3SwapAdapter for an existing vault
make deploy-referrals         # GmoonReferrals behind a proxy (USDG_ADDRESS, CONTROLLER_PROXY_ADDRESS, HOOK_ADDRESS)
make wire-referrals           # controller.setReferrals + referrals.setVault + vault.setProtocolRecipient(referrals) + check
make upgrade-referrals-proxy  # new referrals implementation
make set-protocol-recipient [RECIPIENT=0x..] [DRY_RUN=1]  # recipient of the protocol share in the vault, referrals by default
make set-protocol-share SHARE=2500                        # protocol share in the vault, bps of the fee
make distribute-referrals REFERRERS="[0x..,0x..]"  # pay out referrals
make withdraw-protocol-fees TO=0x..                # withdraw the protocol part
make set-asset-allowed VAULT=0x.. FILE=tools/scripts/robinhood-assets.csv [DRY_RUN=1]  # reward-asset allowlist
make set-route ADAPTER=0x.. FILE=tools/scripts/robinhood-assets.csv [DRY_RUN=1]        # asset routes on the adapter
make create-tokens [COUNT=10] [DRY_RUN=1]  # devnet only: COUNT memes with random names, rates and rewards
make deploy-lens              # GemoonLens via CREATE2, repeatable (make simulate-lens to dry-run)
make contracts-map [ARGS="--depth 2 --exclude allowedAssets"]  # read-only: contract graph for contracts-map
```

`upgrade-controller-proxy` requires `GEMOON_OWNER` explicitly: `reinitialize` reassigns the controller's owner.

### Contract graph for contracts-map

`make contracts-map ENVIRONMENT=dev` (or `ENVIRONMENT=dev node tools/contracts-map/generate.mjs`)
writes `deploy-logs/contracts-map-<ENVIRONMENT>.json`: the list of the network's contracts and the links between them —
which contract points at which, and through which method. Read-only (`eth_call`,
`eth_getStorageAt`, `eth_getCode`); the RPC URL does not end up in the file. ABIs are taken from `out/`, so run
`forge build` first.

Starting addresses come from `env/<ENVIRONMENT>.mk`: controller, hook, vault, referrals, adapter, Lens, ProxyAdmin,
timelock, Uniswap, USDG, owners. For every contract the EIP-1967 slots (`implementation`,
`admin`, `beacon`), `owner()` and all argument-less view functions whose result contains an address are read
(including tuples and arrays). Every non-zero address is a link; an unknown address becomes a node and
is explored up to `--depth` steps from the starting ones (1 by default). An implementation gets the ABI of its proxy.

| Flag | Purpose |
|---|---|
| `--out <file>` | Where to write, `deploy-logs/contracts-map-<ENVIRONMENT>.json` by default. |
| `--rpc <url>` | A different RPC instead of `RPC` from the env file. |
| `--depth <n>` | How many steps from the starting addresses new nodes are added. |
| `--exclude <regex>` | Skip getters with a matching name, e.g. `allowedAssets` (65 reward assets). |
| `--no-abi` | No ABI in the file (the file is several times smaller, but the card will have no methods). |
| `--no-eoa` | No addresses without code (EOA owners). |

Format (`format: "contracts-map/links@1"`):

```json
{
  "format": "contracts-map/links@1",
  "environment": "dev",
  "chainId": 1488,
  "block": 84314009,
  "generatedAt": "2026-10-24T12:00:00.000Z",
  "contracts": [
    { "address": "0x…", "name": "Vault", "kind": "proxy", "source": "env:VAULT_PROXY_ADDRESS",
      "artifact": "Vault.sol:Vault", "abi": [] }
  ],
  "links": [
    { "from": "0x…", "to": "0x…", "method": "swapAdapter", "signature": "swapAdapter()", "path": "", "kind": "call" },
    { "from": "0x…", "to": "0x…", "method": "implementation", "signature": "eip1967.implementation", "path": "", "kind": "slot" }
  ]
}
```

Contract `kind`: `proxy`, `implementation`, `contract`, `eoa`. `path` is the location of the address inside
the result (`protocol.hook`, `[3]`). The file is imported into contracts-map with the "Import" button (file),
"Import URL" or the `?import=<url>` link, and is merged into the current map.

Connecting referrals to an existing deployment: `deploy-referrals` → proxy address into
`REFERRALS_PROXY_ADDRESS` → `wire-referrals` from the owner of the controller, vault and referrals. From that
moment the vault sends the protocol share to referrals instead of the previous `PROTOCOL_FEE_RECIPIENT`;
what the hook accumulated before the switch will go there too on the next payout. The controller must be upgraded first
(`upgrade-controller-proxy`, `GEMOON_VERSION` = 2).

**Migrating from a hook that split the fee itself.** Old-scheme hooks (with `PROTOCOL_SHARE_BIPS` and
`protocolRecipient`) hand the vault only its part, while the new vault takes the protocol share from every
`notifyFees`. If you upgrade the vault under live pools of an old hook and enable `protocolShareBps` > 0,
the old hook's memes will pay the protocol twice. Order: upgrade the vault (the share after the upgrade is 0,
no recipient set, behaviour unchanged) → deploy the new hook and switch to it with
`controller.setHook` / `vault.setHook` → only if no old pools with volume remain, enable the share with
`make set-protocol-share`. Otherwise the protocol share stays 0 while the old pools trade.

After deployment `GEMOON_OWNER`, if it is not the deployer, calls `acceptOwnership` on Vault, GmoonReferrals
and UniswapV3SwapAdapter. The controller belongs to it at once; the hook and TokenDeployer have no owner. The deploy report
(`deploy-logs/deploy-<time>-report.md`) lists the addresses, the purpose of the contracts, the wiring,
owners, the result for each asset and the remaining steps. Details — `docs/DEPLOY.md`.

The project token (`ProjectToken`) is deployed separately, without the controller:
`forge script script/DeployProjectToken.s.sol:DeployProjectToken --rpc-url $RPC --private-key $PRIVATE_KEY --broadcast`
with the variables `PROJECT_TOKEN_NAME`, `PROJECT_TOKEN_SYMBOL`, `PROJECT_TOKEN_URI` (optional), `PROJECT_POOL_FEE`
(10000 = 1% by default), `PROJECT_TOKENS_PER_PAIR` (50 000 tokens per 1 USDG by default, a $20k cap), plus
`USDG_ADDRESS`, `POOL_MANAGER`, `POSITION_MANAGER`, `PERMIT2` from the env file.

The hook is deployed via CREATE2 without a proxy: the contract address must carry the permission bits.
The script mines the salt itself and verifies the result. The network must have the deterministic
CREATE2 deployer `0x4e59b44847b379578588920cA78FbF26c0B4956C`.

**New token code without redeploying the controller.** Edit `Token.sol` (or a new token contract in
`TokenDeployer`) → `make simulate-token-deployer ENVIRONMENT=…` → `make deploy-token-deployer ENVIRONMENT=…`
from the controller owner → the printed `TOKEN DEPLOYER` into `TOKEN_DEPLOYER_ADDRESS` → `make verify-wiring`
and `make -f verify.mk verify-token-deployer`. Memes created after `setTokenDeployer` get the new code,
earlier ones keep their own. If the controller is owned by a timelock, the script only deploys the TokenDeployer and
prints the `setTokenDeployer` calldata for the Safe. Details — `docs/DEPLOY.md`, section 4.

## Tests

```sh
make unit-tests     # everything except test/fork
make devnet-tests ENVIRONMENT=dev  # devnet fork, needs DEVNET_RPC in env/dev.mk
forge test          # all together; fork tests run only if DEVNET_RPC is in the environment
```

- `test/VaultTest.sol`, `test/VaultUpgradeTest.sol`: the vault, including threshold auto-conversion,
  cooldown, invariants and upgrade.
- `test/VaultPrecisionTest.sol`: rounding of a small stake's share next to whales (800M /
  100M / 1 meme) with production parameters, fuzz on the rounding-loss boundaries.
- `test/UniswapV3SwapAdapterTest.sol`: the adapter against a mock pool with configurable TWAP and spot,
  fuzz on the tolerance boundary, integration with the real vault.
- `test/DeployGemoonTest.sol`: deploy script logic, wiring, ownership, hook address bits,
  `onlyController` access to the hook, TokenDeployer in the wiring and replacing the deployer with the
  `DeployTokenDeployer` script (from the controller owner and from an outsider).
- `test/TokenDeployerTest.sol`: TokenDeployer on its own — only its own controller can call it
  (fuzz over the caller), the whole supply goes to the caller, the token carries the passed metadata,
  creator and `feeInfo` (fuzz).
- `test/DeployGemoonFullTest.sol`: a full deploy with all five contracts, asset file parsing,
  allowlist and routes skipping non-convertible assets (fuzz on the invariant
  "allowed ⇔ has a route"), ownership, report, `run()` from the environment.
- `test/ControllerDeployTokenTest.sol`: in-process end-to-end. The real PoolManager,
  PositionManager and Permit2 from `lib/`, deploy through the script, `deployToken`, swaps, the fee path
  to the protocol and vault, fuzz on the invariant "protocol + vault + accrued = the whole fee taken",
  swaps in the raised-fee window, dev buy: limit, slippage, base rate, fuzz on exact
  output, fee conservation and cost monotonicity. TokenDeployer: `setTokenDeployer` (access,
  foreign controller), changing the token code of new memes without a controller upgrade, fuzz on the invariant
  "the whole supply is in the position or `TokenSupplyMismatch` reverts" for any token code. `changeAdmin`:
  only a meme admin, fuzz on the invariant "a non-admin does not change the admin list".
- `test/HookFeeMath.sol`: the hook's fee curve, window boundaries, fuzz on range and monotonicity.
- `test/fork/ControllerDevnetForkTest.sol`: the same against devnet, an anvil fork of Robinhood Chain with
  its Uniswap V4, chain id 1488 (addresses from `env/dev.mk`). Broadcasts nothing, the fork lives inside the run. `DEVNET_BLOCK`
  pins the block, `OPERATOR_ADDRESS` sets a funded devnet account. Details in
  `docs/DEPLOY.md`, section "Devnet".
- `test/fork/DevBuyDevnetForkTest.sol`: end-to-end dev buy on devnet. The creator deploys a meme with a
  3%-of-supply buy; it checks that they hold exactly 3%, the fee at the base rate reaches the
  protocol and vault (immediately via `distribute` or with the next swap), the vault closes several
  epochs, the creator claims rewards as a staker and as a creator. The reward asset here is USDG itself,
  because the fork has no V3 pools for mock USDG. Same environment as the fork test above.

A fork test checks contract logic one to one with the network, but does not check transaction
infrastructure: the contract size limit, the block gas limit, the real CREATE2 deployer and the script's
`run()` path. That needs a live run of the script with broadcast.

## Documentation

- `docs/FEES.md`: the full fee map — what percentage goes where, in which contract and in which
  function it is computed, how it is rounded, what can be changed and by whom.
- `docs/DEPLOY.md`: deployment, upgrades, swap adapter, devnet.
- `docs/upgrades.md`: proxy upgrade rules.
- `indexing.md`: which events and view functions provide the Main Page and Staking Page metrics,
  the price, volume, reward and yield formulas for the indexer.
- `README-ABI.md`: ABI generation.
- `TEST-CASES.md`: all test cases as tables: what each test checks, which invariants it
  covers and the expected result. Updated together with the tests.

## Known issues and tech debt

- **The first swap does not pay out the fee.** The router transfers the swapper's USDG after `afterSwap`,
  and all pools are single-sided, so USDG appears in the PoolManager only from buys. The fee stays
  accrued until the next swap or `distribute`.
- **Liquidity is locked.** The position NFT sits on the controller, and the controller has no function to manage it.
  Liquidity can only be recovered through an upgrade.
- **`ProxyGemoonControllerUpgrade` calls `reinitialize`** under `reinitializer(GEMOON_VERSION)`, and the
  version equals 1, which `initialize` already took. An upgrade with the call fails until the version is
  raised. The vault upgrade skips the call if the version has not changed.
- **A controller upgraded from a version without TokenDeployer** (before the deployer was extracted into a separate
  contract, an external `Deployer` library created tokens straight from the controller) has no
  `tokenDeployer` after the upgrade: `deployToken` fails with `TokenDeployerNotSet` until the owner runs
  `make deploy-token-deployer`. `make verify-wiring` also fails until then. Memes of the old version are
  created by the controller itself (CREATE), new ones by TokenDeployer: memes can no longer be found
  by the controller's nonce, the source is the `TokenCreated` event.
- **`ProxyGemoonControllerUpgrade` takes the pair token from `USDG_ADDRESS`** and, before sending, checks it
  against the current hook's `pairToken()`: `reinitialize` overwrites the controller's pair token.
- **HookManager is no longer behind a proxy (version 2):** `initialize`/`reinitialize`, `ProxyHookUpgrade`,
  `make upgrade-hook-proxy` and `HOOK_PROXY_ADMIN_ADDRESS` were removed, `HOOK_PROXY_ADDRESS` was renamed to
  `HOOK_ADDRESS`. Hooks deployed earlier remain proxies: their pools are bound to the proxy address.
  To make them immutable too, the owner of the old hook's ProxyAdmin can renounce ownership
  (`renounceOwnership`) — this is irreversible.
- **HookManager without an owner (version 3):** `Ownable2Step`, `setVault`, `setController` and the
  `VaultUpdated`/`ControllerUpdated` events were removed; the constructor is `(poolManager, pairToken, vault,
  controller, feeBips)` without `owner`. The reason: scanners flagged "the owner can become the controller
  and start the 80% window" and "the owner can redirect fees". `TransferToGovernance` no longer
  touches the hook. Version 2 hooks (with an owner) stay as they are, their pools are bound to them.
- **`deployToken` changed its ABI in 2.0:** the strategy-name parameter was removed and the `devBuy` field was
  added to `DeployConfig`; integrations must be updated.
- **The controller is exempt from the anti-snipe fee** by `sender == controller` in the hook. This is
  safe as long as the controller swaps only in dev buy; any new controller function with a swap
  will also get the base rate.
- **`Vault.allowedAssets()` was added by a storage upgrade** (`s_allowedAssetList` at the end of the layout).
  For a vault proxy deployed earlier the list is empty after the upgrade, although `isAssetAllowed` is correct.
  The owner fills it by calling `setAssetAllowed(asset, true)` again for each already allowed
  asset; the call is idempotent.
- **The dead slot `_deployStrategies`** in the controller is kept for the proxy's storage layout.
- **Admins from `tokenConfig.admins` do not reach the meme.** `deployToken` reserves slots for them
  but does not fill them: a meme has only two non-removable admins, the controller and the caller (README-ABI and the
  `deployToken` NatSpec say otherwise), so there is nobody to replace via `changeAdmin` yet.
  Controllers before the `NotTokenAdmin` check let `changeAdmin` through from anyone: the meme sees the
  controller as the caller, and the controller is its admin. Fixing the admin copy is possible only together with
  a controller upgrade to a version with this check.
