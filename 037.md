Flaky Mossy Wallaby

medium

# block.timestamp means different things on different L2s

## Summary

It is stated in the documents of the project that it will be published on the Arbitrum network.
```js

### Q: On what chains are the smart contracts going to be deployed?
Mainnet, Polygon, Optimism, Arbitrum, Avalanche

```

How do block.timestamp and block.number work on Arbitrum?

Solidity calls to block.number and block.timestamp on Arbitrum will return the block number/ timestamp of the underlying L1 on a slight delay; i.e., updated every few minutes. 


https://developer.arbitrum.io/for-devs/troubleshooting-building#how-do-blocktimestamp-and-blocknumber-work-on-arbitrum

## Vulnerability Detail

https://github.com/sherlock-audit/2023-06-Index/blob/main/index-protocol/contracts/protocol/modules/v1/AuctionRebalanceModuleV1.sol#L275

https://github.com/sherlock-audit/2023-06-Index/blob/main/index-protocol/contracts/protocol/modules/v1/AuctionRebalanceModuleV1.sol#L806

https://github.com/sherlock-audit/2023-06-Index/blob/main/index-protocol/contracts/protocol/modules/v1/AuctionRebalanceModuleV1.sol#L1190

## Impact
block.timestamp means different things on different L2s


## Code Snippet

```solidity
index-protocol/contracts/protocol/modules/v1/AuctionRebalanceModuleV1.sol:
   274          rebalanceInfo[_setToken].quoteAsset = _quoteAsset;
   275:         rebalanceInfo[_setToken].rebalanceStartTime = block.timestamp; // @audit-issue block.timestamp;
   276          rebalanceInfo[_setToken].rebalanceDuration = _rebalanceDuration;

   805              _componentQuantity,
   806:             block.timestamp.sub(rebalanceInfo[_setToken].rebalanceStartTime),
   807              rebalanceInfo[_setToken].rebalanceDuration,

  1189          RebalanceInfo storage rebalance = rebalanceInfo[_setToken];
  1190:         return (rebalance.rebalanceStartTime.add(rebalance.rebalanceDuration)) <= block.timestamp; // @audit-issue block.timestamp;
  1191      }
  ```

## Tool used

Manual Review

## Recommendation
Have the user pass a block.timestamp as the function argument