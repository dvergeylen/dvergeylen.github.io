---
layout: post
series_title: "Code Snippets"
toc_title: "Javascript (ES6+)"
title:  "[Javascript] Useful code snippets"
date:   2020-11-26 12:30:00 +0200
tag: snippets
permalink: /javascript-code-snippets
---

#### Introduction
Here are common Javascript patterns I use all the time, mainly in Svelte apps.
Functional patterns are always easier to reason about and less error prone. You should always favor such patterns, when possible.

I have written the Ruby equivalent [here](/ruby-code-snippets).

### Code Snippets

##### Filter
```javascript
ar = [1,2,3,4,5,6]
ar.filter(elem => (elem % 2) == 0)
// ↳ [2, 4, 6]
```

##### Map
```javascript
ar = [1,2,3,4,5,6]
ar.map(elem => (elem * 2))
// ↳ [2, 4, 6, 8, 10, 12]
```

##### Reduce
```javascript
// Reduce
// trailing '0' is accumulator initial value (ALWAYS set one!)
ar.reduce((acc, elem) => acc + elem, 0)
ar = [1,2,3,4,5,6]
// ↳ 21

// Reduce is also useful with hashes...
hash = {a: 'b', c: 'd', e: 'f'}
Object.keys(hash).reduce((acc, key) => acc = {...acc, [key]: hash[key]}, {})
// ↳ { a: 'b', c: 'd', e: 'f' }

// ... or grouping by (unknown) keys
ar_h = [{key1: 'b'}, {key2: 'a'}, {key1: 'd'}, {key2: 'c'}]
ar_h.reduce((acc, elem) => ({
  ...acc,
  ...Object.keys(elem).reduce((elemAcc, key) => ({
    ...elemAcc,
    [key]: (elemAcc[key] ? elemAcc[key] + elem[key] : elem[key])
    }), {}),
}), {})
// ↳ {:key1=>"bd", :key2=>"ac"}
```

##### Sort
```javascript
// Sort takes 2 args
ar = [1,2,3,4,5,6]
ar.sort((a,b) => b - a)
// ↳ [6, 5, 4, 3, 2, 1]
```

##### Find
```javascript
// Find
ar_h = [{mykey: 'b'}, {mykey: 'a'}, {mykey: 'd'}, {mykey: 'c'}]
ar_h.find((elem) => elem['mykey'] == 'd')
// ↳ {mykey: 'd'}
```

##### Use of concurrent Promises
A common misunderstanding is that "`node` is single threaded" which is incorrect. `node` uses a single Event loop (`libuv`) yes, but powered by a worker pool (4, by default). More information can be found in [this blog post](https://medium.com/@joydipand/how-does-thread-pool-work-in-node-js-c48f3b3662a9).

This means it's useful to handle multiple asynchronous operations in parallel, via Promises. Never do this in `.forEach()` statements, though!

```javascript
// forEach
ar.forEach((elem) => {
  // do stuff...
  // ... but no async function!
  // Because forEach doesn't return something
  // Use map() instead (see below)
})

// Promise Example
// Notice that if at least 1 promise fails,
// Promise.all will fail as well.
// ALWAYS wrap Promises in try / catch blocks
(async () => {
  try {
    const ar = [1, 2, 3, 4, 5, 6];
    const promises = ar.map(async (elem) => {
      await new Promise((resolve, reject) => setTimeout(resolve, 1000));
      return elem;
    });
    const results = await Promise.all(promises);
    console.log(results);
  } catch(e) {
    console.log(e);
  }
})();
```

##### Optional chaining:
```javascript
// Javascript Optional Chaining:
// use ?. to navigate safely accross objects:
// <p>The item name is: ${item?.name}</p>
```