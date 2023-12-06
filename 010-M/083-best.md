Ancient Inky Kitten

medium

# Last bidder funds can remain locked

## Summary
Auction which can not be settled will result in locked funds.

## Vulnerability Detail
In [Auction.sol#L444-L455](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/auction/Auction.sol#L444-L455)
If `referralRewardsBPS + builderRewardsBPS` >= 70% the founder could set a `_founderRewardBps` that breaks the contract therefore the auctions can't be settled and the last bidder funds would be locked permanently unless the founder pauses the contract and changes the `_founderRewardBps`. 
In the `setFounderRewards` function there is this check that ensures the founderRewards are not greater that 30%
```solidity
if (reward.percentBps > MAX_FOUNDER_REWARD_BPS) revert INVALID_REWARDS_BPS();
```
but it does not check if the total rewards are greater than 100%.

Additionally there is a condition in `_computeTotalRewards()` function which says:
 ```solidity
// Verify percentage is not more than 100
 if (totalBPS >= BPS_PER_100_PERCENT) {
            revert INVALID_REWARD_TOTAL();
    }
```
and this is an off-by-one issue, since the comment above the if statement states that this should only happen if the `totalBPS` exceeds the `BPS_PER_100_PERCENT`.

[Auction.sol#L465-L508](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/auction/Auction.sol#L465-L508)

## Impact
This could lead to a state where the Auction can't be settled and the rewards never distributed and the last bidder could never withdraw his funds unless the founder is willing to reduce his own reward.

## Code Snippet
We have coded a PoC:
```solidity
function test_OwnerLocksBidderFunds() public {
        deployMock();
        // assume the builder bps was set to 90%
        auction.setBuilderBPS(9000);

        vm.startPrank(founder);
        // Set minimum possible bid increment
        auction.setMinimumBidIncrement(1);
        auction.setReservePrice(0);

        auction.unpause();
        vm.stopPrank();
        // bidder 1 bids
        vm.prank(bidder1);
        auction.createBid{ value: 0 }(2);

        // bidder 2 bids
        vm.prank(bidder2);
        auction.createBid{ value: 20 ether }(2);

        vm.startPrank(auction.owner());
        auction.pause();
        // change founder reward so totalFees is >= 100 %
        auction.setFounderReward(AuctionTypesV2.FounderReward(founder, 2000));
        auction.unpause();
        vm.stopPrank();

        (, uint256 highestBid, address highestBidder, , , ) = auction.auction();
        assertEq(highestBidder, bidder2);
        assertEq(highestBid, 20 ether);
        assertEq(address(auction).balance ,20 ether);

        // the auction expires
        vm.warp(block.timestamp + 999999999999999999999);
        // it cant be settled, and therefore bidder funds are stuck
        vm.expectRevert(abi.encodeWithSignature("INVALID_REWARD_TOTAL()"));
        // it cant be settled nor withdrawn, funds are locked
        auction.settleCurrentAndCreateNewAuction();
        
    }
```
## Tool used

Manual Review

## Recommendation
Set a check in the `setFounderRewards` function that ensures the `totalRewards` do not exceed 100%.
```solidity
function setFounderReward(FounderReward calldata reward) external onlyOwner whenPaused {
        // Ensure the founder reward is not more than max
        if (reward.percentBps > MAX_FOUNDER_REWARD_BPS) revert INVALID_REWARDS_BPS();
        uint256 totalBPS = _founderRewardBps + referralRewardsBPS + builderRewardsBPS;
        if (totalBPS > BPS_PER_100_PERCENT) {
            revert INVALID_REWARD_TOTAL();
           }
```        
Change the if statement as follows:
 ```solidity
 if (totalBPS > BPS_PER_100_PERCENT) {
            revert INVALID_REWARD_TOTAL();
    }
```