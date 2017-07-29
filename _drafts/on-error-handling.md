---
layout: post
title: "On Error Handling"
date: 2017-07-29 14:48:00 +0100
categories: java
---

## Why talk about error handling

* handling success vs. handling failure
* Importance in API design
* Importance in a system level design (sources? Can I expand this?)

## Anatomy of a Failure

* Operations complete with a result (Success or Failure)
* Language ability to express Result
* Language ability to express unexpected conditions e.g. runtime failures, error that were not initially considered, etc.

## Java case study: Exceptions

* How to express that an operation failed, in order to the let client handle them accordingly?
* Reasoning behind Exceptions i.e. they model exceptional conditions
* Is a business logic failure an exceptional condition?
* Is a component failure an exceptional condition?
* What about unknown failures?
* Checked vs Unchecked exceptions
* API design example:
    * handling success
    * handling known failures
    * handling unknown failures

* Concurrency + Exceptions: how to communicate failure across thread boundaries

### Go case study

_expand..._

### Rust case study

_expand..._

* correlate both studies with initial premise

## Error handling + concurrent environments is not fun

* No single train of though in concurrent environment
* Often, in distributed systems, you're dealing with asynchronous events. This means that any actions you decide to take, may not complete immediately. 
* Examples: missing replies, network failures, overloaded components, etc.
* Important that operation results are encoded in their responses
* be able to separate a failure result from a component failure.
* Failure result - your business logic should be considering such events
* Component failure - you might have predicted some of these events, but the scope tends to be wider than the former. Thus it is harder for your business logic to handle them.

