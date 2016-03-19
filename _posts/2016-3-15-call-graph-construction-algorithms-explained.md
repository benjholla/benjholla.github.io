---
layout: post
title: Call Graph Construction Algorithms Explained
draft: true
---

A call graph is an artifact produced by program analysis tools to record the relationships between a function and the functions it calls. The figure below shows a call graph for a simple Java program that indicates the Java program has a method `B` that calls the method `C` and that `C` in turn calls methods `B` and `D`. If you've ever wondered how these call graphs actually get generated then keep reading because in this post I'll be exploring several call graph construction algorithms and their tradeoffs in a way that should be simple to understand.

![Atlas Call Graph](/images/posts/call-graph-construction-algorithms-explained/AtlasCallGraph.png)

If you just want the nitty-gritty formal stuff then check out the [paper by Tip and Palsberg](http://web.cs.ucla.edu/~palsberg/paper/oopsla00.pdf), which inspired this post.

I've also implemented each variation of the algorithms discussed below in an open source Eclipse plugin project called the [call-graph-toolbox](https://ensoftcorp.github.io/call-graph-toolbox), which runs on top of the [Atlas](http://www.ensoftcorp.com/atlas/) program analysis framework. Atlas provides an interface for creating *Smart Views* that allow you to click on source code or program graphs to instantly produce an updated program graph relevant to what you clicked. The *call-graph-toolbox* provides each call graph implementation as a *Smart View* so the differences in the algorithms can be quickly observed visually. If you want to try out the algorithms on some of your own code I'd encourage you to install the *call-graph-toolbox* plugin.

## The Problem(s)

Let's start by understanding why it might be hard to create a call graph. To keep things simple we are just going to be considering Java, but know that the problem gets harder when we start talking about dynamically typed langauges (such as Python or Ruby) or languages with function pointers (such as C, C++, or Objective-C).

Consider the following program. 

- What will it print if we run it?
- What methods would be called at runtime?
- What edges should the ideal call graph have?

{% gist 8011fd547efa1286ce0d %}

### Static vs. Dynamic Dispatch
In Java, when we call a method, the binding between the callsite and the method to be called may be done at compile time or runtime. If the method binding is resolvable at compile time we call it a static dispatch. If at compile time we can't resolve the callsite we call it a dynamic dispatch and resolve the callsite later at runtime. Dynamic dispatches are extremely common in Object Oriented languages like Java, so we should spend some time making sure we understand how they work.

Since a static dispatch is resolvable at compile time, we know exactly where to find the code for the method we are calling before we even run the program. In Java, calls to methods marked `static` and calls to constructors are both static dispatches. Since everything in Java is an object it makes sense that we should always know exactly where the code to construct new objects is located in our program. Methods that are marked static (class methods) do not require that we create an instance of an object to invoke the method. Every executable Java program has at least one static method, the main method, which makes sense because before we run the program we haven't created any new objects yet, but we still want to invoke some procedure to use as an entry point into the program.

	// a static dispatch can be called directly on a Type without an instance
	// such as this hypothetical class method that runs a simulation procedure
	Animal.runSimulation();

Java methods that are not marked static (instance methods) require an instance of an object. We can imagine a `Person` class with a method `getName()` that returns the name of a person. Since we don't want to assign the same name to all people, the `getName()` method should be associated with an instance of a person. This means we need to call `getName()` on a `Person` object (and not directly on the type like a class method). A call to an instance method is resolved at runtime using a dynamic dispatch and the reason why will become clear soon.

Let's take a look at the `print` callsite in the main method of our example program from above. The `print` method is declared in class `A` and then overridden by `A`'s children in classes `B` and `C`. We know that `print` is an instance method because it is not marked as a `static` method. If we run this program, the `print` method declared in class `B` will get executed.  This is because the variable `b2` is an object of type `B`. If you wanted to call the `print` method declared in class `C` you could replace `b2` with `c2`. This means that we have to know the type of the object the instance method is being called on to know which method will get called at runtime!

Now remember that every object in Java can be drawn in a tree hierarchy with `java.lang.Object` at the very top.  The type hierarchy for our small program is shown below.  

![Type Hierarchy](/images/posts/call-graph-construction-algorithms-explained/TypeHierarchy.png)

Any non-private instance method declared in a parent type is inherited by child, unless that child provides an alternative implementation of the inherited method (by overriding the method). As a result of Java's type hierarchy we can also assigned any object type to a variable of the same type or a variable with the type of any of the object's parent types (including `java.lang.Object`). This means that someone could write the following code.

	Object o;
	
	// getRandomObjectType() returns a declared type of
	// java.lang.Object, which could be any runtime type
	o = getRandomObjectType();
	
	o.toString();

The instance method `toString` is declared in `java.lang.Object` so it can be called on any object type. Since we don't know the type of the object in variable `o` we have to assume that `java.lang.Object`'s `toString` method or any object that overrides `toString` could be called at runtime! Since Java highly encourages developers to override the `toString` method this leaves us with quite a few possibilities (and only one correct answer).

To answer our questions from above, we would all probably agree that an edge from the `print` callsite (at `b2.print(c2)`) to `B`'s `print` method implementation should be in our call graph because `B`'s `print` method is what gets executed at runtime. We also know that it could be tricky to figure out exactly which method would get called at runtime as a result of a dynamic dispatch. So should we conservatively add edges to `A` and `B`'s `print` methods as well if we can't figure out the type of `b2` even though `A` and `B`'s `print` methods are never called? What's better a call graph with all of the possible edges or a call graph with only the edges we are sure are correct?

We say a call graph is "sound" if it has all the edges that are possible at runtime. We say a call graph is "precise" if it does not have edges that do not occur at runtime. It's easy to be sound, but its hard to be sound *and* precise.

### Whole vs. Partial Program Analysis
Partial program analysis occurs when we don't have the entire program or when we can't scale up our analysis to run over the entire program.  You might think your Hello World program is small, but don't forget you are running it on top of a vast set of libraries provided with the Java Runtime Environment. If you're really crazy you might also consider the native implementation of the virtual machine itself or the operating system that virtual machine is running on and so on and so on!

## Idea 1 - Class Hierarchy Analysis (CHA)
Let's start by building a sound call graph (we'll worry about precision later). The first algorithm you might think to implement would probably be a variation of Reachability Analysis (RA). The basic idea is that for each method, we visit each callsite and then get the name of the method the callsite is referencing. We then add an edge from the method containing the callsite to every method with the same name as the callsite.

## Idea 2 - Rapid Type Analysis (RTA)
Class Hierarchy Analysis gives us a sound and cheap call graph.  In fact it is the default call graph implementation for most static analysis tools (including Atlas), but can we do better? Remember that a dynamic dispatch has to be called on an instance of an object, which means that in order for a dispatch to be made to a particular type's instance method at least one object of that type (or subtype) must have been allocated somewhere in the program. This is the core idea behind Rapid Type Analysis (RTA), which was proposed by [Bacon and Sweeney](http://researcher.watson.ibm.com/researcher/files/us-bacon/Bacon96Fast.pdf) (implementation details available in the [extended technical report](http://digitalassets.lib.berkeley.edu/techreports/ucb/text/CSD-98-1017.pdf)).

	RAPID TYPE ANALYSIS
	RTA = call graph of only methods (no edges)
	CHA = class hierarchy analysis call graph
	W = worklist containing the main method
        T = set of allocated types in M 
        T = T union allocated types in RTA callers of M
        for each callsite (C) in M
				M' = M' intersection methods declared in T or supertypes of T
				add each method in M' to W

The result of RTA on our first example is shown below.

![Rapid Type Analysis](/images/posts/call-graph-construction-algorithms-explained/RTA.png)

RTA produces a more precise call graph than CHA (because it is a subset of CHA and removes edges that are likely not feasible at runtime), but the result is no longer sound.  Let's consider a few cases where RTA might not produce a sound call graph.

	public static Object v;

	public static void main(String[] args){
		foo();
		bar();
	}

	public static void foo(){
		Object o = new A();
		v = o;
	}
	
	public static void bar(){
		v.toString()
	}

In the example above `main` calls `foo`, which allocates a new `A` type and stores the instance to a field `v`. Then `main` calls `bar`, which accesses from the field and calls `toString` on `v`. RTA will not include an edge from `bar` to `toString` because neither `bar` or its parents (`main`) allocated any instance that `toString` could be called on (however some implementations might consider the `String[] args` to be a special implicit allocation of an array type of String types, but the type `A` still does not reach `bar` in the basic RTA implementation).

	public static void main(String[] args){
		Object o = foo();
		bar(o);
	}

	public static Object foo(){
		return new A();
	}
	
	public static void bar(Object o){
		o.toString()
	}
	
In this example `main` calls `foo`, which returns an allocation of type `A` that is then passed as a parameter in the call to `bar`. Again in this case the `toString` edge to `A`'s `toString` method would be missing because neither `bar` or its parents (`main`) allocated a type of `A`.

As you may have guessed we could modify RTA to consider fields and method return/parameter types to improve the accuracy of RTA, which is exactly what [Tip and Palsberg](http://web.cs.ucla.edu/~palsberg/paper/oopsla00.pdf) proposed to do in the paper that inspired this post. In their paper, Tip and Palsberg propose to add several more constraints to RTA.

![XTA](/images/posts/call-graph-construction-algorithms-explained/XTA.png)
				
## Idea 3 - Variable Type Analysis (VTA)

![Variable Type Analysis](/images/posts/call-graph-construction-algorithms-explained/0CFA.png)

