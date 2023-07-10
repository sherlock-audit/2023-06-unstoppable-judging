BugBusters

high

# Interested calculated is ampliefied by multiple of 1000 in `_debt_interest_since_last_update`

## Summary
Interest calculated in the `_debt_interest_since_last_update` function is amplified by multiple of 1000, hence can completely brick the system and debt calculation. Because we divide by PERCENTAGE_BASE instead of PERCENTAGE_BASE_HIGH which has more precision and which is used in utilization calculation.
## Vulnerability Detail
Following function calculated the interest accured over a certain interval :
```solidity
def _debt_interest_since_last_update(_debt_token: address) -> uint256:

    return (

        (block.timestamp - self.last_debt_update[_debt_token])* self._current_interest_per_second(_debt_token)
        * self.total_debt_amount[_debt_token]
        / PERCENTAGE_BASE 
        / PRECISION
    )
```

But the results from the above function are amplified by factor of 1000 due to the reason that the intrest per second as per test file is calculated as following:

```solidity
    # accordingly the current interest per year should be 3% so 3_00_000
    # per second that is (300000*10^18)/(365*24*60*60)
    expected_interest_per_second = 9512937595129375

    assert (
        expected_interest_per_second
        == vault_configured.internal._current_interest_per_second(usdc.address)
    )
```
So yearly interest has the precision of 5 as it is calculated using utilization rate and `PERCENTAGE_BASE_HIGH_PRECISION` is used which has precision of 5 .and per second has the precision of 18, so final value has the precision of 23.

Interest per second has precision = 23.

But if we look at the code:

```solidity
        (block.timestamp - self.last_debt_update[_debt_token])* self._current_interest_per_second(_debt_token)
        * self.total_debt_amount[_debt_token]
        / PERCENTAGE_BASE 
        / PRECISION
```
We divide by PERCENTAGE_BASE that is = 100_00 = precision of => 2
And than by PRECISION = 1e18 => precision of 18.  So accumulated precision of 20, where as we should have divided by value precises to 23 to match the nominator.

Where is we should have divided by PERCENTAGE_BASE_HIGH instead of PERCENTAGE_BASE

Hence the results are amplified by enormous multiple of thousand.
## Impact
Interest are too much amplified, that impacts the total debt calculation and brick whole leverage, liquidation and share mechanism.

Note: Dev confirmed that the values being used in the tests are the values that will be used in production.
## Code Snippet
https://github.com/sherlock-audit/2023-06-unstoppable/blob/94a68e49971bc6942c75da76720f7170d46c0150/unstoppable-dex-audit/contracts/margin-dex/Vault.vy#L1069-L1076
https://github.com/sherlock-audit/2023-06-unstoppable/blob/94a68e49971bc6942c75da76720f7170d46c0150/unstoppable-dex-audit/tests/vault/test_variable_interest_rate.py#L314-L333
## Tool used

Manual Review

## Recommendation
Use PERCENTAGE_BASE_HIGH in division instead of PERCENTAGE_BASE.