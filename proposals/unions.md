
# Fast, Efficient Unions

Many types of unions, including discriminated unions, have been proposed for C#. This proposal, unlike others that came before, is focused on building a new, efficient primitive for C# and .NET. As of now, C# and .NET have lacked a mechanism for expressing any kind of type union, meaning an "or" relationship between types.

While developers might reasonably prefer a wide variety of designs for union types depending on their domain, as a language and type system it should be a goal to build broad, useful primitives on which many separate complex structures could be built.

To that end, any union primitive we construct should flexible and as efficient in both space and computation.

Given this constraint, we know that we want a struct-based solution. There are also two union designs which stand out as good candidates.

The first is what we might call a "pure" union -- a combination of multiple types in an "or" relationship. For this we could introduce a pure "or" type-level combinator, which for this proposal we will call `|` and use in combination with type variables in the form `(A|B|C)`.

The second design is what is usually described as a "discriminated" or "tagged" union. In this design we would define a new type declaration form to introduce the union, and a series of cases which delimit precisely the set of types that comprise the state of the union when in that case. For instance,

```csharp
enum struct E
{
    Case1(A),
    Case2(B),
    Case3(C),
}
```

It should be noted that, although only the second form is described as a "tagged" union, in fact the "pure" union is tagged as well, since all values carry type tags in the .NET runtime and can be queried through type tests. The usefulness and usability of each tagging system is covered in more detail later.

## Semantics

To discuss the different designs usefully, we must assign semantics to the different designs. These semantics could be debated, but I will do my best to define what I see as the most "generous" capabilities in each design, without regard for implementation. When implementation is covered later, we will note problems and limitations.

To start we may note the similarities in both union designs. Both provide the ability to represent multiple values of different types in a single value and single type specification.

### Pure unions

In the case of pure unions, the inline `|` combinator provides the ability to represent anonymous unions directly in type syntax. This is paired with a trivial introduction form -- a value of pure union type may be created simply by assigning a value of one of its cases to it. This makes pure unions particularly good at *ad hoc* modeling. In any type location a pure union could be provided, and in general, no additional code is required to create values of that type, either in the method return position, method argument position, or local variable assignment position.

On the other hand, while pure unions are simple for *ad hoc* modeling, they may not be as effective for larger scale programs. While it is useful that the `|` combinator does not require a type declaration, it can be problematic that it also doesn't allow for one. Consider that C# variable names are rarely pithy single characters like `A` or `B` and in fact are usually quite long. `A|B` looks good in examples but,

```csharp
string|List<string>|Dictionary<string, string> M(string|List<string>|Dictionary<string, string> u)
{ ... }
```

isn't quite as attractive or readable. The inability to declare unions using the pure union syntax is made more complicated by the fact that C# doesn't have a particularly good way of declaring new type aliases. The best answer is the `using` directive, which cannot be public, cannot represent open generics, can only appear at the top of a file, and often require specifying the fully qualified type name of any alias targets. While these limitations could be changed or lessened, these changes are not currently proposed -- partially because C# has historically not preferred this style of programming over direct declarations.

Aside from declaration simplicity, the union `|` combinator is also conceptually simple and familiar. Many C# programmers are already familiar with the `|` value combinator and its extension to the type level is mostly natural. The type-level combinator is commutative and associative, just like its value form. Subtyping also follows the same simple set theory rules that hold true of the set "union" combinator, which many developers will be familiar with, e.g. `A|B` is a subtype of `A|B|C`, as is `C|A`.

Lastly, the elimination form is also simple and familiar -- dynamic type checking through pattern matching. To deconstruct a union `A|B` we can use simple pattern matching to check for each of the cases, i.e.

```csharp
void M<A, B>(A|B union)
{
    if (union is A a)
    {
        ...
    }
    if (union is B b)
    {
        ...
    }
}
```

or with a switch expression

```csharp
void M<A, B>(A|B union)
{
    var x = union switch {
        A a => ...,
        B b => ...
    };
}
```

In the second case, we are even automatically provided exhaustiveness checking, since all cases of the union have been handled in the switch.

Unfortunately, the simplicity defined above does come with some significant downsides. In particular, the flexibility of `|`, including allowing generic type parameters as cases, prevents unions from guaranteeing disjointedness. That is, for any union of the form `A|B`, there is no guarantee that `A` and `B` do not have a subtype or identity relationship. For instance, if the method `M` in the previous example were instantiated as `M<object, string>`, the union of `A|B` would be substituted as `object|string`. This creates potential problems for uses of the elmination forms, as case `B` would never be executed in this situation. While the switch expression in the second example would continue to check for exhaustiveness in this case, it would not check for subsumption.

### Tagged unions

In constrast with pure unions, tagged unions provide only a declaration form, not an inline form. The benefits and drawbacks are almost the mirror image of the pure union cases described above. A tagged union is declared via the syntax `enum struct`, deliberately reminiscent of simple C# `enum`s. Inside the body of the `enum struct` is a series of cases. Each case begins with a case name and is followed by a comma-separated list of types, similar to a tuple type:

```csharp
enum struct S
{
    Case1(A),
    Case2(A, B)
}
```

Unlike C# enums, the case names don't directly represent values, instead they are simply names used in the introduction and elimination forms. For the introduction form, each case name corresponds to a public static method on the `enum struct` type. This method is defined to have a parameter list with parameter types and arities that match the list following the case name in the `enum struct` declaration.

For instance, in the above example a valid expression would be `S.Case1(a)`, where `a` is a variable of type `A`. The result of this expression would be a value of type `S`.

To use the values passed into case methods on `S`, there is a corresponding elimination form in pattern matching to retrieve the values, similar to deconstruction. The case name once again appears as a new language construct, also nested under the enum struct name scope, which we might simply call a "variant." Variants may appear only as patterns, in the same position as a type pattern. Following the variant, there may be a deconstruction pattern which matches the arity and types of the original parameter list to the enum struct case. For instance,

```csharp
if (x is S.Case1(A a))
{

}
if (x is S.Case2(A a, B b))
{

}
```

or

```csharp
var y = x switch {
    S.Case1(A a) => ...
    S.Case2(A a, B b) => ...
}
```

Like pure unions, the above elimination forms guarantee exhuastiveness. Unlike pure unions, however, tagged unions are guaranteed to be disjoint and therefore they also guarantee subsumption. Also unlike pure unions, they do not have a "transparent" form, which means that they cannot be eliminated without using the new, variant-case elimination form.

What may be most notable about the above formulation is that, while pure unions may be conceptually familiar, tagged unions should be particularly familiar to C# programmers, as they behave quite similarly to traditional C# `enum`s (thus the choice of similar syntax). Like enums, tagged unions have only a declaration form. Like enums, they have a series of named cases. Like enums these cases can be switched on. The primary difference is that each enum struct case carries values of arbitrary different types, while all traditional enums are guaranteed to have a single value of one shared underlying primitive type.

## Implementations

### Pure unions

For pure unions, there is one obvious encoding to the current CLR type system. For any union type `(T1|T2|T3|...Tn)`, this could be translated to an instantiation of a predefined struct type `OneOf<T1, T2, T3, ... Tn>`. This fits well because:

- **Sharing a type can remove duplicate declarations**. The `(A|B)` syntax is a type syntax, not a declaration syntax. Therefore you would expect the same type to repeated often throughout signatures. If each term were to create a different type declaration, this could incur large compile- and run-time size costs.
- **Sharing a type makes conversions simpler.** If each `A|B` type syntax corresponded to a different CLR type, then conversion between variables could be costly.

That said, our optimizations are limited. In particular, because CLR structs lack variance capabilities, we cannot cheaply implement the subtyping relationship that we described in the semantics above. Instead we would be forced to emit a series of type checks for each case, converting each one individually to the new type. This is a small, but non-zero cost. We would have to provide the same trick for case reordering, as the CLR would see `OneOf<A, B>` and `OneOf<B, A>` as different types, despite our desire that `A|B` and `B|A` represent the same type.

This is particularly unfortunate in places where the CLR forces nested type equivalence, like in assigning `List<OneOf<A, B>>` to `List<OneOf<B, A>>`. Here such a conversion would be particularly expensive, likely forcing the allocation of new backing memory.

### Tagged unions

Tagged unions would preferably be implemented using standard struct types, with some optimizations. Every enum struct would need to contain a tag, to enforce our guarantee of disjointedness. It would also need to contain fields for each type in each type list in each case. Lastly it would also need to contain methods for creating instances of the enum struct. A declaration like

```csharp
enum struct S
{
    Case1(A),
    Case2(A, B)
}
```

may look like

```csharp
readonly struct S
{
    public readonly byte Tag;
    public readonly A <>Case1_1;
    public readonly A <>Case2_1;
    public readonly B <>Case2_2;

    public static S Case1(A a) => new S(1, a, default, default);
    public static S Case2(A a, B b) => (2, default, a, b);

    private S(byte tag, A case1_1, A case2_1, B case2_2)
    {
        Tag = tag;
        <>Case1_1 = case1_1;
        <>Case2_1 = case2_1;
        <>Case2_2 = case2_2;
    }
}
```

and the elimination form `x is Case2(A a, B b)` would look like

```csharp
x.Tag == 2 && (x.<>Case2_1, x.<>Case2_2 is (A a, B b))
```

In general, most operations should be simple operations on the larger struct, since cases cannot be represented directly, only held inside a value of the parent type, or deconstructed to constituents. Improvements on basic structs would likely be left to the runtime.


## Space efficiency

First, let's consider the space efficiency of possible implementations of the options. For all unions, space efficiency is defined relative to a tuple type with each of the cases, since tuples can represent all possible values of all possible union cases simultaneously.

For pure unions, any space optimization comes from runtime optimization. Since each type parameter in `OneOf` may be an incompatible overlapping type, the runtime will have to be responsible for special-casing the `OneOf` type to provide better space utilization.

For tagged unions, this same optimization strategy could be employed, or the C# compiler could participate. Since certain types and members would be known to be overlap-compatible, the compiler could emit `FieldOffset` as appropriate to optimize space. Space usage of tagged unions may be uniformly higher since they may be forced to always include an extra tag field.

## Computation efficiency

As mentioned previously, pure unions may be forced to do more individual type checks and assignment to provide conversion semantics.

Tagged unions likely will not need any such overhead. Most overhead will likely come from checking the tag. If tag checking is cheaper than type checkking, tagged unions may be slightly more efficient, or vice versa.

## Conclusion

Overall, it seems that both proposals have significant advantages and disadvantages.

However, tagged unions appear to initially fit better with C#. The syntax and semantic forms align more closely with the way C# has been designed and how it is widely used. As an analogy, we might note that tuples are to pure unions as structs/classes are to tagged unions. While tuples are a valuable language feature with unique advantages, their usage pales in comparison to structs and classes. C# is simply optimized for the kind of programming that favors structs and classes.

Indeed, one of the biggest weakness of pure unions appears to be that many of the "elegant" parts of their semantics would be difficult to efficiently implement in C#. With these disadvantages, the field is tilted even more towards tagged unions.

However, in the same way that tuples and structs/classes are both part of C# and provide unique advantages, it might be worthwhile to consider adding both designs.

If we must add only one, or prioritize one, then tagged unions should be preferred.