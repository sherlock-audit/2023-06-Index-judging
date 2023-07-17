Hot Inky Sidewinder

medium

# Exponential and logarithmic argument overflow protection checks are wrong.

## Summary

The exponential and logarithmic argument overflow checks that the timeBucket is greater than the max uint256 divided by timeCoefficient when it should check that its greater than max int256, leading to potential int256 overflows.

## Vulnerability Detail
Exponential Argument Overflow:
`
        if (timeBucket > type(uint256).max / timeCoefficient) {
            return _getBoundaryPrice(isDecreasing, maxPrice, minPrice);
        }
        int256 expArgument = int256(timeCoefficient * timeBucket);
        
        // Protect against exponential overflow and increasing relative error
        if (expArgument > MAX_EXP_ARG) {
            return _getBoundaryPrice(isDecreasing, maxPrice, minPrice);
        }
        uint256 expExpression = uint256(FixedPointMathLib.expWad(expArgument));

        // Protect against priceChange overflow
        if (scalingFactor > type(uint256).max / expExpression) {
`
Whenever timeBucket > type(int256).max / timeCoefficient and timeBucket <= type(uint256).max / timeCoefficient , 
the unsafe int256 casting that happens at line [61](https://github.com/IndexCoop/index-protocol/blob/663e64efaa95df2247afa8926d4cfb42948f54fe/contracts/protocol/integration/auction-price/BoundedStepwiseExponentialPriceAdapter.sol#L61) will result in an overflow of expArgument. This value will always be a large negative value less than [-42139678854452767551](https://github.com/Vectorized/solady/blob/7175c21f95255dc7711ce84cc32080a41864abd6/src/utils/FixedPointMathLib.sol#L125). As a result the expExpression will return 0, leading to a divide by zero EVM revert since the priceChange overflow check puts `expExpression` in the denominator. 

Logarithmic Argument Overflow:

`
        if (timeBucket > type(uint256).max / timeCoefficient) {
            return _getBoundaryPrice(isDecreasing, maxPrice, minPrice);
        }
        int256 lnArgument = int256(timeBucket * timeCoefficient);
        
        // Protect against logarithmic overflow and increasing relative error
        if (lnArgument > MAX_LOG_ARG) {
            return _getBoundaryPrice(isDecreasing, maxPrice, minPrice);
        }
        uint256 lnExpression = uint256(FixedPointMathLib.lnWad(lnArgument + WAD));
 `
 Similar issue as the exponential argument overflow. Except this time the `FixedPointMathLib.lnWad(lnArgument + WAD)` function will revert since [`lnArgument + WAD`](https://github.com/IndexCoop/index-protocol/blob/663e64efaa95df2247afa8926d4cfb42948f54fe/contracts/protocol/integration/auction-price/BoundedStepwiseLogarithmicPriceAdapter.sol#L67) will always be negative in the case of an overflowing lnArgument value. 

## Impact

Since the expExpression will return zero in the case outlined above, the check for priceChange overflow will result in a divide by zero EVM revert every time. This will cause all bids on the associated component to revert for the remainder of the rebalance duration or until enough time has passed so that timeBucket > type(uint256).max / timeCoefficient. 

## Code Snippet

https://github.com/IndexCoop/index-protocol/blob/663e64efaa95df2247afa8926d4cfb42948f54fe/contracts/protocol/integration/auction-price/BoundedStepwiseExponentialPriceAdapter.sol#L58-L70

## Tool used

Manual Review

## Recommendation

Use type(int256).max instead of type(uint256).max for the exponential argument overflow protection check. 

