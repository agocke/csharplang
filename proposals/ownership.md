# C# Ownership

Ownership systems have recently achieved popularity through the Rust language. Ownership is the feature which lets Rust achieve its primary claim to fame: memory safety without a garbage collector / automatic memory manager.

While making memory management safe without a GC is certainly a great use case, there are many other interesting use cases for an owner system. However, to understand them it is still useful to understand ownership in the context of memory management. Therefore, this document will start with memory management and move to other use cases.

To understand Rust, it's helpful to understand the basic design of C++ memory management, the flaws that lead to unsafety, and the ways that ownership systems prevent safety problems.

First, the C++ memory management system. The most basic system is one from C: malloc and free. The simple rules are as follows:

1. All mallocs must be followed by at most one free.
2. Freed references must not be used (dereferenced) after free.
3. All access must fall in the range of memory initially allocated.

In real languages like Rust, C, or C++ there are in fact many more rules needed to ensure memory safety, but for our purposes these are sufficient. In fact, the first two are our main concern.

These rules seem simple but are very difficult to follow in practice. The core problem is _aliasing_. In real programs, variable reference doesn't follow a simple linear path through the method. Pointers are not assigned to a single variable — numerous copies are made and dereferenced both inside the method and inside the various helper methods that are called by both the original function and the helper functions themselves. At each point confusion can arise. It can be unclear whether the callee function or caller function is meant to free the variable. And it can be equally confusing to determine when all aliased uses are complete and the variable can be freed. Worse, the aliases are not invalidated after free, so it is easy for functions to mistakenly believe their alias is still valid.

The main innovation on this front came from C++ in the form of RAII. Put simply, RAII is a system where variables are automatically, and transitively, freed at the end of variable scope. The most important thing is that it ties the variable's memory to its lexical scope. This cleanly removes confusion about where and when a variable is freed — it is freed by the allocating `unique_ptr`, and it is freed at the end of lexical scope. `unique_ptr` makes it possible to call all methods directly, as though accessing a standard pointer. However, it prohibits copying, so new `unique_ptr` aliases cannot be created. C++ references (`&`) and raw pointers may still be created for temporary access in helper functions.

This is a huge improvement for C++ but it has one huge drawback — it is entirely advisory. There are many ways to directly access the underlying pointer and many ways to violate the rules, either by accessing a pointer after the `unique_ptr` lifetime has ended or by freeing a raw pointer that was managed by a `unique_ptr`.

What Rust tried to solve is enforcement. The `unique_ptr` pattern is practical and effective, but code should not be able to violate the rules. And, equally important, this validation should happen at compile time, with no runtime overhead. The result is a series of type system features and a rule enforcement system called the "borrow checker."

The Rust ownership type system rules look very similar to C++ `unique_ptr` rules, but enforced by the language at compile time. The basic rules are:

1. All values have a unique "owning" variable.
2. When the owning variable exits scope, all owned values are destroyed.
3. Ownership may be transferred to a new owning reference, but cannot be shared.
4. Owning variables may produce "borrowed" references that may not destroy the borrowed value and whose lifetime cannot exceed the lifetime of the owned variable.

There are more rules, especially around mutability, but they will be examined in more detail later. There are also specific details around how these rules are implemented by Rust — these are mostly not relevant to the C# design and will be described inline if needed.

## Motivation

The obvious question a C# programmer might ask is: why should I want ownership rules? What do I do with them? This is a good question to ask. After all, Rust designed the ownership system to provide memory safety without a memory management runtime. C# already has one (the GC) so why bother?

The simple answer is that an ownership system may be used to manage memory, but that isn't the only thing it can do. C#'s memory model views memory as infinite, but there are many other things in C# apps that are not. For instance, file handles on most operating systems are both finite and easily exhaustible. They should also not be accessed after they have been closed, and they should only be closed once. Similarly, the standard library has a popular API called `ArrayPool`. Arrays may be borrowed from the pool using `Rent` and returned using `Return`. Returning the same object twice is invalid, as is using the object after it has been returned. The results can be severe — multiple writers may believe they own the pooled object and effectively cause memory corruption. This has caused serious issues in practice — multiple CVEs have been issued in the framework due to incorrect usage of `ArrayPool`.

Even simple cases that don't seem to look like memory management may be solvable using ownership. There is a popular .NET type called `ImmutableArray`. Because `ImmutableArray` is a fully immutable type, it is often created using a builder type that produces the immutable array using a final `ToImmutable()` method. Internally, even if the array is exactly the right size, a copy is made. This is because there is no way to prove that the builder is discarded after `ToImmutable` is called. If it is not, the builder would still have a reference to the underlying array and could mutate the contents — even though it is supposed to be impossible. Ownership provides the exact semantics desired — if `ToImmutable` takes the Builder ownership, it can ensure that no copies were made and no references are still valid. Therefore, it would be safe to return the underlying array.

## Design

At this point, the goal is clear. C# should have an optional notion of ownership that can be applied to user-defined types. Note that this must be optional precisely because existing C# types do not follow ownership requirements. Unlimited aliasing and copying is the default behavior of C# today.

To indicate that a C# type is an owned resource we will introduce a new interface — `IResource`. The definition of `IResource` is as follows:

```csharp
interface IResource
{
    void Drop();
}
```

Any type that implements `IResource` will be considered an owned resource, which carries some special requirements and privileges. Types will not have to provide an implementation for `Drop()`; the language will automatically provide an implementation that calls `Drop` on all fields that implement `IResource`, in field declaration order. If a type does provide an implementation of `Drop`, that is _in addition to_ the default implementation — after a user implementation of `Drop` is run, the language defined transitive `Drop` implementation is run. This ensures that resources cannot accidentally forget to be dropped.

The following rules apply to all resource types `R`:

- All variables of plain type `R` are considered owning references.
  - The result of a constructor of `R` is an owning reference.
  - At any given point in the program, only one owning reference to a value is permitted. Therefore,
    - Copying resource variables is not permitted. All implicit struct copy behavior is disabled for resource types.
    - Resource variables may not appear as fields of non-resource types.
    - Assigning an owning reference to another owning reference is considered a transfer of ownership.
      - If the variable may be assigned before another assignment, assigning it ends its lifetime
      - After transfer, use of the original reference is disallowed.
  - There may be only one mutable reference, or any number of read-only references (aliasing rules).
  - `R` may not be substituted for generic type parameters
  - `R` may not be boxed into `object`, an interface, `dynamic`, or captured inside a delegate.
  - `R` may only inherit from `object`, `ValueType`, or another resource type

Let's look at a few examples:

- For a resource type `R`:
  - The expression `new R()` produces an owning reference to a value of `R`.
  - The assignment `var v = new R();` produces a transfer of ownership from the new owning reference `R` to the new owner, `v`.
- Both classes and structs may implement `IResource` and be considered resource types. Plain variables are considered "owners" in both cases, regardless of the fact that variables of struct type are not generally considered "references".
- The assignment expression (vs assignment statement) is a somewhat interesting case. It both assigns the right-hand side to the variable on the left-hand side, and produces the value through the evaluation of the expression. For a resource RHS in an ownership assignment, ownership would be transferred to the LHS. However, it would then immediately be transferred to the evaluation result of the expression. This renders the LHS variable unusable and makes the form useless.

```csharp
struct R : IResource
{
    public void M() {}
}

static void Helper(R r)
{
    r.M();
}
```

```csharp
R local = new R();
Helper(local);
local.M();
```

In the above code, the expression `local.M()` produces an error. This is because `Helper(local)` has taken ownership of the value called `R`. After ownership has been transferred, the previous owner of a value may no longer access it.

The same principle is true for the following:

```csharp
R local = new R();
R r2 = local;
local.M();
```

In the above, `local.M();` is an error. `r2 = local` did not make a copy or create a new alias — it took ownership of the value, and `local` lost it.

### Drop rules

- `Drop()` is compiler-invoked and cannot be called directly.
- `Drop` is invoked when leaving the variable's lexical declaring scope
    - This includes exception unwinding
- When an owning variable is re-assigned, `Drop` is called on the overwritten value
- A variable that has been transferred is not dropped
- Owning variables that leave scope together are dropped in reverse declaration order
- The receiver of `Drop` is owned, not borrowed (see [borrow projection](#borrow-projection))
- Drop implementations should not throw

### Borrowing references

Now that we've covered owning references we need to outline borrowed references. Borrowing is very important because it makes helper functions and modularity possible. Consider a generic `ReadByte` function that operates on a `File` type. If `ReadByte` were forced to take ownership of the `File`, `ReadByte` could only be called a single time on a given reference — very inconvenient for files more than one byte long.

Instead, helper functions should use borrowed references. These are references that explicitly do not take ownership of the resource. However, they are guaranteed to live no longer than the resource. Notably, borrowed references do not transfer when ownership transfers. Or, put another way, all borrow lifetimes must end before ownership is transferred.

Lastly, it is important to note that borrowed references are truly references to the owned value, not copies. Operations on borrowed references must occur on the same value as the owned value. This has some different implications for classes and structs. For structs it implies that borrowed references can never be "normal" struct values — borrowed struct references are always true managed by-ref variables.

For classes, the opposite is true. A by-ref to a class variable produces a reference to the local variable. Borrowed references are aliases of the owned value, not the owning reference. This means that all class borrowed references are regular variables with no special ref kind. However, they are different from regular references.

To represent borrowed class references, two new special types will be added to the core library. The definition is as follows:

```csharp
struct Borrow<T>(T value) where T : class
{
    public T Value { get; set; } = value;
}
readonly struct ReadOnlyBorrow<T>(T value) where T : class
{
    public T Value { get; } = value;
}
```

Note that there is a mutable and read-only version of the `Borrow` type. These are analogous to `ref` and `ref readonly` and serve similar purposes. They fall into the same aliasing restrictions above: there may be either one mutable borrowed reference or any number of read-only borrows, but they are mutually exclusive.

Also note that the Borrow types are intrinsic -- they can violate some other rules, like substitution of resource types for generics. It is also illegal to copy a `Borrow<T>` of a resource type, as this would create multiple mutable references. For the same reason, a mutable borrow of a resource may not be stored in a field. A read-only borrow may be stored in a field of a ref struct or a resource type, and the containing type becomes lifetime-bearing, as described in [lifetime-bearing types](#lifetime-bearing-types). Copying such a type copies the read-only borrow and preserves its lifetime. The owner cannot be transferred, reassigned, or dropped while it is borrowed, which [loan checking](#loan-checking) enforces.

The `Value` property will be illegal to access by all code except the compiler. Note that all instance methods of resource types consider their receiver borrowed, so this includes all instance members. In fact, the compiler is responsible for analyzing all operations on `Borrow<T>` as if they were operations on `T` and automatically translating them through calls to `Value`.

> **N.B.** All instance members have a borrowed receiver.

Values returned from instance members get no special treatment. Suppose a member of a resource type creates a `Span<T>` from one of the resource's fields. That span can't outlive `this` because of the type that reading the field produces, not because of where the span is returned. [Borrow projection](#borrow-projection) describes this.

#### Borrow lifetimes

Unlike owned values, borrowed values have a special restriction: they can't live longer than their parent value. The language already has a system of lifetimes for `ref` and `ref struct`. The [ref safety model](ref-safety-model.md) describes it as two layers: a type system in which lifetimes are explicit, and inference that maps C# source onto that type system. Ownership adopts the model and extends it. The extensions are new types, a new kind of location, a new typing rule, new cases in the signature translation, and a flow analysis that runs alongside the type system. The model's rules describe the types and locations that exist in C# today, and none of the extensions changes them. A program that uses no resource types is checked exactly as it is today.

This section uses the notation of the model: `ref<$a> T` for a `ref` whose target lives for at least `$a`, `$heap` for the lifetime of the heap, `$local` for the outermost block of the current method, and `$ro` and `$cm` for the *return-only* and *caller-context* lifetime parameters that every method is given.

##### The `Borrow` types

A `Borrow` must not outlive its target, so its full formal definition carries a lifetime, just like `ref<$a> T`:

```csharp
ref struct Borrow<out $a, T> where T : class { ... }
ref struct ReadOnlyBorrow<out $a, T> where T : class { ... }
```

In the model these are `ref struct` types, and `$a` is the single lifetime parameter that the model gives every `ref struct`. The model's rules therefore apply to them unchanged:

- `$a` is covariant. A `Borrow<$heap, C>` converts to `Borrow<$local, C>`, and the reverse is an error.
- The signature translation types them like any other `ref struct`. A `Borrow<T>` parameter is a `Borrow<$cm, T>` and a returned `Borrow<T>` is a `Borrow<$ro, T>`.

Two conversions connect borrows to ordinary references:

- An ordinary reference of type `T` converts implicitly to `Borrow<$heap, T>` and to `ReadOnlyBorrow<$heap, T>`. The garbage collector keeps its target alive for as long as the borrow exists.
- There is no conversion from a borrow to `T`. It would discard the lifetime.

What can be done with the target of a borrow is limited:

- A member of a class can be invoked through a borrow only if its receiver is borrowed: an instance member of a resource type, or a member marked `[BorrowedReceiver]`, as described in [bridging the worlds](#bridging-the-worlds).
- Through a `Borrow<$a, T[]>`, element access produces a `ref<$a> T`, `Length` can be read, and the borrow converts to `Span<$a, T>` and `ReadOnlySpan<$a, T>`. The elements of an array are `$heap` locations, so producing a `ref<$a> T` is the model's ordinary covariant conversion from `ref<$heap> T`.
- Fields of the target are read by [borrow projection](#borrow-projection).

##### Owner lifetimes

A borrow of an owning variable has the lifetime of the variable's location, as the model's location table defines it. That is the lifetime of the declaring block for a local, and `$local` for a parameter of resource type passed by value. For a resource class, this is the lifetime of the variable, not of the object, which lives on the heap:

```csharp
class C : IResource {}

void M()
{
    var c = new C();
    Borrow<C> b = c;   // Borrow<$local, C>
    M2(b);
}

void M2(Borrow<C> b) { ... }

// Maps to

void M2<$ro, $cm>(Borrow<$cm, C> b) { ... }
```

Even though the data of `C` lives on the heap, the borrow is bound to the lexical lifetime of `c`, which is the body of `M`. The type system therefore guarantees that no borrow of `c` outlives the variable `c`. It does not guarantee that `c` still owns its value. Ownership can be transferred before the block ends, while borrows of `c` are still in scope. [Loan checking](#loan-checking) prevents that.

##### Lifetime-bearing types

A type that contains a borrow must carry its lifetime. The model gives every `ref struct` exactly one lifetime parameter, `out $this`, which applies to every field that can carry a lifetime. Types that contain borrows use the same single parameter:

```csharp
struct E : IResource
{
    public ReadOnlyBorrow<C> Value;
}

// Maps to

struct E<out $this> : IResource
{
    public ReadOnlyBorrow<$this, C> Value;
}
```

A lifetime-bearing resource type is translated like a `ref struct`.

Only `ref struct` types, resource structs and resource classes may contain borrows. The model only allows a location with the lifetime `$a` to hold a value of type `T<$b>` when `$b` is convertible to `$a`. The fields of an ordinary class, the elements of an array and a boxed value are all `$heap` locations, so they could only ever hold borrows whose lifetime is `$heap`. Boxing, capture and unrestricted generic substitution would move a value into such a location without the type system seeing it. A `ref struct` can't be boxed or captured, and it can only be substituted for a type parameter that `allows ref struct`, which the model treats as a `ref struct` itself. Resource structs can't be boxed, captured or substituted for type parameters at all, which is at least as restrictive, so they can carry a lifetime.

Resource classes need one more rule, because their fields are on the heap. The model's location table gives the fields of the classes that exist today the lifetime `$heap`. Ownership adds a row for a new kind of location, the field of a resource class, which has the lifetime of the expression `e` it is accessed through. That is the lifetime of the owning variable, or `$b` when `e` is a `Borrow<$b, R<$a>>`. In either case it is no longer than the class's `$this`:

```csharp
class Parser : IResource
{
    private ReadOnlyBorrow<Tokenizer> _tokens;

    public Parser(ReadOnlyBorrow<Tokenizer> tokens)
    {
        _tokens = tokens;
    }
}

// Maps to

class Parser<out $this> : IResource
{
    private ReadOnlyBorrow<$this, Tokenizer> _tokens;

    static void ctor<$ro>(Borrow<$local, Parser<$ro>> @this, ReadOnlyBorrow<$ro, Tokenizer> tokens)
    {
        // okay: the field is ReadOnlyBorrow<$ro, Tokenizer> and so is the argument
        @this._tokens = tokens;
    }
}
```

This is sound because a resource class object is reachable only through its owner and through borrows of its owner. Both are bounded by `$this`, so the object's storage can outlive `$this`, but no field can be read after `$this` ends. The rules that make this so are the ones for resource types in general:

- A resource can't be converted to `object`, to an interface, or to a base type that isn't a resource. It also can't be boxed, captured or substituted for a type parameter.
- `this` is a borrow in every member, including constructors (see [receivers](#receivers)). No member can produce an ordinary reference to the object.
- A member inherited from a type that isn't a resource, such as `object.MemberwiseClone`, can't be invoked through a borrow, because its receiver isn't borrowed.

The runtime adds one limit. Byrefs can't be stored on the heap, so a resource class can't contain `ref` fields or fields of `ref struct` type, which include `Span<T>`. It can contain `Borrow` and `ReadOnlyBorrow` fields, which are object references at runtime, and lifetime-bearing resource structs that contain them.

This proposal doesn't add a corresponding row for the elements of arrays of resources. Those elements are `$heap` locations, so arrays of resources can't hold borrows.

##### Receivers

The model's signature translation is a choice of defaults, and safety doesn't depend on it. Ownership adds the following cases:

| Member | Type of `this` |
| --- | --- |
| Instance member of a resource struct `R` | `ref<$ro> R` |
| `readonly` instance member of a resource struct `R` | `ref readonly<$ro> R` |
| Instance member of a resource class `R` | `Borrow<$ro, R>` |
| `readonly` instance member of a resource class `R` | `ReadOnlyBorrow<$ro, R>` |
| `[BorrowedReceiver]` member of a class `C` | `Borrow<$ro, C>`, or `ReadOnlyBorrow<$ro, C>` if `readonly` |
| Constructor of a resource class `R` | `Borrow<$local, R>` |
| `Drop` of a resource type `R` | `R`, owned |

When `R` is lifetime-bearing, `R` in this table stands for `R<$cm>`, except in a constructor, where it stands for `R<$ro>`. This matches the model's translation of `ref struct` receivers, and of constructors as `out` parameters.

In a struct that is not a resource, `this` is implicitly `scoped`: `ref<$local> S`. The receiver of a resource struct is `ref<$ro> R` instead, as though it were marked `[UnscopedRef]`. This lets a member return values derived from `this`, and at the call site those values are bounded by the receiver. Constructors are unchanged.

Parameters of resource type passed by value are owned. `ref`, `in` and `out` parameters of resource type are translated as the model translates them.

##### Borrow projection

A borrow doesn't protect anything if the target's fields can be copied out of it. A resource that holds an ordinary array in a field, like `RentedArray` in the [worked examples](#worked-examples), would otherwise hand out `Span<$heap, T>` values that outlive the rental. So reading a field through a borrow produces a borrow.

Let `e` be an expression of type `ref<$a> R`, `ref readonly<$a> R`, `Borrow<$a, R>` or `ReadOnlyBorrow<$a, R>`, or a borrow produced by projection, and let `F` be a field of its target. The expression `e.F` is typed as follows:

- If `F` is of a resource struct type, `e.F` is a location with the lifetime `$a`, accessed by reference: `ref<$a> F`, or `ref readonly<$a> F` if `e` is read-only. When `e` is a `ref`, this is what the model's location table already says for a field of a struct.
- If `F` is of a resource class type `U`, `e.F` is a `Borrow<$a, U>`, or a `ReadOnlyBorrow<$a, U>` if `e` is read-only.
- If `F` is of any other reference type `U`, `e.F` is a `Borrow<$a, U>`. `U` is not a resource, so the aliasing rules in [mutability](#mutability) don't apply to it. The borrow may be copied, and what can be done with its target is governed by `U`, as it is for a `readonly` field of type `U` today.
- If `F` is of a struct type that contains references, its value can't be copied out of `e`. It is accessed by reference, and the references inside it are read by projection.
- Otherwise, `F` contains no references and is read as an ordinary value.

Element reads through a `Borrow<$a, U[]>` follow the same rules, so the elements of a jagged array are borrows too.

Projection depends only on the type of `e`, not on the kind of member the expression appears in. It applies in static members and nested types as well as instance members. It also never changes the lifetime of a location. An array element is still a `$heap` location, and `ref<$heap> T` converts to `ref<$a> T` by the model's ordinary covariance.

```csharp
public readonly struct RentedArray : IResource
{
    private readonly T[] _rented;

    public Span<T> Span => _rented;
}

// Maps to

static Span<$ro, T> get_Span<$ro>(ref readonly<$ro> RentedArray @this)
{
    // okay: @this._rented is a Borrow<$ro, T[]>, which converts to Span<$ro, T>
    return @this._rented;
}
```

Every way of leaking the array is an ordinary type error:

```csharp
// error: Borrow<$ro, T[]> does not convert to T[]
public T[] Raw => _rented;

// error: Borrow<$ro, T[]> does not convert to Memory<T>
public Memory<T> Memory => _rented;

// error: Borrow<$ro, T[]> does not convert to T[]
public void Cache() => s_cache = _rented;

// okay: out Span<T> maps to ref<$local> Span<$ro, T>, so the caller's span is bounded
// by the receiver
public void Get(out Span<T> span) => span = _rented;

// error: r is a ref<$local> RentedArray, so r._rented is a Borrow<$local, T[]>, which
// does not convert to the return type Span<$ro, T>
public static Span<T> Leak(scoped ref RentedArray r) => r._rented;
```

Values that are owned, rather than borrowed, are read differently. When `e` is owned (the receiver of `Drop`, an owning variable, or a parameter passed by value), reading a field that is not of a resource type produces its ordinary value. Fields of resource type are still accessed by reference or as borrows, with the lifetime of `e`. They can't be moved out, because the compiler-provided part of `Drop` drops them after the user-defined part runs.

If the resource type has a user-defined `Drop`, a field can only be read as an ordinary value inside `Drop` itself. Anywhere else, the user-defined `Drop` would run when the owner leaves scope, while the value read from the field might still be in use.

##### Loan checking

The type system guarantees that no borrow outlives the location of its owner. It doesn't guarantee that the owner still owns its value. An owning variable can be transferred or reassigned before its block ends, and a member called through a mutable borrow can reassign a field of resource type, which drops the field's old value. Loan checking prevents both. Like definite assignment, it is a flow analysis, and it is not part of the type system.

```csharp
var rented = pool.Rent(n);
Span<byte> span = rented.Span;   // Span<$local>: rented is loaned
Consume(rented);                 // error: rented may be loaned
span[0] = 1;
```

The type of `span`, `Span<$local>`, is valid until the end of the block, so the type system accepts `span[0] = 1`. Loan checking rejects the program at the transfer instead. As a result, no point in the program has a live borrow of `rented` after `rented` has stopped owning the array. If the transfer comes first, the borrow is rejected instead, because using a variable that may have been transferred is an error.

A *place* is one of the following:

- An owning variable.
- `this`, in a member with a borrowed receiver.
- A parameter passed by `ref` or `in`, or as a borrow.
- A field of resource type reached from one of these.

An expression *loans* a place when it borrows the place, or something reached through it, and stores a value whose lifetime is bounded by that borrow. The value might be stored in a local, in a field, in the target of an `out` or `ref` argument, or in the method's return value.

- Borrowing a place means taking a `ref` to it, converting it to a borrow, reading through it by projection, or passing it as a borrowed receiver or argument.
- Body inference determines whether the borrow bounds a stored value. The borrow constrains the `$ro` or `$cm` that inference chooses for a call, and the call's outputs either use that lifetime or don't.
- A borrow that only flows into calls that store nothing ends with the call. `ToUtf8(s, rented.Span)` does not loan `rented`, and neither does `builder[i] = i`.
- The loaned place is the last place of resource type on the path the borrow was taken through. `rented.Span` loans `rented`, `this._inner.Span` loans `this._inner`, and inside `RentedArray`, `_rented` loans `this`.

A loan is *read-only* when the borrow is read-only, such as `ref readonly`, `in`, `ReadOnlyBorrow`, or a call to a `readonly` member. Otherwise it is *mutable*.

Loans are recorded at the expression that creates them. They are not found later from the types of variables. Covariance can convert a lifetime to a shorter one, so the type of a value doesn't always show which place it was borrowed from.

A place is *possibly loaned* at a point in the method if some path through the method reaches that point after an expression that loans the place. Where paths merge, including along loop back edges, the analysis takes the union:

```csharp
Span<byte> span = default;
if (condition)
{
    span = rented.Span;
}
Consume(rented);                 // error: rented may be loaned
```

A loan lasts until the end of the scope of its place. For `this` and parameters, that is the end of the method. Two places *overlap* when one is the other or is reached through it. At a point where a place `p` is possibly loaned, it is an error to do any of the following to a place that overlaps `p`:

- Transfer it.
- Assign it, which drops its previous value.
- Borrow it mutably, if the loan of `p` is read-only. A mutable borrow could reassign a field of resource type and drop the target of the loan.
- Use it at all, if the loan of `p` is mutable.

No check is needed when a place's scope ends. Every borrow of the place has a lifetime no longer than its block, so every borrow is out of scope too. Owning variables are dropped in reverse declaration order, because a resource declared after `v` may hold a borrow of `v` and use it in its `Drop`.

Loan checking is lexical. A loan lasts until the end of its place's scope even when the values that hold the borrow are no longer used:

```csharp
var rented = pool.Rent(n);
Span<byte> span = rented.Span;
ToUtf8(s, span);
Consume(rented);                 // error: rented is loaned, although span is no longer used
```

##### Relationship to the ref safety model

Ownership adds to the [ref safety model](ref-safety-model.md) without changing it. The model's rules describe the types and locations that exist in C# today, and everything ownership introduces is new:

- `Borrow<$a, T>` and `ReadOnlyBorrow<$a, T>` are `ref struct` types with a single covariant lifetime parameter, which the model already describes.
- The field of a resource class is a new kind of location. It gets a new row in the location table, as described in [lifetime-bearing types](#lifetime-bearing-types). Unlike the existing rows, its lifetime comes from reachability rather than from how long its storage exists: the object is reachable only through its owner and through borrows of it, so no field can be read after `$this` ends. The rules for resource types guarantee this, and the model already reasons this way about the target of a `ref<$a> T`, which it treats as a location with the lifetime `$a` whatever its actual lifetime.
- Borrow projection is a new typing rule. Apart from the fields of resource classes, the lifetimes it produces come from the model's existing covariant conversions.
- The receivers of resource members, of `[BorrowedReceiver]` members and of `Drop` are new cases in the signature translation, whose defaults the model leaves open.
- Owner lifetimes are the lifetimes of locations, as the location table defines them, so the lifetimes in scope remain totally ordered.
- Loan checking is a separate flow analysis.

The rule that a value returned from an instance member of a resource type can't outlive `this` doesn't need to be stated. It follows from the `ref<$ro>` receiver and borrow projection. The same reasoning covers `out` parameters, static members, and every other way a value can leave a member.

### Mutability

At any point, a resource value permits either:

- one mutable borrow, or
- any number of read-only borrows.

These states are mutually exclusive. [Loan checking](#loan-checking) enforces them, and it also prevents the owner from being transferred or reassigned while it is borrowed. These rules apply to resources. A borrow of a value that is not a resource, such as one produced by [borrow projection](#borrow-projection), only carries a lifetime.

To accommodate read-only and mutable borrows we also have to adjust the rules for classes themselves. The `readonly` keyword will now be legal for class methods, just like struct methods. It will have the same rules. An ordinary (non-readonly) class instance method will have a `Borrow<this>` receiver. A `readonly` instance member has a `ReadOnlyBorrow<this>` receiver. Importantly, read-only borrows may only call readonly members. Note that this is shallow mutability -- non-resource members may effectively be mutated due to lack of mutability requirements on interior members.

### Bridging the worlds

If owned resources and GC types do not cooperate they will force parallel worlds to be created and code to be duplicated. Fortunately, the restrictions are smaller than they appear. The biggest problem with allowing regular GC references to appear in `Borrow`, like owned resources, is that the methods on the GC type do not internally follow ownership rules. Fundamentally, this is about the receiver. In classes, the receiver type is the parent type -- meaning a normal GC tracked reference. For normal GC references, the lifetime of `this` is not tracked and may escape to any other part of the program.

The solution is to allow individual methods of classes to opt-in to having _borrowed receivers_. This feature requires a new attribute:

```C#
public sealed class BorrowedReceiver : System.Attribute {}
```

Arbitrary class types may apply it to any number of instance methods:

```C#
class C
{
    [BorrowedReceiver]
    public void M1() { ... }

    public void M2() { ... }
}
```

The effect is to change the type of the receiver from the containing type `C`, to the borrow type `Borrow<C>`.  The full formal lifetime would look like,

```C#
class C
{
    [BorrowedReceiver]
    public static void M1<$ro>(Borrow<$ro, C> @this) { ... }
}
```

Like all borrows, it would be illegal to convert a borrowed reference to a GC reference. Thus, `M1` is restricted from escaping the receiver outside the current method. Inside a `[BorrowedReceiver]` member, the fields of `this` are read by [borrow projection](#borrow-projection), so the member can't leak references to the object's contents either. This cleanly allows arbitrary class types to appear as borrowed types. `BorrowedReceiver` is also legal on constructors -- because constructors can access `this` they have the same risk of exposing it outside the constructor method, unless it is typed as a borrow.

However, converting just GC types to borrows is not particularly useful on its own. We would like to be able to treat a class as either GC-tracked or owned, _per instantiation_. To do so we will need another instrinsic type:

```C#
public readonly struct Owned<T> : IResource
    where T : class
{
    public T Value { get; }
}
```

This is another primitive type with special permissions and restrictions.

* It has no public constructor. The `Owned` type effectively re-exports the constructors of `T`. Each constructor must be marked `BorrowedReceiver` to indicate that it is borrow safe.

* Existing instances of `T` may never be converted to `Owned` -- `Owned` must completely control the lifetime of the instance.
* Direct access to `Value` is prohibited. Instead, all `BorrowedReceiver` methods on `T` will re-exported as theough on the `Owned<T>` instance and the compiler is responsible for translating all such invocations through `Value`.

`Owned` therefore allows individual instantiations to control whether they are GC-tracked or borrowed. `string` would remain GC-tracked by default, but an `Owned<string>` could also be created that would follow ownership rules. This may provide performance advantages in certain scenarios. This also does not lift all restrictions related to ownership: optionally-GC-tracked types may never contain fields of a resource type.

Notably, there is one very useful type that `Owned` does not help: arrays. Resource types may not safely appear as element types in normal arrays for multiple reasons:

* Arrays do not carry aliasing restrictions, meaning that multiple aliases to the parent array could appear, violating unique ownership
* Arrays would not call Drop on the elements

All of these problems are solved using a new language construct: _conditional implementation_. In existing C#, all types either implement an interface or not. This proposal would add a new special case, just for arrays: for a substituted type `T[]`, `T[]` implements the `IResource` interface if and only if `T` implements `IResource`. Correspondly, `T[]` is a resource if and only if `T` is a resource. The implementation of `Drop` is also specified as walking the array from beginning to end and calling `Drop` on each element.

This means that arrays of resource types can be created and, if they are, they are resources themselves. `string[]` is not a resource because `string` is not a resource. `CustomResource[]`, however, (where `CustomResource` is defined as `class CustomResource : IResource`) would be considered a resource and would be required to follow all resource rules.

### Worked examples

We now have the fundamental building blocks of our ownership system. To that end, let's revisit all our examples and demonstrate how they are solved with an ownership system. The three examples were:

- Native finite resources
- `ArrayPool`
- `ImmutableArray` Builder

Let's examine each in order.

For finite native resources, `File` is a good example. We can reliably close the file inside Drop, preventing leaking file handles or duplicate close. Note that for a real File, we may have buffered I/O that needs to be written -- because Drop is not `async` we cannot guarantee that all data is written, just that the file is closed.

```csharp
class File : IResource
{
    private IntPtr _nativeHandle;

    void IResource.Drop()
    {
        NativeClose(_nativeHandle);
    }
}
```

This may seem simple, but it is effective. This is enough to verify that all file objects are dropped automatically and at most once.

Next we have `ArrayPool`. This is more complicated because the restriction is stricter, and we are more likely to need to interact with code we don't own. One possible new API shape is as follows.

```csharp
class OwnedArrayPool<T> // unowned
{
    private readonly ArrayPool<T> _pool;

    public static OwnedArrayPool<T> Shared { get; } =
        new OwnedArrayPool<T>(ArrayPool<T>.Shared);

    private OwnedArrayPool(ArrayPool<T> pool)
    {
        _pool = pool;
    }

    public RentedArray Rent(int minimumLength)
    {
        return new RentedArray(_pool.Rent(minimumLength), this);
    }

    private void Return(T[] array)
    {
        _pool.Return(array);
    }

    public readonly struct RentedArray : IResource
    {
        private readonly T[] _rented;
        private readonly OwnedArrayPool<T> _pool;

        private RentedArray(T[] rented, OwnedArrayPool<T> pool)
        {
            _rented = rented;
            _pool = pool;
        }

        public Span<T> Span => _rented;

        void IResource.Drop()
        {
            _pool.Return(_rented);
        }
    }
}
```

The wrapper returns a new `RentedArray` resource type instead of exposing the rented array. `RentedArray` provides access through a `Span`, which borrows the rented array. This is sufficient for many scenarios that currently use the array directly.

Creating a span from an ordinary array would normally give the span a heap lifetime. Here, `Span` is a member of a resource type, so its receiver is `ref readonly<$ro> RentedArray`. Reading `_rented` through that receiver produces a `Borrow<$ro, T[]>` by [borrow projection](#borrow-projection). The span converted from it is a `Span<$ro, T>`, which can't outlive the `RentedArray` it was obtained from. The same rule prevents the array from leaking any other way. There is no conversion from the borrow to `T[]`, so the array can't be returned, stored in a field, or converted to `Memory<T>`.

`Drop` has an owned receiver, so it reads `_rented` and `_pool` as ordinary values and can return the array to the pool. Outside of `Drop`, `_pool` would also be read as a borrow.

The guarantee covers every reference obtained through a `RentedArray`. It doesn't cover references kept by the code that created the `RentedArray`. The constructor receives an ordinary array, so it is up to the author of the type to make sure no other alias exists. Here the constructor is private, and it is only called with an array fresh from the pool.

The usage would look like:

```csharp
void Write(string s)
{
    if (s.Length < MAXSTACK)
    {
        Span<byte> utf8 = stackalloc byte[MAXSTACK * 2];
        ToUtf8(s, utf8);
    }
    else
    {
        var rented = OwnedArrayPool<byte>.Shared.Rent(s.Length * 2);
        ToUtf8(s, rented.Span);
    }
}
```

This is the classic "stackalloc a span if small, or use a rented array if large" pattern. In both cases the usage is local and simple. Unlike direct use of `ArrayPool`, there is no risk of forgetting to return the array or returning it twice. `Return` is not exposed to the renter. Instead, the `RentedArray` is automatically dropped at the end of its scope, and its `Drop` implementation returns the array to the pool. Access is limited to a `Span` whose lifetime is tied to the `RentedArray`, so the span cannot outlive the rental. `ToUtf8` doesn't store the span, so the borrow of `rented` ends with the call. If the span had been stored in a local, `rented` would be loaned for the rest of its scope, and [loan checking](#loan-checking) would reject any attempt to transfer it.

Last is the case of `ImmutableArray.Builder`. This looks extremely similar to the `ArrayPool` case.

```csharp
struct OwnedImmutableBuilder<T> : IResource
{
    private T[] _backing;

    public OwnedImmutableBuilder(int len)
    {
        _backing = new T[len];
    }

    public T this[int index] { ... }

    public static ImmutableArray<T> ToImmutable(OwnedImmutableBuilder builder)
    {
        return ImmutableArray.UnsafeCreate( // Assume an internal unsafe method
            builder._backing                //   to make an ImmutableArray
        );
    }
}
```

We use it very simply:

```csharp
var builder = new OwnedImmutableBuilder<int>(10);
for (int i = 0; i < 10; i++)
{
    builder[i] = i;
}
var array = OwnedImmutableBuilder.ToImmutable(builder);
```

This looks very similar to how we use the rented array, with one notable exception: the static method `ToImmutable`. The main difference here is between instance and static methods on resource types. Static methods, or any method except the receiver of an instance method, will consider parameters of a resource type to be <u>owning references</u>, not borrowing references. Functionally, this is what allows us to guarantee that the array is transferred safely — it is never exposed by the builder, the builder is a resource so it is never copied, and the `ToImmutable` function takes ownership of the builder, meaning no references could live past the transfer.

The indexer's receiver is `ref<$ro> OwnedImmutableBuilder<T>`, so it reads `_backing` as a `Borrow<$ro, T[]>`, through which it can read and write elements. `ToImmutable` owns `builder`, so it reads `_backing` as an ordinary array. This is allowed because `OwnedImmutableBuilder<T>` has no user-defined `Drop`. If it had one, for example a `Drop` that cleared the array, reading the field as an ordinary value would be an error. That `Drop` would run when `builder` leaves scope at the end of `ToImmutable`, after the array had been handed to the `ImmutableArray`.

## Open questions

- **Shared fields.** Projection applies to every reference field of a resource, including fields that refer to shared objects rather than owned ones, such as `_pool` in `RentedArray` or a field of type `string`. A member can only be called through a borrow if it has a borrowed receiver, so such fields are hard to use outside of `Drop`. A per-field opt-out could exempt them. Projection should remain the default, because a field whose author forgot to opt in is exactly how an owned array leaks.
- **Existing APIs.** A borrow can't be passed to an existing method that takes `T[]` or another class type. Extending `[BorrowedReceiver]` to parameters would let such APIs accept borrows: the parameter would be typed as `Borrow<$ro, T>`, or as `Borrow<$local, T>` when it is `scoped`. An ordinary reference converts to `Borrow<$heap, T>`, so annotating a parameter this way wouldn't break existing callers. The implicit conversion from `T[]` to `Span<T>` is a candidate.
- **Struct fields that contain references.** Projection makes these accessible only by reference, so a field of type `ImmutableArray<T>` can't be copied out of a resource. Calling its members would require borrowed receivers on structs.
- **Generic substitution.** Resource types can't be substituted for type parameters. A `ref struct` can be substituted for a type parameter that `allows ref struct`, and a similar anti-constraint could allow resources. Generic code would then have to treat such a type parameter as a resource: values of it are transferred rather than copied, are dropped at the end of their scope, and have borrowed receivers. If the type argument can be lifetime-bearing, the type parameter carries a lifetime, as the [ref safety model](ref-safety-model.md) already describes for `allows ref struct`.
- **Depth of `readonly`.** Projection keeps the shallow `readonly` of C#, so a `readonly` member can return a `Span<T>` over an array field. If read-only borrows should also prevent writes to contents that are not resources, projection through a read-only borrow would have to produce read-only results. `RentedArray` could then only expose a `ReadOnlySpan<T>`.
- **Loan duration.** Loans last until the end of their place's scope. Ending a loan when the variables that hold its borrows go out of scope would accept more programs, but loan checking would then have to track which variables hold borrowed values.
- **Arrays of resources holding borrows.** By the same argument as resource classes, the elements of an array of resources could be given the lifetime of the array instead of `$heap`. This would be another new row in the location table.
- **Non-lexical lifetimes.** Ending an owner's lifetime where it is transferred would accept more programs. It is a larger extension than the others. Body inference in the [ref safety model](ref-safety-model.md) relies on the lifetimes in scope being totally ordered, and a lifetime that ends partway through a block breaks that. Inference would have to choose the longest lifetime no longer than each constraint, rather than simply the shortest constraint. It would also need a rule that a value is only used within the lifetimes in its type. It would not break existing code. Only owners would have such lifetimes, so in a program without resource types the lifetimes remain totally ordered, and the generalized inference chooses the same lifetimes as today.

## Acknowledgements

1. Walker, David (2002). "Substructural Type Systems"
2. The Rust programming language for general understanding of ergonomics
3. Special thanks to [boats](https://without.boats/)), who's writing has heavily influenced my framing of how to think of a GC hybrid language
