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

Let's dig into the source code to see [what Keyword
does](https://github.com/clojure/clojurescript/blob/4eebd45bd82f40c8e656d97ee996ed91c48a3ec5/src/cljs/cljs/core.cljs#L2778).

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

Ok, so Keyword implenents IFn which makes it a callable
function. There are two signatures which call ``get``. Let's [check
that
next](https://github.com/clojure/clojurescript/blob/4eebd45bd82f40c8e656d97ee996ed91c48a3ec5/src/cljs/cljs/core.cljs#L1567):

```
      (cond
        (implements? ILookup o)
        (-lookup ^not-native o k)
...
        (implements? ILookup o)
        (-lookup ^not-native o k not-found)
```

It turns out that get checks if the container (o) implements the
[ILookup
protocol](https://github.com/clojure/clojurescript/blob/4eebd45bd82f40c8e656d97ee996ed91c48a3ec5/src/cljs/cljs/core.cljs#L391),
it calls the ``-lookup`` methods on it. So what we need to do is
create our own container type which implements that protocol. Let's
try that out.

```
(deftype EntryWrapper [data-map]
  ILookup
  (-lookup
    [o k] [k (get data-map k)])
  (-lookup [coll k not-found] [k (if-let [val (k data-map)]
                                   val
                                   not-found)]))
```

This code creates a new type EntryWrapper. It's pretty stupid: it
wraps an ordinary map and when it's ``-lookup`` methods are called, it
delegates to the map it contains. But unlike ordinary map, the lookup
returns a vector containing key and value (this is actually what find
already does, but bare with me :)).

Now, because Keyword used get, and get used the ILookup methods, we should be able to fetch [key value] vectors like this:

```
(def ew (EntryWrapper. {:a 1}))
(:a ew)
->
[:a 1]
```

Nice, but not that useful. Now let's try to do something more
interesting: a half-assed implementation of JavaScript-style
prototype-based inheritance.


```
(declare proto-get)

(deftype ProtoContainer [values proto]
  ILookup
  (-lookup
    [o k] (proto-get (with-meta values {::proto proto}) k nil))
  (-lookup [coll k not-found] (proto-get (with-meta values {::proto proto}) k not-found)))
```

Above we declared a function proto-get which we will implement
later. Then we implemented a new type ProtoContainer which again
implements the protocol ILookup. It also takes a map of values and a
prototype as it's "constructor parameters". All the interesting stuff
has been delegated to proto-get. Let's see what it does:


```
(defn proto-get [values k not-found]
  (if values
      (let [v (get values k)]
        (if (not (nil? v))
            v
          (recur (::proto (meta values)) k not-found)))
    not-found))
```

Nice and simple, it just uses the get function to retrieve a value
from the map of values. If it isn't there, the function calls itself
recursively with the prototype of our map.

```
(defn proto-container
  ([values] (ProtoContainer. values nil))
  ([values proto] (ProtoContainer. values proto)))
```

Now let's see if the inheritance works. _The following example is a variation
of an example from Steve Yegge's [Universal Design Pattern
article](http://steve-yegge.blogspot.fi/2008/10/universal-design-pattern.html)
and an example in [the Joy of Clojure book](http://www.joyofclojure.com/) in Chapter 9._

```
(def cat (proto-container {:likes-dogs true :likes-other-cats true}))
(def morris (proto-container {:name "Morris"} cat))
(:name morris)
-> "Morris"
(:likes-dogs morris)
-> true
```

Above we created a prototype ``cat`` and an instance of cat called
``morris``. In addition to the base properties ``:likes-dogs`` and
``:likes-other-cats`` Morris has a property called name.

Next Morris has an encounter with a nasty dog and starts hating dogs:

```
(def post-traumatic-morris (proto-container {:likes-dogs false} morris))
(:name post-traumatic-morris)
-> "Morris"
(:likes-dogs post-traumatic-morris)
-> false
(:likes-dogs morris)
-> true
```

As seen above, we were able to specialize a new Morris which has the
same properties as original Morris except for the not liking dogs
part.