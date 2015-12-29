---
layout: post
title:  "Merklizing ASTs"
date:   2015-12-29 13:22:09
categories: ethereum ast webassembly
comments: true
---
Special thanks to [Jaun Benet](http://juan.benet.ai/) for mentioning this idea to me at Devcon 1. I recently wrote a draft of [EIP 105](https://github.com/ethereum/EIPs/issues/48) which propose using a subset of Webassemble as Ethereum’s VM. If you aren’t aware Webassemble  “is a new, portable, size- and load-time-efficient format suitable for compilation to the web.”   One interesting note about Webassemble doesn’t compile to linear byte code. Instead it uses an  [Abstract Syntax Tree (AST)](https://en.wikipedia.org/wiki/Abstract_syntax_tree). This might not surprise you if you have an experience with LLVM IR. But for me it was a new concept. 

So how does this AST look like?

>Each node [of the AST] represents an expression. Each function body consists of exactly one expression. All expressions and operators are typed, with no implicit conversions or overloading rules.

To give you a better idea here is a textual representation of an AST using [s-expressions](https://en.wikipedia.org/wiki/S-expression).

```
  ;; Recursive factorial
  (func $factorial (param $i i64) (result i64)
    (if_else (i64.eq (get_local $i) (i64.const 0))
      (i64.const 1)
      (i64.mul (get_local $i) (call $factorial (i64.sub (get_local $i) (i64.const 1))))))
```

Which could also be displayed like

![AST example](https://cdn.rawgit.com/wanderer/wanderer.github.io/master/_posts/images/Merklizing%20ASTs.svg)
figure 1 - an AST

This can then be [serialized](https://github.com/WebAssembly/design/blob/master/BinaryEncoding.md#serialized-ast) and sent across the wire.

## Why AST?
Well the [rationale that Webassemble](https://github.com/WebAssembly/design/blob/master/Rationale.md) gives is 

>Why not a stack-, register- or SSA-based bytecode?
* Trees allow a smaller binary encoding: [JSZap][], [Slim Binaries][].
* [Polyfill prototype][] shows simple and efficient translation to asm.js.

  [JSZap]: https://research.microsoft.com/en-us/projects/jszap/
  [Slim Binaries]: https://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.108.1711
  [Polyfill prototype]: https://github.com/WebAssembly/polyfill-prototype-1

Some auxiliary reason might be:
* Effective to JIT 
* Code Deduplication

The idea has a bit of an interesting history. It appears that [Michael Franz](http://www.michaelfranz.com/) first used the idea to compress java bytecode in a paper on Slim Binaries. The slim binaries were also implemented in the [Oberon OS](https://en.wikipedia.org/wiki/Oberon_(operating_system))

In addition to the Slim Binaries paper here are some more papers if you are interested in the subject.
* Adaptive Compression of Syntax Trees andIterative Dynamic Code Optimization:Two Basic Technologies for Mobile-Object Systems ftp://ftp.cis.upenn.edu/pub/cis700/public_html/papers/Franz97b.pdf
* A Tree-Based Alternative to Java Byte-Codes ftp://ftp.cis.upenn.edu/pub/cis700/public_html/papers/Kistler96.pdf

## Merkle ASTs
So another fun thing to do with AST is to merklize them. To review; a [merkle tree](https://en.wikipedia.org/wiki/Merkle_tree) is just a tree which links its nodes together by using the cryptographic hashes of the nodes.  

![a merkle tree](https://upload.wikimedia.org/wikipedia/commons/9/95/Hash_Tree.svg)

The result is one root hash that points to the root node. This allows for efficient and secure verification of the contents of large data structures.  To turn a AST into Merkle AST all you have walk from the leaf nodes up to the root hashing each node along the path. 

## Efficiency 
If you look at figure one you can count 19 nodes. This is a fairly small program and the number of nodes in a larger program can add up fast. The computation time needed to merklize larger AST might start to add up rather fast. One method to increase efficiency would be to store entire subroutines in a single node and only create merkle tree branches at the point where the AST has a branch condition. For example see figure 3.

![](https://cdn.rawgit.com/wanderer/wanderer.github.io/master/_posts/images/Merklizing%20ASTs-grouping.svg)
figure 3 - nodes contain entire subroutines

In figure 3 the green blocks represent subroutines that are in a single node. Note how the `if else` still forms a block by itself since it is a branch condition. This also provide an nice opportunity for parallelization. The interpreter or JIT can run the first branch while the next branch is still being fetched.  

## Why Merkle ASTs?

Other reason is secure code. It would make DLL hijack impossible. Of course how would use DLL’s would probably be quite a bit different. Instead of open a dynamic link by a name you would reference the root hash of the routine you wanted to use. This way you have 100% confidence that you're getting the code that you want. Have you ever had a software problem that you googled and found that 100s of forum posts with the same problem as you but none of their solutions worked for you? This is maybe in part caused by the fact you computer is in a different configuration or ‘state’ then thiers. If we had secure merklized code this would a lot less common since the state of any given piece of software could be immediately determined or set by the root hash.

For one thing massive code duplication. How many version of the same library you have on your computer? Or how many time the same function and routine is duplicated through programs? I would assume quite a bit. 

Perhaps one the most convincing reason is bandwidth saving. 
![](https://cdn.rawgit.com/wanderer/wanderer.github.io/master/_posts/images/Merklizing%20ASTs-bandwidth.svg)
Let's say you have you already have the green nodes since they are common subroutines. You would only have to download the orange nodes. Where this bandwidth saving could be very important is things like ethereum light clients but also for general computation. As Web Pages begin to more and more resemble apps the larger their code size becomes. In the age of ephemeral webapps repeat code is downloaded many times. How many times do you think you have downloaded jquery? Couple this with a peer-to-peer distribution method like [IPFS](https://ipfs.io/) and I think you have a very efficient system.

