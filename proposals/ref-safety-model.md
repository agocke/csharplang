# Ref safety model

The ref safety rules of C# are specified in [span safety](csharp-7.2/span-safety.md) and [low level struct improvements](csharp-11.0/low-level-struct-improvements.md). They are written in terms of *safe-context* and *ref-safe-context*, which is how the compiler computes them. Underneath them is a claim that can be stated more simply: a `ref` never outlives the location it refers to. This document describes the design as two layers that separate that claim from everything else.

1. A **type system** in which lifetimes are explicit. A program that type checks in it is safe. This is the only layer that safety depends on.
2. **Inference** that maps C# source, which has no syntax for lifetimes, onto that type system. It has two parts. The *signature translation* gives every type and member declaration its lifetimes from its syntax alone. *Body inference* chooses lifetimes for the locals and calls inside a method.

Inference only produces programs for the type system to check, so it cannot make an unsafe program type check. Its choices decide which safe programs are accepted and where errors are reported. The rules in those proposals can then be read as consequences of the two layers, and [derived rules](#derived-rules) lists them.

This document is not a complete formalization, and its correspondence with the rules as the compiler implements them has not been proven. It is meant to establish a common understanding and vocabulary for discussing the design and potential changes to it.

## Type system

### Lifetimes

The syntax `$a` refers to a lifetime named `a`. A named lifetime has no meaning by itself, but it can be related to other lifetimes with a `where $a : $b` constraint. This establishes that `$a` is convertible to `$b`, which can be thought of as `$a` lasting at least as long as `$b`.

Lifetimes come from three places:

- `$heap` is the lifetime of any value on the heap. It is convertible to every lifetime: `where $heap : $a` for every `$a`.
- Every block in a method body has a lifetime, which is the lifetime of the locals declared in it. The lifetime of a block is convertible to the lifetime of every block nested inside it. `$local` is the lifetime of the outermost block of the current method, *function-member*. The samples use `$local` for every local unless the nesting matters.
- Every other lifetime is a generic parameter of a method or type. Lifetime parameters are written before any type parameters, for example `M<$ro, $cm>`.

Every lifetime parameter that a method can name is convertible to `$local`. A lifetime that the caller supplies always lasts at least as long as the call. `$local` can appear in a method signature, but not in any output position.

### Values, locations and refs

A type does not have a lifetime of its own. Two things do:

- A *value* of a `ref struct` type has a lifetime that limits where the value can flow. This is its *safe-context*, and it is written as the lifetime argument of the type: `S<$a>`. A type parameter that `allows ref struct` is treated the same way. Every other type has no lifetime argument, because its values cannot contain a `ref`: an `int` can be copied anywhere.
- A *location* has a lifetime that limits how long a `ref` to it is valid. This is its *ref-safe-context*. Taking a `ref` to a location whose lifetime is `$a` produces a `ref<$a> T`. For example, a `ref` to an array element is a `ref<$heap> T`.

The two are independent, with one rule connecting them: a value must last at least as long as the location that holds it. In `ref<$a> T<$b>`, `$b` must be convertible to `$a`.

The lifetime of a location is:

| Location | Lifetime |
| --- | --- |
| An array element, a `static` field, or a field `e.F` where `e` is of a reference type | `$heap` |
| A local | The lifetime of the block that declares it |
| A value parameter | `$local` |
| The target of a `ref` local, parameter, field or return of type `ref<$a> T` | `$a` |
| A field `e.F` where `e` is of a struct type | The lifetime of the location `e` |

A `ref` field needs no rule of its own. Its type is `ref<$this> T`, so its target falls under the fourth row and has the `$this` lifetime of `e`.

The samples make the lifetimes of an expression explicit by annotating it, for example `ref<$heap> array[0]` or `default(S<$heap>)`.

### Lifetime parameters on `ref struct` types

A `ref struct` has a single *safe-context*, so in the model every `ref struct` has exactly one lifetime parameter, `out $this`. It is applied to every field that can carry a lifetime:

- A `ref` field has the type `ref<$this> T`.
- A field of `ref struct` type has the type `T<$this>`.

```csharp
ref struct S<out $this>
{
    ref<$this> int RefField;
    Other<$this> StructField;
    int Field;
}
```

The declaration always has this shape, so the parameter can be left off when it is not relevant to the discussion. Nothing is lost by using `$this` for every field: a value with a longer lifetime, such as a `ref<$heap> int`, is convertible to `ref<$this> int`.

`$this` is covariant, and [lifetime variance](#lifetime-variance) explains why that is valid. Covariance supports the existing invariant that `T` can be assigned to `scoped T` for every type. In the model, that means `T<$a>` is assignable to `T<$local>` for every lifetime `$a` convertible to `$local`. It also supports assigning a `Span<T>` from the heap to one on the stack, and similar conversions. 

A single parameter means a type cannot give its fields differing lifetimes, but that is the reality of C# today. Changing it would be a significant change to the C# rules, and [ref fields to ref struct](csharp-11.0/low-level-struct-improvements.md#ref-fields-to-ref-struct) describes what the model would need.

### Lifetime variance
<a name="lifetime-variance"></a>

An assignment or return is legal when the underlying conversion between lifetime types is legal, and the variance of a lifetime parameter determines which conversions are legal. Variance restrictions come from a combination of two factors: whether a location may be aliased, and whether it can be read or written through that alias.

- A location that may be aliased and only allows reading is covariant.
- A location that may be aliased and only allows writing is contravariant.
- A location that may be aliased and allows both reading and writing is invariant.
- A value that cannot be aliased, because it is always copied, can only be read from the point of view of its source and hence is covariant.
- A parameter that is not used at all is bivariant.

For conversions a `ref` is considered part of the type of the expression: `ref<$a> T<...>` is logically `ref<$a, T<...>>`, where `$a` is covariant and `T` is invariant. The rules above explain that variance. A `ref` can be thought of as a primitive `ref struct` that contains a pointer:

```csharp
ref struct ByRef<out $a, T>
{
    public T Value { get; set; }
}
```

The lifetime `$a` describes the `ref` itself, how long the referenced location remains valid, not the value it refers to. A `ref` is copied when it is assigned so `$a` is covariant: a `ref<$heap> int` can be converted to a `ref<$local> int` because a location that is valid for `$heap` is also valid for `$local`. The reverse conversion would allow a `ref` to a stack location to be treated as a `ref` to the heap and is illegal.

The type `T`, including its lifetimes, sits behind the pointer. That location is aliased and can be both read and written through the `ref`, so `T` is invariant. This is rule 2 of [ref reassignment](csharp-11.0/low-level-struct-improvements.md#rules-ref-reassignment): in `e1 = ref e2` the *safe-context* of `e1` must equal the *safe-context* of `e2`. 

```csharp
static void M<$ro, $cm>(ref<$ro> Span<$cm> span)
{
    // error: if T were covariant this would be legal and the next line would write a 
    // Span<$local> into a location the caller views as Span<$cm>
    ref<$local> Span<$local> r = ref span;
    r = stackalloc int[42];
}
```

The value behind a `ref readonly` can only be read, so by the rules above `T` could soundly be covariant in `ref readonly<$a> T`. The language does not take advantage of this for ref reassignment: rule 2 applies to `ref readonly` the same as it does to `ref`. It does take advantage of it in [method arguments must match](csharp-11.0/low-level-struct-improvements.md#rules-method-arguments-must-match), which considers `ref` arguments of `ref struct` type but not `in` arguments, because the callee cannot write a value through them.

A `ref struct` is an aggregate of its fields, so the variance of its lifetime parameters follows from how its fields use them. A lifetime parameter can only be covariant if every field uses it in a covariant position. This is why `$this` can be declared `out $this`: `ref` fields have the form `ref<$this> T`, which uses `$this` in the covariant `$a` position of `ref<$a, T>`, and fields of `ref struct` type have the form `T<$this>`, which is covariant by the same rule. `$this` never appears behind a pointer, which would be an invariant position, because a `ref` field cannot refer to a `ref struct`.

### Methods

A method declares lifetime parameters and constraints between them. Inside the method an assignment, a `return` or an argument is legal when the conversion between the lifetime types is legal.

A call supplies a lifetime for each lifetime parameter of the method. Every argument must convert to the type of its parameter once those lifetimes are filled in, and the result of the call has the return type with the same lifetimes filled in.

One signature can stand in for another, as in an override, an interface implementation or a delegate conversion, when it accepts every argument the other accepts and returns only what the other can return. Parameter types are contravariant and the return type is covariant.

### Safety

A location is live for at least as long as its lifetime. The property that the type system is meant to guarantee is:

> In a program that type checks, whenever a `ref<$a> T` is read or written the location it refers to is still live. The same holds for every `ref` contained in a value of type `S<$a>`.

The argument has the following shape. It is a sketch, not a proof, and it does not cover `unsafe` code.

- A `ref` is created with the lifetime of its location, by the table above.
- A conversion can shorten the lifetime of a `ref` but never lengthen it, because that lifetime is covariant.
- A conversion cannot change the lifetime of anything behind a pointer, because that position is invariant. A location therefore never holds a value that lasts for less time than some view of the location claims.
- A value cannot be stored in a location that outlives it, because the value must last at least as long as the location.
- A value leaves a method only through its return or through a `ref` or `out` parameter. The types of those name only lifetimes the caller supplied, and `$local` cannot appear in them, so nothing that refers to the stack of the method can leave it.

## Signature translation

The lifetimes of a declaration are fixed by its syntax alone. They never depend on the body of a method, so a caller can be checked against a signature without seeing the implementation.

A `ref struct` declaration is given the lifetime parameter `out $this`, with every `ref` field typed `ref<$this> T` and every field of `ref struct` type typed `T<$this>`.

A method is given two lifetime parameters, named `$ro` for *return-only* and `$cm` for *caller-context*, with the constraint `where $cm : $ro`. The constraint is left off the samples for brevity. The parameters and return of the method are then typed as follows:

- `ref` and `in` parameters have a ref lifetime of `$ro`.
- Parameters of `ref struct` type have a value lifetime of `$cm`.
- `ref` returns have a ref lifetime of `$ro`.
- Returns of `ref struct` type have a value lifetime of `$ro`.
- `scoped` changes the lifetime it applies to into `$local`: the ref lifetime for `scoped ref`, and the value lifetime for a `scoped` parameter of `ref struct` type.
- `out` parameters are implicitly `scoped`, and as an output of the method a `ref struct` value has a value lifetime of `$ro`: `out S s` maps to `ref<$local> S<$ro> s`.
- `[UnscopedRef]` removes an implicit `scoped`. On a `ref` parameter, which has none, it raises the ref lifetime from `$ro` to `$cm`.

Instance methods and constructors need no rules of their own. Both are translated to `static` methods with an explicit `this` parameter, and the list above gives its type:

- In an instance method of a `struct`, `this` is an implicitly `scoped` `ref` parameter: `ref<$local> S<$cm> this`. With `[UnscopedRef]` it is `ref<$ro> S<$cm> this`.
- In a constructor, `this` is an `out` parameter, as described in [modeling constructors](csharp-11.0/low-level-struct-improvements.md#modeling-constructors): `ref<$local> S<$ro> this`. A `new` expression is annotated with the type of the value that the constructor produces, for example `new S<$heap>(...)`.

```csharp
ref struct S
{
    S(ref int i) { }
    void M(ref S s) { }
}

// Maps to 

ref struct S<out $this>
{
    static void ctor<$ro>(ref<$local> S<$ro> this, ref<$ro> int i) { }
    static void M<$ro, $cm>(ref<$local> S<$cm> this, ref<$ro> S<$cm> s) { }
}
```

The translation is a choice of defaults, and safety does not depend on it. For example, a translation with a single lifetime parameter, where `ref` parameters and `ref struct` parameters both use `$cm`, is also safe. With it a method can store a `ref` parameter into a `ref struct` parameter, so every call that passes a `ref` to a local alongside a `ref struct` that lasts longer has to be an error. Giving `ref` parameters the shorter lifetime `$ro` makes the store an error inside the one method that attempts it and keeps those calls legal. That is the purpose of the *return-only* context.

## Body inference

Inside a method the lifetimes that the source does not state are chosen as follows. When no choice type checks, the program is in error.

**Locals.** The location of a local has the lifetime of the block that declares it. The ref lifetime of a `ref` local and the value lifetime of a local of `ref struct` type are taken from the initializer. A `scoped` local uses the lifetime of its block instead.

**Calls.** The caller picks the longest lifetimes for which every argument converts to the type of its parameter:

- `$cm` can be no longer than the value lifetime of any `ref struct` argument, because `S<$a>` converts to `S<$cm>` only when `$a` converts to `$cm`.
- `$ro` can be no longer than `$cm`, or than the ref lifetime of any `ref` argument.
- An argument to a `scoped` parameter adds no constraint. Every lifetime in the caller converts to the `$local` of the method being called.
- A `ref struct` argument passed by `ref` sits behind a pointer, which is an invariant position, so its value lifetime must be exactly `$cm`. An `out` argument is only written, so `$ro` must convert to its value lifetime. An `in` argument adds no such constraint, as described in [lifetime variance](#lifetime-variance).

The result of the call has the return type with those lifetimes filled in, so a returned `ref` or `ref struct` has the chosen `$ro`. A variable declared by an `out` argument, as in `M(out var x)`, has a value lifetime of the chosen `$ro`.

**Object initializers.** A member initializer that assigns to the object being created is treated as an additional argument to the constructor.

## Derived rules

The rules in [low level struct improvements](csharp-11.0/low-level-struct-improvements.md) follow from the layers above. This table gives the reason for each one informally. 

| Rule | Follows from |
| --- | --- |
| [Field lifetimes](csharp-11.0/low-level-struct-improvements.md#rules-field-lifetimes) | The table of location lifetimes. A `ref` field is typed `ref<$this> T`, so its target has the value lifetime of the containing expression. |
| [Ref reassignment](csharp-11.0/low-level-struct-improvements.md#rules-ref-reassignment), rule 1 | The lifetime of a `ref` is covariant. |
| [Ref reassignment](csharp-11.0/low-level-struct-improvements.md#rules-ref-reassignment), rule 2 | The type behind a pointer is invariant. |
| [`scoped`](csharp-11.0/low-level-struct-improvements.md#rules-scoped) | The translation to `$local`, which cannot appear in an output position and is not convertible to any lifetime parameter. |
| [`out` parameters](csharp-11.0/low-level-struct-improvements.md#out-compat-change) | The translation of `out S s` to `ref<$local> S<$ro> s`. The argument is only written, so its previous value adds no constraint to the call. |
| [Return-only](csharp-11.0/low-level-struct-improvements.md#return-only) | The translation gives a method both `$ro` and `$cm` with `where $cm : $ro`. A returned expression must convert to the return type, which uses `$ro`. |
| [Method invocation](csharp-11.0/low-level-struct-improvements.md#rules-method-invocation) | Body inference picks the longest lifetimes for a call, so the result is limited by the narrowest argument that contributes. |
| [Declaration expressions](csharp-11.0/low-level-struct-improvements.md#infer-safe-to-escape-of-declaration-expressions) | A variable declared by an `out` argument receives a value of type `S<$ro>`. |
| Object initializers | Member initializers are constructor arguments. |
| [Method arguments must match](csharp-11.0/low-level-struct-improvements.md#rules-method-arguments-must-match) | A `ref struct` passed by `ref` is in an invariant position. The call is an error when no choice of `$cm` matches every such argument. |
| [Parameter scope variance](csharp-11.0/low-level-struct-improvements.md#scoped-mismatch) | Parameter types are contravariant. Adding `scoped` replaces a lifetime with `$local`, to which every lifetime converts, so the new signature accepts every argument the old one did. |
| [`[UnscopedRef]`](csharp-11.0/low-level-struct-improvements.md#rules-unscoped) | The translation: it removes an implicit `scoped`, or raises `$ro` to `$cm`. |
| [Cyclic assignment](csharp-11.0/low-level-struct-improvements.md#cyclic-assignment) and capturing a `ref` parameter | `$ro` is not convertible to `$cm`. See the examples below. |

## Examples

A method that returns a `ref` parameter:

```csharp
ref int Identity(ref int i) => ...

// Maps to 

ref<$ro> int Identity<$ro>(ref<$ro> int i)
{
    // okay: the ref lifetime $ro is the same as the return lifetime $ro
    return ref i;

    // okay: the ref lifetime $heap is convertible to $ro
    int[] array = new int[42];
    return ref array[0];

    // error: the ref lifetime $local is not convertible to $ro
    int local = 42;
    return ref local;
}
```

The same example using a `ref struct`: 

```csharp
ref struct S
{
    ref int Field;

    S(ref int f)
    {
        Field = ref f;
    }
}

S M2(ref int i, S span1, scoped S span2) => ...

// Maps to 

ref struct S<out $this>
{
    // Implicitly 
    ref<$this> int Field;

    static void ctor<$ro>(ref<$local> S<$ro> this, ref<$ro> int f)
    {
        // okay: this.Field is ref<$ro> int and ref f is ref<$ro> int
        this.Field = ref f;
    }
}

S<$ro> M2<$ro, $cm>(
    ref<$ro> int i,
    S<$cm> span1,
    S<$local> span2)
{
    // okay: $cm is convertible to $ro and the lifetime parameter of S is covariant
    return span1;

    // error: $local is not convertible to $ro
    return span2;

    // okay: $heap is convertible to $ro and the lifetime parameter of S is covariant
    return default(S<$heap>);

    // okay: the ref lifetime of i is $ro so this is an identity conversion
    S<$ro> s = new S<$ro>(ref i);
    return s;

    int[] array = new int[42];
    // okay: S<$heap> is convertible to S<$ro>
    return new S<$heap>(ref<$heap> array[0]);

    // okay: the constructor parameter is ref<$ro> int and the argument is ref<$heap> int, 
    // which is convertible
    return new S<$ro>(ref<$heap> array[0]);

    // error: the ref lifetime of local is $local, so the constructed value is S<$local>, 
    // which is not convertible to S<$ro>
    int local = 42;
    return new S<$local>(ref<$local> local);
}
```

The cyclic self assignment problem:

```csharp
ref struct S
{
    int field;
    ref int refField;

    static void SelfAssign(ref S s)
    {
        s.refField = ref s.field;
    }
}

// Maps to 

ref struct S<out $this>
{
    int field;
    ref<$this> int refField;

    static void SelfAssign<$ro, $cm>(ref<$ro> S<$cm> s)
    {
        // error: this is ref<$cm> int = ref<$ro> int, which is illegal because $ro is not 
        // convertible to $cm (the relationship goes the other direction)
        s.refField = ref<$ro> s.field;
    }
}
```

The silly capture parameter problem: 

```csharp
ref struct S
{
    ref int refField;

    void Use(ref int parameter)
    {
        // error: this needs to be an error, otherwise every call to this.Use(ref local) 
        // would fail because the compiler would assume the `ref` was captured by ref.
        this.refField = ref parameter;
    }
}

// Maps to 

ref struct S<out $this>
{
    ref<$this> int refField;
    
    // Using static form of this method signature so the type of this is explicit. 
    static void Use<$ro, $cm>(ref<$local> S<$cm> @this, ref<$ro> int parameter)
    {
        // error: refField is ref<$cm> int and ref parameter is ref<$ro> int. $ro is not 
        // convertible to $cm so this reassignment is illegal.
        @this.refField = ref<$ro> parameter;
    }
}
```

