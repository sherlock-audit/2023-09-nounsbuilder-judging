Bright Fossilized Seagull

high

# Incorrect token checking leading to unintended free NFTs for founders

## Summary

A critical flaw in the token validation process of the `_mintWithVesting` function allows founders to unintentionally receive NFTs without cost, potentially skewing their voting influence. This issue stems from an incorrect method of checking if a given token is for a founder or not.

## Vulnerability Detail

The `_mintWithVesting` mints a new token for auction and mints any tokens that are marked for founders. However, the function erroneously identifies which tokens are intended for founders, leading to the unintended minting of additional NFTs for them.

The flawed process involves converting a given `tokenId` into a `baseTokenId` and then verifying if this `baseTokenId` corresponds to a founder. If the baseTokenId is found to be assigned to a founder, the `_tokenId` is mistakenly minted for the founder.

        /// @dev Checks if a given token is for a founder and mints accordingly
        /// @param _tokenId The ERC-721 token id
        function _isForFounder(uint256 _tokenId) private returns (bool) {
            // Get the base token id
            uint256 baseTokenId = _tokenId % 100;

            // If there is no scheduled recipient:
            if (tokenRecipient[baseTokenId].wallet == address(0)) {
                return false;

                // Else if the founder is still vesting:
            } else if (block.timestamp < tokenRecipient[baseTokenId].vestExpiry) {
                // Mint the token to the founder
                _mint(tokenRecipient[baseTokenId].wallet, _tokenId);

                return true;

                // Else the founder has finished vesting:
            } else {
                // Remove them from future lookups
                delete tokenRecipient[baseTokenId];

                return false;
            }
        }

https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L261-L285

For instance, a founder is eligible for tokenId 20. Subsequently, when tokenId 120 undergoes distribution checks, the `_isForFounder` function incorrectly identifies it as 20 after conversion, leading to an unwarranted minting for the founder if the vesting period is still active.

## Impact

This flaw enables founders to acquire additional NFTs at no cost, which could unfairly amplify their voting power in the ecosystem.

## Code Snippet

https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L261-L285

## Tool used

Manual Review

## Recommendation

To rectify this issue, the function should validate the original `_tokenId` instead of the derived `baseTokenId`. 

```diff
        function _isForFounder(uint256 _tokenId) private returns (bool) {
            // Get the base token id
-           uint256 baseTokenId = _tokenId % 100;

            // If there is no scheduled recipient:
-           if (tokenRecipient[baseTokenId].wallet == address(0)) {
+           if (tokenRecipient[_tokenId].wallet == address(0)) {
                return false;

                // Else if the founder is still vesting:
-           } else if (block.timestamp < tokenRecipient[baseTokenId].vestExpiry) {
+           } else if (block.timestamp < tokenRecipient[_tokenId].vestExpiry) {
                // Mint the token to the founder
-               _mint(tokenRecipient[baseTokenId].wallet, _tokenId);
+               _mint(tokenRecipient[_tokenId].wallet, _tokenId);

                return true;

                // Else the founder has finished vesting:
            } else {
                // Remove them from future lookups
-               delete tokenRecipient[baseTokenId];
+               delete tokenRecipient[_tokenId];

                return false;
            }
        }
```