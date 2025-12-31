# Unstoppable Vault - Vulnerability Analysis & Fix

## Executive Summary

The `UnstoppableVault` contract contains a critical vulnerability that allows anyone with even 1 wei of the vault's token to permanently disable all flash loan functionality. This document explains the vulnerability, provides a fixed implementation, and demonstrates the exploit.

## The Vulnerability

### Location
`UnstoppableVault.sol:85`

```solidity
uint256 balanceBefore = totalAssets();
if (convertToShares(totalSupply) != balanceBefore) revert InvalidBalance();
```

### Root Cause

The contract attempts to enforce an ERC4626 invariant by checking that `convertToShares(totalSupply)` equals `totalAssets()`. However, this check is fundamentally flawed:

1. **Wrong conversion function**: The code uses `convertToShares(totalSupply)` which calculates:
   ```
   totalSupply * totalSupply / totalAssets()
   ```
   This only equals `totalAssets()` when `totalSupply == totalAssets()` (1:1 ratio).

2. **Vulnerable to direct transfers**: `totalAssets()` returns `balanceOf(address(this))`, which can be increased by anyone via direct token transfers, without minting shares.

### Attack Vector

```solidity
// Attacker with ANY amount of tokens (even 1 wei):
token.transfer(address(vault), amount);
```

This simple transfer:
- ✅ Increases `totalAssets()` (the vault's token balance)
- ❌ Does NOT increase `totalSupply` (no shares minted)
- 💥 Breaks the invariant permanently

## The Impact

- **Severity**: Critical
- **Cost to exploit**: Minimal (even 1 wei)
- **Reversibility**: Irreversible without privileged access
- **Business impact**: Complete DoS of flash loan functionality

## The Exploit

See `src/unstoppable/UnstoppableExploit.sol`:

```solidity
contract UnstoppableExploit {
    function attack(uint256 amount) external {
        // Transfer tokens from attacker
        token.transferFrom(msg.sender, address(this), amount);

        // Send directly to vault (bypassing deposit mechanism)
        token.transfer(address(vault), amount);

        // All flash loans now revert with InvalidBalance()
    }
}
```

## The Fix

See `src/unstoppable/UnstoppableVaultFixed.sol`

### Strategy: Internal Accounting

Instead of relying on `balanceOf(address(this))`, track deposited assets internally:

```solidity
contract UnstoppableVaultFixed is ERC4626 {
    // Track assets internally
    uint256 private _totalManagedAssets;

    function totalAssets() public view override returns (uint256) {
        return _totalManagedAssets;  // ← Use internal accounting
    }

    function afterDeposit(uint256 assets, uint256 shares) internal override {
        _totalManagedAssets += assets;  // ← Update on deposit
    }

    function beforeWithdraw(uint256 assets, uint256 shares) internal override {
        _totalManagedAssets -= assets;  // ← Update on withdrawal
    }

    function flashLoan(...) external returns (bool) {
        // ...
        _totalManagedAssets += fee;  // ← Track fees
        // ...
    }
}
```

### Why This Works

- Direct token transfers increase `balanceOf(vault)` but NOT `_totalManagedAssets`
- The invariant check now uses internal accounting that can't be manipulated externally
- Extra tokens sent to the vault are simply ignored (bonus for vault shareholders)
- The correct invariant is now: `convertToAssets(totalSupply) == _totalManagedAssets`

## Test Results

Run: `forge test --match-contract UnstoppableExploitTest -vv`

### Vulnerable Vault
```
Before attack:
  Total assets: 1000000000000000000000000
  Total supply: 1000000000000000000000000
  Token balance: 1000000000000000000000000

After attack (sent 10 tokens):
  Total assets: 1000010000000000000000000
  Total supply: 1000000000000000000000000
  Mismatch: 19999900000999990001

Result: ❌ All flash loans revert with InvalidBalance()
```

### Fixed Vault
```
Before attack:
  Total assets (internal): 1000000000000000000000000
  Total supply: 1000000000000000000000000
  Token balance: 1000000000000000000000000

After attack (sent 10 tokens):
  Total assets (internal): 1000000000000000000000000
  Total supply: 1000000000000000000000000
  Token balance: 1000010000000000000000000
  Extra tokens ignored: 10000000000000000000

Result: ✅ Flash loans continue to work normally
```

## Alternative Fixes

### Option 1: Remove the check entirely
```solidity
// Just remove line 85
// Simpler, but loses the safety check
```

### Option 2: Use >= instead of ==
```solidity
if (totalAssets() < convertToAssets(totalSupply)) revert InvalidBalance();
// Allows extra tokens, prevents underfunding
```

### Option 3: Fix the conversion (still vulnerable)
```solidity
if (convertToAssets(totalSupply) != balanceBefore) revert InvalidBalance();
// Correct math, but still vulnerable to direct transfers
```

**Recommendation**: Use internal accounting (implemented in `UnstoppableVaultFixed.sol`)

## Lessons Learned

1. **Never rely on `balanceOf()` for critical invariants** - Anyone can send tokens to any address
2. **Understand ERC4626 deeply** - `convertToShares` vs `convertToAssets` matter
3. **Test boundary conditions** - What happens with direct transfers?
4. **Use internal accounting** - Track what you control, not what others can manipulate

## Files

- **Vulnerability**: `src/unstoppable/UnstoppableVault.sol`
- **Fixed Version**: `src/unstoppable/UnstoppableVaultFixed.sol`
- **Exploit**: `src/unstoppable/UnstoppableExploit.sol`
- **Tests**: `test/unstoppable/UnstoppableExploit.t.sol`

## References

- [ERC4626 Specification](https://eips.ethereum.org/EIPS/eip-4626)
- [ERC3156 Flash Loans](https://eips.ethereum.org/EIPS/eip-3156)
- [Solmate ERC4626 Implementation](https://github.com/transmissions11/solmate/blob/main/src/tokens/ERC4626.sol)
