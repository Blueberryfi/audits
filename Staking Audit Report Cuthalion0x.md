# Blueberry StakeVest - Security Review

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
- [Attribution](#attribution)

# Protocol Summary

The Blueberry staking module distributes the first wave of `BLB` governance tokens to users of the Blueberry protocol who stake eligible `ibToken` lending positions. `bdBLB` rewards are accounted proportionally to each user's staked amount and staking duration throughout an initial 60-day "lockdrop" period, after which they can be claimed. These `bdBLB` balances are then convertible to actual `BLB` tokens over a one-year vesting period. They can be distributed at any point during this period, but a double penalty is assessed proportionally to the amount of time remaining in the vesting period. The first component of the penalty is assessed directly on the `BLB` token balance, leaving some `BLB` tokens behind to be redistributed to other vesters; and the second component is assessed in stable coins which are transferred from the user's account into a Blueberry treasury.

# Disclaimer

A smart contract security review can never assert the complete absence of vulnerabilities. We make every effort to find as many vulnerabilities as possible but hold no responsibility for the findings included in or missing from this document. Subsequent security reviews and bug bounty programs are strongly recommended. This review focuses solely on the security aspects of the smart contract implementations and does not constitute an endorsement of the underlying business or product.

# Risk Classification

The risk classification matrix maps (impact, likelihood) pairs to overall severities. The system is imperfect and allows for some subjectivity on the part of the reviewer.

| Severity Level         | Impact: High     | Impact: Medium     | Impact: Low     |
| ---------------------- | ---------------- | ------------------ | --------------- |
| **Likelihood: High**   | High             | High               | Medium          |
| **Likelihood: Medium** | High             | Medium             | Low             |
| **Likelihood: Low**    | Medium           | Low                | Low             |

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

# Executive Summary

- **Repository:** [https://github.com/blueberryfi/blueberry-stakevest](https://github.com/blueberryfi/blueberry-stakevest)
- **Commit hash:** [8d2864e3b0ae5ff718d4fc2527e74c6a35903b72](https://github.com/Blueberryfi/blueberry-stakevest/commit/8d2864e3b0ae5ff718d4fc2527e74c6a35903b72)
- **Report Date:** March 4, 2024

## Scope 

The following smart contracts were in scope for this review:

- [BlueberryStaking.sol](https://github.com/Blueberryfi/blueberry-stakevest/blob/8d2864e3b0ae5ff718d4fc2527e74c6a35903b72/src/BlueberryStaking.sol)

This smart contract is already deployed to mainnet, and for security's sake only minimum viable upgrades should be made moving forward. As such, this review is focused on loss of funds, DOS, and similar vulnerabilities. It ignores quality control issues and gas optimizations.

## Privileged Roles

For the purposes of this review, the following privileged entities are considered trusted. Nonetheless, we still make every effort to report egregious administrative powers that, if misused even accidentally, could render the system inoperable. "Egregious" is subjective.

- `BlueberryStaking` has a single `OWNER` with authority to reconfigure almost any aspect of the system, including but not limited to: the reward token, the set of staking tokens, the reward amounts, the staking and vesting schedules, and early claim fees.

## Findings Summary

| Severity      | Count |
| ------------- | ----- |
| High          | 3     |
| Medium        | 8     |
| Low           | 10    |

# Findings

## High

### [H-01] Erasure of user's `vest.amount` when `redistributedBLB` is non-zero.

#### Status

Fixed by [PR 7](https://github.com/Blueberryfi/blueberry-staking/pull/7).

#### Description

**NOTE: This finding has already been reported through both a previous review and a bug bounty program. But as of commit `8d2864e3b0ae5ff718d4fc2527e74c6a35903b72`, it has not yet been fixed, so a simplified report is provided here for the sake of completeness.**

At [BlueberryStaking.sol:303](https://github.com/Blueberryfi/blueberry-stakevest/blob/8d2864e3b0ae5ff718d4fc2527e74c6a35903b72/src/BlueberryStaking.sol#L303-L307), the following code block is intended to distribute some of the `redistributedBLB` to the caller:

```solidity
if (epochs[_vestEpoch].redistributedBLB > 0) {
    vest.amount =
        (vest.amount * epochs[_vestEpoch].redistributedBLB) /
        epochs[_vestEpoch].totalBLB;
}
```

It aims to increase the caller's `vest.amount` proportionally to their share of the total `bdBLB` allocated for the given `_vestEpoch`. However, the code contains a typo which performs a strict assignment (`=`) rather than incrementing (`+=`).

#### Recommendation

Change `=` to `+=`.

```diff
  if (epochs[_vestEpoch].redistributedBLB > 0) {
-     vest.amount =
+     vest.amount +=
          (vest.amount * epochs[_vestEpoch].redistributedBLB) /
          epochs[_vestEpoch].totalBLB;
  }
```

### [H-02] Under-distribution of `redistributedBLB` locks excess `BLB` in the staking contract.

#### Status

Fixed by [PR 8](https://github.com/Blueberryfi/blueberry-staking/pull/8).

#### Description

**NOTE: Because [H-01](#h-01-erasure-of-users-vestamount-when-redistributedblb-is-non-zero) so severely miscalculates `vest.amount`, this issue will not clearly manifest until after H-01 is fixed.**

When a user calls `startVesting()`, the value of `epochs[n].totalBLB` is incremented to account for the new balance of `bdBLB`. But when `BLB` tokens are distributed to the caller of either `startVesting()` or `accelerateVesting()`, the value of `epochs[n].totalBLB` remains unchanged.

If `epochs[n].totalBLB` is not reduced when a user's `bdBLB` position is destroyed, this will negatively impact the accounting of redistributed `BLB` tokens in `updateVests()`:

```solidity
// @audit Hypothetical version of this code block after fixing H-01.
if (epochs[_vestEpoch].redistributedBLB > 0) {
    vest.amount +=
        (vest.amount * epochs[_vestEpoch].redistributedBLB) /
        epochs[_vestEpoch].totalBLB;
}
```

The over-valued `epochs[_vestEpoch].totalBLB` will ensure that `epochs[_vestEpoch].redistributedBLB` is never fully distributed. Whatever tokens remain after all vested positions are closed will be locked in the `BlueberryStaking` contract unless it is upgraded.

#### Proof of Concept

Because the issue cannot be exploited until after H-01 is fixed, this proof of concept is written in procedural verbiage rather than source code. The PoC assumes that H-01 has been fixed in the manner suggested in this report.

Consider the following scenario, which tracks the would-be values of relevant state variables throughout:

##### 1. Alice: `startVesting()` at epoch N

```solidity
epochs[n].totalBLB = vesting[alice][0].amount
```

##### 2. Alice: `accelerateVesting()` at epoch N

```solidity
// becomes non-zero, actual value unimportant
epochs[n].redistributedBLB = ...
```

(This operation also withdraws Alice's `BLB` tokens but does not modify `epochs[n].totalBLB` to account for the withdrawal.)

##### 3. Bob: `startVesting()` at epoch N

```solidity
epochs[n].totalBLB =
    vesting[alice][0].amount +
    vesting[bob][0].amount
```

##### 4. Bob: `completeVesting()` in the far future

As the only remaining shareholder, Bob should be entitled to 100% of the value of `epochs[n].redistributedBLB`. But he can only extract a portion of it because `epochs[n].totalBLB` still includes Alice's withdrawn share:

```solidity
vesting[bob][0].amount +=
    (vesting[bob][0].amount * epochs[n].redistributedBLB) /
    epochs[n].totalBLB
```

So, the protocol fails to distribute Bob's total value and locks some `BLB` tokens in the contract.

#### Recommendation

Account for withdrawn `BLB` tokens in `epochs[n].totalBLB`. Consider the following pseudo-code:

```solidity
// Context: updateVests()
// Leave `vest.amount` unchanged here.
vest.extra =
    (vest.amount * epochs[_vestEpoch].redistributedBLB) /
    epochs[_vestEpoch].totalBLB;

// Context: completeVesting() or accelerateVesting()
totalbdblb += vest.amount + vest.extra;
epochs[_vestEpoch].totalBLB -= vest.amount;
delete vest;
blb.transfer(msg.sender, totalbdblb);
```

### [H-03] Over-distribution of `redistributedBLB` leads to insolvency of `BlueberryStaking` contract and leaves some users unable to claim.

#### Status

Fixed by [PR 9](https://github.com/Blueberryfi/blueberry-staking/pull/9).

#### Description

**NOTE: Because both [H-01](#h-01-erasure-of-users-vestamount-when-redistributedblb-is-non-zero) and [H-02](#h-02-under-distribution-of-redistributedblb-locks-excess-blb-in-the-staking-contract) contribute to misallocation of `redistributedBLB`, this issue will not clearly manifest until after both H-01 and H-02 are fixed.**

When a user calls `accelerateVesting()`, the value of `epochs[n].redistributedBLB` is incremented to account for the balance of `bdBLB` left behind for the acceleration fee. But when `BLB` tokens are distributed to the caller of either `startVesting()` or `accelerateVesting()`, the value of `epochs[n].redistributedBLB` remains unchanged.

If `epochs[n].redistributedBLB` is not reduced when a user's `bdBLB` position is destroyed, this will negatively impact the accounting of redistributed `BLB` tokens in `updateVests()`:

```solidity
// @audit Hypothetical version of this code block after fixing H-01.
if (epochs[_vestEpoch].redistributedBLB > 0) {
    vest.amount +=
        (vest.amount * epochs[_vestEpoch].redistributedBLB) /
        epochs[_vestEpoch].totalBLB;
}
```

The over-valued `epochs[_vestEpoch].redistributedBLB` will ensure that earlier withdrawers receive too many tokens and later withdrawers are left with reverting withdrawal transactions as the `BlueberryStaking` contract becomes insolvent.

#### Proof of Concept

Because the issue cannot be neatly exploited until after both H-01 and H-02 are fixed, this proof of concept is written in procedural verbiage rather than source code. The PoC assumes that both H-01 and H-02 have been fixed in the manner suggested in this report.

Consider the following scenario, which tracks the would-be values of relevant state variables throughout:

##### 1. Alice: `startVesting()` at epoch N

```solidity
epochs[n].totalBLB = vesting[alice][0].amount
```

##### 2. Bob: `startVesting()` at epoch N

```solidity
epochs[n].totalBLB =
    vesting[alice][0].amount +
    vesting[bob][0].amount
```

##### 3. Alice: `accelerateVesting()` at epoch N

```solidity
// increases, exact value unimportant
epochs[n].redistributedBLB++
epochs[n].totalBLB = vesting[bob][0].amount
```

##### 4. Bob: `accelerateVesting()` at epoch N

```solidity
// increases, exact value unimportant
epochs[n].redistributedBLB++
epochs[n].totalBLB = 0
```

(This operation also withdraws Bob's share of `epochs[n].redistributedBLB` but does not reduce `epochs[n].redistributedBLB` to account for the withdrawal.)

##### 5. Charlie: `startVesting()` at epoch N

```solidity
epochs[n].totalBLB = vesting[charlie][0].amount
```

##### 6. Charlie: `completeVesting()` in the far future

As the only remaining shareholder, Charlie is entitled to 100% of the value of `epochs[n].redistributedBLB`. But some of these `BLB` tokens have already been distributed to Bob without accounting for it, so the contract lacks the funds to pay Charlie.

```solidity
// @audit `vesting[charlie][0].amount` becomes greater than the contract's balance of `BLB` tokens.
vesting[charlie][0].amount +=
    (vesting[charlie][0].amount * epochs[n].redistributedBLB) /
    epochs[n].totalBLB
```

The transaction will revert, so Charlie will be denied service.

#### Recommendation

Account for withdrawn `BLB` tokens in `epochs[n].redistributedBLB`. Consider the following pseudo-code:

```solidity
// Context: updateVests()
// Leave `vest.amount` unchanged here.
vest.extra =
    (vest.amount * epochs[_vestEpoch].redistributedBLB) /
    epochs[_vestEpoch].totalBLB;

// Context: completeVesting() or accelerateVesting()
totalbdblb += vest.amount + vest.extra;
epochs[_vestEpoch].totalBLB -= vest.amount;
epochs[_vestEpoch].redistributedBLB -= vest.extra;
delete vest;
blb.transfer(msg.sender, totalbdblb);
```

## Medium

### [M-01] Missing `_disableInitializers()` in constructor of upgradeable implementation contract.

#### Status

Fixed by [PR 11](https://github.com/Blueberryfi/blueberry-staking/pull/11).

#### Description

OpenZeppelin [officially recommends](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/7417c5946f8a213a8e61eca8d3c5247bf3854249/contracts/proxy/utils/Initializable.sol#L39-L52) adding `_disableInitializers()` to the implementation contract behind any upgradeable proxy.

This is most critical for UUPS proxies, whose implementation contracts contain their own upgrade logic. But it is considered a best practice for all upgradeable contracts because the impact of the issue is difficult to determine. It is best to err on the side of caution and include this one-liner.

#### Recommendation

Add `_disableInitializers()` to the constructor:

```diff
- constructor() {}
+ constructor() {
+     _disableInitializers();
+ }
```

### [M-02] Pausing the system locks user funds indefinitely.

#### Status

Fixed by [PR 19](https://github.com/Blueberryfi/blueberry-staking/pull/19).

#### Description

The `owner` has access to a `pause()` function which prevents any of the main system functions from being called by anyone. This exists for the worst-case scenario in which a vulnerability is identified in production code, and the pause mechanism can be used to put the system on hold until a mitigation (usually a contract upgrade) can be put in place.

However, the best practice would be to constrain the entire system with the exception of a lone withdrawal function that allows users to exit unscathed. Otherwise, the admin puts all user funds at unnecessary risk by locking them indefinitely.

In this system, the `unstake()` function is included within the pausable set, and so users are left with their principal funds stuck in the contract during mitigation events. A simple contract upgrade error could then permanently lock user funds in the contract. It is best if users have access to `unstake()`, or some logic-minimized version of it, even while the contracts are paused. This way they can safely exit the system at any time.

#### Recommendation

Remove the `whenNotPaused` modifier from `unstake()`, or else add an `emergencyUnstake()` function that allows users to withdraw their principal without updating any rewards accounting. Be careful to consider the consequences that an `emergencyUnstake()` function might have after the contract is unpaused.

### [M-03] `BLB` token price schedule does not align with documentation.

#### Status

Fixed by [PR 12](https://github.com/Blueberryfi/blueberry-staking/pull/12).

#### Description

The price schedule of the `BLB` token is documented as follows:

- Days  0-29: $0.02
- Days 30-59: $0.04
- Days   60+: TWAP

But due to a typo in the `getPrice()` function, it actually returns:

- Days  0-59: $0.02
- Days 60-89: $0.04
- Days   90+: TWAP

In other words, the price is fixed at $0.02 for the first 60 days instead of only the first 30.

#### Recommendation

Change `<=` to `<`:

```diff
  // month 1: $0.02 / blb
- if (_month <= 1) {
+ if (_month < 1) {
      _price = 0.02e18;
  }
  // month 2: $0.04 / blb
- else if (_month <= 2 || uniswapV3Pool == address(0)) {
+ else if (_month < 2 || uniswapV3Pool == address(0)) {
      _price = 0.04e18;
  }
```

### [M-04] Miscalculated `rewardRate` due to premature update of `finishAt`.

#### Status

Fixed by [PR 18](https://github.com/Blueberryfi/blueberry-staking/pull/18).

#### Description

The `modifyRewardAmount()` function contains a loop at [BlueberryStaking.sol:732](https://github.com/Blueberryfi/blueberry-stakevest/blob/8d2864e3b0ae5ff718d4fc2527e74c6a35903b72/src/BlueberryStaking.sol#L732-L750):

```solidity
for (uint256 i; i < _ibTokens.length; ++i) {
    address _ibToken = _ibTokens[i];
    uint256 _amount = _amounts[i];

    if (block.timestamp > finishAt) {
        rewardRate[_ibToken] = _amount / rewardDuration;
    } else {
        uint256 remaining = finishAt - block.timestamp;
        uint256 leftover = remaining * rewardRate[_ibToken];
        rewardRate[_ibToken] = (_amount + leftover) / rewardDuration;
    }

    if (rewardRate[_ibToken] == 0) {
        revert InvalidRewardRate();
    }

    finishAt = block.timestamp + rewardDuration;
    lastUpdateTime[_ibToken] = block.timestamp;
}
```

The conditional within this loop ensures that any dangling rewards are still paid out through the new period, in addition to any incoming amounts. But the condition depends on the value of the storage variable `finishAt` prior to entering the loop, whereas `finishAt` is actually updated within the loop itself.

As a result, this condition will evaluate properly on the first token in the `_ibTokens` array, and then `finishAt` will be updated. All subsequent loop iterations will enter the `else` block of the condition, regardless of the initial value of `finishAt`. When they do, they will evaluate like this:

```solidity
// Computed at the end of the previous loop.
finishAt = block.timestamp + rewardDuration;

// Computed in the current loop.
uint256 remaining = finishAt - block.timestamp;
                  = (block.timestamp + rewardDuration) - block.timestamp;
                  = rewardDuration;

uint256 leftover = remaining * rewardRate[_ibToken];
                 = rewardDuration * rewardRate[_ibToken];

rewardRate[_ibToken] = (_amount + leftover) / rewardDuration;
                     = (_amount + (rewardDuration * rewardRate[_ibToken])) / rewardDuration;
                     = rewardRate[_ibToken] + (_amount / rewardDuration);
```

The final value of `rewardRate[_ibToken]` is equivalent to the old rate plus the new rate. It should instead be equivalent to the old pro-rated (for time remaining in the previous period) rate plus the new rate.

Effectively, this bug causes the code to behave as though the previous rewards were never paid out, and the period is just starting now. The resulting rates will be far higher than the caller likely intends. In the event that the previous reward period has already ended, the full balance of those rewards will be paid out again, in addition to any incoming amounts.

For example, consider the following scenario:

- 100 tokens are earmarked for rewards distribution over a period of 100 days.
- 90 days pass, leaving 90 tokens distributed and 10 left over.
- `modifyRewardAmount()` is called with a new amount of 100 tokens (for another 100 days).
- The **expected** outcome is to distribute **110 tokens** over 100 days (100 new + 10 leftover).
- The **actual** outcome is to distribute **200 tokens** over 100 days (100 new + 100 original).

#### Recommendation

Update `finishAt` after the loop, not within the loop.

```diff
  for (uint256 i; i < _ibTokens.length; ++i) {
      address _ibToken = _ibTokens[i];
      uint256 _amount = _amounts[i];
  
      if (block.timestamp > finishAt) {
          rewardRate[_ibToken] = _amount / rewardDuration;
      } else {
          uint256 remaining = finishAt - block.timestamp;
          uint256 leftover = remaining * rewardRate[_ibToken];
          rewardRate[_ibToken] = (_amount + leftover) / rewardDuration;
      }
  
      if (rewardRate[_ibToken] == 0) {
          revert InvalidRewardRate();
      }
  
-     finishAt = block.timestamp + rewardDuration;
      lastUpdateTime[_ibToken] = block.timestamp;
  }
  
+ finishAt = block.timestamp + rewardDuration;
  
  emit RewardAmountModified(_ibTokens, _amounts, block.timestamp);
```

### [M-05] Removing an `ibToken` locks users' staked tokens indefinitely.

#### Status

Fixed by [PR 18](https://github.com/Blueberryfi/blueberry-staking/pull/18).

#### Description

The `removeIbTokens()` function sets `isIbToken` to `false` for any tokens that `owner` passes in as arguments. When `isIbToken` is `false`, calls to the `unstake()` function become impossible for the given token because `isIbToken` is checked inside `_updateRewards()` at [BlueberryStaking.sol:210](https://github.com/Blueberryfi/blueberry-stakevest/blob/8d2864e3b0ae5ff718d4fc2527e74c6a35903b72/src/BlueberryStaking.sol#L210-L212).

With calls to `unstake()` being prohibited, and no other avenue available to withdraw staked tokens, users' deposits are locked indefinitely. Only a contract upgrade or `owner` reinstating the removed tokens could enable withdrawals.

#### Recommendation

Delete the `removeIbToken()` function, or consider setting a per-token `finishAt` variable to stop new rewards from accruing without impacting existing deposits.

### [M-06] Removing an `ibToken` erases all unclaimed user rewards.

#### Status

Fixed by [PR 18](https://github.com/Blueberryfi/blueberry-staking/pull/18).

#### Description

The `removeIbTokens()` function sets `isIbToken` to `false` for any tokens that `owner` passes in as arguments. When `isIbToken` is `false`, calls to the `startVesting()` function become impossible for the given token because `isIbToken` is checked inside `_updateRewards()` at [BlueberryStaking.sol:210](https://github.com/Blueberryfi/blueberry-stakevest/blob/8d2864e3b0ae5ff718d4fc2527e74c6a35903b72/src/BlueberryStaking.sol#L210-L212).

As a result, it is not possible to create new vesting positions from previously accrued rewards, and so all rewards earned up to the `removeIbTokens()` call are forfeit.

#### Recommendation

Delete the `removeIbToken()` function, or consider setting a per-token `finishAt` variable to stop new rewards from accruing without zeroing old rewards.

### [M-07] Undefined behavior: if epoch length changes after vesting has already begun, all prior epoch-based accounting will be invalidated.

#### Status

Fixed by [PR 10](https://github.com/Blueberryfi/blueberry-staking/pull/10).

#### Description

The `owner` has access to a `changeEpochLength()` function which is needlessly dangerous, even for a trusted administrator. This function sets the `epochLength` to whatever arbitrary value the `owner` chooses, which could be shorter or longer than the current `epochLength`.

Epoch-based accounting is used for redistributing `BLB` token rewards as part of the early withdrawal penalty. It depends on two quantities: `totalBLB` and `redistributedBLB`. When the `epochLength` is changed, all previous accounting of these variables is invalidated, resulting in undefined behavior which depends on the exact nature of the change in length. Consider the following code block:

```solidity
// @audit Hypothetical version of this code block after fixing H-01.
if (epochs[_vestEpoch].redistributedBLB > 0) {
    vest.amount +=
        (vest.amount * epochs[_vestEpoch].redistributedBLB) /
        epochs[_vestEpoch].totalBLB;
}
```

The `epochs[_vestEpoch].totalBLB` is intended to be a summation of all users' `vest.amount` for the given `_vestEpoch`. But `epochs[_vestEpoch].totalBLB` is defined when positions are opened within `startVesting()`, whereas individual users' vesting positions are not assigned epochs until those positions are closed. In `completeVesting()` and `accelerateVesting()`, user epochs are computed on the fly:

```solidity
uint256 _vestEpoch = (vest.startTime - deployedAt) / epochLength;
```

This creates a mismatch of epoch accounting which holds as long as the `epochLength` remains constant but breaks when it changes. It could even create a situation where `vest.amount` is larger than `epochs[_vestEpoch].totalBLB`, in which the calling user would receive `redistributedBLB` worth more than the total pot of `redistributedBLB`.

#### Recommendation

Do not ever call the `changeEpochLength()` function after any user has called `startVesting()`. Consider removing `changeEpochLength()` altogether or adding a `lock()` function that prevents certain state variables from being updated once configuration is finalized.

### [M-08] Under-distribution of `redistributedBLB` if no user vests in the corresponding epoch.

#### Status

Fixed by [PR 10](https://github.com/Blueberryfi/blueberry-staking/pull/10).

#### Description

Accelerated vest penalties are served from one user to other users via `epochs[n].redistributedBLB`. When a user calls `accelerateVesting()` at some epoch N, some `BLB` rewards are earmarked for all other vesters who opened their positions in that same epoch N.

The system does not account for the possibility that not a single user vested at epoch N, or that all existing vests in epoch N are closed via acceleration. In these cases, some `redistributedBLB` will be accounted but never distributed to users, thus locking `BLB` tokens in the contract.

Notably, the epoch system is not used outside of vest acceleration. Epochs do not factor into staking, reward rates, or fully vested positions. The same accounting of `redistributedBLB` and `totalBLB` should (after fixing [H-02](#h-02-under-distribution-of-redistributedblb-locks-excess-blb-in-the-staking-contract) and [H-03](#h-03-over-distribution-of-redistributedblb-leads-to-insolvency-of-blueberrystaking-contract-and-leaves-some-users-unable-to-claim)) be possible at a global level instead of per-epoch, and this would ensure that all `redistributedBLB` are eventually distributed (unless the very last remaining vester chooses to accelerate their position).

#### Recommendation

Remove the epoch system altogether. The same redistribution of accelerated vest penalties can be accomplished using global `totalBLB` and `redistributedBLB` if properly accounted as in H-01 and H-02.

## Low 

### [L-01] Possibility of misconfiguring `stableDecimals`.

#### Status

Fixed by [commit 03b0bbe](https://github.com/Blueberryfi/blueberry-staking/commit/03b0bbef1d1dbcbc84e7ccc2e9a8e89c52648368).

#### Description

In the `initialize()` function, the caller specifies an arbitrary contract address for the `stableAsset`, but `stableDecimals` is hard-coded to `6`. There is no guarantee that these two values correspond in any way.

The `stableDecimals` value can be reconfigured if `owner` calls `changeStableAddress()`, but still the value is passed in manually as an argument, allowing unnecessary room for error.

Instead, in both cases, the value should be read directly from the `stableAsset` using the `ERC20.decimals()` function. This ensures accurate correspondence and removes any possibility of high-impact decimal accounting errors.

#### Recommendation

Invoke `ERC20.decimals()` when setting `stableAsset`.

### [L-02] Vesting timeline contradicts documentation.

#### Status

Acknowledged.

#### Description

The documentation implies that the vesting timeline looks something like this:
- Days 0-59: staking period
- Days 60-424 (52 weeks): vesting period

In actuality, it is possible to call `startVesting()` as soon as any non-zero `rewards` have accrued, which could be as soon as one block after the staking period begins. With various users' vesting periods beginning somewhere in the range of days 0-59, we should expect that the average vesting period therefore starts on day 30 and ends on day 394. The vesting timeline is therefore about 30 days shorter than advertised.

#### Recommendation

Clarify this in the documentation so that users are not surprised when other users' tokens become fully vested up to 60 days early.

### [L-03] Possible mismatch between `BLB` token pricing and actual lockdrop timeline.

#### Status

Acknowledged.

#### Description

The `getPrice()` function uses a hard-coded timeline in terms of months (30-day increments) from contract deployment. It returns a fixed price in the first month, a different fixed price in the second month, and a TWAP beyond that. The documentation implies that this aligns with the "lockdrop" period during which rewards accrue, but actually `owner` can set an arbitrary `finishAt` time so that rewards finalize either before or after 60 days.

The `BLB` token pricing therefore may not align very well with the actual staking period. It may be better to set pricing relative to `finishAt` rather than relative to a fixed 60 days.

#### Recommendation

Avoid mixing hard-coded timelines with configurable timelines.

### [L-04] The `canClaim()` function provides no protection as it always returns true.

#### Status

Fixed by [PR 10](https://github.com/Blueberryfi/blueberry-staking/pull/10).

#### Description

At [BlueberryStaking.sol:317](https://github.com/Blueberryfi/blueberry-stakevest/blob/8d2864e3b0ae5ff718d4fc2527e74c6a35903b72/src/BlueberryStaking.sol#L317-L319), the `startVesting()` function checks if the caller is eligible to claim in the current epoch. The intention seems to be to prevent a user from claiming more than once in any given epoch.

It is not clear why this is a desirable feature. The user's `rewards` are zeroed after each claim, and this alone should be sufficient to prevent double-claiming. In any case, the `canClaim()` function doesn't actually do anything because of a typo:

```solidity
function canClaim(address _user) public view returns (bool) {
    uint256 _currentEpoch = currentEpoch();
    return lastClaimedEpoch[_user] <= _currentEpoch;
}
```

It should use `<` instead of `<=`. As written, the function returns true even when the `lastClaimedEpoch` matches the `currentEpoch`, which matches the conditions immediately following a claim. There is nothing stopping a user from claiming again as soon as the next block (when new `rewards` accrue).

#### Recommendation

Remove this function entirely, or else reconsider its necessity and change `<=` to `<`.

```diff
  function canClaim(address _user) public view returns (bool) {
      uint256 _currentEpoch = currentEpoch();
-     return lastClaimedEpoch[_user] <= _currentEpoch;
+     return lastClaimedEpoch[_user] < _currentEpoch;
  }
```

### [L-05] Not possible to add a new `ibToken` without reconfiguring `finishAt` for all tokens.

#### Status

Fixed by [PR 18](https://github.com/Blueberryfi/blueberry-staking/pull/18).

#### Description

When `owner` calls `addIbToken()` to introduce a new `ibToken` to the list of eligible staking tokens, the new token is appended to the array and validated within `isIbToken`, but nothing else happens. To `stake()` this token is meaningless until a corresponding `rewardRate` is defined.

To set this `rewardRate`, `owner` must call `modifyRewardAmount()`, which **always** modifies the global `finishAt` time for all tokens. As such, it is not possible to introduce rewards for a new `ibToken` without impacting the rewards schedule of every other token at the same time. The `owner` could very carefully select a new `rewardDuration` before the `modifyRewardAmount()` call which would theoretically leave `finishAt` unchanged, but to actually avoid changing it would require knowing in advance when the transaction will be included in a block, so there is always room for error.

#### Recommendation

Use a per-token `finishAt` or allow setting the `rewardRate` while adding a new token rather than necessitating the extra call to `modifyRewardAmount()`.

### [L-06] Setting reward amounts does not guarantee availability of reward tokens.

#### Status

Fixed by [PR 18](https://github.com/Blueberryfi/blueberry-staking/pull/18).

#### Description

When `owner` calls `modifyRewardAmount()` to set `rewardRate` for each `ibToken`, this activates `rewards` accounting for each user but does not guarantee availability of `BLB` rewards at the future claim time.

The `modifyRewardAmount()` function should transfer `BLB` tokens into the contract from the caller so that users can rest assured the `BLB` tokens will actually be available at claim time. As is, the staking contract is insolvent by default and must manually be made solvent by a `BLB` token donor who may or may not possess the funds necessary to match the full total of promised reward amounts.

#### Recommendation

Add a `blb.transferFrom()` call to `modifyRewardAmount()` to seed the contract with the required sum of reward tokens.

### [L-07] The maximum `basePenaltyRatioPercent` should be 50% rather than 100% because the fee is double-charged.

#### Status

Fixed by [PR 14](https://github.com/Blueberryfi/blueberry-staking/pull/14).

#### Description

The `setBasePenaltyRatioPercent()` function constrains `basePenaltyRatioPercent` between `0` and `1e18` using this block:

```solidity
if (_ratio > 1e18) {
    revert InvalidPenaltyRatio();
}
```

Within this code base, a fee of `1e18` is equivalent to a fee of 100% of principal value, so this may seem like a reasonable upper bound. However, the `basePenaltyRatioPercent` is actually assessed twice: once as the early withdrawal penalty denominated in `BLB` token rewards, and again as the acceleration fee denominated in stable coins. This is exemplified by the following code block from `accelerateVesting()`, where the value is utilized within both `getEarlyUnlockPenaltyRatio()` and `getAccelerationFeeStableAsset()`:

```solidity
// Context: accelerateVesting()
// @audit Note call to getEarlyUnlockPenaltyRatio().
uint256 _earlyUnlockPenaltyRatio = getEarlyUnlockPenaltyRatio(
    msg.sender,
    _vestIndex
);

if (_earlyUnlockPenaltyRatio == 0) {
    revert("Vest complete, nothing to accelerate");
}

// calculate acceleration fee and log it to ensure eth value is sent
// @audit Note call to getAccelerationFeeStableAsset().
uint256 _accelerationFee = getAccelerationFeeStableAsset(
    msg.sender,
    _vestIndex
);

// Context: getAccelerationFeeStableAsset()
// @audit Note call to getEarlyUnlockPenaltyRatio().
uint256 _earlyUnlockPenaltyRatio = getEarlyUnlockPenaltyRatio(
    _user,
    _vestingScheduleIndex
);

accelerationFee =
    ((((_vest.priceUnderlying * _vest.amount) / 1e18) *
        _earlyUnlockPenaltyRatio) / 1e18) /
    (10 ** (18 - stableDecimals));

// Context: getEarlyUnlockPenaltyRatio()
if (_vestTimeElapsed <= 0) {
    // @audit Note basePenaltyRatioPercent used here.
    penaltyRatio = basePenaltyRatioPercent;
}
// If the vesting period is mid-acceleration, calculate the penalty ratio based on the time passed
else if (_vestTimeElapsed < vestLength) {
    penaltyRatio =
        ((vestLength - _vestTimeElapsed).divWad(vestLength) *
            // @audit Note basePenaltyRatioPercent used here.
            basePenaltyRatioPercent) /
        1e18;
}
// If the vesting period is over, return 0
else {
    return 0;
}
```

So, a more reasonable upper bound for `basePenaltyRatioPercent` would be 50%, or `50e16`. Any larger would ensure that users are charged **more** than their total reward value and actually incur a net loss by claiming rewards.

#### Recommendation

```diff
- if (_ratio > 1e18) {
+ if (_ratio > 50e16) {
      revert InvalidPenaltyRatio();
  }
```

### [L-08] Incoming ETH is locked in the contract unless it is upgraded to enable transfers.

#### Status

Fixed by [PR 13](https://github.com/Blueberryfi/blueberry-staking/pull/13).

#### Description

The `BlueberryStaking` contract includes empty `receive()` and `fallback()` functions to accept incoming `ETH` donations, but it lacks the capability to invoke `address.call()`, `address.send()`, or `address.transfer()` to do anything with donated `ETH`. A contract upgrade would be required to enable this functionality.

#### Recommendation

Remove the `receive()` and `fallback()` functions unless and until there is some plan in place to utilize donated `ETH`.

### [L-09] Double-accounting in `getAccumulatedRewards()` if a token is removed from and re-added to `ibTokens`.

#### Status

Fixed by [PR 18](https://github.com/Blueberryfi/blueberry-staking/pull/18).

#### Description

The `removeIbTokens()` function has undesirable side effects (see [M-05](#m-05-removing-an-ibtoken-locks-users-staked-tokens-indefinitely) and [M-06](#m-06-removing-an-ibtoken-erases-all-unclaimed-user-rewards)) that could warrant re-adding tokens that have already been removed. This would be accomplished by calling the `addIbTokens()` function.

Whereas the `addIbTokens()` function pushes new elements to the `ibTokens` array, the `removeIbTokens()` function does not remove them. As a result, the `ibTokens` array strictly grows, and if an element is "removed" and later re-added, it is actually duplicated.

The `getAccumulatedRewards()` function iterates through the entire `ibTokens` array and sums rewards for all valid tokens. This would double-count rewards for any duplicated entries.

#### Recommendation

Consider actually removing elements from the `ibTokens` array when calling `removeIbTokens()`.

### [L-10] Unnecessary storage variable `rewardDuration` creates convoluted and error-prone workflow for admin configuration.

#### Status

Acknowledged.

#### Description

The storage variable `rewardDuration` is set inside the `owner`-only `setRewardDuration()` function and only used inside the `owner`-only `modifyRewardAmount()` function. Given its singular usage, there is no reason for the disconnected setter. In fact, there is no reason to hold a public storage variable at all because external parties get more useful information from `finishAt`.

The concept of `rewardDuration` is tightly coupled to `rewardRate` and should therefore be used as a calldata argument to `modifyRewardAmount()` rather than a storage variable. Otherwise the likelihood of admin error is unacceptably high as each `modifyRewardAmount()` call depends tightly on a prior `setRewardDuration()` call.

#### Recommendation

Move `rewardDuration` into `modifyRewardAmount()` as an argument.

# Attribution
- Special thanks to both [Cyfrin](https://github.com/Cyfrin/audit-report-templating) and [pashov](https://github.com/pashov/audits) whose open source report templates served as inspiration.
- Special thanks to both [CodeHawks](https://docs.codehawks.com/hawks-auditors/how-to-evaluate-a-finding-severity) and [Cantina](https://docs.cantina.xyz/cantina-docs/enter-the-cantina/cantina-competitions/finding-severities) for their severity classification guidelines.