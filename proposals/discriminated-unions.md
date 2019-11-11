
# `enum class`

`enum class`es are a new kind of type declaration, sometimes referred to as discriminated unions,
where each every possible instance the type is listed, and each instance is non-overlapping.

An `enum class` is defined using the following syntax:

```
enum_class
    : 'enum class' <identifier> '{' <enum_class_case> '}'

```

Sample syntax:

```C#
enum class Shape
{
    Rectangle(float Width, float Length),
    Circle(float Radius),
}
```

## Semantics

An `enum class` defines a root type, which is abstract, and a set of non-overlapping type
members, each of which must inherit from the root type of the `enum class`. 

### Switch expression

The switch expression is specified to produce a warning if it cannot be determined that
the switch is exhaustive. For `enum class`es, the switch expression is considered exhaustive
if the input type is the `enum class` type and there is an arm for every case type.


## Implementation

An `enum class` is syntax sugar for a type hierarchy based on nested classes. The `enum class`
root type is an abstract class with an internal constructor taking no arguments. Each of the
type members is a nested record class which inherits from the root.