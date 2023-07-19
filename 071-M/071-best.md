Mysterious Infrared Stork

medium

# Variable _setToken.totalSupply() can cause unintended behaviour in AuctionRebalanceModuleV1.sol

## Summary
When setToken is not locked,it allows for user to redeem,issue setTokens which in turn changes `_setToken.totalSupply()`
totalSupply is used to calculate `currentNotional` and `targetNotional` both of these value are then used to calculate `maxComponentQty` (quantity of the component to be exchanged) or `bidInfo.auctionQuantity`,Changed setToken supply can cause either dos or bypass `targetUnit` upon bidding


## Vulnerability Detail

1)When `setToken` is not locked people are free to issue or redeem underlying tokens,Upon these actions `setToken` are either minted or burned.

2)when `setToken.totalSupply()` decreases or increases it affects `currentNotional` and `targetNotional`,
https://github.com/IndexCoop/index-protocol/blob/9a7a951d1246b46f7ac28d453d35c6804e1d89ed/contracts/protocol/modules/v1/AuctionRebalanceModuleV1.sol#L1030C1-L1046C6

3)lets suppose  `targetUnit` are set to 25e18(buy auction)(current position 20e18) for some component if some user decides to redeem some underlying token,`setToken.totalSupply()` will decrease lets suppose from 1e18 -> 0.5e18. `targetUnit` will remain same, now `currentNotional` and `targetNotional` both will be calculated according to new `setToken.totalSupply()` after calculation the new `maxComponentQty` will differ(less than real targetUnit),if some user wants to bid `_componentAmount=5e18` but `maxComponentQty`(bidInfo.auctionQuantity) will evaluate to be less than _componentAmount or real targetUnit,
this will cause require statement to revert and cause dos
https://github.com/IndexCoop/index-protocol/blob/9a7a951d1246b46f7ac28d453d35c6804e1d89ed/contracts/protocol/modules/v1/AuctionRebalanceModuleV1.sol#L796

4)Similar case for sell auction

5)Now lets suppose(buy auction)(current position 20) user decides to issue some underlying tokens which in turn will increase `setToken.totalSupply()` ,
`targetUnit` are set to 25e18, `setToken.totalSupply()` will increase lets suppose from 1e18 -> 3e18 `targetUnit` will remain same, now `currentNotional` and `targetNotional` both will be calculated according to new `setToken.totalSupply()` after calculation the new `maxComponentQty` will differ(greater than real targetUnit),if some user wants to bid `_componentAmount=15e18` then `maxComponentQty`(bidInfo.auctionQuantity) will evaluate to be greater than real `targetUnit`,by this user can bypass targetUnit
(only if enough quote assests are avaliable)

## Impact
user can either face dos or bypass `targetUnit` which breaks functionality of protocol and this is not intended behaviour

## Code Snippet
https://github.com/IndexCoop/index-protocol/blob/9a7a951d1246b46f7ac28d453d35c6804e1d89ed/contracts/protocol/modules/v1/BasicIssuanceModule.sol#L86C1-L119C6
https://github.com/IndexCoop/index-protocol/blob/9a7a951d1246b46f7ac28d453d35c6804e1d89ed/contracts/protocol/modules/v1/BasicIssuanceModule.sol#L129C1-L163C6
https://github.com/IndexCoop/index-protocol/blob/9a7a951d1246b46f7ac28d453d35c6804e1d89ed/contracts/protocol/modules/v1/AuctionRebalanceModuleV1.sol#L784C1-L797C1
https://github.com/IndexCoop/index-protocol/blob/9a7a951d1246b46f7ac28d453d35c6804e1d89ed/contracts/protocol/modules/v1/AuctionRebalanceModuleV1.sol#L898C1-L927C6
https://github.com/IndexCoop/index-protocol/blob/9a7a951d1246b46f7ac28d453d35c6804e1d89ed/contracts/protocol/modules/v1/AuctionRebalanceModuleV1.sol#L1030C1-L1046C6


## Tool used

Manual Review

## Recommendation
add extra checks in `_calculateAuctionSizeAndDirection` for targetunits

