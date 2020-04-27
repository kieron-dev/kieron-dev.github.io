---
title: Introduction to the lager.Logger
tags: [tutorial, logging, golang]
---

## Exploring the lager.Logger

The [lager.Logger](https://github.com/cloudfoundry/lager) is an *opinionated
logger for Go*. It differs from the standard library's log package by
outputting structured data in JSON format (for simple machine parsing) and by
emitting logs at various levels.

Let's start from the basics and discover the best practices for using lager.Logger.

### Basic logging

Consider an example app where a method on object `objA` calls a method on
object `objB` which in turn calls a method on object `objC`. We'll show this in
a single file for convenience.  The object dependencies are injected in the
constructors, and wired up in the `main()` function, before the top level
method is invoked.

```golang
package main

type objA struct {
	b *objB
}

func newA(b *objB) *objA {
	return &objA{
		b: b,
	}
}

func (a *objA) doItA() {
	a.b.doItB()
}

type objB struct {
	c *objC
}

func newB(c *objC) *objB {
	return &objB{
		c: c,
	}
}

func (b *objB) doItB() {
	b.c.doItC()
}

type objC struct{}

func newC() *objC {
	return &objC{}
}

func (c *objC) doItC() {

}

func main() {
	c := newC()
	b := newB(c)
	a := newA(b)

	a.doItA()
}
```

Now suppose we would like use the lager.Logger to provide some debug logging in
the `doIt*()` methods. We could make that happen easily with a global `logger`
variable, which we set up in the `main()` method. In *lager*, we create a new
`Logger` passing a name, and associate it with a *Sink*, which determines log
level filtering and the output location.

```golang
package main

import (
	"os"
	"code.cloudfoundry.org/lager"
)

var logger lager.Logger
//...

func (a *objA) doItA() {
	logger.Debug("obj-a.do-it-a")
	a.b.doItB()
}
//...


func main() {
	logger = lager.NewLogger("logging-example")
	logger.RegisterSink(lager.NewPrettySink(os.Stderr, lager.DEBUG))

    c := newC()
//...
```

Here is the output, where we've added simliar lines for `doItB()` and `doItC()`.

```
{"timestamp":"2020-04-27T20:31:36.749494926Z","level":"debug","source":"logging-example","message":"logging-example.obj-a.do-it-a","data":{}}
{"timestamp":"2020-04-27T20:31:36.749894682Z","level":"debug","source":"logging-example","message":"logging-example.obj-b.do-it-b","data":{}}
{"timestamp":"2020-04-27T20:31:36.749928734Z","level":"debug","source":"logging-example","message":"logging-example.obj-c.do-it-c","data":{}}
```

### Introducing Sessions

Notice how the base name of the logger, 'logging-example', is passed through to
the message name in the individual logs above. If we use the *Session* feature,
we can show a kind of call-trace in the log message names.

This requires passing the logger through the method calls, and creating derived
loggers with the `Session` method. So we ditch our global logger variable, and
pass loggers through the method parameters.

```golang
//...

func (a *objA) doItA(logger lager.Logger) {
	logger = logger.Session("obj-a")
	logger.Debug("do-it-a")
	a.b.doItB(logger)
}
//...

func (b *objB) doItB(logger lager.Logger) {
	logger = logger.Session("obj-b")
	logger.Debug("do-it-b")
	b.c.doItC(logger)
}
```

This both creates the call-trace dotted notation in the log messages, but also
adds a session field to the log data.

```
{"timestamp":"2020-04-27T20:39:15.802802995Z","level":"debug","source":"logging-example","message":"logging-example.obj-a.do-it-a","data":{"session":"1"}}
{"timestamp":"2020-04-27T20:39:15.803104838Z","level":"debug","source":"logging-example","message":"logging-example.obj-a.obj-b.do-it-b","data":{"session":"1.1"}}
{"timestamp":"2020-04-27T20:39:15.803170929Z","level":"debug","source":"logging-example","message":"logging-example.obj-a.obj-b.obj-c.do-it-c","data":{"session":"1.1.1"}}
```

We can use that session data value to correlate nested calls in busy output
from a multi-threaded program. The session numbers are incremented with each
`Session()` call on a parent session. So if we duplicate the `doItC()` and 
`doItA()` calls, we would see logs as follows.

```
{"timestamp":"2020-04-27T20:41:40.450739775Z","level":"debug","source":"logging-example","message":"logging-example.obj-a.do-it-a","data":{"session":"1"}}
{"timestamp":"2020-04-27T20:41:40.450876100Z","level":"debug","source":"logging-example","message":"logging-example.obj-a.obj-b.do-it-b","data":{"session":"1.1"}}
{"timestamp":"2020-04-27T20:41:40.450904331Z","level":"debug","source":"logging-example","message":"logging-example.obj-a.obj-b.obj-c.do-it-c","data":{"session":"1.1.1"}}
{"timestamp":"2020-04-27T20:41:40.450920867Z","level":"debug","source":"logging-example","message":"logging-example.obj-a.obj-b.obj-c.do-it-c","data":{"session":"1.1.2"}}
{"timestamp":"2020-04-27T20:41:40.450937039Z","level":"debug","source":"logging-example","message":"logging-example.obj-a.do-it-a","data":{"session":"2"}}
{"timestamp":"2020-04-27T20:41:40.450960417Z","level":"debug","source":"logging-example","message":"logging-example.obj-a.obj-b.do-it-b","data":{"session":"2.1"}}
{"timestamp":"2020-04-27T20:41:40.450979056Z","level":"debug","source":"logging-example","message":"logging-example.obj-a.obj-b.obj-c.do-it-c","data":{"session":"2.1.1"}}
{"timestamp":"2020-04-27T20:41:40.450997785Z","level":"debug","source":"logging-example","message":"logging-example.obj-a.obj-b.obj-c.do-it-c","data":{"session":"2.1.2"}}
```

### Logging some data

Messages are fine, but when we want to log data items in a structured way, we
can use the `lager.Data` parameter to the log methods. We have already seen how
'session' is output in the this structure, but lager will happily print
arbitrary data passed in the log method.

When data is common to a number of logs in a method, we can add the data to the
session to avoid duplication. The data is shown in that session and all derived
sessions too.

In our example, the code looks like this.

```golang
//...

func (a *objA) doItA(logger lager.Logger, target string) {
	logger = logger.Session("obj-a", lager.Data{"target": target})
	logger.Debug("do-it-a")
	a.b.doItB(logger)
}
//...

func main() {
//...

	a.doItA(logger, "foo")
}
```

And here are the resulting logs.

```
{"timestamp":"2020-04-27T20:50:40.180077434Z","level":"debug","source":"logging-example","message":"logging-example.obj-a.do-it-a","data":{"session":"1","target":"foo"}}
{"timestamp":"2020-04-27T20:50:40.180660614Z","level":"debug","source":"logging-example","message":"logging-example.obj-a.obj-b.do-it-b","data":{"session":"1.1","target":"foo"}}
{"timestamp":"2020-04-27T20:50:40.180747403Z","level":"debug","source":"logging-example","message":"logging-example.obj-a.obj-b.obj-c.do-it-c","data":{"session":"1.1.1","target":"foo"}}
{"timestamp":"2020-04-27T20:50:40.180766041Z","level":"debug","source":"logging-example","message":"logging-example.obj-a.obj-b.obj-c.do-it-c","data":{"session":"1.1.2","target":"foo"}}
{"timestamp":"2020-04-27T20:50:40.180785451Z","level":"debug","source":"logging-example","message":"logging-example.obj-a.do-it-a","data":{"session":"2","target":"bar"}}
{"timestamp":"2020-04-27T20:50:40.180803857Z","level":"debug","source":"logging-example","message":"logging-example.obj-a.obj-b.do-it-b","data":{"session":"2.1","target":"bar"}}
{"timestamp":"2020-04-27T20:50:40.180822930Z","level":"debug","source":"logging-example","message":"logging-example.obj-a.obj-b.obj-c.do-it-c","data":{"session":"2.1.1","target":"bar"}}
{"timestamp":"2020-04-27T20:50:40.180841260Z","level":"debug","source":"logging-example","message":"logging-example.obj-a.obj-b.obj-c.do-it-c","data":{"session":"2.1.2","target":"bar"}}
```

### A Refactoring Warning

Suppose now that our app begins to grow and do more work. There will be more methods
where we want to perform logging. Maybe a new method on `objB` would look like this.

```golang
func (b *objB) doAnotherB(logger lager.Logger) {
	logger = logger.Session("obj-b")
	logger.Debug("do-another-b")
}
```

This is fine, and will produce output in line with what we've produced so far.
However, a developer might see repetition in the `logger.Session()` calls, and
might dislike the pattern of passing the logger into every function requiring it.
Why not inject the logger into objects during wiring instead?

Here's the code rewritten in this way. Please don't copy it!

```golang
//...
type objA struct {
	b      *objB
	logger lager.Logger
}

func newA(b *objB, logger lager.Logger) *objA {
	return &objA{
		b:      b,
		logger: logger.Session("obj-a"),
	}
}

func (a *objA) doItA(target string) {
	a.logger.Debug("do-it-a", lager.Data{"target": target})
	a.b.doItB()
	a.b.doAnotherB()
}
//...
```

And here is the output.

```
{"timestamp":"2020-04-27T20:59:52.899927351Z","level":"debug","source":"logging-example","message":"logging-example.obj-a.do-it-a","data":{"session":"3","target":"foo"}}
{"timestamp":"2020-04-27T20:59:52.900503742Z","level":"debug","source":"logging-example","message":"logging-example.obj-b.do-it-b","data":{"session":"2"}}
{"timestamp":"2020-04-27T20:59:52.900558186Z","level":"debug","source":"logging-example","message":"logging-example.obj-c.do-it-c","data":{"session":"1"}}
{"timestamp":"2020-04-27T20:59:52.900599882Z","level":"debug","source":"logging-example","message":"logging-example.obj-c.do-it-c","data":{"session":"1"}}
{"timestamp":"2020-04-27T20:59:52.900654487Z","level":"debug","source":"logging-example","message":"logging-example.obj-b.do-another-b","data":{"session":"2"}}
{"timestamp":"2020-04-27T20:59:52.900689227Z","level":"debug","source":"logging-example","message":"logging-example.obj-a.do-it-a","data":{"session":"3","target":"bar"}}
{"timestamp":"2020-04-27T20:59:52.900782735Z","level":"debug","source":"logging-example","message":"logging-example.obj-b.do-it-b","data":{"session":"2"}}
{"timestamp":"2020-04-27T20:59:52.900842129Z","level":"debug","source":"logging-example","message":"logging-example.obj-c.do-it-c","data":{"session":"1"}}
{"timestamp":"2020-04-27T20:59:52.900889468Z","level":"debug","source":"logging-example","message":"logging-example.obj-c.do-it-c","data":{"session":"1"}}
{"timestamp":"2020-04-27T20:59:52.900935760Z","level":"debug","source":"logging-example","message":"logging-example.obj-b.do-another-b","data":{"session":"2"}}
```

Notice we have lost the session stacking. There is no call-trace in the message names, and
the session numbers cannot be correlated across the calls. If you think about it, it's clear
that we won't see any stack information without passing context about the stack between the 
methods.

Speaking of context - if you are already passing a golang Context object between methods, and
you want to introduce logging, you can piggy-back lager.Sessions within the Context object
and achieve all the tracing capabilities mentioned above. See the [lagerctx package](https://github.com/cloudfoundry/lager/blob/master/lagerctx/context.go) for an implementation.

### Common Patterns

#### Start / End

By including start and end logs when entering a method, it is easy
to see code structure in the logs, and to get simple method timings.

```golang
func DoSomething(logger lager.Logger, id string, ...) {
    logger = logger.Session("do-something", lager.Data{"id": id})
    logger.Debug("start")
    defer logger.Debug("end")
    //...
}
```

#### Naming

Use a name to uniquely identify a method as the session name.
In logs, use a description of the part of the method or output data.

```golang
func (f FileBackingStore) DoSomething(logger lager.Logger, id string, ...) {
    logger = logger.Session("file-backing-store.do-something", lager.Data{"id": id})
    //...
    logger.Info("bytes-written", lager.Data{"count": count})
    //...
}
```

#### Logging errors

Use the `Error()` logging method to include golang error information.

```golang
    logger.Error("my-message", err, lager.Data{"other": info})
```

#### Adding common data without a new session

Use `logger.WithData()` to create a derived logger including new data
without generating a new session.

#### Use Fatal() to log and terminate the program

```
    logger.Fatal("my-message", err, lager.Data{"foo", bar})
```

#### Use lagertest.TestLogger in ginkgo tests

If you want to assert on logs, use the `lagertest.TestLogger` which 
implements the Logger interface. This integrates with the gomega
`gbytes.Say()` method to match on log lines.

```golang
    var logger lager.Logger

    BeforeEach(func() {
        logger = lagertest.NewTestLogger("some-name")
    })

    JustBeforeEach(func() {
        DoIt(logger)
    })

    It("logs something", func() {
        Expect(logger).To(gbytes.Say("my-special-message"))
    })
```
