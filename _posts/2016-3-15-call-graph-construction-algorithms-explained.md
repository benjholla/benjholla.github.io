---
layout: post
title: Call Graph Construction Algorithms Explained
draft: true
---

A call graph is an artifact produced by program analysis tools to record the relationships between a function and the functions that a function calls. The figure below shows a call graph for a simple Java program that indicates the Java program has a method `B` that calls the method `C` and that `C` in turn calls methods `C` and `D`. If you've ever wondered how these call graphs actually generated then keep reading because in this post I'll be exploring several call graph construction algorithms in a way that should be simple to understand.

![Atlas Call Graph](/images/posts/call-graph-construction-algorithms-explained/AtlasCallGraph.png)

If you just want the nitty-gritty formal stuff then check out the [paper by Tip and Palsberg](http://web.cs.ucla.edu/~palsberg/paper/oopsla00.pdf), which inspired this post.

## The Problem(s)

Let's start by understanding why it might be hard to create a call graph. To keep things simple we are just going to be considering Java, but know that the problem gets harder when we start talking about dynamically typed langauges (such as Python or Ruby) or languages with function pointers (such as C, C++, or Objective-C).

Consider the following program. 

- What will it print if we run it?
- What methods were called at runtime?
- What edges should an ideal call graph have?

{% gist 8011fd547efa1286ce0d %}

### Static vs. Dynamic Dispatch
In Java, when we call a method the binding to the method we are calling may be done at compile time or runtime. If the method binding is resolvable at compile time we call the method call a static dispatch. If we can't resolve the method that will be called at compile time we call the method call a dynamic dispatch. Dynamic dispatches are extremely common in Object Oriented languages like Java, so we should spend some time making sure we understand how they work.

A static dispatch is resolvable at compile time, which means that we know exactly where to find the code for the method we are calling before we even run the program. In Java, calls to methods marked `static` and calls to constructors are both static dispatches. Since everything in Java is an object it makes sense that we should always know where the code to construct new objects is located in our program. Methods that are marked static (class methods) do not require that we create an instance of an object to invoke the method. Every executable Java program has at least one static method, the main method, which makes sense because before we run the program we haven't created any new objects yet, but we still want to an entry point into the program.

Java methods that are not marked static (instance methods) require an instance of a object. We can imagine a `Person` class with a method `getName()` that returns the name of a person. Since we don't want to assign the same name to all people, the `getName()` method should be associated with an instance of a person. This means we need to call `getName()` on a `Person` object (and not directly like a class method). A call to an instance method is resolved at runtime using a dynamic dispatch and the reason why will become clear soon.

Let's take a look at the `print` callsite in the main method of our example program from above. The `print` method is declared in class `A` and then overriden by `A`'s children in classes `B` and `C`. We know that `print` is an instance method because it is not marked as a `static` method. If we run this program, the `print` method declared in class `B` will get executed.  This is because the variable `b2` is an object of type `B`. If you wanted to call the `print` method declared in class `C` you could replace `b2` with `c2`. This means that we have to know the type of the object the instance method is being called on to know which method will get called at runtime!

Now remember that every object in Java can be drawn in a tree hierarchy with `java.lang.Object` at the very top.  The type hierarchy for our small program is shown below.  Any instance method declared in a parent type is inherited by child, unless that child provides an alternative implementation of the inherited method (by overriding the method). As a result of type hierarchy we can be assigned any object type to a variable of the same type or a variable with the type of any of the object's parent types (including `java.lang.Object`). This means that someone could write the following code.

	Object o;
	
	// getObject() returns a declared type of java.lang.Object,
	// which could be any runtime type
	o = getObject();
	
	o.toString();

The instance method `toString` is declared in `java.lang.Object` so it can be called on any object type. Since we don't know the type of the object in variable `o` we have to assume that `java.lang.Object`'s `toString` method or any object that overrides `toString` could be called at runtime! Since Java highly encourages developers to override the `toString` method this leaves us with quite a few possiblities (an only one correct answer).

### Whole vs. Partial Program Analysis