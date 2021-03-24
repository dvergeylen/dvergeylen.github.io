---
layout: post
series_title: "Code Snippets"
toc_title: "Ruby (2.7.0+)"
title:  "[Ruby] Useful code snippets"
date:   2020-11-26 12:30:00 +0200
tag: snippets
permalink: /ruby-code-snippets
---

#### Introduction
Here are common Ruby patterns I use all the time, mainly in Rails apps.
Working on already fetched data is far more efficient than doing multiple queries, especially if the number of entries is rather "small" (to be defined, depends on your application).

I avoid using `.group_by` as much as I can as it can be very slow, depending on what Ruby can eager load. Doing something like `User.last(1000).group_by(&:team_id)` is different from `User.last(1000).group_by(&:team)`, the former grouping wrt an integer, the latter with an object! The latter will be very slow as it doesn't eager load it and you will end up with an [N+1 problem](https://guides.rubyonrails.org/active_record_querying.html#eager-loading-associations).

I have written the Javascript equivalent [here](/javascript-code-snippets).

### Code Snippets

##### Filter
```ruby
ar = [1,2,3,4,5,6]
ar.select{|a| (a % 2) == 0}
# ↳ [2, 4, 6]
```

##### Map
```ruby
ar = [1,2,3,4,5,6]
ar.collect{|a| a*2}
# ↳ [2, 4, 6, 8, 10, 12]
```

##### Reduce
```ruby
# Reduce starts with an initial accumulator value (here 0).
# The block function is evaluated for each element of the iterator it
# is called on (in order) and must return a new accumulator value.
# The accumulator is an intermediate state shared among every block call.
# Accumulator is returned after after last block call.
ar = [1,2,3,4,5,6]
ar.reduce(0){|acc, a| acc + a}
# ↳ 21

# Reduce is also useful with hashes...
hash = {a: 'b', c: 'd', e: 'f'}
hash.reduce({}){|acc, (k, v)| acc.merge({k => v})
# ↳ {:a=>"b", :c=>"d", :e=>"f"}

# ... or grouping by (unknown) keys
ar_h = [{key1: 'b'}, {key2: 'a'}, {key1: 'd'}, {key2: 'c'}]
ar_h.reduce({}){|acc, elem|
  acc.merge(elem.collect{|k,v|
    [k, acc[k].nil? ? v : acc[k] + v]
  }.to_h)
}
# ↳ {:key1=>"bd", :key2=>"ac"}
```

##### Sort
```ruby
# Sort takes 2 args
ar = [1,2,3,4,5,6]
ar.sort{|a,b| b - a}
# ↳ [6, 5, 4, 3, 2, 1]

# Sort by (only takes 1 arg)
ar_h = [{mykey: 'b'}, {mykey: 'a'}, {mykey: 'd'}, {mykey: 'c'}]
ar_h.sort_by{|elem| elem[:mykey]}
# ↳ [{:mykey=>"a"}, {:mykey=>"b"}, {:mykey=>"c"}, {:mykey=>"d"}]
```

##### Find
```ruby
ar_h = [{key1: 'b'}, {key2: 'a'}, {key1: 'd'}, {key2: 'c'}]
ar_h.find{|elem| elem[:mykey] == 'd'}
# ↳ {mykey: 'd'}
```

##### Optional chaining
```ruby
# Ruby Safe navigation operator (optional chaining):
# use '&.' to navigate safely accross objects e.g:
@contract&.user&.name
```