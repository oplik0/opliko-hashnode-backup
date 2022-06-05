## SEETF 2022 Bonjour Writeup

> This is a writeup for [SEETF 2022](https://play.seetf.sg/) which I participated in as a member of [DistributedLivelock](https://ctftime.org/team/187094) team. You can find my other writeups for this CTF [here](https://blog.opliko.dev/series/seetf-2022)

# Introduction

Bonjour is an introductory smart contract challenge that's just meant to check if you are able to use the system and interact with smart contracts.

# setup

The full guide to connecting to the environment can be found [here](https://github.com/Social-Engineering-Experts/ETH-Guide), but the TL;DR is that we need to install MetaMask, connect to the SEETF test network and create an account there, then get some funds via [their ETH faucet](http://awesome.chall.seetf.sg:40001/) and then finally connect to the challenge server with `nc` and following the steps there to deploy the contract.

To interact with the network and edit the code I found it easiest to use the [Remix IDE](https://remix.ethereum.org/) in the browser.

# what is our goal

In all smart contract challenges the goal is getting `isSolved()` function of the deployed smart contract to return `true`. In this case the conditions are easy to see when looking at the code:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Bonjour {

  string public welcomeMessage;

  constructor() {
    welcomeMessage = "Bonjour";
  }

  function setWelcomeMessage(string memory _welcomeMessage) public {
    welcomeMessage = _welcomeMessage;
  }

  function isSolved() public view returns (bool) {
    return keccak256(abi.encodePacked("Welcome to SEETF")) == keccak256(abi.encodePacked(welcomeMessage));
  }
}
```

However, if you didn't have contact with solidity before just like me, you might be a bit confused about the weird `keccak256` and `abi.encodePacked` usage - don't worry, it's just how that language does string comparaison. So this:
```solidity
keccak256(abi.encodePacked("Welcome to SEETF")) == keccak256(abi.encodePacked(welcomeMessage));
```
would be just `"Welcome to SEETF" == "Welcome to SEETF"` in most languages...

But well, this isn't time to complain about language design - let's get back to work. Since this is an introductory challenge the solution is quite trivial. We literally have a function to set the only variable to anything we want.

What we now need it just contract address, which we are given after deployment:
![logs from contract deployment with address (0xe3a06E50dDbEeFEe8804C1a1f4934746E090ccA0) and transaction hash](https://cdn.hashnode.com/res/hashnode/image/upload/v1654461656272/iWiZE0wtR.png align="center")

# solution

And now we can use the Remix GUI to interact with it. Just remember to compile our contract and set the environment to "Injected web3". Paste the address into the box next to "At address" button and we get what we need:
![Part of Remix IDE GUI with an "at address" button next to which the contract address from the previous step was pasted and a deployed contracts tab, where we can see a Bonjour contract and all functions it exposes with their parameters as input boxes](https://cdn.hashnode.com/res/hashnode/image/upload/v1654461858384/3E58rHlhe.png align="center")

Let's then just input our desired string - `Welcome to SEETF` - into the box and send. 
![setWelcomeMessage box with a text input next to it, where "Welcome to SEETF" has been pasted](https://cdn.hashnode.com/res/hashnode/image/upload/v1654462033016/ObXwTyMxb.png align="center")

Let's just run a quick sanity check since we can...
![a isSolved button with `bool: true` result below it and `welcomeMessage` button with `string: Welcome to SEETF` result below it](https://cdn.hashnode.com/res/hashnode/image/upload/v1654462090104/NCGB80q9t.png align="center")

And all that remains is to grabbing the flag:
![nc connection to server which returns the flag: `SEE{W3lc0mE_t0_SEETF_a71cda2f322e7834169418a9d1a036a0}`](https://cdn.hashnode.com/res/hashnode/image/upload/v1654462165245/p6KBtK2CK.png align="center")

> This is a writeup for [SEETF 2022](https://play.seetf.sg/) which I participated in as a member of [DistributedLivelock](https://ctftime.org/team/187094) team. You can find my other writeups for this CTF [here](https://blog.opliko.dev/series/seetf-2022)