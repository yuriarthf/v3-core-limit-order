# Uniswap V3 with Limit Orders

This repository was forked from UniswapV3 in order to add an additional feature to the core pool: limit orders. Currently, users can simulate limit orders by providing liquidity to a tick interval that is above or below the current tick. Since the ticks are not the active ones, they provide just one token as liquidity, and when the price reaches this tick, the tokens are swapped to the opposite one. For instance, `token0` is swapped for `token1` when the limit order tick is greater than the current tick, and `token1` is swapped for `token0` when it's less than the current tick.

One problem is that users need to be aware of when the swap takes place, or the price trend can reverse, and all swapped tokens will revert to the previous one. To address this issue, the UniswapV3 Core contracts were forked, and two new functions were added to it: `createLimitOrder`, which allows users to create limit orders, and `collectLimitOrder`, which allows users to collect the swapped tokens or cancel their orders.


## Changes

### Added Lines
[UniswapV3Pool.sol](./contracts/UniswapV3Pool.sol):
1. Additional Variables: [**101-120**](./contracts/UniswapV3Pool.sol?plain=1#L101-L120)
2. `createLimitOrder` function: [**512-578**](./contracts/UniswapV3Pool.sol?plain=1#L512-L578)
3. `_removeUserLimitWithFees` helper function: [**580-628**](./contracts/UniswapV3Pool.sol?plain=1#L580-L628) 
4. `collectLimitOrder` function: [**630-711**](./contracts/UniswapV3Pool.sol?plain=1#L630-L711)
5. Limit order liquidation function: [**822-854**](./contracts/UniswapV3Pool.sol?plain=1#L822-L854)

[IUniswapV3PoolState.sol](./contracts/interfaces/pool/IUniswapV3PoolState.sol): [**99-123**](./contracts/interfaces/pool/IUniswapV3PoolState.sol?plain=1#L99-L123)

[IUniswapV3PoolLimitOrder.sol](./contracts/interfaces/pool/IUniswapV3PoolLimitOrder.sol): [**all**](./contracts/interfaces/pool/IUniswapV3PoolLimitOrder.sol?plain=1#L1-L19)

[UniswapV3Pool.limit.spec.ts](./test/UniswapV3Pool.limit.spec.ts): [**all**](./test/UniswapV3Pool.limit.spec.ts?plain=1#L1-L258)


### Added Functions:
1. `createLimitOrder`: Places a limit order at a given [`tickLower`, `tickLower + tickSpacing`) interval.
```
function createLimitOrder(address recipient, int24 tickLower, uint128 amount) external;
```

2. `collectLimitOrder`: Claims limit orders placed at a given [`tickLower`, `tickLower + tickSpacing`) interval. If it has been liquidated, claims swapped tokens, if not, return the provided liquidity.
```
function collectLimitOrder(address recipient, int24 tickLower) external;
```

### Additional variables
1. `currentLimitEpoch`: Used to keep track of the current limit order epoch.
```
// tickLower => epoch
mapping(int24 => uint256) public currentLimitEpoch;
```

2. `userEpochInfos`: Used to store the epochs that the user participated (`epochs`) and the index of the last collected one.
```
struct UserEpochInfo {
  uint256 currIndex;
  uint256[] epochs;
}
/// (tickLower, user) => UserEpochInfo
mapping(bytes32 => UserEpochInfo) private userEpochInfo;
```

3. `limitOrderStatuses`: Variables containing the status/metadata for each limit order instance. Such as the way it's going to be liquidated (`zeroForOne`), if any limit order was already placed for this tick and epoch (`initialized`), the total liquidity provided (`totalLiquidity`) and how much of the order was filled (`totalFilled`).
```
struct LimitOrderStatus {
    bool initialized;
    bool zeroForOne;
    uint128 totalFilled;
    uint128 totalLiquidity;
}
/// (tickLower, epoch) => Status of limit orders (Metadata for limit orders)
mapping(bytes32 => LimitOrderStatus) public limitOrderStatuses;
```

4. `usersLimitLiquidity`: How much each user has provided of liquidity for each of the `tickLower` and `epoch`. It's used to calculate the user share of the liquidated limit orders when claiming.
```
/// (tickLower, epoch) => User limit liquidity 
mapping(bytes32 => uint128) public usersLimitLiquidity;
```

### Tests
Test can be found [here](./test/UniswapV3Pool.limit.spec.ts). (UniswapV3Pool.limit.spec.ts file)

`TODO: Create more test cases to get 100% coverage`

## Rationale
When a user creates a limit order, internally, it creates a liquidity position with the `DEAD` address as the owner. This design choice was made to eliminate the necessity of looping over all limit orders when someone performs a swap that crosses the upper bound of the limit order interval.

When a swap is performed, crossing the upper bound (`tickUpper`) of the limit order, all the limit order instances that were crossed are liquidated (liquidity from the `DEAD` address is removed), and the swapped tokens and fees are accounted for in the `totalFilled` variable.

When limit order users claim their swapped tokens, they receive an amount proportional to the liquidity they provided for each of the limit orders.

## Partial Fills
Partial fills are currently not implemented since the UniswapV3 architecture does not immediately support it.

One of the reasons is that when a partial liquidity position is liquidated inside the liquidity interval to which it was assigned, a proportion of both `token0` and `token1` will be withdrawn, not just the token being liquidated to. Even if another liquidity position is created after the operation, the same amount of both tokens will need to be provided back, influenced by the current price (constant product formula).

One way to tackle this problem, although it would probably be better to rethink the architecture itself, is to create orders that might not be included in the position's liquidity pool and liquidate them when the price crosses the assigned `lowerTick`. One problem with this approach, which is solvable, is the need to determine the logic of which liquidity is going to be used first or which proportion of both will be used. Note that if a liquidity position is not created for the limit order tick, the limit order won't be executed until someone does so.
