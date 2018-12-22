---
title: Mocking Ethereum contracts
author: Kobi
comments: true
---
Testing frameworks have become important for development processes. They allow us to instrument our code and make sure it handles different cases. Mocking frameworks make it even better - if you have dependencies on external factors, you can make your code believe those dependencies act in a specified way so you can check your code knows how to deal with the different responses. Additionally, as Alex and Roman mentioned in a chat we had, mocks can help you develop when you don't have the dependency ready, i.e. when someone else develops it and haven't finished.

For node.js, for example, we have Mocha as a testing framework a Sinon.JS as a mocking framework. This is a code example that shows the usage of these two together:
```
var assert = require('assert');
var sinon = require('sinon');
                                                                                                                                                                                                                                                                                                                                                                           
it("returns the return value from the original function", function () {
  var myAPI = {
    method: function () {
      return 5;
    }
  };
  var mock = sinon.mock(myAPI);
  mock.expects("method").returns(42);

  var result = myAPI.method();
  assert.equal(result, 42);
});
```

In this example, you take a function, `myAPI.method`, that usually returns `5`, and make it return `42`. This is useful when you want to see how your code behaves when, for example, `result` is larger than `30`.

A more complex example would be:
```
var assert = require('assert');
var sinon = require('sinon');
                                                                                                                                                                                                                                                                                                                                                                           
function once(fn) {
  var returnValue, called = false;
  return function () {
      if (!called) {
        called = true;
          returnValue = fn.apply(this, arguments);
      }
      return returnValue;
  };
}

it("returns the return value from the original function", function () {
  var myAPI = { 
    method: function () {
      return 5;
    } 
  };
  var mock = sinon.mock(myAPI);
  mock.expects("method").returns(42);

  var proxy = once(myAPI.method);

  assert.equals(proxy(), 42);
});
```

In this example, we don't call `myAPI.method` directly. Instead, `once` calls it. So although we didn't call it directly, the method was still mocked and returned `42` instead of `5`.

# Testing in Ethereum
[Ether.camp](https://live.ether.camp/){:target="_blank"} has been working on a pretty cool testing framework, utilizing the sandbox they developed for the excellent IDE, **Ethereum Studio**. You can see examples of it [here](https://github.com/ether-camp/ethereum-testing-reference){:target="_blank"}.

Let's look at a specific example. Let's say you wrote this simple math performing contract, **Math.sol**, using Solidity:
```javascript
contract Math {
  function sum(uint a, uint b) returns (uint result) {	    
    result = a + b;
  }

  function mul(uint a, uint b) returns (uint result) {	    
    result = a * b;
  }
	
  function sub(uint a, uint b) returns (uint result) {	    
    result = a - b;
  }

  function div(uint a, uint b) returns (uint result) {	    
    result = a / b;
  }
}
```

A simple test case would be:
```
var assert = require('assert');

var Workbench = require('ethereum-sandbox-workbench');
var workbench = new Workbench({
  defaults: {
    from: '0xcd2a3d9f938e13cd947ec05abc7fe734df8dd826'
  }
});
var sandbox = workbench.sandbox;

workbench.startTesting('Math', function(contracts) {
  it('sum-test', function() {
    return contracts.Math.new()
    .then(function(contract) {
      return contract.sum(2,2);
    })
    .then(function (txHash) {
      return workbench.waitForSandboxReceipt(txHash);
    })
    .then(function (receipt) {
      var result = sandbox.web3.toBigNumber(receipt.returnValue).toNumber();
      assert.equal(result, 4);
      assert.notEqual(result, 5);
    });
  });
});
```

This test case does the following:

1. Start a configured sandbox environment to which contracts can be deployed.
2. Deploys the **Math** contract to the sandbox.
3. Checks that addition actually works correctly in Ethereum. If you run this test case with Mocha, you'll be sure to see it does!

You probably noticed that after calling `startTesting` we have access to the compiled contracts with an easy-to-use, promise based, interface. This is thanks to [Ether Pudding](https://github.com/ConsenSys/ether-pudding){:target="_blank"}.

# Mocking in Ethereum
But what about mocking? When you're testing Ethereum contracts, it's more than a matter of modifying javascript functions. Ethereum contracts are pieces of code living on the Ethereum blockchain, accessible by sending transactions or calling their address.

For example, let's look at the following contract:
```
contract GoldPrice {function GoldPrice(); function price() constant returns(uint256 ); function notifyCallback(); function getPriceWithParameter(uint input) returns(uint);}

contract GoldPriceChecker {
  function GoldPriceChecker(address goldPriceContractAddress) {
    goldPriceContract = goldPriceContractAddress;
  }

  function getGoldPriceHappinessMeter() returns (uint) {
    uint feedPrice = GoldPrice(goldPriceContract).price();
    if (feedPrice > 200) {
      return 1;
    } else {
      return 2;
    }
  }

  address goldPriceContract;
}
```

This contract checks the price of gold and returns a "happiness meter" depending on the price. It depends on a **GoldPrice** contract, of which's address is provided in our contract's constructor. The interface of the **GoldPrice** contract is:
```
contract GoldPrice {
  function GoldPrice(); 
  function price() constant returns(uint256); 
  function notifyCallback(); 
  function getPriceWithParameter(uint input) returns(uint);
}
```

This means that if we were to mock an Ethereum contract, we would want an entity living on the blockchain that will respond instead of the mocked contract, but still adhering to the same interface.

# Enter proxy contract
A solution for this may be fulfilled a contract providing the following features:

1. Can be accessed with the same interface of the mocked contract.
2. Can be configured with desired responses to calls to mocked functions.
3. Can be configured with desired follow-up transactions to mocked functions.
4. Can notify when a mocked contract function was called.

In order to understand how to create such a contract, we need to understand how transactions and calls in Ethereum work.

# Ethereum Contract ABI
Let's say we want to call the Solidity function `getPriceWithParameter` with the parameter `5`. According to the [Ethereum Contract ABI](https://github.com/ethereum/wiki/wiki/Ethereum-Contract-ABI){:target="_blank"}, we first calculate the Method ID: first 4 bytes of `sha3("getPriceWithParameter(uint256)")`. In our case, it would be `0x60360e31`. Then, we encode the parameters. In our case, it would be `0x0000000000000000000000000000000000000000000000000000000000000005`.  Then, we concatenate them.

Ether Pudding and web3 provide us with easy-to-use javascript wrapper to do all of this automatically. That means that if we have an Ether Pudding instance of `GoldPrice` called `goldPrice` which was deployed at address `0xabcdefababcdefababcdefababcdefababcdefab`, then calling the following:
```
goldPrice.getPriceWithParaemeter(5, {
  from: '0xcd2a3d9f938e13cd947ec05abc7fe734df8dd826'
});
```

is equivalent to:
```
web3.eth.call({
  from: '0xcd2a3d9f938e13cd947ec05abc7fe734df8dd826',
  to: '0xabcdefababcdefababcdefababcdefababcdefab',
  data: '0x60360e310000000000000000000000000000000000000000000000000000000000000005'
});
```

In the Solidity code of the contract iself, it's possible to access the `data` using the globally available `msg.data`, which is of type `bytes`, and the method ID using `msg.sig`, which is of type `bytes4`.

# Proxy contract
Without further ado, I present you the Proxy contract:
```
contract Proxy {

  bool _traceFunctionCalls = false;
  event Trace(address caller, bytes data, uint value);

  function Proxy(bool traceFunctionCalls) {
    _traceFunctionCalls = traceFunctionCalls;
  }

  function setMock(bytes4 method, uint8 operationType, address target, bytes data) {
    operations[method] = Operation(operationType, target, data);
  }

  function setMockWithArgs(bytes methodWithData, uint8 operationType, address target, bytes data) {
    operationsWithArgs[methodWithData] = Operation(operationType, target, data);
  }

  function() {
    if (_traceFunctionCalls) {
      Trace(msg.sender, msg.data, msg.value);
    }
    Operation memory operation = operationsWithArgs[msg.data];
    uint8 operationType = operation.operationType;
    if (operationType == 0) {
      operation = operations[msg.sig];
      operationType = operation.operationType;
      if (operationType == 0) {
        throw;
      }
    } 

    if (operationType == 1) {
      address target = operation.target;
      target.call(operation.data);
    } else if (operationType == 2) {
      bytes memory data = operation.data;
      assembly {
        return(add(data, 32), mload(data))
      }
    } else {
      throw;
    }
  }

  struct Operation {
    uint8 operationType;
    address target;
    bytes data;
  }

  mapping (bytes4 => Operation) operations;
  mapping (bytes => Operation) operationsWithArgs;

}
```

As can be seen in the contract code, it has three functions:

1. `setMock` that allows you to set return values or a follow-up transaction when invoking the contract with the specified method ID (`msg.sig`).
2. `setMockWithArgs` that allows you to set return values or a follow-up transaction when invoking the contract with the specified method ID + arguments (`msg.data`).
3. Fallback function (`function () {}`) that is called when invoked with neither of the previous ones. The fallback function checks if a mock has been configured and acts accordingly.

The way the proxy contract works for mocking is by pointing the Ether Pudding interface of the mocked contract, i.e. `GoldPrice` at the address of a deployed proxy contract. Then, when functions are invoked on that instance, the `msg.data` of the resulting call will be an attempt to invoke the function of the `GoldPrice` contract on the proxy contract itself. 

Let's look at a concrete example. Let's assume we've deployed the proxy contract. Then, we invoked `setMock` with the following arguments: `setMock(0x60360e31, 2, 0x0, 0x0000000000000000000000000000000000000000000000000000000000000005)`. In other words, we've configured the proxy contract to return `5` when called with a `msg.sig` of `0x6036e31`. By pointing the Ether Pudding interface of `GoldPrice` at the deployed proxy contract, the resulting object being `goldPrice`, this is equivalent to calling `goldPrice.getPriceWithParameter` with any parameter.

Before I'll show you how to use it in the mocking framework, I want to point you to the following part in the proxy contract:
```
bytes memory data = operation.data;
assembly {
  return(add(data, 32), mload(data))
}
```

There's a bit of inline assembly here. This is required because we want to be able to return any type of value from the contract. That means that we don't want to limit ourselves to being able to make only functions that return, i.e., `uint256`. By using inline assembly and the `return` opcode, we are able force the contract to return with the bytes saved when the mock was configured. 

To understand how this works, we should look at how a `bytes` variable is stored in memory - first the number of bytes in the array padded to 32 bytes and then the data itself. 

The quoted code segment does the following:

1. Copies `operation.data` from storage to memory, by using the `memory` keyword. This is needed, because the `return` opcode returns from memory.
2. `mload` loads the first 32 bytes of the `data` variables. This would be the length of the array.
3. `add(data, 32)` is the address of the actual data in the `bytes` array.
4. `return(p,s)` itself terminates the contract execution and returns `s` bytes of data starting  in memory address `p`.  

To learn more about in-line assembly, check out the [Solidity documentation](https://solidity.readthedocs.io/en/latest/control-structures.html#inline-assembly){:target="_blank"}.

# Mocking framework
To make life easy, we added the mocking framework to the [ethereum-sandbox-workbench](https://github.com/ether-camp/ethereum-sandbox-workbench){:target="_blank"}. This is an example of working with it:
```
var assert = require('assert');
var Workbench = require('ethereum-sandbox-workbench');

var fromAddress = '0xcd2a3d9f938e13cd947ec05abc7fe734df8dd826';
var workbench = new Workbench({
  defaults: {
    from: fromAddress
  }
});

var sandbox = workbench.sandbox;

workbench.startTesting(['GoldPrice', 'GoldPriceChecker'], function(contracts) {
  var goldPrice;
  var mockContract;
  var goldPriceChecker;

  it('tests-deploy', function() {
    return contracts.GoldPrice.new()
    .then(function(contract) {
      if (contract.address) {
        goldPrice = contract;
      } else {
        throw new Error('No address for deployed contract');
      }
    });
  });

  it('tests-deploy-proxy', function() {
    return contracts.GoldPrice.newMock({
      traceFunctionCalls: true
    })
    .then(function(contract) {
      mockContract = contract;
      return mockContract.price.mockCallReturnValue(299);
    })
    .then(function(receipt) {
      return mockContract.price.call();
    })
    .then(function(value) {
      assert(value.equals(299));
    });
  });

  it('tests-deploy-checker', function() {
    return contracts.GoldPriceChecker.new(mockContract.address)
    .then(function(contract) {
      goldPriceChecker = contract;
      return goldPriceChecker.getGoldPriceHappinessMeter.call();
    })
    .then(function(value) {
      assert.equal(value.toString(), '1');
      return mockContract.price.mockCallReturnValue(5);
    })
    .then(function(receipt) {
      return goldPriceChecker.getGoldPriceHappinessMeter.call();
    })
    .then(function(value) {
      assert.equal(value.toString(), '2');
    });
  });
});
```

Let's recap what we see here:

1. To deploy a proxy contract but have the interface of the `GoldPrice` contract, we deploy using `newMock` instead of `new`.
2. To set-up a new mock for a call return value, we use `mockCallReturnValue` on the `price` function.
3. Then the `goldPriceChecker` which calls the `GoldPrice` contract in its code sees the mocked value.

Similarly, you can mock a transaction response using `mockTransactionForward` and trace function calls (making sure the functions were called in a specific transaction) using `wasCalled` on the mocked function. To see examples of those, check out the [test case](https://raw.githubusercontent.com/ether-camp/ethereum-testing-reference/master/test/mock/mock-test.js){:target="_blank"}!
