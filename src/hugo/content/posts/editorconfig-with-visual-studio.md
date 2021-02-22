---
title: Editorconfig integration with Visual Studio and .NET
date: 2021-02-21T19:01:55-05:00
date: 2021-02-21T19:01:55-05:00
summary: Taking advantage of Editorconfig to define your coding styles with Visual Studio
tags:
- visual studio
- .net
keywords: editorconfig, visual studio, visual studio code, .net
---

If you are not familiar with [EditorConfig](https://editorconfig.org/) files, you're missing out

Today, I'll walk you through some of the basic features, as well as some extensions to the format supported by Visual Studio and the .NET tools.

# The basics

Editorconfig is a way to define some coding styles by adding files called `.editorconfig` to your code repository.
There's pretty wide support from editors out there that supports it either natively, like [Visual Studio](https://docs.microsoft.com/en-us/visualstudio/ide/create-portable-custom-editor-options?view=vs-2019), or through add-ons, like [Visual Studio Code](https://marketplace.visualstudio.com/items?itemName=EditorConfig.EditorConfig).

## Hierarchy

Editorconfig us a hierarchical configuration for code style, meaning that when a code file is open, it will apply every settings from `.editorconfig` files it finds, from the file's folder, up to the root of the file system.

The closer to the code file the config is, the higher the priority.

You can make prevent scanning parent directories by specifying `root = true` in your `.editorconfig file`.

## Syntax

Editorconfig files are split into sections. Each section starts with a file selector, as determined by square brackets `[]`.

This selector can apply to every file like `[*]`, a simple wildcard like `[*.json]`, or a list of extensions like `[*.{cs, vb}]`

Consider the following example:

```
root = true

[*]
indent_style = space

[*.{cs,vb}]
indent_size = 4
insert_final_newline = true
charset = utf-8-bom
```

First, we have `root = true`, meaning that we won't be scanning any parent folders for more `.editorconfig` files.

Then, we have a `[*]` block that applies to every file, saying we will be using [space](https://www.youtube.com/watch?v=SsoOG6ZeyUI) to indent.

And last, we have the `[*.{cs,vb}]` block that will apply to all files with a `.cs` or `.vb` extension.
Those files will have a indent of 4 (using spaces since the content of the `[*]` block is still being applied), we want to ensure the presence of an empty line at the end of each file, and we are using the UTF-8 character set with the byte order mark.

Tip: If using git, you will want to make sure that your [gitattribute](https://git-scm.com/docs/gitattributes) don't conflict with your `.editorconfig` settings, otherwise your editor might trying to undo what git is doing on checkout, causing a bunch of changes that aren't wanted.

## .NET specific extensions

With the Roslyn compiler, there's a number of extensions to the Editorconfig format that allows to configure code analysis.

For example, you can use Editorconfig to tell whether using clauses should be sorted with the `System` namespace first

```
[*.{cs,vb}]
dotnet_sort_system_directives_first = true
```

You want contributors to use `var` for C# built-in types? Done!

```
csharp_style_var_for_built_in_types = true
```

You can even make it an error:
```
csharp_style_var_for_built_in_types = true:error
```

You can also configure casing, as well as prefix and suffix. This will give a warning for any fields that are not camel case, and not prefixed with an underscore `_`.

```
dotnet_naming_rule.instance_fields_should_be_camel_case.severity = warning
dotnet_naming_rule.instance_fields_should_be_camel_case.symbols = instance_fields
dotnet_naming_rule.instance_fields_should_be_camel_case.style = instance_field_style
dotnet_naming_symbols.instance_fields.applicable_kinds = field
dotnet_naming_style.instance_field_style.capitalization = camel_case
dotnet_naming_style.instance_field_style.required_prefix = _
```

See the [official documentation](https://docs.microsoft.com/en-us/dotnet/fundamentals/code-analysis/overview) for more options. The [Roslyn .editorconfig] also has number of good examples.

## Configuring code analysis

If you're familiar with Visual Studio and Code Analysis, you might have had to configure it through xml via [.ruleset](https://docs.microsoft.com/en-us/visualstudio/code-quality/using-rule-sets-to-group-code-analysis-rules?view=vs-2019) files.

That's no longer necessary, as it can be done through Editorconfig!

All you need is a line that reads `dotnet_diagnostic.<Code>.severity = <Severity>`, such as:

```
[*.{cs,vb}]
dotnet_diagnostic.RCS1007.severity = error
```

# Conclusion

Whenever you a small or a large team, having common code styles in your project can help minimize head scratching when reading a teammate's code (or let's be honest, your past self's code as well).
With Editorconfig, you have a standard way that's not tied to specific technologies to help make those styling standard happens.

What coding styles do you apply to your project?