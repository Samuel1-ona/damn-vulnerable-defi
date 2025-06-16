# Free Rider

A new marketplace of Damn Valuable NFTs has been released! There's been an initial mint of 6 NFTs, which are available for sale in the marketplace. Each one at 15 ETH.

A critical vulnerability has been reported, claiming that all tokens can be taken. Yet the developers don't know how to save them!

They're offering a bounty of 45 ETH for whoever is willing to take the NFTs out and send them their way. The recovery process is managed by a dedicated smart contract.

You've agreed to help. Although, you only have 0.1 ETH in balance. The devs just won't reply to your messages asking for more.

If only you could get free ETH, at least for an instant.

## Vulnerability Analysis

The vulnerability exists in the `FreeRiderNFTMarketplace` contract, specifically in the `_buyOne` function. Here's the vulnerable code:

```solidity
function _buyOne(uint256 tokenId) private {
    uint256 priceToPay = offers[tokenId];
    if (priceToPay == 0) {
        revert TokenNotOffered(tokenId);
    }

    if (msg.value < priceToPay) {
        revert InsufficientPayment();
    }

    --offersCount;

    // transfer from seller to buyer
    DamnValuableNFT _token = token; // cache for gas savings
    _token.safeTransferFrom(_token.ownerOf(tokenId), msg.sender, tokenId);

    // pay seller using cached token
    payable(_token.ownerOf(tokenId)).sendValue(priceToPay);

    emit NFTBought(msg.sender, tokenId, priceToPay);
}
```

The vulnerability stems from the order of operations and state changes:

1. The function first transfers the NFT from seller to buyer
2. Then it attempts to pay the seller using `_token.ownerOf(tokenId)`
3. However, after the transfer, `ownerOf(tokenId)` returns the buyer's address instead of the seller's
4. This means the buyer effectively pays themselves instead of the seller

## Exploit Strategy

The exploit takes advantage of this vulnerability in the following way:

1. Get a flash loan of 15 ETH from Uniswap V2
2. Use this ETH to buy the first NFT
3. Due to the vulnerability, we receive the 15 ETH back to ourselves
4. Reuse the same 15 ETH to buy the next NFT
5. Repeat for all 6 NFTs
6. Send all NFTs to the recovery manager to claim the 45 ETH bounty
7. Use some of the bounty to repay the flash loan
8. Keep the remaining ETH as profit

The key components of the exploit are:
- Using Uniswap V2 for the flash loan
- The marketplace vulnerability that allows us to get our payment back
- The recovery manager's bounty system

## Impact

This vulnerability allows an attacker to:
1. Acquire all NFTs without actually paying the sellers
2. Claim the full bounty from the recovery manager
3. Make a significant profit (approximately 45 ETH - flash loan fees)
4. All of this can be done starting with just 0.1 ETH

## Prevention

To prevent this vulnerability, the contract should:
1. Cache the seller's address before the NFT transfer
2. Use the cached address for payment
3. Or reverse the order of operations (pay first, then transfer)

Example fix:
```solidity
function _buyOne(uint256 tokenId) private {
    uint256 priceToPay = offers[tokenId];
    if (priceToPay == 0) {
        revert TokenNotOffered(tokenId);
    }

    if (msg.value < priceToPay) {
        revert InsufficientPayment();
    }

    --offersCount;

    // Cache seller address before transfer
    address seller = _token.ownerOf(tokenId);
    
    // transfer from seller to buyer
    _token.safeTransferFrom(seller, msg.sender, tokenId);

    // pay seller using cached address
    payable(seller).sendValue(priceToPay);

    emit NFTBought(msg.sender, tokenId, priceToPay);
}
```
