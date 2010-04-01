Author:  Scott Olson (scott_)
Created: 2010-03-32
Type:    Fix
Status:  Draft

Properties
==========

  + Motivation
  + The Problem
  + The Solution
  + Rationale
  + Concerns


Motivation
----------

It has been brought to my attention that the "ooc" language does not
support many of the standard set of operators programmers have come to
expect. Operators are an integral part of a programming language, and
the absense of standard operators will cause confusion for newcomers
to the language. Herein I will address the problem and hopefully
describe an adequate solution.


The Problem
-----------

The biggest problem with ooc today, according to the limited polling I
carried out, is it's inexplicable lack of the many operators that
programmers find most indispensible. As was pointed out to me, ooc
even goes so far as to use nonstandard operators for common operations
that have long since had standard associated operators. This problem
must be addressed swiftly, before even one more user is discouraged by
the glaring lack of operators. First impressions stick.

I have created a number of examples demonstrating what normal
programmers would expect to use in the ooc language, and what actually
exists today. To do something as simple as increment an int variable
(a very important tool in a high-level language), one would expect to
see code like `foo++` (you can remember this with the mnemonic "plus
podé". Podé is "one" in a dialect of Ubangian.). Not so in
ooc. Programmers have to completely relearn their operator tables,
using such unnatural constructs as `foo += 1` which I am told actually
means `foo = foo + 1`. As you can see, 'the ooc way' is much more
verbose, and having multiple forms to perform one task is sure to
cause inconsistency all across the ooc world.

The problem runs deeper than that. A user attempting to perform string
replication to write "PENIS" 1,000,000 times or maybe spam an IRC
channel with the phrase "you are what already forever seldom yes",
simple tasks in most languages, would be hard pressed to find the
solution in ooc. He would of course assume the usual operator for
string replication: `x`. (You can remember this operator with the
mnemonic "x means muliplication in arithmetic but not in programming
so we might as well use it for string replication.") Unfortunately,
his attempt `"you are what already forever seldom yes " x 1000000` is
greeted by this unsightly error:

    Missing semi-colon at the end of a line (got a 'Decimal' instead)
      "you are what already forever seldom yes " x 1000000
                                                   ^^^^^^^

Our protagonist doesn't have time to decipher cryptic errors, he has
spa... uhh, work to do! Unfortunately, in the ooc language, instead of
the standard `x` operator, the designers have opted to use the *same*
operator that is used in math for multiplying **numbers** to replicate
**strings**. Whoever created this fatal design flaw surely should be
flogged, but this proposal wasn't written just to throw blame or name
names, nddrylliog.

Another example of these obtuse design choices can be found again with
strings. The regular operator for concatenating strings, as you all
probably know is `.`. Unfortunately for the users of ooc, the chosen
operator for this process is the same one used for addition for
numbers. If anyone can devise a method of remembering this odd
operator, I am eagerly awaiting a message.

If you've ever had to compare strings in a programming language, you
know how easy it can be. The standard operators for string comparisons
are easily remembered, as they are just simple text abbreviations (eq,
ne, gt, ge, lt, le). Again, ooc just plain fucked everything up. They
used the **same** set of comparison operators for numbers *and*
strings. Tell me, if the same set of comparison operators are used for
number and strings, how can you tell the types of the variables in
`foo == bar`? You simply can't! This makes your code much more
ambiguous and implicit. And now all the programmers who have avoided
working with numbers because of all the confusing comparison operators
will have no choice but to avoid strings, or actually learn something
(something I hear is looked down upon in many computer science
courses.) If you attempt to use the regular operators for string
equality in ooc, it will give you another extrememly cryptic error:

    [ERROR] Can't resolve access to member String.eq
      if(foo eq bar) {
             ^^

I'm starting to think ooc error messages need a major overhaul. But
that's a topic for another proposal, maybe sometime next year around
March 32nd.

The two biggest proofs that ooc has a serious operator shortage are
thus: There is absolutely **no** variable decorating operator and
**no** member access operator!

The variable decorating operator, in most languages, is `$`. In ooc,
without this operator, there is simply no way for the parser to tell
the difference between the variable `x` and the operator `x`. And
there is no `x` operator.

As for the member access operator, one of the most abundant operators
in regular languages, ooc chose to use nothing more than space
characters. Supposedly the reasoning behind this decision was to make
ooc more "readable" and less cluttered, but leading experts agree
"more syntactic noise means more didactic joys." (Source unknown -
believed to be a paid Perl documentation writer)


The Solution
------------

After much deliberation and a tea party, I have come to a solution
that I think will appeal to everyone. The first option I considered
was to revamp the entire ooc parser and update all existing code
repositories to work under the new 'ooc with operators' parser, but as
you may be aware, that may take some measure of effort. Instead of
inflicting a ton of work on the language designer responsible for this
mess, I propose that we simply port all existing code to a more mature
language with support for all the proper operators. The language I
have in mind is perfect for the job, as it's next version is in active
development, coming with even more of the standard operators
implemented. When the next version is released, we should immidiately
begin porting all of our code to the new language, enjoying the sweet
succulent taste of freedom from the ooc operator oppression (known as
OOO). The full list of operators this language will support is handily
collected on a
[single page](http://glyphic.s3.amazonaws.com/ozone/mark/periodic/Periodic%20Table%20of%20the%20Operators%20A4%20300dpi.jpg)
for your reference.

Be right back.

Okay, hello again. I've got my crowbar-proof jacket on. No reason.


Rationale
---------

I got involved in the ooc community as soon as I learned about it, and
even I was long fooled by the proponents of this language, but a flaw
as prodigious as this could not escape the scrutiny of a mind with too
much time on its hands. I am now convinced that the only way to save
ooc is to have every single ooc user port all his code to Perl 6.


Concerns
--------

I've heard wispers in the hall, so to speak, that ooc's inner circle
of programmers and language designers may attempt to resist this move,
using such arguments as "Scott is insane." or "you are what already
forever seldom yes". Do not be fooled! Language designers are out to
get you. Never forget it.
