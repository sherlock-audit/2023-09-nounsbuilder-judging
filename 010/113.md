Fresh Obsidian Spider

medium

# Allowing Bids During Pause State Due to Missing `whenNotPaused`Modifier

## Summary
Certain key functions lack the `whenNotPaused` modifier, allowing bids to be placed even when the auction is paused. This can lead to confusion and potentially unfair bidding situations.

## Vulnerability Detail
The contract includes various administrative functions with a whenPaused modifier, ensuring they can only be executed when the auction is paused.
```solidity
    function setDuration(uint256 _duration) external onlyOwner whenPaused {
    function setReservePrice(uint256 _reservePrice) external onlyOwner whenPaused {
    function setTimeBuffer(uint256 _timeBuffer) external onlyOwner whenPaused {
    function setMinimumBidIncrement(uint256 _percentage) external onlyOwner whenPaused {
    function setFounderReward(FounderReward calldata reward) external onlyOwner whenPaused {
```
However, functions like `createBidWithReferral` and `createBid` are missing the `whenNotPaused` modifier. This oversight allows users to place bids even when the auction is paused, potentially leading to unexpected changes in auction parameters after a bid is made.

## Impact
Users might place bids without realizing the auction is paused and settings may change after their bids, leading to confusion and potential loss of funds or trust.

## Code Snippet
https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/auction/Auction.sol#L144
https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/auction/Auction.sol#L151
PoC
```solidity
    function test_CreateBidWhenPaused() public {
        deployMock();
        // Unpoause to create Auction and transfer Ownership to treasury
        vm.prank(founder);
        auction.unpause();
        // Pause to make critical changes
        vm.prank(address(treasury));
        auction.pause();

        vm.prank(bidder1);
        auction.createBid{ value: 0.5 ether }(2);

        (, uint256 highestBid, address highestBidder, , , ) = auction.auction();

        assertEq(highestBidder, bidder1);
        assertEq(highestBid, 0.5 ether);
    }
```
```bash
forge test --match-test test_CreateBidWhenPaused
Running 1 test for test/AuctionAudit.t.sol:AuctionTest
[PASS] test_CreateBidWhenPaused() (gas: 3655064)
Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 3.97ms
```

## Tool used
vscode

Manual Review

## Recommendation
Add the whenNotPaused modifier to the createBidWithReferral and createBid functions:
```solidity
function createBidWithReferral(uint256 _tokenId, address _referral) external payable nonReentrant whenNotPaused {
function createBid(uint256 _tokenId) external payable nonReentrant whenNotPaused {
```
```bash
forge test --match-test test_CreateBidWhenPaused
Running 1 test for test/AuctionAudit.t.sol:AuctionTest
[FAIL. Reason: PAUSED()] test_CreateBidWhenPaused() (gas: 3649220)
Test result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 4.52ms
```
