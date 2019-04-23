---
title: "C# out vars and discards"
date: 2018-09-10T15:50:08+05:30
featuredImage: "posts/c-sharp-out-vars-discards/images/discards.jpg"
---

C# 7 introduces an improved syntax for joining declaration and usage of an `out` variable.

<img src="images/discards.jpg"/>

Consider the following example:

```csharp
int integerVal;
int.TryParse("123", out integerVal);
```

The same can now be re-written in c# 7 as follows:

```csharp
int.TryParse("123", out int integerVal);
```

Notice the `int` right after the `out` keyword, which enables you to declare the variable in the method call itself, instead of a separate declaration before the method call.
Additionally, you can also use an implicitly typed local variable, instead of defining a typed variable.

```csharp
int.TryParse("123", out var integerVal);
```

However a typed declaration is preferable for sake of clarity and readability of code.

Discards
These are temporary, dummy variables, which we intentionally want to leave them unused. Consider the following code:

```csharp
void Main()
{
    Calculate(5, 1, out int sum, out int _);
}

void Calculate(int x, int y, out int sum, out int diff)
{
    sum = x + y;
    diff = x - y;
}
```

Notice the "`_`" as the last parameter to `Calculate` method. Here, we do not intend to use the value of `out diff` and we replace a variable declaration with an "`_`".

Read more about Discards on [.NET docs](https://docs.microsoft.com/en-us/dotnet/csharp/discards)

GoLang also has something similar, by name [blank identifier](https://golang.org/ref/spec#Blank_identifier)
