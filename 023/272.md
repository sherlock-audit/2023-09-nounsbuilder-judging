Bright Fossilized Seagull

high

# Founder's ability to mint after transferring ownership to DAO

## Summary

Founders can potentially bypass restrictions to mint tokens even after ownership transfer to the DAO.

## Vulnerability Detail

Initially, founders are trusted entities until contract ownership shifts to the DAO. After that, any potential for malicious activity by founders represents a significant vulnerability, potentially compromising other users. From sponsor:

        yes they are trusted until DAO auctions are started then admin controls go to the DAO

In the early stages, the Token contract’s initial owner is the founder.

        // Initialize each instance with the provided settings
        IToken(token).initialize({
            founders: _founderParams,
            initStrings: _tokenParams.initStrings,
            reservedUntilTokenId: _tokenParams.reservedUntilTokenId,
            metadataRenderer: metadata,
            auction: auction,
            initialOwner: founder
        });

https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/manager/Manager.sol#L132-L140

Ownership transfers to the DAO upon the launch of the first NFT auction. Post-transfer, the founder should lose the ability to mint new NFTs, preventing any unfair influence or inflation of voting power.
        
        // Transfer ownership of the token contract to the DAO
        token.onFirstAuctionStarted();

https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/auction/Auction.sol#L347-L348


However, a vulnerability exists in the Token contract. The `updateMinters` function allows the owner to grant unrestricted minting privileges. This loophole enables the founder, prior to transferring ownership, to assign the minter role to a secondary account. This backdoor could be exploited later, even under DAO control, to mint NFTs for personal gain, such as manipulating votes or selling NFTs in the market.

        function updateMinters(MinterParams[] calldata _minters) external onlyOwner {
            // Update each minter
            for (uint256 i; i < _minters.length; ++i) {
                // Skip if the minter is already set to the correct value
                if (minter[_minters[i].minter] == _minters[i].allowed) continue;

                emit MinterUpdated(_minters[i].minter, _minters[i].allowed);

                // Update the minter
                minter[_minters[i].minter] = _minters[i].allowed;
            }
        }

https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L465-L476

## Impact

This backdoor undermines the DAO's integrity, potentially diminishing member voting rights in favor of a malicious founder.

## Code Snippet

https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L465-L476

## Tool used

Manual Review

## Recommendation

Do not allow founder as the owner of Token contract. The owner of the Token contract will be granted to the DAO after the DAO is launched. 