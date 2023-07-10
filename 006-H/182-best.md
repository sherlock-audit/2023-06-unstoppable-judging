chainNue

high

# Adversary manipulate the middle path when calling `execute_dca_order`, resulting user loss, benefiting the attacker

## Summary

Adversary manipulate the middle path when calling `execute_dca_order`, resulting user loss, benefiting the attacker

## Vulnerability Detail

Allowing anyone to call the `execute_dca_order` function with a custom `_uni_hop_path` introduce security issue. If an attacker constructs a malicious path with their own token in the middle, they could manipulate the liquidity and perform an exploit. The `_uni_hop_path` parameter in `execute_dca_order` is really dangerous input which can be manipulated by attacker.

For example a user want to Dca `USDC -> WETH` by posting `post_dca_order` with `_token_in` is USDC, and `_token_out` is WETH, and with the `min_amount_out` generated from `_calc_min_amount_out` which is based on twap and slippage percentage (in contrary, `LimitOrders.vy` use input `min_amount_out` for token out from the `post_limit_order` function, thus will not facing same effect like this issue)

When attacker see this Dca order, they can create a new ERC20 `MYTOKEN` (and will provide liquidity in uniswap) and plan to attack the Dca by selecting his own token as the middle path, so `execute_dca_order` will be executed using `USDC -> MYTOKEN -> WETH` path.

Since attacker can manipulate the liquidity of `MYTOKEN` in Uniswap, resulting user will get bad swap amount due to the custom path provided by attacker.

Slippage and twap length as protection are not enough, because the middle token path is the problem here. Moreover, using twap with twap_length can also be manipulated by preparing / pre-attack pool since the `last_execution` and `seconds_between_executions` is known, also the order's twap_length is visible, so attack scenario can prepared beforehand.

```python
File: Dca.vy
163: @external
164: @nonreentrant('lock')
165: def execute_dca_order(_uid: bytes32, _uni_hop_path: DynArray[address, 3], _uni_pool_fees: DynArray[uint24, 2], _share_profit: bool):
...
174:     # ensure path is valid
175:     assert len(_uni_hop_path) in [2, 3], "[path] invlid path"
176:     assert len(_uni_pool_fees) == len(_uni_hop_path)-1, "[path] invalid fees"
177:     assert _uni_hop_path[0] == order.token_in, "[path] invalid token_in"
178:     assert _uni_hop_path[len(_uni_hop_path)-1] == order.token_out, "[path] invalid token_out"
...
206:     # Vyper way to accommodate abi.encode_packed
207:     path: Bytes[66] = empty(Bytes[66])
208:     if(len(_uni_hop_path) == 2):
209:         path = concat(convert(_uni_hop_path[0], bytes20), convert(_uni_pool_fees[0], bytes3), convert(_uni_hop_path[1], bytes20))
210:     elif(len(_uni_hop_path) == 3):
211:         path = concat(convert(_uni_hop_path[0], bytes20), convert(_uni_pool_fees[0], bytes3), convert(_uni_hop_path[1], bytes20), convert(_uni_pool_fees[1], bytes3), convert(_uni_hop_path[2], bytes20))
212:
213:     min_amount_out: uint256 = self._calc_min_amount_out(order.amount_in_per_execution, _uni_hop_path, _uni_pool_fees, order.twap_length, order.max_slippage)
214:
215:     uni_params: ExactInputParams = ExactInputParams({
216:         path: path,
217:         recipient: self,
218:         deadline: block.timestamp,
219:         amountIn: order.amount_in_per_execution,
220:         amountOutMinimum: min_amount_out
221:     })
222:     amount_out: uint256 = UniswapV3SwapRouter(UNISWAP_ROUTER).exactInput(uni_params)
...
238:
```

Steps:

1. A malicious user creates a custom token called "MYTOKEN" and provides liquidity for it on Uniswap v3. They allocate liquidity to create extreme price ranges for MYTOKEN.

2. A regular user intends to swap USDC to WETH.

3. The malicious user intercepts the transaction and modifies the path by adding intermediate token with MYTOKEN. The new path becomes USDC -> MYTOKEN -> WETH.

4. Uniswap v3 receives the modified path and attempts to execute the swap based on the provided path.

5. Uniswap v3 calculates the best available price based on the modified path, which includes MYTOKEN.

6. Due to the manipulated liquidity pool of MYTOKEN, the price of MYTOKEN is significantly skewed, leading to an unfair price for the regular user executing the multi-hop swap.

7. The malicious user takes advantage of the distorted liquidity pool by executing multi-hop swaps involving MYTOKEN at highly advantageous prices.

8. Innocent users who unknowingly interact with the manipulated liquidity pool may receive unfavorable prices for their USDC to WETH swaps, leading to financial losses.

In this scenario, the malicious user modifies the path of the multi-hop swap by replacing the intended intermediate token with MYTOKEN. As a result, the malicious user exploits the manipulated liquidity pool and executes trades at favorable prices, while causing losses for other traders and disrupting the market equilibrium

## Impact

User will get a bad rate (far from the normal rate) due to middle token rate is manipulated, thus losing his asset.

## Code Snippet

https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/spot-dex/Dca.vy#L163-L237

## Tool used

Manual Review

## Recommendation

Might need to have a minimal desired output amount of range (for the output token), or remove the option to input manual path with anyone can call the `execute_dca_order` then replace it with the Oracle price rate. Other way, consider implement a whitelist token for swap path.
