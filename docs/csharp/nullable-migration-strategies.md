---
title: Update your codebase to use nullable reference types
description: Choose the best strategy for upgrading your codebase to use nullable reference types.
ms.technology: csharp-null-safety
ms.date: 07/31/2019
---
# Update libraries to use nullable reference types and communicate nullable rules to callers

The addition of [nullable reference types](nullable-references.md) means you can declare whether or not a `null` value is allowed or expected for every variable. In addition, you can apply a number of attributes: `AllowNull`, `DisallowNull`, `MaybeNull`, `NotNull`, `NotNullWhen`, `MaybeNullWhen`, and `NotNullIfNotNull` to completely describe the null states of argument and return values. That provides a great experience as you write code. You get warnings if a non-nullable variable might be set to `null`. You get warnings if a nullable variable isn't null-checked before you dereference it. Updating your libraries can take time, but the payoffs are worth it. The more information you provide to the compiler about *when* a `null` value is allowed or prohibited, the better warnings users of your API will get. Let's start with a familiar example. Imagine your library has the following API to retrieve a resource string:

```csharp
bool TryGetMessage(string key, out string message)
```

The preceding example follows the familiar `Try*` pattern in .NET. There are two reference arguments for this API: the `key` and the `message` parameter. This API has the following rules relating to the nullness of these arguments:

- Callers shouldn't pass `null` as the argument for `key`.
- Callers can pass a variable whose value is `null` as the argument for `message`.
- If the `TryGetMessage` method returns `true`, the value of `message` isn't null. If the return value is `false,` the value of `message` (and its null state) is null.

The rule for `key` can be completely expressed by the variable type: `key` should be a non-nullable reference type. The `message` parameter is more complex. It allows `null` as the argument, but guarantees that, on success, that `out` argument isn't null. For these scenarios, you need a richer vocabulary to describe the expectations.

Updating your library for nullable references requires more than sprinkling `?` on some of the variables and type names. The preceding example shows that you need to examine your APIs and consider your expectations for each input argument. Consider the guarantees for the return value, and any `out` or `ref` arguments upon the method's return. Then communicate those rules to the compiler, and the compiler will provide warnings when callers don't abide by those rules.

This work takes time. Let's start with strategies to make your library or application nullable-aware, while balancing other requirements and deliverables. You'll see how to balance ongoing development enabling nullable reference types. You'll learn challenges for generic type definitions. You'll learn to apply attributes to describe pre- and post-conditions on individual APIs.

## Choose a strategy for nullable reference types

The first choice is whether nullable reference types should be on or off by default. You have two strategies:

- Enable nullable reference types for the entire project, and disable it in code that's not ready.
- Only enable nullable reference types for code that's been annotated for nullable reference types.

The first strategy works best when you're adding other features to the library as you update it for nullable reference types. All new development is nullable aware. As you update existing code, you enable nullable reference types in those classes.

Following this first strategy, you do the following:

1. Enable nullable reference types for the entire project by adding the `<Nullable>enable</Nullable>` element to your *csproj* files.
1. Add the `#nullable disable` pragma to every source file in your project.
1. As you work on each file, remove the pragma and address any warnings.

This first strategy has more up-front work to add the pragma to every file. The advantage is that every new code file added to the project will be nullable enabled. Any new work will be nullable aware; only existing code must be updated.

The second strategy works better if the library is generally stable, and the main focus of the development is to adopt nullable reference types. You turn on nullable reference types as you annotate APIs. When you've finished, you enable nullable reference types for the entire project.

Following this second strategy you do the following:

1. Add the `#nullable enable` pragma to the file you want to make nullable aware.
1. Address any warnings.
1. Continue these first two steps until you've made the entire library nullable aware.
1. Enable nullable types for the entire project by adding the `<Nullable>enable</Nullable>` element to your *csproj* files.
1. Remove the `#nullable enable` pragmas, as they're no longer needed.

This second strategy has less work up-front. The tradeoff is that the first task when you create a new file is to add the pragma and make it nullable aware. If any developers on your team forget, that new code is now in the backlog of work to make all code nullable aware.

Which of these strategies you pick depends on how much active development is taking place in your project. The more mature and stable your project, the better the second strategy. The more features being developed, the better the first strategy.

## Should nullable warnings introduce breaking changes?

Before you enable nullable reference types, variables are considered *nullable oblivious*. Once you enable nullable reference types, all those variables are *non-nullable*. The compiler will issue warnings if those variables aren't initialized to non-null values.

Another likely source of warnings is return values when the value hasn't been initialized.

The first step in addressing the compiler warnings is to use `?` annotations on parameter and return types to indicate when arguments or return values may be null. When reference variables must not be null, the original declaration is correct. As you do this, your goal isn't just to fix warnings. The more important goal is to make the compiler understand your intent for potential null values. As you examine the warnings, you reach your next major decision for your library. Do you want to consider modifying API signatures to more clearly communicate your design intent? A better API signature for the `TryGetMessage` method examined earlier could be:

```csharp
string? TryGetMessage(string key);
```

The return value indicates success or failure, and carries the value if the value was found. In many cases, changing API signatures can improve how they communicate null values.

However, for public libraries, or libraries with large user bases, you may prefer not introducing any API signature changes. For those cases, and other common patterns, you can apply attributes to more clearly define when an argument or return value may be `null`. Whether or not you consider changing the surface of your API, you'll likely find that type annotations alone aren't sufficient for describing `null` values for arguments or return values. In those instances, you can apply attributes to more clearly describe an API.

## Attributes extend type annotations

Several attributes have been added to express additional information about the null state of variables. All code you wrote before C# 8 introduced nullable reference types was *null oblivious*. That means any reference type variable may be null, but null checks aren't required. Once your code is *nullable aware*, those rules change. Reference types should never be the `null` value, and nullable reference types must be checked against `null` before being dereferenced.

The rules for your APIs are likely more complicated, as you saw with the `TryGetValue` API scenario. Many of your APIs have more complex rules for when variables can or can't be `null`. In these cases, you'll use attributes to express those rules. The attributes that describe the semantics of your API are found in the article on [Attributes that impact nullable analysis](./language-reference/attributes/nullable-analysis.md).

## Generic definitions and nullability

Correctly communicating the null state of generic types and generic methods requires special care. This stems from the fact that a nullable value type and a nullable reference type are fundamentally different. An `int?` is a synonym for `Nullable<int>`, whereas `string?` is `string` with an attribute added by the compiler. The result is that the compiler can't generate correct code for `T?` without knowing if `T` is a `class` or a `struct`.

This doesn't mean you can't use a nullable type (either value type or reference type) as the type argument for a closed generic type. Both `List<string?>` and `List<int?>` are valid instantiations of `List<T>`.

What it does mean is that you can't use `T?` in a generic class or method declaration without constraints. For example, <xref:System.Linq.Enumerable.FirstOrDefault%60%601(System.Collections.Generic.IEnumerable%7B%60%600%7D)?displayProperty=nameWithType> won't be changed to return `T?`. You can overcome this limitation by adding either the `struct` or `class` constraint. With either of those constraints, the compiler knows how to generate code for both `T` and `T?`.

You may want to restrict the types used for a generic type argument to be non-nullable types. You can do that by adding the `notnull` constraint on that type argument. When that constraint is applied, the type argument must not be a nullable type.
