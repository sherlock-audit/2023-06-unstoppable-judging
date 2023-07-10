n33k

high

# Vault: The attacker can sandwich attack himself on swaps in open_position, close_position and reduce_position to make a bad debt

## Summary

The attacker could call MarginDex to open/close postions in Vault. The swap slippage is controlled by the attacker. He can set the slippage to infinite to steal from swaps and leave a bad postion/debt.

## Vulnerability Detail

In open_position of Vault, max_leverage is first checked to ensure the position that is about to open will be healthy. Then token is borrowed and swapped into postion token. The attacker can set slippage to infinite and sandwich attack the swap to steal almost all of the swapped token. Then the amount swapped out is recorded in postion and the postion will be undercollateralized.

Swaps in close_position and reduce_position are also vulnerable to this sandwich attack. The attacker can attack to steal from swap and make a bad debt.

## Impact

The protocol could be drained because of this sandwich attack.

## Code Snippet

https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/margin-dex/Vault.vy#L182-L184

https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/margin-dex/Vault.vy#L252-L257

## Tool used

Manual Review

## Recommendation

Call `_is_liquidatable` in the end of open_position and reduce_position.

Check pool price deviation to oracle price inside close_position.