Bitter White Ape

high

# TWAP Based Off First Transaction In A Block Makes Manipulation Risk Free

## Summary

Widely used TWAP Oracles such as Uniswap v3 only take data from the end of the last transaction in a block. This means that an attacker trying to manipulate it has to leave themselves open to large arbitrage losses because they have to leave the pool reserves imbalanced over a block. Dodo's TWAP instead bases the prices after the first transaction, so an attacker can manipulate it and then reset the price back to the orignal price giving them risk free manipulation that cannot be arbitraged.

## Vulnerability Detail

In the `sellBaseToken` and `sellQuoteToken` functions, `_twapUpdate` is called after the swap logic. When this is the first swap, liquidity or flash loan action of the block, the TWAP is updateed off the reserves after this swap.

The subsequent actions in a block do not update the TWAP, as the `timeElapsed` is `0`:

```solidity
        uint32 timeElapsed = blockTimestamp - _BLOCK_TIMESTAMP_LAST_;
        // if timeElapsed is greater than 0 and the reserves are not 0, update the twap price
        if (timeElapsed > 0 && _BASE_RESERVE_ != 0 && _QUOTE_RESERVE_ != 0) {
            _BASE_PRICE_CUMULATIVE_LAST_ += getMidPrice() * timeElapsed;
```

This is actually different from other TWAP designs such as Uniswap v3 TWAP. In those oracles, the price update is based on the price _at the end of a block_. This makes it resistant manipulation as somebody that wanted to set the TWAP result to an extreme value had to leave it there until the end of the block, in which case it can easily be arbitrage.

In DODO's case, the TWAP can be set to an extreme value simply by bunding these 2 transactions in a flashbots bundle so no arbitrage transactions can be inserted between:

1. making a large swap through the reserves. This changes the recorded `TWAP` price for the time range since the last update.
2. swap back to exactly the original price. This prevents any losses to arbitrage. Since `timeElapsed==0`, the TWAP is not changed by this action 

## Impact

TWAP can be manipulated easily and without risk.

## Code Snippet

https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/main/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPVault.sol#L86-L97

## Tool used

Manual Review

## Recommendation

Only take prices at the end of a past block. See Uniswap v3's TWAP implementation.