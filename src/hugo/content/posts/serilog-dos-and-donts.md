---
title: "Serilog Do's and Don'ts"
date: 2020-07-12T18:08:20-04:00
summary: Some tips on how to effectively use structure logging using Serilog
tags:
- Serilog
- .NET
---

[Serilog](https://serilog.net/) is a structured logger for [.NET](https://dotnet.microsoft.com/), and as such, for folks who are used to traditional console loggers, requires the developer to think about their logs a little more than normal.

This post is not meant to be a how to, but rather a non-comprehensive list of mistakes I've seen over time.

## Templating

One thing to keep in mind is that when writing an event to the log, you are not passing a message, but a *template*. The tags you pass within curly branches do have meaning, and will be used to add data the event. It is therefore important to not format the message before passing logging it.

Consider a logger with a basic console sink

```cs
var textLogger = Log.Logger = new LoggerConfiguration()
    .WriteTo.Console()
    .CreateLogger();
```

With two very basic statements

```cs
textLogger.Information($"Hello {world}");
textLogger.Information("Hello {value}", world);
```

Will output:

```
[18:19:18 INF] Hello World
[18:19:18 INF] Hello World
```

Can you spot the difference? Spoiler: There isn't one! So what's the problem then?

Well, let's use the [JSOM Formatter](https://github.com/serilog/serilog-formatting-compact) instead:

(note: I'm using the [Rendered](https://github.com/serilog/serilog-formatting-compact#rendered-events) formatter so we can see the rendered template within each event)

```cs
var jsonLogger = Log.Logger = new LoggerConfiguration()
    .WriteTo.Console(new RenderedCompactJsonFormatter())
    .CreateLogger();
```

With the same statements

```cs
jsonLogger.Information($"Hello {world}");
jsonLogger.Information("Hello {value}", world);
```

Will give us:

```json
{
	"@t": "2020-07-12T22:26:19.6616913Z",
	"@m": "Hello World",
	"@i": "00ba5a14"
}
{
	"@t": "2020-07-12T22:26:19.6659300Z",
	"@m": "Hello \"World\"",
	"@i": "40c34e37",
	"value": "World"
}
```

When using `textLogger.Information("Hello {value}", world)`, there is now a field called `value` with the argument we passed to the logging statements.
In addition, the `@i` field will have the same value every time this statement will run, so you have a fixed item to search for if you want to search for every value being logged by this particular statement.

### TLDR
**DON'T** format messages before passing them to the logger

## Destructuring

One of the benefit of having a structured logger versus a flat one, is, well, *structure*. You are not limited to logging simple values, you can include entire object graphs. This process is called *Destructuring*

Let's consider the following classes

```cs
public class TestInnerClass
{
    public string SomeOtherProperty { get; set; }
}

public class TestClass
{
    public string SomeProperty { get; set; }
    public TestInnerClass SomeInnerClass { get; set; }
}
```

Let's initialize an instance

```cs
var obj = new TestClass
{
    SomeInnerClass = new TestInnerClass
    {
        SomeOtherProperty = "I am a property within a child"
    },
    SomeProperty = "I am a property at the root level"
};
```

And let's include it a logging statement

```cs
jsonLogger.Information("Useful log for object {object}", obj);
```

This will output

```json
{
	"@t": "2020-07-12T22:43:28.1191964Z",
	"@m": "Useful log for object \"SerilogTest.TestClass\"",
	"@i": "146d94ee",
	"object": "SerilogTest.TestClass"
}
```

What? `SerilogTest.TestClass`? By default, Serilog will simply call `ToString` on your object. To destructure it, simply add `a` before the field in your template

```cs
jsonLogger.Information("Useful log for object {@object}", obj);
```

And the output will be

```json
{
	"@t": "2020-07-12T22:43:28.1525906Z",
	"@m": "Useful log for object TestClass { SomeProperty: \"I am a property at the root level\", SomeInnerClass: TestInnerClass { SomeOtherProperty: \"I am a property within a child\" } }",
	"@i": "99aea052",
	"object": {
		"SomeProperty": "I am a property at the root level",
		"SomeInnerClass": {
			"SomeOtherProperty": "I am a property within a child",
			"$type": "TestInnerClass"
		},
		"$type": "TestClass"
	}
}
```

As you can see, the entire object graph was included within the event. So much contextual information!

But what if there are properties that you do not want to log? After all, it's typically frowned upon to include information like passwords or personally identifiable information within them.

Let's reconfigure the logger to only log `SomeProperty`:

```cs
var jsonLogger = Log.Logger = new LoggerConfiguration()
    .WriteTo.Console(new RenderedCompactJsonFormatter())
    .Destructure.ByTransforming<TestClass>(c => new { c.SomeProperty })
    .CreateLogger();
```

Will give us

```json
{
	"@t": "2020-07-12T22:49:44.7386801Z",
	"@m": "Useful log for object { SomeProperty: \"I am a property at the root level\" }",
	"@i": "99aea052",
	"object": {
		"SomeProperty": "I am a property at the root level"
	}
}
```

We only included the property we wanted!

Note that there are different ways to configure destructuring. A good start in the (awesomely named) [Destructurama!](https://github.com/destructurama]) organization.

### Type conflicts

Just a side note about destructing objects regarding conflicting types. Some log aggregating services will use the parameter names within templates to define types.

An example of this is Elasticsearch.

With the following statement

```cs
jsonLogger.Information("Hello {value}", "world");
jsonLogger.Information("Hello {value}", new { value = "world" });
```

Since the first statement will generate a string field called "value" in the mapping, the second statement will not be logged, as Elasticsearch will complain about trying to fit an object within a string field.

### TLDR
**DO** include object graph when they add contextual information to your statements
**AVOID** including unnecessary objects. They can clutter your logs, and if you're pushing them to external services that charges by the size of logs you send them, it will quite literally cost you.
**DO** Be mindful of conflicts that can happen with your parameter names

## Exceptions

The one thing to remember is that Serilog's methods all have an overload that accepts an `Exception` as the first parameter.

So instead of writing statements including the exception as part of the message template

```cs
var exception = new Exception("Top exception", new Exception("Inner exception"));
jsonLogger.Information(exception, "Exception");
```

Which will include the exception within the `exception` parameter

```json
{
	"@t": "2020-07-12T23:01:14.1165769Z",
	"@m": "Exception! \"System.Exception: Top exception\r\n ---> System.Exception: Inner exception\r\n   --- End of inner exception stack trace ---\"",
	"@i": "e22b2a9a",
	"exception": "System.Exception: Top exception\r\n ---> System.Exception: Inner exception\r\n   --- End of inner exception stack trace ---"
}
```

Simply include the exception as the first argument

```cs
var exception = new Exception("Top exception", new Exception("Inner exception"));
jsonLogger.Information(exception, "Exception");
```

And now all logged exceptions will be within the `@x` field.

```json
{
	"@t": "2020-07-12T23:01:14.1532597Z",
	"@m": "Exception",
	"@i": "7d53da39",
	"@x": "System.Exception: Top exception\r\n ---> System.Exception: Inner exception\r\n   --- End of inner exception stack trace ---"
}
```

Note that this behaviour can change depending on your sink. For example, the [Elasticsearch sink](https://github.com/serilog/serilog-sinks-elasticsearch) will include the exceptions within an object array.

### TLDR
**DO** use Serilog's overloads to pass exceptions instead of using the argument list.

# Conclusion

Those were just a few basic things to keep in mind when using Serilog. I stringly suggest visiting the [Writing log events](https://github.com/serilog/serilog/wiki/Writing-Log-Events) page for more information.
