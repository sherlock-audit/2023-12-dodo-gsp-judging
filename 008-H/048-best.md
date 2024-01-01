Witty Cerulean Panther

high

# First depositor can lock the quote target value to zero

## Summary
When the initial deposit occurs, it is possible for the quote target to be set to 0. This situation significantly impacts other LPs as well. Even if subsequent LPs deposit substantial amounts, the quote target remains at 0 due to multiplication with this zero value. 0 _QUOTE_TARGET_ value will impact the swaps that pool facilities
## Vulnerability Detail
When the first deposit happens, _QUOTE_TARGET_ is set as follows:
```solidity
 if (totalSupply == 0) {
            // case 1. initial supply
            // The shares will be minted to user
            shares = quoteBalance < DecimalMath.mulFloor(baseBalance, _I_)
                ? DecimalMath.divFloor(quoteBalance, _I_)
                : baseBalance;
            // The target will be updated
            _BASE_TARGET_ = uint112(shares);
            _QUOTE_TARGET_ = uint112(DecimalMath.mulFloor(shares, _I_));
```

In this scenario, the 'shares' value can be a minimum of 1e3, as indicated here: [link to code snippet](https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPVault.sol#L295).

This implies that if someone deposits minuscule amounts of quote token and base token, they can set the _QUOTE_TARGET_ to zero because the `mulFloor` operation uses a scaling factor of 1e18:

```solidity
function mulFloor(uint256 target, uint256 d) internal pure returns (uint256) {
        return target * d / (10 ** 18);
    }
```

Should the quote target become 0, subsequent deposits will not increase due to the multiplication with "0" on the quote target. This situation is highly problematic because the swaps depend on the value of the quote target:
https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPFunding.sol#L74-L75
```solidity
// @review 0 + (0 * something) = 0! doesn't matter what amount has been deposited !
_QUOTE_TARGET_ = uint112(uint256(_QUOTE_TARGET_) + (DecimalMath.mulFloor(uint256(_QUOTE_TARGET_), mintRatio)));
```

Here a PoC shows that if the first deposit is tiny the _QUOTE_TARGET_ is 0. Also, whatever deposits after goes through the _QUOTE_TARGET_ still 0 because of the multiplication with 0! 

```solidity
function test_StartWithZeroTarget() external {
        // tapir deposits tiny amounts to make quote target 0
        vm.startPrank(tapir);
        dai.safeTransfer(address(gsp), 1 * 1e5);
        usdc.transfer(address(gsp), 1 * 1e5);
        gsp.buyShares(tapir);

        console.log("Base target", gsp._BASE_TARGET_());
        console.log("Quote target", gsp._QUOTE_TARGET_());
        console.log("Base reserve", gsp._BASE_RESERVE_());
        console.log("Quote reserve", gsp._QUOTE_RESERVE_());

        // quote target is indeed 0!
        assertEq(gsp._QUOTE_TARGET_(), 0);

        vm.stopPrank();

        // hippo deposits properly
        vm.startPrank(hippo);
        dai.safeTransfer(address(gsp), 1000 * 1e18);
        usdc.transfer(address(gsp), 10000 * 1e6);
        gsp.buyShares(hippo);

        console.log("Base target", gsp._BASE_TARGET_());
        console.log("Quote target", gsp._QUOTE_TARGET_());
        console.log("Base reserve", gsp._BASE_RESERVE_());
        console.log("Quote reserve", gsp._QUOTE_RESERVE_());

        // although hippo deposited 1000 USDC as quote tokens, target is still 0 due to multiplication with 0
        assertEq(gsp._QUOTE_TARGET_(), 0);
    }
```

Test result and logs:
<img width="479" alt="image" src="https://github.com/sherlock-audit/2023-12-dodo-gsp-mstpr/assets/120012681/762c190b-f35c-4d2a-9bf2-9654b94dd119">

## Impact
Since the quote target is important and used when pool deciding the swap math I will label this as high.
## Code Snippet
https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPFunding.sol#L31-L82
## Tool used

Manual Review

## Recommendation
According to the quote tokens decimals, multiply the quote token balance with the proper decimal scalor.