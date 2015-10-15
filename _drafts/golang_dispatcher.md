---
layout: post
title: Dispatching jobs in Golang
summary: A simple job dispatcher in go.
categories: tech
tags: golang queue
---

Trying to be OO, want to have a Job object people subclass. However, go has no generics, so the best you can do is `interface{}`.

I went a different route and built a `Work` interface, so if your struct implements the interface, it can be used with the dispatcher. I'm not sure if this is golang best practices, but it felt "generic".

[go]:   https://golang.org    "Go"
