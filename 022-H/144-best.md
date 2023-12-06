Skinny Frost Puppy

high

# during migration, attacker can sell his NFT tokens in L1 after migration snapshot and then receive it in L2 after migration

## Summary
DAO owner can migrate it to the L2, for migration a snapshot of token holder in L1 is taken then that snapshot will be send to L2 to deploy the new DAO in L2. the issue is that DAO Token contract is not pausable, so during the migration, attacker which owns DAO NFT, after the snapshot is taken, can sell his NFT in L1 and later receive his NFT in L2. This will cause loss to any user that buys or receive the NFTs during the migration. DAO NFT Token should have pause feature and DAO should be able to pause the token during the migration so no one could transfer it during the migration and after it.

## Vulnerability Detail
This is the migration process:
![image](https://github.com/sherlock-audit/2023-09-nounsbuilder-0xunforgiven/assets/108366834/488b808b-5241-4655-890c-43d90f7cc367)

attacker can exploit this process because during the migration and after the snapshot of NFT ownership is taken, DAO NFT token is transferrable and attacker can sell his token in L1 and later receive that token in L2. DAO owner can pause Auction, but that will only stop minting new tokens and it won't stop Token transfers. 

warning users about the migration and saying that L1 NFT is going to be worthless is not gonna solve this issue, because there are a lot of DEX and on-chain marketplaces that attacker can sell its NFT during migration in L1.

## Impact
users who buys or receives NFT tokens during migration in L1 will loss their funds.

## Code Snippet
https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/db232c649b425c36f5a93607c95cfdf0e5962b2f/nouns-protocol/src/deployers/L2MigrationDeployer.sol#L92-L123
https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/db232c649b425c36f5a93607c95cfdf0e5962b2f/nouns-protocol/src/auction/Auction.sol#L362-L367

## Tool used
Manual Review

## Recommendation
add pause feature to DAO NFT Token, and allow owner to pause the DAO Token during the migration.