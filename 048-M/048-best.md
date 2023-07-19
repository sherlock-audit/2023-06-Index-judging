Bent Menthol Tortoise

medium

# Bidding could spend more quotes than it is supposed to

## Summary

After a bid is executed, the position units are updated through `Position.sol # calculateAndEditDefaultPosition`. Under certain conditions, it could allow the rebalance module to spend more quotes than it is supposed to.

## Vulnerability Detail

While bidding, after the bid is executed the position units of `sendToken` and `receiveToken` should be updated through `_updatePositionState`. Here, the preBidTokenBalance represents the token balance before bid execution.
```solidity
    /* AuctionRebalanceModuleV1.sol # _createBidInfo */
        // Store pre-bid token balances for later use.
        bidInfo.preBidTokenSentBalance = bidInfo.sendToken.balanceOf(address(_setToken));

    /* AuctionRebalanceModuleV1.sol # _updatePositionState */
        // Calculate and update positions for send tokens.
        (uint256 postBidSendTokenBalance,,) = setToken.calculateAndEditDefaultPosition(
            address(_bidInfo.sendToken),
            _bidInfo.setTotalSupply,
            _bidInfo.preBidTokenSentBalance
        );
```

In Position.sol, 
```solidity
    function calculateAndEditDefaultPosition(
    ...
        uint256 currentBalance = IERC20(_component).balanceOf(address(_setToken));
        uint256 positionUnit = _setToken.getDefaultPositionRealUnit(_component).toUint256();

        uint256 newTokenUnit;
        if (currentBalance > 0) {
            newTokenUnit = calculateDefaultEditPositionUnit(
                _setTotalSupply,
                _componentPreviousBalance,
                currentBalance,
                positionUnit
            );
        } else {
            newTokenUnit = 0;
        }
    ...

    function calculateDefaultEditPositionUnit(
        uint256 _setTokenSupply,
        uint256 _preTotalNotional,
        uint256 _postTotalNotional,
        uint256 _prePositionUnit
    )
        internal
        pure
        returns (uint256)
    {
        // If pre action total notional amount is greater then subtract post action total notional and calculate new position units
        uint256 airdroppedAmount = _preTotalNotional.sub(_prePositionUnit.preciseMul(_setTokenSupply));
        //@audit could revert if _postTotalNotional < airdroppedAmount
        return _postTotalNotional.sub(airdroppedAmount).preciseDiv(_setTokenSupply);
    }
```

As you can see, underflows could happen if `airdroppedAmount > _postTotalNotional`.
For this to be the case, `_preTotalNotional` should be greater than `_postTotalNotional` ( = for sendTokens) and the difference(spent amount) should be bigger than the tracked amount (`_prePositionUnit * _setTokenSupply` in the `airdroppeAmount` calculation).
```solidity
    /* _preTotalNotional = airdroppedAmount + trackedAmount
     * spentAmount = _preTotalNotional - _postTotalNotional
     * airdroppedAmount > _postTotalNotional
     * (airdroppedAmount + trackedAmount) + spentAmount > (_postTotalNotional + spentAmount) + trackedAmount
     * spentAmount > trackedAmount
     */
```
Reverting on excess spending of quote assets (more than the tracked amount) could be a desired feature, since it is not allowed for modules to spend over the tracked units.

For non-quote component assets, since there are proper checks against the maximum bid quantity, this could not be the case.
However for quote assets, this might be the case since there is no checks on the quote asset usage boundary.
```solidity
    /* AuctionRebalanceModuleV1.sol # _createBidInfo */
        // Calculate the auction size and direction.
        (bidInfo.isSellAuction, bidInfo.auctionQuantity) = _calculateAuctionSizeAndDirection(
            _setToken,
            _component,
            bidInfo.setTotalSupply
        );

        // Ensure that the component quantity in the bid does not exceed the available auction quantity.
        require(_componentQuantity <= bidInfo.auctionQuantity, "Bid size exceeds auction quantity");

    //@audit There are checks for components but...

        // Validate quote asset quantity against bidder's limit.
        _validateQuoteAssetQuantity(
            bidInfo.isSellAuction,
            quoteAssetQuantity,
            _quoteQuantityLimit,
            bidInfo.preBidTokenSentBalance
        );

    /* AuctionRebalanceModuleV1.sol # _validateQuoteAssetQuantity */
    //@audit No checks on tracked units for quote assets
    function _validateQuoteAssetQuantity(bool isSellAuction, uint256 quoteAssetQuantity, uint256 _quoteQuantityLimit, uint256 preBidTokenSentBalance) private pure {
        if (isSellAuction) {
            require(quoteAssetQuantity <= _quoteQuantityLimit, "Quote asset quantity exceeds limit");
        } else {
            require(quoteAssetQuantity >= _quoteQuantityLimit, "Quote asset quantity below limit");
            require(quoteAssetQuantity <= preBidTokenSentBalance, "Insufficient quote asset balance");
        }
    }
```

Returning to Position.sol, if the `currentBalance` is zero, it would not revert; allowing the rebalance module to spend up all the quote assets, including the airdropped amounts.

```solidity
    /* Position.sol # calculateAndEditDefaultPosition */
        if (currentBalance > 0) {
            newTokenUnit = calculateDefaultEditPositionUnit(
                _setTotalSupply,
                _componentPreviousBalance,
                currentBalance,
                positionUnit
            );
        } else {
            newTokenUnit = 0;
        }
```

## Impact

The module could spend untracked quote assets; which could interrupt other modules' functionalities

## Code Snippet

https://github.com/sherlock-audit/2023-06-Index/blob/main/index-protocol/contracts/protocol/modules/v1/AuctionRebalanceModuleV1.sol#L867-L874

## Tool used

Manual Review

## Recommendation

Add checks for quote assets if it spends more than the tracked units