Puny Coffee Crocodile

medium

# Subsequent bid can be created with same price, if `reservePrice` is 0

## Summary
Subsequent bid can be created with same price, if `reservePrice` is 0

## Vulnerability Detail
Let's see how this will work
reservePrice = 0 & minBidIncrement = 10
- Bidder 1 will place the bid at 0 ETH
- Bidder 2 will place the bid at 1 wei
```solidity
   unchecked {
                // Compute the minimum bid
        @>        minBid = lastHighestBid + ((lastHighestBid * settings.minBidIncrement) / 100);
            }
```
- Now, midBid =  1 + (1 * 10)/100 which is 1 + 0(rounding) = 1 wei
- Bidder 3 will bid at 1 wei & bidder 3 will be highest bidder

// Here is the POC
Before running the test make `reservePrice = 0` & `minBidIncrement = 10` also a bidder3

```solidity
function test_CreateSubsequentBidWithSameAmount() public {
        deployMock();

        vm.prank(founder);
        auction.unpause();

        // First bidder is placing bid at 0 Eth ie reservePrice
        uint256 reservePrice = 0 ether;
        vm.prank(bidder1);
        auction.createBid{ value: reservePrice }(2);

        // Second bidder is placing bid at 1 wei
        vm.prank(bidder2);
        auction.createBid{ value: reservePrice + 1 }(2);

        // Third bidder is also placing bid at 1 wei
        vm.prank(bidder3);
        auction.createBid{ value: reservePrice + 1 }(2);

        (, uint256 highestBid, address highestBidder,,,) = auction.auction();

        assertEq(highestBid, 1 wei);
        assertEq(highestBidder, bidder3);
    }
```
To run the test
```solidity
forge test --mt test_CreateSubsequentBidWithSameAmount  -vvv
```

Note: This will work until `minBidIncrement = 99` because of rounding

## Impact
- DAO or Treasury will lose funds

## Code Snippet
https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/auction/Auction.sol#L214

## Tool used
- Manual Review

## Recommendation
- Ensure a min of 10 wei in `reservePrice`
