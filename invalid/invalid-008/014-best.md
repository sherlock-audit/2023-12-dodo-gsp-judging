Huge Magenta Tardigrade

medium

# `GSPVault.sol`

## Summary

[`GSP.sol`](https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSP.sol) continues to accrue fees even if there is no [`_MAINTAINER_`](https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPStorage.sol#L28) address which can collect them.

## Vulnerability Detail

[`GSP.sol`](https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSP.sol) permits initializers to define optional addresses for maintainers and maintenance fees.

When calling [`init()`](https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSP.sol#L34) on a [`GSP`](https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSP.sol), construction parameters are not validated to detect the erroneous combination of non-zero [`_MT_FEE_RATE_`](https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSP.sol#L64C9-L64C22) in combination with a zero address.

This scenario leads to the collection of fees with no possible way to withdraw them.

> [!TIP]
> In [**Dodo V2**](https://github.com/DODOEX/contractV2/), the absence of an implementation contract address on an `IFeeReceiver` [would result in zero fees](https://github.com/DODOEX/contractV2/blob/14efeca42bcddfe8d7ac2879c56a60f2b43c392f/contracts/lib/FeeRateModel.sol#L29).
>
> Similarly, the absence of a `feeAddr` on a [`FeeRateDIP3Impl`](https://etherscan.io/address/0x2c32DFC4df92DF02AE9d9ad0750A3F209ddCA61A#code) would also result in fees being disabled, even in the presence of non-zero fee rates.
>
> A core deliverable of the gas savings optimisations provided by the GSP relates encapsulation of the originally decoupled fee tracking logic originally handled by an `IFeeReceiver`. Consequently, failure to handle the zero address should be considered a **regression**.

## Impact

Maintenance fees are permanently locked inside the vault, leading to suboptimal economic outcomes for traders and an avoidable loss of yield.

## Code Snippet

```solidity
/**
 * @notice Function will be called in factory, init risk should not be included.
 * @param maintainer The dodo's address, who can claim mtFee and own this pool
 * @param baseTokenAddress The base token address
 * @param quoteTokenAddress The quote token address
 * @param lpFeeRate The rate of lp fee, with 18 decimal
 * @param mtFeeRate The rate of mt fee, with 18 decimal
 * @param i The oracle price, possible to be changed only by maintainer
 * @param k The swap curve parameter
 * @param isOpenTWAP Use TWAP price or not
 */
function init(
    address maintainer,
    address baseTokenAddress,
    address quoteTokenAddress,
    uint256 lpFeeRate,
    uint256 mtFeeRate,
    uint256 i,
    uint256 k,
    bool isOpenTWAP
) external {
    // GSP can only be initialized once
    require(!_GSP_INITIALIZED_, "GSP_INITIALIZED");
    // _GSP_INITIALIZED_ is set to true after initialization
    _GSP_INITIALIZED_ = true;
    // baseTokenAddress and quoteTokenAddress should not be the same
    require(baseTokenAddress != quoteTokenAddress, "BASE_QUOTE_CAN_NOT_BE_SAME");
    // _BASE_TOKEN_ and _QUOTE_TOKEN_ should be valid ERC20 tokens
    _BASE_TOKEN_ = IERC20(baseTokenAddress);
    _QUOTE_TOKEN_ = IERC20(quoteTokenAddress);

    // i should be greater than 0 and less than 10**36
    require(i > 0 && i <= 10**36);
    _I_ = i;
    // k should be greater than 0 and less than 10**18
    require(k <= 10**18);
    _K_ = k;

    // _LP_FEE_RATE_ is set when initialization
    _LP_FEE_RATE_ = lpFeeRate;
    // _MT_FEE_RATE_ is set when initialization
    _MT_FEE_RATE_ = mtFeeRate;
    // _MAINTAINER_ is set when initialization, the address receives the fee
    _MAINTAINER_ = maintainer;
    _IS_OPEN_TWAP_ = isOpenTWAP;
    // if _IS_OPEN_TWAP_ is true, _BLOCK_TIMESTAMP_LAST_ is set to the current block timestamp
    if (isOpenTWAP) _BLOCK_TIMESTAMP_LAST_ = uint32(block.timestamp % 2**32);

    string memory connect = "_";
    string memory suffix = "GSP";
    // name of the shares is the combination of suffix, connect and string of the GSP
    name = string(abi.encodePacked(suffix, connect, addressToShortString(address(this))));
    // symbol of the shares is GLP
    symbol = "GLP";
    // decimals of the shares is the same as the base token decimals
    decimals = IERC20Metadata(baseTokenAddress).decimals();

    // ============================== Permit ====================================
    uint256 chainId;
    assembly {
        chainId := chainid()
    }
    // DOMAIN_SEPARATOR is used for approve by signature
    DOMAIN_SEPARATOR = keccak256(
        abi.encode(
            // keccak256('EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)'),
            0x8b73c3c69bb8fe3d512ecc4cf759cc79239f7b179b0ffacaa9a75d522b39400f,
            keccak256(bytes(name)),
            keccak256(bytes("1")),
            chainId,
            address(this)
        )
    );
    // ==========================================================================
}
```

## Tool used

Visual Studio Code

## Recommendation

There are four possible solutions:

1. Attempts to [`init()`](https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSP.sol#L34) a [`GSP`](https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSP.sol) with non-zero fees and a zero-address maintainer should `revert`.
2. In the instance of a zero-address [`_MAINTAINER_`](https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPStorage.sol#L28) and non-zero maintenance fees, a protocol-defined default address should be fallen back to.
3. For a zero address [`_MAINTAINER_`](https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPStorage.sol#L28), maintenance fees should be disabled.
4. Require a [`_MAINTAINER_`](https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPStorage.sol#L28) to be defined.