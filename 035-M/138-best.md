Skinny Frost Puppy

high

# funds may be lost if founder(EOA) migrates DAO to L2 using CrossDomainMessanger and OptimisimPortal

## Summary
function `deploy()` in L2MigrationDeployer contract, receives the L1->L2 transaction and process it. the issue is that this function always assumes that L1 address is aliased when it's not send with CrossDomainMessenger, but this assumption is wrong and if L1 sender was EOA then OptimisimPortal doesn't alias the sender address. as we know when DAO is created the owner of the DAO is the first funder which can be an EOA account. so if the first founder decides to migrate to L2 and uses the mixed of OptimisimPortal and CrossDomainMessenger to migrate and setup DAO in L2, the funds will be lose because L2MigrationDeployer would calculate two different address in `_xMsgSender()` for the funder.

the docs should explicitly say that L2MigrationDeployer shouldn't be used with EOAs.

## Vulnerability Detail
This is `_xMsgSender()` code in the L2MigrationDeployer, as you can see when the sender is not `crossDomainMessenger` then code always undo the L1ToL2 aliasing. but the caller may be an EOA in the L1 (the first funder) and it would be wrong to undo aliasing.
```javascript
    function _xMsgSender() private view returns (address) {
        // Return the xDomain message sender
        return
            msg.sender == crossDomainMessenger
                ? ICrossDomainMessenger(crossDomainMessenger).xDomainMessageSender()
                : OPAddressAliasHelper.undoL1ToL2Alias(msg.sender);
    }
```

This is the POC for the issue:
1. User1 would create DOA1 and become the first funder and owner of the DAO.
2. after some test and user interactions, team would realize that gas cost is high and decide to migrate to L2.
3. User1 as owner would call OptimisimPortal with migration transaction.
4. L2MitrationDeployer would receive a call with sender User1 and it would deploy a DAO for address `unalised(User1)` and save the token address in `crossDomainDeployerToToken[]` mapping.
5. now User1 would wants to send the DAO's funds(ETH) from L1 to L2, this time User1 would use CrossDomainMessenger to make sure funds will be delivered(CrossDomainMessenger has replay feature).
6. when the new L1->L2 messages reaches the L2MigrationDeployer, the code would see that `_xMsgSender()` is CrossDomainMessenger and won't apply the unaliasing for sender address, but because now sender is User1 code wouldn't detect any mapping and deployed token for this address and transaction would revert.
7. the transferred ETH would be lost because that transaction won't be replayable. 
8. of course the issue can happen if we change the scenario and User1 is going to make so many calls from L1->L2 and if he uses mixed of CrossDomainMessenger and OptimisimPortal the issue will happen.


## Impact
funds will be lost if first founder (EOA) tries to migrate the DOA to L2.

## Code Snippet
https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/db232c649b425c36f5a93607c95cfdf0e5962b2f/nouns-protocol/src/deployers/L2MigrationDeployer.sol#L193-L199
https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/db232c649b425c36f5a93607c95cfdf0e5962b2f/nouns-protocol/src/deployers/L2MigrationDeployer.sol#L185-L191


## Tool used
Manual Review

## Recommendation
either handle the sender address differently in L2MigrationDeployer or warn users about this risk.