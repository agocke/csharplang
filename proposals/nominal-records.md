
# Nominal Records

* [x] Proposed
* [ ] Prototype: 
* [ ] Implementation: 
* [ ] Specification: 

## Summary
[summary]: #summary

Nominal records are a companion proposal to the existing [records](records.md) proposal which add a new
class and struct modifier, `data`.

The addition of the `data` modifier would:

1. Automatically generate `Equals`, `GetHashCode`, `ToString`, `==`, `!=`, and `IEquatable<T>`
   based on the member data of the type.
1. Allow object initializers to also initialize readonly members.

## Motivation
[motivation]: #motivation

The exiting records proposal (hereafter "positional records") address an important use case, but have
two major problems:

1. They're very different from the classes used as records right now.
2. Adding or re-ordering members is almost always a binary and source breaking change.

Nominal records try to satify many of the same goals as positional records, primarily automatic value
semantics (generated value-based equality). However, they focus on augmenting existing
language constructs used for records, like the following:

```C#
public class LoginResource
{
    public string Username { get; set; }
    public string Password { get; set; }
    public bool RememberMe { get; set; } = false;
}
```

In this case, rewriting this class as a positional record has a few problems. First is that the syntax
is quite different, e.g.

```C#
public class LoginResource(
    string Username,
    string Password,
    bool RememberMe = false
);
```

This isn't too bad of a change for three members, but it's much worse for types like ones used to hold
10+ members, like command-line option types.

Second, this introduces a significant change for consumers of the API. Previously, they would create
the type as follows:

```C#
var x = new LoginResource {
    Username = "andy",
    Password = password
};
```

Now they would have to use a constructor. The biggest problem there is that it would be easy to forget
to use a named parameter, causing any reordering of members in the record to be a breaking change to the
consumer. In essence, adopting positional records brings benefits, but also a number of concerns that
didn't exist before.

## Detailed design
[design]: #detailed-design

Positional records tries to 1) meet people where they are and 2) create a robust story for evolving records
in public APIs. On the declaration side, the only addition is a new modifier, `data`:

```C#
public class LoginResource(
    string Username,
    string Password,
    bool RememberMe = false
);
```

On the consumption side, everything looks the same if they were using an object initializer.

There are three large semantic changes that `data` implies to make these things work:

1. Automatic generation of equality based on members
2. Read-only member initialization
3. With-expression support to allow easily constructing new records from existing instances

### Member equality

First, the generation of equality support. Data members are only public
fields and auto-properties. This allows data classes to have private
implementation details without giving up simple equality semantics. There are
a few places this could be problematic. For instance, only auto-properties
are considered data members by default, but it's not uncommon to have some
simple validation included in a property getter that does not meaningfully
change the semantics, e.g.

```C#
{
    ...
    private int _field;
    public int Field
    {
        get
        {
            Debug.Assert(_field >= 0);
            return _field;
        }
        set { ... }
    }
}
```

To support these cases and provide an easy escape hatch, I propose a
new attribute, `DataMemberAttribute` with a boolean flag argument on the
constructor. This allows users to override the normal behavior and include
or exclude extra members in equality. The previous example would now read:

```C#
{
    ...
    private int _field;

    [DataMember(true)]
    public int Field
    {
        get
        {
            Debug.Assert(_field >= 0);
            return _field;
        }
    }
}
```

Equality itself would be defined in terms of its data members. A `data` type
is equal to another `data` type when there is an implicit conversion between
the target type and the source type and each of the corresponding members
are equal. The members are compared by `==` if it is available. Otherwise,
the method `Equals` is tried according to overload resolution rules (st. an
`Equals` method with an identity conversion to the target type is preferred
over the virtual `Equals(object)` method).

- [ ] **Open Issue**: Is it better to just call
    `EqualityComparer<T>.Default.Equals()`?

There is also one hidden data member, `protected virtual Type
EqualityContractOrigin { get; }`, that is always considered in equality. By
default this member always returns the static type of its containing type,
i.e. `typeof(Containing)`. This means that sub-classes are not, by default,
considered equal to their base classes, or vice versa. This also ensures that
equality is commutative and `GetHashCode` matches the results of `Equals`.
These methods are virtual, so they can be overridden, but then it is the
user's responsibility to ensure that they abide by the appropriate contract.

`GetHashCode` would be implemented by calling `GetHashCode` on each of
the data members.

### Read-only member initialization

Since one of the primary goals of this proposal is to prevent positional
dependency, constructor parameters are inherently positional, and readonly
members must only be assigned in a constructor according to CLR rules, we
need an intermediary type for readonly member initialization. This
intermediary is referred to as the "builder" for the type. Here's what a
builder would look like in generated code:

```C#
public class LoginResource
{
    public struct Builder
    {
        public string Username;
        public string Password;
        public bool RememberMe;

        public static Builder Create()
        {
            var builder = new RecordBuilder();
            builder.RememberMe = false;
            return builder;
        }
    }

    private readonly Builder <>ReadonlyFields;

    public LoginResource() {}

    public LoginResource(Builder builder)
    {
        <>ReadonlyFields = builder;
    }

    public string Username => <>ReadonlyFields.Username;
    public string Password => <>ReadonlyFields.Password;
    public string RememberMe => <>ReadonlyFields.RememberMe;
}
```

and here is what the previous example would generate on the creator side:

```C#
var builder = LoginResource.Builder.Create();
builder.Username = "andy"
builder.Password = password;
var resource = new LoginResource(builder);
```

There are may details here to cover. Here are the new items created:

1. A new builder type and a private field for the builder type
2. A new constructor for the builder
3. Properties for the readonly members
4. Recognition of the builder and code generation by the object initializer

We'll cover each individually.

#### 1. Generating the builder type

The builder type generated is always a struct containing public fields
corresponding to all members in the record. It contains a static `Create`
method that creates a new default builder, executes the field initializers
for the record, and then returns the builder. This provides a place for
field initializers to run without overwriting the object initializer code.
A readonly field of the builder type is created inside the record, to be
initialized by the constructor.

#### 2. A new constructor for the builder

There are two design goals here: allow a simple type to always have a
constuctor generated, but let users opt-in or -out of readonly initialization
if they have some different initialization contract.

To satisfy the first requirement, a constructor with a builder (a "builder
constructor") is always generated in place of the default empty constructor.
However, if there is a user-defined constructor, the empty builder constructor
is still generated, but it is private. This allows users to create their own
constructors with builders and then use the `this()` call to initialize readonly
members. For example,

```C#
public data class LoginResource
{
    ...
    public LoginResource(bool extraInfo, LoginResource.Builder builder)
      : this(builder)
    {
        ...
    }
    ...
}
```

The object initializer pattern for recognizing builder constructors is
described later.

### 3. Properties for readonly members

To prevent data duplication and metadata bloat, the backing data for public
readonly data members in nominal records is stored in the builder type
directly. To allow public access to that data, properties are generated which
delegate to the builder field. For fields of records, which odn't have
backing fields, the field itself is still moved into the builder, but instead
of an ordinary property being generated on the record, a `ref
readonly`-returning property is generated instead. So as not to create subtle
differences between readonly and mutable fields, this is true for *all*
fields in nominal records: they are all generated as `ref`-returning
auto-properties.

- [ ] **Open issue:** Find places where this is observable aside from
reflection

### 4. Object initializer builder binding

Object initializer syntax functions by looking at the type and seeing if it
has a suitable builder type (the builder may need an attribute to prevent a
breaking change). If so, the constructor call is bound *without* the builder
argument, which is required to be the last argument, if an object initializer
is present. Overload resolution then proceeds as normal. The generated code

1. Calls `builder.Create()` to get a builder.
1. Assigns all builder fields using the names and values from the object
   initializer.
1. Calls the selected constructor with the constructed builder as the last
   argument.

### With-expression support

`with` expressions are proposed in the positional record proposal and in that
proposal the `with` expression is mostly syntactic sugar for a call to a
`With` method that contains arguments for each data member. Nominal records
are similar, but they cannot use arguments for data members because that
would create a dependency on argument order. Instead, nominal records re-use
the builder pattern:

```C#
public class LoginResource
{
    ...
    public LoginResource With(Builder builder)
    {
        var newResource = new LoginResource(builder);
        // Copy over any mutable data members
        return newResource;
    }
}
```

The creating side would look like:

```C#
var builder = existingResource.<>PrivateFields;
builder.Password = newPassword;
var newResource = existingResource.With(builder);
// Assign mutable members
```

Notably, because there is extra code emitted on the creation side, this
feature actually does require a new `with` expression form, unlike the
positional records, which propose just using the `With` methods.

- [ ] **Open issue:** This design requires making the <>PrivateFields
  field public. Should the field be unspeakable to prevent public API
  pollution? Or should this be part of a public pattern?

## Drawbacks
[drawbacks]: #drawbacks

One records proposal is simpler than two records proposals. One can also see
a fair bit of overlap in use cases. Does the space really merit two
solutions? How well do the two solutions compose with each other?

## Alternatives
[alternatives]: #alternatives

The primary alternative was to just do positional records and do nothing for
the use cases of nominal records.

## Unresolved questions
[unresolved]: #unresolved-questions

- The builder pattern is mostly public. This could introuce a fair bit of API
  pollution. Do we want to hide more things as compiler implementation details,
  at the expense of user customization?
