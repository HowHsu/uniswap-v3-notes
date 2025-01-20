# There are two kinds of fees: swap fee and protocol fee
## swap fee
This goes to the LPs. Currently the fee is set by the owner of UniswapV3Factory.sol.
Factory is the entry point for pools.

```solidity
      /// @inheritdoc IUniswapV3Factory
      mapping(uint24 => int24) public override feeAmountTickSpacing;
      /// @inheritdoc IUniswapV3Factory
      mapping(address => mapping(address => mapping(uint24 => address))) public override getPool;

      constructor() {
          owner = msg.sender;
          emit OwnerChanged(address(0), msg.sender);

          feeAmountTickSpacing[500] = 10;
          emit FeeAmountEnabled(500, 10);
          feeAmountTickSpacing[3000] = 60;
          emit FeeAmountEnabled(3000, 60);
          feeAmountTickSpacing[10000] = 200;
          emit FeeAmountEnabled(10000, 200);
      }
```

The unit of numbers 500, 3000, 10000 is bp%, so by default, 3 types of fee, 0.05%, 0.30%, 1.00%
only the owner can enable other numbers.
```solidity
      /// @inheritdoc IUniswapV3Factory
      function enableFeeAmount(uint24 fee, int24 tickSpacing) public override {
          require(msg.sender == owner);
          require(fee < 1000000);
          // tick spacing is capped at 16384 to prevent the situation where tickSpacing is so large that
          // TickBitmap#nextInitializedTickWithinOneWord overflows int24 container from a valid tick
          // 16384 ticks represents a >5x price change with ticks of 1 bips
          require(tickSpacing > 0 && tickSpacing < 16384);
          require(feeAmountTickSpacing[fee] == 0);

          feeAmountTickSpacing[fee] = tickSpacing;
          emit FeeAmountEnabled(fee, tickSpacing);
      }
```

the range of fee can be [0, 100%]

for now, only 0.05%, 0.30%, 1.00% enabled

## protocol fee
this goes to the community/uniswap team, and it's disabled for now

```solidity
      function initialize(uint160 sqrtPriceX96) external override {
          require(slot0.sqrtPriceX96 == 0, 'AI');

          int24 tick = TickMath.getTickAtSqrtRatio(sqrtPriceX96);

          (uint16 cardinality, uint16 cardinalityNext) = observations.initialize(_blockTimestamp());

          slot0 = Slot0({
              sqrtPriceX96: sqrtPriceX96,
              tick: tick,
              observationIndex: 0,
              observationCardinality: cardinality,
              observationCardinalityNext: cardinalityNext,
              feeProtocol: 0,
              unlocked: true
          });

          emit Initialize(sqrtPriceX96, tick);
      }
```

## How is swap fee counted?
Unlike uniswap v2, we cannot infer the total fee by `fee = amountIn * fee_rate`, since only liquidity
used can get the fee. So it is actually calculated tick by tick.

To be specific, say we have got fee from swapping from [tick0, tick1] as feeAmount, it is used
to calculate a global per unit(liquidity) profit:

 `state.feeGrowthGlobalX128 += FullMath.mulDiv(step.feeAmount, FixedPoint128.Q128, state.liquidity);`

(state.liquidity is the current available liquidity)

and after the swap is done:
`feeGrowthGlobal0X128 = state.feeGrowthGlobalX128;`

the global variable `feeGrowthGlobal0X128` stands for the profit for single unit of liquidity in this pool.
It is meanly leveraged by `mint()` and `burn()`, they both call `_modifyPosition() -> _updatePosition()`,
this is the core logic of fee calculation.

`_updatePosition()`

```solidity

          (uint256 feeGrowthInside0X128, uint256 feeGrowthInside1X128) =
              ticks.getFeeGrowthInside(tickLower, tickUpper, tick, _feeGrowthGlobal0X128, _feeGrowthGlobal1X128);

          position.update(liquidityDelta, feeGrowthInside0X128, feeGrowthInside1X128);

```
`ticks.getFeeGrowthInside()`
```solidity

          feeGrowthInside0X128 = feeGrowthGlobal0X128 - feeGrowthBelow0X128 - feeGrowthAbove0X128;
          feeGrowthInside1X128 = feeGrowthGlobal1X128 - feeGrowthBelow1X128 - feeGrowthAbove1X128;
```

Depends on where the current tick is, we can calculate `feeGrowthBelow0X128` and `feeGrowthAbove0X128`,
these two are maintained during the price/tick moves.

`feeGrowthInside0X128` means profit for per unit liquidity that locates in this tick range.
At last, we can calculate the total fee (as tokens) a LP should get when he burn the liquidity.

```solidity
          // calculate accumulated fees
          uint128 tokensOwed0 =
              uint128(
                  FullMath.mulDiv(
                      feeGrowthInside0X128 - _self.feeGrowthInside0LastX128,
                      _self.liquidity,
                      FixedPoint128.Q128
                  )
              );
          uint128 tokensOwed1 =
              uint128(
                  FullMath.mulDiv(
                      feeGrowthInside1X128 - _self.feeGrowthInside1LastX128,
                      _self.liquidity,
                      FixedPoint128.Q128
                  )
              );

          // update the position
          if (liquidityDelta != 0) self.liquidity = liquidityNext;
          self.feeGrowthInside0LastX128 = feeGrowthInside0X128;
          self.feeGrowthInside1LastX128 = feeGrowthInside1X128;
          if (tokensOwed0 > 0 || tokensOwed1 > 0) {
              // overflow is acceptable, have to withdraw before you hit type(uint128).max fees
              self.tokensOwed0 += tokensOwed0;
              self.tokensOwed1 += tokensOwed1;
          }
```
