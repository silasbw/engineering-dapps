# Engineering Decentralized Applications with Smart Contracts

An introduction and guide to the fundamentals of engineering
decentralized applications (dapps) to use smart contracts.

## Setup a network

We'll use an Ethereum network to run our smart contract programs. At a
high-level, executing a smart contract on an Ethereum network is
analogous to running a script on a virtual machine, or deploying your
web app to a set AWS services. Anyone can start an Ethereum network,
and unlike their centralized analogs, there are public Ethereum
networks that anyone can access and contribute computing power
to. All Ethereum networks generate ether that can be used in that
network.

We have several types of Ethereum networks we can use:

1. The mainnet (network ID 1). This is the main Ethereum network with many
   participants. This ether is valuable. The mainnet is similar to a
   production cluster.
2. A public testnet, like Rinkeby. Real miners and developers
   participate in this network, but the ether is worthless. Public
   testnets work well as staging environments before deploying your
   application to the Frontier mainnet.
3. A private testnet running on a local machine, or a small private
   cluster. Private testnets facilitate development because you have
   full control of the network.

### Private testnet

Private testnets are convient to develop with. There are roughly two
types of tools we can use to create a private testnet:

1. [an Ethereum client, like geth](https://github.com/ethereum/go-ethereum/wiki/Setting-up-private-network-or-local-cluster); or
2. [an Ethereum framework, like Truffle](http://truffleframework.com/docs/ganache/using).

Using clients you can configure a network of nodes that implement
Ethereum. Frameworks facilitate smart contract development and have
tools to simulate and control a network, and create a node connected
to that network.

To get started we'll use the
[Ganache](http://truffleframework.com/docs/ganache/using) from the
Truffle framework to setup our private testnet:

Install Ganache CLI and run it locally:

```
npm i ganache-cli --save-dev
node_modules/.bin/ganache-cli
```

`ganache-cli` does two things:

1. Create an Ethereum network with 10 transactions, each transaction
   mints 100 ether to a different account (represented by a public key).
2. Create an Ethereum node with access to the 10 accounts (access is
   granted via the private key) and particpating in the network. You
   can access this node locally via JSON-RPC at http://127.0.0.1:8545.

### Connect to the network

Your locally running Ethereum node acts as a gateway to the
testnet. You can access the node with an Ethereum library, like
[web3.js](https://github.com/ethereum/web3.js), or a client, like the
[Ethereum wallet](https://github.com/ethereum/mist/releases).

We'll use `web3.js`, but see \[[1]\] to access with the Ethereum
wallet.

Install `web3.js`:

```
npm i web3
```

and write some JavaScript to connect to your local node:

```js
const Web3 = require('web3');
const web3 = new Web3(new Web3.providers.HttpProvider('http://localhost:8545'));

async function main() {
  // Public keys for all the accounts the node manages.
  const accounts = await web3.eth.getAccounts();
  console.log(accounts);
});

main();
```

The `web3` instance implements the Ethereum API. We construct is
manually in this example, but in many frameworks or environments, the
runtime will construct it and inject into your JavaScript.

So far we're just using `web3.js` to print the accounts we have access
to, but the next step is use it to create a transaction.

## Access the blockchain

The Ethereum blockchain the testnet manages is responsible for storing
the state of the Ethereum network. We can read the blockchain using
our own compute power to read and verify it. Writing to the blockchain
is done with a transaction.  Transactions are more complicated to
create than reads (or "calls"), because you need to convince miners in
the network to expend their computing power to excute your transaction
and record the result.

### Execute a read/call

Write a function to print the balances of each account:

```js
async function printBalances() {
  const accounts = await web3.eth.getAccounts();
  accounts.forEach(async account => {
    const wei = await web3.eth.getBalance(account);
    const ether = web3.utils.fromWei(wei);
    console.log(`${ account }: wei ${ wei } ether ${ ether }`);
  });
}
```

### Execute a write/transaction

Our `web3` instance has some default values for funding the
transaction.

```js
async function transfer({ ether, from = 0, to = 1 }) {
  const wei = web3.utils.toWei(ether, 'ether');
  const accounts = await web3.eth.getAccounts();
  // https://web3js.readthedocs.io/en/1.0/web3-eth.html#sendtransaction
  const res = await web3.eth.sendTransaction({ accounts[from], accounts[to], value: wei });
}
```

## Develop a smart contract

We'll write our smart contracts in a the high-level programming
language
[Solidity](http://solidity.readthedocs.io/en/v0.4.21). Solidity
compiles to a bytecode that an Ethereum Virtual Machine (EVM) knows
how to execute. Each mining node in an Ethereum network runs an EVM.

We'll continue to use Truffle when developing our smart contract.

### Create a project

Create a project to help organize our smart contracts and traditional
code that might interact with them:

```
truffle init
```

and edit `truffle.js` to use our testnet:

```js
module.exports = {
  networks: {
    development: {
      host: '127.0.0.1',
      port: 8545,
      network_id: "*" // Match any network id
    }
  }
};
```

### Write a smart contract

Define a contract that allows anyone to set and get a number on the
blockchain. Creat it in `contracts/Storage.sol`:

```
pragma solidity ^0.4.18;

contract Storage {
  uint data;

  function set(uint value) public {
    data = value;
  }

  function get() public constant returns (uint) {
    return data;
  }
}
```

### Compile a smart contract

Compile your Solidity contracts:

```
truffle compile
```

Look at the results in `build/contracts`. Check out the EVM
bytecode.


### Deploying

Like deloying or migrating database schema changes:

http://truffleframework.com/docs/getting_started/migrations

```
truffle migrate
```

Try to access your Storage contract via the truffle console:

```
truffle(development)> Storage.deployed().then(hi => console.log(hi));
Error: Storage has not been deployed to detected network
(network/artifact mismatch)
    at
    /home/sbw/node_modules/truffle/build/webpack:/~/truffle-contract/contract.js:454:1
```

Oops, we need to deploy it. Let's use a migration for that.  Create
`migrations/2_deploy_contracts.js`:

```js
const Storage = artifacts.require('./Storage.sol');

module.exports = function(deployer) {
  deployer.deploy(Storage);
}
```

Migrate again:

```
truffle migrate
```

Now it should work.

## Writing your decentralized application (dapp)

### Call your smart contract

From scratch (see https://web3js.readthedocs.io/en/1.0/web3-eth-contract.html):

```js
const web3 = new Web3(
  new Web3.providers.HttpProvider('http://localhost:8545')
);
const data = require('./build/contracts/Storage.json');
const address = data.networks[Object.keys(data.networks)[0]].address;

const Storage = new web3.eth.Contract(data.abi, address);
Storage.methods.get().call().then(console.log);
```

With Truffle:

```js
const contract = require('truffle-contract');
const data = require('./build/contracts/Storage.json');
const Storage = contract(data);
Storage.setProvider(new Web3.providers.HttpProvider('http://localhost:8545'));

//
// See https://github.com/trufflesuite/truffle-contract/issues/56
//
Storage.currentProvider.sendAsync = function () {
  return Storage.currentProvider.send.apply(Storage.currentProvider, arguments);
};

async function printValue() {
  const instance = await Storage.deployed();
  console.log(instance);
  console.log((await instance.get()).toString());
}

printValue();
```

### Protect your Storage

Ensure only an account of your choosing can write to your Storage
contract:

```
contract Storage {
  uint data;
  address public owner;
  
  function Storage() public {
    owner = msg.sender;
  }
  
  function set(uint value) public {
    if (msg.sender != owner) return;
    data = value;
  }

  function get() public constant returns (uint) {
    return data;
  }
}
```

Redploy/rest:

```
truffle migrate --reset
```

### Events

http://solidity.readthedocs.io/en/v0.4.21/contracts.html#events

Declare a `Modified` event:

```
pragma solidity ^0.4.18;

//
// http://solidity.readthedocs.io/en/v0.4.21/contracts.html
//
contract Storage {
  uint data;
  address public owner;
  
  event Modified(
                 uint oldData,
                 uint newData
                 );

  function Storage() public {
    owner = msg.sender;
  }
  
  function set(uint value) public {
    if (msg.sender != owner) return;
    uint oldData = data;
    data = value;
    emit Modified(oldData, data);
  }

  function get() public constant returns (uint) {
    return data;
  }
}
```

Truffle compile and migrate.

Listen to `Modified` events:

```
async function listen() {
  const instance = await Storage.deployed();
  instance.Modified((err, res) => {
    if (err) return console.error(err);
    console.log(`${ res.args.oldData.toString() } -> ${ res.args.newData.toString }`);
  });
}
```

## Apendix

### Ethereum wallet

Use the [Ganache CLI](http://truffleframework.com/docs/ganache/using)
from the Truffle framework to start a testnet and a connected node:

[1]: #ethereum-wallet
