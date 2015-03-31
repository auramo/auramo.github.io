---
layout: post
title: Using keywords as functions for your own container in ClojureScript
date: 2015-04-01 10:00:00
---

In Clojure and ClojureScript, a common, maybe even idiomatic way to access values from a map is using this form:

```(:mykey mymap)```

We are using a keyword as the function to access mymap. What if I want to create my own container which can be accessed
in similar way? Python programmers can implement __getattr__or __getitem__ and
do similar things. ClojureScript is a powerful and flexible
language so I should be able to do that right?

Let's dig into the source code to see [what Keyword does](https://github.com/clojure/clojurescript/blob/4eebd45bd82f40c8e656d97ee996ed91c48a3ec5/src/cljs/cljs/core.cljs#L2778).

```
(deftype Keyword
...
IFn
  (-invoke [kw coll]
    (get coll kw))
  (-invoke [kw coll not-found]
    (get coll kw not-found))
...
```

Ok, so Keyword implenents IFn which makes it a callable function. There are two signatures which call ``get``. Let's [check that next](https://github.com/clojure/clojurescript/blob/4eebd45bd82f40c8e656d97ee996ed91c48a3ec5/src/cljs/cljs/core.cljs#L1567):

```
      (cond
        (implements? ILookup o)
        (-lookup ^not-native o k)
...
        (implements? ILookup o)
        (-lookup ^not-native o k not-found)
```

It turns out that get checks if the container (o) implements the ILookup protocol, it calls the ``-lookup`` methods on it. So what we need to do is create our own container type which implements that protocol. Let's try that out.

```
(deftype EntryWrapper [data-map]
  ILookup
  (-lookup
    [o k] [k (get data-map k)])
  (-lookup [coll k not-found] [k (if-let [val (k data-map)]
                                   val
                                   not-found)]))
```

This code creates a new type EntryWrapper. It's pretty stupid: it wraps an ordinary map and when it's ``-lookup`` methods are called, it delegates to the map it contains. But unlike ordinary map, the lookup returns a vector containing key and value (this is actually what find already does, but bare with me :)).

Now, because Keyword used get, and get used the ILookup methods, we should be able to fetch [key value] vectors like this:

```
(:mykey entry-wrapper-instance)
```


ILookup
https://github.com/clojure/clojurescript/blob/4eebd45bd82f40c8e656d97ee996ed91c48a3ec5/src/cljs/cljs/core.cljs#L4931
