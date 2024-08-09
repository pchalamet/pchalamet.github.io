---
title: "F# nullable preview"
date: 2024-08-08T10:31:13+01:00
draft: false
---

After poking the F# team (thanks [Vlæd Zá](https://x.com/vzarytovskii) !), I was able to test the nullable feature support [RFC 1060](https://github.com/fsharp/fslang-design/blob/main/RFCs/FS-1060-nullable-reference-types.md).

Why is this a big deal for F# ? After all, we already have different tools to express this. For me, it's all about interop and alignment with the .net platform: C# and BCL. BCL has been annotated and it's incredibly useful in C#. It's also really great to help modeling and ensure your code is doing great - that is only in C#.

### Productivity
Personnaly, this is something I was eargely waiting for. It's something that has been promised for years and unfortunately never completed. F# was lagging behind platform improvements to the point it was ridiculous to invest on - even decided F# was not worth the game anymore (with few exceptions for algorithmic stuffs). Having used C# and nullable extensively, it's a real productivity improvement and a real safety net for developers.

Productivity comes from:
- data models are cleaner and safer (null is now explicit)
- functions are explicit about `null` values
- return values are explicit and `null` as specified
- compiler nullability mismatches are caught at compile time (it's even better with <TreatWarningsAsErrors> on 😋)

Note this does not remove null reference, you can probably easily inject null reference on non-nullable value with little effort (deserialization, Reflection, ...). It only reduces cognitive load and focus design on what's matter.

Now it's baked in F#, it's probably not completely ready but great step forward ! Let's try to compare implementation and usage against C#.

### Declaration
```csharp
// C#
string? name = "toto";

// F#
let name: string | null = "toto"
```

Despite behind nearly the same, there is a strong difference in philosophy imho.

In F#, nullability is supported through the type (`string | null`) whereas in C# nullability is supported through the binding site (field, variable, argument). Probably this is why you can define a nullable type in F#:

```fsharp
type NullableString = string | null
let name: NullableString = "toto"
```

This is something you can't do in C#. Nullability is always defined at the binding site (sorry, I do not have a better word to describe this) - not when declaring the type itself. My view on C# can probably better expressed as (if that was valid C#):
```csharp
string (?name) = "toto";
```

F# is type oriented - this does make sense nullability is baked at type declaration level and not at binding level. I think this sucks a little bit despite this can be seen as a value as well. Looks like reason is F# designers are trying to bring syntax closer to [RFC 1092](https://github.com/fsharp/fslang-design/blob/main/RFCs/FS-1092-anonymous-type-tagged-unions.md) - but hell yeah, I feel it's weird to mix storage and type definition (I've always the feeling nullable is an access problem, not a type declaration one). Moreover, this definitively confusing with distriminated union declarations. I would have probably gone for anonymous nullable type only (with `<type>?` syntax - and less verbose as well). Time for eyes to adjust probably, but you have my point of view.

### Test drive
I have a bunch of projects where NRT would be interesting:
- [PresqueYaml](https://github.com/MagnusOpera/PresqueYaml): a Yaml deserializer, with lot of Reflection usage
- [Terrabuild](https://github.com/MagnusOpera/Terrabuild): a build tool for monorepo. It does use F# Compiler.Service and Reflection for scripts

But before, I've setup a [test project](https://github.com/pchalamet/test-fs-null) to play with this feature. Check the `Makefile`.

#### PresqueYaml
First build: a lot of errors but it's ok as I've not checked `null` for Reflection operation. This can be easily fixed. It's a nice recall operations can lead to `null` and this must be handled. Fixed using nonNull to assert and remove nullability:
```fsharp
let private readMethod (ty: Type) = ty.GetMethod("Read") |> nonNull
```

Also I've assumed reference type can be nullable, again this can be changed easily:
```fsharp
[<Sealed>]
type ListConverter<'T>() =
    inherit YamlConverter<List<'T> | null>()
```

Note there are strange error reporting. Here error is reported on `converterType` line 3 but definitively shall be on line 6:
```fsharp showLineNumbers
override _.CreateConverter (typeToConvert: Type, options:YamlSerializerOptions) =
    let converterType = typedefof<YamlNodeConverter<_>>
    converterType
        .MakeGenericType([| typeToConvert.GetGenericArguments().[0] |])
        .GetConstructor([| |])
        .Invoke([| |])
    :?> YamlConverter
```

This can be rewritten as:
```fsharp
    override _.CreateConverter (typeToConvert: Type, options:YamlSerializerOptions) =
        let converterType = typedefof<YamlNodeConverter<_>>
        let ctor =
            converterType
                .MakeGenericType([| typeToConvert.GetGenericArguments().[0] |])
                .GetConstructor([| |])
                |> nonNull
        ctor.Invoke([| |]) :?> YamlConverter
```

This is probably explained by [this remark in the RFC](https://github.com/fsharp/fslang-design/blob/main/RFCs/FS-1060-nullable-reference-types.md#typeoft-and-typedefoft).

All in all, pretty easy to enable NRT on this project. Note I choose the ugly path (i.e. use `nonNull`) and probably this ought be handled better than throwing NRE.

#### Terrabuild
Terrabuild does use Reflection to bind parameters to script arguments - otherwise it's a full fledge F# project and I do not expect nullable errors to pop up but Reflection ones. 

Obviously, some `obj` have to be converted to `objnull` but nothing dramatic.

An error is annoying tho, there is an exception type in Terrabuild defined like this:
```fsharp
type TerrabuildException(msg, ?innerException: Exception) =
    inherit Exception(msg, innerException |> Option.defaultValue null)

    static member Raise(msg, ?innerException) =
        TerrabuildException(msg, ?innerException=innerException) |> raise
```

For which compilation ends with:
```
Terrabuild.Common/Errors.fs(6,66): warning FS3261: Nullness warning: The type 'Exception' does not support 'null'.
```

Error is not really obvious but here is the reason:\
`innerException` is of type `Exception option` - hence `defaultValue null` is still an `Exception option` - not an `Exception` as expected for `innerException` so the warning.

I fixed it by removing optional argument (moving from option to nullable):
```fsharp
type TerrabuildException(msg, innerException: Exception | null) =
    inherit Exception(msg, innerException)

    static member Raise(msg, ?innerException) =
        TerrabuildException(msg, innerException |> Option.defaultValue null) |> raise
```

### nonNull / if / match
`nonNull` is pretty brutal and throws and NRE if `null`. That's pretty quick and dirty and probably enough for some situation. Not really useful in practice (as the consequence are same) - but better from a type system point of view. This also can act as a marker for unsafe null removal.

Anyway, the correct way is to correctly handle `null` case obviously. For this, you can use `if` but F# is not C#. C# decided to go with the analyzer route and remove `null` on else branch. This is not perfect but works well in practice - despite I won't trust it with mutable instances probably. I feel this is nuked by var default nullability, strange decision. F# decided against that - binding does always carry the type information. I more confident with F# philosophy and feels more robust (nothing more than guts feeling).

But there is another way to remove nullability: use `match`. Beware of order and check for `null` first (otherwise you will get `error FS0026: This rule will never be matched`)
```fsharp
let s: string | null = "toto"
let s =
    match s with
    | null -> "null"
    | s -> s
// s is now non nullable string
```

### The end
I've just played a little bit with this new F# feature - I've barely scratched the surface. All in all, it's great and will enhance dramatically interop with C# and BCL. Regarding the last error I had, I clearly think days of option types are counted (and that's great, there are too many ways to express `null` state). This is why I think `| null` is not great and declaration site would have been better and easier to read (and more in line with C# practice). But ok, this brings F# in line with C# and BCL which is great. Again, eyes will adjust, take my words with a pinch of salt.

For the future, I'd rather go with a big transparent unification of `Nullable<>` and `Option<>` with implicit `null` support (let say `<type>?`). I do not care about struct and reference difference when matching - I only care when designing types. I'm probably just tired of the schism between structs and references as this ought be handled gracefully by compilers.

But hell yeah, congrats to F# team for bringing this long awaited feature !
