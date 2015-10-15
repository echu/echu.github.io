---
layout:     post
title:      A word on notation
summary:    A description of the notation this blog uses.
date:       2015-10-07
categories: intro
tags:       notation math programs code
---

When possible, I've tried to be consistent with my notation on this blog.

## For math
I use [MathJax][] to render variables ($$x$$, $$y$$, $$z$$, *etc*.), mathematical functions ($$f$$, $$g$$, $$h$$, *etc*.), and mathematical objects ($$\mathbf{R}^n$$, *etc*.). I adopt single letters for variables and function names and use **boldface** for typical domains, $$\mathbf{R}^n$$, $$\mathbf{Z}^n$$, and $$\mathbf{C}^n$$.

I *do not* use italics or arrows to distinguish vectors from scalars: these I (should) make clear by the inclusion in the proper set. For instance, a three element vector $$x$$ would be introduced as: $$x \in \mathbf{R}^3$$.

Finally, I use uppercase letters to denote matrices, $$A$$, $$B$$, $$C$$, *etc*.

## For code
When referring to a *binary* application, such as `redis-client`, I use [Markdown][] with backticks. I also use backticks to refer to a method, function, or variable in the code. For instance, I might reference `foobar` from the following snippet.

{% highlight python3 lineanchors %}
def foobar():
    print("hello")
{% endhighlight %}

For code literals, I will also use backticks, `3`, `"abc"`, *etc*.

I also use backticks to refer to paths on a filesystem, such as `/home/foobar`, and I use shell variable expansion `$YOUR_API_KEY` to refer to user-dependent environment variables.

Finally, I use long variable names for naming functions, classes, modules, *etc*. I do this even when naming variables corresponding to mathematical objects:

{% highlight python3 lineanchors %}
def matrix_multiply(matrix_A, vector_x):
    return matrix_A @ vector_x
{% endhighlight %}

## On functions
I try not to conflate a function *in math* with a function *in code*. These are subtly different in that a function *in code* can have side effects and causes it to behave differently from the usual function *in math*.

Certain languages (the functional kind) try to close this gap, but most do not. I find it useful to keep this distinction in mind when writing about both math and code.

[mathjax]:  https://www.mathjax.org/                        "MathJax"
[markdown]: https://daringfireball.net/projects/markdown/   "markdown"
