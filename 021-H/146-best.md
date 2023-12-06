Abundant Glossy Ladybug

high

# L2MigrationDeployer#depositToTreasury() sends funds to an incorrect address

## Summary

`depositToTreasury()` in `L2MigrationDeployer.sol` attempts to use `call()` ”to deposit ether from L1 DAO treasury to L2 DAO treasury”, as per the NatSpec comment. However you cannot simply use `call()` to transfer ether from the mainnet to a L2.

## Vulnerability Detail

`depositToTreasury()` first gets the L2 treasury address associated with the `msg.sender`, [L2MigrationDeployer.sol#L156](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/deployers/L2MigrationDeployer.sol#L156):

```solidity
(, , , address treasury, ) = _getDAOAddressesFromSender();
```

The function then attempts to send the `msg.value` to the treasury using `call()`, [L2MigrationDeployer.sol#L159](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/deployers/L2MigrationDeployer.sol#L159):

```solidity
(bool success, ) = treasury.call{ value: msg.value }("");
```

The problem is that you cannot bridge ether from the mainnet to an OP Stack layer 2 such as Optimism using `call()`. This does not send the ether to the L2 treasury, rather it sends it to the corresponding address on the mainnet which likely does not exist or is controlled by someone else.

## Impact

There are three outcomes that determine the impact:

1. The receiving address does not correspond to an existing contract or EOA. In this case the ether will be successfully transferred to that address. Since there is no associated contract logic or an active EOA controlling it, the ether will essentially be permanently locked at that address. This is the most likely outcome.
2. The receiving address is an existing EOA. In this case the ether will be successfully transferred to that address and they can use it as they see fit.
3. The receiving address is an existing contract. If the contract can accept ether (i.e., it has a payable fallback function), the ether will be deposited into the contract. If the contract cannot accept ether, the transaction will fail and the ether will not be sent. This is the most unlikely outcome.

To summarise, the impact is most probably that the ether will be sent to a non-existing EOA and permanently locked at that address.

## Code Snippet

[L2MigrationDeployer.sol#L154-L165](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/deployers/L2MigrationDeployer.sol#L154-L165):

```solidity
///@notice Helper method to deposit ether from L1 DAO treasury to L2 DAO treasury
    function depositToTreasury() external payable {
        (, , , address treasury, ) = _getDAOAddressesFromSender();

        // Transfer ether to treasury
        (bool success, ) = treasury.call{ value: msg.value }("");

        // Revert if transfer fails
        if (!success) {
            revert TRANSFER_FAILED();
        }
    }
```

## Tool used

Manual Review

## Recommendation

To bridge ether from an address on the mainnet to an OP Stack layer 2 address, you must use the `depositETHTo()` function on the [L1StandardBridge](https://github.com/ethereum-optimism/optimism/blob/65ec61dde94ffa93342728d324fecf474d228e1f/packages/contracts-bedrock/contracts/L1/L1StandardBridge.sol#L123-L143) contract. Here is how it can be implemented in `L2MigrationDeployer.sol`, [L2MigrationDeployer.sol#L154-L165](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/deployers/L2MigrationDeployer.sol#L154-L165):

```solidity
	// SPDX-License-Identifier: MIT
	pragma solidity 0.8.16;
	
	...
	
+	interface IL1StandardBridge {
+		function depositETHTo(address _to, uint32 _minGasLimit, bytes calldata _extraData) external payable;
+	}
	
	contract L2MigrationDeployer {
+		IL1StandardBridge public l1StandardBridge;
		
		...
		
		constructor(
			address _manager,
			address _merkleMinter,
			address _crossDomainMessenger,
+			address _l1StandardBridge
		) {
			manager = _manager;
			merkleMinter = _merkleMinter;
			crossDomainMessenger = _crossDomainMessenger;
+			l1StandardBridge = IL1StandardBridge(_l1StandardBridge);
		}
		
		...
		
		///@notice Helper method to deposit ether from L1 DAO treasury to L2 DAO treasury
		function depositToTreasury(
+			uint32 minGasLimit,
+			bytes calldata extraData,
+			uint256 amount
-		) external payable {
+		) external {
			(, , , address treasury, ) = _getDAOAddressesFromSender();
			
			
			// Transfer ether to treasury
-			(bool success, ) = treasury.call{ value: msg.value }("");
			
			
			// Revert if transfer fails
-			if (!success) {
-				revert TRANSFER_FAILED();
-			}
		
+			l1StandardBridge.depositETHTo{value: amount}(address(treasury), minGasLimit, extraData);
		}
	...
	}
```

You can see more about this method here: [Depositing ETH](https://community.optimism.io/docs/developers/bridge/standard-bridge/#depositing-eth).