Orbiting Carob Bison

medium

# Inconsistent custom metadata renderer initialization in `Manager`

## Summary
`Manager` creates proxies and initializes the custom metadata renderer in two places, in certain circumstances the 
metadata proxy may remain uninitialized.


## Vulnerability Detail
`IBaseMetadata` is the interface for Custom Metadata Renderers and defines the [initialization function](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/db232c649b425c36f5a93607c95cfdf0e5962b2f/nouns-protocol/src/token/metadata/interfaces/IBaseMetadata.sol#L22-L28)
```solidity
    /// @notice Initializes a DAO's token metadata renderer
    /// @param initStrings The encoded token and metadata initialization strings
    /// @param token The associated ERC-721 token address
    function initialize(
        bytes calldata initStrings,
        address token
    ) external;
```

### Deploy
`Manager.deploy` uses a semaphore derived from input arguments on which metadata render to use and then creates the [metadata proxy](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/db232c649b425c36f5a93607c95cfdf0e5962b2f/nouns-protocol/src/manager/Manager.sol#L120-L124)
```solidity
    // Check if the deployer is using an alternate metadata renderer. If not default to the standard one
    address metadataImplToUse = _tokenParams.metadataRenderer != address(0) ? _tokenParams.metadataRenderer : metadataImpl;

    // Deploy the remaining DAO contracts
    metadata = address(new ERC1967Proxy{ salt: salt }(metadataImplToUse, ""));
```

`Manager.deploy` initializes `metadata` directly, passing in [_tokenParams.initStrings](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/db232c649b425c36f5a93607c95cfdf0e5962b2f/nouns-protocol/src/manager/Manager.sol#L141-L141)
```solidity
        IBaseMetadata(metadata).initialize({ initStrings: _tokenParams.initStrings, token: token });
```

After `Manger.deploy` is complete, `metadata.initialize` will definitely have been called, irrespective of the length of ` _tokenParams.initStrings`. 


### setMetadataRenderer
`Manager.setMetadataRenderer` contains the same parameters, but uses [different variable names](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/db232c649b425c36f5a93607c95cfdf0e5962b2f/nouns-protocol/src/manager/Manager.sol#L172-L176)
```solidity
    function setMetadataRenderer(
        address _token,
        address _newRendererImpl,
        bytes memory _setupRenderer
    ) external returns (address metadata) {
```

The proxy is created from the given implementation, then initialization occurs only when `_setupRenderer` is [non-zero length](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/db232c649b425c36f5a93607c95cfdf0e5962b2f/nouns-protocol/src/manager/Manager.sol#L181-L186)
```solidity
        metadata = address(new ERC1967Proxy(_newRendererImpl, ""));
        daoAddressesByToken[_token].metadata = metadata;

        if (_setupRenderer.length > 0) {
            IBaseMetadata(metadata).initialize(_setupRenderer, _token);
        }
```

This is different to the behaviour of `Manager.deploy`, where the same zero length init string would have resulted in `initialize` being called.


## Impact
The Custom Metadata render section of the [Bali Spec](https://hackmd.io/peXISQ2CSQOwRGmvpUpK9A?view#Custom-Metadata-Renderers) defines the expectation of supporting custom renders (i.e. renderers not created by NounsBuilder).

During `Manager.deploy`, if a custom renderer is passed in as `_tokenParams.metadataRenderer` with an empty `initStrings` the created proxy will be initialized.

During `Manager.setMetadataRenderer`, if the given `_setupRenderer` is empty, then the created proxy will be left uninitialized.

There is no requirement that a custom renderer must use the `initStrings` parameter of `IBaseMetadata.initialize`, simply it must implement the function, meaning an empty `initStrings` is acceptable input.

When supporting custom renderers, the impact of being uninitialized cannot be known (as we are decoupled from the implementation specifics), but for proxies initialization is part of the expected life-cycle.

A hacky workaround is possible; in `setMetadataRenderer` you can pass in dummy data for `_setupRenderer` to make the length non-zero and get `initialize` called, but that is ...hacky. 


## Code Snippet
https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/db232c649b425c36f5a93607c95cfdf0e5962b2f/nouns-protocol/src/manager/Manager.sol#L184-L186


## Tool used
Manual Review


## Recommendation
Always initialize the Custom Metadata Renderer proxy.

Remove the guard around the initialize in [Manager.setMetadataRenderer](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/db232c649b425c36f5a93607c95cfdf0e5962b2f/nouns-protocol/src/manager/Manager.sol#L184-L186)
```diff
-        if (_setupRenderer.length > 0) {
             IBaseMetadata(metadata).initialize(_setupRenderer, _token);
-        }
```
