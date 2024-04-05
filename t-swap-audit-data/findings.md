## High

### [H-1] Incorrect fee calculation in `TSwapPool::getinputAmountBasedOnOutput` causes protocol to take too many tokens from users, resulting in lost fees. 

**Description:** The `getinputAmountBasedOnOutput` function is used to calculate the amount of input tokens to take from the users given an amount of output tokens. However, the function miscalculates the resulting amount. When calculating the fee, it scales the amount by 10,000 instead of 1000.

**Impact:** The protocol takes too many tokens from users, resulting in lost fees.


**Recommended Mitigation:**

```diff
- return ((inputReserves * outputAmount) * 10000) /((outputReserves - outputAmount) * 997);

+ return((inputReserves * outputAmount) * 1000) /((outputReserves - outputAmount) * 997);
    
    

```

### [H-2] Lack of slippage protection in `TSwapPool::swapExactOutput` causes users to potentialy spend more tokens as input amount than expected.

**Description:** The `swapExactOutput` function does not have slippage protection, meaning that users could potentially have to spend more Input tokens to swap an exact amount of output tokens than they expected. The `TswapPool::SwapExactInput` function is similar to what is done in `swapExactOutput`, but it does have slippage protection by including `minOutputAmount` parameter, which is the minimum amount of output tokens that the user expects to receive and it reverts if the user does not receive at least that amount.

```javascript
  function swapExactOutput(
        IERC20 inputToken,
        IERC20 outputToken,
        uint256 outputAmount,
        uint64 deadline
    )

```

**Impact:** . This could severly drain the users funds if the market conditions are not favorable to the user.

**Proof of Concept:**

1. The price of 1 WETH right now is 1,000 USDC
2. User inputs a swapExactOutput looking for 1 WETH
    inputToken = USDC
    outputToken = WETH
    outputAmount = 1
    deadline = whatever
3. The function does not offer a maxInput amount
4. As the transaction is pending in the mempool, the market changes! And the price moves HUGE -> 1 WETH is now 10,000 USDC. 10x more than the user expected
5. The transaction completes, but the user sent the protocol 10,000 USDC instead of the expected 1,000 USDC

**Recommended Mitigation:**
We should include a maxInputAmount so the user only has to spend up to a specific amount, and can predict how much they will spend on the protocol.

```diff
    function swapExactOutput(
        IERC20 inputToken, 
+       uint256 maxInputAmount,
.
.
.
        inputAmount = getInputAmountBasedOnOutput(outputAmount, inputReserves, outputReserves);
+       if(inputAmount > maxInputAmount){
+           revert();
+       }        
        _swap(inputToken, inputAmount, outputToken, outputAmount);



```

### [H-3] `TSwapPool::sellPoolTokens` mismatches input and output tokens causing users to receive the incorrect amount of tokens

**Description:**  The `sellPoolTokens` function is intended to allow users to easily sell pool tokens and receive `WETH` in exchange. Users indicate how many pool tokens they're willing to sell in the `poolTokenAmount` parameter. However, the function currently miscalculates the swapped amount.

This is due to the fact that the swapExactOutput function is called, whereas the swapExactInput function is the one that should be called. Because users specify the exact amount of input tokens, not output.

**Impact:** Users will swap the wrong amount of tokens, which is a severe disruption of protcol functionality.

**Proof of Concept**


**Recommended Mitigation:**Consider changing the implementation to use `swapExactInput` instead of `swapExactOutput`. Note that this would also require changing the `sellPoolTokens` function to accept a new parameter (ie`minWethToReceive` to be passed to `swapExactInput`)

```diff
    function sellPoolTokens(
        uint256 poolTokenAmount,
+       uint256 minWethToReceive,    
        ) external returns (uint256 wethAmount) {
-        return swapExactOutput(i_poolToken, i_wethToken, poolTokenAmount, uint64(block.timestamp));
+        return swapExactInput(i_poolToken, poolTokenAmount, i_wethToken, minWethToReceive, uint64(block.timestamp));
    }


```


### [H-4] In `TSwapPool::_swap` the extra tokens given to users after every swapCount breaks the protocol invariant of x * y = k

**Description:**  The protocol follows a strict invariant of x * y = k. 
Where:

x: The balance of the pool token
y: The balance of WETH
k: The constant product of the two balances

This means, that whenever the balances change in the protocol, the ratio between the two amounts should remain constant, hence the k. However, this is broken due to the extra incentive in the `_swap` function. Meaning that over time the protocol funds will be drained.

The follow block of code is responsible for the issue.

```javascript
        swap_count++;
        if (swap_count >= SWAP_COUNT_MAX) {
            swap_count = 0;
            outputToken.safeTransfer(msg.sender, 1_000_000_000_000_000_000);
        }


```

**Impact:**A user could maliciously drain the protocol of funds by doing a lot of swaps and collecting the extra incentive given out by the protocol.

Most simply put, the protocol's core invariant is broken.

**Proof of Concept**

Proof of Concept:

1. A user swaps 10 times, and collects the extra incentive of 1_000_000_000_000_000_000 tokens
2. That user continues to swap untill all the protocol funds are drained.

<details>
<summary>Proof of Concept Code</summary>

```javascript
 function testInvariantBroken() public {
        vm.startPrank(liquidityProvider);
        weth.approve(address(pool), 100e18);
        poolToken.approve(address(pool), 100e18);
        pool.deposit(100e18, 100e18, 100e18, uint64(block.timestamp));
        vm.stopPrank();

        uint256 outputWeth = 1e17;

        vm.startPrank(user);
        poolToken.approve(address(pool), type(uint256).max);
        poolToken.mint(user, 100e18);
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));

        int256 startingY = int256(weth.balanceOf(address(pool)));
        int256 expectedDeltaY = int256(-1) * int256(outputWeth);

        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        vm.stopPrank();

        uint256 endingY = weth.balanceOf(address(pool)); //Here, some extra WETH has been sent to the swapper along with expected
        // ouput amount because the protocol incentivizes the swapper after every 10 transaction. This will break the invariant.
        //The endingY will be less than what is was supposed to be and subtracting it from startingY will not be equal to expectedDeltaY.
        int256 actualDeltaY = int256(endingY) - int256(startingY);
        assertEq(actualDeltaY, expectedDeltaY);
    }


```

</details>

**Recommended Mitigation:** Remove the extra incentive mechanism. If you want to keep this in, we should account for the change in the x * y = k protocol invariant. Or, we should set aside tokens in the same way we do with fees.

```diff
-        swap_count++;
-        // Fee-on-transfer
-        if (swap_count >= SWAP_COUNT_MAX) {
-            swap_count = 0;
-            outputToken.safeTransfer(msg.sender, 1_000_000_000_000_000_000);
-        }

```

## Medium

### [M-1] `TSwapPool::deposit` is missing deadline check causing  transactions to complete even after deadline

**Description:** The `deposit` function accepts a deadline parameter which according to the documentation sis "The Deadline for transaction to be completed by". however, this paramter is not used in the function and therefore transactions can be completed even after the deadline, even in market conditions where the market conditions are not favorable to the user.
**Impact:** Transactions could be sent when market conditions are unfavourabke to the user, causing the user to lose money.


**Recommended Mitigation:**

```diff

 function deposit(
        uint256 wethToDeposit,
        uint256 minimumLiquidityTokensToMint,
        uint256 maximumPoolTokensToDeposit,
        uint64 deadline
+    ) external revertIfDeadlinePassed(uint64 deadline)

```

## Low

### [L-1] `TswapPool::LiquidityAdded` event has parameters out of order causing event to emit incorrect infodrmation

**Description:**  When the `LiquidityAdded` event is emitted, the `wethToDepoisit`
 and `poolTokensToDeposit` parameters are out of order, causing the event to emit incorrect information.
 
**Impact:** Event emmisions are incorrect, leading to off-chain functionsa potentially using incorrect information.



**Recommended Mitigation:**

```diff
- emit LiquidityAdded(msg.sender, poolTokensToDeposit, wethToDeposit);

+ emit LiquidityAdded(msg.sender, wethToDeposit, poolTokensToDeposit);

```






## Informationals

### [I-1] `PoolFactory__PoolDoesNotExist` is not used and should be removed

```diff

contract PoolFactory {
-    error PoolFactory__PoolDoesNotExist(address tokenAddress);
}

```
### [I-2] lacking zero address checks

```diff
    constructor(address wethToken) {
+   if(wethToken == address(0)){
    revert();
}        
        i_wethToken = wethToken;
    }

```

### [I-3] `PoolFactory::createPool` shoudl use `.symbol()` instead `.name()`

```diff
+   string memory liquidityTokenSymbol = string.concat("ts", IERC20(tokenAddress).symbol());
-   string memory liquidityTokenSymbol = string.concat("ts", IERC20(tokenAddress).name());

```
