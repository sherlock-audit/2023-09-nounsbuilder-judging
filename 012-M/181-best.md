Furry Peach Turkey

high

# [High] Token upgradeable contract inherits from EIP712, but didn't initialize it via __EIP712_init()

## Summary
[Token](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L22) contract inherits [ERC721Votes](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/lib/token/ERC721Votes.sol#L15) which in turn inherits [EIP712](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/lib/utils/EIP712.sol). But [EIP712](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/lib/utils/EIP712.sol#L48-L54) initialization never called.

Also need to say, that currently Token's [supportsInterface()](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/lib/token/ERC721.sol#L61-L66) wouldn't correctly handle check for IEIP712.

## Vulnerability Detail
Because of upgradable nature of the contracts, it's mandatory to call __EIP712_init() for proper initialization and ability to work correctly with it's functionality.
```solidity
    function __EIP712_init(string memory _name, string memory _version) internal onlyInitializing {
        HASHED_NAME = keccak256(bytes(_name));
        HASHED_VERSION = keccak256(bytes(_version));

        INITIAL_CHAIN_ID = block.chainid;
        INITIAL_DOMAIN_SEPARATOR = _computeDomainSeparator();
    }
```

ERC721Votes contract use [DOMAIN_SEPARATOR()](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/lib/utils/EIP712.sol#L62-L70) during the [delegateBySig()](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/lib/token/ERC721Votes.sol#L144-L174) to calculate digest for signature verification.

```solidity
        unchecked {
            // Compute the hash of the domain seperator with the typed delegation data
            digest = keccak256(
                abi.encodePacked("\x19\x01", DOMAIN_SEPARATOR(), keccak256(abi.encode(DELEGATION_TYPEHASH, _from, _to, nonces[_from]++, _deadline)))
            );
        }
```

## Impact
Without proper initialization DOMAIN_SEPARATOR() will provide unexpected data, and name and version of the contract wouldn't be included to the result calculation.
Because DOMAIN_SEPARATOR() function is in active use during signature verification it can lead into errors and unexpected vulnerabilities.

## Code Snippet
[Token](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L22):
```solidity
contract Token is IToken, VersionedContract, UUPS, Ownable, ReentrancyGuard, ERC721Votes, TokenStorageV1, TokenStorageV2, TokenStorageV3 
```
[ERC721Votes](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/lib/token/ERC721Votes.sol#L15):
```solidity
abstract contract ERC721Votes is IERC721Votes, EIP712, ERC721 
```

[EIP712](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/lib/utils/EIP712.sol#L48-L54):
```solidity
    function __EIP712_init(string memory _name, string memory _version) internal onlyInitializing {
        HASHED_NAME = keccak256(bytes(_name));
        HASHED_VERSION = keccak256(bytes(_version));

        INITIAL_CHAIN_ID = block.chainid;
        INITIAL_DOMAIN_SEPARATOR = _computeDomainSeparator();
    }
```


## Tool used

Manual Review

## Recommendation
Call  __EIP712_init() during the Token initialization to keep both upgreadability and proper [IEIP712](https://eips.ethereum.org/EIPS/eip-712#definition-of-domainseparator[EIP-712]) implementation.

Also, add EIP-712 verification into Token's supportsInterface() implementation.