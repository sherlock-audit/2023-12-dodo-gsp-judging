Modern Citron Ant

high

# Access Control Vulnerability in Dodo Decentralized Exchange Smart Contract

## Summary
The Dodo decentralized exchange smart contract contains a critical access control vulnerability that allows any address to call the init function, potentially leading to unintended and unauthorized initialization of the contract. The vulnerability arises from a lack of access control checks in the initialization process.

## Vulnerability Detail
The primary issue lies in the init function, which is intended to be called by a factory contract during the initialization process. However, the current implementation does not enforce any access control, allowing any address to call this function and initialize the contract. This oversight poses a serious security risk, as it could lead to unauthorized parties modifying critical parameters of the contract.

## Impact
The impact of this vulnerability is significant, as it enables malicious actors to interfere with the initialization process of the Dodo decentralized exchange smart contract. Unauthorized calls to the init function may result in unintended changes to parameters such as token addresses, fee rates, and other critical configurations. This could compromise the integrity and security of the decentralized exchange and will lead to financial damage.


## Code Snippet

 https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/main/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSP.sol#L43

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
    ...
    }

## Tool used
Manual Review

## Recommendation
add onlyOwner modifier to restrict access