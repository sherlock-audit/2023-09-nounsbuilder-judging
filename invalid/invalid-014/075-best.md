Mean Bamboo Whale

high

# totalBPS is not validated in the right function

## Summary

Validation of Total Basic Points is not done while values are assigned but when auction is settled, which can result in locked funds and a DoS

## Vulnerability Detail

In Auction.sol, `totalBPS` can't be equal or more than `BPS_PER_100_PERCENT = 10_000`. However, the validation is not done within the `constructor()` but in `_computeTotalRewards`, which can be called by `settleCurrentAndCreateNewAuction` or `settleAuction` through `_settleAuction`.

`totalBPS` is calculated by `founderReward.percentBps + referralRewardsBPS + builderRewardsBPS`: 
- `founderReward.percentBps (uint16)` is validated at the time of assignment in `initialize()` to revert if more than `MAX_FOUNDER_REWARD_BPS = 3_000`
-  `referralRewardsBPS (uint16)` is not validated at the time of assignment
- `builderRewardsBPS (uint16)` is not validated at the time of assignment

## Impact

If `builderRewardsBPS` and/or `referralRewardsBPS` are assigned a value which results in `totalBPS >= 10_000`, `_computeTotalRewards` will always revert with `INVALID_REWARD_TOTAL()`. Thus, it will not be possible to compute the total rewards for a bid and funds will permanently be locked in the contract.

## Code Snippet

https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/auction/Auction.sol#L32-L33

https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/auction/Auction.sol#L70-L82

https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/auction/Auction.sol#L465-L479

## Tool used

Manual Review

## Recommendation

Update the code to do the validation in the `constructor()` 

```diff
    constructor(
        address _manager,
        address _rewardsManager,
        address _weth,
        uint16 _builderRewardsBPS,
        uint16 _referralRewardsBPS
    ) payable initializer {
+       uint256 totalBPS = MAX_FOUNDER_REWARD_BPS + _builderRewardsBPS + _referralRewardsBPS;
+       if (totalBPS >= BPS_PER_100_PERCENT) {
+            revert INVALID_REWARD_TOTAL();
+       }
        manager = Manager(_manager);
        rewardsManager = IProtocolRewards(_rewardsManager);
        WETH = _weth;
        builderRewardsBPS = _builderRewardsBPS;
        referralRewardsBPS = _referralRewardsBPS;
    }
```
and not in `_computeTotalRewards`

```diff
    function _computeTotalRewards(
        address _currentBidRefferal,
        uint256 _finalBidAmount,
        uint256 _founderRewardBps
    ) internal view returns (RewardSplits memory split) {
        // Get global builder recipient from manager
        address builderRecipient = manager.builderRewardsRecipient();

-        // Calculate the total rewards percentage
-       uint256 totalBPS = _founderRewardBps + referralRewardsBPS + builderRewardsBPS;

-        // Verify percentage is not more than 100
-        if (totalBPS >= BPS_PER_100_PERCENT) {
-            revert INVALID_REWARD_TOTAL();
-        }
```
