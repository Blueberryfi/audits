# Blueberry Fix Review

### Reviewed by: 0x52 ([@IAm0x52](https://twitter.com/IAm0x52))

### Review Dates: 10/16/23 - 10/18/23

# Disclaimer

The following review was not completed on the entire repo. Review was focuses specifically on PR's created to fix issues uncovered in the [Sherlock Audit Report](https://github.com/sherlock-audit/2023-07-blueberry-judging). These have been reviewed to resolve these specific issues and to prevent introducing new ones in the process. 

# Scope

PR's created to fix issue uncovered in the [Sherlock Audit Report](https://github.com/sherlock-audit/2023-07-blueberry-judging):

- [PR #78](https://github.com/Blueberryfi/blueberry-core/pull/78)
- [PR #79](https://github.com/Blueberryfi/blueberry-core/pull/79)
- [PR #80](https://github.com/Blueberryfi/blueberry-core/pull/80)
- [PR #81](https://github.com/Blueberryfi/blueberry-core/pull/81)
- [PR #82](https://github.com/Blueberryfi/blueberry-core/pull/82)
- [PR #83](https://github.com/Blueberryfi/blueberry-core/pull/83)
- [PR #84](https://github.com/Blueberryfi/blueberry-core/pull/84)
- [PR #85](https://github.com/Blueberryfi/blueberry-core/pull/85)
- [PR #86](https://github.com/Blueberryfi/blueberry-core/pull/86)
- [PR #87](https://github.com/Blueberryfi/blueberry-core/pull/87)
- [PR #89](https://github.com/Blueberryfi/blueberry-core/pull/89)
- [PR #90](https://github.com/Blueberryfi/blueberry-core/pull/90)
- [PR #91](https://github.com/Blueberryfi/blueberry-core/pull/91)
- [PR #92](https://github.com/Blueberryfi/blueberry-core/pull/92)
- [PR #94](https://github.com/Blueberryfi/blueberry-core/pull/94)
- [PR #95](https://github.com/Blueberryfi/blueberry-core/pull/95)
- [PR #97](https://github.com/Blueberryfi/blueberry-core/pull/97)
- [PR #98](https://github.com/Blueberryfi/blueberry-core/pull/98)
- [PR #99](https://github.com/Blueberryfi/blueberry-core/pull/99)
- [PR #100](https://github.com/Blueberryfi/blueberry-core/pull/100)

Deployment Chain(s)
- Ethereum Mainnet

# Summary of Mitigations

|  Identifier  | Title                        | Severity      | Mitigated |
| ------ | ---------------------------- | ------------- | ----- |
| [H-01] | [Stable BPT valuation is incorrect and can be exploited to cause protocol insolvency](#h-01-stable-bpt-valuation-is-incorrect-and-can-be-exploited-to-cause-protocol-insolvency) | High | X |
| [H-02] | [CurveTricryptoOracle incorrectly assumes that WETH is always the last token in the pool which leads to bad LP pricing](#h-02-curvetricryptooracle-incorrectly-assumes-that-weth-is-always-the-last-token-in-the-pool-which-leads-to-bad-lp-pricing) | High | X |
| [H-03] | [CurveTricryptoOracle#getPrice contains math error that causes LP to be priced completely wrong](#h-03-curvetricryptooraclegetprice-contains-math-error-that-causes-lp-to-be-priced-completely-wrong)  | High | X |
| [H-04] | [CVX/AURA distribution calculation is incorrect and will lead to loss of rewards at the end of each cliff](#h-04-cvxaura-distribution-calculation-is-incorrect-and-will-lead-to-loss-of-rewards-at-the-end-of-each-cliff) | High | X |
| [H-05] | [wrong bToken's exchangeRateStored used for calculate ColleteralValue](#h-05-wrong-btokens-exchangeratestored-used-for-calculate-colleteralvalue) | High | X |
| [M-01] | [Users will fail to close their Convex position if the Curve pool is killed](#m-01-users-will-fail-to-close-their-convex-position-if-the-curve-pool-is-killed) | Med | X |
| [M-02] | [getPrice in WeightedBPTOracle.sol uses totalSupply for price calculations which can lead to wrong results](#m-02-getprice-in-weightedbptoraclesol-uses-totalsupply-for-price-calculations-which-can-lead-to-wrong-results) | Med | X |
| [M-03] | [ConvexSpell/CurveSpell.openPositionFarm will revert in some cases](#m-03-convexspellcurvespellopenpositionfarm-will-revert-in-some-cases) | Med | X |
| [M-04] | [AMainnet oracles are incompatible with wstETH causing many popular yields strategies to be broken](#m-04-mainnet-oracles-are-incompatible-with-wsteth-causing-many-popular-yields-strategies-to-be-broken) | Med | X |
| [M-05] | [AuraSpell#closePositionFarm exits pool with single token and without any slippage protection](#m-05-auraspellclosepositionfarm-exits-pool-with-single-token-and-without-any-slippage-protection) | Med | X |
| [M-06] | [AuraSpell#closePositionFarm will take reward fees on underlying tokens when borrow token is also a reward](#m-06-auraspellclosepositionfarm-will-take-reward-fees-on-underlying-tokens-when-borrow-token-is-also-a-reward) | Med | X |
| [M-07] | [Adversary can abuse hanging approvals left by PSwapLib.swap to bypass reward fees](#m-07-adversary-can-abuse-hanging-approvals-left-by-pswaplibswap-to-bypass-reward-fees) | Med | X |
| [M-08] | [ConvexSpell is completely broken for any curve LP that utilizes native ETH](#m-08-convexspell-is-completely-broken-for-any-curve-lp-that-utilizes-native-eth) | Med | X |
| [M-09] | [Issue #47 from Update #1 is still present in ConvexSpell](#m-09-issue-47-from-update-1-is-still-present-in-convexspell) | Med | X |
| [M-10] | [WAuraPools doesn't correctly account for AuraStash causing all deposits to be permanently lost](#m-10-waurapools-doesnt-correctly-account-for-aurastash-causing-all-deposits-to-be-permanently-lost) | Med | X |
| [M-11] | [WConvexPool.sol will be broken on Arbitrum due to improper integration with Convex Arbitrum contracts](#m-11-wconvexpoolsol-will-be-broken-on-arbitrum-due-to-improper-integration-with-convex-arbitrum-contracts) | Med | X |
| [M-12] | [Invalid - AuraSpell openPositionFarm will revert when the tokens contains lpToken](#m-12-invalid---auraspell-openpositionfarm-will-revert-when-the-tokens-contains-lptoken) | Invalid | NA |
| [M-13] | [approve() call with incorrect function signature will make any SoftVault deployed with USDT as the underlying token unusable](#m-13-approve-call-with-incorrect-function-signature-will-make-any-softvault-deployed-with-usdt-as-the-underlying-token-unusable) | Med | X |

# Summary of Additional Findings

|  Identifier  | Title                        | Severity      | Mitigated |
| ------ | ---------------------------- | ------------- | ----- |
| [A-01] | [Minting the same PID twice in the same block causes funds to be permanently locked](#a-01-high-risk---minting-the-same-pid-twice-in-the-same-block-causes-funds-to-be-permanently-locked) | High | X |
| [A-02] | [Killed pools that utilize ETH would still revert when withdrawing](#a-02-medium-risk---killed-pools-that-utilize-eth-would-still-revert-when-withdrawing) | Med | X |
| [A-03] | [PswapLib is incompatible with USDT](#a-03-medium-risk---pswaplib-is-incompatible-with-usdt)  | Med | X |
| [A-04] | [StableBPTOracle is incompatible with older stable pools](#a-04-low-risk---stablebptoracle-is-incompatible-with-older-stable-pools)  | Low | NA |

## [H-01] Stable BPT valuation is incorrect and can be exploited to cause protocol insolvency

### Details

[Full Report](https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/97)

Root cause was incorrect implementation of BPT valuation that doesn't work for all stable pools.

### Remediation

Fixed in [PR #89](https://github.com/Blueberryfi/blueberry-core/pull/89). Valuation methodology has been updated to utilize built-in rate providers to properly adjusted token valuation.

## [H-02] CurveTricryptoOracle incorrectly assumes that WETH is always the last token in the pool which leads to bad LP pricing

### Details 

[Full Report](https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/98)

Root cause was that valuation assumed WETH was always last token but this is not the case for all pools.

### Remediation

Fixed in [PR #84](https://github.com/Blueberryfi/blueberry-core/pull/84). WETH is no longer assumed to be the last token.

## [H-03] CurveTricryptoOracle#getPrice contains math error that causes LP to be priced completely wrong

### Details 

[Full Report](https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/100)

Root cause was that the valuation methodology returned prices in units of ETH instead of USD.

### Remediation

Fixed in [PR #85](https://github.com/Blueberryfi/blueberry-core/pull/85). Valuation methodology no longer normalizes price to ETH.

## [H-04] CVX/AURA distribution calculation is incorrect and will lead to loss of rewards at the end of each cliff

### Details 

[Full Report](https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/109)

Root cause was that distribution didn't account for users who held balances across CVX/AURA cliffs.

### Remediation

Fixed in [PR #92](https://github.com/Blueberryfi/blueberry-core/pull/92). Creates a hybrid system of allocated and pending tokens. Together both systems ensure the user is paid correctly.

## [H-05] wrong bToken's exchangeRateStored used for calculate ColleteralValue

### Details 

[Full Report](https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/117)

Root cause was that the exchange rate used was for the wrong token. It used the debt token rate instead of the underlying token rate.

### Remediation

Fixed in [PR #81](https://github.com/Blueberryfi/blueberry-core/pull/81). Changes debt token reference to underlying token, resulting in correct valuation.

## [M-01] Users will fail to close their Convex position if the Curve pool is killed

### Details

[Full Report](https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/15)

Root cause was that implementation didn't account for curve pools being killed which would trap user funds.

### Remediation

Fixed in [PR #91](https://github.com/Blueberryfi/blueberry-core/pull/91). When pool is killed, all tokens will be withdrawn instead of only a single token.

## [M-02] getPrice in WeightedBPTOracle.sol uses totalSupply for price calculations which can lead to wrong results

### Details 

[Full Report](https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/18)

Root cause was using total supply instead of real supply for LP.

### Remediation

Fixed in [PR #78](https://github.com/Blueberryfi/blueberry-core/pull/78). Contract will now query pool for real supply and otherwise will fall back to total supply. This allows it to support new pools (which use real supply) as well as old pools (which use total supply).

## [M-03] ConvexSpell/CurveSpell.openPositionFarm will revert in some cases

### Details 

[Full Report](https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/60)

Root cause was that approvals to balancer pool did not account for tokens already present in the contract leading to potential DOS via donation.

### Remediation

Fixed in [PR #98](https://github.com/Blueberryfi/blueberry-core/pull/98). Contract now approves all tokens in the contract preventing this DOS.

## [M-04] Mainnet oracles are incompatible with wstETH causing many popular yields strategies to be broken

### Details 

[Full Report](https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/96)

Root cause was that the oracles were unable to properly value wstETH, making it completely incompatible with use in the contracts.

### Remediation

Fixed in [PR #83](https://github.com/Blueberryfi/blueberry-core/pull/83). Oracle was updated to pull exchange rate of wstETH to properly value it.

## [M-05] AuraSpell#closePositionFarm exits pool with single token and without any slippage protection

### Details 

[Full Report](https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/102)

Root cause was that slippage controls on withdraws didn't exist allowing the user to sandwich attacked for large losses.

### Remediation

Fixed in [PR #80](https://github.com/Blueberryfi/blueberry-core/pull/80). By allowing users to specify the minimum amount out.

## [M-06] AuraSpell#closePositionFarm will take reward fees on underlying tokens when borrow token is also a reward

### Details 

[Full Report](https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/103)

Root cause was that LP was burned before reward cuts were taking leading to the potential situation that fees were applied to collateral tokens, causing very large fees.

### Remediation

Fixed in [PR #95](https://github.com/Blueberryfi/blueberry-core/pull/95). Order of operations was changed to take reward cut before burning LP.

## [M-07] Adversary can abuse hanging approvals left by PSwapLib.swap to bypass reward fees

### Details 

[Full Report](https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/104)

Root cause was that approvals were not removed after swapping which would allows users to bypass reward fees.

### Remediation

Fixed in [PR #87](https://github.com/Blueberryfi/blueberry-core/pull/87). Hanging approvals are now removed after the swap is complete.

## [M-08] ConvexSpell is completely broken for any curve LP that utilizes native ETH

### Details 

[Full Report](https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/105)

Root cause was the Curve utilizes native ETH while Blueberry only supports WETH (wrapped ETH).

### Remediation

Fixed in [PR #86](https://github.com/Blueberryfi/blueberry-core/pull/86). ConvexSpell now automatically handles wrapping and unwrapping WETH so it can utilize native ETH when needed.

## [M-09] Issue #47 from Update #1 is still present in ConvexSpell

### Details 

[Full Report](https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/106)

Root cause was that fixes applied to CurveSpell weren't applied to ConvexSpell.

### Remediation

Fixed in [PR #98](https://github.com/Blueberryfi/blueberry-core/pull/98). Same changes to CurveSpell were applied to ConvexSpell.

## [M-10] WAuraPools doesn't correctly account for AuraStash causing all deposits to be permanently lost

### Details 

[Full Report](https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/108)

Root cause was that WAuraPools would lock depositor funds if AuraStash was a secondary reward.

### Remediation

Fixed in [PR #90](https://github.com/Blueberryfi/blueberry-core/pull/90). StashAura is now properly recognized and handled.

## [M-11] WConvexPool.sol will be broken on Arbitrum due to improper integration with Convex Arbitrum contracts

### Details 

[Full Report](https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/119)

Root cause was that Convex implements different across networks and the Blueberry implementation was not compatible with the Convex structure implemented on Arbitrum.

### Remediation

Contract will only be deployed on mainnet, removing the need for remediation.

## [M-12] Invalid - AuraSpell openPositionFarm will revert when the tokens contains lpToken

### Details 

[Full Report](https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/125)

While initially accepted during the contest, this was later determined to be invalid.

### Remediation

NA

## [M-13] approve() call with incorrect function signature will make any SoftVault deployed with USDT as the underlying token unusable

### Details 

[Full Report](https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/135)

Root cause was that ERC20 interface was not compatible with USDT approve() calls, causing incompatibility.

### Remediation

Fixed in [PR #79](https://github.com/Blueberryfi/blueberry-core/pull/79). Contracts now utilize forceApprove to support these tokens.

## [A-01] HIGH RISK - Minting the same PID twice in the same block causes funds to be permanently locked

### Details 

Introduced in [PR #92](https://github.com/Blueberryfi/blueberry-core/pull/92). auraPerShareDebt[id] incorrectly uses `+=` when setting. This causes debt to be overestimated  if the same PID is minted twice in a single block. When calculating rewards  this causes pendingRewards to underflow and revert. Due to the call to this method during account valuation, the affected account would be permanently locked.

### Recommendation

Use `=` instead of `+=`

### Remediation

Significant rework of the WAuarPool, now renamed WAuraBooster. Reviewed version [here](https://github.com/Blueberryfi/blueberry-core/blob/90c72ed75e88327fc6e3ba2f0568d79bf24e7e89/contracts/wrapper/WAuraBooster.sol)

## [A-02] MEDIUM RISK - Killed pools that utilize ETH would still revert when withdrawing

### Details 

Introduced in [PR #91](https://github.com/Blueberryfi/blueberry-core/pull/91). When handling withdrawals from killed pools that utilize ETH, ConvexSpell fails to be deposit and swap it causing all withdrawal transactions to revert.

### Recommendation

Wrap and swap ETH returned under these circumstances

### Remediation

Fixed in [PR #101](https://github.com/Blueberryfi/blueberry-core/pull/101). ETH is now wrapped and swapped for killed pools that contain ETH.

## [A-03] MEDIUM RISK - PswapLib is incompatible with USDT

### Details 

Introduced in [PR #87](https://github.com/Blueberryfi/blueberry-core/pull/87). PswapLib has not been updated with UniversalERC20 as expected.

### Recommendation

Update PswapLib to use UniversalERC20

### Remediation

Fixed in [PR #102](https://github.com/Blueberryfi/blueberry-core/pull/102). Issue was fixed as recommended.

## [A-04] LOW RISK - StableBPTOracle is incompatible with older stable pools

### Details 

Introduced in [PR #89](https://github.com/Blueberryfi/blueberry-core/pull/89). Older pools don't contain built in rate providers like newer pools. This makes so that older pools are not compatible.

### Recommendation

Use try-catch block to make contracts compatible with both sets of pools.

### Remediation

Balancer developers have stated that older pools will soon migrate to the new pool implementations, removing the need to make changes.