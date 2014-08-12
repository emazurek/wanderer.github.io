---
layout: post
title:  "How to run contracts and creating stack traces with Node.js"
date:   2014-05-21 00:00:00
categories: ethereum nodejs code
comments: true
---

<img src="https://i.imgur.com/8jug5Cc.jpg"/>
> A valid state transition is one which comes about through a transaction. Formally:
> σ<sub>t</sub>t ≡ Υ(σ<sub>t</sub> , T )
> where Υ is the Ethereum state transition function. In ☰thereum, Υ, together with σ are considerably more powerful then any existing comparable system; Υ allows components to carry out arbitrary computation, while σ allows components to store arbitrary state between transactions.


At the core of Etheruem is the state transition function.  Within the this function lies the ☰thereum ∇irtual Machine which upon ☰∇M code runs. In this post we will be running transactions thourgh the VM and looking at the results. A note on Terminology: the VM as descibed in the yellow paper is a subset of the state transition function. With an executioner that manipulates accounts and sets up the enviorment for the VM. Here we will just refer to the state transition function as the VM although this may not be techicanally correct. I did not like having executioner in my code. It's a bit to barbaric for my taste.  Altogether the VM in the sense handles all changes done to the state, which wholly resided in a trie.

This post will explore creating transaction and processing them with the VM. You can find all the code for this post [here](https://github.com/wanderer/ethereum-lib-node/blob/master/examples/vm.js);

# Running Contracts
To get stated we will be using two libraries. `async` and [`ethereum-lib`](https://github.com/wanderer/ethereum-lib-node)
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
    rlp = Ethereum.rlp;

//creating a trie that just resides in memory
var stateTrie = new Trie();

//create a new VM instance
var vm = new VM(stateTrie);

//we will use this later
var storageRoot;
{% endhighlight %}

Lets set up the state trie. We need to give the account which is sending the transaction enougth wei to send transaction and run the code.

{% highlight javascript %}
//sets up the initial state and runs  the callback when complete
function setup(cb) {
    //the address we are sending from
    var address = new Buffer('9bdf9e2cc4dfa83de3c35da792cdf9b9e9fcfabd', 'hex');
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
var rawTx = ['00',
    '09184e72a000',
    '2710',
    '0000000000000000000000000000000000000000',
    '00',
    '7f4e616d65526567000000000000000000000000000000000000000000000000003057307f4e616d6552656700000000000000000000000000000000000000000000000000577f436f6e666967000000000000000000000000000000000000000000000000000073661005d2720d855f1d9976f88bb10c1a3398c77f5773661005d2720d855f1d9976f88bb10c1a3398c77f7f436f6e6669670000000000000000000000000000000000000000000000000000573360455760c75160c46000396000f2007f72656769737465720000000000000000000000000000000000000000000000006000350e0f604859602035560f6032590033560f603d596000335657336020355760203533570060007f756e7265676973746572000000000000000000000000000000000000000000006000350e0f6076595033560f6084596000335657600033570060007f6b696c6c000000000000000000000000000000000000000000000000000000006000350e0f60b55950604556330e0f60bb5933ff6000355660005460206000f2',
    '1b',
    '1da35136b9e0a791450628096b10bc51792aa1c21117c518d81d34aa032c23ff',
    '5d818440711a8b234ffea2bcb32932aa011fc87889725afd9d62c079f84e6ea5'
];
{% endhighlight %}

Since running a transaction in the VM is asynchronous lets wrap this in a function which we will use later. Running this transaction should create a new name register contract

{% highlight javascript %}
//runs a transaction through the vm
function runTx(raw, cb) {
  
    //create a new transaction out of the raw json
    var tx = new Transaction(raw);
    
    //run the tx
    vm.runTx(tx, function (err, results) {
        var createdAddress = results.createdAddress;
        //log some results 
        console.log('gas used: ' + results.gasUsed.toString());
        if (createdAddress) console.log('address created: ' + createdAddress.toString('hex'));
        cb(err);
    });
}
{% endhighlight %}

`vm.runTx` 1) checks if the sending `account` has enough wie to run then 2) runs the code and lastly 3) saves the changed accounts in the  trie. It uses a callback to returns the results of running the transaction. We are only going to log some of the results here but if you are curious george you can look at the (docs)[https://github.com/wanderer/ethereum-lib-node/blob/master/docs/VM.md#vmruntxtx-block-cb]

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
    var createdAddress = new Buffer('c8b97e77d29c5ccb5e9298519c707da5fb83c442', 'hex');
    
    //fetch the new account from the trie.
    stateTrie.get(createdAddress, function (err, val) {

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

And run. We should see the details of the new created contract. Next thing we can do is send a message to our newly created contract. Lets create another transaction to do so. This transaction should register the sending address as "null_radix"

{% highlight javascript %}
var rawTx2 = ['01',
    '09184e72a000',
    '2710',
    'c8b97e77d29c5ccb5e9298519c707da5fb83c442',
    '00',
    '72656769737465720000000000000000000000000000000000000000000000006e756c6c5f726164697800000000000000000000000000000000000000000000',
    '1b',
    'a1f22689203a3303552b5360c28f7153ac31de94ad4db8ef80307acd02a9951d',
    '3522808a9024b67a0b3ba800536c8b06d09fab134c8d7a15486960ef073f0c40'
];

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
    stream.on('data', function (data) {
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

# Debugging
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

Now when you run you should see a complete trace. `onStep` provides an object that contians all the information on the current state of the `VM`. And now I'm out of stuff to say.
