---
layout: post
title:  "Code snippets [Ruby]"
date:   2020-11-26 12:30:00 +0200
categories: programming ruby
permalink: /ruby-code-snippets
---

#### Introduction
Here are common ruby patterns I use all the time. I thought it would be useful to group them all in one place.

I have done the same for javascript: [see here](/javascript-code-snippets).

### Code Snippets
```ruby
ar = [1,2,3,4,5,6]
ar_h = [{key1: 'b'}, {key2: 'a'}, {key1: 'd'}, {key2: 'c'}]
hash = {a: 'b', c: 'd', e: 'f'}

# Filter
ar.select{|a| (a % 2) == 0}
# ↳ [2, 4, 6]

# Map
ar.collect{|a| a*2}
# ↳ [2, 4, 6, 8, 10, 12]

# Reduce
ar.reduce(0){|acc, a| acc + a}
# ↳ 21

# Reduce is also useful with hashes...
hash.reduce({}){|acc, (k, v)| acc.merge({k => v})
# ↳ {:a=>"b", :c=>"d", :e=>"f"}

# ... or grouping by (unknown) keys
ar_h.reduce({}){|acc, elem|
  acc.merge(elem.collect{|k,v|
    [k, acc[k].nil? ? v : acc[k] + v]
  }.to_h)
}
# ↳ {:key1=>"bd", :key2=>"ac"}

# Sort (takes 2 args)
ar.sort{|a,b| b - a}
# ↳ [6, 5, 4, 3, 2, 1]

# Sort by (only takes 1 arg)
ar_h = [{mykey: 'b'}, {mykey: 'a'}, {mykey: 'd'}, {mykey: 'c'}]
ar_h.sort_by{|elem| elem[:mykey]}
# ↳ [{:mykey=>"a"}, {:mykey=>"b"}, {:mykey=>"c"}, {:mykey=>"d"}]

# Find
ar_h.find{|elem| elem[:mykey] == 'd'}
# ↳ {mykey: 'd'}

# Ruby Safe navigation operator (optional chaining):
# use '&.' to navigate safely accross objects e.g:
@contract&.user&.name
```