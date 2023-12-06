Plain Chambray Wasp

medium

# Blocking of owner to change `reservedUntilTokenId` if non-reserved tokens have not been minted yet

## Summary
Not possible to update `reservedUntilTokenId` because of the wrong calculation of `settings.mintCount`.

## Vulnerability Detail
The owner has the right to change the `reservedUntilTokenId` variable only if non-reserved tokens have not been minted yet.

```solidity
function setReservedUntilTokenId(uint256 newReservedUntilTokenId) external onlyOwner {
        // Cannot change the reserve after any non reserved tokens have been minted
        // Added to prevent making any tokens inaccessible
        if (settings.mintCount > 0) {
            revert CANNOT_CHANGE_RESERVE();
        }
```

When the `mintFromReserveTo` function is called, the `settings.mintCount` is not incremented because only reserved tokens are minted at this point. Another way to mint reserved tokens is through an auction if the next `tokenId` belongs to the founders. It will loop until the `tokenId` does not belong to the founders. In each cycle, when the `tokenId` belongs to some of the founders, `settings.mintCount` will be incremented by 1. This is incorrect because the storage variable `settings.mintCount` holds the number of non-reserved minted tokens.

This will lead to incorrect incrementing of `settings.mintCount` and make it impossible for the owner to change `reservedUntilTokenId` if non-reserved tokens are not yet minted. Additionally, this variable will not be directly equal to all non-reserved tokens because if a token is burned because is not sold, `mintCount` is not decremented by 1.

For example:
1. The founder reserves 10 tokens.
2. The `Auctions.sol` contract is unpaused, and the first 10 tokens are minted to the founder, with the 11th set for the first auction, ready for bidding.
3. The 11th token is not sold and is burned.
4. The owner pauses the `Auctions.sol` contract.
5. The owner decides to increase the reserved tokens to 20, but this will not be possible because `settings.mintCount` is equal to 11.

## Impact
Blocking of owner to change `reservedUntilTokenId` if non-reserved tokens have not been minted yet.

## Code Snippet
https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L235

https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L486-L491
## Tool used

Manual Review

## Recommendation
Only increment `settings.mintCount` by 1 if next `tokenId` does not belong to the founder. Also, decrement `settings.mintCount` by 1 if the token is burned because is not sold.

