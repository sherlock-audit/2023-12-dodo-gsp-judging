Witty Cerulean Panther

high

# Buying shares can be frontrunned

## Summary
When users buy shares, they deposit tokens and receive a ratio of share tokens in exchange. However, the issue lies in users not being able to determine the minimum number of shares they are willing to receive. The share tokens to be minted are determined within the function execution. This creates vulnerability to front-running, where an unfortunate transaction could result in users receiving fewer LP tokens than they would typically get.
## Vulnerability Detail
When users deposits token this is how the shares to be minted is calculated:
```solidity
uint256 baseInputRatio = DecimalMath.divFloor(baseInput, baseReserve);
uint256 quoteInputRatio = DecimalMath.divFloor(quoteInput, quoteReserve);
uint256 mintRatio = quoteInputRatio < baseInputRatio ? quoteInputRatio : baseInputRatio;
// The shares will be minted to user
shares = DecimalMath.mulFloor(totalSupply, mintRatio);
```

It's evident that the `mintRatio` is determined as the lowest ratio between the divisions of baseInput and quoteInput by their respective reserves. Therefore, if a preceding transaction alters the ratios, the number of shares to be minted can change.

For example, consider a pool initialized with 100-100 tokens. If someone intends to deposit 80-50 tokens, normally it would result in:
min(80/100, 50/100) = 50/100
Hence, the mintRatio is calculated as 50/100, resulting in the user receiving shares equivalent to totalSupply * 50/100. However, if another user deposits 10-20 just before the initial user, the ratios will change due to the updated reserves. Consequently, the initial user will receive fewer shares despite depositing the same amount of tokens.
Here a coded PoC illustrates the above scenario:
```solidity
// @review this is how much hippo would get if tapir wouldn't frontrun his tx
    function test_normalCondition() external {
        vm.startPrank(tapir);
        dai.safeTransfer(address(gsp), 100 * 1e18);
        usdc.transfer(address(gsp), 100 * 1e6);
        gsp.buyShares(tapir);
        vm.stopPrank();

        vm.startPrank(hippo);
        dai.safeTransfer(address(gsp), 80 * 1e18);
        usdc.transfer(address(gsp), 50 * 1e6);
        gsp.buyShares(hippo);

        // hippo wants to deposits properly, but it will fail because of the minimum mint value!
        console.log("Hippos shares", gsp.balanceOf(hippo));
    }

    // @review this is how much hippo gets with the same amount but this time he get frontrunned
    function test_FrontrunMintingShares() external {
        // tapir deposits 100-100 to initiate the pool
        vm.startPrank(tapir);
        dai.safeTransfer(address(gsp), 100 * 1e18);
        usdc.transfer(address(gsp), 100 * 1e6);
        gsp.buyShares(tapir);

        // tapir then frontruns hippo and adds 10-20
        dai.safeTransfer(address(gsp), 10 * 1e18);
        usdc.transfer(address(gsp), 20 * 1e6);
        gsp.buyShares(tapir);
        vm.stopPrank();

        vm.startPrank(hippo);
        dai.safeTransfer(address(gsp), 80 * 1e18);
        usdc.transfer(address(gsp), 50 * 1e6);
        gsp.buyShares(hippo);

        // hippo wants to deposits properly, but it will fail because of the minimum mint value!
        console.log("Hippos shares", gsp.balanceOf(hippo));
    }
```

Test result and Logs:
<img width="484" alt="image" src="https://github.com/sherlock-audit/2023-12-dodo-gsp-mstpr/assets/120012681/220a26dc-afb3-41d5-ba99-10f9a97004ea">

## Impact
High since every user can get easily frontrunned or simply get unfortunate tx'ed. 
## Code Snippet
https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPFunding.sol#L31-L82
## Tool used

Manual Review

## Recommendation
Add an "minimumAmountShares" variable to buyShares so that the users can determine their minimum LP in return of their liquidity provision.