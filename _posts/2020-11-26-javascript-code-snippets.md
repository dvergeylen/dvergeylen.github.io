---
layout: post
title:  "Code snippets [Javascript]"
date:   2020-11-26 12:30:00 +0200
categories: programming javascript
permalink: /javascript-code-snippets
---

#### Introduction
Here are common javascript patterns I use all the time. I thought it would be useful to group them all in one place.

I have done the same for ruby: [see here](/ruby-code-snippets).

### Code Snippets
```javascript
ar = [1,2,3,4,5,6]
ar_h = [{key1: 'b'}, {key2: 'a'}, {key1: 'd'}, {key2: 'c'}]
hash = {a: 'b', c: 'd', e: 'f'}

// Filter
ar.filter(elem => (elem % 2) == 0)
// ↳ [2, 4, 6]

// Map
ar.map(elem => (elem * 2))
// ↳ [2, 4, 6, 8, 10, 12]

// Reduce
ar.reduce((acc, elem) => acc + elem, 0)
// ↳ 21

// Reduce is also useful with hashes...
Object.keys(hash).reduce((acc, key) => acc = {...acc, [key]: hash[key]}, {})
// ↳ { a: 'b', c: 'd', e: 'f' }

// ... or grouping by (unknown) keys
ar_h.reduce((acc, elem) => ({
  ...acc,
  ...Object.keys(elem).reduce((elemAcc, key) => ({
    ...elemAcc,
    [key]: (elemAcc[key] ? elemAcc[key] + elem[key] : elem[key])
    }), {}),
}), {})
// ↳ {:key1=>"bd", :key2=>"ac"}

// Sort (takes 2 args)
ar.sort((a,b) => b-a)
// ↳ [6, 5, 4, 3, 2, 1]

// Find
ar_h = [{mykey: 'b'}, {mykey: 'a'}, {mykey: 'd'}, {mykey: 'c'}]
ar_h.find((elem) => elem['mykey'] == 'd')
// ↳ {mykey: 'd'}

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
// Always wrap Promises in try / catch blocks
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


// Javascript Optional Chaining:
// use ?. to navigate safely accross objects:
// <p>The item name is: ${item?.name}</p>
```