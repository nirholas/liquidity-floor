# LiquidityFloor

**A liquidity commitment enforced per position: a fraction of what you add cannot leave until the pool's unlock date, and every provider is held to that fraction of their own stake rather than to a shared pool total.**

A production Uniswap v4 hook. It holds no funds and takes no fee for itself. No owner, no pause switch, no upgrade path.

- **Site:** https://liquidity-floor.pages.dev
- **Catalogue:** https://hookforge.pages.dev
- **Contract:** [`src/hooks/LiquidityFloorHook.sol`](src/hooks/LiquidityFloorHook.sol)
- **Licence:** MIT

## How it works

A new pool has a bootstrapping problem that is really a credibility problem. Swappers cannot tell the difference between liquidity that intends to stay and liquidity that will leave the moment the pool is quoted by an aggregator, and the second kind is indistinguishable from the first right up until it is gone. The published answers lock whole positions for a term.

That works and it is also badly mispriced: a provider who would happily commit a third of their capital for six months has to choose between committing all of it and committing none. Term locks therefore select for the providers least sensitive to the lock, which is not the same set as the providers most useful to the pool. This hook takes the commitment as a *fraction*.

Configure `floorBps` and `unlockTimestamp`, and thereafter each position may freely withdraw down to `floorBps` of everything it has ever added, with the remainder released at the unlock. Adding more liquidity raises your own floor proportionally, so topping up is never a trap: you keep the same ratio of free to committed capital that you signed up for. The design is deliberately race-free, which is the property that distinguishes it from a pool-wide minimum.

A floor expressed as "total pool liquidity must stay above X" is a bank run waiting to happen: it is satisfiable by whoever withdraws first and binding only on whoever is last, so rational providers race for the exit precisely when the pool most needs them. Holding each position to its own commitment removes the race entirely. Nothing another provider does can change what you are allowed to withdraw.

The hook takes no fee, holds no funds and has no privileged role. It cannot stop a provider from ceasing to quote, only from removing committed liquidity, and it does not restrict swaps at all. Position identity is the v4 position key: the address that called `modifyLiquidity` on the `PoolManager` (in practice a position manager or a router), the tick range, and the caller's salt.

Two providers sharing one position manager therefore share a commitment only if they also share a salt, which position managers do not do. Prior art: `LiquidityLock`, `Timelock Addition` and `LockingLiquidity` all lock positions wholesale for a term. Fractional, per-position, top-up-safe commitments are the contribution here.

## Prior art

LiquidityLock, Timelock Addition and LockingLiquidity all lock positions wholesale for a term. Fractional, per-position, top-up-safe commitments are the contribution here, along with the observation that a pool-wide minimum is a bank run rather than a floor.

## Where it does not help

The commitment binds the v4 position key, which is the address that called modifyLiquidity. A provider who routes through a position manager that pools many users under one key would share a commitment with them; every mainstream position manager gives each position its own key, but a custom router need not.

## Using it

Uniswap v4 removed `hookData` from `initialize`, so per-pool parameters arrive out of band. Fix them for a pool key whose pool does not exist yet, then initialize. Nobody can change them afterwards, including you.

```solidity
hook.configure(
    key,
    LiquidityFloorHook.Config({
        floorBps: /* uint16 */ 0,
        unlockTimestamp: /* uint64 */ 0
    })
);

poolManager.initialize(key, startingSqrtPriceX96);
```


### Parameters

| Parameter | Type | Units |
| --- | --- | --- |
| `floorBps` | `uint16` | basis points (`10000` = 100%) |
| `unlockTimestamp` | `uint64` | unix seconds |

## What it reverts with

| Error | Meaning |
| --- | --- |
| `CommitmentBreached(uint256,uint256)` | The withdrawal would take the position below its commitment. |
| `InvalidFloor()` | `floorBps` was zero (no commitment) or above 100%. |
| `PoolAlreadyInitialized()` | The pool already exists, so its configuration is final. |
| `PoolNotConfigured()` | The pool was initialized without a configuration for this hook. |
| `UnlockInThePast()` | The unlock must be in the future at configuration time, or the commitment means nothing. |

## The callbacks it claims

Uniswap v4 reads a hook's permissions from the low fourteen bits of its own address, which is why deploying one means mining a CREATE2 salt. This hook claims 3 of the fourteen:

- `afterInitialize`
- `afterAddLiquidity`
- `beforeRemoveLiquidity`

Mask: `0x1600`, so every deployment of this hook has an address ending in those bits.

## It says what it is, on-chain

Every hook in this family implements `IHookMetadata`: four view functions that let an indexer, a wallet, a router or an agent identify a hook from its address alone, with no registry in the loop.

```bash
cast call $HOOK "hookName()(string)"    # LiquidityFloor
cast call $HOOK "hookVersion()(string)" # 1.0.0
cast call $HOOK "specURI()(string)"     # the machine-readable manifest
cast call $HOOK "hookTags()(string[])"  # lp-economics, commitment, launch, rug-resistance
```

The manifest this repository ships as [`hook.json`](hook.json) is what `specURI()` points at.

## Build and test

```bash
git clone --recurse-submodules https://github.com/nirholas/liquidity-floor
cd liquidity-floor
forge build
forge test
```

Foundry 1.7 or newer, Solidity 0.8.26, EVM version `cancun` (Uniswap v4 requires transient storage).

## Deploy

```bash
# Dry run: mines the salt and prints the address without sending anything.
forge script script/Deploy.s.sol --rpc-url $RPC_URL

# For real.
forge script script/Deploy.s.sol --rpc-url $RPC_URL --broadcast --verify
```

Needs `PRIVATE_KEY` in the environment and a funded deployer on the target chain. See [`docs/deploying.md`](docs/deploying.md).

## Status

**Unaudited.** Built to an audited shape, on OpenZeppelin's audited hook bases, and tested against a real `PoolManager`. No third party has reviewed it. Read "where it does not help" above before putting money behind it.

Not affiliated with Uniswap Labs.
