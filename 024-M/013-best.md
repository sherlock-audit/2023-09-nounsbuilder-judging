Cold Blonde Sparrow

medium

# The claimer using the `MerkleReserveMinter::mintFromReserve()` function may be frontrun by chance, resulting in the deposited assets being stuck and preventing the `treasury` from receiving them

## Summary

The claimer using the `MerkleReserveMinter::mintFromReserve()` function may be frontrun by chance, resulting in the deposited assets being stuck in the `MerkleReserveMinter` contract and preventing the `treasury` from receiving them.

## Vulnerability Detail

The DAO can establish the [MerkleReserveMinter](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/minters/MerkleReserveMinter.sol) contract as a minter within the token contract using the [Token::updateMinters()](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L465) function. Users can then utilize the [MerkleReserveMinter::mintFromReserve()](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/minters/MerkleReserveMinter.sol#L129) function in receive their corresponding tokens. The DAO can specify the [pricePerToken](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/minters/MerkleReserveMinter.sol#L60) and other settings so the users can mints reserved tokens based on a merkle tree, additionally those settings can be changed by the DAO using [MerkleReserveMinter::setMintSettings()](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/minters/MerkleReserveMinter.sol#L216) function.

The issue arises when the claimer may be frontrun by chance by a `pricePerToken` change generated by the owner/tresury. Please consider the next scenario:

1. The `MerkleReserveMinter` is setting up as a minter using the `pricePerToken` to `1 ether`.
2. Some users claim tokens.
3. Users want to change the `pricePerToken` to `zero`, so they create a proposal in order to change the price.
4. Meanwhile, `ClaimerA` obtains the `pricePerToken` using the [MerkleReserveMinter::getTotalFeesForMint()](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/minters/MerkleReserveMinter.sol#L180), so the `pricePerToken` is 1 ether + some builder fees and calls the [MerkleReserveMinter::mintFromReserve()](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/minters/MerkleReserveMinter.sol#L129) func.
5. Unfortunately, the price-changing transaction from `step 3` is executed before the `mintFromReserve()` transaction from `step 4`. Now the `pricePerToken` is `zero`.
6. The `step 4` finally is executed in the blockchain, the problem is that the transaction is transferred `1 ether + builder fees` (the previous price per token). Now those assets are stuck in the `MerkleReserveMinter` contract and preventing the `treasury` from receiving them.

## Impact

Claimers may be frontrun by chance, resulting in the deposited assets being trapped in the `MerkleReserveMinter` contract and preventing the `treasury` from receiving them. It is possible that the changing price action may occur before the claimer minting process because the [treasury executor](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/governance/governor/Governor.sol#L345) may pay more gas so the transaction is executed before the claimer minting transaction.

I consider this issue to be of medium severity because the price-changing can be executed by [anyone](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/governance/governor/Governor.sol#L345) at any time, claimants may send assets during the execution of price-changing proposals, resulting in the loss of those assets by the Treasury.

The next test scenario is initially set with a `pricePerToken` of `1 ether`. A claimer then obtains the price per token and calls `mintFromReserve` but the price-changing to zero transaction is executed before the minting action, resulting in the deposited assets being trapped in the contract and preventing the `treasury` from receiving them. Test steps:

1. Set minter with a pricePerToken `1 ether`.
2. `Claimer` gets the price for mint, it is `1 ether + fees`.
3. An update by chance for the `pricePerToken to zero price` is executed before the claimer transaction.
4. The claimer transaction is executed using the price he previously got (1 ether + fees).
5. The claimer receives his token but the fees (msg.value) are now trapped in the `MerkleReserveMinter` contract.


```solidity
    function test_MintFlowWithValueCanBeStuck() public {
        // The claimer who is using the MerkleReserveMinter can be frontran by chance causing
        // the assets that is deposited to be stuck in the MerkleReserveMinter contract
        //
        //
        // 1. Set minter with a pricePerToken 1 ether
        deployAltMock(20);
        bytes32 root = bytes32(0x5e0da80989496579de029b8ad2f9c234e8de75f5487035210bfb7676e386af8b);
        MerkleReserveMinter.MerkleMinterSettings memory settings = MerkleReserveMinter.MerkleMinterSettings({
            mintStart: 0,
            mintEnd: uint64(block.timestamp + 1000),
            pricePerToken: 1 ether,
            merkleRoot: root
        });
        vm.prank(address(founder));
        minter.setMintSettings(address(token), settings);
        TokenTypesV2.MinterParams memory params = TokenTypesV2.MinterParams({ minter: address(minter), allowed: true });
        TokenTypesV2.MinterParams[] memory minters = new TokenTypesV2.MinterParams[](1);
        minters[0] = params;
        vm.prank(address(founder));
        token.updateMinters(minters);
        bytes32[] memory proof = new bytes32[](1);
        proof[0] = bytes32(0xd77d6d8eeae66a03ce8ecdba82c6a0ce9cff76f7a4a6bc2bdc670680d3714273);
        MerkleReserveMinter.MerkleClaim[] memory claims = new MerkleReserveMinter.MerkleClaim[](1);
        claims[0] = MerkleReserveMinter.MerkleClaim({ mintTo: claimer1, tokenId: 5, merkleProof: proof });
        //
        // 2. Claimer gets the price for mint, it is 1 ether + fees
        uint256 fees = minter.getTotalFeesForMint(address(token), claims.length);
        assertEq(fees, 1 ether + minter.BUILDER_DAO_FEE());
        //
        // 3. An update by chance for the pricePerToken to zero price is executed before the claimer transaction.
        settings = MerkleReserveMinter.MerkleMinterSettings({
            mintStart: 0,
            mintEnd: uint64(block.timestamp + 1000),
            pricePerToken: 0,  // zero price per token
            merkleRoot: root
        });
        vm.prank(address(founder));
        minter.setMintSettings(address(token), settings);
        //
        // 4. Claimer transaction is executed using the value he previously got (1 ether + fees)
        vm.deal(claimer1, fees);
        vm.prank(claimer1);
        minter.mintFromReserve{ value: fees }(address(token), claims);
        //
        // 5. The claimer gets his token but fees (msg.value) now are stuck in the Minter contract
        assertEq(token.ownerOf(5), claimer1);
        assertEq(address(treasury).balance, 0);
        assertEq(address(minter).balance, fees);
    }
```

## Code Snippet

- [MerkleReserveMinter::mintFromReserve()](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/minters/MerkleReserveMinter.sol#L129) 
- [MerkleReserveMinter::setMintSettings()](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/minters/MerkleReserveMinter.sol#L216)

## Tool used

Manual review

## Recommendation

The recommendation is to add a parameter which helps the claimer to specify the `pricePerToken` he is willing to pay:

```diff
--  function mintFromReserve(address tokenContract, MerkleClaim[] calldata claims) public payable {
++  function mintFromReserve(address tokenContract, MerkleClaim[] calldata claims, uint64 _pricePerToken) public payable {
        MerkleMinterSettings memory settings = allowedMerkles[tokenContract];
++      if (_pricePerToken != settings.pricePerToken) revert();
        uint256 claimCount = claims.length;
    ...
    ...
    ...
    }
```