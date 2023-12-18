# Deployment

Alright, our pool contract is done. Now, let's see how we can deploy it to a local Ethereum network so we could use it from a front-end app later on.

## Choosing Local Blockchain Network

Smart contracts development requires running a local network, where you deploy your contracts during development and testing. This is what we want from such a network:
1. Real blockchain. It must be a real Ethereum network, not an emulation. We want to be sure that our contract will work in the local network exactly as it would in the mainnet.
1. Speed. We want our transactions to be minted immediately, so we could iterate quickly.
1. Ether. To pay transaction fees, we need some ether, and we want the local network to allow us to generate any amount of ether.
1. Cheat codes. Besides providing the standard API, we want a local network to allow us to do more. For example, we want to be able to deploy contracts at any address, execute transactions from any address (impersonate other address), change contract state directly, etc.

There are multiple solutions as of today:
1. [Ganache](https://trufflesuite.com/ganache/) from Truffle Suite.
1. [Hardhat](https://hardhat.org/), which is a development environment that includes a local node besides other useful things.
1. [Anvil](https://github.com/foundry-rs/foundry/tree/master/anvil) from Foundry.

All of these are viable solutions and each of them will satisfy our needs. Having said that, projects have been slowly migrating from Ganache (which is the oldest of the solutions) to Hardhat (which seems to be the most widely used these days), and now there's the new kid on the block: Foundry. Foundry is also the only of these solutions that uses Solidity for writing tests (the others use JavaScript). Moreover, Foundry also allows to write deployment scripts in Solidity.  Thus, since we've decided to use Solidity everywhere, we'll use Anvil to run a local development blockchain, and we'll write deployment scripts in Solidity.

## Running Local Blockchain

Anvil doesn't require configuration, we can run it with a single command and it'll do:

```shell
$ anvil --code-size-limit 50000
                             _   _
                            (_) | |
      __ _   _ __   __   __  _  | |
     / _` | | '_ \  \ \ / / | | | |
    | (_| | | | | |  \ V /  | | | |
     \__,_| |_| |_|   \_/   |_| |_|

    0.1.0 (d89f6af 2022-06-24T00:15:17.897682Z)
    https://github.com/foundry-rs/foundry
...
Listening on 127.0.0.1:8545
```

> We're going to write big contracts that don't fit into the Ethereum contract size limit (which is `24576` bytes), thus we need to tell Anvil to allow bigger smart contracts.

Anvil runs a single Ethereum node, so this is not really a network, but that's ok. By default, it creates 10 accounts with 10,000 ETH in each of them. It prints the addresses and related private keys when it starts–we'll be using one of these addresses when deploying and interacting with the contract from UI.

Anvil exposes JSON-RPC API interface at `127.0.0.1:8545`–this interface is the main way of interacting with Ethereum nodes. You can find full API reference [here](https://ethereum.org/en/developers/docs/apis/json-rpc/). And this is how you can call it via `curl`:
```shell
$ curl -X POST -H 'Content-Type: application/json' \
  --data '{"id":1,"jsonrpc":"2.0","method":"eth_chainId"}' \
  http://127.0.0.1:8545
{"jsonrpc":"2.0","id":1,"result":"0x7a69"}
$ curl -X POST -H 'Content-Type: application/json' \
  --data '{"id":1,"jsonrpc":"2.0","method":"eth_getBalance","params":["0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266","latest"]}' \
  http://127.0.0.1:8545
{"jsonrpc":"2.0","id":1,"result":"0x21e19e0c9bab2400000"}
```

You can also use `cast` (part of Foundry) for that:
```shell
$ cast chain-id
31337
$ cast balance 0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266
10000000000000000000000
```

Now, let's deploy the pool and manager contracts to the local network.

## First Deployment

At its core, deploying a contract means:
1. Compiling source code into EVM bytecode.
1. Sending a transaction with the bytecode.
1. Creating a new address, executing the constructor part of the bytecode, storing initialized bytecode on the address.  This step is done automatically by an Ethereum node, when your contract creation transaction is mined.

Deployment usually consists of multiple steps: preparing parameters, deploying auxiliary contracts, deploying main contracts, initializing contracts, etc. Scripting helps to automate these steps, and we'll write scripts in Solidity!

Create `scripts/DeployDevelopment.sol` contract with this content:
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.14;

import "forge-std/Script.sol";

contract DeployDevelopment is Script {
    function run() public {
      ...
    }
}
```

It looks very similar to the test contract, with only difference is that it inherits from `Script` contract, not from `Test`. And, by convention, we need to define `run` function which will be the body of our deployment script. In the `run` function, we define the parameters of the deployment first:
```solidity
uint256 wethBalance = 1 ether;
uint256 usdcBalance = 5042 ether;
int24 currentTick = 85176;
uint160 currentSqrtP = 5602277097478614198912276234240;
```
These are the same values we used before. Notice that we're about to mint 5042 USDC–that's 5000 USDC we'll provide as liquidity into the pool and 42 USDC we'll sell in a swap.

Next, we define the set of steps that will be executed as the deployment transaction (well, each of the steps will be a separate transaction). For this, we're using `startBroadcast/endBroadcast` cheat codes:
```solidity
vm.startBroadcast();
...
vm.stopBroadcast();
```

> These cheat codes are [provided by of Foundry](https://github.com/foundry-rs/foundry/tree/master/forge#cheat-codes).  We got them in the script contract by inheriting from `forge-std/Script.sol`.

Everything that goes after the `broadcast()` cheat code or between `startBroadcast()/stopBroadcast()` is converted to transactions and these transactions are sent to the node that executes the script.

Between the broadcast cheat codes, we'll put the actual deployment steps. First, we need to deploy the tokens:
```solidity
ERC20Mintable token0 = new ERC20Mintable("Wrapped Ether", "WETH", 18);
ERC20Mintable token1 = new ERC20Mintable("USD Coin", "USDC", 18);
```
We cannot deploy the pool without having tokens, so we need to deploy them first.

> Since we're deploying to a local development network, we need to deploy the tokens ourselves. In the mainnet and public test networks (Ropsten, Goerli, Sepolia), the tokens are already created. Thus, to deploy to those networks, we'll need to write network-specific deployment scripts.

The next step is to deploy the pool contract:
```solidity
UniswapV3Pool pool = new UniswapV3Pool(
    address(token0),
    address(token1),
    currentSqrtP,
    currentTick
);
```

Next goes Manager contract deployment:
```solidity
UniswapV3Manager manager = new UniswapV3Manager();
```

And finally, we can mint some amount of ETH and USDC to our address:
```solidity
token0.mint(msg.sender, wethBalance);
token1.mint(msg.sender, usdcBalance);
```

> `msg.sender` in Foundry scripts is the address that sends transactions within the `broadcast` block. We'll be able to set it when running scripts.

Finally, at the end of the script, add some `console.log` calls to print the addresses of deployed contracts:
```solidity
console.log("WETH address", address(token0));
console.log("USDC address", address(token1));
console.log("Pool address", address(pool));
console.log("Manager address", address(manager));
```

Alright, let's run the script (ensure Anvil is running in another terminal window):
```shell
$ forge script scripts/DeployDevelopment.s.sol --broadcast --fork-url http://localhost:8545 --private-key $PRIVATE_KEY  --code-size-limit 50000
```

> We're increasing the smart contract code size again so that the compiler doesn't fail.

`--broadcast` enables broadcasting of transactions. It's not enabled by default because not every script sends transactions. `--fork-url` sets the address of the node to send transactions to. `--private-key` sets the sender wallet: a private key is needed to sign transactions. You can pick any of the private keys printed by Anvil when it's starting. I took the first one:
> 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80

Deployment takes several seconds. In the end, you'll see a list of transactions it sent. It'll also save transactions receipts to `broadcast` folder. In Anvil, you'll also see many lines with `eth_sendRawTransaction`, `eth_getTransactionByHash`, and `eth_getTransactionReceipt`–after sending transactions to Anvil, Forge uses the JSON-RPC API to check their status and get transaction execution results (receipts).

Congratulations! You've just deployed a smart contract!

## Interacting With Contracts, ABI

Now, let's see how we can interact with the deployed contracts.

Every contract exposes a set of public functions. In the case of the pool contract, these are `mint(...)` and `swap(...)`.  Additionally, Solidity creates getters for public variables, so we can also call `token0()`, `token1()`, `positions()`, etc.  However, since contracts are compiled bytecodes, **function names are lost during compilation and not stored on blockchain**. Instead, every function is identified by a selector, which is the first 4 bytes of the hash of the signature of the function. In pseudocode:
```
hash("transfer(address,address,uint256)")[0:4]
```

> EVM uses [the Keccak hashing algorithm](https://en.wikipedia.org/wiki/SHA-3), which was standardized as SHA-3.  Specifically, the hashing function in Solidity is `keccak256`.

Knowing this, let's make two calls to the deployed contracts: one will be a low-level call via `curl`, and one will be made using `cast`.

### Token Balance
Let's check the WETH balance of the deployer address. The signature of the function is `balanceOf(address)` (as defined in [ERC-20](https://eips.ethereum.org/EIPS/eip-20)). To find the ID of this function (its selector), we'll hash it and take the first four bytes:
```shell
$ cast keccak "balanceOf(address)"| cut -b 1-10
0x70a08231
```

To pass the address, we simply append it to the function selector (and add left padding up to 32 digits since addresses take 32 bytes in function call data):
> 0x70a08231000000000000000000000000f39fd6e51aad88f6f4ce6ab8827279cfffb92266

`0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266` is the address we're going to check balance of. This is our address, the first account in Anvil.

Next, we execute `eth_call` JSON-RPC method to make the call. Notice that this doesn't require sending a transaction–this endpoint is used to read data from contracts.

```shell
$ params='{"from":"0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266","to":"0xe7f1725e7734ce288f8367e1bb143e90bb3f0512","data":"0x70a08231000000000000000000000000f39fd6e51aad88f6f4ce6ab8827279cfffb92266"}'
$ curl -X POST -H 'Content-Type: application/json' \
  --data '{"id":1,"jsonrpc":"2.0","method":"eth_call","params":['"$params"',"latest"]}' \
  http://127.0.0.1:8545
{"jsonrpc":"2.0","id":1,"result":"0x00000000000000000000000000000000000000000000011153ce5e56cf880000"}
```

> The "to" address is the USDC token. It's printed by the deployment script and it can be different in your case.

Ethereum nodes return results as raw bytes, to parse them we need to know the type of a returned value. In the case of `balanceOf` function, the type of a returned value is `uint256`. Using `cast`, we can convert it to a decimal number and then convert it to ethers:
```shell
$ cast --to-dec 0x00000000000000000000000000000000000000000000011153ce5e56cf880000| cast --from-wei
5042.000000000000000000
```

The balance is correct! We minted 5042 USDC to our address.

### Current Tick and Price

The above example is a demonstration of low-level contract calls. Usually, you never do calls via `curl` and use a tool or library that makes it easier. And Cast can help us here again!

Let's get the current price and tick of a pool using `cast`:
```shell
$ cast call POOL_ADDRESS "slot0()"| xargs cast --abi-decode "a()(uint160,int24)"
5602277097478614198912276234240
85176
```

Nice! The first value is the current $\sqrt{P}$ and the second value is the current tick.

> Since `--abi-decode` requires full function signature we have to specify "a()" even though we only want to decode function output.

### ABI

To simplify interaction with contracts, Solidity compiler can output ABI, Application Binary Interface.

ABI is a JSON file that contains the description of all public methods and events of a contract. The goal of this file is to make it easier to encode function parameters and decode return values. To get ABI with Forge, use this command:

```shell
$ forge inspect UniswapV3Pool abi
```

Feel free skimming through the file to better understand its content.