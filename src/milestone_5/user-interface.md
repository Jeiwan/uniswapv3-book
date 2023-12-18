# User Interface

In this milestone, we've added the ability to remove liquidity from a pool and collect accumulated fees. Thus, we need to reflect these changes in the user interface to allow users to remove liquidity.

## Fetching Positions

To let the user choose how much liquidity to remove, we first need to fetch the user's positions from a pool. To make this easier, we can add a helper function to the Manager contract, which will return the user position in a specific pool:
```solidity
function getPosition(GetPositionParams calldata params)
    public
    view
    returns (
        uint128 liquidity,
        uint256 feeGrowthInside0LastX128,
        uint256 feeGrowthInside1LastX128,
        uint128 tokensOwed0,
        uint128 tokensOwed1
    )
{
    IUniswapV3Pool pool = getPool(params.tokenA, params.tokenB, params.fee);

    (
        liquidity,
        feeGrowthInside0LastX128,
        feeGrowthInside1LastX128,
        tokensOwed0,
        tokensOwed1
    ) = pool.positions(
        keccak256(
            abi.encodePacked(
                params.owner,
                params.lowerTick,
                params.upperTick
            )
        )
    );
}
```

This will free us from calculating a pool address and a position key on the front end.

Then, after the user has typed in a position range, we can try fetching a position:
```js
const getAvailableLiquidity = debounce((amount, isLower) => {
  const lowerTick = priceToTick(isLower ? amount : lowerPrice);
  const upperTick = priceToTick(isLower ? upperPrice : amount);

  const params = {
    tokenA: token0.address,
    tokenB: token1.address,
    fee: fee,
    owner: account,
    lowerTick: nearestUsableTick(lowerTick, feeToSpacing[fee]),
    upperTick: nearestUsableTick(upperTick, feeToSpacing[fee]),
  }

  manager.getPosition(params)
    .then(position => setAvailableAmount(position.liquidity.toString()))
    .catch(err => console.error(err));
}, 500);
```

## Getting Pool Address

Since we need to call `burn` and `collect` on a pool, we still need to compute the pool's address on the front end. Recall that pool addresses are computed using the `CREATE2` opcode, which requires a salt and the hash of the contract's code. Luckily, Ether.js has the `getCreate2Address` function that allows to compute `CREATE2` in JavaScript:

```js
const sortTokens = (tokenA, tokenB) => {
  return tokenA.toLowerCase() < tokenB.toLowerCase ? [tokenA, tokenB] : [tokenB, tokenA];
}

const computePoolAddress = (factory, tokenA, tokenB, fee) => {
  [tokenA, tokenB] = sortTokens(tokenA, tokenB);

  return ethers.utils.getCreate2Address(
    factory,
    ethers.utils.keccak256(
      ethers.utils.solidityPack(
        ['address', 'address', 'uint24'],
        [tokenA, tokenB, fee]
      )),
    poolCodeHash
  );
}
```

However, the pool's codehash has to be hard coded because we don't want to store its code on the front end to calculate the hash. So, we'll use Forge to get the hash:

```bash
$ forge inspect UniswapV3Pool bytecode| xargs cast keccak 
0x...
```

And then use the output value in a JS constant:
```js
const poolCodeHash = "0x9dc805423bd1664a6a73b31955de538c338bac1f5c61beb8f4635be5032076a2";
```

## Removing Liquidity

After obtaining the liquidity amount and the pool address, we're ready to call `burn`:

```js
const removeLiquidity = (e) => {
  e.preventDefault();

  if (!token0 || !token1) {
    return;
  }

  setLoading(true);

  const lowerTick = nearestUsableTick(priceToTick(lowerPrice), feeToSpacing[fee]);
  const upperTick = nearestUsableTick(priceToTick(upperPrice), feeToSpacing[fee]);

  pool.burn(lowerTick, upperTick, amount)
    .then(tx => tx.wait())
    .then(receipt => {
      if (!receipt.events[0] || receipt.events[0].event !== "Burn") {
        throw Error("Missing Burn event after burning!");
      }

      const amount0Burned = receipt.events[0].args.amount0;
      const amount1Burned = receipt.events[0].args.amount1;

      return pool.collect(account, lowerTick, upperTick, amount0Burned, amount1Burned)
    })
    .then(tx => tx.wait())
    .then(() => toggle())
    .catch(err => console.error(err));
}
```

If burning was successful, we immediately call `collect` to collect the token amounts that were freed during burning.