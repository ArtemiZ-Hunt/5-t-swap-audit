# High

### [H-1] Incorrect fee calculation in `TSwapPool::getInputAmountBasedOnOutput` causes protocol to take too many tokens from users, resulting in lost fees

**Description:** 

The `getInputAmountBasedOnOutput` function is intended to calculate the amount of tokens a user should deposit given an amount of output tokens. However, the function currently miscalculates the resulting amount. When calculating the fee, it scales the amount by 10_000 instead of 1_000. 

**Impact:** 

Protocol takes more fees than expected from users.

**Prove of Code**

```javascript

    function testGetInputAmountBasedOnOutput() public {
        uint256 outputAmount = 100;
        uint256 inputReserves = 67;
        uint256 outputReserves = 150;

        uint256 expectedResult = ((inputReserves * outputAmount) * 1000) / ((outputReserves - outputAmount) * 997);
        uint256 actualResult = pool.getInputAmountBasedOnOutput(outputAmount, inputReserves, outputReserves);

        assert(expectedResult != actualResult);
    }
```

**Recommended Mitigation:** Consider using a cryptographically provable random number generator such as Chainlink VRF.

```diff

    function getInputAmountBasedOnOutput(
    uint256 outputAmount,
    uint256 inputReserves,
    uint256 outputReserves
    )
        public
        pure
        revertIfZero(outputAmount)
        revertIfZero(outputReserves)
        returns (uint256 inputAmount)
    {
-        return ((inputReserves * outputAmount) * 10000) / ((outputReserves - outputAmount) * 997);

+        return ((inputReserves * outputAmount) * 1000) / ((outputReserves - outputAmount) * 997);
    }

```

### [H-2] Lack of slippage protection in `TSwapPool::swapExactOutput` causes users to potentially receive way fewer tokens

**Description:** 

The `swapExactOutput` function does not include any sort of slippage protection. This function is similar to what is done in `TSwapPool::swapExactInput`, where the function specifies a `minOutputAmount`, the `swapExactOutput` function should specify a `maxInputAmount`.

**Impact:** 

If market conditions change before the transaction processes, the user could get a much worse swap.

**Prove of Concept**

1. The price of 1 WETH right now is 1,000 USDC
2. User inputs a `swapExactOutput` looking for 1 WETH
    1. inputToken = USDC
    2. outputToken = WETH
    3. outputAmount = 1
    4. deadline = whatever
3. The function does not offer a maxInput amount
4. As the transaction is pending in the mempool, the market changes!
And the price moves HUGE -> 1 WETH is now 10,000 USDC. 10x more than the user expected.
5. The transaction completes, but the user sent the protocol 10,000 USDC instead of the expected 1,000 USDC.

**Recommended Mitigation:** We should include a `maxInputAmount` so the user only has to spend up to a specific amount, and can predict how much they will spend on the protocol.

```diff

    function swapExactOutput(
        IERC20 inputToken,
        IERC20 outputToken,
        uint256 outputAmount,
+       uint256 maxInputAmount
        uint64 deadline
    )
        public
        revertIfZero(outputAmount)
        revertIfDeadlinePassed(deadline)
        returns (uint256 inputAmount)
    {
        uint256 inputReserves = inputToken.balanceOf(address(this));
        uint256 outputReserves = outputToken.balanceOf(address(this));

        inputAmount = getInputAmountBasedOnOutput(outputAmount, inputReserves, outputReserves);

+       if(inputAmount > maxInputAmount){
+           revert();
+       }

        _swap(inputToken, inputAmount, outputToken, outputAmount);
    }

```

### [H-3] `TSwapPool::sellPoolTokens` mismatches input and output tokens causing users to receive the incorrect amount of tokens

**Description:** 

The `sellPoolTokens` function is intended to allow users to easily sell pool tokens and receive WETH in exchange. Users indicate how many pool tokens they are willing to sell in the `poolTokenAmount` parameter. However, the function currently miscalculates the swapped amount.

This is due to the fact that the `swapExactOutput` function is called whereas the `swapExactInput` function is the one that should be called. Because users specify the exact amount of input tokens, not output. 

**Impact:** 

Users will swap the wrong amount of tokens, which is a severe disruption of protocol functionality.

**Recommended Mitigation:** 

Consider changing the implementation to use `swapExactInput` instead of `swapExactOutput`. Note that this would also require changing the `sellPoolTokens` function to accept a new parameter (ie `minWethToReceive` to be passed to `swapExactInput`).

```diff

    function sellPoolTokens(
        uint256 poolTokenAmount,
+       uint256 minWethToReceive 
        ) external returns (uint256 wethAmount) {
-       return swapExactOutput(i_poolToken, i_wethToken, poolTokenAmount, uint64(block.timestamp));
+       return swapExactInput(i_poolToken, poolTokenAmount, i_wethToken, minWethToReceive, uint64(block.timestamp));
    }

```

### [H-4] In `TSwapPool::_swap` the extra tokens given to users after every `swapCount` breaks the protocol invariant of `x * y = k`

**Description:** 

The protocol follows a strict invariant of `x * y = k`. Where:
- `x`: The balance of the pool token
- `y`: The balance of WETH
- `k`: The constant product of the two balances 

This means, that whatever the balances change in the protocol, the ratio between the two amounts should remain constant, hence the `k`. However, this is broken die to the extra incentive in the `_swap` function. Meaning that over time the protocol funds will be drained.

The following block of code is responsible for the issue

```javascript

        swap_count++;
        if (swap_count >= SWAP_COUNT_MAX) {
            swap_count = 0;
            outputToken.safeTransfer(msg.sender, 1_000_000_000_000_000_000);
        }

```
**Impact:** 

The protocol's core invariant is broken.

**Prove of Concept**

1. A user swaps 10 times, and collects the extra incentive of `1_000_000_000_000_000_000` tokens

2. That user continues to swap untill all the protocol funds are drained.

<detail>
<summary>Proof of Code</summary>

Place the following into `TSwapPool.t.sol`:

```javascript

function testInvariantBroken() public {
        vm.startPrank(liquidityProvider);
        weth.approve(address(pool), 100e18);
        poolToken.approve(address(pool), 100e18);
        pool.deposit(100e18, 100e18, 100e18, uint64(block.timestamp));
        vm.stopPrank();

        uint256 outputWeth = 1e17;
        int256 startingY = int256(weth.balanceOf(address(pool)));
        int256 expectedDeltaY = int256(-1) * int256(outputWeth);

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
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        vm.stopPrank();

        //actual

        uint256 endingY = weth.balanceOf(address(pool));

        int256 actualDeltaY = int256(endingY) - int256(startingY);

        assertEq(actualDeltaY, expectedDeltaY);
    }

```

</detail>

**Recommended Mitigation:** 

Remove the extra incentive mechanism. If you want to keep this in, we should account for the change in x * y = K protocol incentive. Or, we should set aside tokens in the same way we do with fees.

```diff

-      swap_count++;
-      if (swap_count >= SWAP_COUNT_MAX) {
-            swap_count = 0;
-            outputToken.safeTransfer(msg.sender, 1_000_000_000_000_000_000);
-        }

```

# Medium

### [M-1] `TSwapPool::deposit` is missing deadline check causing transaction to complete even after the deadline

**Description:** 

The `TSwapPool::deposit` function accepts a deadline parameter, which according to the documentation is "The deadline for the transaction to be completed by". However, this parameteris never used. As a consequence, operations that add liquidity to the pool might be executed at unexpected times, in market conditions where the deposit rate is unfavorable.

<!-- MEV attacks -->

**Impact:** Transaction could be sent when market conditions are unfavorible to deposit, even when adding a deadline parameter.

**Proof of Concept:**

The `deadline` parameter is unused. 


**Recommended Mitigation:** 

Consider making the following change to the function

```diff

    function deposit(
        uint256 wethToDeposit,
        uint256 minimumLiquidityTokensToMint,
        uint256 maximumPoolTokensToDeposit,
        uint64 deadline
    )
        external
+       revertIfDeadlinePassed(deadline) 
        revertIfZero(wethToDeposit)
        returns (uint256 liquidityTokensToMint)
    {

```

# Low

### [L-1] `LiquidityAdded` event is not emmited correctly in `TSwapPool::_addLiquidityMintAndTransfer`

**Description** Two of the parameters given to the `LiquidityAdded` event are swapped

```javascript

    event LiquidityAdded(address indexed liquidityProvider, uint256 wethDeposited, uint256 poolTokensDeposited);


    emit LiquidityAdded(msg.sender, poolTokensToDeposit, wethToDeposit);

```

**Impact**

Event emission is incorrect, leading to off-chain functions potentially malfunctioning.

**Recommended Mitigation**

The easiest recommendation would be to change the places of the parameters. 


```diff

-    emit LiquidityAdded(msg.sender, poolTokensToDeposit, wethToDeposit);

+    emit LiquidityAdded(msg.sender, wethToDeposit, poolTokensToDeposit);

```

### [L-2] Default value returned by `TSwapPool::swapExactInput` results in incorrect return value given

**Description** 

The `swapExactInput` function is expected to return the actual amount of tokens bought by the caller. However, while it declares the named return value `output` it is never assigned a value, nor uses an explicit return statement.

**Impact**

The return value will always be 0, giving incorrect information to the caller.

**Recommended Mitigation**


```diff

        uint256 inputReserves = inputToken.balanceOf(address(this));
        uint256 outputReserves = outputToken.balanceOf(address(this));

-        uint256 outputAmount = getOutputAmountBasedOnInput(inputAmount, inputReserves, outputReserves);

+        output = getOutputAmountBasedOnInput(inputAmount, inputReserves, outputReserves);

-        if (outputAmount < minOutputAmount) {
-            revert TSwapPool__OutputTooLow(outputAmount, minOutputAmount);
-        }

+        if (output < minOutputAmount) {
+            revert TSwapPool__OutputTooLow(output, minOutputAmount);
+        }

-        _swap(inputToken, inputAmount, outputToken, outputAmount);
+        _swap(inputToken, inputAmount, outputToken, outout);

```


# Gas

### [G-1] Declared variable `poolTokenReserves` storing data, but is never used

`poolTokenReserves` in `TSwapPool::deposit` is used to store data, which is never used. This leads to unnecessary gas spending, which could be avoid.

```diff

-     uint256 poolTokenReserves = i_poolToken.balanceOf(address(this));

```

# Informationals

### [I-1]: Custom error `PoolFactory::PoolFactory__PoolDoesNotExist` is not used anywhere in the code, causing missleading information about the eventual error that can occur

Consider removing this error or finding where this error could occur, so that it could be used and the user know what might go wrong within the function.

<details><summary>Custom error</summary>


- Found in src/PoolFactory.sol [Line: 23](src/PoolFactory.sol)

```diff 
- error PoolFactory__PoolDoesNotExist(address tokenAddress);
```

</details>

### [I-2]: Lacking zero address checker in `PoolFactory::constructor`

```diff 
    constructor(address wethToken) {
+       if (wethToken == address(0)){
+           revert()
+       }
        i_wethToken = wethToken;
    }
```

</details>

### [I-3]: Instead of .name() in `PoolFactory::createPool`, .symbol() should be used

<details>


 ```diff
-    string memory liquidityTokenSymbol = string.concat("ts", IERC20(tokenAddress).name());
+    string memory liquidityTokenSymbol = string.concat("ts", IERC20(tokenAddress).symbol());
```

</details>

### [I-4]: If there are more than three parameters in an event, they should be indexed

Index event fields make the field more quickly accessible to off-chain tools that parse events. However, note that each index field costs extra gas during emission, so it's not necessarily best to index the maximum allowed per event (three fields). Each event should use three indexed fields if there are three or more fields, and gas usage is not particularly of concern for the events in question. If there are fewer than three fields, all of the fields should be indexed.

```diff

-    event Swap(address indexed swapper, IERC20 tokenIn, uint256 amountTokenIn, IERC20 tokenOut, uint256 amountTokenOut);

+    event Swap(address indexed swapper, IERC20 indexed tokenIn, uint256 indexed amountTokenIn, IERC20 indexed tokenOut, uint256 indexed amountTokenOut);

-    event LiquidityAdded(address indexed liquidityProvider, uint256 wethDeposited, uint256 poolTokensDeposited);

+    event LiquidityAdded(address indexed liquidityProvider, uint256 indexed wethDeposited, uint256 indexed poolTokensDeposited);

-    event LiquidityRemoved(address indexed liquidityProvider, uint256 wethWithdrawn, uint256 poolTokensWithdrawn);

+    event LiquidityRemoved(address indexed liquidityProvider, uint256 indexed wethWithdrawn, uint256 indexed poolTokensWithdrawn);    

```


### [I-5]: Lacking zero address checker in `TSwapPool::constructor`

```diff 
constructor(
        address poolToken,
        address wethToken,
        string memory liquidityTokenName,
        string memory liquidityTokenSymbol
    )
        ERC20(liquidityTokenName, liquidityTokenSymbol)
    {

+        if (wethToken == address(0) || poolToken == address(0)){
+            revert();
+        }

        i_wethToken = IERC20(wethToken);
        i_poolToken = IERC20(poolToken);
    }
```

</details>

### [I-6]: `MINIMUM_WETH_LIQUIDITY`is a constant in `TSwapPool` and is not required to be emmited as in `TSwapPool::deposit` 

```diff 

-       error TSwapPool__WethDepositAmountTooLow(uint256 minimumWethDeposit, uint256 wethToDeposit);
+       error TSwapPool__WethDepositAmountTooLow(uint256 wethToDeposit);



        if (wethToDeposit < MINIMUM_WETH_LIQUIDITY) {
-           revert TSwapPool__WethDepositAmountTooLow(MINIMUM_WETH_LIQUIDITY, wethToDeposit);
+           revert TSwapPool__WethDepositAmountTooLow(wethToDeposit);
        }
```

</details>

### [I-7]: Not following CEI (Checks, Effects, Impact) in `TSwapPool::deposit` 

`TSwapPool::deposit` does not follow CEI, which in the current case is not going to have an impact but is a good practice for writting code which we consider is good to be followed

```diff 

} else {

+            // external call
+            // updating variable
+            liquidityTokensToMint = wethToDeposit;

-            // This will be the "initial" funding of the protocol. We are starting from blank here!
-            // We just have them send the tokens in, and we mint liquidity tokens based on the weth
-            _addLiquidityMintAndTransfer(wethToDeposit, maximumPoolTokensToDeposit, wethToDeposit);

+            // This will be the "initial" funding of the protocol. We are starting from blank here!
+            // We just have them send the tokens in, and we mint liquidity tokens based on the weth
+            _addLiquidityMintAndTransfer(wethToDeposit, maximumPoolTokensToDeposit, wethToDeposit);

-            // external call
-            // updating variable
-            liquidityTokensToMint = wethToDeposit;
        }

```

</details>

### [I-8]: Use of "magic" numbers is discouraged

It can be confucing to see number literals in a codebase, and it's much more readable  if the numbers are given a name in `TSwapPool::getOutputAmountBasedOnInput` and `TSwapPool::getInputAmountBasedOnOutput`.


Examples:

```javascript

    uint256 inputAmountMinusFee = inputAmount * 997;
    uint256 numerator = inputAmountMinusFee * outputReserves;
    uint256 denominator = (inputReserves * 1000) + inputAmountMinusFee;
```

```javascript

    return ((inputReserves * outputAmount) * 10000) / ((outputReserves - outputAmount) * 997);
       
        
```
Instead, you could use:

```javascript

    uint256 public constant PRIZE_POOL_PERCENTAGE = 1000;
    uint256 public constant FEE_PERCENTAGE = 997;

```

### [I-9]: Missing natspec for `TSwapPool::swapExactInput`

Having a nastspec for your functions is a good practice. In this way you help the auditors and the users understand better the contract they interact with.

```diff

+   /// @param inputToken input token to swap / sell ie: DAI
+   /// @param inputAmount amount of input token to sell ie: DAI
+   /// @param outputToken output token to buy / buy ie: WETH
+   /// @param minOutputAmount minimum output amount expected to receive
+   /// @param deadline deadline for when the transaction should expire
    function swapExactInput(
        IERC20 inputToken, // e 
        uint256 inputAmount, // e 
        IERC20 outputToken, // e 
        uint256 minOutputAmount, // e minimum output amount expected to receive
        uint64 deadline // e deadline for when the transaction should expire
    )
        public
        revertIfZero(inputAmount)
        revertIfDeadlinePassed(deadline)
        returns (
            uint256 output
        )
```

## [I-10]: `public` functions not used internally could be marked `external`

Instead of marking a function as `public`, consider marking it as `external` if it is not used internally.

<details><summary>2 Found Instances</summary>


- Found in src/TSwapPool.sol [Line: 251](src/TSwapPool.sol)

    ```solidity
        function swapExactInput(
    ```

- Found in src/TSwapPool.sol [Line: 390](src/TSwapPool.sol)

    ```solidity
        function totalLiquidityTokenSupplyt(
    ```

</details>
