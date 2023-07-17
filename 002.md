Proud Sepia Rattlesnake

medium

# Change in components or positionMultiplier can cause the user to lose funds

## Summary
A sudden change in `components` (add or remove), `positionMultiplier` or  `positionUnit` can cause the user to loose funds or execute bad trades.

## Vulnerability Detail
In [`BasicIssuanceModule`](https://github.com/sherlock-audit/2023-06-Index/blob/main/index-protocol/contracts/protocol/modules/v1/BasicIssuanceModule.sol) there are 2 functions that are gonna be called by users wanting to participate in a given setToken. [`issue`](https://github.com/sherlock-audit/2023-06-Index/blob/main/index-protocol/contracts/protocol/modules/v1/BasicIssuanceModule.sol#L86-L119) && [`redeem`](https://github.com/sherlock-audit/2023-06-Index/blob/main/index-protocol/contracts/protocol/modules/v1/BasicIssuanceModule.sol#L129-L163), which work fine, exept for the fact that there are no measures taken if any changes to the setToken are made during or before a trade. 

**Example:** 
- **1** User calls [`issue`](https://github.com/sherlock-audit/2023-06-Index/blob/main/index-protocol/contracts/protocol/modules/v1/BasicIssuanceModule.sol#L86-L119) on setToken (with 50% BTC 50% ETH, with 1k usd in BTC and 1k usd in ETH), but due to congestion in the network or just high gas prices the trade remains in the mem-pool.
- **2** Owner decides to call   [`editDefaultPositionUnit`](https://github.com/sherlock-audit/2023-06-Index/blob/main/index-protocol/contracts/protocol/SetToken.sol#L238-L244), which changes the % of each asset, for example making it 90% BTC 10% ETH
- **3** The owner call gets executed first and the user's second, now the user has a setToken for which he payed 1800 usd in BTC and 200 usd in ETH.

**There are 2 more similar scenarios involving:**
- [`addComponent`](https://github.com/sherlock-audit/2023-06-Index/blob/main/index-protocol/contracts/protocol/SetToken.sol#L217-L223) - where the user unwantingly spends a different ERC20 token
- [`editPositionMultiplier`](https://github.com/sherlock-audit/2023-06-Index/blob/main/index-protocol/contracts/protocol/SetToken.sol#L317-L323) - where the user gets minted less setTokens, because of how [`getRequiredComponentUnitsForIssue`](https://github.com/sherlock-audit/2023-06-Index/blob/main/index-protocol/contracts/protocol/modules/v1/BasicIssuanceModule.sol#L203-L223) is calculated

## Impact
User enters bad trades.

## Code Snippet
[BasicIssuanceModule.sol/L86-L119](https://github.com/sherlock-audit/2023-06-Index/blob/main/index-protocol/contracts/protocol/modules/v1/BasicIssuanceModule.sol#L86-L119)
## Tool used

Manual Review

## Recommendation
This vulnerability has slippage like roots, tho it differs a little bit. To fix it you can add an array of tokens for max input and a `uint` for min setTokens received. 
 ```jsx
    function issue(
        ISetToken _setToken,
        uint256 _quantity,
        address _to,
+      uint minSetTokens,
+      uint[] calldata maxComponentAmounts
        )
```