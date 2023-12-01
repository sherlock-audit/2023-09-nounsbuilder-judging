Skinny Frost Puppy

medium

# Token contract doesn't use safeMint, tokens may be lost

## Summary
Token contract is an NFT token that supports safe transfer that calls the hook function of the receiver if it's a contract. but the mint functions are not implemented in a safe way, code may mint tokens for other contracts addresses while they can't handle the NFT token.

## Vulnerability Detail
This is mint codes in Token contract:
```javascript
    /// @notice Mints tokens to the recipient and handles founder vesting
    function mintTo(address recipient) external nonReentrant onlyAuctionOrMinter returns (uint256 tokenId) {
        tokenId = _mintWithVesting(recipient);
    }

    /// @notice Mints tokens from the reserve to the recipient
    function mintFromReserveTo(address recipient, uint256 tokenId) external nonReentrant onlyMinter {
        // Token must be reserved
        if (tokenId >= reservedUntilTokenId) revert TOKEN_NOT_RESERVED();

        // Mint the token without vesting (reserved tokens do not count towards founders vesting)
        _mint(recipient, tokenId);
    }

    /// @notice Mints the specified amount of tokens to the recipient and handles founder vesting
    function mintBatchTo(uint256 amount, address recipient) external nonReentrant onlyAuctionOrMinter returns (uint256[] memory tokenIds) {
        tokenIds = new uint256[](amount);
        for (uint256 i = 0; i < amount; ) {
            tokenIds[i] = _mintWithVesting(recipient);
            unchecked {
                ++i;
            }
        }
    }

    function _mintWithVesting(address recipient) internal returns (uint256 tokenId) {
        // Cannot realistically overflow
        unchecked {
            do {
                // Get the next token to mint
                tokenId = reservedUntilTokenId + settings.mintCount++;

                // Lookup whether the token is for a founder, and mint accordingly if so
            } while (_isForFounder(tokenId));
        }

        // Mint the next available token to the recipient for bidding
        _mint(recipient, tokenId);
    }

    /// @dev Overrides _mint to include attribute generation
    /// @param _to The token recipient
    /// @param _tokenId The ERC-721 token id
    function _mint(address _to, uint256 _tokenId) internal override {
        // Mint the token
        super._mint(_to, _tokenId);

        // Increment the total supply
        unchecked {
            ++settings.totalSupply;
        }

        // Generate the token attributes
        if (!settings.metadataRenderer.onMinted(_tokenId)) revert NO_METADATA_GENERATED();
    }
```
as you can see their is no safeMint and code doesn't call receiver's hook function when the receiver is a contract.

## Impact
some token may be minted for addresses that can't handle NFT and those NFTs will be locked forever.

## Code Snippet
https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/db232c649b425c36f5a93607c95cfdf0e5962b2f/nouns-protocol/src/token/Token.sol#L205-L259

## Tool used
Manual Review

## Recommendation
implement safeMint mechanism for minting reserve and auction tokens.