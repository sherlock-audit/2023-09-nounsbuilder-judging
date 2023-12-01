Rare Blue Monkey

medium

# Bids should not be allowed when Auction is paused, as the changes made can affect bidders and their strategies

## Summary
When the auction is paused, various settings can be changed. But the bidding is still allowed while the auction is paused. As the auction settings can be changed while the bidding is going on, it can negatively affect bidders in the strategy that they may be employing.


## Vulnerability Detail

When the auction is paused, the following auction settings can be changed:
* `settings.duration` - The time duration of each auction
* `settings.reservePrice` - The reserve price of each auction
* `settings.timeBuffer` - The time buffer of each auction
* `settings.minBidIncrement` - The minimum bid increment of each subsequent bid
* `founderReward` - The founder reward recipient address

Only the `settings.duration` is cached to the current bid, and everything else is a global setting common across all the auctions.

When the auction is paused, the auction settings can change, while the bidding is going on. This can throw off the strategy of the bidders.

Here are two simple examples:
* The `settings.minBidIncrement` gets reduced. The unaware bidder places their new bid with the previous value of `settings.minBidIncrement` in mind to meet the minimum bid value, which can lead to a higher bid amount than the minimum required after the update.
* The `founderRewards` gets increased. The unaware bidder places their bid without realizing that the `founderRewards` got increased, and they may not be supportive of this change.

## Proof of Concept

The test below can be run using this command: `forge test --match-test "test_bad_referral"`

(The actual attack happens in the `test_paused_bids()` function. Majority of the code is just setup and deployment steps)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.16;

import "forge-std/console.sol";
import { Test } from "forge-std/Test.sol";
import { IToken, Token } from "../src/token/Token.sol";
import { NounsBuilderTest } from "./utils/NounsBuilderTest.sol";
import { MockERC721 } from "./utils/mocks/MockERC721.sol";
import { MockImpl } from "./utils/mocks/MockImpl.sol";
import { MockPartialTokenImpl } from "./utils/mocks/MockPartialTokenImpl.sol";
import { Auction } from "../src/auction/Auction.sol";
import { IAuction } from "../src/auction/IAuction.sol";
import { AuctionTypesV2 } from "../src/auction/types/AuctionTypesV2.sol";
import { WETH } from "./utils/mocks/WETH.sol";
import { IManager, Manager } from "../src/manager/Manager.sol";
import { MetadataRenderer } from "../src/token/metadata/MetadataRenderer.sol";
import { MetadataRendererTypesV1 } from "../src/token/metadata/types/MetadataRendererTypesV1.sol";
import { IGovernor, Governor } from "../src/governance/governor/Governor.sol";
import { ITreasury, Treasury } from "../src/governance/treasury/Treasury.sol";
import { ERC1967Proxy } from "../src/lib/proxy/ERC1967Proxy.sol";
import { MockProtocolRewards } from "./utils/mocks/MockProtocolRewards.sol";

contract AuctionTest is Test {
    IManager.FounderParams[] internal foundersArr;
    IManager.TokenParams internal tokenParams;
    IManager.AuctionParams internal auctionParams;
    IManager.GovParams internal govParams;

    address internal aliceTheBidder;
    address internal bobTheBidder;
    address internal chuckTheBidder;

    address internal founder;
    address internal founder2;

    Token internal token;
    MetadataRenderer internal metadataRenderer;
    Auction internal auction;
    Treasury internal treasury;
    Governor internal governor;

    address internal dao;

    address internal managerImpl0;
    address internal managerImpl;
    address internal tokenImpl;
    address internal metadataRendererImpl;
    address internal auctionImpl;
    address internal treasuryImpl;
    address internal governorImpl;

    Manager internal manager;
    address internal rewards;

    address internal weth;

    function setUp() public virtual {
        aliceTheBidder = vm.addr(0xB1);
        chuckTheBidder = vm.addr(0xB2);

        founder = vm.addr(0xCAB);
        founder2 = vm.addr(0xDAD);

        dao = vm.addr(0xB0B);

        vm.deal(aliceTheBidder, 100 ether);
        vm.deal(bobTheBidder, 100 ether);
        vm.deal(chuckTheBidder, 100 ether);

        weth = address(new WETH());

        managerImpl0 = address(new Manager(address(0), address(0), address(0), address(0), address(0), address(0)));
        manager = Manager(address(new ERC1967Proxy(managerImpl0, abi.encodeWithSignature("initialize(address)", dao))));
        rewards = address(new MockProtocolRewards());

        tokenImpl = address(new Token(address(manager)));

        metadataRendererImpl = address(new MetadataRenderer(address(manager)));

        auctionImpl = address(new Auction(address(manager), address(rewards), weth, 0, 10));

        treasuryImpl = address(new Treasury(address(manager)));

        governorImpl = address(new Governor(address(manager)));

        managerImpl = address(new Manager(tokenImpl, metadataRendererImpl, auctionImpl, treasuryImpl, governorImpl, dao));

        vm.prank(dao);
        manager.upgradeTo(managerImpl);

        setFounderParams();
        setTokenParams();
        setAuctionParams();
        setGovParams();
        deploy();
        setMockMetadata();
    }

    function setFounderParams() internal virtual {
        address[] memory wallets = new address[](2);
        uint256[] memory percents = new uint256[](2);
        uint256[] memory vestingEnds = new uint256[](2);

        wallets[0] = founder;
        wallets[1] = founder2;

        percents[0] = 10;
        percents[1] = 5;

        vestingEnds[0] = 4 weeks;
        vestingEnds[1] = 4 weeks;

        uint256 numFounders = wallets.length;

        require(numFounders == percents.length && numFounders == vestingEnds.length);

        unchecked {
            for (uint256 i; i < numFounders; ++i) {
                foundersArr.push();

                foundersArr[i] = IManager.FounderParams({ wallet: wallets[i], ownershipPct: percents[i], vestExpiry: vestingEnds[i] });
            }
        }
    }

    function setTokenParams() internal virtual {
        bytes memory initStrings = abi.encode(
            "Mock Token",
            "MOCK",
            "This is a mock token",
            "ipfs://Qmew7TdyGnj6YRUjQR68sUJN3239MYXRD8uxowxF6rGK8j",
            "https://nouns.build",
            "http://localhost:5000/render"
        );

        tokenParams = IManager.TokenParams({ initStrings: initStrings, metadataRenderer: address(0), reservedUntilTokenId: 0 });
    }

    function setAuctionParams() internal virtual {
        auctionParams = IManager.AuctionParams({ reservePrice: 0, duration: 10 minutes, founderRewardRecipent: address(0), founderRewardBps: 0 });
    }

    function setGovParams() internal virtual {
        govParams = IManager.GovParams({
            timelockDelay: 2 days,
            votingDelay: 1 seconds,
            votingPeriod: 1 weeks,
            proposalThresholdBps: 50,
            quorumThresholdBps: 1000,
            vetoer: founder
        });
    }

    function setMockMetadata() internal {
        string[] memory names = new string[](1);
        names[0] = "testing";

        MetadataRendererTypesV1.ItemParam[] memory items = new MetadataRendererTypesV1.ItemParam[](2);
        items[0] = MetadataRendererTypesV1.ItemParam({ propertyId: 0, name: "failure1", isNewProperty: true });
        items[1] = MetadataRendererTypesV1.ItemParam({ propertyId: 0, name: "failure2", isNewProperty: true });

        MetadataRendererTypesV1.IPFSGroup memory ipfsGroup = MetadataRendererTypesV1.IPFSGroup({ baseUri: "BASE_URI", extension: "EXTENSION" });

        vm.prank(metadataRenderer.owner());
        metadataRenderer.addProperties(names, items, ipfsGroup);
    }

    function deploy() internal virtual {
        (address _token, address _metadata, address _auction, address _treasury, address _governor) = manager.deploy(
            foundersArr,
            tokenParams,
            auctionParams,
            govParams
        );

        token = Token(_token);
        metadataRenderer = MetadataRenderer(_metadata);
        auction = Auction(_auction);
        treasury = Treasury(payable(_treasury));
        governor = Governor(_governor);
    }

    function test_paused_bids() public {
        vm.prank(founder);
        auction.unpause();

        (uint256 tokenId, uint256 highestBid, address highestBidder, uint256 startTime, uint256 endTime, bool settled) = auction.auction();
        assertEq(tokenId, 2);
        assertEq(highestBid, 0);
        assertEq(highestBidder, address(0));
        assertEq(startTime, 1);
        assertEq(endTime, 1 + auctionParams.duration);
        assertEq(settled, false);

        // Alice places the first bid, and is the higest bidder
        vm.prank(aliceTheBidder);
        auction.createBid{ value: 1 ether }(2);

        (, uint256 highestBid1, address highestBidder1, , , ) = auction.auction();
        assertEq(highestBid1, 1 ether);
        assertEq(highestBidder1, aliceTheBidder);

        // The auction gets paused
        vm.prank(address(treasury));
        auction.pause();

        uint256 minBidIncrement1 = auction.minBidIncrement();
        assertEq(minBidIncrement1, 10);

        // Bob needs atleast 1.1 ether to be able to bid successfully
        vm.expectRevert(abi.encodeWithSignature("MINIMUM_BID_NOT_MET()"));
        vm.prank(bobTheBidder);
        auction.createBid{ value: 1.1 ether - 1 }(2);

        // Bob somehow arranges for 1.1 ether

        // Right before Bob places their bid, the minimumBidIncrement changes to 5,
        // and now Bob actually only requires 1.05 ether to out bid Alice, but the changes happened in parallel
        vm.prank(address(treasury));
        auction.setMinimumBidIncrement(5);

        uint256 minBidIncrement2 = auction.minBidIncrement();
        assertEq(minBidIncrement2, 5);

        // Unfortunately, the minimumBidIncrement changed right before Bob places their bid.
        // Now Bob is the highest bidder, but had to spend an extra 0.05 ether unnecessarily
        vm.prank(bobTheBidder);
        auction.createBid{ value: 1.10 ether }(2);

        (, uint256 highestBid2, address highestBidder2, , , ) = auction.auction();
        assertEq(highestBid2, 1.10 ether);
        assertEq(highestBidder2, bobTheBidder);
    }
}
```

## Impact

Because the auction parameters can be changed while bidders are placing bids, this can impact strategies of bidders, and lead to highers bids that are not necessary, and bidders might not realize the changes being made.

For each of the auction settings that are applicable to all the auction, here are a few examples of the impacts:

#### `settings.reservePrice` - The reserve price of each auction
If the `settings.reservePrice` is lowered, then the bidder unaware of this change might post their first bid based on the previous value of the `settings.reservePrice`.

The impact of this gets worse if the different between the old and new value is too high.

In summary, this can lead to over expending, that is almost loss of funds for the user which could have been avoided.

#### `settings.timeBuffer` - The time buffer of each auction

If thet time buffer is increased, then it might allow other bidders to get more time to arrange for funds, or re-think their strategies. In such cases, the current highest bidder will get into a disadvantage - if a new bid is placed, then they will need to place an even higher bid while accounting for the `settings.minBidIncrement`, which can lead them to lose the bid, and hence a failure of their strategy that they might have employed.

Similarly, if the time is reduced, the current highest bidder might have had a strategy where they woud time their strategy if another bidder tried to place a higher bid. But if the time is reduced, then the previous bidder might not even get a chance to place an even higher bid.

In summary, changes to this can lead to planned winning strategies to not work out for the bidders.

####  `settings.minBidIncrement` - The minimum bid increment of each subsequent bid

If this decreases, then the bidder might unintentionally place a bid based on the previous higher value, leading to unnecessarily higher bid.

Similar to the `settings.reservePrice`, this can lead to over expending, that is almost loss of funds for the user which could have been avoided.

#### `founderReward` - The founder reward recipient address

If this is increased, then the bidders might not agree with the decision, and would rather want to stop bidding or exist the auction. They might miss this change while placing their bid, which otherwise they would have not placed.

## Code Snippet

https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/auction/Auction.sol#L158

## Tool used

Manual Review

## Recommendation

There are a couple of options to make it fair for the bidders:
* Make all the settings freeze for the ongoing auction, and any setting changes are applied only to the subsequent auctions
* Do not allow bidding while the auction is paused
* Use timelocks for changes that might require time for users to evaluate their support, for instance the `founderReward`