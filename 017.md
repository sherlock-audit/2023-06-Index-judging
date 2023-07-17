Bouncy Blue Goldfish

medium

# Protocol fees can be bypassed entirely

## Summary
SetToken holders are incentivized to call `BasicIssuanceModule.redeem()` prior to a rebalance auction starting. This will redeem their assets at the current value and will avoid exposure to any protocol fees, since protocol fees are collected upon rebalancing. They are then able to call `BasicIssuanceModule.issue()` to mint their SetTokens again.

Prior to the start of a rebalance, it will be likely to see a high amount of redeems which is negative for the protocol. Index will lose out on fees.

## Vulnerability Detail
Fees are taken from the SetToken's balances after each rebalance operation. With the inclusion of the `BasicIssuanceModule.sol`, a user is able to redeem before any rebalancing can occur (front-run the `startRebalance()` call) and then re-issue the set token after the rebalance auction has finished.

After the rebalance swap has been made, the Index fees are taken directly from the SetToken's token balances:

```solidity
    function _accrueProtocolFee(BidInfo memory _bidInfo) internal returns (uint256) {
        IERC20 receiveToken = IERC20(_bidInfo.receiveToken);
        ISetToken setToken = _bidInfo.setToken;


        // Calculate the amount of tokens exchanged during the bid.
        uint256 exchangedQuantity = receiveToken.balanceOf(address(setToken))
            .sub(_bidInfo.preBidTokenReceivedBalance);
        
        // Calculate the protocol fee.
        uint256 protocolFee = getModuleFee(AUCTION_MODULE_V1_PROTOCOL_FEE_INDEX, exchangedQuantity);
        
        // Transfer the protocol fee from the SetToken to the protocol recipient.
        payProtocolFeeFromSetToken(setToken, address(_bidInfo.receiveToken), protocolFee);
        
        return protocolFee;
    }
```

## Impact
- Fees earned via rebalancing are able to be bypassed

## Code Snippet
https://github.com/sherlock-audit/2023-06-Index/blob/8d348ed344635a068d458aa04956f966b6d3d4f3/index-protocol/contracts/protocol/modules/v1/AuctionRebalanceModuleV1.sol#L963-L978

https://github.com/sherlock-audit/2023-06-Index/blob/8d348ed344635a068d458aa04956f966b6d3d4f3/index-protocol/contracts/protocol/modules/v1/BasicIssuanceModule.sol#L129-L163

## Tool used
Manual Review

## Recommendation
It is recommended to always lock the SetToken to avoid redeeming during rebalance operations, however, this will not mitigate the ability to frontrun a `startRebalance()` call. I recommend obfuscated `lock()` call prior to initiating the rebalance auction to ensure that funds are not redeemed with malicious intent.
