# Web3eg-Smart-Contract-Auditor-Reentrancy-
- The provided Bank contract is vulnerable to a reentrancy attack. This vulnerability occurs because the contract updates the user's balance (balances[msg.sender] = 0) after sending Ether to the user. An attacker can exploit this by recursively calling the withdrawEth function before the balance is updated, draining the contract's funds.

```solidity
//SPDX-Liecense-Identifier: MIT
pragma solidity ^0.8.26;

contract Bank{

    mapping(address =>uint256) public balances;

    function depositEth() payable{
        balances[msg.sender] += msg.value;
        
    }

    function withdrawEth() public{
        uint amount = balances[msg.sender];
        (bool sent,) = msg.sender.call{value : amount}("");
        require(sent,"Failed to send Ether");

        balances[msg.sender] = 0 ;
    }
}
```

- Below, I'll explain how to exploit this vulnerability and provide an example of an attacker contract that performs the reentrancy attack.
  The withdrawEth function has the following vulnerable logic:
   It checks the user's balance: uint amount = balances[msg.sender];
   It sends Ether to the user: msg.sender.call{value: amount}("").
   It updates the user's balance to 0: balances[msg.sender] = 0.

