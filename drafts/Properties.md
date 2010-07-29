Author:  Scott Olson (scott_)  
Created: 2010-03-30  
Type:    Feature  
Status:  Draft  

Properties
==========

  + Motivation
  + What is a Property?
  + What about speed?!
  + Rationale
  + Issues


Motivation
----------

This idea is the result of a few long and thoughtful IRC
conversations. People wanted the ability to write custom accessors
that still use the familiar attribute setting syntax on the outside,
and we finally came up with an elegant, consistent syntax for the
accessor definitions. They are easy to understand, but still quite
powerful. A big thanks goes out to the people in the #ooc-lang IRC
channel on Freenode.


What is a Property?
-------------------

Properties are an extension to the idea of attributes that ooc
currently uses for classes and covers. They are simply attributes with
an associated block of code that defines a custom getter or setter
method. Here is a simple example of a property:

    Person: class
        age: Int {
            set(value) {
                age = value
            }
            get {
                age
            }
        }
    }

Now we can call those custom accessor methods with a very familiar
syntax:

    frank := Person new()
    frank age = 12
    frank age toString() println()

That example does nothing more than a regular attribute, but
demonstrates the simple syntax for setters and getters. Note that the
argument to `set` and the return type of `get` will be typechecked
against the type of the property (`Int` in this case.) The same
shortcut used in `init` and other methods to assign an attribute can
be used here:

    set(=age) {}

A property may be read-only or write-only, that is, having only a
getter, or a setter, respectively. Simply leave out `set` and your
property will not be able to be assigned, or leave out `get` and it
will not be able to be accessed. For example, assuming all people die
at 80:

    yearsUntilDeath: Int {
        get { 80 - age }
    }

Simple, right? If someone tries to do `frank yearsUntilDeath = 40`,
they will get a compile-time error. However, in the case of this
example, it would be much more fitting to write a method
`yearsUntilDeath()` instead of a read-only getter. Properties are very
powerful and flexible, which unfortunately also means they will likely
be abused.

A property with a setter and no getter is just the opposite:

    foo: Int {
        set(value) {
            doSomething(foo)
        }
    }

If you are very perceptive, you might already be wondering how to
create a property that has a custom setter but still has the default
getter or vice-versa. Well, it's simple. Simply write `set` or `get`
with no arguments and code body and it will have the default
direct-access setter or getter.

    name: String {
        set
        get
    }
    
    // The above example is exactly equivalent to this:
    name: String
    
    age: Int {
        set(=age) {
            if(age > 100)
                "Wow, you're so... elderly!" println()
        }
        get
    }

A final characteristic of properties you should know about is how they
are accessed from within their own accessor methods. If you use `age`
or `age = blah` inside the setter/getter of a property called `age`,
it will **not** call the custom setter/getter, but will return the
underlying value of `age`. This is to avoid the endless recursion
resulting in a stack overflow that would happen if `age` in `age`s
getter method called `age`s getter method... or vice versa for
setters. However, it is not impossible to call the custom accessors
from within the accessors. In fact it is quite simple:

    Dog: class {
        fleas: Int {
            set(value) {
                if(value < 10) {
                    // Because dogs have twice as many fleas than what
                    // you counted :(
                    set(value * 2)
                } else {
                    fleas = value
                }
            }
            get
        }
    }

It is also perfectly legal to call `get()` from the setter or `set(x)`
from the getter. When using any form of set() or get() inside your
accessor methods, you must be very careful that you don't infinitely
recurse.

Properties can also user extern setter or getter functions:

    Foo: cover from struct foo {
        bar: Int {
            set: extern(Foo_setBar)
            get: extern(Foo_getBar)
        }
    }

In this case, the set function must match
`void Foo_setBar(struct foo, int)` and the get function must match
`int Foo_getBar(struct foo)`. You can mix and match extern and
non-extern setters and getters in any way.

What about speed?!
------------------

Properties with custom accessors will be just as performant as a
`setFoo` method call. When you use the direct-access setter/getter
(that is, `set` or `get` without a code body) it will perform the same
as an attribute did before properties were introduced. A property with
only `set` and `get` with no code bodies is the exact same as a
property with no accessor definitions at all.


Rationale
---------

Almost everywhere in the language, assignment is done using the symbol
"=". Why then, should we have to break with convention with methods
like `setAge()` or `getName()` just to have accessor methods that do
something special? Properties are an elegant solution to this problem,
allowing programmers to use `person age = x` and `person name` syntax
with their custom accessor methods.


Issues
------

Properties are really easy to do, and there isn't too much that can go
wrong, but I'll cover it here anyways. Say, for some reason, you happen
to write:

    height: Int {}

This property has no getter or setter, meaning it's really completely
useless. There is no way you could ever set it or get it, so the
compiler will give you an error telling you what's wrong.

Another issue that may come up is the case of using the same name for
the property and the argument in `set`. The solution is to qualify the
property with `this`. This works because when you access the property
from it's own accessor, it will return the underlying value, ignoring
any custom accessors (thus avoiding infinite recursion.)  Of course,
you could always just use a different name for the argument instead...

    width: Int {
        set(width) {
            calculatePerimeter(width, height)
            this width = width
        }
        get
    }

There, it shouldn't be any trouble. (Another option would be to use
`=width` as the argument, and leave out `this width = width`.)

The final possible name clashing is if your class has a `set` or `get`
method. Again the solution is to just use `this` to qualify the name.

    SubtractiveBox: class {
        contents: Int {
            set
            get {
                // calling 'set(12)' here would call the setter for
                // contents like usual. Calling 'this set(12)' would
                // call the method on SubtractiveBox.
                set(contents - 1)
                contents
            }
	
        set: func(.contents) { this contents = contents - 1 }
        get: func -> Int { contents }
    }

The `set()` and `get()` calling is also dangerous because of potential
for infinite loops. They should be avoided as much as possible, usually.

These issues are easily avoided and don't pose a real problem, but
they are documented here so at least you know what's going on if you
stumble into one accidentally. =)
