n33k

medium

# Vault: `_update_debt` does not accrue interest

## Summary

`_update_debt` call `_debt_interest_since_last_update` to accrue interest but `_debt_interest_since_last_update` always return 0 in `_update_debt`.

## Vulnerability Detail

`_update_debt` sets `self.last_debt_update[_debt_token]` to `block.timestamp` and then calls `_debt_interest_since_last_update`.

```python
def _update_debt(_debt_token: address):
    ....
    self.last_debt_update[_debt_token] = block.timestamp
    ....
    self.total_debt_amount[_debt_token] += self._debt_interest_since_last_update(
        _debt_token
    )
```

`_debt_interest_since_last_update` always returns 0. because `block.timestamp - self.last_debt_update[_debt_token]` is always 0.

```python
def _debt_interest_since_last_update(_debt_token: address) -> uint256:
    return (
        (block.timestamp - self.last_debt_update[_debt_token])
        * self._current_interest_per_second(_debt_token)
        * self.total_debt_amount[_debt_token]
        / PERCENTAGE_BASE
        / PRECISION
    )
```

## Impact

Debt fee is not accrued.

## Code Snippet

https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/margin-dex/Vault.vy#L1050-L1076

## Tool used

Manual Review

## Recommendation

Call `_debt_interest_since_last_update` then update `last_debt_update`.