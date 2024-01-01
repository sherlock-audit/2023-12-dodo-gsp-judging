Witty Cerulean Panther

medium

# Adjusting "_I_" requires a twap update because it changes the mid price immediately

## Summary
Whenever a price-changing event occurs, such as adding liquidity (buyShares) or selling quote/base tokens, the TWAP is updated if the TWAP update flag is set to true. However, there is another scenario where the price change occurs, which is the alteration of "I" through the adjustPrice function. Presently, the code does not update the TWAP price in this case. Consequently, anything that relies on the TWAP might receive inaccurate pricing due to the stale price update. 
## Vulnerability Detail
As stated in the summary, any price changing event should update the twap. Since "_I_" variable is directly responsible for determining the price in the equation it will change the price. Here an example that demonstrates the price difference:

```solidity
function test_Adjusting_I_RequiresTwapUpdate() external {
        // @review mid price changes when "_I_" changes and since the price changes
        // a twap update is necessary!
        vm.startPrank(tapir);

        //  Buy shares with tapir, 10 - 10
        dai.safeTransfer(address(gsp), 10 * 1e18);
        usdc.transfer(address(gsp), 10 * 1e6);
        gsp.buyShares(tapir);

        // print some stuff
        console.log("Base target", gsp._BASE_TARGET_());
        console.log("Quote target", gsp._QUOTE_TARGET_());
        console.log("Base reserve", gsp._BASE_RESERVE_());
        console.log("Quote reserve", gsp._QUOTE_RESERVE_());

        // check the mid price
        uint256 midPrice = gsp.getMidPrice();
        console.log("Mid price", midPrice);
        vm.stopPrank();

        // update the "_I_" to a reasonable value, like 0.99
        vm.prank(MAINTAINER);
        gsp.adjustPrice(999000);

        // check the mid price one more time after the "_I_" update
        uint256 after_I_MidPrice = gsp.getMidPrice();
        console.log("Mid price", after_I_MidPrice);

        // mid price has changed! 
        assertTrue(after_I_MidPrice != midPrice);
    }
```

Running the above test will prove that the price changed after the "_I_" update via `adjustPrice`!
## Impact
Assuming the pool interactions are frequent this should be validated as a medium.
## Code Snippet
https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPVault.sol#L169-L174
## Tool used

Manual Review

## Recommendation
Change the function to this:
```solidity
function adjustPrice(uint256 i) external onlyMaintainer {
        uint256 offset = i > _I_ ? i - _I_ : _I_ - i;
        require((offset * 1e6 / _I_) <= _PRICE_LIMIT_, "EXCEED_PRICE_LIMIT");
        _I_ = i;
        if (_IS_OPEN_TWAP_) _twapUpdate(); // @review add this line!
    }
```