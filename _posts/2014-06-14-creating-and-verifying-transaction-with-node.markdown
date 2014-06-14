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

```javascript

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

