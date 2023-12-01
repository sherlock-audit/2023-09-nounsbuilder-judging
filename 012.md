Electric Khaki Mantis

medium

# The founder who is required to receive all base tokens will not be taken into consideration

## Summary
The initialize function of the ```Token``` contract receives an array of ```FounderParams```, which contains the ownership percent of each founder as a ```uint256```. 

## Vulnerability Detail
The initialize function checks that the sum of the percents is not above 99. Any founder with a token ownership percent of 100 will make the ```_addFounders()``` function reverting while it should not be the case. Note that the specific attack vector that would trigger this can only be triggered at initial setup which justifies the medium severity.

## Impact

This can lead to wrong assignment of the base tokens, and can also lead to a situation where not all users will get the correct share of base tokens.

## Code Snippet

https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L141C35-L141C35

## Proof of Concept

On the following test code, we initialised the proxy using an array of one founder which hold an ownership percent of 100. As described, the ```_addFounder()``` function reverted with ``` INVALID_FOUNDER_OWNERSHIP()```.

```Solidity
// SPDX-License-Identifier: MIT

pragma solidity 0.8.16;

import { Test } from "forge-std/Test.sol";
import { IManager } from "../src/manager/Manager.sol";
import { IToken, Token } from "../src/token/Token.sol";
import { TokenTypesV1 } from "../src/token/types/TokenTypesV1.sol";
import { ERC1967Proxy } from "../src/lib/proxy/ERC1967Proxy.sol";

contract FounderGettingAllBaseTokensBug is Test, TokenTypesV1 {
    Token imp;
    address proxy;

    function setUp() public virtual {
        // Deploying the implementation and the proxy
        imp = new Token(address(this));
        proxy = address(new ERC1967Proxy(address(imp), ""));
    }

    function testFounderGettingAllBaseTokensBug() public {
        IToken token = IToken(proxy);
        address AliceFounder = address(0xdeadbeef);

        // Creating 1 founders with `ownershipPct = 100`
        IManager.FounderParams[] memory founders = new IManager.FounderParams[](2);
        founders[0] = IManager.FounderParams({ wallet: AliceFounder, ownershipPct: 100, vestExpiry: 1 weeks });

        // Initializing the proxy with the founders data
        vm.expectRevert(abi.encodeWithSignature("INVALID_FOUNDER_OWNERSHIP()"));
        token.initialize(
            founders,
            // we don't care about these
            abi.encode("", "", "", "", ""),
            0,
            address(0),
            address(0),
            address(0)
        );
    }
}

```

## Tool used

Manual Review

## Recommendation

We recommend to check a totalOwnership of strictly more than 100 instead of 99 to ensure token owners having a base token ownership percent of 100%.
