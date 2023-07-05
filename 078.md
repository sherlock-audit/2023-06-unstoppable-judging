shealtielanz

medium

# Did Not `Approve --->  Zero` First

## Summary
Some ERC20 tokens (like USDT) do not work when changing the allowance from an existing non-zero allowance value. For example, Tether (USDT)'s approve() function will revert if the current approval is not zero, to protect against front-running changes of approvals.

## Vulnerability Detail
The following attempt to call the approve() function without setting the allowance to zero first.
- https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/margin-dex/SwapRouter.vy#L68C1-L68C57
 ```vyper
def swap(
    _token_in: address,
    _token_out: address,
    _amount_in: uint256,
    _min_amount_out: uint256,
) -> uint256:
    ERC20(_token_in).transferFrom(msg.sender, self, _amount_in)
    ERC20(_token_in).approve(UNISWAP_ROUTER, _amount_in)
```
- https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/margin-dex/Vault.vy#L384C1-L395C59
```vyper
def _swap(
    _token_in: address,
    _token_out: address,
    _amount_in: uint256,
    _min_amount_out: uint256,
) -> uint256:
    """
    @notice
        Triggers a swap in the referenced swap_router.
        Ensures min_amount_out is respected.
    """
    ERC20(_token_in).approve(self.swap_router, _amount_in)
```

- https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/spot-dex/Dca.vy#L204C1-L204C81
```vyper
    # transfer token_in from user to self
    self._safe_transfer_from(order.token_in, order.account, self, order.amount_in_per_execution)

    # approve UNISWAP_ROUTER to spend amount token_in
    ERC20(order.token_in).approve(UNISWAP_ROUTER, order.amount_in_per_execution)
```
- https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/spot-dex/LimitOrders.vy#L162C1-L164C67
```vyper
    # approve UNISWAP_ROUTER to spend amount token_in
    ERC20(order.token_in).approve(UNISWAP_ROUTER, order.amount_in)
```
- https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/spot-dex/TrailingStopDex.vy#L173C4-L174C67
```vyper
 # approve UNISWAP_ROUTER to spend token_in
    ERC20(order.token_in).approve(UNISWAP_ROUTER, order.amount_in)
```
However, if the token involved is an ERC20 token that does not work when changing the allowance from an existing non-zero allowance value, it will break a number of key functions or features of the protocol as the Approve function is utilized extensively within the contracts as shown above.
## Impact
A number of features within the contracts will not work if the approve function reverts.
## Code Snippet
- https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/margin-dex/SwapRouter.vy#L68C1-L68C57
- https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/margin-dex/Vault.vy#L384C1-L395C59
- https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/spot-dex/Dca.vy#L204C1-L204C81
- https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/spot-dex/LimitOrders.vy#L162C1-L164C67
- https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/spot-dex/TrailingStopDex.vy#L173C4-L174C67

## Tool used

Manual Review

## Recommendation
It is recommended to set the allowance to zero before increasing the allowance and use safeApprove/safeIncreaseAllowance