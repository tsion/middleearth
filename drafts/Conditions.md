(This is a shameless copy of
http://www.gigamonkeys.com/book/beyond-exception-handling-conditions-and-restarts.html
translated to psuedo-ooc that hopefully will lose the 'psuedo'
modifier soon!)

# Beyond Exception Handling: Conditions and Restarts

One of ooc's great features is its condition system. It serves a
similar purpose to the exception handling systems in Java, Python, and
C++ but is more flexible. In fact, its flexibility extends beyond
error handling--conditions are more general than exceptions in that a
condition can represent any occurrence during a program's execution
that may be of interest to code at different levels on the call
stack. For example, in the section "Other Uses for Conditions," you'll
see that conditions can be used to emit warnings without disrupting
execution of the code that emits the warning while allowing code
higher on the call stack to control whether the warning message is
printed. For the time being, however, I'll focus on error handling.

The condition system is more flexible than exception systems because
instead of providing a two-part division between the code that signals
an error and the code that handles it, the condition system splits the
responsibilities into three parts--signaling a condition, handling it,
and restarting. In this chapter, I'll describe how you could use
conditions in part of a hypothetical application for analyzing log
files. You'll see how you could use the condition system to allow a
low-level function to detect a problem while parsing a log file and
signal an error, to allow mid-level code to provide several possible
ways of recovering from such an error, and to allow code at the
highest level of the application to define a policy for choosing which
recovery strategy to use.

To start, I'll introduce some terminology: errors, as I'll use the
term, are the consequences of Murphy's law. If something can go wrong,
it will: a file that your program needs to read will be missing, a
disk that you need to write to will be full, the server you're talking
to will crash, or the network will go down. If any of these things
happen, it may stop a piece of code from doing what you want. But
there's no bug; there's no place in the code that you can fix to make
the nonexistent file exist or the disk not be full. However, if the
rest of the program is depending on the actions that were going to be
taken, then you'd better deal with the error somehow or you will have
introduced a bug. So, errors aren't caused by bugs, but neglecting to
handle an error is almost certainly a bug.

So, what does it mean to handle an error? In a well-written program,
each function is a black box hiding its inner workings. Programs are
then built out of layers of functions: high-level functions are built
on top of the lower-level functions, and so on. This hierarchy of
functionality manifests itself at runtime in the form of the call
stack: if high calls medium, which calls low, when the flow of control
is in low, it's also still in medium and high, that is, they're still
on the call stack.

Because each function is a black box, function boundaries are an
excellent place to deal with errors. Each function--low, for
example--has a job to do. Its direct caller--medium in this case--is
counting on it to do its job. However, an error that prevents it from
doing its job puts all its callers at risk: medium called low because
it needs the work done that low does; if that work doesn't get done,
medium is in trouble. But this means that medium's caller, high, is
also in trouble--and so on up the call stack to the very top of the
program. On the other hand, because each function is a black box, if
any of the functions in the call stack can somehow do their job
despite underlying errors, then none of the functions above it needs
to know there was a problem--all those functions care about is that
the function they called somehow did the work expected of it.

In most languages, errors are handled by returning from a failing
function and giving the caller the choice of either recovering or
failing itself. Some languages use the normal function return
mechanism, while languages with exceptions return control by throwing
or raising an exception. Exceptions are a vast improvement over using
normal function returns, but both schemes suffer from a common flaw:
while searching for a function that can recover, the stack unwinds,
which means code that might recover has to do so without the context
of what the lower-level code was trying to do when the error actually
occurred.

Consider the hypothetical call chain of high, medium, low. If low
fails and medium can't recover, the ball is in high's court. For high
to handle the error, it must either do its job without any help from
medium or somehow change things so calling medium will work and call
it again. The first option is theoretically clean but implies a lot of
extra code--a whole extra implementation of whatever it was medium was
supposed to do. And the further the stack unwinds, the more work that
needs to be redone. The second option--patching things up and
retrying--is tricky; for high to be able to change the state of the
world so a second call into medium won't end up causing an error in
low, it'd need an unseemly knowledge of the inner workings of both
medium and low, contrary to the notion that each function is a black
box.

## The ooc Way

ooc's error handling system gives you a way out of this conundrum by
letting you separate the code that actually recovers from an error
from the code that decides how to recover. Thus, you can put recovery
code in low-level functions without committing to actually using any
particular recovery strategy, leaving that decision to code in
high-level functions.

To get a sense of how this works, let's suppose you're writing an
application that reads some sort of textual log file, such as a Web
server's log. Somewhere in your application you'll have a function to
parse the individual log entries. Let's assume you'll write a
function, `parseLogEntry`, that will be passed a string containing the
text of a single log entry and that is supposed to return a `LogEntry`
object representing the entry. This function will be called from a
function, `parseLogFile`, that reads a complete log file and returns a
list of objects representing all the entries in the file.

To keep things simple, the `parseLogEntry` function will not be
required to parse incorrectly formatted entries. It will, however, be
able to detect when its input is malformed. But what should it do when
it detects bad input? In C you'd return a special value to indicate
there was a problem. In Java or Python you'd throw or raise an
exception. In ooc, you signal a condition.

## Conditions

A condition is an object whose class indicates the general nature of
the condition and whose instance data carries information about the
details of the particular circumstances that lead to the condition
being signaled. In this hypothetical log analysis program, you might
define a condition class, `MalformedLogEntryError`, that
`parseLogEntry` will signal if it's given data it can't parse.

Condition classes are defined as regular ooc classes that inherit from
the `Condition` class (directly or through other subclasses).

When using the condition system for error handling, you should define
your conditions as subclasses of `Error`, a subclass of
`Condition`. Thus, you might define `MalformedLogEntryError`, with a
slot to hold the argument that was passed to `parseLogEntry`, like
this:

    MalformedLogEntryError: class extends Error {
        text: String    
        init: func (=text) {}
    }

## Condition Handlers

In `parseLogEntry` you'll signal a `MalformedLogEntryError` if you
can't parse the log entry. You signal errors with the method `throw`,
which calls the lower-level method `signal` and aborts the program if
the condition isn't handled. You could write `parseLogEntry` like
this, eliding the details of actually parsing a log entry:

    parseLogEntry: func (text: String) -> LogEntry {
        if(wellFormedLogEntry?(text)) {
            LogEntry new(text)
        } else {
            MalformedLogEntryError new(text) throw()
        }
    }

What happens when the error is signaled depends on the code above
`parseLogEntry` on the call stack. To avoid aborting the program, you
must establish a condition handler in one of the functions leading to
the call to `parseLogEntry`. When a condition is signaled, the
signaling machinery looks through a list of active condition handlers,
looking for a handler that can handle the condition being signaled
based on the condition's class. Each condition handler consists of a
type specifier indicating what types of conditions it can handle and a
function that takes a single argument, the condition. At any given
moment there can be many active condition handlers established at
various levels of the call stack. When a condition is signaled, the
signaling machinery finds the most recently established handler whose
type specifier is compatible with the condition being signaled and
calls its function, passing it the condition object.

The handler function can then choose whether to handle the
condition. The function can decline to handle the condition by simply
returning normally, in which case control returns to the `signal`
function, which will search for the next most recently established
handler with a compatible type specifier. To handle the condition, the
function must transfer control out of `signal` via a nonlocal exit. In
the next section, you'll see how a handler can choose where to
transfer control. However, many condition handlers simply want to
unwind the stack to the place where they were established and then run
some code. The try/catch expression establishes this kind of condition
handler. The basic form of a try/catch expression is as follows:

    try {
        doSomething()
        doSomethingElse()
    } catch e: SomeError {
        fixIt()
    } catch (foo: FooError) {
        bar()
    } catch OtherError {
        blah()
    }

The var, if included, is the name of the variable that will hold the
condition object when the handler code is executed. If the code
doesn't need to access the condition object, you can omit the variable
name.

One way to handle the `MalformedLogEntryError` signaled by
`parseLogEntry` in its caller, `parseLogFile`, would be to skip the
malformed entry. In the following function, the try/catch expression
will either collect the value returned by parse-log-entry or do
nothing if a `MalformedLogEntryError` is signaled.

    parseLogFile: func (file: File) -> ArrayList<LogEntry> {
        entries := ArrayList<LogEntry> new()
        file eachLine(|line|
            try {
                entries add(parseLogEntry(line))
            } catch MalformedLogEntryError {}
        )
        entries
    }

When `parseLogEntry` returns normally, its value will be added to
`entries`, but if `parseLogEntry` signals a `MalformedLogEntryError`,
then the catch will be invoked, which in this case just ignores it.

This version of `parseLogFile` has one serious deficiency: it's doing
too much. As its name suggests, the job of `parseLogFile` is to parse
the file and produce a list of log-entry objects; if it can't, it's
not its place to decide what to do instead. What if you want to use
`parseLogFile` in an application that wants to tell the user that the
log file is corrupted or one that wants to recover from malformed
entries by fixing them up and re-parsing them? Or maybe an application
is fine with skipping them but only until a certain number of
corrupted entries have been seen.

You could try to fix this problem by moving the try/catch to a
higher-level function. However, then you'd have no way to implement
the current policy of skipping individual entries--when the error was
signaled, the stack would be unwound all the way to the higher-level
function, abandoning the parsing of the log file altogether. What you
want is a way to provide the current recovery strategy without
requiring that it always be used.

## Restarts

The condition system lets you do this by splitting the error handling
code into two parts. You place code that actually recovers from errors
into restarts, and condition handlers can then handle a condition by
invoking an appropriate restart. You can place restart code in mid- or
low-level functions, such as `parseLogFile` or `parseLogEntry`, while
moving the condition handlers into the upper levels of the
application.

To change `parseLogFile` so it establishes a restart instead of a
condition handler, you can change the try/catch to a try/restart. The
form of try/restart is quite similar to a try/catch except the names
of restarts are just names, not necessarily the names of condition
classes. In general, a restart name should describe the action the
restart takes. In `parseLogFile`, you can call the restart
`skipLogEntry` since that's what it does. The new version will look
like this:

    parseLogFile: func (file: File) -> ArrayList<LogEntry> {
        entries := ArrayList<LogEntry> new()
        file eachLine(|line|
            try {
                entries add(parseLogEntry(line))
            } restart "skipLogEntry" {}
        )
        entries
    }

If you invoke this version of `parseLogFile` on a log file containing
corrupted entries, it won't handle the error; the program will
abort. To avoid this, you can establish a condition handler that
invokes the `skipLogEntry` restart automatically.

The advantage of establishing a restart rather than having
`parseLogFile` handle the error directly is it makes `parseLogFile`
usable in more situations. The higher-level code that invokes
`parseLogFile` doesn't have to invoke the `skipLogEntry` restart. It
can choose to handle the error at a higher level. Or, as I'll show in
the next section, you can add restarts to `parseLogEntry` to provide
other recovery strategies, and then condition handlers can choose
which strategy they want to use.

But before I can talk about that, you need to see how to set up a
condition handler that will invoke the `skipLogEntry` restart. You can
set up the handler anywhere in the chain of calls leading to
`parseLogFile`. This may be quite high up in your application, not
necessarily in `parseLogFile`'s direct caller. For instance, suppose
the main entry point to your application is a function, `logAnalyser`,
that finds a bunch of logs and analyzes them with the function
`analyzeLog`, which eventually leads to a call to
`parseLogFile`. Without any error handling, it might look like this:

    logAnalyzer: func {
        findAllLogs() each(|log| analyzeLog(log))
    }

The job of `analyzeLog` is to call, directly or indirectly,
`parseLogFile` and then do something with the list of log entries
returned. An extremely simple version might look like this:

    analyzeLog: func (log: File) {
        parseLogFile(log) each(|entry| analyzeEntry(entry))
    }

where the function `analyzeEntry` is presumably responsible for
extracting whatever information you care about from each log entry and
stashing it away somewhere.

Thus, the path from the top-level function, `logAnalyser`, to
`parseLogEntry`, which actually signals an error, is as follows:

Assuming you always want to skip malformed log entries, you could
change this function to establish a condition handler that invokes the
`skipLogEntry` restart for you. However, you can't use try/catch to
establish the condition handler because then the stack would be
unwound to the function where the try/catch appears. Instead, you need
to use the try/handle expression. The basic form of try/handle is as
follows:

    try {
        ...
    } handle var: ConditionType {
        ...
    } handle AnotherCondition {
        ...
    }

An important difference between try/handle and try/catch is that the
handler function bound by try/handle will be run without unwinding the
stack--the flow of control will still be in the call to
`parseLogEntry` when this function is called. The use of
`invokeRestart` will find and invoke the most recently bound restart
with the given name. So you can add a handler to `logAnalyser` that
will invoke the `skipLogEntry` restart established in `parseLogFile`
like this:

    logAnalyzer: func {
        try {
            findAllLogs() each(|log| analyzeLog(log))
        } handle MalformedLogEntryError {
            invokeRestart("skipLogEntry")
        }
    }

In this try/handle, the handler invokes the restart `skipLogEntry`.

As written, the `MalformedLogEntryError` handler assumes that a
`skipLogEntry` restart has been established. If a
`MalformedLogEntryError` is ever signaled by code called from
`logAnalyser` without a `skipLogEntry` having been established, the
call to `invokeRestart` will signal a `ControlError` when it fails to
find the `skipLogEntry` restart. If you want to allow for the
possibility that a `MalformedLogEntryError` might be signaled from
code that doesn't have a `skipLogEntry` restart established, you could
change the handler to this:

        } handle MalformedLogEntryError {
            restart := findRestart("skipLogEntry")
            if(restart)
                invokeRestart("skipLogEntry")
        }

`findRestart` looks for a restart with a given name and returns an
object representing the restart if the restart is found and `null` if
not. You can invoke the restart by passing the restart object to
`invokeRestart`. Thus, when `skipLogEntry` is bound with try/handle,
it will handle the condition by invoking the `skipLogEntry` restart if
one is available and otherwise will return normally, giving other
condition handlers, bound higher on the stack, a chance to handle the
condition.

## Providing Multiple Restarts

Since restarts must be explicitly invoked to have any effect, you can
define multiple restarts, each providing a different recovery
strategy. As I mentioned earlier, not all log-parsing applications
will necessarily want to skip malformed entries. Some applications
might want `parseLogFile` to include a special kind of object
representing malformed entries in the list of `LogEntry` objects;
other applications may have some way to repair a malformed entry and
may want a way to pass the fixed entry back to `parseLogEntry`.

To allow more complex recovery protocols, restarts can take arbitrary
arguments, which are passed in the call to `invokeRestart`. You can
provide support for both the recovery strategies I just mentioned by
adding two restarts to `parseLogEntry`, each of which takes a single
argument. One simply returns the value it's passed as the return value
of `parseLogEntry`, while the other tries to parse its argument in the
place of the original log entry.

    parseLogEntry: func (text: String) -> LogEntry {
        if(wellFormedLogEntry?(text)) {
            LogEntry new(text)
        } else {
            try {
                MalformedLogEntryError new(text) throw()
            } restart "useValue" (value: LogEntry) {
                value
            } restart "reparseEntry" (fixedText: String) {
                parseLogEntry(fixedText)
            }
        }
    }

The name `useValue` is a standard name for this kind of restart.

If you wanted to change the policy on malformed entries to one that
created an instance of `MalformedLogEntry`, you could change
`logAnalyser` to this (assuming the existence of a `MalformedLogEntry`
class):

    logAnalyzer: func {
        try {
            findAllLogs() each(|log| analyzeLog(log))
        } handle e: MalformedLogEntryError {
            invokeRestart("useValue", MalformedLogEntry new(e text))
        }
    }

You could also have put these new restarts into `parseLogFile` instead
of `parseLogEntry`. However, you generally want to put restarts in the
lowest-level code possible. It wouldn't, though, be appropriate to
move the `skipLogEntry` restart into `parseLogEntry` since that would
cause `parseLogEntry` to sometimes return normally with `null`, the
very thing you started out trying to avoid. And it'd be an equally bad
idea to remove the `skipLogEntry` restart on the theory that the
condition handler could get the same effect by invoking the `useValue`
restart with `null` as the argument; that would require the condition
handler to have intimate knowledge of how the `parseLogFile` works. As
it stands, the `skipLogEntry` is a properly abstracted part of the
log-parsing API.

## Other Uses for Conditions

While conditions are mainly used for error handling, they can be used
for other purposes--you can use conditions, condition handlers, and
restarts to build a variety of protocols between low- and high-level
code. The key to understanding the potential of conditions is to
understand that merely signaling a condition has no effect on the flow
of control.

The primitive signaling method `signeal` implements the mechanism of
searching for an applicable condition handler and invoking its handler
function. The reason a handler can decline to handle a condition by
returning normally is because the call to the handler function is just
a regular function call--when the handler returns, control passes back
to `signal`, which then looks for another, less recently bound handler
that can handle the condition. If `signal` runs out of condition
handlers before the condition is handled, it also returns normally.

The `throw` method you've been using calls `signal`. If the error is
handled by a condition handler that transfers control via try/catch or
by invoking a restart, then the call to `signal` never returns. But if
`signal` returns, `throw` aborts the program. Thus, a call to `throw`
can never return normally; the condition must either be handled by a
condition handler or cause an abort.

Another condition signaling function, `warn`, provides an example of a
different kind of protocol built on the condition system. Like
`throw`, `warn` calls `signal` to signal a condition. But if `signal`
returns, `warn` doesn't invoke the debugger--it prints the condition
to stderr and returns, allowing its caller to proceed. `warn` also
establishes a restart, `muffleWarning`, around the call to `signal`
that can be used by a condition handler to make `warn` return without
printing anything. Of course, a condition signaled with `warn` could
also be handled in some other way--a condition handler could "promote"
a warning to an error by handling it as if it were an error.

For instance, in the log-parsing application, if there were ways a log
entry could be slightly malformed but still parsable, you could write
`parseLogEntry` to go ahead and parse the slightly defective entries
but to signal a condition with `warn` when it did. Then the larger
application could choose to let the warning print, to muffle the
warning, or to treat the warning like an error, recovering the same
way it would from a `MalformedLogEntryError`.

You can also build your own protocols on `signal`--whenever low-level
code needs to communicate information back up the call stack to
higher-level code, the condition mechanism is a reasonable mechanism
to use. But for most purposes, one of the standard error or warning
protocols should suffice.

Unfortunately, it's the fate of error handling to always get short
shrift in programming texts--proper error handling, or lack thereof,
is often the biggest difference between illustrative code and
hardened, production-quality code. The trick to writing the latter has
more to do with adopting a particularly rigorous way of thinking about
software than with the details of any particular programming language
constructs. That said, if your goal is to write that kind of software,
you'll find the ooc condition system is an excellent tool for writing
robust code.
