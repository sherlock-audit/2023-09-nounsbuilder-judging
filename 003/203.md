Fancy Mauve Hare

medium

# A user can claim tokens from the `MerkleReserveMinter` contract multiple times because the burned tokens can be minted again.

## Summary
A user can claim tokens from the `MerkleReserveMinter` after migration by providing a Merkle proof within the claiming period. The claimed tokens can be transferred to a minter, who has the capability to burn the tokens. If this burning process occurs within the claiming period, the user can subsequently reclaim the burned tokens by submitting the original Merkle proof utilized during the initial claiming.
## Vulnerability Detail
"The process of claiming tokens through the `MerkleReserveMinter` contract involves users calling the `mintFromReserve` function, which executes the following tasks:

1. Calculates the fee based on the token price.
2. Verifies the Merkle proof provided by the user.
3. Mints the corresponding token if the provided proof is legitimate.

However, a critical vulnerability exists in the absence of a sanity check to ascertain whether the token has already been claimed. This loophole poses a security risk, allowing users to exploit the following sequence of events:

1. Bob claims tokenId(5) by providing the necessary Merkle proof, establishing ownership of tokenId(5).
2. Subsequently, Bob transfers tokenId(5) to Charlie, who is a designated minter.
3. As a minter, Charlie has the ability to burn tokens, and he proceeds to burn tokenId(5) within the claiming period.
4. Despite the burning of tokenId(5), the absence of a check in the `MerkleReserveMinter` contract allows Bob to reclaim tokenId(5) by submitting the same Merkle proof used initially. This oversight enables Bob to regain ownership of tokenId(5).

To address this vulnerability, it is crucial to implement a check within the `mintFromReserve` function to ensure that a token has not been previously claimed before allowing the transaction to proceed.
`Note : I`ve Already Discussed this issue with the dev @Neokry and he confirmed its a valid issue`
https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/minters/MerkleReserveMinter.sol#L154-L167
https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L211-L217
https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/lib/token/ERC721.sol#L191-L207
https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L293-L300


## Impact
This vulnerability has significant implications, particularly in the context of voting power. Assuming a one-to-one correspondence between votes and tokens (1 vote = 1 token), the impact on voting power is notable. If Bob initially owns only one token and transfers it to a minter, resulting in the token being burned, Bob theoretically should have no votes. However, due to the identified vulnerability, Bob can exploit the situation by reclaiming the burned token, thereby increasing his voting power by one vote.

The potential consequences are particularly pronounced in scenarios where users are required to secure tokens through competitive auctions, bidding the highest amount. The vulnerability introduces the risk of users obtaining tokens at a significantly lower cost by reclaiming tokens that were initially burned by the minter. This undermines the integrity of the token acquisition process and compromises the expected correlation between token value and the investment made during the auction.

## Code Snippet
`Test :`
```solidity
    function test_MintFlow() public {
        deployAltMock(20);

        bytes32 root = bytes32(0x5e0da80989496579de029b8ad2f9c234e8de75f5487035210bfb7676e386af8b);

        MerkleReserveMinter.MerkleMinterSettings memory settings = MerkleReserveMinter.MerkleMinterSettings({
            mintStart: 0,
            mintEnd: uint64(block.timestamp + 1000),
            pricePerToken: 0 ether,
            merkleRoot: root
        });

        vm.prank(address(founder));
        minter.setMintSettings(address(token), settings);

        (uint64 mintStart, uint64 mintEnd, uint64 pricePerToken, bytes32 merkleRoot) = minter.allowedMerkles(address(token));
        assertEq(mintStart, settings.mintStart);
        assertEq(mintEnd, settings.mintEnd);
        assertEq(pricePerToken, settings.pricePerToken);
        assertEq(merkleRoot, settings.merkleRoot);

        TokenTypesV2.MinterParams memory params = TokenTypesV2.MinterParams({ minter: address(minter), allowed: true });
        TokenTypesV2.MinterParams memory params1 = TokenTypesV2.MinterParams({ minter: claimer1, allowed: true }); // Setting the 
 // Claimer1 as the minter so that he can burn the claimed token for making the test simple.
        TokenTypesV2.MinterParams[] memory minters = new TokenTypesV2.MinterParams[](2);
        minters[0] = params;
        minters[1] = params1;
        vm.prank(address(founder));
        token.updateMinters(minters);

        bytes32[] memory proof = new bytes32[](1);
        proof[0] = bytes32(0xd77d6d8eeae66a03ce8ecdba82c6a0ce9cff76f7a4a6bc2bdc670680d3714273);

        MerkleReserveMinter.MerkleClaim[] memory claims = new MerkleReserveMinter.MerkleClaim[](1);
        claims[0] = MerkleReserveMinter.MerkleClaim({ mintTo: claimer1, tokenId: 5, merkleProof: proof });

        minter.mintFromReserve(address(token), claims); //Initial claim 

        vm.startPrank(token.ownerOf(5));
        token.burn(5); // burning the claimed token
        vm.stopPrank();

        MerkleReserveMinter.MerkleClaim[] memory claims1 = new MerkleReserveMinter.MerkleClaim[](1);
        claims1[0] = MerkleReserveMinter.MerkleClaim({ mintTo: claimer1, tokenId: 5, merkleProof: proof });

        minter.mintFromReserve(address(token), claims1); // Second claim

        assertEq(token.ownerOf(5), claimer1);
    }
```

`Result :`

```solidity
Running 9 tests for test/MerkleReserveMinter.t.sol:MerkleReserveMinterTest
[PASS] testRevert_InvalidProof() (gas: 3310228)
[PASS] testRevert_InvalidValue() (gas: 3318272)
[PASS] testRevert_MintEnded() (gas: 3307581)
[PASS] testRevert_MintNotStarted() (gas: 3307495)
[PASS] test_MintFlow() (gas: 3490797) //Passing Test
[PASS] test_MintFlowSetFromToken() (gas: 3437856)
[PASS] test_MintFlowWithValue() (gas: 3488261)
[PASS] test_MintFlowWithValueMultipleTokens() (gas: 3617296)
[PASS] test_ResetMint() (gas: 3241395)
Test result: ok. 9 passed; 0 failed; finished in 38.09ms

```
## Tool used

Manual Review, Foundry

## Recommendation
Create a mapping that tracks the claiming of tokens.
`mapping(address user=> uint tokenId => bool isClaimed) public TokenClaims;`
