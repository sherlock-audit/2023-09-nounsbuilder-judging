Tame Flint Okapi

medium

# Using transferFrom for ERC721 tokens without checking the results can result in tokens being lost or trapped if the transaction fails or the recipient cannot handle

## Summary

Using `transferFrom` to transfer `ERC721` tokens and not checking the results can cause the tokens to be lost or trapped if the transfer fails or the recipient cannot handle the `ERC721` tokens

## Vulnerability Detail

In the internal function `_settleAuction` which aims to settle current auction, there is a transaction to transfer tokens to `highestBidder` using `transferFrom` where this function is not safe. The code is below :

```solidity
File : nouns-protocol/src/auction/Auction.sol

280 : token.transferFrom(address(this), _auction.highestBidder, _auction.tokenId);
```

If the recipient is not a EOA, `safeTransferFrom` ensures that the contract is able to safely receive the token. In the worst-case scenario, it may result in tokens lost or trapped, as the following code uses `transferFrom`, which [[doesn't check](https://github.com/ethereum/EIPs/blob/78e2c297611f5e92b6a5112819ab71f74041ff25/EIPS/eip-721.md?plain=1#L103-L113)](https://github.com/ethereum/EIPs/blob/78e2c297611f5e92b6a5112819ab71f74041ff25/EIPS/eip-721.md?plain=1#L103-L113) if the recipient can handle the NFT. And also in this codebase there is no checking whether the transfer was successful or not

## Impact

Tokens being lost or trapped

## Code Snippet

https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/auction/Auction.sol#L280

## Tool used

Manual review

## Recommendation

Consider using `safeTransferFrom` library for handling the transfer execution
