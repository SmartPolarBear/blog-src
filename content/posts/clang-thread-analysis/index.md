---
title: "Achieve Better Parallel Code with Clang Static Thread Analysis"
date: 2021-02-08T22:50:42+08:00
draft: true
slug: "clang-thread-analysis"
description:  "Achieve thread safety without headache"
categories:
    - Blog
    - Programing Languages
tags:
    - Clang
    - Static Analysis
---

In the recent two years, thanks to AMD's great job to push the industry forward,  we can see the trend that the number of cores of CPU grow rapidly, reaching 8c16t for average gaming laptop and high-performance PC, while the growth of single core performance seems to be slower. That's probably why parallelism plays such a important role in modern programming. With out it, it is impossible to make the most of the progress of hardware.   

Nevertheless, it's none of a easy job. Parallel programs are known for difficulties in debugging.  And, as an effective way to synchronization, locks are widely used, but the programs are not that easy to reason about. Surely programmers should learn well about the techniques for concurrency, but, hey, when coding a real project we can hardly spare enough effort to reason about locks while solving the problem, can we? Fortunately, clang provide a mechanism called [Thread Safety Analysis](https://clang.llvm.org/docs/ThreadSafetyAnalysis.html) to help us out! 

## Introduction


## Basics

## Attributes for Locks

## Attributes for Lock Users