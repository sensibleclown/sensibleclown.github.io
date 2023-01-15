---
title: Big-O Notation in Algorithm
date: 2023-01-15 08:00:00 +/-TTTT
categories: [programming]
tags: [algorithm]   # TAG names should always be lowercase
---

Big-O notation is a method of communicating the upper limit of the running time complexity of an algorithm; it is basically a way to illustrate how the runtime of an algorithm increases as the size of the input grows. 

This notation typically utilizes a symbol, generally denoted as "O", to portray the maximum running time of an algorithm. For instance, O(n) implies that the runtime of an algorithm rises linearly with the input size n. 

Big-O notation is generally used to contrast the relative performance of different algorithms. For example, if algorithm A has a time complexity of O(n) and algorithm B has a time complexity of O(n^2), algorithm A is usually deemed to be more effective as its running time will expand more slowly as the input size increases. 

It is important to understand that Big-O notation merely provides an upper limit on the running time, it does not provide an exact measure of the running time. Additionally, it only takes into account the most significant term of the running time and therefore often disregards lower order terms and constants. 

Examples of some of the more common time complexities using Big-O notation include: 
- O(1): Constant time, the runtime does not depend on the input size;
- O(log n): Logarithmic time, often seen in algorithms that divide the input in half with each iteration;
- O(n): Linear time, the running time increases linearly with the input size;
- O(n log n): Linear logarithmic time, often seen in sorting algorithms;
- O(n^2): Quadratic time, the running time increases at a rate of n^2 with the input size;
- O(2^n): Exponential time, the running time increases rapidly with the input size.

It is also important to keep in mind that the time complexity of an algorithm can be influenced by various factors, such as the data structure and the implementation of the algorithm.
