---
layout: post
title:  "How to run contracts and create traces with Node.js"
date:   2014-08-12 00:00:00
categories: ethereum nodejs code
comments: true
---

<img src="https://i.imgur.com/8jug5Cc.jpg"/>
<blockquote>
A valid state transition is one which comes about through a transaction. Formally:
σ<sub>t+1</sub> ≡ Υ(σ<sub>t</sub> , T )
where Υ is the Ethereum state transition function. In ☰thereum, Υ, together with σ are <b>considerably more powerful then any existing comparable system;</b> Υ allows components to carry out arbitrary computation, while σ allows components to store arbitrary state between transactions. - The Yellow Paper
</blockquote>

At the core of Etheruem is the state transition function.  Within the this function lies the ☰thereum ∇irtual Machine which upon ☰∇M code runs. In this post we will be running transactions thourgh the VM and looking at the results. A note on Terminology: the VM as descibed in the yellow paper is a subset of the state transition function. With an executioner that manipulates accounts and sets up the enviorment for the VM. Here we will just refer to the state transition function as the VM although this may not be techicanally correct. I did not like having an executioner in my code. It's a bit too barbaric for my taste.  Altogether the VM in this sense handles all changes done to the state, which wholly resided in a trie.

This post will explore creating transactions and processing them with the VM. You can find all the code for this post [here](https://github.com/ethereum/ethereumjs-lib/blob/master/examples/vm.js);

## Running Contracts
To get stated we will be using two libraries. `async` and [`ethereum-lib`](https://github.com/ethereum/ethereumjs-lib)    
`npm install async`  
`npm install ethereum-lib`  

First import the necessary libraries and initailize some varibles.

{% highlight javascript %}
var async = require('async'),
    Ethereum = require('ethereum-lib'),
    VM = Ethereum.VM,
    Account = Ethereum.Account,
    Transaction = Ethereum.Transaction,
    Trie = Ethereum.Trie,
    rlp = Ethereum.rlp
    utils = Ethereum.utils;

//creating a trie that just resides in memory
var stateTrie = new Trie();

//create a new VM instance
var vm = new VM(stateTrie);

//we will use this later
var storageRoot;

//the private/public key pare. used to sign the transactions and generate the addresses
var secretKey = '3cd7232cd6f3fc66a57a6bedc1a8ed6c228fff0a327e169c2bcc5e869ed49511';
var publicKey = '0406cc661590d48ee972944b35ad13ff03c7876eae3fd191e8a2f77311b0a3c6613407b5005e63d7d8d76b89d5f900cde691497688bb281e07a5052ff61edebdc0';

{% endhighlight %}

Lets set up the state trie. We need to give the sender's account enougth wei to send the transaction and run the code.

{% highlight javascript %}
//sets up the initial state and runs  the callback when complete
function setup(cb) {
  //the address we are sending from
  var address = utils.pubToAddress(new Buffer(publicKey, 'hex'));

  //create a new account
  var account = new Account();

  //give the account some wei. 
  //This needs to be a `Buffer` or a string. all strings need to be in hex.
  account.balance = 'f00000000000000000';

  //store in the trie
  stateTrie.put(address, account.serialize(), cb);
}
{% endhighlight %}

Next we will create a transaction and process it. If you want to know more about generating transaction see this [post](https://wanderer.github.io/ethereum/2014/06/14/creating-and-verifying-transaction-with-node/) In this example we will forgo generating a transaction from scratch and just use the some json I generated earlier. `Transaction` have a `toJSON` method if you want to do the same. This transaction contains the initializtion code for the [name register](https://github.com/ethereum/dapp-bin/blob/master/namereg/namereg.lll).

{% highlight javascript %}
var rawTx = {
  nonce: '00',
  gasPrice: '09184e72a000',
  gasLimit: '2710',
  data: '7f4e616d65526567000000000000000000000000000000000000000000000000003055307f4e616d6552656700000000000000000000000000000000000000000000000000557f436f6e666967000000000000000000000000000000000000000000000000000073661005d2720d855f1d9976f88bb10c1a3398c77f5573661005d2720d855f1d9976f88bb10c1a3398c77f7f436f6e6669670000000000000000000000000000000000000000000000000000553360455560df806100c56000396000f3007f726567697374657200000000000000000000000000000000000000000000000060003514156053576020355415603257005b335415603e5760003354555b6020353360006000a233602035556020353355005b60007f756e72656769737465720000000000000000000000000000000000000000000060003514156082575033545b1560995733335460006000a2600033545560003355005b60007f6b696c6c00000000000000000000000000000000000000000000000000000000600035141560cb575060455433145b1560d25733ff5b6000355460005260206000f3'
};
{% endhighlight %}

Since running a transaction in the VM is asynchronous lets wrap this in a function which we will use later. Running this transaction should create a new name register contract. It is important to note that `vm.runTx()` would also need a `Block` a the second parameter if the EVM code where to use any opcodes that accessed the block propeties. 

{% highlight javascript %}
//runs a transaction through the vm
function runTx(raw, cb) {
  //create a new transaction out of the json
  
  var tx = new Transaction(raw);
  
  tx.sign(new Buffer(secretKey, 'hex'));

  //run the tx
  vm.runTx(tx, function(err, results) {
    var createdAddress = results.createdAddress;
    //log some results 
    console.log('gas used: ' + results.gasUsed.toString());
    if (createdAddress) console.log('address created: ' + createdAddress.toString('hex'));
    cb(err);
  });
}
{% endhighlight %}

`vm.runTx` 1) checks if the sending `account` has enough wei to run then 2) runs the code and lastly 3) saves the changed accounts in the  trie. It uses a callback to returns the results of running the transaction. We are only going to log some of the results here but if you are curious george you can look at the [docs](https://github.com/ethereum/ethereumjs-lib/blob/master/docs/VM.md#vmruntxtx-block-cb)

Now lets actully run this!

{% highlight javascript %}
//run the asynchronous functions in series
async.series([
    setup,
    async.apply(runTx, rawTx)
]);
{% endhighlight %}

So you should see how much gas the transaction used and the addrres of the created contract `c8b97e77d29c5ccb5e9298519c707da5fb83c442`.  But not too exciting yet. Lets make sure that things worked. The VM should have created a new account for the contract in the trie. Lets make sure.

{% highlight javascript %}
function checkResults(cb) {
  var createdAddress = new Buffer('692a70d2e424a56d2c6c27aa97d1a86395877b3a', 'hex');

  //fetch the new account from the trie.
  stateTrie.get(createdAddress, function(err, val) {

    var account = new Account(val);

    //we will use this later! :)
    storageRoot = account.stateRoot;

    console.log('------results------');
    console.log('nonce: ' + account.nonce.toString('hex'));
    console.log('blance in wei: ' + account.balance.toString('hex'));
    console.log('stateRoot: ' + storageRoot.toString('hex'));
    console.log('codeHash:' + account.codeHash.toString('hex'));
    console.log('-------------------');
    cb(err);
  });
}
{% endhighlight %}

Add this to `async.series` like so

{% highlight javascript %}
async.series([
    setup,
    async.apply(runTx, rawTx),
    checkResults
]);
{% endhighlight %}

And run. We should see the details of the new created contract. Next thing we can do is send a message to our newly created contract. Lets create another transaction to do so. This transaction should register the sending address as "null_radix". The data field here is the RLP of  pad_32_bytes('register') and  pad_32_bytes('null_radix')

{% highlight javascript %}
//This transaction should register the sending address as "null_radix"
var rawTx2 = {
  nonce: '01',
  gasPrice: '09184e72a000',
  gasLimit: '2710',
  to: '692a70d2e424a56d2c6c27aa97d1a86395877b3a',
  data: '72656769737465720000000000000000000000000000000000000000000000006e756c6c5f726164697800000000000000000000000000000000000000000000'
};

//run everything
async.series([
    setup,
    async.apply(runTx, rawTx),
    async.apply(runTx, rawTx2),
    checkResults
]);
{% endhighlight %}

So if everything went right we should have  "null_radix" stored at "0x9bdf9e2cc4dfa83de3c35da792cdf9b9e9fcfabd". To see this we need to print out the name register's storage trie.

{% highlight javascript %}
//reads and prints the storage of a contract
function readStorage(cb) {
  //we need to create a copy of the state root
  var storageTrie = stateTrie.copy();

  //Since we are using a copy we won't change the root of `stateTrie` 
  storageTrie.root = storageRoot;

  var stream = storageTrie.createReadStream();

  console.log('------Storage------');

  //prints all of the keys and values in the storage trie
  stream.on('data', function(data) {
    console.log('key: ' + data.key.toString('hex'));
    //remove the 'hex' if you want to see the ascii values
    console.log('Value: ' + rlp.decode(data.value).toString('hex'));
  });

  stream.on('end', cb);
}

//and finally 
//run everything
async.series([
    setup,
    async.apply(runTx, rawTx),
    async.apply(runTx, rawTx2),
    checkResults,
    readStorage
]);
{% endhighlight %}

After running you should see the contents of the contract's storage.

## Debugging with traces
Lastly lets create a trace of the EMV code. This can be very usefully for debugging (and deherbing, don't smoke and code m'kay?) contracts.

The VM provides a simple hook for each step the VM takes while running EVM code.

{% highlight javascript %}
//runs on each opcode
vm.onStep = function (info, done) {
    //prints the program counter, the current opcode and the amount of gas left 
    console.log('[vm] ' + info.pc + ' Opcode: ' + info.opcode + ' Gas: ' + info.gasLeft.toString());

    //prints out the current stack
    info.stack.forEach(function (item) {
        console.log('[vm]    ' + item.toString('hex'));
    });
    //important! call `done` when your done messing around
    done();
};
{% endhighlight %}

Now when you run you should see a complete trace. `onStep` provides an object that contians all the information on the current state of the `VM`. 

the `vm` also provides a `runBlock` function that will process a `block` as well as a few other usefull functions.
And now I'm out of stuff to say.
