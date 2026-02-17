---
name: uniswap-v4-developer
description: >
  Use this skill when building, deploying, or reasoning about Uniswap v4 contracts,
  hooks, pool interactions, flash accounting, custom curves, position management, or
  Permit2 integrations. Covers architecture, code patterns, security, and common errors.
sources:
  - https://docs.uniswap.org/contracts/v4/security
  - https://docs.uniswap.org/contracts/v4/guides/hooks/your-first-hook
  - https://docs.uniswap.org/contracts/v4/guides/hooks/hook-deployment
  - https://docs.uniswap.org/contracts/v4/guides/position-manager
  - https://docs.uniswap.org/contracts/v4/guides/calculate-lp-fees
  - https://docs.uniswap.org/contracts/v4/guides/unlock-callback
  - https://docs.uniswap.org/contracts/v4/guides/read-pool-state
  - https://docs.uniswap.org/contracts/v4/guides/custom-accounting
  - https://docs.uniswap.org/contracts/v4/guides/ERC-6909
  - https://docs.uniswap.org/contracts/v4/guides/state-view
  - https://docs.uniswap.org/contracts/v4/guides/swap-routing
  - https://docs.uniswap.org/contracts/v4/guides/flash-accounting
  - https://docs.uniswap.org/contracts/v4/guides/accessing-msg.sender-using-hook
  - https://docs.uniswap.org/contracts/v4/reference/errors/
  - https://docs.uniswap.org/contracts/v4/reference/core/libraries/BitMath
  - https://docs.uniswap.org/contracts/v4/reference/core/types/BalanceDelta
  - https://docs.uniswap.org/contracts/permit2/overview
---

# Uniswap v4 Developer Skill

## Overview

Uniswap v4 introduces a **singleton architecture** with:
- **Hooks** — arbitrary smart contract logic at key lifecycle points (swap, liquidity, donate)
- **Flash Accounting** — delta-based token tracking settled at end of transaction
- **ERC-6909 Claims** — internal token accounting to avoid repeated ERC-20 transfers
- **Permit2** — signature-based token approvals

The central contract is `PoolManager` which holds ALL pool state. All interactions go through it.

---

## Architecture Fundamentals

### PoolManager & Singleton Pattern
All pools live inside a single `PoolManager` contract. There are no separate pool contracts as in v2/v3. This enables gas-efficient multi-hop operations and shared liquidity management.

### The Unlock Pattern
**You cannot interact with pool liquidity directly.** You must:
1. Call `poolManager.unlock(data)` — passes `data` to your callback
2. PoolManager calls `unlockCallback(data)` on your contract
3. Inside the callback: execute operations (swap, modifyLiquidity, donate, take, settle, etc.)
4. All deltas MUST be zero when callback returns — or the tx reverts

```solidity
import {SafeCallback} from "v4-periphery/src/base/SafeCallback.sol";

contract MyIntegration is SafeCallback {
    constructor(IPoolManager _pm) SafeCallback(_pm) {}

    function doSomething() external {
        bytes memory data = abi.encode(/* params */);
        poolManager.unlock(data);
    }

    function _unlockCallback(bytes calldata data) internal override returns (bytes memory) {
        // decode and execute operations here
        // must resolve ALL deltas before returning
    }
}
```

---

## Flash Accounting & Deltas

All operations produce **deltas** (signed int128 values):
- **Negative delta** = PoolManager is owed tokens (you must `settle`)
- **Positive delta** = PoolManager owes you tokens (you can `take`)

Delta-resolving operations:
- `settle(currency)` — transfer tokens TO PoolManager (resolves negative delta)
- `take(currency, recipient, amount)` — transfer tokens FROM PoolManager (resolves positive delta)
- `mint(currency, to, amount)` — mint ERC-6909 claims instead of transferring out
- `burn(currency, amount)` — burn ERC-6909 claims to settle

**Important convention:**
- `amountSpecified < 0` → exact-input swap (you're specifying how much to spend)
- `amountSpecified > 0` → exact-output swap (you're specifying how much to receive)

```solidity
// Resolving deltas after a swap
BalanceDelta delta = poolManager.swap(key, swapParams, hookData);
// delta.amount0() < 0 → settle token0
// delta.amount1() > 0 → take token1
if (delta.amount0() < 0) {
    IERC20(Currency.unwrap(key.currency0)).transfer(address(poolManager), uint128(-delta.amount0()));
    poolManager.settle(key.currency0);
}
if (delta.amount1() > 0) {
    poolManager.take(key.currency1, recipient, uint128(delta.amount1()));
}
```

---

## Hooks

### What Are Hooks
Hooks are external contracts called at lifecycle points:
- `beforeInitialize` / `afterInitialize`
- `beforeAddLiquidity` / `afterAddLiquidity`
- `beforeRemoveLiquidity` / `afterRemoveLiquidity`
- `beforeSwap` / `afterSwap`
- `beforeDonate` / `afterDonate`

### Hook Flags (Encoded in Address)
**This is v4-specific and critical.** The hook contract's address must have specific bits set to indicate which hooks it implements. The PoolManager reads these bits to decide what to call.

Key flags (bit positions in the address):
```
BEFORE_INITIALIZE_FLAG  = 1 << 13
AFTER_INITIALIZE_FLAG   = 1 << 12
BEFORE_ADD_LIQUIDITY_FLAG    = 1 << 11
AFTER_ADD_LIQUIDITY_FLAG     = 1 << 10
BEFORE_REMOVE_LIQUIDITY_FLAG = 1 << 9
AFTER_REMOVE_LIQUIDITY_FLAG  = 1 << 8
BEFORE_SWAP_FLAG        = 1 << 7
AFTER_SWAP_FLAG         = 1 << 6
BEFORE_DONATE_FLAG      = 1 << 5
AFTER_DONATE_FLAG       = 1 << 4
```

### Building a Hook

Extend `BaseHook` and implement `getHookPermissions()`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {BaseHook} from "v4-periphery/src/utils/BaseHook.sol";
import {Hooks} from "v4-core/src/libraries/Hooks.sol";
import {IPoolManager} from "v4-core/src/interfaces/IPoolManager.sol";
import {PoolKey} from "v4-core/src/types/PoolKey.sol";
import {BalanceDelta} from "v4-core/src/types/BalanceDelta.sol";
import {BeforeSwapDelta, BeforeSwapDeltaLibrary} from "v4-core/src/types/BeforeSwapDelta.sol";

contract MyHook is BaseHook {
    constructor(IPoolManager _poolManager) BaseHook(_poolManager) {}

    function getHookPermissions() public pure override returns (Hooks.Permissions memory) {
        return Hooks.Permissions({
            beforeInitialize: false,
            afterInitialize: false,
            beforeAddLiquidity: false,
            afterAddLiquidity: true,   // ← enable what you need
            beforeRemoveLiquidity: false,
            afterRemoveLiquidity: false,
            beforeSwap: false,
            afterSwap: true,           // ← enable what you need
            beforeDonate: false,
            afterDonate: false,
            beforeSwapReturnDelta: false,
            afterSwapReturnDelta: false,
            afterAddLiquidityReturnDelta: false,
            afterRemoveLiquidityReturnDelta: false
        });
    }

    function afterSwap(
        address sender,
        PoolKey calldata key,
        IPoolManager.SwapParams calldata params,
        BalanceDelta delta,
        bytes calldata hookData
    ) external override returns (bytes4, int128) {
        // your logic here
        return (BaseHook.afterSwap.selector, 0);
    }
}
```

### Hook Deployment (Mining)
Because flags are encoded in the address, you must **mine** a deployment address. Use `HookMiner.sol` from `v4-periphery`:

```solidity
import {HookMiner} from "v4-periphery/src/utils/HookMiner.sol";

// Compute the required flags
uint160 flags = uint160(
    Hooks.AFTER_SWAP_FLAG | Hooks.AFTER_ADD_LIQUIDITY_FLAG
);

// Mine for a salt that produces an address with those flags
(address hookAddress, bytes32 salt) = HookMiner.find(
    deployerAddress,
    flags,
    type(MyHook).creationCode,
    abi.encode(address(poolManager))
);

// Deploy using the found salt
MyHook hook = new MyHook{salt: salt}(IPoolManager(poolManager));
require(address(hook) == hookAddress, "hook address mismatch");
```

For local Foundry testing, use `deployCodeTo` cheatcode to bypass mining.

### Accessing msg.sender Inside a Hook
Hook callbacks receive the PoolManager as `msg.sender`. To get the original caller, use `hookData`:

```solidity
// In your router/integration contract:
bytes memory hookData = abi.encode(msg.sender);
poolManager.unlock(abi.encode(key, swapParams, hookData));

// In your hook:
function afterSwap(..., bytes calldata hookData) external override returns (bytes4, int128) {
    address originalCaller = abi.decode(hookData, (address));
    // use originalCaller
}
```

---

## Custom Accounting & Hook Fees

Hooks can intercept and modify token flows using `BeforeSwapDelta` (returned from `beforeSwap`) and `afterSwapReturnDelta`:

- `beforeSwap` can return a `BeforeSwapDelta` to subtract amounts from what the PoolManager processes (custom curve / fee)
- `afterSwap` with `afterSwapReturnDelta: true` can return an `int128` to adjust the final output

```solidity
// Example: 1% hook fee in beforeSwap
function beforeSwap(address, PoolKey calldata key, IPoolManager.SwapParams calldata params, bytes calldata)
    external override returns (bytes4, BeforeSwapDelta, uint24)
{
    int128 hookFee = int128(params.amountSpecified / 100); // 1%
    // mint ERC-6909 claims to collect the fee
    poolManager.mint(key.currency0, address(this), uint128(hookFee));
    return (
        BaseHook.beforeSwap.selector,
        toBeforeSwapDelta(hookFee, 0), // subtract hookFee from input
        0
    );
}
```

---

## ERC-6909 Claims

ERC-6909 is a multi-token standard used as internal "claim" tokens inside the PoolManager. Instead of transferring ERC-20 tokens out and back in (gas-intensive), you can hold claims and burn them to settle.

```solidity
// Mint claims (keep tokens inside PoolManager, get a claim token)
poolManager.mint(currency, recipient, amount);

// Burn claims to settle a negative delta
poolManager.burn(currency, amount);
// then call poolManager.settle(currency)
```

This is useful for hooks collecting fees or for multi-step operations.

---

## Position Manager

`PositionManager` is the periphery contract for managing LP positions. It uses **Actions** encoded as bytes to batch operations:

```solidity
import {Actions} from "v4-periphery/src/libraries/Actions.sol";

bytes memory actions = abi.encodePacked(
    uint8(Actions.MINT_POSITION),
    uint8(Actions.SETTLE_PAIR)
);
bytes[] memory params = new bytes[](2);
params[0] = abi.encode(key, tickLower, tickUpper, liquidity, amount0Max, amount1Max, recipient, hookData);
params[1] = abi.encode(key.currency0, key.currency1);

positionManager.modifyLiquidities(abi.encode(actions, params), deadline);
```

Key actions: `MINT_POSITION`, `INCREASE_LIQUIDITY`, `DECREASE_LIQUIDITY`, `BURN_POSITION`, `SETTLE_PAIR`, `TAKE_PAIR`.

---

## Reading Pool State

### StateView Contract
Use the `StateView` periphery contract for off-chain or view-only reads:

```solidity
IStateView stateView = IStateView(STATE_VIEW_ADDRESS);

// Get slot0 (sqrtPriceX96, tick, etc.)
(uint160 sqrtPriceX96, int24 tick, uint24 protocolFee, uint24 lpFee) = stateView.getSlot0(poolId);

// Get liquidity
uint128 liquidity = stateView.getLiquidity(poolId);

// Get tick info
(uint256 feeGrowthOutside0X128, uint256 feeGrowthOutside1X128, int128 liquidityNet, uint128 liquidityGross)
    = stateView.getTickInfo(poolId, tick);
```

### Reading Inside Unlock Callback (On-chain)
```solidity
// Access pool state directly via PoolManager during unlock
(uint160 sqrtPriceX96, int24 tick,,) = poolManager.getSlot0(poolId);
uint128 liquidity = poolManager.getLiquidity(poolId);
```

---

## Swap Routing

To execute a swap, your contract must unlock the PoolManager, execute the swap, and resolve deltas. The `UniversalRouter` from Uniswap handles this for users. For custom integrations:

```solidity
IPoolManager.SwapParams memory params = IPoolManager.SwapParams({
    zeroForOne: true,               // swap token0 → token1
    amountSpecified: -1e18,         // exact input: spend 1e18 of token0
    sqrtPriceLimitX96: MIN_SQRT_RATIO + 1  // price limit
});

BalanceDelta delta = poolManager.swap(key, params, hookData);
```

Multi-hop swaps: chain multiple `poolManager.swap()` calls inside a single unlock callback. Intermediate deltas offset each other without token transfers.

---

## Calculating LP Fees

LP fees accrue as `feeGrowthGlobal` inside the pool. To calculate uncollected fees for a position:

```solidity
// Fees owed = liquidity × (feeGrowthInside - feeGrowthInsideLast) / Q128
uint256 feesOwed0 = FullMath.mulDiv(
    liquidity,
    feeGrowthInside0X128 - position.feeGrowthInside0LastX128,
    FixedPoint128.Q128
);
```

Use `StateView.getPositionInfo(poolId, owner, tickLower, tickUpper, salt)` to get current fee growth inside.

---

## Permit2

Permit2 enables gasless token approvals via signatures. Two modes:

**SignatureTransfer** — one-time, single-tx signature:
```solidity
// User signs off-chain, router calls:
permit2.permitTransferFrom(
    permit,       // ISignatureTransfer.PermitTransferFrom
    transferDetails, // to + requestedAmount
    owner,        // user who signed
    signature     // EIP-712 signature
);
```

**AllowanceTransfer** — persistent allowance with expiry:
```solidity
// User approves once:
USDC.approve(PERMIT2_ADDRESS, type(uint256).max);
// Then signs a permit granting your contract allowance
permit2.permit(owner, permitSingle, signature);
// Later your contract can transfer:
permit2.transferFrom(owner, recipient, amount, tokenAddress);
```

Integration in v4: `PositionManager` uses Permit2 for token approvals. Users approve Permit2 once per token, then sign permits per operation.

---

## Common Errors Reference

| Error | Selector | Meaning |
|-------|----------|---------|
| `CurrencyNotSettled` | `0x5212cba1` | Deltas not resolved at end of unlock |
| `PoolNotInitialized` | `0x486aa307` | Interacting with uninitialized pool |
| `AlreadyUnlocked` | `0x5090d6c6` | Calling unlock when already unlocked |
| `ManagerLocked` | `0x54e3ca0d` | Calling pool operation without unlocking first |
| `TickSpacingTooLarge` | `0xb02b5dc2` | tickSpacing exceeds MAX_TICK_SPACING |
| `TickSpacingTooSmall` | `0x16fe7696` | tickSpacing below 1 |
| `CurrenciesOutOfOrderOrEqual` | `0xeaa6c6eb` | currency0 must be < currency1 (address sort) |
| `UnauthorizedDynamicLPFeeUpdate` | `0x30d21641` | Only hook can update dynamic fee |
| `SwapAmountCannotBeZero` | `0xbe8b8507` | amountSpecified = 0 |
| `HookAddressNotValid` | — | Hook address bits don't match permissions |
| `InvalidHookResponse` | — | Hook returned wrong selector |

---

## Security Best Practices

### Risk Scoring Dimensions
The Uniswap Foundation provides a self-scoring security framework. Key risk dimensions:
- **Accounting complexity** — custom curves, return deltas
- **External dependencies** — oracles, external calls in hooks
- **Upgradeability** — proxy patterns, admin keys
- **Liquidity behavior** — can hook trap or steal liquidity?
- **Math complexity** — custom bonding curves

### Risk Tiers
- **Low risk** → extensive tests + 1 audit
- **Medium risk** → tests + 2 audits + monitoring
- **High risk** → tests + 2+ audits + formal verification + bug bounty + monitoring

### Universal Checklist
- Never allow hooks to hold user funds without explicit user action
- Always validate `PoolKey` matches expected pool in hook callbacks
- Use `onlyPoolManager` modifier on hook callback functions (BaseHook does this)
- Beware of reentrancy via `unlock` — PoolManager is NOT re-entrant safe by default
- Validate `msg.sender == address(poolManager)` inside hook callbacks
- If hook has admin functions, use timelocks for sensitive changes
- For `beforeSwap` returning deltas, ensure math cannot overflow int128
- Test with `forge test` and fuzzing before any deployment
- Use Foundry's `deployCodeTo` cheatcode for local hook testing (avoids mining)

---

## Development Setup

```bash
# Clone the v4 template
git clone https://github.com/uniswapfoundation/v4-template.git
cd v4-template
forge install
forge test

# Key dependencies installed:
# v4-core  → PoolManager, Hooks, PoolKey, BalanceDelta, etc.
# v4-periphery → BaseHook, SafeCallback, HookMiner, PositionManager, StateView
```

### Key Imports Cheatsheet
```solidity
// Core types
import {PoolKey} from "v4-core/src/types/PoolKey.sol";
import {PoolId, PoolIdLibrary} from "v4-core/src/types/PoolId.sol";
import {BalanceDelta} from "v4-core/src/types/BalanceDelta.sol";
import {Currency, CurrencyLibrary} from "v4-core/src/types/Currency.sol";
import {BeforeSwapDelta, toBeforeSwapDelta} from "v4-core/src/types/BeforeSwapDelta.sol";

// Interfaces
import {IPoolManager} from "v4-core/src/interfaces/IPoolManager.sol";

// Libraries
import {Hooks} from "v4-core/src/libraries/Hooks.sol";
import {TickMath} from "v4-core/src/libraries/TickMath.sol";
import {FullMath} from "v4-core/src/libraries/FullMath.sol";
import {TransientStateLibrary} from "v4-core/src/libraries/TransientStateLibrary.sol";

// Periphery
import {BaseHook} from "v4-periphery/src/utils/BaseHook.sol";
import {SafeCallback} from "v4-periphery/src/base/SafeCallback.sol";
import {HookMiner} from "v4-periphery/src/utils/HookMiner.sol";
import {Actions} from "v4-periphery/src/libraries/Actions.sol";
```

---

## Common Patterns Summary

| Goal | Pattern |
|------|---------|
| Build a hook | Extend `BaseHook`, implement `getHookPermissions()` + callbacks |
| Deploy a hook | Mine address with `HookMiner.find()`, deploy with found salt |
| Execute a swap | Implement `IUnlockCallback`, call `poolManager.unlock()` |
| Add liquidity | Use `PositionManager.modifyLiquidities()` with encoded Actions |
| Read pool state | Use `StateView` periphery contract |
| Collect LP fees | `StateView.getPositionInfo()` + calculate via feeGrowthInside |
| Custom fee | `beforeSwap` → return `BeforeSwapDelta` + `mint` fee to hook |
| Gasless approvals | Use Permit2 `SignatureTransfer` or `AllowanceTransfer` |
| Multi-hop swap | Chain multiple `poolManager.swap()` in single unlock callback |
| Pass caller to hook | Encode `msg.sender` in `hookData` |
