Ancient Inky Kitten

medium

# Unsafe cast can result in Auction DoS

## Summary
Unsafe cast can result in `Auction` DoS

## Vulnerability Detail
In the `_createAuction()` function [Auction.sol#L305-L312](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/auction/Auction.sol#L305-L312), there's a problem with an "Unsafe Cast" to `uint40` for the `endTime` variable. While `Settings.Duration` is safely cast to `uint40` in the `Initialize` function, the subsequent calculation of `endTime` by adding `block.timestamp` and `settings.Duration` may lead to an overflow which may result in `endTime < startTime`.

## Impact
This will result in a broken Auction contract, while the other contracts of the DAO may work properly. Mainly the createBid()
function will result in DoS, because it will revert with Auction Over and no one will be able to create bids for this particular Auction. Note that the _duration value is set arbitrarily by the DAO deployer and therefore this bug is likely to happen.

## Code Snippet
We have coded a PoC:
```Solidity
function test_CreateAuctionWhereNoBidsCanBeCreated() public {
        deployMock();

        //Assume the duration has been set to uint40.max,
        //so it won't revert on Initialization because it uses SafeCast.toUint40()
        uint256 dur = uint256(type(uint40).max);
        auction.changeDuration(dur);
        vm.prank(founder);
        
        auction.unpause();
        vm.prank(bidder1);

        //No one will be able to create bid because endTime will always be lower than block.timestamp 
        //due to overflow in the unsafe cast of endTime in _createAuction()
        vm.expectRevert(abi.encodeWithSignature("AUCTION_OVER()"));
        auction.createBid{ value: 0.420 ether }(2);

        vm.prank(bidder2);

        vm.expectRevert(abi.encodeWithSignature("AUCTION_OVER()"));
        auction.createBid{ value: 1 ether }(2);

        //Auction can always be settled
        auction.settleCurrentAndCreateNewAuction();
    }
```
## Tool used

Manual Review

## Recommendation
Since safe cast won't work to prevent this issue as it will result in overflow and revert, we suggest you add a max duration check on the Initialize function, preventing from defective Auction contracts being deployed.
We suggest you set e MAX_DURATION variable which is not far in the future and add this check in the `Initialize` function: 
```Solidity
if(_duration > MAX_DURATION) revert MAX_DURATION_EXCEEDED;
```