Flaky Sapphire Elk

medium

# Pool balancing swappers aren't attracted due to unupdated target state after sync action

## Summary
More pool balancers could have been attracted whenever there's a reserve change. But DODO v3 GSP doesn't update on certain actions.

## Vulnerability Detail

- Whenever  there's a change in the reserves of a pool, the base or quote targets are updated.
- But targets are not updated after [GSPVault.sync()](https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPVault.sol#L135)
- This should increase / decrease the targerts based on the `R state` and reserves.
- It should also update on `sellBase` and `sellQuote` actions meaning the `_BASE_TARGET_` is updated after `sellBase` action, but what if quote tokens are sent and sellBase action is called. Now reserves are changed, but the targets aren't. So reduction in attraction of pool balancers(arbitragers).
- If the targets are updated, we could have faster pool balancing.


## Impact
Missing out the involvement of pool balancers due to unUpdated target states.

## Code Snippet

```solidity
    function _sync() internal {
        uint256 baseBalance = _BASE_TOKEN_.balanceOf(address(this)) - uint256(_MT_FEE_BASE_);
        uint256 quoteBalance = _QUOTE_TOKEN_.balanceOf(address(this)) - uint256(_MT_FEE_QUOTE_);
        require(baseBalance <= type(uint112).max && quoteBalance <= type(uint112).max, "OVERFLOW");
        if (baseBalance != _BASE_RESERVE_) {
            _BASE_RESERVE_ = uint112(baseBalance);
        }
        if (quoteBalance != _QUOTE_RESERVE_) {
            _QUOTE_RESERVE_ = uint112(quoteBalance);
        }
        if (_IS_OPEN_TWAP_) _twapUpdate();
    }

    function sync() external nonReentrant {
        _sync();
    }
```

## Tool used

Manual Review

## Recommendation
- Add target updating code on `sync` function.
- And also add if anyone transfers this way ( transferring base token and then calling sellQuote ( 0 in, 0 out) but reserves changed)