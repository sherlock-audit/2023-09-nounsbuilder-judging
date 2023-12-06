Alert Lime Alpaca

high

# Minted tokens' attributes will be lost and can never be regenerated if new default metadata renderer is set

## Summary
Minted tokens' attributes will be lost and can never be regenerated if new default metadata renderer is set.

## Vulnerability Detail
Protocol provides [setMetadataRenderer(...)](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/manager/Manager.sol#L169-L191) function for token owner to set new metadata renderer:
```solidity
metadata = address(new ERC1967Proxy(_newRendererImpl, ""));
daoAddressesByToken[_token].metadata = metadata;

if (_setupRenderer.length > 0) {
    IBaseMetadata(metadata).initialize(_setupRenderer, _token);
}

IToken(_token).setMetadataRenderer(IBaseMetadata(metadata));
```
When a new token is [minted](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L248-L259), `IBaseMetadata`'s [onMinted(...)](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/metadata/interfaces/IBaseMetadata.sol#L32) function will be called to generate attributes for the token.
BuidlerDAO providers a default implementation of `IBaseMetadata` and its corresponding [onMinted(...)](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/metadata/MetadataRenderer.sol#L242-L277) function, which can only be called by `Token` contract:
```solidity
function onMinted(uint256 _tokenId) external override returns (bool) {
    // Ensure the caller is the token contract
    if (msg.sender != settings.token) revert ONLY_TOKEN();

    // Compute some randomness for the token id
    uint256 seed = _generateSeed(_tokenId);

    // Get the pointer to store generated attributes
    uint16[16] storage tokenAttributes = attributes[_tokenId];

    // Cache the total number of properties available
    uint256 numProperties = properties.length;

    if (numProperties == 0) {
        return false;
    }

    // Store the total as reference in the first slot of the token's array of attributes
    tokenAttributes[0] = uint16(numProperties);

    unchecked {
        // For each property:
        for (uint256 i = 0; i < numProperties; ++i) {
            // Get the number of items to choose from
            uint256 numItems = properties[i].items.length;

           // Use the token's seed to select an item
            tokenAttributes[i + 1] = uint16(seed % numItems);

            // Adjust the randomness
            seed >>= 16;
        }
    }

    return true;
}
```
The problem is that if this default implementation is used to replace the current metadata render after some tokens have been minted, then the minted tokens will lose attributes, even worse, because the `onMinted(...)` function is called only when a new token is minted by `Token` contract, so there is no way to regenerated attributes for those minted tokens.

Please note, setting new metadata render when there are some tokens have been mintet is a __valid input__ for the trusted token owner, and even if there is no token minted, an eligible user may still mint tokens from `MerkleReserveMinter ` before new metadata render is set, due to the random ordering of transactions.

Please see blew test codes and run in `Token.t.sol` to verify:
```solidity
   function test_Audit() public {
        deployMock();

        vm.prank(token.auction());
        uint256 tokenId = token.mint();

        // tokenURI can be retrieved before setting new metadata render
        token.tokenURI(tokenId);

        vm.startPrank(token.owner());

        manager.setMetadataRenderer(address(token), manager.metadataImpl(), tokenParams.initStrings);

        string[] memory names = new string[](1);
        names[0] = "testing";
        MetadataRendererTypesV1.ItemParam[] memory items = new MetadataRendererTypesV1.ItemParam[](2);
        items[0] = MetadataRendererTypesV1.ItemParam({ propertyId: 0, name: "failure1", isNewProperty: true });
        items[1] = MetadataRendererTypesV1.ItemParam({ propertyId: 0, name: "failure2", isNewProperty: true });
        MetadataRendererTypesV1.IPFSGroup memory ipfsGroup = MetadataRendererTypesV1.IPFSGroup({ baseUri: "BASE_URI", extension: "EXTENSION" });
        MetadataRenderer(token.metadataRenderer()).addProperties(names, items, ipfsGroup);

        vm.stopPrank();

        // tokenURI can not be retrieved after setting new metadata render
        bytes4 selector = bytes4(keccak256("TOKEN_NOT_MINTED(uint256)"));
        vm.expectRevert(abi.encodeWithSelector(selector, tokenId));
        token.tokenURI(tokenId);
    }
```

## Impact
Token can not be properly rendered, because of a lack of attributes.

## Code Snippet
https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L258

## Tool used
Manual Review

## Recommendation
Add a new function in `Token` contract for token owner to call `onMinted(...)` function to generate attributes for the minted tokens after new metadata render is set.
