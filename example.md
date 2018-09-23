### Wise enum: a C++ smart enum for all your reflective enum needs

#### Nir Friedman


### Another one?

A common problem, must be tons of great solutions already, right?


### Another one?

Nope!

Note:
- Surprisingly few, maintained, polished, self contained (or even not; e.g.
  boost surprisingly doesn't have a solution for this)


### Define problem

 - Convert enum to string
 - String to enum
 - Number of enumerators
 - Iterate over enumerators


### Solution Approach

 - Alas, we're still reflection-less
 - So, have to generate code
 - Explicit codegen:
   - requires manually rerunning scripts, OR
   - build system integration
 - So... macros?


### Examples - Declare

```
// Equivalent to enum Color {GREEN = 2, RED};
WISE_ENUM(Color, (GREEN, 2), RED)
static_assert(wise_enum::size<Color> == 2, "");
```


### Examples - Iterate

```
std::cerr << "Enum values and names:\n";
for (auto e : wise_enum::range<Color>) {
  std::cerr << static_cast<int>(e.value) << " " << e.name << "\n";
}
```


### Examples - Convert

```
// Convert any enum to a string
std::cerr << wise_enum::to_string(Color::RED) << "\n";

// Convert any string to an optional<enum>
auto x1 = wise_enum::from_string<Color>("GREEN");
auto x2 = wise_enum::from_string<Color>("Greeeeeeen");

assert(x1.value() == Color::GREEN);
assert(!x2);
```


### Examples - Traits and TMP

```
static_assert(wise_enum::is_wise_enum_v<my_lib::Color>, "");
```


### How it works

```
WISE_ENUM(Color, (GREEN, 2), RED)
->
enum Color {GREEN = 2, RED};
constexpr std::array<::wise_enum::detail::value_and_name<Color>, 2>
wise_enum_detail_array(::wise_enum::detail::Tag<name>) {
  return { {GREEN, "GREEN"}, {RED, "RED"} };
}
```


### Why it's great
 - Idiomatic 11, 14, 17 support
 - The macro declares an actual enum, not an enum-like class
 - Supports full range of declaration functionality
 - Adapter for 3rd part enums
 - Constexpr to the max
 - No heap allocations, dynamic initialization, or exceptions
 - Good performance; switch case for enum -> string for perf


### Conclusion
 - More fun stuff coming: switch-case visitor, enum sets
 - Use it!
