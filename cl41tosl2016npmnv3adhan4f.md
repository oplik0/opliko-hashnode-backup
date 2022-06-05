## SEETF  2022 You Only Have One Chance Writeup

> This is a writeup for [SEETF 2022](https://play.seetf.sg/) which I participated in as a member of [DistributedLivelock](https://ctftime.org/team/187094) team. You can find my other writeups for this CTF [here](https://blog.opliko.dev/series/seetf-2022)

# Introduction

You Only Have One Chance is the first and the easiest (after the sanity check that was Bonjour) of smart contract challenges in this edition of SEETF.

#### Challenge description:
> Sometimes in life, you only have one chance. Your goal is to make isSolved() function returns true!

#### Challenge author: AtlanticBase

# Setup

The full guide to connecting to the environment can be found [here](https://github.com/Social-Engineering-Experts/ETH-Guide), but the TL;DR is that we need to install MetaMask, connect to the SEETF test network and create an account there, then get some funds via [their ETH faucet](http://awesome.chall.seetf.sg:40001/) and then finally connect to the challenge server with `nc` and following the steps there to deploy the contract.

To interact with the network and edit the code I found it easiest to use the [Remix IDE](https://remix.ethereum.org/) in the browser.

# What is our goal

In all smart contract challenges the goal is getting `isSolved()` function of the deployed smart contract to return `true`. The full code can be retrieved from the SEETF server for this challenge:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract YouOnlyHaveOneChance {
    uint256 public balanceAmount;
    address public owner;
    uint256 randNonce = 0;

    constructor() {
        owner = msg.sender;

        balanceAmount =
            uint256(
                keccak256(
                    abi.encodePacked(block.timestamp, msg.sender, randNonce)
                )
            ) %
            1337;
    }

    function isBig(address _account) public view returns (bool) {
        uint256 size;
        assembly {
            size := extcodesize(_account)
        }
        return size > 0;
    }

    function increaseBalance(uint256 _amount) public {
        require(tx.origin != msg.sender);
        require(!isBig(msg.sender), "No Big Objects Allowed.");
        balanceAmount += _amount;
    }

    function isSolved() public view returns (bool) {
        return balanceAmount == 1337;
    }
}
```
The condition here is quite simple - we need to get balance up to exactly `1337`. Presumably this can be done by using the `increaseBalance` function, but obviously it's not so easy.

While for Bojour running on production immediately was preferable, here I recommend starting off with a javascript VM that Remix even has selected by default. Just compile the contract and deploy it there:
![obraz.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1654462933381/IZ0cQAI4O.png align="center")

We can also see that the balance we start with is a *pseudo*-random (with the *pseudo* doing a lot of work here) number mod 1337 - so basically, some positive integer below our goal.
We could actually find the block timestamp and sender from the transaction hash we got, and the nonce is constant (since someone didn't get what a *nonce* is). But we don't need to do any of that. Recall the very beginning of the contract:
```solidity
uint256 public balanceAmount;
```
Yup, it's just public, and we can query it:
![obraz.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1654463225317/RdEtYihkg.png align="center")

So that problem is solved, we just need to increase the balance by 1337 minus whatever this value is.
Now all that remains is actually exploiting a "bug" in this contract.

# How to be small

The `increaseBalance` function has just three lines, with the last one doing what we need: increasing the balance by the value we provide. The issue are the first two: they are both checks against the sender.
```solidity
function increaseBalance(uint256 _amount) public {
    require(tx.origin != msg.sender);
    require(!isBig(msg.sender), "No Big Objects Allowed.");
    balanceAmount += _amount;
}
```

First one checks if the sender is the origin of the transaction. If you don't know what the difference between these two is just like me at the time, a quick google search away you have answers like [this stackexchange one](https://ethereum.stackexchange.com/a/1892). Origin in always the account that started the transaction, while the sender is directly the *thing* that called the function, which can be an account or a contract. So for them to be different we'll need to interact with the contract via another contract.

The second one is a call to another function that seems more complicated:
```solidity
function isBig(address _account) public view returns (bool) {
    uint256 size;
    assembly {
        size := extcodesize(_account)
    }
    return size > 0;
}
```
We have some call to assembly here, but fortunately, we don't even really need to know what this even means - we just need to find out what `extcodesize` is and how to get it to return `0`. Again, just a search away is [this stackexchange question](https://ethereum.stackexchange.com/questions/15641/how-does-a-contract-find-out-if-another-address-is-a-contract), the title of which already it clear what that can do:
![obraz.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1654464004321/2XIxYvT8c.png align="center")
So we basically need to be a contract and not be a contract?

Well, let's actually look at the [top answer](https://ethereum.stackexchange.com/a/15642/102053). It confirms that suspicion, but also gives us the solution to our paradox:
![obraz.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1654464100821/bRZHe23yp.png align="center")

So, we just need to call it from the constructor of a contract. This also explains the challenge name: a contract only gets one chance to exploit this one, since after it's created it will be blocked.

# Exploitation

Let's quickly write a contract that will exploit this issue. I just added it to the end of the file, but you can of course be more fancy with imports etc.
```solidity
contract Hack {
    constructor(address _target) {
        uint256 amount = 1337 - YouOnlyHaveOneChance(_target).balanceAmount();
        YouOnlyHaveOneChance(_target).increaseBalance(amount);
    }
}
```
These few lines even do the *oh so hard* job of finding out the balance and calculating the amount still needed for us.

After compiling it we can use Remix to deploy the contract with the address of our target:

![obraz.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1654464611787/n1Q1tyuP4.png align="center")

And we can see we succeeded:
![obraz.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1654464656965/mlBdIXUny.png align="center")

If you were using a VM to this point like me, all that's left is to switch the environment to `Injected Web3` and deploy our `Hack` contract with the address we were given from the server before. 

After waiting for that deployment to complete, all that remains is to grabbing the flag:

![nc connection to server which returns the flag: `SEE{s0m3t1me5_ch4nce5_4re_h4rd_t0_g3T}` ](https://cdn.hashnode.com/res/hashnode/image/upload/v1654464819686/GaPTaVg0e.png align="center")



> This is a writeup for [SEETF 2022](https://play.seetf.sg/) which I participated in as a member of [DistributedLivelock](https://ctftime.org/team/187094) team. You can find my other writeups for this CTF [here](https://blog.opliko.dev/series/seetf-2022)