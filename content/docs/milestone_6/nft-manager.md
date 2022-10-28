---
title: "NFT Manager"
weight: 3
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# NFT Manager Contract

Obviously, we're not going to add NFT-related functionality to the pool contract–we need a separate contract that will
merge NFTs and liquidity positions. Recall that, while working on our implementation, we built the `UniswapV3Manager`
contract to facilitate interaction with pool contracts (to make some calculations simpler and to enable multi-pool
swaps). This contract was a good demonstration of how core Uniswap contracts can be extended. And we're going to push
this idea a little bit further.

We'll need a manager contract that will implement the ERC721 standard and will manage liquidity positions. The contract
will have the standard NFT functionality (minting, burning, transferring, balances and ownership tracking, etc.) and will
allow to provide and remove liquidity to pools. The contract will need to be the actual owner of liquidity in pools
because we don't want to let users to add liquidity without minting a token and removing entire liquidity without burning
one.  We want every liquidity position to be linked to an NFT token, and we want to them to be synchronized.

Let's see what functions we'll have in the new contract:
1. since it'll be an NFT contract, it'll have all the ERC721 functions, including `tokenURI`, which returns the URI of
the image of an NFT token;
1. `mint` and `burn` to mint and burn liquidity and NFT tokens at the same time;
1. `addLiquidity` and `removeLiquidity` to add and remove liquidity in existing positions;
1. `collect`, to collect tokens after removing liquidity.

Alright, let's get to code.

## The Minimal Contract

Since we don't want to implement the ERC721 standard from scratch, we're going to use a library. We already have [Solmate](https://github.com/transmissions11/solmate)
in the dependencies, so we're going to use [its ERC721 implementation](https://github.com/transmissions11/solmate/blob/main/src/tokens/ERC721.sol).

> Using [the ERC721 implementation from OpenZeppelin](https://github.com/OpenZeppelin/openzeppelin-contracts/tree/master/contracts/token/ERC721)
is also an option, but I personally prefer the gas optimized contracts from Solmate.

This will be the bare minimum of the NFT manager contract:

```solidity
contract UniswapV3NFTManager is ERC721 {
    address public immutable factory;

    constructor(address factoryAddress)
        ERC721("UniswapV3 NFT Positions", "UNIV3")
    {
        factory = factoryAddress;
    }

    function tokenURI(uint256 tokenId)
        public
        view
        override
        returns (string memory)
    {
        return "";
    }
}
```

`tokenURI` will return an empty string until we implement a metadata and SVG renderer. We've added the stub so that
the Solidity compiler doesn't fail while we're working on the rest of the contract (the `tokenURI` function in the Solmate
ERC721 contract is virtual, so we must implement it).

## Minting

Minting, as we discussed earlier, will involve two operations: adding liquidity to a pool and minting an NFT.

To keep the links between pool liquidity positions and NFTs, we'll need a mapping and a structure:

```solidity
struct TokenPosition {
    address pool;
    int24 lowerTick;
    int24 upperTick;
}
mapping(uint256 => TokenPosition) public positions;
```

To find a position we need:
1. a pool address;
1. an owner address;
1. the boundaries of a position (lower and upper ticks).

Since the NFT manager contract will be the owner of all positions created via it, we don't need to store position's owner
address and we can only store the rest data. The keys in the `positions` mapping are token IDs; the mapping links NFT
IDs to the position data that's required to find a liquidity position.

Let's implement minting:

```solidity
struct MintParams {
    address recipient;
    address tokenA;
    address tokenB;
    uint24 fee;
    int24 lowerTick;
    int24 upperTick;
    uint256 amount0Desired;
    uint256 amount1Desired;
    uint256 amount0Min;
    uint256 amount1Min;
}

function mint(MintParams calldata params) public returns (uint256 tokenId) {
    ...
}
```

The minting parameters are identical to those of `UniswapV3Manager`, with an addition of `recipient`, which will allow
to mint NFT to another address.

In the `mint` function, we first add liquidity to a pool:

```solidity
IUniswapV3Pool pool = getPool(params.tokenA, params.tokenB, params.fee);

(uint128 liquidity, uint256 amount0, uint256 amount1) = _addLiquidity(
    AddLiquidityInternalParams({
        pool: pool,
        lowerTick: params.lowerTick,
        upperTick: params.upperTick,
        amount0Desired: params.amount0Desired,
        amount1Desired: params.amount1Desired,
        amount0Min: params.amount0Min,
        amount1Min: params.amount1Min
    })
);
```

`_addLiquidity` is identical to the body of `mint` function in the `UniswapV3Manager` contract: it converts ticks to
$sqrt(P)$, computes liquidity amount, and calls `pool.mint()`.

Next, we mint an NFT:

```solidity
tokenId = nextTokenId++;
_mint(params.recipient, tokenId);
totalSupply++;
```

`tokenId` is set to the current `nextTokenId` and the latter is then incremented. The `_mint` function is provided by
the ERC721 contract from Solmate. After minting a new token, we update `totalSupply`.

Finally, we need to store the information about the new token and the new position:

```solidity
TokenPosition memory tokenPosition = TokenPosition({
    pool: address(pool),
    lowerTick: params.lowerTick,
    upperTick: params.upperTick
});

positions[tokenId] = tokenPosition;
```

This will later help us find liquidity position by token ID.

## Adding Liquidity

Next, we'll implement a function to add liquidity to an existing position, in the case when we want more liquidity to
a position that already has some. In such cases, we don't want to mint an NFT, but only to increase the amount of liquidity
in an existing. position. For that, we'll only need to provide a token ID and token amounts:

```solidity
function addLiquidity(AddLiquidityParams calldata params)
    public
    returns (
        uint128 liquidity,
        uint256 amount0,
        uint256 amount1
    )
{
    TokenPosition memory tokenPosition = positions[params.tokenId];
    if (tokenPosition.pool == address(0x00)) revert WrongToken();

    (liquidity, amount0, amount1) = _addLiquidity(
        AddLiquidityInternalParams({
            pool: IUniswapV3Pool(tokenPosition.pool),
            lowerTick: tokenPosition.lowerTick,
            upperTick: tokenPosition.upperTick,
            amount0Desired: params.amount0Desired,
            amount1Desired: params.amount1Desired,
            amount0Min: params.amount0Min,
            amount1Min: params.amount1Min
        })
    );
}
```

This function ensures there's an existing token and calls `pool.mint()` with parameters of an existing position.

## Remove Liquidity

Recall that in the `UniswapV3Manager` contract we didn't implement a `burn` function because we wanted users to be owners
of liquidity positions. Now, we want the NFT manager to be the owner. And we can have liquidity burning implemented in
it:

```solidity
struct RemoveLiquidityParams {
    uint256 tokenId;
    uint128 liquidity;
}

function removeLiquidity(RemoveLiquidityParams memory params)
    public
    isApprovedOrOwner(params.tokenId)
    returns (uint256 amount0, uint256 amount1)
{
    TokenPosition memory tokenPosition = positions[params.tokenId];
    if (tokenPosition.pool == address(0x00)) revert WrongToken();

    IUniswapV3Pool pool = IUniswapV3Pool(tokenPosition.pool);

    (uint128 availableLiquidity, , , , ) = pool.positions(
        poolPositionKey(tokenPosition)
    );
    if (params.liquidity > availableLiquidity) revert NotEnoughLiquidity();

    (amount0, amount1) = pool.burn(
        tokenPosition.lowerTick,
        tokenPosition.upperTick,
        params.liquidity
    );
}
```

We're again checking that provided token ID is valid. And we also need to ensure that a position has enough liquidity
to burn.

## Collecting Tokens

The NFT manager contract can also collect tokens after burning liquidity. Notice that collected tokens are send to `msg.sender`
since the contract manages liquidity on behalf of the caller:

```solidity
struct CollectParams {
    uint256 tokenId;
    uint128 amount0;
    uint128 amount1;
}

function collect(CollectParams memory params)
    public
    isApprovedOrOwner(params.tokenId)
    returns (uint128 amount0, uint128 amount1)
{
    TokenPosition memory tokenPosition = positions[params.tokenId];
    if (tokenPosition.pool == address(0x00)) revert WrongToken();

    IUniswapV3Pool pool = IUniswapV3Pool(tokenPosition.pool);

    (amount0, amount1) = pool.collect(
        msg.sender,
        tokenPosition.lowerTick,
        tokenPosition.upperTick,
        params.amount0,
        params.amount1
    );
}
```

## Burning

Finally, burning. Unlike the other functions of the contract, this function doesn't do anything with a pool: it only
burns an NFT. And to burn an NFT, the underlying position must be empty and tokens must be collected. So, if we want
to burn an NFT, we need to:
1. call `removeLiquidity` an remove the entire position liquidity;
1. call `collect` to collect the tokens after burning the position;
1. call `burn` to burn the token.

```solidity
function burn(uint256 tokenId) public isApprovedOrOwner(tokenId) {
    TokenPosition memory tokenPosition = positions[tokenId];
    if (tokenPosition.pool == address(0x00)) revert WrongToken();

    IUniswapV3Pool pool = IUniswapV3Pool(tokenPosition.pool);
    (uint128 liquidity, , , uint128 tokensOwed0, uint128 tokensOwed1) = pool
        .positions(poolPositionKey(tokenPosition));

    if (liquidity > 0 || tokensOwed0 > 0 || tokensOwed1 > 0)
        revert PositionNotCleared();

    delete positions[tokenId];
    _burn(tokenId);
    totalSupply--;
}
```

That's it!

{{< katex display >}} {{</ katex >}}