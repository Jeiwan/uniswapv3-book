# NFT Renderer

Now we need to build an NFT renderer: a library that will handle calls to `tokenURI` in the NFT manager contract. It will render JSON metadata and an SVG for each minted token. As we discussed earlier, we'll use the data URI format, which requires base64 encoding–this means we'll need a base64 encoder in Solidity. But first, let's look at what our tokens will look like.


## SVG Template

I built this simplified variation of the Uniswap V3 NFTs:

![SVG template for NFT tokens](images/nft_template.png)

This is what its code looks like;
```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 300 480">
  <style>
    .tokens {
      font: bold 30px sans-serif;
    }

    .fee {
      font: normal 26px sans-serif;
    }

    .tick {
      font: normal 18px sans-serif;
    }
  </style>

  <rect width="300" height="480" fill="hsl(330,40%,40%)" />
  <rect x="30" y="30" width="240" height="420" rx="15" ry="15" fill="hsl(330,90%,50%)" stroke="#000" />

  <rect x="30" y="87" width="240" height="42" />
  <text x="39" y="120" class="tokens" fill="#fff">
    WETH/USDC
  </text>

  <rect x="30" y="132" width="240" height="30" />
  <text x="39" y="120" dy="36" class="fee" fill="#fff">
    0.05%
  </text>

  <rect x="30" y="342" width="240" height="24" />
  <text x="39" y="360" class="tick" fill="#fff">
    Lower tick: 123456
  </text>

  <rect x="30" y="372" width="240" height="24" />
  <text x="39" y="360" dy="30" class="tick" fill="#fff">
    Upper tick: 123456
  </text>
</svg>
```

This is a simple SVG template, and we're going to make a Solidity contract that fills the fields in this template and returns it in `tokenURI`. The fields that will be filled uniquely for each token:
1. the color of the background, which is set in the first two `rect`s; the hue component (330 in the template) will be unique for each token;
1. the names of the tokens of a pool the position belongs to (WETH/USDC in the template);
1. the fee of a pool (0.05%);
1. tick values of the boundaries of the position (123456).

Here are examples of NFTs our contract will be able to produce:

![NFT example 1](images/nft_example_2.png)
![NFT example 2](images/nft_example_3.png)


## Dependencies

Solidity doesn't provide a native Base64 encoding tool so we'll use a third-party one. Specifically, we'll use [the one from OpenZeppelin](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/Base64.sol).

Another tedious thing about Solidity is that it has very poor support for operations with strings. For example, there's no way to convert integers to strings–but we need that to render pool fee and position ticks in the SVG template. We'll use [the Strings library from OpenZeppelin](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/Strings.sol) to do that.

## Format of the Result

The data produced by the renderer will have this format:

```
data:application/json;base64,BASE64_ENCODED_JSON
```

The JSON will look like this:
```json
{
  "name": "Uniswap V3 Position",
  "description": "USDC/DAI 0.05%, Lower tick: -520, Upper text: 490",
  "image": "BASE64_ENCODED_SVG"
}
```

The image will be the above SVG template filled with position data and encoded in Base64.

## Implementing the Renderer

We'll implement the renderer in a separate library contract to not make the NFT manager contract too noisy:

```solidity
library NFTRenderer {
    struct RenderParams {
        address pool;
        address owner;
        int24 lowerTick;
        int24 upperTick;
        uint24 fee;
    }

    function render(RenderParams memory params) {
        ...
    }
}
```

In the `render` function, we'll first render an SVG, then a JSON. To keep the code cleaner, we'll break down each step into smaller steps.

We begin with fetching token symbols:
```solidity
function render(RenderParams memory params) {
    IUniswapV3Pool pool = IUniswapV3Pool(params.pool);
    IERC20 token0 = IERC20(pool.token0());
    IERC20 token1 = IERC20(pool.token1());
    string memory symbol0 = token0.symbol();
    string memory symbol1 = token1.symbol();

    ...
```

### SVG Rendering

Then we can render the SVG template:
```solidity
string memory image = string.concat(
    "<svg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 300 480'>",
    "<style>.tokens { font: bold 30px sans-serif; }",
    ".fee { font: normal 26px sans-serif; }",
    ".tick { font: normal 18px sans-serif; }</style>",
    renderBackground(params.owner, params.lowerTick, params.upperTick),
    renderTop(symbol0, symbol1, params.fee),
    renderBottom(params.lowerTick, params.upperTick),
    "</svg>"
);
```

The template is broken down into multiple steps:
1. first comes the header, which includes the CSS styles;
1. then the background is rendered;
1. then the top position information is rendered (token symbols and fee);
1. finally, the bottom information is rendered (position ticks).

The background is simply two `rect`s. To render them we need to find the unique hue of this token and then we concatenate all the pieces together:
```solidity
function renderBackground(
    address owner,
    int24 lowerTick,
    int24 upperTick
) internal pure returns (string memory background) {
    bytes32 key = keccak256(abi.encodePacked(owner, lowerTick, upperTick));
    uint256 hue = uint256(key) % 360;

    background = string.concat(
        '<rect width="300" height="480" fill="hsl(',
        Strings.toString(hue),
        ',40%,40%)"/>',
        '<rect x="30" y="30" width="240" height="420" rx="15" ry="15" fill="hsl(',
        Strings.toString(hue),
        ',100%,50%)" stroke="#000"/>'
    );
}
```

The top template renders token symbols and pool fees:
```solidity
function renderTop(
    string memory symbol0,
    string memory symbol1,
    uint24 fee
) internal pure returns (string memory top) {
    top = string.concat(
        '<rect x="30" y="87" width="240" height="42"/>',
        '<text x="39" y="120" class="tokens" fill="#fff">',
        symbol0,
        "/",
        symbol1,
        "</text>"
        '<rect x="30" y="132" width="240" height="30"/>',
        '<text x="39" y="120" dy="36" class="fee" fill="#fff">',
        feeToText(fee),
        "</text>"
    );
}
```

Fees are rendered as numbers with a fractional part. Since all possible fees are known in advance we don't need to convert integers to fractional numbers and can simply hardcode the values:
```solidity
function feeToText(uint256 fee)
    internal
    pure
    returns (string memory feeString)
{
    if (fee == 500) {
        feeString = "0.05%";
    } else if (fee == 3000) {
        feeString = "0.3%";
    }
}
```

In the bottom part, we render position ticks:
```solidity
function renderBottom(int24 lowerTick, int24 upperTick)
    internal
    pure
    returns (string memory bottom)
{
    bottom = string.concat(
        '<rect x="30" y="342" width="240" height="24"/>',
        '<text x="39" y="360" class="tick" fill="#fff">Lower tick: ',
        tickToText(lowerTick),
        "</text>",
        '<rect x="30" y="372" width="240" height="24"/>',
        '<text x="39" y="360" dy="30" class="tick" fill="#fff">Upper tick: ',
        tickToText(upperTick),
        "</text>"
    );
}
```

Since ticks can be positive and negative, we need to render them properly (with or without the minus sign):
```solidity
function tickToText(int24 tick)
    internal
    pure
    returns (string memory tickString)
{
    tickString = string.concat(
        tick < 0 ? "-" : "",
        tick < 0
            ? Strings.toString(uint256(uint24(-tick)))
            : Strings.toString(uint256(uint24(tick)))
    );
}
```

### JSON Rendering

Now, let's return to the `render` function and render the JSON. First, we need to render a token description:
```solidity
function render(RenderParams memory params) {
    ... SVG rendering ...

    string memory description = renderDescription(
        symbol0,
        symbol1,
        params.fee,
        params.lowerTick,
        params.upperTick
    );

    ...
```

A token description is a text string that contains all the same information that we render in the token's SVG:
```solidity
function renderDescription(
    string memory symbol0,
    string memory symbol1,
    uint24 fee,
    int24 lowerTick,
    int24 upperTick
) internal pure returns (string memory description) {
    description = string.concat(
        symbol0,
        "/",
        symbol1,
        " ",
        feeToText(fee),
        ", Lower tick: ",
        tickToText(lowerTick),
        ", Upper text: ",
        tickToText(upperTick)
    );
}
```

We can now assemble the JSON metadata:
```solidity
function render(RenderParams memory params) {
    string memory image = ...SVG rendering...
    string memory description = ...description rendering...

    string memory json = string.concat(
        '{"name":"Uniswap V3 Position",',
        '"description":"',
        description,
        '",',
        '"image":"data:image/svg+xml;base64,',
        Base64.encode(bytes(image)),
        '"}'
    );
```

And, finally, we can return the result:

```solidity
return
    string.concat(
        "data:application/json;base64,",
        Base64.encode(bytes(json))
    );
```

### Filling the Gap in `tokenURI`

Now we're ready to return to the `tokenURI` function in the NFT manager contract and add the actual rendering:

```solidity
function tokenURI(uint256 tokenId)
    public
    view
    override
    returns (string memory)
{
    TokenPosition memory tokenPosition = positions[tokenId];
    if (tokenPosition.pool == address(0x00)) revert WrongToken();

    IUniswapV3Pool pool = IUniswapV3Pool(tokenPosition.pool);

    return
        NFTRenderer.render(
            NFTRenderer.RenderParams({
                pool: tokenPosition.pool,
                owner: address(this),
                lowerTick: tokenPosition.lowerTick,
                upperTick: tokenPosition.upperTick,
                fee: pool.fee()
            })
        );
}
```

# Gas Costs

With all its benefits, storing data on-chain has a huge disadvantage: contract deployments become very expensive. When deploying a contract, you pay for the size of the contract, and all the strings and templates increase gas spending significantly. This gets even worse the more advanced your SVGs are: the more there are shapes, CSS styles, animations, etc. the more expensive it gets.

Keep in mind that the NFT renderer we implemented above is not gas optimized: you can see the repetitive `rect` and `text` tag strings that can be extracted into internal functions. I sacrificed gas efficiency for the readability of the contract.  In real NFT projects that store all data on-chain, code readability is usually very poor due to heavy gas cost optimizations.

# Testing

The last thing I wanted to focus on here is how we can test the NFT images. It's very important to keep all changes in NFT images tracked to ensure no change breaks rendering. For this, we need a way to test the output of `tokenURI` and its different variations (we can even pre-render the whole collection and have tests to ensure no image gets broken during development).

To test the output of `tokenURI`, I added this custom assertion:

```solidity
assertTokenURI(
    nft.tokenURI(tokenId0),
    "tokenuri0",
    "invalid token URI"
);
```

The first argument is the actual output and the second argument is the name of the file that stores the expected one. The assertion loads the content of the file and compares it with the actual one:

```solidity
function assertTokenURI(
    string memory actual,
    string memory expectedFixture,
    string memory errMessage
) internal {
    string memory expected = vm.readFile(
        string.concat("./test/fixtures/", expectedFixture)
    );

    assertEq(actual, string(expected), errMessage);
}
```

We can do this in Solidity thanks to the `vm.readFile()` cheat code provided by the `forge-std` library, which is a helper library that comes with Forge. Not only this is simple and convenient, but this is also secure: we can configure filesystem permissions to allow only permitted file operations. Specifically, to make the above test work, we need to add this
`fs_permissions` rule to `foundry.toml`:
```toml
fs_permissions = [{access='read',path='.'}]
```

And this is how you can read the SVG from a `tokenURI` fixture:
```shell
$ cat test/fixtures/tokenuri0 \
    | awk -F ',' '{print $2}' \
    | base64 -d - \
    | jq -r .image \
    | awk -F ',' '{print $2}' \
    | base64 -d - > nft.svg \
    && open nft.svg
```

> Ensure you have [jq tool](https://stedolan.github.io/jq/) installed.