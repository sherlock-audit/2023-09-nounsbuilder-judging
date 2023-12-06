Low Graphite Tardigrade

high

# Contracts can't get initialized

https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/db232c649b425c36f5a93607c95cfdf0e5962b2f/nouns-protocol/src/token/Token.sol#L57-L83
https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/db232c649b425c36f5a93607c95cfdf0e5962b2f/nouns-protocol/src/auction/Auction.sol#L70-L106
https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/db232c649b425c36f5a93607c95cfdf0e5962b2f/nouns-protocol/src/auction/Auction.sol#L70-L106
https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/db232c649b425c36f5a93607c95cfdf0e5962b2f/nouns-protocol/src/manager/Manager.sol#L51-L79

## Summary

In the upgradable contracts you are using both `constructor` and `initialize` function for initializing the contract 

## Vulnerability Detail
In contracts deployed in the Manager contract, in their constructor you first set the manager's contract address and in the `initialize` function you require the caller to be the manager however the problem is since these contracts meant to be upgradable the initialize function will always fail in this case because when you set the manager address in the `constructor`  the address will be stored in implementation contract's storage not in the proxy contract, so when you try to initialize the contract from manager contract the tx will fail because in the proxy contract's storage the manager address is still `address(0)` , So any `initialize`  function that require the caller to be manager will faill because the manager is not set in the proxy contract.
```solidity
File: 2023-09-nounsbuilder\nouns-protocol\src\token\Token.sol
57:     constructor(address _manager) payable initializer {
58:         manager = IManager(_manager);
59:     }


72:     function initialize(
73:         IManager.FounderParams[] calldata _founders,
74:         bytes calldata _initStrings,
75:         uint256 _reservedUntilTokenId,
76:         address _metadataRenderer,
77:         address _auction,
78:         address _initialOwner
79:     ) external initializer {
80:         // Ensure the caller is the contract manager
81:         if (msg.sender != address(manager)) {
82:             revert ONLY_MANAGER();
83:         }
```

This problem also exist for Manager contract itself because you use both `constructor` and `initialize` function for initializing the contract, any value set in the constructor won't exist in the proxy contract and most of the contract's functionality won't work since important values are not set.

## Impact

those contracts that is meant to be deployed by manager won't be deployed and fail, and those upgradable contracts which use both `constructor` and `initialize` function will not work as expected.

## Code Snippet
here is the `Manager.deploy()` which deploys and intialize the upgradable contracts
```solidity
File: 2023-09-nounsbuilder\nouns-protocol\src\manager\Manager.sol
124:             metadata = address(new ERC1967Proxy{ salt: salt }(metadataImplToUse, ""));
125:             auction = address(new ERC1967Proxy{ salt: salt }(auctionImpl, ""));
126:             treasury = address(new ERC1967Proxy{ salt: salt }(treasuryImpl, ""));
127:             governor = address(new ERC1967Proxy{ salt: salt }(governorImpl, ""));
128: 
129:             daoAddressesByToken[token] = DAOAddresses({ metadata: metadata, auction: auction, treasury: treasury, governor: governor });
130:         }
131: 
132:         // Initialize each instance with the provided settings
133:         IToken(token).initialize({
134:             founders: _founderParams,
135:             initStrings: _tokenParams.initStrings,
136:             reservedUntilTokenId: _tokenParams.reservedUntilTokenId,
137:             metadataRenderer: metadata,
138:             auction: auction,
139:             initialOwner: founder
140:         });
141:         IBaseMetadata(metadata).initialize({ initStrings: _tokenParams.initStrings, token: token });
142:         IAuction(auction).initialize({
143:             token: token,
144:             founder: founder,
145:             treasury: treasury,
146:             duration: _auctionParams.duration,
147:             reservePrice: _auctionParams.reservePrice,
148:             founderRewardRecipent: _auctionParams.founderRewardRecipent,
149:             founderRewardBps: _auctionParams.founderRewardBps
150:         });
151:         ITreasury(treasury).initialize({ governor: governor, timelockDelay: _govParams.timelockDelay });
152:         IGovernor(governor).initialize({
153:             treasury: treasury,
154:             token: token,
155:             vetoer: _govParams.vetoer,
156:             votingDelay: _govParams.votingDelay,
157:             votingPeriod: _govParams.votingPeriod,
158:             proposalThresholdBps: _govParams.proposalThresholdBps,
159:             quorumThresholdBps: _govParams.quorumThresholdBps
160:         });
```


## Tool used

Manual Review

## Recommendation
Since contracts are deployed and initialized in same transaction there is no risk of front running for the initialize function, So consider not setting any values in the constructor for upgradable contract the only thing you need to do in constructor is disable the initializer and set all of the values in the initialize function 
