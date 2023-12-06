Proud Orange Blackbird

medium

# Founders may receive a higher number of tokens than initially anticipated.

## Summary
There are predefined percentage values that signify the proportion of tokens allocated to `founders`. 
For instance, if the cumulative percentage value for all `founders` is `50`, it implies that approximately half of the tokens should be allocated to the `founders`. 
This mechanism is designed to maintain a balance in the distribution of tokens between `founders` and regular `users` within the protocol.
However, this equilibrium can be disrupted when there are no bids for certain tokens.
## Vulnerability Detail
When creating an `auction`, a new token was minted and subsequently assigned to the auction contract.
```solidity
function _createAuction() private returns (bool) {
    try token.mint() returns (uint256 tokenId) {}
}
```
```solidity
function mint() external nonReentrant onlyAuctionOrMinter returns (uint256 tokenId) {
    tokenId = _mintWithVesting(msg.sender);
}
```
To generate this new token, a certain number of tokens were minted and allocated to the `founders` based on the specified `proportion`.
```solidity
function _mintWithVesting(address recipient) internal returns (uint256 tokenId) {
    unchecked {
        do {
            tokenId = reservedUntilTokenId + settings.mintCount++;
        } while (_isForFounder(tokenId));
    }
    _mint(recipient, tokenId);
}
```
If there is no bid for this token, it is necessary to burn the token.
```solidity
function _settleAuction() private {
    if (_auction.highestBidder != address(0)) {
    } else {
         token.burn(_auction.tokenId);
    }
}
```
However, in the `burn` function, we do not decrease `settings.mintCount`. 
Therefore, irrespective of whether this token is assigned to a `user` or burned, the same `process` will continue for subsequent tokens.
```solidity
function _burn(uint256 _tokenId) internal override {
    super._burn(_tokenId);
    unchecked {
        --settings.totalSupply;
    }
}
```

The POC test is as below:
```solidity
function test_NoBid() public {
      address f1Wallet = founder;
      uint256 f1Percentage = 50;
      address[] memory founders = new address[](1);
      uint256[] memory percents = new uint256[](1);
      uint256[] memory vestingEnds = new uint256[](1);
      founders[0] = f1Wallet;
      percents[0] = f1Percentage;
      vestingEnds[0] = 4 weeks;

      deployWithCustomFounders(founders, percents, vestingEnds);
      Founder memory f1 = token.getFounder(0);
      assertEq(f1.ownershipPct, f1Percentage);

      vm.prank(founder);
      auction.unpause();

      uint256 skipTime = 20 minutes;
      for (uint256 i; i < 99; i++) {
          vm.warp(skipTime * (i + 1));
          auction.settleCurrentAndCreateNewAuction();
      }

      assertEq(token.balanceOf(f1Wallet), 100);
      // The final auction has not been settled yet.
      assertEq(token.totalSupply(), 101);
  }
```
As evident, the protocol balance is `50`, but all tokens were allocated to the `founders`.
## Impact
After deploying the `DAO` and initiating its auction, there is no guarantee that the `auction` process will unfold as expected. 
If there are no `bids` for a certain period due to various reasons, all created tokens will be assigned to the `founders`, disrupting the intended balance. 
To manage the balance, additional measures such as updating the `founder`'s percentage or implementing other adjustments may be necessary.
## Code Snippet
https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/auction/Auction.sol#L294
https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L201-L203
https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L230-L243
https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/auction/Auction.sol#L285
https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L302-L310
## Tool used

Manual Review

## Recommendation
Modify `_burn` function as below:
```solidity
function _burn(uint256 _tokenId) internal override {
    // Call the parent burn function
    super._burn(_tokenId);

    // Reduce the total supply
    unchecked {
        --settings.totalSupply;
        
        + if (settings.mintCount > 0 && reservedUntilTokenId + settings.mintCount - 1 == _tokenId) settings.mintCount--;
    }
}
```
This ensures that no new tokens are assigned to the `founders` until a successful `auction` takes place.
As a result, the overall balance of the protocol will be maintained.