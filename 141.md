Skinny Frost Puppy

high

# non-EOA NFT holders in L1 will lose their funds if DAO is migrated to L2

## Summary
DAO owners can migrate DAO from L1 to L2. to migrate, they would take a snapshot of NFT holders and send the merkle tree root hash of the snapshot to the L2 that shows which address own which otkenId. the issue is that L1 contracts address in L2 is aliased, and L1 contract can't handle their exact address in L2. there is no info about NFT owned by contracts addresses in L1 when migrating them, if the NFTs minted for exact addresses in L2 then those NFTs will be lost. the second issue is that, L1 contracts may not be able to handle the aliased address in L2, because L1 contract need to send transactions to L2 through OptimisimPortal but contract may not have that ability, so these contract will lose their funds too.

## Vulnerability Detail
This is migration process explained by sponsors:
![image](https://github.com/sherlock-audit/2023-09-nounsbuilder-0xunforgiven/assets/108366834/77a26847-e9fc-47a7-b231-85170a8a66b4)

as you can see it doesn't explain what will happen to NFTs that are owned by contracts addresses in L1. there is no info about it in the docs too.
there is two option for minting those  NFTs in L2: (mining mean adding (address, id) pair to Minter merkle tree)
1. mint those NFTs owned by contract addresses, for exact addresses in the L2.
2. mint those NFTs owned by contract addresses, for aliased addresses in the L2.

the option 1 will defiantly  make those NFTs to be stucked forever, because whenever L1 contract send transaction to L2 it needs to use OptimisimPortal and OptimisimPortal will alias the L1 contract address, in short L1 contract control the L2 aliased address. further more the exact L2 address may belong to other person (for example in vault factories, the same vault address in L1 and L2 may belong to different users in L1 and L2).

the option 2 will make most of the NFTs to be lost, L1 contract can only handle its NFT in L2 (in its aliased L2 address) if it has functionality to send L2 transaction by using OptimisimPortal, for sure not all contracts in L1 are going to be able to handle their NFT in L2.

so either way those tokens are going to be lost. 

## Impact
NFTs owned by contract addresses in L1 will be lost if migration to L2 happens.

## Code Snippet
https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/db232c649b425c36f5a93607c95cfdf0e5962b2f/nouns-protocol/src/deployers/L2MigrationDeployer.sol#L185-L191

## Tool used
Manual Review

## Recommendation
either implement a mirror L1 Token that replaced by L1 DAO token after migration and when L1 contracts transfers their mirror token in L1, their real NFT token is transferred in L2 (the mirror token will forwards the transfer to L2 token).
or warn users and DAOs about this risk.