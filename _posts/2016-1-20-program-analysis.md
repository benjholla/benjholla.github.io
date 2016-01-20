---
layout: post
title: Program Analysis
draft: true
---

I'm sure this will be an evolving post as I develop these thoughts more, but after a few years of doing program analysis I've found that all my headaches, roadblocks, and general problems in program analysis seem to fit nicely into four distinct buckets. There is also a fifth bucket (program comprehension), which some might find objectionable, but that I currently see as a workaround to the problems posed by the first four buckets.

- Abstraction/Modeling
- Points-to Analysis
- Separate Compilation Assumptions
- Open vs. Closed World Assumptions

	and

- Program Comprehension

I'll pick on these one at a time to explain my thinking.

### Abstraction/Modeling
There are two main ideas that I'd categorize in this bucket.  First let's informally review the halting problem and why we need abstractions, then let's talk about modeling.  

#### Abstraction
To solve the halting problem, we are given an arbitrary program and an input for which we must decide whether or not the program will eventually halt or run forever.  Sadly the halting problem is provably undecidable, meaning we cannot answer if a program with halt or run forever for *all* program-input pairs. For someone doing program analysis this is bad news, because it means that any problem that is reducible to the halting problem is unsolvable in the *general* case. While that's not to say there aren't large classes of specific cases that we can't solve, it does mean our noble quest for the ultimate precision is a fruitless effort. The way we get around this is by creating a simpler abstraction of the problem that we can solve.

Let's consider a concrete example. Say I started on a passionate quest to detect all buffer overflows in all of the open source C/C++ projects on GitHub. At first the problem seems simple to model right? We'll talk more about modeling later, but for now lets just informally say that a buffer overflow occurs when data is written past the boundary of the allocated amount of memory in the buffer (array). While the fix is simple on paper, just add some extra logic to ensure that data being written to the array never exceeds the length the array; buffer overflows manifest in the wild so frequently that they have been one of the most common security vulnerabilities for past two decades. That's a good motivation to get back to our quest to detect all possible buffer overflows. Let's say we wrote some analysis that identified unchecked array stores. We run our analysis and we identify several array stores that are potential buffer overflows, but we must now show that it is feasible for data to be written past the boundary of the array. However, we could trivially reduce the problem of detecting feasible buffer overflows to solving the halting problem by simply inserting code that causes a buffer overflow before all halting instructions in a program. Since we know we can't solve the halting problem, we are left with two choices. We can either reduce the scope of our quest and only report on specific buffer overflows that we could show were feasible and report "I don't know" for everything else, or we create a simpler abstraction of the problem (that we can solve) and conservatively report on all *potential* buffer overflows. The trick is coming up with a good abstraction that reduces your false positives/false negatives, getting you as close to the ideal solution as possible.

#### Modeling
TODO

### Points-to Analysis
TODO

### Separate Compilation Assumptions
TODO: Whole program analysis vs. Summarization

### Open vs. Closed World Assumptions
Simply put, we can't analyze what do we don't have. 

TODO: Reflection/ClassLoaders/User Inputs, etc. -> Practical concerns of complex systems, mixed code, Android, etc.

### Program Comprehension
Developer mental model.  OODA Loops.