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
  - There may be only one exclusive borrow, or any number of shared borrows (aliasing rules).
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

Note that a variable may be transferred on some paths but not others. For example,

```csharp
R local = new R();
if (b)
{
    Helper(local);   // transfers local
}
local.M();           // error: local may have been transferred
```

Using a variable after it may have been transferred is an error, so the only question is whether to drop it. In this case `local` should be dropped at the end of its scope only if `b` was false. When this can't be determined statically, the compiler tracks it with a hidden _drop flag_ -- a local that is cleared when ownership is transferred, and checked before calling `Drop`. The same flag decides whether reassigning the variable drops its previous value. Drop flags are only needed for variables that are conditionally transferred, so most variables don't need one.

### Borrowing references

Now that we've covered owning references we need to outline borrowed references. Borrowing is very important because it makes helper functions and modularity possible. Consider a generic `ReadByte` function that operates on a `File` type. If `ReadByte` were forced to take ownership of the `File`, `ReadByte` could only be called a single time on a given reference — very inconvenient for files more than one byte long.

Instead, helper functions should use borrowed references. These are references that explicitly do not take ownership of the resource. However, they are guaranteed to live no longer than the resource. Notably, borrowed references do not transfer when ownership transfers. Or, put another way, all borrow lifetimes must end before ownership is transferred.

Lastly, it is important to note that borrowed references are truly references to the owned value, not copies. Operations on borrowed references must occur on the same value as the owned value. This has some different implications for classes and structs. For structs it implies that borrowed references can never be "normal" struct values — borrowed struct references are always true managed by-ref variables.

For classes, the opposite is true. A by-ref to a class variable produces a reference to the local variable. Borrowed references are aliases of the owned value, not the owning reference. This means that all class borrowed references are regular variables with no special ref kind. However, they are different from regular references.

To represent borrowed class references, two new special types will be added to the core library. The definition is as follows:

```csharp
readonly struct Borrow<T>(T value) where T : class
{
    public T Value { get; } = value;
}
struct ExclusiveBorrow<T>(T value) where T : class
{
    public T Value { get; set; } = value;
}
```

Note that there is a shared and an exclusive version of the `Borrow` type. Any number of shared borrows of a value may exist at once, while an exclusive borrow guarantees that nothing else can access the value as long as it exists. These are analogous to `ref readonly` and `ref`, and fall into the same aliasing restrictions above: there may be either one exclusive borrow or any number of shared borrows, but not both. Shared borrows are the common case, so they get the shorter name.

Also note that the Borrow types are intrinsic -- they can violate some other rules, like substitution of resource types for generics. It is also illegal to copy an `ExclusiveBorrow<T>`, as this would create multiple exclusive borrows. For the same reason, an exclusive borrow may not be stored in a field. A shared borrow may be stored in a field of a ref struct or a resource type, and the containing type becomes lifetime-bearing, as described in [lifetime-bearing types](#lifetime-bearing-types). Copying such a type copies the shared borrow and preserves its lifetime. The owner cannot be transferred, reassigned, or dropped while it is borrowed, which [loan checking](#loan-checking) enforces.

The `Value` property will be illegal to access by all code except the compiler. Note that all instance methods of resource types consider their receiver borrowed, so this includes all instance members. In fact, the compiler is responsible for analyzing all operations on `Borrow<T>` as if they were operations on `T` and automatically translating them through calls to `Value`.

> **N.B.** All instance members have a borrowed receiver.

Note that values returned from instance members don't get any special treatment. If a member of a resource type creates a `Span<T>` from one of the resource's fields, the span can't outlive `this` -- but that's because of the type produced by reading the field, not because of where the span is returned. We'll see how this works in [borrow projection](#borrow-projection).

#### Borrow lifetimes

Unlike owned values, borrowed values have a special restriction: they can't live longer than their parent value. This rule is enforced by the language through static analysis. Luckily, the language already has a concept of lifetimes and a lifetime analysis system for by-ref types. The [ref safety model](ref-safety-model.md) describes that system as a type system in which lifetimes are explicit, plus inference that maps C# source onto it. We can adopt this model without modification. The model describes the types and locations that exist in C# today, and everything we add here is new, so none of its rules need to change. A program that doesn't use resource types is checked exactly as it is today.

We'll use the notation of the model throughout. `ref<$a> T` is a by-ref whose target lives for at least `$a`. `$heap` is the lifetime of the heap and `$local` is the lifetime of the outermost block of the current method. `$ro` and `$cm` are the *return-only* and *caller-context* lifetime parameters that every method is given.

For by-ref variables, we do not need to make any further modifications. The most important change is that we now have a new type of ref — the `Borrow<T>` type. Remember the ownership constraint — a `Borrow<T>` instance may not live longer than its target. To do so, `Borrow` must be instantiated with the lifetime of its target. Therefore, the full formal definition of `Borrow` must include a lifetime parameter, just like by-ref. Therefore the full formal definition is

```csharp
ref struct Borrow<out $a, T> where T : class { ... }
ref struct ExclusiveBorrow<out $a, T> where T : class { ... }
```

Note that these are formally ref structs. The model gives every ref struct exactly one lifetime parameter, and `$a` is that parameter, so all the model's rules apply to borrows as-is. In particular, `$a` is covariant -- a `Borrow<$heap, C>` may be converted to a `Borrow<$local, C>`, but not the reverse. Similarly, a `Borrow<T>` parameter is a `Borrow<$cm, T>` and a returned `Borrow<T>` is a `Borrow<$ro, T>`, just like any other ref struct.

There are also conversions between borrows and regular references. A regular reference of type `T` may be implicitly converted to `Borrow<$heap, T>`, since the GC keeps the target alive for as long as the borrow exists. However, it may not be converted to `ExclusiveBorrow<$heap, T>`. Other references to the same object may exist, so the borrow wouldn't really be exclusive -- and code receiving an `ExclusiveBorrow<T>` would have no way to tell it apart from one that is. Lastly, there is no conversion from a borrow back to `T`, as that would discard the lifetime.

Between the two kinds of borrow, an exclusive borrow may be used wherever a shared borrow is expected, but not the reverse.

Lastly, what can be done through a borrow is limited. Members of a class may only be called through a borrow if their receiver is borrowed -- either because they are instance members of a resource type, or because they are marked `[BorrowedReceiver]` (see [bridging the worlds](#bridging-the-worlds)). Arrays get a few intrinsic operations: a `Borrow<$a, T[]>` can be indexed, producing a `ref<$a> T`, its `Length` can be read, and it can be converted to `Span<$a, T>` or `ReadOnlySpan<$a, T>`. Array elements are `$heap` locations, so producing a `ref<$a> T` is just the model's ordinary covariant conversion from `ref<$heap> T`. Fields are read by [borrow projection](#borrow-projection), described below.

The other half is, of course, lifetime inference. By-ref variables currently acquire their lifetime from the target variable's language-defined lifetime. Owned variables are not quite so simple. In particular, an owned reference may have local scope, but heap storage. For example,

```csharp
class C : IResource {}

void M()
{
    var c = new C();
    ...
}
```

In this example, the lifetime of `c` is defined based on the lexical lifetime of variable `c`. Therefore if we were to introduce a borrow,

```csharp
void M()
{
    var c = new C();
    Borrow<C> b = c;   // Borrow<$local, C>
    M2(b);
    ...
}

void M2<$ro, $cm>(Borrow<$cm, C> b)
{
    ...
}
```

We can see that the lifetime parameterization of `Borrow<C>` is based on the declaration of the `c` variable. More precisely, a borrow of an owning variable has the lifetime of the variable's location, as the model defines it -- the lifetime of the declaring block for a local, or `$local` for a parameter. So even though the data of `C` lives on the heap, the borrow is bound to the lexical lifetime of `c`, namely the body of the method `M`.

Note that this only guarantees that no borrow outlives the variable `c`. It doesn't guarantee that `c` still owns its value -- ownership may be transferred away before the end of the block, while borrows of `c` are still in scope. Preventing that is the job of [loan checking](#loan-checking).

#### Lifetime-bearing types

This does have some further implications. In particular, types that contain `Borrow` instances. Just like ref structs, they must allow for lifetime parameterization. For example, the ref struct `E` can be defined to hold a by-ref:

```csharp
ref struct E
{
    public ref int Value;
}
```

The full formal definition is the type:

```csharp
ref struct E<out $this>
{
    public ref<$this> int Value;
}
```

The model gives every ref struct exactly one lifetime parameter, `$this`, which is used for every field that can carry a lifetime. Types that contain borrows work the same way -- there is only ever one lifetime parameter, no matter how many borrow fields there are. For example, we might define

```csharp
struct E : IResource
{
    public Borrow<C> Value;
}
```

which has the full formal definition

```csharp
struct E<out $this> : IResource
{
    public Borrow<$this, C> Value;
}
```

analogous to our by-ref struct `E`. Lifetime-bearing resource types are otherwise treated just like ref structs.

However, not every type can be lifetime-bearing. The model only allows a location with the lifetime `$a` to hold a value of type `T<$b>` when `$b` is convertible to `$a`. The fields of an ordinary class, the elements of an array, and boxed values are all `$heap` locations, so they could only ever hold borrows of `$heap`. Boxing, capture, and unrestricted generic substitution would let a value move into such a location without the type system noticing. Ref structs avoid this because they can't be boxed or captured, and can only be substituted for type parameters that `allows ref struct`, which the model treats as ref structs themselves. Resource structs are even more restrictive -- they can't be boxed, captured, or substituted for type parameters at all -- so they can safely carry a lifetime. Therefore, ref structs, resource structs, and resource classes may contain borrows, but other types may not.

Resource classes need one more rule, since their fields live on the heap. The model's location table gives the fields of today's classes the lifetime `$heap`. We will add a new kind of location: the field of a resource class, which has the lifetime of the expression it is accessed through. That is the lifetime of the owning variable, or `$b` when it is accessed through a `Borrow<$b, R<$a>>`. Either way, it is no longer than the class's `$this`. For example,

```csharp
class Parser : IResource
{
    private Borrow<Tokenizer> _tokens;

    public Parser(Borrow<Tokenizer> tokens)
    {
        _tokens = tokens;
    }
}
```

has the full formal definition

```csharp
class Parser<out $this> : IResource
{
    private Borrow<$this, Tokenizer> _tokens;

    static void ctor<$ro>(ExclusiveBorrow<$local, Parser<$ro>> @this, Borrow<$ro, Tokenizer> tokens)
    {
        // okay: the field is Borrow<$ro, Tokenizer> and so is the argument
        @this._tokens = tokens;
    }
}
```

Unlike the existing locations, the lifetime of this location doesn't come from how long its storage exists -- the object may live on in the heap long after `$this` ends. Instead, it comes from reachability. A resource class object is only reachable through its owner and through borrows of its owner, both of which are bounded by `$this`, so no field can be read after `$this` ends. The rules for resource types guarantee this. A resource can't be converted to `object`, an interface, or a non-resource base type, and can't be boxed, captured, or substituted for a type parameter. `this` is a borrow in every member, including constructors, so no member can produce a regular reference to the object. And members inherited from non-resource types, like `object.MemberwiseClone`, can't be called through a borrow, since their receivers aren't borrowed. This isn't a new kind of reasoning for the model, either -- the target of a `ref<$a> T` is treated as a location with the lifetime `$a`, whatever its actual lifetime.

There is one additional limitation from the runtime. By-refs can't be stored on the heap, so resource classes can't contain `ref` fields or ref struct fields, including `Span<T>`. They can contain `Borrow` fields, which are just object references at runtime, and lifetime-bearing resource structs that contain them.

We will not add a corresponding rule for the elements of arrays of resources, so they remain `$heap` locations and can't hold borrows.

#### Borrowed receivers

We've already said that all instance members of resource types have a borrowed receiver. Now we can make that precise. The model translates instance methods into static methods with an explicit `this` parameter, and notes that this translation is just a choice of defaults -- safety doesn't depend on it. So we are free to choose new defaults for resource types:

| Member | Type of `this` |
| --- | --- |
| Instance member of a resource struct `R` | `ref<$ro> R` |
| `readonly` instance member of a resource struct `R` | `ref readonly<$ro> R` |
| Instance member of a resource class `R` | `ExclusiveBorrow<$ro, R>` |
| `readonly` instance member of a resource class `R` | `Borrow<$ro, R>` |
| Constructor of a resource class `R` | `ExclusiveBorrow<$local, R>` |
| `[BorrowedReceiver]` member of a non-resource class `C` | `Borrow<$ro, C>` |
| `[BorrowedReceiver(Exclusive = true)]` member of a non-resource class `C` | `ExclusiveBorrow<$ro, C>` |
| `[BorrowedReceiver]` constructor of a non-resource class `C` | `ExclusiveBorrow<$local, C>` |
| `Drop` of a resource type `R` | `R`, owned |

For example,

```csharp
struct R : IResource
{
    void M() { }
    readonly void RO() { }
}

class C : IResource
{
    C() { }
    void M() { }
    readonly void RO() { }
}

// Maps to

struct R : IResource
{
    static void M<$ro>(ref<$ro> R @this) { }
    static void RO<$ro>(ref readonly<$ro> R @this) { }
}

class C : IResource
{
    static void ctor(ExclusiveBorrow<$local, C> @this) { }
    static void M<$ro>(ExclusiveBorrow<$ro, C> @this) { }
    static void RO<$ro>(Borrow<$ro, C> @this) { }
}
```

Note that `Drop` is the exception: its receiver is owned, not borrowed, since it is called at the end of the value's lifetime. If the resource type is lifetime-bearing, `R` stands for `R<$cm>` in these signatures, or `R<$ro>` in constructors, matching the model's translation of ref struct receivers and constructors.

Note the receiver of a resource struct. In an ordinary struct, `this` is implicitly `scoped`: `ref<$local> S`. Resource structs instead get `ref<$ro> R`, as though the member were marked `[UnscopedRef]`. This is what lets a member return values derived from `this` -- at the call site, those values are bounded by the receiver. Constructors are unchanged.

Other parameters work just as you'd expect. Parameters of a resource type passed by value are owning references, while `ref`, `in`, and `out` parameters are translated exactly as in the model.

#### Borrow projection

Borrowed receivers aren't enough on their own. Consider the `RentedArray` type from the [worked examples](#worked-examples):

```csharp
public readonly struct RentedArray : IResource
{
    private readonly T[] _rented;

    public Span<T> Span => _rented;
}
```

The receiver of `Span` is a borrow, but `_rented` is a plain `T[]`, and a span created from a plain array has the lifetime `$heap`. Nothing ties the span back to `this`. It's worth noting that this isn't specific to ownership. The same thing happens today with a ref struct that holds a heap-backed span: a span returned from an instance member has the lifetime of the field's type, not the lifetime of the receiver.

The core problem is that once the array is read out of the field, its provenance is lost -- `T[]` has no place to record a lifetime. So instead we will change what reading the field produces. Reading a field through a borrow produces a borrow with the same lifetime. We call this _borrow projection_.

Concretely, suppose `e` is an expression of type `ref<$a> R`, `ref readonly<$a> R`, `ExclusiveBorrow<$a, R>`, or `Borrow<$a, R>`, or a borrow that was itself produced by projection, and `F` is a field of the target. Then `e.F` is typed as follows:

| Type of `F` | If `e` is exclusive | If `e` is shared |
| --- | --- | --- |
| Resource struct `F` | `ref<$a> F` | `ref readonly<$a> F` |
| Resource class `U` | `ExclusiveBorrow<$a, U>` | `Borrow<$a, U>` |
| Other reference type `U` | `Borrow<$a, U>` | `Borrow<$a, U>` |
| Struct that contains references | Accessed only by reference; its references are read by projection | Same |
| Type that contains no references | Its normal value | Same |

A few of these deserve explanation. For a resource struct field, `e.F` is a location with the lifetime `$a`. When `e` is a by-ref, this is exactly what the model already says for fields of structs. For a reference field that isn't a resource, the result is always a shared borrow, since the aliasing rules from [exclusivity](#exclusivity) only apply to resources. What can be done with the target is governed by `U`, just as for a `readonly` field of type `U` today. And a struct field that contains references can't be copied out of `e`, since copying it would copy out the references inside it without their lifetime.

Element reads through a borrow of an array follow the same rules, so the elements of a jagged array are borrows too.

Note that projection only depends on the type of `e`, not on the kind of member it appears in, so it applies equally in static members and nested types. Also note that projection never changes the lifetime of a location. Array elements are still `$heap` locations, and `ref<$heap> T` is converted to `ref<$a> T` by ordinary covariance.

With projection, the `Span` property has the full formal definition

```csharp
static Span<$ro, T> get_Span<$ro>(ref readonly<$ro> RentedArray @this)
{
    // okay: @this._rented is a Borrow<$ro, T[]>, which converts to Span<$ro, T>
    return @this._rented;
}
```

and every way of leaking the array becomes an ordinary type error:

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

Owned values are slightly different. When `e` is owned -- the receiver of `Drop`, an owning variable, or a parameter passed by value -- reading a non-resource field produces its normal value. Resource fields are still accessed by reference or as borrows, and can't be moved out, since the compiler-provided part of `Drop` drops them after the user-defined part runs.

There is one catch. If the resource type has a user-defined `Drop`, fields can only be read as normal values inside `Drop` itself. Anywhere else, the user-defined `Drop` would run when the owner leaves scope, while the value read from the field might still be in use.

#### Loan checking

The type system guarantees that no borrow outlives the location of its owner. Unfortunately, that isn't quite enough, because it doesn't guarantee that the owner still owns its value. Consider:

```csharp
var rented = pool.Rent(n);
Span<byte> span = rented.Span;   // Span<$local>: rented is loaned
Consume(rented);                 // error: rented may be loaned
span[0] = 1;
```

The type of `span` is `Span<$local>`, which is valid until the end of the block, so the type system has no problem with `span[0] = 1`. But by that point `Consume` has taken ownership of the rental, and may have already returned the array to the pool. Similarly, a member called through an exclusive borrow could reassign a resource field, dropping its old value while a borrow of it is still in scope.

To prevent this we will add a flow analysis called _loan checking_. Like definite assignment, it isn't part of the type system. Rather than rejecting the use of `span`, it rejects the transfer of `rented`, so there is never a point where a borrow of `rented` exists and `rented` no longer owns the array. If the transfer comes first, the borrow is rejected instead, since use after transfer is already disallowed.

Loan checking operates on _places_. A place is an owning variable, `this` in a member with a borrowed receiver, a parameter passed by `ref`, `in`, or as a borrow, or a resource field reached from one of these.

An expression _loans_ a place when it borrows the place, or something reached through it, and stores a value whose lifetime is bounded by that borrow -- in a local, a field, the target of an `out` or `ref` argument, or the method's return value. Borrowing a place means taking a `ref` to it, converting it to a borrow, reading through it by projection, or passing it as a borrowed receiver or argument. Body inference tells us whether a stored value is bounded by the borrow: the borrow constrains the `$ro` or `$cm` that inference chooses for a call, and the call's outputs either use that lifetime or they don't.

A borrow that only flows into calls that don't store anything ends with the call. So `ToUtf8(s, rented.Span)` does not loan `rented`, and neither does `builder[i] = i`. The loaned place is the last place of resource type on the path the borrow was taken through: `rented.Span` loans `rented`, `this._inner.Span` loans `this._inner`, and inside `RentedArray`, reading `_rented` loans `this`. A loan is shared if the borrow was shared (`ref readonly`, `in`, `Borrow`, or a call to a `readonly` member), and exclusive otherwise.

Note that loans are recorded at the expression that creates them, not inferred later from the types of variables. Covariance can convert a lifetime to a shorter one, so the type of a value doesn't always tell us which place it was borrowed from.

A place is _possibly loaned_ at a point in the method if some path through the method reaches that point after an expression that loans the place. The same analysis tracks whether a variable is _possibly transferred_. Where paths merge, including along loop back edges, the analysis takes the union:

```csharp
Span<byte> span = default;
if (condition)
{
    span = rented.Span;
}
Consume(rented);                 // error: rented may be loaned
```

A loan lasts until the end of the scope of its place, or until the end of the method for `this` and parameters. Two places _overlap_ when one is the other, or is reached through it. At a point where a place `p` is possibly loaned, the following rules apply to any place that overlaps `p`:

| Operation | `p` has a shared loan | `p` has an exclusive loan |
| --- | --- | --- |
| Read, or borrow shared | Allowed | Error |
| Borrow exclusively | Error | Error |
| Assign, dropping the previous value | Error | Error |
| Transfer | Error | Error |

Exclusive borrows are prohibited even under a shared loan because an exclusive borrow could reassign a resource field, dropping the target of the loan.

Since overlap is based on places, different resource fields of the same value don't overlap, and may be borrowed independently, even exclusively:

```csharp
struct Buffer : IResource
{
    public Span<int> GetSpan() { ... }   // not readonly, so it borrows exclusively
}

struct Pair : IResource
{
    Buffer A;
    Buffer B;

    void Copy()
    {
        Span<int> a = A.GetSpan();   // exclusive loan of this.A
        Span<int> b = B.GetSpan();   // okay: this.B doesn't overlap this.A
        a.CopyTo(b);
        M();                         // error: this overlaps this.A and this.B
    }

    void M() { }
}
```

The call to `M` borrows all of `this`, which overlaps both loaned fields. Similarly, a caller of a member that returns a borrow derived from one field sees a loan of the whole receiver, since it can't see which field the borrow came from.

Exceptions also create paths. An exception may be thrown at any point inside a `try` block, so a transfer or loan anywhere in the block is considered possibly done at the start of each `catch` and `finally` block. When an exception leaves the scope of an owning variable, the variable is dropped as usual, using its drop flag if it may have been transferred.

No check is needed when a place leaves scope. Every borrow of the place has a lifetime no longer than its block, so every borrow is out of scope too. Owning variables are dropped in reverse declaration order, since a resource declared after `v` may hold a borrow of `v` and use it in its `Drop`.

Lastly, loan checking is lexical. A loan lasts until the end of its place's scope, even if the values holding the borrow are no longer used. This rejects some safe programs:

```csharp
var rented = pool.Rent(n);
Span<byte> span = rented.Span;
ToUtf8(s, span);
Consume(rented);                 // error: rented is loaned, although span is no longer used
```

A more interesting case is a borrow that is returned on one path, while the other path needs the owner:

```csharp
static Borrow<Node> GetOrAdd(ref Map map, int key)
{
    if (map.TryGetBorrow(key, out var value))
    {
        return value;
    }

    map.Add(key, new Node());    // error: map may be loaned
    return map.GetBorrow(key);
}
```

`TryGetBorrow` writes a borrow of `map` to `value`, so it loans `map` on both paths, even though the borrow is only used on the path that returns. This is intentionally rejected. Rust's non-lexical lifetimes reject it as well, and only a more precise analysis, like Rust's experimental Polonius, can accept it. The workaround is to check without borrowing first:

```csharp
static Borrow<Node> GetOrAdd(ref Map map, int key)
{
    if (!map.ContainsKey(key))
    {
        map.Add(key, new Node());
    }
    return map.GetBorrow(key);
}
```

#### Safety

Putting these pieces together, the property that ownership is meant to guarantee is:

> In a program that type checks and passes loan checking, whenever a borrow is used -- including a by-ref or span derived from one -- the value it was borrowed from is still owned. It hasn't been transferred, reassigned, or dropped.

The argument has the following shape. Like the model's, it is a sketch rather than a proof, and it doesn't cover `unsafe` code or reflection.

- A borrow of an owner is created with the lifetime of the owner's location, or by projection with the lifetime of the borrow it was projected through. By the model's argument, conversions only shorten that lifetime, and the borrow can't be stored anywhere that outlives it. So every borrow of an owner is out of scope by the end of the owner's scope. For resource classes, this relies on the object only being reachable through its owner and borrows of it.
- Any borrow that outlives the expression that created it is recorded as a loan of the place it was taken through. Loans are recorded at creation, merged over every path, including exceptional ones, and last until the end of the place's scope. So wherever a borrow may be used, its place is possibly loaned.
- While a place is possibly loaned, neither it nor any overlapping place can be transferred or reassigned, and it can't be borrowed in a way that could reassign its resource fields. So from the loan to the end of the scope, the place keeps its value, and so do its resource fields.
- At the end of the scope, every borrow of the place is out of scope too, by the first point, so dropping it is safe. Reverse declaration order means that a resource holding a borrow of another is dropped first, while its borrow is still valid.
- Each value is dropped at most once. A transferred variable is not dropped, and drop flags settle the cases where it is only transferred on some paths.

#### Relationship to the ref safety model

To summarize, ownership adds to the [ref safety model](ref-safety-model.md) without changing it. The model's rules describe the types and locations that exist in C# today, and everything ownership introduces is new. `Borrow<$a, T>` and `ExclusiveBorrow<$a, T>` are ref structs with a single covariant lifetime parameter, which the model already describes. The field of a resource class is a new kind of location, with a new row in the location table. Borrow projection is a new typing rule, and apart from resource class fields, the lifetimes it produces come from the model's existing covariant conversions. The receivers of resource members, `[BorrowedReceiver]` members, and `Drop` are new cases in the signature translation, whose defaults the model leaves open. Owner lifetimes are just the lifetimes of locations, so the lifetimes in scope remain totally ordered. And loan checking is a separate flow analysis.

Notably, the rule that a value returned from an instance member of a resource type can't outlive `this` doesn't need to be stated at all. It falls out of the `ref<$ro>` receiver and borrow projection. The same reasoning covers `out` parameters, static members, and every other way a value can leave a member.

### Exclusivity

At any point, a resource value permits either:

- one exclusive borrow, or
- any number of shared borrows.

These states are mutually exclusive, and are enforced by [loan checking](#loan-checking), which also prevents the owner from being transferred or reassigned while it is borrowed. Note that these rules only apply to resources. Borrows of non-resource values, like the ones produced by [borrow projection](#borrow-projection), are always shared and just carry a lifetime.

It's important to note that the difference between the two kinds of borrow is uniqueness, not mutability. Mutability is a complicated topic in C#, with `readonly` only applying to the outer layer of a value. Instead, what matters for ownership is whether a borrow can invalidate a value that someone else is using. An exclusive borrow is the only way to access the value while it exists, which makes it safe to reassign the value's fields -- and, by doing so, drop its resource fields. A shared borrow may coexist with others, so its holder may not reassign the value's fields, or one holder could drop a resource field that another is still using. That restriction is exactly C#'s existing, shallow `readonly`. Neither kind of borrow says anything about the contents of non-resource fields, which may be modified through either one, as governed by their own types.

To accommodate shared borrows we also have to adjust the rules for classes themselves. The `readonly` keyword will now be legal for class methods, just like struct methods. It will have the same rules. An ordinary (non-readonly) instance method of a resource class will have an `ExclusiveBorrow<this>` receiver. A `readonly` instance member has a `Borrow<this>` receiver. Importantly, shared borrows of a resource may only call readonly members. For structs, the same distinction is spelled `ref` for an exclusive borrow and `ref readonly` for a shared one.

Note that the `readonly` restriction only exists to protect resource fields, so it doesn't apply to classes that aren't resources. They can't contain resource fields, so reassigning their fields can't drop anything. A shared borrow of such a class may call any member with a borrowed receiver, `readonly` or not -- this is just the interior mutability that C# has always had. For these classes, the meaningful choice is between shared and exclusive receivers, as described in [bridging the worlds](#bridging-the-worlds).

Since regular references can't be converted to exclusive borrows, an `ExclusiveBorrow<T>` always comes from an owner -- a resource, or an `Owned<T>` -- and code that receives one can rely on it being the only access to the value. Among other things, this means nothing else can access the value's fields while the borrow exists, including from another thread. Note that this only covers the value's own fields, not other objects that they refer to, unless those are owned as well.

### Bridging the worlds

If owned resources and GC types do not cooperate they will force parallel worlds to be created and code to be duplicated. Fortunately, the restrictions are smaller than they appear. The biggest problem with allowing regular GC references to appear in `Borrow`, like owned resources, is that the methods on the GC type do not internally follow ownership rules. Fundamentally, this is about the receiver. In classes, the receiver type is the parent type -- meaning a normal GC tracked reference. For normal GC references, the lifetime of `this` is not tracked and may escape to any other part of the program.

The solution is to allow individual methods of classes to opt-in to having _borrowed receivers_. This feature requires a new attribute:

```C#
public sealed class BorrowedReceiver : System.Attribute
{
    public bool Exclusive { get; init; }
}
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

Like all borrows, it would be illegal to convert a borrowed reference to a GC reference. Thus, `M1` is restricted from escaping the receiver outside the current method. Inside a `[BorrowedReceiver]` member, the fields of `this` are read by [borrow projection](#borrow-projection), so the member can't leak references to the object's contents either. This cleanly allows arbitrary class types to appear as borrowed types. `BorrowedReceiver` is also legal on constructors -- because constructors can access `this` they have the same risk of exposing it outside the constructor method, unless it is typed as a borrow. A constructor's receiver is always exclusive, since the object is new and nothing else can refer to it yet.

Note that the receiver of `M1` is a shared borrow, even though `M1` isn't `readonly`. That's what lets `M1` be called on a regular GC reference, which can only be converted to a shared borrow. Since `C` isn't a resource, this is safe -- nothing in `C` can be dropped, so mutating it through a shared borrow can't invalidate anything another holder is using.

Some members need more than that. A member whose correctness depends on nothing else accessing the object at the same time -- because it temporarily breaks an invariant, or because a data race would corrupt its state -- can ask for an exclusive receiver instead:

```C#
class C
{
    [BorrowedReceiver(Exclusive = true)]
    public void M3() { ... }
}

// Maps to

class C
{
    public static void M3<$ro>(ExclusiveBorrow<$ro, C> @this) { ... }
}
```

Such a member can't be called on a regular GC reference, since there is no conversion to an exclusive borrow. It can only be called through an owner, like the `Owned<C>` described below. In exchange, it can rely on its receiver being the only access to the object. (The attribute is a placeholder until dedicated syntax for borrows is considered.)

This is very similar to how Rust handles reference-counted objects. A Rust `Rc<T>` only provides shared references, `&T`, since other `Rc`s to the same value may exist, and getting a `&mut T` requires a runtime check that the count is one. A regular C# reference is the same -- it only converts to a shared borrow. The difference is what can be done through a shared reference. In Rust, any mutation can invalidate memory, like a `Vec` reallocating its buffer, so mutating through an `Rc` requires interior mutability types like `RefCell`. In C#, mutating a non-resource object never invalidates memory, since the GC keeps everything alive. The only thing that can invalidate a value is dropping a resource, and non-resource types can't contain resources. So C# can allow mutation through shared borrows of non-resource classes, which effectively gives every GC object interior mutability for free.

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

Members with exclusive receivers can only be called through an exclusive borrow of the `Owned<T>`. `Owned` therefore allows individual instantiations to control whether they are GC-tracked or borrowed. `string` would remain GC-tracked by default, but an `Owned<string>` could also be created that would follow ownership rules. This may provide performance advantages in certain scenarios. This also does not lift all restrictions related to ownership: optionally-GC-tracked types may never contain fields of a resource type.

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

Creating a span from an ordinary array would normally give the span a heap lifetime. However, `Span` is a member of a resource type, so its receiver is `ref readonly<$ro> RentedArray`, and reading `_rented` through it produces a `Borrow<$ro, T[]>` (see [borrow projection](#borrow-projection)). The span is therefore a `Span<$ro, T>`, which can't outlive the `RentedArray` it came from. The same rule prevents the array from leaking any other way -- since there's no conversion from the borrow to `T[]`, the array can't be returned, stored in a field, or converted to `Memory<T>`.

`Drop`, on the other hand, has an owned receiver. It reads `_rented` and `_pool` as normal values, so it can return the array to the pool. (Outside of `Drop`, `_pool` would also be read as a borrow.)

Note that this only covers references obtained through a `RentedArray`, not references kept by whoever created it. The constructor receives a regular array, so it's up to the author of the type to make sure no other aliases exist. In this case the constructor is private and is only called with an array fresh from the pool.

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

This is the classic "stackalloc a span if small, or use a rented array if large" pattern. In both cases the usage is local and simple. Unlike direct use of `ArrayPool`, there is no risk of forgetting to return the array or returning it twice. `Return` is not exposed to the renter. Instead, the `RentedArray` is automatically dropped at the end of its scope, and its `Drop` implementation returns the array to the pool. Access is limited to a `Span` whose lifetime is tied to the `RentedArray`, so the span cannot outlive the rental. Since `ToUtf8` doesn't store the span, the borrow of `rented` ends with the call. If we had stored the span in a local, `rented` would be loaned for the rest of its scope, and [loan checking](#loan-checking) would reject any attempt to transfer it.

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

Note how projection applies here. The indexer's receiver is `ref<$ro> OwnedImmutableBuilder<T>`, so it reads `_backing` as a `Borrow<$ro, T[]>`, which it can use to read and write elements. `ToImmutable`, on the other hand, owns `builder`, so it reads `_backing` as a normal array. This is only allowed because `OwnedImmutableBuilder<T>` has no user-defined `Drop`. If it had one -- say, a `Drop` that cleared the array -- reading the field as a normal value would be an error, since that `Drop` would run when `builder` leaves scope at the end of `ToImmutable`, after the array had already been handed to the `ImmutableArray`.

## Open questions

- [ ] Shared fields. Projection applies to every reference field of a resource, even fields that refer to shared objects rather than owned ones, like `_pool` in `RentedArray`, or a `string`. Since members can only be called through a borrow if they have borrowed receivers, these fields are hard to use outside of `Drop`. We could add a per-field opt-out. However, projection should stay the default -- a field whose author forgot to opt in is exactly how an owned array leaks.
- [ ] Existing APIs. A borrow can't be passed to an existing method that takes `T[]` or another class type. We could extend `[BorrowedReceiver]` to parameters, typing them as `Borrow<$ro, T>`, or `Borrow<$local, T>` when `scoped`. Since a regular reference converts to `Borrow<$heap, T>`, annotating a parameter this way wouldn't break existing callers. The implicit conversion from `T[]` to `Span<T>` would be a good candidate.
- [ ] Struct fields that contain references. Projection makes these accessible only by reference, so a field of type `ImmutableArray<T>` can't be copied out of a resource. Calling its members would require borrowed receivers on structs.
- [ ] Generic substitution. Resource types can't be substituted for type parameters, but ref structs can be substituted for type parameters that `allows ref struct`. A similar anti-constraint could allow resources. Generic code would then have to treat the type parameter as a resource -- its values would be transferred rather than copied, dropped at the end of their scope, and have borrowed receivers. If the type argument can be lifetime-bearing, the type parameter would carry a lifetime, just as the [ref safety model](ref-safety-model.md) already describes for `allows ref struct`.
- [ ] Data races. An exclusive borrow guarantees that nothing else can access the value's own fields, which rules out data races on them. Two gaps remain. First, the guarantee isn't transitive: other objects that the fields refer to may be shared, unless they are owned too. This interacts with the shared fields question above. Second, resources currently can't cross threads at all, since they can't be captured, stored in static fields, or substituted for type parameters, so they are race-free by construction. Once resources can cross threads, through generics or async, we'll need rules similar to Rust's `Send` and `Sync`: transferring an owned resource to another thread is fine, but sharing a borrow across threads is only safe if the target's shared-borrow members are thread-safe.
- [ ] Depth of `readonly`. Projection keeps the shallow `readonly` of C#, so a `readonly` member can return a `Span<T>` over an array field. If shared borrows should also prevent writes to non-resource contents, projection through a shared borrow would have to produce read-only results, and `RentedArray` could only expose a `ReadOnlySpan<T>`.
- [ ] Loan duration. Loans last until the end of their place's scope. Ending a loan when the variables holding its borrows go out of scope would accept more programs, but loan checking would then need to track which variables hold borrowed values.
- [ ] Arrays of resources holding borrows. By the same argument as resource classes, the elements of an array of resources could be given the lifetime of the array instead of `$heap`. This would be another new row in the location table.
- [ ] Non-lexical lifetimes. Ending an owner's lifetime where it is transferred would accept more programs, but it's a larger extension than the others. Body inference in the [ref safety model](ref-safety-model.md) relies on the lifetimes in scope being totally ordered, and a lifetime that ends partway through a block breaks that. Inference would need to choose the longest lifetime no longer than each constraint, rather than simply the shortest constraint, and we'd need a rule that a value is only used within the lifetimes in its type. This wouldn't break existing code, though. Only owners would have such lifetimes, so in a program without resource types the lifetimes remain totally ordered, and the generalized inference chooses the same lifetimes as today.

## Acknowledgements

1. Walker, David (2002). "Substructural Type Systems"
2. The Rust programming language for general understanding of ergonomics
3. Special thanks to [boats](https://without.boats/)), who's writing has heavily influenced my framing of how to think of a GC hybrid language
