# Blueberry Core - Security Review

Prepared by: [cuthalion0x](https://twitter.com/cuthalion0x)

# Table of Contents
- [Protocol Summary](#protocol-summary)
- [Disclaimer](#disclaimer)
- [Risk Classification](#risk-classification)
  - [Impact](#impact)
  - [Likelihood](#likelihood)
  - [Recommended Actions](#recommended-actions)
- [Executive Summary](#executive-summary)
  - [Scope](#scope)
  - [Privileged Roles](#privileged-roles)
  - [Findings Summary](#findings-summary)
- [Findings](#findings)
  - [High](#high)
  - [Medium](#medium)
  - [Low](#low)
  - [Informational](#informational)
- [Attribution](#attribution)

# Protocol Summary

## From the [Protocol Docs](https://docs.blueberry.garden/blueberry-overview/what-is-blueberry)

> Blueberry protocol is a decentralized lending market enabling lending and leveraged borrowing up to 20x or more of your collateral value. You can use leveraged loans to deploy into in any integrated strategy on the protocol. Blueberry's flexible and modular design allows support for every strategy on Ethereum over time. Examples of currently integrated strategies include Convex, Curve, Leverage Trading, Yield Arbitraging, and Uniswap v3 automated vaults.

## Reviewer Overview

There are two key components to the `blueberry-core` repository: a money market forked from [Iron Bank's version of Compound Finance](https://github.com/ibdotxyz/compound-protocol), and the Blueberry Strategy framework which is built on top. Blueberry strategies (called Spells) allow users to borrow from the money market on leverage and participate in administrator-approved strategies such as ERC-20 short/long or AMM yield farms. In order to maintain a healthy market, all user positions are held within the Blueberry Bank where they can be liquidated as net value approaches zero.

Opening a position follows a simple pattern:

1. The User makes an initial contribution of value `uv`.
2. The Spell posts the user's contribution, minus a deposit fee, as collateral to the Bank. The collateral value is denoted `cv`.
3. The Spell borrows from the Bank on leverage (up to a configurable maximum LTV), incurring a debt of `ov`.
4. The Spell uses the borrowed funds to execute its core strategy, e.g., to go short/long on a token or to enter an AMM yield farm. This leveraged position, whose value is denoted `pv`, is posted to the Bank.

At any point, the position's net value is `(pv + cv) - ov`, and oracles must exist for all three values (it is impossible to open a position if these oracles do not exist). The user's PnL is denoted `(pv + cv) - (ov + uv)`, but the system itself does not care about `uv`.

The position can be liquidated by another user to maintain healthy markets. The conditions for liquidation are as follows:
* If `pv >= ov`, then the position is over-collateralized, and there is no liquidation.
* If `pv < ov`, then the position's risk is defined as `(ov - pv) / cv`, and it can be liquidated if the risk exceeds a configurable threshold.
* If `(pv + cv) < ov`, then the position is underwater, and the protocol has token on bad debt. This scenario should never occur but depends on timely and profitable liquidation.

Closing a position follows an approximate reversal of the opening pattern.

# Disclaimer

A smart contract security review can never assert the complete absence of vulnerabilities. We make every effort to find as many vulnerabilities as possible but hold no responsibility for the findings included in or missing from this document. Subsequent security reviews and bug bounty programs are strongly recommended. This review focuses solely on the security aspects of the smart contract implementations and does not constitute an endorsement of the underlying business or product.

# Risk Classification

The risk classification matrix maps (impact, likelihood) pairs to overall severities. The system is imperfect and allows for some subjectivity on the part of the reviewer.

| Severity Level         | Impact: High     | Impact: Medium     | Impact: Low     |
| ---------------------- | ---------------- | ------------------ | --------------- |
| **Likelihood: High**   | High             | High               | Medium          |
| **Likelihood: Medium** | High             | Medium             | Low             |
| **Likelihood: Low**    | Medium           | Low                | Low             |

An additional severity level, **Informational**, is provided for findings that have no immediate impact but which present opportunities to bolster code quality. These findings are largely subjective and reflect the reviewers' personal engineering experience. They should be considered a form of shared wisdom which may or may not be relevant to the protocol developers and can be safely ignored.

## Impact

Impact refers to the potential harm or consequence to the protocol or its users as a direct result of the vulnerability.

- **High Impact:** Funds are **directly** at risk, or functionality/availability is **severely** disrupted.
- **Medium Impact:** Funds are **indirectly** at risk, or functionality/availability is **somewhat** disrupted.
- **Low Impact:** Funds are **not** at risk, but the protocol's **behavior** may not match the developer's intentions.

## Likelihood

Likelihood represents the probability of the impact occurring.

- **High Likelihood:** An exploit is **easy** to perform or **well** incentivized.
- **Medium Likelihood:** An exploit is **difficult** to perform or **conditionally** incentivized.
- **Low Likelihood:** An exploit is **almost impossible** to perform or **poorly** incentivized.

## Recommended Actions

We recommend the following actions be taken in the event of a valid finding, depending on its severity.

- **High Severity:** Mitigate immediately (if deployed) and fix as soon as possible.
- **Medium Severity:** Strongly consider fixing.
- **Low Severity:** Consider fixing.
- **Informational Severity:** Consider for a future version of the code base.

# Executive Summary

- **Repository:** [https://github.com/Blueberryfi/blueberry-core](https://github.com/Blueberryfi/blueberry-core)
- **Commit hash:** [408d02f2ec7475b795a7fe461aa1d435bc260b7d](https://github.com/Blueberryfi/blueberry-core/commit/408d02f2ec7475b795a7fe461aa1d435bc260b7d)
- **Report Date:** April 25, 2024

## Scope 

All smart contracts within [contracts/](https://github.com/Blueberryfi/blueberry-core/tree/408d02f2ec7475b795a7fe461aa1d435bc260b7d/contracts) were in scope for this review, with the following exceptions:

* All third-party interfaces in [contracts/interfaces/](https://github.com/Blueberryfi/blueberry-core/tree/408d02f2ec7475b795a7fe461aa1d435bc260b7d/contracts/interfaces) were out of scope.
* All contracts in [contracts/liquidation/](https://github.com/Blueberryfi/blueberry-core/tree/408d02f2ec7475b795a7fe461aa1d435bc260b7d/contracts/liquidation) were out of scope.
* All mock contracts in [contracts/mock/](https://github.com/Blueberryfi/blueberry-core/tree/408d02f2ec7475b795a7fe461aa1d435bc260b7d/contracts/mock) were out of scope.
* The Convex Spells and Wrapper were out of scope:
    * [contracts/spell/ConvexSpell.sol](https://github.com/Blueberryfi/blueberry-core/blob/408d02f2ec7475b795a7fe461aa1d435bc260b7d/contracts/spell/ConvexSpell.sol)
    * [contracts/spell/ConvexSpellV2.sol](https://github.com/Blueberryfi/blueberry-core/blob/408d02f2ec7475b795a7fe461aa1d435bc260b7d/contracts/spell/ConvexSpellV2.sol)
    * [contracts/wrapper/WConvexBooster.sol](https://github.com/Blueberryfi/blueberry-core/blob/408d02f2ec7475b795a7fe461aa1d435bc260b7d/contracts/wrapper/WConvexBooster.sol)

These smart contracts are already deployed to mainnet, and for security's sake only minimum viable upgrades should be made moving forward. As such, this review is focused on loss of funds, DOS, and similar vulnerabilities. It ignores gas optimizations entirely. However, many informational findings are also provided, as there is room to improve the code architecture for future versions.

## Privileged Roles

For the purposes of this review, the following privileged entities are considered trusted. Nonetheless, we still make every effort to report egregious administrative powers that, if misused even accidentally, could render the system inoperable. "Egregious" is subjective.

Blueberry Strategies:
- `BlueberryBank` has a single `OWNER` with authority to whitelist strategies and tokens, configure liquidation thresholds, and pause any part of the system including lending, withdrawing, borrowing, repaying, and liquidating.
- `ProtocolConfig` has a single `OWNER` with authority to manage deposit, withdrawal, and reward fees.
- `CoreOracle` has a single `OWNER` with authority to set per-token price oracles and pause the full oracle system.
- Each `Spell` has a single `OWNER` with authority to configure per-token strategies and their minimum and maximum position sizes.
- Each `SoftVault` and `HardVault` has a single `OWNER`, but these appear to be powerless for now.
- Each `Wrapper` has a single `OWNER`, but these appear to be powerless for now.

Money Market:
- `Comptroller` has a single `ADMIN` with authority to add and delist markets, configure borrowing and liquidation thresholds, set per-token price oracles, configure supply and borrow caps, pause any part of the system, set credit limits, and configure other privileged roles.
- `Comptroller` has a single `GUARDIAN`, configured by `ADMIN`, with authority to pause any part of the system.
- `Comptroller` has a single `CREDIT_LIMIT_MANAGER`, configured by `ADMIN`, with authority to set credit limits.

## Findings Summary

| Severity      | Count |
| ------------- | ----- |
| High          | 2     |
| Medium        | 3     |
| Low           | 7     |
| Informational | 11    |

# Findings

## High

### [H-01] Missing oracle implementation compatible with money market.

#### Status

Fixed by [PR 178](https://github.com/Blueberryfi/blueberry-core/pull/178).

#### Description

**NOTE: This finding has already been discovered via a hack which is described in [this post-mortem](https://medium.com/@blueberryprotocol/2-22-24-exploit-post-mortem-6f6be7c1dcc3). But as of commit `408d02f2ec7475b795a7fe461aa1d435bc260b7d`, it has not yet been fixed, so a simplified report is provided here for the sake of completeness.**

The [contracts/money-market](https://github.com/Blueberryfi/blueberry-core/tree/408d02f2ec7475b795a7fe461aa1d435bc260b7d/contracts/money-market) directory does not include a complete price oracle, so it stands to reason that the intention is to use the [Blueberry `CoreOracle`](https://github.com/Blueberryfi/blueberry-core/blob/408d02f2ec7475b795a7fe461aa1d435bc260b7d/contracts/oracle/CoreOracle.sol). But this oracle always prices "one" (`10 ** decimals()`) token, whereas Compound and its forks expect an oracle that prices `1e18` tokens regardless of `decimals()`. For USDC, the Blueberry `CoreOracle` would produce a price close to `1e18`, whereas Compound would expect a price of `1e30` due to USDC's 6 decimals.

As a result, the money market will grossly misprice some assets and allow outsized borrowing. This, coupled with the fact that normal users are allowed to borrow from the money market (see [H-02](#h-02-normal-user-accounts-are-allowed-to-borrow-from-the-money-market)), resulted in the hack described above.

#### Recommendation

Add a new price oracle compatible with Compound's money market.

### [H-02] Normal user accounts are allowed to borrow from the money market.

#### Status

Fixed by [PR 172](https://github.com/Blueberryfi/blueberry-core/pull/172).

#### Description

There is no restriction on borrowing in any of the following functions:
- [`BErc20.borrow()`](https://github.com/Blueberryfi/blueberry-core/blob/408d02f2ec7475b795a7fe461aa1d435bc260b7d/contracts/money-market/BErc20.sol#L86)
- [`BCollateralCapErc20.borrow()`](https://github.com/Blueberryfi/blueberry-core/blob/408d02f2ec7475b795a7fe461aa1d435bc260b7d/contracts/money-market/BCollateralCapErc20.sol#L89)
- [`BWrappedNative.borrow()`](https://github.com/Blueberryfi/blueberry-core/blob/408d02f2ec7475b795a7fe461aa1d435bc260b7d/contracts/money-market/BWrappedNative.sol#L146)
- [`BWrappedNative.borrowNative()`](https://github.com/Blueberryfi/blueberry-core/blob/408d02f2ec7475b795a7fe461aa1d435bc260b7d/contracts/money-market/BWrappedNative.sol#L157)

The Blueberry system is a walled garden. It is critical to Blueberry's leveraged strategy model that the Bank contract custody all funds for centralized liquidation. Given this assumption, we can expect the money market parameters to be quite loose; they may allow for dangerous borrowing conditions that would put the system in a state of bad debt if it weren't managed directly by the Bank.

Allowing normal users to borrow from the money market circumvents the central Bank and puts the system at unnecessary risk. Given the Bank's role in managing a healthy system, all user flows such as lending and borrowing should be routed through the Bank itself.

#### Recommendation

Disallow normal users from borrowing through the money market; they should be forced to execute strategies using `Spell.openPosition()` which invokes `BlueberryBank.borrow()`.

One suggestion is to enforce credit-only borrowing at the Comptroller level. Only credit accounts would be allowed to borrow, and only up to their credit limits. The Bank is expected to be the only such credit account for now, but this option leaves open the possibility of creating additional credit accounts later.

## Medium

### [M-01] Leveraged withdrawal fees put user principal at risk.

#### Status

Acknowledged.

#### Description

All Spells charge a withdrawal fee on isolated collateral, but the `ShortLongSpell` incidentally charges an additional fee on the full position value. This happens because the `ShortLongSpell` uniquely [stores its position collateral within a `SoftVault`](https://github.com/Blueberryfi/blueberry-core/blob/408d02f2ec7475b795a7fe461aa1d435bc260b7d/contracts/spell/ShortLongSpell.sol#L163-L165), so when closing a position it also [withdraws that position from the `SoftVault`](https://github.com/Blueberryfi/blueberry-core/blob/408d02f2ec7475b795a7fe461aa1d435bc260b7d/contracts/spell/ShortLongSpell.sol#L192-L193). The `SoftVault` takes a withdrawal fee, which in this case is assessed on the full value of the position being withdrawn.

The only correlation between the value of the isolated collateral and the value of the full position is a leverage factor, which is advertised as "up to 20x or more." So, consider the impact of a hypothetical 1% withdrawal fee.

Immediately after opening a position with 20x leverage, a user's position looks like this:

```
cv = 1
ov = 20
pv = 20
value = cv + pv - ov = 1
```

For mathematical simplicity, assume the user then withdraws before any price action.

```
fee = 1% * (cv + pv) = 0.21
withdraw: value - fee = 1 - 0.21 = 0.79
```

The user loses 21% of the value of their position on a 1% withdrawal fee! Note that this is only a sample hypothetical; depending on the relations among `cv`, `ov`, and `pv`, this impact percentage could be larger or smaller. Importantly, we note the mismatch between how the user's value is stored and how the fee is charged. The user's initial deposit goes to `cv`, which enables opening a leveraged position of `pv`. By then charging the fee on `pv`, the protocol destroys the assumed relationship between the fee percentage and the user's value. The fee is levered up just like `pv` from `cv`, and the user's entire PnL could be at stake.

Note: This impacts `Spell.closePosition()` but not `BlueberryBank.liquidate()`. This stands to reason because:
* A liquidator supplies `ov` to extract `cv + pv`. These values are on the same (leveraged) scale, so the fee does not hurt the liquidator's PnL in an outsized way.
* By contrast, a user supplies `cv` but is charged a fee on `pv` at a much different scale. This is the root of the issue and what puts user PnL at risk.

#### Recommendation

Remove the `SoftVault` withdrawal fee; charge fees only in the `BlueberryBank` itself.

### [M-02] `WERC20` token IDs are expected to be unique, but collisions are inevitable.

#### Status

Fixed by [PR 173](https://github.com/Blueberryfi/blueberry-core/pull/173).

#### Description

In [`WERC20._encodeTokenId()`](https://github.com/Blueberryfi/blueberry-core/blob/408d02f2ec7475b795a7fe461aa1d435bc260b7d/contracts/wrapper/WERC20.sol#L104-L111), the ID is composed solely of the underlying token address. This means that for each underlying token, there can be only one unique ID.

```solidity
function _encodeTokenId(address underlyingToken) internal pure returns (uint) {
    return uint256(uint160(underlyingToken));
}
```

[`BaseWrapper._validateTokenId()`](https://github.com/Blueberryfi/blueberry-core/blob/408d02f2ec7475b795a7fe461aa1d435bc260b7d/contracts/wrapper/BaseWrapper.sol#L24-L32) enforces uniqueness of IDs. This call is included within [`WERC20.mint()`](https://github.com/Blueberryfi/blueberry-core/blob/408d02f2ec7475b795a7fe461aa1d435bc260b7d/contracts/wrapper/WERC20.sol#L69), thus preventing any two minted batches from occupying the same ID.

```solidity
function _validateTokenId(uint256 id) internal view {
    if (balanceOf(msg.sender, id) != 0) {
        revert Errors.DUPLICATE_TOKEN_ID(id);
    }
}
```

Imagine that Alice wants to short `WETH`. She calls [`ShortLongSpell.openPosition()`](https://github.com/Blueberryfi/blueberry-core/blob/408d02f2ec7475b795a7fe461aa1d435bc260b7d/contracts/spell/ShortLongSpell.sol#L82), thereby minting some `WERC20` tokens with an ID determined by the address of `WETH`. Now Bob also wants to short `WETH`. When he calls `ShortLongSpell.openPosition()`, the call will revert because he attempts to mint more `WERC20` tokens with an occupied ID. Even Alice will not be allowed to short more `WETH` until she first closes her original position, after which any other single user can short `WETH`.

This makes `ShortLongSpell` unusable for a multi-user system.

#### Recommendation

Either use a more collision-resistant ID scheme, such as a nonce, or remove ID validation altogether. We show in [I-01](#i-01-wrapper-id-validation-is-unnecessary-as-there-is-no-conceivable-benefit-to-uniqueness) that removal is favorable.

### [M-03] `WConvexBooster` allows one user to redeem other users' positions using `amount = type(uint256).max`.

#### Status

Fixed by [PR 173](https://github.com/Blueberryfi/blueberry-core/pull/173).

#### Description

[This burn amount pattern](https://github.com/Blueberryfi/blueberry-core/blob/408d02f2ec7475b795a7fe461aa1d435bc260b7d/contracts/wrapper/WConvexBooster.sol#L170-L172) in `WConvexBooster` allows the caller to redeem its entire balance of a given token ID:

```solidity
if (amount == type(uint256).max) {
    amount = balanceOf(msg.sender, id);
}
```

Since the caller is always a Spell, the full balance represents the Spell's full balance rather than a given user's. So in order for this pattern to work, we must assume that each ID corresponds to a unique user position such that it is not possible for one user to steal another user's funds.

In most Wrappers, uniqueness is enforced, so there is no risk in using this pattern. But `WConvexBooster.mint()` is missing a call to `_validateTokenId()`, so uniqueness is not strictly enforced here. If there is any possibility of ID collision, it will result in a direct theft vector.

#### Recommendation

Either validate IDs, or remove ID validation and this burn amount pattern altogether. We show in [I-01](#i-01-wrapper-id-validation-is-unnecessary-as-there-is-no-conceivable-benefit-to-uniqueness) that removal is favorable.

## Low 

### [L-01] `BlueberryBank.withdrawLend()` double-charges the withdrawal fee.

#### Status

Acknowledged.

#### Description

The [following block](https://github.com/Blueberryfi/blueberry-core/blob/408d02f2ec7475b795a7fe461aa1d435bc260b7d/contracts/BlueberryBank.sol#L284-L296) of `BlueberryBank.withdrawLend()` double-charges a withdrawal fee:

```solidity
// @audit `wAmount` already has a fee extracted (see below).
uint256 wAmount;
if (_isSoftVault(token)) {
    IERC20(bank.softVault).universalApprove(bank.softVault, shareAmount);
    // @audit `SoftVault` charges a fee here.
    wAmount = ISoftVault(bank.softVault).withdraw(shareAmount);
} else {
    // @audit `HardVault` charges a fee here.
    wAmount = IHardVault(bank.hardVault).withdraw(token, shareAmount);
}

IFeeManager feeManager = getFeeManager();

pos.underlyingVaultShare -= shareAmount;
IERC20(token).universalApprove(address(feeManager), wAmount);
// @audit Here we charge a second fee on `wAmount`.
wAmount = feeManager.doCutWithdrawFee(token, wAmount);
```

#### Recommendation

Remove the `SoftVault` and `HardVault` withdrawal fees; charge fees only in the `BlueberryBank` itself.

### [L-02] Users can create outsized positions by front-running `AuraSpell.openPositionFarm()`.

#### Status

Fixed by [PR 186](https://github.com/Blueberryfi/blueberry-core/pull/186).

#### Description

In [`AuraSpell._getJoinPoolParamsAndApprove()`](https://github.com/Blueberryfi/blueberry-core/blob/408d02f2ec7475b795a7fe461aa1d435bc260b7d/contracts/spell/AuraSpell.sol#L279-L288), the following code block populates a Balancer join using all of the contract's available token balances:

```solidity
for (i; i < length; ++i) {
    if (tokens[i] != lpToken) {
        amountsIn[j] = IERC20(tokens[i]).balanceOf(address(this));
        if (amountsIn[j] > 0) {
            IERC20(tokens[i]).universalApprove(vault, amountsIn[j]);
            maxAmountsIn[i] = amountsIn[j];
        }
        ++j;
    } else isLPIncluded = true;
}
```

This approach contradicts two observations:

1. At this point in the execution, the contract is expected to hold only `debtToken`, so why check balances for multiple tokens?
2. The contract knows exactly how much `debtToken` it borrowed (`param.borrowAmount`), so why read live balances at all?

Reading these live balances has one trivial consequence: it consumes unnecessary gas by making external calls in a loop. But a more interesting consequence is that it opens up the possibility of front-running.

A user can front-run their own call or another user's call to `AuraSpell.openPositionFarm()` by sending tokens to the `AuraSpell` contract. These tokens would then get included alongside the borrowed amount of `debtToken` and boost the size of the caller's position.

This "attack" has no obvious negative consequences for the protocol, as the front-runner provides their own tokens at full risk of liquidation. It only breaks the implicit invariant that `pv ~= ov` immediately after opening a position. It creates `pv > ov` which replicates the conditions of a position earning profit over time.

Nonetheless, this is a somewhat dangerous pattern which has potentially unforeseen consequences, and since `AuraSpell` is the only Spell to behave in this way, it is worth reporting the abnormality.

#### Recommendation

Supply only the borrowed amount of `debtToken` to the Balancer join. Do not use the contract's live token balances, and set all amounts other than the `debtToken`'s to zero.

### [L-03] `BlueberryBank.execute()` might calculate risk using stale data.

#### Status

Fixed by [PR 185](https://github.com/Blueberryfi/blueberry-core/pull/185).

#### Description

On [L235 of `BlueberryBank`](https://github.com/Blueberryfi/blueberry-core/blob/408d02f2ec7475b795a7fe461aa1d435bc260b7d/contracts/BlueberryBank.sol#L235), the `execute()` function makes a call to `isLiquidatable()`:

```solidity
if (isLiquidatable(positionId)) revert Errors.INSUFFICIENT_COLLATERAL();
```

The [`isLiquidatable()`](https://github.com/Blueberryfi/blueberry-core/blob/408d02f2ec7475b795a7fe461aa1d435bc260b7d/contracts/BlueberryBank.sol#L659-L661) function calls `getPositionRisk()`:

```solidity
return getPositionRisk(positionId) >= _banks[_positions[positionId].underlyingToken].liqThreshold;
```

And [`getPositionRisk()`](https://github.com/Blueberryfi/blueberry-core/blob/408d02f2ec7475b795a7fe461aa1d435bc260b7d/contracts/BlueberryBank.sol#L641-L656) contains the logic for computing risk from the values of isolated collateral, debt, and position collateral. The [`getIsolatedCollateralValue()`](https://github.com/Blueberryfi/blueberry-core/blob/408d02f2ec7475b795a7fe461aa1d435bc260b7d/contracts/BlueberryBank.sol#L626-L638) function, in particular, makes a call to `BErc20.exchangeRateStored()`:

```solidity
underlyingAmount =
    (IBErc20(_banks[pos.underlyingToken].bToken).exchangeRateStored() * pos.underlyingVaultShare) /
    Constants.PRICE_PRECISION;
```

Compound's `exchangeRateStored()` returns the latest exchange rate from storage, which is to say that the data is stale. Liveness of this data is only guaranteed if Compound's `accrueInterest()` has already been called in the same block. Most relevant Blueberry functions - such as `liquidate()`, `lend()`, `withdrawLend()`, `borrow()`, and `repay()` - guarantee liveness by using the [`poke()` modifier](https://github.com/Blueberryfi/blueberry-core/blob/408d02f2ec7475b795a7fe461aa1d435bc260b7d/contracts/BlueberryBank.sol#L110-L114):

```solidity
modifier poke(address token) {
    accrue(token);
    _;
}
```

As the name implies, `accrue()` is a `BlueberryBank` function which indirectly invokes `BErc20.accrueInterest()`.

In `BlueberryBank.execute()`, it is tempting to believe that `spell.call(data)` must have executed a poking operation, and so this data is already live. But this offloads an assumption and responsibility to an unknown Spell contract which is not architecturally guaranteed to do anything in particular. In fact, most Spells offer the opportunity to partially close a position without touching the isolated collateral at all, in which case the `underlyingToken` will not be poked.

As such, it seems pertinent to guarantee liveness of this data by adding a call to `accrue()` within `Blueberry.execute()` itself.

#### Recommendation

Add `accrue(_positions[positionId].underlyingToken)` directly ahead of the call to `isLiquidatable()`.

### [L-04] Dangerous propagation of `amount = type(uint256).max` through sequential operations.

#### Status

Fixed by [PR 173](https://github.com/Blueberryfi/blueberry-core/pull/173).

#### Description

[This pattern](https://github.com/Blueberryfi/blueberry-core/blob/408d02f2ec7475b795a7fe461aa1d435bc260b7d/contracts/spell/AuraSpell.sol#L188-L196) appears in `AuraSpell` and in similar form in all other Spells:

```solidity
bank.takeCollateral(param.amountPosRemove);

/// 1. Burn the wrapped tokens, retrieve the BPT tokens, and claim the AURA rewards
{
    address[] memory rewardTokens;
    (rewardTokens, ) = _wAuraBooster.burn(pos.collId, param.amountPosRemove);
    /// 2. Swap each reward token for the debt token
    _sellRewards(rewardTokens, expectedRewards, swapDatas);
}
```

Note that `param.amountPosRemove` can be `type(uint256).max`, and in this case `Bank.takeCollateral()` will withdraw the user's entire position. The same behavior is modeled in this pattern in `Wrapper.burn()`:

```solidity
if (amount == type(uint256).max) {
    amount = balanceOf(msg.sender, id);
}
```

But given the strong possibility of ID collisions, and sparse validation of ID uniqueness, it seems wise to remove this pattern altogether. It also seems unwise to filter the same input amount multiple times throughout a sequence; this assumes that each function has an identical filtering mechanism, which may not always be true.

So, instead, we should avoid using `param.amountPosRemove` as the input to both `Bank.takeCollateral()` and `Wrapper.burn()`. `Bank.takeCollateral()` has an unused return value which tells us exactly how many tokens were removed, and that is the value that we should pass to `Wrapper.burn()`.

#### Recommendation

Use the return value of `BlueberryBank.takeCollateral()` as the input amount for `Wrapper.burn()`.

### [L-05] Excessive entry points to lending make both `ibToken` and `bToken` available to users.

#### Status

Fixed by [PR 172](https://github.com/Blueberryfi/blueberry-core/pull/172).

#### Description

There is no restriction on minting in any of the following functions:
- [`BErc20.mint()`](https://github.com/Blueberryfi/blueberry-core/blob/408d02f2ec7475b795a7fe461aa1d435bc260b7d/contracts/money-market/BErc20.sol#L53)
- [`BCollateralCapErc20.mint()`](https://github.com/Blueberryfi/blueberry-core/blob/408d02f2ec7475b795a7fe461aa1d435bc260b7d/contracts/money-market/BCollateralCapErc20.sol#L56)
- [`BWrappedNative.mint()`](https://github.com/Blueberryfi/blueberry-core/blob/408d02f2ec7475b795a7fe461aa1d435bc260b7d/contracts/money-market/BWrappedNative.sol#L67)
- [`BWrappedNative.mintNative()`](https://github.com/Blueberryfi/blueberry-core/blob/408d02f2ec7475b795a7fe461aa1d435bc260b7d/contracts/money-market/BWrappedNative.sol#L78)

It is not clear why, but the Blueberry system makes use of a `SoftVault` contract which introduces an extra layer of wrapping around the money market's `bToken`. `SoftVault` produces `ibToken` which are 1:1 exchangeable for `bToken`.

If an ecosystem is to arise around interest-bearing Blueberry tokens, it would be very confusing and redundant to have both `ibToken` and `bToken` in the wild. All user interactions with `BErc20.mint()` should be forced through the `SoftVault` so that users end up with `ibToken` instead of `bToken`. Therefore, normal users should not be allowed to call `BErc20.mint()` directly to obtain `bToken`.

#### Recommendation

Enforce that only a `bToken`'s corresponding `SoftVault` can mint. This way, `bToken` never exists outside the scope of the Blueberry system, and all external interactions by users and contracts deal exclusively with `ibToken`.

### [L-06] Prefer failing gracefully when `treasury` is unconfigured.

#### Status

Acknowledged.

#### Description

The [`FeeManager._doCutFee()`](https://github.com/Blueberryfi/blueberry-core/blob/408d02f2ec7475b795a7fe461aa1d435bc260b7d/contracts/FeeManager.sol#L90-L108) function is at the center of many key operations throughout the Blueberry system, such as lending, withdrawing, and harvesting rewards. It takes an `amount` as input and returns the same `amount` minus whatever `fee` is collected.

This line in particular aims to protect against an unconfigured system:

```solidity
if (treasury == address(0)) revert Errors.NO_TREASURY_SET();
```

But if this reverts, then all of those core system operations like lending, withdrawing, and harvesting rewards are bricked. And there is no need to brick them. The obvious return value in this situation is simply `amount`, because `fee` should be zero when the `treasury` address is unset. Returning `amount` would allow the system to continue behaving properly even if fees aren't being collected.

#### Recommendation

Do not revert when `treasury` is unset; simply return `amount` without alteration.

### [L-07] Amount mismatches in event emissions and token approvals.

#### Status

Acknowledged.

#### Description

The following code locations contain mismatched amounts:
* [BlueberryBank.sol:L324](https://github.com/Blueberryfi/blueberry-core/blob/408d02f2ec7475b795a7fe461aa1d435bc260b7d/contracts/BlueberryBank.sol#L324-L326) - transfers `borrowedAmount`, emits `amount`.
* [SoftVault.sol:L113](https://github.com/Blueberryfi/blueberry-core/blob/408d02f2ec7475b795a7fe461aa1d435bc260b7d/contracts/vault/SoftVault.sol#L113-L120) - approves `amount`, mints `uBalanceAfter - uBalanceBefore`, emits `amount`.
* [WERC20.sol:L71](https://github.com/Blueberryfi/blueberry-core/blob/408d02f2ec7475b795a7fe461aa1d435bc260b7d/contracts/wrapper/WERC20.sol#L71-L73) - mints `balanceAfter - balanceBefore`, emits `amount`.

In practice, all of these amounts are the same (see [I-02](#i-02-fee-on-transfer-fot-tokens-should-not-be-supported)), and the mismatches do not create real risk, but technically the code is incorrect. Other such instances probably also exist.

#### Recommendation

When dealing with multiple similar values, ensure situational usage of the proper values.

## Informational

### [I-01] Wrapper ID validation is unnecessary as there is no conceivable benefit to uniqueness.

#### Status

Fixed by [PR 173](https://github.com/Blueberryfi/blueberry-core/pull/173).

#### Description

[`BaseWrapper._validateTokenId()`](https://github.com/Blueberryfi/blueberry-core/blob/408d02f2ec7475b795a7fe461aa1d435bc260b7d/contracts/wrapper/BaseWrapper.sol#L24-L32) is designed to ensure uniqueness of ERC-1155 token IDs:

```solidity
function _validateTokenId(uint256 id) internal view {
    if (balanceOf(msg.sender, id) != 0) {
        revert Errors.DUPLICATE_TOKEN_ID(id);
    }
}
```

This is usually called from `Wrapper.mint()` to ensure that it is impossible to create multiple batches of tokens with the same ID. But the aim of uniqueness enforcement is not clear; individual position sizes are already accounted in the `BlueberryBank`. These token IDs essentially represent metadata attached to each position, but these metadata could be accounted differently (in a storage struct), and there's no need for each position's metadata to be unique.

The uniqueness assumption creates problems elsewhere in the code base (see [M-02](#m-02-werc20-token-ids-are-expected-to-be-unique-but-collisions-are-inevitable), [M-03](#m-03-wconvexbooster-allows-one-user-to-redeem-other-users-positions-using-amount--typeuint256max), and [L-04](#l-04-dangerous-propagation-of-amount--typeuint256max-through-sequential-operations)). Since it is unnecessary, it should simply be removed.

#### Recommendation

Remove ID validation and all associated artifacts.

### [I-02] Fee-on-Transfer (FoT) tokens should NOT be supported.

#### Status

Acknowledged.

#### Description

The following pattern appears throughout the code base, in far too many locations to count:

```solidity
uint256 balanceBefore = IERC20Upgradeable(token).balanceOf(address(this));
IERC20Upgradeable(token).safeTransferFrom(msg.sender, address(this), amount);
uint256 balanceAfter = IERC20Upgradeable(token).balanceOf(address(this));
```

Its only usage is for tokens that fail to transfer precisely `amount` tokens when calling `transfer(to, amount)` or `transferFrom(from, to, amount)`. Such tokens do not break the explicit ERC-20 interface but do break the implicit ERC-20 specification and are incompatible with many smart contracts. The most common form of these aberrant tokens is the Fee-on-Transfer (FoT) token.

These tokens cause so many problems that they are not worth the effort to support, and the effort made here by using this pattern is insufficient. If these tokens were truly to be supported, many more changes to the code base would likely be required. As such, we highly recommend avoiding this pattern and suggest that Blueberry never whitelist such tokens as the code base was not reviewed with these tokens in mind.

Furthermore, the pattern is a waste of gas.

#### Recommendation

Never endeavor to support Fee-on-Transfer (FoT) tokens or other such tokens that break the implicit ERC-20 transfer rules. Remove this pattern from future versions of the code base.

### [I-03] Deposit and withdrawal fees misalign incentives.

#### Status

Acknowledged.

#### Description

The Blueberry system charges fees when users deposit and withdraw, as well as fees on any rewards that users accrue during their time in the system.

Reward fees represent a perfect alignment of incentives: the better returns the Blueberry protocol supplies to its users, the more fees are collected by the Blueberry treasury. The fee is collected purely on PnL and does not represent a loss to the user.

Deposit and withdrawal fees represent a misalignment of incentives: a user is charged merely for the privilege of entering the system (which also costs gas) and again, perhaps suprisingly, when exiting the system (which also costs gas). The fee is charged against the user's principal value and represents a direct loss. The user is in the hole immediately after entering the system and is pressured to keep a position open long enough to overcome both the past entry fee and the future exit fee.

Fees should always be charged on PnL rather than user principal.

#### Recommendation

Remove all deposit and withdrawal fees from future versions of the code base, and charge fees exclusively on user PnL.

### [I-04] Unreachable code and unnecessary complexity in BPT oracles.

#### Status

Acknowledged.

#### Description

The [`_getMarketPrice()` function](https://github.com/Blueberryfi/blueberry-core/blob/408d02f2ec7475b795a7fe461aa1d435bc260b7d/contracts/oracle/StableBPTOracle.sol#L217-L234) in each BPT oracle attempts to address potential "nested" BPT tokens, which is to say a Balancer pool whose constituent set includes another Balancer pool:

```solidity
function _getMarketPrice(address token) internal view returns (uint256) {
    try _base.getPrice(token) returns (uint256 price) {
        return price;
    } catch {
        try _weightedPoolOracle.getPrice(token) returns (uint256 price) {
            return price;
        } catch {
            return getPrice(token);
        }
    }
}
```

It does this by checking a pool's constituent `token` against various oracles to determine its type:
* The `CoreOracle` - presumably to check if it is a "regular" ERC-20 token.
* The `WeightedBPTOracle`.
* The `StableBPTOracle`.

However, this is unnecessary, and all code beyond the call to the `CoreOracle` is unreachable. This is because the `CoreOracle` already registers the type of each constituent token and routes to the proper oracle, even if that is a BPT oracle.

#### Recommendation

Remove the additional complexity of the `_getMarketPrice()` function from both `StableBPTOracle.sol` and `WeightedBPTOracle.sol`.

### [I-05] Major code duplication across Spells.

#### Status

Acknowledged.

#### Description

Each Spell contains the same boilerplate logic in `openPosition()` and `closePosition()`, but the logic is duplicated in every Spell and differs slightly in implementation (but not outcome). This creates a messy and error-prone code base which is especially problematic when modularity is a core goal of the system. Future Spell developers are constrained in ways that may not appear obvious at first glance and are not trivially audited. It would be better to force them into a more rigid system by taking advantage of an architecture that reduces boilerplate logic.

A typical `openPosition()` function performs the following sequence of operations:

1. `BlueberryBank.lend()` - `User` provides `underlyingToken` as isolated collateral.
2. `BlueberryBank.borrow()` - `Spell` receives `debtToken` from the money market.
3. “deposit” - this is a black box unique to each `Spell`. It must consume `debtToken` and return `vaultToken`.
4. `BasicSpell._validateMaxLTV()`
5. `BasicSpell._validatePosSize()`
6. `BlueberryBank.takeCollateral()` + `Wrapper.burn()` - `Spell` receives additional `vaultToken`, `User` receives rewards to date.
7. `Wrapper.mint()` + `BlueberryBank.putCollateral()` - `BlueberryBank` receives `collToken`.

All `openPosition()` workflows begin with the `User`'s `underlyingToken` and end with the `BlueberryBank` holding `collToken`. Each Spell contains identical logic for lending, borrowing, validating parameters, wrapping, and posting collateral; the only difference among Spells is how exactly the `debtToken` gets converted to `vaultToken`.

The Spell framework could be rewritten with a singleton `openPosition()` function. Each Spell would simply need to provide a `deposit()` hook which is executed in step #3 above.

Similarly, a typical `closePosition()` function performs the following sequence of operations:

1. `BlueberryBank.takeCollateral()` + `Wrapper.burn()` - `Spell` receives `vaultToken`.
2. “redeem” - this is a black box unique to each `Spell`. It must consume `vaultToken` and return `debtToken`.
3. `BlueberryBank.withdrawLend()` - `Spell` receives `underlyingToken`.
4. Swap `underlyingToken` for `debtToken` via Paraswap.
5. `BlueberryBank.repay()` - the money market receives owed `debtToken`.
6. `BasicSpell._validateMaxLTV()`
7. `BasicSpell._doRefund()` - `user` receives any remaining `underlyingToken` and `debtToken`.

All `closePosition()` workflows begin with the `BlueberryBank`'s `collToken`, which the `Spell` receives as `vaultToken`; they end with the `User` receiving `underlyingToken` (plus any excess `debtToken`). It is clear that this is a mirror image of the `openPosition()` workflow. Each Spell contains identical logic for removing collateral, unwrapping, withdrawing, swapping, repaying, validating parameters, and refunding; the only difference among Spells is how exactly the `vaultToken` gets converted to `debtToken`.

The Spell framework could be rewritten with a singleton `closePosition()` function. Each Spell would simply need to provide a `redeem()` hook which is executed in step #2 above.

Rewriting the Spell framework as described would create a truly modular architecture in which it is almost trivial to write and verify new Spells, which would undoubtedly bolster the Blueberry developer ecosystem.

#### Recommendation

Rewrite the Spell framework as singleton `openPosition()` and `closePosition()` functions where each Spell simply provides `deposit()` and `redeem()` hooks to fill the middle of the respective workflows.

### [I-06] Insufficient `Spell.closePosition()` conditions open possibility of future Spell collisions with undefined behavior.

#### Status

Acknowledged.

#### Description

All Spells include a pattern similar to [this one from `ShortLongSpell.closePosition()`](https://github.com/Blueberryfi/blueberry-core/blob/408d02f2ec7475b795a7fe461aa1d435bc260b7d/contracts/spell/ShortLongSpell.sol#L114-L115):

```solidity
if (IWERC20(posCollToken).getUnderlyingToken(collId) != vault) revert Errors.INCORRECT_UNDERLYING(vault);
if (posCollToken != address(werc20)) revert Errors.INCORRECT_COLTOKEN(posCollToken);
```

In short, these checks enforce that the provided `positionId` corresponds to a position whose parameters match those of this particular Spell and `strategyId`. The two parameters verified here are the `vaultToken` and the `collToken`.

The spirit of these checks is to validate that the given `positionId` represents a position that was opened by this [Spell, `strategyId`] pair. Only if it was opened by this Spell and `strategyId` should it be closed by the same.

But while the conditions above may be a strong enough proxy today, this may not always be the case (pending future Spell development). What if two Spells use the same type of collateral Wrapper, or two strategies use the same `vaultToken`? The potential consequences of opening a position with one Spell/strategy and closing it with another are unknowable.

Ideally, the `BlueberryBank`'s `Position` data should include which specific Spell address opened the position, and to which `strategyId` it corresponds. Checking these instead of the proxy conditions above would ensure that it is impossible to close a position unless it was opened in the same way.

#### Recommendation

The `Position` data should include which Spell (and even which `strategyId`) opened the position so that it can be properly closed.

### [I-07] `BlueberryBank.accrue()` does not use the return value from `BErc20.borrowBalanceCurrent()`.

#### Status

Acknowledged.

#### Description

The purpose of [`BlueberryBank.accrue()`](https://github.com/Blueberryfi/blueberry-core/blob/408d02f2ec7475b795a7fe461aa1d435bc260b7d/contracts/BlueberryBank.sol#L380-L385) is to update accrued money market interest for the `bToken` corresponding to the given `token`:

```solidity
function accrue(address token) public override {
    Bank storage bank = _banks[token];
    if (!bank.isListed) revert Errors.BANK_NOT_LISTED(token);
    IBErc20(bank.bToken).borrowBalanceCurrent(address(this));
}
```

Perplexingly, it does this by making a call to `BErc20.borrowBalanceCurrent()`, and it ignores the return value. `BErc20.borrowBalanceCurrent()` is a storage-writing call that returns the latest borrow balance. It does this by first updating accrued interest using `BErc20.accrueInterest()`, and then returning the (newly updated) borrow balance from storage.

We can see quite easily that the money market already has a function for updating interest-related storage variables: `accrueInterest()`. Since we do not care about the return value of `borrowBalanceCurrent()`, we are wasting a storage read by making that call here.

#### Recommendation

Use `BErc20.accrueInterest()` instead of `BErc20.borrowBalanceCurrent()`.

### [I-08] Avoid hard-coding values that have no meaning to the reader.

#### Status

Acknowledged.

#### Description

The `AuraSpell` encodes a Balancer join [like this](https://github.com/Blueberryfi/blueberry-core/blob/408d02f2ec7475b795a7fe461aa1d435bc260b7d/contracts/spell/AuraSpell.sol#L123-L133):

```solidity
vault.joinPool(
    _wAuraBooster.getBPTPoolId(lpToken),
    address(this),
    address(this),
    IBalancerVault.JoinPoolRequest({
        assets: tokens,
        maxAmountsIn: maxAmountsIn,
        userData: abi.encode(1, amountsIn, _minimumBPT), // @audit What is 1???
        fromInternalBalance: false
    })
);
```

This includes a hard-coded value that is not at all self-explanatory for the reader: the value of 1 actually corresponds to `JoinKind.EXACT_TOKENS_IN_FOR_BPT_OUT`.

Similarly, the exit is encoded as follows:

```solidity
_wAuraBooster.getVault().exitPool(
    IBalancerV2Pool(lpToken).getPoolId(),
    address(this),
    address(this),
    IBalancerVault.ExitPoolRequest(
        tokens,
        minAmountsOut,
        abi.encode(0, amountPosRemove, borrowTokenIndex), // @audit What is 0???
        false
    )
);
```

Here, 0 corresponds to `ExitKind.EXACT_BPT_IN_FOR_ONE_TOKEN_OUT`.

The reader has no context for these values and must go digging in the Balancer code base to understand.

#### Recommendation

Always import a third-party interface when dealing with obscure constants like these. This will vastly improve code readability.

### [I-9] Checking `_validatePosSize()` in the middle of `Spell.openPosition()` creates unnecessary complexity.

#### Status

Acknowledged.

#### Description

As noted in [I-05](#i-05-major-code-duplication-across-spells), a typical Spell's `openPosition()` function is structured like this:

1. `BlueberryBank.lend()` - `User` provides `underlyingToken` as isolated collateral.
2. `BlueberryBank.borrow()` - `Spell` receives `debtToken` from the money market.
3. “deposit” - this is a black box unique to each `Spell`. It must consume `debtToken` and return `vaultToken`.
4. `BasicSpell._validateMaxLTV()`
5. `BasicSpell._validatePosSize()`
6. `BlueberryBank.takeCollateral()` + `Wrapper.burn()` - `Spell` receives additional `vaultToken`, `User` receives rewards to date.
7. `Wrapper.mint()` + `BlueberryBank.putCollateral()` - `BlueberryBank` receives `collToken`.

The locations of those validation functions (steps #4 and #5) in the sequence are perplexing. First, the maximum LTV could be validated immediately after lending and borrowing, so why doesn't it occur prior to the "deposit" hook? But that's somewhat irrelevant and would only serve to minimize gas consumption on failed calls; the most logical location for both validators would be the very end of the operation.

The location of `_validatePosSize()` is the most unusual because it checks the size of a position which is not yet fully populated. It won't be fully populated until after `BlueberryBank.putCollateral()`, so again, the logical location for this check would be the end of the workflow.

And because it is called prior to the position being fully populated, it contains convoluted logic to assess the size of not only the position itself (`collToken`) but also the intermediate, undeposited position (`vaultToken`). The [definition in `BasicSpell`](https://github.com/Blueberryfi/blueberry-core/blob/408d02f2ec7475b795a7fe461aa1d435bc260b7d/contracts/spell/BasicSpell.sol#L281-L311) is needlessly long and contains superfluous external calls which cost gas:

```solidity
function _validatePosSize(uint256 strategyId) internal view {
    IBank bank = getBank();
    Strategy memory strategy = _strategies[strategyId];
    IBank.Position memory pos = bank.getCurrentPositionInfo();

    /// Get previous position size
    uint256 prevPosSize;
    if (pos.collToken != address(0)) {
        prevPosSize = bank.getOracle().getWrappedTokenValue(pos.collToken, pos.collId, pos.collateralSize);
    }

    /// Get newly added position size
    uint256 addedPosSize;
    IERC20 lpToken = IERC20(strategy.vault);
    uint256 lpBalance = lpToken.balanceOf(address(this));
    uint256 lpPrice = bank.getOracle().getPrice(address(lpToken));

    addedPosSize = (lpPrice * lpBalance) / 10 ** IERC20MetadataUpgradeable(address(lpToken)).decimals();

    // Check if position size is within bounds
    if (prevPosSize + addedPosSize > strategy.maxPositionSize) {
        revert Errors.EXCEED_MAX_POS_SIZE(strategyId);
    }
    if (prevPosSize + addedPosSize < strategy.minPositionSize) {
        revert Errors.EXCEED_MIN_POS_SIZE(strategyId);
    }
}
```

#### Recommendation

Validate position size at the very end of `Spell.openPosition()` and remove excessive logic from `BasicSpell._validatePosSize()`.

### [I-10] Consider removing support for native ETH unless absolutely necessary.

#### Status

Acknowledged.

#### Description

The `BasicSpell._doBorrow()` function contains [this block](https://github.com/Blueberryfi/blueberry-core/blob/408d02f2ec7475b795a7fe461aa1d435bc260b7d/contracts/spell/BasicSpell.sol#L393-L403) of code which adds support for borrowing native ETH:

```solidity
bool isETH = IERC20(token).isETH();

IBank bank = getBank();
address weth = getWETH();

if (isETH) {
    borrowedAmount = bank.borrow(weth, amount);
    IWETH(weth).withdraw(borrowedAmount);
} else {
    borrowedAmount = bank.borrow(token, amount);
}
```

A similar block exists in `BasicSpell._doRepay()`.

This does not cause any explicit issues, but it is unclear why it is necessary to enable native ETH borrowing. Native asset handling in smart contracts should be considered a liability in general. And typically, it is included as a matter of UX improvement, but in the Blueberry system the end user never receives borrowed tokens; they exist only in an intermediate state within Spell contracts. So this complexity does not seem beneficial here.

On the contrary, it opens the door for poorly written Spells to suffer reentrancy attacks from ETH transfer callbacks.

#### Recommendation

Remove support for native ETH.

### [I-11] Behavioral mismatch: refunding vs selling reward tokens.

#### Status

Acknowledged.

#### Description

Spell behavior is inconsistent when it comes to distributing rewards. On some occasions, rewards are refunded directly to users, whereas on others they are sold so that the user receives a lump sum of `underlyingToken` or `debtToken` instead.

Rewards are sold in these instances:
* [AuraSpell.closePositionFarm()](https://github.com/Blueberryfi/blueberry-core/blob/408d02f2ec7475b795a7fe461aa1d435bc260b7d/contracts/spell/AuraSpell.sol#L195)
* [ConvexSpell.closePositionFarm()](https://github.com/Blueberryfi/blueberry-core/blob/408d02f2ec7475b795a7fe461aa1d435bc260b7d/contracts/spell/ConvexSpell.sol#L289)
* [ConvexSpellV2.closePositionFarm()](https://github.com/Blueberryfi/blueberry-core/blob/408d02f2ec7475b795a7fe461aa1d435bc260b7d/contracts/spell/ConvexSpellV2.sol#L290)

Rewards are refunded in these instances:
* [AuraSpell.openPositionFarm()](https://github.com/Blueberryfi/blueberry-core/blob/408d02f2ec7475b795a7fe461aa1d435bc260b7d/contracts/spell/AuraSpell.sol#L157)
* [ConvexSpell.openPositionFarm()](https://github.com/Blueberryfi/blueberry-core/blob/408d02f2ec7475b795a7fe461aa1d435bc260b7d/contracts/spell/ConvexSpell.sol#L246)
* [ConvexSpellV2.openPositionFarm()](https://github.com/Blueberryfi/blueberry-core/blob/408d02f2ec7475b795a7fe461aa1d435bc260b7d/contracts/spell/ConvexSpellV2.sol#L247)
* [IchiSpell.openPositionFarm()](https://github.com/Blueberryfi/blueberry-core/blob/408d02f2ec7475b795a7fe461aa1d435bc260b7d/contracts/spell/IchiSpell.sol#L143)
* [IchiSpell.closePositionFarm()](https://github.com/Blueberryfi/blueberry-core/blob/408d02f2ec7475b795a7fe461aa1d435bc260b7d/contracts/spell/IchiSpell.sol#L184)

The behavioral difference seems arbitrary and could be confusing to end users who expect more uniformity.

#### Recommendation

Decide on a uniform pattern for distributing rewards.

# Attribution
- Special thanks to both [Cyfrin](https://github.com/Cyfrin/audit-report-templating) and [pashov](https://github.com/pashov/audits) whose open source report templates served as inspiration.
- Special thanks to both [CodeHawks](https://docs.codehawks.com/hawks-auditors/how-to-evaluate-a-finding-severity) and [Cantina](https://docs.cantina.xyz/cantina-docs/enter-the-cantina/cantina-competitions/finding-severities) for their severity classification guidelines.