---
layout: post
title: TLSH, Hamming codes, hashes, and S3 buckets
summary: How to efficiently find things that are close to one another.
categories: math
tags: distance metric hash
---

We have a dataset that requires a bit of exploratory analysis at scale in order
to determine meaningful questions to ask of it. Essentially, we realize that
there is a wealth of information contained in the dataset, but we are unsure
what sorts of analysis to perform on it and what sorts of questions might be
interesting to ask.

Consequently, what would be really nice would be to "slice" the data along
several axes and look at data that is "grouped" together. The typical
approach for something like this is $$k$$-means, which unfortunately is a bit
lacking in interpretability and requires choosing the parameter $$k$$.
Furthermore, I'd like to support the following use case: a data "explorer"
receives a new record containing fields such as geolocation, phone number,
date, and most recent Twitter post.

The explorer (dropping the quotes, now) wants to find other records that occur
in the same geographic region; that share the same phone number; that occur
within 5 days of the record; or that share similar text. It is also possible
that the explorer wants to find records that share similar text occuring
within 5 days of the record as well (so conjunctions across fields are also a
possibility). The point of this exercise is to locate similar records (along
one or more axes) in order for a *human* to make informed decisions about what
to do next.

While $$k$$-means would let the explorer look at record groupings, it does
little to empower the explorer to formulate informed or meaningful questions;
it is much more ideal for the human to slice the data along different axes in
order to determine more meaningful questions to ask.

The key to solving this problem is to embed each field into a metric space. A
*metric space* is a set $$S$$ along with a function $$d(x,y)$$ that computes a
*distance* between any two points $$x, y \in S$$. The function $$d : S \times S
\to \mathbf{R}$$ has several properties:

* it is nonnegative for all $$x, y \in S$$,
* it is symmetric $$d(x,y) = d(y,x)$$,
* it yields zero if and only if $$x = y$$, and, finally,
* it satisfies the triangle inequality $$d(x,y) + d(y,z) \geq d(x,z)$$.

By embedding each field into the metric space, we can arbitrarily pick an
"origin" and create a total ordering on our fields. This total ordering allows
us to sort our data, and it's much easier to search over sorted data than over
unsorted data.

Searching for "similar" items for a specific field, then, is just a matter of
finding the items location on a number line and (linearly) retrieving items
"nearby." To do this efficiently, we might bin each axis according to some
efficiently compute key, for instance, the leading digits of $$d(x,0)$$. Then,
finding similar items amounts to looking up the bin and performing a (linear)
search over the items in that bin.

When searching over multiple fields, we would look at two different buckets and
take the intersection of the buckets and perform a (linear) search over the
remaining items. This allows for efficient retrieval of similar rectangles.

### For numerical data
The distance metric is pretty easy, $$d(x,y) = \| x - y \|_2$$.

### For categorical data
Distance metrics probably don't make sense in this space. It might just be

$$
d(x,y) = \begin{cases}
0 & x = y \\
1 & \mbox{otherwise}
\end{cases}
$$

(You'll have to convince yourself that this is actually a metric.)

### For text data
Text data is the most interesting one since there isn't a natural, meaningful
metric for it. We're going to go with the Hamming distance of TLSH. In other
words, we perform an extra step where text is mapped to a set of hashes and the
Hamming distance between the hashes is used in the metric space.

## So have you tried this?
No. But I want to.

Concentration of Gaussian measures  http://www.math.lsa.umich.edu/~barvinok/total710.pdf

prob (|d(Pi x, Pi y) - d(x,y)| < epsilon)
