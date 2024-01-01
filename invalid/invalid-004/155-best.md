Ambitious Ruby Blackbird

high

# DoS due to unexpected revert in twap update

## Summary
Due to an underflow error at the `GSPVault._twapUpdate` function the protocol functionality will be blocked. So users will not be able to redeem shares.

## Vulnerability Detail
There is a normalization of the `block.timestamp` value to `uint32` at the `GSPVault._twapUpdate` function which works well in solidity 0.6.9 but in recent versions will throw an underflow error.
https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPVault.sol#L89-L90
In case the `_IS_OPEN_TWAP_` parameter will be initialized as `true` this issue can cause a permanent asset blocking at the pool when `block.timestamp` exceeds `type(uint32).max`.  

## Impact
Permanent asset blocking at the pool.

## Code Snippet
https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPVault.sol#L89-L90

## Tool used
Manual Review

## Recommendation
Consider using `unchecked` braces for this calculation:
```diff
    function _twapUpdate() internal {
        // blockTimestamp is the timestamp of the current block
        uint32 blockTimestamp = uint32(block.timestamp % 2**32);
        // timeElapsed is the time elapsed since the last update
+       unchecked {
            uint32 timeElapsed = blockTimestamp - _BLOCK_TIMESTAMP_LAST_;
+       }
        // if timeElapsed is greater than 0 and the reserves are not 0, update the twap price
        if (timeElapsed > 0 && _BASE_RESERVE_ != 0 && _QUOTE_RESERVE_ != 0) {
            _BASE_PRICE_CUMULATIVE_LAST_ += getMidPrice() * timeElapsed;
        }
        // update the last block timestamp
        _BLOCK_TIMESTAMP_LAST_ = blockTimestamp;
    }
```