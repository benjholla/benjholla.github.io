---
layout: post
title: Call Graph Construction Algorithms Explained
draft: true
---

A call graph is an artifact produced by program analysis tools to record the relationships between a function and the functions it calls. The figure below shows a call graph for a simple Java program that indicates the Java program has a method `B` that calls the method `C` and that `C` in turn calls methods `B` and `D`. If you've ever wondered how these call graphs actually get generated then keep reading because in this post I'll be exploring several call graph construction algorithms and their tradeoffs in a way that should be simple to understand.

![Atlas Call Graph](/images/posts/call-graph-construction-algorithms-explained/AtlasCallGraph.png)

If you just want the nitty-gritty formal stuff then check out the [paper by Tip and Palsberg](http://web.cs.ucla.edu/~palsberg/paper/oopsla00.pdf), which inspired this post.

## The Problem(s)

Let's start by understanding why it might be hard to create a call graph. To keep things simple we are just going to be considering Java, but know that the problem gets harder when we start talking about dynamically typed langauges (such as Python or Ruby) or languages with function pointers (such as C, C++, or Objective-C).

Consider the following program. 

- What will it print if we run it?
- What methods would be called at runtime?
- What edges should an ideal call graph have?

{% gist 8011fd547efa1286ce0d %}

### Static vs. Dynamic Dispatch
In Java, when we call a method, the binding between the callsite and the method to be called may be done at compile time or runtime. If the method binding is resolvable at compile time we call it a static dispatch. If at compile time we can't resolve the callsite we call it a dynamic dispatch and resolve the callsite later at runtime. Dynamic dispatches are extremely common in Object Oriented languages like Java, so we should spend some time making sure we understand how they work.

Since a static dispatch is resolvable at compile time, we know exactly where to find the code for the method we are calling before we even run the program. In Java, calls to methods marked `static` and calls to constructors are both static dispatches. Since everything in Java is an object it makes sense that we should always know exactly where the code to construct new objects is located in our program. Methods that are marked static (class methods) do not require that we create an instance of an object to invoke the method. Every executable Java program has at least one static method, the main method, which makes sense because before we run the program we haven't created any new objects yet, but we still want to invoke some procedure to use as an entry point into the program.

	// a static dispatch can be called directly on a Type without an instance
	// such as this hypothetical class method that runs a simulation procedure
	Animal.runSimulation();

Java methods that are not marked static (instance methods) require an instance of an object. We can imagine a `Person` class with a method `getName()` that returns the name of a person. Since we don't want to assign the same name to all people, the `getName()` method should be associated with an instance of a person. This means we need to call `getName()` on a `Person` object (and not directly on the type like a class method). A call to an instance method is resolved at runtime using a dynamic dispatch and the reason why will become clear soon.

Let's take a look at the `print` callsite in the main method of our example program from above. The `print` method is declared in class `A` and then overridden by `A`'s children in classes `B` and `C`. We know that `print` is an instance method because it is not marked as a `static` method. If we run this program, the `print` method declared in class `B` will get executed.  This is because the variable `b2` is an object of type `B`. If you wanted to call the `print` method declared in class `C` you could replace `b2` with `c2`. This means that we have to know the type of the object the instance method is being called on to know which method will get called at runtime!

Now remember that every object in Java can be drawn in a tree hierarchy with `java.lang.Object` at the very top.  The type hierarchy for our small program is shown below.  Any non-private instance method declared in a parent type is inherited by child, unless that child provides an alternative implementation of the inherited method (by overriding the method). As a result of Java's type hierarchy we can also assigned any object type to a variable of the same type or a variable with the type of any of the object's parent types (including `java.lang.Object`). This means that someone could write the following code.

	Object o;
	
	// getRandomObjectType() returns a declared type of java.lang.Object,
	// which could be any runtime type
	o = getRandomObjectType();
	
	o.toString();

The instance method `toString` is declared in `java.lang.Object` so it can be called on any object type. Since we don't know the type of the object in variable `o` we have to assume that `java.lang.Object`'s `toString` method or any object that overrides `toString` could be called at runtime! Since Java highly encourages developers to override the `toString` method this leaves us with quite a few possibilities (and only one correct answer).

To answer our questions from above, we would all probably agree that an edge from the `print` callsite (at `b2.print(c2)`) to `B`'s `print` method implementation should be in our call graph because `B`'s `print` method is what gets executed at runtime. We also know that it could be tricky to figure out exactly which method would get called at runtime as a result of a dynamic dispatch. So should we conservatively add edges to `A` and `B`'s `print` methods as well if we can't figure out the type of `b2` even though `A` and `B`'s `print` methods are never called? What's better a call graph with all of the possible edges or a call graph with only the edges we are sure are correct?

We say a call graph is "sound" if it has all the edges that are possible at runtime. We say a call graph is "precise" if it does not have edges that do not occur at runtime. It's easy to be sound, but its hard to be sound *and* precise.

### Whole vs. Partial Program Analysis
Partial program analysis occurs when we don't have the entire program or when we can't scale up our analysis to run over the entire program.  You might think your Hello World program is small, but don't forget you are running it on top of a vast set of libraries provided with the Java Runtime Environment. If you're really crazy you might also consider the native implementation of the virtual machine itself or the operating system that virtual machine is running on and so on and so on!In practice sometimes we just want to analyze a specific library in which case we have to do partial program analysis. Consider our example from before.  What would happen if we removed the `main` method and just consider the classes `A`, `B`, and `C`? Since the `main` method is the only place any types are actually created, it would look like the three `print` instance methods are just dead code, but don't forget that this "library" could be used by *any* Java program! Since we could conceive of Java programs that create types that could be used to invoke all three `print` methods we can't consider any of them dead code. So with that in mind it is going to be important to specify whether or not we are analyzing a partial program or a whole program when we construct our call graphs.