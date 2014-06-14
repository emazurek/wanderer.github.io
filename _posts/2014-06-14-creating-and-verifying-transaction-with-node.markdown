---
layout: post
title:  "Creating Ethereum contracts and verifying messages with node.js"
date:   2014-06-14 23:39:25
categories: ethereum
comments: true
---

# Creating Ethereum contracts and verifying messages with node.js

In this article I'll be covering ethereum transaction creation and verifying transactions. First of all; whats the rational for creating transaction with with javascript? Well I, as always just want to explore and play around with ethereum. But I imagine this would be usefull in testing compilers and even more usefull, create contracts programmaticall. So lets get started! :D

First of all I recently pulled out all the core function of [node-ethereum](https://github.com/josephyzhou/node-ethereum), and some code from [ethereumjs-lib](https://github.com/ethereum/ethereumjs-lib) to create ~~Frankenstein~~ [ethereum-node-lib](https://github.com/wanderer/ethereum-node-lib). I haven't put it on npm yet as things are still to much in flux so you will have To install it directly from git `npm install git+https://github.com/wanderer/ethereum-node-lib`

After that let create a new file and put the following in it. For the those how are cut and paste handicap here is the [full exmaple](https://github.com/wanderer/ethereum-node-lib/blob/master/examples/transactions.js).  

```
var Ethereum = require("ethereum-lib");
var Transaction = Ethereum.Transaction;

//create a blank transaction
var tx = new Transaction();


// So now we have created a blank transaction but Its not quiet valid yet. We
// need to add some things to it. Lets start with 

tx.nonce = 0;
tx.gasPrice = 100;
tx.gasLimit = 1000;
tx.to = 0;
tx.value = 0;
tx.data = "7f4e616d65526567000000000000000000000000000000000000000000000000003057307f4e616d6552656700000000000000000000000000000000000000000000000000573360455760415160566000396000f20036602259604556330e0f600f5933ff33560f601e5960003356576000335700604158600035560f602b590033560f60365960003356573360003557600035335700";
```
hopefull this is pretty self explainitory. nonce is the number of transaction that the sending acount has sent. gasPrice is the amount you are will to pay for gas gasAmount if the amount of gas you are will to spend value is the is the amount you are sending and data is well data. the data is the Name Registrar contract. You can use one of the compilers to to generate that. The two main ones seem to be lllc wich is part of cpp-ethereum and serptant. Ok now we have a transaction with data, now we need to sign it. To do this we will need a privat key. There are mutilpe way to aquire one but for now we are just going to steal one from ethereum-cpp that has a few ether in it. That way I know that I'm create valid transaction that actully has the ether that this transaction is saying that it has. If you have althazero running. You can select tools>export select key, and the copy the private key out. Here is mine.

![exporting private key](https://i.imgur.com/N0S4q3l.png) 

```
var privateKey = new Buffer("e331b6d69882b4cb4ea581d88e0b604039a3de5967688d3dcffdd2270c0fd109");
tx.sign(privateKey);
```

We have a signed transaction, Now for it to be total the account that we signed it with needs to have a certain amount of wei in to. To see how much that this account needs we can use the getTotalFee

```
console.log("Total Amount of wei needed:" + tx.getTotalFee());
```

this gives the amount in wei that the account needs to have. if your wondering how that is caculated it is
//data lenght in bytes * 5
// + 500 Default transaction fee
// + gasAmount * gasPrice

lets seriliaze the transaction

```
console.log("---Serialized TX----");
console.log(tx.serialize().toString("hex"));
console.log("--------------------");
```

Now that we have the serialized transaction we can get AlethZero to except by selecting debug>inject transaction and pasting the transaction serialization and it should show up in pending transaction.

![inject transaction](https://i.imgur.com/YPEkMTx.png) 



## Parsing & Validating transactions
If you have a transaction that you want to verify you can parse it. If you got it directly from the network it will be rlp encoded. You can decode you the rlp module. After that you should have something like

```
var rawTx =  [
        "00",
        "09184e72a000",
        "2710",
        "0000000000000000000000000000000000000000",
        "00",
        "7f7465737432000000000000000000000000000000000000000000000000000000600057",
        "1c",
        "5e1d3a76fbf824220eafc8c79ad578ad2b67d01b0c2425eb1f1347e8f50882ab",
        "5bd428537f05f9830e93792f90ea6a3e2d1ee84952dd96edbae9f658f831ab13"
    ];

var tx = new Transaction(rawTx);
```

Note rlp.decode will actully produce an array of buffers `new Transaction` will take either and array of buffers or and array of hex strings. So assuming that you were able to parse the tranaction, we will now get the sender's address

```
console.log("Senders Address" + tx.getSenderAddress());
```

Cool now we know who sent the tx! Lets verfy the signuate to make sure it not some poser.

```
if(tx.verifySignature()){
    console.log("Signature Checks out!");
}
```

And hopefull its verified. For the transaction to be tottal valid we would  also need to check the account of the sender and see if they have at least  `TotalFee`. 

