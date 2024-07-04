# Blueberry Pendle Audit Report

### Reviewed by: 0x52 ([@IAm0x52](https://twitter.com/IAm0x52))

### Review Date(s): 6/29/24 - 6/30/24

### Fix Review Date(s): 6/31/24

### Fix Review Hash: [3b41d12](https://github.com/Blueberryfi/blueberry-core/blob/3b41d121b779e852209b5fa115410e1a9eef2e3d/contracts/spell/PendleSpell.sol)

# 0x52 Background

As an independent smart contract auditor I have completed over 100 separate reviews. I primarily compete in public contests as well as conducting private reviews (like this one here). I have more than 30 1st place finishes (and counting) in public contests on [Code4rena](https://code4rena.com/@0x52) and [Sherlock](https://audits.sherlock.xyz/watson/0x52). I have also partnered with [SpearbitDAO](https://cantina.xyz/u/iam0x52) as a Lead Security researcher. My work has helped to secure over $1 billion in TVL across 100+ protocols.

# Scope

The [blueberry-core](https://github.com/Blueberryfi/blueberry-core) repo was reviewed at commit hash [d0ed247](https://github.com/Blueberryfi/blueberry-core/blob/d0ed24769704cf5d9a8b0616cf534f29db32f6ca/)

In-Scope Contracts
- contracts/spell/PendleSpell.sol
- contracts/oracle/PendleBaseOracle.sol
- contracts/oracle/PendleLPOracle.sol
- contracts/oracle/PendlePTOracle.sol

Deployment Chain(s)
- example mainnet

# Summary of Findings

|  Identifier  | Title                        | Severity      | Mitigated |
| ------ | ---------------------------- | ------------- | ----- |
| [M-01] | [PT donation attack will DOS spell deposit permanently](#m-01-pt-donation-attack-will-dos-spell-deposit-permanently) | Med | ✔️ |

# Detailed Findings

## [M-01] PT donation attack will DOS spell deposit permanently

### Details 

[PendleSpell.sol#L128-L137](https://github.com/Blueberryfi/blueberry-core/blob/d0ed24769704cf5d9a8b0616cf534f29db32f6ca/contracts/spell/PendleSpell.sol#L128-L137)

    (uint256 ptAmount, , ) = IPendleRouter(_pendleRouter).swapExactTokenForPt(
        address(this),
        market,
        minPtOut,
        params,
        input,
        limitOrder
    );

    if (ptAmount != IERC20Upgradeable(pt).balanceOf(address(this))) revert Errors.SWAP_FAILED(pt);

After swapping from debt token to PT, the contract makes an exact check against the balance of the contract to ensure the swap executed as expected. The issue with this is that even if it is a single wei over, the contract will revert. This makes it trivial to permanently DOS opening positions via the contract by donating a small amount of PT to the contract.

### Lines of Code

[PendleSpell.sol#L101-L147](https://github.com/Blueberryfi/blueberry-core/blob/d0ed24769704cf5d9a8b0616cf534f29db32f6ca/contracts/spell/PendleSpell.sol#L101-L147)

### Recommendation

Check should be `>` rather than `!=`

### Remediation

Fixed as suggested.