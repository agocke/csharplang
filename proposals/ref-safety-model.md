# Ref safety model

The ref safety rules of C# are specified in [span safety](csharp-7.2/span-safety.md) and [low level struct improvements](csharp-11.0/low-level-struct-improvements.md). They are written in terms of *safe-context* and *ref-safe-context*, which is how the compiler computes them. Underneath them is a claim that can be stated more simply: a `ref` never outlives the location it refers to. This document describes the design as two layers that separate that claim from everything else.

1. A **type system** in which lifetimes are explicit. The claim is that a program that type checks in it is safe. This is the only layer that safety depends on.
2. **Inference** that maps C# source, which has no syntax for lifetimes, onto that type system. It has two parts. The *signature translation* gives every type and member declaration its lifetimes from its syntax alone. *Body inference* chooses lifetimes for the locals and calls inside a method.

Inference only produces programs for the type system to check, so if the claim holds, inference cannot make an unsafe program type check. Its choices decide which safe programs are accepted and where errors are reported. The rules in those proposals can then be read as consequences of the two layers, and [derived rules](#derived-rules) lists them.

This document is not a complete formalization, and its correspondence with the rules as the compiler implements them has not been proven. It is meant to establish a common understanding and vocabulary for discussing the design and potential changes to it.

## Type system

### Lifetimes

A lifetime is a span of the program's execution, and the syntax `$a` refers to a lifetime named `a`. The lifetime `$a` is convertible to `$b`, written `$a : $b`, when `$a` lasts at least as long as `$b`. Every lifetime is convertible to itself, and if `$a : $b` and `$b : $c` then `$a : $c`.

Lifetimes come from three places:

- `$heap` is the whole execution of the program. It is the lifetime given to every location on the heap, because the garbage collector keeps such a location alive for as long as anything refers to it. It is convertible to every lifetime: `$heap : $a` for every `$a`.
- Every block in a method body has a lifetime, which is the lifetime of the locals declared in it. It is one execution of the block: each call of the method, and each iteration of a loop, is a separate lifetime that the type system refers to by the same name. The lifetime of a block is convertible to the lifetime of every block nested inside it. `$local` is the lifetime of the outermost block of the current method, *function-member*. The samples use `$local` for every local unless the nesting matters.
- Every other lifetime is a generic parameter of a method or type. Lifetime parameters are written before any type parameters, for example `M<$ro, $cm>`. A lifetime parameter stands for whatever lifetime the caller supplies. Inside the declaration nothing is known about it except its `where $a : $b` constraints and the rule below.

Every lifetime parameter that a method can name is convertible to `$local`. A lifetime that the caller supplies always lasts at least as long as the call. `$local` can appear in a method signature, but not in an output position: the return type, or the type of the target of a `ref` or `out` parameter. In a signature it means the `$local` of the method being declared, which a caller has no name for.

### Locations and types

Only locations have lifetimes. A location is a piece of storage, and its lifetime is the span during which that storage exists. It is determined by where the location is:

| Location | Lifetime |
| --- | --- |
| An array element, a `static` field, or a field `e.F` where `e` is of a reference type | `$heap` |
| A local | The lifetime of the block that declares it |
| A value parameter | `$local` |
| Memory allocated by `stackalloc` | `$local`. It is released when the method returns, not when the block ends. |
| A temporary that holds the value of an expression | The lifetime of the block that contains the expression |
| A field `e.F` where `e` is of a struct type | The lifetime of the location `e` |

Values and types do not have lifetimes. A type whose values can refer to locations instead takes a lifetime *argument*, which is a lower bound on the lifetimes of those locations:

- `ref<$a> T` is the type of a `ref` to a location that is live for at least `$a`. Taking a `ref` to a location whose lifetime is `$a` produces a `ref<$a> T`. For example, a `ref` to an array element is a `ref<$heap> T`.
- `S<$a>`, where `S` is a `ref struct`, is the type of a value in which every `ref` is to a location that is live for at least `$a`. A type parameter that `allows ref struct` is treated the same way.
- Every other type takes no lifetime argument, because its values cannot contain a `ref`: an `int` can be copied anywhere.

The argument of a `ref` is called its *ref lifetime* and the argument of a `ref struct` type is called its *value lifetime*. In `ref<$a> S<$b>`, `$a` is the ref lifetime and `$b` is the value lifetime.

The argument is a bound, not a property of the value, so one value can have more than one type: a `ref<$heap> int` is also a `ref<$local> int`. [Lifetime variance](#lifetime-variance) describes when this is allowed.

The target of a `ref<$a> T` is a location whose actual lifetime is not known. Only the bound is, so the type system treats the target as a location with the lifetime `$a`. A `ref` field needs no rule of its own: its type is `ref<$this> T`, so its target is treated as having the `$this` argument of the containing value.

One rule connects locations to the values stored in them. A location must not outlive anything its contents refer to, so a location with the lifetime `$a` can only hold a value of type `T<$b>` when `$b` is convertible to `$a`. This is why `$b` must be convertible to `$a` in `ref<$a> T<$b>`.

These correspond to the terms used by the rest of the rules. The *ref-safe-context* of an expression is the lifetime of the location it denotes, and the *safe-context* of an expression is the value lifetime of its type. An expression whose type takes no lifetime argument has no restriction, which those rules call a *safe-context* of *caller-context*.

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

- The type of a location that may be aliased and only allows reading is a covariant position.
- The type of a location that may be aliased and only allows writing is a contravariant position.
- The type of a location that may be aliased and allows both reading and writing is an invariant position.
- The type of a value that cannot be aliased, because it is always copied, is a covariant position. From the point of view of its source the value can only be read.
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

The target of an `out` parameter is the opposite case. The method being called must assign it before reading it, which definite assignment enforces, so the caller's location is only written. Its type is therefore a contravariant position for the argument: a method that writes an `S<$a>` can be given a location that holds an `S<$b>` when `$a` is convertible to `$b`.

A `ref struct` is an aggregate of its fields, so the variance of its lifetime parameters follows from how its fields use them. A lifetime parameter can only be covariant if every field uses it in a covariant position. This is why `$this` can be declared `out $this`: `ref` fields have the form `ref<$this> T`, which uses `$this` in the covariant `$a` position of `ref<$a, T>`, and fields of `ref struct` type have the form `T<$this>`, which is covariant by the same rule. `$this` never appears behind a pointer, which would be an invariant position, because a `ref` field cannot refer to a `ref struct`.

### Methods

A method declares lifetime parameters and constraints between them. Inside the method an assignment, a `return` or an argument is legal when the conversion between the lifetime types is legal.

A call supplies a lifetime for each lifetime parameter of the method. The lifetimes supplied must satisfy the constraints of the method. Every argument must convert to the type of its parameter once those lifetimes are filled in, and the result of the call has the return type with the same lifetimes filled in.

One signature can stand in for another, as in an override, an interface implementation or a delegate conversion, when it accepts every argument the other accepts and returns only what the other can return. Parameter types are contravariant and the return type is covariant.

### Safety

The property that the type system is meant to guarantee is:

> In a program that type checks, whenever a `ref` is read or written the location it refers to is still live. This includes every `ref` contained in a value of a `ref struct` type.

The argument has the following shape. It is a sketch, not a proof. It does not cover `unsafe` code, and it relies on definite assignment for `out` parameters.

- A `ref` is created with the lifetime of its location as its ref lifetime, by the table above. Its bound is true when it is created.
- A conversion can shorten a ref lifetime or a value lifetime but never lengthen it, because those positions are covariant. A bound that was true stays true.
- A conversion cannot change the type of anything behind a pointer, because that position is invariant. Every alias of a location therefore agrees on the bounds of the value stored in it, and a write through one alias cannot break the bound that another alias reads.
- A location with the lifetime `$a` only holds values whose lifetime arguments are convertible to `$a`. A location therefore never holds a `ref` to a location that ends before it does. This applies to the heap as well: a location with the lifetime `$heap` cannot hold a `ref` to the stack.
- When a block ends, the only values that can refer to its locals have a lifetime argument no longer than the block. By the previous point they are stored only in locations that end with the block or before it.
- A value leaves a method only through its return, or by being written to the target of a `ref` or `out` parameter. The types of those are output positions, which name only lifetimes the caller supplied. `$local` and the lifetimes of nested blocks are not convertible to any of them, so nothing that refers to the stack of the method can leave it.

## Signature translation

The lifetimes of a declaration are fixed by its syntax alone. They never depend on the body of a method, so a caller can be checked against a signature without seeing the implementation.

A `ref struct` declaration is given the lifetime parameter `out $this`, with every `ref` field typed `ref<$this> T` and every field of `ref struct` type typed `T<$this>`.

A method is given two lifetime parameters, named `$ro` for *return-only* and `$cm` for *caller-context*, with the constraint `where $cm : $ro`. The constraint is left off the samples for brevity. The parameters and return of the method are then typed as follows:

- `ref` and `in` parameters have a ref lifetime of `$ro`.
- Parameters of `ref struct` type have a value lifetime of `$cm`.
- `ref` returns have a ref lifetime of `$ro`.
- Returns of `ref struct` type have a value lifetime of `$ro`. A `ref` return of `ref struct` type is the exception: its target is behind a pointer, so it has the same value lifetime as the `ref` parameters it can come from, `$cm`. `ref S M(...)` maps to `ref<$ro> S<$cm> M(...)`.
- `scoped` changes the lifetime it applies to into `$local`: the ref lifetime for `scoped ref`, and the value lifetime for a `scoped` parameter of `ref struct` type.
- `out` parameters are implicitly `scoped`, and as an output of the method a `ref struct` value has a value lifetime of `$ro`: `out S s` maps to `ref<$local> S<$ro> s`.
- `[UnscopedRef]` removes an implicit `scoped`. On a `ref` parameter, which has none, it raises the ref lifetime from `$ro` to `$cm`.

`scoped` and `[UnscopedRef]` add no rules to the type system. They only select which lifetime a declaration is given, and everything they allow or prevent follows from type checking the result. A `scoped ref` parameter cannot be captured because `$local` is not convertible to any lifetime in an output position. An `[UnscopedRef] ref` parameter can be stored in a `ref struct` parameter because both have the lifetime `$cm`.

The annotations are still part of the signature of a member. They are emitted to metadata, and they are compared when a member overrides or implements another or is converted to a delegate type.

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

The translation is a choice of defaults, and safety does not depend on it. For example, a translation with a single lifetime parameter, where `ref` parameters and `ref struct` parameters both use `$cm`, is also safe. With it a method can store a `ref` parameter into a `ref struct` that is passed by `ref`, so every call that passes a `ref` to a local alongside a `ref` to a `ref struct` that lasts longer has to be an error. Giving `ref` parameters the shorter lifetime `$ro` makes the store an error inside the one method that attempts it and keeps those calls legal. That is the purpose of the *return-only* context.

## Body inference

Inside a method the lifetimes that the source does not state are chosen as follows. When no choice type checks, the program is in error.

The lifetimes in scope at any point in a method are totally ordered: `$heap`, then `$cm`, `$ro`, `$local`, and the lifetime of each enclosing nested block in turn. "The longest lifetime" that satisfies a set of constraints is therefore well defined.

**Locals.** The location of a local has the lifetime of the block that declares it. The ref lifetime of a `ref` local and the value lifetime of a local of `ref struct` type are taken from the initializer. A `scoped` local uses the lifetime of its block instead. A local of `ref struct` type with no initializer has a value lifetime of `$cm`.

A value lifetime taken from an initializer is never longer than `$cm`. The rules as the compiler implements them do not distinguish `$heap` from `$cm` inside a method, and nothing in a method requires a `ref struct` value to be `$heap`. Without this limit a local initialized with `default` would be an `S<$heap>`, and passing it by `ref` together with a parameter of type `S<$cm>` would fail the exact match that an invariant position requires.

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

