---
layout: post
title:  "Exploring Ethereum's state trie with Node.js"
date:   2014-05-21 13:22:09
categories: ethereum nodejs code
comments: true
---

If you not familiar with ethereum start [here](https://www.ethereum.org/). In this post I will attempt to read [cpp-ethereum's](https://github.com/ethereum/cpp-ethereum) state db give a root using Node.js and the [merkle-patricia-tree](https://github.com/wanderer/merkle-patricia-tree) module. The merkle-patricia-tree module is hot off the press so there might be bugs. If you find any please let me know. The State DB is like a global ledger; It stores balances of all the accounts was well as the code for all of the contracts and their respective balances.

The global state is stored in a modified merkle patriate tree (trie), which you can read more about in the [yellow paper](http://gavwood.com/Paper.pdf) or on the [wiki](https://github.com/ethereum/wiki/wiki/%5BEnglish%5D-Patricia-Tree). The important part is understanding what it is suppose to do, which is "The core of the trie, and its sole requirement in terms of the protocol specification is to provide a single 32-byte value that identifies a given set of key-value pairs."

## Reading the db

cpp-ethereum stores the trie in a level db. On linux the path to the state db is ~/.ethereum/state. Let try just opening the database and seeing what we get. To do this we are going to need install the levelup module `npm install levelup`

We are going to look up one of the state roots. You can get them from althezero. In the example I'm using the genesis root hash from the latest cpp code from the dev branch. Looking up the root should give the top node in the trie.

{% highlight javascript %}
var levelup = require('levelup');
var db = levelup('./.ethereum/state');

//the genesis state root
var root = '12582945fc5ad12c3e7b67c4fc37a68fc0d52d995bb7f7291ff41a2739a7ca16';

//Note: we are doing everything using binary encoding.
db.get(new Buffer(root, 'hex'), {
  encoding: 'binary'
}, function (err, value) {
  console.log(value);
});

{% endhighlight %}

We should get a print out of a Buffer... not necessarily usefully. Each node in the trie is `rlp` encoded, so let decoded the node. To do that we are going to need to install the [rlp module](https://github.com/wanderer/rlp) `npm install rlp`. and add it the code


{% highlight javascript %}
var rlp = require('rlp');

....

db.get(new Buffer(root, 'hex'), {
  encoding: 'binary'
}, function (err, value) {
  var decoded = rlp.decoded(value);
  console.log(decoded);
});

{% endhighlight %}

Yay we got one node! (hopefully). Now the node decoded should print out an array of buffers. In our case we should get a branch node with is an array of the length 17. 1-16 are more nodes and 17 is a value.  

## Looking up Values in the Trie

Now lets try to look up a key. After looking up the root node finding a key is as simple as following the key path to the right leaf. I'm skimming over the details, but you can read the specs [here](https://github.com/ethereum/wiki/wiki/%5BEnglish%5D-Patricia-Tree). First install `npm install merkle-patricia-tree`. This module allows you to read and manipulate the trie. Lets try looking up Gav's, our patron saint of cpp programming, account. His address is "8a40bfaa73256b60764c1bf40675a99083efb075".  

{% highlight javascript %}

var Trie = require('merkle-patricia-tree');
var rlp = require('rlp');
var levelup = require('levelup');
var db = levelup('/home/null/.ethereum/state');

//the genesis state root
var root = '12582945fc5ad12c3e7b67c4fc37a68fc0d52d995bb7f7291ff41a2739a7ca16';
var trie = new Trie(db, root);

//gav's address
var gav = new Buffer('8a40bfaa73256b60764c1bf40675a99083efb075', 'hex');

trie.get(gav, function (err, val) {
  var decoded = rlp.decode(val);
  console.log(decoded);
});

{% endhighlight %}

The accounts are rlp encoded before they are stored so we need to decoded them after we retrieve them. After decoding them we should get an array of the length 4. Which are the four fields in an account, `balance`, `nonce`, `stateRoot` and `codeHash` 

## Streaming the Trie

The Last thing we want to do is to read out the entire trie. This might be handy for debugging or just poking around for fun. To do this we are going to use `streams`. If your not familiar with node streams [here](http://nodeschool.io/#stream-adventure) is a really fun place to get stated. Here the on `data` event returns an `object`that has two propeties; the `key` and the `value`. Both should be buffers.

{% highlight javascript %}

var Trie = require('./index.js'),
  rlp = require('rlp'),
  levelup = require('levelup'),
  Sha3 = require('sha3');

var db = levelup('/home/null/.ethereum/state');
var root = "12582945fc5ad12c3e7b67c4fc37a68fc0d52d995bb7f7291ff41a2739a7ca16"
var trie = new Trie(db, root);

//get a read stream object
var stream = trie.createReadStream();

stream.on('data', function (data) {
  console.log('key:' + data.key.toString('hex'));

  //accouts are rlp encoded
  var decodedVal = rlp.decode(data.value);
  console.log(decodedVal);
});

stream.on('end', function (val) {
  console.log('done reading!');
});

{% endhighlight %}

This should read out all the account's address and the account's contents in the trie.

## The End?
[merkle-patricia-tree](https://github.com/wanderer/merkle-patricia-tree) module can do more than what I went over here. It can create, update and delete values too! amazing! And if you actually read all the way to the bottom, thanks for reading.

