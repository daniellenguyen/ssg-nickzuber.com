---
layout: post
title: "Rolling Hash Data Structure and Applications"
date: 2016-07-24 17:24:13 -0400
permalink: blog/rolling-hash-data-structure-and-applications
use_math: true
---

Let's say you were given a massive text document and were alloted the task of finding a particular sentence somewhere within it. This sentence may or may not actually _exist_ inside of this document, but it's your job to try and find it anyways.

## Basic Approaches

The naive approach to this problem might be to take the sentence as a string, and compare it against every substring of the same size within the text document. It's pretty clear that this solution is far from optimal -- given that the text document is of size $n$ and the target string is of size $k$, this algorithm would have a asymptotic run time of $O(n \cdot k)$. 

```
// Let D be equal to the text document
// Let K be equal to the substring we're looking for
for n := 0 to n - k
	if K == D.substring(n, n + k)
		return true
return false
```

We can make this approach more practical by introducing hashing. So, let's refer to our target string as the _key_ that we're trying to find within our document. Given a hash function $h(S)$, we define the hashed value of the _key_ as $h_k$. Now, since the _key_ is of size $k$, we will define a subset of the document as a _window_ named $w$ that starts at the $n^{th}$ character and has a length of $k$ such that $w_n=\sum_{i=n}^{k}\ s_n$. From here, we need to compare $h_{w_n}$ and $h_k$ for all $n\ \|\ 0 \leq n \leq n-k$ so that we end up with a set of _potential_ matches $P=\\{p\in S\ \|\ h_{w_n}=h_k\\}$. Assuming that we have a relatively good hash function, we can generally expect $\| P \|={1 \over \| b \|}$ such that $b$ is the image of $h(S;x)$. 

```
// Let D be equal to the text document
// Let K be equal to the substring we're looking for
// Let h be a relatively good hash function
Hk := h(K)
for n := 0 to n - k
	if Hk == h(D.substring(n, n + k))
		if K == D.substring(n, n + k)
			return true
return false
```

As we traverse the text document looking for the key, each time we shift the window and compute a new hash value to compare to the key, we're spend $O(k)$ time doing so due to the cost of recomputing the hash. If we do encounter a situation where $h_{w_n}=h_k$, we need to compare the two strings to ensure that we've indeed found a match and not a false positive -- this process still takes $O(k)$ (remember that comparing strings is expensive since what's actually going on under the hood is $k$ character comparisons). As long as the number of collisions is relatively low (which we've determined it is, given that we have a good hash function), then the entire process of finding the key within the text document is $O(n \cdot k)$.

So at this point, it might be hard to see where the optimization comes in -- even though we've traded the cost of string comparisons for number comparisons, we've consequently added the complexity of calculating and recalcuating hash values each time we want to compare the window hash to our key hash. This doesn't seem to be any different from the original non-hashing approach (because it _isn't_), but this gives us an excellent segue into the rolling hash data structure.


## The Rolling Hash

Imagine we were given the same problem as before, and took the hashing approach. If we were able to rehash the window in $O(1)$ time, our algorithm's overall runtime would drop from $O(n \cdot k)$ down to $O(n)$ -- which is a _significant_ improvement. We can actually accomplish this constant rehashing goal by using the [rolling hash](https://en.wikipedia.org/wiki/Rolling_hash){:target="_blank"} data structure. 

To put it simply, a rolling hash is a data structure which contains the hashed value of some sequence of elements, and is able to update that hashed value by either appending the hashed value of any single new element or by removing the hashed value of a single existing element within the hashed sequence. You can think of this process as _hot loading_ but for a hashed value. With that being said, let's lay out the abstract structure of what a rolling hash should look like -- and keep in mind that these properties will all make sense once we start to unveil how it works:

```
operations:
	skip(o: element)
	append(n: element)
	slide(o: element, n: element)
	hash(s: [ element ])
	set(s: [ element ])

internal properties:
	state: number
	base: number
	cache: number
	inverseBase: number
	modularOffset: number
```

The `state` is the one and only internal variable that holds the current value of the hashed sequence. We typically refer to the sequence being hashed as the _window_. So our main goal here is to try and figure out how we can shift the window through the entire text document, recomputing the hash in constant time. 

## Hashing Collections Efficiently

First, to formally define a _shift_ within the window, we're talking about taking some window `w`, such that $w \subset S$ where $S$ is defined as the sequence that we are performing the rolling hash operations on. We then go on to further define $w$ as its own sequence, starting at the $n^{th}$ character of $S$ and has a length of $k$ such that $w_n=\sum_{i=n}^{k}\ S_n$. A _shift_, or _slide_, operation on $w$ can be defined as an function $G(w;d)=w_{n+d}$ where $d$ is a fixed parameter that holds the value $-1$ or $+1$ respectively, depending on the direction you're trying to shift $w$.

Naturally, a shift operation would imply that we need to do two things; removing an element from $w$ and adding an element to $w$. Let's visualize this process with an example; instead of using strings, which would typically be base 256, we're going to use a collection of base 10 numbers just because that's easier to work with mathematically:

```
// The collection that we want to apply our rolling hash to.
S := [1, 2, 3, 4, 5, 6]

// Our window which contains a sub sequence of S.
// It can be any size, but we chose 3 here for the example. 
w := [1, 2, 3]
```

We want our hashing function to first convert the sequence it's given into a number. So we do something like this:

```
// Our hash function
//  - S: sequence (ex. [1, 2, 3])
//  - b: base     (ex. 10)
HASH(S, b)
	r := 0
	for i := S.length - 1 down to 0
		r: = r + (S[i] * b^i)
	return r;

// Hash our current window
wh := HASH(w) // => 123
```

So we've concluded that the hash of our window is $h(w_n)=123$, therefore, using the same hashing function, we compute the hash value of the next window in our sequence $w_{n+1}=[2, 3, 4]$ which comes out to be $h(w_{n+1})=234$. Remember that our goal is to find a way to covert $h(w_n)$ to $h(w_{n+1})$, so we basically need to find a way to convert `123` into `234`.

To put into even simpler terms, we need to get rid of the `1` from `123` and then add a `4`. So to remove the `1`, we need to subtract $123 - (1 \cdot 10^2) = 23$, which translates to $h(w_n)-(x \cdot b^{k-1})$ such that $b$ represents the base of the elements of $w$, $x$ being the element we're trying to remove, and $k$ is the size or amount of digits in base $b$ within $w_n$. Now, adding the `4` is trivial -- we just need to multiply $w_n$ by $b$ and then just _add_ `4`. This would look like $(23 \cdot 10) + 4$, which translates to $(h(w_n) \cdot b) + y$, where $y$ represents the element we're adding.

This helps us with building our sliding operation on $w$; we know that if we want to get from $h(w_n)$ to $h(w_{n+1})$, we need to perform the following operation on $w$: $h(w_{n+1})=(h(w_n)-(x \cdot b^{k-1})) \cdot b + y$ which can be reduced down to $h(w_{n+1})=h(w_n) \cdot b - x \cdot b^k + y$.



<!--
// talk about next window (slide result) and its hash and how they compare
// step through how we pop and append, and derive a formula
// bring in modulo and why its important and adjust formula
// write out constructor, append, pop algorithms
// ...
-->
